# GrowthBook Holdout 分组与全局排除规则实现分析

## 一、Holdout 分组机制

### 1.1 Holdout 数据结构

**文件：** `packages/shared/src/validators/holdout.ts`

```typescript
export const holdoutValidator = z.object({
  id: z.string(),
  organization: z.string(),
  dateCreated: z.date(),
  dateUpdated: z.date(),
  projects: z.array(z.string()),
  name: z.string(),
  skipAsDefaultHoldout: z.boolean().optional(),
  experimentId: z.string(),
  linkedExperiments: z.record(z.string(), holdoutLinkedItemValidator),
  linkedFeatures: z.record(z.string(), holdoutLinkedItemValidator),
  environmentSettings: z.record(z.string(), featureEnvironment),
  analysisStartDate: z.date().optional(),
  statusUpdateSchedule: statusUpdateScheduleValidator.optional().nullable(),
  nextScheduledStatusUpdate: nextScheduledStatusUpdateValidator.optional().nullable(),
}).strict();
```

### 1.2 Feature 中的 Holdout 关联

Feature 通过 `holdout` 字段关联 holdout：

```typescript
interface FeatureInterface {
  holdout?: {
    id: string;      // holdout 的 ID
    value: string;   // 当用户在 holdout 控制组时返回的值
  };
}
```

### 1.3 Holdout 在规则树中的表达形式

**文件：** `packages/back-end/src/util/features.ts:592-616`

Holdout 规则在 SDK payload 中被实现为 **非阻塞先决条件规则**，插入在 feature 规则列表的 **最前面**：

```typescript
const holdoutRule: FeatureDefinitionRule[] =
  hasPrerequisites &&
  feature.holdout &&
  holdoutsMap &&
  holdoutsMap.get(feature.holdout.id)?.holdout.environmentSettings?.[
    environment
  ]?.enabled
    ? [
        {
          ...(includeRuleIds
            ? { id: `holdout_${md5(feature.id + feature.holdout.id)}` }
            : {}),
          parentConditions: [
            {
              id: getHoldoutFeatureDefId(feature.holdout.id),  // "$holdout:{holdoutId}"
              condition: { value: "holdoutcontrol" },
              // 注意：没有 gate: true —— 这是非阻塞型先决条件
            },
          ],
          force: getJSONValue(feature.valueType, feature.holdout.value),
        },
      ]
    : [];
```

**关键区别 — Holdout 规则 vs Feature-level Prerequisites：**

| 属性 | Holdout 规则 | Feature-level Prerequisite |
|-----|-------------|---------------------------|
| `gate` 字段 | **无**（未设置） | `gate: true` |
| 条件失败时 | `continue rules` — 跳过当前规则，继续下一条 | `return null` — 整个 feature 返回空 |
| 行为 | 如果用户不在 holdout 控制组，继续执行业务规则 | 如果前置条件不满足，整个 feature 不返回值 |

**代码证据（feature-level prerequisites 有 `gate: true`）：**
**文件：** `packages/back-end/src/util/features.ts:618-635`

```typescript
const prerequisiteRules = hasPrerequisites
  ? (feature.prerequisites ?? [])?.map((p) => {
      const condition = getParsedCondition(groupMap, p.condition);
      if (!condition) return null;
      return {
        parentConditions: [
          {
            id: p.id,
            condition,
            gate: true,  // ← feature-level prerequisite 是阻塞型的
          },
        ],
      };
    }).filter(isDefined)
  : [];
```

**规则树结构：**
```
Feature Rules
├─ [0] Holdout Gate Rule (force rule, parentConditions 无 gate)
│    └─ parentConditions: [{ id: "$holdout:hld_xxx", condition: { value: "holdoutcontrol" } }]
│    └─ force: holdout_value (当用户在 holdout 控制组时返回)
├─ [1] Prerequisite Rules (force rule, parentConditions 有 gate: true)
│    └─ parentConditions: [{ id: "other_feature", condition: {...}, gate: true }]
├─ [2] Regular Rules (force/rollout/experiment)
└─ ...
```

### 1.4 Holdout 虚拟 Feature 定义

**文件：** `packages/back-end/src/services/features.ts:234-269`

每个 holdout 会生成一个虚拟的 feature 定义，用于分桶计算：

```typescript
export function generateHoldoutsPayload({
  holdoutsMap,
}: {
  holdoutsMap: Map<
    string,
    { holdout: HoldoutInterface; holdoutExperiment: ExperimentInterface }
  >;
}): Record<string, FeatureDefinition> {
  const holdoutDefs: Record<string, FeatureDefinition> = {};
  holdoutsMap.forEach((holdoutWithExperiment) => {
    const exp = holdoutWithExperiment.holdoutExperiment;
    const holdout = holdoutWithExperiment.holdout;
    if (!exp) return;

    const def: FeatureDefinition = {
      defaultValue: "genpop",
      rules: [
        {
          id: getHoldoutFeatureDefId(holdout.id),
          coverage: exp.phases[0].coverage,
          hashAttribute: exp.hashAttribute,
          seed: exp.phases[0].seed,
          hashVersion: 2,
          variations: ["holdoutcontrol", "holdouttreatment"],
          weights: [0.5, 0.5],       // 写死 50/50
          key: exp.trackingKey,
          phase: `${exp.phases.length - 1}`,
          meta: [{ key: "0" }, { key: "1" }],
          // 注意：没有 filters，没有 namespace
          // 即使 exp.phases[0].namespace 有配置，也不会被应用
        },
      ],
    };
    holdoutDefs[getHoldoutFeatureDefId(holdout.id)] = def;
  });
  return holdoutDefs;
}
```

**Holdout 分桶逻辑：**
- 虚拟 feature 默认值：`"genpop"`（普通人群，未被 holdout）
- 通过实验规则分桶：写死 `weights: [0.5, 0.5]`，50% 进入 `"holdoutcontrol"`，50% 进入 `"holdouttreatment"`
- `coverage` 来自 `exp.phases[0].coverage`，控制有多少用户参与分桶
- 只有 `"holdoutcontrol"` 组的用户会触发 holdout 规则，返回预设的 holdout 值

**与普通实验的关键区别：`generateHoldoutsPayload` 不调用 `applyNamespaceToPayload`。** 普通实验（feature-ref 规则、visual 实验、redirect 实验）都会在 payload 生成时调用 `applyNamespaceToPayload` 应用 namespace，但 holdout 虚拟 feature 直接构造规则对象，跳过了 namespace 应用。

## 二、分桶阶段排除机制

### 2.1 分桶流程概览

**文件：** `packages/sdk-js/src/core.ts`

`runExperiment` 函数中的分桶流程：

```
┌─────────────────────────────────────────────────────────┐
│                    runExperiment                        │
├─────────────────────────────────────────────────────────┤
│  1. 检查 variations 数量 (<2 则排除)                    │
│  2. 检查上下文是否禁用                                   │
│  3. 合并实验覆盖                                       │
│  4. URL 目标检查                                       │
│  5. 查询字符串强制变体                                  │
│  6. 上下文强制变体                                     │
│  7. 检查实验状态 (draft/active)                        │
│  8. 获取 hashAttribute (缺失则排除)                     │
├─────────────────────────────────────────────────────────┤
│  ┌───────────────────────────────────────────────────┐  │
│  │  STICKY BUCKET 检查 (如果启用)                     │  │
│  │  - 找到则跳过后续大部分排除检查                    │  │
│  └───────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────┤
│  9. FILTERS / NAMESPACE 排除检查  ←──── 分桶阶段排除    │
│     ├─ experiment.filters → isFilteredOut()            │
│     └─ experiment.namespace → inNamespace()            │
│  10. include 函数检查                                  │
│  11. Condition 条件检查                                │
│  12. Prerequisites 先决条件检查                        │
│  13. Group 组检查                                      │
│  14. URL 检查 (旧版)                                   │
├─────────────────────────────────────────────────────────┤
│  15. 计算 hash 值                                      │
│  16. 根据 ranges/weights 选择变体                      │
│  17. Sticky Bucket 版本阻塞检查                        │
│  18. coverage 覆盖范围检查                             │
│  19. 强制变体检查 (force)                              │
│  20. QA 模式检查                                      │
│  21. 实验停止状态检查                                 │
└─────────────────────────────────────────────────────────┘
```

### 2.2 Filters 排除机制

**文件：** `packages/sdk-js/src/core.ts:832-840`

```typescript
function isFilteredOut(filters: Filter[], ctx: EvalContext): boolean {
  return filters.some((filter) => {
    const { hashValue } = getHashAttribute(ctx, filter.attribute);
    if (!hashValue) return true;  // 缺失 hashValue → 该 filter 排除

    const n = hash(filter.seed, hashValue, filter.hashVersion || 2);
    if (n === null) return true;  // hash 失败 → 该 filter 排除

    return !filter.ranges.some((r) => inRange(n, r));
    // 任一范围匹配 → 该 filter 不排除（返回 false）
    // 所有范围都不匹配 → 该 filter 排除（返回 true）
  });
}
```

**Filter 数据结构：**

```typescript
export interface Filter {
  attribute?: string;       // 覆盖实验的 hashAttribute
  seed: string;             // hash 种子，用于命名空间隔离
  hashVersion: number;      // hash 算法版本
  ranges: VariationRange[]; // 允许的分桶范围列表
}
```

**`inRange` 精确语义（左闭右开区间）：**
**文件：** `packages/sdk-js/src/util.ts:54-56`

```typescript
export function inRange(n: number, range: VariationRange): boolean {
  return n >= range[0] && n < range[1];
}
```

> **关键：** 左端点包含，右端点不包含。`n ∈ [0, 1)` 时，范围 `[0.5, 1.0]` 实际是 `[0.5, 1.0)`。

---

#### 多 Filter 的精确排除语义

| 层级 | 逻辑 | 语义 | 代码表达式 |
|-----|------|------|-----------|
| **单个 Filter 内部** | OR（`ranges.some`） | 任一范围匹配 → 通过该 filter | `!filter.ranges.some(r => inRange(n, r))` |
| **多个 Filter 之间** | OR（`filters.some`） | 任一 filter 排除 → 整体排除 | `filters.some(filter => isFilteredOutByOne(filter))` |

**真值表示例（2 个 filters，每个 1 个范围）：**

| 用户在 Filter 1 范围内 | 用户在 Filter 2 范围内 | `isFilteredOut` 返回值 | 最终结果 |
|----------------------|----------------------|----------------------|---------|
| ✅ 是 | ✅ 是 | `false` | 入组 |
| ✅ 是 | ❌ 否 | `false` | 入组 |
| ❌ 否 | ✅ 是 | `false` | 入组 |
| ❌ 否 | ❌ 否 | `true` | 排除 |

> **结论：多 Filter 是 OR 关系，只需通过任一 Filter 即可入组。**
> 这意味着：**Filter 越多，入组概率越高**（OR 语义，不是 AND）。

---

#### 多范围的入组语义

单个 Filter 内的多范围也是 OR 关系：

```
Filter: { seed: "ns1", ranges: [[0, 0.2], [0.5, 0.7]] }

hash(n) = 0.15 → 在 [0, 0.2] 内 → ✅ 通过
hash(n) = 0.60 → 在 [0.5, 0.7] 内 → ✅ 通过
hash(n) = 0.30 → 不在任何范围内 → ❌ 排除
```

**多范围的典型用途：**
1. **增量放量**：`[[0, 0.1], [0.1, 0.2]]` → 先 10% 验证，再扩展到 20%，保留原有用户
2. **多段分配**：`[[0, 0.4], [0.6, 1.0]]` → 给其他实验留出中间 20%

---

### 2.3 旧版 Namespace 机制

**文件：** `packages/sdk-js/src/util.ts`

```typescript
export function inNamespace(
  hashValue: string,
  namespace: [string, number, number],
): boolean {
  const n = hash("__" + namespace[0], hashValue, 1);
  if (n === null) return false;
  return n >= namespace[1] && n < namespace[2];
}
```

### 2.4 分桶排除的时序

**重要：** Filters/Namespace 排除发生在 **实际分桶计算之前**。

---

### 2.5 Sticky Bucket 对分桶复用与检查顺序的影响

**文件：** `packages/sdk-js/src/core.ts:499-663`

Sticky Bucket（粘性分桶）对检查顺序有**重大影响**：如果用户已有 sticky bucket 分配，会**跳过大部分前置检查**，直接复用已有的分桶结果。

**`runExperiment` 中的关键代码：**

```typescript
let assigned = -1;
let foundStickyBucket = false;
let stickyBucketVersionIsBlocked = false;

if (ctx.user.saveStickyBucketAssignmentDoc && !experiment.disableStickyBucketing) {
  const { variation, versionIsBlocked } = getStickyBucketVariation({
    ctx,
    expKey: experiment.key,
    expBucketVersion: experiment.bucketVersion,
    expHashAttribute: experiment.hashAttribute,
    expFallbackAttribute: experiment.fallbackAttribute,
    expMinBucketVersion: experiment.minBucketVersion,
    expMeta: experiment.meta,
  });
  foundStickyBucket = variation >= 0;
  assigned = variation;
  stickyBucketVersionIsBlocked = !!versionIsBlocked;
}

// ⚠️ 关键：只有没找到 sticky bucket 时才执行这些检查
if (!foundStickyBucket) {
  // 检查 1: Filters / Namespace 排除
  if (experiment.filters && isFilteredOut(experiment.filters, ctx)) {
    return { result: getExperimentResult(...) };  // 排除
  }

  // 检查 2: include 函数
  if (experiment.include && !isIncluded(experiment.include)) {
    return { result: getExperimentResult(...) };  // 排除
  }

  // 检查 3: Condition 条件
  if (experiment.condition && !conditionPasses(experiment.condition, ctx)) {
    return { result: getExperimentResult(...) };  // 排除
  }

  // 检查 4: Parent Conditions / Prerequisites
  if (experiment.parentConditions) {
    // ... 递归评估 parentConditions ...
  }

  // 检查 5: Group 组检查
  if (experiment.groups && !inGroups(experiment.groups, ctx)) {
    return { result: getExperimentResult(...) };  // 排除
  }
}

// ⚠️ 这些检查即使有 sticky bucket 也会执行
// 检查 6: URL 目标检查（旧版）
if (experiment.url && !urlIsValid(experiment.url as RegExp, ctx)) {
  return { result: getExperimentResult(...) };  // 排除
}

// 检查 7: 计算 hash 值（无论是否 sticky 都要计算）
const n = hash(experiment.seed || key, hashValue, experiment.hashVersion || 1);

// ⚠️ 只有没找到 sticky bucket 时才重新分桶
if (!foundStickyBucket) {
  const ranges = experiment.ranges || getBucketRanges(...);
  assigned = chooseVariation(n, ranges);
}

// ⚠️ Sticky bucket 版本阻塞检查（即使有 sticky bucket 也执行）
if (stickyBucketVersionIsBlocked) {
  return { result: getExperimentResult(..., true) };  // 排除
}
```

---

#### Sticky Bucket 对 Holdout 分桶的影响

**Holdout 虚拟 feature 的实验规则也支持 sticky bucket**，因为它就是一个普通的实验规则。

**Sticky Bucket 跳过的检查（Holdout 实验）：**

| 检查项 | 有 Sticky Bucket 时 | 无 Sticky Bucket 时 |
|-------|-------------------|-------------------|
| Filters / Namespace | ❌ **跳过** | ✅ 执行 |
| `experiment.include` 函数 | ❌ **跳过** | ✅ 执行 |
| Condition 条件 | ❌ **跳过** | ✅ 执行 |
| Parent Conditions | ❌ **跳过** | ✅ 执行 |
| Group 组检查 | ❌ **跳过** | ✅ 执行 |
| URL 检查 | ✅ 仍执行 | ✅ 执行 |
| Sticky Bucket 版本阻塞 | ✅ 仍执行 | ✅ 执行 |

**Holdout 特有的影响：**

由于 Holdout 虚拟 feature 的实验规则**没有 filters/namespace**（代码证据见 4.3 节），sticky bucket 对 Holdout 的主要影响是：

1. **分桶复用**：用户一旦被分入 `holdoutcontrol` 或 `holdouttreatment`，后续会一直保持这个分配，不受后续 coverage 变化影响
2. **coverage 变化不影响已有用户**：如果 Holdout 的 coverage 从 0.5 调整到 0.2，已经在 sticky bucket 中的用户不会被踢出
3. **但版本阻塞会生效**：如果 `minBucketVersion` 提高，旧版本的 sticky bucket 会被阻塞，用户会重新分桶

---

#### evalFeature 中 force 规则与 sticky bucket 的关系

**文件：** `packages/sdk-js/src/core.ts:251-294`

对于 Holdout 规则（force rule + parentConditions），**sticky bucket 影响的是递归评估的 `$holdout:xxx` 虚拟 feature**，而不是当前 feature 的 force 规则本身：

```typescript
// evalFeature 中的规则评估顺序
rules: for (const rule of feature.rules) {
  // 1. 评估 parentConditions（递归调用 evalFeature）
  //    这里会触发 $holdout:xxx 虚拟 feature 的评估
  //    $holdout:xxx 的实验规则可能命中 sticky bucket
  if (rule.parentConditions) {
    for (const parentCondition of rule.parentConditions) {
      const parentResult = evalFeature(parentCondition.id, ctx);
      // ... 检查条件 ...
      if (!evaled) {
        if (parentCondition.gate) return getFeatureResult(...);
        continue rules;  // Holdout 规则走这里
      }
    }
  }

  // 2. Filters 检查（evalFeature 层面的 rule.filters）
  if (rule.filters && isFilteredOut(rule.filters, ctx)) continue;

  // 3. Force 规则的条件检查（rule.condition）
  if ("force" in rule) {
    if (rule.condition && !conditionPasses(rule.condition, ctx)) continue;

    // 4. Force 规则的 rollout 检查（isIncludedInRollout）
    //    注意：这里也可能命中 sticky bucket（如果 force 规则有 hashAttribute）
    if (!isIncludedInRollout(ctx, ...)) continue;

    // 5. 返回 force 值
    return getFeatureResult(ctx, id, rule.force, "force", rule.id);
  }
}
```

**Holdout 的完整检查顺序（含 sticky bucket）：**

```
evalFeature("my_feature")
    │
    └─ 规则 0: Holdout Gate Rule
        ├─ parentConditions: [{ id: "$holdout:hld_123", condition: { value: "holdoutcontrol" } }]
        │   └─ evalFeature("$holdout:hld_123")
        │       └─ 评估 $holdout:hld_123 的实验规则
        │           ├─ 🔍 查找 sticky bucket
        │           │   ├─ ✅ 找到 → 跳过 filters/include/condition/group/prerequisites
        │           │   │   └─ 直接复用已有的变体
        │           │   └─ ❌ 未找到 → 执行所有检查 → 计算 hash → 分桶
        │           └─ 返回变体值（"holdoutcontrol" / "holdouttreatment" / "genpop"）
        │
        ├─ 检查条件: evalCondition({ value: result }, { value: "holdoutcontrol" })
        │   ├─ ✅ 匹配 → 执行 force → 返回 holdout 值
        │   └─ ❌ 不匹配 → continue rules → 继续下一条规则
        │
        ├─ rule.filters 检查（Holdout 规则通常没有）
        ├─ rule.condition 检查（Holdout 规则通常没有）
        └─ isIncludedInRollout 检查（Holdout 规则通常没有）
```

**关键点：**
- **Sticky bucket 作用于 `$holdout:xxx` 虚拟 feature**，不是作用于业务 feature
- 一旦用户在 Holdout 中有了 sticky bucket，其 `holdoutcontrol` / `holdouttreatment` 状态会被**永久锁定**（除非版本阻塞）
- Sticky bucket 跳过的是 Holdout 实验本身的前置检查，**不跳过** Holdout 规则的 parentCondition 评估

## 三、跨实验互斥（Mutual Exclusion）

### 3.1 互斥实现原理

GrowthBook 通过 **Namespace / Filters 机制** 实现跨实验互斥。

**核心思想：**
- 多个实验使用 **相同的 namespace seed** 和 **相同的 hashAttribute**
- 每个实验分配 **不重叠的分桶范围**
- 使用 `isFilteredOut()` 在分桶前排除不在范围内的用户
- 确保用户最多只能进入其中一个实验

### 3.2 Namespace 格式转换

**文件：** `packages/back-end/src/util/features.ts:425-476`

```typescript
export function applyNamespaceToPayload(
  rule: FeatureDefinitionRule,
  namespace: NamespaceValue,
  namespacesMap?: Map<
    string, { hashAttribute?: string; seed?: string; format?: "legacy" | "multiRange" }
  >,
): void {
  const nsDefinition = namespacesMap?.get(namespace.name);

  const multiRange = nsDefinition
    ? nsDefinition.format === "multiRange"
    : isMultiRangeNamespaceFormat(namespace);

  const ranges = getNamespaceRanges(namespace).map(
    ([start, end]) => [Number(start) || 0, Number(end) || 0] as [number, number],
  );

  if (multiRange) {
    const filterAttribute = getNamespaceHashAttribute(
      namespace,
      nsDefinition?.hashAttribute || rule.hashAttribute || "id",
    );
    const filterHashVersion = ("hashVersion" in namespace && namespace.hashVersion) || 2;
    const seed = nsDefinition?.seed || namespace.name;

    rule.filters = [
      ...(rule.filters || []),
      {
        attribute: filterAttribute,
        seed,
        hashVersion: filterHashVersion,
        ranges,
      },
    ];
    return;
  }

  const [start, end] = ranges[0] ?? [0, 0];
  rule.namespace = [namespace.name, start, end];
}
```

**`applyNamespaceToPayload` 的调用点：**

| 调用位置 | 文件 | 行号 |
|---------|------|------|
| Feature experiment-ref 规则 | `back-end/src/util/features.ts` | 713-718 |
| Feature rollout 规则 | `back-end/src/util/features.ts` | 856-857 |
| Visual/Redirect 实验 | `back-end/src/services/features.ts` | 448-455 |

**注意：`generateHoldoutsPayload` 不调用此函数。**

### 3.3 互斥示例

假设有两个实验需要互斥：
- 实验 A：`exp_button_color`
- 实验 B：`exp_checkout_flow`

**配置：**
```
Namespace: "checkout_flow"
- 实验 A 分配范围: [0, 0.5]
- 实验 B 分配范围: [0.5, 1.0]
```

**SDK payload 中的 filters：**
```javascript
// 实验 A 的规则
{
  key: "exp_button_color",
  filters: [{
    attribute: "user_id",
    seed: "checkout_flow",
    hashVersion: 2,
    ranges: [[0, 0.5]]
  }],
  variations: ["red", "blue"],
}

// 实验 B 的规则
{
  key: "exp_checkout_flow",
  filters: [{
    attribute: "user_id",
    seed: "checkout_flow",  // 相同的 seed
    hashVersion: 2,
    ranges: [[0.5, 1.0]]    // 不重叠的范围
  }],
  variations: ["v1", "v2"],
}
```

### 3.4 多范围 Namespace

**文件：** `packages/shared/src/util/namespaces.ts`

```typescript
type MultiRangeNamespaceValue = {
  enabled: boolean;
  name: string;
  ranges: [number, number][];
  hashAttribute?: string;
  hashVersion?: number;
  format: "multiRange";
};

export function getNamespaceRanges(namespace: NamespaceValue): [number, number][] {
  if (isLegacyNamespaceFormat(namespace)) {
    return namespace.range ? [namespace.range] : [];
  }
  return namespace.ranges ?? [];
}
```

## 四、Holdout 与互斥的协同工作

### 4.1 完整的规则评估流程

**文件：** `packages/sdk-js/src/core.ts`

```typescript
export function evalFeature<V = unknown>(id: string, ctx: EvalContext): FeatureResult<V | null> {
  // 1. 循环依赖检测
  if (ctx.stack.evaluatedFeatures.has(id)) {
    return getFeatureResult(ctx, id, null, "cyclicPrerequisite");
  }
  ctx.stack.evaluatedFeatures.add(id);

  // 2. 全局强制值覆盖
  const forcedValues = getForcedFeatureValues(ctx);
  if (forcedValues.has(id)) {
    return getFeatureResult(ctx, id, forcedValues.get(id), "override");
  }

  // 3. 获取 feature 定义
  const feature: FeatureDefinition<V> = ctx.global.features[id];

  // 4. 遍历规则（按顺序）
  if (feature.rules) {
    rules: for (const rule of feature.rules) {
      // 4.1 先决条件评估（递归调用 evalFeature）
      if (rule.parentConditions) {
        for (const parentCondition of rule.parentConditions) {
          const parentResult = evalFeature(parentCondition.id, ctx);
          if (parentResult.source === "cyclicPrerequisite") {
            return getFeatureResult(ctx, id, null, "cyclicPrerequisite");
          }
          const evalObj = { value: parentResult.value };
          const evaled = evalCondition(evalObj, parentCondition.condition || {});

          if (!evaled) {
            if (parentCondition.gate) {
              // 阻塞型（feature-level prerequisite）：整个 feature 返回空
              return getFeatureResult(ctx, id, null, "prerequisite");
            }
            // 非阻塞型（holdout 规则）：跳过当前规则，继续下一个
            continue rules;
          }
        }
      }

      // 4.2 Filter/Namespace 排除检查（分桶阶段）
      if (rule.filters && isFilteredOut(rule.filters, ctx)) {
        continue;
      }

      // 4.3 Force 规则处理
      if ("force" in rule) {
        if (rule.condition && !conditionPasses(rule.condition, ctx)) continue;
        if (!isIncludedInRollout(...)) continue;
        return getFeatureResult(ctx, id, rule.force as V, "force", rule.id);
      }

      // 4.4 Experiment 规则处理
      if (rule.variations) {
        const exp: Experiment<V> = { /* 从 rule 构建 experiment */ };
        const { result } = runExperiment(exp, id, ctx);
        if (result.inExperiment && !result.passthrough) {
          return getFeatureResult(ctx, id, result.value, "experiment", rule.id, exp, result);
        }
      }
    }
  }

  // 5. 返回默认值
  return getFeatureResult(ctx, id, feature.defaultValue ?? null, "defaultValue");
}
```

### 4.2 Holdout 作为先决条件的执行路径

当 feature 关联了 holdout 时，SDK 侧的完整执行路径：

```
evalFeature("my_feature")
│
├─ 规则 0: Holdout Gate Rule (parentConditions, 无 gate)
│   └─ parentConditions: [{ id: "$holdout:hld_123", condition: { value: "holdoutcontrol" } }]
│       └─ evalFeature("$holdout:hld_123")  ← 递归评估虚拟 holdout feature
│           ├─ 默认值: "genpop"
│           └─ 运行 holdout 实验规则
│               ├─ hash(seed, user_id, 2) → n
│               ├─ coverage 检查: n < coverage ?
│               ├─ 在 coverage 内 → weights [0.5, 0.5] 选择变体
│               │   ├─ n ∈ [0, 0.5) → "holdoutcontrol"
│               │   └─ n ∈ [0.5, 1.0) → "holdouttreatment"
│               └─ 在 coverage 外 → 返回默认值 "genpop"
│
├─ 检查 parentCondition: evalCondition({ value: result }, { value: "holdoutcontrol" })
│   ├─ result = "holdoutcontrol": 条件匹配
│   │   └─ 执行 force 规则 → 返回 holdout value（如 "off"）
│   │      → 整个 feature 评估结束，用户不会进入后续任何规则/实验
│   │
│   ├─ result = "holdouttreatment": 条件不匹配，无 gate
│   │   └─ continue rules → 继续执行下一条规则
│   │
│   └─ result = "genpop": 条件不匹配，无 gate
│       └─ continue rules → 继续执行下一条规则
│
└─ 规则 1: 普通业务规则（force/rollout/experiment）
    └─ 正常执行（可能包含 namespace/filters）
```

**关键机制：Holdout 的"软互斥"**
- Holdout 规则是**非阻塞型**的（没有 `gate: true`）
- 当用户被分入 `holdoutcontrol` 组时，`force` 规则生效，用户**不会进入后续业务实验**
- 当用户被分入 `holdouttreatment` 组或不在 coverage 内时，`continue rules` 继续执行后续规则
- 这是在**规则评估层面**的互斥，不是分桶层面的互斥

### 4.3 Holdout 与跨实验互斥的关系

**核心事实：Holdout 不参与 Namespace 互斥，因为其虚拟 feature 没有 namespace/filter。**

**代码证据对比：**

| 组件 | 处理函数 | 是否调用 applyNamespaceToPayload | 是否有 filters/namespace |
|-----|---------|--------------------------------|------------------------|
| Holdout 虚拟 feature | `generateHoldoutsPayload()` | ❌ 不调用 | ❌ 没有 |
| Feature experiment-ref 规则 | `getFeatureDefinition()` → 713-718 行 | ✅ 调用 | ✅ 有 |
| Feature rollout 规则 | `getFeatureDefinition()` → 856-857 行 | ✅ 调用 | ✅ 有 |
| Visual 实验 | `generateAutoExperimentsPayload()` → 448-455 行 | ✅ 调用 | ✅ 有 |
| Redirect 实验 | `generateAutoExperimentsPayload()` → 448-455 行 | ✅ 调用 | ✅ 有 |

**这意味着：**
1. Holdout 分桶是**全局的**，不受 namespace 约束
2. 即使 holdout 实验在界面上配置了 namespace，该 namespace 也**不会**应用到 holdout 的虚拟 feature 上
3. Holdout 分桶和业务实验的 namespace 分桶是**两次独立的 hash 计算**，使用不同的 seed
4. Holdout 无法通过 namespace 与其他实验实现"硬互斥"（分桶层面的互斥）

**Holdout 与业务实验互斥的两种方式：**

**方式 1（推荐）：Prerequisite + Force 软互斥**
- Holdout 规则作为首条规则插入，用户在 `holdoutcontrol` 组时 `force` 返回 holdout 值
- 用户不会进入后续的 namespace 约束实验
- 优点：自动生效，无需手动协调 namespace 范围
- 注意：这只是在当前 feature 内的互斥，不影响其他 feature

**方式 2：手动协调 namespace 范围**
- Holdout 的 `coverage` 决定了全局有多少用户参与分桶
- 业务实验的 namespace 范围需要手动避开 holdout coverage 对应的用户
- 缺点：holdout 和 namespace 使用不同的 hash seed，分桶结果无对应关系，无法精确避开

## 五、生效门禁与能力开关

### 5.1 Prerequisites Capability 能力开关

**文件：** `packages/back-end/src/util/features.ts:570-590`

Holdout 功能**完全依赖** `prerequisites` capability：

```typescript
const hasPrerequisites =
  capabilities === undefined || capabilities.includes("prerequisites");

// 1. Holdout 规则仅在有 prerequisites 能力时生成
const holdoutRule: FeatureDefinitionRule[] =
  hasPrerequisites &&
  feature.holdout &&
  holdoutsMap &&
  holdoutsMap.get(feature.holdout.id)?.holdout.environmentSettings?.[
    environment
  ]?.enabled
    ? [/* holdout 规则 */]
    : [];

// 2. 没有 prerequisites 能力时，包含 gates 的 feature 整个被排除
if (capabilities !== undefined && !hasPrerequisites) {
  const hasTopLevelPrereqs = !!feature.prerequisites?.length;
  const hasRuleLevelGates = rules?.some((r) => {
    if (r.type === "experiment-ref") {
      const exp = experimentMap.get(r.experimentId);
      const phase = exp?.phases?.slice(-1)?.[0];
      return !!phase?.prerequisites?.length;
    }
    return !!(r as { prerequisites?: unknown[] }).prerequisites?.length;
  });
  if (hasTopLevelPrereqs || hasRuleLevelGates) {
    return null;
  }
}
```

**能力开关影响总结：**

| 场景 | 有 prerequisites 能力 | 无 prerequisites 能力 |
|-----|---------------------|---------------------|
| Holdout 规则 | 生成并插入到规则最前面 | Holdout 规则完全不生成 |
| Feature-level prerequisites | 转换为 parentConditions 规则（gate: true） | 不生成，若有则整个 feature 被排除 |
| Rule-level prerequisites | 转换为 parentConditions | 不生成，若有则整个 feature 被排除 |

### 5.2 项目门禁（Project Gating）

**文件：** `packages/back-end/src/services/features.ts:209-232`

```typescript
function buildHoldoutsMapForProjects(
  holdoutsMap: Map<string, { holdout: HoldoutInterface; holdoutExperiment: ExperimentInterface }>,
  projects: string[],
): Map<...> {
  const result = new Map();
  holdoutsMap.forEach((value, id) => {
    const { holdout } = value;
    const allowed =
      projects.length === 0 ||
      holdout.projects.length === 0 ||
      holdout.projects.some((p) => projects.includes(p));
    if (allowed) result.set(id, value);
  });
  return result;
}
```

### 5.3 Holdout 进入 Payload 的完整过滤条件

**文件：** `packages/back-end/src/models/HoldoutModel.ts:169-217`

`getAllPayloadHoldouts` 是 Holdout 进入 payload 的第一道也是最关键的一道门，包含 **5 个过滤条件**，**全部满足**才会进入后续流程：

```typescript
public async getAllPayloadHoldouts(environment?: string): Promise<Map<...>> {
  const holdouts = await this._find({});
  const holdoutsWithExperiments = await Promise.all(
    holdouts.map(async (h) => {
      const holdoutExperiment = await getExperimentById(
        this.context,
        h.experimentId,
      );
      return { holdout: h, holdoutExperiment };
    }),
  );

  const filteredHoldouts = holdoutsWithExperiments.filter(
    (h): h is { holdout: HoldoutInterface; holdoutExperiment: ExperimentInterface } => {
      // 条件 1：关联的实验必须存在
      if (!h.holdoutExperiment) return false;

      // 条件 2：关联的实验未被归档
      if (h.holdoutExperiment.archived) return false;

      // 条件 3：关联的实验状态必须是 running
      if (h.holdoutExperiment.status !== "running") return false;

      // 条件 4：必须有至少一个关联的实验或 feature
      // （既没有 linkedExperiments 也没有 linkedFeatures 的 holdout 不会进入 payload）
      if (
        Object.keys(h.holdout.linkedExperiments).length === 0 &&
        Object.keys(h.holdout.linkedFeatures).length === 0
      )
        return false;

      // 条件 5：在指定环境中必须启用
      if (
        environment &&
        !h.holdout.environmentSettings[environment]?.enabled
      ) {
        return false;
      }

      return true;
    },
  );
  return new Map(filteredHoldouts.map((h) => [h.holdout.id, h]));
}
```

**Holdout 进入 payload 的完整条件链（AND 关系）：**

```
getAllPayloadHoldouts()
    │
    ├─ 1. holdoutExperiment 存在 ✅
    ├─ 2. holdoutExperiment.archived = false ✅
    ├─ 3. holdoutExperiment.status = "running" ✅
    ├─ 4. linkedExperiments.length > 0 OR linkedFeatures.length > 0 ✅
    ├─ 5. environmentSettings[env].enabled = true ✅
    │
    ├─ buildHoldoutsMapForProjects()  → 项目过滤
    │   └─ holdout.projects 与 connection.projects 匹配 ✅
    │
    ├─ generateHoldoutsPayload()      → 生成虚拟 $holdout:xxx feature
    │
    └─ pruneUnreferencedHoldouts()    → 裁剪未被引用的
        └─ 被至少一个 feature 规则的第一条 parentCondition 引用 ✅
```

**环境门禁的双重检查：**
1. `getAllPayloadHoldouts(environment)` — 虚拟 feature 生成前过滤
2. `getFeatureDefinition()` 中 `holdout.environmentSettings[environment]?.enabled` — 规则生成时再次检查

### 5.4 Holdout 定义被裁剪的完整生效链路

**文件：** `packages/back-end/src/services/features.ts:1035-1196`

Holdout 虚拟 feature 的生效经过**四级过滤**：

```
SDK Connection 配置
    │
    ├─ 第一级：环境过滤
    │   └─ getAllPayloadHoldouts(environment)
    │       └─ 只返回 environmentSettings[env].enabled = true 的 holdout
    │
    ├─ 第二级：项目过滤
    │   └─ buildHoldoutsMapForProjects(holdoutsMap, projectList)
    │       └─ 只返回 holdout.projects 与 connection.projects 匹配的 holdout
    │
    ├─ 第三级：生成所有 holdout 虚拟 feature
    │   └─ generateHoldoutsPayload(holdoutsMapForConnection)
    │       └─ 为每个 holdout 生成 $holdout:xxx 虚拟 feature
    │       └─ 注意：这里生成的是全部通过前两级过滤的 holdout
    │
    └─ 第四级：裁剪未被引用的 holdout
        └─ pruneUnreferencedHoldouts(holdoutFeatureDefinitions, featureDefinitions)
            └─ 扫描所有 feature 的所有规则
            └─ 检查 rule.parentConditions?.[0]?.id 是否以 "$holdout:" 开头
            └─ 只保留被引用的 holdout 虚拟 feature
```

**第四级裁剪的关键代码：**
**文件：** `packages/back-end/src/services/features.ts:1035-1050`

```typescript
function pruneUnreferencedHoldouts(
  holdouts: Record<string, FeatureDefinition>,
  features: Record<string, FeatureDefinition>,
): Record<string, FeatureDefinition> {
  const referenced = new Set<string>();
  for (const k in features) {
    for (const rule of features[k]?.rules ?? []) {
      const pcId = rule.parentConditions?.[0]?.id;
      if (pcId?.startsWith("$holdout:")) referenced.add(pcId);
    }
  }
  return Object.fromEntries(
    Object.entries(holdouts).filter(([key]) => referenced.has(key)),
  );
}
```

**裁剪逻辑的关键细节：**
1. **只检查第一条 parentCondition**：`rule.parentConditions?.[0]?.id`，如果 holdout 不在第一条则不会被检测到
2. **只被 feature 规则引用才保留**：auto experiments（visual/redirect）引用的 holdout 不会被保留
3. **最终合并**：`featuresWithHoldouts = { ...featureDefinitions, ...holdoutsInUse }`

### 5.5 多级门禁联动效果

```
用户请求 SDK Payload
    │
    ├─ Connection 配置
    │   ├─ environment: "production"
    │   ├─ projects: ["p1"]
    │   └─ capabilities: ["prerequisites", "bucketingV2"]
    │
    ├─ Holdout 筛选
    │   ├─ 步骤 1: getAllPayloadHoldouts("production") → 环境过滤
    │   ├─ 步骤 2: buildHoldoutsMapForProjects(map, ["p1"]) → 项目过滤
    │   ├─ 步骤 3: generateHoldoutsPayload() → 生成虚拟 feature
    │   └─ 步骤 4: pruneUnreferencedHoldouts() → 裁剪未被引用的
    │
    └─ Feature 筛选
        ├─ 步骤 1: getAllFeatures({ projects: ["p1"] }) → 项目过滤
        ├─ 步骤 2: getFeatureDefinition()
        │   ├─ 检查 prerequisites capability
        │   ├─ 检查 holdout environmentSettings.enabled
        │   ├─ 生成 holdout 规则（最前面，无 gate）
        │   ├─ 生成 prerequisite 规则（有 gate: true）
        │   └─ 生成业务规则（可能包含 namespace）
        └─ 步骤 3: 按环境过滤规则
```

## 六、跨实验互斥的真实保证机制

### 6.1 核心结论：互斥是约定性的，没有运行时强制保证

**代码证据：** GrowthBook 的 namespace 互斥机制**完全依赖手动配置**，代码中**没有**任何运行时验证逻辑来确保同一 namespace 下的实验范围不重叠。

**`validateNonOverlappingRanges` 存在但未被使用：**
**文件：** `packages/shared/src/util/namespaces.ts:102-127`

```typescript
export function rangesOverlap(
  range1: [number, number],
  range2: [number, number],
): boolean {
  return range1[0] < range2[1] && range2[0] < range1[1];
}

export function validateNonOverlappingRanges(ranges: [number, number][]): {
  valid: boolean;
  error?: string;
} {
  for (let i = 0; i < ranges.length; i++) {
    for (let j = i + 1; j < ranges.length; j++) {
      if (rangesOverlap(ranges[i], ranges[j])) {
        return { valid: false, error: `Ranges [...] overlap` };
      }
    }
  }
  return { valid: true };
}
```

**全局搜索确认**：`rangesOverlap` 和 `validateNonOverlappingRanges` 只在 `packages/shared/src/util/namespaces.ts` 中定义，未被任何其他文件导入或调用。这些函数是工具函数，**没有用于 SDK payload 生成时的跨实验范围验证**。

### 6.2 `applyNamespaceToPayload` 不做范围冲突检查

**文件：** `packages/back-end/src/util/features.ts:425-476`

```typescript
export function applyNamespaceToPayload(
  rule: FeatureDefinitionRule,
  namespace: NamespaceValue,
  namespacesMap?: Map<string, { hashAttribute?: string; seed?: string; format?: ... }>,
): void {
  // ... 获取 nsDefinition, ranges, seed, filterAttribute ...

  if (multiRange) {
    rule.filters = [
      ...(rule.filters || []),
      {
        attribute: filterAttribute,
        seed,
        hashVersion: filterHashVersion,
        ranges,  // 直接使用配置的范围，不检查是否与其他实验重叠
      },
    ];
    return;
  }

  const [start, end] = ranges[0] ?? [0, 0];
  rule.namespace = [namespace.name, start, end];
}
```

### 6.3 互斥失效的场景

| 场景 | 结果 |
|-----|------|
| 同一 namespace 下两个实验配置了重叠范围 | 用户可能同时进入两个实验 |
| 同一 namespace 下两个实验使用不同的 hashAttribute | 分桶基数不同，完全无法保证互斥 |
| 同一 namespace 下两个实验使用不同的 hashVersion | hash 结果不同，完全无法保证互斥 |
| Holdout 与 namespace 实验共存 | Holdout 分桶不受 namespace 约束，可能同时出现在 holdout 和 namespace 实验中 |

### 6.4 正确使用互斥的前提条件

要实现真正的跨实验互斥，**必须同时满足**：
1. 使用相同的 namespace name（确保 `seed` 一致）
2. 使用相同的 `hashAttribute`
3. 使用相同的 `hashVersion`
4. 各实验的 ranges 不重叠
5. 通过组织级 namespace 定义统一 `seed` 和 `hashAttribute`

## 七、关键代码位置总结

| 功能模块 | 文件位置 | 关键函数/类型 |
|---------|---------|-------------|
| Holdout 验证器 | `packages/shared/src/validators/holdout.ts` | `holdoutValidator` |
| Holdout 规则生成 | `packages/back-end/src/util/features.ts:592-616` | `getFeatureDefinition()` 中的 holdoutRule |
| Holdout 虚拟 Feature | `packages/back-end/src/services/features.ts:234-269` | `generateHoldoutsPayload()` |
| Holdout 定义裁剪 | `packages/back-end/src/services/features.ts:1035-1050` | `pruneUnreferencedHoldouts()` |
| Prerequisites Capability | `packages/back-end/src/util/features.ts:570-590` | `hasPrerequisites` 变量 |
| 项目过滤 | `packages/back-end/src/services/features.ts:209-232` | `buildHoldoutsMapForProjects()` |
| 环境过滤 | `packages/back-end/src/models/HoldoutModel.ts` | `getAllPayloadHoldouts()` |
| Feature 定义生成 | `packages/back-end/src/util/features.ts:478-978` | `getFeatureDefinition()` |
| 分桶排除检查 | `packages/sdk-js/src/core.ts` | `isFilteredOut()` |
| 实验运行流程 | `packages/sdk-js/src/core.ts` | `runExperiment()` |
| Feature 评估 | `packages/sdk-js/src/core.ts` | `evalFeature()` |
| Namespace 工具 | `packages/shared/src/util/namespaces.ts` | `getNamespaceRanges()`, `rangesOverlap()` |
| Namespace 应用 | `packages/back-end/src/util/features.ts:425-476` | `applyNamespaceToPayload()` |
| Filter 类型 | `packages/sdk-js/src/types/growthbook.ts` | `Filter` 接口 |
| Auto Experiments Payload | `packages/back-end/src/services/features.ts:282-498` | `generateAutoExperimentsPayload()` |

## 八、核心结论

### 8.1 Holdout 生效机制

1. **完全依赖 prerequisites capability**：
   - 没有该能力时，Holdout 规则完全不生成
   - 包含 prerequisites/gates 的 feature 会被整个排除

2. **Holdout 规则是非阻塞型先决条件**：
   - 没有 `gate: true`（与 feature-level prerequisites 不同）
   - 条件不匹配时 `continue rules`，不会阻塞整个 feature

3. **四级过滤生效链路**：
   - 环境过滤 → 项目过滤 → 生成虚拟 feature → 裁剪未被引用的

4. **Holdout 分桶是全局的**：
   - `generateHoldoutsPayload` 不调用 `applyNamespaceToPayload`
   - 虚拟 feature 没有 namespace/filter
   - `weights: [0.5, 0.5]` 写死 50/50
   - 即使 holdout 实验配置了 namespace，也不会应用到虚拟 feature

### 8.2 跨实验互斥机制

1. **互斥是约定性的，没有运行时强制保证** ⚠️：
   - `rangesOverlap()` / `validateNonOverlappingRanges()` 存在但未被任何代码调用
   - `applyNamespaceToPayload` 直接使用配置的范围，不检查冲突
   - 完全依赖手动配置 namespace 的范围不重叠

2. **互斥的正确实现**：
   - 普通实验（feature-ref、rollout、visual、redirect）都通过 `applyNamespaceToPayload` 应用 namespace
   - 组织级 namespace 提供统一的 `hashAttribute` 和 `seed`
   - 必须手动保证同一 namespace 下的实验 ranges 不重叠

### 8.3 Holdout 与互斥的关系

1. **Holdout 不参与 namespace 互斥**：
   - Holdout 虚拟 feature 没有 namespace/filter，分桶是全局的
   - Holdout 分桶和 namespace 分桶是两次独立的 hash 计算，使用不同的 seed
   - 无法通过 namespace 实现 Holdout 与其他实验的"硬互斥"（分桶层面的互斥）

2. **Holdout 的"软互斥"机制**：
   - 通过 `parentConditions`（无 gate）+ `force` 规则实现
   - 用户在 `holdoutcontrol` 组时，force 规则生效，跳过后续所有规则
   - 这是在**规则评估层面**的互斥，只对当前 feature 有效
   - 不影响其他 feature 的评估
