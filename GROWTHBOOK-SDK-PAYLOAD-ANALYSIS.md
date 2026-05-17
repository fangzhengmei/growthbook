# GrowthBook 特性开关 SDK Payload 概念级分析报告

## 1. 概述

GrowthBook 是一个开源的特性开关与 A/B 测试平台。其核心数据流是：**后台定义的规则 → 编排打包成 SDK Payload → 多语言 SDK 拉取并在客户端执行评估**。

本报告聚焦于中间的规则编排、缓存与版本同步机制，以及后台与 SDK 之间的协议层契约。

---

## 2. 规则编排逻辑

### 2.1 规则模型演进（v0 → v1 → v2）

GrowthBook 的特性（Feature）在磁盘上经历了三代存储格式的演进：

| 版本 | 存储结构 | 关键特征 |
|------|---------|---------|
| v0 | 顶级 `rules` 数组 + `environments` 字符串数组 | 无环境隔离，规则全局生效 |
| v1 | `environmentSettings[env].rules` 按环境存储规则数组 | 每个环境独立规则集合，规则无跨环境标识 |
| v2 | 顶级 `rules` 数组 + 规则带 `allEnvironments` / `environments` 字段 | 规则扁平化，单条规则可作用于多个环境 |

**JIT 迁移**：读取时通过 `FeatureModel.toInterface` 将所有旧格式统一归一化为 v2 格式；写入时总是输出 v2 格式。

### 2.2 规则类型系统

每条 `FeatureRule` 可能是以下类型之一（通过隐式字段区分）：

- **Force 规则**：`force` 字段存在 —— 直接返回固定值
- **Rollout 规则**：`coverage` + 无 `variations` —— 按百分比放量
- **Experiment 规则**：`variations` + `weights` —— A/B 测试分流
- **ExperimentRef 规则**：`experimentId` —— 引用独立的实验对象
- **Schedule 规则**：`schedule` —— 按时间调度生效

### 2.3 规则的组合与优先级

规则在顶级 `rules` 数组中按顺序排列，SDK 评估时**自上而下匹配，第一条命中的规则生效**。

规则可包含的约束维度：
```
FeatureRule
├── condition: ConditionInterface      // 用户属性条件（MongoRule 语法）
├── parentConditions: ParentCondition[] // 依赖其他特性的前置条件
├── savedGroups: SavedGroupTargeting[]  // 用户分群匹配
├── coverage: number                    // 流量百分比
├── namespace: NamespaceValue           // 命名空间隔离
└── environments / allEnvironments      // 生效环境
```

### 2.4 修订系统（Revision）

特性变更不直接覆盖，而是通过 `FeatureRevision` 进行版本管理：

```
特性（Feature）
└── 版本号（version）：每次发布递增
    └── 修订（FeatureRevision）
        ├── status: draft / pending-review / approved / published / discarded
        ├── baseVersion：基于哪个已发布版本
        ├── rules：变更后的规则集
        ├── defaultValue：变更后的默认值
        └── environmentsEnabled：各环境开关状态
```

**发布流程**：
1. 创建草稿（draft）→ 可请求审核（pending-review）→ 审核通过（approved）→ 发布（published）
2. 发布时执行 `autoMerge` 进行三路合并（live ← base ← draft）
3. 合并成功后递增特性 `version`，标记修订为 `published`

---

## 3. 缓存机制与版本同步

### 3.1 SDK Payload 缓存架构

```
SDK 请求 → /api/features/:key （公共端点，无认证）
    ↓
getFeaturesPublic()
    ↓
getFeatureDefinitionsWithCache()
    ├─ 查 sdkConnectionCache（MongoDB 持久化缓存，集合名：sdkcache）
    │   ├─ 命中 → 直接返回
    │   └─ 未命中 → 实时构建
    └─ 实时构建 → buildSDKPayloadForConnection()
        └─ 异步写回缓存（fire-and-forget）
```

**缓存键**：
- 新 SDK Connection：`sdk-` 前缀的唯一 key（如 `sdk-abc123`）
- 旧版 API Key：`legacy:{apiKey}:{environment}:{project}` 格式合成

**缓存存储位置**：
- 默认：MongoDB（`SDK_PAYLOAD_CACHE=mongo`）
- 可配置为禁用（`SDK_PAYLOAD_CACHE=none`）
- 设计预留了 S3/GCS 后端扩展点（TODO 注释）

### 3.2 缓存失效与主动刷新

缓存不是被动等待 TTL，而是**变更驱动的主动刷新**。

触发缓存刷新的入口点（通过 `queueSDKPayloadRefresh` 异步触发）：
- `FeatureModel`：特性创建、更新、删除、发布、回滚
- `ExperimentModel`：实验状态变更
- `SavedGroupModel`：用户分群变更
- `EnvironmentModel`：环境配置变更
- `HoldoutModel`：Holdout 配置变更
- `ProjectModel`：项目配置变更
- `SdkConnectionModel`：SDK 连接配置变更
- 定时任务：`updateScheduledFeatures`（调度规则生效）、`updateRampSchedules`（渐进式放量）
- `CustomFieldModel`：自定义字段定义变更

**刷新优化**：
1. 计算受影响的 `payloadKeys`（`{environment, project}` 组合）
2. 仅刷新环境匹配、项目范围匹配的 SDK Connection
3. 批量处理：`promiseAllChunks(promises, 4)` 控制并发，避免 DB 风暴
4. 触发后异步执行，不阻塞用户请求

### 3.3 版本同步机制

**后端侧**：
- 特性 `version` 单调递增，每次发布 +1
- `dateUpdated` 时间戳随变更更新
- 修订系统保留完整历史，支持回滚到任意已发布版本

**SDK 侧**：
- Payload 带 `dateUpdated` 时间戳
- SDK 可通过 `stale-while-revalidate` 策略后台刷新
- 支持 SSE 流式更新（`startStreaming`）实时接收变更

---

## 4. 协议层契约

### 4.1 SDK 公共接口契约

| 端点 | 方法 | 认证 | 用途 | 可用环境 |
|------|------|------|------|---------|
| `/api/features/:key` | GET | 否（CORS 开放） | SDK 拉取 Payload（本地评估模式） | Self-hosted + Cloud |
| `/api/eval/:key` | POST | 否（CORS 开放） | 远程评估模式：提交 attributes，返回评估结果 | **Self-hosted ONLY** |
| `/api/sdk-payload/:key` | GET | 是（内部 API） | 后台管理用，非 SDK 直接调用 | 内部 |
| `/api/v1/sdk-payload/:key` | GET | 是（内部 API） | v1 路径别名 | 内部 |

> **重要修正**：之前版本提到的 `/api/eval-features/:key` 是错误路径，正确公共端点为 `/api/eval/:key`，且仅在 self-hosted 环境下通过 `if (!IS_CLOUD)` 守卫启用。Cloud 环境必须使用独立的远程评估基础设施。

**响应头（CDN 缓存控制）**：
```
Cache-Control: public, max-age=30, stale-while-revalidate=3600, stale-if-error=36000
Surrogate-Key: {orgId} {apiKey} {envId}  // Fastly 清缓存用
x-unrecoverable: 1                       // 不可恢复错误时标记，便于 CDN 缓存错误响应
```

### 4.2 SDK Connection 配置（后端 → Payload 生成参数）

每个 SDK Connection 定义了 Payload 的生成参数，这是后台与 SDK 之间的**配置契约**：

```typescript
SDKConnectionInterface {
  // 基础标识
  key: string;                    // SDK 拉取用的唯一 key
  environment: string;            // 目标环境（production / staging 等）
  projects: string[];             // 项目过滤
  
  // SDK 能力声明（影响 Payload 格式）
  languages: SDKLanguage[];       // SDK 语言：javascript, python, go, java 等（支持多值）
  sdkVersion: string;             // SDK 版本号，用于能力协商
  
  // Payload 内容开关
  includeVisualExperiments?: boolean;
  includeDraftExperiments?: boolean;
  includeExperimentNames?: boolean;
  includeRedirectExperiments?: boolean;
  includeRuleIds?: boolean;
  savedGroupReferencesEnabled?: boolean;
  
  // 元数据开关
  includeProjectIdInMetadata?: boolean;
  includeCustomFieldsInMetadata?: boolean;
  allowedCustomFieldsInMetadata?: string[];
  includeTagsInMetadata?: boolean;
  
  // 安全选项
  encryptPayload: boolean;
  encryptionKey: string;
  hashSecureAttributes?: boolean;
  
  // 评估模式
  remoteEvalEnabled?: boolean;    // true = 后端评估，false = SDK 本地评估
}
```

### 4.3 能力协商机制（SDK Capabilities）

#### 单语言场景

当 `languages` 数组长度 ≤ 1 时：
```typescript
capabilities = getSDKCapabilities(language, sdkVersion)
```

根据语言和版本号，累积该版本及之前所有版本引入的能力。

#### 多语言场景（交集策略）

当 `languages` 数组长度 > 1 时（`getConnectionSDKCapabilities` 函数，shared/src/sdk-versioning/index.ts:162-199）：

```
输入：languages = ["javascript", "python", "go"], sdkVersion = "1.5.0"

步骤1：忽略 sdkVersion，每种语言使用最小默认版本（0.0.0）
       确保不依赖任何语言的高版本独有特性

步骤2：对每种语言分别计算能力集
       javascript @ 0.0.0 → {bucketingV2, looseUnmarshalling}
       python @ 0.0.0     → {bucketingV2, looseUnmarshalling, prerequisites}
       go @ 0.0.0         → {bucketingV2}

步骤3：取所有语言能力的交集
       {bucketingV2, looseUnmarshalling} ∩ {bucketingV2, looseUnmarshalling, prerequisites} ∩ {bucketingV2}
       = {bucketingV2}

输出：capabilities = ["bucketingV2"]
```

**关键规则**：
- 多语言配置下 `sdkVersion` 被忽略，强制使用最小版本（`undefined` → 默认版本）
- 最终能力 = 所有语言能力的**交集**
- 这确保生成的 Payload 能被连接中的**所有**语言 SDK 正确解析
- 能力缺失意味着对应的 Payload 字段会被裁剪或转换

**能力对 Payload 裁剪的影响**：

| 能力缺失 | 裁剪/转换行为 |
|---------|-------------|
| `savedGroupReferences` | 将条件中的 `$inGroup` 内联展开为 `$in`，把用户 ID 列表直接嵌入规则 |
| `prerequisites` | 过滤掉所有带 `parentConditions` 的规则，或在生成时剔除前置条件 |
| `redirects` | 过滤掉所有 URL 重定向实验 |
| `visualExperiments` | 过滤掉所有可视化实验 |
| `bucketingV2` | 降级使用 v1 分桶算法 |

### 4.4 Payload 输出格式（后端 → SDK）

**核心类型**：`FeatureDefinitionSDKPayload`（后端）↔ `FeatureApiResponse`（SDK）

```typescript
// 后端输出
FeatureDefinitionSDKPayload {
  features: Record<string, FeatureDefinition>;    // 特性定义字典
  experiments?: AutoExperiment[];                  // 可视化/重定向实验
  dateUpdated: Date | null;                        // Payload 生成时间
  
  // 加密字段（encryptPayload = true 时）
  encryptedFeatures?: string;                      // AES-GCM 加密的 features JSON
  encryptedExperiments?: string;                   // AES-GCM 加密的 experiments JSON
  encryptedSavedGroups?: string;                   // AES-GCM 加密的 savedGroups JSON
  
  // 保存组（savedGroupReferencesEnabled = true 时）
  savedGroups?: SavedGroupsValues;                 // 保存组 ID → 值列表映射
}

// 特性定义
FeatureDefinition {
  defaultValue: any;                               // 默认值
  rules?: FeatureDefinitionRule[];                 // 规则数组
  metadata?: {                                     // 可选元数据
    projects?: string[];
    customFields?: Record<string, unknown>;
    tags?: string[];
  };
}

// 单条规则
FeatureDefinitionRule {
  id?: string;                                     // 规则 ID（includeRuleIds = true 时）
  condition?: ConditionInterface;                  // 用户属性条件（MongoRule）
  parentConditions?: ParentConditionInterface[];   // 前置条件
  force?: T;                                       // Force 规则值
  variations?: T[];                                // 实验变体
  weights?: number[];                              // 实验权重
  coverage?: number;                               // 流量覆盖率
  key?: string;                                    // 实验 trackingKey
  hashAttribute?: string;                          // 分桶属性（如 "id", "email"）
  hashVersion?: number;                            // 分桶算法版本（1 或 2）
  seed?: string;                                   // 分桶种子
  range?: VariationRange;                          // 单变体分桶范围 [start, end)
  ranges?: VariationRange[];                       // 多变体分桶范围
  meta?: VariationMeta[];                          // 变体元数据（key, name）
  phase?: string;                                  // 实验阶段索引
}
```

### 4.5 多语言 SDK 的统一契约

所有语言 SDK 实现相同的评估逻辑，遵循同一套协议：

1. **加载 Payload**：`loadFeatures()` → 发送 GET 请求到 `/api/features/:key`
2. **本地评估**：`evalFeature(featureKey)` → 按规则顺序匹配，返回结果
3. **跟踪曝光**：`trackingCallback` → 上报实验分配事件
4. **刷新机制**：`refreshFeatures()` → 手动触发；或 SSE 流式更新

**加密支持**：
- 后端使用 `encryptionKey` 加密 payload（AES-GCM）
- SDK 端使用相同 key 解密
- 加密时 `features`、`experiments`、`savedGroups` 字段为空数组/对象，真实数据在 `encrypted*` 字段中
- 远程评估模式（`remoteEvalEnabled=true`）禁用加密

---

## 5. 端到端时序与不一致窗口

```
时间轴 →
  │
  T0  用户在后台 UI 点击「发布」按钮
  │   ↓
  │   POST /api/v1/features/:id/revisions/:version/publish
  │   ├─ 权限校验
  │   ├─ autoMerge 三路合并（live ← base ← draft）
  │   ├─ 更新 Feature 主文档，version += 1
  │   └─ Revision 标记为 published
  │
  T1  [queueSDKPayloadRefresh()] 被调用（异步，不阻塞 HTTP 响应）
  │   ├─ 计算受影响的 payloadKeys = [{env: "production", project: ""}]
  │   ├─ 查询所有匹配的 SDK Connection（假设有 5 个）
  │   └─ 提交到后台队列，立即返回 HTTP 200
  │
  ├─────────────────────────────────────────────────────────────────
  │                     🔴 不一致窗口开始（T1 ~ T3）
  │                     SDK 仍拉取到旧版本缓存
  │
  T2  后台 worker 开始执行 refreshSDKPayloadCache()
  │   ├─ 拉取全量数据：features, experiments, savedGroups, holdouts...
  │   ├─ 为每个 Connection 构建 payload（buildSDKPayloadForConnection）
  │   ├─ 并发控制：promiseAllChunks(promises, 4)
  │   └─ 逐个写入 sdkConnectionCache
  │
  T3  最后一个 SDK Connection 缓存更新完成
  │
  ├─────────────────────────────────────────────────────────────────
  │                     🟢 不一致窗口结束
  │                     新的 SDK 请求将获取新版本
  │
  T4  SDK 客户端发起 GET /api/features/:key
  │   ├─ CDN 层：max-age=30s，可能还缓存着旧版本（如果 T3-T0 < 30s）
  │   ├─ 源站：命中新缓存 → 返回新版 payload
  │   └─ SDK 本地：stale-while-revalidate，先返回旧版再后台刷新
  │
  T5  SDK 本地评估 → 用户看到新特性值
```

**不一致窗口分析**：

| 阶段 | 持续时间 | 影响范围 | 缓解措施 |
|------|---------|---------|---------|
| T1 → T3 | 通常 1~5 秒，取决于 Connection 数量和 DB 性能 | 所有使用该 Connection 的 SDK | 异步刷新，批量并发 4 |
| T3 → CDN 失效 | 最长 30 秒（max-age） | Cloud/CDN 部署环境 | stale-while-revalidate 允许后台刷新 |
| SDK 本地缓存 | 取决于 SDK 刷新策略 | 单个客户端 | 手动 refreshFeatures() 或 SSE 流式更新 |

**最坏情况不一致窗口**：约 30 秒（CDN max-age）+ SDK 本地缓存时间。

---

## 6. 关键设计决策

### 6.1 缓存优先 + 主动刷新
- 读路径：缓存优先，未命中时实时构建并异步写回
- 写路径：变更后立即触发所有相关缓存的刷新
- 权衡：读路径延迟极低（< 10ms），但写路径有放大效应（一个特性变更可能触发数十个 Connection 的缓存刷新）

### 6.2 能力协商而非版本锁定
- 不采用 "v1 API / v2 API" 的硬版本切分
- 而是通过 SDK 声明的 `languages` + `sdkVersion` 动态推导能力
- 多语言时取交集，确保最低共同兼容性
- 向后兼容：旧 SDK 收到的是它能理解的子集

### 6.3 扁平规则（v2）替代按环境存储（v1）
- 解决跨环境规则重复定义问题
- 相同内容的规则跨环境合并，减少 payload 体积
- 规则 ID 全局稳定，便于追踪和调试

### 6.4 修订系统作为变更总线
- 所有特性变更必须经过 Revision 流程
- 支持审核、回滚、冲突合并
- 发布时才真正写入 Feature 主文档并触发缓存刷新

### 6.5 公共端点与内部端点分离
- `/api/features/:key` 完全公开，CORS 开放，便于浏览器端 SDK 使用
- `/api/eval/:key` 仅限 self-hosted，Cloud 通过独立基础设施提供远程评估
- 内部管理 API（`/api/sdk-payload/:key`）需要认证，不对外暴露

---

## 7. 边界情况与注意事项

1. **缓存穿透**：大量无效 API Key 会导致每次都走实时构建路径，可能打垮后端
2. **风暴效应**：大规模实验发布时，数十个 SDK Connection 同时刷新，DB 压力骤增（通过 `promiseAllChunks(4)` 限流缓解）
3. **最终一致性**：缓存刷新是异步的，SDK 可能在数秒内拿到旧版本（通过 `stale-while-revalidate` 平衡）
4. **能力矩阵爆炸**：15+ 种 SDK 语言 × 无数个版本号，能力推导逻辑需精心维护
5. **加密密钥管理**：`encryptionKey` 存储在 SDK Connection 中，泄露风险需控制
6. **多语言能力退化**：多语言配置下能力取交集，可能导致高级特性（如前置条件、保存组引用）无法使用
7. **远程评估边界**：Cloud 环境禁用 `/api/eval/:key`，需注意部署环境差异

---

## 8. 总结

GrowthBook 的 SDK Payload 机制是一个**配置驱动、缓存优先、能力协商、最终一致**的分布式系统设计：

- **规则编排**：通过 v2 扁平规则 + Revision 版本管理，实现灵活的跨环境规则组合
- **缓存同步**：变更驱动的主动刷新 + CDN 边缘缓存，平衡了实时性与性能
- **协议契约**：SDK Connection 作为配置契约，能力协商（单语言取并集、多语言取交集）实现多语言 SDK 的向后兼容
- **扩展点**：Webhook、SSE 流式更新、远程评估等机制支持更复杂的集成场景
- **部署边界**：Self-hosted 与 Cloud 在远程评估端点上有明确差异，需注意能力对齐

这套设计使得 GrowthBook 能够支撑从单项目到超大规模企业的特性管理需求，同时保持 SDK 的轻量级和高性能。
