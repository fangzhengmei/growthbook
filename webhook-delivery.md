# Webhook 事件从触发到送达全流程解析

GrowthBook 中存在三种独立的 Webhook 实现，每类都有各自的触发场景、签名机制和重试策略：

| Webhook 类型 | 主文件 | 用途 |
|------------|--------|------|
| **事件 Webhook** | `EventWebHookNotifier.ts` | 通用事件通知（Slack、Discord、自定义 JSON 等） |
| **旧版 SDK Webhook** | `jobs/webhooks.ts` | 旧版 SDK 特性变更通知（legacy） |
| **新版 SDK Webhook** | `jobs/sdkWebhooks.ts` | 新一代 SDK Webhook，支持多种 payload 格式 |

---

## 零、事件通知队列入口与异步调度语义

### 0.1 事件创建与分发的完整链路

```
业务操作触发事件 → createEvent()  [EventModel.ts:236]
  ↓
createEventWithPayload()  [EventModel.ts:108]
  ├─ 生成 eventId: `event-${randomUUID()}`
  ├─ 写入 MongoDB Event 集合
  └─ new EventNotifier(event.id).perform()  // 入队调度，无 await！
  ↓
EventNotifier.perform()  [EventNotifier.ts:59]
  ├─ Agenda 任务入队（job name: "eventCreated"）
  ├─ job.unique({ "data.eventId": eventId })  // 幂等约束
  └─ job.schedule(new Date())  // 立即执行
  ↓
Agenda 调度 → EventNotifier.jobHandler()  [EventNotifier.ts:42]
  ├─ 从 DB 读取 event 完整数据
  ├─ webHooksEventHandler(event, context)     // 无 await，并发异步
  └─ slackEventHandler(event, context)        // 无 await，并发异步
```

### 0.2 关键异步语义

**事件创建是非阻塞的**（`EventModel.ts:142`）：
```typescript
// 注意：这里没有 await！事件入队后立即返回，不等待调度
new EventNotifier(event.id).perform();
```

**事件分发是并发异步的**（`EventNotifier.ts:55-56`）：
```typescript
// 两个 handler 都没有 await，并发执行，互不等待
webHooksEventHandler(event, context);
slackEventHandler(event, context);
```

**调度语义总结**：
| 阶段 | 同步/异步 | 失败影响 |
|-----|----------|---------|
| 事件写入 DB | 同步 | 事件丢失 |
| EventNotifier.perform() 入队 | 同步 | 事件不会被分发 |
| Agenda 调度 jobHandler | 异步 | 可重试 |
| webHooksEventHandler 执行 | 异步并发 | 不影响 Slack 通知 |
| 单个 webhook 请求发送 | 异步 | 触发该 webhook 的重试机制 |

---

## 一、事件 Webhook（EventWebHookNotifier）

### 1.1 完整调用链路

```
EventNotifier.jobHandler 分发 → webHooksEventHandler()  [webHooksEventHandler.ts:15]
  ↓
┌─ 特殊分支：event === "webhook.test" ───────────────────────────┐
│  直接从 event.data 提取 webhookId，精确查找单个 webhook        │
│  跳过标签、项目、环境过滤，跳过 enabled 检查！                   │
└───────────────────────────────────────────────────────────────┘
  ↓
常规分支：按事件名、标签、项目、环境筛选匹配的 webhook
  ↓
为每个匹配的 webhook 创建 EventWebHookNotifier → enqueue()  [EventWebHookNotifier.ts:65]
  ↓
Agenda 任务入队（job name: "eventWebHook"）
  ├─ job.unique({ "data.eventId", "data.eventWebHookId" })  // 幂等约束
  └─ 初始 retryCount = 0
  ↓
Agenda 调度 → handleAgendaJob()  [EventWebHookNotifier.ts:83]
  ├─ 构建 payload（json/raw/slack/discord 格式）
  ├─ 生成签名 → sendDataToWebHook()  [EventWebHookNotifier.ts:211]
  ├─ 发送请求 → cancellableFetch()
  └─ 结果处理
     ├─ 成功：handleWebHookSuccess()
     │   ├─ updateEventWebHookStatus(state: "success")
     │   └─ createEventWebHookLog()
     └─ 失败：handleWebHookError()  [EventWebHookNotifier.ts:320]
         ├─ updateEventWebHookStatus(state: "error")
         ├─ createEventWebHookLog()
         └─ retryJob()  → 最多重试 3 次
```

### 1.2 webhook.test 特殊匹配路径

**触发入口**：用户点击"测试"按钮 → `POST /event-webhooks/test`

**完整链路**：
```
前端点击测试 → createTestEventWebHook()  [event-webhooks.controller.ts:366]
  ↓
sendEventWebhookTestEvent(context, webhookId)  [EventWebhookModel.ts:457]
  ↓
createEvent() 构造特殊事件：
  object: "webhook"
  event: "test"              → 最终事件名："webhook.test"
  data: { object: { webhookId: webhook.id } }
  projects: [], tags: [], environments: []
  ↓
事件入队 → webHooksEventHandler 匹配阶段  [webHooksEventHandler.ts:22-34]
```

**匹配逻辑**（`webHooksEventHandler.ts:22-34`）：
```typescript
if (event.data.event === "webhook.test") {
  // 直接从 event data 中提取 webhookId
  const webhookId = event.version
    ? event.data.data.object.webhookId    // 新版格式
    : event.data.data.webhookId;          // 旧版兼容格式
  
  // 精确查找单个 webhook，绕过所有过滤条件
  const webhook = await getEventWebHookById(webhookId, event.organizationId);
  return webhook ? [webhook] : [];
}
```

**特殊行为**：
- ✅ 不检查 webhook 的 `enabled` 状态（`getEventWebHookById` 不验证 enabled）
- ✅ 跳过标签（tags）过滤
- ✅ 跳过项目（projects）过滤
- ✅ 跳过环境（environments）过滤
- ✅ 忽略 webhook 配置的事件类型匹配
- ❗ 测试事件可以发送给已禁用的 webhook！

### 1.3 签名拼装

**签名算法**：HMAC-SHA256（十六进制输出）

**实现位置**：`event-webhooks-utils.ts:39-49`

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

**请求头**：
```
X-GrowthBook-Signature: {hmac-sha256-hex}
Content-Type: application/json
User-Agent: GrowthBook Webhook
```

### 1.4 失败重试策略

**触发方式**：请求失败后在 `handleWebHookError()` 中**主动调用** `retryJob()`

**重试逻辑**：`EventWebHookNotifier.ts:371-389`

```typescript
private static async retryJob(job: Job<EventWebHookJobData>) {
  if (job.attrs.data.retryCount >= 3) {
    return;  // 最多重试 3 次，超过放弃
  }

  let nextRunAt = Date.now();
  if (job.attrs.data.retryCount === 0) {
    nextRunAt += 30000;   // 第1次失败：30秒后重试
  } else {
    nextRunAt += 300000;  // 第2、3次失败：5分钟后重试
  }

  job.attrs.data.retryCount++;
  job.attrs.nextRunAt = new Date(nextRunAt);
  await job.save();
}
```

| 重试阶段 | retryCount 值 | 等待时间 |
|---------|--------------|---------|
| 首次执行 | 0 | - |
| 第1次重试 | 0 → 1 | 30 秒 |
| 第2次重试 | 1 → 2 | 5 分钟 |
| 第3次重试 | 2 → 3 | 5 分钟 |
| 放弃 | 3 | - |

**重试次数**：共 3 次重试机会

---

## 二、旧版 SDK Webhook（jobs/webhooks.ts）

### 2.1 完整调用链路

```
特性/实验配置变更 → queueLegacySdkWebhooks()  [webhooks.ts:139]
  ↓
匹配 webhook（项目、环境维度），跳过禁用的 webhook
  ↓
Agenda 任务入队（job name: "fireWebhook"）
  ├─ job.unique({ webhookId })
  └─ 初始 retryCount = 0
  ↓
Agenda 调度 → job handler  [webhooks.ts:25]
  ├─ 获取最新特性定义 getFeatureDefinitionsWithCache()
  ├─ 构建 payload（timestamp + features + dateUpdated）
  ├─ 生成签名
  ├─ 发送请求 cancellableFetch()
  └─ 结果处理
     ├─ 成功：setLastSdkWebhookError(webhook, "")
     └─ 失败：setLastSdkWebhookError(webhook, error) + throw Error
         ↓
agenda.on("fail:fireWebhook") 监听器触发  [webhooks.ts:110]
  └─ 重试调度 → 最多重试 2 次
```

### 2.2 签名拼装

**签名算法**：HMAC-SHA256（十六进制输出）

**实现位置**：`webhooks.ts:78-80`

```typescript
const signature = createHmac("sha256", webhook.signingKey)
  .update(payload)  // payload = JSON.stringify(body)
  .digest("hex");
```

**请求头**：
```
X-GrowthBook-Signature: {hmac-sha256-hex}
Content-Type: application/json
```

### 2.3 失败重试策略

**触发方式**：Job 抛出异常后，通过 Agenda 的 `fail` 事件**被动监听**触发重试

**重试逻辑**：`webhooks.ts:110-136`

```typescript
agenda.on("fail:" + WEBHOOK_JOB_NAME, async (error: Error, job: WebhookJob) => {
  if (!job.attrs.data) return;

  const retryCount = job.attrs.data.retryCount;
  let nextRunAt = Date.now();
  
  if (retryCount === 0) {
    nextRunAt += 30000;   // 第1次失败：30秒后重试
  } else if (retryCount === 1) {
    nextRunAt += 300000;  // 第2次失败：5分钟后重试
  } else {
    return;  // 放弃（TODO: 邮件通知组织所有者）
  }

  job.attrs.data.retryCount++;
  job.attrs.nextRunAt = new Date(nextRunAt);
  await job.save();
});
```

| 重试阶段 | retryCount 值 | 等待时间 |
|---------|--------------|---------|
| 首次执行 | 0 | - |
| 第1次重试 | 0 → 1 | 30 秒 |
| 第2次重试 | 1 → 2 | 5 分钟 |
| 放弃 | 2 | - |

**重试次数**：共 2 次重试机会

**额外熔断机制**：连续失败超过 `WEBHOOK_CONSECUTIVE_FAILURES_THRESHOLD` 阈值时自动禁用 webhook（`WebhookModel.ts:119-121`）

---

## 三、新版 SDK Webhook（jobs/sdkWebhooks.ts）

### 3.1 完整调用链路

```
SDK 配置变更 → queueWebhooksByConnections()  [sdkWebhooks.ts:114]
  ↓
根据 SDK 连接 ID 查找关联 webhook
  ↓
Agenda 任务入队（job name: "fireWebhooks"）
  ├─ job.unique({ "data.webhookId" })
  └─ 初始 retryCount = 0
  ↓
Agenda 调度 → fireWebhooks()  [sdkWebhooks.ts:37]
  ↓
fireSdkWebhook()  [sdkWebhooks.ts:308]
  ├─ 获取关联的 SDK connections
  ├─ 对每个 connection 获取特性定义
  └─ runWebhookFetch()  [sdkWebhooks.ts:127]
      ├─ 根据 payloadFormat 构建请求体
      ├─ 生成两层签名
      ├─ 发送请求 cancellableFetch()
      └─ 结果处理
         ├─ 成功：createSdkWebhookLog(success) + setLastSdkWebhookError("")
         └─ 失败：createSdkWebhookLog(error) + setLastSdkWebhookError(msg) + throw Error
                ↓
agenda.on("fail:fireWebhooks") 监听器触发  [sdkWebhooks.ts:75]
  └─ 重试调度 → 最多重试 2 次
```

### 3.2 签名拼装（两层签名机制）

**实现位置**：`sdkWebhooks.ts:156-230`

**第一层签名（兼容旧版）**：
```typescript
const signature = createHmac("sha256", signingKey)
  .update(sendPayload ? jsonPayload : "")
  .digest("hex");
const secret = `whsec_${signature}`;
```

**第二层签名（标准签名，包含时间戳防重放）**：
```typescript
const webhookID = `msg_${md5(key + date.getTime()).substr(0, 16)}`;
const timestamp = Math.floor(date.getTime() / 1000);
const standardSignatureBody = `${webhookID}.${timestamp}.${body || ""}`;
const standardSignature = "v1," +
  createHmac("sha256", signingKey)
    .update(standardSignatureBody)
    .digest("base64");
```

**请求头**：
```
webhook-id: msg_{md5}
webhook-timestamp: {unix-seconds}
webhook-signature: v1,{base64-hmac-sha256}
webhook-secret: whsec_{hex-hmac-sha256}
webhook-sdk-key: {sdk-key}
Content-Type: application/json
User-Agent: GrowthBook Webhook
```

**Payload 格式支持**（`sdkWebhooks.ts:172-222`）：
| 格式 | 说明 |
|------|------|
| `none` | GET 请求，无 body |
| `standard-no-payload` | 仅元数据，无 payload 内容 |
| `standard` | 完整格式：`{ type, timestamp, data: { payload } }` |
| `sdkPayload` | 原始 SDK payload JSON |
| `edgeConfig` | Vercel Edge Config 格式（payload 转义） |
| `edgeConfigUnescaped` | Vercel Edge Config 格式（payload 不转义） |
| `vercelNativeIntegration` | Vercel 原生集成格式 |

### 3.3 失败重试策略

**触发方式**：与旧版 SDK Webhook 相同，通过 Agenda `fail` 事件监听触发

**重试逻辑**：`sdkWebhooks.ts:75-101`（与旧版 SDK Webhook 完全相同）

```typescript
agenda.on("fail:" + SDK_WEBHOOKS_JOB_NAME, async (error: Error, job: SDKWebhookJob) => {
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

| 重试阶段 | retryCount 值 | 等待时间 |
|---------|--------------|---------|
| 首次执行 | 0 | - |
| 第1次重试 | 0 → 1 | 30 秒 |
| 第2次重试 | 1 → 2 | 5 分钟 |
| 放弃 | 2 | - |

**重试次数**：共 2 次重试机会

**额外熔断机制**：与旧版相同，连续失败超过阈值自动禁用 webhook

---

## 四、cancellableFetch 响应体超限中断的结果判定

### 4.1 核心实现逻辑

**实现位置**：`http.util.ts:57-128`

```typescript
const readResponseBody = async (res: Response): Promise<string> => {
  for await (const chunk of res.body) {
    received += chunk.length;
    chunks.push(chunk.toString());

    if (received > abortOptions.maxContentSize) {
      abortController.abort();  // 超限主动中断
      break;
    }
  }
  return chunks.join("");
};

try {
  response = await fetch(url, { ... });
  stringBody = await readResponseBody(response);
  return { responseWithoutBody: response, stringBody };
} catch (e) {
  // 关键：响应体超限导致的 AbortError 不抛出异常！
  if (e.name === "AbortError" && response) {
    logger.warn(e, `Response aborted due to content size: ${received}`);
    return {
      responseWithoutBody: response,  // 保留原始响应（含状态码）
      stringBody,                     // 已读取的部分内容
    };
  }
  throw e;  // 其他错误正常抛出
}
```

### 4.2 对结果判定的影响

**三种响应场景对比**：

| 场景 | HTTP 状态码 | 行为 | 结果判定 |
|-----|------------|------|---------|
| 正常响应 | 200 OK | 完整读取响应体 | ✅ 成功 |
| 响应体超限（≤1000字符限制） | 200 OK | 中断读取，返回部分内容 | ✅ **被判定为成功！** |
| 服务端错误 | 500 Internal Server Error | 正常读取错误响应 | ❌ 失败，触发重试 |
| 网络超时 | - | 超时中断 | ❌ 失败，触发重试 |
| 连接拒绝 | - | 连接失败 | ❌ 失败，触发重试 |

### 4.3 关键设计隐忧

**响应体超限被视为成功的风险**：
1. **状态码优先**：只要 HTTP 状态码是 2xx，即使响应体被截断，也会被判定为成功
2. **静默丢失**：对端返回的关键信息可能被截断，但调用方无法感知
3. **无重试**：这种情况下不会触发重试机制
4. **日志残缺**：`stringBody` 只包含截断前的部分内容，影响问题排查

**三类 Webhook 都受此影响**：
- 事件 Webhook：`responseWithoutBody.ok` 为 true 即走成功分支
- 旧版 SDK Webhook：同上
- 新版 SDK Webhook：同上

**配置值**：`maxContentSize = 1000` 字符（约 1KB）

---

## 五、三类 Webhook 对比汇总

### 5.1 重试策略对比

| 对比项 | 事件 Webhook | 旧版 SDK Webhook | 新版 SDK Webhook |
|-------|------------|----------------|----------------|
| **重试触发方式** | 主动调用 `retryJob()` | Agenda `fail` 事件监听 | Agenda `fail` 事件监听 |
| **重试次数** | 3 次 | 2 次 | 2 次 |
| **首次重试间隔** | 30 秒 | 30 秒 | 30 秒 |
| **后续重试间隔** | 5 分钟 × 2 次 | 5 分钟 × 1 次 | 5 分钟 × 1 次 |
| **熔断机制** | 无 | 连续失败过多自动禁用 | 连续失败过多自动禁用 |
| **重试代码位置** | `EventWebHookNotifier.ts:371` | `webhooks.ts:110` | `sdkWebhooks.ts:75` |

### 5.2 签名机制对比

| 对比项 | 事件 Webhook | 旧版 SDK Webhook | 新版 SDK Webhook |
|-------|------------|----------------|----------------|
| **签名层数** | 单层 | 单层 | 两层 |
| **算法** | HMAC-SHA256 hex | HMAC-SHA256 hex | HMAC-SHA256 hex + base64 |
| **签名头** | `X-GrowthBook-Signature` | `X-GrowthBook-Signature` | `webhook-signature` + `webhook-secret` |
| **时间戳防重放** | 无 | 无 | 有（`webhook-timestamp`） |
| **请求唯一标识** | 无 | 无 | 有（`webhook-id`） |
| **签名代码位置** | `event-webhooks-utils.ts:39` | `webhooks.ts:78` | `sdkWebhooks.ts:156` |

### 5.3 事件匹配逻辑对比

| 对比项 | 事件 Webhook（普通） | 事件 Webhook（test） | SDK Webhook |
|-------|---------------------|---------------------|------------|
| **匹配维度** | 事件名、标签、项目、环境 | 仅 webhookId | 项目、环境 |
| **检查 enabled** | 是 | 否 | 是 |
| **特殊路径** | 无 | webhook.test 绕过所有过滤 | 无 |

### 5.4 其他共性特性

| 特性 | 说明 |
|-----|------|
| **队列框架** | 全部使用 Agenda |
| **幂等保障** | `job.unique()` 防止重复入队 |
| **HTTP 客户端** | 统一使用 `cancellableFetch()` |
| **超时配置** | 30 秒超时，1000 字符响应体限制 |
| **审计日志** | 每次调用都记录详细日志 |
| **代理支持** | 支持通过代理发送请求 |
| **响应体超限处理** | 都被视为成功（状态码优先） |
