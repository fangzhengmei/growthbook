# 指标定义双路径衔接分析

## 概述

GrowthBook 中指标定义支持两条独立路径：
1. **SQL 直接定义路径**（Legacy Metric）：用户直接编写 SQL 定义指标
2. **Fact Table 路径**（Fact Metric）：指向预定义的 Fact Table，通过列引用和聚合方式定义指标

两条路径在查询时经过**定义层抽象**、**查询计划生成**和 **warehouse 端 SQL 发出**三个阶段最终合并，输出统一格式的查询结果。

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

## 3. Warehouse 端 SQL 生成

### 3.1 Legacy Metric SQL 生成：`getExperimentMetricQuery()`

**文件**：`packages/back-end/src/integrations/sql/queries/experiment-metric-query.ts:38-615`

单指标 SQL 生成流程：
1. CTE 结构：`__metric` → `__userMetricJoin` → `__userMetricAgg` → 最终聚合
2. 支持 Ratio、Funnel、CUPED、百分位封盖等高级特性
3. 每个指标独立 SQL，互不干扰

### 3.2 Fact Metric SQL 生成：`getExperimentFactMetricsQuery()`

**文件**：`packages/back-end/src/integrations/sql/queries/experiment-fact-metrics-query.ts:32-660`

多指标合并 SQL 生成流程：
1. **Fact Table CTE 生成**：`getFactMetricCTE()` 为每个 Fact Table 生成一个 CTE，一次性投影所有相关指标的数值列
2. **多指标 JOIN**：`__userMetricJoin` 同时 JOIN 所有 Fact Table，一次性获取所有指标值
3. **批量聚合**：`__userMetricAgg` 在单个 GROUP BY 中计算所有指标的聚合值
4. **统计 CTE**：`getExperimentFactMetricStatisticsCTE()` 一次性输出所有指标的统计结果

关键优化点（`experiment-fact-metrics-query.ts:266-631`）：
```typescript
// 为每个 Fact Table 生成 CTE
factTablesWithIndices.map(f => {
  // __factTableX: 从 Fact Table SQL 投影所有指标列
  // __userMetricJoinX: 与用户维度 JOIN，应用时间窗口过滤
  // __userMetricAggX: 按用户聚合
});

// 最终统计：一次性 SELECT 所有指标的统计结果
getExperimentFactMetricStatisticsCTE(dialect, {
  metricData,  // 所有指标的元数据
  ...
})
```

### 3.3 结果合并与统一输出

两条路径最终都通过相似的结果处理函数输出统一格式：

- **Legacy Metric**：`runExperimentMetricQuery()`（`SqlIntegration.ts:511-619`）
- **Fact Metric**：`runExperimentFactMetricsQuery()`

两者都解析出相同结构的结果行：
```typescript
{
  variation: string,
  dimension_*: string,  // 维度列
  users: number,
  main_sum: number,
  main_sum_squares: number,
  // ... 其他统计字段（ratio、covariate、uncapped 等）
}
```

---

## 4. 完整调用链

### 4.1 Legacy Metric 路径
```
ExperimentResultsQueryRunner.startQueries()
  → getFactMetricGroups() [筛选出 legacyMetricSingles]
  → getExperimentMetricQuery() [单指标 SQL 生成]
    → getMetricCTE() [定义层抽象，使用 metric.sql]
      → getMetricColumns() [解析列]
  → runExperimentMetricQuery() [执行 SQL]
    → runQuery() [warehouse 调用]
  → analyzeExperimentResults() [统一结果分析]
```

### 4.2 Fact Metric 路径
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
  → analyzeExperimentResults() [统一结果分析]
```

---

## 5. 关键设计要点

### 5.1 定义层抽象的好处
1. **向后兼容**：Legacy Metric 和 Fact Metric 可以共存，平滑迁移
2. **统一接口**：上层调用者无需关心底层是 SQL 还是 Fact Table
3. **扩展灵活**：新增指标类型只需添加类型守卫和相应的 SQL 生成逻辑

### 5.2 Fact Table 路径的优化点
1. **批量查询**：同一 Fact Table 的多个指标共享一次表扫描
2. **列裁剪**：CTE 中只投影需要的列，减少数据传输
3. **谓词下推**：日期过滤器和指标过滤器下推到 Fact Table CTE 中
4. **增量刷新**：支持 KLL sketch 等预聚合结构，避免全表重扫

### 5.3 两条路径的差异

| 维度 | SQL 直接定义 (Legacy) | Fact Table |
|------|----------------------|------------|
| SQL 源 | 用户编写的 `metric.sql` | 预定义 `factTable.sql` |
| 粒度 | 每个指标独立 SQL | 同 Fact Table 指标合并查询 |
| 优化能力 | 有限（单指标优化） | 丰富（批量扫描、列裁剪、谓词下推） |
| 复用性 | 低（SQL 散落各处） | 高（Fact Table 可被多个指标复用） |
| 迁移成本 | 低（直接写 SQL） | 中（需要先定义 Fact Table） |

---

## 6. 核心文件索引

| 文件 | 职责 |
|------|------|
| `packages/shared/types/metric.d.ts` | Legacy Metric 类型定义 |
| `packages/shared/types/fact-table.d.ts` | Fact Table / Fact Metric 类型定义 |
| `packages/shared/src/experiments/experiments.ts` | 类型守卫、通用工具函数 |
| `packages/back-end/src/services/experimentQueries/experimentQueries.ts` | 指标分组逻辑 `getFactMetricGroups()` |
| `packages/back-end/src/queryRunners/ExperimentResultsQueryRunner.ts` | 查询计划生成与路由 |
| `packages/back-end/src/integrations/sql/ctes/metric-cte.ts` | 定义层汇合点 `getMetricCTE()` |
| `packages/back-end/src/integrations/sql/ctes/fact-metric-cte.ts` | Fact Table CTE 生成 |
| `packages/back-end/src/integrations/sql/queries/experiment-metric-query.ts` | Legacy Metric SQL 生成 |
| `packages/back-end/src/integrations/sql/queries/experiment-fact-metrics-query.ts` | Fact Metric SQL 生成 |
| `packages/back-end/src/integrations/sql/columns/metric-columns.ts` | 统一列解析 `getMetricColumns()` |
| `packages/back-end/src/integrations/SqlIntegration.ts` | SQL 执行与结果处理 |
