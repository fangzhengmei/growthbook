# 指标定义双路径衔接分析

## 概述

GrowthBook 中指标定义支持两条独立路径：
1. **SQL 直接定义路径**（Legacy Metric）：用户直接编写 SQL 定义指标
2. **Fact Table 路径**（Fact Metric）：指向预定义的 Fact Table，通过列引用和聚合方式定义指标

两条路径在查询时经过**定义层抽象**、**查询计划生成**、**warehouse 端 SQL 发出**和**统计引擎并轨**四个阶段最终合并，输出统一格式的分析结果。

---

## 1. 定义层抽象：类型系统与判断逻辑

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

### 1.3 定义层汇合点：`getMetricCTE()`

**文件**：`packages/back-end/src/integrations/sql/ctes/metric-cte.ts:16-170`

这是两条路径在 SQL 生成层面的第一个汇合点。该函数接收 `ExperimentMetricInterface` 类型，内部通过类型判断分发到不同的 SQL 源：

```typescript
export function getMetricCTE(dialect, { metric, ... }) {
  // 1. 统一获取列信息（内部已处理两种类型）
  const cols = getMetricColumns(dialect, metric, factTableMap, "m", useDenominator);
  
  // 2. 判断类型
  const isFact = isFactMetric(metric);
  const queryFormat = isFact ? "fact" : getMetricQueryFormat(metric);
  
  // 3. 根据类型获取 SQL 源
  let sql = "";
  if (isFact && factTable && columnRef) {
    // Fact Table 路径：从 fact table 定义中获取 SQL
    sql = factTable.sql;
    // 应用 Fact Table 特有的过滤器
    getColumnRefWhereClause({ factTable, columnRef, ... }).forEach(f => where.push(f));
  } else if (!isFact && queryFormat === "sql") {
    // SQL 直接定义路径：使用用户编写的 SQL
    sql = metric.sql || "";
  }
  // Query Builder 路径（legacy）：直接使用 metric.table
  
  // 4. 统一输出 CTE SQL
  return compileSqlTemplate(`SELECT ${userIdCol} as ${baseIdType}, ${cols.value} as value, ... FROM (${sql}) m ...`);
}
```

### 1.4 列解析抽象：`getMetricColumns()`

**文件**：`packages/back-end/src/integrations/sql/columns/metric-columns.ts:17-102`

统一处理两种类型指标的列解析：
- **Fact Metric**：通过 `factTableId` 查找 Fact Table，解析 `columnRef.column`，支持 JSON 字段提取
- **Legacy Metric (SQL 模式)**：直接使用 `metric.value` 或常量 `1`（binomial 类型）
- **Legacy Metric (Builder 模式)**：使用 `metric.column` 和 `metric.timestampColumn`

---

## 2. 查询计划生成：分组与路由

### 2.1 指标分组逻辑：`getFactMetricGroups()`

**文件**：`packages/back-end/src/services/experimentQueries/experimentQueries.ts:256-351`

查询计划生成的核心分流函数，将指标分为可优化组和单查询组：

```typescript
export function getFactMetricGroups(metrics, settings, integration, organization): GroupedMetrics {
  // 第一步：按类型分流
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

### 2.3 查询执行路由：`startExperimentResultQueries()`

**文件**：`packages/back-end/src/queryRunners/ExperimentResultsQueryRunner.ts:208-297`

根据分组结果生成不同的查询计划：

```typescript
const { factMetricGroups, legacyMetricSingles } = getFactMetricGroups(...);

// 路径 1：Legacy Metric - 每个指标单独查询
for (const m of legacyMetricSingles) {
  const queryParams: ExperimentMetricQueryParams = { metric: m, ... };
  queries.push(await startQuery({
    name: m.id,
    query: integration.getExperimentMetricQuery(queryParams),  // 单指标查询
    run: (query) => integration.runExperimentMetricQuery(query),
    queryType: "experimentMetric",
  }));
}

// 路径 2：Fact Metric - 按组批量查询
for (const [i, m] of factMetricGroups.entries()) {
  const queryParams: ExperimentFactMetricsQueryParams = { metrics: m, ... };
  queries.push(await startQuery({
    name: `group_${i}`,
    query: integration.getExperimentFactMetricsQuery(queryParams),  // 多指标合并查询
    run: (query) => integration.runExperimentFactMetricsQuery(query),
    queryType: "experimentMultiMetric",
  }));
}
```

---

## 3. Warehouse 端 SQL 生成与结果格式

### 3.1 Legacy Metric SQL 生成：`getExperimentMetricQuery()`

**文件**：`packages/back-end/src/integrations/sql/queries/experiment-metric-query.ts:38-615`

单指标 SQL 生成流程：
1. CTE 结构：`__metric` → `__userMetricJoin` → `__userMetricAgg` → 最终聚合
2. 支持 Ratio、Funnel、CUPED、百分位封盖等高级特性
3. 每个指标独立 SQL，互不干扰

**输出行格式**（`packages/shared/types/integrations.d.ts:606-632`）：
```typescript
export type ExperimentMetricQueryResponseRows = {
  variation: string;
  users: number;
  count: number;
  main_cap_value?: number;
  main_sum: number;
  main_sum_squares: number;
  denominator_cap_value?: number;
  denominator_sum?: number;
  denominator_sum_squares?: number;
  main_denominator_sum_product?: number;
  covariate_sum?: number;
  covariate_sum_squares?: number;
  main_covariate_sum_product?: number;
  theta?: number;           // bandits only
  quantile?: number;
  quantile_n?: number;
  quantile_lower?: number;
  quantile_upper?: number;
  // ... 更多 uncapped 字段
  [key: string]: any;
}[];
```

**特点**：每行只有一个指标的数据，字段名直接使用 `main_sum`、`denominator_sum` 等，无前缀。

### 3.2 Fact Metric SQL 生成：`getExperimentFactMetricsQuery()`

**文件**：`packages/back-end/src/integrations/sql/queries/experiment-fact-metrics-query.ts:32-660`

多指标合并 SQL 生成流程：
1. **Fact Table CTE 生成**：`getFactMetricCTE()` 为每个 Fact Table 生成一个 CTE，一次性投影所有相关指标的数值列
2. **多指标 JOIN**：`__userMetricJoin` 同时 JOIN 所有 Fact Table，一次性获取所有指标值
3. **批量聚合**：`__userMetricAgg` 在单个 GROUP BY 中计算所有指标的聚合值
4. **统计 CTE**：`getExperimentFactMetricStatisticsCTE()` 一次性输出所有指标的统计结果

**输出行格式**（`packages/shared/types/integrations.d.ts:633-638`）：
```typescript
export type ExperimentFactMetricsQueryResponseRows = {
  variation: string;
  users: number;
  count: number;
  // 多指标前缀字段：m0_id, m0_main_sum, m0_main_sum_squares, ...
  //                  m1_id, m1_main_sum, m1_main_sum_squares, ...
  [key: string]: number | string;
}[];
```

**特点**：每行包含多个指标的数据，每个指标的字段带 `m{i}_` 前缀（如 `m0_main_sum`、`m1_main_sum`），通过 `m{i}_id` 标识指标 ID。

### 3.3 结果行处理

**Legacy Metric 行处理**：`runExperimentMetricQuery()`（`SqlIntegration.ts:511-619`）
```typescript
async runExperimentMetricQuery(query, setExternalId, queryMetadata) {
  const { rows, statistics } = await this.runQuery(query, setExternalId, queryMetadata);
  return {
    rows: rows.map(row => ({
      variation: row.variation ?? "",
      ...dimensionData,
      users: parseIntWithDefault(row.users, 0),
      main_sum: parseFloat(row.main_sum as string) || 0,
      main_sum_squares: parseFloat(row.main_sum_squares as string) || 0,
      // ... 解析其他字段（denominator、covariate、quantile 等）
    })),
    statistics,
  };
}
```

**Fact Metric 行处理**：`processExperimentFactMetricsQueryRows()`（`process-experiment-fact-metrics-query-rows.ts:11-51`）
```typescript
export function processExperimentFactMetricsQueryRows(rows) {
  return rows.map(row => {
    let metricData = {};
    // 遍历所有可能的指标槽位（最多 MAX_METRICS_PER_QUERY 个）
    for (let i = 0; i < MAX_METRICS_PER_QUERY; i++) {
      const prefix = `m${i}_`;
      if (!row[prefix + "id"]) break;  // 没有更多指标
      
      metricData[prefix + "id"] = row[prefix + "id"];
      // 解析所有非 quantile 浮点字段
      ALL_NON_QUANTILE_METRIC_FLOAT_COLS.forEach(col => {
        if (row[prefix + col] !== undefined) {
          metricData[prefix + col] = parseFloat(row[prefix + col]) || 0;
        }
      });
      // 解析 quantile 边界
      metricData = { ...metricData, ...getQuantileBoundsFromQueryResponse(row, prefix) };
    }
    return {
      variation: row.variation ?? "",
      ...dimensionData,
      users: parseIntWithDefault(row.users, 0),
      count: parseIntWithDefault(row.users, 0),
      ...metricData,  // 展开所有带前缀的指标字段
    };
  });
}
```

---

## 4. 统计引擎并轨：统一输入与分析

### 4.1 核心汇合点：`getMetricsAndQueryDataForStatsEngine()`

**文件**：`packages/back-end/src/services/stats.ts:345-455`

这是两条路径在统计引擎层面的**最终汇合点**。该函数接收 `QueryMap`（包含所有查询结果），将两种格式的结果统一转换为统计引擎可接受的格式。

```typescript
export function getMetricsAndQueryDataForStatsEngine(
  queryData: QueryMap,
  metricMap: Map<string, ExperimentMetricInterface>,
  settings: ExperimentSnapshotSettings,
) {
  const queryResults: QueryResultsForStatsEngine[] = [];
  const metricSettings: Record<string, MetricSettingsForStatsEngine> = {};
  let unknownVariations: string[] = [];

  // 遍历所有查询结果
  queryData.forEach((query, key) => {
    
    // ========== 路径 A：Fact Metric 组查询 ==========
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

    // ========== 路径 B：Legacy Metric 单查询 ==========
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

### 4.2 统一输出格式：`QueryResultsForStatsEngine`

**文件**：`packages/shared/types/stats.d.ts:252-258`

```typescript
export interface QueryResultsForStatsEngine {
  rows: ExperimentMetricQueryResponseRows | ExperimentFactMetricsQueryResponseRows;
  metrics: (string | null)[];  // 该行数据对应的指标 ID 列表
  sql?: string;
}
```

**关键统一机制**：
- `metrics` 字段明确指示当前行数据包含哪些指标
- 对于 Legacy Metric，`metrics` 是单元素数组（如 `["metric_123"]`）
- 对于 Fact Metric，`metrics` 是多元素数组（如 `["metric_123", "metric_456", null]`）
- 统计引擎内部根据 `metrics` 字段和行数据的前缀/字段名对应关系，正确提取每个指标的统计值

### 4.3 指标元数据统一：`getMetricSettingsForStatsEngine()`

**文件**：`packages/back-end/src/services/stats.ts:267-343`

为两种类型的指标生成统一的统计引擎配置：

```typescript
export function getMetricSettingsForStatsEngine(
  metricDoc: ExperimentMetricInterface,
  metricMap: Map<string, ExperimentMetricInterface>,
  settings: ExperimentSnapshotSettings,
  optimizedFactMetric: boolean = false,
): MetricSettingsForStatsEngine {
  const metric = cloneDeep(metricDoc);
  applyMetricOverrides(metric, settings);

  // 统一判断指标类型
  const ratioMetric = isRatioMetric(metric, denominator);
  const quantileMetric = quantileMetricType(metric);
  const regressionAdjusted = settings.regressionAdjustmentEnabled && 
    isRegressionAdjusted(metric, denominator) &&
    (!isRatioMetric(metric, denominator) || optimizedFactMetric);
  
  const mainMetricType = quantileMetric
    ? "quantile"
    : isBinomialMetric(metric)
      ? "binomial"
      : "count";

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

### 4.4 统计引擎执行：`runStatsEngine()`

**文件**：`packages/back-end/src/services/stats.ts:140-176`

统一调用 Python 统计引擎（gbstats），完全不区分指标来源：

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
  analyses: AnalysisSettingsForStatsEngine[];      // 分析配置
  metrics: Record<string, MetricSettingsForStatsEngine>;  // 所有指标的元数据
  query_results: QueryResultsForStatsEngine[];    // 统一格式的查询结果
  bandit_settings?: BanditSettingsForStatsEngine;
}
```

### 4.5 结果解析统一：`parseStatsEngineResult()`

**文件**：`packages/back-end/src/services/stats.ts:464-588`

统计引擎返回的结果格式完全统一，不区分指标来源：

```typescript
function parseStatsEngineResult({ analysisSettings, snapshotSettings, queryResults, unknownVariations, result }) {
  const experimentReportResults: ExperimentReportResults[] = [];
  
  analysisSettings.forEach((_, i) => {
    const dimensionMap: Map<string, ExperimentReportResultDimension> = new Map();
    
    // result 是所有指标的分析结果数组
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

## 5. 完整并轨流程

### 5.1 总览：四层汇合架构

```
┌─────────────────────────────────────────────────────────────────┐
│                     定义层抽象 (Definition)                     │
│  ExperimentMetricInterface = MetricInterface | FactMetricInterface│
│  isFactMetric() / isLegacyMetric() 类型守卫                      │
│  getMetricCTE() / getMetricColumns() 统一 SQL 生成                │
└─────────────────────────┬───────────────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────────────┐
│                   查询计划层 (Query Planning)                    │
│  getFactMetricGroups() 分流：                                    │
│    - legacyMetricSingles → 单指标查询                            │
│    - factMetricGroups → 按 Fact Table 分组批量查询                │
│  startExperimentResultQueries() 路由执行                         │
└─────────────────────────┬───────────────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────────────┐
│                   Warehouse 执行层 (Execution)                   │
│  Legacy: getExperimentMetricQuery() → runExperimentMetricQuery()│
│          行格式：{ variation, main_sum, ... }                    │
│  Fact:   getExperimentFactMetricsQuery() → runExperiment...()   │
│          行格式：{ variation, m0_main_sum, m1_main_sum, ... }    │
└─────────────────────────┬───────────────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────────────┐
│                    统计引擎层 (Stats Engine)                     │
│  getMetricsAndQueryDataForStatsEngine() 核心汇合点               │
│    - 统一为 QueryResultsForStatsEngine 格式                       │
│    - metrics 字段标记每行包含的指标                               │
│  runStatsEngine() 统一调用 Python gbstats                        │
│  parseStatsEngineResult() 统一解析结果                           │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 Legacy Metric 完整路径

```
ExperimentResultsQueryRunner.startQueries()
  → getFactMetricGroups() [筛选出 legacyMetricSingles]
  → getExperimentMetricQuery() [单指标 SQL 生成]
    → getMetricCTE() [定义层抽象，使用 metric.sql]
      → getMetricColumns() [解析列]
  → runExperimentMetricQuery() [执行 SQL]
    → runQuery() [warehouse 调用]
    → 结果行解析：{ variation, main_sum, main_sum_squares, ... }
  → getMetricsAndQueryDataForStatsEngine() [汇合点]
    → 包装为 { metrics: ["metric_id"], rows: [...] }
    → 生成 MetricSettingsForStatsEngine
  → runStatsEngine() [统一统计分析]
  → parseStatsEngineResult() [统一结果解析]
  → 输出 ExperimentReportResults
```

### 5.3 Fact Metric 完整路径

```
ExperimentResultsQueryRunner.startQueries()
  → getFactMetricGroups() [分组优化]
  → getExperimentFactMetricsQuery() [多指标 SQL 生成]
    → getFactTablesForMetrics() [收集涉及的 Fact Table]
    → getMetricData() [解析每个指标的聚合元数据]
    → getFactMetricCTE() [定义层抽象，使用 factTable.sql]
      → getFactMetricColumn() [解析列引用]
    → getExperimentFactMetricStatisticsCTE() [批量统计]
  → runExperimentFactMetricsQuery() [执行 SQL]
    → runQuery() [warehouse 调用]
    → processExperimentFactMetricsQueryRows() [前缀解析]
    → 结果行：{ variation, m0_id, m0_main_sum, m1_id, m1_main_sum, ... }
  → getMetricsAndQueryDataForStatsEngine() [汇合点]
    → 提取 m{i}_id 得到 metrics: ["metric1", "metric2", ...]
    → 为每个指标生成 MetricSettingsForStatsEngine
    → 包装为 { metrics: ["metric1", "metric2"], rows: [...] }
  → runStatsEngine() [统一统计分析]
  → parseStatsEngineResult() [统一结果解析]
  → 输出 ExperimentReportResults
```

---

## 6. 关键设计要点

### 6.1 四层抽象的设计价值

1. **定义层**：通过联合类型和类型守卫，实现两种指标的多态处理
2. **查询计划层**：根据指标类型和特征进行优化分组，最大化查询效率
3. **Warehouse 层**：两种路径独立优化，Legacy 保持简单兼容，Fact Table 实现批量优化
4. **统计引擎层**：通过 `QueryResultsForStatsEngine` 实现完全解耦，统计引擎无需关心数据来源

### 6.2 统计引擎汇合的关键机制

- **前缀约定**：Fact Metric 使用 `m{i}_` 前缀在单行中打包多个指标数据
- **指标标记**：`metrics` 数组明确每行数据对应的指标 ID，实现自描述
- **元数据分离**：`metricSettings` 单独提供所有指标的统计配置，与行数据解耦
- **统一分析**：统计引擎内部根据 `metrics` 数组和字段前缀自动匹配数据

### 6.3 Fact Table 路径的优化点

1. **批量查询**：同一 Fact Table 的多个指标共享一次表扫描
2. **列裁剪**：CTE 中只投影需要的列，减少数据传输
3. **谓词下推**：日期过滤器和指标过滤器下推到 Fact Table CTE 中
4. **增量刷新**：支持 KLL sketch 等预聚合结构，避免全表重扫

### 6.4 两条路径的差异对比

| 维度 | SQL 直接定义 (Legacy) | Fact Table |
|------|----------------------|------------|
| SQL 源 | 用户编写的 `metric.sql` | 预定义 `factTable.sql` |
| 粒度 | 每个指标独立 SQL | 同 Fact Table 指标合并查询 |
| 行格式 | 单行单指标，无前缀 | 单行多指标，`m{i}_` 前缀 |
| 结果处理 | 直接字段映射 | 前缀解析 + 指标 ID 提取 |
| 统计引擎输入 | `metrics: [singleId]` | `metrics: [id1, id2, ...]` |
| 优化能力 | 有限（单指标优化） | 丰富（批量扫描、列裁剪、谓词下推） |
| 复用性 | 低（SQL 散落各处） | 高（Fact Table 可被多个指标复用） |
| 迁移成本 | 低（直接写 SQL） | 中（需要先定义 Fact Table） |

---

## 7. 核心文件索引

| 文件 | 职责 |
|------|------|
| `packages/shared/types/metric.d.ts` | Legacy Metric 类型定义 |
| `packages/shared/types/fact-table.d.ts` | Fact Table / Fact Metric 类型定义 |
| `packages/shared/types/integrations.d.ts` | 查询结果行类型定义 |
| `packages/shared/types/stats.d.ts` | 统计引擎输入输出类型定义 |
| `packages/shared/src/experiments/experiments.ts` | 类型守卫、通用工具函数 |
| `packages/back-end/src/services/experimentQueries/experimentQueries.ts` | 指标分组逻辑 `getFactMetricGroups()` |
| `packages/back-end/src/queryRunners/ExperimentResultsQueryRunner.ts` | 查询计划生成与路由 |
| `packages/back-end/src/integrations/sql/ctes/metric-cte.ts` | 定义层汇合点 `getMetricCTE()` |
| `packages/back-end/src/integrations/sql/ctes/fact-metric-cte.ts` | Fact Table CTE 生成 |
| `packages/back-end/src/integrations/sql/queries/experiment-metric-query.ts` | Legacy Metric SQL 生成 |
| `packages/back-end/src/integrations/sql/queries/experiment-fact-metrics-query.ts` | Fact Metric SQL 生成 |
| `packages/back-end/src/integrations/sql/columns/metric-columns.ts` | 统一列解析 `getMetricColumns()` |
| `packages/back-end/src/integrations/sql/processing/process-experiment-fact-metrics-query-rows.ts` | Fact Metric 行前缀解析 |
| `packages/back-end/src/integrations/SqlIntegration.ts` | SQL 执行与结果处理 |
| `packages/back-end/src/services/stats.ts` | 统计引擎汇合点 `getMetricsAndQueryDataForStatsEngine()`、结果解析 |
