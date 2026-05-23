# GrowthBook 功能开关分阶段放量与用户粘性分桶协同机制分析

## 1. 核心概念与架构概览

### 1.1 功能开关分阶段放量（Rollout）

分阶段放量是指将功能逐步推送给一部分用户，通过控制 `coverage`（覆盖范围）参数实现。当 coverage 从 0 → 10% → 25% → 50% → 75% → 100% 的渐进式放量过程。

**核心代码位置：**
- `packages/sdk-js/src/core.ts:842-868` - `isIncludedInRollout` 函数
- `packages/sdk-js/src/util.ts:184-232` - `getBucketRanges` 函数

### 1.2 用户粘性分桶（Sticky Bucketing）

粘性分桶确保用户在实验配置变更时保持相同的变体分配，避免用户在不同变体间"跳动"。

**核心代码位置：**
- `packages/sdk-js/src/sticky-bucket-service.ts` - 粘性分桶服务抽象与多种存储实现
- `packages/sdk-js/src/core.ts:983-1036` - `getStickyBucketVariation` 函数

---

## 2. 命中规则分析

### 2.1 Hash 算法与分桶机制

GrowthBook 采用 FNV-32a 哈希算法，支持两个版本：

```typescript
// packages/sdk-js/src/util.ts:31-47
export function hash(seed: string, value: string, version: number): number | null {
  // v2: 无偏哈希算法 (hashFnv32a(hashFnv32a(seed + value) % 10000) / 10000
  // v1: 有偏哈希算法 (hashFnv32a(value + seed) % 1000) / 1000
}
```

**Hash 版本差异：**
- v2 使用 `seed + value` 顺序，结果范围 [0, 1)，精度 0.0001
- v1 使用 `value + seed` 顺序，结果范围 [0, 1)，精度 0.001

### 2.2 分阶段放量命中规则

`isIncludedInRollout` 函数（`packages/sdk-js/src/core.ts:842-868`）：

```typescript
function isIncludedInRollout(
  ctx: EvalContext,
  seed: string,
  hashAttribute: string | undefined,
  fallbackAttribute: string | undefined,
  range: VariationRange | undefined,
  coverage: number | undefined,
  hashVersion: number | undefined,
): boolean
```

**判断逻辑：**
1. 无 `range` 且 `coverage` 未定义 → 全部命中
2. 无 `range` 且 `coverage === 0` → 全部不命中
3. 获取 `hashAttribute`，若无则尝试 `fallbackAttribute`
4. 计算 `hash(seed, hashValue, hashVersion || 1)`
5. 若提供 `range` → 使用 `inRange(n, range)` 判断
6. 提供 `coverage` → 使用 `n <= coverage` 判断

### 2.3 功能开关规则评估流程

`evalFeature` 函数（`packages/sdk-js/src/core.ts:172-383`）的完整评估链：

```
1. 检查 feature 是否存在
2. 遍历 rules 数组（按优先级顺序）
   ├─ 2.1 检查前置条件（parentConditions）
   ├─ 2.2 检查过滤器（filters）
   ├─ 2.3 检查条件（condition）
   ├─ 2.4 若是 force 规则：
   │   └─ 调用 isIncludedInRollout 判断是否在放量范围内
   └─ 2.5 若是 experiment 规则：
       └─ 调用 runExperiment 进行实验分桶
```

### 2.4 分桶范围计算

`getBucketRanges` 函数（`packages/sdk-js/src/util.ts:184-232`）：

```typescript
export function getBucketRanges(
  numVariations: number,
  coverage: number | undefined,
  weights?: number[],
): VariationRange[]
```

**算法：**
1. coverage 规范化到 [0, 1] 范围
2. weights 校验（默认均分，校验总和 ≈ 1）
3. 计算每个变体的范围：`[start, start + coverage * weight]`

**示例：** coverage=0.5, weights=[0.5, 0.5]
- 变体 0: [0, 0.25)
- 变体 1: [0.25, 0.5)
- 未覆盖: [0.5, 1.0)

---

## 3. 粘性分桶与分阶段放量的协同机制

### 3.1 粘性分桶数据结构

```typescript
// packages/sdk-js/src/types/growthbook.ts
type StickyAssignmentsDocument = {
  attributeName: string;      // 用于分桶的属性名（如 "id"）
  attributeValue: string;       // 属性值（如 "user123"）
  assignments: StickyAssignments;  // 实验键 → 变体键的映射
}

type StickyAssignments = Record<StickyExperimentKey, string>;
// key: `${experimentKey}__${bucketVersion}`
// value: 变体的 meta.key（如 "0", "1", "2"）
```

### 3.2 粘性分桶键生成

```typescript
// packages/sdk-js/src/core.ts:1038-1044
function getStickyBucketExperimentKey(
  experimentKey: string,
  experimentBucketVersion?: number,
): StickyExperimentKey {
  return `${experimentKey}__${experimentBucketVersion || 0}`;
}

// packages/sdk-js/src/core.ts:1046-1051
export function getStickyBucketAttributeKey(
  attributeName: string,
  attributeValue: string,
): StickyAttributeKey {
  return `${attributeName}||${attributeValue}`;
}
```

### 3.3 协同工作流程

`runExperiment` 函数（`packages/sdk-js/src/core.ts:385-760`）中的粘性分桶与放量协同：

```
步骤 1: 获取 hashAttribute 和 fallbackAttribute
         ↓
步骤 2: 检查粘性分桶（若启用且未禁用）
         ├─ 调用 getStickyBucketVariation()
         │   ├─ 检查 minBucketVersion 版本阻挡
         │   ├─ 查找 fallbackAttribute 的粘性分配
         │   └─ 查找 hashAttribute 的粘性分配
         └─ 若找到 → 直接使用该变体，跳过后续过滤
         ↓
步骤 3: 若无粘性分配
         ├─ 检查 filters/namespace
         ├─ 检查 condition
         ├─ 检查 groups
         ├─ 计算 hash 值
         ├─ 计算 bucketRanges（含 coverage）
         └─ chooseVariation() 选择变体
         ↓
步骤 4: 若命中变体
         ↓
步骤 5: 保存粘性分配（若启用）
         └─ generateStickyBucketAssignmentDoc()
```

### 3.4 粘性分桶查询逻辑

`getStickyBucketVariation` 函数（`packages/sdk-js/src/core.ts:983-1036`）：

```typescript
function getStickyBucketVariation({
  ctx, expKey, expBucketVersion, expHashAttribute,
  expFallbackAttribute, expMinBucketVersion, expMeta
}): { variation: number; versionIsBlocked?: boolean }
```

**关键逻辑：**
1. 构建当前版本的 bucket key
2. 调用 `getStickyBucketAssignments` 获取合并后的分配
   - 先合并 fallbackAttribute 的分配
   - 再合并 hashAttribute 的分配（优先级更高）
3. 检查 minBucketVersion 阻挡
   - 若 expMinBucketVersion > 0，检查 0 到 expMinBucketVersion-1 版本是否存在
   - 存在则返回 `{ variation: -1, versionIsBlocked: true }`
4. 查找当前版本的分配
   - 找到 → 通过 meta.key 匹配找到变体索引
   - 未找到 → 返回 -1

### 3.5 粘性分配持久化

`generateStickyBucketAssignmentDoc` 函数（`packages/sdk-js/src/core.ts:1087-1116`）：

```typescript
function generateStickyBucketAssignmentDoc(
  ctx, attributeName, attributeValue, assignments
): { key, doc, changed }
```

**持久化触发条件**（`packages/sdk-js/src/core.ts:728-760`）：
- `ctx.user.saveStickyBucketAssignmentDoc` 为 true
- `experiment.disableStickyBucketing` 为 false

---

## 4. 变更后的回退机制

### 4.1 版本控制机制

通过两个关键参数控制回退：

| 参数 | 类型 | 作用 |
|------|------|------|
| `bucketVersion` | number | 当前分桶版本，变更时递增 |
| `minBucketVersion` | number | 最小允许版本，低于此版本的用户被排除 |

**类型定义**（`packages/sdk-js/src/types/growthbook.ts:35-36, 110-111`）：

```typescript
type FeatureRule<T> = {
  bucketVersion?: number;
  minBucketVersion?: number;
  // ...
}
```

### 4.2 版本阻挡逻辑

当实验配置变更（如权重调整、coverage 变化）时：

**场景 1：调整 coverage 从 50% → 10%，保留老用户：**
- 仅调整 `coverage` 参数
- 老用户因粘性分桶保持原分配
- 新用户按新 coverage 计算

**场景 2：发现 bug 修复后重新放量，排除老用户：**
- 递增 `bucketVersion` (0 → 1)
- 设置 `minBucketVersion = 1`
- 老用户因版本 0 存在被阻挡，返回 -1

**版本阻挡代码**（`packages/sdk-js/src/core.ts:1014-1025`）：

```typescript
if (expMinBucketVersion > 0) {
  for (let i = 0; i < expMinBucketVersion; i++) {
    const blockedKey = getStickyBucketExperimentKey(expKey, i);
    if (assignments[blockedKey] !== undefined) {
      return { variation: -1, versionIsBlocked: true };
    }
  }
}
```

### 4.3 回退处理流程

`packages/sdk-js/src/core.ts:646-663` 版本阻挡后的处理：

```typescript
if (stickyBucketVersionIsBlocked) {
  return {
    result: getExperimentResult(
      ctx, experiment, -1, false, featureId, undefined, true
    ),
  };
}
```

**效果：** 用户被排除在实验之外，使用默认值（control）。

### 4.4 自动回退（Safe Rollouts）

安全放量功能（`docs/docs/features/safe-rollouts.mdx`）提供自动化回退能力：

- 配置 guardrail metrics（护栏指标）监控
- 检测到回归时自动禁用 rollout 规则
- 支持自动回退（Auto Rollback）开关

**放量阶梯：** 10% → 25% → 50% → 75% → 100%
**监控周期：** 前 25% 时间完成放量，剩余时间监控

---

## 5. 跨 SDK 一致性表现

### 5.1 跨 SDK 能力矩阵

从 `packages/shared/src/sdk-versioning/CAPABILITIES.md` 中的 `stickyBucketing` 支持情况：

| SDK | 最低版本 |
|-----|----------|
| JavaScript | 0.32.0 |
| Node.js | 0.32.0 |
| Next.js | 0.1.0 |
| React | 0.22.0 |
| PHP | 1.6.0 |
| Python | 1.1.0 |
| Ruby | 1.3.0 |
| Java | 0.9.3 |
| Android | 1.1.44 |
| iOS | 1.0.49 |
| Go | 0.2.3 |
| Flutter | 3.8.0 |
| C# | 1.1.0 |
| Rust | 0.1.0 |
| Edge (Cloudflare/Fastly/Lambda/Other) | 0.1.11 / 0.1.5 / 0.0.6 / 0.1.4 |

### 5.2 一致性保证机制

**1. 统一的 Hash 算法一致性：**
所有 SDK 必须实现相同的 FNV-32a 哈希算法

**2. 标准化的数据结构：**
- `StickyAssignmentsDocument` 结构跨语言标准化

**3. 存储服务抽象：**
`StickyBucketService` 抽象类定义了标准接口：
```typescript
// packages/sdk-js/src/sticky-bucket-service.ts:50-95
abstract class StickyBucketService {
  abstract getAssignments(attributeName, attributeValue)
  abstract saveAssignments(doc)
  async getAllAssignments(attributes)
  getKey(attributeName, attributeValue)
}
```

**4. 多存储实现：**
- `LocalStorageStickyBucketService` - 浏览器 LocalStorage
- `BrowserCookieStickyBucketService` - 浏览器 Cookie
- `ExpressCookieStickyBucketService` - 后端 Express Cookie
- `RedisStickyBucketService` - Redis 分布式存储

### 5.3 跨端一致性方案

**前后端一致场景**（`docs/docs/sticky-bucketing.mdx:129-134`）：

```
前端 (BrowserCookieStickyBucketService)
    ↓ Cookie 传输
后端 (ExpressCookieStickyBucketService)
```

**关键点：** 必须使用相同的 `prefix` 配置

**跨服务一致场景：**
使用 `RedisStickyBucketService` 作为集中存储

**跨设备一致场景：**
- 主 hashAttribute: `userId`
- fallbackAttribute: `deviceId` / `cookieId`
- 登录后自动"升级"到 userId 分配

### 5.4 Fallback Attribute 机制

`getStickyBucketAssignments` 函数（`packages/sdk-js/src/core.ts:1053-1085`）：

```typescript
function getStickyBucketAssignments(
  ctx, expHashAttribute, expFallbackAttribute?
): StickyAssignments {
  // 1. 获取 hashAttribute 的分配
  const hashKey = getStickyBucketAttributeKey(hashAttribute, hashValue);
  // 2. 获取 fallbackAttribute 的分配
  const fallbackKey = getStickyBucketAttributeKey(fallbackAttribute, fallbackValue);
  // 3. 合并：先 fallback，后 hashAttribute（优先级更高）
  const assignments = {};
  if (fallbackKey && docs[fallbackKey]) {
    Object.assign(assignments, docs[fallbackKey].assignments);
  }
  if (docs[hashKey]) {
    Object.assign(assignments, docs[hashKey].assignments);
  }
  return assignments;
}
```

**升级逻辑**（测试用例 `packages/sdk-js/test/sticky-buckets.test.ts:337-393`）：
1. 未登录时使用 `deviceId` → 分配变体 A
2. 登录后提供 `id` 和 `deviceId`
   - 先找到 `deviceId` 的分配（变体 A）
   - 保存时同时写入 `id` 的分配
3. 登出后仅 `id` → 仍命中变体 A

---

## 6. 关键代码路径总结

### 6.1 功能开关放量评估路径

```
evalFeature(id, ctx)
  ↓
for (rule of feature.rules)
  ↓
isIncludedInRollout(ctx, rule.seed || id,
                      rule.hashAttribute,
                      rule.fallbackAttribute,
                      rule.range, rule.coverage,
                      rule.hashVersion)
  ↓
  ├─ 有 range → inRange(hash(...), range)
  └─ 有 coverage → hash(...) <= coverage
```

### 6.2 实验分桶评估路径

```
runExperiment(experiment, featureId, ctx)
  ↓
getStickyBucketVariation(...)
  ↓
  ├─ 找到粘性分配 → 使用
  └─ 未找到 →
       ↓
       ├─ 检查 filters/namespace/condition/groups
       ├─ hash(seed || key, hashValue, hashVersion || 1)
       ├─ getBucketRanges(numVariations, coverage, weights)
       └─ chooseVariation(n, ranges)
  ↓
getExperimentResult(...)
  ↓
saveStickyBucketAssignmentDoc(...)  [若启用]
```

### 6.3 粘性分桶读写路径

**读取：**
```
getAllStickyBucketAssignmentDocs(ctx, service, data)
  ↓
getStickyBucketAttributes(ctx, data)  // 收集所需的 attributes
  ↓
stickyBucketService.getAllAssignments(attributes)
  ↓
// 各存储实现的 getAssignments / getAllAssignments
```

**写入：**
```
generateStickyBucketAssignmentDoc(ctx, attributeName, attributeValue, assignments)
  ↓
stickyBucketService.saveAssignments(doc)
```

---

## 7. 测试用例验证

### 7.1 粘性分桶持久化测试

`packages/sdk-js/test/sticky-buckets.test.ts` 中的关键测试场景：

**测试 1：localStorage 驱动的读写升级**
- 验证：无 `id`，有 `deviceId` → 分配变体
- 后：有 `id` 和 `deviceId` → 保持原变体
- 后：无 `deviceId`，有 `id` → 仍保持原变体

**测试 2：bucketVersion 变更重置**
- bucketVersion: 0 → 1
- 预期：stickyBucketUsed 从 true → false

**测试 3：minBucketVersion 阻挡**
- minBucketVersion: 0 → 1
- 预期：已分配用户被排除，返回默认值

**测试 4：disableStickyBucketing 禁用**
- disableStickyBucketing: true
- 预期：不使用粘性分配，重新计算

---

## 8. 架构设计考量

### 8.1 优点

1. **用户体验一致性：** 粘性分桶避免用户体验跳动
2. **灵活的版本控制：** `bucketVersion`/`minBucketVersion` 提供精确控制
3. **多存储后端适配：** Cookie/LocalStorage/Redis 等
4. **跨 SDK 标准化：** 统一的 Hash 算法和数据结构
5. **渐进式放量：** Safe Rollouts 自动阶梯放量 + 护栏监控

### 8.2 权衡点

1. **统计偏差风险**（Fallback Attribute 可能导致的统计偏差）
   - 同一用户多设备被计为多个样本
   - 变体分配不满足独立同分布假设

2. **存储成本**：粘性分配需要持久化存储

3. **版本管理复杂度**：bucketVersion 管理需要谨慎

### 8.3 最佳实践

1. **coverage 调整不更改 bucketVersion：** 仅调整流量比例时保持版本
2. **重大变更**：递增 `bucketVersion` + `minBucketVersion`
3. **跨端一致**：前后端使用相同 `prefix` + Redis 存储
4. **监控**：合理设置 guardrail metrics
5. **回退预案**：配置 Auto Rollback 降低风险

---

## 9. 核心文件索引

| 文件 | 核心功能 |
|------|----------|
| `packages/sdk-js/src/core.ts` | 核心评估逻辑、粘性分桶查询与回退 |
| `packages/sdk-js/src/util.ts` | Hash 算法、分桶范围计算 |
| `packages/sdk-js/src/sticky-bucket-service.ts` | 粘性分桶存储抽象与实现 |
| `packages/sdk-js/src/types/growthbook.ts` | 类型定义 |
| `packages/sdk-js/test/sticky-buckets.test.ts` | 粘性分桶测试用例 |
| `packages/shared/src/sdk-versioning/CAPABILITIES.md` | 跨 SDK 能力矩阵 |
| `docs/docs/sticky-bucketing.mdx` | 粘性分桶文档 |
| `docs/docs/features/safe-rollouts.mdx` | 安全放量文档 |
