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

## 五、生效门禁与能力开关

### 5.1 Prerequisites Capability 能力开关

**文件：** `packages/back-end/src/util/features.ts:563-590`

Holdout 功能**完全依赖** `prerequisites` capability：

```typescript
// 判断是否有 prerequisites 能力
const hasPrerequisites =
  capabilities === undefined || capabilities.includes("prerequisites");

// 1. Holdout 规则仅在有 prerequisites 能力时生成
const holdoutRule: FeatureDefinitionRule[] =
  hasPrerequisites &&
  feature.holdout &&
  holdoutsMap &&
  holdoutsMap.get(feature.holdout.id)?.holdout.environmentSettings?.[environment]?.enabled
    ? [/* holdout 规则 */]
    : [];

// 2. 没有 prerequisites 能力时，feature-level prerequisites 也不会生成
const prerequisiteRules = hasPrerequisites
  ? (feature.prerequisites ?? [])?.map(/* 转换规则 */)
  : [];

// 3. 没有 prerequisites 能力时，包含 gates 的 feature 整个被排除
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
    return null;  // 整个 feature 被排除
  }
}
```

**能力开关影响总结：**

| 场景 | 有 prerequisites 能力 | 无 prerequisites 能力 |
|-----|---------------------|---------------------|
| Holdout 规则 | 生成并插入到规则最前面 | Holdout 规则完全不生成 |
| Feature-level prerequisites | 转换为 parentConditions 规则 | 不生成，若有则整个 feature 被排除 |
| Rule-level prerequisites | 转换为 parentConditions | 不生成，若有则整个 feature 被排除 |
| Experiment-level prerequisites | 转换为 parentConditions | 不生成，若有则整个 feature 被排除 |

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
      projects.length === 0 ||              // 无项目过滤
      holdout.projects.length === 0 ||      // holdout 无项目限制
      holdout.projects.some((p) => projects.includes(p));  // 项目匹配
    if (allowed) result.set(id, value);
  });
  return result;
}
```

**项目门禁生效路径：**

```
SDK Connection (projects: ["p1", "p2"])
    │
    ├─ Feature 过滤
    │   └─ getAllFeatures(context, { projects: ["p1", "p2"] })
    │       └─ 只返回 project 匹配的 feature
    │
    ├─ Experiment 过滤
    │   └─ getAllPayloadExperiments(context, ["p1", "p2"])
    │       └─ 只返回 project 匹配的 experiment
    │
    └─ Holdout 过滤
        └─ buildHoldoutsMapForProjects(holdoutsMap, ["p1", "p2"])
            └─ 只返回 project 匹配的 holdout
                └─ generateHoldoutsPayload()
                    └─ 只生成过滤后的 holdout 虚拟 feature
```

### 5.3 环境门禁（Environment Gating）

**文件：** `packages/back-end/src/models/HoldoutModel.ts:169-217`

```typescript
public async getAllPayloadHoldouts(environment?: string): Promise<Map<...>> {
  const holdouts = await this._find({});
  const filteredHoldouts = holdoutsWithExperiments.filter((h) => {
    // ... 其他过滤条件
    
    // 环境门禁：只返回在该环境启用的 holdout
    if (environment && !h.holdout.environmentSettings[environment]?.enabled) {
      return false;
    }
    return true;
  });
  return new Map(filteredHoldouts.map((h) => [h.holdout.id, h]));
}
```

**环境门禁生效点：**

| 组件 | 环境门禁位置 |
|-----|-------------|
| Holdout 规则生成 | `getFeatureDefinition()` 中检查 `holdout.environmentSettings[environment]?.enabled` |
| Holdout 虚拟 Feature | `getAllPayloadHoldouts(environment)` 中过滤 |
| Feature 规则 | `getRulesForEnvironment()` 根据环境过滤规则 |
| Experiment 规则 | `applyNamespaceToPayload()` 根据环境应用 |

### 5.4 多级门禁联动效果

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
    │   └─ 步骤 3: 生成虚拟 $holdout:xxx feature
    │
    └─ Feature 筛选
        ├─ 步骤 1: getAllFeatures({ projects: ["p1"] }) → 项目过滤
        ├─ 步骤 2: getFeatureDefinition()
        │   ├─ 检查 prerequisites capability
        │   ├─ 检查 holdout environmentSettings.enabled
        │   ├─ 生成 holdout 规则（最前面）
        │   ├─ 生成 prerequisite 规则
        │   └─ 生成业务规则
        └─ 步骤 3: 按环境过滤规则
```

## 六、跨实验互斥的真实保证机制

### 6.1 核心结论：**没有全局强制保证**

**代码证据：** GrowthBook 的 namespace 互斥机制**完全依赖手动配置**，代码中**没有**任何强制验证逻辑来确保同一 namespace 下的实验范围不重叠。

**验证工具函数存在但不用于全局保证：**
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
        return {
          valid: false,
          error: `Ranges [${ranges[i][0]}, ${ranges[i][1]}] and [${ranges[j][0]}, ${ranges[j][1]}] overlap`,
        };
      }
    }
  }
  return { valid: true };
}
```

**搜索结果验证：** 全局搜索 `validateNamespaceRanges`、`overlapping.*namespace` 等关键词，**没有找到任何在 SDK payload 生成时进行跨实验范围验证的代码**。

### 6.2 互斥的实现原理（约定而非强制）

**文件：** `packages/back-end/src/util/features.ts:425-476`

```typescript
export function applyNamespaceToPayload(
  rule: FeatureDefinitionRule,
  namespace: NamespaceValue,
  namespacesMap?: Map<string, { hashAttribute?: string; seed?: string; format?: ... }>,
): void {
  const nsDefinition = namespacesMap?.get(namespace.name);
  
  // 从组织设置获取统一的 hashAttribute 和 seed
  const filterAttribute = getNamespaceHashAttribute(
    namespace,
    nsDefinition?.hashAttribute || rule.hashAttribute || "id",
  );
  const seed = nsDefinition?.seed || namespace.name;
  
  // 直接应用到 rule，不做任何范围冲突检查
  rule.filters = [
    ...(rule.filters || []),
    {
      attribute: filterAttribute,
      seed,
      hashVersion: filterHashVersion,
      ranges,  // 直接使用配置的范围，不检查是否与其他实验重叠
    },
  ];
}
```

**组织级 Namespace 定义：**
**文件：** `packages/shared/types/organization.d.ts:156-173`

```typescript
export interface NamespaceBase {
  name: string;
  label: string;
  description: string;
  status: "active" | "inactive";
}

export interface MultiRangeNamespace extends NamespaceBase {
  format: "multiRange";
  hashAttribute: string;   // 组织级统一配置
  seed: string;            // 组织级统一配置
}
```

### 6.3 互斥失效的场景

由于缺乏全局强制保证，以下情况会导致互斥失效：

| 场景 | 结果 |
|-----|------|
| 同一 namespace 下两个实验配置了重叠范围 | 用户可能同时进入两个实验 |
| 同一 namespace 下两个实验使用不同的 hashAttribute | 分桶基数不同，完全无法保证互斥 |
| 同一 namespace 下两个实验使用不同的 hashVersion | hash 结果不同，完全无法保证互斥 |
| 手动配置错误导致范围重叠 | 用户可能同时进入多个实验 |

### 6.4 互斥的正确使用方式（必须手动保证）

```
组织级 Namespace 定义（一次性配置）
    ├─ name: "checkout_flow"
    ├─ hashAttribute: "user_id"  ← 统一
    ├─ seed: "checkout_flow"     ← 统一
    └─ format: "multiRange"

实验 A 配置
    └─ namespace:
        ├─ name: "checkout_flow"
        └─ ranges: [[0, 0.4]]     ← 手动分配，不重叠

实验 B 配置
    └─ namespace:
        ├─ name: "checkout_flow"
        └─ ranges: [[0.4, 0.8]]   ← 手动分配，不重叠

Holdout 配置（注意：holdout 本身没有 namespace！）
    └─ holdout 实验分桶是全局的
```

## 七、Holdout 与 Namespace/Filter 的真实关系

### 7.1 核心发现：**Holdout 分桶是全局的，不参与 Namespace 互斥**

**代码证据：** `packages/back-end/src/services/features.ts:234-269`

```typescript
export function generateHoldoutsPayload({ holdoutsMap }): Record<string, FeatureDefinition> {
  const holdoutDefs: Record<string, FeatureDefinition> = {};
  holdoutsMap.forEach((holdoutWithExperiment) => {
    const exp = holdoutWithExperiment.holdoutExperiment;
    const holdout = holdoutWithExperiment.holdout;
    
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
          weights: [0.5, 0.5],
          // 注意：这里没有 filters！也没有 namespace！
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

**对比：普通实验的 namespace 应用**
**文件：** `packages/back-end/src/util/features.ts:713-719`

```typescript
if (phase.namespace && phase.namespace.enabled && phase.namespace.name) {
  applyNamespaceToPayload(rule, phase.namespace, namespacesMap);
}
```

### 7.2 Holdout 与普通实验的分桶对比

| 特性 | Holdout 分桶 | 普通实验分桶 |
|-----|-------------|-------------|
| Namespace/Filter | ❌ 没有 | ✅ 可配置 |
| 分桶范围 | 全局 | 可限制在 namespace 内 |
| 与其他实验互斥 | ❌ 不互斥 | ✅ 通过手动配置 namespace 实现 |
| 分桶种子 | holdout 实验自己的 seed | 可使用 namespace 统一 seed |
| hashAttribute | holdout 实验自己的配置 | 可使用 namespace 统一配置 |

### 7.3 Holdout 与 Namespace 协同的正确方式

由于 Holdout 分桶是全局的，要实现 Holdout 与其他实验的互斥，需要：

**方案 1：在业务实验上配置足够大的 coverage 排除**
```
Holdout: coverage = 0.2 (全局 20% 用户)
业务实验 A: namespace = "main", ranges = [[0.2, 1.0]] (剩下的 80%)
业务实验 B: namespace = "other", ranges = [[0.2, 1.0]] (剩下的 80%)
```

**方案 2：使用 prerequisites 实现间接互斥**
```
Feature A (关联 holdout)
    └─ holdout 规则作为 prerequisite (最前面)
        └─ 如果用户在 holdoutcontrol 组，直接返回 holdout 值
        └─ 否则继续执行后续规则（可能包含 namespace）
```

### 7.4 Holdout 互斥的实际生效路径

```
evalFeature("checkout_button")
    │
    ├─ 规则 0: Holdout Gate Rule (parentConditions)
    │   └─ evalFeature("$holdout:hld_main")
    │       └─ 分桶逻辑：全局的，无 namespace
    │           ├─ hash(holdout_seed, user_id)
    │           ├─ coverage = 0.5
    │           ├─ 50% → "holdoutcontrol"
    │           └─ 50% → "holdouttreatment" (或 genpop)
    │
    ├─ 检查 parentCondition: { value: "holdoutcontrol" }
    │   ├─ ✅ 匹配 → 执行 force: "off" → 返回，不进入后续实验
    │   └─ ❌ 不匹配 → continue 到下一条规则
    │
    └─ 规则 1: 业务实验规则
        └─ filters: [{ seed: "checkout_flow", ranges: [[0, 0.5]] }]
            └─ 分桶逻辑：namespace 内的
                ├─ hash("checkout_flow", user_id)
                └─ 在范围内才进入实验
```

**关键点：**
- Holdout 分桶和业务实验的 namespace 分桶是**两次独立的 hash 计算**
- 使用不同的 seed，因此分桶结果是独立的
- Holdout 是通过 **prerequisite + force** 实现的"软互斥"，而不是 namespace 层面的"硬互斥"

## 八、关键代码位置总结

| 功能模块 | 文件位置 | 关键函数/类型 |
|---------|---------|-------------|
| Holdout 验证器 | `packages/shared/src/validators/holdout.ts` | `holdoutValidator` |
| Holdout 规则生成 | `packages/back-end/src/util/features.ts:592-616` | `getFeatureDefinition()` 中的 holdoutRule |
| Holdout 虚拟 Feature | `packages/back-end/src/services/features.ts:234-269` | `generateHoldoutsPayload()` |
| Prerequisites Capability 检查 | `packages/back-end/src/util/features.ts:563-590` | `hasPrerequisites` 变量 |
| 项目门禁 | `packages/back-end/src/services/features.ts:209-232` | `buildHoldoutsMapForProjects()` |
| 环境门禁 | `packages/back-end/src/models/HoldoutModel.ts:169-217` | `getAllPayloadHoldouts()` |
| Feature 定义生成 | `packages/back-end/src/util/features.ts:478-978` | `getFeatureDefinition()` |
| 分桶排除检查 | `packages/sdk-js/src/core.ts:832-840` | `isFilteredOut()` |
| 实验运行流程 | `packages/sdk-js/src/core.ts:385-785` | `runExperiment()` |
| Feature 评估 | `packages/sdk-js/src/core.ts:172-383` | `evalFeature()` |
| Namespace 工具 | `packages/shared/src/util/namespaces.ts` | `getNamespaceRanges()`, `rangesOverlap()` |
| Namespace 应用 | `packages/back-end/src/util/features.ts:425-476` | `applyNamespaceToPayload()` |
| Filter 类型 | `packages/sdk-js/src/types/growthbook.ts:547-556` | `Filter` 接口 |

## 九、核心结论

1. **Holdout 依赖 prerequisites capability**：没有该能力时，Holdout 规则完全不生成，包含 prerequisites 的 feature 被整个排除

2. **项目/环境门禁是多级联动的**：Feature、Experiment、Holdout 各自有独立的项目和环境过滤逻辑

3. **跨实验互斥没有全局强制保证**：完全依赖手动配置，代码中没有验证机制，配置错误会导致用户同时进入多个实验

4. **Holdout 分桶是全局的**：Holdout 虚拟 feature 没有 namespace/filter，分桶是全局的，与业务实验的分桶是两次独立计算

5. **Holdout 互斥是"软互斥"**：通过 prerequisite + force 规则实现，而不是 namespace 层面的"硬互斥"
