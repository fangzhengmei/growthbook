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

---

## 附录 A：JavaScript/Node、Python、Go 三个 SDK 逐段对照

> 本附录基于以下信息源进行分析：
> 1. **JavaScript/Node SDK** - 本仓库中完整的源代码实现（`packages/sdk-js/`）
> 2. **Python SDK** - 官方文档（`docs/docs/lib/python.mdx`）+ SDK 构建规范（`docs/docs/lib/build-your-own.mdx`）
> 3. **Go SDK** - 官方文档（`docs/docs/lib/go.mdx`）+ SDK 构建规范 + 跨 SDK 测试用例（`packages/sdk-js/test/cases.json`）

### A.1 版本支持与能力矩阵

| 能力 | JavaScript/Node | Python | Go |
|------|----------------|--------|----|
| **stickyBucketing 最低版本** | 0.32.0 | 1.1.0 | 0.2.3 |
| **bucketingV2（Hash v2）** | 0.23.0 | 1.0.0 | 0.1.4 |
| **prerequisites** | 0.34.0 | 1.1.0 | 0.2.0 |
| **savedGroupReferences** | 1.1.0 | 1.2.1 | 0.2.0 |
| **SDK 最新版本** | 1.6.5 | 2.1.1 | 0.2.6 |
| **异步客户端支持** | ✅ Node.js 异步 | ✅ GrowthBookClient（v1.2.0+） | ✅ 原生 goroutine 支持 |
| **内置存储实现** | LocalStorage、Cookie（Browser/Express）、Redis | InMemory、SQLite（示例） | InMemory（内置） |

### A.2 粘性分桶服务接口定义对照

#### A.2.1 JavaScript/Node SDK 接口

```typescript
// packages/sdk-js/src/sticky-bucket-service.ts:50-95
abstract class StickyBucketService {
  abstract getAssignments(
    attributeName: string,
    attributeValue: string
  ): StickyAssignmentsDocument | null | undefined;

  abstract saveAssignments(doc: StickyAssignmentsDocument): void;

  async getAllAssignments(
    attributes: Record<string, string>
  ): Promise<Record<string, StickyAssignmentsDocument>> {
    // 批量获取实现
  }

  getKey(attributeName: string, attributeValue: string): string {
    return `${attributeName}||${attributeValue}`;
  }
}

// 数据结构
interface StickyAssignmentsDocument {
  attributeName: string;
  attributeValue: string;
  assignments: Record<string, string>; // `${experimentKey}__${bucketVersion}` → `${variationKey}`
}
```

#### A.2.2 Python SDK 接口

```python
# docs/docs/lib/python.mdx:974-981
class AbstractStickyBucketService:
    def get_assignments(
        self,
        attribute_name: str,
        attribute_value: str
    ) -> Optional[Dict]:
        """Lookup a sticky bucket document"""
        return None

    def save_assignments(self, doc: Dict) -> None:
        """Save sticky bucket assignments"""
        pass

# 数据结构（文档示例）
{
  "attributeName": "id",
  "attributeValue": "123",
  "assignments": {"exp1__0": "control"}
}
```

#### A.2.3 Go SDK 接口

```go
// docs/docs/lib/go.mdx:619-623
type StickyBucketService interface {
    GetAssignments(
        attributeName string,
        attributeValue string
    ) (*StickyBucketAssignmentDoc, error)

    SaveAssignments(doc *StickyBucketAssignmentDoc) error

    GetAllAssignments(
        attributes map[string]string
    ) (StickyBucketAssignments, error)
}

// 数据结构
type StickyBucketAssignmentDoc struct {
    AttributeName  string            `json:"attributeName"`
    AttributeValue string            `json:"attributeValue"`
    Assignments    map[string]string `json:"assignments"`
}
```

#### A.2.4 接口对照分析

| 特性 | JavaScript/Node | Python | Go |
|------|----------------|--------|----|
| **抽象基类** | `abstract class` | `AbstractStickyBucketService` | `interface` |
| **getAssignments 返回类型** | `Document \| null \| undefined` | `Optional[Dict]` | `(*Document, error)` |
| **saveAssignments 返回类型** | `void` | `None` | `error` |
| **getAllAssignments** | ✅ `async` 实现 | ❌ 无（接口未定义） | ✅ 接口定义 |
| **getKey 工具方法** | ✅ 基类实现 | ❌ 无 | ❌ 无 |
| **异常处理** | 依赖语言 try/catch | 依赖 try/except | Go 多返回值 error |

### A.3 粘性分桶命中逻辑对照

#### A.3.1 命中执行流程（共性）

所有 SDK 必须遵循以下执行顺序（来自 `build-your-own.mdx:924-967` 规范）：

```
1. 获取 hashAttribute 值
   ├─ 若为空，尝试 fallbackAttribute
   └─ 两者都为空 → 不命中，返回默认值
2. 检查是否允许粘性分桶
   ├─ experiment.disableStickyBucketing !== true
   └─ StickyBucketService 已配置
3. 查询粘性分配（getStickyBucketVariation）
   ├─ 优先查询 hashAttribute 的分配
   ├─ 其次查询 fallbackAttribute 的分配
   ├─ 检查 minBucketVersion 版本阻挡
   └─ 找到 → 跳过后续过滤，直接使用
4. 未命中粘性分配 → 执行标准流程
   ├─ 检查 filters/namespace
   ├─ 检查 condition
   ├─ 检查 coverage → getBucketRanges
   └─ chooseVariation() 选择变体
5. 命中变体 → 保存粘性分配（若允许）
```

#### A.3.2 粘性分桶查询实现（JavaScript/Node）

```typescript
// packages/sdk-js/src/core.ts:983-1036
function getStickyBucketVariation({
  ctx, expKey, expBucketVersion, expHashAttribute,
  expFallbackAttribute, expMinBucketVersion, expMeta
}): { variation: number; versionIsBlocked?: boolean } {
  // 1. 构建当前版本的 bucket key
  const key = getStickyBucketExperimentKey(expKey, expBucketVersion || 0);

  // 2. 获取合并后的分配（fallbackAttribute 优先，hashAttribute 覆盖）
  const assignments = getStickyBucketAssignments(
    ctx, expHashAttribute, expFallbackAttribute
  );

  // 3. 检查 minBucketVersion 阻挡
  if (expMinBucketVersion > 0) {
    for (let i = 0; i < expMinBucketVersion; i++) {
      const blockedKey = getStickyBucketExperimentKey(expKey, i);
      if (assignments[blockedKey] !== undefined) {
        return { variation: -1, versionIsBlocked: true };
      }
    }
  }

  // 4. 查找当前版本的分配
  const assigned = assignments[key];
  if (assigned !== undefined) {
    // 通过 meta.key 匹配找到变体索引
    const idx = expMeta?.findIndex(m => m.key === assigned) ?? -1;
    if (idx >= 0) return { variation: idx };
  }

  return { variation: -1 };
}
```

#### A.3.3 版本键生成规范（共性）

```typescript
// 所有 SDK 必须遵循的键格式
experimentKey = `${experimentKey}__${bucketVersion}`  // 如 "feature-exp__3"
attributeKey = `${attributeName}||${attributeValue}`   // 如 "id||i123"
```

**测试用例验证**（`cases.json:6541-6624`）：
```
场景：hashAttribute 和 fallbackAttribute 同时有分配
- fallback(anonymousId): "feature-exp__0" → "2"
- hashAttribute(id): "feature-exp__0" → "1"
- 预期结果：使用 hashAttribute 的分配 "1"
- stickyBucketUsed: true
```

#### A.3.4 命中逻辑差异分析

| 逻辑环节 | JavaScript/Node | Python | Go |
|----------|----------------|--------|----|
| **fallbackAttribute 取值时机** | hashAttribute 为空时，第 6 步（run 函数） | 同规范 | 同规范 |
| **分配合并顺序** | fallback → hashAttribute（hash 覆盖 fallback） | 同规范 | 同规范 |
| **minBucketVersion 检查范围** | 0 到 minBucketVersion-1 的所有版本 | 同规范 | 同规范 |
| **变体键匹配** | 通过 experiment.meta[i].key 匹配 | 同规范 | 同规范 |
| **meta 缺失时的键** | `String(variationIndex)` 如 "0"、"1" | 同规范 | 同规范 |
| **StickyBucketUsed 设置时机** | 使用了粘性分配时设为 true | 同规范 | 同规范 |

### A.4 分阶段放量（Rollout）命中规则对照

#### A.4.1 isIncludedInRollout 实现（共性规范）

```typescript
// build-your-own.mdx:719-746
function isIncludedInRollout(
  seed: string,
  hashAttribute: string | null,
  range: BucketRange | null,
  coverage: float | null,
  hashVersion: integer | null
): boolean {
  // 1. 无 coverage 无 range → 全部命中
  if (range === null && coverage === null) return true;

  // 2. coverage 为 0 → 全部不命中（edge case）
  if (range === null && coverage === 0) return false;

  // 3. 获取 hash 值
  const hashAttr = hashAttribute || "id";
  const hashValue = context.attributes[hashAttr] || "";
  if (hashValue === "") return false;

  // 4. 计算 hash
  const n = hash(seed, hashValue, hashVersion || 1);

  // 5. 判断是否在范围内
  if (range) return inRange(n, range);
  if (coverage !== null) return n <= coverage;
  return true;
}
```

#### A.4.2 Hash 算法规范（绝对共性）

```
所有 SDK 必须实现完全相同的 FNV-32a 哈希算法：

v2（无偏）: hashFnv32a(hashFnv32a(seed + value) + "") % 10000 / 10000
v1（有偏）: hashFnv32a(value + seed) % 1000 / 1000
```

**测试用例验证**（`cases.json` 中 hash 测试用例共 12 个，所有 SDK 必须 100% 通过）：
```
[seed, value, version, expected]
["", "", 1, 0.361]
["test", "abc", 1, 0.619]
["test", "abc", 2, 0.5069]
["foo", "bar", 2, 0.6281]
...
```

#### A.4.3 放量规则与粘性分桶的交互

**关键交互点**（所有 SDK 必须一致）：

| 场景 | 行为 | 测试用例位置 |
|------|------|-------------|
| **粘性命中时跳过 coverage 检查** | 找到粘性分配 → 跳过 filters、condition、coverage 检查 | `cases.json:6366-6424` |
| **粘性分配升级（fallback → hash）** | 有 fallback 分配 + hashAttribute 有值 → 同时写入两个文档 | `cases.json:6488-6552` |
| **bucketVersion 变更重置** | 新版本无粘性分配 → 重新计算 coverage | `cases.json:6627-6686` |
| **minBucketVersion 阻挡** | 低版本存在 → 排除用户，不保存新分配 | `cases.json:6688-6736` |

### A.5 变更回退机制对照

#### A.5.1 版本控制参数（共性）

| 参数 | JavaScript/Node 类型 | Python 类型 | Go 类型 |
|------|---------------------|-------------|---------|
| `bucketVersion` | `number`（可选，默认 0） | `int`（可选） | `int`（可选） |
| `minBucketVersion` | `number`（可选，默认 0） | `int`（可选） | `int`（可选） |
| `disableStickyBucketing` | `boolean`（可选） | `bool`（可选） | `bool`（可选） |

#### A.5.2 回退场景处理（共性）

**场景 1：仅调整 coverage（50% → 10%），保留老用户**
- 操作：仅修改 `coverage` 参数
- 行为：
  - 老用户因粘性分桶保持原分配（跳过 coverage 检查）
  - 新用户按新 coverage 计算
- 跨 SDK 一致性：✅ 所有 SDK 行为一致

**场景 2：Bug 修复后重新放量，排除老用户**
- 操作：
  - `bucketVersion`: 0 → 1
  - `minBucketVersion`: 1
- 行为：
  - 老用户因版本 0 存在，被 minBucketVersion=1 阻挡
  - 返回 `variation: -1`，使用默认值
  - 不保存新的粘性分配
- 跨 SDK 一致性：✅ 所有 SDK 行为一致
- 测试用例：`cases.json:6688-6736`

**场景 3：完全重置实验**
- 操作：
  - `bucketVersion`: 0 → 1
  - `minBucketVersion`: 0（或不设置）
- 行为：
  - 老用户无版本 1 的粘性分配
  - 重新计算 coverage，可能分配到不同变体
  - 保存新的版本 1 分配（保留旧版本 0 分配）
- 跨 SDK 一致性：✅ 所有 SDK 行为一致
- 测试用例：`cases.json:6627-6686`

#### A.5.3 回退逻辑差异分析

| 回退环节 | JavaScript/Node | Python | Go |
|----------|----------------|--------|----|
| **版本阻挡返回值** | `{ variation: -1, versionIsBlocked: true }` | 同规范（返回 -1） | 同规范（返回 -1） |
| **阻挡后是否保存分配** | ❌ 不保存 | ❌ 不保存 | ❌ 不保存 |
| **阻挡后 inExperiment** | false | false | false |
| **旧版本分配保留** | ✅ 保留（不删除） | ✅ 保留 | ✅ 保留 |
| **多版本分配共存** | ✅ 允许（如同时有 `__0` 和 `__3`） | ✅ 允许 | ✅ 允许 |

### A.6 存储实现对照

#### A.6.1 JavaScript/Node SDK 存储实现

```typescript
// packages/sdk-js/src/sticky-bucket-service.ts
// 1. LocalStorage 实现
class LocalStorageStickyBucketService extends StickyBucketService {
  constructor(private prefix = "gbStickyBuckets__") {}

  getAssignments(attributeName, attributeValue) {
    const key = this.getKey(attributeName, attributeValue);
    const json = localStorage.getItem(this.prefix + key);
    return json ? JSON.parse(json) : null;
  }

  saveAssignments(doc) {
    const key = this.getKey(doc.attributeName, doc.attributeValue);
    localStorage.setItem(this.prefix + key, JSON.stringify(doc));
  }
}

// 2. Browser Cookie 实现
class BrowserCookieStickyBucketService extends StickyBucketService {
  constructor(private prefix = "gbStickyBuckets__", private days?: number) {}

  // Cookie 读写逻辑
}

// 3. Express Cookie 实现
class ExpressCookieStickyBucketService extends StickyBucketService {
  constructor(private req: Request, private res: Response, ...) {}
}
```

#### A.6.2 Python SDK 存储实现

```python
# docs/docs/lib/python.mdx:1082-1128
# SQLite 示例实现
class SQLiteStickyBucketService(AbstractStickyBucketService):
    def __init__(self, db_path="sticky_buckets.db"):
        self.conn = sqlite3.connect(db_path)
        # 创建表：attribute_name, attribute_value, assignments (JSON)

    def get_assignments(self, attribute_name, attribute_value):
        cursor = self.conn.execute(
            "SELECT assignments FROM sticky_buckets WHERE attribute_name=? AND attribute_value=?",
            (attribute_name, attribute_value)
        )
        row = cursor.fetchone()
        if row:
            return {
                "attributeName": attribute_name,
                "attributeValue": attribute_value,
                "assignments": json.loads(row[0])
            }
        return None

    def save_assignments(self, doc):
        self.conn.execute(
            "INSERT OR REPLACE INTO sticky_buckets VALUES (?, ?, ?)",
            (doc["attributeName"], doc["attributeValue"], json.dumps(doc["assignments"]))
        )
        self.conn.commit()
```

#### A.6.3 Go SDK 存储实现

```go
// docs/docs/lib/go.mdx:581-593
// 内置 InMemory 实现
service := gb.NewInMemoryStickyBucketService()

// 接口定义
type StickyBucketService interface {
    GetAssignments(attributeName, attributeValue string) (*StickyBucketAssignmentDoc, error)
    SaveAssignments(doc *StickyBucketAssignmentDoc) error
    GetAllAssignments(attributes map[string]string) (StickyBucketAssignments, error)
}

// 内置实现使用 sync.RWMutex 保证线程安全
type InMemoryStickyBucketService struct {
    mu    sync.RWMutex
    docs  map[string]*StickyBucketAssignmentDoc
}
```

#### A.6.4 存储特性对照

| 特性 | JavaScript/Node | Python | Go |
|------|----------------|--------|----|
| **内置 InMemory 实现** | ✅ `InMemoryStickyBucketService` | ✅ `InMemoryStickyBucketService` | ✅ `NewInMemoryStickyBucketService` |
| **内置持久化实现** | ✅ LocalStorage、Browser Cookie、Express Cookie | ❌ 仅提供 SQLite 示例 | ❌ 无（需自行实现） |
| **线程安全** | ❌ 依赖 JS 单线程模型（浏览器） | ❌ 需自行实现 | ✅ 内置 sync.RWMutex |
| **异步接口** | ✅ getAllAssignments async | ❌ 全同步 | ❌ 全同步（Go 并发由 goroutine 处理） |
| **prefix 配置** | ✅ 构造函数参数 | ❌ 无（接口未定义） | ❌ 无（接口未定义） |

### A.7 客户端架构模式对照

#### A.7.1 JavaScript/Node SDK 架构

```typescript
// 两种使用模式
// 1. 传统模式（每个请求创建实例）
const gb = new GrowthBook({
  attributes: { id: "user123" },
  stickyBucketService: new RedisStickyBucketService()
});
gb.load_features();
const result = gb.evalFeature("my-feature");
gb.destroy();

// 2. Node.js 异步客户端（v1.2.0+，类似 Python）
// 参考 Python GrowthBookClient 模式
```

#### A.7.2 Python SDK 架构

```python
# docs/docs/lib/python.mdx:64-125
# 1. 传统同步模式（每个请求创建实例）
gb = GrowthBook(
    attributes={"id": "user123"},
    sticky_bucket_service=MyStickyService()
)
gb.load_features()
result = gb.eval_feature("my-feature")
gb.destroy()

# 2. 异步客户端模式（v1.2.0+，推荐）
client = GrowthBookClient(
    Options(
        api_host="https://cdn.growthbook.io",
        client_key="sdk-abc123",
        sticky_bucket_service=MyStickyService()
    )
)
await client.initialize()

# 每个请求创建 UserContext
user = UserContext(attributes={"id": "user123"})
result = await client.eval_feature("my-feature", user)
```

#### A.7.3 Go SDK 架构

```go
// docs/docs/lib/go.mdx:30-95
// 单例客户端 + 子客户端模式（推荐）
client, err := gb.NewClient(
    context.Background(),
    gb.WithClientKey("sdk-XXXX"),
    gb.WithStickyBucketService(NewInMemoryStickyBucketService()),
    gb.WithSseDataSource(),
)
defer client.Close()

// 每个请求创建子客户端（共享数据，隔离 attributes）
attrs := gb.Attributes{"id": "user123"}
child, err := client.WithAttributes(attrs)
result := child.EvalFeature(context.Background(), "my-feature")
```

#### A.7.4 架构模式差异

| 特性 | JavaScript/Node | Python | Go |
|------|----------------|--------|----|
| **单例客户端模式** | ❌ 传统模式无（Node.js v1.2.0+ 支持） | ✅ GrowthBookClient（v1.2.0+） | ✅ NewClient + WithAttributes |
| **属性隔离方式** | 实例级 attributes | UserContext 参数传递 | WithAttributes 创建子客户端 |
| **数据共享方式** | 全局缓存（feature_repo） | 客户端实例共享 | 子客户端共享父客户端数据 |
| **生命周期管理** | 每个请求 create/destroy | 全局初始化，请求级 UserContext | 全局初始化，请求级 WithAttributes |
| **多用户并发安全** | ❌ 需每个请求独立实例 | ✅ UserContext 隔离 | ✅ 子客户端隔离 |

### A.8 测试用例覆盖对照

所有 SDK 必须通过 `cases.json` 中的 12 个 stickyBucket 测试用例：

| 测试用例名称 | 验证点 |
|-------------|--------|
| use fallbackAttribute when missing hashAttribute | fallback 属性缺失时使用 hashAttribute |
| performs evaluation without sticky bucket | 无粘性分配时的正常计算 |
| evaluates based on stored sticky bucket | 粘性分配命中时跳过过滤 |
| does not consume a sticky bucket not belonging to the user | 不属于用户的粘性分配不被使用 |
| upgrades a sticky bucket doc from a fallbackAttribute to a hashAttribute | fallback → hash 的升级逻辑 |
| favors a sticky bucket doc based on hashAttribute over fallbackAttribute | hashAttribute 优先级高于 fallback |
| resets sticky bucketing when the bucketVersion changes | bucketVersion 变更时重置 |
| stops test enrollment when and existing sticky bucket is blocked by version | minBucketVersion 阻挡逻辑 |
| uses a sticky bucket when sticky bucket version == minBucketVersion == bucketVersion | 版本号匹配时使用粘性分配 |
| skips assignment when sticky bucket version < experiment.minBucketVersion | 版本低于 minBucketVersion 时跳过 |
| resets sticky bucketing when bucket version > experiment.bucketVersion | 版本过高时重置 |
| resets sticky bucketing when bucket version < experiment.bucketVersion | 版本过低时重置 |
| disables sticky bucketing when disabled by experiment | disableStickyBucketing=true 时禁用 |

---

## 附录 B：共性、差异与潜在不一致风险分析

### B.1 共性分析（所有 SDK 必须保持一致）

#### B.1.1 算法层共性（100% 一致性要求）

1. **Hash 算法**：FNV-32a 实现完全一致
   - v1: `hashFnv32a(value + seed) % 1000 / 1000`
   - v2: `hashFnv32a(hashFnv32a(seed + value) + "") % 10000 / 10000`
   - 所有 SDK 必须通过 12 个 hash 测试用例

2. **分桶范围计算**：`getBucketRanges` 算法完全一致
   - coverage 规范化到 [0, 1]
   - weights 校验（长度匹配 + 总和 ≈ 1）
   - 范围计算：`[start, start + coverage * weight]`

3. **粘性分桶键格式**：完全一致
   - 实验键：`${experimentKey}__${bucketVersion}`
   - 属性键：`${attributeName}||${attributeValue}`

4. **命中优先级**：完全一致
   ```
   粘性分配命中 → 跳过 filters/condition/coverage → 直接使用
   未命中 → 按顺序检查 filters → namespace → condition → coverage
   ```

#### B.1.2 数据结构共性（100% 一致性要求）

```typescript
// 跨 SDK 统一的数据结构
interface StickyAssignmentsDocument {
  attributeName: string;           // 如 "id", "deviceId"
  attributeValue: string;          // 如 "user123"
  assignments: Record<string, string>;  // 实验键 → 变体键
}

interface Experiment {
  key: string;
  bucketVersion?: number;          // 默认 0
  minBucketVersion?: number;       // 默认 0
  disableStickyBucketing?: boolean; // 默认 false
  hashAttribute?: string;          // 默认 "id"
  fallbackAttribute?: string;      // 可选
  meta?: Array<{key: string}>;     // 变体键定义
}
```

#### B.1.3 执行流程共性（100% 一致性要求）

```
runExperiment() 执行顺序：
1. 检查 variations 数量（<2 直接返回）
2. 检查 URL querystring 强制
3. 检查 context.forcedVariations
4. 检查 experiment.active
5. 获取 hashValue（hashAttribute → fallbackAttribute）
6. 检查粘性分桶（若启用）→ 命中则跳过后续过滤
7. 检查 filters → namespace → condition → groups
8. 计算 hash 和 bucketRanges → chooseVariation
9. 检查 experiment.force
10. 检查 qaMode
11. 构建 ExperimentResult
12. 保存粘性分配（若启用且命中）
13. 触发 trackingCallback
14. 返回结果
```

### B.2 差异分析（语言/生态特性导致的合理差异）

#### B.2.1 接口设计差异

| 差异点 | 原因分析 | 风险等级 |
|--------|---------|---------|
| **JavaScript/Node 缺少 GetAllAssignments 抽象** | JavaScript 版本较旧（0.32.0 支持），早期设计 | ⚠️ 中 |
| **Python 接口缺少 GetAllAssignments** | Python 接口设计简化 | ⚠️ 中 |
| **Go 接口返回 error** | Go 语言惯用法 | ✅ 低 |
| **JavaScript/Node 有 getKey 工具方法** | 基类提供便利实现 | ✅ 低 |
| **Python/Go 无 prefix 配置** | 接口设计未包含，需用户自行实现 | ⚠️ 中 |

#### B.2.2 存储实现差异

| 差异点 | 原因分析 | 风险等级 |
|--------|---------|---------|
| **JavaScript/Node 内置多种存储实现** | 浏览器/Node.js 生态成熟 | ✅ 低 |
| **Python 仅提供 SQLite 示例** | Python 生态数据库选择多样 | ✅ 低 |
| **Go 仅提供 InMemory 实现** | Go 强调组合优于继承 | ✅ 低 |
| **Go 内置线程安全** | Go goroutine 并发模型需要 | ✅ 低 |
| **Python/JavaScript 无内置线程安全** | 依赖使用模式（每个请求实例） | ⚠️ 中 |

#### B.2.3 客户端架构差异

| 差异点 | 原因分析 | 风险等级 |
|--------|---------|---------|
| **Go 使用子客户端模式** | Go 并发模型 + 性能优化 | ✅ 低 |
| **Python 使用 UserContext 参数** | Python 异步编程模型 | ✅ 低 |
| **JavaScript/Node 传统模式无单例** | 早期设计，Node.js v1.2.0 已支持 | ⚠️ 中 |
| **属性传递方式不同** | 语言/框架习惯差异 | ✅ 低 |

### B.3 潜在不一致风险

#### B.3.1 高风险（必须避免）

**风险 1：Hash 算法实现不一致**
- **场景**：某 SDK 的 FNV-32a 实现有偏差
- **影响**：同一用户在不同端得到不同变体分配
- **验证**：所有 SDK 必须通过 `cases.json` 中的 12 个 hash 测试用例
- **发生概率**：低（有测试用例强制约束）

**风险 2：粘性分桶键格式不一致**
- **场景**：Python SDK 使用 `__` 作为属性键分隔符而非 `||`
- **影响**：跨端粘性分桶失效，用户体验不一致
- **验证**：`cases.json:6541-6624` 升级测试用例
- **发生概率**：低（有测试用例强制约束）

**风险 3：minBucketVersion 阻挡逻辑不一致**
- **场景**：某 SDK 检查 `<= minBucketVersion` 而非 `< minBucketVersion`
- **影响**：该排除的用户未排除，或不该排除的被排除
- **验证**：`cases.json:6688-6849` 多个版本阻挡测试用例
- **发生概率**：低（有测试用例强制约束）

#### B.3.2 中风险（需要关注）

**风险 4：存储 prefix 不一致**
- **场景**：JavaScript SDK 使用 `gbStickyBuckets__` prefix，Python/Go 无 prefix
- **影响**：使用同一 Redis 存储时，键冲突或无法读取
- **解决方案**：
  - 所有 SDK 配置相同的 prefix
  - 或在存储层统一处理
- **发生概率**：中（接口未强制要求）

**风险 5：StickyBucketService 异步/同步差异**
- **场景**：JavaScript `getAllAssignments` 是 async，Python/Go 是 sync
- **影响**：高并发场景下性能表现差异
- **注意**：功能一致性不受影响，仅性能差异
- **发生概率**：中（语言特性导致）

**风险 6：线程安全问题**
- **场景**：Python/JavaScript SDK 在多线程环境下共享实例
- **影响**：粘性分配读写冲突，数据不一致
- **解决方案**：
  - JavaScript：浏览器单线程无问题，Node.js 使用 async 客户端
  - Python：每个请求创建新实例或使用 async 客户端
  - Go：内置线程安全，无需担心
- **发生概率**：中（使用模式不当导致）

**风险 7：fallbackAttribute 升级逻辑差异**
- **场景**：某 SDK 在升级时未同时写入 hashAttribute 和 fallbackAttribute
- **影响**：用户登出后无法保持原变体分配
- **验证**：`cases.json:6488-6552` 升级测试用例
- **发生概率**：低（有测试用例强制约束）

#### B.3.3 低风险（可接受差异）

**风险 8：异常处理方式差异**
- **场景**：Go 返回 error，Python/JavaScript 抛出异常
- **影响**：错误处理代码不同，但功能一致
- **发生概率**：高（语言特性，可接受）

**风险 9：属性传递方式差异**
- **场景**：Go 用 WithAttributes，Python 用 UserContext，JS 用实例属性
- **影响**：调用方式不同，但评估逻辑一致
- **发生概率**：高（设计差异，可接受）

**风险 10：存储实现差异**
- **场景**：各 SDK 存储后端不同
- **影响**：跨端一致需要集中存储（如 Redis）
- **解决方案**：使用相同的存储后端和数据格式
- **发生概率**：中（部署架构选择，可控制）

### B.4 跨 SDK 一致性保障最佳实践

#### B.4.1 开发阶段保障

1. **严格遵循测试用例**：
   - 所有 SDK 必须 100% 通过 `cases.json` 测试用例
   - 重点关注 stickyBucket 部分的 12 个用例
   - 新增特性必须先添加测试用例再实现

2. **使用规范文档作为唯一真理来源**：
   - `docs/docs/lib/build-your-own.mdx` 是所有 SDK 实现的规范
   - 任何与规范不一致的实现都是 bug

3. **代码审查清单**：
   ```
   ▢ Hash 算法是否与规范完全一致？
   ▢ 粘性分桶键格式是否正确？
   ▢ minBucketVersion 检查范围是否正确（0 到 minBucketVersion-1）？
   ▢ 分配合并顺序是否正确（fallback → hashAttribute 覆盖）？
   ▢ 粘性命中时是否跳过了所有过滤？
   ▢ 版本阻挡时是否保存了新分配？
   ```

#### B.4.2 部署阶段保障

1. **统一存储配置**：
   - 所有 SDK 使用相同的 prefix 配置
   - 跨端一致场景使用集中存储（Redis）
   - Cookie 场景使用相同的 cookie name 和 domain

2. **版本兼容性检查**：
   - 确保所有使用的 SDK 版本支持所需特性
   - 参考 `CAPABILITIES.md` 矩阵进行版本选择
   - 避免使用低版本 SDK 不支持的特性

3. **监控告警**：
   - 监控各端变体分配比例一致性
   - 设置异常波动告警（如某端分配比例偏差 > 5%）
   - 监控 `stickyBucketUsed` 指标（应接近 100% 对已有用户）

#### B.4.3 跨端一致场景配置指南

**场景 1：前后端一致（浏览器 + Node.js）**
```
配置要点：
- 前端：BrowserCookieStickyBucketService(prefix="gb_")
- 后端：ExpressCookieStickyBucketService(prefix="gb_")
- Cookie：domain=.yourdomain.com，path=/
- 一致性：✅ 100%（同一浏览器请求）
```

**场景 2：微服务一致（Python + Go + Node.js）**
```
配置要点：
- 所有服务：RedisStickyBucketService（同一 Redis 实例）
- 所有服务：相同的 prefix 配置
- 一致性：✅ 100%（集中存储）
- 注意：序列化/反序列化格式一致（JSON）
```

**场景 3：跨设备一致（Web + App）**
```
配置要点：
- hashAttribute: "userId"
- fallbackAttribute: "deviceId" / "cookieId"
- 存储：集中式 Redis
- 一致性：✅ 登录后一致，未登录按设备
- 注意：登录后触发"升级"逻辑
```

---

## 附录 C：核心一致性验证测试用例速查

### C.1 Hash 算法验证（12 个用例）

| seed | value | version | expected |
|------|-------|---------|----------|
| "" | "" | 1 | 0.361 |
| "test" | "abc" | 1 | 0.619 |
| "test" | "abc" | 2 | 0.5069 |
| "foo" | "bar" | 2 | 0.6281 |
| ... | ... | ... | ... |

### C.2 粘性分桶关键验证点

| 测试场景 | 预期结果 | 用例位置 |
|---------|---------|---------|
| 无 hashAttribute，使用 fallback | fallback 生效，stickyBucketUsed=false | cases.json:6273-6309 |
| 有粘性分配，跳过过滤 | 使用粘性分配，stickyBucketUsed=true | cases.json:6366-6424 |
| fallback → hashAttribute 升级 | 同时写入两个文档 | cases.json:6488-6552 |
| hashAttribute 优先级高于 fallback | 使用 hashAttribute 的分配 | cases.json:6554-6624 |
| bucketVersion 变更重置 | 重新计算，stickyBucketUsed=false | cases.json:6627-6686 |
| minBucketVersion 阻挡 | 返回 null，不保存新分配 | cases.json:6688-6736 |
| disableStickyBucketing=true | 忽略粘性分配 | cases.json:6980-7037 |

### C.3 跨 SDK 版本兼容性检查

| 特性 | JavaScript 最低 | Python 最低 | Go 最低 |
|------|---------------|------------|---------|
| Sticky Bucketing | 0.32.0 | 1.1.0 | 0.2.3 |
| Hash v2 | 0.23.0 | 1.0.0 | 0.1.4 |
| Prerequisites | 0.34.0 | 1.1.0 | 0.2.0 |
| Saved Groups | 1.1.0 | 1.2.1 | 0.2.0 |
| Async Client | -（Node.js 1.2.0+） | 1.2.0 | -（原生支持） |
