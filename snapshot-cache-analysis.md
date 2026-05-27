# 实验结果快照缓存机制分析

## 目录

1. [核心概念与术语](#1-核心概念与术语)
2. [快照生成主入口与 useCache 传递链](#2-快照生成主入口与-usecache-传递链)
3. [缓存键设计与 TTL 策略](#3-缓存键设计与-ttl-策略)
4. [查询运行器类型决策](#4-查询运行器类型决策)
5. [增量模式：首次构建 vs 后续更新](#5-增量模式首次构建-vs-后续更新)
6. [仪表盘探索性快照的两条缓存路径](#6-仪表盘探索性快照的两条缓存路径)
7. [增量兼容性门槛](#7-增量兼容性门槛)
8. [触发入口、运行器与数据新鲜度关联表](#8-触发入口运行器与数据新鲜度关联表)
9. [缓存 vs 重跑完整决策流](#9-缓存-vs-重跑完整决策流)
10. [故障排除指南](#10-故障排除指南)
11. [核心代码位置](#11-核心代码位置)

---

## 1. 核心概念与术语

| 术语 | 定义 |
|------|------|
| `useCache` | 顶层缓存开关，控制查询级缓存是否允许复用 |
| `fullRefresh` | 增量模式下的全量刷新标志，控制是否重建单位表 |
| `incrementalRefreshModel` | 增量刷新状态文档，记录单位表位置、配置 hash 等 |
| `unitsTableFullName` | 物化单位表的完整名称，是增量模式就绪的标志 |
| `runnerKind` | 查询运行器类型：`results` / `incremental` / `incremental-exploratory` |
| `snapshotType` | 快照类型：`standard` / `exploratory` / `report` |
| `triggeredBy` | 触发来源：`manual` / `schedule` / `manual-dashboard` / `update-dashboards` 等 |

---

## 2. 快照生成主入口与 useCache 传递链

### 2.1 两个创建函数的默认值差异

**`createExperimentSnapshot()`** — 对外主入口
- 位置：`services/experiments.ts:1718`
- `useCache` 默认值：**`true`**
- 调用方：API、仪表盘手动刷新、Holdout 导入

**`createSnapshot()`** — 内部调用入口
- 位置：`services/experiments.ts:1627`
- `useCache` 默认值：**`false`**
- 调用方：定时任务、仪表盘自动更新

### 2.2 各触发入口的实际 useCache 值

| 触发入口 | 调用函数 | 实际 useCache | triggeredBy | snapshotType | 代码位置 |
|----------|----------|--------------|-------------|-------------|----------|
| 手动刷新 API | `createExperimentSnapshot()` | **`true`**（硬编码） | `manual` | `standard` | `postExperimentSnapshot.ts:63` |
| 定时自动刷新 | `createSnapshot()` | **`true`**（显式传入） | `schedule` | `standard` | `updateExperimentResults.ts:189` |
| 仪表盘手动刷新（标准快照） | `planExperimentSnapshot()` + `createExperimentSnapshotFromPlan()` | **`false`**（显式传入） | `manual-dashboard` | `standard` | `dashboards.controller.ts:202` |
| 仪表盘手动刷新（维度快照） | `createExperimentSnapshot()` | **`false`**（显式传入） | `manual-dashboard` | `exploratory` | `dashboards.controller.ts:254` |
| 仪表盘自动更新 | `createSnapshot()` | **`true`**（显式传入） | `update-dashboards` | `exploratory` | `enterprise/services/dashboards.ts:206` |
| Holdout 自动刷新 | `createExperimentSnapshot()` | **`true`**（显式传入） | — | — | `holdout.controller.ts:269` |
| Demo 项目创建 | `createSnapshot()` | **`true`**（显式传入） | `manual` | `standard` | `demo-datasource-project.controller.ts:519` |

### 2.3 useCache 在规划阶段的作用

**`planSnapshot()`** (`experiments.ts:1335-1348`):

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

### 3.2 TTL 默认值

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

## 6. 仪表盘探索性快照的两条缓存路径

### 6.1 两条路径概览

仪表盘探索性快照有两条完全独立的生成路径，它们的 `useCache` 配置**相反**：

| 维度 | 路径 A：手动刷新 | 路径 B：自动更新 |
|------|------------------|-----------------|
| 触发方式 | 用户点击仪表盘"刷新"按钮 | 标准快照完成后后台自动触发 |
| 调用函数 | `refreshDashboardData()` | `updateExperimentDashboards()` |
| 调用入口 | `routers/dashboards/dashboards.controller.ts:180` | `enterprise/services/dashboards.ts:82` |
| 触发时机 | 用户主动操作 | 标准快照（`type="standard"`）完成后，且 `triggeredBy !== "manual-dashboard"` |
| triggeredBy | `manual-dashboard` | `update-dashboards` |
| 快照类型 | `exploratory` | `exploratory` |

### 6.2 路径 A：手动刷新（`refreshDashboardData()`）

**代码位置**：`routers/dashboards/dashboards.controller.ts:180-263`

**完整流程**：

```
用户点击仪表盘刷新
    ↓
1. 创建标准快照（无维度）
   ├─→ planExperimentSnapshot(useCache=false, type="standard", triggeredBy="manual-dashboard")
   ├─→ createExperimentSnapshotFromPlan() 执行
   └─→ 这是 manual-dashboard 的 standard 快照
    ↓
2. 遍历仪表盘 blocks
   ├─→ 判断 block 是否可被标准快照满足（snapshotSatisfiesBlock）
   │   ├─→ 是 → 复用标准快照的 snapshotId
   │   └─→ 否 → 需要创建探索性快照
    ↓
3. 对需要维度的 blocks，按维度分组
   └─→ 每组维度创建一个探索性快照
       ├─→ createExperimentSnapshot(useCache=false, type="exploratory", triggeredBy="manual-dashboard")
       └─→ 强制重跑，不使用缓存
```

**useCache 配置**：**全部为 `false`**（显式传入）

**为什么手动刷新禁用缓存**：
- 用户主动刷新仪表盘期望看到最新数据
- 保证数据新鲜度优先于性能
- 每个维度快照都强制重跑，不复用历史查询

### 6.3 路径 B：自动更新（`updateExperimentDashboards()`）

**代码位置**：`enterprise/services/dashboards.ts:82-233`

**触发时机**：`createSnapshotFromPlan()` 末尾 (`experiments.ts:1573-1597`)

```typescript
// 当标准快照刷新时，后台异步更新关联仪表盘
if (
  runningSnapshot.type === "standard" &&
  runningSnapshot.triggeredBy !== "manual-dashboard"
) {
  updateExperimentDashboards({
    context,
    experiment,
    mainSnapshot: runningSnapshot,
    ...
  });
}
```

**触发条件**：
- `runningSnapshot.type === "standard"` — 标准快照
- `runningSnapshot.triggeredBy !== "manual-dashboard"` — 排除手动仪表盘刷新（避免循环）

**完整流程**：

```
标准快照完成（triggeredBy ≠ manual-dashboard）
    ↓
1. 查找所有关联仪表盘（enableAutoUpdates=true）
    ↓
2. 遍历所有 blocks
   ├─→ 过滤出有 snapshotId 且属于当前实验的 blocks
   ├─→ 判断 block 是否可被主快照满足
   │   ├─→ 是 → 跳过（主快照已覆盖）
   │   └─→ 否 → 需要创建探索性快照
    ↓
3. 收集需要创建快照的 blocks
   ├─→ 提取之前的 snapshotId，加载历史快照
   ├─→ 从历史快照中提取分析设置
   └─→ 按快照设置去重（相同设置的 blocks 只跑一次）
    ↓
4. 为每组唯一设置创建探索性快照
   └─→ createSnapshot(useCache=true, type="exploratory", triggeredBy="update-dashboards")
       └─→ 允许使用缓存，优先复用 60 分钟内的相同查询
```

**useCache 配置**：**`true`**（显式传入）

**为什么自动更新启用缓存**：
- 自动更新由定时任务或 API 触发，可能频繁执行
- 60 分钟内的数据对于仪表盘通常足够新
- 性能优先，减少数据源负载

### 6.4 主快照复用逻辑（两条路径共享）

**`snapshotSatisfiesBlock()`** (`shared/enterprise/dashboards/utils.ts:123-137`):

```typescript
export function snapshotSatisfiesBlock(
  snapshot: ExperimentSnapshotInterface,
  block: DashboardBlockInterfaceOrData<DashboardBlockInterface>,
) {
  const blockSettings = getBlockSnapshotSettings(block);
  // 如果快照有维度，必须匹配 block 的维度
  if (snapshot.dimension) {
    return snapshot.dimension === blockSettings.dimensionId;
  }
  if (!blockSettings.dimensionId) return true;
  // 如果快照没有维度，检查请求的维度是否在预计算维度中
  return snapshot.settings.dimensions.some(
    ({ id }) => blockSettings.dimensionId === id,
  );
}
```

**复用规则**：
- 无维度的标准快照可以满足：无维度的 block，或维度在预计算维度列表中的 block
- 有维度的快照只能满足：相同维度的 block
- 不满足的 block 需要创建独立的探索性快照

### 6.5 两条路径的关键差异对比

| 对比维度 | 路径 A：手动刷新 | 路径 B：自动更新 |
|----------|-----------------|-----------------|
| useCache | `false`（强制重跑） | `true`（允许缓存） |
| 数据新鲜度 | 最新（强制重跑所有查询） | 60 分钟内可接受 |
| 触发方式 | 用户主动 | 系统自动 |
| 执行时机 | 同步等待 | 异步后台 |
| 运行器 | 取决于增量兼容性 | 取决于增量兼容性 |
| 适用场景 | 用户需要最新数据 | 定期维护仪表盘 |

### 6.6 仪表盘产品分析探索块的缓存

**代码位置**：`enterprise/services/dashboards.ts:348-377`

仪表盘的产品分析探索块（`metric-exploration` / `fact-table-exploration` / `data-source-exploration`）使用独立的缓存策略：

```typescript
// updateDashboardExplorations()
for (const block of explorationBlocks) {
  const exploration = await runProductAnalyticsExploration(
    context,
    block.config,
    { cache: "never" },  // 总是强制重跑
  );
  block.explorerAnalysisId = exploration.id;
}
```

**配置**：`cache: "never"` — 每次刷新仪表盘时强制重跑产品分析探索查询

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

## 8. 触发入口、运行器与数据新鲜度关联表

### 8.1 完整决策矩阵

| triggeredBy | 调用函数 | useCache | runnerKind | fullRefresh | 数据新鲜度 | 说明 |
|-------------|----------|----------|------------|-------------|-----------|------|
| `manual` | `createExperimentSnapshot()` | `true` | `results` 或 `incremental` | 取决于状态 | 60 分钟内可缓存 | 用户主动刷新实验结果 |
| `schedule` | `createSnapshot()` | `true` | `results` 或 `incremental` | 取决于状态 | 60 分钟内可缓存 | 定时自动刷新 |
| `manual-dashboard` | `createExperimentSnapshot()` | `false` | `results` 或 `incremental` | 取决于状态 | **最新**（强制重跑） | 用户手动刷新仪表盘 |
| `update-dashboards` | `createSnapshot()` | `true` | `results` 或 `incremental` | 取决于状态 | 60 分钟内可缓存 | 标准快照后自动更新仪表盘 |

### 8.2 运行器与缓存的关系

| runnerKind | useCache（运行器内） | 查询级缓存 | 增量刷新 | 数据新鲜度 |
|------------|---------------------|-----------|---------|-----------|
| `results` | 传入值（可 `true` 可 `false`） | 取决于传入值 | 不支持 | 取决于 useCache |
| `incremental` | **`false`（硬编码）** | 不使用查询缓存 | 支持（`fullRefresh` 控制首次/后续） | 增量模式保证最新 |
| `incremental-exploratory` | **`false`（硬编码）** | 不使用查询缓存 | 支持（只读模式） | 增量模式保证最新 |

### 8.3 数据新鲜度层级

| 新鲜度层级 | 触发方式 | 说明 |
|-----------|---------|------|
| **最新（强制重跑）** | `manual-dashboard` | 每次都跑所有新查询，数据绝对最新 |
| **增量最新** | `manual` + `incremental` 运行器 | 从上次增量位置更新，数据接近实时 |
| **60 分钟内** | `manual`/`schedule`/`update-dashboards` + `results` 运行器 | 60 分钟内的查询可被复用 |
| **增量缓存** | `schedule` + `incremental` 运行器 | 增量模式下不使用查询缓存，但增量更新保证数据较新 |

---

## 9. 缓存 vs 重跑完整决策流

### 9.1 顶层决策流程图

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
    ↓
如果 triggeredBy ≠ manual-dashboard 且 snapshotType=standard:
    └─→ 异步触发 updateExperimentDashboards()
        └─→ 为仪表盘创建 exploratory 快照（useCache=true）
```

### 9.2 关键决策点速查表

| 决策点 | 影响因素 | 结果 |
|--------|---------|------|
| useCache 初始值 | 调用入口（见 2.2 节） | `true` 或 `false` |
| fullRefresh 计算 | useCache + incrementalRefreshModel + unitsTableFullName | `true` 或 `false` |
| runnerKind 选择 | 数据源设置 + 实验兼容性 + 快照类型 + 维度 | `results` / `incremental` / `incremental-exploratory` |
| 运行器 useCache | runnerKind | 增量运行器恒为 `false`，传统运行器跟随传入值 |
| 实际 fullRefresh | plan.fullRefresh + snapshot.type="standard" | 仅 standard 快照可能全量刷新 |
| 查询级缓存 | 运行器 useCache + SQL精确匹配 + TTL（60分钟） | 命中或未命中 |

---

## 10. 故障排除指南

### 10.1 快照数据不更新

| 现象 | 可能原因 | 排查步骤 |
|------|---------|---------|
| 手动刷新后数据没变 | 仪表盘手动刷新走 `useCache=true`？不可能，`manual-dashboard` 是 `false` | 检查 triggeredBy 是否为 `manual-dashboard` |
| 定时刷新后数据没变 | TTL=60 分钟，1 小时内的查询可能被缓存 | 检查查询是否在 60 分钟内已执行过 |
| 增量模式数据不更新 | 增量刷新锁被占用 | 检查 `incrementalrefresh` 集合中 `currentExecutionSnapshotId` |
| 增量模式数据不更新 | 配置 hash 不匹配 | 检查 `experimentSettingsHash` 和 `metricSources[].settingsHash` |

### 10.2 仪表盘探索性快照问题

| 现象 | 可能原因 | 排查步骤 |
|------|---------|---------|
| 仪表盘数据延迟 | 自动更新走 `useCache=true`，60 分钟内的查询被复用 | 检查 triggeredBy 是否为 `update-dashboards` |
| 仪表盘刷新慢 | 手动刷新走 `useCache=false`，所有查询强制重跑 | 正常行为，手动刷新保证最新 |
| 仪表盘数据与实验页面不一致 | 仪表盘有独立的探索性快照，可能使用缓存 | 对比快照的 `triggeredBy` 字段 |
| 仪表盘只更新了部分 blocks | 主快照复用跳过部分 blocks | 检查 `snapshotSatisfiesBlock()` 逻辑 |

### 10.3 增量模式问题

| 现象 | 可能原因 | 排查步骤 |
|------|---------|---------|
| 增量刷新报错 | 兼容性检查失败 | 查看 `validateIncrementalPipeline()` 抛出的错误信息 |
| 增量刷新报错 | 配置 hash 不匹配 | 触发一次手动全量刷新（设置 `useCache=false`） |
| 增量刷新报错 | 数据源不支持增量刷新 | 检查 `integration.getSourceProperties().hasIncrementalRefresh` |
| 单位表不更新 | 锁未被正确释放 | 检查 `currentExecutionSnapshotId`，如超过 1 小时可认为是脏锁 |

### 10.4 查询缓存问题

| 现象 | 可能原因 | 排查步骤 |
|------|---------|---------|
| 查询被意外缓存 | TTL=60 分钟内的相同 SQL | 检查 SQL 文本是否完全一致 |
| 查询未被缓存 | SQL 文本有微小差异（空格、别名等） | 对比两次的完整 SQL |
| 查询未被缓存 | 查询失败（status ≠ succeeded/running） | 检查查询状态 |
| 查询未被缓存 | 从缓存复制的查询（`cachedQueryUsed` 存在）不会再次被复用 | 设计如此，防止级联复用 |

---

## 11. 核心代码位置

### 11.1 快照生成与规划

| 功能 | 文件 | 关键函数 |
|------|------|----------|
| 快照创建（默认 useCache=true） | `services/experiments.ts:1718` | `createExperimentSnapshot()` |
| 快照创建（默认 useCache=false） | `services/experiments.ts:1627` | `createSnapshot()` |
| 快照规划（核心决策） | `services/experiments.ts:1297` | `planSnapshot()` |
| 快照创建执行 | `services/experiments.ts:1434` | `createSnapshotFromPlan()` |
| 手动刷新 API | `api/experiments/postExperimentSnapshot.ts` | `postExperimentSnapshot` |
| 定时刷新任务 | `jobs/updateExperimentResults.ts:179` | `updateSingleExperiment()` |

### 11.2 缓存机制

| 功能 | 文件 | 关键函数 |
|------|------|----------|
| TTL 默认值定义 | `util/secrets.ts:152-156` | `QUERY_CACHE_TTL_MINS` |
| 查询缓存查找 | `models/QueryModel.ts:208-234` | `getRecentQuery()` |
| 缓存查询复制 | `models/QueryModel.ts:315-348` | `createNewQueryFromCached()` |
| 查询运行器基类 | `queryRunners/QueryRunner.ts` | `startQuery()` |
| 结果查询运行器 | `queryRunners/ExperimentResultsQueryRunner.ts` | `startQueries()` |

### 11.3 增量刷新

| 功能 | 文件 | 关键函数 |
|------|------|----------|
| 运行器类型决策 | `services/experiments.ts:1189-1226` | `getSnapshotQueryRunnerKind()` |
| 运行器规划（含兼容性检查） | `services/experiments.ts:1228-1287` | `planSnapshotQueryRunner()` |
| 增量兼容性检查 | `services/dataPipeline.ts:21-170` | `validateIncrementalPipeline()` |
| 增量刷新运行器 | `queryRunners/ExperimentIncrementalRefreshQueryRunner.ts` | `startQueries()` |
| 增量刷新锁与状态 | `models/IncrementalRefreshModel.ts` | `acquireLock()`, `releaseLock()` |

### 11.4 仪表盘路径

| 功能 | 文件 | 关键函数 |
|------|------|----------|
| 仪表盘手动刷新 | `routers/dashboards/dashboards.controller.ts:180` | `refreshDashboardData()` |
| 仪表盘自动更新 | `enterprise/services/dashboards.ts:82` | `updateExperimentDashboards()` |
| 主快照复用判断 | `shared/enterprise/dashboards/utils.ts:123` | `snapshotSatisfiesBlock()` |
| 仪表盘产品分析探索 | `enterprise/services/dashboards.ts:348` | `updateDashboardExplorations()` |
