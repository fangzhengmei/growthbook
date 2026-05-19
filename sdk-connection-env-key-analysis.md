# SDK 连接发布载荷与环境密钥强绑定机制分析

## 概述

GrowthBook SDK 连接机制中，发布载荷（SDK Payload）与所属环境密钥强绑定。整个链路分为三段：

1. **密钥生成与吊销** —— 控制台创建/编辑/删除 SDK 连接时生成唯一 `key` 和 `encryptionKey`

2. **载荷构造与缓存** —— 后端根据连接配置（环境、项目、加密选项等）生成并缓存 SDK 载荷

3. **客户端拉取与重连** —— SDK 客户端使用 `clientKey` 拉取载荷、SSE 实时更新

---

## 一、密钥生成与吊销

### 1.1 密钥体系设计

| 密钥类型 | 生成时机 | 用途 | 存储位置 | 代码位置

--- | --- | --- | --- | ---

`key` (SDK 连接 Key | 创建 SDK 连接时 | 客户端请求载荷的唯一标识，URL 路径参数 | `sdkconnections` 集合 | `SdkConnectionModel.ts:217-221`

`encryptionKey` | 创建 SDK 连接时 | AES- CBC 128 位密钥，加密载荷内容 | `sdkconnections` 集合 | `api-key.util.ts:50-62`

`proxy.signingKey` | 创建 SDK 连接时 | 代理签名密钥 | `sdkconnections` 集合 | `SdkConnectionModel.ts:246`

### 1.2 密钥生成代码

**SDK 连接 Key 生成 `SdkConnectionModel.ts:217-221`

```typescript
function generateSDKConnectionKey() {

  return generateSigningKey("sdk-", 12);

}
```

- 前缀固定为 `sdk-`，用于 API 路由匹配 `/sdk-payload/:key` 中通过正则 `/^sdk-/` 识别

**加密密钥生成 `api-key.util.ts:50-62`

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

### 1.3 密钥绑定关系 `SdkConnectionInterface` 中强绑定字段 `sdk-connection.d.ts:66-105`

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

  // ... 其他配置

}
```

### 1.4 密钥吊销（删除连接）

删除 SDK 连接时，后端删除 `deleteSDKConnectionModel` 函数 `SdkConnectionModel.ts:451-461`：

```typescript
export async function deleteSDKConnectionModel(

  context: ReqContext,

  sdkConnection: SDKConnectionInterface,

) {

  await SDKConnectionModel.deleteOne({

    organization: context.org.id,

    id: sdkConnection.organization,

    id: sdkConnection.id,

  });

}
```

删除后，该 `key` 失效，客户端将无法拉取载荷。

---

## 二、载荷构造与缓存

### 2.1 载荷构造入口

**API 路由 `sdk-payload.router.ts`

```typescript
// GET /sdk-payload/:key
```

**请求处理流程 `getSdkPayload.ts:19-46`

```typescript
export const getSdkPayload = createApiRequestHandler({

  paramsSchema: z.object({ key: z.string() }),

  method: "get",

  path: "/sdk-payload/:key",

})(async (req): Promise<FeatureDefinitionSDKPayload & { status: number }> {

  const { key } = req.params;

  const params = await getPayloadParamsFromApiKey(key, req);

  const defs = await getFeatureDefinitionsWithCache({

    context: req.context,

    params,

  });

  return { status: 200, ...defs };

});
```

### 2.2 API Key 解析与环境绑定 `controllers/features.ts:321-410`

```typescript
export async function getPayloadParamsFromApiKey(

  key: string,

  req: Request,

): Promise<SDKPayloadParams> {

  // 匹配 SDK 连接 Key (sdk- 前缀) {

    const connection = await findSDKConnectionByKey(key);

    if (!connection) throw new UnrecoverableApiError("Invalid API Key");

    if (!connection.connected) {

      markSDKConnectionUsed(key);

    }

    return {

      key: connection.key,

      organization: connection.organization,

      environment: connection.environment,

      projects: connection.projects,

      encryptPayload: connection.encryptPayload,

      encryptionKey: connection.encryptionKey,

      // ... 其他配置

    };

  }

}
```

**关键点：**

- 通过 `key` 反查 `SDKConnection` 文档

- 从文档中提取 `environment`、`projects`、`encryptPayload`、`encryptionKey` 等绑定字段

- 首次使用时标记 `connected: true`

### 2.3 载荷构造核心 `services/features.ts:1052-1224`

```typescript
export async function buildSDKPayloadForConnection(

  input: SDKPayloadBuildInput,

): Promise<FeatureDefinitionSDKPayload> {

  const { connection, data } = input;

  // 1. 按项目过滤 features、experiments

  // 2. 生成 features 定义

  const featureDefinitions = generateFeaturesPayload({

    features: filteredFeatures,

    environment,

    // ...

  });

  // 3. 生成自动 experiments 定义

  const experimentsDefinitions = generateAutoExperimentsPayload({

    // ...

  });

  // 4. 加密（如配置）

  return getFeatureDefinitionsResponse({

    features: featuresWithHoldouts,

    experiments: experimentsDefinitions,

    encryptPayload: connection.encryptPayload,

    encryptionKey: connection.encryptionKey,

    // ...

  });

}
```

### 2.4 加密逻辑 `services/features.ts:843-968`

```typescript
export async function getFeatureDefinitionsResponse({

  features,

  encryptPayload,

  encryptionKey,

  // ...

}): Promise<{

  features: Record<string, FeatureDefinition>;

  encryptedFeatures?: string;

}> {

  if (!encryptPayload || !encryptionKey) {

    return { features, dateUpdated };

  }

  const encryptedFeatures = await encrypt(

    JSON.stringify(features),

    encryptionKey,

  );

  return {

    features: {},

    encryptedFeatures,

  };

}
```

**加密算法：`AES-CBC` 128 位，密钥来自 `encryptionKey`

### 2.5 缓存机制 `controllers/features.ts:414-501`

```typescript
export async function getFeatureDefinitionsWithCache({

  context,

  params,

}): Promise<FeatureDefinitionSDKPayload> {

  // 1. 尝试从缓存读取

  const cached = await context.models.sdkConnectionCache.getById(params.key);

  if (cached) {

    try {

      defs = JSON.parse(cached.contents);

    } catch (e) {

      // 解析失败，重新生成

    }

  }

  // 2. 缓存未命中，实时生成

  if (!defs) {

    defs = await getFeatureDefinitions({ /* ... */ });

    // 回写缓存

    context.models.sdkConnectionCache.upsert(params.key, JSON.stringify(defs));

  }

  return defs;

}
```

**缓存键：** `connection.key`（即 `sdk-xxxx）

**缓存存储：`sdkconnectioncaches` 集合

### 2.6 缓存刷新触发时机 `services/features.ts:607-826`

```typescript
export function queueSDKPayloadRefresh(data: {

  context,

  payloadKeys: SDKPayloadKey[],

  sdkConnections?: SDKConnectionInterface[],

}) {

  refreshSDKPayloadCache({ ...data });

}
```

**刷新触发场景：

- 创建/更新 SDK 连接时

- 发布 Feature 变更时

- 实验变更时

---

## 三、客户端拉取与重连

### 3.1 客户端初始化 `GrowthBookClient.ts:124-147`

```typescript
public async init(options?: InitOptions): Promise<InitResponse> {

  if (options.payload) {

    await this.setPayload(options.payload);

    startStreaming(this, options);

    return { success: true, source: "init" };

  } else {

    const { data } = await this._refresh({

      ...options,

      allowStale: true,

    });

    startStreaming(this, options);

    await this.setPayload(data || {});

    return res;

  }

}
```

### 3.2 拉取请求构造 `feature-repository.ts:50-55`

```typescript
export const helpers: Helpers = {

  fetchFeaturesCall: ({ host, clientKey, headers }) => {

    return fetch(`${host}/api/features/${clientKey}`, { headers });

  },

};
```

**请求 URL 格式：`${apiHost}/api/features/${clientKey}`

`clientKey` 即连接的 `key` 字段

### 3.3 缓存策略 `feature-repository.ts:206-256`

```typescript
async function fetchFeaturesWithCache({

  instance,

  allowStale,

  timeout,

  skipCache,

}): Promise<FetchResponse> {

  // 1. 检查内存缓存

  const existing = cache.get(cacheKey);

  if (existing && (allowStale || existing.staleAt > now) {

    // 缓存命中，后台静默刷新

    if (existing.staleAt < now) {

      fetchFeatures(instance);

    }

    return { data: existing.data, success: true, source: "cache" };

  }

  // 2. 缓存未命中，网络请求

  const res = await promiseTimeout(fetchFeatures(instance), timeout);

  return res;

}
```

**缓存配置：**

- `staleTTL: 60 秒（1 分钟后视为过期）

- `maxAge: 4 小时（最大缓存时间）

- `maxEntries: 10`（最多缓存 10 个不同的 key）

### 3.4 SSE 实时更新 `feature-repository.ts:449-503`

```typescript
function startAutoRefresh(

  instance,

  forceSSE: boolean = false,

): void {

  if (cacheSettings.backgroundSync &&

      supportsSSE.has(key) &&

      polyfills.EventSource) {

    const channel: ScopedChannel = {

      src: helpers.eventSourceCall({

        host: streamingHost,

        clientKey,

      }),

      cb: (event: MessageEvent<string>) => {

        if (event.type === "features-updated") {

          fetchFeatures(instance);

        } else if (event.type === "features") {

          const json = JSON.parse(event.data);

          onNewFeatureData(key, cacheKey, json);

        }

      },

    };

    streams.set(key, channel);

  }

}
```

**SSE 连接路径：`${streamingHost}/sub/${clientKey}`

**事件类型：

- `features-updated` — 通知有更新，触发重新拉取

- `features` — 直接推送新的载荷数据

### 3.5 错误重试机制 `feature-repository.ts:505-521`

```typescript
function onSSEError(channel: ScopedChannel) {

  channel.errors++;

  if (channel.errors > 3) {

    // 指数退避

    const delay = Math.pow(3, channel.errors - 3) * (1000 + Math.random() * 1000);

    disableChannel(channel);

    setTimeout(() => enableChannel(channel), Math.min(delay, 300000));

  }

}
```

**重试策略：

- 连续 3 次错误后开始退避

- 指数退避：3^(n-3) * 随机延迟

- 最大延迟 5 分钟

### 3.6 解密逻辑 `core.ts:1172-1205`

```typescript
export async function decryptPayload(

  data: FeatureApiResponse,

  decryptionKey: string | undefined,

): Promise<FeatureApiResponse> {

  if (data.encryptedFeatures && decryptionKey) {

    const featuresJSON = await decrypt(

      data.encryptedFeatures,

      decryptionKey,

    );

    data.features = JSON.parse(featuresJSON);

    delete data.encryptedFeatures;

  }

  // 同理处理 encryptedExperiments、encryptedSavedGroups

  return data;

}
```

`GrowthBookClient.ts:85-99`

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

## 四、三段链路拼合图

```
控制台 (前端)                          后端)                          SDK 客户端)

┌─────────────────┐          ┌────────────────────────┐          ┌──────────────────┐

│ SDKConnectionForm  │          │  创建/更新/删除

│  - 选择环境          │ POST/PUT /sdk-connections    │          │  GrowthBook.init()

│  - 选择项目          │  SdkConnectionModel     │          │  - clientKey

│  - 加密选项          │  - 生成 key         │          │  - decryptionKey   │

│  - 其他配置          │  - 生成 encryptionKey│          │                │

└────────┬────────┘          └──────────┬─────────────┘          └───────┬────────┘

         │                             │                              │

         │                             │                              │

         │                             │ 1. init()                      │

         │                             │                              │

         │                             │ ┌───────────────────────────┐      │

         │                             │ │ GET /sdk-payload/:key    │      │

         │                             │ │ getPayloadParamsFromApiKey│      │

         │                             │ │  - 反查 SDKConnection  │      │

         │                             │ │  - 提取 environment  │      │

         │                             │ │  - 提取 encryptionKey │      │

         │                             │ └───────────┬───────────────┘      │

         │                             │             │                      │

         │                             │             ▼                      │

         │                             │  getFeatureDefinitionsWithCache │

         │                             │  - 查缓存 → 未命中 → 生成 → 回写        │

         │                             │  buildSDKPayloadForConnection        │

         │                             │  - 按 environment/projects 过滤            │

         │                             │  - 加密 (encryptPayload)            │

         │                             │  - 返回 { features, encrypted* }   │

         │                             │             │                      │

         │                             │             ▼                      │

         │                             │  响应载荷 (可能加密)                 │

         │                             └──────────────┬──────────────────────┘

         │                                            │

         │                                            │ 2. 客户端接收

         │                                            │

         │                                            ▼

         │                                     setPayload()

         │                                     decryptPayload()

         │                                     - 使用 decryptionKey 解密

         │                                     - 解析 features

         │

         │                                            │

         │                                            │ 3. SSE 订阅

         │                                            │

         │                                            ▼

         │                                     startStreaming()

         │                                     EventSource /sub/:key

         │                                     - features-updated → 重新拉取

         │                                     - features → 直接更新

         │                                            │

         │                                            │

         ▼                                            ▼

 功能发布变更                          变更 → queueSDKPayloadRefresh()

  (features/experiments 变更

  → 刷新缓存

  → 触发 SSE 推送
```

---

## 五、关键绑定关系总结

### 5.1 环境绑定字段

| 层级 | 绑定字段 | 说明 | 代码位置

--- | --- | --- | ---

连接层 | `environment` | 所属环境，只能拉取该环境的 features | `sdk-connection.d.ts:80`

| `projects` | 所属项目列表，过滤 features | `sdk-connection.d.ts:81`

加密层 | `encryptPayload` | 是否加密载荷 | `sdk-connection.d.ts:82`

| `encryptionKey` | AES 加密密钥 | `sdk-connection.d.ts:83`

标识层 | `key` | 客户端拉取的唯一标识 | `sdk-connection.d.ts:96`

### 5.2 安全边界

- 每个 SDK 连接与环境、项目强绑定，无法跨环境拉取

- 加密密钥与连接一一对应，更换连接必须更换密钥

- 删除连接即吊销密钥，客户端永久失效

- 载荷缓存以 `key` 为键，环境/连接变更自动失效

### 5.3 性能优化点

1. **服务端缓存：`sdkconnectioncaches` 集合缓存预先生成

2. **CDN 缓存：`Cache-control: max-age=30, stale-while-revalidate=3600`

3. **客户端缓存：内存 + localStorage 双层缓存

4. **SSE 实时更新：减少轮询开销

---

## 六、核心文件索引

| 模块 | 文件路径 | 主要职责

--- | --- | ---

类型定义 | `packages/shared/types/sdk-connection.d.ts` | SDKConnectionInterface 定义

| `packages/shared/types/apikey.d.ts` | API Key 类型

后端模型 | `packages/back-end/src/models/SdkConnectionModel.ts` | 连接 CRUD、密钥生成

| `packages/back-end/src/models/SdkConnectionCacheModel.ts` | 载荷缓存模型

后端服务 | `packages/back-end/src/services/features.ts` | 载荷构造、刷新、加密

后端控制器 | `packages/back-end/src/controllers/features.ts` | API Key 解析、缓存读取

后端路由 | `packages/back-end/src/api/sdk-payload/getSdkPayload.ts` | 拉取 API 入口

前端 UI | `packages/front-end/components/Features/SDKConnections/SDKConnectionForm.tsx` | 连接创建表单

SDK 客户端 | `packages/sdk-js/src/GrowthBookClient.ts` | 客户端主类

| `packages/sdk-js/src/feature-repository.ts` | 拉取、SSE、缓存
