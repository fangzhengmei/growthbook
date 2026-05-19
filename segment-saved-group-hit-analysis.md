# Segment 与 Saved Group 命中判定机制分析

## 一、核心差异概述

| 维度 | Saved Group | Segment |
|------|-------------|---------|
| **执行层** | SDK 侧 / 服务端（分配时） | 分析端（评估时） |
| **判定方式** | 按 ID 列表 / 条件即时匹配 | 按 SQL 子集关联过滤 |
| **生效时机** | 实验分配阶段 | 实验评估 / 回溯阶段 |
| **数据来源** | GrowthBook 内部维护的 ID 列表 | 数仓用户表 / 事实表 |
| **可回溯性** | 分配后固定，无法回溯修改 | 分析时可更换 segment 重新计算 |
| **代码位置** | `packages/sdk-js/src/mongrule.ts` + `packages/back-end/src/util/features.ts` | `packages/back-end/src/integrations/sql/queries/experiment-units-query.ts` + `packages/back-end/src/integrations/sql/ctes/segment-cte.ts` |

---

## 二、Saved Group 命中判定机制

### 2.1 数据结构

Saved Group 有两种类型（`packages/shared/types/saved-group.d.ts`）：

```typescript
type SavedGroupType = "list" | "condition";

interface SavedGroupInterface {
  id: string;
  groupName: string;
  type: SavedGroupType;
  attributeKey?: string;      // list 类型：匹配的用户属性字段
  values?: string[];          // list 类型：ID 列表
  condition?: string;         // condition 类型：JSON 条件表达式
  useEmptyListGroup?: boolean;
  // ...
}
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

### 3.1 数据结构

Segment 同样有两种类型（`packages/shared/types/segment.d.ts`）：

```typescript
interface SegmentInterface {
  id: string;
  name: string;
  type: "SQL" | "FACT";
  userIdType: "user_id" | "anonymous_id";  // segment 使用的 ID 类型
  datasource: string;                      // 关联数据源
  
  // SQL 类型
  sql?: string;                // 用户自定义 SQL，需返回 (user_id, date) 两列
  
  // FACT 类型
  factTableId?: string;        // 关联事实表
  filters?: Filter[];          // 过滤条件
}
```

### 3.2 SQL 过滤逻辑（分析端）

Segment 过滤发生在实验单元查询 `__experimentUnits` CTE 中（`packages/back-end/src/integrations/sql/queries/experiment-units-query.ts:148-163, 218-239`）：

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

`getSegmentCTE()` 负责生成 segment 的 CTE SQL（`packages/back-end/src/integrations/sql/ctes/segment-cte.ts`）：

```typescript
function getSegmentCTE(
  dialect: SqlDialect,
  segment: SegmentInterface,
  baseIdType: string,
  idJoinMap: Record<string, string>,
  factTableMap: FactTableMap,
): string {
  if (segment.type === "SQL") {
    segmentSql = segment.sql;
  } else {
    // FACT 类型：基于事实表 + 过滤器生成 SQL
    segmentSql = getFactSegmentCTE(dialect, {
      factTable,
      filters: segment.filters,
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

## 四、评估与回溯阶段的协作机制

### 4.1 全流程时序图

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
评估阶段（分析端）
  │
  ├─ 读取实验配置：experiment.segment
  ├─ 构建分析 SQL：
  │   ├─ 从数仓读取曝光表（已被 saved group 过滤）
  │   ├─ getSegmentCTE() → 构建 __segment
  │   ├─ 曝光数据 JOIN __segment → 二次过滤
  │   └─ 对过滤后的用户计算指标
  └─ 输出实验结果
```

### 4.2 协作关键点

#### 4.2.1 双重过滤的叠加效应

```
全体用户
    │
    ├─ [saved group 过滤] → 曝光表仅含命中用户
    │     └─ 分配阶段完成，数据固化在曝光表中
    │
    └─ [segment 过滤] → 分析用户集 = 曝光表 ∩ segment
          └─ 分析阶段完成，可修改 segment 重新计算
```

- **Saved Group** 是"前置过滤"：决定谁能看到实验
- **Segment** 是"后置过滤"：在已曝光用户中筛选分析对象
- 最终分析样本是两者的交集

#### 4.2.2 时间一致性保障

在 `experiment-units-query.ts:239` 中：

```sql
WHERE s.date <= e.timestamp
```

确保用户在**曝光时刻**已经属于该 segment，避免"先曝光后入组"的时间错位问题。

#### 4.2.3 回溯分析的灵活性

| 机制 | 分配后可修改？ | 对已产生曝光的影响 |
|------|---------------|-------------------|
| Saved Group | 不建议 | 修改后新用户按新规则匹配，但历史曝光数据不变 |
| Segment | 支持 | 可更换 segment 重新分析相同的曝光数据，得到不同的分析结果 |

#### 4.2.4 ID 类型对齐

- Saved Group 使用 `attributeKey` 指定匹配字段（如 `id`、`email`），SDK 直接匹配
- Segment 使用 `userIdType`，分析端通过 `idJoinMap` 处理跨类型 join（如 `anonymous_id` → `user_id`）

---

## 五、代码溯源索引

### Saved Group 相关
| 功能 | 文件 | 行号 |
|------|------|------|
| 类型定义 | `packages/shared/types/saved-group.d.ts` | 1-39 |
| 条件构建（服务端） | `packages/back-end/src/util/features.ts` | 102-208 |
| 条件评估（SDK） | `packages/sdk-js/src/mongrule.ts` | 238-241 |
| 旧版 SDK 展开逻辑 | `packages/back-end/src/util/features.ts` | 565-568, 757-768 |
| Payload 中的 savedGroups | `packages/sdk-js/src/core.ts` | 828, 1200 |

### Segment 相关
| 功能 | 文件 | 行号 |
|------|------|------|
| 类型定义 | `packages/shared/types/segment.d.ts` | 1-4 |
| Segment CTE 构建 | `packages/back-end/src/integrations/sql/ctes/segment-cte.ts` | 8-83 |
| 实验单元查询中的 segment JOIN | `packages/back-end/src/integrations/sql/queries/experiment-units-query.ts` | 148-163, 220-239 |
| 指标查询中的 segment | `packages/back-end/src/integrations/sql/queries/experiment-metric-query.ts` | 48, 211 |
| 快照分析时 segment 读取 | `packages/back-end/src/queryRunners/ExperimentResultsQueryRunner.ts` | 115-120 |

### 协作相关
| 功能 | 文件 | 行号 |
|------|------|------|
| 实验 phase 保存 savedGroups | `packages/back-end/src/services/experiments.ts` | 2339-2342 |
| 实验保存 segment | `packages/back-end/src/services/experiments.ts` | 700, 2358 |
| 快照设置包含 segment | `packages/shared/types/experiment-snapshot.d.ts` | 84, 204 |
