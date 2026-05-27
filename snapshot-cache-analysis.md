# 实验结果快照缓存机制分析

## 目录

1. [快照生成流程](#1-快照生成流程)
2. [缓存键设计与查询匹配](#2-缓存键设计与查询匹配)
3. [缓存刷新触发机制](#3-缓存刷新触发机制)
4. [缓存 vs 重跑决策逻辑](#4-缓存-vs-重跑决策逻辑)
5. [增量模式首次构建与后续更新分界](#5-增量模式首次构建与后续更新分界)
6. [增量兼容性门槛](#6-增量兼容性门槛)
7. [核心代码位置](#7-核心代码位置)

---

## 1. 快照生成流程

### 1.1 快照生成入口

快照生成有两个主要函数作为主函数，但它们的 `useCache` 默认值不同：

**1. `createExperimentSnapshot()** — 默认 `useCache = true`
- 文件：`packages/back-end/src/services/experiments.ts:1718`
- 被以下入口调用：
  - `POST /api/v1/experiments/:id/snapshots`（手动刷新 API）
  - 定时任务 `updateExperimentResults.ts`（自动刷新）
  - Holdout 导入时的自动刷新
  - Demo 项目创建

**2. `createSnapshot()`** — 默认 `useCache = false`
- 文件：`packages/back-end/src/services/experiments.ts:1627`
- 被仪表盘探索性快照内部使用

### 1.2 快照生成核心流程

```
createExperimentSnapshot() / createSnapshot()
    ↓
planExperimentSnapshot() / planSnapshot()  # 规划阶段
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

### 2.2 缓存 TTL 策略

**TTL 默认值（重点关注，之前的分析有误！**

**真实默认值：60 分钟**，不是 24 小时

**TTL配置**:
- 环境变量：`QUERY_CACHE_TTL_MINS`，默认值 `60`
- 数据源级别覆盖：`datasource.settings.queryCacheTTLMins`
- 作用：仅复用在 TTL 时间窗内创建的查询

**代码位置**: `packages/back-end/src/util/secrets.ts:152-156`

```typescript
export const QUERY_CACHE_TTL_MINS = parseEnvInt(
  process.env.QUERY_CACHE_TTL_MINS,
  60,  // 默认 60 分钟
  { min: 0, name: "QUERY_CACHE_TTL_MINS" },
);
```

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
3. 历史查询在 TTL 时间窗内
4. 历史查询状态为 `succeeded` 或 `running`
5. 历史查询不是从其他缓存复制来的（`cachedQueryUsed` 字段不存在）

**命中后的处理**:
- **状态 = running**: 每 3 秒轮询原查询状态，直到完成
- **状态 = succeeded**: 直接复用结果，触发 `onQueryFinish()`
- 创建一个新的查询文档，通过 `cachedQueryUsed` 字段指向原查询

---

## 3. 缓存刷新触发机制

### 3.1 不同触发入口的 useCache 差异

**这是之前分析遗漏的关键点**：不同入口对 `useCache` 的设置不同，直接影响缓存决策路径：

| 触发入口 | 函数 | useCache | 说明 |
|----------|------|----------|------|
| 手动刷新 API | `postExperimentSnapshot.ts:63` | `true` | 硬编码 |
| 定时自动刷新 | `updateExperimentResults.ts:189` | `true` | 硬编码 |
| 仪表盘探索性快照 | `dashboards.controller.ts:254` | `false` | 硬编码，每次强制重跑 |
| Holdout 自动刷新 | `holdout.controller.ts:269` | `true` | 硬编码 |
| Demo 项目创建 | `demo-datasource-project.controller.ts:519` | `true` | 硬编码 |

**为什么仪表盘探索性快照 useCache=false 的原因**：
- 仪表盘快照类型为 `exploratory`，带维度
- 需要确保每次刷新仪表盘时强制重跑，保证数据最新
- 由 `createExperimentSnapshot()` 调用，传入

### 3.2 自动刷新触发条件

**定时任务触发** (`packages/back-end/src/jobs/updateExperimentResults.ts`):

1. **固定时间间隔**:
   - 默认每 6 小时刷新一次
   - 可通过组织设置 `updateSchedule` 自定义
   - 支持 cron 表达式、固定小时数等多种调度方式

2. **多臂老虎机 (MAB) 特殊调度**:
   - **探索阶段 (explore)**: 按 `banditBurnInValue` 调度
   - **开发阶段 (exploit)**: 按 `banditScheduleValue` 调度
   - 权重更新时立即触发快照

**代码位置**:
- `determineNextDate()` - 普通实验调度 `experiments.ts:763-789`
- `determineNextBanditSchedule()` - 老虎机实验调度 `experiments.ts:791-839`

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

`useCache` 参数是控制是否使用缓存的总开关。

**两个 create 函数的 useCache 默认值不同**：

```typescript
// createExperimentSnapshot() - 默认 true
export async function createExperimentSnapshot({
  useCache = true,  // 默认 true
  ...
})

// createSnapshot() - 默认 false
export async function createSnapshot({
  useCache = false,  // 默认 false
  ...
})
```

**各运行器中的 useCache 设置：

| 运行器类型 | useCache 值 | 说明 |
|------------|------------|------|
| ExperimentResultsQueryRunner | `plan.useCache`（传入值） | 由调用方决定 |
| ExperimentIncrementalRefreshQueryRunner | `false`（硬编码） | 总是忽略缓存 |
| ExperimentIncrementalRefreshExploratoryQueryRunner | `false`（硬编码） | 总是忽略缓存 |

**代码位置**: `packages/back-end/src/services/experiments.ts:1502-1527`

```typescript
switch (plan.runnerKind) {
  case "incremental-exploratory":
    queryRunner = new ExperimentIncrementalRefreshExploratoryQueryRunner(
      context, snapshot, integration,
      false, // TODO(incremental-refresh): allow cache + cache override for exploratory queries
    );
    break;
  case "incremental":
    queryRunner = new ExperimentIncrementalRefreshQueryRunner(
      context, snapshot, integration,
      false, // always ignore cache for incremental refresh queries
    );
    break;
  case "results":
    queryRunner = new ExperimentResultsQueryRunner(
      context, snapshot, integration,
      plan.useCache,  // 由调用方传入
    );
    break;
}
```

### 4.2 查询运行器类型决策

快照运行器有三种类型，决定了缓存策略的差异：

**决策函数**: `getSnapshotQueryRunnerKind()` (`experiments.ts:1189-1226`

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

### 4.3 单查询级缓存决策

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

### 4.4 缓存失效场景

**主动失效**:
1. **显式禁用缓存**: 调用时设置 `useCache = false`
2. **TTL过期**: 查询创建时间超过配置的缓存时长（默认 60 分钟）
3. **查询失败**: 仅缓存成功的查询（`succeeded` 状态）

**隐式失效（SQL 变化导致不匹配）**:
任何 SQL 文本的变化都会导致缓存不命中，包括但不限于：
- 实验阶段变化（日期范围、权重等）
- 指标配置变化（添加/删除指标）
- 维度变化
- 分段过滤变化
- 统计引擎设置变化
- 回归调整/序贯检验等高级设置变化

### 4.5 缓存复用的性能影响

**缓存命中的好处**:
1. 避免重复执行昂贵的 SQL 查询
2. 减少数据源负载
3. 加快快照生成速度（秒级 vs 分钟级

**缓存未命中的成本**:
1. 额外的数据库查询（查找历史查询）
2. 通常可忽略（有索引：`organization + datasource + status + createdAt`）

**并发场景处理**:
- 多个快照同时请求相同查询时，会复用同一个正在运行的查询
- 通过轮询机制等待首个查询完成
- 避免了"惊群效应"（thundering herd）

---

## 5. 增量模式首次构建与后续更新分界

### 5.1 fullRefresh 的计算

```typescript
// packages/back-end/src/services/experiments.ts:1335-1348

// 只有在 useCache === true 时才会查询 incrementalRefreshModel
const incrementalRefreshModel = useCache
  ? await context.models.incrementalRefresh.getByExperimentId(experiment.id)
  : null;

// fullRefresh 为 true 的条件：
const fullRefresh =
  !useCache ||                                    // 条件1: 禁用缓存
  !incrementalRefreshModel ||                       // 条件2: 无增量状态记录
  !incrementalRefreshModel.unitsTableFullName;       // 条件3: 单位表未创建
```

### 5.2 首次构建 vs 后续更新

**首次构建（fullRefresh = true）:

| 场景 | 条件 | 说明 |
|------|------|------|
| 第一次增量刷新 | `incrementalRefreshModel` 为 null | 没有任何增量状态记录 |
| 首次构建单位表失败后 | `unitsTableFullName` 为 null | 状态记录存在但单位表未创建 |
| 手动全量刷新 | `useCache = false` | 显式禁用缓存 |

**后续更新（fullRefresh = false）:
- `incrementalRefreshModel` 存在
- `unitsTableFullName` 有值
- `useCache = true`

### 5.3 全量刷新的实际执行

**注意**：`fullRefresh` 值在 `startAnalysis()` 中还有一层判断：

```typescript
// packages/back-end/src/services/experiments.ts:1549-1557

if (plan.runnerKind === "incremental") {
  await (queryRunner as ExperimentIncrementalRefreshQueryRunner).startAnalysis({
    ...analysisProps,
    experimentId: experiment.id,
    incrementalRefreshStartTime: new Date(),
    // 只有标准快照才会真正执行全量刷新
    fullRefresh: plan.fullRefresh && plan.snapshot.type === "standard",
  });
}
```

**关键细节**：
- 即使 `plan.fullRefresh = true` 且快照类型为 `standard` → 真正执行全量刷新
- 即使 `plan.fullRefresh = true` 但快照类型为 `exploratory` → 不执行全量刷新
- 探索性增量查询 (`incremental-exploratory` 运行器不接收 `fullRefresh` 参数

---

## 6. 增量兼容性门槛

### 6.1 validateIncrementalPipeline 完整检查

**代码位置**: `packages/back-end/src/services/dataPipeline.ts:21-170`

**完整检查项：

| 检查项 | 错误信息 |
|--------|----------|
| 不能使用 `skipPartialData` | "'Exclude In-Progress Conversions' is not supported for incremental refresh queries while in beta." |
| 数据源必须支持增量刷新 | "Integration does not support incremental refresh queries" |
| 组织必须有 incremental-refresh 高级功能 | "Organization does not have access to incremental refresh feature" |
| 数据源设置 pipeline mode 必须是 "incremental" | "Integration does not have Pipeline Incremental enabled" |
| 不能有 activation metric | "Activation metrics are not supported for incremental refresh while in beta." |
| 实验必须在 pipeline 的 includedExperimentIds 中（如果设置了） | "Experiment is not included in the Pipeline Incremental scope" |
| 所有指标必须是 fact metrics | "Only fact metrics are supported with incremental refresh." |
| Ratio metrics 的分子分母必须在同一个 fact table | "Ratio metrics must have the same numerator and denominator fact table with incremental refresh." |
| Event quantile metrics 需要数据源支持 KLL | "Event quantile metrics are not supported with incremental refresh on this data source." |
| 配置 hash 必须匹配（仅 main-update 类型） | "The experiment configuration is outdated. Please run a Full Refresh." |
| 指标设置 hash 必须匹配（仅 main-update 类型） | "The metric \"{name}\" configuration is outdated. Please run a Full Refresh." |

### 6.2 配置 hash 检查

**仅在 `analysisType === "main-update"` 且 `incrementalRefreshModel` 存在时检查**：

```typescript
// packages/back-end/src/services/dataPipeline.ts:128-169

if (analysisType === "main-update" && incrementalRefreshModel) {
  // 检查实验配置 hash
  const currentSettingsHash = getExperimentSettingsHashForIncrementalRefresh(snapshotSettings);
  if (
    incrementalRefreshModel.experimentSettingsHash &&
    currentSettingsHash !== incrementalRefreshModel.experimentSettingsHash
  ) {
    throw new Error("The experiment configuration is outdated. Please run a Full Refresh.");
  }

  // 检查每个指标的设置 hash
  selectedMetrics.filter(isFactMetric).forEach((m) => {
    const storedHash = existingMetricHashMap.get(m.id);
    if (!storedHash) return;
    
    const currentHash = getMetricSettingsHashForIncrementalRefresh({...});
    if (currentHash !== storedHash) {
      throw new Error(`The metric "${metricName}" configuration is outdated. Please run a Full Refresh.`);
    }
  });
}
```

---

## 7. 核心代码位置

### 7.1 快照生成与规划

| 功能 | 文件 | 关键函数 |
|------|------|----------|
| 快照创建（默认 useCache=true | `services/experiments.ts` | `createExperimentSnapshot()` |
| 快照创建（默认 useCache=false） | `services/experiments.ts` | `createSnapshot()` |
| 快照规划 | `services/experiments.ts` | `planSnapshot()` |
| 快照API | `api/experiments/postExperimentSnapshot.ts` | `postExperimentSnapshot` |
| 定时刷新任务 | `jobs/updateExperimentResults.ts` | `updateSingleExperiment()` |
| 快照模型 | `models/ExperimentSnapshotModel.ts` | `createExperimentSnapshotModel()`, `updateSnapshot()` |

### 7.2 缓存机制

| 功能 | 文件 | 关键函数 |
|------|------|----------|
| TTL 默认值定义 | `util/secrets.ts:152-156` | `QUERY_CACHE_TTL_MINS` |
| 查询缓存查找 | `models/QueryModel.ts` | `getRecentQuery()` |
| 缓存查询复制 | `models/QueryModel.ts` | `createNewQueryFromCached()` |
| 查询运行器基类 | `queryRunners/QueryRunner.ts` | `startQuery()`, `executeQuery()` |
| 结果查询运行器 | `queryRunners/ExperimentResultsQueryRunner.ts` | `startQueries()` |

### 7.3 增量刷新

| 功能 | 文件 | 关键函数 |
|------|------|----------|
| 运行器类型决策 | `services/experiments.ts` | `getSnapshotQueryRunnerKind()`, `planSnapshotQueryRunner()` |
| 增量兼容性检查 | `services/dataPipeline.ts` | `validateIncrementalPipeline()` |
| 增量刷新运行器 | `queryRunners/ExperimentIncrementalRefreshQueryRunner.ts` | `startAnalysis()` |
| 增量刷新锁 | `models/IncrementalRefreshModel.ts` | `acquireLock()`, `releaseLock()` |

### 7.4 分析分块

| 功能 | 文件 | 关键函数 |
|------|------|----------|
| 分析键生成 | `shared/snapshot-analysis-chunks.ts` | `buildAnalysisKey()` |
| 分块编码 | `shared/snapshot-analysis-chunks.ts` | `encodeSnapshotAnalysisChunks()` |
| 分块解码 | `shared/snapshot-analysis-chunks.ts` | `decodeSnapshotAnalysisChunks()` |
| 分块写入 | `models/ExperimentSnapshotModel.ts` | `chunkAndStripAnalyses()` |

---

## 总结：缓存选择决策流程图

```
用户请求快照生成
        ↓
调用哪个函数？
        │
        ├─→ createExperimentSnapshot() → useCache 默认 true
        │       │
        │       ├─→ 手动刷新 API → useCache=true
        │       ├─→ 定时自动刷新 → useCache=true
        │       ├─→ Holdout 自动刷新 → useCache=true
        │       └─→ Demo 项目 → useCache=true
        │
        └─→ createSnapshot() → useCache 默认 false
                │
                └─→ 仪表盘探索性快照 → useCache=false
        ↓
数据源启用增量模式?
        │
        ├─→ 是
        │    ↓
        │  实验兼容增量 (validateIncrementalPipeline)?
        │    │
        │    ├─→ 是
        │    │    ↓
        │    │  incrementalRefreshModel 存在且 unitsTableFullName 有值?
        │    │    │
        │    │    ├─→ 是 → fullRefresh=false（后续增量更新）
        │    │    │      └─→ 使用 incremental 运行器，useCache=false
        │    │    └─→ 否 → fullRefresh=true（首次构建或全量刷新）
        │    │           └─→ 使用 incremental 运行器，useCache=false
        │    │              └─→ 仅 standard 快照才真正全量刷新
        │    └─→ 否 → 回退到传统模式
        │         └─→ 使用 results 运行器，useCache=传入值
        └─→ 否 → 使用 results 运行器，useCache=传入值
             ↓
           对每个SQL查询:
             ├─→ useCache=true → 查找最近相同SQL（TTL=60分钟内）
             │   ├─→ 找到 → 复用结果/等待完成
             │   └─→ 未找到 → 执行新查询
             └─→ useCache=false → 直接执行新查询
```
