# 可视化编辑器全链路代码理解

## 一、整体架构概述

GrowthBook 可视化编辑器允许用户在浏览器中直接选择页面元素并绑定到 A/B 实验的变体，无需编写代码。整个系统由以下核心组件构成：

1. **浏览器扩展（GrowthBook DevTools）**：负责在目标页面上注入编辑能力，捕获用户选择的 DOM 元素并生成选择器
2. **前端管理后台**：管理实验、变体和可视化变更集
3. **后端服务**：存储和分发可视化变更集数据
4. **JavaScript SDK**：在运行时拉取变更并应用到页面

---

## 二、核心数据结构

### 2.1 可视化变更集（VisualChangeset）

定义位置：`packages/shared/types/visual-changeset.d.ts`

```typescript
interface VisualChangesetInterface {
  id: string;                    // 变更集ID (vcs_前缀)
  organization: string;          // 组织ID
  urlPatterns: URLPattern[];     // URL匹配规则
  editorUrl: string;             // 编辑器打开的目标URL
  experiment: string;            // 关联的实验ID
  visualChanges: VisualChange[]; // 各变体的可视化变更
}

interface VisualChange {
  id: string;                    // 变更ID (vc_前缀)
  description: string;           // 变更描述
  css: string;                   // 全局CSS样式
  js?: string;                   // 自定义JavaScript
  variation: string;             // 关联的变体ID
  domMutations: DOMMutation[];   // DOM变更列表
}

interface DOMMutation {
  selector: string;              // CSS选择器
  action: "append" | "set" | "remove";  // 操作类型
  attribute: string;             // 目标属性 (html, class, href等)
  value?: string;                // 新值
  parentSelector?: string;       // 父元素选择器(用于拖拽)
  insertBeforeSelector?: string; // 插入位置选择器(用于拖拽)
}
```

### 2.2 SDK 自动实验类型

定义位置：`packages/sdk-js/src/types/growthbook.ts`

```typescript
type AutoExperimentVariation = {
  domMutations?: DOMMutation[];
  css?: string;
  js?: string;
  urlRedirect?: string;
};
```

---

## 三、浏览器扩展取证机制

### 3.1 扩展检测与通信

**关键文件**：`packages/front-end/components/OpenVisualEditorLink.tsx`

扩展检测通过尝试 fetch 扩展内部资源来实现：

```typescript
// Chrome 扩展ID: opemhndcehfgipokneipaafbglcecjia
// Firefox 扩展ID: a69dc869-b91d-4fd3-adb2-71dc23cdc01c

// 检测扩展是否安装
res = await fetch(
  "chrome-extension://opemhndcehfgipokneipaafbglcecjia/js/logo128.png",
  { method: "HEAD" }
);
```

### 3.2 消息传递协议

通过 `window.postMessage` 进行跨页面通信：

```typescript
// 请求打开可视化编辑器
window.postMessage(
  {
    type: "GB_REQUEST_OPEN_VISUAL_EDITOR",
    data: {
      apiHost,      // GrowthBook API 地址
      apiKey,       // visualEditor 角色的 API Key（持久化，非临时）
    },
  },
  window.location.origin
);
```

### 3.3 API Key 获取

**关键文件**：`packages/back-end/src/controllers/experiments.ts:3682-3707`

> **⚠️ 事实修正**：API Key **不是临时的**，而是持久化的用户级 API Key。

```typescript
// GET /visual-editor/key → 调用 findOrCreateVisualEditorToken()
export async function findOrCreateVisualEditorToken(req, res) {
  // 1. 先查找用户是否已有 visualEditor 角色的 API Key
  let visualEditorKey = await context.models.apiKeys.getVisualEditorApiKey(
    req.userId
  );

  // 2. 不存在则创建（持久化存储，无过期时间）
  if (!visualEditorKey) {
    visualEditorKey = await context.models.apiKeys.createUserVisualEditorApiKey({
      userId: req.userId,
      description: `Created automatically for the Visual Editor`,
    });
  }

  res.status(200).json({ key: visualEditorKey.key });
}
```

**Key 特性**（`packages/back-end/src/models/ApiKeyModel.ts:197-213`）：
- `role: "visualEditor"` —— 专属角色，权限受控
- `secret: true` —— 密钥类型
- 每个用户只有一个，复用而非每次创建
- 无过期时间，除非被手动删除

### 3.4 URL 参数传递

打开可视化编辑器时，通过 URL 查询参数传递上下文：

```
?vc-id=vcs_xxx          // 变更集ID
&v-idx=1                // 变体索引
&exp-url=...            // 实验管理页面URL
&ai-enabled=true        // AI功能开关
```

> **证据层级说明**：
> - ✅ **仓内已证据**：扩展检测、postMessage 格式、API Key 获取逻辑、URL 参数
> - ⚠️ **仓外推测**：扩展接收 postMessage 后的存储方式、扩展如何使用 API Key
> - ❓ **未知待验证**：扩展内部状态管理、API Key 在扩展中的有效期

---

## 四、DOM 选择器数据结构与存储

### 4.1 选择器格式约定

**仓内已证据**：数据结构定义（`packages/shared/types/visual-changeset.d.ts`）

```typescript
interface DOMMutation {
  selector: string;              // CSS 选择器字符串
  action: "append" | "set" | "remove";
  attribute: string;
  value?: string;
  parentSelector?: string;       // 仅拖拽时使用
  insertBeforeSelector?: string; // 仅拖拽时使用
}
```

### 4.2 操作类型约定

三种基础 DOM 操作（仓内已证据：类型定义 + 测试用例）：

| 操作 | 含义 | 示例 attribute |
|------|------|---------------|
| `set` | 设置/替换属性值 | `html` (innerHTML), `class`, `href`, `src` |
| `append` | 追加内容 | `html`, `class` |
| `remove` | 移除元素或属性 | `element`, `class`, `attribute` |

### 4.3 拖拽重排支持

**仓内已证据**：`parentSelector` 和 `insertBeforeSelector` 字段存在于类型定义中

```typescript
// 字段存在但无仓内代码示例，格式约定来自类型定义
{
  selector: ".item-3",
  action: "set",
  attribute: "position",    // 约定值
  parentSelector: ".container",
  insertBeforeSelector: ".item-1"
}
```

> **证据层级说明**：
> - ✅ **仓内已证据**：字段存在、类型定义、测试用例中使用简单选择器
> - ⚠️ **仓外推测**：选择器生成算法、唯一性验证、拖拽交互
> - ❓ **未知待验证**：Shadow DOM 支持、iframe 内元素选择

---

## 五、运行时注入与回放机制

### 5.1 核心执行流程

**关键文件**：`packages/sdk-js/src/GrowthBook.ts`

#### 5.1.1 变更应用入口 `_runAutoExperiment` (L676-802)

```typescript
private _runAutoExperiment(experiment: AutoExperiment, forceRerun?: boolean) {
  // 1. 检查是否为手动触发的实验
  // 2. 检查是否被上下文设置禁用
  // 3. 运行实验分配逻辑，确定用户进入哪个变体
  const { result } = runExperiment(experiment, null, this._getEvalContext());
  
  // 4. 如果变体值未变化，跳过重新应用
  if (existing && existing.valueHash === valueHash) return result;
  
  // 5. 撤销现有变更
  if (existing) this._undoActiveAutoExperiment(experiment);
  
  // 6. 应用新变更
  if (result.inExperiment) {
    const changeType = getAutoExperimentChangeType(experiment);
    if (changeType === "visual") {
      const undo = this._applyDOMChanges(result.value);
      this._activeAutoExperiments.set(experiment, { undo, valueHash });
    }
  }
}
```

#### 5.1.2 DOM 变更应用 `_applyDOMChanges` (L1084-1110)

使用 `dom-mutator` 库（v0.6.0）进行声明式 DOM 操作：

```typescript
import mutate, { DeclarativeMutation } from "dom-mutator";

private _applyDOMChanges(changes: AutoExperimentVariation) {
  if (!isBrowser) return;
  const undo: (() => void)[] = [];
  
  // 1. 注入CSS
  if (changes.css) {
    const s = document.createElement("style");
    s.innerHTML = changes.css;
    document.head.appendChild(s);
    undo.push(() => s.remove());
  }
  
  // 2. 注入JavaScript
  if (changes.js) {
    const script = document.createElement("script");
    script.innerHTML = changes.js;
    if (this._options.jsInjectionNonce) {
      script.nonce = this._options.jsInjectionNonce;
    }
    document.head.appendChild(script);
    undo.push(() => script.remove());
  }
  
  // 3. 应用DOM变更
  if (changes.domMutations) {
    changes.domMutations.forEach((mutation) => {
      undo.push(mutate.declarative(mutation as DeclarativeMutation).revert);
    });
  }
  
  // 返回聚合的undo函数
  return () => {
    undo.forEach((fn) => fn());
  };
}
```

### 5.2 dom-mutator 库调用约定

**仓内已证据**：`import mutate from "dom-mutator"` + `mutate.declarative()` 调用

**仓外依赖（已发布开源库）**：`dom-mutator@^0.6.0`（GrowthBook 官方维护，https://github.com/growthbook/dom-mutator）

```typescript
// 来自 dom-mutator 官方文档的 DeclarativeMutation 格式
// 与 SDK 中 DOMMutation 类型完全兼容
type DeclarativeMutation = {
  selector: string;
  action: 'set' | 'append' | 'remove';
  attribute: 'html' | 'class' | 'position' | string;
  value?: string;
  parentSelector?: string;
  insertBeforeSelector?: string;
};
```

**dom-mutator 内部机制（来自官方文档，非本仓代码）**：
- 全局共享一个 `MutationObserver` 监听 `document.body`
- 选择器匹配到元素后，为每个元素建立独立的 `MutationObserver`
- 外部修改（如 React 重渲染）触发后自动重新应用变更
- `revert()` 函数可撤销到最近一次外部设置的值

### 5.3 变更生命周期管理

#### 5.3.1 激活状态追踪

```typescript
private _activeAutoExperiments: Map<
  AutoExperiment,
  { valueHash: string; undo: () => void }
>;
```

- **valueHash**：`JSON.stringify(result.value)`，用于检测变体值是否变化
- **undo**：撤销函数，用于 URL 变化或实验停止时回滚

#### 5.3.2 URL 变化响应

```typescript
public setURL(url: string) {
  this._options.url = url;
  this._updateAllAutoExperiments();
}

private _updateAllAutoExperiments(forceRerun?: boolean) {
  // 1. 停止不再匹配的实验
  this._activeAutoExperiments.forEach((v, k) => {
    if (!keys.has(k)) {
      v.undo();
      this._activeAutoExperiments.delete(k);
    }
  });
  
  // 2. 运行新匹配的实验
  for (const exp of experiments) {
    this._runAutoExperiment(exp, forceRerun);
  }
}
```

### 5.4 防闪烁机制

**关键文件**：`packages/sdk-js/src/auto-wrapper.ts`

```typescript
function setAntiFlicker() {
  // 注入防闪烁CSS
  const styleTag = document.createElement("style");
  styleTag.setAttribute("id", "gb-anti-flicker-style");
  styleTag.innerHTML = ".gb-anti-flicker { opacity: 0 !important; pointer-events: none; }";
  document.head.appendChild(styleTag);
  
  // 给根元素添加class
  document.documentElement.classList.add("gb-anti-flicker");
  
  // 3.5秒超时保护
  antiFlickerTimeout = window.setTimeout(unsetAntiFlicker, timeoutMs);
}

function unsetAntiFlicker() {
  document.documentElement.classList.remove("gb-anti-flicker");
}
```

---

## 六、元素选择到变体绑定的全链路

### 6.1 完整流程图（标注证据层级）

```
用户在GrowthBook后台创建实验
        ↓  ✅ 仓内已证据
1. 创建 VisualChangeset（前端主用路径）
   POST /experiments/:id/visual-changeset（单数，JWT认证）
   ├─ 路径证据: packages/back-end/src/app.ts:756-758
   ├─ 调用证据: packages/front-end/components/Experiment/VisualChangesetModal.tsx:80
   ├─ 控制器: experimentsController.postVisualChangeset
   ├─ id: vcs_xxx
   ├─ experiment: exp_xxx
   ├─ editorUrl: https://target.com/page
   └─ urlPatterns: [{type: "simple", pattern: "/page"}]

   （API 兼容路径：POST /api/v1/experiments/:id/visual-changesets，复数，API Key认证）
        ↓  ✅ 仓内已证据
2. 用户点击 "Open Visual Editor"
   ├─ 检测浏览器扩展是否安装 (fetch 扩展资源)
   ├─ 获取API密钥: GET /visual-editor/key (持久化，非临时)
   └─ 通过 postMessage 向扩展传递 apiHost + apiKey
        ↓  ✅ 仓内已证据
3. 跳转到目标页面（带vc-id参数）
   https://target.com/page?vc-id=vcs_xxx&v-idx=1
        ↓  ❓ 未知待验证（扩展内部逻辑）
4. 浏览器扩展检测到 vc-id 参数
   ├─ [推测] 注入编辑器UI覆盖层到页面
   ├─ [推测] 从GrowthBook API拉取变更集数据
   └─ [推测] 初始化选择模式
        ↓  ❓ 未知待验证（扩展内部逻辑）
5. 用户选择页面元素（选择模式）
   ├─ [推测] 扩展监听鼠标hover和click事件
   ├─ [推测] 高亮目标元素
   ├─ [推测] 捕获点击的 DOM 元素
   └─ [推测] 生成 CSS 选择器
        ↓  ❓ 未知待验证（扩展内部逻辑）
6. 用户编辑变体内容
   ├─ 修改 InnerHTML / 属性 / CSS类
   ├─ 或拖拽元素调整位置
   ├─ [推测] 实时预览变更效果
   └─ 生成 DOMMutation 对象
        ↓  ✅ 仓内已证据
7. 保存变更到后端（路径随调用方不同）
   前端 UI 调用：PUT /visual-changesets/:id（JWT认证）
   浏览器扩展调用：PUT /api/v1/visual-changesets/:id（API Key认证）
   {
     visualChanges: [{
       id: vc_xxx,
       variation: var_xxx,
       css: "...",
       js: "...",
       domMutations: [...]
     }]
   }
        ↓  ✅ 仓内已证据
8. 后端更新并触发SDK缓存刷新
   ├─ 更新 VisualChangesetModel
   ├─ 标记 experiment.hasVisualChangesets = true
   └─ queueSDKPayloadRefresh() → CDN更新
        ↓  ✅ 仓内已证据
9. 生产环境SDK拉取实验配置
   GET /api/features/:clientKey
   返回 payload.experiments 数组中包含 visual 类型实验
        ↓  ✅ 仓内已证据
10. SDK运行时应用变更
    ├─ 检查URL匹配: isURLTargeted(url, urlPatterns)
    ├─ 实验分配: runExperiment() → 确定变体
    └─ 应用变更: _applyDOMChanges() → 注入CSS/JS + 调用dom-mutator
```

### 6.2 关键环节详解

#### 6.2.1 实验与变体绑定

**关键文件**：`packages/back-end/src/models/VisualChangesetModel.ts`

创建变更集时，自动为每个变体生成空的 VisualChange：

```typescript
export const genNewVisualChange = (variation: Variation): VisualChange => ({
  id: uniqid("vc_"),
  variation: variation.id,  // 绑定到变体ID
  description: "",
  css: "",
  domMutations: [],
});

// 创建变更集时自动同步变体
export const createVisualChangeset = async ({ experiment, ... }) => {
  const visualChangeset = await VisualChangesetModel.create({
    // ...
    visualChanges: getLatestPhaseVariations(experiment).map(genNewVisualChange),
  });
  
  // 标记实验有可视化变更
  await updateExperiment({
    changes: { hasVisualChangesets: true },
  });
};
```

#### 6.2.2 API 路径边界：两套接口并存

代码库中存在**两套独立的路由系统**，分别用于不同的调用方：

| 路由系统 | 认证方式 | 路径前缀 | 前端实际调用 | 外部 API 调用 |
|----------|----------|----------|-------------|--------------|
| **旧路由（主用）** | JWT（会话Cookie） | 无前缀 | ✅ 是 | ❌ 否 |
| **新路由（兼容）** | API Key | `/api/v1/` | ❌ 否 | ✅ 是 |

##### 表 1：前端主用路径（JWT 认证，app.ts 直接挂载）

**关键证据**：前端 `apiCall` 调用路径无 `/api/v1` 前缀（`auth.tsx:342`）→ `fetch(getApiHost() + url)`

| 操作 | 方法 | 实际路径（单数/复数混用） | 调用方 | 控制器入口 | 证据位置 |
|------|------|--------------------------|--------|------------|----------|
| 创建变更集 | POST | `/experiments/:id/visual-changeset`（**单数**） | 前端 UI | `experimentsController.postVisualChangeset` | `app.ts:756-758`, `VisualChangesetModal.tsx:80` |
| 更新变更集 | PUT | `/visual-changesets/:id`（复数） | 前端 UI / 扩展 | `experimentsController.putVisualChangeset` | `app.ts:760`, `VisualChangesetModal.tsx:89` |
| 删除变更集 | DELETE | `/visual-changesets/:id`（复数） | 前端 UI | `experimentsController.deleteVisualChangeset` | `app.ts:761-764`, `VisualChangesetTable.tsx:152` |
| 获取编辑器Key | GET | `/visual-editor/key` | 前端 UI | `experimentsController.findOrCreateVisualEditorToken` | `app.ts:772-776`, `OpenVisualEditorLink.tsx` |

> **⚠️ 关键发现**：创建接口使用**单数** `/visual-changeset`，更新/删除使用**复数** `/visual-changesets`，存在单复数不一致。

##### 表 2：API 兼容路径（API Key 认证，/api/v1 前缀）

**关键证据**：挂载在 `apiRouter` 下（`app.ts:369-376`）→ 路径自动带 `/api/v1/` 前缀

| 操作 | 方法 | 规范路径（全复数） | 调用方 | Handler 入口 | 证据位置 |
|------|------|-------------------|--------|-------------|----------|
| 列出变更集 | GET | `/api/v1/experiments/:id/visual-changesets` | 外部 API | `listVisualChangesets` | `visual-changesets.ts:119` |
| 创建变更集 | POST | `/api/v1/experiments/:id/visual-changesets`（**复数**） | 外部 API | `postVisualChangesets` | `visual-changesets.ts:136` |
| 获取变更集 | GET | `/api/v1/visual-changesets/:id` | 外部 API / 扩展 | `getVisualChangeset` | `visual-changesets.ts:168` |
| 更新变更集 | PUT | `/api/v1/visual-changesets/:id` | 外部 API / 扩展 | `putVisualChangeset` | `visual-changesets.ts:238` |
| 添加单条变更 | POST | `/api/v1/visual-changesets/:id/visual-change` | 外部 API | `postVisualChange` | `visual-changesets.ts:285` |
| 更新单条变更 | PUT | `/api/v1/visual-changesets/:id/visual-change/:visualChangeId` | 外部 API | `putVisualChange` | `visual-changesets.ts:313` |

##### 表 3：控制器与 Handler 对应关系

| 功能 | 旧控制器（app.ts 路由） | 新 Handler（apiRouter 路由） | 共用底层模型 |
|------|------------------------|-----------------------------|-------------|
| 创建 | `experimentsController.postVisualChangeset` | `postVisualChangesets` | `createVisualChangeset()` |
| 更新 | `experimentsController.putVisualChangeset` | `putVisualChangeset` | `updateVisualChangeset()` |
| 删除 | `experimentsController.deleteVisualChangeset` | 无（旧路由独占） | `deleteVisualChangesetById()` |
| 获取单个 | 无（新路由独占） | `getVisualChangeset` | `findVisualChangesetById()` |
| 列表 | 无（新路由独占） | `listVisualChangesets` | `findVisualChangesetsByExperiment()` |

> **⚠️ 事实修正**：
> 1. 前端实际调用的创建路径是 **单数** `/experiments/:id/visual-changeset`，不是复数 `/visual-changesets`
> 2. 存在两套接口：旧路由（JWT，前端用）和新路由（API Key，外部用）
> 3. 单复数不一致是历史遗留问题：创建是单数，更新/删除是复数

#### 6.2.3 URL 匹配逻辑

**关键文件**：`packages/sdk-js/src/util.ts` (L86-182)

```typescript
export function isURLTargeted(url: string, targets: UrlTarget[]) {
  let hasIncludeRules = false;
  let isIncluded = false;

  for (const target of targets) {
    const match = _evalURLTarget(url, target.type, target.pattern);
    
    // 排除规则优先：只要匹配一个排除规则，立即返回false
    if (target.include === false) {
      if (match) return false;
    } else {
      // 包含规则：只要有一个匹配就通过
      hasIncludeRules = true;
      if (match) isIncluded = true;
    }
  }
  
  // 没有包含规则时默认通过（只有排除规则的情况）
  return isIncluded || !hasIncludeRules;
}
```

**Simple 模式匹配算法** (`_evalSimpleUrlTarget`)：
1. 将 `*` 通配符替换为临时占位符 `_____`
2. 转义其他正则特殊字符
3. 将 `_____` 换回 `.*` 作为正则
4. 分别比较 host、pathname、hash、searchParams

#### 6.2.4 SDK 有效负载分发

**关键文件**：`packages/back-end/src/models/VisualChangesetModel.ts` (L415-473)

变更集创建/更新/删除时，自动触发 SDK 有效负载刷新：

```typescript
const onVisualChangesetUpdate = async ({ newVisualChangeset, ... }) => {
  // 检查是否有实际变更
  if (!visualChangesetsHaveChanges({ oldVisualChangeset, newVisualChangeset }))
    return;

  // 获取需要刷新的SDK连接
  const payloadKeys = getPayloadKeys(context, experiment);
  
  // 队列刷新（异步）
  queueSDKPayloadRefresh({
    context,
    payloadKeys,
    auditContext: {
      event: "updated",
      model: "visualchangeset",
      id: newVisualChangeset.id,
    },
  });
};
```

---

## 七、安全与权限控制

### 7.1 能力开关

**关键文件**：`packages/sdk-js/src/GrowthBook.ts` (L1006-1055)

```typescript
private _isAutoExperimentBlockedByContext(experiment: AutoExperiment): boolean {
  const changeType = getAutoExperimentChangeType(experiment);
  
  if (changeType === "visual") {
    // 全局禁用可视化实验
    if (this._options.disableVisualExperiments) return true;
    
    // 禁用JS注入时，如果实验包含JS则阻止
    if (this._options.disableJsInjection) {
      if (experiment.variations.some((v) => v.js)) return true;
    }
  }
  
  // 按changeId屏蔽特定变更
  if (experiment.changeId && 
      (this._options.blockedChangeIds || []).includes(experiment.changeId)) {
    return true;
  }
  
  return false;
}
```

### 7.2 CSP 支持

支持 script nonce 规避 CSP 限制：

```typescript
// 初始化时传入 nonce
const gb = new GrowthBook({
  jsInjectionNonce: "random-nonce-value",
});

// 注入脚本时使用
const script = document.createElement("script");
script.nonce = this._options.jsInjectionNonce;
script.innerHTML = changes.js;
```

### 7.3 自定义 DOM 变更回调

允许应用层接管 DOM 变更应用：

```typescript
const gb = new GrowthBook({
  applyDomChangesCallback: (changes) => {
    // 自定义应用逻辑（如React/Vue组件内更新）
    console.log("Applying changes:", changes);
    
    // 返回undo函数
    return () => { /* 自定义撤销逻辑 */ };
  },
});
```

---

## 八、关键技术点总结

| 技术点 | 实现方式 | 关键文件 | 证据层级 |
|--------|----------|----------|----------|
| 扩展检测 | `fetch` 扩展内部资源 | `OpenVisualEditorLink.tsx` | ✅ 仓内已证据 |
| API Key 获取 | `findOrCreateVisualEditorToken`（持久化） | `experiments.ts:3682-3707` | ✅ 仓内已证据 |
| 消息通信 | `window.postMessage` | `OpenVisualEditorLink.tsx` | ✅ 仓内已证据 |
| 数据结构 | `DOMMutation`、`VisualChange` 类型 | `visual-changeset.d.ts` | ✅ 仓内已证据 |
| DOM 变更回放 | `dom-mutator` 声明式变更 | `GrowthBook.ts:1084-1110` | ✅ 仓内已证据 |
| dom-mutator 内部机制 | MutationObserver 监听与重应用 | dom-mutator 官方文档 | ⚠️ 仓外依赖 |
| 变更撤销 | 每个操作生成 undo 函数，聚合执行 | `GrowthBook.ts:1086-1109` | ✅ 仓内已证据 |
| URL 匹配 | Simple + Regex 双模式 | `util.ts:86-182` | ✅ 仓内已证据 |
| 防闪烁 | 先隐藏页面，变更应用后显示 | `auto-wrapper.ts:60-97` | ✅ 仓内已证据 |
| 变体绑定 | `VisualChange.variation` 关联 | `VisualChangesetModel.ts:252-258` | ✅ 仓内已证据 |
| SDK 分发 | 变更集更新触发 payload 刷新 | `VisualChangesetModel.ts:439-473` | ✅ 仓内已证据 |
| 旧路由（前端用） | JWT 认证，单复数混用 | `app.ts:756-764` | ✅ 仓内已证据 |
| 新路由（API用） | API Key 认证，全复数，`/api/v1/` 前缀 | `api.router.ts:36`, `visual-changesets.router.ts` | ✅ 仓内已证据 |
| DOM 选择器生成 | 从元素向上遍历 DOM 树 | 扩展代码（外部） | ❓ 未知待验证 |
| 元素高亮交互 | hover 时高亮样式 | 扩展代码（外部） | ❓ 未知待验证 |
| 编辑器 UI 覆盖层 | iframe 或 fixed 侧边栏 | 扩展代码（外部） | ❓ 未知待验证 |

---

## 九、相关测试验证

**关键文件**：`packages/sdk-js/test/visual-changes.test.ts`

测试覆盖的场景包括：

1. **基本变更应用**：CSS 注入、DOM 变更、JS 注入
2. **URL 切换**：URL 变化时自动撤销/重新应用
3. **手动触发实验**：`triggerExperiment()` 行为
4. **多页面实验**：不同 URL 匹配不同变更集
5. **实验定义更新**：运行中修改实验定义的响应
6. **阻止机制**：`blockedChangeIds`、`disableVisualExperiments`、`disableJsInjection`
7. **自定义回调**：`applyDomChangesCallback` 接管逻辑

---

## 十、证据层级三重边界：仓内已证据 / 仓外推测 / 未知待验证

### 10.1 ✅ 仓内已证据（有完整代码实现）

以下功能在代码库中有完整的源代码证据，可直接阅读、调试、修改：

| 模块 | 关键文件 | 功能描述 |
|------|----------|----------|
| 扩展检测 | `packages/front-end/components/OpenVisualEditorLink.tsx` | 通过 fetch 扩展内部资源探测扩展是否安装 |
| 消息通信 | 同上 | 通过 `window.postMessage` 发送 `GB_REQUEST_OPEN_VISUAL_EDITOR` 消息 |
| API Key 获取 | `packages/back-end/src/controllers/experiments.ts:3682-3707` | `findOrCreateVisualEditorToken` 获取持久化的 `visualEditor` 角色 Key |
| API Key 模型 | `packages/back-end/src/models/ApiKeyModel.ts:197-261` | Key 的创建、查找、角色定义 |
| 数据结构 | `packages/shared/types/visual-changeset.d.ts` | `VisualChangesetInterface`、`VisualChange`、`DOMMutation` 类型定义 |
| 验证器 | `packages/shared/src/validators/visual-changesets.ts` | 所有 API 的请求/响应 schema 验证 |
| 后端模型 | `packages/back-end/src/models/VisualChangesetModel.ts` | CRUD、变体绑定、SDK 负载刷新 |
| 后端路由 | `packages/back-end/src/api/visual-changesets/*.ts` | REST API 接口定义 + handler 实现 |
| 变更回放 | `packages/sdk-js/src/GrowthBook.ts:1084-1110` | `_applyDOMChanges` 注入 CSS/JS 并调用 `dom-mutator` |
| 生命周期管理 | `packages/sdk-js/src/GrowthBook.ts:676-802` | `_runAutoExperiment`、`_updateAllAutoExperiments` |
| URL 匹配 | `packages/sdk-js/src/util.ts:86-182` | `isURLTargeted`、`_evalSimpleUrlTarget` |
| 防闪烁 | `packages/sdk-js/src/auto-wrapper.ts:60-97` | `setAntiFlicker` / `unsetAntiFlicker` |
| 安全控制 | `packages/sdk-js/src/GrowthBook.ts:1006-1055` | `_isAutoExperimentBlockedByContext` 各种开关 |
| 测试用例 | `packages/sdk-js/test/visual-changes.test.ts` | 11 个完整的端到端测试场景 |
| 环境检测 | `packages/sdk-js/src/GrowthBook.ts:61` | `isBrowser = typeof window !== "undefined"` |
| 旧路由系统 | `packages/back-end/src/app.ts:756-764` | JWT 认证，单复数混用路径（前端用） |
| 新路由系统 | `packages/back-end/src/api/api.router.ts:36` | API Key 认证，`/api/v1/` 前缀（外部用） |
| 前端调用 | `packages/front-end/services/auth.tsx:342` | `apiCall` 调用无 `/api/v1` 前缀 |
| 前端路由调用 | `packages/front-end/components/Experiment/VisualChangesetModal.tsx:80` | 调用单数 `/visual-changeset` 创建路径 |

### 10.2 ⚠️ 仓外推测（基于接口约定反推）

以下功能不在此代码库中，但可通过公开的接口约定、类型定义、第三方库文档进行合理推测：

#### 10.2.1 dom-mutator 库行为（公开开源库）

来源：https://github.com/growthbook/dom-mutator

```
推测1: 选择器匹配不到元素时，dom-mutator 不会抛异常
  依据: 官方文档说明 "works even if the selector doesn't exist yet"
  行为: 用全局 MutationObserver 持续监听，元素出现后自动应用
  超时: 无内置超时，一直等待到 revert() 被调用

推测2: 选择器匹配多个元素时，变更会应用到所有匹配元素
  依据: 官方文档示例未限制单个元素，querySelectorAll 语义
  验证: 可通过测试用例手动验证

推测3: 外部修改（React/Vue 重渲染）会触发自动重应用
  依据: 官方文档说明 "re-applies if there's an external change"
  机制: 每个匹配元素有独立的 MutationObserver 监听目标属性
```

#### 10.2.2 浏览器扩展行为（基于类型定义反推）

```
推测1: 选择器生成优先级（基于 DOM 选择器最佳实践）
  id → data-* 属性 → class 组合 → 标签名层级 → nth-child
  依据: 类型定义要求 selector 是标准 CSS 选择器
  注意: 这是合理推测，不是确定性结论

推测2: 拖拽重排使用 insertBefore API
  依据: 类型定义中有 parentSelector 和 insertBeforeSelector 字段
  对应 DOM API: parent.insertBefore(element, referenceSibling)

推测3: attribute 字段约定
  "html" → innerHTML
  "class" → classList 操作
  "position" → 元素位置移动
  其他字符串 → setAttribute / getAttribute
  依据: dom-mutator 官方文档中的 declarative 格式说明
```

### 10.3 ❓ 未知待验证（完全无代码证据）

以下功能完全在浏览器扩展中实现，此代码库中没有任何证据，需要实际调试扩展或查看扩展源码才能确认：

| 功能 | 接口约定 | 未知点 |
|------|----------|--------|
| **DOM 选择器生成算法** | 输出标准 CSS 选择器字符串 | 具体遍历策略、优先级权重、冲突解决机制 |
| **选择器唯一性验证** | 无仓内证据 | 是否验证 `querySelectorAll(selector).length === 1` |
| **元素高亮交互** | 无仓内证据 | 高亮样式（颜色、边框）、实现方式（outline / box-shadow） |
| **编辑器 UI 覆盖层** | 通过 `?vc-id=` URL 参数触发 | UI 框架、样式类名、iframe 还是直接注入 DOM |
| **拖拽重排交互** | `parentSelector` + `insertBeforeSelector` 格式 | 使用 HTML5 Drag & Drop 还是自定义鼠标事件 |
| **实时预览** | 无仓内证据 | 是否即时调用 dom-mutator，还是有防抖节流 |
| **元素属性拾取** | `attribute` 字段约定 | 是否过滤掉 `on*` 事件属性，如何处理 `style` 属性 |
| **选择器回退策略** | 无仓内证据 | 首选选择器失效时是否生成备选选择器 |
| **Shadow DOM 支持** | 无仓内证据 | 是否能穿透 Shadow DOM 选择元素 |
| **iframe 支持** | 无仓内证据 | 是否能选择跨域 iframe 内的元素 |

> **核心边界说明**：代码库与扩展的交互边界在 `DOMMutation` 数据结构。扩展负责生成符合该结构的数据，代码库负责存储、分发和回放。**扩展内部的选择器生成算法不影响 SDK 运行**，只要输出的选择器能被 `document.querySelector` 正确解析即可。

---

## 十一、DOM 选择器约束条件（基于仓内证据反推）

### 11.1 选择器必须满足的约束

从 `dom-mutator` 接口规范和测试用例反推，选择器必须满足：

```
约束1: 有效的 CSS 选择器语法
  ↓ 必须能通过 document.querySelector(selector) 解析
  ↓ 不能包含伪元素 ::before/::after（不可变）
  ↓ 不能包含伪类 :hover/:active（运行时不生效）
  依据: dom-mutator 官方文档 + 浏览器标准 API

约束2: 可被 MutationObserver 持续监听
  ↓ 选择器匹配的元素在 DOM 树中必须是可观察的
  ↓ Shadow DOM 支持情况未知（待验证）
  依据: dom-mutator 内部使用 MutationObserver

约束3: 唯一性（推荐但不强制）
  ↓ 建议 document.querySelectorAll(selector).length === 1
  ↓ 若不唯一，dom-mutator 会应用到所有匹配元素
  依据: querySelectorAll 标准语义
```

### 11.2 attribute 字段值约定

**仓内已证据**：测试用例 + dom-mutator 文档

| attribute 值 | 对应 DOM API | 支持的 action | 示例 |
|--------------|--------------|---------------|------|
| `html` | `element.innerHTML` | `set`, `append`, `remove` | `<h1> → 替换为新标题` |
| `class` | `element.classList` | `set`, `append`, `remove` | 添加/移除 `.active` 类 |
| `position` | `parent.insertBefore()` | `set` | 移动元素到新位置 |
| 其他字符串 | `element.setAttribute()` | `set`, `append`, `remove` | `href`, `src`, `title`, `data-*` |

---

## 十二、排障边界清单（可直接用于问题定位）

### 12.1 仓内可排障边界（有调试手段）

| 问题场景 | 排查方式 | 代码位置 / 调试手段 |
|----------|----------|---------------------|
| **SDK 初始化失败** | 查看控制台错误 | `GrowthBook.loaded` 状态 |
| **用户未分配到变体** | 打印分配结果 | `gb.getExperimentValue(experimentKey, defaultValue)` |
| **URL 不匹配** | 调用匹配函数验证 | `import { isURLTargeted } from '@growthbook/growthbook'` |
| **变更未应用** | 查看激活实验 Map | `console.log(gb._activeAutoExperiments)` |
| **可视化实验被阻止** | 检查开关配置 | `disableVisualExperiments`, `disableJsInjection`, `blockedChangeIds` |
| **CSP 阻止 JS 注入** | 查看控制台 CSP 错误 | 设置 `jsInjectionNonce` |
| **SSR 环境不生效** | 检查环境检测 | `!isBrowser` 时直接 return |
| **变更撤销异常** | 查看 undo 函数调用 | URL 切换 / destroy() 时自动调用 |
| **选择器匹配不到** | 控制台手动验证 | `document.querySelector(selector) !== null` |
| **选择器匹配多个** | 控制台手动验证 | `document.querySelectorAll(selector).length` |
| **自定义回调接管** | 打印变更内容 | `applyDomChangesCallback` 中添加日志 |

### 12.2 仓内不可排障边界（需调试浏览器扩展）

| 问题场景 | 排查方式 | 边界说明 |
|----------|----------|----------|
| **扩展未注入编辑器** | 检查扩展是否启用、URL 参数是否正确 | 扩展内部逻辑，仓内无代码 |
| **元素无法选中** | 检查是否在 Shadow DOM / iframe 内 | 扩展元素拾取逻辑 |
| **生成的选择器不稳定** | 检查页面结构是否频繁变化 | 扩展选择器生成算法 |
| **高亮样式异常** | 检查扩展是否注入了样式表 | 扩展 UI 实现 |
| **拖拽无反应** | 检查元素是否可拖拽（CSS user-select） | 扩展拖拽交互逻辑 |
| **实时预览不生效** | 检查网络请求（扩展 → GrowthBook API） | 扩展数据同步逻辑 |
| **保存失败** | 检查扩展控制台网络请求 | 扩展 API 调用逻辑 |

### 12.3 dom-mutator 可观测行为（来自官方文档）

| 行为 | 说明 | 验证方式 |
|------|------|----------|
| **静默等待元素出现** | 选择器无匹配时不报错，持续监听 | 动态渲染的元素出现后自动应用变更 |
| **自动重应用** | 外部修改后自动重新应用 | 修改 element.innerHTML 后观察是否恢复 |
| **revert 可撤销** | 调用 revert 后恢复到外部值 | 调用 mutation.revert() 观察 |
| **全局观察者可暂停** | 批量 DOM 操作前可暂停 | `import { disconnectGlobalObserver } from 'dom-mutator'` |

### 12.4 典型排查路径（标注可排障边界）

```
问题: 页面元素没有变更
  ↓
步骤1: 确认 SDK 初始化正常 → ✅ 仓内可排障
  检查 growthbook.loaded / 控制台错误
  ↓ 否
修正 SDK 配置（clientKey, apiHost）
  ↓ 是
步骤2: 确认用户被分配到实验变体 → ✅ 仓内可排障
  console.log(gb.getExperimentValue('exp-key', 'default'))
  ↓ 否
检查 targeting 条件 / hashAttribute / 流量分配
  ↓ 是
步骤3: 确认 URL 匹配 → ✅ 仓内可排障
  调用 isURLTargeted(window.location.href, experiment.urlPatterns)
  ↓ 否
修正 urlPatterns 规则
  ↓ 是
步骤4: 确认选择器有效 → ✅ 仓内可排障
  document.querySelector(selector) !== null
  ↓ 否
  ├─ 元素未渲染 → 等待 SPA 路由加载完成
  ├─ 选择器过期 → ❗ 重新在编辑器中选择（需扩展）
  └─ Shadow DOM / iframe → ❗ 需确认扩展支持
  ↓ 是
步骤5: 检查阻止开关 → ✅ 仓内可排障
  disableVisualExperiments / disableJsInjection / blockedChangeIds
  ↓ 否
步骤6: 检查 CSP / 控制台错误 → ✅ 仓内可排障
  ↓ 否
步骤7: 检查 dom-mutator 行为 → ⚠️ 仓外依赖，可通过 API 验证
  手动调用 mutate.declarative() 观察是否生效
  ↓ 否
可能是 dom-mutator 与页面冲突，需简化变更
  ↓ 是
步骤8: 检查扩展是否正确保存变更 → ❓ 未知待验证，需调试扩展
  检查 Network 面板中 PUT /visual-changesets/:id 请求
  ↓
最终定位根因
```

### 12.5 仓内可用调试工具链

```javascript
// 1. 查看当前激活的实验
console.log(gb._activeAutoExperiments);  // Map<experiment, {undo, valueHash}>

// 2. 自定义回调接管变更，打印调试信息
const gb = new GrowthBook({
  applyDomChangesCallback: (changes) => {
    console.log('[GB Debug] Applying changes:', changes);
    // 逐个验证选择器
    changes.domMutations?.forEach(m => {
      const el = document.querySelector(m.selector);
      if (!el) console.warn('[GB Debug] Selector not found:', m.selector);
      else console.log('[GB Debug] Found element for', m.selector, el);
    });
    // 返回默认实现（注意：此写法会覆盖默认行为）
    // 如需保留默认行为，需手动调用 _applyDOMChanges 逻辑
  }
});

// 3. 手动验证 URL 匹配
import { isURLTargeted } from '@growthbook/growthbook';
console.log('URL match:', isURLTargeted(window.location.href, [
  { type: 'simple', pattern: 'example.com/page', include: true }
]));

// 4. 暂停/恢复 dom-mutator 的全局观察（需单独导入 dom-mutator）
// import { disconnectGlobalObserver, connectGlobalObserver } from 'dom-mutator';
// 批量 DOM 操作前暂停，避免频繁重应用
// disconnectGlobalObserver();
// ... 操作 DOM ...
// connectGlobalObserver();

// 5. 检查运行环境
console.log('Is browser:', typeof window !== 'undefined');
console.log('Current URL:', gb.getURL());
```

---

## 已修正的事实错误汇总

| 错误表述 | 修正后内容 | 证据位置 |
|----------|------------|----------|
| "获取临时API密钥" | "获取持久化 API Key（visualEditor 角色，每个用户一个，无过期时间）" | `experiments.ts:3690-3702`, `ApiKeyModel.ts:197-261` |
| "创建接口路径 `/experiments/:id/visual-changesets`" | 前端主用路径是**单数** `/experiments/:id/visual-changeset`（JWT认证），复数 `/api/v1/experiments/:id/visual-changesets` 是 API 兼容路径（API Key认证） | `app.ts:756-758`, `VisualChangesetModal.tsx:80`, `visual-changesets.ts:136` |
| "API 路径描述不完整" | 存在**两套独立路由系统**：旧路由（JWT，无前缀，前端用）和新路由（API Key，`/api/v1/` 前缀，外部用） | `app.ts:756-764`, `app.ts:369-376` |
| "单复数一致性" | 创建接口是单数 `/visual-changeset`，更新/删除接口是复数 `/visual-changesets`，为历史遗留不一致 | `app.ts:756-764` |

---

## 新增：路由边界排障清单

| 问题场景 | 排查要点 | 验证命令/方式 |
|----------|----------|--------------|
| **前端创建变更集 404** | 检查是否调用了复数路径（应调用单数 `/experiments/:id/visual-changeset`） | 查看 Network 面板请求 URL |
| **扩展保存变更 401** | 检查是否缺少 API Key，或调用了无前缀的旧路由（扩展应调用 `/api/v1/visual-changesets/:id`） | 检查请求头 `Authorization: Bearer <apiKey>` |
| **外部 API 调用 404** | 检查是否遗漏了 `/api/v1/` 前缀 | 正确路径应为 `/api/v1/visual-changesets/:id` |
| **PUT 更新返回空 data** | 检查是否调用了旧控制器（返回 `data` 字段）还是新 Handler（返回 `visualChangeset` 字段） | 旧控制器: `{status:200, data:{...}}` <br> 新 Handler: `{visualChangeset:{...}}` |
| **POST 创建返回格式不一致** | 旧控制器返回 `{status:200, visualChangeset:{...}}`，新 Handler 返回 `{visualChangeset:{...}}` | 根据调用路径判断响应格式 |

### 调用方路径选择速查表

| 调用方 | 认证方式 | 创建路径 | 更新/删除路径 |
|--------|----------|----------|--------------|
| **前端 UI** | JWT (Cookie) | `/experiments/:id/visual-changeset`（单数） | `/visual-changesets/:id`（复数） |
| **浏览器扩展** | API Key (Bearer) | `/api/v1/experiments/:id/visual-changesets`（复数） | `/api/v1/visual-changesets/:id`（复数） |
| **外部集成** | API Key (Bearer) | `/api/v1/experiments/:id/visual-changesets`（复数） | `/api/v1/visual-changesets/:id`（复数） |
