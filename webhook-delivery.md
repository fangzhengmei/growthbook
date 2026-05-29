# SDK Webhook 投递与重试代码分析

## 概述

GrowthBook 代码库中存在 **三套独立的 Webhook 体系**，各自有不同的使用场景和实现方式：

| 体系 | 任务名 | 适用场景 | 核心文件 |
|------|--------|----------|----------|
| Legacy SDK Webhook | `fireWebhook` | 旧版 SDK 连接 webhook | `packages/back-end/src/jobs/webhooks.ts` |
| SDK Webhook | `fireWebhooks` | 新版 SDK 连接 webhook | `packages/back-end/src/jobs/sdkWebhooks.ts` |
| Event Webhook | `eventWebHook` | 系统事件通知 | `packages/back-end/src/events/handlers/webhooks/` |

---

## 一、事件入队流程

### 1.1 触发源

Feature 变更时，`services/features.ts:821` 作为统一触发点：

```typescript
// services/features.ts:821
triggerWebhookJobs(context, payloadKeys, connectionsUpdated, true).catch(
  (e) => {
    logger.error(e, "Error triggering webhook jobs");
  }
);
```

### 1.2 统一入队分发器

`jobs/updateAllJobs.ts:17-58` 中的 `triggerWebhookJobs` 并行触发三个入队通道：

```
triggerWebhookJobs
├─→ queueWebhooksByConnections()  # 新版 SDK webhooks
├─→ fireGlobalSdkWebhooks()       # 全局配置 webhooks
└─→ queueLegacySdkWebhooks()      # 旧版 SDK webhooks
```

### 1.3 新版 SDK Webhook 入队 (`sdkWebhooks.ts:114-125`)

```typescript
export async function queueWebhooksByConnections(
  context: ReqContext | ApiReqContext,
  connections: SDKConnectionInterface[],
) {
  const sdkKeys = connections.map((c) => c.id);
  const webhooks =
    await context.models.sdkWebhooks.findAllSdkWebhooksByConnectionIds(sdkKeys);
  for (const webhook of webhooks) {
    if (webhook && !webhook.disabled) await queueSingleSdkWebhookJob(webhook);
  }
}
```

入队时的关键约束：
```typescript
// sdkWebhooks.ts:104-112
const job = agenda.create(SDK_WEBHOOKS_JOB_NAME, {
  webhookId: webhook.id,
  retryCount: 0,
});
job.unique({ "data.webhookId": webhook.id });  // 防重复
job.schedule(new Date());                      // 立即执行
await job.save();
```

### 1.4 旧版 SDK Webhook 入队 (`webhooks.ts:139-180`)

过滤逻辑：根据 `payloadKeys` 匹配 webhook 的 `project` 和 `environment`

```typescript
if (!payloadKeys.some(
  (key) =>
    key.project === (webhook.project || "") &&
    key.environment === (webhook.environment || "production"),
)) {
  continue;  // 跳过不相关的 webhook
}

// 额外过滤：featuresOnly 标记、disabled 状态
if (!isFeature && webhook.featuresOnly) continue;
if (webhook.disabled) continue;
```

### 1.5 Event Webhook 入队链

Event Webhook 走独立的事件系统：

```
EventModel.create(...) 
  → EventNotifier(eventId).perform()
    → Agenda job: "eventCreated"
      → webHooksEventHandler(event)
        → getAllEventWebHooksForEvent(...)  # 按事件名查找 webhook
        → EventWebHookNotifier.enqueue()
          → Agenda job: "eventWebHook"
```

`EventNotifier` 实现 (`events/notifiers/EventNotifier.ts:59-66`):
```typescript
async perform() {
  const job = this.agenda.create<EventNotificationData>("eventCreated", {
    eventId: this.eventId,
  });
  job.unique({ "data.eventId": this.eventId });
  job.schedule(new Date());
  await job.save();
}
```

---

## 二、签名生成机制

### 2.1 新版 SDK Webhook 双重签名 (`sdkWebhooks.ts:156-230`)

**签名 1 - `webhook-secret`** (兼容旧版):
```typescript
const signature = createHmac("sha256", signingKey)
  .update(sendPayload ? jsonPayload : "")
  .digest("hex");
const secret = `whsec_${signature}`;
```

**签名 2 - `webhook-signature`** (新版标准):
```typescript
const standardSignatureBody = `${webhookID}.${timestamp}.${body || ""}`;
const standardSignature =
  "v1," +
  createHmac("sha256", signingKey)
    .update(standardSignatureBody)
    .digest("base64");
```

**请求头**:
```
webhook-id: msg_xxx
webhook-timestamp: 1620000000
webhook-signature: v1,abc123...
webhook-secret: whsec_xyz789...
webhook-sdk-key: sdk-abc123
```

### 2.2 旧版 SDK Webhook 单一签名 (`webhooks.ts:78-80`)

```typescript
const signature = createHmac("sha256", webhook.signingKey)
  .update(payload)
  .digest("hex");
```

**请求头**:
```
X-GrowthBook-Signature: abc123...
Content-Type: application/json
```

### 2.3 Event Webhook 签名 (`event-webhooks-utils.ts:39-49`)

```typescript
export const getEventWebHookSignatureForPayload = <T>({
  signingKey,
  payload,
}: {
  signingKey: string;
  payload: T;
}): string => {
  const requestPayload = JSON.stringify(payload);
  return createHmac("sha256", signingKey).update(requestPayload).digest("hex");
};
```

---

## 三、核心投递逻辑

### 3.1 新版 SDK Webhook 投递 (`sdkWebhooks.ts:308-354`)

```
fireSdkWebhook(context, webhook)
  ├─→ findSDKConnectionsByIds(webhook.sdks)
  ├─→ 并行获取所有 SDK connections 的 payload
  │    └─→ getFeatureDefinitionsWithCache(connection)
  └─→ 逐个调用 runWebhookFetch()
       ├─→ 生成双重签名
       ├─→ 按 payloadFormat 构建 body
       ├─→ cancellableFetch() 发送请求
       ├─→ 成功: createSdkWebhookLog(成功) + 重置错误状态
       └─→ 失败: createSdkWebhookLog(失败) + 记录错误 + throw
```

**Payload Format 支持**:
- `standard`: 标准格式 `{ type, timestamp, data: { payload } }`
- `sdkPayload`: 原始 SDK payload
- `edgeConfig`: Vercel Edge Config 格式
- `edgeConfigUnescaped`: 非转义的 Edge Config 格式
- `vercelNativeIntegration`: Vercel 原生集成格式
- `standard-no-payload`: 不含 payload 的标准格式
- `none`: 不发送 body

### 3.2 旧版 SDK Webhook 投递 (`webhooks.ts:25-108`)

```
fireWebhook(job)
  ├─→ 构建 legacy cache key
  ├─→ getFeatureDefinitionsWithCache()
  ├─→ 构建 payload: { timestamp, features, dateUpdated, overrides?, experiments? }
  ├─→ 生成签名
  ├─→ cancellableFetch()
  ├─→ 成功: setLastSdkWebhookError(webhook, "")
  └─→ 失败: setLastSdkWebhookError(webhook, error) + throw
```

### 3.3 Event Webhook 投递 (`EventWebHookNotifier.ts:211-277`)

```
sendDataToWebHook()
  ├─→ getEventWebHookSignatureForPayload()
  ├─→ 应用 SecretsReplacer (替换 URL 和 headers 中的密钥占位符)
  ├─→ cancellableFetch()
  ├─→ 成功: 返回 { result: "success", ... }
  └─→ 失败: 返回 { result: "error", ... }
```

---

## 四、失败重试机制

### 4.1 重试策略（三类 Webhook 共用相同策略）

| 重试次数 | 等待时间 |
|----------|----------|
| 第 1 次失败 | 30 秒 |
| 第 2 次失败 | 5 分钟 |
| 第 3 次失败 | 放弃 |

### 4.2 新版/旧版 SDK Webhook 重试实现

通过 Agenda 的 `fail` 事件监听实现 (`sdkWebhooks.ts:75-101` / `webhooks.ts:110-136`):

```typescript
agenda.on(
  "fail:" + JOB_NAME,
  async (error: Error, job: WebhookJob) => {
    const retryCount = job.attrs.data.retryCount;
    let nextRunAt = Date.now();
    
    if (retryCount === 0) {
      nextRunAt += 30000;      // 30秒
    } else if (retryCount === 1) {
      nextRunAt += 300000;     // 5分钟
    } else {
      return;  // 第3次失败，放弃
    }

    job.attrs.data.retryCount++;
    job.attrs.nextRunAt = new Date(nextRunAt);
    await job.save();
  }
);
```

### 4.3 Event Webhook 重试实现 (`EventWebHookNotifier.ts:371-389`)

显式调用 `retryJob()` 方法：

```typescript
private static async retryJob(job: Job<EventWebHookJobData>) {
  if (job.attrs.data.retryCount >= 3) return;  // 最多3次

  let nextRunAt = Date.now();
  if (job.attrs.data.retryCount === 0) {
    nextRunAt += 30000;      // 30秒
  } else {
    nextRunAt += 300000;     // 第2、3次都是5分钟
  }

  job.attrs.data.retryCount++;
  job.attrs.nextRunAt = new Date(nextRunAt);
  await job.save();
}
```

---

## 五、失败回退与熔断机制

### 5.1 连续失败熔断 (`WebhookModel.ts:109-131`)

阈值常量 (`shared/constants.ts:261`):
```typescript
export const WEBHOOK_CONSECUTIVE_FAILURES_THRESHOLD = 10;
```

熔断逻辑 (`WebhookModel.ts:113-122`):
```typescript
public async setLastSdkWebhookError(
  webhook: WebhookInterface,
  error: string,
) {
  if (error) {
    const consecutiveFailures = (webhook.consecutiveFailures || 0) + 1;
    const updates: UpdateProps<WebhookInterface> = {
      error,
      consecutiveFailures,
    };
    // 连续失败 10 次自动禁用
    if (consecutiveFailures >= WEBHOOK_CONSECUTIVE_FAILURES_THRESHOLD) {
      updates.disabled = true;
    }
    await this.update(webhook, updates);
  } else {
    // 成功时重置所有状态
    await this.update(webhook, {
      error: "",
      lastSuccess: new Date(),
      consecutiveFailures: 0,
      disabled: false,
    });
  }
}
```

### 5.2 投递日志记录

**SDK Webhook 日志** (`SdkWebhookLogModel`):
```typescript
interface SdkWebHookLogInterface {
  id: string;
  webhookId: string;
  webhookRequestId?: string;
  organizationId: string;
  dateCreated: Date;
  responseCode: number | null;
  responseBody: string | null;
  result: "error" | "success";
  payload: Record<string, unknown>;
}
```

**Event Webhook 日志** (`EventWebHookLogModel`):
每次投递（无论成功失败）都会创建日志记录，包含 payload、响应状态码、响应体等完整信息。

---

## 六、完整调用链路图

### 6.1 SDK Webhook 完整链路

```
Feature 变更
    ↓
services/features.ts: updateFeatures()
    ↓
triggerWebhookJobs(context, payloadKeys, connections)
    ├─→ queueWebhooksByConnections()
    │    └─→ Agenda: fireWebhooks { webhookId, retryCount: 0 }
    │          ↓
    │        fireWebhooks(job)
    │          ├─→ getFeatureDefinitionsWithCache()
    │          ├─→ 生成双重签名
    │          ├─→ runWebhookFetch() → HTTP POST
    │          ├─→ 成功: 重置错误状态 + 记录日志
    │          └─→ 失败: 记录错误 + throw → 触发 fail 事件 → 重试
    │
    ├─→ fireGlobalSdkWebhooks()  # 直接执行，无重试
    │
    └─→ queueLegacySdkWebhooks()
         └─→ Agenda: fireWebhook { webhookId, retryCount: 0 }
               ↓
             fireWebhook(job)
               ├─→ 生成单一签名
               ├─→ cancellableFetch()
               ├─→ 成功: 重置错误状态
               └─→ 失败: 记录错误 + throw → 触发 fail 事件 → 重试
```

### 6.2 Event Webhook 完整链路

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
    ├─→ webHooksEventHandler(event)
    │    ├─→ getAllEventWebHooksForEvent(eventName)
    │    └─→ 为每个 webhook 创建 EventWebHookNotifier.enqueue()
    │          ↓
    │        Agenda: eventWebHook { eventId, eventWebHookId, retryCount: 0 }
    │          ↓
    │        EventWebHookNotifier.handleAgendaJob()
    │          ├─→ 按 payloadType 构建 payload (json/raw/slack/discord)
    │          ├─→ 生成签名
    │          ├─→ sendDataToWebHook() → HTTP POST
    │          ├─→ 成功: 更新状态 + 记录日志
    │          └─→ 失败: 更新状态 + 记录日志 + retryJob()
    │
    └─→ slackEventHandler(event)  # 并行的 Slack 通知
```

---

## 七、关键设计要点

1. **防重复入队**: 所有 Agenda job 都使用 `job.unique()` 确保同一 webhook 同一事件不会重复入队

2. **超时控制**: 使用 `cancellableFetch()` 设置 30 秒超时和 1000 字节最大响应大小

3. **并行但隔离**: 不同 webhook 体系完全独立，互不影响

4. **幂等性设计**: 
   - `webhook-id` 用于接收方去重
   - `webhook-timestamp` 用于防止重放攻击

5. **渐进式重试**: 指数退避策略（30秒 → 5分钟 → 放弃）避免轰炸接收方

6. **熔断保护**: 连续 10 次失败自动禁用，避免无效资源消耗
