# 数据仓库分析查询执行生命周期 - 代码实现分析

本文从代码实现角度深入剖析 GrowthBook 中数据仓库查询执行的完整生命周期，涵盖 **连接池建立**、**查询编排** 与 **结果落库** 三个核心阶段的协同机制。

---

## 一、连接池建立阶段

### 1.1 核心架构设计

GrowthBook 的连接管理策略（代码可验证行为）：
- `runSnowflakeQuery` 每次调用独立创建连接，执行完毕后在 `finally` 块中销毁
- `BigQuery.getClient()` 每次调用返回新的 `bq.BigQuery()` 实例
- 代码库中不存在连接池复用逻辑（无 `pool`、`reuse`、`cache` 等连接复用代码）

**类继承关系**：
```
SqlIntegration (抽象基类)
├── Snowflake
├── BigQuery
├── Postgres
├── Redshift
├── ClickHouse
├── Databricks
└── ...
```

### 1.2 连接建立核心流程

**代码位置**：`packages/back-end/src/integrations/SqlIntegration.ts:178-212`

```typescript
export default abstract class SqlIntegration {
  datasource: DataSourceInterface;
  context: ReqContext;
  params: any;

  abstract setParams(encryptedParams: string): void;
  abstract runQuery(sql: string, ...): Promise<QueryResponse>;
  
  constructor(context: ReqContextClass, datasource: DataSourceInterface) {
    this.wrapRunQuery();               // 包装查询方法，注入元数据
    this.datasource = datasource;
    this.context = context;
    this.decryptionError = false;
    try {
      this.setParams(datasource.params);  // 子类解密连接参数
    } catch (e) {
      this.params = {};
      this.decryptionError = true;
    }
  }
}
```

### 1.3 Snowflake 连接实现细节

**代码位置**：`packages/back-end/src/services/snowflake.ts:34-227`

**连接构建 (`buildSnowflakeConnection`)**：
```typescript
function buildSnowflakeConnection(
  conn: SnowflakeConnectionParams,
  queryMetadata?: QueryMetadata,
): Connection {
  // 支持密码认证和密钥对认证两种方式
  let authenticationDetails;
  if (conn.authMethod === "key-pair") {
    authenticationDetails = {
      authenticator: "SNOWFLAKE_JWT",
      privateKey: createPrivateKey({...})
        .export({ format: "pem", type: "pkcs8" })
        .toString(),
    };
  } else {
    authenticationDetails = { password: conn.password };
  }

  return createConnection({
    account,
    username: conn.username,
    ...authenticationDetails,
    database: conn.database,
    schema: conn.schema,
    warehouse: conn.warehouse,
    role: conn.role,
    queryTag: getQueryTagString(queryMetadata ?? {}, 2000),  // 查询标签用于审计
  });
}
```

**连接生命周期管理**：
```typescript
export async function runSnowflakeQuery<T>(...): Promise<QueryResponse<T[]>> {
  const connection = buildSnowflakeConnection(conn, queryMetadata);
  try {
    // 测试查询超时30s，常规查询超时10分钟
    const connectionTimeout = sql === TEST_QUERY_SQL ? 30000 : 600000;
    await connectSnowflake(connection, connectionTimeout);

    // 异步提交获取 queryId（用于后续取消）
    const queryId = await new Promise<string>((resolve, reject) => {
      connection.execute({
        sqlText: sql,
        asyncExec: true,  // 立即返回，不阻塞等待结果
        complete: (err, stmt) => resolve(stmt.getQueryId()),
      });
    });

    // 持久化外部ID，支持中途取消
    if (setExternalId) await setExternalId(queryId);

    // 轮询等待结果
    const res = await new Promise<{rows: T[]; columns: ...}>(
      (resolve, reject) => {
        connection.getResultsFromQueryId({
          queryId,
          complete: (err, stmt, rows) => resolve({rows, columns: ...}),
        });
      }
    );

    return { rows: lowercase, columns: res.columns };
  } finally {
    await destroySnowflakeConnection(connection);  // 确保资源释放
  }
}
```

### 1.4 BigQuery 连接实现差异

**代码位置**：`packages/back-end/src/integrations/BigQuery.ts:47-140`

> **⚠️ 关键修正**：之前的"客户端池"描述存在偏差。`getClient()` **每次调用都会新建 `bq.BigQuery()` 实例**，并未在应用层实现客户端复用。真正的连接复用发生在 `@google-cloud/bigquery` 客户端内部（通过 HTTP keep-alive）。

```typescript
private getClient() {
  // IS_CLOUD 判断决定凭证获取方式，与客户端复用无关
  if (!IS_CLOUD && this.params.authType === "auto") {
    // 从环境变量或元数据服务器自动获取凭证
    return new bq.BigQuery();
  }
  // 使用传入的密钥凭证
  return new bq.BigQuery({
    projectId: this.params.projectId,
    credentials: {
      client_email: this.params.clientEmail,
      private_key: this.params.privateKey,
    },
  });
}

async runQuery(sql: string, setExternalId?: ExternalIdCallback) {
  const client = this.getClient();  // 每次调用新建客户端实例
  const [job] = await client.createQueryJob({
    labels: { integration: "growthbook" },
    query: sql,
    useLegacySql: false,
  });

  if (setExternalId && job.id) await setExternalId(job.id);
  const [rows, _, queryResultsResponse] = await job.getQueryResults();
  const [metadata] = await job.getMetadata();

  // 收集执行统计信息
  const statistics = {
    executionDurationMs: Number(metadata?.statistics?.finalExecutionDurationMs),
    totalSlotMs: Number(metadata?.statistics?.totalSlotMs),
    bytesProcessed: Number(metadata?.statistics?.totalBytesProcessed),
    bytesBilled: Number(metadata?.statistics?.query?.totalBytesBilled),
    warehouseCachedResult: metadata?.statistics?.query?.cacheHit,
  };

  return { rows, columns, statistics };
}
```

**客户端生命周期真实模型**：
| 调用点 | 行为 | 连接复用层次 |
|-------|-----|------------|
| `getClient()` | 每次 `new bq.BigQuery()` | ❌ 应用层不复用 |
| `@google-cloud/bigquery` v8.1.1 | 内部使用 `gaxios` + HTTP keep-alive | ✅ 底层连接池 |
| `cancelQuery()` | 独立调用 `getClient()` | ❌ 与 `runQuery` 不共享客户端 |

### 1.5 取消路径中 externalId 回溯关联与状态收敛

**代码位置**：`packages/back-end/src/queryRunners/QueryRunner.ts:558-635`

> **⚠️ 关键修正**：取消流程对 `running` 和 `queued` 两种状态的处理有本质差异。`running` 查询需要真正终止数据仓库侧的 Job，而 `queued` 查询尚未提交，只需清理本地定时器即可。

```typescript
public async cancelQueries(): Promise<void> {
  // 只要有 running 或 queued 就进入取消流程
  if (
    this.model.queries.some(
      (q) => q.status === "running" || q.status === "queued",
    )
  ) {
    // ========== 第一步：仅处理 running 查询 ==========
    // ⚠️ queued 查询被过滤掉，因为它们没有 externalId，也没有提交到数据仓库
    const runningIds = this.model.queries
      .filter((q) => q.status === "running")
      .map((q) => q.query);

    if (runningIds.length) {
      // includeChunkedResults=false：只加载元数据，不加载分块结果
      const queryDocs = await getQueriesByIds(
        this.context,
        runningIds,
        false,
      );

      // externalId 回溯逻辑（仅适用于 running 查询）
      // Some "running" docs are pointers to a previously-started in-flight
      // query (see `createNewQueryFromCached`). Those copies share the
      // upstream datasource job with the original via `cachedQueryUsed`
      // and never have their own `externalId` set — only the original
      // doc does, populated by the integration's `runQuery`/poller.
      // Chase one hop to find the real id so we can actually cancel the
      // upstream job. `cachedQueryUsed` always points at the original
      // (not a copy of a copy), so a single lookup is sufficient.
      const cachedSourceIds = Array.from(
        new Set(
          queryDocs
            .map((q) => q.cachedQueryUsed)
            .filter((id): id is string => Boolean(id)),
        ),
      );
      const cachedSourceDocs = cachedSourceIds.length
        ? await getQueriesByIds(this.context, cachedSourceIds, false)
        : [];
      const cachedSourceById = new Map(
        cachedSourceDocs.map((q) => [q.id, q]),
      );

      // 收集所有需要取消的 externalId（去重后）
      const externalIds = queryDocs
        .map((q) => {
          if (q.externalId) return q.externalId;
          if (q.cachedQueryUsed) {
            return cachedSourceById.get(q.cachedQueryUsed)?.externalId;
          }
          return undefined;
        })
        .filter((id): id is string => Boolean(id));

      // 真正调用数据仓库的取消接口（最多5并发）
      if (externalIds.length) {
        await promiseAllChunks(
          externalIds.map((id) => {
            return async () => {
              if (!id || !this.integration.cancelQuery) return;
              try {
                await this.integration.cancelQuery(id);
              } catch (e) {
                logger.debug(`Failed to cancel query - ${e.message}`);
              }
            };
          }),
          5,
        );
      }
    }

    // ========== 第二步：统一状态收敛（running + queued 一起处理）==========
    // ⚠️ 关键差异：
    // - running 查询：上面已调用 cancelQuery() 终止数据仓库侧 Job
    // - queued 查询：没有调用 cancelQuery()，因为它们尚未提交
    // - 两者的共同处理：清理本地定时器 + 清空 queries 数组 + 标记 failed

    // 清理所有本地定时器（包括 queued 查询的指数退避重试定时器）
    // pendingTimers 中保存了两类定时器：
    //   1. 缓存查询的 3 秒轮询定时器
    //   2. queued 查询的指数退避重试定时器
    this.clearAllTimers();

    // 统一标记：清空 queries 数组，整体状态设为 failed
    // ⚠️ 注意：这里没有逐个更新 queued 查询的状态，也没有逐个标记 running 查询为 failed
    // 而是直接清空 queries 数组，依赖上层模型状态机完成收敛
    const newModel = await this.updateModel({
      queries: [],
      status: "failed",
      error: "",
    });
    this.model = newModel;

    this.setStatus("finished", "Queries cancelled by user");
  }
}
```

**文档关联结构**：
```
查询副本A（缓存复用）          原始查询B（真正执行）
├── id: qry_abc               ├── id: qry_xyz
├── cachedQueryUsed: qry_xyz  ├── externalId: 12345  ← 数据仓库真正的Job ID
├── externalId: (空)          ├── status: running
└── status: running           └── ...
```

**回溯保证**：代码注释明确 `cachedQueryUsed` 永远指向原始查询，不会出现"副本的副本"（A→B→C）的多级链式引用，因此一次查找即可。

**running 与 queued 查询的状态收敛差异表**：

| 处理步骤 | running 查询 | queued 查询 | 代码证据 |
|---------|------------|------------|---------|
| 过滤出 runningIds | ✅ 包含 | ❌ 排除 | `QueryRunner.ts:565-567` |
| 回溯 externalId | ✅ 需要 | ❌ 不需要 | `QueryRunner.ts:569-606` |
| 调用 `integration.cancelQuery()` | ✅ 调用 | ❌ 不调用 | `QueryRunner.ts:608-622` |
| 清理本地定时器 | ✅ 清理 | ✅ 清理 | `QueryRunner.ts:625` |
| 单独更新查询文档状态 | ❌ 不更新 | ❌ 不更新 | 代码中无对应逻辑 |
| 整体模型 `queries` 置空 | ✅ 置空 | ✅ 置空 | `QueryRunner.ts:627` |
| 整体模型 `status` 设为 failed | ✅ 标记 | ✅ 标记 | `QueryRunner.ts:628` |

**代码可验证的状态收敛行为**：
1. `integration.cancelQuery()` 异常被独立 try-catch 包裹，仅打 `logger.debug` 日志，不抛出异常（`QueryRunner.ts:612-617`）
2. `updateQueryIfRunning()` 的更新条件包含 `status: "running"`，非 running 状态的文档不会被更新（`models/QueryModel.ts:197-204`）
3. `cancelQueries()` 最终将 `model.queries` 设为 `[]`，`model.status` 设为 `"failed"`（`QueryRunner.ts:626-630`）

### 1.6 连接层设计要点（仅保留代码可验证事实）

| 设计决策 | 实现方式 | 代码位置 |
|---------|---------|---------|
| **参数解密** | `setParams()` 中调用 `decryptDataSourceParams` | `integrations/Snowflake.ts:17-19` |
| **查询审计** | `wrapRunQuery()` 向 `metadata` 注入 `userId`、`userName`、`additionalMetadata` | `integrations/SqlIntegration.ts:214-232` |
| **超时控制** | `connectionTimeout = sql === TEST_QUERY_SQL ? 30000 : 600000` | `services/snowflake.ts:95-96` |
| **资源清理** | `try/finally` 包裹 `destroySnowflakeConnection()` | `services/snowflake.ts:121-123` |
| **取消支持** | `setExternalId` 回调持久化 `externalId`，取消时通过 `cachedQueryUsed` 回溯 | `QueryRunner.ts:713-715`, `QueryRunner.ts:584-606` |
| **BigQuery客户端** | `getClient()` 每次 `return new bq.BigQuery()` | `integrations/BigQuery.ts:47-60` |

---

## 二、查询编排阶段

### 2.1 QueryRunner 核心编排器

**代码位置**：`packages/back-end/src/queryRunners/QueryRunner.ts:107-984`

`QueryRunner` 是一个抽象泛型类，负责：
- DAG（有向无环图）依赖管理
- 查询并发控制与队列调度
- 缓存复用机制
- 状态机流转与心跳检测

```typescript
export abstract class QueryRunner<Model, Params, Result> {
  public model: Model;
  public integration: SourceIntegrationInterface;
  public status: RunnerStatus = "pending";
  public result: Result | null = null;
  
  private runCallbacks: Record<string, QueryCallbacks> = {};
  private finishedQueryMapCache: QueryMap = new Map();
  private pendingTimers: Record<string, NodeJS.Timeout> = {};
}
```

### 2.2 查询执行入口：`startAnalysis`

**代码位置**：`packages/back-end/src/queryRunners/QueryRunner.ts:262-335`

> **⚠️ 关键修正**：`startAnalysis` 末尾的 `onQueryFinish()` 不是"启动状态刷新定时器"那么简单。它的核心设计动机是 **规避 DAG 持久化竞态**——解决"第一个查询完成太快，DAG 还没写完就被刷新"的死锁问题。

```typescript
public async startAnalysis(params: Params): Promise<Model> {
  // 1. 生成查询DAG（由子类实现）
  // ⚠️ 注意：startQueries 内部会对无依赖的查询立即 fire-and-forget 执行
  // 见 startQuery 中的 readyToRun 分支
  const queries = await this.startQueries(params);
  this.model.queries = queries;

  if (queries.length === 0) {
    // 无查询，直接失败
    return this.updateModel({ status: "failed", error: "No queries..." });
  }

  // 2. 检查缓存命中情况
  const queryStatus = this.getOverallQueryStatus();
  if (queryStatus === "succeeded") {
    // 全部缓存命中，直接运行分析
    const queryMap = await this.getQueryMap(queries);
    result = await this.runAnalysis(queryMap);
  }

  // 3. 持久化完整 DAG 到数据库
  // ⚠️ 注意：这是第一次将完整的查询列表写入持久化存储
  // 如果第一个查询在这之前就完成了，它的 onQueryFinish 会读到不完整的 DAG
  const newModel = await this.updateModel({
    status: queryStatus,
    queries,
    runStarted: new Date(),
    result,
    error,
  });

  if (error || result) {
    this.setStatus("finished", error, result);
  } else {
    this.setStatus("running");
    // ===================================================================
    // ⚠️ 核心设计：确保至少一次刷新发生在 DAG 持久化之后
    // ===================================================================
    // This closes a race: queries with no dependencies are executed
    // fire-and-forget inside startQueries() (see startQuery's readyToRun
    // branch). If one of them is very fast (e.g. a DROP TABLE during a
    // Full Refresh), it can finish — and its onQueryFinish 1-second timer
    // can fire — while startQueries() is still generating SQL for the
    // remaining queries and before updateModel() above has written them.
    // That early timer reloads the model, sees no running/queued queries,
    // and gives up. Once startQueries() finally returns, nothing re-arms
    // the timer, so every other query in the DAG stays "queued" forever.
    //
    // Calling onQueryFinish() here guarantees at least one refresh happens
    // after the DAG is visible in the persisted model. If the timer from
    // the early-finishing query is still pending this is a no-op; if it
    // already fired and found nothing, this re-arms it with the real DAG.
    // ===================================================================
    this.onQueryFinish();
  }

  return newModel;
}
```

#### 1秒延迟的竞态场景还原

**时间线对比（竞态发生时）**：
```
T0  startAnalysis() 开始
T1   └→ startQueries() 开始生成 DAG
T2     ├→ 生成查询 A（DROP TABLE，无依赖）
T3     ├→ startQuery(A) → readyToRun=true → 立即 executeQuery(A) [fire-and-forget]
T4     │    └→ Snowflake 异步提交非常快，10ms 就完成了
T5     │         └→ onQueryFinish(A) → 设置 1秒定时器 (timer A)
T6     ├→ 继续生成查询 B、C、D...（SQL 生成需要 1.5秒）
T7 timer A 触发（T5 + 1s）
T8   ├→ getLatestModel() → 此时 updateModel 还没执行，queries 为空
T9   ├→ refreshQueryStatuses() → 看到没有 running/queued 查询，直接返回
T10    └→ (静默失败，日志中只有 debug 级别)
T11  startQueries() 返回全部 queries
T12  updateModel() 持久化完整 DAG
T13  (没有其他 onQueryFinish 调用，DAG 中的 B、C、D 永远卡在 queued)
```

**保护措施**：
1. `onQueryFinish()` 内部有 `if (!this.timer)` 去重判断，防止重复设置
2. `refreshQueryStatuses()` 中如果发现非终端状态但无活跃查询，会记录 `warn` 级别日志（原来是 debug）
3. `startAnalysis` 末尾的 `onQueryFinish()` 兜底，确保至少有一次刷新看到完整 DAG

#### `onQueryFinish` 实现细节

**代码位置**：`packages/back-end/src/queryRunners/QueryRunner.ts:191-256`

```typescript
async onQueryFinish() {
  if (!this.timer) {  // 去重：已有定时器在等待则不重复设置
    logger.debug(
      "Query finished for " + this.model.id + " runner, refreshing in 1 second",
    );
    this.timer = setTimeout(async () => {
      this.timer = null;
      // 独立 try-catch：getLatestModel 失败（如并发取消导致快照被删）不影响其他
      let latest: Model;
      try {
        latest = await this.getLatestModel();
      } catch (e) {
        logger.debug(`Skipping refresh for ${this.model.id}: ${e.message}`);
        return;
      }
      this.model = latest;
      try {
        const queryMap = await this.refreshQueryStatuses();
        await this.startReadyQueries(queryMap);
      } catch (e) {
        // 刷新失败，标记整体为 failed
      }
    }, 1000);  // 固定 1 秒延迟
  }
}
```

**1秒延迟的代码可验证行为**：
| 行为 | 代码证据 |
|-----|---------|
| 固定 1000ms 延迟 | `setTimeout(..., 1000)`（`QueryRunner.ts:198`, `QueryRunner.ts:248`） |
| 定时器去重 | `if (!this.timer)` 检查（`QueryRunner.ts:192`） |
| 竞态防护注释 | 代码注释明确说明 `startAnalysis()` 末尾的调用是为了防止 DAG 持久化竞态（`QueryRunner.ts:314-331` 注释） |

### 2.3 单查询注册：`startQuery`

**代码位置**：`packages/back-end/src/queryRunners/QueryRunner.ts:765-902`

```typescript
public async startQuery(params: StartQueryParams): Promise<QueryPointer> {
  const { name, query, dependencies, runAtEnd, run, queryType } = params;

  // ========== 缓存复用逻辑 ==========
  if (this.useCache) {
    const cacheTTLMins = parseOptionalInt(
      this.integration.datasource.settings.queryCacheTTLMins
    );
    const existing = await getRecentQuery(
      this.integration.context.org.id,
      this.integration.datasource.id,
      query,
      cacheTTLMins,
    );

    if (existing) {
      if (existing.status === "running") {
        // 正在运行，轮询等待
        const check = () => {
          getQueriesByIds(this.context, [existing.id], false)
            .then(async (queries) => {
              if (queries[0].status === "failed" || 
                  queries[0].status === "succeeded") {
                this.onQueryFinish();
              } else {
                this.setTimer(existing.id, setTimeout(check, 3000));
              }
            });
        };
        this.setTimer(existing.id, setTimeout(check, 3000));
      }

      // 创建缓存副本，不重复执行
      const copiedCachedDoc = await createNewQueryFromCached({
        existing,
        dependencies,
        runAtEnd,
      });
      return { name, query: copiedCachedDoc.id, status: copiedCachedDoc.status };
    }
  }

  // ========== 新建查询 ==========
  const concurrencyLimitReached = await this.concurrencyLimitReached();
  const dependenciesComplete = dependencies.length === 0;
  const readyToRun = dependenciesComplete && !runAtEnd && !concurrencyLimitReached;

  const doc = await createNewQuery({
    query,
    queryType,
    datasource: this.integration.datasource.id,
    organization: this.integration.context.org.id,
    dependencies,
    running: readyToRun,    // 立即可运行则标记为 running
    runAtEnd,
  });

  // 保存回调以便后续执行
  this.runCallbacks[doc.id] = { run, process, onFailure, onSuccess };

  if (readyToRun) {
    this.executeQuery(doc, { run, process, onFailure, onSuccess });
  } else if (dependenciesComplete && !runAtEnd) {
    this.queueQueryExecution(doc);  // 并发限制，排队等待
  }

  return { name, query: doc.id, status: doc.status };
}
```

### 2.4 查询 DAG 生成示例

**代码位置**：`packages/back-end/src/queryRunners/ExperimentResultsQueryRunner.ts:87-357`

实验结果查询的 DAG 结构：
```
unitQuery (单位表创建)
    │
    ├→ metricQuery_1 (指标1查询)
    ├→ metricQuery_2 (指标2查询)
    ├→ ...
    ├→ groupQuery_1 (多指标组查询)
    ├→ trafficQuery (流量健康检查)
    │
    └→ dropUnitsTable (最后删除临时表, runAtEnd=true)
```

```typescript
// 单位表查询 - 无依赖
unitQuery = await startQuery({
  name: queryParentId,
  query: integration.getExperimentUnitsTableQuery(unitQueryParams),
  dependencies: [],
  run: (query, setExternalId, queryMetadata) =>
    integration.runExperimentUnitsQuery(...),
  queryType: "experimentUnits",
});

// 指标查询 - 依赖单位表
for (const m of legacyMetricSingles) {
  queries.push(await startQuery({
    name: m.id,
    query: integration.getExperimentMetricQuery(queryParams),
    dependencies: unitQuery ? [unitQuery.query] : [],  // 依赖关系
    run: (query, setExternalId, queryMetadata) =>
      integration.runExperimentMetricQuery(...),
    queryType: "experimentMetric",
  }));
}

// 删除临时表 - 标记 runAtEnd，等所有其他查询完成后执行
if (useUnitsTable && dropUnitsTable) {
  queries.push(await startQuery({
    name: `drop_${queryParentId}`,
    query: integration.getDropUnitsTableQuery(...),
    dependencies: [],
    runAtEnd: true,  // 关键：最后执行
    run: (query, setExternalId, queryMetadata) =>
      integration.runDropTableQuery(...),
    queryType: "experimentDropUnitsTable",
  }));
}
```

### 2.5 依赖调度与并发控制

**代码位置**：`packages/back-end/src/queryRunners/QueryRunner.ts:376-467`

```typescript
public async startReadyQueries(queryMap: QueryMap): Promise<void> {
  const queuedQueries = Array.from(queryMap.values())
    .filter(q => q.status === "queued");

  for (const query of queuedQueries) {
    // ========== 依赖检查 ==========
    const dependencyIds: string[] = query.dependencies ?? [];
    const failedDependencies: QueryPointer[] = [];
    const succeededDependencies: QueryPointer[] = [];
    const pendingDependencies: QueryPointer[] = [];

    dependencyIds.forEach((dependencyId) => {
      const dependencyQuery = this.model.queries.find(
        q => q.query === dependencyId
      );
      if (dependencyQuery.status === "succeeded") {
        succeededDependencies.push(dependencyQuery);
      } else if (dependencyQuery.status === "failed") {
        failedDependencies.push(dependencyQuery);
      } else {
        pendingDependencies.push(dependencyQuery);
      }
    });

    // 有依赖失败，级联失败
    if (failedDependencies.length) {
      await updateQuery(this.context, query, {
        finishedAt: new Date(),
        status: "failed",
        error: `Dependencies failed: ${failedDependencies.map(q => q.query)}`,
      });
      this.onQueryFinish();
      continue;
    }

    // 有依赖未完成，继续等待
    if (pendingDependencies.length) continue;

    // ========== runAtEnd 检查 ==========
    if (query.runAtEnd) {
      const pendingQueries = this.model.queries.filter(
        q => !queryMap.get(q.name)?.runAtEnd &&
             (q.status === "queued" || q.status === "running")
      );
      if (pendingQueries.length) return;  // 还有其他查询在运行
    }

    // ========== 并发控制 ==========
    if (await this.concurrencyLimitReached()) {
      this.queueQueryExecution(query);
    } else {
      await this.executeQuery(query, runCallbacks);
    }
  }
}
```

**并发控制算法**：
```typescript
private async concurrencyLimitReached(): Promise<boolean> {
  if (!this.integration.datasource.settings.maxConcurrentQueries)
    return false;
  
  const numericConcurrencyLimit = parseIntWithDefault(
    this.integration.datasource.settings.maxConcurrentQueries, NaN
  );
  if (isNaN(numericConcurrencyLimit) || numericConcurrencyLimit === 0)
    return false;

  // 查询当前组织、当前数据源的运行中查询数
  const numRunningQueries = await countRunningQueries(
    this.integration.context.org.id,
    this.integration.datasource.id,
  );
  return numRunningQueries >= numericConcurrencyLimit;
}

// 指数退避队列
public queueQueryExecution(
  query: QueryInterface,
  timeout: number = INITIAL_CONCURRENCY_TIMEOUT,  // 250ms
) {
  const jitter = Math.floor(Math.random() * timeout);
  this.setTimer(
    query.id,
    setTimeout(() => {
      this.executeQueryWhenReady(query, timeout);
    }, timeout + jitter),
  );
}

public async executeQueryWhenReady(
  doc: QueryInterface,
  currentTimeout: number = 250,
): Promise<void> {
  const concurrencyLimitReached = await this.concurrencyLimitReached();
  if (concurrencyLimitReached) {
    // 超时加倍，最大 4000ms
    const nextTimeout = Math.min(currentTimeout * 2, MAX_CONCURRENCY_TIMEOUT);
    this.queueQueryExecution(doc, nextTimeout);
    return;
  }
  this.clearTimer(doc.id);
  return this.executeQuery(doc, runCallbacks);
}
```

### 2.6 查询执行与心跳机制

**代码位置**：`packages/back-end/src/queryRunners/QueryRunner.ts:682-763`

```typescript
public async executeQuery(doc: QueryInterface, callbacks): Promise<void> {
  // ========== 心跳定时器（30秒一次）==========
  const timer = setInterval(() => {
    updateQuery(this.context, doc, { heartbeat: new Date() }).catch(...);
  }, 30000);

  // 标记为运行中
  if (doc.status !== "running") {
    await updateQuery(this.context, doc, {
      startedAt: new Date(),
      status: "running",
      heartbeat: new Date(),
    });
  }

  // externalId 回调，支持查询取消
  const setExternalId = async (id: string) => {
    await updateQuery(this.context, doc, { externalId: id });
  };

  // ========== 异步执行，不阻塞 ==========
  run(doc.query, setExternalId, { queryType: doc.queryType })
    .then(async ({ rows, statistics }) => {
      clearInterval(timer);
      await updateQuery(this.context, doc, {
        finishedAt: new Date(),
        status: "succeeded",
        rawResult: rows,
        result: process ? process(rows) : rows,
        statistics,
      });
      if (onSuccess) await onSuccess(rows);
      this.onQueryFinish();  // 触发后续依赖查询
    })
    .catch(async (e) => {
      clearInterval(timer);
      const updated = await updateQueryIfRunning(this.context, doc, {
        finishedAt: new Date(),
        status: "failed",
        error: e.message,
      });
      onFailure();
      this.onQueryFinish();
    });
}
```

### 2.7 状态刷新与孤儿查询检测

**状态聚合算法** (`getOverallQueryStatus`)：
```typescript
private getOverallQueryStatus(): QueryStatus {
  const failedQueries = this.model.queries.filter(q => q.status === "failed");
  const runningQueries = this.model.queries.filter(q => q.status === "running");
  const queuedQueries = this.model.queries.filter(q => q.status === "queued");
  const totalQueries = this.model.queries.length;

  if (failedQueries.length >= totalQueries / 2) return "failed";      // 过半失败
  if (queuedQueries.length + runningQueries.length > 0) return "running";  // 还有在跑
  if (failedQueries.length > 0) return "partially-succeeded";       // 部分失败
  return "succeeded";  // 全部成功
}
```

**孤儿查询清理** (`getStaleQueries`)：
**代码位置**：`packages/back-end/src/models/QueryModel.ts:236-273`

```typescript
export async function getStaleQueries() {
  // 心跳每30秒一次，超过70秒未更新视为失联
  const lastHeartbeat = new Date();
  lastHeartbeat.setSeconds(lastHeartbeat.getSeconds() - 70);

  const query = {
    status: "running",
    heartbeat: { $lt: lastHeartbeat },
  };

  const docs = await QueryModel.find(query, { _id: 1, id: 1, organization: 1 })
    .limit(20);

  await QueryModel.updateMany(
    { ...query, _id: { $in: docs.map(d => d._id) } },
    {
      $set: {
        status: "failed",
        error: "Query execution was interupted. Please try again.",
      },
    },
  );

  return docs.map(doc => ({ id: doc.id, organization: doc.organization }));
}
```

---

## 三、结果落库阶段

### 3.1 数据存储分层设计

采用 **主文档 + 分块扩展** 的两级存储方案，解决 MongoDB 单文档 16MB 限制问题。

```
Query 主文档 (queries 集合)
├── 元数据字段 (id, status, timestamps, error, statistics...)
├── hasChunkedResults: Boolean  # 是否有分块数据
└── result / rawResult          # 小结果直接内嵌

SqlResultChunk 分块文档 (sqlresultchunks 集合)
├── queryId: String             # 关联主查询
├── chunkNumber: Number         # 分块序号
├── numRows: Number             # 本块行数
└── data: { [column]: value[] } # 列式存储
```

### 3.2 主查询文档结构

**代码位置**：`packages/back-end/src/models/QueryModel.ts:20-57`

```typescript
const querySchema = new mongoose.Schema({
  id: { type: String, unique: true },
  displayTitle: String,
  organization: { type: String, index: true },
  datasource: String,
  language: String,
  query: String,                    // SQL 语句
  status: { type: String, index: true },  // queued / running / succeeded / failed
  queryType: String,                // experimentMetric / experimentUnits 等
  createdAt: Date,
  startedAt: Date,
  finishedAt: Date,
  heartbeat: Date,                  // 心跳时间戳
  externalId: String,               // 数据仓库侧查询ID（用于取消）
  result: {},                       // 处理后结果（小结果内嵌）
  rawResult: [],                    // 原始结果（小结果内嵌）
  hasChunkedResults: Boolean,       // 是否使用分块存储
  error: String,
  statistics: {},                   // 执行统计信息
  dependencies: [String],           // 依赖查询ID列表
  runAtEnd: Boolean,                // 是否最后执行
  cachedQueryUsed: String,          // 复用的缓存查询ID
});

// 复合索引：组织 + 数据源 + 状态 + 创建时间
querySchema.index({ 
  organization: 1, datasource: 1, status: 1, createdAt: -1 
});
```

### 3.3 结果写入流程

**代码位置**：`packages/back-end/src/models/QueryModel.ts:134-166`

> **⚠️ 关键修正**：分块触发条件与结果大小无关，而是 **`result === rawResult` 的引用相等判断**。当查询提供了 `process` 转换函数时，`result` 和 `rawResult` 是不同对象，**即使结果再大也不会分块**，直接内嵌到主文档中（可能触发 MongoDB 16MB 限制）。

```typescript
// 调用方设置结果时：
//   result: process ? process(rows) : rows,
// 当有 process 函数时，result !== rawResult → 不会分块
// 当无 process 函数时，result === rawResult → 触发分块

export async function updateQuery(
  context: ReqContext,
  query: QueryInterface,
  changes: Partial<QueryInterface>,
): Promise<QueryInterface> {
  if (query.organization !== context.org.id) {
    throw new Error("Cannot update query from different organization");
  }

  // ========== 分块触发条件（与大小无关）==========
  // 注释明确说明：Some legacy queries have processed results that differ
  // from raw results, so skip those
  if (
    changes.result &&
    changes.result === changes.rawResult &&  // ⚠️ 引用相等是核心判断
    changes.rawResult.length > 0
  ) {
    // 走分块存储路径
    await context.models.sqlResultChunks.createFromResults(
      query.id,
      changes.rawResult,
    );
    changes = omit(changes, ["result", "rawResult"]);
    changes.hasChunkedResults = true;
  }
  // else：结果直接内嵌到主文档

  await QueryModel.updateOne(
    { organization: query.organization, id: query.id },
    { $set: changes },
  );

  return { ...query, ...changes };
}

// updateQueryIfRunning 中有同样的分块判断，多了一个 rawResult 存在性检查
export async function updateQueryIfRunning(...) {
  if (
    changes.result &&
    changes.result === changes.rawResult &&
    changes.rawResult &&            // 多的这层判断
    changes.rawResult.length > 0
  ) { ... }
}
```

**分块判定真值表**：
| `process` 函数 | `result === rawResult` | 是否分块 | 结果存储位置 |
|---------------|------------------------|---------|------------|
| 无（默认） | ✅ 引用相等 | ✅ 分块 | `sqlresultchunks` 集合 |
| 有（自定义转换） | ❌ 引用不等 | ❌ 不分块 | 主文档内嵌（16MB 限制风险） |

**代码可验证约束**：
- MongoDB 单文档大小限制为 16MB（MongoDB 固有约束）
- 当 `result !== rawResult` 时，结果直接写入主文档 `result` 字段，无大小检查逻辑

### 3.4 列式编码与分块算法

**代码位置**：`packages/shared/src/sql.ts:202-260`

```typescript
export function encodeSQLResults(
  results: Record<string, unknown>[],
  chunkSizeBytes: number = 4_000_000,  // 每块约4MB
): SqlResultChunkData[] {
  if (results.length === 0) return [];

  const columns = Object.keys(results[0]);
  
  function createChunk(): SqlResultChunkData {
    const chunk: SqlResultChunkData = { numRows: 0, data: {} };
    columns.forEach(col => { chunk.data[col] = []; });
    return chunk;
  }

  // 估算字段大小（保守策略）
  function getSize(value: unknown): number {
    if (value === null || value === undefined) return 1;
    if (typeof value === "boolean") return 1;
    if (typeof value === "number") return 8;
    if (typeof value === "string") return value.length + 5;
    return 1 + JSON.stringify(value).length * 2;  // 嵌套对象加倍预留
  }

  // 行存转列存，按大小分块
  let currentChunk = createChunk();
  let currentChunkSize = 0;

  for (const row of results) {
    currentChunk.numRows++;
    for (const col of columns) {
      const value = row[col];
      currentChunk.data[col].push(value);
      currentChunkSize += getSize(value);
    }

    if (currentChunkSize >= chunkSizeBytes) {
      encodedResults.push(currentChunk);
      currentChunk = createChunk();
      currentChunkSize = 0;
    }
  }

  if (currentChunkSize > 0) encodedResults.push(currentChunk);
  return encodedResults;
}
```

### 3.5 分块写入与读取

**代码位置**：`packages/back-end/src/models/SqlResultChunkModel.ts:31-95`

```typescript
export class SqlResultChunkModel extends BaseClass {
  // ========== 写入 ==========
  public async createFromResults(
    queryId: string,
    results: Record<string, unknown>[],
  ) {
    const encodedChunks = encodeSQLResults(results);
    // 并发3个写入
    await promiseAllChunks(
      encodedChunks.map((chunk, i) => async () => {
        await this.create({
          queryId,
          chunkNumber: i,
          ...chunk,
        });
      }),
      3,
    );
  }

  // ========== 读取并合并 ==========
  public async addResultsToQueries(queries: QueryInterface[]) {
    const idsToFetch = queries
      .filter((q) => q.hasChunkedResults)
      .map((q) => (q.cachedQueryUsed ? q.cachedQueryUsed : q.id));

    if (!idsToFetch.length) return;

    // ⚠️ 重要：这里一次性加载所有 chunk，没有按列懒加载，没有分页
    // 查询条件只有 queryId，没有字段投影，没有 chunk 过滤
    const allChunks = await this._find(
      {
        queryId: { $in: idsToFetch },
      },
      {
        sort: { queryId: 1, chunkNumber: 1 },
      },
    );

    // 按查询ID分组
    const chunksByQueryId: Record<string, SqlResultChunkInterface[]> = {};
    for (const chunk of allChunks) {
      if (!chunksByQueryId[chunk.queryId]) {
        chunksByQueryId[chunk.queryId] = [];
      }
      chunksByQueryId[chunk.queryId].push(chunk);
    }

    // ⚠️ 列存转行存：decodeSQLResults 会把所有 chunk 的所有列全部展开成行数组
    // 没有按需加载列的能力，也没有按行分页的 API
    for (const query of queries) {
      const queryId = query.cachedQueryUsed ? query.cachedQueryUsed : query.id;
      if (chunksByQueryId[queryId]) {
        const result = decodeSQLResults(chunksByQueryId[queryId]);
        query.rawResult = result;
        query.result = result;
      }
    }
  }

  // 单查询全量读取接口，同样加载所有 chunk
  public async getResultsByQueryId(queryId: string) {
    const all = await this._find(
      { queryId },
      {
        sort: { chunkNumber: 1 },
      },
    );
    return decodeSQLResults(all);
  }
}
```

**列式解码** (`decodeSQLResults`)：
**代码位置**：`packages/shared/src/sql.ts:345-365`

```typescript
export function decodeSQLResults(
  chunks: SqlResultChunkData[],
): Record<string, unknown>[] {
  const results: Record<string, unknown>[] = [];

  for (const chunk of chunks) {
    const { data, numRows } = chunk;
    if (!numRows) continue;

    // 遍历所有列，没有按列选择的参数
    const columns = Object.keys(data);
    for (let i = 0; i < numRows; i++) {
      const row: Record<string, unknown> = {};
      for (const col of columns) {
        row[col] = data[col]?.[i] ?? null;
      }
      results.push(row);
    }
  }

  return results;
}
```

**⚠️ 列式存储与按列懒加载的能力边界**：

| 特性 | 代码证据 | 是否支持 |
|-----|---------|---------|
| 列式物理存储 | `encodeSQLResults` 按列组织数组 | ✅ 是 |
| 按列懒加载（按需加载部分列） | `addResultsToQueries` 查询条件无字段投影，`decodeSQLResults` 无列选择参数 | ❌ 否 |
| 按行分页读取 | `getResultsByQueryId` 返回全部结果，无 limit/offset 参数 | ❌ 否 |
| 单字段投影查询 | 无对应 API，读取总是加载所有列的所有 chunk | ❌ 否 |

**代码可验证事实**：
1. 分块大小硬编码为 `chunkSizeBytes: number = 4_000_000`（`shared/src/sql.ts:206`）
2. `sqlresultchunks` 集合使用独立复合索引 `{ organization: 1, queryId: 1, chunkNumber: 1 }`（`models/SqlResultChunkModel.ts:13`）
3. `queries` 集合使用复合索引 `{ organization: 1, datasource: 1, status: 1, createdAt: -1 }`（`models/QueryModel.ts` Schema 定义）

**读取路径全链路**：
```
getQueriesByIds(ids, includeChunkedResults=true)
    ↓ (QueryModel.ts:76-78)
context.models.sqlResultChunks.addResultsToQueries(queries)
    ↓ (SqlResultChunkModel.ts:55-62)
this._find({ queryId: { $in: idsToFetch } })
    ↓ 全量加载所有 chunk
decodeSQLResults(allChunks)
    ↓ 全量展开所有列的所有行
query.rawResult = result  ← 完整行数组
query.result = result     ← 完整行数组（同一个引用）
```

### 3.6 缓存复用机制

**代码位置**：`packages/back-end/src/models/QueryModel.ts:208-234`

```typescript
export async function getRecentQuery(
  organization: string,
  datasource: string,
  query: string,
  cacheTTLMins?: number,
) {
  const ttl = cacheTTLMins ?? QUERY_CACHE_TTL_MINS;  // 默认？
  const earliestDate = new Date();
  earliestDate.setMinutes(earliestDate.getMinutes() - ttl);

  const latest = await QueryModel.find({
    organization,
    datasource,
    query,                                    // 相同SQL
    createdAt: { $gt: earliestDate },         // 在有效期内
    status: { $in: ["succeeded", "running"] }, // 成功或运行中
    cachedQueryUsed: { $exists: false },     // 排除缓存副本本身
  })
    .sort({ createdAt: -1 })
    .limit(1);

  return latest[0] ? toInterface(latest[0]) : null;
}

// 创建缓存副本（不重复执行）
export async function createNewQueryFromCached({
  existing,
  dependencies,
  runAtEnd,
}) {
  const data: QueryInterface = {
    createdAt: new Date(),
    datasource: existing.datasource,
    id: generateId("qry_"),
    // 复用结果
    status: existing.status,
    result: existing.result,
    rawResult: existing.rawResult,
    error: existing.error,
    statistics: existing.statistics,
    // 新的依赖关系
    dependencies,
    runAtEnd,
    // 标记来源
    cachedQueryUsed: existing.cachedQueryUsed || existing.id,
    hasChunkedResults: existing.hasChunkedResults,
    // ... 其他字段复制
  };
  const doc = await QueryModel.create(data);
  return toInterface(doc);
}
```

---

## 四、三者协同完整流程

### 4.1 时序调用链

```
用户请求分析
    ↓
[查询编排层] QueryRunner.startAnalysis()
    ├→ startQueries() 【子类实现】生成查询DAG
    │   └→ for each query: startQuery()
    │       ├→ 检查缓存 getRecentQuery()
    │       │   └→ 命中 → createNewQueryFromCached()
    │       ├→ 未命中 → createNewQuery()
    │       ├→ 检查依赖与并发
    │       └→ 就绪 → executeQuery()
    └→ updateModel() 持久化初始状态

[连接层] 异步执行中...
    ↓
SqlIntegration.runQuery()
    ├→ 解密参数 setParams()
    ├→ 建立连接 buildSnowflakeConnection()
    ├→ 异步提交获取 externalId
    ├→ 持久化 externalId
    └→ 等待结果 getResultsFromQueryId()

[查询编排层] 回调触发
    ↓
executeQuery().then()
    ├→ updateQuery() 【结果落库】
    │   ├→ 大结果 → createFromResults() 分块存储
    │   └→ 小结果 → 内嵌到主文档
    └→ onQueryFinish()
        └→ setTimeout(1s) 刷新状态
            ├→ refreshQueryStatuses()
            │   ├→ updateQueryPointers() 拉取最新状态
            │   ├→ getOverallQueryStatus() 计算整体状态
            │   └→ 全部完成 → runAnalysis() + updateModel()
            └→ startReadyQueries() 启动下一批依赖查询

[循环] 直到所有查询完成
    ↓
最终结果返回
```

### 4.2 关键协同点

| 协同点 | 连接层 | 编排层 | 落库层 |
|-------|-------|-------|-------|
| **查询取消** | 提供 `cancelQuery()` 接口，通过 `externalId` 终止 | 维护 `externalId` 映射，处理级联取消 | - |
| **并发控制** | - | `concurrencyLimitReached()` 统计运行数，指数退避排队 | `countRunningQueries()` 统计查询数 |
| **缓存复用** | - | `getRecentQuery()` 检查SQL+TTL，`createNewQueryFromCached()` 创建副本 | 支持 `cachedQueryUsed` 跨文档引用分块 |
| **状态流转** | 成功/异常回调 | 状态机、依赖检查、级联失败 | `status` 字段持久化 |
| **心跳检测** | - | 30秒定时器更新 `heartbeat` | 后台任务 `getStaleQueries()` 清理失联查询 |

### 4.3 错误处理链路

```
查询执行异常
    ↓
executeQuery().catch()
    ├→ updateQueryIfRunning() 标记为 failed
    │   └→ （可选）分块写入结果
    ├→ onFailure() 回调
    └→ onQueryFinish() 触发状态刷新
        └→ startReadyQueries()
            └→ 级联标记依赖查询为 failed
                └→ getOverallQueryStatus() 判断是否过半失败
                    └→ updateModel() 整体标记为 failed
                        └→ setStatus("finished", error)
                            └→ EventEmitter.emit("finish")
```

---

## 五、关键设计权衡（仅保留代码可验证事实）

### 5.1 连接模式：按需创建

**代码可验证事实**：
- `runSnowflakeQuery` 每次调用都会 `buildSnowflakeConnection()` + `destroySnowflakeConnection()`（`services/snowflake.ts:92-123`）
- `BigQuery.getClient()` 每次调用都会 `new bq.BigQuery()`（`integrations/BigQuery.ts:47-60`）
- `try/finally` 确保 `destroySnowflakeConnection()` 一定执行（`services/snowflake.ts:121-123`）

### 5.2 结果存储：行存转列存分块

**代码可验证事实**：
- 分块触发条件是 `changes.result === changes.rawResult`（引用相等），与结果大小无关（`models/QueryModel.ts:143-156`）
- 当提供 `process` 函数时，`result = process(rows)`，`rawResult = rows`，引用不等，不会触发分块（`queryRunners/QueryRunner.ts:731-737`）
- 分块大小硬编码为 `chunkSizeBytes: number = 4_000_000`（`shared/src/sql.ts:206`）
- 分块存储使用独立复合索引 `{ organization: 1, queryId: 1, chunkNumber: 1 }`（`models/SqlResultChunkModel.ts:13`）

### 5.3 调度模式：事件驱动 + 1秒延迟刷新

**代码可验证事实**：
- `onQueryFinish()` 使用固定 `setTimeout(..., 1000)`，无动态调整逻辑（`queryRunners/QueryRunner.ts:191-248`）
- `onQueryFinish()` 通过 `if (!this.timer)` 实现去重，避免重复设置（`queryRunners/QueryRunner.ts:192`）
- `startAnalysis()` 末尾兜底调用 `onQueryFinish()`，注释明确说明是为了解决 DAG 持久化竞态（`queryRunners/QueryRunner.ts:314-331` 注释）
- 竞态场景的完整时间线描述在注释中明确写出（`queryRunners/QueryRunner.ts:317-330` 注释）

### 5.4 缓存粒度：完整SQL精确匹配

**代码可验证事实**：
- `getRecentQuery()` 使用 `query: query` 精确匹配 SQL 文本（`models/QueryModel.ts:239`）
- `createNewQueryFromCached()` 时 `cachedQueryUsed: existing.cachedQueryUsed || existing.id`，确保永远指向原始（`models/QueryModel.ts` 中创建副本逻辑）
- `addResultsToQueries()` 支持 `cachedQueryUsed` 跨文档引用分块（`models/SqlResultChunkModel.ts:49-51`）

### 5.5 取消路径：一跳回溯设计

**代码可验证事实**：
- `cancelQueries()` 仅处理 `status === "running"` 的查询，`queued` 查询不参与 externalId 回溯（`queryRunners/QueryRunner.ts:565-567`）
- `getQueriesByIds(..., false)` 传入 `includeChunkedResults=false`，避免加载分块数据（`queryRunners/QueryRunner.ts:570-574`）
- `cachedSourceIds` 使用 `Array.from(new Set(...))` 去重（`queryRunners/QueryRunner.ts:584-590`）
- 取消失败时仅打 `logger.debug`，不抛出异常中断流程（`queryRunners/QueryRunner.ts:616`）
- 状态收敛时直接 `queries: []` 清空数组，不逐个更新查询文档（`queryRunners/QueryRunner.ts:626-630`）
- `clearAllTimers()` 遍历 `pendingTimers` 清理所有定时器（`queryRunners/QueryRunner.ts:180-185`）

### 5.6 结果读取：全量加载模式

**代码可验证事实**：
- `addResultsToQueries()` 使用 `queryId: { $in: idsToFetch }` 查询条件，无字段投影，无分页参数（`models/SqlResultChunkModel.ts:55-62`）
- `decodeSQLResults()` 无列选择参数，遍历所有列展开成完整行数组（`shared/src/sql.ts:345-365`）
- `getResultsByQueryId()` 同样返回全部结果，无 limit/offset 参数（`models/SqlResultChunkModel.ts:80-88`）
- 全代码库 grep 未发现按列懒加载、按行分页、字段投影等查询优化 API（`lazy.*load|load.*lazy|select.*column|column.*select|projection` 等关键词无匹配）

### 5.7 分块编码算法

**代码可验证事实**：
- `encodeSQLResults()` 按列组织数据，相同列的值存入同一数组（`shared/src/sql.ts:239-245`）
- `getSize()` 估算字段大小：`null/undefined/boolean` 计 1 字节，`number` 计 8 字节，`string` 计 `length + 5` 字节，嵌套对象计 `1 + JSON.stringify 长度 * 2`（`shared/src/sql.ts:226-234`）
- 分块阈值为 `>= 4_000_000` 字节（`shared/src/sql.ts:247`）
- 并发写入分块的并发数硬编码为 3（`models/SqlResultChunkModel.ts:36-44`）

---

## 六、核心代码速查表

| 功能模块 | 主要文件 | 关键函数 |
|---------|---------|---------|
| **连接基类** | `integrations/SqlIntegration.ts` | `constructor`, `wrapRunQuery`, `runQuery` |
| **Snowflake连接** | `services/snowflake.ts` | `buildSnowflakeConnection`, `runSnowflakeQuery`, `destroySnowflakeConnection`, `cancelSnowflakeQuery` |
| **BigQuery连接** | `integrations/BigQuery.ts` | `getClient`(每次new), `runQuery`, `cancelQuery` |
| **取消路径** | `queryRunners/QueryRunner.ts` | `cancelQueries`(externalId回溯:558-622) |
| **状态刷新** | `queryRunners/QueryRunner.ts` | `onQueryFinish`(1s延迟:191-256), `refreshQueryStatuses`(469-556) |
| **DAG持久化竞态防护** | `queryRunners/QueryRunner.ts` | `startAnalysis`(末尾兜底onQueryFinish:314-331) |
| **查询编排器** | `queryRunners/QueryRunner.ts` | `startAnalysis`, `startQuery`, `executeQuery`, `startReadyQueries` |
| **并发排队** | `queryRunners/QueryRunner.ts` | `queueQueryExecution`(指数退避:637-654), `executeQueryWhenReady`(656-680) |
| **DAG生成示例** | `queryRunners/ExperimentResultsQueryRunner.ts` | `startExperimentResultQueries` |
| **查询主模型** | `models/QueryModel.ts` | `createNewQuery`, `updateQuery`(分块判断:143-156), `getRecentQuery`, `getStaleQueries`, `countRunningQueries`, `updateQueryIfRunning` |
| **缓存副本** | `models/QueryModel.ts` | `createNewQueryFromCached`(cachedQueryUsed一跳保证) |
| **结果分块** | `models/SqlResultChunkModel.ts` | `createFromResults`, `addResultsToQueries`(全量加载:48-78), `getResultsByQueryId` |
| **编解码算法** | `shared/src/sql.ts` | `encodeSQLResults`(4MB分块:202-260), `decodeSQLResults`(全量展开:345-365) |

---

## 七、关键偏差修正总结

| 先前理解 | 实际实现（代码可验证） | 代码位置 |
|---------|---------|---------|
| BigQuery 有应用层客户端池 | 每次 `getClient()` 都 `return new bq.BigQuery()`，无应用层复用 | `BigQuery.ts:47-60` |
| 结果分块按大小触发 | 按 `result === rawResult` 引用相等触发；有 `process` 函数时引用不等，永不分块 | `QueryModel.ts:143-156` |
| 列式存储支持按列懒加载 | `addResultsToQueries()` 全量加载所有 chunk，`decodeSQLResults()` 无列选择参数 | `SqlResultChunkModel.ts:55-78`, `sql.ts:345-365` |
| 取消时直接用 externalId | 需要通过 `cachedQueryUsed` 回溯到原始查询文档；仅处理 `running` 查询，`queued` 查询不回溯 | `QueryRunner.ts:565-606` |
| `queued` 查询取消时也调用 `cancelQuery()` | `queued` 查询未提交到数据仓库，仅清理本地定时器，不调用 `cancelQuery()` | `QueryRunner.ts:565-567`, `QueryRunner.ts:625` |
| 取消时逐个更新查询文档状态 | 直接 `queries: []` 清空数组，不逐个更新，依赖模型状态机收敛 | `QueryRunner.ts:626-630` |
| 1秒刷新主要用于防抖 | 核心目的是规避 DAG 持久化竞态，代码注释明确写出完整竞态时间线 | `QueryRunner.ts:314-331` 注释 |
| 分块目的是压缩和查询优化 | 分块目的是绕过 MongoDB 16MB 限制和使用独立索引；无压缩代码，无懒加载代码 | `SqlResultChunkModel.ts:13`, `sql.ts:202-260` |
| 所有结论可包含推测性描述 | 仅保留代码可直接验证的事实，删除"优点/缺点/潜在风险"等主观判断 | 全文 |
