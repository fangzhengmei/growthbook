# GrowthBook Environment SDK Payload 缓存机制分析

## 1. 整体架构概览

SDK Payload 缓存的核心目标是：当 SDK 客户端请求 feature flags 数据时，避免每次都执行昂贵的全量计算（遍历所有 features、experiments、saved groups、holdouts 等），而是将生成好的 JSON payload 持久化到 MongoDB，后续请求直接命中缓存。

缓存体系分三层：

```
┌──────────────────────────────────────────────────────┐
│  Layer 3: CDN (Fastly)                               │
│  HTTP Cache-Control: stale-while-revalidate          │
│  Surrogate-Key: {orgId}_{envId}                      │
├──────────────────────────────────────────────────────┤
│  Layer 2: MongoDB (sdkcache collection)              │
│  _id = cacheKey, contents = JSON(payload)            │
│  schemaVersion 版本号                                │
├──────────────────────────────────────────────────────┤
│  Layer 1: In-Memory (MemoryCache, 30s TTL)           │
│  仅用于 API Key 查找等轻量场景                        │
└──────────────────────────────────────────────────────┘
```

### 核心文件索引

| 文件 | 职责 |
|------|------|
| `packages/back-end/src/models/SdkConnectionCacheModel.ts` | 缓存 Model：MongoDB 读写、缓存键格式化、存储后端选择 |
| `packages/back-end/src/services/cache.ts` | 通用内存缓存（MemoryCache / LruCache），非 payload 专用 |
| `packages/back-end/src/controllers/features.ts` | 读路径入口：`getFeatureDefinitionsWithCache`（缓存优先 + JIT 回源写回） |
| `packages/back-end/src/services/features.ts` | 写路径入口：`refreshSDKPayloadCache` / `queueSDKPayloadRefresh` + payload 生成 |
| `packages/back-end/src/util/features.ts` | Diff 算法：`getSDKPayloadKeysByDiff` / `getAffectedSDKPayloadKeys` |
| `packages/back-end/src/util/cdn.util.ts` | CDN surrogate key 生成与 Fastly purging |
| `packages/back-end/src/jobs/updateAllJobs.ts` | 刷新后触发：webhook / proxy / CDN purge |

---

## 2. 缓存键计算（Cache Key Generation）

缓存键决定了每条缓存记录在 MongoDB `sdkcache` 集合中的唯一标识。系统存在两类 key 来源：

### 2.1 SDK Connection Key（新式）

```
key = connection.key   // 例如 "sdk-abc123def456"
```

SDK Connection 创建时通过 `generateSigningKey("sdk-", 12)` 生成唯一 key（`SdkConnectionModel.ts:221`）。这个 key 直接作为缓存键使用，天然具备：

- **唯一性**：每个 Connection 一个 key
- **环境隔离**：一个 Connection 只绑定一个 `environment`，因此 key 本身隐含了环境信息
- **项目隔离**：Connection 的 `projects` 字段在 payload 生成时做过滤，但**不影响缓存键**

> 缓存键 **不包含** environment / project 信息。环境与项目的隔离完全通过 Connection 的属性在 payload 生成时实现，而非通过键的复合编码。

### 2.2 Legacy API Key（旧式）

```
key = "legacy:{apiKey}:{environment}:{project}"
```

格式化函数 `formatLegacyCacheKey`（`SdkConnectionCacheModel.ts:76-88`）：

```ts
export function formatLegacyCacheKey({
  apiKey,
  environment,
  project,
}: {
  apiKey: string;
  environment?: string;
  project: string;
}): string {
  const env = environment || "production";
  const parts = [LEGACY_KEY_PREFIX + apiKey, env, project];
  return parts.join(":");
}
```

旧式 key 的特点：

- **三段式编码**：`legacy:` + apiKey + `:` + environment + `:` + project
- **environment 默认值**：未指定时默认 `"production"`
- **project 可为空**：空字符串表示全局（所有项目）
- **正则前缀匹配删除**：`deleteAllLegacyCacheEntries()` 用 `/^legacy:/` 正则批量删除所有旧式缓存

### 2.3 schemaVersion 版本号

MongoDB 文档结构（`sdkConnectionCacheValidator`，`shared/src/validators/sdk-connection-cache.ts`）：

```ts
{
  contents: string,       // JSON.stringify(payload)
  schemaVersion: number,  // 当前 = 1
  audit?: {
    dateUpdated: Date,
    event: string,
    model: string,
    id?: string,
    stack: string,
    connection: Record<string, unknown>,
  }
}
```

`LATEST_SDK_PAYLOAD_SCHEMA_VERSION = 1`（`SdkConnectionCacheModel.ts:8`）。

- **读取**：`getById(id)` 在查询时附带 `schemaVersion: LATEST_SDK_PAYLOAD_SCHEMA_VERSION` 条件，版本不匹配的缓存自然不会被命中（等效于 cache miss）
- **写入**：`upsert(id, contents)` 先按 `id` 查找（忽略版本），找到则更新（含新版本号），找不到则新建。这意味着 schema 升级时旧版本缓存会在下一次 upsert 时自动替换

---

## 3. 缓存过期触发机制（Expiration & Refresh）

GrowthBook 的 SDK Payload 缓存 **没有 TTL 式的被动过期**，而是采用 **事件驱动的主动失效**（event-driven invalidation）。缓存记录本身没有 `expiresAt` 字段。

### 3.1 三层过期机制

| 层级 | 过期方式 | 时间窗口 |
|------|---------|---------|
| CDN (Fastly) | `Cache-Control: public, max-age=30, stale-while-revalidate=3600, stale-if-error=36000` | 新鲜 30s，允许陈旧 1h，源站故障容忍 10h |
| MongoDB | 事件驱动 upsert 覆盖 | 无 TTL，直到下次刷新 |
| In-Memory | `MemoryCache` TTL=30s | 30 秒后自动失效 |

CDN 默认值来自环境变量（`secrets.ts:169-183`）：

```
CACHE_CONTROL_MAX_AGE            = 30     (秒)
CACHE_CONTROL_STALE_WHILE_REVALIDATE = 3600   (秒, 1小时)
CACHE_CONTROL_STALE_IF_ERROR     = 36000  (秒, 10小时)
```

### 3.2 主动刷新触发源

所有数据变更最终汇聚到 `queueSDKPayloadRefresh()`（`services/features.ts:607-621`），它异步调用 `refreshSDKPayloadCache()`。触发源包括：

| 触发源 | 文件 | 传入的 payloadKeys |
|--------|------|-------------------|
| Feature 创建/删除 | `FeatureModel.ts:873,899` | `getAffectedSDKPayloadKeys(feature, envIds)` |
| Feature 更新 | `FeatureModel.ts:927` | `getSDKPayloadKeysByDiff(old, new, envIds)` — 按 diff 精确计算受影响的环境+项目 |
| Feature 发布/回滚 | `controllers/features.ts` postFeaturePublish | 通过 `onFeatureUpdate` 间接触发 |
| Experiment 状态变更 | `ExperimentModel.ts:2013,2044` | 受影响 experiment 的环境+项目 |
| SDK Connection 创建 | `SdkConnectionModel.ts:281` | `payloadKeys: []`, `sdkConnections: [newConn]` |
| SDK Connection 更新 | `SdkConnectionModel.ts:412` | 仅当 `keysRequiringProxyUpdate` 中字段变化时 |
| Environment 更新 | `environment.controller.ts:211` | `payloadKeys: []`, `sdkConnections: affectedConnections` |
| Saved Group 变更 | `savedGroups.ts:24` | 受影响的 payloadKeys |
| Holdout 状态变更 | `holdout.controller.ts` 多处 | 受影响的环境 |
| Safe Rollout 变更 | `SafeRolloutModel.ts:85` | 受影响的 payloadKeys |
| Visual Changeset 更新 | `VisualChangesetModel.ts:428,464,495` | 受影响的 payloadKeys |
| URL Redirect 更新 | `UrlRedirectModel.ts:127,158` | 受影响的 payloadKeys |
| Custom Field 变更 | `CustomFieldModel.ts:275,308` | 受影响的 payloadKeys |
| Project 变更 | `ProjectModel.ts:143` | 受影响的 payloadKeys |
| 定时 Holdout 状态检查 | `updateHoldoutStatus.ts:138,171,201` | 受影响的 payloadKeys |

### 3.3 Diff 算法：精确计算受影响的 Payload Keys

`getSDKPayloadKeysByDiff`（`util/features.ts:283-378`）实现了细粒度的 diff，避免全量刷新：

1. **全局字段变更** → 所有已启用环境都受影响：
   - `archived`, `defaultValue`, `project`, `valueType`, `nextScheduledUpdate`, `holdout`, `prerequisites`

2. **规则级别变更** → 仅规则所覆盖的环境受影响：
   - 按 rule id 做 diff，只对变更的规则计算其环境覆盖范围
   - 规则重排序也算变更（`oldIdOrder !== newIdOrder`）

3. **环境设置变更** → 仅该环境受影响：
   - `environmentSettings[env]` 前后不等

4. **项目变更** → 旧项目 + 新项目都需刷新

输出 `SDKPayloadKey[]`，结构为 `{ environment: string, project: string }`。

### 3.4 刷新后的级联动作

`refreshSDKPayloadCache` 完成后调用 `triggerWebhookJobs`（`jobs/updateAllJobs.ts`）：

```
triggerWebhookJobs
├── queueWebhooksByConnections()    → SDK Webhook 推送
├── fireGlobalSdkWebhooks()         → 全局 Webhook 推送
├── queueProxyUpdate()              → 代理缓存更新
├── queueLegacySdkWebhooks()        → 旧式 Webhook 推送
└── purgeCDNCache()                 → Fastly Surrogate-Key 批量清除
```

CDN purge 的 surrogate key 格式为 `{orgId}_{envId}`（`cdn.util.ts:5-15`），由 `getSurrogateKeysFromEnvironments` 生成。这确保同一环境下的所有 SDK Connection 共享一个 CDN 缓存组，环境级变更可一次清除。

---

## 4. 缓存更新与回源路径（Cache Update & Fallback）

### 4.1 读路径：Cache-First + JIT 回源写回

入口：`getFeatureDefinitionsWithCache`（`controllers/features.ts:415-501`）

```
SDK 请求 → getSdkPayload / getFeaturesPublic
          ↓
  getPayloadParamsFromApiKey(key, req)
     ├── key 匹配 /^sdk-/ → findSDKConnectionByKey → SDK Connection 属性
     └── 否则 → dangerousLookupOrganizationByApiKey → 合成 legacy cache key
          ↓
  getFeatureDefinitionsWithCache({context, params})
     │
     ├─ storageLocation === "none" → 跳过缓存，直接生成
     │
     ├─ 尝试: sdkConnectionCache.getById(params.key)
     │   ├─ 命中 + JSON.parse 成功 → 返回缓存
     │   └─ 命中但 JSON.parse 失败 → logger.warn, 视为 miss
     │
     └─ 缓存未命中 / 不可用
         │
         ├─ 计算capabilities (legacy硬编码["bucketingV2"] / SDK版本推导)
         ├─ filterProjectsByEnvironmentWithNull
         ├─ getFeatureDefinitions() ← 完整生成 payload
         │
         └─ fire-and-forget: sdkConnectionCache.upsert(key, JSON.stringify(defs))
            ↑ 回写缓存，不阻塞响应
```

关键设计决策：

- **fire-and-forget 写回**：JIT 生成后立即返回给 SDK，缓存的写回是异步的（`.catch()` 吞掉错误），不影响请求延迟
- **缓存损坏容错**：JSON.parse 失败时 warn 并回源，不会返回损坏数据
- **缓存可关闭**：环境变量 `SDK_PAYLOAD_CACHE=none` 可完全禁用 MongoDB 缓存层

### 4.2 写路径：批量刷新

入口：`refreshSDKPayloadCache`（`services/features.ts:623-826`）

```
queueSDKPayloadRefresh (fire-and-forget 封装)
  ↓
refreshSDKPayloadCache
  │
  ├─ 切换到 background context (完整 org 读取权限)
  ├─ 过滤无效环境 (allowedEnvs)
  ├─ 过滤 skipRefreshForProject
  ├─ deleteAllLegacyCacheEntries() ← 清除所有旧式缓存
  │
  ├─ 一次性加载 rawData:
  │   ├─ getAllFeatures()
  │   ├─ getAllPayloadExperiments()
  │   ├─ getAllPayloadSafeRollouts()
  │   ├─ getAllSavedGroups() + getSavedGroupMap()
  │   ├─ getAllVisualExperiments()
  │   └─ getAllURLRedirectExperiments()
  │
  ├─ 按 environment 加载 holdoutsMap
  │
  ├─ 两种连接选取路径:
  │   ├─ payloadKeys.length > 0 → findSDKConnectionsByOrganization → 全量连接
  │   │   然后 isSDKConnectionAffectedByPayloadKey 过滤
  │   └─ payloadKeys.length === 0 → 直接用 sdkConnectionsToUpdate
  │
  └─ 并发构建 + 写入 (4 并发):
      ├─ buildSDKPayloadForConnection() ← 共享 rawData
      └─ sdkConnectionCache.upsert(connection.key, JSON.stringify(contents))
```

两条路径的选择：

| 路径 | 触发条件 | 连接选取 |
|------|---------|---------|
| Bulk（批量） | 传入了 `payloadKeys` | 加载 org 全量连接，按 `isSDKConnectionAffectedByPayloadKey` 过滤 |
| Targeted（定向） | 只传入 `sdkConnections` | 直接使用传入的连接列表，不查数据库 |

`isSDKConnectionAffectedByPayloadKey`（`services/features.ts:581-603`）的匹配逻辑：

```
connection.environment !== payloadKey.environment → 不受影响
treatEmptyProjectAsGlobal && !payloadKey.project → 全局影响，匹配
!connection.projects.length → 连接无项目过滤（全局），匹配
connection.projects.includes(payloadKey.project) → 连接包含该项目，匹配
其他 → 不受影响
```

### 4.3 `SDK_PAYLOAD_CACHE` 环境变量

```ts
export function getSDKPayloadCacheLocation(): "mongo" | "none" {
  const loc = process.env.SDK_PAYLOAD_CACHE;
  if (loc === "none") return "none";
  return "mongo";
}
```

- `"none"`：完全跳过 MongoDB 缓存读写，每次请求都实时生成
- 默认（`"mongo"`）：启用 MongoDB 缓存

> 注：代码中有 TODO 注释 `"add support for S3 and GCS storage backends"`，但目前仅支持 MongoDB。

---

## 5. 多环境隔离实现（Multi-Environment Isolation）

### 5.1 隔离模型

GrowthBook 的环境隔离是 **Connection 粒度**，而非 Cache Key 粒度：

```
Organization
├── Environment: production
│   ├── SDK Connection A (env=production, projects=[p1])
│   │   └── Cache Key: "sdk-xxx" → payload 含 production + p1 的 features
│   └── SDK Connection B (env=production, projects=[])
│       └── Cache Key: "sdk-yyy" → payload 含 production + 所有 projects 的 features
└── Environment: staging
    └── SDK Connection C (env=staging, projects=[])
        └── Cache Key: "sdk-zzz" → payload 含 staging + 所有 projects 的 features
```

**关键点**：缓存键本身就是 Connection key，而一个 Connection 天然只属于一个 environment。因此不同环境的缓存记录天然隔离在不同的 key 空间中。

### 5.2 刷新时的环境过滤

`refreshSDKPayloadCache` 中的环境过滤（`services/features.ts:651-658`）：

```ts
const allowedEnvs = new Set(getEnvironmentIdsFromOrg(context.org));
payloadKeys = payloadKeys.filter((k) => allowedEnvs.has(k.environment));
```

已删除的环境不会触发刷新，避免无效计算。

### 5.3 CDN 层环境级清除

Surrogate Key 格式 `{orgId}_{envId}`（`cdn.util.ts:5-15`）：

```ts
export function getSurrogateKeysFromEnvironments(
  orgId: string,
  environments: string[],
): string[] {
  return environments.map((k) => {
    const key = `${orgId}_${k}`;
    return key.replace(/[^a-zA-Z0-9_-]/g, "");
  });
}
```

这确保 CDN 缓存按环境分组，清除 `org1_staging` 不会影响 `org1_production` 的 CDN 缓存。

### 5.4 Legacy API Key 的环境隔离

旧式 API key 的缓存键编码了环境信息：

```
legacy:{apiKey}:production:p1
legacy:{apiKey}:staging:p1
```

但旧式缓存无法精确跟踪哪些 environment 受影响，因此 `refreshSDKPayloadCache` 每次都调用 `deleteAllLegacyCacheEntries()` **删除该 org 下所有旧式缓存**，下次请求时 JIT 回源。

### 5.5 payload 生成时的环境过滤

`buildSDKPayloadForConnection`（`services/features.ts:1052-1224`）中：

- `environment` 参数传给 `generateFeaturesPayload`，仅提取该环境下 enabled 的 feature rules
- `filterProjectsByEnvironmentWithNull` 根据 environment 的 project 配置进一步过滤
- `holdoutsMap` 按 environment 独立加载（`getAllPayloadHoldouts(environment)`）

---

## 6. 完整数据流图

```
                          ┌─────────────────────┐
                          │   Feature/Experiment │
                          │   /SavedGroup 等变更  │
                          └────────┬────────────┘
                                   │
                      queueSDKPayloadRefresh()
                                   │
                    ┌──────────────┴──────────────┐
                    │  refreshSDKPayloadCache()    │
                    │  1. 过滤无效 env             │
                    │  2. deleteAllLegacyCache     │
                    │  3. 批量加载 rawData          │
                    │  4. 匹配受影响的 Connection   │
                    │  5. buildSDKPayload 4并发     │
                    │  6. upsert → MongoDB         │
                    └──────────────┬──────────────┘
                                   │
                    triggerWebhookJobs()
                    ┌──────────────┼──────────────┐
                    │              │              │
              Webhook 推送    Proxy 更新    CDN Purge
              (按Connection)  (按Connection) (按envId)


          ┌────────────────────────────────────────────┐
          │            SDK 读取路径                      │
          │                                            │
          │  GET /api/features/:key                    │
          │       │                                    │
          │  getPayloadParamsFromApiKey()              │
          │       │                                    │
          │  getFeatureDefinitionsWithCache()          │
          │       │                                    │
          │  ┌────┴────┐                               │
          │  │ MongoDB │ ──命中──→ JSON.parse → 返回   │
          │  │ Cache   │ ──未命中─→ getFeatureDefs()  │
          │  └─────────┘         │    ↓ 生成 payload  │
          │                      │    ↓ 返回给 SDK     │
          │                      └──→ upsert (异步写回)│
          │                                            │
          │  Cache-Control: max-age=30                 │
          │  Surrogate-Key: {orgId}_{envId}            │
          └────────────────────────────────────────────┘
```

---

## 7. 设计特点与注意事项

### 7.1 无 TTL 的主动失效策略

MongoDB 缓存层没有 TTL 索引，完全依赖事件驱动的 upsert 覆盖。这意味着：

- **优点**：不会出现「缓存存在但已过时」的中间态——任何数据变更都立即触发重算
- **风险**：如果 `queueSDKPayloadRefresh` 的 fire-and-forget 丢失（进程崩溃），缓存可能长期不更新。CDN 的 `stale-while-revalidate=3600` 提供了 1 小时的兜底窗口

### 7.2 Legacy 缓存的全量清除

旧式 API key 的缓存无法精确追踪，每次刷新都执行 `deleteAllLegacyCacheEntries()`（正则匹配 `^legacy:`）。这在旧 key 较多时可能有性能影响，但保证了正确性。

### 7.3 缓存键不含 schemaVersion

读取时通过 `getById` 的查询条件附带 `schemaVersion`，写入时 upsert 自动升级版本。这种设计实现了 schema 版本升级时的零停机迁移——旧版本缓存自然 miss，JIT 回源写入新版本。

### 7.4 并发控制

`refreshSDKPayloadCache` 使用 `promiseAllChunks(promises, 4)` 限制 4 并发写入 MongoDB，避免大规模组织（数百个 SDK Connection）刷新时压垮数据库。

### 7.5 共享 rawData 的不可变性

批量刷新时，所有 Connection 共享同一份 `rawData`（features、experimentMap 等），`buildSDKPayloadForConnection` 内部通过 `cloneDeep` 保证不修改共享数据。测试（`sdk-payload-lifecycle.test.ts:311-363`）专门验证了这一点。
