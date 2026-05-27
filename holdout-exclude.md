# GrowthBook Holdout 分组与全局排除规则实现分析

## 一、Holdout 分组机制

### 1.1 Holdout 数据结构

**文件：** `packages/shared/src/validators/holdout.ts:24-46`

```typescript
export const holdoutValidator = z.object({
  id: z.string(),
  organization: z.string(),
  dateCreated: z.date(),
  dateUpdated: z.date(),
  projects: z.array(z.string()),
  name: z.string(),
  skipAsDefaultHoldout: z.boolean().optional(),
  experimentId: z.string(),                    // 关联的实验 ID
  linkedExperiments: z.record(z.string(), holdoutLinkedItemValidator),
  linkedFeatures: z.record(z.string(), holdoutLinkedItemValidator),
  environmentSettings: z.record(z.string(), featureEnvironment),
  analysisStartDate: z.date().optional(),
  statusUpdateSchedule: statusUpdateScheduleValidator.optional().nullable(),
  nextScheduledStatusUpdate: nextScheduledStatusUpdateValidator.optional().nullable(),
}).strict();
```

### 1.2 Feature 中的 Holdout 关联

**文件：** `packages/shared/types/feature.d.ts`

Feature 文档中通过 `holdout` 字段关联 holdout：

```typescript
interface FeatureInterface {
  // ... 其他字段
  holdout?: {
    id: string;      // holdout 的 ID
    value: string;   // 当用户在 holdout 组时返回的值
  };
}
```

### 1.3 Holdout 在规则树中的表达形式

**文件：** `packages/back-end/src/util/features.ts:592-616`

Holdout 规则在 SDK payload 中被实现为 **前置先决条件规则（prerequisite rule）**，插入在 feature 规则列表的 **最前面**：

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
              condition: { value: "holdoutcontrol" },        // 检查值是否为控制组
            },
          ],
          force: getJSONValue(feature.valueType, feature.holdout.value),
        },
      ]
    : [];
```

**规则树结构：**
```
Feature Rules
├─ [0] Holdout Gate Rule (force rule with parentConditions)
│    └─ parentConditions: [{ id: "$holdout:hld_xxx", condition: { value: "holdoutcontrol" } }]
│    └─ force: holdout_value (当用户在 holdout 组时返回)
├─ [1] Prerequisite Rules (feature-level prerequisites)
├─ [2] Regular Rules (force/rollout/experiment)
└─ ...
```

### 1.4 Holdout 虚拟 Feature 定义

**文件：** `packages/back-end/src/services/features.ts:234-269`

每个 holdout 会生成一个虚拟的 feature 定义，用于分桶计算：

```typescript
function generateHoldoutsPayload({ holdoutsMap }) {
  const holdoutDefs: Record<string, FeatureDefinition> = {};
  holdoutsMap.forEach((holdoutWithExperiment) => {
    const exp = holdoutWithExperiment.holdoutExperiment;
    const holdout = holdoutWithExperiment.holdout;
    
    const def: FeatureDefinition = {
      defaultValue: "genpop",  // 默认：普通人群
      rules: [
        {
          id: getHoldoutFeatureDefId(holdout.id),
          coverage: exp.phases[0].coverage,
          hashAttribute: exp.hashAttribute,
          seed: exp.phases[0].seed,
          hashVersion: 2,
          variations: ["holdoutcontrol", "holdouttreatment"],
          weights: [0.5, 0.5],  // 50% 控制组，50% 处理组
          key: exp.trackingKey,
          phase: `${exp.phases.length - 1}`,
          meta: [{ key: "0" }, { key: "1" }],
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
- 通过实验规则分桶：50% 概率进入 `"holdoutcontrol"`，50% 进入 `"holdouttreatment"`
- 只有 `"holdoutcontrol"` 组的用户会触发 holdout 规则，返回预设的 holdout 值

## 二、分桶阶段排除机制

### 2.1 分桶流程概览

**文件：** `packages/sdk-js/src/core.ts:385-785`

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
    // 1. 获取 filter 专属的 hash attribute 值
    const { hashValue } = getHashAttribute(ctx, filter.attribute);
    if (!hashValue) return true;  // 缺失则排除
    
    // 2. 使用 filter 专属的 seed 和 hashVersion 计算 hash
    const n = hash(filter.seed, hashValue, filter.hashVersion || 2);
    if (n === null) return true;    // 版本无效则排除
    
    // 3. 检查是否在允许的范围内 (任意范围满足则不排除)
    return !filter.ranges.some((r) => inRange(n, r));
  });
}
```

**Filter 数据结构：**
**文件：** `packages/sdk-js/src/types/growthbook.ts:547-556`

```typescript
export interface Filter {
  attribute?: string;       // 覆盖实验的 hashAttribute
  seed: string;             // hash 种子，用于命名空间隔离
  hashVersion: number;      // hash 算法版本
  ranges: VariationRange[]; // 允许的分桶范围列表
}
```

**关键特性：**
1. **独立分桶**：Filter 使用独立的 `seed` 和 `attribute`，与实验本身的分桶计算分离
2. **多范围支持**：支持多个不连续的范围 `[[0, 0.2], [0.5, 0.7]]`
3. **多 Filter 逻辑**：多个 Filter 为 **AND** 关系，全部通过才不被排除

### 2.3 旧版 Namespace 机制

**文件：** `packages/sdk-js/src/util.ts:58-65`

```typescript
export function inNamespace(
  hashValue: string,
  namespace: [string, number, number],  // [name, start, end]
): boolean {
  // 使用 "__" + namespace name 作为 seed，hashVersion=1
  const n = hash("__" + namespace[0], hashValue, 1);
  if (n === null) return false;
  return n >= namespace[1] && n < namespace[2];
}
```

### 2.4 分桶排除的时序

**重要：** Filters/Namespace 排除发生在 **实际分桶计算之前**

**文件：** `packages/sdk-js/src/core.ts:519-606`

```typescript
// Some checks are not needed if we already have a sticky bucket
if (!foundStickyBucket) {
  // 7. Exclude if user is filtered out (used to be called "namespace")
  if (experiment.filters) {
    if (isFilteredOut(experiment.filters, ctx)) {
      return { result: getExperimentResult(..., -1, false, ...) };
    }
  } else if (experiment.namespace && !inNamespace(hashValue, experiment.namespace)) {
    return { result: getExperimentResult(..., -1, false, ...) };
  }
  
  // ... 其他排除检查
  
  // 8.1. Exclude if user is not in a required group
  if (experiment.groups && !hasGroupOverlap(...)) {
    return { result: getExperimentResult(..., -1, false, ...) };
  }
}

// 8.2. Old style URL targeting (sticky bucket 用户也会检查)
if (experiment.url && !urlIsValid(...)) {
  return { result: getExperimentResult(..., -1, false, ...) };
}

// 9. 实际分桶计算（在所有排除检查之后）
const n = hash(experiment.seed || key, hashValue, experiment.hashVersion || 1);
if (!foundStickyBucket) {
  const ranges = experiment.ranges || getBucketRanges(...);
  assigned = chooseVariation(n, ranges);
}
```

**Sticky Bucket 优化：**
- 如果用户已有 sticky bucket 分配，**跳过** filters/namespace/condition/group/include/prerequisites 检查
- 但仍然会检查 URL 目标、QA 模式、实验停止状态等

## 三、跨实验互斥（Mutual Exclusion）

### 3.1 互斥实现原理

GrowthBook 通过 **Namespace / Filters 机制** 实现跨实验互斥。

**核心思想：**
- 多个实验使用 **相同的 namespace seed**
- 每个实验分配 **不重叠的分桶范围**
- 使用 **相同的 hash attribute** 计算分桶
- 确保用户最多只能进入其中一个实验

### 3.2 Namespace 格式转换

**文件：** `packages/back-end/src/util/features.ts:425-476`

```typescript
export function applyNamespaceToPayload(
  rule: FeatureDefinitionRule,
  namespace: NamespaceValue,
  namespacesMap?: Map<string, { hashAttribute?: string; seed?: string; format?: ... }>,
): void {
  const nsDefinition = namespacesMap?.get(namespace.name);
  const multiRange = nsDefinition
    ? nsDefinition.format === "multiRange"
    : isMultiRangeNamespaceFormat(namespace);

  const ranges = getNamespaceRanges(namespace).map(
    ([start, end]) => [Number(start) || 0, Number(end) || 0] as [number, number],
  );

  if (multiRange) {
    // 新格式：使用 filters 实现互斥
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

  // 旧格式：使用 tuple 形式
  const [start, end] = ranges[0] ?? [0, 0];
  rule.namespace = [namespace.name, start, end];
}
```

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
    ranges: [[0, 0.5]]  // 只允许前 50% 用户
  }],
  variations: ["red", "blue"],
  // ...
}

// 实验 B 的规则
{
  key: "exp_checkout_flow",
  filters: [{
    attribute: "user_id",
    seed: "checkout_flow",  // 相同的 seed
    hashVersion: 2,
    ranges: [[0.5, 1.0]]    // 只允许后 50% 用户
  }],
  variations: ["v1", "v2"],
  // ...
}
```

**分桶结果：**
- 用户 A (hash=0.3)：通过实验 A 的 filter，被实验 A 排除实验 B
- 用户 B (hash=0.7)：通过实验 B 的 filter，被实验 B 排除实验 A
- 保证了两个实验的用户完全不重叠

### 3.4 多范围 Namespace

**文件：** `packages/shared/src/util/namespaces.ts:13-52`

```typescript
type MultiRangeNamespaceValue = {
  enabled: boolean;
  name: string;
  ranges: [number, number][];  // 多个不连续范围
  hashAttribute?: string;      // 自定义 hash attribute
  hashVersion?: number;        // 自定义 hash 版本
  format: "multiRange";
};

export function getNamespaceRanges(namespace: NamespaceValue): [number, number][] {
  if (isLegacyNamespaceFormat(namespace)) {
    return namespace.range ? [namespace.range] : [];
  }
  return namespace.ranges ?? [];
}
```

**多范围用途：**
1. **流量分配调整**：如 `[[0, 0.2], [0.5, 0.7]]` 分配 40% 流量但不连续
2. **增量放量**：从 10% 扩展到 20% 但保留原有用户：`[[0, 0.1], [0.3, 0.4]]`

## 四、Holdout 与互斥的协同工作

### 4.1 完整的规则评估流程

**文件：** `packages/sdk-js/src/core.ts:172-383`

```typescript
export function evalFeature<V = unknown>(id: string, ctx: EvalContext): FeatureResult<V | null> {
  // 1. 循环依赖检测
  if (ctx.stack.evaluatedFeatures.has(id)) {
    return getFeatureResult(ctx, id, null, "cyclicPrerequisite");
  }
  
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
          const evalObj = { value: parentResult.value };
          const evaled = evalCondition(evalObj, parentCondition.condition || {});
          
          if (!evaled) {
            if (parentCondition.gate) {
              // 阻塞型先决条件：整个 feature 返回
              return getFeatureResult(ctx, id, null, "prerequisite");
            }
            // 非阻塞型：跳过当前规则，继续下一个
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

当 feature 关联了 holdout 时：

```
evalFeature("my_feature")
│
├─ 规则 0: Holdout Gate Rule
│   └─ parentConditions: [{ id: "$holdout:hld_123", condition: { value: "holdoutcontrol" } }]
│       └─ evalFeature("$holdout:hld_123")
│           ├─ 默认值: "genpop"
│           └─ 运行 holdout 实验规则
│               ├─ hash(seed, user_id, 2) → 0.6
│               ├─ 50% 分桶: [0, 0.5] → "holdoutcontrol", [0.5, 1] → "holdouttreatment"
│               └─ 返回: value="genpop" 或 "holdoutcontrol" 或 "holdouttreatment"
│
├─ 检查 parentCondition 条件: { value: "holdoutcontrol" }
│   ├─ 如果 holdout 值 = "holdoutcontrol": 条件匹配，执行 force 规则
│   │   └─ 返回 holdout 值（如 "off"），跳过后续所有规则
│   └─ 如果 holdout 值 != "holdoutcontrol": 条件不匹配
│       └─ continue rules → 继续执行下一条规则
│
└─ 规则 1: 普通业务规则（force/rollout/experiment）
    └─ 正常执行 ...
```

### 4.3 互斥与 Holdout 的关系

Holdout 本身也是一个实验，因此也可以通过 namespace/filters 与其他实验互斥：

```typescript
// Holdout 实验也可以配置 namespace
const holdoutExperiment = {
  id: "exp_holdout_main",
  phases: [{
    coverage: 0.2,
    namespace: {
      enabled: true,
      name: "main_flow",           // 与其他实验共享同一个 namespace
      ranges: [[0.8, 1.0]],        // 占用 20% 流量
      format: "multiRange" as const,
    },
    // ...
  }],
};
```

这种配置下：
- 实验 A 使用 `[[0, 0.4]]` → 40% 流量
- 实验 B 使用 `[[0.4, 0.8]]` → 40% 流量  
- Holdout 使用 `[[0.8, 1.0]]` → 20% 流量
- 三者互斥，用户最多进入一个

## 五、关键代码位置总结

| 功能模块 | 文件位置 | 关键函数/类型 |
|---------|---------|-------------|
| Holdout 验证器 | `packages/shared/src/validators/holdout.ts` | `holdoutValidator` |
| Holdout 规则生成 | `packages/back-end/src/util/features.ts:592-616` | `getFeatureDefinition()` 中的 holdoutRule |
| Holdout 虚拟 Feature | `packages/back-end/src/services/features.ts:234-269` | `generateHoldoutsPayload()` |
| Feature 定义生成 | `packages/back-end/src/util/features.ts:478-978` | `getFeatureDefinition()` |
| 分桶排除检查 | `packages/sdk-js/src/core.ts:832-840` | `isFilteredOut()` |
| 实验运行流程 | `packages/sdk-js/src/core.ts:385-785` | `runExperiment()` |
| Feature 评估 | `packages/sdk-js/src/core.ts:172-383` | `evalFeature()` |
| Namespace 工具 | `packages/shared/src/util/namespaces.ts` | `getNamespaceRanges()`, `rangesOverlap()` |
| Namespace 应用 | `packages/back-end/src/util/features.ts:425-476` | `applyNamespaceToPayload()` |
| Filter 类型 | `packages/sdk-js/src/types/growthbook.ts:547-556` | `Filter` 接口 |
