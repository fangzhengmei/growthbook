# 实验结果快照缓存机制分析

## 目录

1. [快照生成流程](#1-快照生成流程)
2. [缓存键设计与查询匹配](#2-缓存键设计与查询匹配)
3. [缓存刷新触发机制](#3-缓存刷新触发机制)
4. [缓存 vs 重跑决策逻辑](#4-缓存-vs-重跑决策逻辑)
5. [核心代码位置](#5-核心代码位置)

---

## 1. 快照生成流程

### 1.1 快照生成入口

快照生成有两个主要入口：

**1. 手动触发 (API)**
- 文件：`packages/back-end/src/api/experiments/postExperimentSnapshot.ts`
- 调用链：`POST /api/v1/experiments/:id/snapshots` → `createExperimentSnapshot()`

**2. 定时任务 (自动刷新)**
- 文件：`packages/back-end/src/jobs/updateExperimentResults.ts`
- 调度频率：每10分钟检查一次需要更新的实验
- 默认更新间隔：6小时（可通过 `EXPERIMENT_REFRESH_FREQUENCY` 配置）

### 1.2 快照生成核心流程

```
createExperimentSnapshot()
    ↓
planSnapshot()  # 规划阶段
    ├─→ 确定快照类型 (standard/exploratory/report)
    ├─→ 确定运行器类型 (results/incremental/incremental-exploratory)
    ├─→ 决定是否使用缓存 (useCache)
    └─→ 决定是否全量刷新 (fullRefresh)
    ↓
createSnapshotFromPlan()  # 执行阶段
    ├─→ 获取增量刷新锁（如适用）
    ├─→ 创建快照数据库记录
    ├─→ 初始化查询运行器 (QueryRunner)
    └─→ 启动分析查询
    ↓
startQueries()  # 生成并执行SQL查询
    ├─→ 对每个查询尝试缓存复用
    └─→ 未命中缓存则创建新查询
    ↓
runAnalysis()  # 统计分析
    └─→ analyzeExperimentResults()
```

### 1.3 快照分析块 (Chunked Analyses)

为了支持大快照的高效存储和并发写入，快照分析结果采用分块存储：

**编码流程** (`packages/shared/src/snapshot-analysis-chunks.ts`):
1. 每个分析分配唯一的 `analysisKey`（格式：`an_` + 唯一ID）
2. 按指标分组，每个指标一个文档
3. 每个分析存储在 `data.<analysisKey>` 子路径下
4. 元数据（SRM、用户数）存储在快照主文档的 `chunkedAnalysesMeta` 中

**解码流程**:
1. 从 `chunkedAnalysesMeta` 重建维度/变体结构
2. 从分块文档中读取指标数据
3. 按 `analysisKey` 合并回完整的分析结果

---

## 2. 缓存键设计与查询匹配

### 2.1 缓存键的构成

**查询级缓存** (`packages/back-end/src/models/QueryModel.ts:208-234`):

缓存键由以下字段组合唯一标识：
- `organization` - 组织ID
- `datasource` - 数据源ID
- `query` - **完整的SQL查询文本**（最关键的匹配字段）
- `status` - 仅 `succeeded` 或 `running` 状态的查询可被复用
- `createdAt` - 查询创建时间（用于TTL检查）

**关键代码**:
```typescript
const latest = await QueryModel.find({
  organization,      // 过滤1: 组织
  datasource,        // 过滤2: 数据源
  query,             // 过滤3: 完整SQL文本匹配
  createdAt: { $gt: earliestDate },  // 过滤4: TTL时间窗
  status: { $in: ["succeeded", "running"] },  // 过滤5: 状态
  cachedQueryUsed: { $exists: false },  // 过滤6: 排除缓存复制的查询
})
```

### 2.2 缓存TTL策略

**TTL配置**:
- 默认值：`QUERY_CACHE_TTL_MINS`（环境变量，通常为24小时）
- 数据源级别覆盖：`datasource.settings.queryCacheTTLMins`
- 作用：仅复用在TTL时间窗内创建的查询

**TTL计算**:
```typescript
const ttl = cacheTTLMins ?? QUERY_CACHE_TTL_MINS;
const earliestDate = new Date();
earliestDate.setMinutes(earliestDate.getMinutes() - ttl);
```

### 2.3 缓存命中判断

在 `QueryRunner.startQuery()` 中 (`packages/back-end/src/queryRunners/QueryRunner.ts:765-902`):

**命中条件**:
1. `this.useCache === true`（调用方允许使用缓存）
2. 存在相同 `organization + datasource + query` 的历史查询
3. 历史查询在TTL时间窗内
4. 历史查询状态为 `succeeded` 或 `running`
5. 历史查询不是从其他缓存复制来的（`cachedQueryUsed` 字段不存在）

**命中后的处理**:
- **状态 = running**: 每3秒轮询原查询状态，直到完成
- **状态 = succeeded**: 直接复用结果，触发 `onQueryFinish()`
- 创建一个新的查询文档，通过 `cachedQueryUsed` 字段指向原查询

---

## 3. 缓存刷新触发机制

### 3.1 自动刷新触发条件

**定时任务触发** (`packages/back-end/src/jobs/updateExperimentResults.ts`):

1. **固定时间间隔**:
   - 默认每6小时刷新一次
   - 可通过组织设置 `updateSchedule` 自定义
   - 支持cron表达式、固定小时数等多种调度方式

2. **多臂老虎机 (MAB) 特殊调度**:
   - **探索阶段 (explore)**: 按 `banditBurnInValue` 调度
   - **开发阶段 (exploit)**: 按 `banditScheduleValue` 调度
   - 权重更新时立即触发快照

**代码位置**:
- `determineNextDate()` - 普通实验调度 `experiments.ts:763-789`
- `determineNextBanditSchedule()` - 老虎机实验调度 `experiments.ts:791-839`

### 3.2 手动刷新触发

**API触发**:
- 端点：`POST /api/v1/experiments/:id/snapshots`
- 参数：`triggeredBy` - 触发源（manual/schedule/dashboard等）
- 总是设置 `useCache: true`

**仪表盘自动更新**:
- 当标准快照刷新时，后台异步更新关联的仪表盘
- 触发器：`updateExperimentDashboards()` 在 `experiments.ts:1583-1597`

### 3.3 增量刷新触发

**增量刷新模式** (`packages/back-end/src/services/experiments.ts:1169-1287`):

当数据源启用 pipeline mode 时，优先使用增量刷新：

**触发条件**:
1. 数据源设置 `pipelineSettings.mode === "incremental"`
2. 实验类型为 `standard`（非多臂老虎机）
3. 实验兼容增量管道（通过 `validateIncrementalPipeline()` 检查）
4. 存在之前的增量刷新状态（`unitsTableFullName` 已创建）

**增量 vs 全量决策**:
```typescript
const fullRefresh =
  !useCache ||                          // 明确禁用缓存
  !incrementalRefreshModel ||           // 无增量状态
  !incrementalRefreshModel.unitsTableFullName;  // 单位表未创建
```

---

## 4. 缓存 vs 重跑决策逻辑

### 4.1 顶层决策：useCache 参数

`useCache` 参数是控制是否使用缓存的总开关，在以下场景设置：

| 场景 | useCache 值 | 说明 |
|------|------------|------|
| 手动刷新 API | `true` | `postExperimentSnapshot.ts:63` |
| 定时自动刷新 | `true` | `updateExperimentResults.ts:189` |
| 增量刷新运行器 | `false` | 总是忽略缓存，强制重跑 |
| 探索性增量查询 | `false` | TODO: 未来支持缓存覆盖 |
| 标准结果查询运行器 | 传入值 | 由调用方决定 |

### 4.2 查询运行器类型决策

快照运行器有三种类型，决定了缓存策略的差异：

**决策函数**: `getSnapshotQueryRunnerKind()` (`experiments.ts:1189-1226`)

```
运行器类型决策树:
├─→ 允许增量刷新 && 数据源启用增量模式 && 实验兼容
│   ├─→ 探索性快照 && 无维度
│   │   ├─→ 有单位表 → incremental
│   │   └─→ 无单位表 → results
│   ├─→ 探索性快照 → incremental-exploratory
│   └─→ 标准快照 → incremental
└─→ 其他情况 → results (传统全量查询)
```

### 4.3 各运行器的缓存策略

**1. ExperimentResultsQueryRunner (传统模式)**
- 缓存策略：完全遵循 `useCache` 参数
- 缓存粒度：每个独立SQL查询
- 缓存键：完整SQL文本 + organization + datasource
- 适用场景：非增量模式的所有快照

**2. ExperimentIncrementalRefreshQueryRunner (增量模式)**
- 缓存策略：`useCache` 强制为 `false`
- 原因：增量刷新依赖于单位表的状态，必须保证最新
- 适用场景：标准快照的增量更新

**3. ExperimentIncrementalRefreshExploratoryQueryRunner (探索性增量)**
- 缓存策略：`useCache` 强制为 `false`
- 备注：代码中标注TODO，未来可能支持缓存覆盖
- 适用场景：维度分析等探索性查询

### 4.4 单查询级缓存决策

在 `QueryRunner.startQuery()` 中对每个查询独立决策：

```
缓存决策流程:
├─→ useCache === true?
│   ├─→ 否 → 创建新查询，立即执行
│   └─→ 是 → 尝试缓存匹配
│       ├─→ 找到匹配的历史查询?
│       │   ├─→ 否 → 创建新查询
│       │   └─→ 是 → 复用缓存
│       │       ├─→ 原查询 running → 轮询等待
│       │       └─→ 原查询 succeeded → 直接使用结果
│       └─→ 异常 → 降级到新查询
└─→ 依赖检查
    ├─→ 无依赖 + 并发限制未达 → 立即执行
    ├─→ 无依赖 + 并发限制达到 → 排队等待
    └─→ 有依赖 → 等待依赖完成
```

### 4.5 缓存失效场景

**主动失效**:
1. **显式禁用缓存**: 调用时设置 `useCache = false`
2. **TTL过期**: 查询创建时间超过配置的缓存时长
3. **查询失败**: 仅缓存成功的查询（`succeeded` 状态）

**隐式失效（SQL变化导致不匹配）**:
任何SQL文本的变化都会导致缓存不命中，包括但不限于：
- 实验阶段变化（日期范围、权重等）
- 指标配置变化（添加/删除指标）
- 维度变化
- 分段过滤变化
- 统计引擎设置变化
- 回归调整/序贯检验等高级设置变化

### 4.6 缓存复用的性能影响

**缓存命中的好处**:
1. 避免重复执行昂贵的SQL查询
2. 减少数据源负载
3. 加快快照生成速度（秒级 vs 分钟级）

**缓存未命中的成本**:
1. 额外的数据库查询（查找历史查询）
2. 通常可忽略（有索引：`organization + datasource + status + createdAt`）

**并发场景处理**:
- 多个快照同时请求相同查询时，会复用同一个正在运行的查询
- 通过轮询机制等待首个查询完成
- 避免了"惊群效应"（thundering herd）

---

## 5. 核心代码位置

### 5.1 快照生成与规划

| 功能 | 文件 | 关键函数 |
|------|------|----------|
| 快照创建入口 | `services/experiments.ts` | `createSnapshot()`, `planSnapshot()` |
| 快照API | `api/experiments/postExperimentSnapshot.ts` | `postExperimentSnapshot` |
| 定时刷新任务 | `jobs/updateExperimentResults.ts` | `updateSingleExperiment()` |
| 快照模型 | `models/ExperimentSnapshotModel.ts` | `createExperimentSnapshotModel()`, `updateSnapshot()` |

### 5.2 缓存机制

| 功能 | 文件 | 关键函数 |
|------|------|----------|
| 查询缓存查找 | `models/QueryModel.ts` | `getRecentQuery()` |
| 缓存查询复制 | `models/QueryModel.ts` | `createNewQueryFromCached()` |
| 查询运行器基类 | `queryRunners/QueryRunner.ts` | `startQuery()`, `executeQuery()` |
| 结果查询运行器 | `queryRunners/ExperimentResultsQueryRunner.ts` | `startQueries()` |

### 5.3 分析分块

| 功能 | 文件 | 关键函数 |
|------|------|----------|
| 分析键生成 | `shared/snapshot-analysis-chunks.ts` | `buildAnalysisKey()` |
| 分块编码 | `shared/snapshot-analysis-chunks.ts` | `encodeSnapshotAnalysisChunks()` |
| 分块解码 | `shared/snapshot-analysis-chunks.ts` | `decodeSnapshotAnalysisChunks()` |
| 分块写入 | `models/ExperimentSnapshotModel.ts` | `chunkAndStripAnalyses()` |

### 5.4 增量刷新

| 功能 | 文件 | 关键函数 |
|------|------|----------|
| 运行器类型决策 | `services/experiments.ts` | `getSnapshotQueryRunnerKind()`, `planSnapshotQueryRunner()` |
| 增量刷新运行器 | `queryRunners/ExperimentIncrementalRefreshQueryRunner.ts` | `startAnalysis()` |
| 增量刷新锁 | `models/IncrementalRefreshModel.ts` | `acquireLock()`, `releaseLock()` |

---

## 总结：缓存选择决策流程图

```
用户请求快照生成
        ↓
useCache = true? (几乎总是 true)
        │
        ├─→ 是
        │    ↓
        │  数据源启用增量模式?
        │    │
        │    ├─→ 是
        │    │    ↓
        │    │  实验兼容增量?
        │    │    │
        │    │    ├─→ 是 → useCache=false（强制重跑）
        │    │    │      └─→ 使用增量刷新运行器
        │    │    └─→ 否 → useCache=true（允许缓存）
        │    │           └─→ 使用传统运行器
        │    └─→ 否 → useCache=true（允许缓存）
        │         └─→ 使用传统运行器
        │              ↓
        │            对每个SQL查询:
        │              ├─→ 查找最近相同SQL（TTL内）
        │              ├─→ 找到 → 复用结果/等待完成
        │              └─→ 未找到 → 执行新查询
        └─→ 否 → 强制全部重跑
              └─→ 使用传统运行器
```
