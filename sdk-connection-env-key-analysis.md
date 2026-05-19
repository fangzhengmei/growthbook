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
| `encryptionKey` | 创建 SDK 连接时 | AES-CBC 128 位密钥，加密载荷内容 | `sdkconnections` 集合 | `SdkConnectionModel.ts:242-252` |
| `proxy.signingKey` | 创建 SDK 连接时 | 代理签名密钥 | `sdkconnections` 集合 | `SdkConnectionModel.ts:246` |

### 2.2 密钥生成代码

**SDK 连接 Key 生成** `SdkConnectionModel.ts:217-221`

```typescript
function generateSDKConnectionKey() {
  return generateSigningKey("sdk-", 12);
}
```

- 前缀固定为 `sdk-`，用于 `getPayloadParamsFromApiKey()` 中通过正则 `/^sdk-/` 识别

**加密密钥生成** `SdkConnectionModel.ts:242-252`

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

### 2.3 强绑定关系 `SdkConnectionInterface` `sdk-connection.d.ts:66-105`

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

### 2.4 密钥吊销（删除连接）`SdkConnectionModel.ts:451-461`

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

### 2.5 吊销验证 `features.ts:326-329`

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

## 三、载荷构造与缓存

### 3.1 缓存键设计

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

`features.ts:427`

```typescript
const cached = await context.models.sdkConnectionCache.getById(params.key);
```

`features.ts:802-806`

```typescript
await context.models.sdkConnectionCache.upsert(
  connection.key,  // 缓存键
  JSON.stringify(contents),
  auditContext,
);
```

### 3.2 载荷构造流程

`getFeatureDefinitionsWithCache()` `features.ts:415-501`

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

### 3.3 缓存失效触发条件

**主动失效**：通过 `queueSDKPayloadRefresh()` 触发缓存刷新，**不是删除缓存，而是覆盖更新**

`features.ts:607-621`

```typescript
export function queueSDKPayloadRefresh(data: {
  context: ReqContext | ApiReqContext;
  payloadKeys: SDKPayloadKey[];
  sdkConnections?: SDKConnectionInterface[];
  // ...
}) {
  refreshSDKPayloadCache({ ...data, stackTrace }).catch((e) => {
    logger.error(e, "Error refreshing SDK Payload Cache");
  });
}
```

**触发时机**（调用 `queueSDKPayloadRefresh` 的场景）：

| 触发场景 | 代码位置 | payloadKeys 参数 |
|---------|---------|-----------------|
| 功能发布/变更 | `postFeature.ts`, `editFeature.ts`, `toggleFeature.ts` 等 | 受影响的 `{ environment, project }` 组合 |
| SDK 连接配置变更（环境/项目/加密等） | `SdkConnectionModel.ts:411-427` | 空数组，直接传入 `sdkConnections` |
| 实验启动/停止 | `experiment-feature.ts` | 受影响的环境 + 项目 |

**匹配逻辑** `features.ts:728-740`

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

### 3.4 加密载荷构造 `features.ts:843-960`

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

## 四、客户端拉取与重连

### 4.1 拉取入口 `feature-repository.ts:50-54`

```typescript
fetchFeaturesCall: ({ host, clientKey, headers }) => {
  return (polyfills.fetch as typeof globalThis.fetch)(
    `${host}/api/features/${clientKey}`,
    { headers },
  );
},
```

### 4.2 SSE 实时更新连接 `feature-repository.ts:67-74`

```typescript
eventSourceCall: ({ host, clientKey, headers }) => {
  if (headers) {
    return new polyfills.EventSource(`${host}/sub/${clientKey}`, { headers });
  }
  return new polyfills.EventSource(`${host}/sub/${clientKey}`);
},
```

> **SSE 端点 `/sub/:key`**：由独立的代理/边缘服务处理，不在主后端 app.ts 路由中。

### 4.3 拉取策略 `feature-repository.ts:384-446`

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

### 4.4 SSE 事件处理 `feature-repository.ts:473-496`

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

### 4.5 错误重试与退避 `feature-repository.ts:505-521`

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

### 4.6 客户端解密 `GrowthBookClient.ts`

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

## 五、完整链路图

```
┌─────────────┐  1. 创建连接  ┌───────────────────────────────────┐
│ 控制台 UI   │──────────────▶│ 后端 /api/v1/sdk-connections      │
│ (SDK 连接)  │               │  - generateSDKConnectionKey()     │
└─────────────┘               │  - generateEncryptionKey()        │
       │                      │  - 写入 sdkconnections 集合        │
       │                      └───────────────────────────────────┘
       │                                         │
       │ 5. 功能变更（发布/编辑）                │
       ▼                                         │
┌─────────────┐  触发刷新  ┌───────────────────────────────────┐
│ 控制台 UI   │──────────▶│ queueSDKPayloadRefresh()           │
│ (功能管理)  │            │  - 匹配受影响的 SDK 连接            │
└─────────────┘            │  - buildSDKPayloadForConnection()  │
                           │  - upsert 到 sdkcache 集合          │
                           └───────────────────────────────────┘
                                                                       
                                                                       
                           ┌───────────────────────────────────┐
                           │ 客户端 SDK                        │
                           │  ┌─────────────────────────────┐  │
                           │  │ 2. GET /api/features/:key   │  │
                           │  │    getFeatureDefinitions... │  │
                           │  │    命中缓存 → 直接返回        │  │
                           │  │    未命中 → 实时生成 → 缓存   │  │
                           │  └─────────────────────────────┘  │
                           │  ┌─────────────────────────────┐  │
                           │  │ 3. EventSource /sub/:key    │  │
                           │  │    features-updated → 重拉   │  │
                           │  │    features → 直接更新       │  │
                           │  │    错误 → 指数退避重试        │  │
                           │  └─────────────────────────────┘  │
                           │  ┌─────────────────────────────┐  │
                           │  │ 4. setPayload()             │  │
                           │  │    decryptPayload()         │  │
                           │  └─────────────────────────────┘  │
                           └───────────────────────────────────┘
```

---

## 六、核心代码文件索引

| 模块 | 文件路径 | 关键函数 |
|------|---------|---------|
| 类型定义 | `packages/shared/types/sdk-connection.d.ts` | `SDKConnectionInterface` |
| 后端模型 | `packages/back-end/src/models/SdkConnectionModel.ts` | `generateSDKConnectionKey`, `generateEncryptionKey`, `deleteSDKConnectionModel`, `updateSDKConnectionModel` |
| 缓存模型 | `packages/back-end/src/models/SdkConnectionCacheModel.ts` | `getById`, `upsert` |
| 载荷生成 | `packages/back-end/src/services/features.ts` | `queueSDKPayloadRefresh`, `refreshSDKPayloadCache`, `buildSDKPayloadForConnection`, `getFeatureDefinitionsResponse` |
| 控制器 | `packages/back-end/src/controllers/features.ts` | `getFeatureDefinitionsWithCache`, `getPayloadParamsFromApiKey`, `getFeaturesPublic` |
| API 路由 | `packages/back-end/src/api/sdk-payload/getSdkPayload.ts` | `getSdkPayload` |
| 客户端拉取 | `packages/sdk-js/src/feature-repository.ts` | `fetchFeatures`, `startAutoRefresh`, `onSSEError`, `enableChannel`, `disableChannel` |
| 客户端解密 | `packages/sdk-js/src/GrowthBookClient.ts` | `setPayload` |
| 前端 UI | `packages/front-end/components/Features/SDKConnections/SDKConnectionForm.tsx` | SDK 连接创建/编辑表单 |

---

## 七、关键结论

1.  **环境强绑定**：`SDKConnectionInterface.environment` 在创建时确定，`getPayloadParamsFromApiKey()` 从 key 反查连接时提取，无法跨环境拉取。

2.  **两个 API 入口职责分离**：
    - `/api/features/:key`：公开、无认证、SDK 客户端主入口
    - `/api/v1/sdk-payload/:key`：需认证、管理端/服务端使用

3.  **缓存键 = 连接 Key**：每个 SDK 连接有独立的缓存，以 `connection.key` 为键存储在 `sdkcache` 集合。

4.  **缓存失效 = 覆盖更新**：没有显式删除缓存的操作，通过 `queueSDKPayloadRefresh()` 触发 `upsert` 覆盖更新。

5.  **密钥吊销 = 删除连接**：删除连接记录后，`findSDKConnectionByKey()` 会抛出 "Invalid API Key" 错误，缓存记录保留但无法访问。

6.  **SSE 连接独立**：`/sub/:key` 端点由独立代理服务处理，推送 `features-updated`（通知重拉）或 `features`（直接推送数据）事件。

7.  **客户端重试机制**：SSE 连接错误采用指数退避（3^n * 1s），最大 5 分钟间隔。
