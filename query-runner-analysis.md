# GrowthBook 查询执行器架构分析

GrowthBook 在接入多种数据仓库（Snowflake、BigQuery、Redshift、PostgreSQL、ClickHouse 等）时，通过一套分层抽象实现了统一的查询执行模型。本文从**驱动注册**、**查询编译**、**结果归一**三个核心维度拆解其设计。

---

## 一、驱动注册机制：工厂模式下的多数据源接入

### 1.1 核心接口定义

所有数据源集成实现 `SourceIntegrationInterface` 接口（`back-end/src/types/Integration.ts:73`），该接口定义了超过 60 个方法，覆盖：
- 连接管理（`testConnection`、`cancelQuery`）
- 查询生成（`getExperimentMetricQuery`、`getDimensionSlicesQuery` 等 20+ 种查询模板）
- 查询执行（`runExperimentMetricQuery`、`runMetricValueQuery` 等执行器）
- 元数据获取（`getInformationSchema`、`getTableData`）
- 能力声明（`getSourceProperties`）

### 1.2 抽象基类分层

为避免重复实现，GrowthBook 采用了两层抽象：

```
SourceIntegrationInterface (接口)
        ↑
SqlIntegration (抽象基类, back-end/src/integrations/SqlIntegration.ts:178)
        ↑
Snowflake / BigQuery / Redshift / ... (具体驱动)
```

**`SqlIntegration`** 提供了 SQL 类数据源的通用实现：
- 统一的 `runQuery` 包装（禁止多语句 SQL、注入查询元数据）
- 各类查询的结果归一化逻辑（`runExperimentMetricQuery` 等）
- 信息_schema 查询、自动指标生成等通用能力
- 模板变量渲染、SQL 格式化

### 1.3 工厂注册与发现

驱动注册采用**简单工厂模式**，在 `back-end/src/services/datasource.ts:71` 中通过 `getIntegrationObj` 函数实现：

```typescript
function getIntegrationObj(
  context: ReqContext,
  datasource: DataSourceInterface,
): SourceIntegrationInterface {
  switch (datasource.type) {
    case "snowflake":
      return new Snowflake(context, datasource);
    case "bigquery":
      return new BigQuery(context, datasource);
    case "postgres":
      return new Postgres(context, datasource);
    // ... 14 种数据源
  }
}
```

**调用链：**
1. `getIntegrationFromDatasourceId(context, id)` → 从数据库加载数据源配置
2. `getSourceIntegrationObject(context, datasource)` → 解密连接参数
3. `getIntegrationObj(context, datasource)` → 根据 type 字段实例化具体驱动
4. 驱动构造函数中调用 `setParams(datasource.params)` 解密并设置连接参数

**设计特点：**
- 连接参数使用 AES 加密存储，`decryptDataSourceParams` 在驱动实例化时解密
- 敏感参数（密码、私钥等）通过 `getSensitiveParamKeys()` 声明，在 API 返回时自动脱敏
- 解密失败时设置 `decryptionError` 标记，上层可选择性抛出

---

## 二、查询编译过程：方言适配与模板化 SQL 生成

### 2.1 方言抽象层

SQL 方言差异通过 `SqlDialect` 接口（`shared/types/sql.d.ts:43`）隔离，每个驱动返回自己的方言实例：

```typescript
// Snowflake 驱动 (back-end/src/integrations/Snowflake.ts:21)
getSqlDialect(): SqlDialect {
  return snowflakeDialect;
}
```

**`baseDialect`**（`back-end/src/integrations/dialects/base.ts:5`）提供默认实现，各驱动按需覆盖：

| 方言方法 | 作用 | Snowflake 覆盖示例 |
|---------|------|-------------------|
| `toTimestamp(date)` | Date → SQL 时间字面量 | `'2024-01-01 12:00:00'` |
| `castToFloat(col)` | 类型转换 | `CAST(col AS DOUBLE)` |
| `jsonExtract(col, path)` | JSON 字段提取 | `PARSE_JSON(col):path::float` |
| `dateTrunc(col, granularity)` | 时间截断 | `DATE_TRUNC('day', col)` |
| `percentileApprox(col, p)` | 近似分位数 | `APPROX_PERCENTILE(col, p)` |
| `hllAggregate(col)` | HLL 草图聚合 | `HLL_ACCUMULATE(col)` |
| `getDataType(type)` | 逻辑类型 → 物理类型 | `string → VARCHAR` |

### 2.2 查询模板化生成

以实验指标查询为例，`SqlIntegration.getExperimentMetricQuery`（`back-end/src/integrations/SqlIntegration.ts:987`）的编译流程：

1. **参数解析**：接收 `ExperimentMetricQueryParams`（包含实验设置、指标、维度、分段等）
2. **CTE 组装**：
   - `getIdentitiesCTE` - 用户身份映射（跨 ID 类型关联）
   - `getSegmentCTE` - 分段过滤
   - `getMetricCTE` - 指标数据聚合
   - `getDimensionCTE` - 维度数据提取
3. **方言适配**：所有 SQL 片段通过方言对象生成，确保语法兼容
4. **SQL 格式化**：通过 `format(sql, dialect.formatDialect)` 做最终格式化

```typescript
// 查询生成调用链
ExperimentResultsQueryRunner.startQueries()
  → startExperimentResultQueries()
     → integration.getExperimentMetricQuery(params)
        → buildExperimentMetricQuerySql(dialect, datasource, params)
           → [CTE 组装 + 方言方法调用]
```

### 2.3 并发与依赖调度

`QueryRunner`（`back-end/src/queryRunners/QueryRunner.ts:107`）是查询执行的编排器，核心机制：

#### 2.3.1 数据结构：QueryPointer vs QueryInterface

理解 `runAtEnd` 的关键在于分清两种数据结构的职责边界：

**QueryPointer**（`shared/src/validators/queries.ts:13`）— 模型层的轻量引用：
```typescript
export const queryPointerValidator = z.object({
  query: z.string(),      // 指向 QueryInterface 的 Mongo _id
  status: queryStatusValidator,  // 本地缓存的状态快照
  name: z.string(),       // 业务名称（如 "metric_revenue"）
}).strict();
```
- 存储在模型（如 ExperimentSnapshot）的 `queries` 数组中
- **不包含 `runAtEnd` 字段**，只存最核心的引用信息
- 状态字段会被 `updateQueryPointers()` 定期同步更新

**QueryInterface**（`shared/types/query.d.ts:123`）— 查询文档的完整结构：
```typescript
export interface QueryInterface {
  id: string;
  query: string;           // 实际 SQL
  status: QueryStatus;     // 真实状态（数据库唯一可信源）
  dependencies?: string[];
  runAtEnd?: boolean;      // ⚠️ 仅存在于 QueryInterface 中
  // ... 其他字段：result, statistics, externalId 等
}
```
- 存储在独立的 `queries` Mongo 集合中
- `runAtEnd` 是查询创建时写入的永久属性，只能通过数据库读取

#### 2.3.2 queryMap 的数据来源与迭代顺序

`queryMap: Map<string, QueryInterface>` 是连接两者的桥梁，有两种构建路径：

**路径 A — `getQueryMap(pointers, cache)`**（`QueryRunner.ts:77`）：
```typescript
const map: QueryMap = new Map(cache);  // 先复制缓存（按缓存插入顺序）
queryDocs.forEach((query) => {
  const pointer = queries.find((qp) => qp.query === query.id);
  if (pointer) {
    map.set(pointer.name, query);  // 按 queryDocs 返回顺序追加
  }
});
```

**路径 B — `updateQueryPointers()`**（`QueryRunner.ts:945`）：
```typescript
const queryMap: QueryMap = new Map(this.finishedQueryMapCache);
queries.forEach((query) => {
  const pointer = this.model.queries.find((p) => p.query === query.id);
  if (!pointer) return;
  queryMap.set(pointer.name, query);  // 按 getQueriesByIds 返回顺序追加
  // ... 状态同步
});
```

**迭代顺序的不确定性：**
- `queryMap` 的迭代顺序 = **缓存插入顺序 + `getQueriesByIds` 返回顺序**
- `getQueriesByIds` 内部是 `QueryModel.find({ _id: { $in: ids } })`，MongoDB `$in` 查询**不保证返回顺序与输入数组顺序一致**
- 这意味着 `queuedQueries = Array.from(queryMap.values()).filter(...)` 的遍历顺序**不一定等于 `this.model.queries` 的数组顺序**

> **关键点**：`queryMap` 的 value 永远是从数据库加载的 **`QueryInterface`** 文档，只有它包含 `runAtEnd` 的真实值。但迭代顺序没有强保证，依赖数据库返回顺序。

#### 2.3.3 `runAtEnd` 的读取位置与判定逻辑

`runAtEnd` 的检查位于 `startReadyQueries()` 中（`QueryRunner.ts:431`）：

```typescript
// 遍历 queued 状态的查询
for (const query of queuedQueries) {  // query 是 QueryInterface
  // ... 依赖检查 ...
  
  // if `runAtEnd = true` run if all queries that are not marked
  // `runAtEnd` are finished
  if (query.runAtEnd) {
    const pendingQueries = this.model.queries.filter(
      (q) =>
        !queryMap.get(q.name)?.runAtEnd &&          // 🔍 从 queryMap 读取
        (q.status === "queued" || q.status === "running"),
    );
    if (pendingQueries.length) {
      logger.debug(`${query.id}: "Run at end query" waiting...`);
      return;  // ⚠️ 直接退出整个 startReadyQueries 方法
    }
  }
  // ... 执行查询 ...
}
```

**读取位置详解：**
- **外层 `if (query.runAtEnd)`**：`query` 来自 `queryMap.values()`，是 `QueryInterface`，直接读其 `runAtEnd` 字段
- **内层 `queryMap.get(q.name)?.runAtEnd`**：`q` 是 `QueryPointer`，需要通过 `queryMap` 查找对应的 `QueryInterface` 才能拿到 `runAtEnd`

**等待条件的精确含义：**
> 所有**非 `runAtEnd`** 的查询中，没有任何一个处于 `queued` 或 `running` 状态。

这意味着：
1. 其他 `runAtEnd` 查询不构成阻塞（排除了 `runAtEnd: true` 的查询）
2. 只要有一个普通查询还在跑或在排队，当前 `runAtEnd` 查询就必须等
3. 失败（`failed`）或成功（`succeeded`）的普通查询不阻塞

#### 2.3.4 关键设计 — `return` 而非 `continue`

```typescript
if (pendingQueries.length) {
  logger.debug(`${query.id}: "Run at end query" waiting...`);
  return;  // ⚠️ 不是 continue！
}
```

**`return` 的全局影响：**
- 当第一个需要等待的 `runAtEnd` 查询出现时，**整个 `startReadyQueries()` 方法立即返回**
- 当前循环中排在它后面的所有 queued 查询（包括其他 `runAtEnd` 和普通查询）全部被跳过
- 这些被跳过的查询只能等下一次 `onQueryFinish()` 事件触发新一轮 `startReadyQueries()`

**"门闩效应"的工作原理：**
```
queuedQueries = [普通查询A, 普通查询B, runAtEnd查询C, 普通查询D, runAtEnd查询E]
                     ↑         ↑
               已执行   此处触发 return，D 和 E 都被跳过
```

#### 2.3.5 queueQueryExecution 的抖动重排

并发受限时，`queueQueryExecution()` 引入随机 jitter 打乱执行顺序（`QueryRunner.ts:637`）：

```typescript
public queueQueryExecution(
  query: QueryInterface,
  timeout: number = INITIAL_CONCURRENCY_TIMEOUT,
) {
  // Queue query randomly within the window [timeout, timeout*2) to reduce race conditions
  const jitter = Math.floor(Math.random() * timeout);
  this.setTimer(
    query.id,
    setTimeout(() => {
      this.executeQueryWhenReady(query, timeout);
    }, timeout + jitter),  // 实际延迟 = timeout + [0, timeout) 随机值
  );
}
```

**对执行顺序的影响：**
- 同一轮被并发限制阻塞的查询，**执行顺序完全随机**，与原队列顺序无关
- 这意味着：
  1. 普通查询之间的执行顺序在并发场景下没有保证
  2. 多个 `runAtEnd` 查询之间的执行顺序也没有保证
  3. 唯一确定的是：`runAtEnd` 查询一定在所有普通查询完成之后才开始排队

**执行顺序的真实保证（修正原结论）：**
- ✅ `runAtEnd` 查询**不会在普通查询完成前开始执行**
- ❌ ~~队列中靠前的 `runAtEnd` 优先执行~~ → **不保证**，取决于 jitter 随机值和并发释放时机
- ❌ ~~普通查询按数组顺序执行~~ → **不保证**，并发场景下 jitter 会重排

#### 2.3.6 缓存命中时的状态继承 — `runAtEnd` 查询会被"提前完成"吗？

`createNewQueryFromCached()` 的状态继承逻辑（`QueryModel.ts:315`）：

```typescript
export async function createNewQueryFromCached({
  existing,
  dependencies,
  runAtEnd,  // ⚠️ 新参数，不是从 existing 继承
}: {
  existing: QueryInterface;
  dependencies: string[];
  runAtEnd?: boolean;
}): Promise<QueryInterface> {
  const data: QueryInterface = {
    // ... 继承字段
    status: existing.status,       // ✅ 继承缓存查询的状态
    runAtEnd: runAtEnd,            // ⚠️ 使用新传入的标记，不继承
    // ...
  };
  // ...
}
```

**关键场景分析：**

| 场景 | existing.status | 新查询 runAtEnd | 新查询 status | 结果 |
|------|-----------------|-----------------|---------------|------|
| 1 | `succeeded` | `true` | `succeeded` | **直接成功，跳过等待检查** |
| 2 | `running` | `true` | `running` | 轮询等待，最终继承 succeeded/failed |
| 3 | `failed` | `true` | `failed` | 直接失败 |

**场景 1 的问题 — 真的是"提前完成"吗？**

`startQuery()` 中缓存命中的完整流程：
```typescript
// QueryRunner.ts:795
if (existing.status === "succeeded") {
  const copiedCachedDoc = await createNewQueryFromCached({
    existing: existing,
    dependencies: dependencies,
    runAtEnd: runAtEnd,  // 新查询的 runAtEnd 标记
  });
  return {
    name,
    query: copiedCachedDoc.id,
    status: copiedCachedDoc.status,  // = "succeeded"
  };
}
```

此时这个 `runAtEnd: true` 的查询：
1. **状态直接是 `succeeded`**，不会出现在 `queuedQueries` 中
2. **永远不会经过 `startReadyQueries()` 的 `runAtEnd` 等待检查**
3. 即使还有很多普通查询在跑，它也已经"完成"了

**这是设计缺陷还是预期行为？**

从代码意图看，这是**预期行为**，理由：
- `runAtEnd` 的语义是"在所有非 runAtEnd 查询**执行完成后再执行**"
- 如果结果已经通过缓存获得，就没有必要"执行"了，直接复用即可
- 但从语义一致性角度，这确实是一个灰色地带：`runAtEnd` 查询的**执行时序约束**被缓存绕过了

**三道防线的真实有效性（修正原结论）：**

| 防线 | 位置 | 有效性 | 说明 |
|-----|------|--------|------|
| 1 | `startQuery()` 创建时 | ❌ 部分失效 | `readyToRun = dependenciesComplete && !runAtEnd && !concurrencyLimitReached` 确保新建的 runAtEnd 查询不立即执行，但缓存命中路径绕过了这一点 |
| 2 | `startReadyQueries()` 等待检查 | ✅ 有效 | 只要状态是 `queued`，就会经过检查 |
| 3 | `queryMap` 实时读取 | ✅ 有效 | `runAtEnd` 标志从数据库实时读取 |

> **修正结论**：`runAtEnd` 查询**不会被提前执行**，但**可能被提前完成**（通过缓存复用）。这是"执行"与"完成"两个概念的区别——缓存复用跳过了执行，但状态是完成态。

#### 2.3.7 并发控制

```typescript
// back-end/src/queryRunners/QueryRunner.ts:905
private async concurrencyLimitReached(): Promise<boolean> {
  const limit = datasource.settings.maxConcurrentQueries;
  const running = await countRunningQueries(orgId, datasourceId);
  return running >= limit;
}
```
- 超出并发限制时，通过 `queueQueryExecution()` 进行指数退避重试（250ms → 500ms → ... → 4000ms 封顶）
- 使用 jitter 避免惊群效应

#### 2.3.8 查询缓存

- `startQuery()` 首先调用 `getRecentQuery()` 查找相同 SQL 的历史执行
- 缓存命中时直接复用结果，正在运行的查询通过轮询等待完成
- 缓存 TTL 可通过数据源设置 `queryCacheTTLMins` 配置

---

## 三、结果归一化：QueryResponse 与行数据处理

### 3.1 统一响应格式

所有数据源的查询结果归一为 `QueryResponse` 类型（`shared/types/integrations.d.ts:678`）：

```typescript
export type QueryResponse<Rows = Record<string, any>[]> = {
  rows: Rows;
  columns?: QueryResponseColumnData[];  // 列元数据（名称、类型）
  statistics?: QueryStatistics;         // 查询统计（扫描字节、耗时等）
};
```

### 3.2 驱动层归一化

每个驱动在 `runQuery` 中处理数据源特有的结果格式：

**Snowflake 示例**（`back-end/src/services/snowflake.ts:217`）：
```typescript
// Snowflake 返回的列名全大写，需转为小写以匹配其他数据源
const lowercase = res.rows.map((row) => {
  return Object.fromEntries(
    Object.entries(row).map(([k, v]) => [k.toLowerCase(), v]),
  ) as T;
});
return { rows: lowercase, columns: res.columns };
```

**BigQuery 示例**（`back-end/src/integrations/BigQuery.ts:94`）：
```typescript
// BigQuery 返回的时间类型需要特殊处理，列元数据映射到内部类型系统
const columns = queryResultsResponse?.schema?.fields?.map((field) => ({
  name: field.name,
  dataType: getFactTableTypeFromBigQueryType(field.type as BigQueryDataType),
}));
```

### 3.3 业务层归一化

`SqlIntegration` 中各 `run*Query` 方法进一步将原始行数据转为业务对象：

**实验指标查询归一化**（`back-end/src/integrations/SqlIntegration.ts:511`）：
```typescript
async runExperimentMetricQuery(query, setExternalId, metadata) {
  const { rows, statistics } = await this.runQuery(...);
  
  return {
    rows: rows.map((row) => ({
      variation: row.variation ?? "",
      users: parseIntWithDefault(row.users, 0),
      main_sum: parseFloat(row.main_sum as string) || 0,
      main_sum_squares: parseFloat(row.main_sum_squares as string) || 0,
      // ... 20+ 个字段的类型转换与默认值处理
    })),
    statistics,
  };
}
```

**归一化内容包括：**
- 类型安全：`parseFloat`、`parseIntWithDefault` 确保数值字段不会是 `null` 或 `NaN`
- 默认值：缺失字段填充合理默认（如 `variation ?? ""`）
- 字段提取：维度字段通过前缀 `dim_` 识别并提取
- 特殊结构：分位数、CUPED 协变量、HLL 草图等高级特性的字段映射

### 3.4 查询状态机

查询生命周期由 `QueryStatus` 状态机管理：

```
queued → running → succeeded
           ↓
         failed
```

**状态聚合逻辑**（`QueryRunner.getOverallQueryStatus()`，`back-end/src/queryRunners/QueryRunner.ts:923`）：

```typescript
const failed = this.model.queries.filter(q => q.status === "failed");
const running = this.model.queries.filter(q => q.status === "running");
const queued = this.model.queries.filter(q => q.status === "queued");
const total = this.model.queries.length;

if (failed.length >= total / 2) return "failed";
if (queued.length + running.length > 0) return "running";
if (failed.length > 0) return "partially-succeeded";
return "succeeded";
```

**判定优先级（从高到低）：**

| 优先级 | 条件 | 返回状态 | 说明 |
|-------|------|---------|------|
| 1 | `failed.length >= total / 2` | `failed` | **半数（含）以上失败即整体失败**。注意用 `>=` 而非 `>`，total=3 时 2 个失败即触发，total=4 时 2 个失败即触发 |
| 2 | `queued.length + running.length > 0` | `running` | 只要还有查询在跑或排队中，整体就是 running |
| 3 | `failed.length > 0` | `partially-succeeded` | **三条件同时成立**：有失败、失败数未达半数、没有 pending 查询 |
| 4 | 以上均不满足 | `succeeded` | 全部查询成功，无失败、无 pending |

**易忽略的细节：**
- `partially-succeeded` 的成立依赖优先级。如果失败数刚好 ≥ total/2，会在第 1 步直接返回 `failed`，不会出现 `partially-succeeded`
- 如果有失败但同时还有 queued/running 查询，会在第 2 步返回 `running`，需要等所有查询收尾后才会重新判定为 `partially-succeeded`
- `refreshQueryStatuses()` 中只有 `newStatus === "succeeded"` 或 `"partially-succeeded"` 才会触发 `runAnalysis()`，`failed` 状态不会跑分析

**状态流转中的副作用**（`refreshQueryStatuses()`，`QueryRunner.ts:516`）：
- `running → failed`：记录错误信息，单查询场景会使用查询自身的错误消息
- `running → succeeded` 或 `running → partially-succeeded`：触发 `runAnalysis()` 执行统计分析
- `running → running`：不触发任何副作用，等待下次查询完成事件

---

## 四、完整调用链路

以一次实验分析为例，端到端调用链：

```
API 请求 (POST /experiments/:id/analyze)
    ↓
ExperimentSnapshotController
    ↓
ExperimentResultsQueryRunner.startAnalysis(params)
    ├─ startQueries(params) 【编译阶段】
    │   └─ startExperimentResultQueries()
    │       ├─ 解析实验设置、指标、维度
    │       ├─ 为每个指标调用 integration.getExperimentMetricQuery()
    │       │   └─ buildExperimentMetricQuerySql(dialect, ...)
    │       │       └─ 方言方法 → 生成适配 SQL
    │       └─ startQuery({ name, query, dependencies, run })
    │           ├─ 检查缓存 → getRecentQuery()
    │           ├─ 无缓存 → createNewQuery() 写入 Mongo
    │           └─ 无依赖且未超限 → executeQuery()
    ├─ refreshQueryStatuses() 【轮询阶段】
    │   ├─ updateQueryPointers() → 从 Mongo 拉取最新状态
    │   ├─ 检测状态变化 → 触发 runAnalysis()
    │   └─ 全部完成 → updateModel() 持久化结果
    └─ runAnalysis(queryMap) 【分析阶段】
        └─ analyzeExperimentResults() → 统计检验、p值计算等
```

---

## 五、关键设计权衡

| 设计决策 | 优点 | 缺点 |
|---------|------|------|
| **胖接口** `SourceIntegrationInterface` | 上层代码无需判断数据源类型，直接调用统一方法 | 接口过大（60+ 方法），新增驱动需实现大量空方法 |
| **简单工厂注册** | 实现直接，无反射开销 | 新增数据源需修改 `datasource.ts`，违反开闭原则 |
| **查询级缓存** | 相同 SQL 重复执行时大幅提升性能 | 缓存键仅基于 SQL 文本，未考虑数据时效性 |
| **客户端并发控制** | 减轻数据仓库压力，避免连接池耗尽 | 状态存储在 Mongo，多实例部署时存在竞态窗口 |
| **SQL 字符串拼接** | 方言适配灵活，调试友好 | 存在 SQL 注入风险（但参数通过方言转义处理） |

---

## 六、扩展新数据源指南

如需接入新的数据仓库，需完成以下步骤：

1. **添加类型定义**：在 `shared/types/integrations/` 下新增连接参数类型
2. **实现驱动类**：继承 `SqlIntegration`，实现以下核心方法：
   - `setParams()` - 解密并设置连接参数
   - `runQuery()` - 执行 SQL 并返回归一化结果
   - `getSqlDialect()` - 返回方言实例
   - `getSensitiveParamKeys()` - 声明敏感字段
   - `cancelQuery()` - 取消运行中的查询（可选但推荐）
3. **实现方言**：在 `integrations/dialects/` 下新增方言对象，覆盖 `baseDialect` 中不兼容的方法
4. **注册工厂**：在 `datasource.ts` 的 `getIntegrationObj()` switch 中添加新 case
5. **前端适配**：在前端数据源配置页面添加新类型的表单

