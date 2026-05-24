# 变更审批工作流状态流转分析

## 一、核心状态定义与流转全景

### 1.1 状态定义

系统支持两套并行的修订（Revision）模型，状态定义基本一致：

| 状态 | 说明 | 对应模型 |
|------|------|----------|
| `draft` | 草稿，正在编辑中 | FeatureRevisionModel / RevisionModel |
| `pending-review` | 待审核，已提交审批请求 | FeatureRevisionModel / RevisionModel |
| `changes-requested` | 需要修改，审核人要求变更 | FeatureRevisionModel / RevisionModel |
| `approved` | 已批准，可以发布 | FeatureRevisionModel / RevisionModel |
| `published` / `merged` | 已发布/已合并，变更已生效 | FeatureRevisionModel 用 `published`，RevisionModel 用 `merged` |
| `discarded` | 已废弃，变更被取消 | 两者通用 |
| `pending-parent` | 等待父修订，用于多步骤发布 | FeatureRevisionModel 特有 |

**状态流转优先级**（决定列表排序，数字越小越靠前）：
```typescript
// packages/front-end/pages/approval-requests.tsx:49-56
const STATUS_PRIORITY: Record<RevisionStatus, number> = {
  "pending-review": 0,     // 最优先，阻塞审核人
  "changes-requested": 1,  // 次优先，请求人需要处理
  "approved": 2,           // 已批准，待发布
  "draft": 3,              // 草稿
  "merged": 4,             // 已发布
  "discarded": 5,          // 已废弃
};
```

### 1.2 状态流转全景图

```
draft ──提交审核──► pending-review ──批准──► approved ──发布──► merged
  │                     │   ▲                 │
  │                     │   │                 │
  │                     ▼   │                 │
  └────编辑──────────── changes-requested    └──绕过审批(bypass)──┐
        │                                                ▲        │
        ▼                                                │        │
  已批准后再编辑 ──resetReviewOnChange=true──► pending-review    │
                                                                 │
  discarded ◄─────废弃───────┴───────────────────────────────────┘
     │
     └──────重新打开──────► draft
```

---

## 二、变更请求的提交（Request Review）

### 2.1 提交流程核心代码

**入口 API**：`POST /features/:id/revisions/:version/request-review`

**处理文件**：
- `packages/back-end/src/api/features/postFeatureRevisionRequestReview.ts:14-79`
- `packages/back-end/src/models/FeatureRevisionModel.ts:1055-1092`

### 2.2 提交条件检查

1. **权限检查** - `canManageFeatureDrafts`（管理草稿权限）
   ```typescript
   // postFeatureRevisionRequestReview.ts:25-27
   if (!req.context.permissions.canManageFeatureDrafts(feature)) {
     req.context.permissions.throwPermissionError();
   }
   ```
   设计意图：贡献者即使没有发布权限，也可以提交审批请求。

2. **状态检查** - 必须是 `draft` 状态
   ```typescript
   // postFeatureRevisionRequestReview.ts:38-42
   if (revision.status !== "draft") {
     throw new BadRequestError(
       `Can only request review on a draft (status is "${revision.status}")`
     );
   }
   ```

### 2.3 提交后的状态变更

```typescript
// FeatureRevisionModel.ts:1055-1092
export async function markRevisionAsReviewRequested(
  context, revision, user, comment?
) {
  await FeatureRevisionModel.updateOne(
    { organization, featureId, version },
    {
      $set: {
        status: "pending-review",    // 核心状态变更
        datePublished: null,        // 清除发布日期
        dateUpdated: new Date(),
        comment: comment,
      },
    }
  );
}
```

### 2.4 通用修订（非 Feature）提交流程

**入口 API**：`POST /revision/:id/submit`

**处理文件**：`packages/back-end/src/routers/revision/revision.controller.ts:384-424`

```typescript
// revision.controller.ts:399-404
if (existingRevision.status !== "draft") {
  return res.status(400).json({
    message: "Only draft revisions can be submitted for review",
  });
}

// 可以由任何有编辑权限的人提交，不局限于作者
if (!getAdapter(existingRevision.target.type).canUpdate(...)) {
  context.permissions.throwPermissionError();
}

await revisionModel.submitForReview(id, userId);
```

---

## 三、审批权限与审核逻辑

### 3.1 审核人权限判定

**谁可以审核？**

| 实体类型 | 审核人条件 | 代码位置 |
|----------|------------|----------|
| Feature | 1. 不是作者<br>2. 有 `canReviewFeatureDrafts` 权限 | `RequestReviewModal.tsx:101-104` |
| SavedGroup | 1. 不是作者<br>2. 有 `canUpdateSavedGroup` 权限 | `helpers.ts:157-159` |
| 通用实体(managedBy=team) | 1. 不是作者<br>2. 是 `ownerTeam` 团队成员 | `helpers.ts:182-188` |
| 通用实体(managedBy=admin) | 1. 不是作者<br>2. 有 `manageOfficialResources` 权限 | `helpers.ts:190-192` |

**前端审核权限判断**：
```typescript
// RequestReviewModal.tsx:101-104
const canReview =
  isPendingReview &&
  createdBy?.id !== user?.id &&
  permissionsUtil.canReviewFeatureDrafts(feature);
```

**后端审核权限判断**（通用修订）：
```typescript
// helpers.ts:127-195
export const canUserReviewEntity = ({
  entityType, revision, entity, userId, teams, userPermissions, canEditEntity
}) => {
  // 已合并/废弃的不能审核
  if (["merged", "discarded"].includes(revision.status)) return false;
  // 不能审核自己的
  if (revision.authorId === userId) return false;
  
  if (entityType === "saved-group") return !!canEditEntity;
  // ... 其他实体类型逻辑
};
```

### 3.2 Feature 审核决策的两层限制机制（Submit Review）

**入口 API**：`POST /features/:id/revisions/:version/submit-review`

**处理文件**：`packages/back-end/src/api/features/postFeatureRevisionSubmitReview.ts:21-123`

**核心限制逻辑**（按顺序检查，先严格后宽松）：

```typescript
// 第一层：创建者（createdBy）限制 — 最严格
// postFeatureRevisionSubmitReview.ts:49-57
if (
  revision.createdBy != null &&
  "id" in revision.createdBy &&
  revision.createdBy.id === req.context.userId &&
  action !== "comment"
) {
  throw new BadRequestError("Cannot submit a review on a draft you created");
}

// 第二层：贡献者（contributors）限制 — 仅阻止 approve
// postFeatureRevisionSubmitReview.ts:59-76
if (action === "approve") {
  const requireReviews = req.context.org.settings?.requireReviews;
  const reviewSetting = Array.isArray(requireReviews)
    ? getReviewSetting(requireReviews, feature)
    : undefined;
  if (reviewSetting?.blockSelfApproval) {
    const isSelfApproval = (revision.contributors ?? []).some(
      (c) => c != null && "id" in c && c.id === req.context.userId,
    );
    if (isSelfApproval) {
      throw new BadRequestError(
        "You cannot approve a draft you contributed to.",
      );
    }
  }
}
```

**两层限制对比表**：

| 限制层级 | 适用对象 | 限制动作 | 允许动作 | 触发条件 |
|---------|----------|----------|----------|----------|
| 第一层 | `createdBy`（创建者） | `approve` + `request-changes` | 仅 `comment` | 无条件，始终生效 |
| 第二层 | `contributors`（贡献者） | 仅 `approve` | `request-changes` + `comment` | `blockSelfApproval=true` |

> **关键设计意图**（代码注释第60行）：
> "request-changes / comment are intentionally allowed."
> 贡献者虽然不能批准自己参与的变更，但可以提出修改要求和评论，这促进了协作而非完全阻塞。

**contributors 字段的维护**：
```typescript
// FeatureRevisionModel.ts:933-937
const contributorUpdate =
  log.user != null ? { $addToSet: { contributors: log.user } } : {};
```
每次调用 `updateRevision` 编辑草稿时，编辑者会被原子地（`$addToSet`）添加到 `contributors` 数组，避免并发编辑导致的数组覆盖问题。

**审核操作允许的状态**：
```typescript
// postFeatureRevisionSubmitReview.ts:78-87
if (
  action !== "comment" &&
  !["pending-review", "changes-requested", "approved"].includes(
    revision.status,
  )
) {
  throw new BadRequestError(...);
}
```
- `comment`：无状态限制，任何状态都可评论
- `approve` / `request-changes`：仅在 `pending-review`、`changes-requested`、`approved` 状态下允许

---

### 3.3 通用修订禁止自审核（Self-Approval Block）

**基础规则**：作者不能批准自己的变更
```typescript
// revision.controller.ts:476-480
if (existingRevision.authorId === userId && decision !== "comment") {
  return res.status(403).json({
    message: "Cannot approve or request changes on your own revision",
  });
}
```

**增强规则**：`blockSelfApproval` 配置（阻止所有贡献者审核）
```typescript
// organization.d.ts:72-73
type RequireReview = {
  blockSelfApproval?: boolean;  // 当 true 时，所有贡献者都不能审核
};

// helpers.ts:85-100
export const isUserBlockedFromApproving = ({
  approvalFlows, entityType, revision, userId
}) => {
  const settings = getApprovalFlowSettings(approvalFlows, entityType);
  if (!settings?.blockSelfApproval) return false;
  const contributors = revision.contributors ?? [revision.authorId];
  return contributors.includes(userId);
};
```

**应用位置**：
```typescript
// revision.controller.ts:487-500
if (decision === "approve" && isUserBlockedFromApproving({...})) {
  return res.status(403).json({
    message: "You contributed to this revision and cannot approve it. " +
             "A separate reviewer is required.",
  });
}
```

### 3.4 审核操作与状态变更

**三种审核决策**（`ReviewDecision`）：

| 决策 | 状态变更 | 代码映射 |
|------|----------|----------|
| `approve` | `pending-review` → `approved` | `RevisionModel.ts:501` |
| `request-changes` | `pending-review` → `changes-requested` | `RevisionModel.ts:503-504` |
| `comment` | 状态不变，仅添加评论 | `RevisionModel.ts:505` |

**审核处理核心逻辑**：
```typescript
// RevisionModel.ts:474-521
async addReview(id, userId, decision, comment) {
  const review = { id, userId, decision, comment, dateCreated: new Date() };
  
  const actionMap = {
    approve: "approved",
    "request-changes": "requested-changes",
    comment: "commented",
  };
  
  const newStatus = decision === "approve" ? "approved"
    : decision === "request-changes" ? "changes-requested"
    : existing.status;
  
  return this.update(existing, {
    reviews: [...existing.reviews, review],
    status: newStatus,
    activityLog: [...existing.activityLog, {
      id, userId, action: actionMap[decision], description: comment, dateCreated
    }],
  });
}
```

### 3.5 审批需求判定（何时需要审批？）

**核心函数**：`checkIfRevisionNeedsReview`

**文件位置**：`packages/shared/src/util/features.ts:1860-1939`

```typescript
export function checkIfRevisionNeedsReview({
  feature, baseRevision, revision, allEnvironments, settings,
  requireApprovalsLicensed = true,
}) {
  // 1. 授权检查：需要 require-approvals 高级功能
  if (!requireApprovalsLicensed) return false;
  
  // 2. 配置检查：支持 boolean 或 数组两种配置格式
  const requireReviews = settings?.requireReviews;
  if (!Array.isArray(requireReviews)) return !!requireReviews;
  
  // 3. 匹配该 feature 的审批配置
  const reviewSetting = getReviewSetting(requireReviews, feature);
  if (!reviewSetting?.requireReviewOn) return false;
  
  // 4. 计算受影响的环境
  const affected = getDraftAffectedEnvironments(revision, baseRevision, allEnvironments);
  
  // 5. 全局变更 vs 环境级变更
  if (affected === "all") {
    // 元数据变更受 featureRequireMetadataReview 控制
    if (!revisionHasMetadataOnlyGlobalChange(revision, baseRevision))
      return true;  // 非元数据全局变更始终需要审批
    return reviewSetting.featureRequireMetadataReview !== false;
  }
  
  // 6. 环境级变更细分
  const envsWithRuleChanges = affected.filter(env => 规则有变化);
  const envKillSwitchChanges = affected.filter(env => 启用状态有变化);
  
  // 规则/值变更始终需要审批（在指定环境范围内）
  if (envsWithRuleChanges.some(env => gatedEnvs.includes(env))) return true;
  
  // kill switch 变更仅当 featureRequireEnvironmentReview=true 时需要审批
  if (envKillSwitchChanges.length > 0 && 
      reviewSetting.featureRequireEnvironmentReview !== false) {
    if (envKillSwitchChanges.some(env => gatedEnvs.includes(env)))
      return true;
  }
  
  return false;
}
```

**审批配置类型**：
```typescript
// organization.d.ts:65-74
type RequireReview = {
  requireReviewOn: boolean;                    // 总开关
  resetReviewOnChange: boolean;               // 变更后重置审批
  environments: string[];                     // 哪些环境需要审批
  projects: string[];                         // 哪些项目需要审批
  featureRequireEnvironmentReview?: boolean;  // 环境启停需要审批
  featureRequireMetadataReview?: boolean;     // 元数据变更需要审批
  blockSelfApproval?: boolean;                // 阻止贡献者自审核
};
```

---

## 四、最终发布（Publish / Merge）

### 4.1 Feature 发布流程

**入口 API**：`POST /features/:id/revisions/:version/publish`

**处理文件**：
- `packages/back-end/src/api/features/postFeatureRevisionPublish.ts:28-179`
- `packages/back-end/src/models/FeatureModel.ts`

### 4.2 发布前置检查

**检查顺序**（逐项严格校验，不满足直接抛出）：

1. **编辑权限** - `canUpdateFeature`
   ```typescript
   // postFeatureRevisionPublish.ts:37-39
   if (!req.context.permissions.canUpdateFeature(feature, {})) {
     req.context.permissions.throwPermissionError();
   }
   ```

2. **状态检查** - 不能是 `published` 或 `discarded`
   ```typescript
   // postFeatureRevisionPublish.ts:50-54
   if (["published", "discarded"].includes(revision.status)) {
     throw new BadRequestError(
       `Cannot publish a revision with status "${revision.status}"`
     );
   }
   ```

3. **合并冲突检查** - `autoMerge`
   ```typescript
   // postFeatureRevisionPublish.ts:67-80
   const mergeResult = autoMerge(live, base, revision, environmentIds, {});
   if (!mergeResult.success) {
     throw new ConflictError(
       "Merge conflicts exist — rebase before publishing",
       mergeResult.conflicts,
     );
   }
   ```

4. **审批需求重新评估**（关键！基于合并后状态）
   ```typescript
   // postFeatureRevisionPublish.ts:96-104
   const requiresReview = checkIfRevisionNeedsReview({
     feature,
     baseRevision: filledLive,        // 注意：用当前线上版本作为基准
     revision: effectiveRevision,     // 注意：用合并后的最终状态
     allEnvironments: environmentIds,
     settings: req.organization.settings,
     requireApprovalsLicensed: req.context.hasPremiumFeature("require-approvals"),
   });
   ```
   > **设计要点**：审批判定基于「合并后的最终状态」与「当前线上状态」的差异，
   > 而不是基于修订创建时的差异。这确保发布时的实际变更符合当前审批策略。

5. **审批状态检查 + 绕过检查**
   ```typescript
   // postFeatureRevisionPublish.ts:107-117
   const canBypass =
     !!req.organization.settings?.restApiBypassesReviews ||
     req.context.permissions.canBypassApprovalChecks(feature);
   
   if (requiresReview && revision.status !== "approved" && !canBypass) {
     throw new BadRequestError(
       `This revision requires approval before publishing (status: "${revision.status}"). ` +
       "Enable 'REST API always bypasses approval requirements' in organization settings, " +
       "or use a role/token that grants bypassApprovalChecks on this project."
     );
   }
   ```

6. **发布环境权限检查** - `canPublishFeature`（按环境精细控制）
   ```typescript
   // postFeatureRevisionPublish.ts:119-128
   const envsToCheck = await getMergeResultPublishEnvs(...);
   if (!req.context.permissions.canPublishFeature(feature, envsToCheck)) {
     req.context.permissions.throwPermissionError();
   }
   ```

### 4.3 发布执行

```typescript
// FeatureModel.ts 中的 publishRevision
const updatedFeature = await publishRevision(
  req.context, feature, revision, mergeResult.result, comment
);
```

**内部状态变更**（`markRevisionAsPublished`）：
```typescript
// FeatureRevisionModel.ts:998-1053
export async function markRevisionAsPublished(
  context, feature, revision, user, comment?
) {
  await FeatureRevisionModel.updateOne(
    { organization, featureId, version },
    {
      $set: {
        status: "published",           // 核心状态变更
        publishedBy: user,             // 记录发布人
        datePublished: new Date(),     // 记录发布时间
        dateUpdated: new Date(),
        comment: revisionComment,
      },
    }
  );
  
  await dispatchRevisionPublishedHook(context, revision);  // 触发后置钩子
}
```

### 4.4 通用修订合并流程

**入口 API**：`POST /revision/:id/merge`

**处理文件**：`packages/back-end/src/routers/revision/revision.controller.ts:885-1009`

**核心步骤**：

1. 权限检查（`canUpdate`）
2. 审批需求检查 + 绕过检查
3. 构建期望状态（`buildMergeDesiredState`）
4. 冲突检查（`checkMergeConflicts`）
5. 变更有效性检查（`hasChanges`）
6. **两步提交**：
   ```typescript
   // revision.controller.ts:987-1007
   // 第一步：更新实体（先落实际变更）
   await adapter.applyChanges(context, entity, desiredState);
   
   // 第二步：标记修订为已合并（后更新状态）
   const mergedRevision = await revisionModel.merge(id, userId, {
     bypass: isBypass,
   });
   ```
   > **设计要点**：两步提交不使用事务。
   > - 如果第一步失败：修订保持原状态，实体不变，可安全重试
   > - 如果第一步成功第二步失败：实体已更新但修订仍为 `approved`，下次发布时 `hasChanges` 检查返回 false 成为空操作，可手动标记合并
   > - 这比先标记合并再更新实体更安全（避免标记了合并但实际变更未生效）

---

## 五、特殊状态流转机制

### 5.1 已批准后变更重置审批

**配置项**：`resetReviewOnChange: boolean`

**触发场景**：修订已 `approved` 后，作者又修改了内容。

**处理逻辑**：
```typescript
// RevisionModel.ts:131-152
private resetApprovalIfNeeded(existing, userId) {
  if (existing.status !== "approved") return {};
  const settings = getApprovalFlowSettings(
    this.context.org.settings?.approvalFlows,
    existing.target.type,
  );
  if (!settings?.resetReviewOnChange) return {};
  
  return {
    status: "pending-review",  // 打回待审核
    resetEntry: {
      id, userId, action: "reopened",
      description: "Approval reset — proposed changes were modified after approval",
      dateCreated: new Date(),
    },
  };
}
```

**调用位置**：
- `updateProposedChanges` - 更新变更内容时
- `rebase` - 变基到最新版本时
- `updateRevision` - Feature 修订更新时

**Feature 修订版本**：
```typescript
// FeatureRevisionModel.ts:905-907
if (resetReview && revision.status === "approved") {
  status = "pending-review";
}
```

### 5.2 变更请求后打回修改

**状态流转**：`pending-review` → `changes-requested` → 编辑 → `pending-review`

```typescript
// FeatureRevisionModel.ts:900-903
// 当有内容变更且状态为 changes-requested 时，自动打回 pending-review
if (revision.status === "changes-requested") {
  status = "pending-review";
}
```

### 5.3 审批绕过（Bypass）

**两种绕过方式**：

| 方式 | 配置 | 影响范围 | 代码位置 |
|------|------|----------|----------|
| REST API 绕过 | `restApiBypassesReviews: true` | 全局，所有 API 调用 | `postFeatureRevisionPublish.ts:108` |
| 权限绕过 | `bypassApprovalChecks` 权限 | 按用户/角色/项目 | `permissionsClass.ts:832-839` |

**权限绕过定义**：
```typescript
// permissionsClass.ts:832-839
public canBypassApprovalChecks(feature) {
  return this.checkProjectFilterPermission(
    { projects: feature.project ? [feature.project] : [] },
    "bypassApprovalChecks",
  );
}
```

**前端 UI 表现**：
```typescript
// RequestReviewModal.ts:83,449-464
const canAdminPublish = permissionsUtil.canBypassApprovalChecks(feature);

// 显示复选框让管理员选择是否绕过
{canAdminPublish && (
  <Checkbox
    label="Bypass approval requirement to publish (optional for Admins only)"
    value={adminPublish}
    setValue={(val) => setAdminPublish(!!val)}
  />
)}
```

---

## 六、关键权限矩阵

### 6.1 Feature 相关权限

| 权限 | 作用 | 检查位置 |
|------|------|----------|
| `manageFeatureDrafts` | 创建/编辑草稿、提交审核请求 | `canManageFeatureDrafts` |
| `canReview` | 审核变更（批准/要求修改） | `canReviewFeatureDrafts` |
| `bypassApprovalChecks` | 绕过审批直接发布 | `canBypassApprovalChecks` |
| `publishFeatures` | 发布到指定环境 | `canPublishFeature(feature, envs)` |
| `manageFeatures` | 编辑 feature 本身 | `canUpdateFeature` |

### 6.2 权限继承关系

```
manageFeatures (manageFeatures)
  ├── 可创建 feature
  ├── 可编辑 feature 元数据
  └── 隐含 manageFeatureDrafts（可编辑草稿）

publishFeatures (环境级)
  └── 可发布到指定环境（需要审批已通过或有 bypass 权限）

canReview (项目级)
  └── 可审核该项目下的变更请求

bypassApprovalChecks (项目级)
  └── 可跳过审批直接发布
```

---

## 七、前端审批交互流程

### 7.1 审批列表页

**文件**：`packages/front-end/pages/approval-requests.tsx`

**三个视图范围**（Scope）：
- `needs-my-review`：我需要审核的（我有审核权 + 我不是作者 + 状态是 pending-review/changes-requested）
- `my-requests`：我发起的请求
- `all`：全部

**默认状态过滤**：`[pending-review, approved, changes-requested]`（不显示 draft 和 已结束状态）

### 7.2 审核弹窗交互流程

**文件**：`packages/front-end/components/Features/RequestReviewModal.tsx`

**按钮文案动态变化**：
```
状态 = draft → "Request Review"
状态 = pending-review + 我可审核 → "Next" → 进入审核表单
状态 = approved → "Publish"
状态 = changes-requested + 我是作者 → "Request Review"（重新提交）
```

**审核表单三个选项**：
```typescript
// RequestReviewModal.tsx:697-724
options = [
  { value: "Comment", label: "Comment", description: "仅提交反馈" },
  { value: "Requested Changes", label: "Request Changes", description: "需要修改后才能发布" },
  { value: "Approved", label: "Approve", description: "批准发布", disabled: isBlockedContributor },
]
```

---

## 八、两套修订系统对比

项目中存在**两套并行**的修订系统，理解它们的区别很重要：

| 维度 | FeatureRevisionModel | RevisionModel（通用） |
|------|---------------------|----------------------|
| 适用对象 | 仅 Feature | SavedGroup 等通用实体 |
| 状态值 | published | merged |
| 变更表示 | 完整快照（rules、defaultValue 等） | JSON Patch (RFC 6902) |
| 配置位置 | `org.settings.requireReviews` | `org.settings.approvalFlows` |
| 审核接口 | `/features/:id/revisions/:version/submit-review` | `/revision/:id/review` |
| 发布接口 | `/features/:id/revisions/:version/publish` | `/revision/:id/merge` |
| 适配器模式 | 无（硬编码逻辑） | 有（EntityRevisionAdapter） |

**注意**：Feature 的审批配置使用 `requireReviews` 数组，SavedGroup 使用 `approvalFlows` 对象。两套配置不共享，需要分别配置。

---

## 九、提交请求、审核决策与发布门禁的关联细节

### 9.1 三个环节的完整链路图

```
┌─────────────────────────────────────────────────────────────────┐
│                    提交请求 (Request Review)                     │
│  API: POST /features/:id/revisions/:version/request-review       │
│  权限: canManageFeatureDrafts                                    │
│  状态: 仅 draft 可提交                                           │
│  结果: draft → pending-review，清除 datePublished                │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                    审核决策 (Submit Review)                      │
│  API: POST /features/:id/revisions/:version/submit-review        │
│  权限: canReviewFeatureDrafts                                    │
│  两层限制（按顺序）:                                             │
│    1. 创建者(createdBy): 禁止 approve + request-changes          │
│       仅允许 comment                                             │
│    2. 贡献者(contributors) + blockSelfApproval=true:             │
│       仅禁止 approve，允许 request-changes + comment             │
│  状态要求:                                                        │
│    comment: 无状态限制                                           │
│    approve/request-changes: pending-review / changes-requested / │
│                             approved                             │
│  结果:                                                           │
│    approve: → approved                                           │
│    request-changes: → changes-requested                          │
│    comment: 状态不变                                             │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                    发布门禁 (Publish)                            │
│  API: POST /features/:id/revisions/:version/publish              │
│  六项检查（按顺序，不满足即终止）:                               │
│    1. 编辑权限: canUpdateFeature                                 │
│    2. 状态检查: 非 published / discarded                         │
│    3. 冲突检查: autoMerge 无冲突                                 │
│    4. 审批需求重估: checkIfRevisionNeedsReview(合并后状态)       │
│       └─ 关键: 基于「合并后最终状态」vs「当前线上状态」          │
│          而非修订创建时的差异                                    │
│    5. 审批状态 + Bypass 检查:                                    │
│       requiresReview && status≠approved && !canBypass → 拒绝     │
│    6. 发布环境权限: canPublishFeature(envsToCheck)               │
│  结果: approved → published                                      │
└─────────────────────────────────────────────────────────────────┘
```

### 9.2 关键关联点详解

#### 关联点 1：状态流转的连续性

| 前序状态 | 操作 | 后续状态 | 允许的下一操作 |
|---------|------|---------|---------------|
| `draft` | request-review | `pending-review` | submit-review (approve/request-changes/comment) |
| `pending-review` | submit-review (approve) | `approved` | publish, submit-review (comment 即可) |
| `pending-review` | submit-review (request-changes) | `changes-requested` | updateRevision (编辑后自动 → pending-review) |
| `changes-requested` | updateRevision (有内容变更) | `pending-review` | submit-review |
| `approved` | updateRevision + resetReviewOnChange=true | `pending-review` | submit-review |

#### 关联点 2：发布时的审批重估 — 最容易忽略的边界

**问题场景**：
- 周一：用户 A 创建修订，修改了生产环境的规则，提交审核
- 周二：审批人 B 批准，状态变为 `approved`
- 周三：管理员修改了组织审批配置，新增了生产环境需要审批
- 周四：用户 A 点击发布

**会发生什么？**
```typescript
// postFeatureRevisionPublish.ts:96-117
const requiresReview = checkIfRevisionNeedsReview({
  feature,
  baseRevision: filledLive,        // 周四的线上状态
  revision: effectiveRevision,     // 周四合并后的最终状态
  // ...
});

if (requiresReview && revision.status !== "approved" && !canBypass) {
  throw new BadRequestError(...);  // 即使周二已批准，周四仍可能被拒绝！
}
```

**关键洞察**：
- `revision.status === "approved"` 只是**必要条件**，不是**充分条件**
- 发布时会用**最新的审批策略**重新评估**最新的变更差异**
- 这意味着：即使已批准，如果审批策略变了，或线上状态变了导致变更范围扩大，发布仍可能被阻止

#### 关联点 3：两层审核限制的协作关系

| 用户角色 | blockSelfApproval | approve | request-changes | comment |
|---------|-------------------|---------|-----------------|---------|
| 非创建者、非贡献者 | false | ✅ | ✅ | ✅ |
| 非创建者、非贡献者 | true | ✅ | ✅ | ✅ |
| 贡献者（非创建者） | false | ✅ | ✅ | ✅ |
| 贡献者（非创建者） | true | ❌ | ✅ | ✅ |
| 创建者（也是贡献者） | false | ❌ | ❌ | ✅ |
| 创建者（也是贡献者） | true | ❌ | ❌ | ✅ |

> **注意**：创建者始终在 `contributors` 数组中（因为创建时的初始编辑会添加自己），所以第二层限制对创建者实际上是冗余的，但第一层限制已经更严格地阻止了所有非评论操作。

#### 关联点 4：Bypass 权限的作用时机

Bypass 权限**仅在发布环节**生效，在审核环节不生效：
- 提交审核（request-review）：不检查 bypass
- 审核决策（submit-review）：不检查 bypass，两层限制始终生效
- 发布（publish）：检查 bypass，可以跳过审批状态要求

这意味着：即使有 bypass 权限，你也不能批准自己的变更，但你可以**跳过审批流程直接发布**。

---

## 十、接口写法易混淆点说明

### 10.1 `/feature`（旧版/错误写法）与 `/features/:id/revisions/:version/*`（正确写法）的区别

在代码审查和文档编写过程中，容易混淆两套不同的 API 接口风格。以下是清晰的对比：

#### 错误/旧版写法：`/feature/:id/:version/*`
- **是否存在**：代码库中**不存在**此路径
- **常见错误点**：
  - 单数 `feature` 而非复数 `features`
  - 缺少 `revisions` 中间路径段
  - 直接在 feature 后拼接 version，而非 `revisions/:version`

#### 正确写法（Feature 审批接口）：`/features/:id/revisions/:version/*`
- **实际路径定义**（`packages/shared/src/validators/feature-revisions.ts`）：
  ```typescript
  // 提交审核请求
  path: "/features/:id/revisions/:version/request-review"
  
  // 提交审核决策
  path: "/features/:id/revisions/:version/submit-review"
  
  // 发布
  path: "/features/:id/revisions/:version/publish"
  ```
- **路径结构解析**：
  - `/features`：复数，表示 Feature 集合
  - `/:id`：Feature 的 ID
  - `/revisions`：表示进入修订子资源
  - `/:version`：修订的版本号（数字，如 1, 2, 3）
  - `/*`：具体操作（`request-review` / `submit-review` / `publish`）

#### 另一套正确写法（通用修订接口）：`/revision/:id/*`
- **实际路径定义**（`packages/back-end/src/routers/revision/revision.controller.ts` 代码注释）：
  ```typescript
  // 提交审核
  // region POST /revision/:id/submit
  
  // 审核决策
  // region POST /revision/:id/review
  
  // 合并/发布
  // region POST /revision/:id/merge
  ```
- **路径结构解析**：
  - `/revision`：**单数**，表示通用修订集合（仅 SavedGroup 等通用实体使用）
  - `/:id`：修订的 ID（字符串 UUID，而非数字版本号）
  - `/*`：具体操作（`submit` / `review` / `merge`）

#### 三套接口路径对比表

| 系统 | 接口前缀 | ID 类型 | 提交审核 | 审核决策 | 发布/合并 |
|------|---------|---------|---------|---------|----------|
| Feature 修订（V1） | `/features/:id/revisions/:version` | 数字 version | `request-review` | `submit-review` | `publish` |
| Feature 修订（V2） | `/v2/features/:id/revisions/:version` | 数字 version | `request-review` | `submit-review` | `publish` |
| 通用修订（SavedGroup） | `/revision/:id` | 字符串 UUID | `submit` | `review` | `merge` |

#### 为什么有两种不同的命名风格？

1. **历史演进原因**：
   - Feature 修订系统先实现，采用了 RESTful 嵌套资源风格：`/features/:id/revisions/:version`
   - 通用修订系统后实现，为了简化路径，采用了根级资源风格：`/revision/:id`

2. **ID 类型差异**：
   - Feature 修订使用**数字版本号**（version），每个 Feature 从 1 开始递增
   - 通用修订使用**字符串 UUID**（id），全局唯一

3. **操作命名差异**：
   | 操作 | Feature 修订 | 通用修订 | 原因 |
   |------|-------------|---------|------|
   | 提交审核 | `request-review` | `submit` | Feature 强调"请求审核"动作，通用修订强调"提交"动作 |
   | 审核决策 | `submit-review` | `review` | Feature 强调"提交审核结果"，通用修订简化为"review" |
   | 发布 | `publish` | `merge` | Feature 是"发布到线上"，通用修订是"合并且应用变更" |

#### 常见错误检查清单

✅ 正确：
- `POST /features/my_feature/revisions/3/request-review`
- `POST /features/my_feature/revisions/3/submit-review`
- `POST /features/my_feature/revisions/3/publish`
- `POST /revision/rev_abc123/submit`
- `POST /revision/rev_abc123/review`
- `POST /revision/rev_abc123/merge`

❌ 错误：
- `POST /feature/my_feature/3/request-review` （单数 feature，缺少 revisions）
- `POST /features/my_feature/3/submit-review` （缺少 revisions 路径段）
- `POST /features/my_feature/revisions/3/review` （应该是 submit-review，不是 review）
- `POST /revisions/rev_abc123/merge` （通用修订是单数 /revision）

---

## 十一、核心代码索引

### 11.1 API 路径定义（Validator）

| API | 文件位置 | 路径定义 |
|-----|----------|---------|
| Feature 提交审核请求 | `packages/shared/src/validators/feature-revisions.ts:272-286` | `path: "/features/:id/revisions/:version/request-review"` |
| Feature 提交审核决策 | `packages/shared/src/validators/feature-revisions.ts:292-306` | `path: "/features/:id/revisions/:version/submit-review"` |
| Feature 发布 | `packages/shared/src/validators/feature-revisions.ts:188-202` | `path: "/features/:id/revisions/:version/publish"` |
| Feature V2 提交审核请求 | `packages/shared/src/validators/feature-revisions-v2.ts:362-373` | `path: "/features/:id/revisions/:version/request-review"` |
| Feature V2 提交审核决策 | `packages/shared/src/validators/feature-revisions-v2.ts:375-386` | `path: "/features/:id/revisions/:version/submit-review"` |
| Feature V2 发布 | `packages/shared/src/validators/feature-revisions-v2.ts:290-301` | `path: "/features/:id/revisions/:version/publish"` |
| 通用修订提交审核 | `packages/back-end/src/routers/revision/revision.controller.ts:369` | `POST /revision/:id/submit` |
| 通用修订审核决策 | `packages/back-end/src/routers/revision/revision.controller.ts:428` | `POST /revision/:id/review` |
| 通用修订合并 | `packages/back-end/src/routers/revision/revision.controller.ts:870` | `POST /revision/:id/merge` |

### 11.2 业务逻辑实现

| 功能 | 文件位置 | 关键函数/类 |
|------|----------|------------|
| Feature 提交审核请求逻辑 | `packages/back-end/src/api/features/postFeatureRevisionRequestReview.ts:14-79` | `requestReview` |
| Feature 提交审核决策逻辑 | `packages/back-end/src/api/features/postFeatureRevisionSubmitReview.ts:21-123` | `submitRevisionReview` |
| Feature 发布逻辑 | `packages/back-end/src/api/features/postFeatureRevisionPublish.ts:28-179` | `publishFeatureRevision` |
| Feature 审核状态变更 | `packages/back-end/src/models/FeatureRevisionModel.ts:1055-1143` | `markRevisionAsReviewRequested`, `submitReviewAndComments` |
| Feature 贡献者追踪 | `packages/back-end/src/models/FeatureRevisionModel.ts:933-937` | `updateRevision` 中的 `$addToSet: { contributors }` |
| 审批判定 | `packages/shared/src/util/features.ts` | `checkIfRevisionNeedsReview` |
| 审核人权限 | `packages/shared/src/revisions/helpers.ts` | `canUserReviewEntity`, `isUserBlockedFromApproving` |
| 通用修订审核操作 | `packages/back-end/src/models/RevisionModel.ts` | `addReview`, `submitForReview` |
| 通用修订控制器 | `packages/back-end/src/routers/revision/revision.controller.ts` | `postReview`, `postMerge`, `postSubmit` |
| 通用修订工具 | `packages/back-end/src/revisions/util.ts` | `buildMergeDesiredState`, `createOrUpdateRevision` |
| 权限定义 | `packages/shared/src/permissions/permissionsClass.ts` | `Permissions` 类 |
| 前端审批列表 | `packages/front-end/pages/approval-requests.tsx` | `ApprovalRequests` 组件 |
| 前端审核弹窗 | `packages/front-end/components/Features/RequestReviewModal.tsx` | `RequestReviewModal` 组件 |
| 类型定义 | `packages/shared/types/organization.d.ts` | `RequireReview`, `ApprovalFlowConfiguration` |
