# GrowthBook Environment SDK Payload 缓存机制分析

## 1. 整体架构概览

SDK Payload 缓存的核心目标是：当 SDK 客户端请求 feature flags 数据时，避免每次都执行昂贵的全量计算（遍历所有 features、experiments、saved groups、holdouts 等），而是将生成好的 JSON payload 持久化到 MongoDB，后续请求直接命中缓存。

缓存体系分**两层**（注意：没有独立的内存缓存层用于 payload）：

```
┌──────────────────────────────────────────────────────┐
│  Layer 2: CDN (Fastly)                               │
│  HTTP Cache-Control: stale-while-revalidate          │
│  Surrogate-Key: {orgId}_{envId}, {connection.key}    │
├──────────────────────────────────────────────────────┤
│  Layer 1: MongoDB (sdkcache collection)              │
│  _id = cacheKey, contents = JSON(payload)            │
│  schemaVersion 版本号                                │
└──────────────────────────────────────────────────────┘
```

> **重要修正**：`MemoryCache`/`LruCache`（`services/cache.ts`）**并未用于 SDK payload 缓存**，仅在 `OpenIdAuthConnection.ts` 中用于 SSO 连接缓存。SDK payload 读写只经过 MongoDB + CDN 两层。

### 核心文件索引

| 文件 | 职责 |
|------|------|
| `packages/back-end/src/models/SdkConnectionCacheModel.ts` | 缓存 Model：MongoDB 读写、缓存键格式化、存储后端选择 |
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

### 3.1 两层过期机制

| 层级 | 过期方式 | 时间窗口 |
|------|---------|---------|
| CDN (Fastly) | `Cache-Control: public, max-age=30, stale-while-revalidate=3600, stale-if-error=36000` | 新鲜 30s，允许陈旧 1h，源站故障容忍 10h |
| MongoDB | 事件驱动 upsert 覆盖 | 无 TTL，直到下次刷新 |

> **重要修正**：不存在 In-Memory 层的 TTL 过期，MemoryCache 未用于 payload 缓存。

CDN 默认值来自环境变量（`secrets.ts:169-183`）：

```
CACHE_CONTROL_MAX_AGE            = 30     (秒)
CACHE_CONTROL_STALE_WHILE_REVALIDATE = 3600   (秒, 1小时)
CACHE_CONTROL_STALE_IF_ERROR     = 36000  (秒, 10小时)
```

### 3.2 主动刷新触发源

所有数据变更最终汇聚到 `queueSDKPayloadRefresh()`（`services/features.ts:607-621`），它异步调用 `refreshSDKPayloadCache()`。触发源包括：

| 触发源 | 文件 | 传入的 payloadKeys | 传入的 sdkConnections |
|--------|------|-------------------|----------------------|
| Feature 创建/删除 | `FeatureModel.ts:873,899` | `getAffectedSDKPayloadKeys(feature, envIds)` | - |
| Feature 更新 | `FeatureModel.ts:927` | `getSDKPayloadKeysByDiff(old, new, envIds)` | - |
| Feature 发布/回滚 | `controllers/features.ts` postFeaturePublish | 通过 `onFeatureUpdate` 间接触发 | - |
| Experiment 状态变更 | `ExperimentModel.ts:2013,2044` | 受影响 experiment 的环境+项目 | - |
| SDK Connection 创建 | `SdkConnectionModel.ts:281` | `[]` (空) | `[newConn]` |
| SDK Connection 更新 | `SdkConnectionModel.ts:412` | `[]` (空) | 受影响的 connections |
| Environment 更新 | `environment.controller.ts:211` | `[]` (空) | 受影响的 connections |
| Saved Group 变更 | `savedGroups.ts:24` | 受影响的 payloadKeys | - |
| Holdout 状态变更 | `holdout.controller.ts` 多处 | 受影响的环境 | - |
| Safe Rollout 变更 | `SafeRolloutModel.ts:85` | 受影响的 payloadKeys | - |
| Visual Changeset 更新 | `VisualChangesetModel.ts:428,464,495` | 受影响的 payloadKeys | - |
| URL Redirect 更新 | `UrlRedirectModel.ts:127,158` | 受影响的 payloadKeys | - |
| Custom Field 变更 | `CustomFieldModel.ts:275,308` | 受影响的 payloadKeys | - |
| Project 变更 | `ProjectModel.ts:143` | 受影响的 payloadKeys | - |
| 定时 Holdout 状态检查 | `updateHoldoutStatus.ts:138,171,201` | 受影响的 payloadKeys | - |

> **关键点**：SDK Connection / Environment 变更时传入 `payloadKeys = []`，这会触发不同的 CDN purge 策略（见第 6 节）。

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

CDN purge 的 surrogate key 策略分两种情况（见第 6 节详细分析）。

---

## 4. 缓存更新与回源路径（Cache Update & Fallback）

### 4.1 读路径：Cache-First + JIT 回源写回

入口：`getFeatureDefinitionsWithCache`（`controllers/features.ts:415-501`）

> **重要修正**：此函数被两个端点调用，但只有 `/api/features/:key` 公共端点设置 CDN 缓存头。`/api/v1/sdk-payload/:key` 端点不设置任何缓存头。

```
SDK 请求 → getSdkPayload / getFeaturesPublic / getEvaluatedFeaturesPublic
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

### 4.2 `getFeatureDefinitionsWithCache` 的调用方清单

此函数是 SDK payload 生成的核心入口，被以下 7 处调用，分三类场景：

#### 场景 1：HTTP 端点（3 处）

| 调用方 | 文件 | 端点 | 说明 |
|--------|------|------|------|
| `getFeaturesPublic` | `controllers/features.ts:525` | `GET /api/features/:key` | 公共 SDK 端点，带 CDN 缓存 |
| `getEvaluatedFeaturesPublic` | `controllers/features.ts:605` | `POST /api/eval/:key` | 远程评估端点，仅自托管 |
| `getSdkPayload` | `api/sdk-payload/getSdkPayload.ts:37` | `GET /api/v1/sdk-payload/:key` | API 端点，需 Secret Key 鉴权 |

#### 场景 2：内部 API（1 处）

| 调用方 | 文件 | 说明 |
|--------|------|------|
| `listFeatures` handler | `api/features/listFeatures.ts:123` | 内部 API，用于列出 features |

#### 场景 3：后台 Jobs（3 处）

| 调用方 | 文件 | 触发时机 |
|--------|------|---------|
| `queueProxyUpdate` | `jobs/proxyUpdate.ts:85` | SDK payload 刷新后，更新代理缓存 |
| `queueWebhooksByConnections` | `jobs/sdkWebhooks.ts:336,363` | SDK payload 刷新后，推送 Webhook |
| `queueLegacySdkWebhooks` | `jobs/webhooks.ts:43` | SDK payload 刷新后，推送旧式 Webhook |

> **注意**：后台 Jobs 调用时，`params` 是直接构造的 `SDKPayloadParams` 对象，不走 `getPayloadParamsFromApiKey` 路径。

### 4.3 写路径：批量刷新

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

| 路径 | 触发条件 | 连接选取 | CDN purge 策略 |
|------|---------|---------|---------------|
| Bulk（批量） | 传入了 `payloadKeys` | 加载 org 全量连接，按 `isSDKConnectionAffectedByPayloadKey` 过滤 | 按 environment 批量 purge |
| Targeted（定向） | `payloadKeys = []` + 传入 `sdkConnections` | 直接使用传入的连接列表，不查数据库 | 按 connection.key 逐个 purge |

`isSDKConnectionAffectedByPayloadKey`（`services/features.ts:581-603`）的匹配逻辑：

```
connection.environment !== payloadKey.environment → 不受影响
treatEmptyProjectAsGlobal && !payloadKey.project → 全局影响，匹配
!connection.projects.length → 连接无项目过滤（全局），匹配
connection.projects.includes(payloadKey.project) → 连接包含该项目，匹配
其他 → 不受影响
```

### 4.4 `SDK_PAYLOAD_CACHE` 环境变量

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

## 5. remoteEvalEnabled 下的读路径分叉

SDK Connection 的 `remoteEvalEnabled` 字段控制着三个公开端点的准入逻辑，形成不同的回源路径：

### 5.1 三个公开端点的行为差异

| 端点 | 方法 | 完整路由 | 处理函数 | 鉴权方式 | CDN 缓存头 | remoteEval=true 时行为 | remoteEval=false 时行为 |
|------|------|---------|---------|---------|-----------|-----------------------|------------------------|
| Features API | GET | `/api/features/:key` | `getFeaturesPublic` | URL path 中的 SDK key | ✅ 设置 | ❌ 抛出错误："Remote evaluation required for this connection" | ✅ 正常返回 payload |
| Remote Eval API | POST | `/api/eval/:key` (仅自托管) | `getEvaluatedFeaturesPublic` | URL path 中的 SDK key | ❌ `no-store` | ✅ 实时计算 evaluated features | ❌ 抛出错误："Remote evaluation disabled for this connection" |
| SDK Payload API | GET | `/api/v1/sdk-payload/:key` | `getSdkPayload` | Authorization header Secret API Key | ❌ 不设置 | ✅ **不检查**，正常返回 payload | ✅ 正常返回 payload |

> **重要修正**：`/api/v1/sdk-payload/:key` 端点**不设置** Cache-Control 或 Surrogate-Key 响应头，也**不支持** SDK Connection key 鉴权。它需要 Secret API Key 通过 Authorization header 鉴权。

### 5.2 路径分叉逻辑详解

#### 路径 A：`GET /api/features/:key` → `getFeaturesPublic`

```ts
// controllers/features.ts:519-523
if (params.remoteEvalEnabled) {
  throw new UnrecoverableApiError(
    "Remote evaluation required for this connection",
  );
}
```

- **鉴权**：通过 URL path 中的 `key` 参数（SDK Connection key），无 Authorization header 要求
- **路由位置**：`app.ts:297-304` 直接注册，**不经过** `authenticateApiRequestMiddleware`
- **缓存头**：完整设置
  ```
  Cache-control: public, max-age=30, stale-while-revalidate=3600, stale-if-error=36000
  Surrogate-Key: {orgId} {connection.key} {orgId}_{envId}
  ```
- 当 Connection 开启 `remoteEvalEnabled` 时，拒绝通过此端点获取原始 payload
- 响应头设置 `x-unrecoverable: 1`，CDN 可据此对 400 错误应用缓存规则

#### 路径 B：`POST /api/eval/:key` → `getEvaluatedFeaturesPublic`

```ts
// controllers/features.ts:588-592
if (!params.remoteEvalEnabled) {
  throw new UnrecoverableApiError(
    "Remote evaluation disabled for this connection",
  );
}
```

- **鉴权**：通过 URL path 中的 `key` 参数（SDK Connection key）
- **路由位置**：`app.ts:318-325` 直接注册，**不经过** `authenticateApiRequestMiddleware`
- **仅自托管版本可用**（`app.ts:315`：`if (!IS_CLOUD)`）
- 云端环境必须使用独立的远程评估基础设施
- 此端点完全绕过 CDN 缓存：`Cache-control: no-store`
- 虽然内部仍调用 `getFeatureDefinitionsWithCache` 复用 MongoDB 缓存，但最终结果是实时计算的 evaluated features

#### 路径 C：`GET /api/v1/sdk-payload/:key` → `getSdkPayload`

```ts
// api/sdk-payload/getSdkPayload.ts:28-46
// 无 remoteEvalEnabled 检查
// 无 Cache-Control / Surrogate-Key 设置
const params = await getPayloadParamsFromApiKey(key, req);
const defs = await getFeatureDefinitionsWithCache({
  context: req.context,
  params,
});
```

- **鉴权**：需要 `Authorization: Bearer <Secret API Key>` header，**不能用 SDK Connection key**
- **路由位置**：通过 `api.router.ts` 注册，**经过** `authenticateApiRequestMiddleware`
  - 中间件要求 Secret API Key，若传入 SDK Endpoint key（`secret=false`）会抛出错误："Must use a Secret API Key for this request, SDK Endpoint key given instead."
- **完整路径**：`/api/v1/sdk-payload/:key`（apiRouter 挂载在 `/api`，路由自动加 `/v1` 前缀）
- **不设置**任何 CDN 缓存头
- **不检查** `remoteEvalEnabled` 标志，始终返回原始 payload
- 此端点走 API Router，通过 `createApiRequestHandler` 封装

##### `/api/v1/sdk-payload/:key` 中两个 key 的角色分工

此端点涉及**两个独立的 key**，扮演完全不同的角色：

| Key 来源 | 位置 | 处理模块 | 角色 | 要求 |
|---------|------|---------|------|------|
| Secret API Key | `Authorization` header | `authenticateApiRequestMiddleware` | **鉴权**：验证请求者身份与权限 | 必须 `secret=true` |
| Payload Key | URL path `:key` 参数 | `getPayloadParamsFromApiKey` | **内容定位**：指定要获取哪个 SDK Connection 的配置 | SDK Connection key (`sdk-*`) 或 Legacy Publishable key |

**执行流程**：

```
请求到达 /api/v1/sdk-payload/sdk-abc123
         ↓
  authenticateApiRequestMiddleware
  (检查 Authorization: Bearer secret_xxx)
         ↓ 鉴权通过，建立 req.context.org
  getPayloadParamsFromApiKey("sdk-abc123", req)
  (按 URL path 查找 SDK Connection 配置)
         ↓
  getFeatureDefinitionsWithCache()
  (使用 Connection 的 key 查 MongoDB 缓存)
```

**关键要点**：

1. **两个 key 相互独立**：Authorization header 的 Secret API Key 用于"证明你有权访问 API"，URL path 的 key 用于"指定你要哪个配置"。一个 Secret Key 可以访问该组织下任意多个 SDK Connection 的 payload。

2. **跨组织访问控制**：中间件建立的 `req.context.org` 限定了可访问的组织范围，URL path 中的 SDK Connection 必须属于该组织（由 `findSDKConnectionByKey` 内部保证）。

3. **Legacy key 特殊路径**：如果 URL path key 不匹配 `/^sdk-/`，则走 Legacy 路径：
   - 调用 `dangerousLookupOrganizationByApiKey(key)`
   - 要求该 key 必须是 Publishable key（`secret=false`），如果是 Secret key 会报错："Must use a Publishable API key to get feature definitions"

### 5.3 两条 SDK 读路径的鉴权边界对比

| 维度 | `/api/features/:key` (公共) | `/api/v1/sdk-payload/:key` (API) |
|------|----------------------------|---------------------------------|
| 路由注册位置 | `app.ts` 直接注册 | `api.router.ts` → `allRoutes` |
| 鉴权中间件 | 无 | `authenticateApiRequestMiddleware` |
| 鉴权方式 | URL path 中的 key（SDK Connection key） | Authorization header（Secret API Key） |
| 支持 SDK key | ✅ 是 | ❌ 否（会报错："SDK Endpoint key given instead"） |
| 支持 Secret key | ❌ 否 | ✅ 是 |
| CDN 缓存头 | ✅ 完整设置 | ❌ 无 |
| remoteEval 检查 | ✅ 检查 | ❌ 不检查 |

> **关键边界**：`authenticateApiRequestMiddleware` 在 `api.router.ts:88` 全局生效，所有 `/api/v1/*` 路由都必须经过。它会检查 API key 的 `secret` 字段，SDK Endpoint key（`secret=false`）会被拒绝。

### 5.4 设计意图

这三条路径的分叉实现了权限控制：

1. **客户端 SDK** → 通过 `/api/features/:key`（公共 SDK 端点）拉取配置进行本地 bucketing
2. **浏览器端** → 当需要远程评估时，服务端通过 `/api/eval/:key` 计算后返回结果
3. **安全隔离** → 开启 `remoteEvalEnabled` 时，禁止通过公开的 `/api/features/:key` 端点暴露原始实验配置
4. **服务端集成** → `/api/v1/sdk-payload/:key` 用于需要 Secret API Key 鉴权的服务端场景

---

## 6. payloadKeys 为空时的 CDN 清理路径

当 `payloadKeys = []` 时（如 SDK Connection 创建/更新、Environment 更新），`triggerWebhookJobs` 中的 CDN purge 逻辑会切换到**按 connection.key 逐个清理**的模式。

### 6.1 两种 CDN purge 模式对比

#### 模式 1：payloadKeys 非空 → 按 environment 批量清理（默认）

```ts
// jobs/updateAllJobs.ts:42-48
const environments = Array.from(
  new Set(payloadKeys.map((k) => k.environment)),
);
const surrogateKeys = getSurrogateKeysFromEnvironments(context.org.id, [
  ...environments,
]);
```

- 提取所有受影响的 environment，去重
- 生成 `{orgId}_{envId}` 格式的 surrogate key
- 一次 purge 清除该环境下所有 Connection 的 CDN 缓存
- **高效**：100 个 Connection 同一环境变更，只需 1 次 purge

#### 模式 2：payloadKeys 为空 → 按 connection.key 逐个清理

```ts
// jobs/updateAllJobs.ts:50-55
connections.forEach((conn) => {
  if (!environments.includes(conn.environment)) {
    surrogateKeys.push(conn.key);
  }
});
```

当 `payloadKeys = []` 时：
- `environments = Array.from(new Set([]))` → 空数组 `[]`
- 遍历所有传入的 `connections`
- 由于 `environments` 为空，`!environments.includes(conn.environment)` 始终为 `true`
- **所有 connection.key 都会被加入 surrogateKeys 列表**

### 6.2 触发模式 2 的场景

| 场景 | 触发源 | payloadKeys | connections | CDN purge 模式 |
|------|--------|------------|------------|---------------|
| 新建 SDK Connection | `SdkConnectionModel.ts:281` | `[]` | `[newConn]` | 按 key 清理 1 个 |
| 更新 SDK Connection | `SdkConnectionModel.ts:412` | `[]` | 受影响 connections | 按 key 清理 N 个 |
| 更新 Environment | `environment.controller.ts:211` | `[]` | 该环境所有 connections | 按 key 清理 N 个 |

### 6.3 边界条件与性能考量

**边界条件 1：Connection 的 environment 不在 payloadKeys 中**

即使 `payloadKeys` 非空，如果某个 Connection 的环境不在 `payloadKeys` 的环境集合中，也会降级为按 key 清理。这发生在：

- 某些变更只影响特定项目，但 Connection 绑定了其他项目
- 跨环境的 Connection 属性变更

**边界条件 2：空 connections 数组**

如果 `payloadKeys = []` 且 `connections = []`，则 `surrogateKeys = []`，不执行任何 CDN purge。

**性能考量**：

- 当一个环境有大量 Connection（如 100+）且触发 Environment 更新时，会生成 100+ 个 surrogate key 进行 purge
- 相比按 environment 批量 purge（1 个 key），这在大规模场景下效率较低
- 但 Environment 更新频率较低，权衡可接受

### 6.4 Surrogate-Key 响应头设置

读路径 `/api/features/:key` 上设置的 Surrogate-Key 包括三个维度（`controllers/features.ts:539-546`）：

```ts
const surrogateKeys = [
  params.organization,              // 组织级：orgId
  key,                               // Connection 级：connection.key
  ...getSurrogateKeysFromEnvironments(params.organization, [
    params.environment,             // 环境级：{orgId}_{envId}
  ]),
];
res.set("Surrogate-Key", surrogateKeys.join(" "));
```

这意味着：

1. **按组织 purge**：清除该组织所有 Connection 的缓存
2. **按 Connection purge**：清除单个 Connection 的缓存（模式 2）
3. **按环境 purge**：清除该环境所有 Connection 的缓存（模式 1）

三种粒度的 purge 都能命中，提供了灵活的缓存失效策略。

---

## 7. 多环境隔离实现（Multi-Environment Isolation）

### 7.1 隔离模型

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

### 7.2 刷新时的环境过滤

`refreshSDKPayloadCache` 中的环境过滤（`services/features.ts:651-658`）：

```ts
const allowedEnvs = new Set(getEnvironmentIdsFromOrg(context.org));
payloadKeys = payloadKeys.filter((k) => allowedEnvs.has(k.environment));
```

已删除的环境不会触发刷新，避免无效计算。

### 7.3 CDN 层环境级清除

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

### 7.4 Legacy API Key 的环境隔离

旧式 API key 的缓存键编码了环境信息：

```
legacy:{apiKey}:production:p1
legacy:{apiKey}:staging:p1
```

但旧式缓存无法精确跟踪哪些 environment 受影响，因此 `refreshSDKPayloadCache` 每次都调用 `deleteAllLegacyCacheEntries()` **删除该 org 下所有旧式缓存**，下次请求时 JIT 回源。

### 7.5 payload 生成时的环境过滤

`buildSDKPayloadForConnection`（`services/features.ts:1052-1224`）中：

- `environment` 参数传给 `generateFeaturesPayload`，仅提取该环境下 enabled 的 feature rules
- `filterProjectsByEnvironmentWithNull` 根据 environment 的 project 配置进一步过滤
- `holdoutsMap` 按 environment 独立加载（`getAllPayloadHoldouts(environment)`）

---

## 8. 完整数据流图

```
                          ┌─────────────────────┐
                          │   Feature/Experiment │
                          │   /SavedGroup 等变更  │
                          └────────┬────────────┘
                                   │
                      queueSDKPayloadRefresh()
                        (payloadKeys? connections?)
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
              (按Connection)  (按Connection)  ┌───────────────┐
                                            │ payloadKeys?   │
                                            ├───────────────┤
                                            │ 非空 → env 批量 │
                                            │ 为空 → key 逐个 │
                                            └───────────────┘


          ┌─────────────────────────────────────────────────────────┐
          │                   SDK 读取路径                             │
          │                                                          │
          │  ┌──────────────────────┐    ┌─────────────────────────┐ │
          │  │ GET /api/features    │    │ POST /api/eval (自托管)  │ │
          │  │ getFeaturesPublic    │    │ getEvaluatedFeaturesPublic │ │
          │  └─────────┬────────────┘    └────────────┬────────────┘ │
          │            │                              │              │
          │      remoteEval=true?                remoteEval=false?    │
          │         ↓ 是 ↓否                          ↓ 是 ↓否        │
          │      错误   正常返回                     正常计算  错误     │
          │       │        │                           │             │
          │       │        └───────────────┬───────────┘             │
          │       │                        │                         │
          │       │     GET /api/v1/sdk-payload (Secret Key 鉴权)    │
          │       │     getSdkPayload - 无缓存头, 不检查 remoteEval    │
          │       │                        │                         │
          │       └────────────────────────┼─────────────────────────┘
          │                                │
          │           getFeatureDefinitionsWithCache()
          │                                │
          │                ┌───────────────┴───────────────┐
          │                │ MongoDB Cache (两层: Mongo + CDN) │
          │                └───────────────┬───────────────┘
          │                                │
          │                ┌───────────────┴───────────────┐
          │                │ 命中      │      未命中        │
          │                └───┬─────────┘     ┌─────┴─────┐
          │                    │               │          │
          │                返回缓存   getFeatureDefs()
          │                                │          │
          │                            返回给 SDK     │
          │                                ↓          │
          │                          upsert (异步写回)
          │                                                          │
          │  Cache-Control: max-age=30 (仅 /api/features 设置)       │
          │  Cache-Control: no-store (/api/eval)                     │
          │  无缓存头 (/api/v1/sdk-payload)                           │
          └─────────────────────────────────────────────────────────┘
```

---

## 9. 设计特点与注意事项

### 9.1 无 TTL 的主动失效策略

MongoDB 缓存层没有 TTL 索引，完全依赖事件驱动的 upsert 覆盖。这意味着：

- **优点**：不会出现「缓存存在但已过时」的中间态——任何数据变更都立即触发重算
- **风险**：如果 `queueSDKPayloadRefresh` 的 fire-and-forget 丢失（进程崩溃），缓存可能长期不更新。CDN 的 `stale-while-revalidate=3600` 提供了 1 小时的兜底窗口

### 9.2 Legacy 缓存的全量清除

旧式 API key 的缓存无法精确追踪，每次刷新都执行 `deleteAllLegacyCacheEntries()`（正则匹配 `^legacy:`）。这在旧 key 较多时可能有性能影响，但保证了正确性。

### 9.3 缓存键不含 schemaVersion

读取时通过 `getById` 的查询条件附带 `schemaVersion`，写入时 upsert 自动升级版本。这种设计实现了 schema 版本升级时的零停机迁移——旧版本缓存自然 miss，JIT 回源写入新版本。

### 9.4 并发控制

`refreshSDKPayloadCache` 使用 `promiseAllChunks(promises, 4)` 限制 4 并发写入 MongoDB，避免大规模组织（数百个 SDK Connection）刷新时压垮数据库。

### 9.5 共享 rawData 的不可变性

批量刷新时，所有 Connection 共享同一份 `rawData`（features、experimentMap 等），`buildSDKPayloadForConnection` 内部通过 `cloneDeep` 保证不修改共享数据。测试（`sdk-payload-lifecycle.test.ts:311-363`）专门验证了这一点。

### 9.7 CDN Purge 的双模式设计

`payloadKeys` 空/非空触发不同的 purge 策略，在效率和精确性之间取得平衡：

- Feature 等数据变更 → 按环境批量 purge（高效）
- Connection/Environment 元数据变更 → 按 key 逐个 purge（精确）

### 9.8 双端点设计：公共 SDK 端点 vs 内部 API 端点

系统存在两个返回 SDK payload 的端点，设计意图不同：

- **`/api/features/:key`**：面向客户端 SDK，公共端点，URL path 传 SDK key，带 CDN 缓存，检查 `remoteEvalEnabled`
- **`/api/v1/sdk-payload/:key`**：面向服务端集成，需 Secret API Key 鉴权，无 CDN 缓存，不检查 `remoteEvalEnabled`

> 关键边界：`authenticateApiRequestMiddleware` 是 `/api/v1/*` 路由的强制准入关卡，SDK Endpoint key 无法通过。
