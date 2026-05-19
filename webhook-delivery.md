# Webhook 事件从触发到送达全流程解析

GrowthBook 中存在三种类型的 Webhook 实现，每种都有不同的应用场景：

1. **旧版 SDK Webhook** (`packages/back-end/src/jobs/webhooks.ts`) - 用于 SDK 特性变更通知
2. **新版 SDK Webhook** (`packages/back-end/src/jobs/sdkWebhooks.ts`) - 新一代 SDK Webhook
3. **事件 Webhook** (`packages/back-end/src/events/handlers/webhooks/EventWebHookNotifier.ts`) - 通用事件通知（支持 Slack、Discord、自定义 JSON 等）

---

## 一、事件入队机制

### 1.1 触发时机

**旧版 SDK Webhook 触发** (`webhooks.ts:139-179`):
- 当特性或实验配置发生变化时调用 `queueLegacySdkWebhooks()`
- 根据 `payloadKeys` 匹配对应的 webhook（项目、环境维度）
- 跳过禁用的 webhook 和不相关的 webhook

**新版 SDK Webhook 触发** (`sdkWebhooks.ts:114-125`):
- 当 SDK 连接的配置发生变化时调用 `queueWebhooksByConnections()`
- 根据 SDK 连接 ID 查找关联的 webhook

**事件 Webhook 触发** (`webHooksEventHandler.ts:15-57`):
- 事件系统通过 `webHooksEventHandler` 接收事件
- 根据事件名称、标签、项目、环境筛选匹配的 webhook
- 为每个匹配的 webhook 创建 `EventWebHookNotifier` 并调用 `enqueue()`

### 1.2 队列实现

所有 Webhook 都使用 **Agenda** 作为任务队列框架：

```typescript
// 入队核心逻辑（以事件 Webhook 为例）
const job = this.agenda.create<EventWebHookJobData>("eventWebHook", {
  eventId: this.options.eventId,
  eventWebHookId: this.options.eventWebHookId,
  retryCount: 0,  // 重试计数器初始化为0
});

// 确保同一事件+webhook组合不会重复入队
job.unique({
  "data.eventId": this.options.eventId,
  "data.eventWebHookId": this.options.eventWebHookId,
});

// 立即执行
job.schedule(new Date());
await job.save();
```

**关键特性**：
- `job.unique()`：防止相同任务重复入队
- `retryCount`：内置重试计数器
- 任务数据包含执行所需的全部上下文

---

## 二、签名拼装机制

### 2.1 签名算法

所有 Webhook 都使用 **HMAC-SHA256** 算法生成签名：

**事件 Webhook 签名** (`event-webhooks-utils.ts:39-49`):
```typescript
export const getEventWebHookSignatureForPayload = <T>({
  signingKey,
  payload,
}: {
  signingKey: string;
  payload: T;
}): string => {
  const requestPayload = JSON.stringify(payload);
  return createHmac("sha256", signingKey)
    .update(requestPayload)
    .digest("hex");
};
```

**旧版 SDK Webhook 签名** (`webhooks.ts:78-80`):
```typescript
const signature = createHmac("sha256", webhook.signingKey)
  .update(payload)
  .digest("hex");
```

**新版 SDK Webhook 签名** (`sdkWebhooks.ts:156-230`):
新版使用了更复杂的签名机制，包含两层签名：

```typescript
// 第一层：基于 payload 的签名（兼容旧版）
const signature = createHmac("sha256", signingKey)
  .update(sendPayload ? jsonPayload : "")
  .digest("hex");
const secret = `whsec_${signature}`;

// 第二层：标准签名（包含 webhook-id、timestamp、body）
const standardSignatureBody = `${webhookID}.${timestamp}.${body || ""}`;
const standardSignature = "v1," +
  createHmac("sha256", signingKey)
    .update(standardSignatureBody)
    .digest("base64");
```

### 2.2 请求头结构

| 版本 | 签名头名称 | 格式 |
|------|-----------|------|
| 事件 Webhook | `X-GrowthBook-Signature` | HMAC-SHA256 hex |
| 旧版 SDK | `X-GrowthBook-Signature` | HMAC-SHA256 hex |
| 新版 SDK | `webhook-signature` | `v1,{base64-hmac}` |
| 新版 SDK | `webhook-secret` | `whsec_{hex-hmac}` |
| 新版 SDK | `webhook-id` | `msg_{md5}` |
| 新版 SDK | `webhook-timestamp` | Unix 时间戳（秒） |
| 新版 SDK | `webhook-sdk-key` | SDK Key |

### 2.3 签名密钥生成

**事件 Webhook 密钥** (`EventWebhookModel.ts:253`):
```typescript
const signingKey = "ewhk_" + md5(randomUUID()).substr(0, 32);
```

**SDK Webhook 密钥** (`WebhookModel.ts:150`):
```typescript
signingKey: "wk_" + md5(uniqid()).slice(0, 16),
```

---

## 三、失败重试机制

### 3.1 重试策略（所有 Webhook 通用）

| 重试次数 | 等待时间 |
|---------|---------|
| 第 1 次失败 | 30 秒 |
| 第 2 次失败 | 5 分钟 |
| 第 3 次失败 | 放弃（TODO: 邮件通知） |

**事件 Webhook 重试实现** (`EventWebHookNotifier.ts:371-389`):
```typescript
private static async retryJob(job: Job<EventWebHookJobData>) {
  if (job.attrs.data.retryCount >= 3) {
    // 最多重试3次，超过则放弃
    return;
  }

  let nextRunAt = Date.now();
  if (job.attrs.data.retryCount === 0) {
    nextRunAt += 30000;  // 第1次：等待30秒
  } else {
    nextRunAt += 300000; // 第2、3次：等待5分钟
  }

  job.attrs.data.retryCount++;
  job.attrs.nextRunAt = new Date(nextRunAt);
  await job.save();
}
```

**旧版 SDK Webhook 重试实现** (`webhooks.ts:110-136`):
通过监听 Agenda 的 `fail` 事件实现重试：
```typescript
agenda.on("fail:" + WEBHOOK_JOB_NAME, async (error: Error, job: WebhookJob) => {
  const retryCount = job.attrs.data.retryCount;
  let nextRunAt = Date.now();
  
  if (retryCount === 0) {
    nextRunAt += 30000;   // 30秒
  } else if (retryCount === 1) {
    nextRunAt += 300000;  // 5分钟
  } else {
    return;  // 放弃
  }
  
  job.attrs.data.retryCount++;
  job.attrs.nextRunAt = new Date(nextRunAt);
  await job.save();
});
```

### 3.2 失败处理流程

**事件 Webhook 失败处理** (`EventWebHookNotifier.ts:320-359`):
1. 更新 webhook 状态为 `error`，记录错误信息
2. 创建 `EventWebHookLog` 日志记录
3. 调用 `retryJob()` 尝试重试

**SDK Webhook 失败处理** (`sdkWebhooks.ts:290-306`):
1. 创建 `SdkWebhookLog` 日志记录
2. 调用 `setLastSdkWebhookError()` 更新错误状态
3. 连续失败超过阈值会自动禁用 webhook

**连续失败自动禁用** (`WebhookModel.ts:109-131`):
```typescript
public async setLastSdkWebhookError(webhook: WebhookInterface, error: string) {
  if (error) {
    const consecutiveFailures = (webhook.consecutiveFailures || 0) + 1;
    const updates: UpdateProps<WebhookInterface> = {
      error,
      consecutiveFailures,
    };
    // 连续失败超过阈值（WEBHOOK_CONSECUTIVE_FAILURES_THRESHOLD）自动禁用
    if (consecutiveFailures >= WEBHOOK_CONSECUTIVE_FAILURES_THRESHOLD) {
      updates.disabled = true;
    }
    await this.update(webhook, updates);
  } else {
    // 成功时重置失败计数
    await this.update(webhook, {
      error: "",
      lastSuccess: new Date(),
      consecutiveFailures: 0,
      disabled: false,
    });
  }
}
```

---

## 四、HTTP 请求与超时控制

### 4.1 cancellableFetch 工具

所有 Webhook 请求都通过 `cancellableFetch` 发送 (`http.util.ts:57-128`)：

```typescript
export const cancellableFetch = async (
  url: string,
  fetchOptions: RequestInit,
  abortOptions: CancellableFetchCriteria,
): Promise<CancellableFetchReturn> => {
  const abortController = new AbortController();
  
  // 超时控制
  const timeout = setTimeout(() => {
    abortController.abort();
  }, abortOptions.maxTimeMs);
  
  // 响应体大小限制
  const readResponseBody = async (res: Response): Promise<string> => {
    for await (const chunk of res.body) {
      received += chunk.length;
      if (received > abortOptions.maxContentSize) {
        abortController.abort();
        break;
      }
    }
  };
  // ...
};
```

### 4.2 统一的超时配置

| 配置项 | 值 | 说明 |
|-------|----|------|
| `maxTimeMs` | 30000ms (30秒) | 请求超时时间 |
| `maxContentSize` | 1000 | 最大响应体大小（字符数） |

---

## 五、完整调用链路

### 5.1 事件 Webhook 流程
```
事件发生 → webHooksEventHandler()
  ↓
筛选匹配的 webhook（按事件名、标签、项目、环境）
  ↓
为每个 webhook 创建 EventWebHookNotifier.enqueue()
  ↓
Agenda 任务入队（带 unique 约束）
  ↓
Agenda 调度执行 handleAgendaJob()
  ├─ 构建 payload（支持多种格式：json/raw/slack/discord）
  ├─ 生成签名：getEventWebHookSignatureForPayload()
  ├─ 发送请求：cancellableFetch()
  └─ 结果处理
     ├─ 成功：更新状态为 success，记录日志
     └─ 失败：更新状态为 error，记录日志，retryJob()
        └─ 重试 ≤ 3 次
```

### 5.2 SDK Webhook 流程
```
SDK 配置变更 → queueWebhooksByConnections()
  ↓
查找关联的 webhook
  ↓
Agenda 任务入队
  ↓
Agenda 调度执行 fireWebhooks()
  ├─ 获取最新的特性定义
  ├─ 根据 payloadFormat 构建请求体
  │  ├─ none：无 body
  │  ├─ standard-no-payload：仅元数据
  │  ├─ standard：完整格式
  │  ├─ sdkPayload：原始 payload
  │  ├─ edgeConfig：Vercel Edge Config 格式
  │  └─ vercelNativeIntegration：Vercel 集成格式
  ├─ 生成两层签名
  ├─ 发送请求：cancellableFetch()
  └─ 结果处理
     ├─ 成功：重置连续失败计数
     └─ 失败：累计失败计数，触发重试
        └─ 连续失败过多 → 自动禁用 webhook
```

---

## 六、关键设计特点

| 特点 | 说明 |
|-----|------|
| **幂等性保障** | 通过 `job.unique()` 防止重复入队 |
| **分级重试** | 30秒 → 5分钟 → 放弃，避免风暴效应 |
| **熔断机制** | 连续失败超过阈值自动禁用 webhook |
| **审计日志** | 每次调用都记录详细日志（请求、响应、状态） |
| **超时保护** | 统一的 30 秒超时和响应体大小限制 |
| **代理支持** | 支持通过代理发送 webhook 请求 |
