# SDK Webhook 投递与重试代码分析

## 概述

GrowthBook 代码库中存在 **5 条投递通道**，各自有不同的重试策略、payload 获取方式和执行时序：

| 体系 | Agenda 任务名 | 适用场景 | 核心文件 | 最大重试次数 | 总执行次数 |
|------|--------------|----------|----------|-------------|-----------|
| Legacy SDK Webhook | `fireWebhook` | 旧版 SDK 连接 | `jobs/webhooks.ts` | 2 | 3 |
| SDK Webhook (新版) | `fireWebhooks` | 新版 SDK 连接 | `jobs/sdkWebhooks.ts` | 2 | 3 |
| Event Webhook | `eventWebHook` | 系统事件通知 | `events/handlers/webhooks/EventWebHookNotifier.ts` | 3 | 4 |
| Global SDK Webhook | *(无 Agenda)* | 环境变量全局配置 | `jobs/sdkWebhooks.ts:fireGlobalSdkWebhooks` | 0 | 1 |
| Proxy Update | `proxyUpdate` | SDK 代理推送 | `jobs/proxyUpdate.ts` | 1 | 2 |

> **术语约定**：本文严格区分"最大重试次数"（首次执行失败后的额外执行次数）与"总执行次数"（首次 + 重试）。源码注释中的 "If it failed 3 times, give up" 指的是总执行 3 次（即重试 2 次后放弃），而非重试 3 次。

---

## 一、事件入队流程

### 1.1 触发源与完整异步调用链

Feature 变更时，完整调用链是 **5 层 fire-and-forget** 的后台任务链，**每一层都不阻塞上层**：

```
Express 路由处理器
    ↓
controllers/features.ts:2433  await updateFeature(...)
    ↓
models/FeatureModel.ts:1053  onFeatureUpdate(...).catch(logger.error)  ← 第 1 层 fire-and-forget
    ↓
models/FeatureModel.ts:927  queueSDKPayloadRefresh(...)  ← 同步函数，立即返回
    ↓
services/features.ts:618  refreshSDKPayloadCache(...).catch(logger.error)  ← 第 2 层 fire-and-forget
    ↓
services/features.ts:821  triggerWebhookJobs(...).catch(logger.error)  ← 第 3 层 fire-and-forget
    ↓
jobs/updateAllJobs.ts:17-58  triggerWebhookJobs 内部:
    ├─ 4 个 webhook/proxy 通道（均 .catch() 不 await）
    └─ await purgeCDNCache()  ← 仅在 triggerWebhookJobs 内部 await，不影响接口响应
```

**关键异步证据**（每层对应代码）：

```typescript
// 第 1 层：FeatureModel.ts:1053 — onFeatureUpdate 不 await
onFeatureUpdate(context, feature, updatedFeature).catch((e) => {
  logger.error(e, "Error refreshing SDK Payload on feature update");
});

// 第 2 层：services/features.ts:618 — refreshSDKPayloadCache 不 await
refreshSDKPayloadCache({ ...data, stackTrace }).catch((e) => {
  logger.error(e, "Error refreshing SDK Payload Cache");
});

// 第 3 层：services/features.ts:821 — triggerWebhookJobs 不 await
triggerWebhookJobs(context, payloadKeys, connectionsUpdated, true).catch(
  (e) => {
    logger.error(e, "Error triggering webhook jobs");
  }
);
```

**结论**：`triggerWebhookJobs` 内部的 `await purgeCDNCache` 仅等待 CDN 清除完成，但由于 `triggerWebhookJobs` 本身是 fire-and-forget 调用，**CDN 清除和所有 webhook 投递都在后台异步执行，不阻塞 Feature 更新 API 的响应**。接口响应延迟由 Mongo 操作（`FeatureModel.updateOne`、`FeatureModel.findOne`）主导。

### 1.2 统一入队分发器 — triggerWebhookJobs

`jobs/updateAllJobs.ts:17-58` 中 `triggerWebhookJobs` 的完整执行逻辑：

```typescript
export const triggerWebhookJobs = async (
  context, payloadKeys, connections, isProxyEnabled, isFeature = true,
) => {
  // ① 异步入队，不等待完成
  queueWebhooksByConnections(context, connections).catch(...);

  // ② 同步发起（内部异步），不等待完成
  fireGlobalSdkWebhooks(context, connections).catch(...);

  // ③ 条件入队，不等待完成
  if (isProxyEnabled) {
    queueProxyUpdate(context, connections).catch(...);
  }

  // ④ 异步入队，不等待完成
  queueLegacySdkWebhooks(context, payloadKeys, isFeature).catch(...);

  // ⑤ 同步等待 CDN 缓存清除完成（仅在函数内部等待）
  await purgeCDNCache(context.org.id, surrogateKeys);
};
```

**时序关键点**：步骤 ①-④ 全部是 fire-and-forget（`.catch()` 吞掉错误），不阻塞后续步骤；只有步骤 ⑤ `purgeCDNCache` 使用 `await` 同步等待。因此 **webhook 入队/投递与 CDN 清除之间是并行关系**，CDN 清除不会等 webhook 完成，webhook 也不会等 CDN。

### 1.3 新版 SDK Webhook 入队 (`sdkWebhooks.ts:114-125`)

```typescript
export async function queueWebhooksByConnections(context, connections) {
  const sdkKeys = connections.map((c) => c.id);
  const webhooks =
    await context.models.sdkWebhooks.findAllSdkWebhooksByConnectionIds(sdkKeys);
  for (const webhook of webhooks) {
    if (webhook && !webhook.disabled) await queueSingleSdkWebhookJob(webhook);
  }
}
```

入队时使用 `job.unique({ "data.webhookId": webhook.id })` 防止同一 webhook 重复入队。

### 1.4 旧版 SDK Webhook 入队 (`webhooks.ts:139-180`)

过滤逻辑：根据 `payloadKeys` 匹配 webhook 的 `project` 和 `environment`，额外跳过 `featuresOnly` 但非 feature 事件、以及 `disabled` 状态的 webhook。

### 1.5 Global SDK Webhook — 无 Agenda 入队

`fireGlobalSdkWebhooks` (`sdkWebhooks.ts:356-421`) **不走 Agenda 队列**，而是直接在当前事件循环中发起 HTTP 请求：

```typescript
for (const connection of connections) {
  const payload = await getFeatureDefinitionsWithCache({context, params: connection});
  WEBHOOKS.forEach((webhook) => {
    runWebhookFetch({...}).catch((e) => {
      logger.error(e, "Failed to fire global webhook");
    });
  });
}
```

每个 connection 的 payload 获取是串行的（`for...of` + `await`），但对每个 WEBHOOK 的 `runWebhookFetch` 调用是并行的（`forEach` + 不 `await`）。

### 1.6 Event Webhook 入队链

Event Webhook 走独立的事件系统：

```
EventModel.create(...)
  → EventNotifier(eventId).perform()
    → Agenda job: "eventCreated"
      → webHooksEventHandler(event)
        → getAllEventWebHooksForEvent(...)
        → EventWebHookNotifier.enqueue()
          → Agenda job: "eventWebHook"
```

### 1.7 Proxy Update 入队 (`proxyUpdate.ts:167-183`)

```typescript
export async function queueProxyUpdate(context, connections) {
  for (const connection of connections) {
    if (IS_CLOUD) {
      await queueSingleProxyUpdate(context.org.id, connection, true);
    }
    await queueSingleProxyUpdate(context.org.id, connection, false);
  }
}
```

每个 connection 可能入队两个 job（cloud proxy + self-hosted proxy），入队间串行。

---

## 二、重试机制差异详解

### 2.1 重试策略对比总表

| 体系 | 最大重试次数 | 第 1 次重试延迟 | 第 2 次重试延迟 | 第 3 次重试延迟 | 重试触发方式 |
|------|-------------|----------------|----------------|----------------|-------------|
| Legacy SDK Webhook | 2 | 30s | 5m | — | Agenda `fail` 事件 |
| SDK Webhook (新版) | 2 | 30s | 5m | — | Agenda `fail` 事件 |
| Event Webhook | 3 | 30s | 5m | 5m | 显式 `retryJob()` 调用 |
| Global SDK Webhook | 0 | — | — | — | 无重试 |
| Proxy Update | 1 | 5s | — | — | Agenda `fail` 事件 |

### 2.2 逐行比对：Legacy SDK / 新版 SDK Webhook — 最多重试 2 次

两套代码的重试逻辑完全一致（`webhooks.ts:110-136` / `sdkWebhooks.ts:75-101`）：

```typescript
agenda.on("fail:" + JOB_NAME, async (error, job) => {
  const retryCount = job.attrs.data.retryCount;
  let nextRunAt = Date.now();

  if (retryCount === 0) {          // 首次失败 → 等待 30s
    nextRunAt += 30000;
  } else if (retryCount === 1) {   // 第 2 次失败 → 等待 5m
    nextRunAt += 300000;
  } else {                          // retryCount >= 2 → 放弃
    return;
  }

  job.attrs.data.retryCount++;
  job.attrs.nextRunAt = new Date(nextRunAt);
  await job.save();
});
```

**执行追踪**：
- 首次执行（`retryCount=0`）失败 → `retryCount` 变为 1，安排 30s 后重试 → **第 1 次重试**
- 第 1 次重试（`retryCount=1`）失败 → `retryCount` 变为 2，安排 5m 后重试 → **第 2 次重试**
- 第 2 次重试（`retryCount=2`）失败 → 进入 `else` 分支，直接 `return` → **不再重试**

因此 **Legacy 和新版 SDK Webhook 最多重试 2 次**，总执行次数 = 1（首次）+ 2（重试）= 3 次。

> 源码注释 `// If it failed 3 times, give up` 中的 "3 times" 指总执行 3 次（含首次），而非重试 3 次。

### 2.3 逐行比对：Event Webhook — 最多重试 3 次

`EventWebHookNotifier.ts:371-389`：

```typescript
private static async retryJob(job) {
  if (job.attrs.data.retryCount >= 3) {
    return;                         // retryCount >= 3 → 放弃
  }

  let nextRunAt = Date.now();
  if (job.attrs.data.retryCount === 0) {
    nextRunAt += 30000;             // 首次失败 → 30s
  } else {
    nextRunAt += 300000;            // 其他 → 5m
  }

  job.attrs.data.retryCount++;
  job.attrs.nextRunAt = new Date(nextRunAt);
  await job.save();
}
```

**执行追踪**：
- 首次执行（`retryCount=0`）失败 → `retryCount` 变为 1，安排 30s → **第 1 次重试**
- 第 1 次重试（`retryCount=1`）失败 → `retryCount` 变为 2，安排 5m → **第 2 次重试**
- 第 2 次重试（`retryCount=2`）失败 → `retryCount` 变为 3，安排 5m → **第 3 次重试**
- 第 3 次重试（`retryCount=3`）失败 → `>= 3`，不再重试

因此 **Event Webhook 最多重试 3 次**，总执行次数 = 1（首次）+ 3（重试）= 4 次。

> 源码注释 `// If it failed 3 times, give up` 中的 "3 times" 指重试 3 次（不含首次），与 Legacy/SDK 的注释含义不同。

**注意**：Event Webhook 的重试不是通过 Agenda `fail` 事件触发的。`handleWebHookError` 在记录日志后显式调用 `retryJob()`，且 `sendDataToWebHook` 在失败时返回 `{ result: "error" }` 而非 `throw`，所以 Agenda 不会自动触发 `fail` 事件——重试完全由业务代码控制。

### 2.4 Global SDK Webhook — 无重试

`fireGlobalSdkWebhooks` 中对 `runWebhookFetch` 的调用：

```typescript
runWebhookFetch({...}).catch((e) => {
  logger.error(e, "Failed to fire global webhook");
});
```

没有 Agenda job 包裹，没有 retryCount，没有 fail 事件监听。失败仅记日志，**不重试**。

### 2.5 Proxy Update — 最多重试 1 次

`proxyUpdate.ts:122-143`：

```typescript
agenda.on("fail:" + PROXY_UPDATE_JOB_NAME, async (error, job) => {
  const retryCount = job.attrs.data.retryCount;
  let nextRunAt = Date.now();

  if (retryCount === 0) {
    nextRunAt += 5000;    // 首次失败 → 5s
  } else {
    return;               // 放弃
  }

  job.attrs.data.retryCount++;
  job.attrs.nextRunAt = new Date(nextRunAt);
  await job.save();
});
```

**最多重试 1 次**，延迟仅 5 秒（远短于 webhook 的 30s），超时也只有 5 秒。

> 源码注释 `// If it failed twice, give up` 中的 "twice" 指总执行 2 次（含首次），即重试 1 次后放弃。

---

## 三、SDK Payload 获取与发送的串并行关系

### 3.1 新版 SDK Webhook (`sdkWebhooks.ts:308-354`)

```
fireSdkWebhook(context, webhook)
│
├─① findSDKConnectionsByIds(webhook.sdks)       ← 查询关联的 SDK 连接
│
├─② BluebirdPromise.reduce(connections, ...)     ← 串行获取每个 connection 的 payload
│     connection[0] → getFeatureDefinitionsWithCache() → [key0, payload0]
│     connection[1] → getFeatureDefinitionsWithCache() → [key1, payload1]
│     ...
│
└─③ BluebirdPromise.each(payloads, ...)          ← 串行发送每个 payload
      runWebhookFetch({key0, payload0}) → HTTP POST
      runWebhookFetch({key1, payload1}) → HTTP POST
      ...
```

**关键**：payload 获取（步骤②）使用 `BluebirdPromise.reduce`，**串行**逐个获取；payload 发送（步骤③）使用 `BluebirdPromise.each`，也是**串行**逐个发送。两者均为串行的原因：避免同时大量调用 `getFeatureDefinitionsWithCache` 和 `cancellableFetch` 导致后端/Mongo 过载。

**一个 webhook 对应多个 SDK 连接时**，会为每个连接独立获取 payload 并独立发送 HTTP 请求，但彼此串行。

### 3.2 旧版 SDK Webhook (`webhooks.ts:25-108`)

```
fireWebhook(job)
│
├─① getFeatureDefinitionsWithCache()    ← 单次获取，使用合成的 cache key
├─② getExperimentOverrides()            ← 条件获取（仅非 featuresOnly）
│
└─③ cancellableFetch()                  ← 单次发送
```

旧版只获取一个 payload、发送一次请求，无串并行问题。

### 3.3 Global SDK Webhook (`sdkWebhooks.ts:356-421`)

```
fireGlobalSdkWebhooks(context, connections)
│
└─ for (connection of connections) {          ← 外层：串行遍历 connection
     │
     ├─① await getFeatureDefinitionsWithCache()   ← 串行获取 payload
     │
     └─② WEBHOOKS.forEach(webhook => {            ← 内层：并行触发所有全局 webhook
          runWebhookFetch({...}).catch(...)             ← 不 await，并行执行
        })
   }
```

**混合模式**：connection 之间串行获取 payload；但对同一个 connection 的多个全局 webhook，发送请求是并行的（`forEach` 不 `await`）。

### 3.4 Event Webhook (`EventWebHookNotifier.ts`)

每个 EventWebHook 只发送一次请求，payload 在 job handler 内按 `payloadType` 就地构建，无串并行问题。

### 3.5 Proxy Update (`proxyUpdate.ts`)

单个 connection 只获取一个 payload 并发送一次，无串并行问题。

### 3.6 串并行关系总结

| 体系 | Payload 获取 | HTTP 发送 | 说明 |
|------|-------------|----------|------|
| 新版 SDK Webhook | **串行** (reduce) | **串行** (each) | 每个连接独立 payload+请求 |
| 旧版 SDK Webhook | 单次 | 单次 | 一个 webhook 一个 payload |
| Global SDK Webhook | **串行** (for…of) | **并行** (forEach) | 同一 payload 发 N 个全局 webhook |
| Event Webhook | 就地构建 | 单次 | — |
| Proxy Update | 单次 | 单次 | — |

---

## 四、triggerWebhookJobs 的完整执行时序

### 4.1 时序图

```
triggerWebhookJobs() 被调用（自身是 fire-and-forget，不阻塞上层）
│
├─ [fire-and-forget] queueWebhooksByConnections()
│    └─→ 串行遍历 webhooks，逐个 agenda.create("fireWebhooks").save()
│         （仅入队，实际执行由 Agenda 调度）
│
├─ [fire-and-forget] fireGlobalSdkWebhooks()
│    └─→ 串行获取 payload，并行 runWebhookFetch()（立即发起 HTTP 请求）
│         （不经过 Agenda，在当前事件循环中直接执行）
│
├─ [fire-and-forget] queueProxyUpdate()          ← 仅当 isProxyEnabled
│    └─→ 串行遍历 connections，逐个 agenda.create("proxyUpdate").save()
│         （仅入队，实际执行由 Agenda 调度）
│
├─ [fire-and-forget] queueLegacySdkWebhooks()
│    └─→ 串行遍历 webhooks，逐个 agenda.create("fireWebhook").save()
│         （仅入队，实际执行由 Agenda 调度）
│
└─ [await] purgeCDNCache()                       ← 仅在 triggerWebhookJobs 内部等待
     └─→ 向 Fastly API 发 POST 清除 surrogate keys
          （批量处理，每批最多 256 个 key）
```

### 4.2 时序关键结论

1. **webhook 入队与 CDN 清除并行**：步骤 ①-④ 全部 fire-and-forget，`purgeCDNCache` 不等它们完成就开始执行，反之亦然。

2. **Global Webhook 是唯一同步投递（但仍不阻塞接口）**：`fireGlobalSdkWebhooks` 不走 Agenda，在 `triggerWebhookJobs` 的调用栈内直接发起 HTTP 请求。但由于 `triggerWebhookJobs` 本身被 `.catch()` 调用（fire-and-forget），所以它也不会阻塞 Feature 更新的主流程。

3. **Proxy Update 的条件性**：只有 `isProxyEnabled=true` 时才入队 proxy 更新。Cloud 用户还会额外入队一个 cloud proxy 更新 job。

4. **CDN 清除的容错性**：`purgeCDNCache` 内部按 256 个 key 一批调用 Fastly API，`catch` 块仅记日志不抛出异常，不会影响整个 `triggerWebhookJobs` 的完成。

5. **入队顺序无保证**：四个入队/投递操作虽然代码上是顺序执行，但各自是异步的。Agenda job 的实际执行顺序取决于 Agenda 调度器的并发度和队列状态。

6. **整个后台链路不阻塞接口**：从 `onFeatureUpdate` 开始的所有后台操作（cache 更新、CDN 清除、webhook 投递）都是异步执行的，不影响 Feature 更新 API 的响应时间。

---

## 五、签名生成机制

### 5.1 新版 SDK Webhook 双重签名 (`sdkWebhooks.ts:156-230`)

**签名 1 - `webhook-secret`**：
```typescript
const signature = createHmac("sha256", signingKey)
  .update(sendPayload ? jsonPayload : "")
  .digest("hex");
const secret = `whsec_${signature}`;
```

**签名 2 - `webhook-signature`**（Svix 兼容格式）：
```typescript
const standardSignatureBody = `${webhookID}.${timestamp}.${body || ""}`;
const standardSignature =
  "v1," +
  createHmac("sha256", signingKey)
    .update(standardSignatureBody)
    .digest("base64");
```

请求头：
```
webhook-id: msg_xxx
webhook-timestamp: 1620000000
webhook-signature: v1,abc123...
webhook-secret: whsec_xyz789...
webhook-sdk-key: sdk-abc123
```

### 5.2 旧版 SDK Webhook 单一签名 (`webhooks.ts:78-80`)

```typescript
const signature = createHmac("sha256", webhook.signingKey)
  .update(payload).digest("hex");
```

请求头：`X-GrowthBook-Signature`

### 5.3 Event Webhook 签名 (`event-webhooks-utils.ts:39-49`)

```typescript
const requestPayload = JSON.stringify(payload);
return createHmac("sha256", signingKey).update(requestPayload).digest("hex");
```

请求头：`X-GrowthBook-Signature`

### 5.4 Proxy Update 签名 (`proxyUpdate.ts:92-94`)

```typescript
const signature = createHmac("sha256", connection.proxy.signingKey)
  .update(payload).digest("hex");
```

请求头：`X-GrowthBook-Signature` + `X-GrowthBook-Api-Key`

---

## 六、失败回退与熔断机制

### 6.1 连续失败熔断 (`WebhookModel.ts:109-131`)

阈值常量 (`shared/constants.ts:261`)：
```typescript
export const WEBHOOK_CONSECUTIVE_FAILURES_THRESHOLD = 10;
```

熔断逻辑：
- 每次投递失败：`consecutiveFailures++`，当达到 10 时设置 `disabled = true`
- 一旦成功：**全部重置**（`error=""`, `lastSuccess=now`, `consecutiveFailures=0`, `disabled=false`）

**注意**：熔断仅影响 SDK Webhook（新版和旧版），Global Webhook 和 Event Webhook 不受此机制约束。

### 6.2 Proxy Update 的熔断 (`proxyUpdate.ts:66-71`)

Proxy Update 也使用相同的 `WEBHOOK_CONSECUTIVE_FAILURES_THRESHOLD = 10` 阈值，但检查位置在 job handler 开头：

```typescript
if ((connection.proxy.consecutiveFailures || 0) >= WEBHOOK_CONSECUTIVE_FAILURES_THRESHOLD) {
  return;  // 连续失败 10 次，跳过执行
}
```

### 6.3 投递日志记录

**SDK Webhook 日志** (`SdkWebHookLogInterface`)：每次投递成功/失败都写入 `webhook-logs` 集合。

**Event Webhook 日志** (`EventWebHookLogModel`)：`handleWebHookSuccess` 和 `handleWebHookError` 各自调用 `createEventWebHookLog`，保证每次投递均有记录。

**Global Webhook 日志**：`runWebhookFetch` 内部也调用 `createSdkWebhookLog`，但因 `global=true` 不会调用 `setLastSdkWebhookError`，所以不影响熔断计数。

---

## 七、完整调用链路图

### 7.1 triggerWebhookJobs 全局时序

```
Feature 变更
    ↓
controllers/features.ts:2433  await updateFeature(...)
    ↓
models/FeatureModel.ts:960  async function updateFeature(...)
    ├─ ① await FeatureModel.updateOne(...)    ← Mongo 写操作（阻塞接口）
    ├─ ② await FeatureModel.findOne(...)      ← Mongo 读操作（阻塞接口）
    ├─ ③ onFeatureUpdate(...).catch(...)      ← fire-and-forget，不阻塞
    └─ ④ return updatedFeature                ← 接口响应返回
         ↓
         onFeatureUpdate 后台异步执行:
         ├─ queueSDKPayloadRefresh(...)
         ├─ refreshSDKPayloadCache(...).catch(...)
         │   ├─ await promiseAllChunks(promises, 4)  ← 更新 SDK connection cache
         │   └─ triggerWebhookJobs(...).catch(...)
         │        ├── [异步] queueWebhooksByConnections()
         │        │    └── Agenda: fireWebhooks → 串行 payload + 串行发送
         │        │         最多重试 2 次: 30s → 5m → 放弃
         │        │
         │        ├── [异步] fireGlobalSdkWebhooks()
         │        │    └── 串行 payload + 并行发送（不重试）
         │        │
         │        ├── [异步, 条件] queueProxyUpdate()
         │        │    └── Agenda: proxyUpdate
         │        │         最多重试 1 次: 5s → 放弃
         │        │
         │        ├── [异步] queueLegacySdkWebhooks()
         │        │    └── Agenda: fireWebhook
         │        │         最多重试 2 次: 30s → 5m → 放弃
         │        │
         │        └── [await] purgeCDNCache()  ← 仅在 triggerWebhookJobs 内等待
         │
         ├─ await logFeatureUpdatedEvent(...)
         └─ await updateVercelExperimentationItemFromFeature(...)
```

### 7.2 Event Webhook 完整链路

```
领域事件发生 (如 feature.created)
    ↓
EventModel.create(eventData)
    ↓
new EventNotifier(eventId).perform()
    ↓
Agenda: eventCreated { eventId }
    ↓
EventNotifier.jobHandler()
    ├── webHooksEventHandler(event)
    │    ├── getAllEventWebHooksForEvent({eventName, enabled:true, tags, projects})
    │    │    └── filterEventForEnvironments() 环境过滤
    │    └── for each eventWebHook:
    │         new EventWebHookNotifier({eventId, eventWebHookId}).enqueue()
    │              ↓
    │         Agenda: eventWebHook { eventId, eventWebHookId, retryCount: 0 }
    │              ↓
    │         EventWebHookNotifier.handleAgendaJob()
    │              ├── getEvent() + getEventWebHookById() + findOrganizationById()
    │              ├── 按 payloadType 构建 payload (json/raw/slack/discord)
    │              ├── getBackEndSecretsReplacer() → 替换 URL/headers 密钥
    │              ├── sendDataToWebHook()
    │              │    ├── getEventWebHookSignatureForPayload()
    │              │    ├── cancellableFetch() → HTTP POST (30s 超时)
    │              │    ├── 成功 → { result: "success" }
    │              │    └── 失败 → { result: "error" } (不 throw)
    │              ├── 成功 → handleWebHookSuccess()
    │              │    ├── updateEventWebHookStatus({state:"success"})
    │              │    └── createEventWebHookLog()
    │              └── 失败 → handleWebHookError()
    │                   ├── updateEventWebHookStatus({state:"error"})
    │                   ├── createEventWebHookLog()
    │                   └── retryJob()
    │                        最多重试 3 次: 30s → 5m → 5m → 放弃
    │
    └── slackEventHandler(event)  ← 并行 Slack 通知
```

---

## 八、关键结论的逐项代码对照

以下每条结论均附源码位置和判定依据，可直接跳转验证。

### 结论 1：5 条通道的重试上限各不相同

| 通道 | 最大重试 | 判定依据 |
|------|---------|---------|
| Legacy SDK | 2 | `webhooks.ts:119-129`：`retryCount===0` 进入、`retryCount===1` 进入、`else` 放弃，故 retryCount 可从 0 递增到 2（即重试 2 次） |
| SDK Webhook (新版) | 2 | `sdkWebhooks.ts:84-94`：与 Legacy 完全一致的 if/else if/else 结构 |
| Event Webhook | 3 | `EventWebHookNotifier.ts:372`：`retryCount >= 3` 时 return，故 retryCount 可从 0 递增到 3（即重试 3 次） |
| Global SDK | 0 | `sdkWebhooks.ts:410-418`：`runWebhookFetch().catch()` 无 retryCount 字段、无 Agenda job 包裹 |
| Proxy Update | 1 | `proxyUpdate.ts:131-136`：`retryCount===0` 进入、`else` 放弃，故 retryCount 只能从 0 递增到 1（即重试 1 次） |

### 结论 2：源码注释中的 "failed N times" 含义不一致

| 文件 | 注释原文 | 实际含义 |
|------|---------|---------|
| `webhooks.ts:126` | `// If it failed 3 times, give up` | 总执行 3 次（首次 + 2 次重试）后放弃 |
| `sdkWebhooks.ts:91` | `// If it failed 3 times, give up` | 同上，总执行 3 次后放弃 |
| `EventWebHookNotifier.ts:373` | `// If it failed 3 times, give up` | 重试 3 次（总执行 4 次）后放弃 |
| `proxyUpdate.ts:134` | `// If it failed twice, give up` | 总执行 2 次（首次 + 1 次重试）后放弃 |

同样的注释措辞 "failed 3 times"，在 Legacy/SDK 中指总执行次数，在 Event Webhook 中指重试次数。这是代码层面的语义不一致，分析时应以代码逻辑（`retryCount` 的递增与判断条件）为准。

### 结论 3：重试触发机制分为两类

**Agenda `fail` 事件驱动**（Legacy / SDK / Proxy）：

```
投递失败 → throw Error → Agenda 标记 job 为 failed
  → agenda.on("fail:" + JOB_NAME, ...) 被触发
  → 在回调中修改 retryCount 和 nextRunAt
  → job.save() 后由 Agenda 重新调度
```

代码位置：`webhooks.ts:110-136`、`sdkWebhooks.ts:75-101`、`proxyUpdate.ts:122-143`

**业务代码显式调用**（Event Webhook）：

```
sendDataToWebHook() 返回 { result: "error" }（不 throw）
  → handleWebHookError() 被调用
  → 在 handleWebHookError 内直接调用 retryJob()
  → retryJob() 修改 retryCount 和 nextRunAt
  → job.save() 后由 Agenda 重新调度
```

代码位置：`EventWebHookNotifier.ts:320-358`（`handleWebHookError` 在第 358 行调用 `retryJob()`）

关键区别：Event Webhook 的 `sendDataToWebHook` 在 HTTP 响应非 ok 或异常时返回 `{ result: "error" }` 而非 throw（见第 253-259 行和第 267-276 行），因此 Agenda 不会触发 `fail` 事件，重试必须由业务代码主动发起。

### 结论 4：新版 SDK Webhook 的 payload 获取和发送均为串行

代码位置：`sdkWebhooks.ts:330-353`

```typescript
// 串行获取：BluebirdPromise.reduce 逐个 await
const payloads = await BluebirdPromise.reduce(
  connections,
  async (payloads, connection) => {
    const defs = await getFeatureDefinitionsWithCache(...);  // 每次都 await
    return [[connection.key, defs], ...payloads];
  },
  [],
);

// 串行发送：BluebirdPromise.each 逐个 await
await BluebirdPromise.each(payloads, ([key, payload]) =>
  runWebhookFetch({...}),  // 每次都 await
);
```

`BluebirdPromise.reduce` 和 `BluebirdPromise.each` 都是串行迭代器，与 `Promise.all` 的并行语义不同。

### 结论 5：Feature 更新接口的响应延迟由 Mongo 操作主导，purgeCDNCache 在后台异步执行

**代码证据 — 完整异步调用链**：

```typescript
// 第 1 层：controllers/features.ts:2433 — 接口 await updateFeature
const updatedFeature = await updateFeature(context, feature, updates);

// 第 2 层：FeatureModel.ts:1053 — updateFeature 内部 fire-and-forget
onFeatureUpdate(context, feature, updatedFeature).catch((e) => {
  logger.error(e, "Error refreshing SDK Payload on feature update");
});
// 紧接着 return updatedFeature —— 接口响应已发送

// 第 3 层：services/features.ts:618 — queueSDKPayloadRefresh 内部 fire-and-forget
refreshSDKPayloadCache({ ...data, stackTrace }).catch((e) => {
  logger.error(e, "Error refreshing SDK Payload Cache");
});

// 第 4 层：services/features.ts:821 — refreshSDKPayloadCache 内部 fire-and-forget
triggerWebhookJobs(context, payloadKeys, connectionsUpdated, true).catch(
  (e) => {
    logger.error(e, "Error triggering webhook jobs");
  }
);

// 第 5 层：updateAllJobs.ts:57 — triggerWebhookJobs 内部 await purgeCDNCache
await purgeCDNCache(context.org.id, surrogateKeys);
```

**关键判定**：
- 第 2 层的 `onFeatureUpdate(...).catch(...)` 没有 `await`，因此 `updateFeature` 函数会在 `onFeatureUpdate` 执行完之前就返回
- 第 1 层的 `await updateFeature` 因此不等待 `onFeatureUpdate` 及其内部任何操作完成
- 接口响应由 `return updatedFeature` 发送，此时 `purgeCDNCache` 和所有 webhook 投递都还在后台执行
- Feature 更新接口的响应延迟由 `FeatureModel.updateOne()` 和 `FeatureModel.findOne()` 两个 Mongo 操作主导

### 结论 6：熔断机制仅覆盖 3 条通道

| 通道 | 是否熔断 | 判定依据 |
|------|---------|---------|
| Legacy SDK | ✅ | `webhooks.ts:103` 调用 `setLastSdkWebhookError(webhook, e)`，该方法在 `WebhookModel.ts:119` 检查 `consecutiveFailures >= 10` 时设 `disabled=true` |
| SDK Webhook (新版) | ✅ | `sdkWebhooks.ts:304` 调用 `setLastSdkWebhookError(webhook, message)`，同上 |
| Event Webhook | ❌ | `EventWebHookNotifier.ts:339` 调用 `updateEventWebHookStatus(eventWebHookId, {state:"error"})`，该方法无 `consecutiveFailures` / `disabled` 逻辑 |
| Global SDK | ❌ | `sdkWebhooks.ts:410-418` 的 `runWebhookFetch` 调用时 `global=true`，此时 `sdkWebhooks.ts:287-288` 和 `303-304` 跳过 `setLastSdkWebhookError` 调用 |
| Proxy Update | ✅ | `proxyUpdate.ts:68-71` 在 job handler 开头检查 `consecutiveFailures >= 10` 直接 return |
