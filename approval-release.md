# 特性开关审批-发布流程代码分析

## 一、整体流程概览

```
用户创建草稿(draft)
      ↓
[审批申请触发] → POST /feature/:id/:version/request
      ↓
状态变为 pending-review
      ↓
[审阅状态机] → POST /feature/:id/:version/submit-review
      ↓
状态变为 approved / changes-requested
      ↓
[最终发布动作] → POST /feature/:id/:version/publish
      ↓
状态变为 published
      ↓
[Ramp Schedule 钩子] → dispatchRevisionPublishedHook
      ↓
pending-parent 子修订自动发布
```

---

## 二、审批申请触发机制

### 2.1 前端触发入口

**文件**：`packages/front-end/components/Features/RequestReviewModal.tsx:192-206`

```typescript
if (!isPendingReview && !approved) {
  try {
    // 前端真实调用路径: /request
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

### 2.2 后端 API 定义与挂载

**后端路由挂载**：`packages/back-end/src/api/features/features.router.ts:76`
```typescript
export const featureRoutes: OpenApiRoute[] = [
  // ...
  postFeatureRevisionRequestReview,  // L76
  postFeatureRevisionSubmitReview,   // L77
  postFeatureRevisionPublish,        // L80
  // ...
];
```

**API 官方定义（V1）**：`packages/shared/src/validators/feature-revisions.ts:272-290`
```typescript
export const postFeatureRevisionRequestReviewValidator = {
  method: "post" as const,
  // 后端官方路径: /features/:id/revisions/:version/request-review
  path: "/features/:id/revisions/:version/request-review",
  operationId: "postFeatureRevisionRequestReview",
  // ...
};
```

**API Handler 包装**：`packages/back-end/src/api/features/postFeatureRevisionRequestReview.ts:81-86`
```typescript
export const postFeatureRevisionRequestReview = createApiRequestHandler(
  postFeatureRevisionRequestReviewValidator,
)(async (req) => {
  const { feature, revision } = await requestReview(req);
  return { revision: toApiRevision(revision, req.context, feature) };
});
```

> **注意**：前端使用的路径 `/feature/:id/:version/request` 与后端 validator 定义的 `/features/:id/revisions/:version/request-review` 不同，这是 V1 API 的路径别名机制，实际由 `createApiRequestHandler` 根据 validator 中的 `path` 字段进行路由匹配。

### 2.3 后端业务处理逻辑

**文件**：`packages/back-end/src/api/features/postFeatureRevisionRequestReview.ts:14-79`

核心函数 `requestReview` 执行流程：

1. **权限检查**：检查用户是否有 `canManageFeatureDrafts` 权限（L25-27）
2. **状态校验**：只有 `draft` 状态的修订才能申请审批（L38-42）
3. **状态更新**：调用 `markRevisionAsReviewRequested` 将状态改为 `pending-review`（L44-49）
4. **审计记录**：记录 `feature.revision.requestReview` 审计事件（L60-68）
5. **事件分发**：触发 `revision.reviewRequested` 事件通知（L70-76）

### 2.4 模型层状态变更

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

## 三、审阅状态机

### 3.1 状态定义

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

export const ACTIVE_DRAFT_STATUSES = [
  "draft",
  "approved",
  "changes-requested",
  "pending-review",
];

// 状态优先级（用于多草稿时的展示）
const DRAFT_STATUS_PRIORITY: Record<ActiveDraftStatus, number> = {
  "changes-requested": 4,
  "pending-review": 3,
  approved: 2,
  draft: 1,
};
```

### 3.2 pending-parent 状态说明

**状态语义**：`packages/shared/src/validators/features.ts:274-276`
```typescript
// Held child revision created by a ramp schedule; auto-published when the parent
// controller revision is approved/published. Not user-actionable directly.
"pending-parent",
```

**状态写入函数**：`packages/back-end/src/models/FeatureRevisionModel.ts:1353-1364`
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

**排除用户操作**：`packages/shared/src/validators/features.ts:281-285`
```typescript
export const activeDraftStatusSchema = revisionStatusSchema.exclude([
  "published",
  "discarded",
  "pending-parent", // Excluded — managed by ramp schedule, not user-actionable
]);
```

**前端过滤**：`packages/front-end/pages/approval-requests.tsx:194-199`
```typescript
// `pending-parent` revisions are held child revisions managed by ramp
// schedules and are not user-actionable, so they should not appear in the
// approvals list.
if (revision.status === "pending-parent") return null;
```

### 3.3 审阅提交流程

**前端调用路径**：`packages/front-end/components/Features/RequestReviewModal.tsx:662`
```typescript
`/feature/${feature.id}/${revision?.version}/submit-review`
```

**后端 API 定义**：`packages/shared/src/validators/feature-revisions.ts:292-311`
```typescript
export const postFeatureRevisionSubmitReviewValidator = {
  method: "post" as const,
  path: "/features/:id/revisions/:version/submit-review",
  operationId: "postFeatureRevisionSubmitReview",
  // ...
  bodySchema: z.object({
    comment: z.string().optional(),
    action: z.enum(["approve", "request-changes", "comment"]).optional(),
  }),
};
```

**后端业务处理**：`packages/back-end/src/api/features/postFeatureRevisionSubmitReview.ts:21-123`

核心函数 `submitRevisionReview` 执行流程：

1. **权限检查**：检查用户是否有 `canReviewFeatureDrafts` 权限（L33-35）
2. **自审批禁止**：创建者不能审批自己的草稿（L50-57）
3. **贡献者自审批禁止**：当 `blockSelfApproval` 配置开启时，贡献者不能审批自己参与的草稿（L61-76）
4. **状态校验**：只有 `pending-review`、`changes-requested`、`approved` 状态可以提交审阅（L78-87）
5. **状态更新**：调用 `submitReviewAndComments` 更新状态（L89-95）
6. **事件分发**：触发对应的审批事件（L112-120）

### 3.4 状态流转核心逻辑

**文件**：`packages/back-end/src/models/FeatureRevisionModel.ts:1094-1143`

```typescript
export async function submitReviewAndComments(
  context: ReqContext | ApiReqContext,
  revision: FeatureRevisionInterface,
  user: EventUser,
  reviewSubmittedType: ReviewSubmittedType,
  comment?: string,
) {
  const action = reviewSubmittedType;
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

**Action 映射**：`packages/back-end/src/api/features/postFeatureRevisionSubmitReview.ts:15-19`
```typescript
export const actionToReviewType: Record<string, ReviewSubmittedType> = {
  approve: "Approved",
  "request-changes": "Requested Changes",
  comment: "Comment",
};
```

### 3.5 编辑时的状态重置

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

## 四、最终发布动作

### 4.1 前端发布触发

**文件**：`packages/front-end/components/Features/RequestReviewModal.tsx:207-224`

```typescript
} else if (approved) {
  try {
    // 前端真实调用路径: /publish
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

### 4.2 后端 API 定义

**文件**：`packages/shared/src/validators/feature-revisions.ts:188-206`
```typescript
export const postFeatureRevisionPublishValidator = {
  method: "post" as const,
  path: "/features/:id/revisions/:version/publish",
  operationId: "postFeatureRevisionPublish",
  summary: "Publish a draft revision",
  // ...
};
```

### 4.3 后端发布前置检查

**文件**：`packages/back-end/src/api/features/postFeatureRevisionPublish.ts:28-179`

核心函数 `publishFeatureRevision` 执行流程：

1. **状态校验**：`published` 或 `discarded` 状态不能再次发布（L50-54）
2. **合并检查**：执行 `autoMerge` 检查是否有冲突（L67-80）
3. **审批需求检查**：调用 `checkIfRevisionNeedsReview` 判断是否需要审批（L96-104）
4. **审批状态校验**：
   ```typescript
   if (requiresReview && revision.status !== "approved" && !canBypass) {
     throw new BadRequestError(
       `This revision requires approval before publishing (status: "${revision.status}")`
     );
   }
   ```
   （L111-117）
5. **权限检查**：检查 `canPublishFeature` 权限（L126-128）
6. **执行发布**：调用 `publishRevision`（L130-136）
7. **审计记录**：记录 `feature.publish` 审计事件（L149-159）
8. **事件分发**：触发 `revision.published` 事件（L170-176）

### 4.4 审批需求判断逻辑

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

### 4.5 发布核心逻辑

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

### 4.6 修订状态标记为已发布

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
  
  // 触发 ramp schedule hook —— 这是 pending-parent 衔接的关键
  await dispatchRevisionPublishedHook(context, revision);
}
```

---

## 五、pending-parent 与发布动作的衔接

### 5.1 钩子注册机制

**钩子类型定义**：`packages/back-end/src/models/FeatureRevisionModel.ts:1330-1339`
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

### 5.2 Ramp Schedule 钩子注册

**文件**：`packages/back-end/src/services/rampSchedule.ts:688-690`
```typescript
export function initRampScheduleHooks(): void {
  registerRevisionPublishedHook(onRevisionPublished);
}
```

### 5.3 钩子处理逻辑

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

### 5.4 激活修订发布后的处理

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

### 5.5 Ramp Schedule 创建时的关联

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

### 5.6 完整衔接流程

```
1. 创建带 Ramp Schedule 的修订时：
   - 创建 ramp schedule，status = "pending"
   - 设置 activatingRevisionVersion = revision.version
   - 如果是多目标审批门控，创建子修订并标记 status = "pending-parent"

2. 父修订审批流程：
   - draft → pending-review → approved

3. 父修订发布时：
   - publishRevision() 被调用
   - markRevisionAsPublished() 将 status 改为 "published"
   - dispatchRevisionPublishedHook() 触发钩子
   - onRevisionPublished() 查找关联的 ramp schedules
   - onActivatingRevisionPublished() 启动 ramp schedule
   - pending-parent 子修订随 ramp schedule 自动发布
```

---

## 六、三者之间的连接方式

### 6.1 核心连接点：revision.status 字段

整个流程通过 `FeatureRevisionModel` 的 `status` 字段串联：

```
draft → [requestReview] → pending-review → [submitReview] → approved → [publish] → published
                                          ↓
                                    changes-requested → [updateRevision] → pending-review
                                          ↓
                                    pending-parent → [ramp hook] → 自动发布
```

### 6.2 API 端点连接（真实路径 vs 官方路径）

| 动作 | 前端调用路径 | 后端官方路径（Validator） | 模型方法 | 状态流转 |
|------|-------------|--------------------------|----------|----------|
| 申请审批 | `POST /feature/:id/:version/request` | `POST /features/:id/revisions/:version/request-review` | `markRevisionAsReviewRequested` | `draft` → `pending-review` |
| 提交审阅 | `POST /feature/:id/:version/submit-review` | `POST /features/:id/revisions/:version/submit-review` | `submitReviewAndComments` | `pending-review` → `approved` / `changes-requested` |
| 发布 | `POST /feature/:id/:version/publish` | `POST /features/:id/revisions/:version/publish` | `publishRevision` → `markRevisionAsPublished` | `approved` → `published` |

> **路径差异说明**：前端使用简化路径 `/feature/:id/:version/*`，后端 validator 定义完整路径 `/features/:id/revisions/:version/*`，由 `createApiRequestHandler` 根据 validator 中的 `path` 字段进行路由匹配。

### 6.3 事件分发机制

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

### 6.4 审批要求的动态检查

审批要求不是静态的，而是在发布时**动态计算**的：

```
创建草稿时：checkIfRevisionNeedsReview() → 计算 requiresReview
发布时：再次调用 checkIfRevisionNeedsReview() → 验证当前状态是否满足要求
```

这样设计的好处是：即使审批配置在草稿创建后发生变化，发布时仍能按照最新配置进行检查。

### 6.5 旁路机制（Bypass）

系统提供两种绕过审批的方式（L107-109 in postFeatureRevisionPublish.ts）：

1. **REST API 旁路**：`restApiBypassesReviews` 组织设置开启时，API 调用可以绕过审批
2. **权限旁路**：用户拥有 `canBypassApprovalChecks` 权限时可以绕过（通常是管理员）

---

## 七、关键代码路径汇总

### 7.1 审批申请完整路径
```
RequestReviewModal.tsx:192-206 → submitButton()
  ↓ API CALL (/feature/:id/:version/request)
postFeatureRevisionRequestReview.ts:14-79 → requestReview()
  ↓
FeatureRevisionModel.ts:1055-1092 → markRevisionAsReviewRequested()
  ↓
status: draft → pending-review
```

### 7.2 审阅批准完整路径
```
RequestReviewModal.tsx:225-226 → setShowSumbmitReview(true)
  ↓ API CALL (/feature/:id/:version/submit-review)
postFeatureRevisionSubmitReview.ts:21-123 → submitRevisionReview()
  ↓
FeatureRevisionModel.ts:1094-1143 → submitReviewAndComments()
  ↓
status: pending-review → approved
```

### 7.3 发布完整路径
```
RequestReviewModal.tsx:207-224 → submitButton()
  ↓ API CALL (/feature/:id/:version/publish)
postFeatureRevisionPublish.ts:28-179 → publishFeatureRevision()
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
pending-parent 子修订随 ramp schedule 自动发布
```

### 7.4 pending-parent 状态衔接路径
```
创建带 ramp schedule 的修订时：
  FeatureModel.ts:1747-1764 → createRampSchedulesForRevision()
    ├─ 创建 ramp schedule，status = "pending"
    ├─ 设置 activatingRevisionVersion = revision.version
    └─ （多目标审批门控时）markRevisionAsPendingParent() → status = "pending-parent"

父修订发布时：
  FeatureRevisionModel.ts:998-1053 → markRevisionAsPublished()
    └─ dispatchRevisionPublishedHook()
        ↓
rampSchedule.ts:688-690 → initRampScheduleHooks() 注册的钩子被触发
        ↓
rampSchedule.ts:608-620 → onRevisionPublished()
        ↓
rampSchedule.ts:558-606 → onActivatingRevisionPublished()
        ↓
ramp schedule 从 pending → running
pending-parent 子修订自动发布
```
