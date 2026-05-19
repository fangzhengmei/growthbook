# 指标定义双路径衔接分析

## 概述

GrowthBook 中指标定义支持两条独立路径：
1. **SQL 直接定义路径**（Legacy Metric）：用户直接编写 SQL 定义指标
2. **Fact Table 路径**（Fact Metric）：指向预定义的 Fact Table，通过列引用和聚合方式定义指标

两条路径**共享大量前置准备步骤**，在 SQL 生成的核心环节分叉进入各自独立的优化路径，最终在统计引擎层实现三层汇合。

---

## 1. 共享前置步骤（两条路径共用）

### 1.1 共享步骤总览

在 `startExperimentResultQueries()` 中，两条路径共享以下前置步骤：

```
startExperimentResultQueries()
├── 1. 基础参数解析
│   ├── 从 snapshotSettings 提取 selectedMetrics
│   ├── 解析 segmentObj
│   ├── 查找 exposureQuery
│   └── 解析 snapshotDimensions
├── 2. Units Table 准备（可选）
│   ├── 判断是否使用临时 units table
│   └── （如需要）启动 experimentUnitsTable 查询
├── 3. 【分叉点】getFactMetricGroups() 分组
│   └── 将指标分为 legacyMetricSingles 和 factMetricGroups
└── 4. 分叉执行：各自的 SQL 生成与查询
```

### 1.2 共享步骤 1：基础参数解析

**文件**：`ExperimentResultsQueryRunner.ts:87-134`

两条路径共享的参数准备：

```typescript
// 1. 提取所有选中的指标（两种类型混合在同一个数组中）
const selectedMetrics = snapshotSettings.metricSettings
  .map((m) => metricMap.get(m.id))
  .filter((m) => m) as ExperimentMetricInterface[];

// 2. 解析 segment
let segmentObj: SegmentInterface | null = null;
if (snapshotSettings.segment) {
  segmentObj = await context.models.segments.getById(snapshotSettings.segment);
}

// 3. 查找 exposure query
const exposureQuery = (settings?.queries?.exposure || []).find(
  (q) => q.id === snapshotSettings.exposureQueryId,
);

// 4. 解析维度
const snapshotDimensions: Dimension[] = (await Promise.all(
  snapshotSettings.dimensions.map(
    async (d) => await parseDimension(d.id, d.slices, org.id),
  ),
)).filter((d): d is Dimension => d !== null);
```

### 1.3 共享步骤 2：Units Table 准备

**文件**：`ExperimentResultsQueryRunner.ts:138-206`

两条路径共享 units table 的创建逻辑：

```typescript
const useUnitsTable = (integration.getSourceProperties().supportsWritingTables && ...);
let unitQuery: QueryPointer | null = null;

if (useUnitsTable) {
  unitQuery = await startQuery({
    name: queryParentId,
    query: integration.getExperimentUnitsTableQuery(unitQueryParams),
    run: (query, setExternalId, queryMetadata) =>
      integration.runExperimentUnitsQuery(query, setExternalId, queryMetadata),
    queryType: "experimentUnits",
  });
  queries.push(unitQuery);
}
```

**关键点**：后续所有 metric 查询（无论 Legacy 还是 Fact）都将 `unitQuery` 作为依赖：
```typescript
// Legacy 路径
dependencies: unitQuery ? [unitQuery.query] : [],

// Fact 路径
dependencies: unitQuery ? [unitQuery.query] : [],
```

### 1.4 共享步骤 3：SQL 生成层的公共前置

在各自的 SQL 生成函数内部，两条路径也共享大量前置逻辑：

| 共享步骤 | Legacy 位置 | Fact 位置 | 共享函数 |
|---------|-------------|-----------|---------|
| Activation Metric 处理 | experiment-metric-query.ts:58-61 | experiment-fact-metrics-query.ts:42-45 | `processActivationMetric()` |
| Metric Overrides 应用 | experiment-metric-query.ts:63-64 | experiment-fact-metrics-query.ts:47-49 | `applyMetricOverrides()` |
| 维度处理 | experiment-metric-query.ts:67-72 | experiment-fact-metrics-query.ts:51-56 | `processDimensions()` |
| Exposure Query 获取 | experiment-metric-query.ts:74-76 | experiment-fact-metrics-query.ts:73-75 | `getExposureQuery()` |
| Identities CTE 生成 | experiment-metric-query.ts:201-225 | experiment-fact-metrics-query.ts:105-126 | `getIdentitiesCTE()` |
| 实验结束日期计算 | experiment-metric-query.ts:228-235 | experiment-fact-metrics-query.ts:129 | `getExperimentEndDate()` |
| 维度列生成 | experiment-metric-query.ts:237-248 | experiment-fact-metrics-query.ts:135-146 | `getDimensionCol()` |
| 激活用户过滤逻辑 | experiment-metric-query.ts:250-266 | experiment-fact-metrics-query.ts:148-166 | 相同逻辑 |

**示例：Identities CTE 生成（两条路径代码几乎完全相同）**

```typescript
// Legacy 路径 (experiment-metric-query.ts:215-225)
const { baseIdType, idJoinMap, idJoinSQL } = getIdentitiesCTE(
  dialect,
  datasource.settings,
  {
    objects: idTypeObjects,
    from: settings.startDate,
    to: settings.endDate,
    forcedBaseIdType: userIdType,
    experimentId: settings.experimentId,
  },
);

// Fact 路径 (experiment-fact-metrics-query.ts:116-126)
const { baseIdType, idJoinMap, idJoinSQL } = getIdentitiesCTE(
  dialect,
  datasource.settings,
  {
    objects: idTypeObjects,
    from: settings.startDate,
    to: settings.endDate,
    forcedBaseIdType: userIdType,
    experimentId: settings.experimentId,
  },
);
```

---

## 2. 分叉点：从共享到各自独立

### 2.1 分叉点 1：查询计划层分组

**位置**：`ExperimentResultsQueryRunner.ts:208`

这是两条路径的**第一个显式分叉点**：

```typescript
// ========== 分叉点 ==========
const { factMetricGroups, legacyMetricSingles } = getFactMetricGroups(
  selectedMetrics,
  params.snapshotSettings,
  integration,
  org,
);

// ========== 路径 A：Legacy Metric ==========
for (const m of legacyMetricSingles) {
  // ... 每个指标单独处理
  const queryParams: ExperimentMetricQueryParams = { metric: m, ... };
  queries.push(await startQuery({
    name: m.id,    // key = metric ID
    query: integration.getExperimentMetricQuery(queryParams),
    run: (query) => integration.runExperimentMetricQuery(query),
    queryType: "experimentMetric",
  }));
}

// ========== 路径 B：Fact Metric ==========
for (const [i, m] of factMetricGroups.entries()) {
  // ... 按组批量处理
  const queryParams: ExperimentFactMetricsQueryParams = { metrics: m, ... };
  queries.push(await startQuery({
    name: `group_${i}`,  // key = group_前缀
    query: integration.getExperimentFactMetricsQuery(queryParams),
    run: (query) => integration.runExperimentFactMetricsQuery(query),
    queryType: "experimentMultiMetric",
  }));
}
```

**分叉标记**：
- Legacy 查询 key = `metric.id`（如 `"metric_abc123"`）
- Fact 查询 key = `group_${i}`（如 `"group_0"`、`"group_1"`）

### 2.2 分叉点 2：SQL 生成核心逻辑

在各自的 SQL 生成函数中，经过共享前置步骤后，从**指标数据获取**开始进入各自独立的逻辑：

#### Legacy 路径：`getExperimentMetricQuery()`
```
getExperimentMetricQuery()
├── [共享前置] processActivationMetric()
├── [共享前置] applyMetricOverrides()
├── [共享前置] processDimensions()
├── [共享前置] getExposureQuery()
├── [共享前置] getIdentitiesCTE()
├── [共享前置] getExperimentEndDate()
├── [共享前置] getDimensionCol()
│
├── 【Legacy 独有】单指标特有处理
│   ├── 单个 metric 的类型判断（ratio/funnel/quantile）
│   ├── 单个 metric 的封盖设置处理
│   ├── 单个 metric 的日期范围计算
│   └── 单个 denominator metric 的处理
│
├── 【Legacy 独有】核心 CTE 生成
│   ├── getMetricCTE() → 单指标 CTE（使用 metric.sql）
│   ├── （可选）__denominator CTE → 分母 CTE
│   ├── （可选）__userCovariateMetric CTE → CUPED 协变量
│   ├── （可选）__capValue CTE → 百分位封盖
│   └── __userMetricAgg CTE → 按用户聚合
│
└── 【Legacy 独有】最终统计 SELECT（内联逻辑，无独立函数）
```

#### Fact 路径：`getExperimentFactMetricsQuery()`
```
getExperimentFactMetricsQuery()
├── [共享前置] processActivationMetric()
├── [共享前置] applyMetricOverrides()
├── [共享前置] processDimensions()
├── [共享前置] getExposureQuery()
├── [共享前置] getIdentitiesCTE()
├── [共享前置] getExperimentEndDate()
├── [共享前置] getDimensionCol()
│
├── 【Fact 独有】多指标批量处理
│   ├── getFactTablesForMetrics() → 收集涉及的所有 Fact Table
│   ├── getMetricData() → 解析每个指标的聚合元数据
│   ├── 批量计算所有指标的日期范围（取并集）
│   └── 批量处理所有指标的封盖设置
│
├── 【Fact 独有】核心 CTE 生成
│   ├── getFactMetricCTE() × N → 每个 Fact Table 一个 CTE（使用 factTable.sql）
│   ├── （可选）Quantile 相关 CTE → KLL sketch 处理
│   ├── __userMetricJoin CTE → 多 Fact Table JOIN
│   └── __userMetricAgg CTE → 按用户批量聚合
│
└── 【Fact 独有】批量统计 CTE
    └── getExperimentFactMetricStatisticsCTE() → 一次性计算所有指标的统计值
```

### 2.3 关键分叉函数对比

| 方面 | Legacy Metric | Fact Metric |
|------|--------------|-------------|
| **指标 SQL 源** | `metric.sql`（用户编写） | `factTable.sql`（预定义） |
| **CTE 生成函数** | `getMetricCTE()`（单指标） | `getFactMetricCTE()`（多指标批量） |
| **SQL 结构** | `__metric` → `__userMetricJoin` → `__userMetricAgg` → SELECT | `__factTableX` × N → `__userMetricJoin` → `__userMetricAgg` → `__statistics` |
| **统计逻辑** | 内联在主函数中 | `getExperimentFactMetricStatisticsCTE()` 独立函数 |
| **列解析** | `getMetricColumns()` | `getFactMetricColumn()` |
| **行数** | 1 指标 = 1 个 SQL 查询 | N 指标（同 Fact Table）= 1 个 SQL 查询 |
| **行格式** | 单行单指标，无前缀 | 单行多指标，`m{i}_` 前缀 |

---

## 3. 路径 A：Legacy Metric SQL 生成流程

### 3.1 入口函数：`getExperimentMetricQuery()`

**文件**：`packages/back-end/src/integrations/sql/queries/experiment-metric-query.ts:38-615`

为单个 Legacy Metric 生成完整 SQL：

```typescript
export function getExperimentMetricQuery(dialect, datasource, params) {
  const { metric: metricDoc, denominatorMetrics, activationMetric } = params;
  
  // ========== 共享前置 ==========
  const activationMetric = processActivationMetric(activationMetricDoc, settings);
  applyMetricOverrides(metric, settings);
  const { unitDimensions } = processDimensions(dialect, params.dimensions, settings, activationMetric);
  const userIdType = getExposureQuery(datasource, settings.exposureQueryId).userIdType;
  const { baseIdType, idJoinMap, idJoinSQL } = getIdentitiesCTE(...);
  const endDate = getExperimentEndDate(settings, maxHoursToConvert);
  const dimensionCols = params.dimensions.map(d => getDimensionCol(dialect, d));
  
  // ========== Legacy 独有 ==========
  const ratioMetric = isRatioMetric(metric, denominator);
  const regressionAdjusted = settings.regressionAdjustmentEnabled && 
    isRegressionAdjusted(metric, denominator) && !isRatioMetric(metric, denominator);
  
  // 单个 metric 的日期范围
  const metricStart = getMetricStart(settings.startDate, minMetricDelay, regressionAdjustmentHours);
  const metricEnd = getMetricEnd(orderedMetrics, settings.endDate, overrideConversionWindows);
  
  // 封盖设置
  const capCoalesceMetric = capCoalesceValue(dialect, { valueCol: "m.value", metric, ... });
  
  // ========== 核心 CTE 生成 ==========
  return format(`
WITH
  ${idJoinSQL}
  ${getExperimentUnitsQuery(...)}  -- 或 units table 引用
  
  -- 指标 CTE
  , __metric as (${getMetricCTE(dialect, { metric, ... })})
  
  -- （可选）分母 CTE
  ${denominator ? `, __denominator as (${getMetricCTE(dialect, { metric: denominator, ... })})` : ""}
  
  -- （可选）CUPED 协变量
  ${regressionAdjusted ? `, __userCovariateMetric as (SELECT ... FROM __metric WHERE ...)` : ""}
  
  -- （可选）百分位封盖
  ${isPercentileCapped ? `, __capValue AS (${dialect.percentileCapSelectClause(...)})` : ""}
  
  -- 按用户聚合
  , __userMetricAgg AS (
    SELECT d.variation, d.${baseIdType}, ${getAggregateMetricColumnLegacyMetrics(dialect, { metric })} as value
    FROM __distinctUsers d JOIN __metric m ON m.${baseIdType} = d.${baseIdType}
    WHERE m.timestamp >= d.first_exposure_timestamp
    GROUP BY d.variation, d.${baseIdType}
  )
  
  -- 最终统计
  SELECT
    m.variation AS variation,
    COUNT(*) AS users,
    SUM(${capCoalesceMetric}) AS main_sum,
    SUM(POWER(${capCoalesceMetric}, 2)) AS main_sum_squares,
    ${ratioMetric ? `, SUM(${capCoalesceDenominator}) AS denominator_sum, ...` : ""}
    ${regressionAdjusted ? `, SUM(${capCoalesceCovariate}) AS covariate_sum, ...` : ""}
  FROM __userMetricAgg m
  ${ratioMetric ? "LEFT JOIN __userDenominatorAgg d ON ..." : ""}
  GROUP BY m.variation
`);
}
```

### 3.2 输出行格式

```typescript
{
  variation: string;
  users: number;
  count: number;
  main_sum: number;           // 无前缀
  main_sum_squares: number;   // 无前缀
  denominator_sum?: number;   // 无前缀
  covariate_sum?: number;     // 无前缀
  // ... 其他字段
}
```

### 3.3 结果处理：`runExperimentMetricQuery()`

**文件**：`SqlIntegration.ts:511-619`

```typescript
async runExperimentMetricQuery(query, setExternalId, queryMetadata) {
  const { rows, statistics } = await this.runQuery(query, setExternalId, queryMetadata);
  return {
    rows: rows.map(row => ({
      variation: row.variation ?? "",
      users: parseIntWithDefault(row.users, 0),
      main_sum: parseFloat(row.main_sum as string) || 0,
      main_sum_squares: parseFloat(row.main_sum_squares as string) || 0,
      // ... 直接字段映射
    })),
    statistics,
  };
}
```

---

## 4. 路径 B：Fact Metric SQL 生成流程

### 4.1 入口函数：`getExperimentFactMetricsQuery()`

**文件**：`packages/back-end/src/integrations/sql/queries/experiment-fact-metrics-query.ts:32-660`

为一组 Fact Metric 生成批量优化的 SQL：

```typescript
export function getExperimentFactMetricsQuery(dialect, datasource, params) {
  const { metrics, activationMetric, settings } = params;
  const metricsWithIndices = metrics.map((m, i) => ({ metric: m, index: i }));
  
  // ========== 共享前置 ==========
  const activationMetric = processActivationMetric(params.activationMetric, settings);
  metricsWithIndices.forEach(m => applyMetricOverrides(m.metric, settings));
  const { unitDimensions } = processDimensions(dialect, params.dimensions, settings, activationMetric);
  const userIdType = getExposureQuery(datasource, settings.exposureQueryId).userIdType;
  
  // ========== Fact 独有：批量处理 ==========
  const factTablesWithIndices = getFactTablesForMetrics(metricsWithIndices, factTableMap);
  const metricData = metricsWithIndices.map(m => 
    getMetricData(dialect, m, settings, activationMetric, factTablesWithIndices, ...)
  );
  
  // 批量计算日期范围（取所有指标的并集）
  const metricStart = metricData.reduce((min, d) => d.metricStart < min ? d.metricStart : min, settings.startDate);
  const metricEnd = metricData.reduce((max, d) => d.metricEnd && d.metricEnd > max ? d.metricEnd : max, settings.endDate);
  
  // ========== 共享前置 ==========
  const { baseIdType, idJoinMap, idJoinSQL } = getIdentitiesCTE(...);
  const endDate = getExperimentEndDate(settings, maxHoursToConvert);
  const dimensionCols = params.dimensions.map(d => getDimensionCol(dialect, d));
  
  // ========== Fact 独有：核心 CTE 生成 ==========
  
  // 1. 为每个 Fact Table 生成 CTE
  const factTableCTEs = factTablesWithIndices.map((f, sourceIndex) => {
    return getFactMetricCTE(dialect, {
      metricsWithIndices: metricsWithIndices.filter(m => 
        m.metric.numerator?.factTableId === f.factTable.id ||
        (isRatioMetric(m.metric) && m.metric.denominator?.factTableId === f.factTable.id)
      ),
      factTable: f.factTable,
      baseIdType, idJoinMap, startDate: metricStart, endDate: metricEnd, ...
    });
  });
  
  // 2. 生成用户-指标 JOIN CTE
  const userMetricJoinCTE = `
    , __userMetricJoin AS (
      SELECT
        d.variation, d.${baseIdType}, d.first_exposure_timestamp, ...
        ${metricData.map(m => `, f${m.numeratorSourceIndex}.m${m.metricIndex}_value AS ${m.alias}_value`).join("")}
      FROM __distinctUsers d
      ${metricData.map(m => 
        `LEFT JOIN __factTable${m.numeratorSourceIndex} f${m.numeratorSourceIndex} ON ...`
      ).join("")}
      WHERE ...
    )
  `;
  
  // 3. 生成用户聚合 CTE
  const userMetricAggCTE = `
    , __userMetricAgg AS (
      SELECT
        variation, ${baseIdType},
        ${metricData.map(m => `${m.numeratorAggFns.reAggregationFunction(m.alias + "_value")} AS ${m.alias}_value`).join(",")}
      FROM __userMetricJoin
      GROUP BY variation, ${baseIdType}
    )
  `;
  
  // 4. 生成批量统计 CTE
  const statisticsCTE = getExperimentFactMetricStatisticsCTE(dialect, {
    metricData, dimensionCols, banditDates, ...
  });
  
  // ========== 最终 SQL 组装 ==========
  return format(`
WITH
  ${idJoinSQL}
  ${getExperimentUnitsQuery(...)}
  ${factTableCTEs.join(",")}
  ${userMetricJoinCTE}
  ${userMetricAggCTE}
  ${statisticsCTE}
SELECT * FROM __statistics
`);
}
```

### 4.2 输出行格式

```typescript
{
  variation: string;
  users: number;
  count: number;
  // 多指标前缀字段：
  m0_id: string;
  m0_main_sum: number;
  m0_main_sum_squares: number;
  m1_id: string;
  m1_main_sum: number;
  m1_main_sum_squares: number;
  // ... 更多指标
}
```

### 4.3 结果处理：`processExperimentFactMetricsQueryRows()`

**文件**：`process-experiment-fact-metrics-query-rows.ts:11-51`

```typescript
export function processExperimentFactMetricsQueryRows(rows) {
  return rows.map(row => {
    let metricData = {};
    // 遍历所有可能的指标槽位
    for (let i = 0; i < MAX_METRICS_PER_QUERY; i++) {
      const prefix = `m${i}_`;
      if (!row[prefix + "id"]) break;
      
      metricData[prefix + "id"] = row[prefix + "id"];
      ALL_NON_QUANTILE_METRIC_FLOAT_COLS.forEach(col => {
        if (row[prefix + col] !== undefined) {
          metricData[prefix + col] = parseFloat(row[prefix + col]) || 0;
        }
      });
    }
    return {
      variation: row.variation ?? "",
      users: parseIntWithDefault(row.users, 0),
      count: parseIntWithDefault(row.users, 0),
      ...metricData,
    };
  });
}
```

---

## 5. 汇合点：统计引擎层的统一处理

### 5.1 第一汇合点：`getMetricsAndQueryDataForStatsEngine()`

**文件**：`packages/back-end/src/services/stats.ts:345-455`

这是两条路径的**核心汇合点**。通过查询 key 的命名约定识别来源，统一转换格式：

```typescript
export function getMetricsAndQueryDataForStatsEngine(queryData, metricMap, settings) {
  const queryResults: QueryResultsForStatsEngine[] = [];
  const metricSettings: Record<string, MetricSettingsForStatsEngine> = {};

  queryData.forEach((query, key) => {
    
    // ========== Fact Metric 查询识别 ==========
    if (key.match(/group_/) || query.queryType === "experimentIncrementalRefreshStatistics") {
      const rows = query.result as ExperimentFactMetricsQueryResponseRows;
      if (!rows?.length) return;
      
      const metricIds: (string | null)[] = [];
      // 从行前缀中提取所有指标 ID
      for (let i = 0; i < MAX_METRICS_PER_QUERY; i++) {
        const prefix = `m${i}_`;
        if (!rows[0]?.[prefix + "id"]) break;
        
        const metricId = rows[0][prefix + "id"] as string;
        const metric = metricMap.get(metricId);
        if (metric) {
          metricIds.push(metricId);
          metricSettings[metricId] = getMetricSettingsForStatsEngine(
            metric, metricMap, settings, true  // true = optimizedFactMetric
          );
        } else {
          metricIds.push(null);
        }
      }
      
      queryResults.push({
        metrics: metricIds,  // 该行包含的所有指标 ID
        rows: rows,          // 带前缀的原始行数据
        sql: query.query,
      });
      return;
    }

    // ========== Legacy Metric 查询识别 ==========
    const metric = metricMap.get(key);
    if (!metric) return;
    metricSettings[key] = getMetricSettingsForStatsEngine(
      metric, metricMap, settings, false  // false = 非优化模式
    );
    queryResults.push({
      metrics: [key],    // 该行只包含一个指标
      rows: (query.result ?? []) as ExperimentMetricQueryResponseRows,
      sql: query.query,
    });
  });

  return { queryResults, metricSettings, unknownVariations };
}
```

### 5.2 统一输出格式：`QueryResultsForStatsEngine`

**文件**：`packages/shared/types/stats.d.ts:252-258`

```typescript
export interface QueryResultsForStatsEngine {
  rows: ExperimentMetricQueryResponseRows | ExperimentFactMetricsQueryResponseRows;
  metrics: (string | null)[];  // 该行数据对应的指标 ID 列表
  sql?: string;
}
```

**关键统一机制**：
- `metrics` 数组是自描述的，明确指示当前行数据包含哪些指标
- Legacy：`metrics` 是单元素数组（如 `["metric_123"]`）
- Fact：`metrics` 是多元素数组（如 `["metric_123", "metric_456", null]`）
- 统计引擎内部根据 `metrics` 数组和行数据的前缀/字段名对应关系，正确提取每个指标的统计值

### 5.3 第二汇合点：`runStatsEngine()`

**文件**：`packages/back-end/src/services/stats.ts:140-176`

统一调用 Python 统计引擎（gbstats），**完全不区分指标来源**：

```typescript
export async function runStatsEngine(statsData) {
  if (process.env.EXTERNAL_PYTHON_SERVER_URL) {
    const retVal = await fetch(`${process.env.EXTERNAL_PYTHON_SERVER_URL}/stats`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(statsData),
    });
    const { results } = await retVal.json();
    return results;
  } else {
    const server = await statsServerPool.acquire();
    try {
      return await server.call(statsData);
    } finally {
      statsServerPool.release(server);
    }
  }
}
```

### 5.4 第三汇合点：`parseStatsEngineResult()`

**文件**：`packages/back-end/src/services/stats.ts:464-588`

统计引擎返回的结果格式**完全统一**，不区分指标来源：

```typescript
function parseStatsEngineResult({ analysisSettings, snapshotSettings, queryResults, unknownVariations, result }) {
  const experimentReportResults: ExperimentReportResults[] = [];
  
  analysisSettings.forEach((_, i) => {
    const dimensionMap: Map<string, ExperimentReportResultDimension> = new Map();
    
    // result 是所有指标的分析结果数组，按 metric ID 索引
    result.forEach(({ metric, analyses }) => {
      const analysisResult = analyses[i];
      if (!analysisResult) return;
      
      // 按维度聚合结果
      analysisResult.dimensions.forEach(row => {
        const dim = dimensionMap.get(row.dimension) || { name: row.dimension, srm: 1, variations: [] };
        row.variations.forEach((v, vi) => {
          const data = dim.variations[vi] || { users: v.users, metrics: {} };
          data.users = Math.max(data.users, v.users);
          data.metrics[metric] = { ...v, buckets: [] };  // 按 metric ID 存储结果
          dim.variations[vi] = data;
        });
        dimensionMap.set(row.dimension, dim);
      });
    });
    
    // 计算 SRM
    const dimensions = Array.from(dimensionMap.values());
    dimensions.forEach(dimension => {
      dimension.srm = checkSrm(
        dimension.variations.map(v => v.users),
        snapshotSettings.variations.map(v => v.weight),
      );
    });
    
    experimentReportResults.push({ multipleExposures, unknownVariations, dimensions });
  });
  
  return experimentReportResults;
}
```

---

## 6. 完整流程总览

### 6.1 共享-分叉-汇合全景图

```
              startExperimentResultQueries()
                      │
              ┌───────▼───────┐
              │  共享前置步骤  │
              ├───────────────┤
              │ 1. 基础参数解析 │
              │    - selectedMetrics │
              │    - segmentObj │
              │    - exposureQuery │
              │    - snapshotDimensions │
              ├───────────────┤
              │ 2. Units Table  │
              │    - useUnitsTable 判断 │
              │    - （可选）unitQuery 启动 │
              └───────┬───────┘
                      │
              ┌───────▼───────┐ 【分叉点】
              │ getFactMetricGroups() │
              │  - legacyMetricSingles │
              │  - factMetricGroups │
              └───┬───────────┬───┘
                  │           │
    ┌─────────────▼─┐       ┌─▼─────────────┐
    │ Legacy 循环   │       │ Fact 循环      │
    │ 每个指标      │       │ 每个组         │
    └───────┬───────┘       └───────┬───────┘
            │                       │
┌───────────▼──────────┐  ┌────────▼────────────┐
│ getExperimentMetricQuery() │ │ getExperimentFactMetricsQuery() │
├──────────────────────┤  ├─────────────────────┤
│ [共享前置]           │  │ [共享前置]          │
│ - processActivationMetric │ │ - processActivationMetric │
│ - applyMetricOverrides     │ │ - applyMetricOverrides     │
│ - processDimensions        │ │ - processDimensions        │
│ - getExposureQuery         │ │ - getExposureQuery         │
│ - getIdentitiesCTE         │ │ - getIdentitiesCTE         │
│ - getExperimentEndDate     │ │ - getExperimentEndDate     │
│ - getDimensionCol          │ │ - getDimensionCol          │
├──────────────────────┤  ├─────────────────────┤
│ [Legacy 独有]        │  │ [Fact 独有]         │
│ - 单指标类型判断     │  │ - getFactTablesForMetrics() │
│ - 单指标封盖设置     │  │ - getMetricData()           │
│ - 单指标日期计算     │  │ - 批量日期计算              │
│ - getMetricCTE()    │  │ - getFactMetricCTE() × N    │
│ - 内联统计 SELECT   │  │ - getExperimentFactMetricStatisticsCTE() │
└───────────┬──────────┘  └──────────┬────────────┘
            │                         │
┌───────────▼──────────┐  ┌──────────▼────────────┐
│ runExperimentMetricQuery() │ │ runExperimentFactMetricsQuery() │
│ - 直接字段映射        │  │ - 前缀解析 + ID 提取  │
└───────────┬──────────┘  └──────────┬────────────┘
            │                         │
            └─────────────┬───────────┘
                          │
              ┌───────────▼───────────┐ 【第一汇合点】
              │ getMetricsAndQueryDataForStatsEngine() │
              │ - 通过 key 前缀识别来源            │
              │ - 统一为 QueryResultsForStatsEngine │
              └───────────┬───────────┘
                          │
              ┌───────────▼───────────┐ 【第二汇合点】
              │   runStatsEngine()     │
              │ - Python gbstats 统一分析 │
              └───────────┬───────────┘
                          │
              ┌───────────▼───────────┐ 【第三汇合点】
              │ parseStatsEngineResult() │
              │ - 结果格式完全统一        │
              │ - 按 metric ID 聚合       │
              └───────────┬───────────┘
                          ▼
              ExperimentReportResults
```

### 6.2 关键节点时间线

| 顺序 | 阶段 | 节点 | 共享/分叉/汇合 |
|------|------|------|----------------|
| 1 | 前置 | 基础参数解析（selectedMetrics、segment、exposureQuery、dimensions） | 共享 |
| 2 | 前置 | Units Table 准备（可选） | 共享 |
| 3 | 分叉 | `getFactMetricGroups()` 分组 | 分叉点 |
| 4 | SQL 生成 | `processActivationMetric()` | 共享（各自函数内） |
| 5 | SQL 生成 | `applyMetricOverrides()` | 共享（各自函数内） |
| 6 | SQL 生成 | `processDimensions()` | 共享（各自函数内） |
| 7 | SQL 生成 | `getExposureQuery()` | 共享（各自函数内） |
| 8 | SQL 生成 | `getIdentitiesCTE()` | 共享（各自函数内） |
| 9 | SQL 生成 | `getExperimentEndDate()` | 共享（各自函数内） |
| 10 | SQL 生成 | `getDimensionCol()` | 共享（各自函数内） |
| 11 | SQL 生成 | Legacy: 单指标特有处理 / Fact: 批量指标处理 | 分叉 |
| 12 | SQL 生成 | Legacy: `getMetricCTE()` / Fact: `getFactMetricCTE()` | 分叉 |
| 13 | SQL 生成 | Legacy: 内联统计 SELECT / Fact: `getExperimentFactMetricStatisticsCTE()` | 分叉 |
| 14 | 执行 | `runExperimentMetricQuery()` / `runExperimentFactMetricsQuery()` | 分叉 |
| 15 | 汇合 | `getMetricsAndQueryDataForStatsEngine()` | 第一汇合点 |
| 16 | 汇合 | `runStatsEngine()` | 第二汇合点 |
| 17 | 汇合 | `parseStatsEngineResult()` | 第三汇合点 |

---

## 7. 关键设计要点

### 7.1 共享步骤的设计价值

1. **代码复用**：Activation Metric、Identities CTE、维度处理等复杂逻辑在两条路径间完全复用
2. **一致性保证**：共享步骤确保两条路径在用户识别、维度解析、日期计算等基础逻辑上完全一致
3. **维护成本**：共享逻辑只需维护一份，修改时无需担心两条路径不一致
4. **平滑迁移**：新功能（如 CUPED、Bandit）在共享层实现后，两条路径可同时受益

### 7.2 分叉设计的权衡

| 设计选择 | 优点 | 缺点 |
|---------|------|------|
| Legacy 路径单指标查询 | 简单、兼容、易于调试 | 查询数量多、表扫描重复 |
| Fact 路径批量查询 | 共享表扫描、列裁剪、谓词下推，性能更优 | 实现复杂、SQL 体积大 |

### 7.3 汇合设计的精妙之处

1. **Key 约定**：通过查询 key 的命名约定（`metric.id` vs `group_${i}`）实现来源识别，无需额外元数据
2. **自描述数据**：`metrics` 数组使 `QueryResultsForStatsEngine` 自包含，统计引擎无需关心数据来源
3. **前缀打包**：Fact Metric 使用 `m{i}_` 前缀在单行中打包多个指标，既高效又保持结构清晰
4. **元数据分离**：`metricSettings` 单独提供所有指标的统计配置，与行数据解耦
5. **三层汇合**：数据格式 → 统计分析 → 结果解析，每层都实现完全解耦

### 7.4 两条路径的详细对比

| 维度 | SQL 直接定义 (Legacy) | Fact Table |
|------|----------------------|------------|
| 分叉入口 | `legacyMetricSingles` 循环 | `factMetricGroups` 循环 |
| SQL 生成函数 | `getExperimentMetricQuery()` | `getExperimentFactMetricsQuery()` |
| 共享前置步骤 | ✅ 全部共享 | ✅ 全部共享 |
| 指标 SQL 源 | 用户编写的 `metric.sql` | 预定义 `factTable.sql` |
| 核心 CTE 函数 | `getMetricCTE()`（单指标） | `getFactMetricCTE()`（多指标批量） |
| 统计逻辑 | 内联在主函数中 | `getExperimentFactMetricStatisticsCTE()` 独立函数 |
| 查询 key | `metric.id` | `group_${i}` |
| 行格式 | 单行单指标，无前缀 | 单行多指标，`m{i}_` 前缀 |
| 结果处理 | 直接字段映射 | 前缀解析 + 指标 ID 提取 |
| 汇合识别 | key 就是 metric ID | key 匹配 `group_` 前缀 |
| 统计引擎输入 | `metrics: [singleId]` | `metrics: [id1, id2, ...]` |
| 优化能力 | 有限（单指标优化） | 丰富（批量扫描、列裁剪、谓词下推） |
| 复用性 | 低（SQL 散落各处） | 高（Fact Table 可被多个指标复用） |

---

## 8. 核心文件索引

| 文件 | 职责 |
|------|------|
| `packages/shared/types/metric.d.ts` | Legacy Metric 类型定义 |
| `packages/shared/types/fact-table.d.ts` | Fact Table / Fact Metric 类型定义 |
| `packages/shared/types/integrations.d.ts` | 查询结果行类型定义 |
| `packages/shared/types/stats.d.ts` | 统计引擎输入输出类型定义 |
| `packages/shared/src/experiments/experiments.ts` | 类型守卫、通用工具函数 |
| `packages/back-end/src/services/experimentQueries/experimentQueries.ts` | 分叉点 `getFactMetricGroups()` |
| `packages/back-end/src/queryRunners/ExperimentResultsQueryRunner.ts` | 查询计划生成、共享前置、分叉执行 |
| `packages/back-end/src/integrations/sql/queries/experiment-metric-query.ts` | Legacy Metric SQL 生成 |
| `packages/back-end/src/integrations/sql/queries/experiment-fact-metrics-query.ts` | Fact Metric SQL 生成 |
| `packages/back-end/src/integrations/sql/ctes/metric-cte.ts` | Legacy 单指标 CTE 生成 |
| `packages/back-end/src/integrations/sql/ctes/fact-metric-cte.ts` | Fact 多指标批量 CTE 生成 |
| `packages/back-end/src/integrations/sql/ctes/experiment-fact-metric-statistics-cte.ts` | Fact 批量统计 CTE |
| `packages/back-end/src/integrations/sql/ctes/identities-cte.ts` | 共享 Identities CTE 生成 |
| `packages/back-end/src/integrations/sql/columns/metric-columns.ts` | Legacy 列解析 `getMetricColumns()` |
| `packages/back-end/src/integrations/sql/columns/fact-metric-column.ts` | Fact 列解析 `getFactMetricColumn()` |
| `packages/back-end/src/integrations/sql/processing/process-activation-metric.ts` | 共享 Activation Metric 处理 |
| `packages/back-end/src/integrations/sql/processing/process-experiment-fact-metrics-query-rows.ts` | Fact 行前缀解析 |
| `packages/back-end/src/integrations/SqlIntegration.ts` | SQL 执行与结果处理 |
| `packages/back-end/src/services/stats.ts` | 汇合点 `getMetricsAndQueryDataForStatsEngine()`、`runStatsEngine()`、`parseStatsEngineResult()` |
