# GrowthBook SDK 上下文与核心协作机制梳理

## 一、整体架构分层

GrowthBook SDK 采用**双层架构**设计：

```
┌──────────────────────────────────────────────────────────┐
│               React SDK (@growthbook/growthbook-react)   │
│  ┌────────────────────────────────────────────────────┐  │
│  │  GrowthBookProvider (React Context)                │  │
│  │  - useFeature / useExperiment hooks               │  │
│  │  - Renderer 注入机制                              │  │
│  └───────────────────┬────────────────────────────────┘  │
│                      │ 持有实例引用                       │
└──────────────────────┼───────────────────────────────────┘
                       │
┌──────────────────────▼───────────────────────────────────┐
│               JS Core SDK (@growthbook/growthbook)       │
│  ┌────────────────────────────────────────────────────┐  │
│  │  GrowthBook / GrowthBookClient (实例层)            │  │
│  │  - 配置持有 (_options)                            │  │
│  │  - 状态管理 (_assigned, _trackedExperiments)      │  │
│  │  - 上下文组装 (_getEvalContext)                   │  │
│  └───────────────────┬────────────────────────────────┘  │
│                      │ 传递 EvalContext                   │
│  ┌───────────────────▼────────────────────────────────┐  │
│  │  core.ts (评估层 - 有副作用)                        │  │
│  │  - evalFeature() / runExperiment()                │  │
│  │  - 依赖注入 + 可变 ctx + 回调触发                   │  │
│  └───────────────────┬────────────────────────────────┘  │
│                      │ 调用 repository                    │
│  ┌───────────────────▼────────────────────────────────┐  │
│  │  feature-repository.ts (模块级全局存储)             │  │
│  │  - 缓存 (cache: Map)                              │  │
│  │  - 活跃请求 (activeFetches: Map)                   │  │
│  │  - SSE 流 (streams: Map)                           │  │
│  │  - 订阅实例 (subscribedInstances: Map)             │  │
│  └────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

> **修正说明**：`core.ts` 不是纯函数层，评估过程会修改传入的 `ctx` 对象并触发回调，属于**有副作用的依赖注入层**。

---

## 二、上下文实例化机制

### 2.1 React 上下文实例化流程

**文件**: `packages/sdk-react/src/GrowthBookReact.tsx`

#### GrowthBookProvider 的核心设计

```typescript
// React Context 只持有 GrowthBook 实例引用，不持有状态
export const GrowthBookContext = React.createContext<GrowthBookContextValue>(
  {} as GrowthBookContextValue
);

export const GrowthBookProvider: React.FC<{
  children: React.ReactNode;
  growthbook: GrowthBook;  // 实例由外部创建并传入
}> = ({ children, growthbook }) => {
  const [_, setRenderCount] = React.useState(0);

  React.useEffect(() => {
    // 关键：向 JS 核心注入渲染回调
    growthbook.setRenderer(() => {
      setRenderCount((v) => v + 1);  // 触发 React 重渲染
    });
    return () => {
      growthbook.setRenderer(() => {});  // 清理
    };
  }, [growthbook]);

  return (
    <GrowthBookContext.Provider value={{ growthbook }}>
      {children}
    </GrowthBookContext.Provider>
  );
};
```

**设计要点**:
- **实例外部化**：`GrowthBook` 实例不在 Provider 内部创建，而是由外部传入
- **渲染控制反转**：JS 核心通过 `setRenderer` 回调控制 React 重渲染时机
- **引用传递而非值传递**：Context 只持有实例引用，避免不必要的重渲染

#### Hooks 如何消费上下文

```typescript
export function useGrowthBook<AppFeatures>(): GrowthBook<AppFeatures> {
  const { growthbook } = React.useContext(GrowthBookContext);
  if (!growthbook) throw new Error("Missing GrowthBookProvider");
  return growthbook as GrowthBook<AppFeatures>;
}

export function useFeature<T>(id: string): FeatureResult<T | null> {
  const growthbook = useGrowthBook();
  return growthbook.evalFeature<T>(id);  // 直接调用实例方法
}
```

### 2.2 JS 核心实例化流程

**文件**: `packages/sdk-js/src/GrowthBook.ts`

#### GrowthBook 类的内部状态

```typescript
export class GrowthBook<AppFeatures> {
  // 配置源（唯一真实来源）
  private context: Options;       // 别名，指向 _options
  private _options: Options;      // 完整配置对象

  // 实例级状态
  private _assigned: Map<string, { experiment: Experiment; result: Result }>;
  private _trackedExperiments: Set<string>;
  private _trackedFeatures: Record<string, string>;
  private _subscriptions: Set<SubscriptionFunction>;
  private _payload: FeatureApiResponse | undefined;
  private _decryptedPayload: FeatureApiResponse | undefined;

  // 公共状态
  public ready: boolean;
  public debug: boolean;
  public logs: Array<LogUnion>;
}
```

#### 构造函数关键初始化逻辑

```typescript
constructor(options?: Options) {
  this._options = this.context = options || {};
  this._trackedExperiments = new Set();
  this._trackedFeatures = {};
  this._subscriptions = new Set();
  this._assigned = new Map();

  // 如果构造时已有 features，标记为 ready
  if (options.features) this.ready = true;

  // 插件初始化
  if (options.plugins) {
    for (const plugin of options.plugins) plugin(this);
  }

  // 浏览器开发者模式暴露
  if (isBrowser && options.enableDevMode) {
    window._growthbook = this;
  }
}
```

---

## 三、配置共享：EvalContext 上下文拆分机制

### 3.1 上下文拆分设计

**核心思想**：将配置拆分为**全局不变**和**用户可变**两部分，通过 `EvalContext` 传递给评估层。

**文件**: `packages/sdk-js/src/GrowthBook.ts:626-674`

```typescript
// 每次评估时动态组装上下文
private _getEvalContext(): EvalContext {
  return {
    user: this._getUserContext(),    // 用户级：每次可能变化
    global: this._getGlobalContext(), // 全局级：相对稳定
    stack: {                         // 调用栈级：单次评估有效
      evaluatedFeatures: new Set(),
    },
  };
}
```

### 3.2 GlobalContext - 全局共享配置

**文件**: `packages/sdk-js/src/types/growthbook.ts:305-331`

```typescript
export type GlobalContext = {
  log: (msg: string, ctx: any) => void;
  features?: FeatureDefinitions;      // 特性定义
  experiments?: AutoExperiment[];     // 自动实验
  enabled?: boolean;                  // SDK 是否启用
  qaMode?: boolean;                   // QA 模式
  savedGroups?: SavedGroupsValues;    // 保存的用户组
  forcedVariations?: Record<string, number>;    // 全局强制变体
  forcedFeatureValues?: Map<string, any>;       // 全局强制特性值
  trackingCallback?: TrackingCallbackWithUser;  // 全局追踪回调（带user参数）
  onFeatureUsage?: FeatureUsageCallbackWithUser; // 全局特性使用回调（带user参数）
  eventLogger?: EventLogger;                    // 事件日志
  onExperimentEval?: (experiment: Experiment, result: Result) => void;
  saveDeferredTrack?: (data: TrackingData) => void;
  recordChangeId?: (changeId: string) => void;
};
```

### 3.3 UserContext - 用户级可变配置

**文件**: `packages/sdk-js/src/types/growthbook.ts:334-356`

```typescript
export type UserContext = {
  enabled?: boolean;
  qaMode?: boolean;
  enableDevMode?: boolean;
  attributes?: Attributes;                    // 用户属性（如 id, email）
  url?: string;                               // 当前 URL
  blockedChangeIds?: string[];                // 阻止的变更 ID
  stickyBucketAssignmentDocs?: Record<...>;   // 粘性分配文档
  saveStickyBucketAssignmentDoc?: (doc) => Promise<unknown>;
  forcedVariations?: Record<string, number>;  // 用户级强制变体
  forcedFeatureValues?: Map<string, any>;     // 用户级强制特性值
  attributeOverrides?: Attributes;            // 属性覆盖
  trackingCallback?: TrackingCallback;        // 用户级追踪回调（不带user参数）
  onFeatureUsage?: FeatureUsageCallback;      // 用户级特性使用回调（不带user参数）
  trackedExperiments?: Set<string>;           // 已追踪实验（去重）
  trackedFeatureUsage?: Record<string, string>; // 已追踪特性（去重）
  devLogs?: LogUnion[];                       // 开发日志
};
```

### 3.4 StackContext - 调用栈级上下文

```typescript
export type StackContext = {
  id?: string;
  evaluatedFeatures: Set<string>;  // 防止循环依赖
};
```

### 3.5 GrowthBook 与 GrowthBookClient 的回调归属差异

#### GrowthBook 的上下文组装（单用户模式）

**文件**: `packages/sdk-js/src/GrowthBook.ts:636-674`

```typescript
private _getUserContext(): UserContext {
  return {
    attributes: this._options.user
      ? { ...this._options.user, ...this._options.attributes }
      : this._options.attributes,
    enableDevMode: this._options.enableDevMode,
    blockedChangeIds: this._options.blockedChangeIds,
    stickyBucketAssignmentDocs: this._options.stickyBucketAssignmentDocs,
    url: this._getContextUrl(),
    forcedVariations: this._options.forcedVariations,      // ← 在 user
    forcedFeatureValues: this._options.forcedFeatureValues, // ← 在 user
    attributeOverrides: this._options.attributeOverrides,
    saveStickyBucketAssignmentDoc: this._saveStickyBucketAssignmentDoc,
    trackingCallback: this._options.trackingCallback,       // ← 在 user
    onFeatureUsage: this._options.onFeatureUsage,           // ← 在 user
    devLogs: this.logs,
    trackedExperiments: this._trackedExperiments,
    trackedFeatureUsage: this._trackedFeatures,
  };
}

private _getGlobalContext(): GlobalContext {
  return {
    features: this._options.features,
    experiments: this._options.experiments,
    log: this.log,
    enabled: this._options.enabled,
    qaMode: this._options.qaMode,
    savedGroups: this._options.savedGroups,
    groups: this._options.groups,
    overrides: this._options.overrides,
    onExperimentEval: this._onExperimentEval,    // ← 实例级回调
    recordChangeId: this._recordChangedId,       // ← 实例级回调
    saveDeferredTrack: this._saveDeferredTrack,  // ← 实例级回调
    eventLogger: this._options.eventLogger,
    // 注意：trackingCallback / onFeatureUsage / forced* 不在 global 中
  };
}
```

#### GrowthBookClient 的上下文组装（多用户模式）

**文件**: `packages/sdk-js/src/GrowthBookClient.ts:262-295`

```typescript
// GrowthBookClient 没有 _getUserContext() 方法
// userContext 由调用方在每次调用 evalFeature 时传入
private _getEvalContext(userContext: UserContext): EvalContext {
  if (this._options.globalAttributes) {
    userContext = {
      ...userContext,
      attributes: {
        ...this._options.globalAttributes,
        ...userContext.attributes,
      },
    };
  }

  return {
    user: userContext,  // ← 直接使用传入的 userContext
    global: this._getGlobalContext(),
    stack: {
      evaluatedFeatures: new Set(),
    },
  };
}

private _getGlobalContext(): GlobalContext {
  return {
    features: this._features,
    experiments: this._experiments,
    log: this.log,
    enabled: this._options.enabled,
    qaMode: this._options.qaMode,
    savedGroups: this._options.savedGroups,
    forcedFeatureValues: this._options.forcedFeatureValues,  // ← 在 global
    forcedVariations: this._options.forcedVariations,         // ← 在 global
    trackingCallback: this._options.trackingCallback,         // ← 在 global
    onFeatureUsage: this._options.onFeatureUsage,             // ← 在 global
    // 注意：没有 onExperimentEval / recordChangeId / saveDeferredTrack
  };
}
```

#### 差异对照表

| 配置项 | GrowthBook 归属 | GrowthBookClient 归属 | 说明 |
|--------|----------------|----------------------|------|
| `forcedFeatureValues` | UserContext | GlobalContext | GrowthBook 是单用户，Client 是多用户共享全局强制值 |
| `forcedVariations` | UserContext | GlobalContext | 同上 |
| `trackingCallback` | UserContext | GlobalContext | GrowthBook 回调不带 user 参数，Client 带 user 参数 |
| `onFeatureUsage` | UserContext | GlobalContext | 同上 |
| `onExperimentEval` | GlobalContext | ❌ 不存在 | GrowthBook 实例级实验评估回调 |
| `recordChangeId` | GlobalContext | ❌ 不存在 | GrowthBook 变更 ID 记录 |
| `saveDeferredTrack` | GlobalContext | ❌ 不存在 | GrowthBook 延迟追踪保存 |
| `_getUserContext()` | ✅ 内部方法 | ❌ 不存在 | Client 需要调用方传入 userContext |

### 3.6 配置合并优先级

在 `core.ts` 中评估时，配置按以下优先级合并：

```
强制特性值优先级:
  UserContext.forcedFeatureValues → 覆盖 → GlobalContext.forcedFeatureValues

强制变体检测优先级:
  UserContext.forcedVariations → 合并 → GlobalContext.forcedVariations

追踪回调优先级:
  同时触发 GlobalContext.trackingCallback 和 UserContext.trackingCallback
```

**代码示例** (`core.ts:39-62`):
```typescript
function getForcedFeatureValues(ctx: EvalContext) {
  const ret: Map<string, any> = new Map();
  // 先加全局
  if (ctx.global.forcedFeatureValues) {
    ctx.global.forcedFeatureValues.forEach((v, k) => ret.set(k, v));
  }
  // 后加用户（覆盖）
  if (ctx.user.forcedFeatureValues) {
    ctx.user.forcedFeatureValues.forEach((v, k) => ret.set(k, v));
  }
  return ret;
}

function getForcedVariations(ctx: EvalContext) {
  // 合并而非覆盖
  if (ctx.global.forcedVariations && ctx.user.forcedVariations) {
    return { ...ctx.global.forcedVariations, ...ctx.user.forcedVariations };
  }
  return ctx.global.forcedVariations || ctx.user.forcedVariations || {};
}
```

---

## 四、特性请求流程

### 4.1 完整调用链路

```
React Component
    │
    ▼ useFeature("my-feature")
GrowthBook.evalFeature(id)
    │
    ▼ _getEvalContext()  [UserContext + GlobalContext + StackContext]
core.evalFeature(id, ctx)  [依赖注入，有副作用]
    │
    ├─► 修改 ctx.stack.evaluatedFeatures.add(id)
    ├─► 检查循环依赖
    ├─► 检查强制值 (getForcedFeatureValues)
    ├─► 检查特性是否存在
    ├─► 遍历规则 (FeatureRule[])
    │   ├─► 先决条件评估 (parentConditions → 递归 evalFeature)
    │   ├─► 过滤器评估 (filters)
    │   ├─► 条件评估 (condition → mongrule)
    │   ├─► 百分比推送 (coverage → hash)
    │   └─► 实验规则 → runExperiment()
    │        ├─► 修改 ctx.user.trackedExperiments
    │        ├─► 修改 ctx.user.stickyBucketAssignmentDocs
    │        ├─► 调用 saveStickyBucketAssignmentDoc 持久化
    │        └─► 触发 onExperimentViewed 回调
    ├─► 修改 ctx.user.trackedFeatureUsage
    ├─► 修改 ctx.user.devLogs
    └─► 触发 onFeatureUsage() 回调
    │
    ▼ 返回 FeatureResult
```

> **修正说明**：评估过程会**修改传入的 ctx 对象**（stack.evaluatedFeatures、user.trackedExperiments、user.trackedFeatureUsage、user.stickyBucketAssignmentDocs）并触发多个回调，**不是纯函数**。

### 4.2 核心评估函数的副作用分析

**文件**: `packages/sdk-js/src/core.ts:172-383`

```typescript
export function evalFeature<V = unknown>(
  id: string,
  ctx: EvalContext  // 依赖注入，不直接引用 GrowthBook 实例
): FeatureResult<V | null> {
  // 副作用 1: 修改 ctx.stack.evaluatedFeatures
  if (ctx.stack.evaluatedFeatures.has(id)) {
    return getFeatureResult(ctx, id, null, "cyclicPrerequisite");
  }
  ctx.stack.evaluatedFeatures.add(id);
  ctx.stack.id = id;

  // ... 强制值检查、特性存在性检查 ...

  const feature = ctx.global.features[id];

  if (feature.rules) {
    for (const rule of feature.rules) {
      // ... 先决条件、过滤器、条件、百分比检查 ...

      if (rule.variations) {
        const exp: Experiment<V> = { /* 构造实验 */ };
        // 调用 runExperiment（内部有大量副作用）
        const { result } = runExperiment(exp, id, ctx);
        if (result.inExperiment && !result.passthrough) {
          return getFeatureResult(ctx, id, result.value, "experiment", rule.id, exp, result);
        }
      }
    }
  }

  return getFeatureResult(ctx, id, feature.defaultValue ?? null, "defaultValue");
}
```

#### getFeatureResult 中的副作用

**文件**: `packages/sdk-js/src/core.ts:787-812`

```typescript
function getFeatureResult<T>(ctx, key, value, source, ruleId?, experiment?, result?): FeatureResult<T> {
  const ret: FeatureResult = { value, on: !!value, off: !value, source, ruleId: ruleId || "" };
  if (experiment) ret.experiment = experiment;
  if (result) ret.experimentResult = result;

  // 副作用 2: 触发 onFeatureUsage 回调
  if (source !== "override") {
    onFeatureUsage(ctx, key, ret);
  }

  return ret;
}
```

#### onFeatureUsage 中的副作用

**文件**: `packages/sdk-js/src/core.ts:125-170`

```typescript
function onFeatureUsage(ctx, key, ret): void {
  // 副作用 3: 修改 ctx.user.trackedFeatureUsage
  if (ctx.user.trackedFeatureUsage) {
    const stringifiedValue = JSON.stringify(ret.value);
    if (ctx.user.trackedFeatureUsage[key] === stringifiedValue) return;
    ctx.user.trackedFeatureUsage[key] = stringifiedValue;

    // 副作用 4: 修改 ctx.user.devLogs
    if (ctx.user.enableDevMode && ctx.user.devLogs) {
      ctx.user.devLogs.push({
        featureKey: key,
        result: ret,
        timestamp: Date.now().toString(),
        logType: "feature",
      });
    }
  }

  // 副作用 5: 触发全局和用户级回调
  if (ctx.global.onFeatureUsage) safeCall(() => ctx.global.onFeatureUsage(key, ret, ctx.user));
  if (ctx.user.onFeatureUsage) safeCall(() => ctx.user.onFeatureUsage(key, ret));
  if (ctx.global.eventLogger) safeCall(() => ctx.global.eventLogger(...));
}
```

#### runExperiment 中的副作用

**文件**: `packages/sdk-js/src/core.ts:727-770`

```typescript
// runExperiment 内部的副作用：

// 副作用 6: 修改 ctx.user.trackedExperiments
function onExperimentViewed(ctx, experiment, result) {
  if (ctx.user.trackedExperiments) {
    const k = getExperimentDedupeKey(experiment, result);
    if (ctx.user.trackedExperiments.has(k)) return [];
    ctx.user.trackedExperiments.add(k);  // ← 修改
  }
  // ... 触发追踪回调
}

// 副作用 7: 修改 ctx.user.stickyBucketAssignmentDocs
// 副作用 8: 调用 saveStickyBucketAssignmentDoc 持久化存储
if (ctx.user.saveStickyBucketAssignmentDoc && !experiment.disableStickyBucketing) {
  const { changed, key: attrKey, doc } = generateStickyBucketAssignmentDoc(...);
  if (changed) {
    ctx.user.stickyBucketAssignmentDocs =
      ctx.user.stickyBucketAssignmentDocs || {};
    ctx.user.stickyBucketAssignmentDocs[attrKey] = doc;  // ← 修改
    ctx.user.saveStickyBucketAssignmentDoc(doc);  // ← 持久化
  }
}

// 副作用 9: 触发 saveDeferredTrack
if (trackingCalls.length === 0 && ctx.global.saveDeferredTrack) {
  ctx.global.saveDeferredTrack({ experiment, result });
}

// 副作用 10: 触发 recordChangeId
"changeId" in experiment && experiment.changeId &&
  ctx.global.recordChangeId && ctx.global.recordChangeId(experiment.changeId);
```

---

## 五、缓存复用机制

### 5.1 模块级全局存储设计

**关键洞察**：`feature-repository.ts` 中的所有状态都是**模块级全局变量**，**不隶属于任何 GrowthBook 实例**。

**文件**: `packages/sdk-js/src/feature-repository.ts:108-117`

```typescript
// 模块级全局状态（所有实例共享）
const subscribedInstances: Map<string, Set<GrowthBook | GrowthBookClient>> = new Map();
let cacheInitialized = false;
const cache: Map<string, CacheEntry> = new Map();                  // 特性缓存
const activeFetches: Map<string, Promise<FetchResponse>> = new Map(); // 进行中的请求
const streams: Map<string, ScopedChannel> = new Map();             // SSE 流
const supportsSSE: Set<string> = new Set();                        // SSE 支持标记
```

**设计意图**：
- 多个 `GrowthBook` 实例共享同一套缓存
- 同一 `clientKey` 的多个实例只会发起一次网络请求
- SSE 流也是共享的，一条更新通知所有订阅实例

### 5.2 缓存 Key 生成策略

**文件**: `packages/sdk-js/src/feature-repository.ts:258-283`

```typescript
// 基础 Key: apiHost + clientKey
function getKey(instance: GrowthBook | GrowthBookClient): string {
  const [apiHost, clientKey] = instance.getApiInfo();
  return `${apiHost}||${clientKey}`;
}

// 缓存 Key：对于 remoteEval 模式，需要包含用户属性
function getCacheKey(instance: GrowthBook | GrowthBookClient): string {
  const baseKey = getKey(instance);

  // 非远程评估模式：仅使用 baseKey
  if (!("isRemoteEval" in instance) || !instance.isRemoteEval()) {
    return baseKey;
  }

  // 远程评估模式：包含影响评估结果的变量
  const attributes = instance.getAttributes();
  const cacheKeyAttributes =
    instance.getCacheKeyAttributes() || Object.keys(instance.getAttributes());
  const ca: Attributes = {};
  cacheKeyAttributes.forEach(key => { ca[key] = attributes[key]; });

  const fv = instance.getForcedVariations();
  const url = instance.getUrl();

  return `${baseKey}||${JSON.stringify({ ca, fv, url })}`;
}
```

### 5.3 remoteEval 缓存键与请求载荷不对齐

#### 缓存键组成（getCacheKey）

```typescript
// 缓存键包含 3 部分（经过过滤）
return `${baseKey}||${JSON.stringify({
  ca,   // 经过 cacheKeyAttributes 过滤的属性子集
  fv,   // forcedVariations
  url,  // 当前 URL
})}`;
```

#### 实际请求载荷（fetchFeatures）

**文件**: `packages/sdk-js/src/feature-repository.ts:392-403`

```typescript
// 请求载荷包含 4 部分（完整）
payload: {
  attributes: instance.getAttributes(),              // 完整属性，不过滤
  forcedVariations: instance.getForcedVariations(),  // 同缓存键
  forcedFeatures: Array.from(instance.getForcedFeatures().entries()),  // ← 缓存键缺少！
  url: instance.getUrl(),                            // 同缓存键
}
```

#### 不对齐问题分析

| 字段 | 缓存键 | 请求载荷 | 说明 |
|------|--------|----------|------|
| `attributes` | `ca`（经 `cacheKeyAttributes` 过滤） | 完整 `getAttributes()` | 可能导致不同属性集但相同缓存键的请求共享缓存 |
| `forcedFeatures` | ❌ 缺失 | ✅ 包含 | 不同 `forcedFeatures` 的请求会错误共享缓存 |
| `forcedVariations` | ✅ `fv` | ✅ 相同 | 对齐 |
| `url` | ✅ `url` | ✅ 相同 | 对齐 |

**边界情况**：
- 当设置了 `forcedFeatures` 时，缓存键不会包含它，导致相同属性/URL/forcedVariations 但不同 forcedFeatures 的请求会命中同一缓存
- 当 `cacheKeyAttributes` 只配置了部分属性时，属性变化但缓存键不变，也会导致缓存命中错误

### 5.4 SWR (Stale-While-Revalidate) 缓存策略

**文件**: `packages/sdk-js/src/feature-repository.ts:206-256`

```typescript
async function fetchFeaturesWithCache({
  instance, allowStale, timeout, skipCache
}): Promise<FetchResponse> {
  const cacheKey = getCacheKey(instance);
  const now = new Date();

  await initializeCache();

  const existing = !cacheSettings.disableCache && !skipCache
    ? cache.get(cacheKey)
    : undefined;

  // 缓存有效或允许陈旧数据
  if (existing && (allowStale || existing.staleAt > now) && existing.staleAt > minStaleAt) {
    if (existing.staleAt < now) {
      fetchFeatures(instance);  // 后台异步刷新
    } else {
      startAutoRefresh(instance);  // 启动 SSE 监听
    }
    return { data: existing.data, success: true, source: "cache" };
  }

  // 缓存失效，发起网络请求
  const res = await promiseTimeout(fetchFeatures(instance), timeout);
  return res || { data: null, success: false, source: "timeout" };
}
```

### 5.5 请求去重机制

**文件**: `packages/sdk-js/src/feature-repository.ts:381-446`

```typescript
async function fetchFeatures(instance: GrowthBook | GrowthBookClient): Promise<FetchResponse> {
  const cacheKey = getCacheKey(instance);

  // 关键：如果已有进行中的请求，直接返回同一个 Promise
  let promise = activeFetches.get(cacheKey);
  if (!promise) {
    const fetcher = remoteEval
      ? helpers.fetchRemoteEvalCall({ /* 远程评估请求 */ })
      : helpers.fetchFeaturesCall({ /* 普通请求 */ });

    promise = fetcher
      .then(res => res.json())
      .then(data => {
        onNewFeatureData(key, cacheKey, data);  // 更新缓存 + 通知所有实例
        startAutoRefresh(instance);
        activeFetches.delete(cacheKey);
        return { data, success: true, source: "network" };
      })
      .catch(e => {
        activeFetches.delete(cacheKey);
        return { data: null, source: "error", success: false, error: e };
      });

    activeFetches.set(cacheKey, promise);
  }
  return promise;
}
```

### 5.6 SSE 只对已订阅实例生效的边界

#### 订阅流程

**文件**: `packages/sdk-js/src/feature-repository.ts:568-581`

```typescript
export function startStreaming(
  instance: GrowthBook | GrowthBookClient,
  options: InitOptions | InitSyncOptions,
) {
  if (options.streaming) {
    if (!instance.getClientKey()) {
      throw new Error("Must specify clientKey to enable streaming");
    }
    if (options.payload) {
      startAutoRefresh(instance, true);
    }
    subscribe(instance);  // ← 只有调用 startStreaming 且 streaming=true 才会订阅
  }
}

function subscribe(instance): void {
  const key = getKey(instance);
  const subs = subscribedInstances.get(key) || new Set();
  subs.add(instance);
  subscribedInstances.set(key, subs);
}
```

#### SSE 事件通知边界

**文件**: `packages/sdk-js/src/feature-repository.ts:473-484`

```typescript
cb: (event: MessageEvent<string>) => {
  try {
    if (event.type === "features-updated") {
      // 只通知已订阅的实例
      const instances = subscribedInstances.get(key);
      instances && instances.forEach(instance => fetchFeatures(instance));
    } else if (event.type === "features") {
      const json = JSON.parse(event.data);
      // onNewFeatureData 也只通知已订阅实例
      onNewFeatureData(key, cacheKey, json);
    }
  } catch (e) { onSSEError(channel); }
}
```

#### onNewFeatureData 通知边界

**文件**: `packages/sdk-js/src/feature-repository.ts:338-371`

```typescript
function onNewFeatureData(key, cacheKey, data): void {
  // ... 更新缓存 ...

  // 只通知已订阅该 key 的实例
  const instances = subscribedInstances.get(key);
  instances && instances.forEach(instance => refreshInstance(instance, data));
}
```

#### 边界条件总结

| 场景 | 是否接收 SSE 更新 | 说明 |
|------|------------------|------|
| `init({ streaming: true })` | ✅ 是 | 调用 `startStreaming` → `subscribe` |
| `init({ streaming: false })` | ❌ 否 | 不订阅 |
| `init()` 不传 streaming | ❌ 否 | 默认不订阅 |
| 直接 `new GrowthBook({ clientKey })` | ❌ 否 | 不调用 init 也不订阅 |
| 实例已 `destroy()` | ❌ 否 | `unsubscribe` 从集合中移除 |
| 同一 clientKey 的不同实例 | 取决于各自是否订阅 | 实例独立管理订阅状态 |

### 5.7 多实例广播机制

```typescript
function onNewFeatureData(
  key: string,        // apiHost + clientKey
  cacheKey: string,   // 完整缓存 key
  data: FeatureApiResponse
): void {
  // 1. 版本检查：内容未变化则只延长过期时间
  const version = data.dateUpdated || "";
  const existing = cache.get(cacheKey);
  if (existing && version && existing.version === version) {
    existing.staleAt = new Date(Date.now() + cacheSettings.staleTTL);
    updatePersistentCache();
    return;
  }

  // 2. 更新内存缓存
  if (!cacheSettings.disableCache) {
    cache.set(cacheKey, {
      data,
      version,
      staleAt: new Date(Date.now() + cacheSettings.staleTTL),
      sse: supportsSSE.has(key),
    });
    cleanupCache();
  }

  // 3. 持久化到 localStorage
  updatePersistentCache();

  // 4. 只通知已订阅该 key 的 GrowthBook 实例
  const instances = subscribedInstances.get(key);
  instances && instances.forEach(instance => refreshInstance(instance, data));
}
```

### 5.8 实例订阅与取消订阅

```typescript
function subscribe(instance: GrowthBook | GrowthBookClient): void {
  const key = getKey(instance);
  const subs = subscribedInstances.get(key) || new Set();
  subs.add(instance);
  subscribedInstances.set(key, subs);
}

export function unsubscribe(instance: GrowthBook | GrowthBookClient): void {
  subscribedInstances.forEach(s => s.delete(instance));
}

// 在 destroy() 中调用
public destroy(options?: DestroyOptions) {
  unsubscribe(this);
  // ... 其他清理
}
```

---

## 六、状态同步机制

### 6.1 React 重渲染触发

**核心机制**：`GrowthBook` 实例通过 `renderer` 回调通知 React 更新。

**文件**: `packages/sdk-js/src/GrowthBook.ts:370-378`

```typescript
private _render() {
  if (this._renderer) {
    try {
      this._renderer();  // 调用 React 注入的 setRenderCount
    } catch (e) {
      console.error("Failed to render", e);
    }
  }
}
```

#### 实际触发 `_render()` 的场景（纠正后）

**文件**: `packages/sdk-js/src/GrowthBook.ts`

1. **setPayload** - 新特性数据到达 (`:213-230`)
   ```typescript
   public async setPayload(payload: FeatureApiResponse): Promise<void> {
     this._payload = payload;
     // ... 解密、更新 _options.features
     this.ready = true;
     this._render();  // ✅ 触发重渲染
   }
   ```

2. **setAttributes** - 用户属性变更 (`:424-435`)
   ```typescript
   public async setAttributes(attributes: Attributes) {
     this._options.attributes = attributes;
     // ... 刷新粘性桶
     this._render();  // ✅ 触发重渲染
   }
   ```

3. **setForcedVariations** - 强制变体变更 (`:454-462`)
   ```typescript
   public async setForcedVariations(vars: Record<string, number>) {
     this._options.forcedVariations = vars || {};
     // ... remoteEval 刷新
     this._render();  // ✅ 触发重渲染
   }
   ```

4. **setForcedFeatures** - 强制特性变更 (`:465-468`)
   ```typescript
   public setForcedFeatures(map: Map<string, any>) {
     this._options.forcedFeatureValues = map;
     this._render();  // ✅ 触发重渲染
   }
   ```

5. **setFeatures** (deprecated) - 直接设置特性 (`:381-385`)
   ```typescript
   public setFeatures(features: Record<string, FeatureDefinition>) {
     this._options.features = features;
     this.ready = true;
     this._render();  // ✅ 触发重渲染
   }
   ```

6. **setURL** - URL 变更 (`:470-480`)
   ```typescript
   public async setURL(url: string) {
     if (url === this._options.url) return;
     this._options.url = url;
     this._redirectedUrl = "";
     if (this._options.remoteEval) {
       await this._refreshForRemoteEval();  // 这会调用 setPayload → _render
       this._updateAllAutoExperiments(true);
       return;
     }
     this._updateAllAutoExperiments(true);  // ❌ 非 remoteEval 模式下不调用 _render
   }
   ```

> **修正说明**：`setURL` 在 **非 remoteEval 模式下不直接触发 `_render()`**，只调用 `_updateAllAutoExperiments()` 更新自动实验。只有在 remoteEval 模式下，通过 `_refreshForRemoteEval()` → `setPayload()` 间接触发重渲染。

### 6.2 订阅机制（Subscriptions）

**文件**: `packages/sdk-js/src/GrowthBook.ts:515-520`

```typescript
public subscribe(cb: SubscriptionFunction): () => void {
  this._subscriptions.add(cb);
  return () => { this._subscriptions.delete(cb); };
}
```

**订阅触发条件** - 实验分配结果变化时 (`GrowthBook.ts:841-870`):

```typescript
private _onExperimentEval<T>(experiment: Experiment<T>, result: Result<T>) {
  const prev = this._assigned.get(experiment.key);
  this._assigned.set(experiment.key, { experiment, result });

  if (this._subscriptions.size > 0) {
    this._fireSubscriptions<T>(experiment, result, prev);
  }
}

private _fireSubscriptions<T>(experiment, result, prev) {
  // 只有分配变化时才触发
  if (!prev ||
      prev.result.inExperiment !== result.inExperiment ||
      prev.result.variationId !== result.variationId) {
    this._subscriptions.forEach(cb => {
      try { cb(experiment, result); } catch (e) { console.error(e); }
    });
  }
}
```

### 6.3 去重追踪机制

**已触发实验去重** (`core.ts:78-84`):
```typescript
function onExperimentViewed(ctx, experiment, result): Promise<void>[] {
  // 确保每个唯一实验只触发一次追踪回调
  if (ctx.user.trackedExperiments) {
    const k = getExperimentDedupeKey(experiment, result);
    if (ctx.user.trackedExperiments.has(k)) return [];
    ctx.user.trackedExperiments.add(k);
  }
  // ... 触发回调
}
```

**已使用特性去重** (`core.ts:131-134`):
```typescript
function onFeatureUsage(ctx, key, ret): void {
  // 只有值变化时才重新追踪
  if (ctx.user.trackedFeatureUsage) {
    const stringifiedValue = JSON.stringify(ret.value);
    if (ctx.user.trackedFeatureUsage[key] === stringifiedValue) return;
    ctx.user.trackedFeatureUsage[key] = stringifiedValue;
  }
  // ... 触发回调
}
```

---

## 七、协作面总结

### 7.1 关键协作接口（修正后）

| 协作面 | 实现方式 | 核心代码位置 |
|--------|---------|-------------|
| **实例持有** | React Context 持有 JS 实例引用 | `GrowthBookReact.tsx:26-28` |
| **渲染控制** | JS 核心通过 `setRenderer` 回调控制 React | `GrowthBookReact.tsx:183-193` |
| **配置传递** | `_getEvalContext()` 拆分为 Global+User+Stack | `GrowthBook.ts:626-674` |
| **评估逻辑** | `core.ts` 依赖注入 + 有副作用的评估 | `core.ts:172-383` |
| **缓存共享** | `feature-repository.ts` 模块级全局变量 | `feature-repository.ts:108-117` |
| **请求去重** | `activeFetches` Map 存储进行中的 Promise | `feature-repository.ts:390-391` |
| **多实例广播** | `subscribedInstances` Map + `onNewFeatureData` | `feature-repository.ts:368-371` |
| **状态同步** | `_render()` + `_fireSubscriptions()` | `GrowthBook.ts:370,849` |

### 7.2 数据流图

```
┌─────────────────────────────────────────────────────────────┐
│  GrowthBookProvider (React)                                 │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  const [_, setRenderCount] = useState(0)             │   │
│  └───────────────────┬──────────────────────────────────┘   │
│                      │ setRenderer(callback)                │
└──────────────────────┼──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│  GrowthBook instance (JS Core)                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  _options: Options                                  │   │
│  │  _assigned: Map                                     │   │
│  │  _renderer: callback                                │───┼─── 调用 _render()
│  └───────────────────┬──────────────────────────────────┘   │
│                      │ _getEvalContext()                     │
│              ┌───────┴───────┐                               │
│              ▼               ▼                               │
│    GlobalContext       UserContext                           │
│    (features,          (attributes,                         │
│     experiments,       url, forced*,                        │
│     savedGroups,       tracked*,                            │
│     instance cb)        devLogs)                            │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│  core.ts (评估层 - 有副作用)                                 │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  evalFeature(id, ctx)                                │───┼─── 修改 ctx.*
│  │  runExperiment(exp, ctx)                             │───┼─── 触发回调
│  └──────────────────────────────────────────────────────┘   │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│  feature-repository.ts (模块级全局)                          │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  cache: Map<cacheKey, CacheEntry>                    │   │
│  │  activeFetches: Map<cacheKey, Promise>               │───┼─── 多实例共享
│  │  subscribedInstances: Map<key, Set<Instance>>        │───┼─── 仅订阅实例可见
│  │  streams: Map<key, ScopedChannel>                    │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 7.3 设计亮点

1. **关注点分离**：React 层只负责 UI 集成，JS 核心负责业务逻辑，core 层负责评估算法
2. **依赖注入**：评估逻辑不直接依赖 GrowthBook 实例，而是通过 EvalContext 注入
3. **缓存全局共享**：模块级存储实现多实例缓存复用，避免重复网络请求
4. **渲染控制反转**：JS 核心通过回调控制 React 重渲染，保持实例的框架无关性
5. **上下文分层**：Global/User/Stack 三层上下文设计，清晰区分不同生命周期的配置
6. **双客户端设计**：GrowthBook（单用户）与 GrowthBookClient（多用户）适配不同场景

### 7.4 已确认的边界与注意点

1. **缓存是全局的**：多个 `GrowthBook` 实例（即使在不同 React 应用中）共享同一缓存
2. **实例必须手动销毁**：调用 `destroy()` 才能从 `subscribedInstances` 中移除，否则内存泄漏
3. **SSE 流也是全局共享**：同一 clientKey 的多个实例共享一个 SSE 连接
4. **remoteEval 缓存键与请求载荷不对齐**：缺少 `forcedFeatures`，`attributes` 经过过滤可能导致缓存错误命中
5. **SSE 只对已订阅实例生效**：必须 `init({ streaming: true })` 才会订阅并接收实时更新
6. **setURL 不总是触发重渲染**：非 remoteEval 模式下 `setURL` 不调用 `_render()`
7. **core.ts 评估层有副作用**：会修改传入的 ctx 对象（tracked*、devLogs、stickyBucketAssignmentDocs 等）
8. **GrowthBook 与 GrowthBookClient 回调归属不同**：单用户 vs 多用户模式导致回调在 Global/User 中的归属不同
