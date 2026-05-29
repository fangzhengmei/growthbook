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
| `packages/back-end/src/integrations/sql/queries/experiment-fact-metrics-query.ts` | 主查询构建入口 |
| `packages/back-end/src/integrations/sql/queries/experiment-units-query.ts` | 实验单元（用户+分组）提取 |
| `packages/back-end/src/integrations/sql/ctes/fact-metric-cte.ts` | 事实表CTE构建 |
| `packages/back-end/src/integrations/sql/ctes/identities-cte.ts` | 身份标识键对齐 |
| `packages/back-end/src/integrations/sql/ctes/experiment-fact-metric-statistics-cte.ts` | 最终统计聚合 |

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
   - 分子列：`m${index}_value`
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

### 4.2 基准ID类型选择算法

**文件**: `util/sql.ts:11-56`

```typescript
export function getBaseIdTypeAndJoins(
  objects: string[][],
  forcedBaseIdType?: string,
) {
  // 1. 统计每种ID类型的使用频率
  const counts: Record<string, number> = {};
  objects.forEach((types) => {
    types.forEach((type) => {
      if (!type) return;
      counts[type] = (counts[type] || 0) + 1;
    });
  });

  // 2. 按频率排序，选择使用最频繁的ID作为基准ID
  // （除非指定了 forcedBaseIdType）
  const baseIdType = forcedBaseIdType || mostFrequentIdType;

  // 3. 确定需要连接的ID类型（最小化连接次数）
  const joinsRequired: Set<string> = new Set();
  sorted.forEach((types) => {
    if (types.includes(baseIdType)) return;
    if (types.filter(type => joinsRequired.has(type)).length > 0) return;
    joinsRequired.add(mostFrequentIdTypeInThisObject);
  });

  return { baseIdType, joinsRequired };
}
```

### 4.3 键对齐执行流程

```
1. 确定基准ID类型（baseIdType）选择
   ↓ 输入：[[曝光ID类型], [事实表ID类型], [维度ID类型], [segment ID类型], [激活指标ID类型]]
   ↓ 算法：选择使用最频繁的ID类型，或使用强制指定的ID类型
   ↓
2. 识别需要连接的ID类型（joinsRequired）
   ↓ 对每个对象检查：是否支持基准ID？是否已支持某个连接ID？
   ↓
3. 为每个需要连接的ID类型生成CTE
   ↓ CTE命名：`__identities_${idType}`
   ↓
4. 返回 idJoinMap 供后续CTE使用
   ↓ 格式：{ [userIdType]: tableName }
```

### 4.4 键对齐调用时机（两种场景）

#### 场景A：unitsSource === "exposureQuery"（最常见）

**文件**: `experiment-fact-metrics-query.ts:105-126`

```typescript
// 1. 收集所有需要的ID类型
const idTypeObjects = [
  [userIdType],                          // 曝光查询ID类型
  factTable.userIdTypes || [],           // 事实表ID类型
  ...unitDimensions.map(d => [d.dimension.userIdType || "user_id"]),  // 维度ID类型
  segment ? [segment.userIdType || "user_id"] : [],                    // segment ID类型
  activationMetric ? getUserIdTypes(activationMetric, factTableMap) : [],  // 激活指标ID类型
];

// 2. 生成身份对齐CTE（包含在 idJoinSQL 中）
const { baseIdType, idJoinMap, idJoinSQL } = getIdentitiesCTE(
  dialect,
  datasource.settings,
  {
    objects: idTypeObjects,
    forcedBaseIdType: userIdType,  // 强制使用曝光查询的ID类型作为基准
    ...
  },
);

// 3. 调用实验单元查询（不重复生成身份CTE）
getExperimentUnitsQuery(dialect, datasource, {
  ...params,
  includeIdJoins: false,  // 关键：不重复生成身份CTE
})
```

#### 场景B：unitsSource === "exposureTable" 或 "otherQuery"

- 跳过实验单元查询的调用
- 直接使用预计算的单元表或自定义SQL
- 身份对齐CTE仍然在主查询中生成

### 4.5 代码实现（事实表侧键对齐）

**文件**: `fact-metric-cte.ts:52-68`

```typescript
if (userIdTypes.includes(baseIdType)) {
  userIdCol = baseIdType;  // 直接使用基准ID，无需连接
} else {
  // 需要通过身份表连接
  for (const userIdType of userIdTypes) {
    if (userIdType in idJoinMap) {
      // JOIN条件：身份表ID类型列 = 事实表ID类型列
      join = `JOIN ${idJoinMap[userIdType]} i ON (i.${userIdType} = m.${userIdType})`;
      userIdCol = `i.${baseIdType}`;  // 从连接后的身份表获取基准ID
      break;
    }
  }
}
```

### 4.6 代码实现（激活指标/Metric侧键对齐）

**文件**: `ctes/metric-cte.ts:76-91`

```typescript
if (userIdTypes.includes(baseIdType)) {
  userIdCol = queryFormat === "builder" ? userIdCol : baseIdType;
} else if (userIdTypes.length > 0) {
  for (const userIdType of userIdTypes) {
    if (userIdType in idJoinMap) {
      const metricUserIdCol = queryFormat === "builder"
        ? cols.userIds[userIdType]
        : `m.${userIdType}`;
      // JOIN条件：身份表.ID类型 = 指标表.ID类型列
      join = `JOIN ${idJoinMap[userIdType]} i ON (i.${userIdType} = ${metricUserIdCol})`;
      userIdCol = `i.${baseIdType}`;
      break;
    }
  }
}
```

### 4.7 身份查询实现（含去重）

**文件**: `identities-query.ts:5-74`

```sql
SELECT ${id1}, ${id2}
FROM (${identityJoinQuery}) i
GROUP BY ${id1}, ${id2}  -- 关键：确保ID映射关系去重，避免笛卡尔积
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

### 5.2 最终用户集合（__distinctUsers）

**文件**: `experiment-fact-metrics-query.ts:232-265`

**输入来源**（三选一）:
- `unitsSource === "exposureQuery"` → 来自 `__experimentUnits` CTE
- `unitsSource === "exposureTable"` → 来自 `params.unitsTableFullName` 预计算表
- `unitsSource === "otherQuery"` → 来自 `params.unitsSql` 自定义SQL

**功能**:
- 从实验单元中提取最终用户列表
- 应用激活指标过滤（如仅保留已激活用户）
- 应用时间窗口过滤（skipPartialData）

```sql
SELECT
  ${baseIdType}
  , variation
  , ${timestampColumn} AS timestamp  -- first_exposure_timestamp 或 first_activation_timestamp
  , DATE_TRUNC('day', first_exposure_timestamp) AS first_exposure_date
FROM ${sourceTable}
WHERE
  -- 激活用户过滤：first_activation_timestamp IS NOT NULL
  -- 部分数据跳过过滤：timestamp <= experimentEndDate
```

---

## 六、事实表与实验分配连接逻辑

### 6.1 连接策略

**文件**: `experiment-fact-metrics-query.ts:521-617`

**CTE**: `__userMetricJoin${suffix}`

**连接方式**: **LEFT JOIN**（左连接）

**实际SQL代码**:
```sql
SELECT
  d.variation AS variation
  , d.timestamp AS timestamp  -- 来自 __distinctUsers 的曝光时间
  , d.${baseIdType} AS ${baseIdType}
  -- 事实表指标列（经过时间窗口过滤）
  , CASE WHEN (时间窗口条件) THEN m.m0_value ELSE NULL END as m0_value
  , CASE WHEN (时间窗口条件) THEN m.m0_denominator ELSE NULL END as m0_denominator
  -- ... 更多指标列
FROM __distinctUsers d
LEFT JOIN __factTable${suffix} m
ON m.${baseIdType} = d.${baseIdType}  -- 连接键：基准ID类型
```

**关键点**:
- **左表**: `__distinctUsers`（实验用户集合，已去重，每个用户一行）
- **右表**: `__factTable${suffix}`（事实表数据，每个事件一行）
- **连接键**: `${baseIdType}`（用户ID，经过身份对齐后统一）
- **连接类型**: LEFT JOIN（保留所有实验用户，即使没有事实表事件）

### 6.2 时间窗口过滤

**文件**: `add-case-when-time-filter.ts:7-40`

**调用位置**: 在 `__userMetricJoin` CTE 的 SELECT 子句中，对每个指标列应用

**功能**: 确保事实表事件发生在实验曝光之后的有效时间窗口内

```typescript
dialect.ifElse(
  `${conversionWindowClause}  -- 曝光时间 ≤ 事件时间 ≤ 曝光时间 + 窗口
  ${metricQuantileSettings?.ignoreZeros && metricQuantileSettings?.type === "event" 
    ? `AND ${col} != 0` 
    : ""}`,
  `${col}`,    // 满足条件：保留原值
  `NULL`       // 不满足条件：设为NULL（后续聚合时自动忽略）
)
```

### 6.3 转换窗口子句详细实现

**文件**: `conversion-window-clause.ts:10-54`

**输入参数**:
- `baseCol`: 曝光时间列（d.timestamp）
- `metricCol`: 事实表时间列（m.timestamp）
- `metric`: 指标定义（含窗口设置）
- `endDate`: 实验结束日期
- `overrideConversionWindows`: 是否覆盖转换窗口

**窗口类型**:

1. **conversion 窗口（默认）**:
   ```sql
   m.timestamp >= d.timestamp + delay_hours
   AND m.timestamp <= d.timestamp + (delay_hours + window_hours)
   -- 注意：可以超过实验结束日期
   ```

2. **experimentDuration 窗口**:
   ```sql
   m.timestamp >= d.timestamp + delay_hours
   AND m.timestamp <= experiment_end_date
   -- 必须在实验结束日期之前
   ```

3. **lookback 窗口**:
   ```sql
   m.timestamp >= d.timestamp + delay_hours
   AND m.timestamp <= experiment_end_date
   AND m.timestamp + window_hours >= experiment_end_date
   -- 只统计实验结束前X小时内的事件
   ```

---

## 七、用户级别聚合

### 7.1 每用户聚合（__userMetricAgg）

**文件**: `experiment-fact-metrics-query.ts:295-412`

**输入来源**: `__userMetricJoin`（左连接结果）

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
- 每个事实表生成独立的 `__userMetricJoin${suffix}` CTE
- 每个事实表生成独立的 `__userMetricAgg${suffix}` CTE
- 最终统计阶段通过 `baseIdType` 进行连接

```sql
-- 最终统计时多表连接
FROM __userMetricAgg m  -- 第一个事实表的用户聚合结果
LEFT JOIN __userMetricAgg1 m1  -- 第二个事实表的用户聚合结果
ON m1.${baseIdType} = m.${baseIdType}  -- 连接键：基准ID
```

---

## 八、最终统计聚合

### 8.1 变体级别统计

**文件**: `experiment-fact-metric-statistics-cte.ts:12-214`

**输入来源**: `__userMetricAgg`（每用户聚合结果）

**聚合维度**:
- variation（变体）
- dimension columns（维度列，如有）

**聚合指标**:
- `COUNT(*) AS users`（用户数）
- `SUM(metric_value)`（指标值总和）
- `SUM(POWER(metric_value, 2))`（平方和，用于方差计算）
- 比率指标：分子、分母、乘积项
- 回归调整：协变量相关统计
- 分位数指标：分位数值、置信区间边界

---

## 九、完整CTE依赖关系图（精确版）

### 9.1 unitsSource === "exposureQuery" 场景（最常见）

```
阶段0：身份键对齐（最先执行）
├─ __identities_${idType}* （身份映射表，已去重）
│   输入：getIdentitiesQuery
│   输出：${baseIdType}, ${otherIdType}
│   连接键：GROUP BY 去重
│
阶段1：实验分配数据提取
├─ __rawExperiment （原始曝光数据）
│   输入：exposureQuery.query（编译后）
│   输出：experiment_id, user_id, variation_id, timestamp
│
├─ __experimentExposures （过滤后曝光数据）
│   输入：__rawExperiment
│   过滤条件：experiment_id, 日期范围, queryFilter
│   输出：${baseIdType}, variation, timestamp
│   └── 依赖身份对齐：SELECT e.${baseIdType}（如果曝光表ID类型不是基准ID，需要JOIN）
│
├─ [可选] __activationMetric （激活指标数据）
│   输入：事实表或自定义SQL
│   输出：${baseIdType}, value, timestamp
│   └── 依赖身份对齐：JOIN __identities_xxx
│
├─ [可选] __segment （用户段数据）
│   输入：segment定义SQL
│   输出：${baseIdType}, date
│   └── 依赖身份对齐：JOIN __identities_xxx
│
├─ [可选] __dim_unit_${dimensionId}* （单位维度数据）
│   输入：dimension定义SQL
│   输出：${baseIdType}, dimension_value
│   └── 依赖身份对齐：JOIN __identities_xxx
│
└─ __experimentUnits （用户级别去重）
   输入：__experimentExposures + [__activationMetric] + [__segment] + [__dim_unit_*]
   分组键：e.${baseIdType}
   聚合：
     - variation: 去重策略（__multiple__ 或 首次曝光）
     - first_exposure_timestamp: MIN(e.timestamp)
     - [first_activation_timestamp: MIN(a.timestamp)]
     - 维度值：每个维度的聚合值
   输出：${baseIdType}, variation, first_exposure_timestamp, ...
   │
阶段2：最终用户集合
└─ __distinctUsers
   输入：__experimentUnits （或 exposureTable 或 otherQuery）
   过滤条件：
     - 激活用户：first_activation_timestamp IS NOT NULL
     - 部分数据跳过：timestamp <= endDate
   输出：${baseIdType}, variation, timestamp, first_exposure_date, ...
   │
阶段3：事实表数据提取（每个事实表独立执行）
└─ __factTable${suffix}*
   输入：factTable.sql（编译后）
   过滤条件：
     - 日期范围：startDate <= timestamp <= endDate
     - 指标过滤器：WHERE子句中的条件
   输出：${baseIdType}, timestamp, m${index}_value, [m${index}_denominator]
   └── 依赖身份对齐：
       - 如果事实表支持基准ID：直接 SELECT ${baseIdType}
       - 否则：JOIN __identities_xxx i ON i.${idType} = m.${idType}
   │
阶段4：事实表与实验分配连接（每个事实表独立执行）
└─ __userMetricJoin${suffix}*
   输入：__distinctUsers d LEFT JOIN __factTable${suffix} m
   连接键：m.${baseIdType} = d.${baseIdType}
   SELECT 处理：
     - 保留 d.*（实验分配数据）
     - 对每个 m.* 列应用 addCaseWhenTimeFilter（时间窗口过滤）
     - [可选] 回归调整协变数列
   输出：variation, timestamp, ${baseIdType}, filtered_metric_columns
   │
阶段5：事件分位数计算（如有需要）
└─ [可选] __eventQuantileMetric${suffix}*
   输入：__userMetricJoin${suffix} 或 __userMetricAggBase${suffix}
   分组键：variation, dimension_cols
   输出：分位数网格数据
   │
阶段6：每用户聚合（每个事实表独立执行）
├─ [可选] __userMetricAggBase${suffix}* （KLL中间聚合）
│   输入：__userMetricJoin${suffix}
│   分组键：variation, dimension_cols, ${baseIdType}
│   输出：每用户KLL sketch
│
└─ __userMetricAgg${suffix}*
   输入：__userMetricJoin${suffix} 或 __userMetricAggBase${suffix}
   分组键：variation, dimension_cols, ${baseIdType}
   聚合：
     - SUM(value)
     - COUNT(rows) 或 SUM(n_events)
     - [KLL特殊处理]
   输出：每用户聚合结果
   │
阶段7：百分位盖帽（如有需要）
└─ [可选] __capValue${suffix}*
   输入：__userMetricAgg${suffix}
   输出：盖帽阈值
   │
阶段8：最终统计聚合
└─ 最终SELECT（无CTE名称）
   输入：
     - __userMetricAgg m
     - [LEFT JOIN __userMetricAgg1 m1 ON m1.${baseIdType} = m.${baseIdType}]（多事实表）
     - [LEFT JOIN __eventQuantileMetric qm ON ...]（分位数指标）
     - [CROSS JOIN __capValue cap]（盖帽指标）
   分组键：m.variation, m.dimension_cols
   聚合：所有统计指标
   输出：最终实验结果
```

### 9.2 关键依赖关系总结表

| 下游CTE | 上游依赖 | 连接键/依赖方式 |
|---------|---------|----------------|
| `__experimentExposures` | `__rawExperiment` | 直接FROM |
| `__experimentUnits` | `__experimentExposures`, `__activationMetric`, `__segment`, `__dim_unit_*` | LEFT JOIN ON ${baseIdType} |
| `__distinctUsers` | `__experimentUnits`（或exposureTable/otherQuery） | 直接FROM + WHERE过滤 |
| `__factTable${suffix}` | 身份CTE（如需要） | JOIN ON idType列 |
| `__userMetricJoin${suffix}` | `__distinctUsers`, `__factTable${suffix}` | LEFT JOIN ON ${baseIdType} |
| `__userMetricAgg${suffix}` | `__userMetricJoin${suffix}` | GROUP BY（用户级聚合） |
| 最终统计 | `__userMetricAgg` + 其他表 | GROUP BY（变体级聚合） |

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

### 10.3 为什么身份对齐在最顶层执行一次

**原因**:
- 避免在每个子查询中重复生成身份CTE
- 确保所有表使用相同的基准ID类型
- 通过 `includeIdJoins: false` 参数控制实验单元查询不重复生成

### 10.4 为什么支持多事实表

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

3. **时间窗口过滤位置优化**:
   - 过滤放在JOIN后的SELECT子句中（CASE WHEN），而不是WHERE子句
   - 优点：保留所有用户行，不满足条件的设为NULL（后续聚合自动忽略）
   - 缺点：需要扫描更多行，但SQL优化器通常能处理

---

## 十二、实际连接条件验证

### 12.1 身份表连接条件

**位置**: `fact-metric-cte.ts:63` 或 `metric-cte.ts:86`

```sql
JOIN __identities_${userIdType} i 
ON i.${userIdType} = m.${userIdType}
```

**验证**:
- 左表：身份映射表（已去重，每行是唯一的ID映射）
- 右表：事实表/指标表（每行是一个事件）
- 连接键：ID类型列（如 anonymous_id）
- 结果：每个事件行被附加对应的基准ID

### 12.2 实验单元与激活指标连接条件

**位置**: `experiment-units-query.ts:236`

```sql
LEFT JOIN __activationMetric a 
ON a.${baseIdType} = e.${baseIdType}
```

**验证**:
- 左表：曝光数据（已按实验过滤）
- 右表：激活指标数据（已按时间过滤）
- 连接键：基准ID（已对齐）
- 附加WHERE条件：segment.date <= e.timestamp（如果有segment）

### 12.3 最终用户与事实表连接条件

**位置**: `experiment-fact-metrics-query.ts:614-616`

```sql
LEFT JOIN __factTable${suffix} m 
ON m.${baseIdType} = d.${baseIdType}
```

**验证**:
- 左表：`__distinctUsers`（每个用户一行，已去重）
- 右表：`__factTable${suffix}`（每个事件一行，已对齐ID）
- 连接键：基准ID（已统一）
- 结果：1:N 连接（一个用户可能有多个事件）
- 后续处理：在 `__userMetricAgg` 中按用户分组聚合
