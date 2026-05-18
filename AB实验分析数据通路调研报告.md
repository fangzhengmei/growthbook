# A/B 实验分析数据通路调研报告

## 1. 概述

GrowthBook 的 A/B 实验分析系统是一条从原始事件到统计结论的完整数据处理链路。本报告深入调研了从**原始事件映射**、**去重归因规则**、**查询编排**、**置信区间计算**、**异常边界处理**到**多级缓存机制**的完整数据流。

---

## 2. 原始事件到实验单位的映射

### 2.1 曝光查询 (Exposure Query)

曝光查询是整个分析链路的起点，负责从原始事件表中提取实验曝光记录。

#### 2.1.1 曝光数据结构

```sql
-- __rawExperiment: 原始曝光数据
SELECT
  e.user_id,
  e.variation_id,
  e.timestamp,
  e.experiment_id
FROM exposure_events e
WHERE e.experiment_id = 'exp_xxx'
  AND e.timestamp BETWEEN startDate AND endDate
```

**代码来源**：`packages/back-end/src/integrations/sql/queries/experiment-units-query.ts:80-122`

#### 2.1.2 身份关联 (Identity Join)

跨设备追踪需要通过身份映射表将不同 ID 类型关联：

```typescript
// 支持的身份类型链：user_id → anonymous_id → device_id
const idTypeObjects = [
  [exposureQuery.userIdType],           // 曝光使用的ID类型
  activationMetric ? getUserIdTypes(activationMetric, factTableMap) : [],
  ...unitDimensions.map(d => [d.dimension.userIdType || "user_id"]),
  segment ? [segment.userIdType || "user_id"] : [],
];

// 生成 Identity CTE 进行多表关联
const { baseIdType, idJoinMap, idJoinSQL } = getIdentitiesCTE(
  dialect, datasource.settings, {
    objects: idTypeObjects,
    from: settings.startDate,
    to: settings.endDate,
    forcedBaseIdType: exposureQuery.userIdType,
    experimentId: settings.experimentId,
  }
);
```

**代码来源**：`packages/back-end/src/integrations/sql/queries/experiment-units-query.ts:52-67`

### 2.2 实验单位去重 (Unit Deduplication)

#### 2.2.1 多曝光去重策略

每个用户可能多次看到实验，需要确定唯一的实验单位：

```sql
-- __experimentUnits: 每个用户一行
SELECT
  e.user_id,
  -- 策略1: 用户出现在多个变体中 → 标记为 '__multiple__'
  CASE
    WHEN count(DISTINCT e.variation_id) > 1 THEN '__multiple__'
    ELSE max(e.variation_id)
  END AS variation,
  -- 策略2: 使用首次曝光时间
  MIN(e.timestamp) AS first_exposure_timestamp
FROM __experimentExposures e
GROUP BY e.user_id
```

**代码来源**：`packages/back-end/src/integrations/sql/queries/experiment-units-query.ts:175-242`

#### 2.2.2 首次曝光确定 (First Exposure)

对于多臂老虎机等需要按时间加权的场景，使用更精确的首次曝光提取：

```typescript
function getFirstVariationValuePerUnit(dialect: SqlDialect): string {
  // 技巧：将时间戳和变体ID拼接后取最小值，再提取变体部分
  return `SUBSTRING(
    MIN(
      CONCAT(
        SUBSTRING(${dialect.formatDateTimeString("e.timestamp")}, 1, 19),
        COALESCE(${dialect.castToString("e.variation")}, 'NULL_VAR')
      )
    ),
    20,  -- 跳过前19位时间戳
    99999
  )`;
}
```

**代码来源**：`packages/back-end/src/integrations/sql/columns/first-variation-value-per-unit.ts:4-14`

#### 2.2.3 多变体冲突处理

| 场景 | 处理方式 | 说明 |
|-----|---------|------|
| 单变体用户 | 正常归入对应变体 | 标准流程 |
| 多变体用户 | 标记为 `__multiple__` | 从统计分析中排除，单独报告 |
| Bandit 场景 | 使用首次曝光变体 | `useFirstExposure` 标志控制 |

---

## 3. 去重与归因规则

### 3.1 归因模型 (Attribution Model)

GrowthBook 支持三种归因模型，控制指标事件的时间窗口：

| 模型 | 描述 | 适用场景 |
|-----|------|---------|
| **firstExposure** | 仅统计首次曝光后的转化窗口内的事件 | 标准实验，关注即时影响 |
| **experimentDuration** | 统计整个实验期间的事件，不受转化窗口限制 | 长期影响评估 |
| **lookbackOverride** | 使用实验结束前的 X 小时数据 | 特定分析需求 |

**代码来源**：`packages/shared/src/validators/experiments.ts:131-136`

### 3.2 转化窗口规则 (Conversion Window)

```typescript
function getConversionWindowClause(
  dialect: SqlDialect,
  baseCol: string,      // 曝光时间戳列
  metricCol: string,    // 指标时间戳列
  metric: ExperimentMetricInterface,
  endDate: Date,
  overrideConversionWindows: boolean
): string {
  const windowHours = getMetricWindowHours(metric.windowSettings);
  const delayHours = getDelayWindowHours(metric.windowSettings);

  // 基础条件：指标事件必须在曝光 + delay 之后
  let metricWindow = `${metricCol} >= ${addHours(dialect, baseCol, delayHours)}`;

  if (metric.windowSettings.type === "conversion" && !overrideConversionWindows) {
    // 转化窗口：事件必须在曝光 + delay + window 之内
    // 可以超过实验结束日期
    metricWindow = `${metricWindow}
      AND ${metricCol} <= ${addHours(dialect, baseCol, delayHours + windowHours)}`;
  } else {
    // 否则事件必须在实验结束日期之前
    metricWindow = `${metricWindow}
      AND ${metricCol} <= ${dialect.toTimestamp(endDate)}`;
  }

  if (metric.windowSettings.type === "lookback") {
    // 回溯窗口：只统计实验结束前 X 小时内的事件
    metricWindow = `${metricWindow}
      AND ${addHours(dialect, metricCol, windowHours)} >= ${dialect.toTimestamp(endDate)}`;
  }

  return metricWindow;
}
```

**代码来源**：`packages/back-end/src/integrations/sql/clauses/conversion-window-clause.ts:10-54`

### 3.3 窗口设置详解

```typescript
interface WindowSettings {
  type: "conversion" | "lookback" | "immediate";
  delayValue: number;       // 延迟时间（曝光后多久开始统计）
  delayUnit: "hours" | "days";
  windowValue: number;      // 窗口持续时间
  windowUnit: "hours" | "days";
}

// 示例：
// 曝光后 1 小时开始，24 小时内的转化 → delay=1, window=24
// 只看曝光当天的转化 → delay=0, window=24, type=conversion
```

### 3.4 激活指标 (Activation Metric)

激活指标用于筛选"真正进入实验"的用户：

```sql
-- 仅统计激活用户
LEFT JOIN __activationMetric a
  ON a.user_id = e.user_id
  AND a.timestamp BETWEEN
    e.timestamp AND e.timestamp + INTERVAL '24 hours'

-- 在单位层面标记激活状态
MIN(CASE WHEN a.timestamp IS NOT NULL THEN a.timestamp ELSE NULL END)
  AS first_activation_timestamp
```

**代码来源**：`packages/back-end/src/integrations/sql/queries/experiment-units-query.ts:202-216`

---

## 4. 关键异常边界处理

### 4.1 异常值截断 (Capping)

#### 4.1.1 截断类型

| 类型 | 配置 | 效果 |
|-----|------|------|
| **absolute** | `cappingSettings.value = 1000` | 所有值 > 1000 的都截断为 1000 |
| **percentile** | `cappingSettings.value = 0.99` | 截断到 P99 分位值 |
| **none** | 无 | 不截断 |

#### 4.1.2 截断实现

```typescript
function capCoalesceValue(dialect, { valueCol, metric, ... }) {
  // 1. 绝对值截断
  if (metric.cappingSettings.type === "absolute" &&
      metric.cappingSettings.value) {
    return `LEAST(
      COALESCE(${valueCol}, 0),
      ${metric.cappingSettings.value}
    )`;
  }

  // 2. 百分位截断（需先计算分位值）
  if (metric.cappingSettings.type === "percentile" &&
      metric.cappingSettings.value < 1) {
    return `LEAST(
      COALESCE(${valueCol}, 0),
      cap.value_cap
    )`;
  }

  // 3. 无截断，仅处理 NULL
  return `COALESCE(${valueCol}, 0)`;
}
```

**代码来源**：`packages/back-end/src/integrations/sql/primitives/cap-coalesce-value.ts:9-59`

#### 4.1.3 百分位截断流程

```sql
-- 步骤1: 计算截断阈值（CTE）
WITH __capValue AS (
  SELECT
    PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY value) AS value_cap
  FROM __userMetricValues
)

-- 步骤2: 应用截断
SELECT
  LEAST(COALESCE(m.value, 0), c.value_cap) AS capped_value
FROM __userMetricValues m
CROSS JOIN __capValue c
```

### 4.2 查询结果行数限制

```typescript
// 单位聚合查询最大行数（避免维度爆炸）
export const MAX_ROWS_UNIT_AGGREGATE_QUERY = 3000;

// 检查并返回错误
if (rows.length == MAX_ROWS_UNIT_AGGREGATE_QUERY) {
  return {
    overall: overallResult,
    dimension: {},
    error: "TOO_MANY_ROWS",
  };
}
```

**代码来源**：
- `packages/back-end/src/services/experimentQueries/constants.ts:4`
- `packages/back-end/src/services/stats.ts:717-723`

### 4.3 多指标查询分块逻辑

```typescript
// 单查询最大指标数（防止SQL列数过多）
export const MAX_METRICS_PER_QUERY = 200;

// 分块逻辑：按列数限制自动分块
function chunkMetrics(metrics, maxColumnsPerQuery, isBandit) {
  // 基础列开销：100维度 + 1variation + 2users/count = 103列
  const baseColumnsNeeded = 103;

  // 每个指标列开销取决于：
  // - 指标类型（mean/ratio/quantile）
  // - 是否启用回归调整（CUPED）
  // - 是否启用百分位截断
  // - 是否需要未截断版本
  // - 是否Bandit模式（额外theta列）
  // 按列数限制自动分块
}
```

**代码来源**：
- 常量定义：`packages/back-end/src/services/experimentQueries/constants.ts:2`
- 分块逻辑：`packages/back-end/src/services/experimentQueries/experimentQueries.ts:177-220`
- 列数计算：`packages/back-end/src/services/experimentQueries/experimentQueries.ts:36-139`

### 4.4 查询状态聚合规则

```typescript
private getOverallQueryStatus(): QueryStatus {
  const failedQueries = this.model.queries.filter(q => q.status === "failed");
  const runningQueries = this.model.queries.filter(q => q.status === "running");
  const queuedQueries = this.model.queries.filter(q => q.status === "queued");

  const totalQueries = this.model.queries.length;

  // 超过半数失败 → 整体失败
  if (failedQueries.length >= totalQueries / 2) return "failed";

  // 有运行中或排队的 → 运行中
  if (queuedQueries.length + runningQueries.length > 0) return "running";

  // 部分失败 → 部分成功
  if (failedQueries.length > 0) return "partially-succeeded";

  // 全部成功
  return "succeeded";
}
```

**代码来源**：`packages/back-end/src/queryRunners/QueryRunner.ts:840-860`

### 4.5 僵尸查询检测

```typescript
async function getStaleQueries() {
  // 查询心跳每30秒更新一次
  // 超过70秒无心跳 → 认为进程已死
  const lastHeartbeat = new Date();
  lastHeartbeat.setSeconds(lastHeartbeat.getSeconds() - 70);

  const stale = await QueryModel.find({
    status: "running",
    heartbeat: { $lt: lastHeartbeat },
  }).limit(20);

  // 标记为失败
  await QueryModel.updateMany(
    { ...query, _id: { $in: stale.map(d => d._id) } },
    {
      $set: {
        status: "failed",
        error: "Query execution was interrupted. Please try again.",
      },
    }
  );
}
```

**代码来源**：`packages/back-end/src/models/QueryModel.ts:236-273`

---

## 5. 置信区间计算链路

置信区间（Confidence Interval, CI）是 A/B 实验统计结论的核心输出。本章结构化说明从配置参数传入统计引擎，到结果格式化的完整链路。

### 5.1 参数传递链：从配置到统计引擎

#### 5.1.1 可配置参数总览

| 参数名 | 前端配置字段 | 后端类型 | 默认值 | 作用 |
|-------|-------------|---------|-------|------|
| **alpha** | `pValueThreshold` | `number` | 0.05 | 显著性水平，决定置信度（1-alpha） |
| **p值校正** | `pValueCorrection` | `boolean` | `false` | 是否启用多重检验校正 |
| **单侧区间** | `oneSidedIntervals` | `boolean` | `false` | `true`=单侧区间，`false`=双侧区间 |
| **统计引擎** | `statsEngine` | `string` | `"frequentist"` | `"bayesian"` 或 `"frequentist"` |
| **差异类型** | `differenceType` | `string` | `"relative"` | `"relative"`（相对）或 `"absolute"`（绝对） |
| **序列检验** | `sequentialTesting` | `boolean` | `false` | 是否启用序列检验 |

#### 5.1.2 参数转换与传入

所有参数通过 `getAnalysisSettingsForStatsEngine` 函数统一转换为 Python 统计引擎的输入格式：

```typescript
export function getAnalysisSettingsForStatsEngine(
  settings: ExperimentSnapshotAnalysisSettings,
  variations: ExperimentReportVariation[],
  coverage: number,
  phaseLengthDays: number,
): AnalysisSettingsForStatsEngine {
  // 1. alpha 转换：pValueThreshold → alpha
  const pValueThresholdNumber =
    Number(settings.pValueThreshold) || DEFAULT_P_VALUE_THRESHOLD;

  const analysisData: AnalysisSettingsForStatsEngine = {
    // 基础配置
    stats_engine: settings.statsEngine,
    alpha: pValueThresholdNumber,          // 传入显著性水平
    difference_type: settings.differenceType,
    
    // 置信区间相关开关
    p_value_corrected: !!settings.pValueCorrection,
    one_sided_intervals: !!settings.oneSidedIntervals,
    sequential_testing_enabled: settings.sequentialTesting ?? false,
    
    // ... 其他参数
  };
  return analysisData;
}
```

**代码来源**：`packages/back-end/src/services/stats.ts:71-114`

#### 5.1.3 统计引擎内部使用

参数传入 Python 统计引擎后，在不同统计范式下的使用方式：

| 统计范式 | alpha 用途 | 单侧区间影响 | p值校正方式 |
|---------|-----------|-------------|------------|
| **频率派** | 计算临界值：`z_{1-alpha/2}`（双侧）或 `z_{1-alpha}`（单侧） | 单侧：CI = [estimate - z_{1-alpha}*SE, +∞) 或 (-∞, estimate + z_{1-alpha}*SE] | holm-bonferroni 法校正 alpha |
| **贝叶斯** | 确定后验分布的 Credible Interval 分位数：`[alpha/2, 1-alpha/2]` | 单侧：分位数取 `[alpha, 1]` 或 `[0, 1-alpha]` | benjamini-hochberg 法控制 FDR |

**代码来源**：`packages/stats/` 目录下 Python 统计引擎

### 5.2 结果格式化：CI 中 null 边界的处理路径

Python 统计引擎返回的 CI 中可能包含 `null` 值，表示边界未定义。后端在结果解析阶段将其归一化为 `±Infinity`。

#### 5.2.1 归一化函数

```typescript
const getFormattedCI = (
  ci?: [number | null, number | null],
): [number, number] | undefined => {
  if (!ci) return undefined;
  // 左边界 null → -Infinity，右边界 null → +Infinity
  return [ci[0] ?? -Infinity, ci[1] ?? Infinity];
};
```

**代码来源**：`packages/back-end/src/services/stats.ts:457-462`

#### 5.2.2 归一化调用点

在 `parseStatsEngineResult` 函数中，对 5 类结果的 CI 逐一进行归一化处理：

```typescript
row.variations.forEach((v, i) => {
  // 1. 主结果 CI
  if ("ci" in v) {
    v.ci = getFormattedCI(v.ci);
  }
  // 2. CUPED 未调整版本 CI
  if (v.supplementalResults?.cupedUnadjusted && "ci" in v.supplementalResults.cupedUnadjusted) {
    v.supplementalResults.cupedUnadjusted.ci = getFormattedCI(v.supplementalResults.cupedUnadjusted.ci);
  }
  // 3. 未截断版本 CI
  if (v.supplementalResults?.uncapped && "ci" in v.supplementalResults.uncapped) {
    v.supplementalResults.uncapped.ci = getFormattedCI(v.supplementalResults.uncapped.ci);
  }
  // 4. 未分层版本 CI
  if (v.supplementalResults?.unstratified && "ci" in v.supplementalResults.unstratified) {
    v.supplementalResults.unstratified.ci = getFormattedCI(v.supplementalResults.unstratified.ci);
  }
  // 5. 无方差缩减版本 CI
  if (v.supplementalResults?.noVarianceReduction && "ci" in v.supplementalResults.noVarianceReduction) {
    v.supplementalResults.noVarianceReduction.ci = getFormattedCI(v.supplementalResults.noVarianceReduction.ci);
  }
});
```

**代码来源**：`packages/back-end/src/services/stats.ts:510-552`

#### 5.2.3 null 边界的业务含义

| CI 原始值 | 归一化后 | 业务场景 |
|----------|---------|---------|
| `[null, null]` | `[-Infinity, Infinity]` | 样本量不足，无法计算有效区间 |
| `[0.01, null]` | `[0.01, Infinity]` | 单侧下限（仅关注提升），上界无约束 |
| `[null, -0.01]` | `[-Infinity, -0.01]` | 单侧上限（仅关注下降），下界无约束 |
| `[-0.02, 0.05]` | `[-0.02, 0.05]` | 正常双侧区间，无需转换 |

#### 5.2.4 最终表达

归一化后的 CI 存储在快照的 `results` 字段中，前端可直接使用：
- `Infinity` / `-Infinity` 在 JSON 序列化时保持原值
- 前端图表渲染时可识别并展示为"无边界"或"未计算"
- 统计显著性判断时，可正确处理区间是否包含 0

### 5.3 置信区间链路总览

```
用户配置（UI/API）
    │
    ▼
settings.pValueThreshold → alpha
settings.pValueCorrection → p_value_corrected
settings.oneSidedIntervals → one_sided_intervals
    │
    ▼
getAnalysisSettingsForStatsEngine()  [stats.ts:71-114]
    │
    ▼
Python 统计引擎（频率派/贝叶斯）
    │  计算 CI，边界缺失时返回 null
    ▼
parseStatsEngineResult()  [stats.ts:510-552]
    │
    ├─> 主结果 CI → getFormattedCI()
    ├─> CUPED 未调整 CI → getFormattedCI()
    ├─> 未截断 CI → getFormattedCI()
    ├─> 未分层 CI → getFormattedCI()
    └─> 无方差缩减 CI → getFormattedCI()
    │
    ▼
快照持久化（ExperimentSnapshot.results）
    │
    ▼
前端展示（图表渲染、显著性判断）
```

---

## 6. 缓存机制详解

### 5.1 三级缓存架构总览

```
┌───────────────────────────────────────────────────────────┐
│  L1: 内存缓存 (MemoryCache)                               │
│  存储位置: Node.js 内存 Map                                │
│  生命周期: 短 TTL（默认 30秒）                            │
│  用途: 热点配置、权限检查、频繁读取的元数据                │
└─────────────────────────────┬─────────────────────────────┘
                              │
┌─────────────────────────────▼─────────────────────────────┐
│  L2: 查询缓存 (QueryModel)                                │
│  存储位置: MongoDB queries 集合                           │
│  生命周期: TTL 60分钟（可配置）                           │
│  用途: SQL 查询结果去重，避免重复计算                      │
└─────────────────────────────┬─────────────────────────────┘
                              │
┌─────────────────────────────▼─────────────────────────────┐
│  L3: 快照持久化 (ExperimentSnapshot)                      │
│  存储位置: MongoDB experimentSnapshots 集合               │
│  生命周期: 永久存储，直到手动删除或实验归档                │
│  用途: 完整实验分析结果，支持回溯和对比                    │
└───────────────────────────────────────────────────────────┘
```

### 5.2 L1: 内存缓存 (MemoryCache)

#### 5.2.1 实现机制

```typescript
class MemoryCache<T, K> {
  private store: Map<K, { expires: number; obj: T }>;
  private expensiveOperation: ExpensiveOperation<T, K>;
  private ttl: number;  // 默认 30秒

  async get(key: K): Promise<T> {
    // 1. 检查缓存是否存在且未过期
    const existing = this.store.get(key);
    if (existing && existing.expires > Date.now()) {
      return existing.obj;  // 缓存命中
    }

    // 2. 缓存未命中，执行原始操作
    const obj = await this.expensiveOperation(key);

    // 3. 写入缓存
    this.store.set(key, {
      expires: Date.now() + this.ttl,
      obj,
    });
    return obj;
  }
}
```

**代码来源**：`packages/back-end/src/services/cache.ts:1-32`

#### 5.2.2 触发条件与失效方式

| 维度 | 说明 |
|-----|------|
| **触发时机** | 首次调用 `.get(key)` 时 |
| **命中条件** | key 存在 且 `expires > Date.now()` |
| **失效方式** | 1. 时间过期（被动失效）<br>2. 进程重启（全部失效）<br>3. 无主动失效 API |
| **典型用途** | 组织配置、权限检查、数据源连接信息 |
| **并发安全** | 无锁，高并发下可能重复执行 expensiveOperation |

### 5.3 L2: 查询缓存 (QueryModel)

#### 5.3.1 缓存匹配逻辑

```typescript
async function getRecentQuery(
  organization: string,
  datasource: string,
  query: string,           // SQL 全文作为缓存键
  cacheTTLMins?: number
) {
  const ttl = cacheTTLMins ?? QUERY_CACHE_TTL_MINS;  // 默认 60分钟
  const earliestDate = new Date();
  earliestDate.setMinutes(earliestDate.getMinutes() - ttl);

  // 精确匹配 SQL 文本 + 时间窗口
  return await QueryModel.findOne({
    organization,
    datasource,
    query,                    // 精确匹配 SQL
    createdAt: { $gt: earliestDate },
    status: { $in: ["succeeded", "running"] },
    cachedQueryUsed: { $exists: false },  // 排除链式缓存
  }).sort({ createdAt: -1 });
}
```

**代码来源**：`packages/back-end/src/models/QueryModel.ts:208-234`

#### 5.3.2 缓存复用流程

```typescript
// QueryRunner 启动查询前的缓存检查
async function startQueries(params) {
  // 1. 生成 SQL
  const sql = buildExperimentQuery(params);

  // 2. 查找最近成功的相同查询
  const cached = await getRecentQuery(orgId, dsId, sql);

  if (cached) {
    // 3. 创建缓存引用记录（不重复执行）
    return await createNewQueryFromCached({
      existing: cached,
      dependencies: [],
    });
  }

  // 4. 无缓存，创建新查询执行
  return await createNewQuery({
    organization: orgId,
    datasource: dsId,
    query: sql,
    ...
  });
}
```

**代码来源**：`packages/back-end/src/models/QueryModel.ts:315-348`

#### 5.3.3 触发条件与失效方式

| 维度 | 说明 |
|-----|------|
| **缓存键** | SQL 全文精确匹配 |
| **触发时机** | 每次创建新查询前自动检查 |
| **命中条件** | 1. SQL 完全相同<br>2. 创建时间在 TTL 内（默认 60分钟）<br>3. 状态为 succeeded 或 running<br>4. 不是从其他缓存复制而来 |
| **失效方式** | 1. 时间过期（被动，下次查询时自然不命中）<br>2. 无主动失效机制<br>3. 非级联失效（cachedQueryUsed 标记防止链式复用） |
| **存储开销** | 查询结果存在单独的 sqlResultChunks 集合（分块存储） |
| **可配置项** | `QUERY_CACHE_TTL_MINS` 环境变量 |

#### 5.3.4 运行时缓存 (finishedQueryMapCache)

```typescript
// QueryRunner 实例内的内存缓存
private finishedQueryMapCache: QueryMap = new Map();

async getQueryMap(pointers: Queries): Promise<QueryMap> {
  return getQueryMap(this.context, pointers, this.finishedQueryMapCache);
}

// 成功的查询自动加入内存缓存
if (query.status === "succeeded" && cache) {
  cache.set(pointer.name, query);
}
```

**代码来源**：`packages/back-end/src/queryRunners/QueryRunner.ts:77-105, 237-239`

### 5.4 L3: 快照持久化 (ExperimentSnapshot)

#### 5.4.1 快照数据结构

```typescript
interface ExperimentSnapshotInterface {
  id: string;
  experiment: string;           // 关联实验ID
  phase: number;                // 实验阶段
  type: SnapshotType;           // full / incremental
  status: "running" | "succeeded" | "failed" | "partially-succeeded";

  settings: ExperimentSnapshotSettings;  // 快照配置（指标、维度、归因等）
  analyses: ExperimentSnapshotAnalysis[]; // 多组分析（不同统计引擎/维度）
  results: ExperimentReportResultDimension[];  // 最终结果

  queries: Queries;             // 执行的查询记录（可追溯）
  error?: string;               // 错误信息
  unknownVariations: string[];  // 未知变体ID
  multipleExposures: number;    // 多曝光用户数

  dateCreated: Date;
  runStarted: Date;
}
```

**代码来源**：`packages/back-end/src/models/ExperimentSnapshotModel.ts:63-150`

#### 5.4.2 快照分析结果存储

```typescript
// 每个维度的结果
interface ExperimentReportResultDimension {
  name: string;                 // 维度值，如 "All", "US", "Mobile"
  srm: number;                  // SRM p-value
  variations: {
    users: number;              // 用户数
    metrics: {
      [metricId: string]: {
        value: number;          // 指标总值
        cr: number;             // 转化率/均值
        ci: [number, number];   // 置信区间
        uplift: { dist: string; mean: number; stddev: number };
        expected: number;       // 期望提升
        chanceToWin?: number;   // 贝叶斯获胜概率
        pValue?: number;        // 频率派 p值
        risk?: [number, number]; // 风险
        supplementalResults?: {  // 补充分析
          cupedUnadjusted?: ...; // 未调整CUPED的结果
          uncapped?: ...;        // 未截断的结果
          unstratified?: ...;    // 未分层的结果
        };
      };
    };
  }[];
}
```

#### 5.4.3 触发条件与失效方式

| 维度 | 说明 |
|-----|------|
| **触发时机** | 1. **手动触发**：用户通过 API 或界面点击"分析"，可指定具体阶段<br>2. **调度触发**：定时任务根据 `nextSnapshotAttempt` 自动执行（需 `autoSnapshots: true` 且 `disableAutoSnapshots: false`）<br>3. **阶段变更不自动触发**：新阶段创建后不会自动生成快照，需手动或等待下一次调度 |
| **创建频率** | 调度周期由组织级 `updateSchedule` 控制（默认每 X 小时，可配置 Cron 表达式），或手动随时触发 |
| **版本管理** | 每次分析创建新快照，不覆盖历史；每个快照通过 `phase` 字段关联到对应实验阶段 |
| **失效方式** | 1. **手动删除**：通过 API 显式删除指定快照<br>2. **无自动过期**：快照永久存储，不会因时间流逝自动失效<br>3. **实验归档不清理**：实验标记为 `archived` 状态时，关联的快照数据保持完整，不会被级联删除 |
| **增量更新** | 支持增量刷新（Incremental Refresh），仅查询新增数据 |
| **可追溯性** | 保存完整 SQL 和查询 ID，可复现历史结果 |

#### 5.4.4 阶段变更与快照触发的关系

快照触发与实验阶段变更是两个独立的流程，不存在自动联动。具体关系如下：

**1. 阶段变更的独立流程**
- 实验阶段（phase）的创建、修改是独立操作，通过 `updateExperiment` API 完成
- 新增阶段后，实验的 `phases` 数组长度增加，但不会触发任何快照创建
- 阶段变更（如调整流量权重、新增变体）仅修改实验元数据，不影响已有快照

**2. 快照触发的三种路径**

| 触发方式 | 触发源 | 目标阶段 | 说明 |
|---------|--------|---------|------|
| **手动触发** | 用户点击界面"分析"按钮 / 调用 `postExperimentSnapshot` API | 可指定任意 `phase`（默认最新阶段） | 立即执行一次完整分析，`triggeredBy` 字段标记为 `"manual"` 或 `"manual-dashboard"` |
| **调度触发** | 后台定时任务（根据 `nextSnapshotAttempt`） | 始终针对最新阶段 `experiment.phases.length - 1` | 由组织级 `updateSchedule` 控制频率，`triggeredBy` 标记为 `"scheduler"`；需满足 `autoSnapshots: true` 且 `disableAutoSnapshots: false` |
| **阶段变更** | 无自动触发 | - | 阶段变更后**不会自动**创建快照；如需分析新阶段数据，需手动触发或等待下一次调度 |

**3. 自动快照开关逻辑**

```typescript
// 自动快照生效条件：
const autoRefreshEnabled = 
  experiment.autoSnapshots === true && 
  experiment.disableAutoSnapshots !== true;

// 下次调度时间计算（determineNextDate）：
// - 默认：EXPERIMENT_REFRESH_FREQUENCY 小时（1~168小时范围）
// - Cron 表达式：解析 cron 规则计算下次执行时间
// - Stale 模式：按指定小时数间隔
// - Never：返回 null，完全禁用自动快照
```

**代码来源**：`packages/back-end/src/services/experiments.ts:763-789, 3550-3555, 3912-3918`

**4. 典型时序示例**

```
Day 1 10:00 — 实验启动，创建 Phase 0
Day 1 10:05 — 用户手动触发快照 → 生成 Snapshot #1 (phase=0)
Day 1 14:00 — 调度触发 → 生成 Snapshot #2 (phase=0)
Day 2 09:00 — 用户新增 Phase 1（调整流量权重）
Day 2 09:00 — 无快照生成（阶段变更不自动触发）
Day 2 10:00 — 用户手动触发分析 → 生成 Snapshot #3 (phase=1)
Day 2 14:00 — 调度触发 → 生成 Snapshot #4 (phase=1，最新阶段)
```

---

## 6. 完整数据通路流程图

```
原始事件表 (exposure_events, metric_events)
         │
         ▼
┌─────────────────────────┐
│  曝光查询 (Exposure)    │
│  - 提取实验曝光记录      │
│  - 时间范围过滤          │
│  - 实验ID匹配            │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│  身份关联 (Identities)  │
│  - user_id ↔ anonymous_id│
│  - 跨设备追踪            │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│  实验单位去重           │
│  - 每个用户唯一分组      │
│  - 首次曝光时间          │
│  - 多变体检测 (__multiple__) │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐     ┌─────────────────────────┐
│  维度/分段处理          │────▶│  激活指标过滤           │
│  - 按维度切片            │     │  - 仅统计激活用户       │
│  - 分段条件应用          │     │  - 激活时间窗口         │
└───────────┬─────────────┘     └─────────────────────────┘
            │
            ▼
┌─────────────────────────┐
│  指标数据关联           │
│  - 转化窗口应用          │
│  - 归因模型选择          │
│  - 异常值截断            │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│  SQL 聚合计算           │
│  - SUM, SUM_SQUARES      │
│  - COUNT(DISTINCT)       │
│  - 按变体/维度分组       │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│  查询缓存检查           │
│  - 60分钟内相同SQL?      │
│  ├─ 是 → 复用结果        │
│  └─ 否 → 执行查询        │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│  Python 统计引擎        │
│  - 置信区间计算          │
│  - CUPED 调整            │
│  - 序列检验校正          │
│  - 贝叶斯/频率派计算     │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│  结果解析与格式化       │
│  - CI 边界处理(null→±∞) │
│  - SRM 检查              │
│  - 维度聚合              │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│  快照持久化             │
│  - 写入 MongoDB         │
│  - 保存完整分析链        │
└─────────────────────────┘
```

---

## 8. 关键技术要点总结

### 7.1 数据正确性保障

| 保障机制 | 实现方式 |
|---------|---------|
| **去重准确** | 每个用户仅归入首次曝光的变体，多变体用户标记排除 |
| **归因正确** | 三层转化窗口控制（delay + window + attribution model） |
| **异常抑制** | 绝对值截断 + 百分位截断，防止极端值影响 |
| **SRM 检测** | 卡方检验监测样本比率是否匹配预期权重 |
| **可追溯性** | 快照保存完整 SQL 和查询 ID，结果可复现 |

### 7.2 性能优化要点

| 优化手段 | 效果 |
|---------|------|
| **查询缓存** | 60分钟内相同查询直接复用，避免重复计算 |
| **分块查询** | 多指标按列数限制分块，避免单查询过大 |
| **增量刷新** | 仅查询新增数据，减少扫描范围 |
| **流水线模式** | 中间结果写入临时表，加速后续查询 |
| **连接池** | Python 统计引擎进程池复用，避免启动开销 |

### 7.3 边界场景处理

| 边界场景 | 处理策略 |
|---------|---------|
| **多变体用户** | 标记为 `__multiple__`，从统计中排除，单独报告 |
| **查询行数爆炸** | MAX_ROWS_UNIT_AGGREGATE_QUERY = 3000，超限报错 |
| **僵尸查询** | 70秒无心跳自动标记为失败 |
| **部分查询失败** | 超过半数失败则整体失败，否则部分成功 |
| **CI 边界值** | null 转换为 ±Infinity，保证前端展示正确 |

---

## 8. 参考代码位置

| 模块 | 文件路径 |
|-----|---------|
| 曝光查询构建 | `packages/back-end/src/integrations/sql/queries/experiment-units-query.ts` |
| 首次曝光提取 | `packages/back-end/src/integrations/sql/columns/first-variation-value-per-unit.ts` |
| 转化窗口规则 | `packages/back-end/src/integrations/sql/clauses/conversion-window-clause.ts` |
| 异常值截断 | `packages/back-end/src/integrations/sql/primitives/cap-coalesce-value.ts` |
| 内存缓存 | `packages/back-end/src/services/cache.ts` |
| 查询缓存 | `packages/back-end/src/models/QueryModel.ts` |
| 查询执行器 | `packages/back-end/src/queryRunners/QueryRunner.ts` |
| 统计引擎调用 | `packages/back-end/src/services/stats.ts` |
| Python 进程池 | `packages/back-end/src/services/python.ts` |
| 快照模型 | `packages/back-end/src/models/ExperimentSnapshotModel.ts` |
