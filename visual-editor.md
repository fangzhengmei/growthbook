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
      apiKey,       // 临时API密钥
    },
  },
  window.location.origin
);
```

### 3.3 URL 参数传递

打开可视化编辑器时，通过 URL 查询参数传递上下文：

```
?vc-id=vcs_xxx          // 变更集ID
&v-idx=1                // 变体索引
&exp-url=...            // 实验管理页面URL
&ai-enabled=true        // AI功能开关
```

> **注意**：DOM 选择器生成的核心逻辑位于浏览器扩展中（不在此代码库）。扩展通过 `document.querySelector` API 反向生成唯一的 CSS 选择器路径。

---

## 四、DOM 选择器生成与存储

### 4.1 选择器格式

虽然选择器生成逻辑在扩展中，但从数据结构可以看出其设计：

- **唯一标识**：通过 CSS 选择器精确定位元素，如 `"#header .nav-item:nth-child(2)"`
- **层级路径**：从目标元素向上遍历到 `document`，组合 id、class、标签名等属性
- **属性优先**：优先使用 `id` → `data-*` 属性 → `class` → 标签名 → `nth-child`

### 4.2 操作类型

三种基础 DOM 操作（`packages/shared/types/visual-changeset.d.ts:1-8`）：

| 操作 | 含义 | 示例 attribute |
|------|------|---------------|
| `set` | 设置/替换属性值 | `html` (innerHTML), `class`, `href`, `src` |
| `append` | 追加内容 | `html`, `class` |
| `remove` | 移除元素或属性 | `element`, `class`, `attribute` |

### 4.3 拖拽重排支持

通过 `parentSelector` 和 `insertBeforeSelector` 支持元素拖拽：

```typescript
// 将 .item-3 移动到 .item-1 之前
{
  selector: ".item-3",
  action: "set",
  attribute: "html",
  parentSelector: ".container",
  insertBeforeSelector: ".item-1"
}
```

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

### 5.2 变更生命周期管理

#### 5.2.1 激活状态追踪

```typescript
private _activeAutoExperiments: Map<
  AutoExperiment,
  { valueHash: string; undo: () => void }
>;
```

- **valueHash**：`JSON.stringify(result.value)`，用于检测变体值是否变化
- **undo**：撤销函数，用于 URL 变化或实验停止时回滚

#### 5.2.2 URL 变化响应

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

### 5.3 防闪烁机制

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

### 6.1 完整流程图

```
用户在GrowthBook后台创建实验
        ↓
1. 创建 VisualChangeset (后端API)
   POST /experiments/:id/visual-changesets
   ├─ id: vcs_xxx
   ├─ experiment: exp_xxx
   ├─ editorUrl: https://target.com/page
   └─ urlPatterns: [{type: "simple", pattern: "/page"}]
        ↓
2. 用户点击 "Open Visual Editor"
   ├─ 检测浏览器扩展是否安装
   ├─ 获取临时API密钥: GET /visual-editor/key
   └─ 通过 postMessage 向扩展传递 apiHost + apiKey
        ↓
3. 跳转到目标页面（带vc-id参数）
   https://target.com/page?vc-id=vcs_xxx&v-idx=1
        ↓
4. 浏览器扩展检测到 vc-id 参数
   ├─ 注入编辑器UI覆盖层到页面
   ├─ 从GrowthBook API拉取变更集数据
   └─ 初始化选择模式
        ↓
5. 用户选择页面元素（选择模式）
   ├─ 扩展监听鼠标hover和click事件
   ├─ 高亮目标元素（添加蓝色边框等）
   ├─ 捕获点击的 DOM 元素
   └─ 生成唯一 CSS 选择器（从元素向上遍历DOM树）
        ↓
6. 用户编辑变体内容
   ├─ 修改 InnerHTML / 属性 / CSS类
   ├─ 或拖拽元素调整位置
   ├─ 实时预览变更效果
   └─ 生成 DOMMutation 对象
        ↓
7. 保存变更到后端
   PUT /visual-changesets/:id
   {
     visualChanges: [{
       id: vc_xxx,
       variation: var_xxx,
       css: "...",
       js: "...",
       domMutations: [...]
     }]
   }
        ↓
8. 后端更新并触发SDK缓存刷新
   ├─ 更新 VisualChangesetModel
   ├─ 标记 experiment.hasVisualChangesets = true
   └─ queueSDKPayloadRefresh() → CDN更新
        ↓
9. 生产环境SDK拉取实验配置
   GET /api/features/:clientKey
   返回 payload.experiments 数组中包含 visual 类型实验
        ↓
10. SDK运行时应用变更
    ├─ 检查URL匹配: isURLTargeted(url, urlPatterns)
    ├─ 实验分配: runExperiment() → 确定变体
    └─ 应用变更: _applyDOMChanges() → 注入CSS/JS + 执行DOM变更
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

#### 6.2.2 URL 匹配逻辑

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

#### 6.2.3 SDK 有效负载分发

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

| 技术点 | 实现方式 | 关键文件 |
|--------|----------|----------|
| 浏览器扩展通信 | `window.postMessage` + 扩展资源探测 | `OpenVisualEditorLink.tsx` |
| DOM 选择器 | 浏览器扩展生成 CSS 选择器字符串 | 扩展代码（外部） |
| DOM 变更回放 | `dom-mutator` 库声明式变更 | `GrowthBook.ts:1084-1110` |
| 变更撤销 | 每个操作生成 undo 函数，聚合执行 | `GrowthBook.ts:1086-1109` |
| URL 匹配 | Simple（通配符）+ Regex 双模式 | `util.ts:86-182` |
| 防闪烁 | 先隐藏页面，变更应用后显示 | `auto-wrapper.ts:60-97` |
| 变体绑定 | `VisualChange.variation` 关联变体ID | `VisualChangesetModel.ts:252-258` |
| SDK 分发 | 变更集更新触发 payload 刷新 | `VisualChangesetModel.ts:439-473` |

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

## 十、仓内可证据部分与扩展外未知部分区分

### 10.1 仓内可证据部分（代码库中有完整实现）

以下功能在代码库中有完整的源代码证据：

| 模块 | 关键文件 | 功能描述 |
|------|----------|----------|
| 扩展检测 | `packages/front-end/components/OpenVisualEditorLink.tsx` | 通过 fetch 扩展内部资源探测扩展是否安装 |
| 消息通信 | 同上 | 通过 `window.postMessage` 发送 `GB_REQUEST_OPEN_VISUAL_EDITOR` 消息 |
| 数据结构 | `packages/shared/types/visual-changeset.d.ts` | `VisualChangesetInterface`、`VisualChange`、`DOMMutation` 类型定义 |
| 验证器 | `packages/shared/src/validators/visual-changesets.ts` | 可视化变更集参数验证逻辑 |
| 后端模型 | `packages/back-end/src/models/VisualChangesetModel.ts` | CRUD、变体绑定、SDK 负载刷新 |
| 后端路由 | `packages/back-end/src/api/visual-changesets/*.ts` | REST API 接口定义 |
| 变更回放 | `packages/sdk-js/src/GrowthBook.ts:1084-1110` | `_applyDOMChanges` 注入 CSS/JS 并调用 `dom-mutator` |
| 生命周期管理 | `packages/sdk-js/src/GrowthBook.ts:676-802` | `_runAutoExperiment`、`_updateAllAutoExperiments` |
| URL 匹配 | `packages/sdk-js/src/util.ts:86-182` | `isURLTargeted`、`_evalSimpleUrlTarget` |
| 防闪烁 | `packages/sdk-js/src/auto-wrapper.ts:60-97` | `setAntiFlicker` / `unsetAntiFlicker` |
| 安全控制 | `packages/sdk-js/src/GrowthBook.ts:1006-1055` | `_isAutoExperimentBlockedByContext` 各种开关 |
| 测试用例 | `packages/sdk-js/test/visual-changes.test.ts` | 11 个完整的端到端测试场景 |

### 10.2 扩展外未知部分（浏览器扩展代码不在此仓库）

以下功能在浏览器扩展中实现，代码库中只有调用接口和数据格式约定：

| 功能 | 接口约定 | 扩展内部实现（推测） |
|------|----------|---------------------|
| **DOM 选择器生成** | 输出标准 CSS 选择器字符串 | 从目标元素向上遍历 DOM 树，组合 id/class/标签名/nth-child |
| **元素高亮交互** | 无直接代码证据 | 鼠标 hover 时添加高亮边框样式（如蓝色轮廓） |
| **编辑器 UI 覆盖层** | 通过 `?vc-id=` URL 参数触发 | iframe 或 fixed 定位的侧边栏/工具栏 |
| **拖拽重排交互** | `parentSelector` + `insertBeforeSelector` 格式 | HTML5 Drag & Drop API 监听拖拽事件 |
| **实时预览** | 无直接代码证据 | 编辑时立即调用 `dom-mutator` 应用变更 |
| **元素属性拾取** | `attribute` 字段约定为标准 HTML 属性名 | `element.getAttributeNames()` + 属性值读取 |
| **选择器冲突检测** | 无直接代码证据 | 验证 `document.querySelectorAll(selector).length === 1` |

> **边界说明**：代码库与扩展的交互边界在 `DOMMutation` 数据结构。扩展负责生成符合该结构的数据，代码库负责存储、分发和回放。扩展内部的选择器生成算法不影响 SDK 运行，只要输出的选择器能被 `document.querySelector` 正确解析即可。

---

## 十一、DOM 选择器生成依据（基于仓内证据反推）

### 11.1 选择器必须满足的约束条件

从 `dom-mutator` 库的接口规范（https://github.com/growthbook/dom-mutator）和测试用例反推，选择器必须满足：

```
约束1: 有效的 CSS 选择器语法
  ↓ 必须能通过 document.querySelector(selector) 解析
  ↓ 不能包含伪元素 ::before/::after（不可变）
  ↓ 不能包含伪类 :hover/:active（运行时不生效）

约束2: 可被 MutationObserver 持续监听
  ↓ 选择器匹配的元素在 DOM 树中必须是可观察的
  ↓ 支持 Shadow DOM（取决于 dom-mutator 实现）

约束3: 唯一性
  ↓ document.querySelectorAll(selector).length === 1
  ↓ 若不唯一，dom-mutator 会应用到所有匹配元素
```

### 11.2 选择器生成优先级（从测试用例和类型反推）

基于 `packages/sdk-js/test/visual-changes.test.ts` 中的测试数据和 `DOMMutation` 类型定义，扩展生成选择器的优先级应为：

```
优先级 1: id 属性 → "#submit-btn"
  依据: id 在页面中唯一，解析速度最快，最稳定

优先级 2: data-* 属性 → "[data-testid='checkout-button']"
  依据: data-testid 等属性专为测试设计，不受样式变化影响

优先级 3: 稳定 class 组合 → ".btn-primary.checkout"
  依据: 多个 class 组合可提高唯一性

优先级 4: 标签名 + 层级路径 → "header > nav > ul > li:nth-child(2)"
  依据: 当无 id/data-* 属性时，使用 DOM 结构定位

优先级 5: nth-child / nth-of-type → ".item:nth-child(3)"
  依据: 同级元素无区分特征时的最后手段
```

### 11.3 `dom-mutator` 库声明式变更格式

GrowthBook 官方维护的 `dom-mutator@0.6.0` 库定义了 `DeclarativeMutation` 格式（与 SDK 中 `DOMMutation` 完全兼容）：

```typescript
// 来自 dom-mutator 官方文档
type DeclarativeMutation = {
  selector: string;           // CSS 选择器
  action: 'set' | 'append' | 'remove';  // 操作类型
  attribute: 'html' | 'class' | 'position' | string;  // 目标属性
  value?: string;             // 新值
  parentSelector?: string;    // 拖拽目标父元素
  insertBeforeSelector?: string;  // 插入位置参照
};
```

**各 attribute 含义**：

| attribute 值 | 对应 DOM API | 示例 |
|--------------|--------------|------|
| `html` | `element.innerHTML` | `<h1> → 替换为新标题` |
| `class` | `element.classList` | 添加/移除 `.active` 类 |
| `position` | `parent.insertBefore()` | 移动元素到新位置 |
| 其他字符串 | `element.setAttribute()` | `href`, `src`, `title`, `data-*` |

---

## 十二、失败时排查边界

### 12.1 选择器运行时失败场景

dom-mutator 库的设计哲学是 **静默等待元素出现**，不会抛出异常。

```
场景 1: 选择器匹配不到元素
  行为: dom-mutator 使用 MutationObserver 持续监听 DOM，元素出现后自动应用
  排查点: 
    - 检查元素是否是动态加载（如 SPA 路由切换后才渲染）
    - 检查选择器是否拼写错误（大小写敏感）
    - 检查元素是否在 Shadow DOM 或 iframe 中
  超时: 无内置超时，一直等待到 destroy() 被调用

场景 2: 选择器匹配到多个元素
  行为: dom-mutator 将变更应用到所有匹配元素
  排查点:
    - 扩展生成选择器时是否做了唯一性验证
    - 页面结构变更导致选择器精度下降
    - 可通过浏览器控制台验证: document.querySelectorAll(selector).length

场景 3: 元素被 React/Vue 重新渲染覆盖
  行为: dom-mutator 监听目标元素的 MutationObserver，外部变更后自动重新应用
  排查点:
    - 检查是否频繁触发重渲染（导致性能问题）
    - 可使用 mutate.attribute() 替代 mutate.html() 减少重绘
```

### 12.2 SDK 层面失败边界

**可在代码库中找到证据的失败场景**：

| 失败场景 | 判定位置 | 行为 | 排查点 |
|----------|----------|------|--------|
| **JS 注入被禁用** | `GrowthBook.ts:1068-1070` | 整个实验被阻止，不应用任何变更 | 检查 `disableJsInjection: true`，且变体包含 `js` 字段 |
| **可视化实验被禁用** | `GrowthBook.ts:1064-1065` | 跳过所有可视化类型实验 | 检查 `disableVisualExperiments: true` |
| **changeId 被屏蔽** | `GrowthBook.ts:1074-1076` | 按 ID 屏蔽特定变更 | 检查 `blockedChangeIds` 数组 |
| **URL 不匹配** | `util.ts:86-182` | 不应用变更，或撤销已应用的变更 | 验证 `urlPatterns` 规则是否正确 |
| **CSP 阻止脚本** | 无直接代码，需浏览器控制台查看 | JS 注入失败，CSS/DOM 变更可能仍生效 | 设置 `jsInjectionNonce` 或放宽 CSP |
| **SSR 环境** | `GrowthBook.ts:1087` | `!isBrowser` 时直接 return | 检查是否在 Node.js 环境调用 |

### 12.3 调试工具链

**仓内可用的调试手段**：

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
    });
    // 返回默认实现
    return () => { /* 自定义撤销 */ };
  }
});

// 3. 手动验证 URL 匹配
import { isURLTargeted } from '@growthbook/growthbook';
console.log(isURLTargeted(window.location.href, experiment.urlPatterns));

// 4. 暂停/恢复 dom-mutator 的全局观察
import { disconnectGlobalObserver, connectGlobalObserver } from 'dom-mutator';
// 批量 DOM 操作前暂停，避免频繁重应用
disconnectGlobalObserver();
// ... 操作 DOM ...
connectGlobalObserver();
```

### 12.4 典型排查路径

```
问题: 页面元素没有变更
  ↓
步骤1: 确认 SDK 初始化正常 → 检查 growthbook.loaded / 控制台错误
  ↓ 否
修正 SDK 配置（clientKey, apiHost）
  ↓ 是
步骤2: 确认用户被分配到实验变体 → console.log(gb.getExperimentValue(...))
  ↓ 否
检查 targeting 条件 / hashAttribute / 流量分配
  ↓ 是
步骤3: 确认 URL 匹配 → 调用 isURLTargeted() 验证
  ↓ 否
修正 urlPatterns 规则
  ↓ 是
步骤4: 确认选择器有效 → document.querySelector(selector) !== null
  ↓ 否
元素未渲染 / 选择器过期，重新在编辑器中选择
  ↓ 是
步骤5: 检查阻止开关 → disableVisualExperiments / disableJsInjection / blockedChangeIds
  ↓ 否
步骤6: 检查 CSP / 浏览器控制台错误
  ↓
最终定位到根因
```

