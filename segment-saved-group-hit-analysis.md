# Segment 与 Saved Group 命中判定机制分析

## 一、核心差异概述

| 维度 | Saved Group | Segment |
|------|-------------|---------|
| **执行层** | SDK 侧 / 服务端（分配时） | 分析端（评估时） |
| **判定方式** | 按 ID 列表 / 条件即时匹配 | 按 SQL 子集关联过滤 |
| **生效时机** | 实验分配阶段 | 实验评估 / 回溯阶段 |
| **数据来源** | GrowthBook 内部维护的 ID 列表 | 数仓用户表 / 事实表 |
| **可回溯性** | 分配后固定，无法回溯修改 | 分析时可更换 segment 重新计算 |
| **结构真源** | `packages/shared/src/validators/saved-group.ts` | `packages/shared/src/validators/segment.ts` |
| **分析层代码** | 不进入分析 SQL | `packages/back-end/src/integrations/sql/queries/experiment-units-query.ts` + `packages/back-end/src/integrations/sql/ctes/segment-cte.ts` |

---

## 二、Saved Group 命中判定机制

### 2.1 结构真源

Saved Group 的类型定义真源在 Zod validator 中（`packages/shared/src/validators/saved-group.ts:9-26`）：

```typescript
export const savedGroupTypeValidator = z.enum(["condition", "list"]);

export const savedGroupValidator = z
  .object({
    id: z.string(),
    organization: z.string(),
    groupName: z.string(),
    owner: ownerField,
    type: savedGroupTypeValidator,
    condition: z.string().optional(),      // condition 类型：JSON 字符串
    attributeKey: z.string().optional(),   // list 类型：匹配字段
    values: z.array(z.string()).optional(), // list 类型：ID 列表
    dateUpdated: z.date(),
    dateCreated: z.date(),
    description: z.string().optional(),
    projects: z.array(z.string()).optional(),
    useEmptyListGroup: z.boolean().optional(),
    archived: z.boolean().optional(),
  })
  .strict();
```

**注意**：`packages/shared/types/saved-group.d.ts` 中的类型只是从 validator 派生的别名（`z.infer<typeof savedGroupValidator>`），不包含结构定义。

Saved Group 在实验 phase 中的目标配置结构（`packages/shared/src/validators/shared.ts:38-44`）：

```typescript
export const savedGroupTargeting = z
  .object({
    match: z.enum(["all", "none", "any"]),  // 匹配模式
    ids: z.array(z.string()),                // saved group ID 列表
  })
  .strict();
```

### 2.2 条件构建（服务端）

在构建 SDK payload 时，`getParsedCondition()` 将 saved group 配置转换为 SDK 可评估的条件（`packages/back-end/src/util/features.ts:125-208`）：

```typescript
function getParsedCondition(
  groupMap: GroupMap,
  condition?: string,
  savedGroups?: SavedGroupTargeting[],
) {
  const conditions: ConditionInterface[] = [];
  
  // 1. 解析常规条件 JSON
  if (condition) conditions.push(JSON.parse(condition));
  
  // 2. 解析 savedGroups 配置（支持 all/any/none 三种匹配模式）
  if (savedGroups) {
    savedGroups.forEach(({ ids, match }) => {
      if (match === "all") {
        // 全部命中：每个 group 单独 AND
        ids.forEach(id => conditions.push(getSavedGroupCondition(id, groupMap, true)));
      } else if (match === "any") {
        // 任意命中：多个 group 用 OR 包裹
        conditions.push({ $or: ids.map(id => getSavedGroupCondition(id, groupMap, true)) });
      } else if (match === "none") {
        // 全部不命中：每个 group 单独 AND NOT
        ids.forEach(id => conditions.push(getSavedGroupCondition(id, groupMap, false)));
      }
    });
  }
  
  return { $and: conditions };
}
```

单个 saved group 的条件转换逻辑 `getSavedGroupCondition()`（`packages/back-end/src/util/features.ts:102-123`）：

```typescript
function getSavedGroupCondition(
  groupId: string,
  groupMap: GroupMap,
  include: boolean,
): null | ConditionInterface {
  const group = groupMap.get(groupId);
  
  // condition 类型：直接使用条件表达式
  if (group.type === "condition" && group.condition) {
    const cond = JSON.parse(group.condition);
    return include ? cond : { $not: cond };
  }
  
  // list 类型：生成 $inGroup / $notInGroup 操作符
  return {
    [group.attributeKey]: { 
      [include ? "$inGroup" : "$notInGroup"]: groupId 
    },
  };
}
```

### 2.3 条件评估（SDK 侧）

SDK 在 `evalCondition()` 中处理 `$inGroup` 和 `$notInGroup` 操作符（`packages/sdk-js/src/mongrule.ts:238-241`）：

```typescript
case "$inGroup":
  return isIn(actual, savedGroups[expected] || []);
case "$notInGroup":
  return !isIn(actual, savedGroups[expected] || []);
```

**关键点**：
- SDK 从 payload 中接收 `savedGroups: Record<string, (string | number)[]>` 字典
- 匹配过程完全在本地内存完成，无需网络请求
- 对于不支持 `savedGroupReferences` 能力的旧版 SDK，服务端会在 payload 构建时直接展开 saved group 为 `$in` 条件（`packages/back-end/src/util/features.ts:565-568, 757-768`）

---

## 三、Segment 命中判定机制

### 3.1 结构真源

Segment 的类型定义真源在 Zod validator 中（`packages/shared/src/validators/segment.ts:9-27`）：

```typescript
const TYPES = ["SQL", "FACT"] as const;

export const segmentValidator = z
  .object({
    id: z.string(),
    organization: z.string(),
    owner: ownerField,
    datasource: z.string(),
    dateCreated: z.date(),
    dateUpdated: z.date(),
    name: z.string(),
    description: z.string(),
    userIdType: z.string(),       // segment 使用的 ID 类型
    type: z.enum(TYPES),          // "SQL" | "FACT"
    managedBy: z.enum(["", "api", "config"]).optional(),
    sql: z.string().optional(),   // SQL 类型：用户自定义 SQL
    factTableId: z.string().optional(), // FACT 类型：关联事实表
    filters: z.array(z.string()).optional(), // FACT 类型：过滤条件
    projects: z.array(z.string()).optional(),
  })
  .strict();
```

**注意**：`packages/shared/types/segment.d.ts` 中的类型只是从 validator 派生的别名（`z.infer<typeof segmentValidator>`），不包含结构定义。

### 3.2 SQL 过滤逻辑（分析端）

Segment 过滤发生在实验单元查询 `__experimentUnits` CTE 中（`packages/back-end/src/integrations/sql/queries/experiment-units-query.ts:148-163, 220-224`）：

```sql
WITH
  -- 1. 构建 segment CTE
  __segment as (
    -- SQL 类型：直接使用用户定义的 SQL
    -- FACT 类型：通过 getFactSegmentCTE 生成
    SELECT user_id, date FROM (...segment SQL...) s
  ),
  
  -- 2. 实验曝光数据
  __experimentExposures AS (
    SELECT 
      e.user_id, 
      e.variation_id, 
      e.timestamp
    FROM __rawExperiment e
    WHERE e.experiment_id = 'exp_xxx'
  ),
  
  -- 3. 实验单元：曝光数据 JOIN segment 过滤
  __experimentUnits AS (
    SELECT
      e.user_id,
      MAX(e.variation_id) AS variation,
      MIN(e.timestamp) AS first_exposure_timestamp
    FROM __experimentExposures e
    -- 关键：INNER JOIN segment，只保留同时在 segment 中的用户
    JOIN __segment s ON (s.user_id = e.user_id)
    -- 时间一致性：确保用户在曝光时刻已属于该 segment
    WHERE s.date <= e.timestamp
    GROUP BY e.user_id
  )
```

### 3.3 Segment CTE 构建

`getSegmentCTE()` 负责生成 segment 的 CTE SQL（`packages/back-end/src/integrations/sql/ctes/segment-cte.ts:8-83`）：

```typescript
function getSegmentCTE(
  dialect: SqlDialect,
  segment: SegmentInterface,
  baseIdType: string,
  idJoinMap: Record<string, string>,
  factTableMap: FactTableMap,
  cteContext?: CteContext,
): string {
  let segmentSql: string;
  
  if (segment.type === "SQL") {
    segmentSql = segment.sql;
  } else {
    // FACT 类型：基于事实表 + 过滤器生成 SQL
    segmentSql = getFactSegmentCTE(dialect, {
      factTable,
      filters: segment.filters,
      cteContext,
      // ...
    });
  }
  
  // 处理跨 ID 类型 join（如 segment 用 user_id，但实验用 anonymous_id）
  if (segment.userIdType !== baseIdType) {
    return `
      SELECT
        i.${baseIdType},
        ${dateCol} as date
      FROM (${segmentSql}) s
      JOIN ${idJoinMap[segment.userIdType]} i 
        ON (i.${segment.userIdType} = s.${segment.userIdType})
    `;
  }
  
  return segmentSql;
}
```

---

## 四、回溯链路深挖：从 QueryRunner 到 SQL 生成

### 4.1 Segment 传递全链路

```
ExperimentResultsQueryRunner.startQueries()
  ↓ (调用 startExperimentResultQueries)
  ├─ 读取 snapshotSettings.segment
  │   (ExperimentResultsQueryRunner.ts:115-120)
  │
  ├─ 构建 unitQueryParams.segment = segmentObj
  │   (ExperimentResultsQueryRunner.ts:174-184)
  │
  ├─ [分支1] useUnitsTable = true
  │   ├─ 调用 integration.getExperimentUnitsTableQuery(unitQueryParams)
  │   │   → SqlIntegration.getExperimentUnitsTableQuery()
  │   │   → 内部调用 this.getExperimentUnitsQuery(params)
  │   │   → buildExperimentUnitsQuerySql()
  │   │   → 生成 CREATE TABLE ... AS SELECT * FROM __experimentUnits
  │   │
  │   └─ 后续指标查询：unitsSource = "exposureTable"
  │       → 直接读临时表，无需再次 JOIN segment
  │
  └─ [分支2] useUnitsTable = false
      └─ 每个指标查询：unitsSource = "exposureQuery"
          → getExperimentMetricQuery() 内嵌 getExperimentUnitsQuery()
          → 每个指标查询都重新 JOIN segment
```

### 4.2 快照设置构建：saved group 缺席的证据

`getSnapshotSettings()` 函数（`packages/back-end/src/services/experiments.ts:429-725`）展示了 snapshotSettings 的完整构建过程：

```typescript
export function getSnapshotSettings({
  experiment,
  phaseIndex,
  // ... 其他参数
}: { /* ... */ }): ExperimentSnapshotSettings {
  const phase = experiment.phases[phaseIndex];
  
  return {
    activationMetric: experiment.activationMetric || null,
    attributionModel: experiment.attributionModel || "firstExposure",
    lookbackOverride: lookbackOverride,
    skipPartialData: !!experiment.skipPartialData,
    segment: experiment.segment || "",           // ✓ segment 被读取
    queryFilter: experiment.queryFilter || "",   // ✓ queryFilter 被读取
    datasourceId: experiment.datasource || "",
    // ... 其他字段
    // ✗ 注意：这里没有任何 phase.savedGroups 的读取逻辑
    // ✗ 没有任何 saved group 相关字段被写入 snapshotSettings
  };
}
```

**代码证据 1：类型定义层面**

`ExperimentSnapshotSettings`（`packages/shared/types/experiment-snapshot.d.ts:191-217`）：

```typescript
export interface ExperimentSnapshotSettings {
  // ...
  segment: string;           // ✓ 有 segment 字段
  queryFilter: string;       // ✓ 有 queryFilter 字段
  // ✗ 没有 savedGroups、savedGroupTargeting 等相关字段
}
```

**代码证据 2：查询构建层面**

在所有分析查询构建代码中（`experiment-units-query.ts`、`experiment-metric-query.ts`、`SqlIntegration.ts`）：
- 函数参数 `ExperimentUnitsQueryParams` 只有 `segment` 字段，没有 saved group 字段
- 只有 `segment` 参数被传入和使用于构建 SQL
- 没有任何代码读取 `phase.savedGroups` 并在分析 SQL 中加入过滤条件

**代码证据 3：分配逻辑与分析逻辑的隔离**

- `phase.savedGroups` 仅在 `getFeatureDefinition()` → `getParsedCondition()` 中被使用（构建 SDK payload）
- 分析查询链路完全不依赖 `phase.savedGroups`，只依赖曝光表中的数据
- 曝光表中的数据是 SDK 侧经过 saved group 过滤后产生的

### 4.3 Units Table 与 Exposure Query 两条路径

#### 路径 A：Units Table（临时表模式）

**触发条件**（`ExperimentResultsQueryRunner.ts:139-145`）：
```typescript
const useUnitsTable =
  (integration.getSourceProperties().supportsWritingTables &&
    settings.pipelineSettings?.allowWriting &&
    settings.pipelineSettings?.mode === "ephemeral" &&
    !!settings.pipelineSettings?.writeDataset &&
    hasPipelineModeFeature) ??
  false;
```

**执行流程**：
1. 先执行 `getExperimentUnitsTableQuery()`（`SqlIntegration.ts:807-818`）：
   ```sql
   CREATE OR REPLACE TABLE growthbook_tmp_units_xxx AS (
     WITH __experimentUnits AS (
       -- 包含 segment JOIN 过滤
       SELECT ... FROM __experimentExposures e
       JOIN __segment s ON (s.user_id = e.user_id)
       WHERE s.date <= e.timestamp
     )
     SELECT * FROM __experimentUnits
   );
   ```
2. 所有后续指标查询通过 `unitsSource: "exposureTable"` 直接读取该临时表（`experiment-metric-query.ts:305-306`）：
   ```sql
   SELECT ... FROM growthbook_tmp_units_xxx
   -- 无需再次 JOIN segment
   ```

**优势**：多指标共享一次 segment 过滤，避免重复计算。

#### 路径 B：Exposure Query（内嵌模式）

**触发条件**：`useUnitsTable = false`（默认模式）

**执行流程**：每个指标查询内嵌完整的 `__experimentUnits` CTE（`experiment-metric-query.ts:273-277`）：
```sql
-- getExperimentMetricQuery() 生成的 SQL
WITH
  __experimentUnits AS (
    -- 每个指标查询都重新做一次 segment JOIN
    SELECT ... FROM __experimentExposures e
    JOIN __segment s ON (s.user_id = e.user_id)
    WHERE s.date <= e.timestamp
  ),
  __distinctUsers AS (
    SELECT ... FROM __experimentUnits
  ),
  __metric AS (...)
SELECT ...
```

**行为差异**：
- `unitsSource: "exposureQuery"` 时，`idTypeObjects` 会额外包含 segment 的 `userIdType`（`experiment-metric-query.ts:208-213`）
- `unitsSource: "exposureTable"` 时，segment 过滤已在临时表中完成，指标查询不再处理 segment

### 4.4 评估与回溯阶段的协作机制

#### 4.4.1 全流程时序图

```
实验配置阶段
  │
  ├─ 设置 phase.savedGroups → 存储于实验 phase
  └─ 设置 experiment.segment → 存储于实验元数据
  │
分配阶段（SDK / 服务端）
  │
  ├─ 构建 SDK payload 时：
  │   ├─ getParsedCondition(groupMap, phase.condition, phase.savedGroups)
  │   ├─ 生成 rule.condition（含 $inGroup 操作符）
  │   └─ savedGroups 字典随 payload 下发
  │
  └─ SDK 运行时：
      ├─ 用户触发实验评估
      ├─ evalCondition() → $inGroup 匹配 saved group
      ├─ 命中 → 分配变体 + 产生曝光事件
      └─ 未命中 → 跳过实验（无曝光记录）
  │
快照创建阶段（重跑历史快照入口）
  │
  ├─ getSnapshotSettings(experiment, phaseIndex)
  │   ├─ 读取 experiment.segment ✓
  │   └─ 不读取 phase.savedGroups ✗
  │
评估阶段（分析端）
  │
  ├─ 读取 snapshotSettings.segment
  ├─ 构建分析 SQL：
  │   ├─ 从数仓读取曝光表（已被 saved group 过滤）
  │   ├─ getSegmentCTE() → 构建 __segment
  │   ├─ 曝光数据 JOIN __segment → 二次过滤
  │   └─ 对过滤后的用户计算指标
  └─ 输出实验结果
```

#### 4.4.2 协作关键点

**双重过滤的叠加效应**：

```
全体用户
    │
    ├─ [saved group 过滤] → 曝光表仅含命中用户
    │     └─ 分配阶段完成，数据固化在曝光表中
    │     └─ saved group 条件仅影响曝光生成，不进入分析 SQL
    │
    └─ [segment 过滤] → 分析用户集 = 曝光表 ∩ segment
          └─ 分析阶段完成，可修改 segment 重新计算
          └─ segment 条件进入分析 SQL，每次重算都重新执行
```

- **Saved Group** 是"前置过滤"：决定谁能看到实验
- **Segment** 是"后置过滤"：在已曝光用户中筛选分析对象
- 最终分析样本是两者的交集

**时间一致性保障**：

在 `experiment-units-query.ts:239` 中：

```sql
WHERE s.date <= e.timestamp
```

确保用户在**曝光时刻**已经属于该 segment，避免"先曝光后入组"的时间错位问题。

---

## 五、"重跑历史快照"场景下的行为边界

### 5.1 固化 vs 重算判定清单

| 行为 | 固化/重算 | 说明 | 代码位置 |
|------|----------|------|---------|
| **Saved Group 过滤** | 🔒 固化 | 分配时已完成，曝光表中只含命中用户，重跑快照不重新判定 | `packages/sdk-js/src/mongrule.ts:238-241` |
| **变体分配结果** | 🔒 固化 | 已记录在曝光表中，重跑快照不重新 hash 分配 | 曝光表 `variation_id` 字段 |
| **曝光时间戳** | 🔒 固化 | 已记录在曝光表中 | 曝光表 `timestamp` 字段 |
| **Segment 过滤** | 🔄 重算 | 每次重跑都重新执行 `JOIN __segment` | `experiment-units-query.ts:220-224` |
| **Metric 计算** | 🔄 重算 | 每次重跑都重新查询指标数据 | `experiment-metric-query.ts` |
| **归因模型** | 🔄 重算 | 在 `getSnapshotSettings` 中读取后应用 | `experiments.ts:697` |
| **Lookback 窗口** | 🔄 重算 | 在 `getSnapshotSettings` 中读取后应用 | `experiments.ts:698` |
| **维度下钻** | 🔄 重算 | 每次重跑都重新 JOIN 维度表 | `ExperimentResultsQueryRunner.ts:176-178` |
| **统计显著性** | 🔄 重算 | 每次重跑都重新计算 | 分析引擎 |
| **Query Filter** | 🔄 重算 | 在 `getSnapshotSettings` 中读取后应用于曝光查询 | `experiments.ts:701` |

### 5.2 两套机制的行为边界

| 场景 | Saved Group 行为 | Segment 行为 |
|------|-----------------|-------------|
| **修改 saved group 后重跑快照** | 历史曝光数据不变，重跑结果不变 | 无影响（segment 未修改） |
| **修改 segment 后重跑快照** | 无影响（saved group 未修改） | 重新 JOIN 过滤，结果可能变化 |
| **实验运行中修改 saved group** | 新曝光按新规则过滤，历史曝光不变 | 无影响 |
| **实验运行中修改 segment** | 无影响 | 不影响正在进行的分配，仅影响后续分析 |
| **删除 saved group 后重跑** | 无影响（曝光已固化） | 无影响 |
| **删除 segment 后重跑** | 无影响 | 分析时跳过 segment 过滤，样本量增加 |

### 5.3 常见误判

#### ❌ 误判 1："修改 saved group 后重跑快照，结果应该会变"

**真相**：不会变。Saved group 的过滤发生在分配阶段，结果已经固化在曝光表中。回溯分析时不会重新执行 saved group 判定。

**代码证据**：
- `getSnapshotSettings()` 不读取 `phase.savedGroups`（`experiments.ts:429-725`）
- `ExperimentSnapshotSettings` 类型没有 saved group 字段（`experiment-snapshot.d.ts:191-217`）
- 分析 SQL 构建链路中没有任何 saved group 过滤逻辑

#### ❌ 误判 2："saved group 和 segment 都是在分析阶段过滤的，可以互换使用"

**真相**：Saved group 是分配时过滤，segment 是分析时过滤。Saved group 影响曝光样本量，segment 不影响曝光只影响分析样本。

**关键区别**：
- 如果用户被 saved group 排除：不会产生曝光，不计入任何分析
- 如果用户被 segment 排除：仍产生曝光，只是不计入本次分析

#### ❌ 误判 3："重跑快照会重新执行完整的实验分配逻辑"

**真相**：重跑快照只重新执行**分析逻辑**，不重新执行**分配逻辑**。分配逻辑（包括 saved group 判定）只在 SDK 侧发生一次。

#### ❌ 误判 4："使用 units table 模式时，segment 只被计算一次，所以更快"

**真相**：是的，这是 units table 模式的设计目标之一。但需要注意：
- 临时表在所有查询完成后会被删除（`ExperimentResultsQueryRunner.ts:337-354`）
- 如果查询失败，临时表可能残留（取决于 `dropUnitsTable` 配置）

### 5.4 调试建议

**确认 saved group 是否生效**：
- 查看曝光表中的用户数是否符合预期
- 检查 SDK payload 中的 `savedGroups` 字典和 `rule.condition`
- 注意：无法通过重跑快照验证 saved group 逻辑

**确认 segment 是否生效**：
- 直接查看生成的 SQL，搜索 `__segment` 和 `JOIN __segment`
- 修改 segment 定义后重跑，观察样本量变化
- 检查 `s.date <= e.timestamp` 条件是否正确处理时间维度

---

## 六、代码溯源索引

### Saved Group 相关
| 功能 | 文件 | 行号 |
|------|------|------|
| 结构真源（validator） | `packages/shared/src/validators/saved-group.ts` | 9-26 |
| 目标配置结构 | `packages/shared/src/validators/shared.ts` | 38-44 |
| 类型别名（d.ts） | `packages/shared/types/saved-group.d.ts` | 1-39 |
| 条件构建（服务端） | `packages/back-end/src/util/features.ts` | 102-208 |
| 条件评估（SDK） | `packages/sdk-js/src/mongrule.ts` | 238-241 |
| 旧版 SDK 展开逻辑 | `packages/back-end/src/util/features.ts` | 565-568, 757-768 |
| Payload 中的 savedGroups | `packages/sdk-js/src/core.ts` | 828, 1200 |
| phase 保存 savedGroups | `packages/back-end/src/services/experiments.ts` | 2339-2342 |

### Segment 相关
| 功能 | 文件 | 行号 |
|------|------|------|
| 结构真源（validator） | `packages/shared/src/validators/segment.ts` | 9-27 |
| 类型别名（d.ts） | `packages/shared/types/segment.d.ts` | 1-4 |
| Segment CTE 构建 | `packages/back-end/src/integrations/sql/ctes/segment-cte.ts` | 8-83 |
| 实验单元查询中的 segment JOIN | `packages/back-end/src/integrations/sql/queries/experiment-units-query.ts` | 148-163, 220-224 |
| 指标查询中的 segment | `packages/back-end/src/integrations/sql/queries/experiment-metric-query.ts` | 48, 208-213, 273-307 |
| 快照分析时 segment 读取 | `packages/back-end/src/queryRunners/ExperimentResultsQueryRunner.ts` | 115-120 |
| 快照设置构建（不含 saved group） | `packages/back-end/src/services/experiments.ts` | 695-724 |

### 回溯链路相关
| 功能 | 文件 | 行号 |
|------|------|------|
| QueryRunner 入口 | `packages/back-end/src/queryRunners/ExperimentResultsQueryRunner.ts` | 87-357 |
| Units Table 路径判断 | `packages/back-end/src/queryRunners/ExperimentResultsQueryRunner.ts` | 139-145 |
| Units Table 创建 | `packages/back-end/src/integrations/SqlIntegration.ts` | 807-818 |
| 指标查询 unitsSource 分支 | `packages/back-end/src/integrations/sql/queries/experiment-metric-query.ts` | 208-213, 273-307 |
| SnapshotSettings 类型定义 | `packages/shared/types/experiment-snapshot.d.ts` | 191-217 |
| 临时表删除逻辑 | `packages/back-end/src/queryRunners/ExperimentResultsQueryRunner.ts` | 337-354 |
