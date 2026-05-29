# 事实表与实验分配关联代码执行路径分析

## 一、整体执行流程概览

事实表与实验分配的关联主要通过 SQL CTE (Common Table Expression) 链式调用来实现，核心执行路径如下：

```
实验分配数据提取 → 用户级别去重 → 身份标识键对齐 → 事实表数据提取 → 时间窗口过滤 → 用户级别聚合 → 最终统计
```

---

## 二、核心代码文件与调用关系

### 2.1 入口文件

| 文件路径 | 主要功能 |
|---------|---------|
| `packages/back-end/src/integrations/sql/queries/experiment-fact-metrics-query.ts | 主查询构建入口 |
| `packages/back-end/src/integrations/sql/queries/experiment-units-query.ts | 实验单元（用户+分组）提取 |
| `packages/back-end/src/integrations/sql/ctes/fact-metric-cte.ts | 事实表CTE构建 |
| `packages/back-end/src/integrations/sql/ctes/identities-cte.ts | 身份标识键对齐 |
| `packages/back-end/src/integrations/sql/ctes/experiment-fact-metric-statistics-cte.ts | 最终统计聚合 |

---

## 三、数据提取阶段

### 3.1 实验分配数据提取

**文件**: `experiment-units-query.ts:78-242`

**核心逻辑**:

1. **原始曝光数据提取** (`__rawExperiment` CTE)
   - 从曝光查询模板中编译SQL，提取实验ID、用户ID、变体ID、时间戳
   - 应用实验ID、日期范围过滤

2. **曝光数据过滤** (`__experimentExposures` CTE)
```sql
SELECT
  e.${baseIdType} as ${baseIdType}
  , CAST(e.variation_id AS STRING) as variation
  , e.timestamp as timestamp
FROM __rawExperiment e
WHERE
  e.experiment_id = '${experimentId}'
  AND e.timestamp >= ${startDate}
  AND e.timestamp <= ${endDate}
```

### 3.2 事实表数据提取

**文件**: `fact-metric-cte.ts:20-216`

**核心逻辑**:

1. **事实表SQL模板编译**
   - 支持 `compileSqlTemplate` 处理模板变量（startDate, endDate, experimentId 等）

2. **指标列投影**
   - 分子列：`m${index}_value
   - 分母列（比率指标）：`m${index}_denominator`
   - KLL合并指标：额外投影 `n_events` 列

3. **行级别过滤**
```typescript
// 应用指标过滤器（如事件名称、属性过滤等）
const filters = getColumnRefWhereClause({...})
const column = filters.length > 0
  ? `CASE WHEN (${filters.join(" AND ")}) THEN ${value} ELSE NULL END`
  : value;
```

---

## 四、键对齐（身份标识连接）

### 4.1 键对齐核心逻辑

**文件**: `identities-cte.ts:7-60`

**核心功能**: 处理不同ID类型之间的映射（如 user_id ↔ anonymous_id）

### 4.2 键对齐执行流程

```
1. 确定基准ID类型（baseIdType）选择
   ↓
2. 识别需要连接的ID类型（joinsRequired）
   ↓
3. 为每个需要连接的ID类型生成CTE
   ↓
4. 返回 idJoinMap 供后续CTE使用
```

**代码实现** (`fact-metric-cte.ts:52-68`):

```typescript
if (userIdTypes.includes(baseIdType)) {
  userIdCol = baseIdType;  // 直接使用基准ID
} else {
  // 需要通过身份表连接
  for (const userIdType of userIdTypes) {
    if (userIdType in idJoinMap) {
      join = `JOIN ${idJoinMap[userIdType]} i ON (i.${userIdType} = m.${userIdType})`;
      userIdCol = `i.${baseIdType}`;
      break;
    }
  }
}
```

### 4.3 身份查询实现

**文件**: `identities-query.ts:5-74`

**去重逻辑**:
```sql
SELECT ${id1}, ${id2}
FROM (${identityJoinQuery) i
GROUP BY ${id1}, ${id2}  -- 确保ID映射去重
```

---

## 五、去重逻辑分析

### 5.1 用户级别去重（实验分配侧）

**文件**: `experiment-units-query.ts:175-242`

**核心CTE**: `__experimentUnits`

**去重策略**:

1. **按用户ID分组**
   ```sql
   GROUP BY e.${baseIdType}
   ```

2. **变体去重规则** (`first-variation-value-per-unit.ts:4-15`):
   ```typescript
   // 方案1：多变体标记为 __multiple__
   dialect.ifElse(
     "count(distinct e.variation) > 1",
     "'__multiple__'",
     "max(e.variation)"
   )
   
   // 方案2：取首次曝光的变体（Bandit模式）
   SUBSTRING(
     MIN(CONCAT(
       SUBSTRING(FORMAT_DATETIME(e.timestamp), 1, 19),
       e.variation
     )),
     20, 99999
   )
   ```

3. **首次曝光时间**
   ```sql
   MIN(e.timestamp) AS first_exposure_timestamp
   ```

### 5.2 最终用户集合

**文件**: `experiment-fact-metrics-query.ts:232-265`

**CTE**: `__distinctUsers`

**功能**:
- 从实验单元中提取最终用户列表
- 应用激活指标过滤（如仅保留已激活用户）
- 应用时间窗口过滤（skipPartialData）

```sql
SELECT
  ${baseIdType}
  , variation
  , first_exposure_timestamp AS timestamp
  , DATE_TRUNC('day', first_exposure_timestamp) AS first_exposure_date
FROM __experimentUnits
WHERE
  -- 激活用户过滤
  -- 部分数据跳过过滤
```

---

## 六、事实表与实验分配连接逻辑

### 6.1 连接策略

**文件**: `experiment-fact-metrics-query.ts:521-617`

**CTE**: `__userMetricJoin${suffix}`

**连接方式**: **LEFT JOIN**（左连接）

```sql
SELECT
  d.variation AS variation
  , d.timestamp AS timestamp
  , d.${baseIdType} AS ${baseIdType}
  -- 事实表指标列
FROM __distinctUsers d
LEFT JOIN __factTable${suffix} m
ON m.${baseIdType} = d.${baseIdType}
```

**关键点**:
- **左表**: `__distinctUsers`（实验用户集合）
- **右表**: `__factTable${suffix}`（事实表数据）
- **连接键**: 用户ID（`${baseIdType}`

### 6.2 时间窗口过滤

**文件**: `add-case-when-time-filter.ts:7-40`

**功能**: 确保事实表事件发生在实验曝光之后的有效时间窗口内

```typescript
dialect.ifElse(
  `${conversionWindowClause  -- 曝光时间 ≤ 事件时间 ≤ 曝光时间 + 窗口
  AND ${metricValue != 0  -- 可选：忽略零值
  `,
  columnValue,
  NULL
)
```

**转换窗口子句** (`conversion-window-clause.ts`):
- `attributionModel = "first Exposure"：使用指标定义的窗口
- `attributionModel = "experimentDuration"`：使用实验持续时间作为窗口
- `attributionModel = "lookbackOverride"`：使用自定义回溯窗口

---

## 七、用户级别聚合

### 7.1 每用户聚合（__userMetricAgg）

**文件**: `experiment-fact-metrics-query.ts:295-412`

**核心逻辑**:

1. **按用户+变体+维度分组**
   ```sql
   GROUP BY umj.variation, umj.${dimensionCols}, umj.${baseIdType}
   ```

2. **聚合函数应用**:
   - 求和：`SUM(metric_value)`
   - 计数：`COUNT(rows)` 或 `SUM(n_events)`
   - KLL合并：`KLL_MERGE_PARTIAL(sketch)`

### 7.2 多事实表支持

**文件**: `fact-tables-for-metrics.ts:8-84`

**限制**: 最多支持2个事实表同时查询

**处理方式**:
- 每个事实表生成独立的 `__factTable${suffix}` CTE
- 每个事实表生成独立的 `__userMetricAgg${suffix}` CTE
- 最终统计阶段通过 `baseIdType` 进行连接

```sql
-- 最终统计时多表连接
FROM __userMetricAgg m
LEFT JOIN __userMetricAgg1 m1
ON m1.user_id = m.user_id
```

---

## 八、最终统计聚合

### 8.1 变体级别统计

**文件**: `experiment-fact-metric-statistics-cte.ts:12-214`

**CTE**: 最终SELECT（无命名CTE名称，直接作为最终输出）

**聚合维度**:
- variation（变体）
- dimension columns（维度列，如有）

**聚合指标**:
- `COUNT(*) AS users`（用户数）
- `SUM(metric_value)`（指标值总和）
- `SUM(POWER(metric_value), 2)`（平方和，用于方差计算）
- 比率指标：分子、分母、乘积项
- 回归调整：协变量相关统计
- 分位数指标：分位数值、置信区间边界

---

## 九、完整CTE依赖关系图

```
__identities_* (键对齐表)
    ↓
__rawExperiment (原始曝光)
    ↓
__experimentExposures (过滤后曝光)
    ↓
__experimentUnits (用户级别去重)
    ↓
__distinctUsers (最终用户集合)
    ↓
__factTable0, __factTable1 (事实表数据)
    ↓
__userMetricJoin0, __userMetricJoin1 (左连接结果)
    ↓
__eventQuantileMetric* (事件分位数，如有)
    ↓
__userMetricAggBase* (KLL中间聚合，如有)
    ↓
__userMetricAgg0, __userMetricAgg1 (每用户聚合)
    ↓
__capValue* (百分位盖帽，如有)
    ↓
最终统计聚合（按变体+维度）
```

---

## 十、关键设计决策

### 10.1 为什么使用LEFT JOIN而不是INNER JOIN

**原因**:
- 保留所有实验用户，即使没有事实表事件
- 确保分母（用户数）统计准确
- 正确计算转化率（有事件用户 / 总用户）

### 10.2 为什么在用户级别先去重再连接

**优点**:
- 减少连接的数据量（1:N 变 1:1）
- 确保每个用户只保留正确的变体分配
- 避免重复计算曝光时间窗口

### 10.3 为什么支持多事实表

**场景**:
- 比率指标的分子和分母来自不同事实表
- 多个指标来自不同事实表（如：购买事件 + 登录事件）

**限制**: 最多2个事实表（当前实现）

---

## 十一、代码优化点观察

1. **KLL合并优化** (`experiment-fact-metrics-query.ts:274-294`):
   - KLL可合并sketch减少数据扫描次数
   - 当所有事件分位数指标都是KLL类型时，从用户级别sketch合并而不是重新扫描事件表

2. **指标分组优化** (`experimentQueries.ts:224-247`):
   - 按事实表ID分组指标
   - 分位数指标单独分组
   - 跨表比率指标单独处理
