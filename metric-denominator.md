# 分母校验与护栏机制配合流程分析

## 1. 分母定义 (Denominator Definition)

### 1.1 Legacy Metric 与 Fact Metric 的分母配置差异

#### Legacy Metric 的分母配置

在 `packages/shared/types/metric.d.ts:63` 中，`MetricInterface` 定义了分
段 ID：

```typescript
export interface MetricInterface {
  // ...
  denominator?: string;  // 分母度量的 ID (字符串)
  // ...
}
```

在 `packages/shared/src/validators/metrics.ts:117` 中，API 验证器定义了 `denominatorMetricId`：

```typescript
sql: z
  .object({
    identifierTypes: z.array(z.string()),
    conversionSQL: z.string(),
    userAggregationSQL: z.string(),
    denominatorMetricId: z.string(),  // 分母度量 ID
  })
  .optional(),
```

**特点**：
- 分母是一个字符串 ID，指向另一个度量
- 需要通过 `metricMap.get()` 查找获取实际度量对象
- 支持递归链式引用（A→B→C）

#### Fact Metric 的分母配置

Fact Metric 的分母存储在度量对象内部（不是 ID 引用）：

```typescript
// Fact Metric 的分子分母结构
export interface FactMetricInterface {
  // ...
  numerator: {
    factTableId: string;
    // ...
  };
  denominator?: {
    factTableId: string;
    // ...
  };
  // ...
}
```

**特点**：
- 分母是完整的对象，不是 ID 引用
- 分母与分子共享相同的结构
- 不支持链式递归引用（分母本身没有自己的分母）

### 1.2 分母度量的用途

分母度量主要用于：
- **比率度量 (Ratio Metrics)**：如 "每个用户的平均收入" (ARPU)
- **漏斗度量 (Funnel Metrics)**：计算转化率
- **标准化指标**：将分子数据按分母标准化，消除样本量偏差

---

## 2. Legacy Metric 分母处理流程

### 2.1 两种处理模式对比

系统中存在两种不同的分母处理模式，分别用于不同的业务场景：

| 处理模式 | 应用场景 | 展开方式 | 环路防护 |
|---------|---------|---------|---------|
| **单层展开** | Safe Rollout 快照设置 | 仅展开一层 | 无显式防护 |
| **递归展开** | 通用实验报表、人口数据分析 | 递归展开所有层 | 有显式 `visited` Set 防护 |

### 2.2 Safe Rollout 中的分母处理 (单层展开)

**位置**：`packages/back-end/src/services/safeRolloutSnapshots.ts:210-215`

```typescript
const denominatorMetrics = allExperimentMetrics
  .filter((m) => m && !isFactMetric(m) && m.denominator)
  .map((m: ExperimentMetricInterface) =>
    metricMap.get(m.denominator as string),
  )
  .filter(Boolean) as MetricInterface[];
```

**代码证据** (`safeRolloutSnapshots.ts:188-234`)：
```typescript
export async function getSettingsForSnapshotMetrics(
  context: ReqContext | ApiReqContext,
  safeRollout: SafeRolloutInterface,
): Promise<{
  regressionAdjustmentEnabled: boolean;
  settingsForSnapshotMetrics: MetricSnapshotSettings[];
}> {
  // ...
  const allExperimentMetrics = allExperimentMetricIds
    .map((id) => metricMap.get(id))
    .filter(isDefined);

  // ⚠️ 单层展开：只展开直接分母，不递归展开分母的分母
  const denominatorMetrics = allExperimentMetrics
    .filter((m) => m && !isFactMetric(m) && m.denominator)
    .map((m: ExperimentMetricInterface) =>
      metricMap.get(m.denominator as string),
    )
    .filter(Boolean) as MetricInterface[];

  for (const metric of allExperimentMetrics) {
    if (!metric) continue;
    const { metricSnapshotSettings } = getMetricSnapshotSettings({
      metric: metric,
      denominatorMetrics: denominatorMetrics,  // 传入单层展开的分母列表
      // ...
    });
    // ...
  }
  // ...
}
```

**关键点**：
- 仅对 Legacy Metric (非 Fact Metric) 提取分母
- **单层展开**：只展开直接分母，不递归处理分母的分母
  - 例如：A→B→C，只获取 B，不获取 C
- 通过 `metricMap.get()` 获取分母度量对象
- 使用 `.filter(Boolean)` 过滤掉不存在的分母度量
- **无显式环路防护**

### 2.3 递归展开函数 (expandDenominatorMetrics)

**定义位置**：`packages/back-end/src/util/sql.ts:183-199`

```typescript
// Recursively create list of metric denominators in order
// For example, a "step3" metric has denominator "step2", which itself has denominator "step1"
// If you pass "step3" into this, it will return ["step1","step2","step3"]
export function expandDenominatorMetrics(
  metric: string,
  map: Map<string, { denominator?: string }>,
  visited?: Set<string>,
): string[] {
  visited = visited || new Set();       // 环路防护：visited 集合
  const m = map.get(metric);
  if (!m) return [];
  if (visited.has(metric)) return [];   // 环路防护：检测到已访问过的度量，返回空数组

  visited.add(metric);
  if (!m.denominator) return [metric];
  // 递归展开：先展开分母，再追加当前度量
  return [...expandDenominatorMetrics(m.denominator, map, visited), metric];
}
```

**环路防护机制详解**：
1. **`visited` Set 参数**：记录已访问的度量 ID
2. **第191行**：`visited = visited || new Set()` - 初始化访问集合
3. **第194行**：`if (visited.has(metric)) return []` - 检测环路，返回空数组终止递归
4. **第196行**：`visited.add(metric)` - 标记当前度量为已访问

**测试用例验证** (`back-end/test/util/sql.test.ts:353-375`)：
```typescript
const metricMap = new Map<string, { denominator?: string }>(
  Object.entries({
    a: { denominator: "b" },
    b: {},
    c: { denominator: "d" },
    d: { denominator: "c" },  // 环路：c→d→c
    e: { denominator: "c" },
    f: { denominator: "f" },  // 自引用
    g: { denominator: "h" },  // h 不存在
  }),
);

expect(expandDenominatorMetrics("a", metricMap)).toEqual(["b", "a"]);
expect(expandDenominatorMetrics("c", metricMap)).toEqual(["d", "c"]);
expect(expandDenominatorMetrics("d", metricMap)).toEqual(["c", "d"]);
expect(expandDenominatorMetrics("e", metricMap)).toEqual(["d", "c", "e"]);  // 递归展开两层
expect(expandDenominatorMetrics("f", metricMap)).toEqual(["f"]);  // 自引用被防护
expect(expandDenominatorMetrics("g", metricMap)).toEqual(["g"]);  // h 不存在，只返回 g
expect(expandDenominatorMetrics("h", metricMap)).toEqual([]);     // h 不存在
```

### 2.4 递归展开的调用路径

#### 调用路径 1: 实验结果查询

**位置**：`packages/back-end/src/queryRunners/ExperimentResultsQueryRunner.ts:215-226`

```typescript
for (const m of legacyMetricSingles) {
  const denominatorMetrics: MetricInterface[] = [];
  if (m.denominator) {
    // 使用递归展开函数
    denominatorMetrics.push(
      ...expandDenominatorMetrics(
        m.denominator,
        metricMap as Map<string, MetricInterface>,
      )
        .map((m) => metricMap.get(m) as MetricInterface)
        .filter(Boolean),
    );
  }
  // ...
}
```

**完整调用链**：
```
ExperimentResultsQueryRunner.startQuery()
        ↓
legacyMetricSingles 循环处理每个度量
        ↓
if (m.denominator) → expandDenominatorMetrics(m.denominator, metricMap)
        ↓
返回 [最底层分母, 中间分母, 目标分母] 数组
        ↓
转换为 MetricInterface 对象并传入查询参数
```

#### 调用路径 2: 人口数据查询

**位置**：`packages/back-end/src/queryRunners/PopulationDataQueryRunner.ts:110-119`

```typescript
const denominatorMetrics: MetricInterface[] = [];
if (m.denominator) {
  denominatorMetrics.push(
    ...expandDenominatorMetrics(
      m.denominator,
      metricMap as Map<string, MetricInterface>,
    )
      .map((m) => metricMap.get(m) as MetricInterface)
      .filter(Boolean),
  );
}
```

#### 调用路径 3: 通用报表链路

**位置**：`packages/back-end/src/services/reports.ts:514-521`

```typescript
const denominatorMetricIds = uniq<string>(
  allReportMetrics
    .map((m) => m?.denominator)
    .filter((d) => d && typeof d === "string") as string[],
);
const denominatorMetrics = denominatorMetricIds
  .map((m) => metricMap.get(m) || null)
  .filter(isDefined) as MetricInterface[];
```

**注意**：报表设置阶段是单层展开，但在实际查询执行阶段（ExperimentResultsQueryRunner）会再次递归展开。

### 2.5 递归结果在查询构建中的使用方式

**查询参数定义** (`packages/shared/types/integrations.d.ts:398-404`)：
```typescript
export interface ExperimentMetricQueryParams extends ExperimentBaseQueryParams {
  metric: MetricInterface;
  denominatorMetrics: MetricInterface[];  // 递归展开的分母数组
  unitsSource: UnitsSource;
  unitsSql?: string;
  forcedUserIdType?: string;
}
```

**查询构建入口** (`packages/back-end/src/integrations/sql/queries/experiment-metric-query.ts:38-88`)：
```typescript
export function getExperimentMetricQuery(
  dialect: SqlDialect,
  datasource: DataSourceInterface,
  params: ExperimentMetricQueryParams,
): string {
  const {
    metric: metricDoc,
    denominatorMetrics: denominatorMetricsDocs,  // 接收递归展开的分母数组
    activationMetric: activationMetricDoc,
    settings,
    segment,
  } = params;

  // 1. 克隆并应用覆盖设置
  const metric = cloneDeep<MetricInterface>(metricDoc);
  const denominatorMetrics = cloneDeep<MetricInterface[]>(denominatorMetricsDocs);
  applyMetricOverrides(metric, settings);
  denominatorMetrics.forEach((m) => applyMetricOverrides(m, settings));

  // 2. 获取实际使用的分母（数组最后一个元素）
  const denominator =
    denominatorMetrics.length > 0
      ? denominatorMetrics[denominatorMetrics.length - 1]
      : undefined;

  // 3. 判断度量类型
  const ratioMetric = isRatioMetric(metric, denominator);
  const funnelMetric = isFunnelMetric(metric, denominator);
  // ...
}
```

**递归结果的使用场景**：

#### 场景 1: 生成所有分母度量的 CTE

**位置**：`experiment-metric-query.ts:315-341`
```typescript
, __metric as (${getMetricCTE(dialect, {
  metric,
  // ...
})})
${denominatorMetrics
  .map((m, i) => {
    // 为每个分母度量生成独立的 CTE
    return `, __denominator${i} as (${getMetricCTE(dialect, {
      metric: m,
      baseIdType,
      idJoinMap,
      startDate: metricStart,
      endDate: metricEnd,
      // ...
      useDenominator: true,  // 标记为分母用途
    })})`;
  })
  .join("\n")}
```

**结果**：
- 如果 `denominatorMetrics = [B, C]`（A→B→C）
- 会生成：`__denominator0` (B), `__denominator1` (C) 两个 CTE

#### 场景 2: Funnel 度量构建用户集合

**位置**：`experiment-metric-query.ts:342-356`
```typescript
${
  funnelMetric
    ? `, __denominatorUsers as (${getFunnelUsersCTE(
        dialect,
        baseIdType,
        denominatorMetrics,  // 传入完整分母链
        settings.endDate,
        dimensionCols,
        regressionAdjusted,
        overrideConversionWindows,
        banditDates,
        "__denominator",
        "__distinctUsers",
      )})`
    : ""
}
```

**Funnel 用户 CTE**：
- 使用完整分母链构建漏斗用户集合
- 例如：Purchase/Signup 漏斗，需要先有 Signup 的用户，然后看其中多少人 Purchase
- 所有分母度量都参与漏斗过滤

#### 场景 3: Ratio 度量的分母聚合

**位置**：`experiment-metric-query.ts:420-474`
```typescript
${
  denominator && ratioMetric
    ? `, __userDenominatorAgg AS (
        SELECT
          d.variation AS variation
          // ...
          , ${getAggregateMetricColumnLegacyMetrics(dialect, {
            metric: denominator,  // 使用最后一个分母
          })} as value
        FROM
          __distinctUsers d
          JOIN __denominator${denominatorMetrics.length - 1} m ON (
            // 只 JOIN 最后一个分母的 CTE
            m.${baseIdType} = d.${baseIdType}
          )
        // ...
      )`
    : ""
}
```

**关键逻辑**：
- Ratio 度量只使用 `denominatorMetrics[denominatorMetrics.length - 1]`（最后一个分母）
- 只 JOIN 最后一个分母的 CTE
- 中间分母（如 B 在 A→B→C 中）仅用于 Funnel 场景

#### 场景 4: 日期范围计算

**位置**：`experiment-metric-query.ts:185-198`
```typescript
// 所有度量（包括全部分母）参与日期范围计算
const orderedMetrics = (activationMetric ? [activationMetric] : [])
  .concat(denominatorMetrics)  // 包含全部分母
  .concat([metric]);
const minMetricDelay = getMetricMinDelay(orderedMetrics);
const metricStart = getMetricStart(settings.startDate, minMetricDelay, ...);
const metricEnd = getMetricEnd(orderedMetrics, settings.endDate, ...);
```

---

## 3. Fact Metric 分母处理流程

### 3.1 Fact Metric 的分母结构

Fact Metric 的分母是内嵌对象，不是 ID 引用：

```typescript
export interface FactMetricInterface {
  id: string;
  metricType: "ratio" | "proportion" | "mean";
  numerator: {
    factTableId: string;
    column: string;
    filters: MetricFilter[];
    aggregation: "count" | "sum" | "countDistinct";
  };
  denominator?: {
    factTableId: string;
    column: string;
    filters: MetricFilter[];
    aggregation: "count" | "sum" | "countDistinct";
  };
  // ...
}
```

### 3.2 Fact Metric 分母的分组逻辑

**位置**：`packages/back-end/src/services/experimentQueries/experimentQueries.ts:209-247`

```typescript
function getFactMetricGroup(metric: FactMetricInterface): string {
  // ...
  if (isRatioMetric(metric)) {
    if (metric.numerator.factTableId !== metric.denominator?.factTableId) {
      // 跨表比率度量：使用分子分母 factTableId 组合作为 group key
      const tableIds = [
        metric.numerator.factTableId,
        metric.denominator?.factTableId,
      ].sort((a, b) => a?.localeCompare(b ?? "") ?? 0);
      return tableIds.length >= 2
        ? `${tableIds[0]} ${tableIds[1]} (cross-table ratio metrics)`
        : metric.id;
    }
  }
  // ...
  return metric.numerator.factTableId || "";
}
```

**分组规则**：
1. **同表比率度量**：按 factTableId 分组，可以与其他同表度量一起查询
2. **跨表比率度量**：单独分组（分子分母 factTableId 组合），不与其他度量合并
3. **分位数度量**：单独分组，防止主查询变慢

### 3.3 Fact Metric 与 Legacy Metric 的分母处理对比

| 维度 | Legacy Metric | Fact Metric |
|------|--------------|-------------|
| **分母存储方式** | 字符串 ID 引用 | 内嵌完整对象 |
| **链式引用支持** | ✅ 支持（A→B→C） | ❌ 不支持 |
| **递归展开需要** | ✅ 需要 | ❌ 不需要 |
| **环路防护需要** | ✅ 需要 | ❌ 不需要 |
| **查询构建方式** | 多 CTE JOIN | 单查询（可能跨表 JOIN） |
| **度量分组策略** | 每个度量单独查询 | 同 factTable 可合并查询 |

---

## 4. 环路防护生效范围 (Loop Guardrail Scope)

### 4.1 生效范围对比

| 链路 | 是否使用 `expandDenominatorMetrics` | 环路防护是否生效 | 说明 |
|------|------------------------------------|-----------------|------|
| **Safe Rollout 快照设置** (`safeRolloutSnapshots.ts`) | ❌ 否 | ❌ 不生效 | 单层展开，依赖 `metricMap.get()` 隐式保护 |
| **实验结果查询** (`ExperimentResultsQueryRunner.ts`) | ✅ 是 | ✅ 生效 | 递归展开，`visited` Set 防护 |
| **人口数据查询** (`PopulationDataQueryRunner.ts`) | ✅ 是 | ✅ 生效 | 递归展开，`visited` Set 防护 |
| **报表设置** (`reports.ts`) | ❌ 否 | ❌ 不生效 | 单层展开，但后续查询阶段会递归 |
| **Fact Metric 查询** | ❌ 否 | ❌ 不适用 | 分母是内嵌对象，不支持链式引用 |

### 4.2 环路风险分析

**Safe Rollout 链路的潜在风险**：
- 若 `A.denominator = B` 且 `B.denominator = A`，形成自引用环路
- Safe Rollout 中只展开一层，因此只会收集 `[B]`，不会无限递归
- 但 B 本身的分母 A 不会被展开，导致数据不完整

**代码风险点** (`safeRolloutSnapshots.ts:210-215`)：
```typescript
// 假设 A.denominator = B, B.denominator = A
const denominatorMetrics = allExperimentMetrics  // allExperimentMetrics = [A]
  .filter((m) => m && !isFactMetric(m) && m.denominator)  // A 有分母 B
  .map((m) => metricMap.get(m.denominator as string))      // 获取 B
  .filter(Boolean);
// 结果: denominatorMetrics = [B]
// B 的分母 A 不会被展开，因为只做了一层 map
```

### 4.3 环路防护的测试验证

测试用例 (`sql.test.ts:353-375`) 验证了以下场景：
1. ✅ 正常链式引用：`a→b` → `["b", "a"]`
2. ✅ 双向环路：`c→d→c` → 输入 c 返回 `["d", "c"]`，输入 d 返回 `["c", "d"]`
3. ✅ 长链式引用：`e→c→d→c` → `["d", "c", "e"]`（c 已被访问，不再递归）
4. ✅ 自引用环路：`f→f` → `["f"]`（检测到已访问，终止递归）
5. ✅ 不存在的分母：`g→h` (h 不存在) → `["g"]`
6. ✅ 不存在的度量：输入 `h` → `[]`

---

## 5. 递归结果使用的边界条件

### 5.1 边界条件 1: 分母数组为空

**代码位置**：`experiment-metric-query.ts:78-81`
```typescript
const denominator =
  denominatorMetrics.length > 0
    ? denominatorMetrics[denominatorMetrics.length - 1]
    : undefined;
```

**影响**：
- `denominator` 为 `undefined`
- `ratioMetric = false`
- `funnelMetric = false`
- 不生成分母相关的 CTE 和聚合逻辑

### 5.2 边界条件 2: Funnel vs Ratio 度量区分

**代码位置**：`experiment-metric-query.ts:82-87`
```typescript
// If the denominator is a binomial, it's just acting as a filter
// e.g. "Purchase/Signup" is filtering to users who signed up and then counting purchases
// When the denominator is a count, it's a real ratio, dividing two quantities
// e.g. "Pages/Session" is dividing number of page views by number of sessions
const ratioMetric = isRatioMetric(metric, denominator);
const funnelMetric = isFunnelMetric(metric, denominator);
```

**判定逻辑** (`packages/shared/src/experiments/experiments.ts:462-468`)：
```typescript
export function isFunnelMetric(
  m: ExperimentMetricInterface,
  denominatorMetric?: ExperimentMetricInterface,
): boolean {
  if (isFactMetric(m)) return false;
  return !!denominatorMetric && isBinomialMetric(denominatorMetric);
}
```

**结果差异**：
- **Funnel 度量**（分母是二项分布）：
  - 生成 `__denominatorUsers` CTE，过滤用户
  - 主查询 JOIN `__denominatorUsers`，不是直接 JOIN 分母数据
  - 使用全部分母链

- **Ratio 度量**（分母是计数型）：
  - 不生成 `__denominatorUsers`
  - 主查询直接 JOIN 最后一个分母的 CTE
  - 只使用最后一个分母

### 5.3 边界条件 3: Fact Metric 的特殊处理

Fact Metric 在查询中被分组处理，不经过 `expandDenominatorMetrics`：

**代码位置**：`ExperimentResultsQueryRunner.ts:259-297`
```typescript
for (const [i, m] of factMetricGroups.entries()) {
  const queryParams: ExperimentFactMetricsQueryParams = {
    activationMetric,
    dimensions: ...,
    metrics: m,  // 直接传入 Fact Metric 数组
    segment: segmentObj,
    settings: snapshotSettings,
    // ...
  };
  // Fact Metric 多度量合并查询
  queries.push(
    await startQuery({
      name: `group_${i}`,
      query: integration.getExperimentFactMetricsQuery(queryParams),
      // ...
    }),
  );
}
```

**注意**：Fact Metric 的分母在 SQL 生成阶段直接从 `metric.denominator` 对象读取，不需要预先展开。

### 5.4 边界条件 4: 百分位截断的特殊处理

**代码位置**：`experiment-metric-query.ts:451-471`
```typescript
${
  denominator && denominatorIsPercentileCapped
    ? `
  , __capValueDenominator AS (
      ${dialect.percentileCapSelectClause(
        [
          {
            valueCol: "value",
            outputCol: "value_cap",
            percentile: denominator.cappingSettings.value ?? 1,
            // ...
          },
        ],
        "__userDenominatorAgg",
        // ...
      )}
    )
  `
    : ""
}
```

**边界条件**：
- 只有 Ratio 度量且分母启用了百分位截断时，才会生成分母的截断 CTE
- 此逻辑只针对最后一个分母

---

## 6. 分母处理链路总览

### 6.1 Legacy Metric 完整数据流转图

```
+-------------------+
|  Legacy Metric    |
|  metric.denominator|
|  (string ID)      |
+---------+---------+
          |
          v
┌─────────────────────────────────────────────────────────────────────┐
│  分母校验链路分支                                                  │
│                                                                    │
│  ┌─────────────────────────────┐    ┌─────────────────────────────┐
│  │  Safe Rollout 链路          │    │  通用查询链路               │
│  │  (safeRolloutSnapshots.ts)  │    │  (ExperimentResultsQuery-  │
│  │                             │    │   Runner.ts)                │
│  └──────────┬──────────────────┘    └──────────┬──────────────────┘
│             │                                    │                   │
│             ▼                                    ▼                   │
│  ┌─────────────────────┐          ┌─────────────────────────────┐ │
│  │  单层展开           │          │  递归展开                   │ │
│  │  allExperiment-     │          │  expandDenominatorMetrics() │ │
│  │  Metrics.filter()   │          │  (递归函数 + visited Set)   │ │
│  │  .map()             │          └──────────┬──────────────────┘ │
│  └──────────┬──────────┘                     │                     │
│             │                                │                     │
│             ▼                                ▼                     │
│  ┌─────────────────────┐          ┌─────────────────────────────┐ │
│  │  环路防护: 无        │          │  环路防护: 有               │ │
│  │  仅 filter(Boolean) │          │  visited Set 检测           │ │
│  └──────────┬──────────┘          └──────────┬──────────────────┘ │
│             │                                │                     │
│             ▼                                ▼                     │
│  ┌─────────────────────┐          ┌─────────────────────────────┐ │
│  │  分母列表           │          │  完整分母链                 │ │
│  │  [B] (A→B→C)        │          │  [B, C, ...]                │ │
│  └──────────┬──────────┘          └──────────┬──────────────────┘ │
│             │                                │                     │
│             ▼                                ▼                     │
│  ┌─────────────────────┐          ┌─────────────────────────────┐ │
│  │  回归调整设置       │          │  SQL 查询构建               │ │
│  │  getMetricSnapshot- │          │  getExperimentMetricQuery() │ │
│  │  Settings()         │          │  - 多 CTE 生成              │ │
│  └─────────────────────┘          │  - Funnel/Ratio 分支        │ │
│                                   │  - 用户聚合计算              │ │
│                                   └─────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

### 6.2 分母处理对比表

| 维度 | Safe Rollout 链路 (Legacy) | 通用查询链路 (Legacy) | Fact Metric 链路 |
|------|---------------------------|----------------------|-----------------|
| **文件** | `safeRolloutSnapshots.ts` | `ExperimentResultsQueryRunner.ts` | 同上，Fact 分支 |
| **展开方式** | 单层 `.map()` | 递归 `expandDenominatorMetrics()` | 无需展开 |
| **环路防护** | ❌ 无显式防护 | ✅ `visited` Set 检测 | ❌ 不适用 |
| **处理深度** | 仅直接分母 | 完整分母链 | N/A |
| **应用场景** | 快照设置、回归调整 | SQL 查询构建、数据获取 | SQL 查询构建 |
| **调用时机** | 快照创建前 | 查询执行时 | 查询执行时 |
| **查询中的使用** | 不直接用于查询 | 全部分母生成 CTE | 直接读取对象 |

### 6.3 递归结果使用总结

| 使用场景 | 使用的分母 | 代码位置 |
|---------|-----------|---------|
| **生成 Metric CTE** | 全部 `denominatorMetrics` | `experiment-metric-query.ts:326-341` |
| **Funnel 用户集合** | 全部 `denominatorMetrics` | `experiment-metric-query.ts:342-356` |
| **Ratio 分母聚合** | 最后一个 (`[length - 1]`) | `experiment-metric-query.ts:420-474` |
| **日期范围计算** | 全部 `denominatorMetrics` | `experiment-metric-query.ts:185-198` |
| **用户 ID 类型** | 全部 `denominatorMetrics` | `experiment-metric-query.ts:201-204` |
| **回归调整协变量** | 主度量，不分母 | `experiment-metric-query.ts:476-550` |

---

## 7. 护栏阈值设置 (Guardrail Threshold Settings)

### 7.1 核心阈值常量

在 `packages/shared/src/constants.ts` 中定义了默认阈值：

| 常量名称 | 默认值 | 说明 |
|---------|-------|------|
| `DEFAULT_SRM_THRESHOLD` | 0.001 | SRM (样本比率不匹配) p-value 阈值 |
| `DEFAULT_SRM_MINIMINUM_COUNT_PER_VARIATION` | 8 | 每个变体最小用户数 |
| `DEFAULT_MULTIPLE_EXPOSURES_THRESHOLD` | 0.01 | 多重暴露百分比阈值 (1%) |
| `DEFAULT_MULTIPLE_EXPOSURES_ENOUGH_DATA_THRESHOLD` | 10 | 多重暴露数据量阈值 |
| `DEFAULT_P_VALUE_THRESHOLD` | 0.05 | 统计显著性 p-value 阈值 |
| `DEFAULT_GUARDRAIL_ALPHA` | 0.05 | 护栏度量显著性水平 |

### 7.2 组织级别设置

在 `packages/shared/src/enterprise/decision-criteria/decisionCriteria.ts:245-261` 中，`getHealthSettings` 函数合并组织设置与默认值：

```typescript
export function getHealthSettings(
  settings?: OrganizationSettings,
  hasDecisionFramework?: boolean,
): ExperimentHealthSettings {
  return {
    decisionFrameworkEnabled:
      (settings?.decisionFrameworkEnabled ?? DEFAULT_DECISION_FRAMEWORK_ENABLED) &&
      !!hasDecisionFramework,
    experimentMinLengthDays:
      settings?.experimentMinLengthDays ?? DEFAULT_EXPERIMENT_MIN_LENGTH_DAYS,
    srmThreshold: settings?.srmThreshold ?? DEFAULT_SRM_THRESHOLD,
    multipleExposureMinPercent:
      settings?.multipleExposureMinPercent ?? DEFAULT_MULTIPLE_EXPOSURES_THRESHOLD,
  };
}
```

### 7.3 度量级别阈值

在 `packages/shared/types/metric.d.ts:74-79` 中，每个度量可以配置独立的阈值：

```typescript
export interface MetricInterface {
  // ...
  winRisk?: number;           // 获胜风险阈值
  loseRisk?: number;          // 失败风险阈值
  maxPercentChange?: number;  // 最大变化百分比阈值
  minPercentChange?: number;  // 最小变化百分比阈值
  minSampleSize?: number;     // 最小样本量
  targetMDE?: number;         // 目标最小可检测效应
  // ...
}
```

---

## 8. 异常拦截机制 (Anomaly Interception Mechanism)

### 8.1 SRM (Sample Ratio Mismatch) 样本比率不匹配检测

**核心逻辑** 在 `packages/shared/src/health/health.ts:74-96`：

```typescript
export function getSRMHealthData({
  srm,                    // 计算出的 SRM p-value
  numOfVariations,        // 变体数量
  totalUsersCount,        // 总用户数
  srmThreshold,           // SRM 阈值 (默认 0.001)
  minUsersPerVariation,   // 每个变体最小用户数
}: {
  srm: number;
  numOfVariations: number;
  totalUsersCount: number;
  srmThreshold: number;
  minUsersPerVariation: number;
}): SRMHealthStatus {
  const minUsersCount = numOfVariations * minUsersPerVariation;

  if (totalUsersCount < minUsersCount) {
    return "not-enough-traffic";  // 流量不足，不做判断
  } else if (srm < srmThreshold) {
    return "unhealthy";           // p-value < 阈值，标记为不健康
  } else {
    return "healthy";             // 健康
  }
}
```

**SRM 值获取** 在 `packages/shared/src/health/health.ts:98-155`：
- 从健康查询结果中获取 `snapshot.health?.traffic?.overall?.srm`
- 如无健康查询结果，回退到主分析结果 `snapshot.analyses?.[0]?.results?.[0]?.srm`

### 8.2 多重暴露检测

**核心逻辑** 在 `packages/shared/src/health/health.ts:25-62`：

```typescript
export function getMultipleExposureHealthData({
  multipleExposuresCount,    // 多重暴露用户数
  totalUsersCount,           // 总用户数
  minCountThreshold,         // 最小数据量阈值 (默认 10)
  minPercentThreshold,       // 最小百分比阈值 (默认 0.01)
}: {
  multipleExposuresCount: number;
  totalUsersCount: number;
  minCountThreshold: number;
  minPercentThreshold: number;
}): MultipleExposureHealthData {
  const multipleExposureDecimal = multipleExposuresCount / totalUsersCount;
  const hasEnoughData = totalUsersCount >= minCountThreshold;
  const isUnhealthy = multipleExposureDecimal >= minPercentThreshold;

  // 返回状态: not-enough-traffic / unhealthy / healthy
}
```

### 8.3 护栏度量异常拦截

在 `packages/shared/src/enterprise/decision-criteria/decisionCriteria.ts:648-665` 中，Safe Rollout 使用专门的决策标准：

```typescript
const ROLLBACK_SAFE_ROLLOUT_DECISION_CRITERIA: DecisionCriteriaData = {
  id: "gbdeccrit_rollback_safe_rollout",
  name: "Rollback Safe Rollout",
  rules: [
    {
      conditions: [
        {
          match: "any",              // 任意一个护栏度量满足条件
          metrics: "guardrails",     // 检查护栏度量
          direction: "statsigLoser", // 状态为 "lost" (统计显著下降)
        },
      ],
      action: "rollback",           // 执行回滚操作
    },
  ],
  defaultAction: "review",
};
```

**状态评估逻辑** 在 `evaluateDecisionRuleOnVariation` 函数 (第40-108行)：
- 遍历所有护栏度量
- 检查度量状态是否为 "lost" (显著下降)
- 如果任意护栏度量状态为 "lost"，触发回滚

---

## 9. 自动回滚执行机制 (Auto Rollback Execution)

### 9.1 自动回滚实际执行位置

**入口点**：`packages/back-end/src/models/SafeRolloutSnapshotModel.ts:158`

当快照更新完成后，在 `afterUpdateOne` hook 中调用自动回滚检查：

```typescript
// 在快照更新完成后
const status = await checkAndRollbackSafeRollout({
  context: this.context,
  updatedSafeRollout,
  safeRolloutSnapshot: updatedDoc,
  ruleId: matchingRule.id,
  feature,
});
```

**核心执行函数**：`packages/back-end/src/enterprise/saferollouts/safeRolloutUtils.ts:68-166`

```typescript
export async function checkAndRollbackSafeRollout({
  context,
  updatedSafeRollout,
  safeRolloutSnapshot,
  ruleId,
  feature,
}: {
  context: ReqContext;
  updatedSafeRollout: SafeRolloutInterface;
  safeRolloutSnapshot: SafeRolloutSnapshotInterface;
  ruleId: string;
  feature: FeatureInterface;
}): Promise<SafeRolloutStatus> {
  // 前置条件检查
  if (updatedSafeRollout.status !== "running") 
    return updatedSafeRollout.status;
  if (!updatedSafeRollout.autoRollback) 
    return updatedSafeRollout.status;  // 未开启自动回滚，直接返回

  // 计算健康状态
  const daysLeft = getSafeRolloutDaysLeft({ ... });
  const healthSettings = getHealthSettings(...);
  const safeRolloutStatus = getSafeRolloutResultStatus({ ... });

  let status: SafeRolloutStatus = updatedSafeRollout.status;
  
  // 关键判定：状态为 "rollback-now" 时执行回滚
  if (safeRolloutStatus?.status && "rollback-now" === safeRolloutStatus.status) {
    status = "rolled-back";
    
    // 1. 创建修订版本
    const revision = await createRevision({
      context,
      feature,
      user: context.auditUser,
      environments: [updatedSafeRollout.environment],
      baseVersion: feature.version,
      org: context.org,
    });
    
    // 2. 编辑 Feature Rule，更新状态为 rolled-back
    await editFeatureRule(
      context,
      feature,
      revision,
      ruleId,
      { status },
      context.auditUser,
      false,
      updatedSafeRollout.environment,
    );
    
    // 3. 合并修订
    const mergeResult = autoMerge(live, base, revision, [...], {});
    
    // 4. 发布修订（立即生效）
    await publishRevision(
      context,
      feature,
      revision,
      mergeResult.result,
      "auto-publish status change",
    );
  }
  return status;
}
```

### 9.2 执行流程详解

```
快照更新完成 (afterUpdateOne hook)
        ↓
调用 checkAndRollbackSafeRollout()
        ↓
┌─────────────────────────────────────┐
│ 前置条件检查                        │
│ 1. safeRollout.status === "running"? │
│ 2. autoRollback === true?           │
└─────────────────────────────────────┘
        ↓ 否
      返回原状态
        ↓ 是
计算 daysLeft + healthSettings + safeRolloutStatus
        ↓
┌─────────────────────────────────────┐
│ 状态判定: status === "rollback-now"? │
└─────────────────────────────────────┘
        ↓ 否
      返回 "running"
        ↓ 是
┌─────────────────────────────────────┐
│ 执行回滚操作                        │
│ 1. createRevision()                 │
│ 2. editFeatureRule({status:"rolled-back"}) │
│ 3. autoMerge()                      │
│ 4. publishRevision()                │
└─────────────────────────────────────┘
        ↓
返回 "rolled-back"
```

---

## 10. 健康异常与回滚决策的判定优先级

### 10.1 判定优先级顺序

在 `getSafeRolloutResultStatus` 函数 (`decisionCriteria.ts:597-717`) 中，判定顺序如下：

```typescript
export function getSafeRolloutResultStatus({ ... }): ... {
  // 1. 无数据分支 - 最先检查
  if (!healthSummary?.totalUsers && hoursRunning > 24) {
    return { status: "no-data" };
  }
  
  // 2. 健康异常检测 (SRM + 多重暴露)
  else if (healthSummary?.totalUsers) {
    // SRM 检测
    const srmHealthData = getSRMHealthData({ ... });
    if (srmHealthData === "unhealthy") {
      unhealthyData.srm = true;
    }
    
    // 多重暴露检测
    const multipleExposuresHealthData = getMultipleExposureHealthData({ ... });
    if (multipleExposuresHealthData.status === "unhealthy") {
      unhealthyData.multipleExposures = { ... };
    }
  }
  
  // 3. 护栏度量回滚决策
  const decisionStatus = resultsStatus
    ? getDecisionFrameworkStatus({ ... })
    : undefined;

  // ⚠️ 优先级 1: 健康异常最高优先级
  if (unhealthyData.srm || unhealthyData.multipleExposures) {
    return {
      status: "unhealthy",
      unhealthyData,
    };
  }

  // ⚠️ 优先级 2: 护栏度量回滚次之
  if (decisionStatus?.status === "rollback-now") {
    return {
      status: "rollback-now",
      variations: decisionStatus.variations,
      ...
    };
  }

  // ⚠️ 优先级 3: 剩余天数判断
  if (daysLeft > 0) {
    return {
      status: "days-left",
      daysLeft,
    };
  }

  // ⚠️ 优先级 4: 监测期结束，建议发布
  if (daysLeft <= 0 && resultsStatus) {
    return {
      status: "ship-now",
      ...
    };
  }
}
```

### 10.2 优先级总结

| 优先级 | 状态 | 触发条件 | 说明 |
|-------|------|---------|------|
| **1 (最高)** | `no-data` | 运行 > 24 小时且无任何用户数据 | 数据缺失，无法做决策 |
| **2** | `unhealthy` | SRM 异常 或 多重暴露 > 1% | 样本质量问题，最高优先级告警 |
| **3** | `rollback-now` | 任意护栏度量显著下降 (status="lost") | 业务指标恶化，触发自动回滚 |
| **4** | `days-left` | 监测期未结束 (daysLeft > 0) | 继续观察 |
| **5 (最低)** | `ship-now` | 监测期结束且无异常 | 建议全量发布 |

**关键结论**：
- **健康异常 (unhealthy) 优先级高于护栏回滚 (rollback-now)**
- 当存在 SRM 或多重暴露问题时，即使护栏度量下降，也只会返回 `unhealthy` 状态，不会触发自动回滚
- `unhealthy` 状态仅标记问题，需要人工介入，不会触发自动回滚
- 只有明确的 `rollback-now` 状态才会执行自动回滚

---

## 11. 无数据与剩余天数分支条件

### 11.1 无数据分支 (no-data)

**触发条件** (`decisionCriteria.ts:616-619`)：
```typescript
if (!healthSummary?.totalUsers && hoursRunning > 24) {
  return {
    status: "no-data",
  };
}
```

**条件说明**：
1. `!healthSummary?.totalUsers` - 总用户数为 0 或 undefined
2. `hoursRunning > 24` - Safe Rollout 已运行超过 24 小时

**场景**：
- 数据集成配置错误
- 曝光事件未正确上报
- 流量完全没有进入实验

### 11.2 剩余天数分支 (days-left)

**触发条件** (`decisionCriteria.ts:696-700`)：
```typescript
if (daysLeft > 0) {
  return {
    status: "days-left",
    daysLeft,
  };
}
```

**daysLeft 计算** (`decisionCriteria.ts:574-595`)：
```typescript
export function getSafeRolloutDaysLeft({
  safeRollout,
  snapshotWithResults,
}: { ... }) {
  const startDate = safeRollout.startedAt
    ? new Date(safeRollout.startedAt)
    : new Date();
  const endDate = addDays(startDate, safeRollout?.maxDuration?.amount);
  const latestSnapshotDate = snapshotWithResults?.runStarted
    ? new Date(snapshotWithResults?.runStarted)
    : null;

  const daysLeft = latestSnapshotDate
    ? differenceInMinutes(endDate, latestSnapshotDate) / 1440
    : safeRollout?.maxDuration?.amount;

  return daysLeft;
}
```

**计算逻辑**：
- 起始日期：`safeRollout.startedAt`（Safe Rollout 启动时间）
- 结束日期：`startDate + maxDuration.amount`（监测期时长）
- 已运行时长：基于最新快照时间计算
- 剩余天数：`(endDate - latestSnapshotDate) / 1440`（分钟转天数）

### 11.3 发布分支 (ship-now)

**触发条件** (`decisionCriteria.ts:703-716`)：
```typescript
if (daysLeft <= 0 && resultsStatus) {
  return {
    status: "ship-now",
    variations: [
      {
        variationId: "1",
        decidingRule: null,
      },
    ],
    sequentialUsed: true,
    powerReached: false,
  };
}
```

**条件说明**：
1. `daysLeft <= 0` - 监测期已结束
2. `resultsStatus` 存在 - 有分析结果数据

---

## 12. 整体配合流程

### 12.1 Safe Rollout 健康评估流程

**入口函数** `getSafeRolloutResultStatus` (`decisionCriteria.ts:597-717`)

```
流程图示:

1. 数据收集阶段
   ↓
2. 无数据检查 (优先级最高)
   ├─ 条件: 运行 > 24小时 且 totalUsers = 0
   └─ 结果: status: "no-data"
   ↓ 有数据
3. SRM 检测
   ├─ 流量检查: totalUsers >= 2 * 8 (每个变体至少8个用户)
   ├─ SRM p-value 计算
   └─ 判断: srm < 0.001 ? 标记 unhealthyData.srm = true : 跳过
   ↓
4. 多重暴露检测
   ├─ 计算: multipleExposures / totalUsers
   ├─ 流量检查: totalUsers >= 10
   └─ 判断: 比例 >= 0.01 ? 标记 unhealthyData.multipleExposures = true : 跳过
   ↓
5. 护栏度量评估
   ├─ 对每个护栏度量进行统计检验
   ├─ 使用顺序检验 (Sequential Testing)
   └─ 检查度量状态是否为 "lost" (显著下降)
   ↓
6. 决策阶段 (按优先级返回)
   ├─ 优先级1: unhealthyData 有值 → status: unhealthy (仅告警，不自动回滚)
   ├─ 优先级2: 护栏度量显著下降 → status: rollback-now (触发自动回滚)
   ├─ 优先级3: 监测期未结束 → status: days-left
   └─ 优先级4: 监测期结束且无异常 → status: ship-now
```

### 12.2 自动回滚触发链

```
快照分析完成
        ↓
afterUpdateOne hook 触发
        ↓
checkAndRollbackSafeRollout()
        ├─ 检查: status === "running"?
        ├─ 检查: autoRollback === true?
        └─ 调用 getSafeRolloutResultStatus()
        ↓
status === "rollback-now"?
        ↓ 否
      不执行操作
        ↓ 是
┌─────────────────────────────┐
│ 执行自动回滚                 │
│ 1. createRevision()         │
│ 2. editFeatureRule()        │
│    - 更新 rule.status       │
│ 3. autoMerge()              │
│ 4. publishRevision()        │
└─────────────────────────────┘
        ↓
Safe Rollout 状态变为 "rolled-back"
```

---

## 13. 关键文件索引

| 文件路径 | 核心功能 |
|---------|---------|
| `packages/shared/types/metric.d.ts` | Metric 类型定义，Legacy/Fact Metric 分母配置差异 |
| `packages/shared/src/constants.ts` | 阈值常量定义 |
| `packages/shared/src/health/health.ts` | SRM 和多重暴露健康检测 |
| `packages/shared/src/enterprise/decision-criteria/decisionCriteria.ts` | 决策框架与护栏评估逻辑 |
| `packages/shared/src/experiments/experiments.ts` | 度量类型判断 (isRatioMetric, isFunnelMetric) |
| `packages/back-end/src/services/safeRolloutSnapshots.ts` | Safe Rollout 快照服务，**单层**分母处理 |
| `packages/back-end/src/util/sql.ts:186-199` | `expandDenominatorMetrics` **递归**展开函数，含环路防护 |
| `packages/back-end/src/queryRunners/ExperimentResultsQueryRunner.ts` | 实验结果查询，**递归**分母展开 |
| `packages/back-end/src/queryRunners/PopulationDataQueryRunner.ts` | 人口数据查询，**递归**分母展开 |
| `packages/back-end/src/integrations/sql/queries/experiment-metric-query.ts` | Legacy Metric SQL 查询构建，分母使用逻辑 |
| `packages/back-end/src/services/experimentQueries/experimentQueries.ts` | Fact Metric 分组逻辑 |
| `packages/back-end/src/enterprise/saferollouts/safeRolloutUtils.ts` | 自动回滚实际执行逻辑 |
| `packages/back-end/src/models/SafeRolloutSnapshotModel.ts` | 快照模型，包含自动回滚触发 hook |
| `packages/back-end/src/services/reports.ts` | 报表服务，分母处理链路 |
| `packages/back-end/test/util/sql.test.ts:353-375` | `expandDenominatorMetrics` 测试用例 |
| `packages/front-end/components/Features/RuleModal/SafeRolloutFields.tsx` | Safe Rollout 配置表单 |
