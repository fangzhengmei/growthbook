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

## 二、完整 CRUD 链路

### 2.1 API 接口层

**文件**: `packages/back-end/src/api/saved-groups/saved-groups.router.ts`

提供的 REST API 端点：

| 方法 | 端点 | 描述 |
|------|------|------|
| GET | `/saved-groups` | 列出所有分组 |
| POST | `/saved-groups` | 创建分组 |
| GET | `/saved-groups/:id` | 获取单个分组 |
| PUT | `/saved-groups/:id` | 更新分组 |
| POST | `/saved-groups/:id/archive` | 归档分组 |
| POST | `/saved-groups/:id/unarchive` | 取消归档 |
| DELETE | `/saved-groups/:id` | 删除分组 |

同时在 `packages/back-end/src/routers/saved-group/saved-group.controller.ts` 中还有旧版 API：
- POST `/saved-groups/:id/add-items` - 批量添加列表项
- POST `/saved-groups/:id/remove-items` - 批量移除列表项
- GET `/saved-groups/:id/references` - 获取分组被引用的资源

### 2.2 创建流程 (`postSavedGroup`)

**文件**: `packages/back-end/src/api/saved-groups/postSavedGroup.ts:7-96`

```
调用链:
POST /saved-groups
  → 权限检查: canCreateSavedGroup()
  → 类型推断: 根据入参推断 type 是 condition 还是 list
  → 类型校验:
     · condition 类型: 调用 validateCondition() 校验 JSON 合法性
     · list 类型: 校验 attributeKey 对应的 datatype 属于 ID_LIST_DATATYPES
  → 调用 models.savedGroups.create() 持久化
  → 转换为 API 格式并返回
```

**创建验证要点**:
- condition 组: 不能指定 attributeKey 或 values，condition 必须非空且有效
- list 组: 必须指定 attributeKey 和 values，不能指定 condition
- ID_LIST_DATATYPES = ["number", "string", "secureString"]

### 2.3 更新流程 (`updateSavedGroup`)

**文件**: `packages/back-end/src/api/saved-groups/updateSavedGroup.ts:9-108`

```
调用链:
PUT /saved-groups/:id
  → getById() 获取现有分组
  → 权限检查: canUpdateSavedGroup()
  → 类型一致性检查:
     · condition 组不能修改 values
     · list 组不能修改 condition
  → 字段对比: 只更新有变化的字段
  → 重新验证:
     · list 组: validateListSize() 检查大小限制
     · condition 组: 重新 validateCondition()
  → 调用 models.savedGroups.update() 持久化
  → 返回更新后的分组
```

**List 组批量增删接口** (`saved-group.controller.ts`):
- `postSavedGroupAddItems`: 合并新 items 到现有 values，去重后更新
- `postSavedGroupRemoveItems`: 从 values 中过滤掉指定 items
- 两个接口都支持审批流（revision 机制）

### 2.4 归档/取消归档流程

**文件**: `packages/back-end/src/api/saved-groups/archiveSavedGroup.ts:26-85`

```
归档流程:
POST /saved-groups/:id/archive
  → 权限检查
  → loadSavedGroupReferences() 检查引用
  → 如果有引用 → 抛出错误 (必须先移除所有引用才能归档)
  → update() 设置 archived: true

取消归档:
POST /saved-groups/:id/unarchive
  → 权限检查
  → update() 设置 archived: false (无需检查引用)
```

**引用检查逻辑** (`services/savedGroups.ts:54-118`):
```typescript
loadSavedGroupReferences(context, id)
  → 检查所有 feature 的 rule.condition / rule.savedGroups
  → 检查所有 experiment 的 phase.condition / phase.savedGroups
  → 检查其他 saved group 的 condition (一级嵌套)
  → 返回: { features, experiments, savedGroups } 引用列表
```

### 2.5 删除流程

**文件**: `packages/back-end/src/api/saved-groups/deleteSavedGroup.ts:4-31`

```
DELETE /saved-groups/:id
  → 权限检查: canDeleteSavedGroup()
  → 必须已归档 (!savedGroup.archived → 错误)
  → deleteById() 从数据库删除
```

**设计考量**: 归档是可逆的，删除是不可逆的，所以强制先归档再删除作为撤销窗口。

### 2.6 变更通知机制

**文件**: `packages/back-end/src/services/savedGroups.ts:15-33`

```typescript
savedGroupUpdated(context)
  → 刷新所有环境/项目的 SDK payload 缓存
  → 因为 saved group 可能跨项目嵌套引用
  → 保守策略: 刷新全部 payload
```

### 2.7 持久化层

**文件**: `packages/back-end/src/models/SavedGroupModel.ts`

核心数据库操作:
- `create(props)`: 插入新文档，生成 `grp_` 前缀 ID
- `getById(id)`: 按 ID 查询
- `getAll(organization?)`: 获取组织下所有分组
- `update(savedGroup, updates)`: 更新字段并记录审计日志
- `deleteById(id)`: 删除文档

---

## 三、match=none 判定差异分析

### 3.1 核心转换函数: `getSavedGroupCondition()`

**文件**: `packages/back-end/src/util/features.ts:102-123`

```typescript
function getSavedGroupCondition(
  groupId: string,
  groupMap: GroupMap,
  include: boolean,  // true=all/any, false=none
): null | ConditionInterface {
  const group = groupMap.get(groupId);
  if (!group) return null;
  
  // condition 类型
  if (group.type === "condition" && group.condition) {
    try {
      const cond = JSON.parse(group.condition);
      return include ? cond : { $not: cond };  // 关键差异
    } catch (e) {
      return null;
    }
  }
  
  // list 类型
  if (!group.attributeKey) return null;
  return {
    [group.attributeKey]: { 
      [include ? "$inGroup" : "$notInGroup"]: groupId  // 关键差异
    },
  };
}
```

### 3.2 两类分组在 match=none 时的展开差异

| 维度 | condition 组 (include=false) | list 组 (include=false) |
|------|-----------------------------|------------------------|
| 输出结构 | `{ $not: <group_condition> }` | `{ <attributeKey>: { $notInGroup: groupId } }` |
| 取反层级 | 对整个条件对象取反 | 对属性成员检查取反 |
| SDK 处理 | `evalCondition()` 处理 `$not` | `evalOperatorCondition()` 处理 `$notInGroup` |
| 空值行为 | condition 为空 → 返回 null (跳过该分组) | values 为空且 `useEmptyListGroup=false` → 跳过 |

### 3.3 运行时评估差异

**condition 组 match=none 评估路径**:
```
SDK 端 evalCondition(attributes, { $not: { country: "US", role: "admin" } })
  → 遇到 $not 键
  → 递归 evalCondition(attributes, { country: "US", role: "admin" })
  → 返回 true → $not 取反为 false → 不命中规则
```

**list 组 match=none 评估路径**:
```
SDK 端 evalCondition(attributes, { userId: { $notInGroup: "grp_vip" } })
  → 遇到属性键 "userId"
  → 调用 evalOperatorCondition("$notInGroup", "user123", "grp_vip", savedGroups)
  → !isIn("user123", savedGroups["grp_vip"] || [])
  → 如果不在列表中 → 返回 true → 命中规则
```

### 3.4 语义差异的边界场景

**场景 1: condition 组条件为 `{}` (空条件)**
- 空条件 `{}` 在 evalCondition 中返回 true
- `{ $not: {} }` → `!true` → `false`
- 结果: match=none 对空 condition 组恒不命中

**场景 2: list 组 values 为空**
- `useEmptyListGroup=false` → 在 getParsedCondition 中被过滤掉，不参与条件
- `useEmptyListGroup=true` → `$notInGroup` → `!isIn(val, [])` → `!false` → `true`
- 结果: match=none 对允许空的 list 组恒命中

**场景 3: condition 组嵌套 OR**
```
组条件: { $or: [{ country: "US" }, { country: "CN" }] }
match=none 展开: { $not: { $or: [{ country: "US" }, { country: "CN" }] } }
等价于 (德摩根定律): { $nor: [{ country: "US" }, { country: "CN" }] }
```

---

## 四、引用展开机制

Saved Groups 的展开分为**两个阶段**：**服务端预处理** 和 **SDK运行时处理**。

### 4.1 第一阶段：服务端条件解析与嵌套展开

#### 4.1.1 引用方式

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

#### 4.1.2 解析入口: `getParsedCondition()`

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

#### 4.1.3 嵌套展开: `expandNestedSavedGroups()`

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

### 4.2 第二阶段：服务端值替换 (旧版SDK兼容)

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

### 4.3 递归遍历工具: `recursiveWalk()`

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

## 五、运行时命中评估

### 5.1 SDK 端数据接收

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

### 5.2 条件评估入口: `evalCondition()`

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

### 5.3 分组操作符处理: `evalOperatorCondition()`

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

### 5.4 成员检查: `isIn()`

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

### 5.5 Feature 评估流程中的调用链

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

### 5.6 实验侧命中路径

**文件**: `packages/sdk-js/src/core.ts:385-580`

```typescript
export function runExperiment<T>(
  experiment: Experiment<T>,
  featureId: string | null,
  ctx: EvalContext,
): { result: Result<T>; trackingCall?: Promise<void> }
```

实验评估的完整调用链:

```
runExperiment(experiment, ctx)
  │
  ├─ 1. 基础检查 (<2 variations, disabled, draft/inactive) → 不命中
  │
  ├─ 2. URL 匹配检查 (urlPatterns)
  │
  ├─ 3. Querystring 强制分配 (qsOverride)
  │
  ├─ 4. DevTools 强制分配 (forcedVariations)
  │
  ├─ 5. 获取 hash 属性 (hashAttribute → hashValue)
  │   → 如果 hashValue 为空 → 不命中
  │
  ├─ 6. 粘性桶检查 (sticky bucketing)
  │   → 如果已有分配 → 使用已有 variation
  │
  ├─ 7. 过滤器/命名空间检查 (filters / namespace)
  │   → isFilteredOut() → 不命中
  │
  ├─ 8. 自定义 include 函数检查
  │
  ├─ 9. 条件检查 (experiment.condition)  ← Saved Group 在这里评估
  │   └─ conditionPasses(experiment.condition, ctx)
  │       └─ evalCondition(attributes, condition, ctx.global.savedGroups)
  │           ├─ $and / $or / $not 递归处理
  │           └─ evalOperatorCondition()
  │               ├─ $inGroup → isIn(attr, savedGroups[groupId])
  │               └─ $notInGroup → !isIn(attr, savedGroups[groupId])
  │
  ├─ 10. 前置条件检查 (parentConditions)
  │
  ├─ 11. 哈希分配 (hash(hashValue + experiment.key) % 1000 < coverage * 1000)
  │
  └─ 12. 返回分配结果 (variationId, inExperiment=true)
```

**实验中 Saved Group 引用的两种方式**:

1. **通过 `experiment.condition` 字段**: 与 Feature Rule 相同，condition JSON 中可以包含 `$inGroup` 操作符

2. **通过 `experiment.phases[].savedGroups` 字段**: 在服务端生成 SDK payload 时，`phases[i].savedGroups` 会被 `getParsedCondition()` 转换为 condition 片段并合并到 `experiment.condition` 中

**实验侧与 Feature 侧的关键差异**:
- 实验的 condition 评估发生在**哈希分配之前**，是准入门槛
- Feature 的 condition 评估发生在**规则遍历中**，每个 rule 独立评估
- 实验支持 `parentConditions`（依赖其他 Feature 的状态）
- 实验有更复杂的分配流程（粘性桶、URL 强制、QS 强制等）

---

## 六、完整流程图

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

## 七、关键代码位置汇总

### 7.1 CRUD 接口层

| 功能模块 | 文件路径 | 关键函数 |
|---------|---------|---------|
| 路由注册 | `packages/back-end/src/api/saved-groups/saved-groups.router.ts` | `savedGroupsRoutes` |
| 创建分组 | `packages/back-end/src/api/saved-groups/postSavedGroup.ts` | `postSavedGroup` |
| 更新分组 | `packages/back-end/src/api/saved-groups/updateSavedGroup.ts` | `updateSavedGroup` |
| 归档/取消归档 | `packages/back-end/src/api/saved-groups/archiveSavedGroup.ts` | `archiveSavedGroup`, `unarchiveSavedGroup` |
| 删除分组 | `packages/back-end/src/api/saved-groups/deleteSavedGroup.ts` | `deleteSavedGroup` |
| 列表项增删 | `packages/back-end/src/routers/saved-group/saved-group.controller.ts` | `postSavedGroupAddItems`, `postSavedGroupRemoveItems` |
| 引用检查 | `packages/back-end/src/services/savedGroups.ts` | `loadSavedGroupReferences` |
| 变更通知 | `packages/back-end/src/services/savedGroups.ts` | `savedGroupUpdated` |

### 7.2 核心逻辑层

| 功能模块 | 文件路径 | 关键函数 |
|---------|---------|---------|
| 类型定义 | `packages/shared/types/saved-group.d.ts` | `SavedGroupInterface` |
| 数据验证 | `packages/shared/src/validators/saved-group.ts` | `savedGroupValidator` |
| 条件解析 | `packages/back-end/src/util/features.ts` | `getParsedCondition`, `getSavedGroupCondition` |
| 嵌套展开 | `packages/shared/src/sdk-versioning/sdk-payload.ts` | `expandNestedSavedGroups` |
| 值替换 (旧SDK兼容) | `packages/shared/src/sdk-versioning/sdk-payload.ts` | `replaceSavedGroups` |
| 递归遍历工具 | `packages/shared/src/util/index.ts` | `recursiveWalk` |
| 值类型转换 | `packages/shared/src/util/saved-groups.ts` | `getTypedSavedGroupValues`, `getSavedGroupValueType` |
| 引用检测 | `packages/shared/src/util/index.ts` | `featuresReferencingSavedGroups`, `experimentsReferencingSavedGroups` |

### 7.3 SDK 运行时层

| 功能模块 | 文件路径 | 关键函数 |
|---------|---------|---------|
| 条件评估入口 | `packages/sdk-js/src/mongrule.ts` | `evalCondition` |
| 操作符处理 | `packages/sdk-js/src/mongrule.ts` | `evalOperatorCondition` |
| 成员检查 | `packages/sdk-js/src/mongrule.ts` | `isIn` |
| Feature 评估 | `packages/sdk-js/src/core.ts` | `evalFeature`, `conditionPasses` |
| 实验评估 | `packages/sdk-js/src/core.ts` | `runExperiment` |
| 数据模型 | `packages/back-end/src/models/SavedGroupModel.ts` | `SavedGroupModel` |

---

## 八、设计要点总结

### 8.1 CRUD 链路设计

1. **归档前置删除**: 强制先归档再删除，提供撤销窗口，防止误删
2. **引用完整性**: 归档前强制检查所有引用（features/experiments/嵌套savedGroups），保证 archived 状态的分组不会被使用
3. **审批流集成**: list 组的批量增删操作走 revision 机制，支持审批流程
4. **保守刷新策略**: saved group 变更时刷新全部 SDK payload 缓存，因为可能存在跨项目的嵌套引用
5. **幂等设计**: 归档/取消归档操作在已处于目标状态时直接返回，避免重复写入

### 8.2 match=none 判定设计

6. **结构差异**: condition 组用 `{ $not: cond }` 整体取反，list 组用 `$notInGroup` 属性级取反，符合各自的语义场景
7. **空值处理差异**: 
   - 空 condition → 跳过该分组（不参与条件）
   - 空 list（`useEmptyListGroup=false`）→ 跳过
   - 空 list（`useEmptyListGroup=true`）→ `$notInGroup` 恒为 true
8. **德摩根定律隐式应用**: `$not` 包裹 `$or` 条件时等价于 `$nor`，但由 SDK 端 `evalCondition` 递归处理，无需服务端转换

### 8.3 实验侧命中设计

9. **分层准入**: Saved Group 条件检查发生在哈希分配之前，作为准入门槛，避免不必要的哈希计算
10. **统一评估逻辑**: 实验与 Feature 共享相同的 `evalCondition` 和 `evalOperatorCondition` 逻辑，确保判定一致性
11. **双路径引用**: 既支持 `experiment.condition` 内嵌 `$inGroup`，也支持 `phase.savedGroups` 声明式引用，给用户灵活选择
12. **前置条件依赖**: 实验的 `parentConditions` 支持依赖其他 Feature 的状态，可构建复杂的准入规则

### 8.4 通用设计原则

13. **两级展开策略**: 服务端负责嵌套引用展开，SDK 负责最终成员检查，平衡了 payload 大小和灵活性
14. **循环/深度防护**: 通过 `visited` Set 和 `MAX_SAVED_GROUP_DEPTH=10` 防止无限递归
15. **错误降级**: 无效引用通过注入 always-false 条件实现优雅降级（不命中该分组）
16. **向后兼容**: 通过 SDK capability 检测决定是展开值还是保留操作符
17. **原地修改**: `recursiveWalk` 的原地修改特性使得嵌套展开可以单遍完成
18. **类型感知**: list 类型分组会根据 `attributeKey` 的数据类型进行值转换（string/number）
