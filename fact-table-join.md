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
  forcedBaseIdType?: string,  // 注意：参数名是 forcedBaseIdType，不是 forcedUserIdType
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

### 4.3 baseIdType 与 forcedBaseIdType 的关系校正

| 概念 | 说明 | 出现位置 |
|-----|------|---------|
| **baseIdType** | 计算得到的基准ID类型（输出） | `getBaseIdTypeAndJoins` 返回值 |
| **forcedBaseIdType** | 强制指定的基准ID类型（输入） | `getBaseIdTypeAndJoins` 输入参数 |

**关键关系**:
- 当 `forcedBaseIdType` 存在时：`baseIdType = forcedBaseIdType`
- 当 `forcedBaseIdType` 不存在时：`baseIdType` = 使用最频繁的ID类型
- 在 exposureQuery 场景下，总是传入 `forcedBaseIdType: exposureQuery.userIdType`

**代码验证**:
- `experiment-units-query.ts:64`: `forcedBaseIdType: exposureQuery.userIdType`
- `experiment-fact-metrics-query.ts:123`: `forcedBaseIdType: userIdType`

### 4.4 键对齐执行流程

```
1. 确定基准ID类型（baseIdType）选择
   ↓ 输入：[[曝光ID类型], [事实表ID类型], [维度ID类型], [segment ID类型], [激活指标ID类型]]
   ↓ 算法：选择使用最频繁的ID类型，或使用强制指定的ID类型（forcedBaseIdType）
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

### 4.5 键对齐调用时机（三种场景）

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

#### 场景B：unitsSource === "exposureTable"（增量刷新）

- 跳过 `getExperimentUnitsQuery()` 调用
- 直接使用预计算的单元表 `params.unitsTableFullName`
- 身份对齐CTE仍然在主查询中生成（用于事实表侧）
- `__distinctUsers` 的 FROM 子句直接使用表名

#### 场景C：unitsSource === "otherQuery"（Power Analysis / Population Data）

- 跳过 `getExperimentUnitsQuery()` 调用
- 直接使用 `params.unitsSql` 作为 `__experimentUnits` CTE 的来源
- 身份对齐CTE仍然在主查询中生成（用于事实表侧）
- **注意**: `unitsSql` 必须包含完整的 `__experimentUnits AS (...)` CTE 定义

### 4.6 otherQuery 模式的精确约束条件（含 __source 与 __experimentUnits 的固定关系）

**代码位置**: `experiment-fact-metrics-query.ts:222-231`

```typescript
${
  params.unitsSource === "exposureQuery"
    ? `${getExperimentUnitsQuery(dialect, datasource, {
        ...params,
        includeIdJoins: false,
      })},`
    : params.unitsSource === "otherQuery"
      ? params.unitsSql  // 直接插入，必须包含完整的CTE定义链
      : ""
}
```

**Power Analysis 场景下的固定 CTE 链**（`power-population-ctes.ts`）:

```sql
-- 第一步：__source CTE（来自 segment 或 factTable）
__source AS (
  -- segment 来源：getSegmentCTE 的输出
  -- 或 factTable 来源：从事实表中提取 userIdType 和 timestamp
)
, __experimentUnits AS (
  SELECT
    ${settings.userIdType}                  -- 用户ID列
    , MIN(timestamp) AS first_exposure_timestamp  -- 首次曝光时间（从 __source 聚合）
    , '' as variation                       -- 空字符串变体（Power Analysis 无变体）
  FROM
    __source                                -- 固定从 __source 读取
  WHERE
    timestamp >= startDate AND timestamp <= endDate
  GROUP BY ${settings.userIdType}          -- 按用户ID去重
)
```

**约束条件精确说明**:

1. **CTE 别名约束**: `unitsSql` 必须包含 `__experimentUnits` 作为最终输出CTE的名称
   - 因为 `__distinctUsers` 的 FROM 子句固定引用 `__experimentUnits`（当 unitsSource != "exposureTable" 时）
   - 参考代码：`experiment-fact-metrics-query.ts:256-258`

2. **Power Analysis 场景的固定关系**:
   - `__source` 是数据源CTE（segment 或 factTable）
   - `__experimentUnits` 固定从 `__source` 中 `GROUP BY` 提取
   - 这是 `getPowerPopulationCTEs()` 的固定实现，不是通用的 otherQuery 模式

3. **必须包含的列**:
   - `${baseIdType}`: 用户ID列（列名必须等于基准ID类型）
   - `variation`: 变体分配列（Power Analysis 中为空字符串）
   - `first_exposure_timestamp`: 首次曝光时间列
   - 如有激活指标：`first_activation_timestamp` 列
   - 如有维度：对应的 `dim_unit_*` 或 `dim_exp_*` 列

4. **数据约束**:
   - 必须保证每个用户一行（已去重）
   - `baseIdType` 必须与 `forcedBaseIdType` 参数一致

5. **实际使用场景**（PopulationDataQueryRunner）:
   - `unitsSql` 由 `getPowerPopulationCTEs()` 生成
   - 包含完整的 CTE 链：`__source`, `__experimentUnits`
   - 参考代码：`SqlIntegration.ts:820-853`

### 4.7 exposureTable 模式的精确约束条件

**代码位置**: `experiment-fact-metrics-query.ts:256-259`

```typescript
FROM ${
  params.unitsSource === "exposureTable"
    ? `${params.unitsTableFullName}`
    : "__experimentUnits"
}
```

**表结构约束**（由 `SqlIntegration.ts:1549-1566` 创建时定义）:

```sql
CREATE TABLE ${unitsTableFullName} (
  ${exposureQuery.userIdType} STRING,        -- 必须：用户ID列（列名=曝光查询ID类型）
  variation STRING,                          -- 必须：变体分配
  first_exposure_timestamp TIMESTAMP,        -- 必须：首次曝光时间
  ${activationMetric ? 
    ", first_activation_timestamp TIMESTAMP" : ""},  -- 可选：首次激活时间
  ${experimentDimensions.map(d => 
    `, dim_exp_${d.id} STRING`).join("\n")}, -- 可选：实验维度列
  max_timestamp TIMESTAMP                    -- 必须：增量刷新用的最大时间戳
)
```

**约束条件**:
1. 用户ID列名必须等于 `exposureQuery.userIdType`（即基准ID类型）
2. 必须包含 `variation`、`first_exposure_timestamp`、`max_timestamp` 列
3. 如有激活指标，必须包含 `first_activation_timestamp` 列
4. 如有实验维度，必须包含对应的 `dim_exp_*` 列
5. 表名通过 `params.unitsTableFullName` 传递

### 4.8 身份键对齐适用范围修正

#### 曝光提取阶段（无需身份连接）

**关键发现**: 曝光提取阶段通过 `forcedBaseIdType: exposureQuery.userIdType` 强制使用曝光查询自身的ID类型作为基准ID。

**文件**: `experiment-units-query.ts:64` 或 `experiment-fact-metrics-query.ts:123`

```typescript
forcedBaseIdType: exposureQuery.userIdType  // 强制使用曝光查询的ID类型
```

**代码验证** (`experiment-units-query.ts:93-112`):
```sql
__experimentExposures AS (
  SELECT
    e.${baseIdType} as ${baseIdType}  -- 直接使用曝光表本身的ID列
    -- 曝光表的SQL模板中必然包含 exposureQuery.userIdType 对应的列
    -- 由于 forcedBaseIdType = exposureQuery.userIdType
    -- 所以 baseIdType = exposureQuery.userIdType
    -- 因此 e.${baseIdType} 就是曝光表原生的ID列，无需JOIN身份表
  FROM __rawExperiment e
  -- 此处无 JOIN 身份表的代码！
)
```

**结论**: 曝光提取阶段（`__rawExperiment` → `__experimentExposures`）**不需要**身份键对齐，因为基准ID就是曝光表自身的ID类型。

#### 事实表阶段（可能需要身份连接）

**文件**: `fact-metric-cte.ts:52-68`

```typescript
if (userIdTypes.includes(baseIdType)) {
  userIdCol = baseIdType;  // 事实表支持基准ID，无需连接
} else {
  // 事实表不支持基准ID，需要通过身份表连接
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

**结论**: 事实表阶段**可能需要**身份键对齐，取决于事实表是否支持基准ID类型。

#### 激活指标/Metric侧（可能需要身份连接）

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

#### Segment 侧（可能需要身份连接）

**文件**: `ctes/segment-cte.ts:56-68`

```typescript
// Need to use an identity join table
if (userIdType !== baseIdType) {
  return `-- Segment (${segment.name})
    SELECT
      i.${baseIdType},
      ${dateCol} as date
    FROM
      (${segmentSql}) s
      JOIN ${idJoinMap[userIdType]} i ON (i.${userIdType} = s.${userIdType})
    `;
}
```

**结论**: Segment 的 `__segment` CTE 内部可能包含身份连接。

#### 维度侧（可能需要身份连接）

**文件**: `ctes/dimension-cte.ts:10-22`

```typescript
// Need to use an identity join table
if (userIdType !== baseIdType) {
  return `-- Dimension (${dimension.name})
    SELECT
      i.${baseIdType},
      d.value
    FROM
      (${dimension.sql}) d
      JOIN ${idJoinMap[userIdType]} i ON (i.${userIdType} = d.${userIdType})
    `;
}
```

**结论**: 维度的 `__dim_unit_*` CTE 内部可能包含身份连接。

### 4.9 __identities CTE 出现的条件分析

**主查询身份CTE生成** (`experiment-fact-metrics-query.ts:105-126`):
- 无论 `unitsSource` 是什么（exposureQuery/exposureTable/otherQuery），主查询都会调用 `getIdentitiesCTE()`
- 目的：为事实表、激活指标等提供身份对齐服务

**实验单元查询身份CTE生成** (`experiment-units-query.ts:52-67`):
- 仅当 `includeIdJoins: true` 时才会在实验单元查询内部生成身份CTE
- 在 exposureQuery 场景下，`includeIdJoins: false`，所以不重复生成

**Power Analysis 场景的特殊处理** (`power-population-source-cte.ts:32`):
```typescript
{}, // no id join map needed as id type is segment id type
```
- Segment 来源下不使用 idJoinMap，因为 `settings.userIdType` 等于 segment 的 `userIdType`
- 因此不需要身份连接

**结论**:
- `__identities` CTE 不是 "仅在特定来源下出现"，而是在主查询中**总是**生成
- 但是否需要实际的 JOIN 操作取决于：各子表（事实表、segment、维度等）的ID类型是否与 `baseIdType` 一致
- 当所有表的ID类型都与 `baseIdType` 一致时，`idJoinMap` 为空，不生成实际的 `__identities_*` CTE

### 4.10 身份查询实现（含去重）

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

4. **首次激活时间**（如有激活指标）
   ```sql
   MIN(CASE WHEN (时间窗口条件) THEN a.timestamp ELSE NULL END) AS first_activation_timestamp
   ```

### 5.2 最终用户集合（__distinctUsers）

**文件**: `experiment-fact-metrics-query.ts:232-265`

**输入来源**（三选一）:
- `unitsSource === "exposureQuery"` → 来自 `__experimentUnits` CTE（由 `getExperimentUnitsQuery` 生成）
- `unitsSource === "exposureTable"` → 来自 `params.unitsTableFullName` 预计算表
- `unitsSource === "otherQuery"` → 来自 `params.unitsSql` 自定义SQL（必须包含 `__experimentUnits` CTE）

**SQL实现**:
```sql
__distinctUsers AS (
  SELECT
    ${baseIdType}
    ${dimensionCols.map((c) => `, ${c.value} AS ${c.alias}`).join("")}
    , variation
    , ${timestampColumn} AS timestamp
    , ${dialect.dateTrunc("first_exposure_timestamp", "day")} AS first_exposure_date
    ${banditDates?.length ? getBanditCaseWhen(dialect, banditDates) : ""}
    ${raMetricSettings.map(...).join("\n")}  -- 回归调整协变量窗口
  FROM ${
    params.unitsSource === "exposureTable"
      ? `${params.unitsTableFullName}`
      : "__experimentUnits"
  }
  ${distinctUsersWhere.length ? `WHERE ${distinctUsersWhere.join(" AND ")}` : ""}
)
```

**功能**:
- 从实验单元中提取最终用户列表
- 投影维度列并重命名为 alias
- 选择时间锚点（曝光时间或激活时间）
- 应用激活指标过滤（如仅保留已激活用户）
- 应用时间窗口过滤（skipPartialData）

---

## 六、四类维度列的依赖差异分析

### 6.1 维度类型定义

**类型定义** (`integrations.d.ts:239-265`):
```typescript
export type UserDimension = {
  type: "user";
  dimension: DimensionInterface;
};
export type ExperimentDimension = {
  type: "experiment";
  id: string;
  specifiedSlices?: string[];
};
export type DateDimension = {
  type: "date";
};
export type ActivationDimension = {
  type: "activation";
};
export type Dimension =
  | UserDimension
  | ExperimentDimension
  | DateDimension
  | ActivationDimension;
```

### 6.2 四类维度的依赖差异对比

| 维度类型 | 别名 | 依赖来源 | 依赖CTE | 连接方式 | 值计算方式 | 代码位置 |
|---------|------|---------|---------|---------|-----------|---------|
| **user** | `dim_unit_${id}` | 用户属性 | `__dim_unit_${id}` | LEFT JOIN | `getDimensionValuePerUnit()`（聚合） | `dimension-col.ts:14-18` |
| **experiment** | `dim_exp_${id}` | 曝光事件 | `__experimentExposures` | 直接SELECT | `getDimensionValuePerUnit()`（聚合） | `dimension-col.ts:9-13` |
| **date** | `dim_pre_date` | 曝光时间 | 无（派生） | 无 | `DATE_TRUNC(first_exposure_timestamp, day)` | `dimension-col.ts:19-25` |
| **activation** | `dim_activation` | 激活状态 | 无（派生） | 无 | `CASE WHEN first_activation_timestamp IS NULL THEN 'Not Activated' ELSE 'Activated' END` | `dimension-col.ts:26-34` |

### 6.3 user 维度（依赖 __dim_unit_* CTE）

**代码位置**: `experiment-units-query.ts:125-136, 225-233, 189-194`

```sql
-- CTE 定义（独立CTE）
__dim_unit_${dimensionId} AS (
  -- dimension-cte.ts 生成，可能包含内部身份连接
)

-- 实验单元阶段 JOIN
LEFT JOIN __dim_unit_${d.dimension.id} __dim_unit_${d.dimension.id} ON (
  __dim_unit_${d.dimension.id}.${baseIdType} = e.${baseIdType}
)

-- 实验单元阶段聚合（__experimentUnits SELECT）
, ${getDimensionValuePerUnit(dialect, d)} AS dim_unit_${d.dimension.id}

-- __distinctUsers 投影
, dim_unit_${dimension.dimension.id} AS dim_unit_${dimension.dimension.id}
```

**特点**:
- 需要独立的 `__dim_unit_*` CTE
- 需要 LEFT JOIN 到 `__experimentExposures`
- 需要在 `__experimentUnits` 中聚合

### 6.4 experiment 维度（依赖 __experimentExposures）

**代码位置**: `experiment-units-query.ts:195-200`

```sql
-- 实验单元阶段聚合（__experimentUnits SELECT）
, ${getDimensionValuePerUnit(dialect, d)} AS dim_exp_${d.id}

-- __distinctUsers 投影
, dim_exp_${dimension.id} AS dim_exp_${dimension.id}
```

**特点**:
- 从 `__experimentExposures` 直接读取（曝光事件中包含实验维度信息）
- 不需要独立CTE
- 不需要额外 JOIN
- 只需要在 `__experimentUnits` 中聚合

### 6.5 date 维度（从 first_exposure_timestamp 派生）

**代码位置**: `dimension-col.ts:19-25`

```sql
-- __distinctUsers SELECT 中直接计算
, ${dialect.formatDate(
  dialect.dateTrunc("first_exposure_timestamp", "day"),
)} AS dim_pre_date
```

**特点**:
- 从 `first_exposure_timestamp` 派生
- 不需要任何 CTE
- 不需要 JOIN
- 直接在 `__distinctUsers` 中计算

### 6.6 activation 维度（从 first_activation_timestamp 派生）

**代码位置**: `dimension-col.ts:26-34`

```sql
-- __distinctUsers SELECT 中直接计算
, ${dialect.ifElse(
  `first_activation_timestamp IS NULL`,
  "'Not Activated'",
  "'Activated'",
)} AS dim_activation
```

**特点**:
- 从 `first_activation_timestamp` 派生
- 不需要任何 CTE
- 不需要 JOIN
- 直接在 `__distinctUsers` 中计算
- 与 `computeOnActivatedUsersOnly` 标志关联

### 6.7 四类维度在 CTE 链中的位置

```
__rawExperiment
    ↓
__experimentExposures  ← experiment维度直接从此读取
    ↓ LEFT JOIN
__dim_unit_*（user维度）  ← user维度需要独立CTE和JOIN
    ↓
__experimentUnits（所有维度在此聚合）
    ↓ 投影派生
__distinctUsers
    ├─ dim_exp_*（experiment维度，直接投影）
    ├─ dim_unit_*（user维度，直接投影）
    ├─ dim_pre_date（date维度，从first_exposure_timestamp派生）
    └─ dim_activation（activation维度，从first_activation_timestamp派生）
```

---

## 七、时间窗口锚点分析

### 7.1 锚点选择逻辑

**文件**: `experiment-fact-metrics-query.ts:148-153`

```typescript
const computeOnActivatedUsersOnly =
  activationMetric !== null &&
  !params.dimensions.some((d) => d.type === "activation");

const timestampColumn = computeOnActivatedUsersOnly
  ? "first_activation_timestamp"    // 激活用户路径：以激活时间为锚点
  : "first_exposure_timestamp";     // 曝光用户路径：以曝光时间为锚点
```

### 7.2 两种路径下的取值差异

| 维度 | 曝光用户路径（默认） | 激活用户路径（computeOnActivatedUsersOnly=true） |
|-----|-------------------|----------------------------------------|
| **触发条件** | 无激活指标 或 按激活维度拆分 | 有激活指标 且 不按激活维度拆分 |
| **锚点列** | `first_exposure_timestamp` | `first_activation_timestamp` |
| **含义** | 用户首次看到实验的时间 | 用户首次完成激活事件的时间 |
| **用户过滤** | 无特殊过滤 | 排除 `first_activation_timestamp IS NULL` 的用户 |
| **部分数据跳过** | `first_exposure_timestamp <= endDate` | `first_activation_timestamp <= endDate` |
| **distinctUsersWhere** | （无激活过滤） | `["first_activation_timestamp IS NOT NULL"]` |

### 7.3 锚点在时间窗口过滤中的应用

**文件**: `experiment-fact-metrics-query.ts:521-617`

**调用位置**: 在 `__userMetricJoin` CTE 中作为 `d.timestamp` 传递

```sql
__userMetricJoin AS (
  SELECT
    d.variation AS variation
    , d.timestamp AS timestamp  -- 这个 d.timestamp 就是上面选择的锚点
    -- 对每个指标列应用时间窗口过滤
    , CASE WHEN (时间窗口条件) THEN m.m0_value ELSE NULL END as m0_value
  FROM __distinctUsers d
  LEFT JOIN __factTable m ON m.${baseIdType} = d.${baseIdType}
)
```

**传递给时间窗口过滤** (`experiment-fact-metrics-query.ts:542-543`):
```typescript
addCaseWhenTimeFilter(dialect, {
  metricTimestampColExpr: "m.timestamp",        // 事实表事件时间
  exposureTimestampColExpr: "d.timestamp",      // 锚点时间（曝光或激活）
  ...
})
```

### 7.4 转换窗口子句详细实现

**文件**: `conversion-window-clause.ts:10-54`

**输入参数**:
- `baseCol`: 曝光/激活时间列（d.timestamp，锚点）
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

## 八、实验单元阶段三类连接的精确实现

### 8.1 Segment 连接（INNER JOIN）

**代码位置**: `experiment-units-query.ts:220-223`

```sql
JOIN __segment s ON (s.${baseIdType} = e.${baseIdType})
```

**过滤条件**: `experiment-units-query.ts:239`
```sql
WHERE s.date <= e.timestamp
```

**详细说明**:
- **连接类型**: `JOIN`（即 INNER JOIN）
- **左表**: `__experimentExposures e`（曝光数据）
- **右表**: `__segment s`（用户段数据）
- **连接键**: `${baseIdType}`（用户ID）
- **过滤条件**: `s.date <= e.timestamp`（用户段日期早于或等于曝光日期）
- **效果**: 只保留在用户段中且满足日期条件的用户

**__segment CTE 内部结构** (`segment-cte.ts`):
- 输出列：`${baseIdType}`, `date`
- 如需身份对齐，在 CTE 内部通过 `JOIN __identities_xxx` 实现

### 8.2 Activation 连接（LEFT JOIN + SELECT 过滤）

**代码位置**: `experiment-units-query.ts:234-238`

```sql
LEFT JOIN __activationMetric a ON (a.${baseIdType} = e.${baseIdType})
```

**过滤条件**: `experiment-units-query.ts:202-215`（在 SELECT 子句中）
```sql
MIN(
  ${dialect.ifElse(
    getConversionWindowClause(
      dialect,
      "e.timestamp",      // 曝光时间
      "a.timestamp",      // 激活事件时间
      activationMetric,   // 激活指标定义（含窗口）
      settings.endDate,
      overrideConversionWindows,
    ),
    "a.timestamp",  // 满足窗口条件：取激活时间
    "NULL",         // 不满足窗口条件：NULL
  )}
) AS first_activation_timestamp
```

**详细说明**:
- **连接类型**: `LEFT JOIN`
- **左表**: `__experimentExposures e`（曝光数据）
- **右表**: `__activationMetric a`（激活指标数据）
- **连接键**: `${baseIdType}`（用户ID）
- **过滤方式**: 在 SELECT 子句中通过 `CASE WHEN` 进行时间窗口过滤，而不是在 WHERE 子句中
- **效果**: 保留所有曝光用户，但只为满足时间窗口的用户计算激活时间
- **聚合**: 使用 `MIN()` 取最早的有效激活时间

**__activationMetric CTE 内部结构** (`metric-cte.ts`):
- 输出列：`${baseIdType}`, `value`, `timestamp`
- 如需身份对齐，在 CTE 内部通过 `JOIN __identities_xxx` 实现
- 已应用日期范围过滤：`startDate <= timestamp <= endDate`

### 8.3 维度连接（LEFT JOIN，无额外过滤）

**代码位置**: `experiment-units-query.ts:225-233`

```sql
LEFT JOIN __dim_unit_${d.dimension.id} __dim_unit_${d.dimension.id} ON (
  __dim_unit_${d.dimension.id}.${baseIdType} = e.${baseIdType}
)
```

**维度值聚合**: `experiment-units-query.ts:189-194`
```sql
, ${getDimensionValuePerUnit(dialect, d)} AS dim_unit_${d.dimension.id}
```

**详细说明**:
- **连接类型**: `LEFT JOIN`
- **左表**: `__experimentExposures e`（曝光数据）
- **右表**: `__dim_unit_${dimensionId}`（维度数据）
- **连接键**: `${baseIdType}`（用户ID）
- **过滤条件**: 无额外过滤条件
- **效果**: 保留所有曝光用户，维度值可能为 NULL
- **聚合**: 通过 `getDimensionValuePerUnit()` 处理（如 MAX、MIN 等）

**__dim_unit_* CTE 内部结构** (`dimension-cte.ts`):
- 输出列：`${baseIdType}`, `value`
- 如需身份对齐，在 CTE 内部通过 `JOIN __identities_xxx` 实现

### 8.4 三类连接对比表

| 类型 | 连接类型 | 过滤位置 | 过滤条件 | 效果 |
|-----|---------|---------|---------|------|
| **Segment** | INNER JOIN | WHERE 子句 | `s.date <= e.timestamp` | 只保留在用户段中的用户 |
| **Activation** | LEFT JOIN | SELECT 子句 | 时间窗口（CASE WHEN） | 保留所有用户，仅有效激活被统计 |
| **维度（user类型）** | LEFT JOIN | 无 | 无 | 保留所有用户，维度值可为NULL |

---

## 九、事实表与实验分配连接逻辑

### 9.1 连接策略

**文件**: `experiment-fact-metrics-query.ts:521-617`

**CTE**: `__userMetricJoin${suffix}`

**连接方式**: **LEFT JOIN**（左连接）

**实际SQL代码**:
```sql
SELECT
  d.variation AS variation
  , d.timestamp AS timestamp  -- 来自 __distinctUsers 的锚点时间（曝光或激活）
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

### 9.2 时间窗口过滤

**文件**: `add-case-when-time-filter.ts:7-40`

**调用位置**: 在 `__userMetricJoin` CTE 的 SELECT 子句中，对每个指标列应用

**功能**: 确保事实表事件发生在实验曝光（或激活）之后的有效时间窗口内

```typescript
dialect.ifElse(
  `${conversionWindowClause}  -- 锚点时间 ≤ 事件时间 ≤ 锚点时间 + 窗口
  ${metricQuantileSettings?.ignoreZeros && metricQuantileSettings?.type === "event" 
    ? `AND ${col} != 0` 
    : ""}`,
  `${col}`,    // 满足条件：保留原值
  `NULL`       // 不满足条件：设为NULL（后续聚合时自动忽略）
)
```

---

## 十、用户级别聚合

### 10.1 每用户聚合（__userMetricAgg）

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

### 10.2 多事实表支持

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

## 十一、最终统计聚合

### 11.1 变体级别统计

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

## 十二、完整CTE依赖关系图（精确版）

### 12.1 unitsSource === "exposureQuery" 场景（最常见）

```
阶段0：身份键对齐（最先执行，仅用于事实表、激活指标、segment、维度）
├─ __identities_${idType}* （身份映射表，已去重）
│   输入：getIdentitiesQuery
│   输出：${baseIdType}, ${otherIdType}
│   连接键：GROUP BY 去重
│   注意：曝光提取阶段不使用此CTE！
│
阶段1：实验分配数据提取（曝光侧无身份连接）
├─ __rawExperiment （原始曝光数据）
│   输入：exposureQuery.query（编译后）
│   输出：experiment_id, user_id, variation_id, timestamp
│   注意：user_id 列名 = exposureQuery.userIdType = baseIdType
│
├─ __experimentExposures （过滤后曝光数据）
│   输入：__rawExperiment
│   过滤条件：experiment_id, 日期范围, queryFilter
│   输出：${baseIdType}, variation, timestamp
│   关键：直接使用 e.${baseIdType}，无需JOIN身份表
│
├─ [可选] __dim_unit_${dimensionId}* （user维度数据，独立CTE）
│   输入：getDimensionCTE(维度定义)
│   输出：${baseIdType}, value
│   └── 内部身份对齐：如果维度表ID类型≠baseIdType，则 JOIN __identities_xxx
│
├─ [可选] __activationMetric （激活指标数据）
│   输入：getMetricCTE(激活指标)
│   输出：${baseIdType}, value, timestamp
│   └── 内部身份对齐：如果指标表ID类型≠baseIdType，则 JOIN __identities_xxx
│
├─ [可选] __segment （用户段数据）
│   输入：getSegmentCTE(segment定义)
│   输出：${baseIdType}, date
│   └── 内部身份对齐：如果segment表ID类型≠baseIdType，则 JOIN __identities_xxx
│
└─ __experimentUnits （用户级别去重 + 多表连接）
   输入：__experimentExposures e + [__activationMetric a] + [__segment s] + [__dim_unit_* d]
   连接：
     - JOIN __segment s ON s.${baseIdType} = e.${baseIdType} （INNER JOIN + WHERE s.date <= e.timestamp）
     - LEFT JOIN __dim_unit_* ON dim.${baseIdType} = e.${baseIdType} （LEFT JOIN，无过滤）
     - LEFT JOIN __activationMetric a ON a.${baseIdType} = e.${baseIdType} （LEFT JOIN，SELECT中过滤）
   分组键：e.${baseIdType}
   聚合：
     - variation: 去重策略（__multiple__ 或 首次曝光）
     - first_exposure_timestamp: MIN(e.timestamp)
     - [first_activation_timestamp: MIN(CASE WHEN 窗口 THEN a.timestamp ELSE NULL END)]
     - dim_unit_*: 每个user维度的聚合值
     - dim_exp_*: 每个experiment维度的聚合值
   输出：${baseIdType}, variation, first_exposure_timestamp, first_activation_timestamp, dim_*
   │
阶段2：最终用户集合（时间锚点选择 + 维度派生）
└─ __distinctUsers
   输入：__experimentUnits （或 exposureTable 或 otherQuery 的 __experimentUnits）
   锚点选择：
     - 激活用户路径：timestampColumn = first_activation_timestamp
     - 曝光用户路径：timestampColumn = first_exposure_timestamp
   维度派生：
     - user维度：直接投影 dim_unit_*
     - experiment维度：直接投影 dim_exp_*
     - date维度：从 first_exposure_timestamp 派生
     - activation维度：从 first_activation_timestamp 派生
   过滤条件（distinctUsersWhere）：
     - 激活用户路径：first_activation_timestamp IS NOT NULL
     - 部分数据跳过：timestampColumn <= endDate
   输出：${baseIdType}, variation, timestamp（锚点）, dim_*, ...
   │
阶段3：事实表数据提取（每个事实表独立执行）
└─ __factTable${suffix}*
   输入：factTable.sql（编译后）
   过滤条件：
     - 日期范围：startDate <= timestamp <= endDate
     - 指标过滤器：WHERE子句中的条件
   输出：${baseIdType}, timestamp, m${index}_value, [m${index}_denominator]
   └── 身份对齐：
       - 如果事实表支持基准ID：直接 SELECT ${baseIdType}
       - 否则：JOIN __identities_xxx i ON i.${idType} = m.${idType}
   │
阶段4：事实表与实验分配连接（每个事实表独立执行）
└─ __userMetricJoin${suffix}*
   输入：__distinctUsers d LEFT JOIN __factTable${suffix} m
   连接键：m.${baseIdType} = d.${baseIdType}
   SELECT 处理：
     - 保留 d.*（实验分配数据，含锚点时间 d.timestamp）
     - 对每个 m.* 列应用 addCaseWhenTimeFilter
       - 锚点：d.timestamp（曝光或激活时间）
       - 条件：m.timestamp 在有效窗口内
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

### 12.2 关键依赖关系总结表

| 下游CTE | 上游依赖 | 连接键/依赖方式 | 身份对齐需求 |
|---------|---------|----------------|------------|
| `__experimentExposures` | `__rawExperiment` | 直接FROM | ❌ 不需要（baseIdType=曝光表ID类型） |
| `__dim_unit_*` | getDimensionCTE | 内部可能JOIN身份表 | ⚠️ 取决于维度ID类型 |
| `__activationMetric` | getMetricCTE | 内部可能JOIN身份表 | ⚠️ 取决于指标ID类型 |
| `__segment` | getSegmentCTE | 内部可能JOIN身份表 | ⚠️ 取决于segment ID类型 |
| `__experimentUnits` | `__experimentExposures` + 其他 | JOIN ON ${baseIdType} | ❌ 不需要（子CTE已内部处理） |
| `__distinctUsers` | `__experimentUnits`/表/SQL | 直接FROM + WHERE过滤 | ❌ 不需要 |
| `__factTable${suffix}` | 事实表SQL | 可能JOIN身份表 | ✅ 取决于是否支持baseIdType |
| `__userMetricJoin${suffix}` | `__distinctUsers` + `__factTable` | LEFT JOIN ON ${baseIdType} | ❌ 不需要（双方已对齐） |
| `__userMetricAgg${suffix}` | `__userMetricJoin${suffix}` | GROUP BY | ❌ 不需要 |
| 最终统计 | `__userMetricAgg` + 其他 | GROUP BY | ❌ 不需要 |

---

## 十三、各阶段连接条件的实际实现验证

### 13.1 身份表连接条件（事实表侧）

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
- 触发条件：事实表/指标表的 `userIdTypes` 不包含 `baseIdType`

### 13.2 身份表连接条件（Segment侧）

**位置**: `segment-cte.ts:66`

```sql
JOIN ${idJoinMap[userIdType]} i 
ON (i.${userIdType} = s.${userIdType})
```

**验证**:
- 在 `__segment` CTE 内部
- 触发条件：`segment.userIdType !== baseIdType`
- 输出列投影为 `i.${baseIdType}`

### 13.3 身份表连接条件（维度侧）

**位置**: `dimension-cte.ts:20`

```sql
JOIN ${idJoinMap[userIdType]} i 
ON (i.${userIdType} = d.${userIdType})
```

**验证**:
- 在 `__dim_unit_*` CTE 内部
- 触发条件：`dimension.userIdType !== baseIdType`
- 输出列投影为 `i.${baseIdType}`

### 13.4 实验单元与Segment连接条件

**位置**: `experiment-units-query.ts:221-223`

```sql
JOIN __segment s ON (s.${baseIdType} = e.${baseIdType})
```

**验证**:
- 左表：`__experimentExposures e`（已按实验过滤）
- 右表：`__segment s`（用户段数据，可能内部已做身份对齐）
- 连接键：基准ID（双方都已对齐到baseIdType）
- 连接类型：INNER JOIN（只保留在用户段中的用户）
- 附加WHERE条件：`s.date <= e.timestamp`（segment日期早于曝光日期）

### 13.5 实验单元与user维度连接条件

**位置**: `experiment-units-query.ts:228-230`

```sql
LEFT JOIN __dim_unit_${dimensionId} __dim_unit_${dimensionId} ON (
  __dim_unit_${dimensionId}.${baseIdType} = e.${baseIdType}
)
```

**验证**:
- 左表：`__experimentExposures e`
- 右表：`__dim_unit_${dimensionId}`（user维度数据，可能内部已做身份对齐）
- 连接键：基准ID
- 连接类型：LEFT JOIN（保留所有用户，维度值可为NULL）
- 无额外过滤条件

### 13.6 实验单元与激活指标连接条件

**位置**: `experiment-units-query.ts:235-237`

```sql
LEFT JOIN __activationMetric a ON (a.${baseIdType} = e.${baseIdType})
```

**验证**:
- 左表：`__experimentExposures e`
- 右表：`__activationMetric a`（激活指标数据，可能内部已做身份对齐）
- 连接键：基准ID
- 连接类型：LEFT JOIN（保留所有用户，即使未激活）
- 过滤在SELECT子句中：`MIN(CASE WHEN 窗口 THEN a.timestamp ELSE NULL END)`

### 13.7 最终用户与事实表连接条件

**位置**: `experiment-fact-metrics-query.ts:614-616`

```sql
LEFT JOIN __factTable${suffix} m 
ON m.${baseIdType} = d.${baseIdType}
```

**验证**:
- 左表：`__distinctUsers`（每个用户一行，已去重）
- 右表：`__factTable${suffix}`（每个事件一行，已对齐ID）
- 连接键：基准ID（双方都已对齐到baseIdType）
- 连接类型：LEFT JOIN（保留所有实验用户）
- 结果：1:N 连接（一个用户可能有多个事件）
- 后续处理：在 `__userMetricAgg` 中按用户分组聚合

### 13.8 多事实表连接条件（最终统计阶段）

**位置**: `experiment-fact-metric-statistics-cte.ts:197-200`

```sql
LEFT JOIN ${joinedMetricTableName}${suffix} m${suffix} ON (
  m${suffix}.${baseIdType} = m.${baseIdType}
)
```

**验证**:
- 左表：`__userMetricAgg`（第一个事实表的用户聚合结果）
- 右表：`__userMetricAgg1`（第二个事实表的用户聚合结果）
- 连接键：基准ID
- 连接类型：LEFT JOIN（保留所有用户，即使第二个事实表无数据）

---

## 十四、关键设计决策

### 14.1 为什么使用LEFT JOIN而不是INNER JOIN

**原因**:
- 保留所有实验用户，即使没有事实表事件
- 确保分母（用户数）统计准确
- 正确计算转化率（有事件用户 / 总用户）

### 14.2 为什么Segment使用INNER JOIN

**原因**:
- Segment 是用户准入条件，只有在用户段中的用户才应被纳入实验
- 通过 `s.date <= e.timestamp` 确保用户在曝光前已进入用户段

### 14.3 为什么Activation过滤放在SELECT而不是WHERE

**原因**:
- 保留所有曝光用户，便于计算激活率（激活用户 / 总曝光用户）
- 通过 `CASE WHEN` 将不满足条件的激活时间设为NULL
- 在 `computeOnActivatedUsersOnly` 模式下再通过WHERE过滤非激活用户

### 14.4 为什么在用户级别先去重再连接

**优点**:
- 减少连接的数据量（1:N 变 1:1）
- 确保每个用户只保留正确的变体分配
- 避免重复计算曝光时间窗口

### 14.5 为什么身份对齐在各子CTE内部执行

**原因**:
- 封装复杂性：`__segment`、`__activationMetric`、`__dim_unit_*` 等CTE内部处理身份对齐
- `__experimentUnits` 连接这些子CTE时只需使用 `baseIdType`，无需关心内部实现
- 提高代码复用性和可维护性

### 14.6 为什么曝光提取阶段不需要身份对齐

**原因**:
- 通过 `forcedBaseIdType: exposureQuery.userIdType` 强制使用曝光表自身的ID类型
- 曝光表的SQL模板必然包含该ID类型的列
- 减少不必要的JOIN操作，提高性能

### 14.7 为什么支持两种时间锚点

**场景**:
- 曝光时间锚点：适用于所有实验，统计从看到实验开始的行为
- 激活时间锚点：适用于有激活指标的实验，统计从完成关键事件开始的行为
- 避免过早统计未真正进入产品的用户

### 14.8 为什么四类维度有不同的依赖方式

**原因**:
- **user维度**：用户属性通常在独立表中，需要JOIN
- **experiment维度**：实验维度信息通常在曝光事件中直接记录
- **date维度**：日期可以从时间戳直接派生，无需额外数据
- **activation维度**：激活状态可以从激活时间派生，无需额外数据

### 14.9 为什么otherQuery模式需要包含完整CTE链

**原因**:
- 灵活性：支持 Power Analysis、Population Data 等特殊场景
- 保持主查询逻辑不变：仅替换 `__experimentUnits` 的来源
- 封装复杂性：让调用方负责构建完整的单元数据链

### 14.10 为什么支持多事实表

**场景**:
- 比率指标的分子和分母来自不同事实表
- 多个指标来自不同事实表（如：购买事件 + 登录事件）

**限制**: 最多2个事实表（当前实现）

---

## 十五、代码优化点观察

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

4. **身份对齐范围优化**:
   - 仅在事实表、激活指标、segment、维度需要时才进行身份连接
   - 曝光提取阶段跳过身份连接，减少JOIN操作
   - 身份对齐封装在各子CTE内部，简化上层逻辑

5. **otherQuery 模式设计**:
   - 通过完整替换 `__experimentUnits` CTE 实现灵活性
   - 支持 Power Analysis、Population Data 等特殊场景
   - 保持主查询逻辑不变，仅替换单元来源

6. **维度分层设计**:
   - 将维度分为四类（user、experiment、date、activation）
   - 每类维度有不同的依赖方式和计算时机
   - 提高代码可读性和可维护性
