# 特性开关审批-发布流程代码分析

## 一、整体流程概览

```
用户创建草稿(draft)
      ↓
[审批申请触发] → POST /feature/:id/:version/request-review
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
```

---

## 二、审批申请触发机制

### 2.1 前端触发入口

**文件**：`packages/front-end/components/Features/RequestReviewModal.tsx:184-206`

```typescript
const submitButton = async () => {
  // ...
  if (!isPendingReview && !approved) {
    // 触发审批申请
    await apiCall(`/feature/${feature.id}/${revision?.version}/request`, {
      method: "POST",
      body: JSON.stringify({
        mergeResultSerialized: JSON.stringify(mergeResult),
        comment,
      }),
    });
    // ...
  }
};
```

**触发条件**：
- 修订状态为 `draft`（不是 `pending-review` 也不是 `approved`）
- 用户拥有 `canManageFeatureDrafts` 权限

### 2.2 后端处理逻辑

**文件**：`packages/back-end/src/api/features/postFeatureRevisionRequestReview.ts:14-79`

核心函数 `requestReview` 执行流程：

1. **权限检查**：检查用户是否有 `canManageFeatureDrafts` 权限（L25-27）
2. **状态校验**：只有 `draft` 状态的修订才能申请审批（L38-42）
3. **状态更新**：调用 `markRevisionAsReviewRequested` 将状态改为 `pending-review`（L44-49）
4. **审计记录**：记录 `feature.revision.requestReview` 审计事件（L60-68）
5. **事件分发**：触发 `revision.reviewRequested` 事件通知（L70-76）

### 2.3 模型层状态变更

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

**文件**：`packages/shared/src/validators/features.ts:267-289`

```typescript
export const revisionStatusSchema = z.enum([
  "draft",           // 草稿
  "published",       // 已发布
  "discarded",       // 已废弃
  "approved",        // 已批准
  "changes-requested", // 需要修改
  "pending-review",  // 待审阅
  "pending-parent",  // 等待父修订（ramp schedule 内部使用）
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

### 3.2 审阅提交流程

**文件**：`packages/back-end/src/api/features/postFeatureRevisionSubmitReview.ts:21-123`

核心函数 `submitRevisionReview` 执行流程：

1. **权限检查**：检查用户是否有 `canReviewFeatureDrafts` 权限（L33-35）
2. **自审批禁止**：创建者不能审批自己的草稿（L50-57）
3. **贡献者自审批禁止**：当 `blockSelfApproval` 配置开启时，贡献者不能审批自己参与的草稿（L61-76）
4. **状态校验**：只有 `pending-review`、`changes-requested`、`approved` 状态可以提交审阅（L78-87）
5. **状态更新**：调用 `submitReviewAndComments` 更新状态（L89-95）
6. **事件分发**：触发对应的审批事件（L112-120）

### 3.3 状态流转核心逻辑

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

### 3.4 编辑时的状态重置

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
  // 状态为 approved 时触发发布
  await apiCall(`/feature/${feature.id}/${revision?.version}/publish`, {
    method: "POST",
    body: JSON.stringify({
      mergeResultSerialized: JSON.stringify(mergeResult),
      comment,
      adminOverride: adminPublish,
      publishExperimentIds: Array.from(selectedExperiments),
    }),
  });
  // ...
}
```

### 4.2 后端发布前置检查

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

### 4.3 审批需求判断逻辑

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

### 4.4 发布核心逻辑

**文件**：`packages/back-end/src/models/FeatureModel.ts:1888-1967`

`publishRevision` 函数执行流程：

1. **状态校验**：只能发布非 `published`/`discarded` 的修订
2. **前置创建**：先创建 ramp schedules（失败则整个发布中止）
3. **应用变更**：调用 `applyRevisionChanges` 更新 feature 主数据
4. **Holdout 副作用**：处理 holdout 关联变更
5. **标记发布**：调用 `markRevisionAsPublished` 将修订状态改为 `published`
6. **清理草稿**：调用 `clearPendingFeatureDraftsForRevision` 清理相关草稿
7. **异常回滚**：如果中间失败，回滚已创建的 ramp schedules
8. **后置处理**：应用 detach actions，清理孤儿 ramp schedules

### 4.5 修订状态标记为已发布

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
  
  // 触发 ramp schedule hook
  await dispatchRevisionPublishedHook(context, revision);
}
```

---

## 五、三者之间的连接方式

### 5.1 核心连接点：revision.status 字段

整个流程通过 `FeatureRevisionModel` 的 `status` 字段串联：

```
draft → [requestReview] → pending-review → [submitReview] → approved → [publish] → published
                                          ↓
                                    changes-requested → [updateRevision] → pending-review
```

### 5.2 API 端点连接

| 动作 | API 路径 | 模型方法 | 状态流转 |
|------|----------|----------|----------|
| 申请审批 | `POST /feature/:id/:version/request-review` | `markRevisionAsReviewRequested` | `draft` → `pending-review` |
| 提交审阅 | `POST /feature/:id/:version/submit-review` | `submitReviewAndComments` | `pending-review` → `approved` / `changes-requested` |
| 发布 | `POST /feature/:id/:version/publish` | `publishRevision` → `markRevisionAsPublished` | `approved` → `published` |

### 5.3 事件分发机制

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

### 5.4 审批要求的动态检查

审批要求不是静态的，而是在发布时**动态计算**的：

```
创建草稿时：checkIfRevisionNeedsReview() → 计算 requiresReview
发布时：再次调用 checkIfRevisionNeedsReview() → 验证当前状态是否满足要求
```

这样设计的好处是：即使审批配置在草稿创建后发生变化，发布时仍能按照最新配置进行检查。

### 5.5 旁路机制（Bypass）

系统提供两种绕过审批的方式（L107-109 in postFeatureRevisionPublish.ts）：

1. **REST API 旁路**：`restApiBypassesReviews` 组织设置开启时，API 调用可以绕过审批
2. **权限旁路**：用户拥有 `canBypassApprovalChecks` 权限时可以绕过（通常是管理员）

---

## 六、关键代码路径汇总

### 6.1 审批申请完整路径
```
RequestReviewModal.tsx:submitButton()
  ↓ API CALL
postFeatureRevisionRequestReview.ts:requestReview()
  ↓
FeatureRevisionModel.ts:markRevisionAsReviewRequested()
  ↓
status: draft → pending-review
```

### 6.2 审阅批准完整路径
```
RequestReviewModal.tsx:submitButton() → renderReviewAndSubmitModal()
  ↓ API CALL
postFeatureRevisionSubmitReview.ts:submitRevisionReview()
  ↓
FeatureRevisionModel.ts:submitReviewAndComments()
  ↓
status: pending-review → approved
```

### 6.3 发布完整路径
```
RequestReviewModal.tsx:submitButton()
  ↓ API CALL
postFeatureRevisionPublish.ts:publishFeatureRevision()
  ├─ checkIfRevisionNeedsReview()  # 检查是否需要审批
  ├─ 验证 status === "approved" 或可以 bypass
  ↓
FeatureModel.ts:publishRevision()
  ├─ applyRevisionChanges()        # 更新 feature 主数据
  └─ markRevisionAsPublished()     # 更新 revision 状态
  ↓
status: approved → published
```
