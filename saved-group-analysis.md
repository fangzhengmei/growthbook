# GrowthBook Saved Groups 代码分析

## 概述

Saved Groups（用户分组）是 GrowthBook 中用于复用的用户分组机制，可以在多个实验/Feature Flag 中被引用。本文档分析其存储定义、引用展开与运行时命中的完整代码流程。

---

## 一、存储定义与数据结构

### 1.1 核心数据模型

**文件**: `packages/shared/types/saved-group.d.ts` + `packages/shared/src/validators/saved-group.ts`

```typescript
// 核心类型定义
export type SavedGroupType = "condition" | "list";

export interface SavedGroupInterface {
  id: string;                     // 分组唯一ID (grp_前缀)
  organization: string;           // 所属组织ID
  groupName: string;              // 分组名称
  owner: string;                  // 所有者
  type: SavedGroupType;           // 类型: condition 或 list
  condition?: string;             // condition类型: JSON字符串条件
  attributeKey?: string;          // list类型: 关联的属性键
  values?: string[];              // list类型: ID列表值
  dateUpdated: Date;              // 更新时间
  dateCreated: Date;              // 创建时间
  description?: string;           // 描述
  projects?: string[];            // 关联项目
  useEmptyListGroup?: boolean;    // 是否允许空列表
  archived?: boolean;             // 是否归档
}
```

### 1.2 数据库模型

**文件**: `packages/back-end/src/models/SavedGroupModel.ts`

- 集合名: `savedgroups`
- ID前缀: `grp_`
- 审计日志事件: `savedGroup.created`, `savedGroup.updated`, `savedGroup.deleted`

**关键迁移逻辑** (`migrateSavedGroup`):
- 旧版 `source` 字段迁移到 `type` 字段
- `source: "runtime"` → `type: "condition"`
- `source: "inline"` → `type: "list"`

### 1.3 辅助数据结构

```typescript
// SDK Payload 输出格式 (仅list类型的values)
export type SavedGroupsValues = Record<string, (string | number)[]>;

// 内部处理用的GroupMap (条件展开时使用)
export type GroupMap = Map<
  string,
  Pick<SavedGroupInterface, "type" | "condition" | "attributeKey" | "useEmptyListGroup"> & {
    values?: (string | number)[];
  }
>;
```

---

## 二、引用展开机制

Saved Groups 的展开分为**两个阶段**：**服务端预处理** 和 **SDK运行时处理**。

### 2.1 第一阶段：服务端条件解析与嵌套展开

#### 2.1.1 引用方式

Feature Rule 或 Experiment Phase 通过两种方式引用 Saved Group：

1. **显式 savedGroups 字段**（推荐）
```typescript
// FeatureRule 或 ExperimentPhase 中
{
  savedGroups: [
    { ids: ["grp_1", "grp_2"], match: "all" | "any" | "none" }
  ]
}
```

2. **Condition 内嵌 `$savedGroups` 操作符**
```json
{
  "country": "US",
  "$savedGroups": ["grp_internal_users"]
}
```

3. **Condition 内嵌 `$inGroup` / `$notInGroup` 操作符**
```json
{
  "userId": { "$inGroup": "grp_vip_users" }
}
```

#### 2.1.2 解析入口: `getParsedCondition()`

**文件**: `packages/back-end/src/util/features.ts:125-208`

```
调用链:
getFeatureDefinition()
  → getParsedCondition(groupMap, condition, savedGroups)
    → 1. 解析原始JSON condition
    → 2. 处理 savedGroups 数组 (match: all/any/none)
    → 3. 对每个条件调用 recursiveWalk(expandNestedSavedGroups)
```

**处理 savedGroups 数组逻辑**:
- `match: "all"` → 每个分组条件用 `$and` 连接
- `match: "any"` → 分组条件用 `$or` 包裹后再 `$and`
- `match: "none"` → 每个分组条件用 `$not` 包裹后 `$and`

#### 2.1.3 嵌套展开: `expandNestedSavedGroups()`

**文件**: `packages/shared/src/sdk-versioning/sdk-payload.ts:113-285`

这是最复杂的递归展开逻辑，用于处理条件中的 `$savedGroups` 关键字。

```
核心逻辑:
1. 遇到 $savedGroups 键时，删除原键
2. 遍历每个 groupId:
   a. 深度检查 (MAX_SAVED_GROUP_DEPTH = 10) → 超限标记 SAVED_GROUP_ERROR_MAX_DEPTH
   b. 循环检查 (visited Set) → 检测到循环标记 SAVED_GROUP_ERROR_CYCLE
   c. 未知 group → 标记 SAVED_GROUP_ERROR_UNKNOWN
   d. list类型: 转换为 { attributeKey: { $inGroup: groupId } }
   e. condition类型: 递归解析其 condition JSON 并展开其中的 $savedGroups
3. 用 $and 合并所有条件到原对象中
```

**错误标记机制**（恒为 false 的条件）:
```typescript
export const SAVED_GROUP_ERROR_MAX_DEPTH = "__sgMaxDepth__";
export const SAVED_GROUP_ERROR_CYCLE = "__sgCycle__";
export const SAVED_GROUP_ERROR_INVALID = "__sgInvalid__";
export const SAVED_GROUP_ERROR_UNKNOWN = "__sgUnknown__";
```

### 2.2 第二阶段：服务端值替换 (旧版SDK兼容)

**文件**: `packages/shared/src/sdk-versioning/sdk-payload.ts:288-310`

当 SDK 不支持 `savedGroupReferences` capability 时（旧版 SDK），服务端会在 payload 生成时将 `$inGroup` / `$notInGroup` 直接替换为实际的 values 列表。

```typescript
// 替换映射
const savedGroupOperatorReplacements = {
  $inGroup: "$in",
  $notInGroup: "$nin",
};

export const replaceSavedGroups = (savedGroups, organization) => ([key, value], object) => {
  if (key === "$inGroup" || key === "$notInGroup") {
    const group = savedGroups[value];
    const values = group ? getTypedSavedGroupValues(group.values, getSavedGroupValueType(group, organization)) : [];
    object[savedGroupOperatorReplacements[key]] = values;
    delete object[key];
  }
};
```

**触发条件** (`shouldExpandSavedGroups`):
- 显式传入了 `savedGroupsMap`
- 且 `savedGroupReferencesEnabled === false` 或 SDK 不支持 `savedGroupReferences` capability

### 2.3 递归遍历工具: `recursiveWalk()`

**文件**: `packages/shared/src/util/index.ts:489-500`

整个展开机制依赖于这个通用的深度优先遍历函数：

```typescript
export const recursiveWalk = (object: any, onNode: NodeHandler) => {
  if (object === null || typeof object !== "object") return;
  
  Object.entries(object).forEach((node) => {
    onNode(node, object);           // 先处理当前节点 (可修改object)
    recursiveWalk(object[node[0]], onNode);  // 再递归子节点 (使用修改后的值)
  });
};
```

**关键特性**: 处理函数可以**原地修改** object，修改后的值会被后续递归使用。这使得 `expandNestedSavedGroups` 能动态插入新条件并继续展开其中的嵌套引用。

---

## 三、运行时命中评估

### 3.1 SDK 端数据接收

SDK 通过 payload 接收两部分数据：

1. **`features`**: 已展开嵌套 saved group 引用的 feature 规则（条件中可能包含 `$inGroup` 操作符）
2. **`savedGroups`**: `SavedGroupsValues` 格式的分组值列表（仅 list 类型）

```typescript
// SDK 初始化时传入
const gb = new GrowthBook({
  features: {...},
  savedGroups: {
    "grp_vip": ["user1", "user2", "user3"],
    "grp_internal": ["emp1001", "emp1002"]
  }
});
```

### 3.2 条件评估入口: `evalCondition()`

**文件**: `packages/sdk-js/src/mongrule.ts:17-46`

```typescript
export function evalCondition(
  obj: TestedObj,           // 用户属性对象 { id: "user1", country: "US", ... }
  condition: ConditionInterface,  // 特征规则条件
  savedGroups?: SavedGroupsValues  // 分组值映射
): boolean {
  savedGroups = savedGroups || {};
  
  for (const [k, v] of Object.entries(condition)) {
    switch (k) {
      case "$or":  return evalOr(obj, v as ConditionInterface[], savedGroups);
      case "$nor": return !evalOr(obj, v as ConditionInterface[], savedGroups);
      case "$and": return evalAnd(obj, v as ConditionInterface[], savedGroups);
      case "$not": return !evalCondition(obj, v as ConditionInterface, savedGroups);
      default:
        // 路径属性评估: k 是属性路径 (如 "userId", "device.browser")
        if (!evalConditionValue(v, getPath(obj, k), savedGroups)) return false;
    }
  }
  return true;
}
```

### 3.3 分组操作符处理: `evalOperatorCondition()`

**文件**: `packages/sdk-js/src/mongrule.ts:198-279`

```typescript
function evalOperatorCondition(
  operator: Operator,
  actual: any,           // 用户属性的实际值
  expected: any,         // 条件中的期望值 (对于 $inGroup 是 groupId)
  savedGroups: SavedGroupsValues,
): boolean {
  switch (operator) {
    // ... 其他操作符
    
    case "$in":
      if (!Array.isArray(expected)) return false;
      return isIn(actual, expected);
      
    case "$inGroup":
      // 关键: 从 savedGroups 中取出该分组的 values 数组，然后用 isIn 检查
      return isIn(actual, savedGroups[expected] || []);
      
    case "$notInGroup":
      return !isIn(actual, savedGroups[expected] || []);
      
    case "$nin":
      if (!Array.isArray(expected)) return false;
      return !isIn(actual, expected);
      
    // ... 其他操作符
  }
}
```

### 3.4 成员检查: `isIn()`

**文件**: `packages/sdk-js/src/mongrule.ts:152-173`

支持单值和数组属性的成员检查：

```typescript
function isIn(
  actual: any,           // 用户属性值 (单值或数组)
  expected: Array<any>,  // 分组值列表
  insensitive: boolean = false,
): boolean {
  if (insensitive) {
    const caseFold = (val: any) => typeof val === "string" ? val.toLowerCase() : val;
    // 用户属性是数组 → 求交集
    if (Array.isArray(actual)) {
      return actual.some((el) => expected.some((exp) => caseFold(el) === caseFold(exp)));
    }
    return expected.some((exp) => caseFold(actual) === caseFold(exp));
  }
  
  // 用户属性是数组 → 用 includes 检查交集
  if (Array.isArray(actual)) {
    return actual.some((el) => expected.includes(el));
  }
  return expected.includes(actual);
}
```

### 3.5 Feature 评估流程中的调用链

**文件**: `packages/sdk-js/src/core.ts`

```
evalFeature(id, ctx)
  → 遍历 feature.rules
     → 对每个 rule:
        1. 评估 parentConditions (前置条件)
        2. 评估 filters (命名空间过滤)
        3. 评估 condition (用户属性匹配)
           → conditionPasses(rule.condition, ctx)
              → evalCondition(attributes, condition, ctx.global.savedGroups)
        4. 评估 rollout 百分比
        5. 分配 variation 或返回 force 值
```

---

## 四、完整流程图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         服务端 (Backend)                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Feature Rule / Experiment Phase                                        │
│  ┌──────────────────────────────────────────────────────────┐           │
│  │  { condition: "{...}", savedGroups: [{ids, match}] }    │           │
│  └──────────────────────────────────┬───────────────────────┘           │
│                                     │                                   │
│                                     ▼                                   │
│  getParsedCondition()               │                                   │
│  ├─ 解析 condition JSON             │                                   │
│  ├─ 处理 savedGroups 数组 → 转换为 condition 片段                   │
│  └─ recursiveWalk(expandNestedSavedGroups) ◄───────────┐                │
│                          │                              │                │
│                          ▼                              │ 递归            │
│  展开 $savedGroups 引用                                │                │
│  ├─ list 类型 → { attr: { $inGroup: id } }             │                │
│  ├─ condition 类型 → 解析 condition JSON → 继续展开 ───┘                │
│  └─ 错误处理 (循环/超限/未知) → 注入 always-false 条件                  │
│                          │                                               │
│                          ▼                                               │
│  shouldExpandSavedGroups?                                               │
│  ├─ Yes (旧SDK) → recursiveWalk(replaceSavedGroups)                     │
│  │              → 将 $inGroup 替换为 $in + values 数组                 │
│  └─ No (新SDK)  → 保留 $inGroup 操作符                                  │
│                          │                                               │
│                          ▼                                               │
│  SDK Payload                                                             │
│  { features: {...}, savedGroups: { grp_1: [...], ...} }                 │
│                          │                                               │
└──────────────────────────┼───────────────────────────────────────────────┘
                           │  HTTP/CDN
┌──────────────────────────┼───────────────────────────────────────────────┐
│                         SDK (Client)                                    │
├──────────────────────────┼───────────────────────────────────────────────┤
│                          ▼                                               │
│  GrowthBook 初始化                                                      │
│  ├─ features: 接收展开后的规则 (可能含 $inGroup)                        │
│  └─ savedGroups: 接收 grp_id → values 映射                              │
│                          │                                               │
│                          ▼                                               │
│  evalFeature("my_feature")                                              │
│  ├─ 遍历 rules                                                          │
│  │   ├─ conditionPasses(condition, ctx)                                 │
│  │   │   └─ evalCondition(attributes, condition, savedGroups)           │
│  │   │       ├─ $and / $or / $not 递归                                  │
│  │   │       └─ evalOperatorCondition()                                 │
│  │   │           ├─ $inGroup → isIn(attr, savedGroups[groupId])        │
│  │   │           └─ $nin / $eq / ...                                    │
│  │   ├─ filters (命名空间)                                              │
│  │   └─ rollout (百分比)                                                │
│  └─ 返回结果 (value + source)                                          │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 五、关键代码位置汇总

| 功能模块 | 文件路径 | 关键函数 |
|---------|---------|---------|
| 类型定义 | `packages/shared/types/saved-group.d.ts` | `SavedGroupInterface` |
| 数据验证 | `packages/shared/src/validators/saved-group.ts` | `savedGroupValidator` |
| 嵌套展开 | `packages/shared/src/sdk-versioning/sdk-payload.ts` | `expandNestedSavedGroups` |
| 值替换 | `packages/shared/src/sdk-versioning/sdk-payload.ts` | `replaceSavedGroups` |
| 条件解析 | `packages/back-end/src/util/features.ts` | `getParsedCondition` |
| Feature定义生成 | `packages/back-end/src/util/features.ts` | `getFeatureDefinition` |
| 递归遍历工具 | `packages/shared/src/util/index.ts` | `recursiveWalk` |
| SDK条件评估 | `packages/sdk-js/src/mongrule.ts` | `evalCondition`, `evalOperatorCondition` |
| SDK Feature评估 | `packages/sdk-js/src/core.ts` | `evalFeature`, `conditionPasses` |
| 数据模型 | `packages/back-end/src/models/SavedGroupModel.ts` | `SavedGroupModel` |

---

## 六、设计要点总结

1. **两级展开策略**: 服务端负责嵌套引用展开，SDK 负责最终成员检查，平衡了 payload 大小和灵活性
2. **循环/深度防护**: 通过 `visited` Set 和 `MAX_SAVED_GROUP_DEPTH=10` 防止无限递归
3. **错误降级**: 无效引用通过注入 always-false 条件实现优雅降级（不命中该分组）
4. **向后兼容**: 通过 SDK capability 检测决定是展开值还是保留操作符
5. **原地修改**: `recursiveWalk` 的原地修改特性使得嵌套展开可以单遍完成
6. **类型感知**: list 类型分组会根据 `attributeKey` 的数据类型进行值转换（string/number）
