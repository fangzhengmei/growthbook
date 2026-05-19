# SDK 连接发布载荷与环境密钥强绑定机制分析

## 概述

GrowthBook SDK 连接机制中，发布载荷（SDK Payload）与所属环境密钥强绑定。整个链路分为三段：

1.  **密钥生成与吊销** —— 控制台创建/删除 SDK 连接时生成唯一 `key` 和 `encryptionKey`，删除即吊销
2.  **载荷构造与缓存** —— 后端根据连接配置（环境、项目、加密选项等）生成并缓存 SDK 载荷
3.  **客户端拉取与重连** —— SDK 客户端使用 `clientKey` 拉取载荷、SSE 实时更新与错误重试

---

## 一、两个 API 入口的明确区分

### 1.1 入口对比

| 路径 | 认证要求 | 处理函数 | 用途 | 代码位置 |
|------|---------|---------|------|---------|
| `/api/features/:key` | 公开（无需 JWT），允许 CORS | `featuresController.getFeaturesPublic` | SDK 客户端拉取载荷的主入口 | `app.ts:297-304` |
| `/api/v1/sdk-payload/:key` | 需要 API Key 认证（经过 `authenticateApiRequestMiddleware`） | `getSdkPayload` | 后台管理或服务端调用 | `api/sdk-payload/getSdkPayload.ts:19-46` |
| `/sub/:key` | SSE 流式连接 | 独立代理/边缘服务处理 | 实时推送更新通知 | `feature-repository.ts:69` |

> **重要校正**：SDK 客户端实际调用的是 `/api/features/:key`，而非 `/sdk-payload/:key`。后者是带认证的管理 API。

### 1.2 `/api/features/:key` 处理流程 `app.ts:297-304`

```typescript
// Public features for SDKs
app.get(
  "/api/features/:key?",
  cors({ credentials: false, origin: "*" }),
  featuresController.getFeaturesPublic,
);
```

调用链：
`getFeaturesPublic` → `getPayloadParamsFromApiKey(key)` → `getFeatureDefinitionsWithCache()`

### 1.3 `/api/v1/sdk-payload/:key` 处理流程 `getSdkPayload.ts:19-46`

```typescript
export const getSdkPayload = createApiRequestHandler({
  paramsSchema: z.object({ key: z.string() }),
  method: "get" as const,
  path: "/sdk-payload/:key",
  // ...
})(async (req) => {
  const { key } = req.params;
  const params = await getPayloadParamsFromApiKey(key, req);
  const defs = await getFeatureDefinitionsWithCache({ context: req.context, params });
  return { status: 200, ...defs };
});
```

---

## 二、密钥生成与吊销

### 2.1 密钥体系设计

| 密钥类型 | 生成时机 | 用途 | 存储位置 | 代码位置 |
|---------|---------|------|---------|---------|
| `key` (SDK 连接 Key) | 创建 SDK 连接时 | 客户端请求载荷的唯一标识，URL 路径参数，缓存键 | `sdkconnections` 集合 | `SdkConnectionModel.ts:217-221` |
| `encryptionKey` | 创建 SDK 连接时 | AES-CBC 128 位密钥，加密载荷内容 | `sdkconnections` 集合 | `api-key.util.ts:50-62` |
| `proxy.signingKey` | 创建 SDK 连接时 | 代理签名密钥 | `sdkconnections` 集合 | `SdkConnectionModel.ts:246` |

### 2.2 `generateEncryptionKey` 定义与调用链路

**定义位置**：`packages/back-end/src/util/api-key.util.ts:50-62`

```typescript
export async function generateEncryptionKey(): Promise<string> {
  const key = await webcrypto.subtle.generateKey(
    { name: "AES-CBC", length: 128 },
    true,
    ["encrypt", "decrypt"],
  );
  return Buffer.from(await webcrypto.subtle.exportKey("raw", key)).toString("base64");
}
```

**调用位置 1**：SDK 连接创建 `SdkConnectionModel.ts:34, 238`

```typescript
// 导入
import { generateEncryptionKey } from "back-end/src/util/api-key.util";

// 创建连接时调用
const connection: SDKConnectionInterface = {
  // ...
  encryptionKey: await generateEncryptionKey(),
  // ...
};
```

**调用位置 2**：旧版 API Key 创建 `ApiKeyModel.ts:5, 360`

```typescript
import { generateEncryptionKey } from "back-end/src/util/api-key.util";

// 创建旧版 API Key 时可选调用
encryptionKey: encryptSDK ? await generateEncryptionKey() : undefined,
```

> **职责澄清**：`generateEncryptionKey` 是通用工具函数，定义在 `api-key.util.ts` 中，被 `SdkConnectionModel` 和 `ApiKeyModel` 共同使用。

### 2.3 SDK 连接 Key 生成 `SdkConnectionModel.ts:217-221`

```typescript
function generateSDKConnectionKey() {
  return generateSigningKey("sdk-", 12);
}
```

- 前缀固定为 `sdk-`，用于 `getPayloadParamsFromApiKey()` 中通过正则 `/^sdk-/` 识别

### 2.4 强绑定关系 `SdkConnectionInterface` `sdk-connection.d.ts:66-105`

```typescript
export interface SDKConnectionInterface {
  id: string;
  organization: string;
  name: string;
  // 强绑定环境
  environment: string;
  projects: string[];
  // 加密配置
  encryptPayload: boolean;
  encryptionKey: string;
  // 客户端拉取用 Key
  key: string;
  // 其他配置...
}
```

### 2.5 密钥吊销（删除连接）`SdkConnectionModel.ts:451-461`

```typescript
export async function deleteSDKConnectionModel(
  context: ReqContext,
  sdkConnection: SDKConnectionInterface,
) {
  await SDKConnectionModel.deleteOne({
    organization: context.org.id,
    id: sdkConnection.id,
  });
  await audit.logDelete(context, sdkConnection);
}
```

> **重要校正**：删除连接时**没有主动删除对应的缓存记录**。缓存记录（`sdkcache` 集合）保留，但后续请求时会因 `findSDKConnectionByKey(key)` 找不到连接而抛出 `UnrecoverableApiError("Invalid API Key")`。

### 2.6 吊销验证 `controllers/features.ts:326-329`

```typescript
if (key.match(/^sdk-/)) {
  const connection = await findSDKConnectionByKey(key);
  if (!connection) {
    throw new UnrecoverableApiError("Invalid API Key");
  }
  // ...
}
```

---

## 三、控制台到后端完整闭环（创建/更新/删除）

### 3.1 创建 SDK 连接

| 层级 | 代码位置 | 关键操作 |
|------|---------|---------|
| 前端调用点 | `SDKConnectionForm.tsx:429-435` | `apiCall(/sdk-connections, { method: "POST", body: ... })` |
| API 路由层 | `sdk-connection.router.ts:11` | `router.post("/", sdkConnectionController.postSDKConnection)` |
| 控制器层 | `sdk-connection.controller.ts:73-90` | 调用 `createSDKConnection()` → `queueSDKPayloadRefresh()` |
| 模型层 | `SdkConnectionModel.ts:230-294` | `createSDKConnection()` 生成 key + encryptionKey → 写入 DB → `queueSDKPayloadRefresh()` |

**前端调用** `SDKConnectionForm.tsx:429-435`

```typescript
const res = await apiCall<{ connection: SDKConnectionInterface }>(
  `/sdk-connections`,
  {
    method: "POST",
    body: JSON.stringify(body),
  },
);
```

**控制器层** `sdk-connection.controller.ts:73-90`

```typescript
const doc = await createSDKConnection(context, {
  ...params,
  encryptPayload,
  hashSecureAttributes,
  remoteEvalEnabled,
  organization: org.id,
});

queueSDKPayloadRefresh({
  context,
  payloadKeys: [],
  sdkConnections: [doc],
  auditContext: { event: "created", model: "sdkconnection", id: doc.id },
});
```

### 3.2 更新 SDK 连接

| 层级 | 代码位置 | 关键操作 |
|------|---------|---------|
| 前端调用点 | `SDKConnectionForm.tsx:423-426` | `apiCall(/sdk-connections/${id}, { method: "PUT", body: ... })` |
| API 路由层 | `sdk-connection.router.ts:13` | `router.put("/:id", sdkConnectionController.putSDKConnection)` |
| 控制器层 | `sdk-connection.controller.ts:98-167` | 调用 `editSDKConnection()` → 如配置变更则 `queueSDKPayloadRefresh()` |
| 模型层 | `SdkConnectionModel.ts:297-429` | `editSDKConnection()` 更新 DB → 如环境/项目/加密等变更则 `queueSDKPayloadRefresh()` |

**前端调用** `SDKConnectionForm.tsx:423-426`

```typescript
await apiCall(`/sdk-connections/${initialValue.id}`, {
  method: "PUT",
  body: JSON.stringify(body),
});
```

**配置变更触发刷新** `SdkConnectionModel.ts:369-427`

```typescript
const keysRequiringProxyUpdate = [
  "sdkVersion", "projects", "environment", "encryptPayload",
  "hashSecureAttributes", "remoteEvalEnabled", "includeVisualExperiments",
  "includeDraftExperiments", "includeExperimentNames", "includeRedirectExperiments",
  "includeRuleIds", "includeProjectIdInMetadata", "includeCustomFieldsInMetadata",
  "allowedCustomFieldsInMetadata", "includeTagsInMetadata",
  "savedGroupReferencesEnabled",
] as const;

keysRequiringProxyUpdate.forEach((key) => {
  if (key in otherChanges && !isEqual(otherChanges[key], connection[key])) {
    needsProxyUpdate = true;
  }
});

if (needsProxyUpdate) {
  queueSDKPayloadRefresh({
    context,
    payloadKeys: [],
    sdkConnections: [{ ...connection, ...fullChanges }],
    auditContext: { event: "updated", model: "sdkconnection", id: connection.id },
  });
}
```

### 3.3 删除 SDK 连接

| 层级 | 代码位置 | 关键操作 |
|------|---------|---------|
| 前端调用点 | `sdks/[sdkid].tsx:159-161` | `apiCall(/sdk-connections/${id}, { method: "DELETE" })` |
| API 路由层 | `sdk-connection.router.ts:21` | `router.delete("/:id", sdkConnectionController.deleteSDKConnection)` |
| 控制器层 | `sdk-connection.controller.ts:146-167` | 调用 `deleteSDKConnectionModel()` |
| 模型层 | `SdkConnectionModel.ts:451-461` | `deleteSDKConnectionModel()` 从 DB 删除连接记录 |

**前端调用** `sdks/[sdkid].tsx:159-161`

```typescript
await apiCall(`/sdk-connections/${connection.id}`, {
  method: "DELETE",
});
```

---

## 四、SDK 连接创建阶段的双重刷新机制分析

### 4.1 问题确认：同一次创建动作触发两次 `queueSDKPayloadRefresh`

在 SDK 连接创建流程中，**模型层**和**控制器层**都独立调用了 `queueSDKPayloadRefresh`，导致同一次创建动作触发两次完整的 payload 刷新。

**调用链概览**：

```
POST /sdk-connections
    │
    ├─→ sdk-connection.controller.ts:postSDKConnection
    │    │
    │    ├─→ createSDKConnection()  [模型层]
    │    │    └─→ queueSDKPayloadRefresh()  ← 第一次调用 (SdkConnectionModel.ts:281)
    │    │
    │    └─→ queueSDKPayloadRefresh()  ← 第二次调用 (sdk-connection.controller.ts:81)
    │
    └─→ 返回响应
```

### 4.2 完整执行路径与代码锚点

| 层级 | 代码位置 | 关键操作 |
|------|---------|---------|
| **前端入口** | `SDKConnectionForm.tsx:429-435` | `apiCall(/sdk-connections, { method: "POST" })` |
| **API 路由层** | `sdk-connection.router.ts:11` | `router.post("/", sdkConnectionController.postSDKConnection)` |
| **控制器层调用** | `sdk-connection.controller.ts:81-90` | 调用 `createSDKConnection()` 返回后，执行 `queueSDKPayloadRefresh()` |
| **模型层调用** | `SdkConnectionModel.ts:281-290` | `createSDKConnection()` 内部写入 DB 后，执行 `queueSDKPayloadRefresh()` |
| **刷新执行** | `services/features.ts:607-621` | `queueSDKPayloadRefresh()` → 异步调用 `refreshSDKPayloadCache()` |

### 4.3 第一次调用（模型层）`SdkConnectionModel.ts:281-290`

```typescript
// createSDKConnection() 函数内部
const doc = await SDKConnectionModel.create(connection);

if (IS_CLOUD) {
  // ... Cloud 环境的 SDK 映射
}

queueSDKPayloadRefresh({
  context,
  payloadKeys: [],
  sdkConnections: [connection],
  auditContext: {
    event: "created",
    model: "sdkconnection",
    id: connection.id,
  },
});

const created = toInterface(doc);
await audit.logCreate(context, created);
return created;
```

**调用时机**：DB 写入完成后，`createSDKConnection` 函数返回前。

### 4.4 第二次调用（控制器层）`sdk-connection.controller.ts:81-90`

```typescript
const doc = await createSDKConnection(context, {
  ...params,
  encryptPayload,
  hashSecureAttributes,
  remoteEvalEnabled,
  organization: org.id,
});

queueSDKPayloadRefresh({
  context,
  payloadKeys: [],
  sdkConnections: [doc],
  auditContext: {
    event: "created",
    model: "sdkconnection",
    id: doc.id,
  },
});

res.status(200).json({
  status: 200,
  connection: doc,
});
```

**调用时机**：`createSDKConnection()` 返回后，响应发送前。

### 4.5 去重/合并保障机制检查

| 机制类型 | 是否存在 | 说明 |
|---------|---------|------|
| **内存去重** | ❌ 不存在 | `queueSDKPayloadRefresh` 直接调用 `refreshSDKPayloadCache`，无任何去重逻辑 |
| **任务队列** | ❌ 不存在 | 直接异步调用，没有经过任务队列或批处理 |
| **防抖/节流** | ❌ 不存在 | 没有任何 debounce 或 throttle 机制 |
| **乐观锁** | ❌ 不存在 | `upsert` 方法没有版本检查或乐观锁 |
| **幂等性** | ✅ 存在 | 两次调用结果相同，最终数据一致 |

**`upsert` 实现分析** `SdkConnectionCacheModel.ts:41-63`：

```typescript
public async upsert(
  id: string,
  contents: string,
  auditContext?: SdkConnectionCacheAuditContext,
) {
  // Find existing doc by id only (ignore version) to support version upgrades
  const existing = await this._findOne({ id });
  const updateData = {
    contents,
    schemaVersion: LATEST_SDK_PAYLOAD_SCHEMA_VERSION,
    audit: auditContext,
  };
  if (existing) {
    return this.update(existing, updateData);  // 第二次调用会走这里
  }
  return this.create({ id, ...updateData });  // 第一次调用会走这里
}
```

> **关键点**：两次调用都会执行完整的 `refreshSDKPayloadCache` 流程，只是最终 `upsert` 时第一次创建、第二次更新。

### 4.6 实际影响分析

| 影响维度 | 分析结论 | 说明 |
|---------|---------|------|
| **刷新次数** | ❌ 2 次完整刷新 | 同一次创建触发两次独立的 `refreshSDKPayloadCache` 执行 |
| **API 响应时延** | ✅ 无影响 | 两次调用都是 fire-and-forget，不阻塞 API 响应 |
| **数据一致性** | ✅ 无影响 | 两次刷新结果相同，最终数据一致 |
| **资源浪费** | ❌ 显著浪费 | 重复执行：<br>• `getAllPayloadExperiments()` <br>• `getAllPayloadSafeRollouts()` <br>• `getAllSavedGroups()` <br>• `getAllFeatures()` <br>• `getAllVisualExperiments()` <br>• `getAllURLRedirectExperiments()` <br>• `buildSDKPayloadForConnection()` |
| **数据库负载** | ❌ 翻倍 | 两次完整的查询 + 两次 upsert 写入 |
| **稳定性风险** | ⚠️ 低风险 | 不会导致错误，但在高并发创建场景下可能加剧数据库压力 |
| **竞态条件** | ⚠️ 理论存在 | 两次异步执行可能交叉，但 `upsert` 是原子操作，最终状态一致 |

### 4.7 设计意图推测与改进建议

**可能的设计意图**：
1. **模型层自包含**：模型层内部确保创建后缓存立即可用
2. **控制器层统一**：控制器层作为统一入口，确保所有 SDK 连接变更都触发刷新
3. **代码重构遗留**：可能是某次重构时遗漏了对重复调用的清理

**改进建议**：
- 移除其中一次调用（建议移除控制器层的调用，保持模型层自包含）
- 或在 `queueSDKPayloadRefresh` 中增加基于 `connection.key` 的短时间窗口去重（如 1 秒内相同 key 只执行一次）

### 4.8 两次刷新执行时序对比

| 时间点 | 第一次刷新（模型层） | 第二次刷新（控制器层） |
|-------|---------------------|---------------------|
| T0 | 触发 `queueSDKPayloadRefresh` | 未触发 |
| T1 | 异步执行 `refreshSDKPayloadCache` | `createSDKConnection` 返回 |
| T2 | 查询所有数据（features, experiments...） | 触发 `queueSDKPayloadRefresh` |
| T3 | 构建 payload | 异步执行 `refreshSDKPayloadCache` |
| T4 | `upsert` 创建缓存记录（第一次） | 查询所有数据（重复查询） |
| T5 | 完成 | 构建 payload（重复构建） |
| T6 | - | `upsert` 更新缓存记录（第二次） |
| T7 | - | 完成 |

### 4.9 下游影响分析：各链路重复触发与去重保障

两次 `refreshSDKPayloadCache` 调用都会在结束时执行 `triggerWebhookJobs()`（`services/features.ts:821`），因此所有下游链路都会被触发两次。以下是各链路的去重机制与实际风险分析。

#### 4.9.1 调用链总览

```
refreshSDKPayloadCache()
    │
    ├─→ 构建并 upsert 缓存（Mongo）
    │
    └─→ triggerWebhookJobs(context, payloadKeys, connectionsUpdated, true)
          │
          ├─→ queueWebhooksByConnections()        ← 普通 SDK Webhook
          ├─→ fireGlobalSdkWebhooks()              ← 全局 Webhook
          ├─→ queueProxyUpdate()                   ← Proxy 更新
          ├─→ queueLegacySdkWebhooks()             ← Legacy Webhook
          └─→ purgeCDNCache()                       ← CDN 缓存清除
```

#### 4.9.2 各链路详细分析

| 链路 | 代码位置 | 是否去重 | 去重键 | 仍可能重复动作 | 实际风险 |
|------|---------|---------|--------|----------------|---------|
| **普通 SDK Webhook** | `sdkWebhooks.ts:114-125` | ✅ **是** | `data.webhookId` | ❌ 无 | Agenda 的 `job.unique()` 确保同一 `webhookId` 只入队一次 |
| **全局 Webhook** | `sdkWebhooks.ts:356-421` | ❌ **否** | 无 | ✅ 全部重复 | 直接 `runWebhookFetch()`，无队列、无去重，会收到两次通知 |
| **Proxy 更新** | `proxyUpdate.ts:167-183` | ✅ **是** | `data.connectionId` + `data.useCloudProxy` | ❌ 无 | Agenda 的 `job.unique()` 确保同一连接 + 代理类型只入队一次 |
| **Legacy Webhook** | `webhooks.ts:139-180` | ✅ **是** | `webhookId` | ❌ 无 | Agenda 的 `job.unique()` 确保同一 `webhookId` 只入队一次 |
| **CDN Purge** | `cdn.util.ts:17-46` | ❌ **否** | 无 | ✅ 全部重复 | 直接调用 Fastly API，但 Fastly purge 幂等，无功能影响 |

#### 4.9.3 普通 SDK Webhook 去重机制 `sdkWebhooks.ts:103-113`

```typescript
async function queueSingleSdkWebhookJob(webhook: WebhookInterface) {
  const job = agenda.create(SDK_WEBHOOKS_JOB_NAME, {
    webhookId: webhook.id,
    retryCount: 0,
  }) as SDKWebhookJob;
  job.unique({
    "data.webhookId": webhook.id,  // ← 去重键
  });
  job.schedule(new Date());
  await job.save();
}
```

> **说明**：Agenda 的 `unique()` 方法会在 MongoDB 中对 `data.webhookId` 建立唯一索引，同一 webhookId 的重复入队会被合并。

#### 4.9.4 全局 Webhook 无去重 `sdkWebhooks.ts:356-421`

```typescript
export async function fireGlobalSdkWebhooks(
  context: ReqContext | ApiReqContext,
  connections: SDKConnectionInterface[],
) {
  if (!connections.length) return;

  for (const connection of connections) {
    const payload = await getFeatureDefinitionsWithCache({
      context,
      params: connection,
    });

    WEBHOOKS.forEach((webhook) => {
      // ... 构造 webhook 对象
      
      // 直接调用，无队列，无去重
      runWebhookFetch({
        webhook: w,
        key: connection.key,
        payload,
        global: true,
        context: context,
      }).catch((e) => {
        logger.error(e, "Failed to fire global webhook");
      });
    });
  }
}
```

> **风险**：全局 webhook 会收到**两次完全相同的 payload.changed 通知**，接收方需要处理重复事件。

#### 4.9.5 Proxy 更新去重机制 `proxyUpdate.ts:146-165`

```typescript
export async function queueSingleProxyUpdate(
  orgId: string,
  connection: SDKConnectionInterface,
  useCloudProxy: boolean = false,
) {
  if (!connectionSupportsProxyUpdate(connection, useCloudProxy)) return;

  const job = agenda.create(PROXY_UPDATE_JOB_NAME, {
    orgId,
    connectionId: connection.id,
    retryCount: 0,
    useCloudProxy,
  }) as ProxyUpdateJob;
  job.unique({
    "data.connectionId": connection.id,       // ← 去重键 1
    "data.useCloudProxy": useCloudProxy,      // ← 去重键 2
  });
  job.schedule(new Date());
  await job.save();
}
```

> **说明**：去重键是 `connectionId + useCloudProxy` 的组合，确保同一连接的同一代理类型只入队一次。

#### 4.9.6 Legacy Webhook 去重机制 `webhooks.ts:172-178`

```typescript
const job = agenda.create(WEBHOOK_JOB_NAME, {
  webhookId: webhook.id,
  retryCount: 0,
}) as WebhookJob;
job.unique({ webhookId: webhook.id });  // ← 去重键
job.schedule(new Date());
await job.save();
```

#### 4.9.7 CDN Purge 无去重但幂等 `cdn.util.ts:17-46`

```typescript
export async function purgeCDNCache(
  orgId: string,
  surrogateKeys: string[],
): Promise<void> {
  if (!FASTLY_SERVICE_ID || !FASTLY_API_TOKEN) return;
  if (!surrogateKeys.length) return;

  const BATCH_SIZE = 256;
  for (let i = 0; i < surrogateKeys.length; i += BATCH_SIZE) {
    const batch = surrogateKeys.slice(i, i + BATCH_SIZE);
    try {
      // 直接调用 Fastly API，无去重
      await fetch(`https://api.fastly.com/service/${FASTLY_SERVICE_ID}/purge`, {
        method: "POST",
        headers: {
          "Fastly-Key": FASTLY_API_TOKEN,
          "surrogate-key": batch.join(" "),
          Accept: "application/json",
        },
      });
    } catch (e) {
      // ... 日志记录
    }
  }
}
```

> **说明**：虽然会调用两次，但 Fastly 的 purge API 是幂等的，对同一 surrogate key 多次 purge 效果相同，无功能影响，仅浪费少量 API 调用配额。

#### 4.9.8 下游影响总结

| 影响类型 | 影响程度 | 说明 |
|---------|---------|------|
| **功能正确性** | ✅ 无影响 | 有去重的链路只执行一次，无去重的链路（全局 webhook、CDN purge）功能上幂等 |
| **全局 Webhook 接收方** | ⚠️ 中等 | 会收到两次完全相同的通知，需要接收方处理重复事件 |
| **API 调用次数** | ❌ 翻倍 | CDN purge API 调用翻倍，全局 webhook 外部调用翻倍 |
| **数据库写入** | ✅ 无影响 | 有去重的链路通过 Agenda unique 索引避免重复写入 |
| **代理更新** | ✅ 无影响 | 通过去重键避免重复 |
| **Legacy Webhook** | ✅ 无影响 | 通过去重键避免重复 |

---

## 五、载荷构造与缓存

### 5.1 缓存键设计

**缓存键 = `connection.key`**（即 SDK 连接的 `key` 字段）

`SdkConnectionCacheModel.ts:34-39`

```typescript
public async getById(id: string) {
  return await this._findOne({
    id,
    schemaVersion: LATEST_SDK_PAYLOAD_SCHEMA_VERSION,
  });
}
```

`controllers/features.ts:427`

```typescript
const cached = await context.models.sdkConnectionCache.getById(params.key);
```

`services/features.ts:802-806`

```typescript
await context.models.sdkConnectionCache.upsert(
  connection.key,  // 缓存键
  JSON.stringify(contents),
  auditContext,
);
```

### 5.2 载荷构造流程

`getFeatureDefinitionsWithCache()` `controllers/features.ts:415-501`

```
┌─────────────────────────────────────────────────────────┐
│ getFeatureDefinitionsWithCache({ context, params })     │
├─────────────────────────────────────────────────────────┤
│  1. 尝试从 Mongo 缓存读取（key = params.key）           │
│     → 命中则返回 JSON.parse(cached.contents)            │
│  2. 未命中则实时生成                                    │
│     → getFeatureDefinitions()                          │
│       → getFeatureDefinitionsResponse()                │
│         → 可选 encrypt() 加密 payload                 │
│  3. 写入缓存（fire-and-forget）                         │
└─────────────────────────────────────────────────────────┘
```

### 5.3 `queueSDKPayloadRefresh` 完整触发矩阵

**主动失效**：通过 `queueSDKPayloadRefresh()` 触发缓存刷新，**不是删除缓存，而是覆盖更新**

`services/features.ts:607-621`

```typescript
export function queueSDKPayloadRefresh(data: {
  context: ReqContext | ApiReqContext;
  payloadKeys: SDKPayloadKey[];
  sdkConnections?: SDKConnectionInterface[];
  skipRefreshForProject?: string;
  treatEmptyProjectAsGlobal?: boolean;
  auditContext?: { event: string; model: string; id?: string };
}) {
  refreshSDKPayloadCache({ ...data, stackTrace }).catch((e) => {
    logger.error(e, "Error refreshing SDK Payload Cache");
  });
}
```

**完整触发场景矩阵**：

| 触发模块 | 触发场景 | 代码位置 | payloadKeys / sdkConnections |
|---------|---------|---------|-----------------------------|
| **Feature** | 创建、更新、删除、切换状态 | `FeatureModel.ts:873, 899, 927` | 受影响的 `{ environment, project }` |
| **SDK Connection** | 创建 | `SdkConnectionModel.ts:281-290` | `sdkConnections: [newConnection]` |
| **SDK Connection** | 更新（环境/项目/加密等配置变更） | `SdkConnectionModel.ts:411-427` | `sdkConnections: [updatedConnection]` |
| **SDK Connection** | 控制器层创建后 | `sdk-connection.controller.ts:81-90` | `sdkConnections: [doc]` |
| **Experiment** | 启动、停止、关联 feature | `ExperimentModel.ts:2013, 2044` | 受影响的 `{ environment, project }` |
| **Project** | `publicId` 变更 | `ProjectModel.ts:143-154` | 所有环境 + `treatEmptyProjectAsGlobal: true` |
| **Custom Field** | 创建、更新 | `CustomFieldModel.ts:275, 308` | 所有环境 + `treatEmptyProjectAsGlobal: true` |
| **Saved Group** | 创建、更新、删除 | `savedGroups.ts:24-32` | 所有环境 + 所有项目（全局刷新） |
| **Holdout** | 创建、更新、删除、状态变更 | `holdout.controller.ts:522, 558, 615, 639, 721` | `getAffectedSDKPayloadKeys(holdout)` |
| **Holdout (Job)** | 定时状态更新 | `updateHoldoutStatus.ts:138, 171, 201` | `getAffectedSDKPayloadKeys(holdout)` |
| **URL Redirect** | 创建、更新、删除 | `UrlRedirectModel.ts:127, 158` | `getPayloadKeys(context, experiment)` |
| **Visual Changeset** | 创建、更新、删除 | `VisualChangesetModel.ts:428, 464, 495` | 受影响的 `{ environment, project }` |
| **Environment** | 创建、更新 | `environment.controller.ts:211` `putEnvironment.ts:65` | 受影响的环境 |
| **Safe Rollout** | 创建、更新、删除 | `SafeRolloutModel.ts:85` | 受影响的 `{ environment, project }` |

**匹配逻辑** `services/features.ts:728-740`

```typescript
sdkConnections.forEach((connection) => {
  if (
    !sdkConnectionsToUpdate.some((c) => c.key === connection.key) &&
    !payloadKeys.some((k) =>
      isSDKConnectionAffectedByPayloadKey(
        connection,
        k,
        treatEmptyProjectAsGlobal,
      ),
    )
  ) {
    return;  // 跳过不受影响的连接
  }
  // 否则刷新此连接的缓存
});
```

### 5.4 加密载荷构造 `services/features.ts:843-960`

`getFeatureDefinitionsResponse()` 中处理加密：

```typescript
export async function getFeatureDefinitionsResponse({
  features,
  encryptPayload,
  encryptionKey,
  // ...
}): Promise<{
  features: Record<string, FeatureDefinition>;
  encryptedFeatures?: string;
  // ...
}> {
  if (!encryptPayload || !encryptionKey) {
    return { features, dateUpdated };
  }
  // AES-CBC 加密
  const encryptedFeatures = await encrypt(JSON.stringify(features), encryptionKey);
  return { features: {}, encryptedFeatures };
}
```

---

## 六、客户端拉取与重连

### 6.1 拉取入口 `feature-repository.ts:50-54`

```typescript
fetchFeaturesCall: ({ host, clientKey, headers }) => {
  return (polyfills.fetch as typeof globalThis.fetch)(
    `${host}/api/features/${clientKey}`,
    { headers },
  );
},
```

### 6.2 SSE 实时更新连接 `feature-repository.ts:67-74`

```typescript
eventSourceCall: ({ host, clientKey, headers }) => {
  if (headers) {
    return new polyfills.EventSource(`${host}/sub/${clientKey}`, { headers });
  }
  return new polyfills.EventSource(`${host}/sub/${clientKey}`);
},
```

> **SSE 端点 `/sub/:key`**：由独立的代理/边缘服务处理，不在主后端 app.ts 路由中。

### 6.3 拉取策略 `feature-repository.ts:384-446`

**SWR（Stale-While-Revalidate）策略**：

```
┌─────────────────────────────────────────────────┐
│ fetchFeatures(instance)                         │
├─────────────────────────────────────────────────┤
│  1. 检查内存缓存                                │
│     → 未过期 → 返回缓存数据                     │
│  2. 检查是否已有正在进行的请求（去重）          │
│  3. 发起网络请求                                │
│     → 响应头 x-sse-support: enabled → 标记支持 SSE │
│     → 成功 → onNewFeatureData → 更新缓存        │
│     → 失败 → 返回错误                           │
└─────────────────────────────────────────────────┘
```

### 6.4 SSE 事件处理 `feature-repository.ts:473-496`

```typescript
cb: (event: MessageEvent<string>) => {
  try {
    if (event.type === "features-updated") {
      // 通知有更新，触发重新拉取
      fetchFeatures(instance);
    } else if (event.type === "features") {
      // 直接推送 payload 数据
      const json: FeatureApiResponse = JSON.parse(event.data);
      onNewFeatureData(key, cacheKey, json);
    }
    channel.errors = 0;  // 重置错误计数
  } catch (e) {
    onSSEError(channel);
  }
},
```

### 6.5 错误重试与退避 `feature-repository.ts:505-521`

```typescript
function onSSEError(channel: ScopedChannel) {
  if (channel.state === "idle") return;
  channel.errors++;
  if (channel.errors > 3 || (channel.src && channel.src.readyState === 2)) {
    // 指数退避：3^(errors-3) * (1000 + random(0-1000)) ms
    const delay =
      Math.pow(3, channel.errors - 3) * (1000 + Math.random() * 1000);
    disableChannel(channel);
    setTimeout(
      () => enableChannel(channel),
      Math.min(delay, 300000),  // 最大 5 分钟
    );
  }
}
```

### 6.6 客户端解密 `GrowthBookClient.ts`

```typescript
public async setPayload(payload: FeatureApiResponse): Promise<void> {
  this._payload = payload;
  const data = await decryptPayload(payload, this._options.decryptionKey);
  this._decryptedPayload = data;
  if (data.features) {
    this._features = data.features;
  }
  this.ready = true;
}
```

---

## 七、完整链路图

```
┌─────────────────────────────────────────────────────────────────────────┐
│ 控制台 UI (front-end)                                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  SDKConnectionForm.tsx                            sdks/[sdkid].tsx     │
│  ├─ POST /sdk-connections (创建)             ├─ DELETE /sdk-connections/:id │
│  └─ PUT /sdk-connections/:id (更新)          └─ ...                     │
│                                                                         │
│  其他管理页面                                                             │
│  ├─ 功能管理（Feature）→ 触发 payloadKeys                                │
│  ├─ 实验管理（Experiment）→ 触发 payloadKeys                             │
│  ├─ 项目管理（Project）→ 触发全局刷新                                     │
│  ├─ 自定义字段（Custom Field）→ 触发全局刷新                              │
│  ├─ 分组管理（Saved Group）→ 触发全局刷新                                 │
│  ├─ Holdout 管理 → 触发受影响环境刷新                                     │
│  ├─ URL Redirect 管理 → 触发关联实验刷新                                  │
│  └─ Visual Changeset 管理 → 触发受影响环境刷新                            │
│                                                                         │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 后端 API 路由层                                                          │
├─────────────────────────────────────────────────────────────────────────┤
│  sdk-connection.router.ts                                                │
│  ├─ POST /          → postSDKConnection                                  │
│  ├─ PUT /:id        → putSDKConnection                                   │
│  └─ DELETE /:id     → deleteSDKConnection                                │
│                                                                         │
│  其他路由（features, experiments, projects, ...）                        │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 后端模型层 / 服务层                                                      │
├─────────────────────────────────────────────────────────────────────────┤
│  SdkConnectionModel.ts                                                   │
│  ├─ createSDKConnection() → generateSDKConnectionKey()                  │
│  │                           → generateEncryptionKey()                  │
│  │                           → queueSDKPayloadRefresh()  ⚠️ 第一次      │
│  ├─ editSDKConnection()   → 如配置变更 → queueSDKPayloadRefresh()       │
│  └─ deleteSDKConnectionModel() → 从 DB 删除记录                          │
│                                                                         │
│  sdk-connection.controller.ts                                            │
│  └─ postSDKConnection() → 调用 createSDKConnection() 返回后             │
│                           → queueSDKPayloadRefresh()  ⚠️ 第二次（重复）  │
│                                                                         │
│  services/features.ts                                                    │
│  ├─ queueSDKPayloadRefresh() → refreshSDKPayloadCache()                 │
│  │                                   ├─ 匹配受影响的 SDK 连接            │
│  │                                   ├─ buildSDKPayloadForConnection()  │
│  │                                   └─ sdkConnectionCache.upsert()      │
│  ├─ getFeatureDefinitionsResponse() → 可选 encrypt() 加密               │
│  └─ ...                                                                  │
│                                                                         │
│  其他模型（FeatureModel, ExperimentModel, ProjectModel, ...）            │
│  └─ 各自 afterUpdate / afterDelete 钩子 → queueSDKPayloadRefresh()       │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 数据库                                                                    │
├─────────────────────────────────────────────────────────────────────────┤
│  sdkconnections 集合       → 存储连接配置（key, encryptionKey, ...）    │
│  sdkcache 集合             → 存储载荷缓存（key = connection.key）        │
│  features / experiments / projects / ... 集合                            │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │
          ┌─────────────────────┴─────────────────────┐
          │                                           │
          ▼                                           ▼
┌───────────────────────────────┐    ┌───────────────────────────────────┐
│ 客户端 SDK (sdk-js)          │    │ 代理 / 边缘服务                    │
├───────────────────────────────┤    ├───────────────────────────────────┤
│  feature-repository.ts        │    │  /sub/:key (SSE 端点)              │
│  ├─ fetchFeatures()           │    │  ├─ features-updated → 通知重拉    │
│  │  → GET /api/features/:key  │    │  └─ features → 直接推送数据        │
│  ├─ startAutoRefresh()        │    │                                   │
│  │  → EventSource /sub/:key   │    │                                   │
│  └─ onSSEError()              │    │                                   │
│     → 指数退避重试            │    │                                   │
│                              │    │                                   │
│  GrowthBookClient.ts          │    │                                   │
│  └─ setPayload()              │    │                                   │
│     → decryptPayload()        │    │                                   │
└───────────────────────────────┘    └───────────────────────────────────┘
```

---

## 八、核心代码文件索引

| 模块 | 文件路径 | 关键函数 |
|------|---------|---------|
| 工具函数 | `packages/back-end/src/util/api-key.util.ts` | `generateEncryptionKey`, `generateSigningKey` |
| CDN 工具 | `packages/back-end/src/util/cdn.util.ts` | `purgeCDNCache`, `getSurrogateKeysFromEnvironments` |
| 类型定义 | `packages/shared/types/sdk-connection.d.ts` | `SDKConnectionInterface` |
| 后端模型 | `packages/back-end/src/models/SdkConnectionModel.ts` | `generateSDKConnectionKey`, `createSDKConnection`, `editSDKConnection`, `deleteSDKConnectionModel` |
| 缓存模型 | `packages/back-end/src/models/SdkConnectionCacheModel.ts` | `getById`, `upsert` |
| 载荷生成 | `packages/back-end/src/services/features.ts` | `queueSDKPayloadRefresh`, `refreshSDKPayloadCache`, `buildSDKPayloadForConnection`, `getFeatureDefinitionsResponse` |
| 控制器 | `packages/back-end/src/controllers/features.ts` | `getFeatureDefinitionsWithCache`, `getPayloadParamsFromApiKey`, `getFeaturesPublic` |
| 连接控制器 | `packages/back-end/src/routers/sdk-connection/sdk-connection.controller.ts` | `postSDKConnection`, `putSDKConnection`, `deleteSDKConnection` |
| 连接路由 | `packages/back-end/src/routers/sdk-connection/sdk-connection.router.ts` | 路由定义 |
| API 路由 | `packages/back-end/src/api/sdk-payload/getSdkPayload.ts` | `getSdkPayload` |
| Webhook 任务调度 | `packages/back-end/src/jobs/updateAllJobs.ts` | `triggerWebhookJobs` |
| SDK Webhook | `packages/back-end/src/jobs/sdkWebhooks.ts` | `queueWebhooksByConnections`, `fireGlobalSdkWebhooks`, `queueSingleSdkWebhookJob` |
| Legacy Webhook | `packages/back-end/src/jobs/webhooks.ts` | `queueLegacySdkWebhooks` |
| Proxy 更新 | `packages/back-end/src/jobs/proxyUpdate.ts` | `queueProxyUpdate`, `queueSingleProxyUpdate` |
| 前端表单 | `packages/front-end/components/Features/SDKConnections/SDKConnectionForm.tsx` | 创建/编辑 SDK 连接 |
| 前端详情 | `packages/front-end/pages/sdks/[sdkid].tsx` | 删除 SDK 连接 |
| 客户端拉取 | `packages/sdk-js/src/feature-repository.ts` | `fetchFeatures`, `startAutoRefresh`, `onSSEError`, `enableChannel`, `disableChannel` |
| 客户端解密 | `packages/sdk-js/src/GrowthBookClient.ts` | `setPayload` |

---

## 九、关键结论

1.  **`generateEncryptionKey` 职责澄清**：定义在 `api-key.util.ts:50-62`，是通用工具函数，被 `SdkConnectionModel`（SDK 连接）和 `ApiKeyModel`（旧版 API Key）共同调用。

2.  **环境强绑定**：`SDKConnectionInterface.environment` 在创建时确定，`getPayloadParamsFromApiKey()` 从 key 反查连接时提取，无法跨环境拉取。

3.  **两个 API 入口职责分离**：
    - `/api/features/:key`：公开、无认证、SDK 客户端主入口
    - `/api/v1/sdk-payload/:key`：需认证、管理端/服务端使用

4.  **控制台到后端完整闭环**：
    - 创建：`SDKConnectionForm.tsx` → `POST /sdk-connections` → `postSDKConnection` → `createSDKConnection` → `queueSDKPayloadRefresh`
    - 更新：`SDKConnectionForm.tsx` → `PUT /sdk-connections/:id` → `putSDKConnection` → `editSDKConnection` → 如配置变更则 `queueSDKPayloadRefresh`
    - 删除：`sdks/[sdkid].tsx` → `DELETE /sdk-connections/:id` → `deleteSDKConnection` → `deleteSDKConnectionModel`

5.  **⚠️ 创建阶段存在双重刷新**：SDK 连接创建时，**模型层**（`SdkConnectionModel.ts:281`）和**控制器层**（`sdk-connection.controller.ts:81`）都独立调用了 `queueSDKPayloadRefresh`，导致同一次创建触发两次完整的 payload 刷新。

6.  **无去重保障机制**：`queueSDKPayloadRefresh` 直接调用 `refreshSDKPayloadCache`，没有内存去重、任务队列、防抖/节流或乐观锁机制。

7.  **双重刷新的影响**：
    - ✅ API 响应时延无影响（fire-and-forget）
    - ✅ 数据一致性无影响（两次结果相同）
    - ❌ 资源显著浪费（重复查询 features/experiments/savedGroups/holdouts 等）
    - ❌ 数据库负载翻倍（两次查询 + 两次 upsert）
    - ⚠️ 高并发场景下可能加剧数据库压力

8.  **下游链路去重保障矩阵**：
    - ✅ 普通 SDK Webhook：有去重，去重键 `data.webhookId`（Agenda unique）
    - ❌ 全局 Webhook：无去重，会收到两次完全相同的通知
    - ✅ Proxy 更新：有去重，去重键 `data.connectionId + data.useCloudProxy`
    - ✅ Legacy Webhook：有去重，去重键 `webhookId`
    - ❌ CDN Purge：无去重，但 Fastly API 幂等，无功能影响

9.  **下游实际风险**：
    - ⚠️ 全局 Webhook 接收方需要处理重复事件
    - ❌ CDN purge API 调用翻倍（但功能上无影响）
    - ✅ 其他链路通过 Agenda unique 索引避免重复执行

10. **`queueSDKPayloadRefresh` 触发矩阵**：覆盖 12+ 个模块，包括 Feature、SDK Connection、Experiment、Project、Custom Field、Saved Group、Holdout、URL Redirect、Visual Changeset、Environment、Safe Rollout。

11. **缓存键 = 连接 Key**：每个 SDK 连接有独立的缓存，以 `connection.key` 为键存储在 `sdkcache` 集合。

12. **缓存失效 = 覆盖更新**：没有显式删除缓存的操作，通过 `queueSDKPayloadRefresh()` 触发 `upsert` 覆盖更新。

13. **密钥吊销 = 删除连接**：删除连接记录后，`findSDKConnectionByKey()` 会抛出 "Invalid API Key" 错误，缓存记录保留但无法访问。

14. **SSE 连接独立**：`/sub/:key` 端点由独立代理服务处理，推送 `features-updated`（通知重拉）或 `features`（直接推送数据）事件。

15. **客户端重试机制**：SSE 连接错误采用指数退避（`3^(errors-3) * (1000 + random(0-1000))` ms），最大 5 分钟间隔。
