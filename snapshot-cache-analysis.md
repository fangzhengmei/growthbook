# 实验结果快照缓存机制分析

## 目录

1. [核心概念与术语](#1-核心概念与术语)
2. [快照生成主入口与 useCache 传递链](#2-快照生成主入口与-usecache-传递链)
3. [缓存键设计与 TTL 策略](#3-缓存键设计与-ttl-策略)
4. [查询运行器类型决策](#4-查询运行器类型决策)
5. [增量模式：首次构建 vs 后续更新](#5-增量模式首次构建-vs-后续更新)
6. [探索性场景的缓存策略分叉](#6-探索性场景的缓存策略分叉)
7. [增量兼容性门槛](#7-增量兼容性门槛)
8. [缓存 vs 重跑完整决策流](#8-缓存-vs-重跑完整决策流)
9. [核心代码位置](#9-核心代码位置)

---

## 1. 核心概念与术语

| 术语 | 定义 |
|------|------|
| `useCache` | 顶层缓存开关，控制是否允许查询级缓存复用 |
| `fullRefresh` | 增量模式下的全量刷新标志，控制是否重建单位表 |
| `incrementalRefreshModel` | 增量刷新状态文档，记录单位表位置、配置 hash 等 |
| `unitsTableFullName` | 物化单位表的完整名称，是增量模式就绪的标志 |
| `runnerKind` | 查询运行器类型：`results` / `incremental` / `incremental-exploratory` |
| `snapshotType` | 快照类型：`standard` / `exploratory` / `report` |

---

## 2. 快照生成主入口与 useCache 传递链

### 2.1 两个创建函数的默认值差异

**`createExperimentSnapshot()`** — 对外主入口
- 位置：`services/experiments.ts:1718`
- `useCache` 默认值：**`true`**
- 典型调用方：API、仪表盘、Holdout 导入

**`createSnapshot()`** — 内部调用入口
- 位置：`services/experiments.ts:1627`
- `useCache` 默认值：**`false`**
- 典型调用方：定时任务

### 2.2 各触发入口的实际 useCache 值

| 触发入口 | 调用函数 | 实际 useCache | 代码位置 |
|----------|----------|--------------|----------|
| 手动刷新 API | `createExperimentSnapshot()` | **`true`**（硬编码） | `postExperimentSnapshot.ts:63` |
| 定时自动刷新 | `createSnapshot()` | **`true`**（显式传入） | `updateExperimentResults.ts:189` |
| 仪表盘维度快照 | `createExperimentSnapshot()` | **`false`**（显式传入） | `dashboards.controller.ts:254` |
| Holdout 自动刷新 | `createExperimentSnapshot()` | **`true`**（显式传入） | `holdout.controller.ts:269` |
| Demo 项目创建 | `createSnapshot()` | **`true`**（显式传入） | `demo-datasource-project.controller.ts:519` |

### 2.3 useCache 在规划阶段的作用

**`planSnapshot()`** (`experiments.ts:1297`):

```typescript
// 只有 useCache=true 时才会查询增量刷新状态
const incrementalRefreshModel = useCache
  ? await context.models.incrementalRefresh.getByExperimentId(experiment.id)
  : null;

// fullRefresh 计算：useCache=false 会强制全量刷新
const fullRefresh =
  !useCache ||                                    // 条件1
  !incrementalRefreshModel ||                       // 条件2
  !incrementalRefreshModel.unitsTableFullName;       // 条件3
```

**关键点**：
- `useCache=false` → `incrementalRefreshModel` 为 null → `fullRefresh=true`
- `useCache=true` → 查询增量状态 → 根据状态决定是否全量刷新

### 2.4 useCache 在运行器创建时的最终值

**`createSnapshotFromPlan()`** (`experiments.ts:1502-1527`):

```typescript
switch (plan.runnerKind) {
  case "incremental-exploratory":
    queryRunner = new ExperimentIncrementalRefreshExploratoryQueryRunner(
      context, snapshot, integration,
      false, // 硬编码：总是忽略缓存
    );
    break;
  case "incremental":
    queryRunner = new ExperimentIncrementalRefreshQueryRunner(
      context, snapshot, integration,
      false, // 硬编码：总是忽略缓存
    );
    break;
  case "results":
    queryRunner = new ExperimentResultsQueryRunner(
      context, snapshot, integration,
      plan.useCache,  // 由调用方传入的值决定
    );
    break;
}
```

**最终结论**：
- **增量运行器** (`incremental` / `incremental-exploratory`)：无论传入什么，`useCache` 最终都是 `false`
- **传统运行器** (`results`)：`useCache` 等于调用方传入的值

---

## 3. 缓存键设计与 TTL 策略

### 3.1 查询级缓存键

**`getRecentQuery()`** (`models/QueryModel.ts:208-234`):

```typescript
const latest = await QueryModel.find({
  organization,      // 过滤1: 组织ID
  datasource,        // 过滤2: 数据源ID
  query,             // 过滤3: 完整SQL文本（精确匹配）
  createdAt: { $gt: earliestDate },  // 过滤4: TTL时间窗
  status: { $in: ["succeeded", "running"] },  // 过滤5: 状态
  cachedQueryUsed: { $exists: false },  // 过滤6: 排除缓存复制的查询
})
  .sort({ createdAt: -1 })
  .limit(1);
```

### 3.2 TTL 默认值（重点修正！）

**真实默认值：60 分钟**，不是 24 小时

**代码位置**：`util/secrets.ts:152-156`

```typescript
export const QUERY_CACHE_TTL_MINS = parseEnvInt(
  process.env.QUERY_CACHE_TTL_MINS,
  60,  // 默认 60 分钟
  { min: 0, name: "QUERY_CACHE_TTL_MINS" },
);
```

**TTL 计算**：
```typescript
const ttl = cacheTTLMins ?? QUERY_CACHE_TTL_MINS;
const earliestDate = new Date();
earliestDate.setMinutes(earliestDate.getMinutes() - ttl);
```

**数据源级别覆盖**：可通过 `datasource.settings.queryCacheTTLMins` 覆盖

### 3.3 缓存命中条件

在 `QueryRunner.startQuery()` 中对每个查询独立判断：

1. `this.useCache === true`（运行器允许缓存）
2. 存在相同 `organization + datasource + query` 的历史查询
3. 历史查询在 TTL 时间窗内（默认 60 分钟）
4. 历史查询状态为 `succeeded` 或 `running`
5. 历史查询不是从其他缓存复制来的（`cachedQueryUsed` 字段不存在）

### 3.4 缓存命中后的处理

| 原查询状态 | 处理方式 |
|-----------|----------|
| `running` | 每 3 秒轮询原查询状态，直到完成 |
| `succeeded` | 直接复用结果，触发 `onQueryFinish()` |

无论哪种情况，都会创建一个新的查询文档，通过 `cachedQueryUsed` 字段指向原查询。

---

## 4. 查询运行器类型决策

### 4.1 决策函数

**`getSnapshotQueryRunnerKind()`** (`experiments.ts:1189-1226`):

```typescript
if (
  allowIncrementalRefresh &&
  isIncrementalRefreshEnabledForSnapshot({ datasource, experiment }) &&
  (experiment.type === undefined || experiment.type === "standard") &&
  isExperimentCompatibleWithIncrementalRefresh
) {
  if (snapshotType === "exploratory" && !hasSnapshotDimensions) {
    // 无维度的探索性快照：有单位表用增量，否则回退到传统
    return hasMaterializedUnitsTable ? "incremental" : "results";
  }

  // 有维度的探索性快照用 exploratory 运行器，标准快照用普通增量运行器
  return snapshotType === "exploratory"
    ? "incremental-exploratory"
    : "incremental";
}

// 不满足增量条件，使用传统运行器
return "results";
```

### 4.2 前置条件检查

**`planSnapshotQueryRunner()`** (`experiments.ts:1228-1287`):

在调用 `getSnapshotQueryRunnerKind()` 之前，会先调用 `validateIncrementalPipeline()` 检查兼容性：

```typescript
try {
  await validateIncrementalPipeline({
    ...,
    analysisType: fullRefresh
      ? "main-fullRefresh"
      : snapshotType === "standard"
        ? "main-update"
        : "exploratory",
  });
  isExperimentCompatibleWithIncrementalRefresh = true;
} catch (error) {
  // 不兼容，回退到传统运行器
  isExperimentCompatibleWithIncrementalRefresh = false;
}
```

---

## 5. 增量模式：首次构建 vs 后续更新

### 5.1 fullRefresh 的计算

**`planSnapshot()`** (`experiments.ts:1335-1348`):

```typescript
// 只有 useCache=true 时才会查询增量刷新状态
const incrementalRefreshModel = useCache
  ? await context.models.incrementalRefresh.getByExperimentId(experiment.id)
  : null;

const fullRefresh =
  !useCache ||                                    // 条件1: 禁用缓存
  !incrementalRefreshModel ||                       // 条件2: 无增量状态
  !incrementalRefreshModel.unitsTableFullName;       // 条件3: 单位表未创建
```

### 5.2 首次构建 vs 后续更新分界

| 场景 | `fullRefresh` | `incrementalRefreshModel` | `unitsTableFullName` | 说明 |
|------|--------------|---------------------------|----------------------|------|
| **首次构建** | `true` | `null` 或存在 | `null` 或存在但为空 | 单位表不存在，需要 CREATE |
| **首次构建失败后** | `true` | 存在 | `null` | 锁已获取但 CREATE 失败 |
| **手动全量刷新** | `true` | 可能存在 | 可能有值 | `useCache=false` 强制 |
| **后续增量更新** | `false` | 存在 | 有有效值 | 单位表已就绪，只 UPDATE |

### 5.3 fullRefresh 在运行时的二次判断

**`createSnapshotFromPlan()`** (`experiments.ts:1549-1557`):

```typescript
if (plan.runnerKind === "incremental") {
  await (queryRunner as ExperimentIncrementalRefreshQueryRunner).startAnalysis({
    ...,
    // 只有 standard 快照才会真正执行全量刷新
    fullRefresh: plan.fullRefresh && plan.snapshot.type === "standard",
  });
}
```

**关键细节**：
- `plan.fullRefresh = true` + `snapshot.type = "standard"` → 真正全量刷新
- `plan.fullRefresh = true` + `snapshot.type = "exploratory"` → **不执行全量刷新**
- `incremental-exploratory` 运行器不接收 `fullRefresh` 参数

### 5.4 增量运行器中 fullRefresh 的实际作用

**`ExperimentIncrementalRefreshQueryRunner.startQueries()`** (`queryRunners/ExperimentIncrementalRefreshQueryRunner.ts:895-899`):

```typescript
const incrementalRefreshModel = params.fullRefresh
  ? null  // 全量刷新：忽略历史状态
  : await this.context.models.incrementalRefresh.getByExperimentId(params.experimentId);
```

当 `fullRefresh=true` 时：
- 忽略历史增量状态
- 执行 DROP → CREATE → UPDATE 完整流程
- 重建所有指标源表

当 `fullRefresh=false` 时：
- 使用历史增量状态
- 仅执行 UPDATE（从上次 max_timestamp 开始）
- 仅更新有新数据的指标源

---

## 6. 探索性场景的缓存策略分叉

### 6.1 探索性快照的定义

- `snapshotType = "exploratory"`
- 通常带有维度（`dimension` 非空）
- 用于仪表盘、维度分析等临时查询

### 6.2 运行器选择分叉

| 场景 | 条件 | 运行器类型 | useCache |
|------|------|-----------|----------|
| 无维度探索性 + 有单位表 | `snapshotType="exploratory"` + `!hasSnapshotDimensions` + `hasMaterializedUnitsTable` | `incremental` | `false`（硬编码） |
| 无维度探索性 + 无单位表 | `snapshotType="exploratory"` + `!hasSnapshotDimensions` + `!hasMaterializedUnitsTable` | `results` | 调用方传入 |
| 有维度探索性 | `snapshotType="exploratory"` + `hasSnapshotDimensions` | `incremental-exploratory` | `false`（硬编码） |

### 6.3 仪表盘快照的特殊处理

**代码位置**：`dashboards.controller.ts:247-257`

```typescript
for (const [dimensionId, blockIds] of Object.entries(dimensionsByBlocks)) {
  const { snapshot } = await createExperimentSnapshot({
    context,
    experiment,
    dimension: dimensionId,
    datasource,
    phase: experiment.phases.length - 1,
    useCache: false,  // 硬编码禁用缓存
    triggeredBy: "manual-dashboard",
    type: "exploratory",
  });
}
```

**为什么仪表盘禁用缓存**：
- 类型为 `exploratory`，带维度
- 需要确保每次刷新仪表盘时强制重跑
- 保证数据最新性优先于性能

### 6.4 探索性增量查询的注释

**代码位置**：`experiments.ts:1505-1510`

```typescript
case "incremental-exploratory":
  queryRunner = new ExperimentIncrementalRefreshExploratoryQueryRunner(
    context,
    snapshot,
    integration,
    false, // TODO(incremental-refresh): allow cache + cache override for exploratory queries
  );
```

当前硬编码为 `false`，未来可能支持缓存覆盖。

---

## 7. 增量兼容性门槛

### 7.1 validateIncrementalPipeline 完整检查

**代码位置**：`services/dataPipeline.ts:21-170`

| 检查项 | 错误信息 |
|--------|----------|
| 不能使用 `skipPartialData` | "'Exclude In-Progress Conversions' is not supported for incremental refresh queries while in beta." |
| 数据源必须支持增量刷新 | "Integration does not support incremental refresh queries" |
| 组织必须有 incremental-refresh 高级功能 | "Organization does not have access to incremental refresh feature" |
| 数据源 pipeline mode 必须是 "incremental" | "Integration does not have Pipeline Incremental enabled" |
| 不能有 activation metric | "Activation metrics are not supported for incremental refresh while in beta." |
| 实验必须在 includedExperimentIds 中（如果设置了） | "Experiment is not included in the Pipeline Incremental scope" |
| 所有指标必须是 fact metrics | "Only fact metrics are supported with incremental refresh." |
| Ratio metrics 分子分母必须在同一 fact table | "Ratio metrics must have the same numerator and denominator fact table with incremental refresh." |
| Event quantile metrics 需要数据源支持 KLL | "Event quantile metrics are not supported with incremental refresh on this data source." |
| **配置 hash 匹配（仅 main-update）** | "The experiment configuration is outdated. Please run a Full Refresh." |
| **指标设置 hash 匹配（仅 main-update）** | "The metric \"{name}\" configuration is outdated. Please run a Full Refresh." |

### 7.2 配置 hash 检查的触发时机

**仅在 `analysisType === "main-update"` 且 `incrementalRefreshModel` 存在时检查**：

```typescript
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

**analysisType 映射**：
- `fullRefresh = true` → `"main-fullRefresh"` → **不检查 hash**
- `fullRefresh = false` + `snapshotType = "standard"` → `"main-update"` → **检查 hash**
- `fullRefresh = false` + `snapshotType = "exploratory"` → `"exploratory"` → **不检查 hash**

---

## 8. 缓存 vs 重跑完整决策流

### 8.1 顶层决策流程图

```
触发快照生成
    ↓
确定 useCache 值（见 2.2 节各入口差异）
    ↓
planSnapshot():
    ├─→ useCache=true?
    │   ├─→ 是 → 查询 incrementalRefreshModel
    │   └─→ 否 → incrementalRefreshModel = null
    └─→ 计算 fullRefresh = !useCache || !model || !model.unitsTableFullName
    ↓
planSnapshotQueryRunner():
    ├─→ 调用 validateIncrementalPipeline() 检查兼容性
    └─→ 调用 getSnapshotQueryRunnerKind() 选择运行器
    ↓
createSnapshotFromPlan():
    ├─→ 根据 runnerKind 创建运行器
    │   ├─→ incremental → useCache=false（硬编码）
    │   ├─→ incremental-exploratory → useCache=false（硬编码）
    │   └─→ results → useCache=传入值
    └─→ 传入 startAnalysis() 的 fullRefresh 需与 snapshot.type="standard" 相与
    ↓
运行器执行查询：
    ├─→ useCache=true → 对每个查询尝试缓存复用（TTL=60分钟）
    │   ├─→ 命中 → 复用/等待
    │   └─→ 未命中 → 执行新查询
    └─→ useCache=false → 所有查询强制重跑
```

### 8.2 关键决策点速查表

| 决策点 | 影响因素 | 结果 |
|--------|---------|------|
| useCache 初始值 | 调用入口（见 2.2 节） | `true` 或 `false` |
| fullRefresh 计算 | useCache + incrementalRefreshModel + unitsTableFullName | `true` 或 `false` |
| runnerKind 选择 | 数据源设置 + 实验兼容性 + 快照类型 + 维度 | `results` / `incremental` / `incremental-exploratory` |
| 运行器 useCache | runnerKind | 增量运行器恒为 `false`，传统运行器跟随传入值 |
| 实际 fullRefresh | plan.fullRefresh + snapshot.type="standard" | 仅 standard 快照可能全量刷新 |
| 查询级缓存 | 运行器 useCache + SQL精确匹配 + TTL（60分钟） | 命中或未命中 |

---

## 9. 核心代码位置

### 9.1 快照生成与规划

| 功能 | 文件 | 关键函数 |
|------|------|----------|
| 快照创建（默认 useCache=true） | `services/experiments.ts:1718` | `createExperimentSnapshot()` |
| 快照创建（默认 useCache=false） | `services/experiments.ts:1627` | `createSnapshot()` |
| 快照规划（核心决策） | `services/experiments.ts:1297` | `planSnapshot()` |
| 快照创建执行 | `services/experiments.ts:1434` | `createSnapshotFromPlan()` |
| 手动刷新 API | `api/experiments/postExperimentSnapshot.ts` | `postExperimentSnapshot` |
| 定时刷新任务 | `jobs/updateExperimentResults.ts:179` | `updateSingleExperiment()` |

### 9.2 缓存机制

| 功能 | 文件 | 关键函数 |
|------|------|----------|
| TTL 默认值定义 | `util/secrets.ts:152-156` | `QUERY_CACHE_TTL_MINS` |
| 查询缓存查找 | `models/QueryModel.ts:208-234` | `getRecentQuery()` |
| 缓存查询复制 | `models/QueryModel.ts:315-348` | `createNewQueryFromCached()` |
| 查询运行器基类 | `queryRunners/QueryRunner.ts` | `startQuery()` |
| 结果查询运行器 | `queryRunners/ExperimentResultsQueryRunner.ts` | `startQueries()` |

### 9.3 增量刷新

| 功能 | 文件 | 关键函数 |
|------|------|----------|
| 运行器类型决策 | `services/experiments.ts:1189-1226` | `getSnapshotQueryRunnerKind()` |
| 运行器规划（含兼容性检查） | `services/experiments.ts:1228-1287` | `planSnapshotQueryRunner()` |
| 增量兼容性检查 | `services/dataPipeline.ts:21-170` | `validateIncrementalPipeline()` |
| 增量刷新运行器 | `queryRunners/ExperimentIncrementalRefreshQueryRunner.ts` | `startQueries()` |
| 增量刷新锁与状态 | `models/IncrementalRefreshModel.ts` | `acquireLock()`, `releaseLock()` |

### 9.4 探索性场景

| 功能 | 文件 | 关键函数 |
|------|------|----------|
| 仪表盘维度快照 | `routers/dashboards/dashboards.controller.ts:247-257` | 循环调用 `createExperimentSnapshot()` |
| 探索性增量运行器 | `queryRunners/ExperimentIncrementalRefreshExploratoryQueryRunner.ts` | `startQueries()` |
