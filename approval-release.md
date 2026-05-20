# 特性开关审批-发布流程代码分析

## 一、整体流程概览

```
用户创建草稿(draft)
      ↓
[审批申请触发] → POST /feature/:id/:version/request (旧路由)
                      或
                  POST /v1/features/:id/revisions/:version/request-review (新路由)
      ↓
状态变为 pending-review
      ↓
[审阅状态机] → POST /feature/:id/:version/submit-review (旧路由)
                      或
                  POST /v1/features/:id/revisions/:version/submit-review (新路由)
      ↓
状态变为 approved / changes-requested
      ↓
[最终发布动作] → POST /feature/:id/:version/publish (旧路由)
                      或
                  POST /v1/features/:id/revisions/:version/publish (新路由)
      ↓
状态变为 published
      ↓
[Ramp Schedule 钩子] → dispatchRevisionPublishedHook
      ↓
关联的 Ramp Schedule 自动启动
```

---

## 二、API 路由体系：旧路由 vs 新路由

### 2.1 路由架构说明

系统存在两套并行的 API 路由体系：

| 维度 | 旧路由（Dashboard 专用） | 新路由（REST API 专用） |
|------|-------------------------|-------------------------|
| **路径格式** | `/feature/:id/:version/*` | `/v1/features/:id/revisions/:version/*` |
| **注册位置** | `app.ts:847-858` 直接挂载 | `features.router.ts:76-80` → `api.router.ts` |
| **处理入口** | `controllers/features.ts` 独立 controller | `api/features/*.ts` 通过 `createApiRequestHandler` 包装 |
| **鉴权方式** | Session + CSRF | API Key / OAuth |
| **使用方** | 前端 Dashboard | 外部 REST API 调用 |

### 2.2 路由注册与处理入口对照表

#### 申请审批

| 路由类型 | 路径 | 处理入口 | 挂载位置 |
|----------|------|----------|----------|
| 旧路由 | `POST /feature/:id/:version/request` | `controllers/features.ts:985` → `postFeatureRequestReview()` | `app.ts:852-854` |
| 新路由 | `POST /v1/features/:id/revisions/:version/request-review` | `api/features/postFeatureRevisionRequestReview.ts:81` → `postFeatureRevisionRequestReview` | `features.router.ts:76` |

#### 提交审阅

| 路由类型 | 路径 | 处理入口 | 挂载位置 |
|----------|------|----------|----------|
| 旧路由 | `POST /feature/:id/:version/submit-review` | `controllers/features.ts:1061` → `postFeatureReviewOrComment()` | `app.ts:855-858` |
| 新路由 | `POST /v1/features/:id/revisions/:version/submit-review` | `api/features/postFeatureRevisionSubmitReview.ts` → `postFeatureRevisionSubmitReview` | `features.router.ts:77` |

#### 发布

| 路由类型 | 路径 | 处理入口 | 挂载位置 |
|----------|------|----------|----------|
| 旧路由 | `POST /feature/:id/:version/publish` | `controllers/features.ts:1230` → `postFeaturePublish()` | `app.ts:847-850` |
| 新路由 | `POST /v1/features/:id/revisions/:version/publish` | `api/features/postFeatureRevisionPublish.ts:28` → `postFeatureRevisionPublish` | `features.router.ts:80` |

### 2.3 旧路由注册（app.ts）

**文件**：`packages/back-end/src/app.ts:847-858`

```typescript
// Features - 旧路由，Dashboard 专用
app.post(
  "/feature/:id/:version/publish",
  featuresController.postFeaturePublish,           // L849
);
app.post(
  "/feature/:id/:version/request",
  featuresController.postFeatureRequestReview,      // L853
);
app.post(
  "/feature/:id/:version/submit-review",
  featuresController.postFeatureReviewOrComment,    // L857
);
```

### 2.4 新路由注册（features.router.ts）

**文件**：`packages/back-end/src/api/features/features.router.ts:76-80`

```typescript
export const featureRoutes: OpenApiRoute[] = [
  // ...
  postFeatureRevisionRequestReview,  // L76 - /v1/features/:id/revisions/:version/request-review
  postFeatureRevisionSubmitReview,   // L77 - /v1/features/:id/revisions/:version/submit-review
  // ...
  postFeatureRevisionPublish,        // L80 - /v1/features/:id/revisions/:version/publish
  // ...
];
```

---

## 三、审批申请触发机制

### 3.1 前端触发入口

**文件**：`packages/front-end/components/Features/RequestReviewModal.tsx:192-206`

```typescript
if (!isPendingReview && !approved) {
  try {
    // 前端调用旧路由: /feature/:id/:version/request
    await apiCall(`/feature/${feature.id}/${revision?.version}/request`, {
      method: "POST",
      body: JSON.stringify({
        mergeResultSerialized: JSON.stringify(mergeResult),
        comment,
      }),
    });
  } catch (e) {
    mutate();
    throw e;
  }
  await mutate();
  close();
}
```

**触发条件**：
- 修订状态为 `draft`（不是 `pending-review` 也不是 `approved`）
- 用户拥有 `canManageFeatureDrafts` 权限

### 3.2 旧路由处理入口（Dashboard 使用）

**文件**：`packages/back-end/src/controllers/features.ts:985-1059`

```typescript
export async function postFeatureRequestReview(
  req: AuthRequest<{ comment: string }, { id: string; version: string }>,
  res: Response,
) {
  const context = getContextFromReq(req);
  const { id, version } = req.params;
  const { comment } = req.body;
  
  // 1. 权限检查: canManageFeatureDrafts (L1001-1003)
  // 2. 状态校验: 只有 draft 可以申请 (L1015-1017)
  // 3. 状态更新: markRevisionAsReviewRequested (L1018-1023)
  // 4. 审计记录: feature.revision.requestReview (L1034-1046)
  // 5. 事件分发: revision.reviewRequested (L1048-1054)
  
  res.status(200).json({ status: 200 });
}
```

### 3.3 新路由处理入口（REST API 使用）

**Validator 定义**：`packages/shared/src/validators/feature-revisions.ts:272-290`
```typescript
export const postFeatureRevisionRequestReviewValidator = {
  method: "post" as const,
  path: "/features/:id/revisions/:version/request-review",
  operationId: "postFeatureRevisionRequestReview",
  // ...
};
```

**Handler 包装**：`packages/back-end/src/api/features/postFeatureRevisionRequestReview.ts:81-86`
```typescript
export const postFeatureRevisionRequestReview = createApiRequestHandler(
  postFeatureRevisionRequestReviewValidator,
)(async (req) => {
  const { feature, revision } = await requestReview(req);
  return { revision: toApiRevision(revision, req.context, feature) };
});
```

**业务逻辑**：`packages/back-end/src/api/features/postFeatureRevisionRequestReview.ts:14-79`
```typescript
export async function requestReview(req) {
  // 与旧路由逻辑基本一致:
  // 1. 权限检查 (L25-27)
  // 2. 状态校验 (L38-42)
  // 3. 状态更新 (L44-49)
  // 4. 审计记录 (L60-68)
  // 5. 事件分发 (L70-76)
}
```

### 3.4 模型层状态变更

**文件**：`packages/back-end/src/models/FeatureRevisionModel.ts:1055-1092`

```typescript
export async function markRevisionAsReviewRequested(
  context: ReqContext | ApiReqContext,
  revision: FeatureRevisionInterface,
  user: EventUser,
  comment?: string,
) {
  await FeatureRevisionModel.updateOne(
    {
      organization: revision.organization,
      featureId: revision.featureId,
      version: revision.version,
    },
    {
      $set: {
        status: "pending-review",  // 关键状态变更
        datePublished: null,
        dateUpdated: new Date(),
        comment: comment,
      },
    },
  );
  // 记录 revision log
}
```

---

## 四、审阅状态机

### 4.1 状态定义

**文件**：`packages/shared/src/validators/features.ts:267-277`

```typescript
export const revisionStatusSchema = z.enum([
  "draft",           // 草稿
  "published",       // 已发布
  "discarded",       // 已废弃
  "approved",        // 已批准
  "changes-requested", // 需要修改
  "pending-review",  // 待审阅
  "pending-parent",  // 等待父修订（ramp schedule 多目标审批门控使用）
]);
```

### 4.2 pending-parent 状态分析

#### 状态语义

**文件**：`packages/shared/src/validators/features.ts:274-276`
```typescript
// Held child revision created by a ramp schedule; auto-published when the parent
// controller revision is approved/published. Not user-actionable directly.
"pending-parent",
```

#### 状态写入函数定义

**文件**：`packages/back-end/src/models/FeatureRevisionModel.ts:1353-1364`
```typescript
// Mark a revision as pending-parent so it waits for its sibling approval revision.
// Used by the ramp service when creating multi-target approval-gated steps.
export async function markRevisionAsPendingParent(
  organization: string,
  featureId: string,
  version: number,
): Promise<void> {
  await FeatureRevisionModel.updateOne(
    { organization, featureId, version },
    { $set: { status: "pending-parent" } },
  );
}
```

#### 调用点检索结论

> **重要事实**：在当前代码版本中，`markRevisionAsPendingParent` 函数**仅有定义，未检索到任何调用点**。
>
> 检索范围：
> - 全仓库 `import` 语句：无结果
> - 全仓库函数调用：无结果（排除文档引用）
> - Ramp Schedule 相关服务：未调用此函数
>
> 该函数为预留接口，用于 ramp schedule 多目标审批门控场景，但当前版本尚未实现实际调用逻辑。

#### 状态排除机制

**Schema 层面排除**：`packages/shared/src/validators/features.ts:281-285`
```typescript
export const activeDraftStatusSchema = revisionStatusSchema.exclude([
  "published",
  "discarded",
  "pending-parent", // Excluded — managed by ramp schedule, not user-actionable
]);
```

**前端列表过滤**：`packages/front-end/pages/approval-requests.tsx:194-199`
```typescript
// `pending-parent` revisions are held child revisions managed by ramp
// schedules and are not user-actionable, so they should not appear in the
// approvals list.
if (revision.status === "pending-parent") return null;
```

### 4.3 审阅提交流程

#### 前端调用

**文件**：`packages/front-end/components/Features/RequestReviewModal.tsx:662`
```typescript
`/feature/${feature.id}/${revision?.version}/submit-review`
```

#### 旧路由处理入口（Dashboard 使用）

**文件**：`packages/back-end/src/controllers/features.ts:1061-1228`
```typescript
export async function postFeatureReviewOrComment(
  req: AuthRequest<{
    comment: string;
    review?: ReviewSubmittedType;
  }, { id: string; version: string }>,
  res: Response,
) {
  const { comment, review = "Comment" } = req.body;
  
  // 1. 权限检查: canReviewFeatureDrafts (L1079-1081)
  // 2. 自审批禁止: 创建者不能审批自己 (L1095-1097)
  // 3. 贡献者自审批禁止 (L1100-1118)
  // 4. 状态更新: submitReviewAndComments (L1142-1147)
  // 5. 事件分发 (L1170-1220)
}
```

#### 新路由处理入口（REST API 使用）

**Validator 定义**：`packages/shared/src/validators/feature-revisions.ts:292-311`
```typescript
export const postFeatureRevisionSubmitReviewValidator = {
  method: "post" as const,
  path: "/features/:id/revisions/:version/submit-review",
  operationId: "postFeatureRevisionSubmitReview",
  bodySchema: z.object({
    comment: z.string().optional(),
    action: z.enum(["approve", "request-changes", "comment"]).optional(),
  }),
};
```

**Action 映射**：`packages/back-end/src/api/features/postFeatureRevisionSubmitReview.ts:15-19`
```typescript
export const actionToReviewType: Record<string, ReviewSubmittedType> = {
  approve: "Approved",
  "request-changes": "Requested Changes",
  comment: "Comment",
};
```

### 4.4 状态流转核心逻辑

**文件**：`packages/back-end/src/models/FeatureRevisionModel.ts:1094-1143`

```typescript
export async function submitReviewAndComments(
  context: ReqContext | ApiReqContext,
  revision: FeatureRevisionInterface,
  user: EventUser,
  reviewSubmittedType: ReviewSubmittedType,
  comment?: string,
) {
  let status = "pending-review";
  
  switch (reviewSubmittedType) {
    case "Approved":
      status = "approved";           // 通过 → approved
      break;
    case "Requested Changes":
      status = "changes-requested";  // 需要修改 → changes-requested
      break;
    default:
      // Comment 不改变状态
      status = revision.status;
  }

  await FeatureRevisionModel.updateOne(
    { /* 查询条件 */ },
    {
      $set: {
        status,
        datePublished: null,
        dateUpdated: new Date(),
      },
    },
  );
  // 记录 revision log
}
```

### 4.5 编辑时的状态重置

**文件**：`packages/back-end/src/models/FeatureRevisionModel.ts:866-907`

```typescript
export async function updateRevision(
  // ...
  resetReview: boolean,
) {
  let status = revision.status;

  if (hasMutableChange) {
    // 当内容发生变化时，如果状态是 changes-requested，自动重置为 pending-review
    if (revision.status === "changes-requested") {
      status = "pending-review";
    }
  }
  // 如果重置审阅标志为 true 且当前是 approved，也重置为 pending-review
  if (resetReview && revision.status === "approved") {
    status = "pending-review";
  }
  // ...
}
```

---

## 五、最终发布动作

### 5.1 前端发布触发

**文件**：`packages/front-end/components/Features/RequestReviewModal.tsx:207-224`

```typescript
} else if (approved) {
  try {
    // 前端调用旧路由: /feature/:id/:version/publish
    await apiCall(`/feature/${feature.id}/${revision?.version}/publish`, {
      method: "POST",
      body: JSON.stringify({
        mergeResultSerialized: JSON.stringify(mergeResult),
        comment,
        adminOverride: adminPublish,
        publishExperimentIds: Array.from(selectedExperiments),
      }),
    });
  } catch (e) {
    mutate();
    throw e;
  }
  await mutate();
  onPublish && onPublish();
  close();
}
```

### 5.2 旧路由处理入口（Dashboard 使用）

**文件**：`packages/back-end/src/controllers/features.ts:1230-1370`

```typescript
export async function postFeaturePublish(
  req: AuthRequest<{
    comment: string;
    mergeResultSerialized: string;
    adminOverride?: boolean;
    publishExperimentIds?: string[];
  }, { id: string; version: string }>,
  res: Response,
) {
  // 1. 状态校验: published/discarded 不能发布 (L1272-1280)
  // 2. 合并检查: autoMerge (L1299-1305)
  // 3. 审批需求检查: checkIfRevisionNeedsReview (L1321-1327)
  // 4. 审批状态校验 (L1328-1335)
  // 5. 权限检查: canPublishFeature (L1342-1344)
  // 6. 执行发布: publishRevision (L1346-1351)
  // 7. 审计记录 (L1353-1362)
}
```

**审批状态校验代码**（L1328-1335）：
```typescript
const canBypass =
  context.permissions.canBypassApprovalChecks(feature) && adminOverride;

if (requiresReview && revision.status !== "approved" && !canBypass) {
  throw new Error(
    `This feature requires approval before publishing (status: "${revision.status}")`
  );
}
```

### 5.3 新路由处理入口（REST API 使用）

**Validator 定义**：`packages/shared/src/validators/feature-revisions.ts:188-206`
```typescript
export const postFeatureRevisionPublishValidator = {
  method: "post" as const,
  path: "/features/:id/revisions/:version/publish",
  operationId: "postFeatureRevisionPublish",
  summary: "Publish a draft revision",
  // ...
};
```

**Handler 包装**：`packages/back-end/src/api/features/postFeatureRevisionPublish.ts:28-179`
```typescript
export async function publishFeatureRevision(req) {
  // 与旧路由逻辑基本一致
  // ...
}
```

### 5.4 审批需求判断逻辑

**文件**：`packages/shared/src/util/features.ts:1860-1939`

`checkIfRevisionNeedsReview` 函数判断规则：

1. **License 检查**：没有 require-approvals license 则不需要审批
2. **全局配置**：`settings.requireReviews` 为布尔值时直接使用
3. **项目级配置**：通过 `getReviewSetting` 匹配项目配置
4. **变更范围判断**：
   - 全局变更（`affected === "all"`）：非元数据变更始终需要审批，元数据变更看配置
   - 环境级变更：
     - 规则/数值变更始终需要审批（在配置的环境范围内）
     - 环境启用/禁用（kill switch）变更只有当 `featureRequireEnvironmentReview` 为 true 时需要

### 5.5 发布核心逻辑

**文件**：`packages/back-end/src/models/FeatureModel.ts:1888-1967`

`publishRevision` 函数执行流程：

1. **状态校验**：只能发布非 `published`/`discarded` 的修订
2. **前置创建**：先创建 ramp schedules（失败则整个发布中止）
3. **应用变更**：调用 `applyRevisionChanges` 更新 feature 主数据
4. **Holdout 副作用**：处理 holdout 关联变更
5. **标记发布**：调用 `markRevisionAsPublished` 将修订状态改为 `published`
6. **清理草稿**：调用 `clearPendingFeatureDraftsForRevision` 清理相关草稿（实验关联）
7. **异常回滚**：如果中间失败，回滚已创建的 ramp schedules
8. **后置处理**：应用 detach actions，清理孤儿 ramp schedules

### 5.6 修订状态标记为已发布

**文件**：`packages/back-end/src/models/FeatureRevisionModel.ts:998-1053`

```typescript
export async function markRevisionAsPublished(
  context: ReqContext | ApiReqContext,
  feature: FeatureInterface,
  revision: FeatureRevisionInterface,
  user: EventUser,
  comment?: string,
) {
  const changes: Partial<FeatureRevisionInterface> = {
    status: "published",    // 最终状态
    publishedBy: user,
    datePublished: new Date(),
    dateUpdated: new Date(),
    comment: revisionComment,
  };

  await FeatureRevisionModel.updateOne(
    { /* 查询条件 */ },
    { $set: changes },
  );
  
  // 触发 ramp schedule hook —— 这是发布后联动的关键
  await dispatchRevisionPublishedHook(context, revision);
}
```

---

## 六、发布后联动：Ramp Schedule 钩子机制

### 6.1 钩子注册机制

**类型定义**：`packages/back-end/src/models/FeatureRevisionModel.ts:1330-1339`
```typescript
type RevisionHook = (
  context: ReqContext | ApiReqContext,
  revision: FeatureRevisionInterface,
) => Promise<void>;

let _onRevisionPublishedHook: RevisionHook | null = null;

export function registerRevisionPublishedHook(hook: RevisionHook): void {
  _onRevisionPublishedHook = hook;
}
```

**钩子分发**：`packages/back-end/src/models/FeatureRevisionModel.ts:1341-1351`
```typescript
export async function dispatchRevisionPublishedHook(
  context: ReqContext | ApiReqContext,
  revision: FeatureRevisionInterface,
): Promise<void> {
  if (!_onRevisionPublishedHook) return;
  try {
    await _onRevisionPublishedHook(context, revision);
  } catch (e) {
    logger.error(e, "Error in revision published ramp hook");
  }
}
```

### 6.2 Ramp Schedule 钩子注册

**文件**：`packages/back-end/src/services/rampSchedule.ts:688-690`
```typescript
export function initRampScheduleHooks(): void {
  registerRevisionPublishedHook(onRevisionPublished);
}
```

### 6.3 钩子处理逻辑

**文件**：`packages/back-end/src/services/rampSchedule.ts:608-620`
```typescript
export async function onRevisionPublished(
  ctx: ReqContext | ApiReqContext,
  revision: FeatureRevisionInterface,
): Promise<void> {
  const activatingRamps =
    await ctx.models.rampSchedules.findByActivatingRevision(
      revision.featureId,
      revision.version,
    );
  for (const schedule of activatingRamps) {
    await onActivatingRevisionPublished(ctx, schedule);
  }
}
```

### 6.4 激活修订发布后的处理

**文件**：`packages/back-end/src/services/rampSchedule.ts:558-606`
```typescript
// Transitions a ramp from "pending" once its activating revision is published.
// No startDate (immediate) → auto-start; startDate set → transition to "ready".
export async function onActivatingRevisionPublished(
  ctx: ReqContext | ApiReqContext,
  schedule: RampScheduleInterface,
): Promise<void> {
  if (schedule.status !== "pending") return;

  const now = new Date();
  const isImmediate = !schedule.startDate || schedule.startDate <= now;

  if (isImmediate) {
    // 立即启动 ramp schedule
    let current = await ctx.models.rampSchedules.updateById(schedule.id, {
      status: "running",
      startedAt: now,
      phaseStartedAt: now,
      // ...
    });

    await applyRampStartActions(ctx, current);

    if (current.steps.length > 0) {
      await advanceUntilBlocked(ctx, current, now);
      // ...
    }
    // ...
  } else {
    // 有开始日期，转为 ready 状态等待
    await ctx.models.rampSchedules.updateById(schedule.id, {
      status: "ready",
      nextProcessAt: computeNextProcessAt({ ...schedule, status: "ready" }),
    });
  }
}
```

### 6.5 Ramp Schedule 创建时的关联

**文件**：`packages/back-end/src/models/FeatureModel.ts:1747-1764`
```typescript
const created = await context.models.rampSchedules.create({
  // ...
  targets: [
    {
      id: targetId,
      entityType: "feature",
      entityId: feature.id,
      ruleId: action.ruleId,
      status: "active",
      // Link this target to the activating revision so onRevisionPublished
      // (and the Agenda recovery path) can transition "pending" → "running".
      activatingRevisionVersion: revision.version,  // 关键关联
    },
  ],
  // ...
  // Start as "pending" — onActivatingRevisionPublished handles the
  // immediate → "running" transition inline when the revision publishes.
  status: "pending",
  // ...
});
```

---

## 七、三者之间的连接方式

### 7.1 核心连接点：revision.status 字段

整个流程通过 `FeatureRevisionModel` 的 `status` 字段串联：

```
draft → [requestReview] → pending-review → [submitReview] → approved → [publish] → published
                                          ↓
                                    changes-requested → [updateRevision] → pending-review
                                          ↓
                                    pending-parent → (预留状态，当前无调用)
```

### 7.2 API 端点完整连接表

| 动作 | 前端旧路由 | 新 REST API 路由 | 旧路由处理入口 | 新路由处理入口 | 模型方法 | 状态流转 |
|------|-----------|----------------|---------------|---------------|----------|----------|
| 申请审批 | `POST /feature/:id/:version/request` | `POST /v1/features/:id/revisions/:version/request-review` | `postFeatureRequestReview` (controllers/features.ts:985) | `requestReview` (api/features/postFeatureRevisionRequestReview.ts:14) | `markRevisionAsReviewRequested` | `draft` → `pending-review` |
| 提交审阅 | `POST /feature/:id/:version/submit-review` | `POST /v1/features/:id/revisions/:version/submit-review` | `postFeatureReviewOrComment` (controllers/features.ts:1061) | `submitRevisionReview` (api/features/postFeatureRevisionSubmitReview.ts:21) | `submitReviewAndComments` | `pending-review` → `approved` / `changes-requested` |
| 发布 | `POST /feature/:id/:version/publish` | `POST /v1/features/:id/revisions/:version/publish` | `postFeaturePublish` (controllers/features.ts:1230) | `publishFeatureRevision` (api/features/postFeatureRevisionPublish.ts:28) | `publishRevision` → `markRevisionAsPublished` | `approved` → `published` |

### 7.3 事件分发机制

**文件**：`packages/back-end/src/services/featureRevisionEvents.ts`

每个关键节点都会触发事件，用于通知和 webhook：

| 动作 | 事件类型 |
|------|----------|
| 申请审批 | `revision.reviewRequested` |
| 批准 | `revision.approved` |
| 要求修改 | `revision.changesRequested` |
| 评论 | `revision.commented` |
| 发布 | `revision.published` |
| 修订更新 | `revision.updated` |

### 7.4 审批要求的动态检查

审批要求不是静态的，而是在发布时**动态计算**的：

```
创建草稿时：checkIfRevisionNeedsReview() → 计算 requiresReview
发布时：再次调用 checkIfRevisionNeedsReview() → 验证当前状态是否满足要求
```

这样设计的好处是：即使审批配置在草稿创建后发生变化，发布时仍能按照最新配置进行检查。

### 7.5 旁路机制（Bypass）

系统提供两种绕过审批的方式（旧路由 L1321-1327，新路由 L107-109）：

1. **REST API 旁路**：`restApiBypassesReviews` 组织设置开启时，API 调用可以绕过审批
2. **权限旁路**：用户拥有 `canBypassApprovalChecks` 权限时可以绕过（通常是管理员）

---

## 八、关键代码路径汇总

### 8.1 审批申请完整路径（旧路由）
```
RequestReviewModal.tsx:192-206 → submitButton()
  ↓ API CALL (/feature/:id/:version/request)
app.ts:852-854 → 路由注册
  ↓
controllers/features.ts:985-1059 → postFeatureRequestReview()
  ├─ 权限检查: canManageFeatureDrafts
  ├─ 状态校验: status === "draft"
  ↓
FeatureRevisionModel.ts:1055-1092 → markRevisionAsReviewRequested()
  ↓
status: draft → pending-review
```

### 8.2 审批申请完整路径（新路由）
```
REST API Call → POST /v1/features/:id/revisions/:version/request-review
  ↓
features.router.ts:76 → 路由注册
  ↓
postFeatureRevisionRequestReview.ts:81 → createApiRequestHandler 包装
  ↓
postFeatureRevisionRequestReview.ts:14-79 → requestReview()
  ↓
FeatureRevisionModel.ts:1055-1092 → markRevisionAsReviewRequested()
  ↓
status: draft → pending-review
```

### 8.3 审阅批准完整路径（旧路由）
```
RequestReviewModal.tsx:225-226 → setShowSumbmitReview(true)
  ↓ API CALL (/feature/:id/:version/submit-review)
app.ts:855-858 → 路由注册
  ↓
controllers/features.ts:1061-1228 → postFeatureReviewOrComment()
  ├─ 权限检查: canReviewFeatureDrafts
  ├─ 自审批禁止
  ├─ 贡献者自审批禁止
  ↓
FeatureRevisionModel.ts:1094-1143 → submitReviewAndComments()
  ↓
status: pending-review → approved
```

### 8.4 发布完整路径（旧路由）
```
RequestReviewModal.tsx:207-224 → submitButton()
  ↓ API CALL (/feature/:id/:version/publish)
app.ts:847-850 → 路由注册
  ↓
controllers/features.ts:1230-1370 → postFeaturePublish()
  ├─ checkIfRevisionNeedsReview()  # 检查是否需要审批
  ├─ 验证 status === "approved" 或可以 bypass
  ↓
FeatureModel.ts:1888-1967 → publishRevision()
  ├─ applyRevisionChanges()        # 更新 feature 主数据
  └─ markRevisionAsPublished()     # 更新 revision 状态
     └─ dispatchRevisionPublishedHook()  # 触发 ramp schedule 钩子
        ↓
rampSchedule.ts:608-620 → onRevisionPublished()
  ↓
rampSchedule.ts:558-606 → onActivatingRevisionPublished()
  ↓
status: approved → published
关联的 ramp schedule 从 pending → running
```

### 8.5 pending-parent 状态说明
```
状态定义: features.ts:276 → "pending-parent"
函数定义: FeatureRevisionModel.ts:1355-1364 → markRevisionAsPendingParent()
调用点: 当前版本未检索到调用点（全仓库搜索无 import 或调用记录）
用途说明: 预留接口，用于 ramp schedule 多目标审批门控场景
排除机制:
  - Schema: activeDraftStatusSchema 排除 (features.ts:284)
  - 前端: approval-requests.tsx:199 过滤不显示
```
