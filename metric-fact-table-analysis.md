# 指标定义双路径衔接分析

## 概述

GrowthBook 中指标定义支持两条独立路径：
1. **SQL 直接定义路径**（Legacy Metric）：用户直接编写 SQL 定义指标
2. **Fact Table 路径**（Fact Metric）：指向预定义的 Fact Table，通过列引用和聚合方式定义指标

两条路径在查询时经过**明确的分叉点**进入各自的 SQL 生成流程，最终在**统计引擎层**实现完全汇合。

---

## 1. 类型系统与分叉前准备

### 1.1 核心类型定义

**共享类型联合** (`packages/shared/src/experiments/experiments.ts:54`)：
```typescript
export type ExperimentMetricInterface = MetricInterface | FactMetricInterface;
```

- `MetricInterface`（`packages/shared/types/metric.d.ts:43`）：传统 SQL 指标
  - 核心字段：`sql`（用户编写的 SQL）、`queryFormat`（"sql" 或 "builder"）、`type`（binomial/count/duration/revenue）
  - 可选字段：`table`、`column`、`conditions`（Query Builder 模式）

- `FactMetricInterface`（`packages/shared/types/fact-table.d.ts:106`）：Fact Table 指标
  - 核心字段：`metricType`（mean/proportion/ratio/quantile 等）、`numerator`（分子列引用）、`denominator`（分母列引用）
  - 列引用 `ColumnRef`：包含 `factTableId`、`column`、`aggregation`、`filters` 等

### 1.2 类型判断函数

**类型守卫** (`packages/shared/src/experiments/experiments.ts:83-94`)：
```typescript
export function isFactMetric(m: ExperimentMetricInterface): m is FactMetricInterface {
  return "metricType" in m;  // Fact Metric 独有字段
}

export function isLegacyMetric(m: ExperimentMetricInterface): m is MetricInterface {
  return !isFactMetric(m);
}
```

---

## 2. 分叉点：查询计划层的路径选择

### 2.1 分叉核心函数：`getFactMetricGroups()`

**文件**：`packages/back-end/src/services/experimentQueries/experimentQueries.ts:256-351`

这是两条路径的**正式分叉点**。该函数接收所有指标，按类型和优化可能性分组：

```typescript
export function getFactMetricGroups(metrics, settings, integration, organization): GroupedMetrics {
  // 第一步：按类型彻底分流
  const legacyMetrics: MetricInterface[] = metrics.filter(isLegacyMetric);
  const factMetrics: FactMetricInterface[] = metrics.filter(isFactMetric);
  
  // 第二步：Fact Metric 分组优化（企业版功能）
  // 分组策略：
  // 1. 共享同一 Fact Table 的指标可以合并查询
  // 2. Ratio 指标需要分子分母在同一 Fact Table
  // 3. Quantile 指标单独分组（避免拖慢主查询）
  // 4. 按数据源能力（列数限制、百分位支持）分批
  
  return {
    factMetricGroups: FactMetricInterface[][],  // 可合并的 Fact Metric 组
    legacyMetricSingles: MetricInterface[]      // 传统 SQL 指标（单独查询）
  };
}
```

### 2.2 分组键生成：`getFactMetricGroup()`

**文件**：`packages/back-end/src/services/experimentQueries/experimentQueries.ts:224-247`

```typescript
export function getFactMetricGroup(metric: FactMetricInterface) {
  // Ratio 指标跨表时单独分组
  if (isRatioMetric(metric) && metric.numerator.factTableId !== metric.denominator?.factTableId) {
    return `${tableIds[0]} ${tableIds[1]} (cross-table ratio metrics)`;
  }
  // Quantile 指标单独分组
  if (quantileMetricType(metric)) {
    return `${metric.numerator.factTableId}_qtile`;
  }
  // 默认按 Fact Table ID 分组
  return metric.numerator.factTableId || "";
}
```

### 2.3 分叉执行：`startExperimentResultQueries()`

**文件**：`packages/back-end/src/queryRunners/ExperimentResultsQueryRunner.ts:208-297`

从这里开始，两条路径进入**完全独立**的 SQL 生成和执行流程：

```typescript
// ========== 分叉点 ==========
const { factMetricGroups, legacyMetricSingles } = getFactMetricGroups(...);

// ========== 路径 A：Legacy Metric ==========
// 每个指标单独生成查询、单独执行
for (const m of legacyMetricSingles) {
  const queryParams: ExperimentMetricQueryParams = { metric: m, ... };
  queries.push(await startQuery({
    name: m.id,    // key = metric ID
    query: integration.getExperimentMetricQuery(queryParams),  // 单指标 SQL 生成
    run: (query) => integration.runExperimentMetricQuery(query),
    queryType: "experimentMetric",
  }));
}

// ========== 路径 B：Fact Metric ==========
// 按组批量生成查询、批量执行
for (const [i, m] of factMetricGroups.entries()) {
  const queryParams: ExperimentFactMetricsQueryParams = { metrics: m, ... };
  queries.push(await startQuery({
    name: `group_${i}`,  // key = group_前缀
    query: integration.getExperimentFactMetricsQuery(queryParams),  // 多指标 SQL 生成
    run: (query) => integration.runExperimentFactMetricsQuery(query),
    queryType: "experimentMultiMetric",
  }));
}
```

**关键分叉标记**：
- Legacy 路径查询的 key = `metric.id`（如 `"metric_abc123"`）
- Fact 路径查询的 key = `group_${i}`（如 `"group_0"`、`"group_1"`）
- 这个 key 约定在后续汇合点用于识别查询来源

---

## 3. 路径 A：Legacy Metric SQL 生成流程

### 3.1 入口函数：`getExperimentMetricQuery()`

**文件**：`packages/back-end/src/integrations/sql/queries/experiment-metric-query.ts:38-615`

为单个 Legacy Metric 生成完整 SQL，流程如下：

```
getExperimentMetricQuery()
├── getExposureQuery()          # 获取曝光数据
├── getIdentitiesCTE()          # 用户身份映射
├── getMetricCTE()              # 指标 CTE（核心）
│   └── getMetricColumns()      # 解析指标列
├── （可选）__denominator CTE   # Ratio 指标的分母 CTE
├── （可选）__userCovariateMetric CTE  # CUPED 协变量
├── （可选）__capValue CTE      # 百分位封盖
├── __userMetricAgg CTE         # 按用户聚合
└── 最终统计 SELECT             # 内联统计逻辑，无独立函数
```

### 3.2 核心 CTE：`getMetricCTE()`

**文件**：`packages/back-end/src/integrations/sql/ctes/metric-cte.ts:16-170`

> ⚠️ 注意：虽然这个函数内部通过 `isFactMetric()` 判断支持两种类型，但在 Legacy 路径中只用于处理 Legacy Metric。

```typescript
export function getMetricCTE(dialect, { metric, ... }) {
  const cols = getMetricColumns(dialect, metric, factTableMap, "m", useDenominator);
  
  const isFact = isFactMetric(metric);
  const queryFormat = isFact ? "fact" : getMetricQueryFormat(metric);
  
  let sql = "";
  if (isFact && factTable && columnRef) {
    sql = factTable.sql;  // Fact Table SQL
  } else if (!isFact && queryFormat === "sql") {
    sql = metric.sql || "";  // Legacy: 用户编写的 SQL
  }
  // Query Builder 模式：直接使用 metric.table
  
  return compileSqlTemplate(`
    SELECT ${userIdCol} as ${baseIdType}, ${cols.value} as value, ...
    FROM (${sql}) m
    ...
  `);
}
```

### 3.3 输出行格式

**文件**：`packages/shared/types/integrations.d.ts:606-632`

```typescript
export type ExperimentMetricQueryResponseRows = {
  variation: string;
  users: number;
  count: number;
  main_sum: number;           // 无前缀
  main_sum_squares: number;   // 无前缀
  denominator_sum?: number;   // 无前缀
  covariate_sum?: number;     // 无前缀
  // ... 其他字段
}[];
```

**特点**：每行只包含**一个指标**的数据，字段名直接使用 `main_sum`、`denominator_sum` 等，无前缀。

### 3.4 结果处理：`runExperimentMetricQuery()`

**文件**：`packages/back-end/src/integrations/SqlIntegration.ts:511-619`

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

为一组 Fact Metric 生成批量优化的 SQL，流程如下：

```
getExperimentFactMetricsQuery()
├── getFactTablesForMetrics()       # 收集涉及的所有 Fact Table
├── getMetricData()                 # 解析每个指标的聚合元数据
├── getExposureQuery()              # 获取曝光数据
├── getIdentitiesCTE()              # 用户身份映射
├── getFactMetricCTE() × N          # 每个 Fact Table 一个 CTE
│   └── getFactMetricColumn()       # 解析列引用（支持多指标）
├── （可选）Quantile 相关 CTE       # KLL sketch / 百分位计算
├── __userMetricJoin CTE            # 多 Fact Table JOIN
├── __userMetricAgg CTE             # 按用户批量聚合
└── getExperimentFactMetricStatisticsCTE()  # 批量统计 CTE
```

### 4.2 核心 CTE：`getFactMetricCTE()`

**文件**：`packages/back-end/src/integrations/sql/ctes/fact-metric-cte.ts:20-217`

> ✅ 这是 Fact Metric 专用的 CTE 生成函数，**不被 Legacy 路径使用**。核心优化是在单个 CTE 中投影多个指标的列。

```typescript
export function getFactMetricCTE(dialect, { metricsWithIndices, factTable, ... }) {
  const sql = factTable.sql;  // 使用预定义的 Fact Table SQL
  const where: string[] = [];
  
  // 日期过滤（谓词下推）
  if (startDate) where.push(`m.timestamp >= ${dialect.toTimestamp(startDate)}`);
  if (endDate) where.push(`m.timestamp <= ${dialect.toTimestamp(endDate)}`);
  
  // 为该 Fact Table 上的所有指标投影列
  const metricCols: string[] = [];
  metricsWithIndices.forEach(({ metric, index }) => {
    if (metric.numerator?.factTableId === factTable.id) {
      const value = getFactMetricColumn(dialect, metric, metric.numerator, factTable, "m").value;
      const filters = getColumnRefWhereClause(...);
      const column = filters.length > 0 
        ? `CASE WHEN (${filters.join(" AND ")}) THEN ${value} ELSE NULL END`
        : value;
      // 使用 m{index}_ 前缀区不同指标
      metricCols.push(`${column} as m${index}_value`);
    }
    // Ratio 指标的分母列也投影到同一 CTE
    if (isRatioMetric(metric) && metric.denominator?.factTableId === factTable.id) {
      // ... 类似逻辑，输出 m${index}_denominator
    }
  });
  
  return compileSqlTemplate(`
    SELECT
      ${userIdCol} as ${baseIdType},
      ${timestampDateTimeColumn} as timestamp,
      ${metricCols.join(",\n")}  -- 多指标列，带 m{i}_ 前缀
    FROM (${sql}) m
    ${where.length ? `WHERE ${where.join(" AND ")}` : ""}
  `);
}
```

### 4.3 批量统计 CTE：`getExperimentFactMetricStatisticsCTE()`

**文件**：`packages/back-end/src/integrations/sql/ctes/experiment-fact-metric-statistics-cte.ts`

Fact Metric 专用的批量统计函数，一次性计算所有指标的统计值。

### 4.4 输出行格式

**文件**：`packages/shared/types/integrations.d.ts:633-638`

```typescript
export type ExperimentFactMetricsQueryResponseRows = {
  variation: string;
  users: number;
  count: number;
  // 多指标前缀字段：
  // m0_id, m0_main_sum, m0_main_sum_squares, ...
  // m1_id, m1_main_sum, m1_main_sum_squares, ...
  [key: string]: number | string;
}[];
```

**特点**：每行包含**多个指标**的数据，每个指标的字段带 `m{i}_` 前缀，通过 `m{i}_id` 标识指标 ID。

### 4.5 结果处理：`processExperimentFactMetricsQueryRows()`

**文件**：`packages/back-end/src/integrations/sql/processing/process-experiment-fact-metrics-query-rows.ts:11-51`

```typescript
export function processExperimentFactMetricsQueryRows(rows) {
  return rows.map(row => {
    let metricData = {};
    // 遍历所有可能的指标槽位（最多 MAX_METRICS_PER_QUERY 个）
    for (let i = 0; i < MAX_METRICS_PER_QUERY; i++) {
      const prefix = `m${i}_`;
      if (!row[prefix + "id"]) break;  // 没有更多指标
      
      metricData[prefix + "id"] = row[prefix + "id"];
      // 解析所有浮点字段
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
      ...metricData,  // 展开所有带前缀的指标字段
    };
  });
}
```

---

## 5. 汇合点：统计引擎层的统一处理

### 5.1 第一汇合点：`getMetricsAndQueryDataForStatsEngine()`

**文件**：`packages/back-end/src/services/stats.ts:345-455`

这是两条路径的**核心汇合点**。该函数接收 `QueryMap`（包含所有查询结果），通过查询 key 的命名约定识别来源，统一转换为统计引擎可接受的格式。

```typescript
export function getMetricsAndQueryDataForStatsEngine(
  queryData: QueryMap,
  metricMap: Map<string, ExperimentMetricInterface>,
  settings: ExperimentSnapshotSettings,
) {
  const queryResults: QueryResultsForStatsEngine[] = [];
  const metricSettings: Record<string, MetricSettingsForStatsEngine> = {};

  // 遍历所有查询结果，通过 key 前缀判断来源
  queryData.forEach((query, key) => {
    
    // ========== 识别 Fact Metric 查询 ==========
    // key 匹配 group_ 前缀，或 queryType 为 experimentIncrementalRefreshStatistics
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
          // 为每个指标生成统计引擎配置
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

    // ========== 识别 Legacy Metric 查询 ==========
    // key 就是 metric ID
    const metric = metricMap.get(key);
    if (!metric) return;
    // 为单个指标生成统计引擎配置
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
- `metrics` 字段是自描述的，明确指示当前行数据包含哪些指标
- Legacy Metric：`metrics` 是单元素数组（如 `["metric_123"]`）
- Fact Metric：`metrics` 是多元素数组（如 `["metric_123", "metric_456", null]`）
- 统计引擎内部根据 `metrics` 数组和行数据的前缀/字段名对应关系，正确提取每个指标的统计值

### 5.3 指标元数据统一：`getMetricSettingsForStatsEngine()`

**文件**：`packages/back-end/src/services/stats.ts:267-343`

为两种类型的指标生成统一的统计引擎配置，内部通过类型守卫透明处理：

```typescript
export function getMetricSettingsForStatsEngine(
  metricDoc: ExperimentMetricInterface,  // 联合类型，两种都可能
  metricMap: Map<string, ExperimentMetricInterface>,
  settings: ExperimentSnapshotSettings,
  optimizedFactMetric: boolean = false,
): MetricSettingsForStatsEngine {
  const metric = cloneDeep(metricDoc);
  applyMetricOverrides(metric, settings);

  // 统一判断指标类型（内部使用 isFactMetric 等守卫）
  const ratioMetric = isRatioMetric(metric, denominator);
  const quantileMetric = quantileMetricType(metric);
  const regressionAdjusted = settings.regressionAdjustmentEnabled && 
    isRegressionAdjusted(metric, denominator) &&
    (!isRatioMetric(metric, denominator) || optimizedFactMetric);
  
  const mainMetricType = quantileMetric ? "quantile" :
                         isBinomialMetric(metric) ? "binomial" : "count";

  return {
    id: metric.id,
    name: metric.name,
    inverse: !!metric.inverse,
    statistic_type: quantileMetric === "unit" ? "quantile_unit" :
                    quantileMetric === "event" ? "quantile_event" :
                    ratioMetric && regressionAdjusted ? "ratio_ra" :
                    ratioMetric && !regressionAdjusted ? "ratio" :
                    regressionAdjusted ? "mean_ra" : "mean",
    main_metric_type: mainMetricType,
    // ... 其他配置字段
    compute_uncapped_metric: eligibleForUncappedMetric(metric),
  };
}
```

### 5.4 第二汇合点：`runStatsEngine()`

**文件**：`packages/back-end/src/services/stats.ts:140-176`

统一调用 Python 统计引擎（gbstats），**完全不区分指标来源**：

```typescript
export async function runStatsEngine(
  statsData: ExperimentDataForStatsEngine[],
): Promise<MultipleExperimentMetricAnalysis[]> {
  if (process.env.EXTERNAL_PYTHON_SERVER_URL) {
    // 调用外部统计服务
    const retVal = await fetch(`${process.env.EXTERNAL_PYTHON_SERVER_URL}/stats`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(statsData),
    });
    const { results } = await retVal.json();
    return results;
  } else {
    // 调用本地 Python 进程池
    const server = await statsServerPool.acquire();
    try {
      return await server.call(statsData);
    } finally {
      statsServerPool.release(server);
    }
  }
}
```

**统计引擎输入结构**（`packages/shared/types/stats.d.ts:260-270`）：
```typescript
export interface DataForStatsEngine {
  analyses: AnalysisSettingsForStatsEngine[];
  metrics: Record<string, MetricSettingsForStatsEngine>;  // 所有指标元数据
  query_results: QueryResultsForStatsEngine[];            // 统一格式的查询结果
  bandit_settings?: BanditSettingsForStatsEngine;
}
```

### 5.5 第三汇合点：`parseStatsEngineResult()`

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
    
    // 计算 SRM 等额外信息
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

### 6.1 分叉与汇合全景图

```
              所有指标 (ExperimentMetricInterface[])
                              │
                              ▼
              getFactMetricGroups() 【分叉点】
                              │
           ┌──────────────────┴──────────────────┐
           │                                     │
           ▼                                     ▼
  legacyMetricSingles                    factMetricGroups
           │                                     │
           ▼                                     ▼
  循环每个指标                          循环每个指标组
           │                                     │
           ▼                                     ▼
  getExperimentMetricQuery()           getExperimentFactMetricsQuery()
           │                                     │
           ├─ getMetricCTE()                     ├─ getFactTablesForMetrics()
           │  (单指标CTE)                        ├─ getMetricData()
           │                                     ├─ getFactMetricCTE() × N
           ▼                                     │  (多指标批量CTE)
  runExperimentMetricQuery()                     ├─ 其他辅助CTE
           │                                     ├─ __userMetricJoin
           ▼                                     ├─ __userMetricAgg
  { variation, main_sum, ... }                   ▼
  (单行单指标, 无前缀)                  getExperimentFactMetricStatisticsCTE()
                                                 │
                                                 ▼
                                       runExperimentFactMetricsQuery()
                                                 │
                                                 ▼
                                       { variation, m0_main_sum, m1_main_sum, ... }
                                       (单行多指标, 带 m{i}_ 前缀)
                                                 │
           ┌─────────────────────────────────────┘
           │
           ▼
  getMetricsAndQueryDataForStatsEngine() 【第一汇合点】
           │  - 通过 key 前缀识别来源
           │  - 统一为 QueryResultsForStatsEngine 格式
           │  - metrics 数组标记每行指标
           ▼
  runStatsEngine() 【第二汇合点】
           │  - Python gbstats 统一分析
           │  - 完全不区分来源
           ▼
  parseStatsEngineResult() 【第三汇合点】
           │  - 结果格式完全统一
           │  - 按 metric ID 聚合
           ▼
  ExperimentReportResults
```

### 6.2 关键节点汇总

| 节点 | 函数 | 位置 | 作用 |
|------|------|------|------|
| **分叉点** | `getFactMetricGroups()` | experimentQueries.ts:256 | 将指标分为 Legacy 单查询组和 Fact 批量组 |
| Legacy SQL 入口 | `getExperimentMetricQuery()` | experiment-metric-query.ts:38 | 单指标 SQL 生成 |
| Legacy CTE | `getMetricCTE()` | metric-cte.ts:16 | 单指标 CTE 生成 |
| Legacy 执行 | `runExperimentMetricQuery()` | SqlIntegration.ts:511 | 单指标查询执行与结果解析 |
| Fact SQL 入口 | `getExperimentFactMetricsQuery()` | experiment-fact-metrics-query.ts:32 | 多指标批量 SQL 生成 |
| Fact CTE | `getFactMetricCTE()` | fact-metric-cte.ts:20 | 多指标批量 CTE 生成 |
| Fact 执行 | `runExperimentFactMetricsQuery()` | SqlIntegration.ts:486 | 多指标查询执行与结果解析 |
| **第一汇合点** | `getMetricsAndQueryDataForStatsEngine()` | stats.ts:345 | 统一数据格式，生成 `QueryResultsForStatsEngine` |
| **第二汇合点** | `runStatsEngine()` | stats.ts:140 | 统一调用 Python 统计引擎 |
| **第三汇合点** | `parseStatsEngineResult()` | stats.ts:464 | 统一解析统计结果 |

---

## 7. 关键设计要点

### 7.1 分叉设计的权衡

| 设计选择 | 优点 | 缺点 |
|---------|------|------|
| Legacy 路径单指标查询 | 简单、兼容、易于调试 | 查询数量多、表扫描重复 |
| Fact 路径批量查询 | 共享表扫描、列裁剪、谓词下推，性能更优 | 实现复杂、SQL 体积大 |

### 7.2 汇合设计的精妙之处

1. **Key 约定**：通过查询 key 的命名约定（`metric.id` vs `group_${i}`）实现来源识别，无需额外元数据
2. **自描述数据**：`metrics` 数组使 `QueryResultsForStatsEngine` 自包含，统计引擎无需关心数据来源
3. **前缀打包**：Fact Metric 使用 `m{i}_` 前缀在单行中打包多个指标，既高效又保持结构清晰
4. **元数据分离**：`metricSettings` 单独提供所有指标的统计配置，与行数据解耦
5. **三层汇合**：数据格式 → 统计分析 → 结果解析，每层都实现完全解耦

### 7.3 两条路径的详细对比

| 维度 | SQL 直接定义 (Legacy) | Fact Table |
|------|----------------------|------------|
| 分叉入口 | `legacyMetricSingles` 循环 | `factMetricGroups` 循环 |
| SQL 生成函数 | `getExperimentMetricQuery()` | `getExperimentFactMetricsQuery()` |
| 核心 CTE 函数 | `getMetricCTE()`（单指标） | `getFactMetricCTE()`（多指标批量） |
| SQL 源 | 用户编写的 `metric.sql` | 预定义 `factTable.sql` |
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
| `packages/back-end/src/queryRunners/ExperimentResultsQueryRunner.ts` | 查询计划生成与分叉执行 |
| `packages/back-end/src/integrations/sql/queries/experiment-metric-query.ts` | Legacy Metric SQL 生成 |
| `packages/back-end/src/integrations/sql/queries/experiment-fact-metrics-query.ts` | Fact Metric SQL 生成 |
| `packages/back-end/src/integrations/sql/ctes/metric-cte.ts` | Legacy 单指标 CTE 生成 |
| `packages/back-end/src/integrations/sql/ctes/fact-metric-cte.ts` | Fact 多指标批量 CTE 生成 |
| `packages/back-end/src/integrations/sql/ctes/experiment-fact-metric-statistics-cte.ts` | Fact 批量统计 CTE |
| `packages/back-end/src/integrations/sql/columns/metric-columns.ts` | Legacy 列解析 `getMetricColumns()` |
| `packages/back-end/src/integrations/sql/columns/fact-metric-column.ts` | Fact 列解析 `getFactMetricColumn()` |
| `packages/back-end/src/integrations/sql/processing/process-experiment-fact-metrics-query-rows.ts` | Fact 行前缀解析 |
| `packages/back-end/src/integrations/SqlIntegration.ts` | SQL 执行与结果处理 |
| `packages/back-end/src/services/stats.ts` | 汇合点 `getMetricsAndQueryDataForStatsEngine()`、`runStatsEngine()`、`parseStatsEngineResult()` |
