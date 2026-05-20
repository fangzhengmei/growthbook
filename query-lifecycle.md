# 数据仓库分析查询执行生命周期 - 代码实现分析

本文从代码实现角度深入剖析 GrowthBook 中数据仓库查询执行的完整生命周期，涵盖 **连接池建立**、**查询编排** 与 **结果落库** 三个核心阶段的协同机制。

---

## 一、连接池建立阶段

### 1.1 核心架构设计

GrowthBook 采用 **"按需连接、按查询隔离"** 的连接管理策略，而非传统的长连接池复用模式。每个查询独立创建连接，执行完毕后立即销毁，避免连接泄漏问题。

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

### 1.5 取消路径中 externalId 回溯关联机制

**代码位置**：`packages/back-end/src/queryRunners/QueryRunner.ts:558-635`

> **⚠️ 关键修正**：externalId 不是简单的映射关系，需要处理 **缓存查询副本** 的特殊情况。缓存查询本身没有 `externalId`，必须通过 `cachedQueryUsed` 字段回溯到原始查询文档。

```typescript
public async cancelQueries(): Promise<void> {
  if (
    this.model.queries.some(
      (q) => q.status === "running" || q.status === "queued",
    )
  ) {
    const runningIds = this.model.queries
      .filter((q) => q.status === "running")
      .map((q) => q.query);

    if (runningIds.length) {
      const queryDocs = await getQueriesByIds(
        this.context,
        runningIds,
        false,
      );

      // ⚠️ 核心回溯逻辑：
      // Some "running" docs are pointers to a previously-started in-flight
      // query (see `createNewQueryFromCached`). Those copies share the
      // upstream datasource job with the original via `cachedQueryUsed`
      // and never have their own `externalId` set — only the original
      // doc does, populated by the integration's `runQuery`/poller.
      // Chase one hop to find the real id so we can actually cancel the
      // upstream job. `cachedQueryUsed` always points at the original
      // (not a copy of a copy), so a single lookup is sufficient.

      // 第一步：收集所有缓存来源ID（去重）
      const cachedSourceIds = Array.from(
        new Set(
          queryDocs
            .map((q) => q.cachedQueryUsed)
            .filter((id): id is string => Boolean(id)),
        ),
      );

      // 第二步：批量加载原始查询文档
      const cachedSourceDocs = cachedSourceIds.length
        ? await getQueriesByIds(this.context, cachedSourceIds, false)
        : [];
      const cachedSourceById = new Map(
        cachedSourceDocs.map((q) => [q.id, q]),
      );

      // 第三步：逐跳获取真实 externalId
      const externalIds = queryDocs
        .map((q) => {
          if (q.externalId) return q.externalId;  // 非缓存查询，直接有
          if (q.cachedQueryUsed) {
            // 缓存查询，回溯到原始文档取
            return cachedSourceById.get(q.cachedQueryUsed)?.externalId;
          }
          return undefined;
        })
        .filter((id): id is string => Boolean(id));

      // 第四步：并发调用取消（最多5并发）
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
    // ... 清理定时器，标记为失败
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

### 1.6 连接层设计要点

| 设计决策 | 实现方式 | 代码位置 |
|---------|---------|---------|
| **参数加密** | `decryptDataSourceParams` 解密敏感参数 | `services/datasource.ts` |
| **查询审计** | `queryTag` 注入用户ID、查询类型等元数据 | `util/integration.ts` |
| **超时控制** | 分场景设置超时（30s 测试 / 10min 常规） | `services/snowflake.ts:136` |
| **资源清理** | `try/finally` 确保连接销毁 | `services/snowflake.ts:224-226` |
| **取消支持** | 持久化 `externalId`，通过 `cachedQueryUsed` 回溯 | `QueryRunner.ts:558-622` |
| **BigQuery客户端** | 每次 `new BigQuery()`，依赖底层 keep-alive | `integrations/BigQuery.ts:47-60` |

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

**1秒延迟的双重作用**：
| 作用 | 说明 |
|-----|------|
| **防抖** | 短时间内多个查询连续完成时，合并为一次刷新（减少 DB 压力） |
| **竞态防护** | 给 `updateModel` 留出持久化窗口，避免读到不完整的 DAG |

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

**潜在风险**：当 `process` 函数只是浅拷贝或简单包装，且结果集很大时，可能导致主文档超出 MongoDB 16MB 限制而写入失败。

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

**代码位置**：`packages/back-end/src/models/SqlResultChunkModel.ts:31-88`

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
      .filter(q => q.hasChunkedResults)
      .map(q => q.cachedQueryUsed ? q.cachedQueryUsed : q.id);

    if (!idsToFetch.length) return;

    // 按查询ID + 分块号排序读取
    const allChunks = await this._find(
      { queryId: { $in: idsToFetch } },
      { sort: { queryId: 1, chunkNumber: 1 } },
    );

    // 按查询ID分组
    const chunksByQueryId: Record<string, SqlResultChunkInterface[]> = {};
    for (const chunk of allChunks) {
      if (!chunksByQueryId[chunk.queryId]) {
        chunksByQueryId[chunk.queryId] = [];
      }
      chunksByQueryId[chunk.queryId].push(chunk);
    }

    // 列存转行存
    for (const query of queries) {
      const queryId = query.cachedQueryUsed ? query.cachedQueryUsed : query.id;
      if (chunksByQueryId[queryId]) {
        const result = decodeSQLResults(chunksByQueryId[queryId]);
        query.rawResult = result;
        query.result = result;
      }
    }
  }
}
```

**列式解码** (`decodeSQLResults`)：
```typescript
export function decodeSQLResults(
  chunks: SqlResultChunkData[],
): Record<string, unknown>[] {
  const results: Record<string, unknown>[] = [];

  for (const chunk of chunks) {
    const { data, numRows } = chunk;
    if (!numRows) continue;

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

## 五、关键设计权衡

### 5.1 连接模式：按需创建 vs 连接池

**选择**：每次查询独立创建连接，执行完毕立即销毁
- ✅ 优点：避免连接泄漏，简化并发控制，天然隔离
- ❌ 缺点：连接握手开销（Snowflake 约 200-500ms），高并发下对 Warehouse 压力大

### 5.2 结果存储：行存转列存分块

**选择**：当 `result === rawResult`（无 `process` 转换时）触发列式存储 + 4MB 分块
- ✅ 优点：相同类型数据连续存储，压缩率更高；单字段访问无需加载全量
- ⚠️ 关键缺陷：触发条件是 **引用相等** 而非 **结果大小**，当提供 `process` 函数时即使 100MB 结果也会内嵌，存在 MongoDB 16MB 溢出风险
- ❌ 缺点：编码/解码有CPU开销，单行随机访问困难

### 5.3 调度模式：事件驱动 + 1秒延迟刷新

**选择**：查询完成后设置 1 秒延迟定时器（带去重），而非主动轮询
- ✅ 核心设计意图：**规避 DAG 持久化竞态**——防止第一个查询完成太快，导致 `updateModel` 还没写入完整 DAG 就触发了状态刷新
- ✅ 附带效果：天然防抖，短时间内多查询完成时合并刷新，降低 DB 压力
- ❌ 代价：存在 1 秒延迟窗口，整体流水线多了固定的 1 秒调度延迟
- ⚠️ 防御措施：`startAnalysis` 末尾再兜底调用一次 `onQueryFinish()`，确保即使竞态发生也能恢复

### 5.4 缓存粒度：完整SQL精确匹配

**选择**：按 SQL 文本精确匹配 + TTL，命中时创建查询副本（`createNewQueryFromCached`）
- ✅ 优点：实现简单，无一致性问题
- ✅ 通过 `cachedQueryUsed` 字段实现结果的"软共享"，分块数据跨文档引用
- ✅ 副本永远指向原始查询（不会出现 A→B→C 链式引用），取消时只需回溯一跳
- ❌ 缺点：SQL 微小差异（如空格、参数顺序）导致缓存失效

### 5.5 取消路径：一跳回溯设计

**选择**：取消时通过 `cachedQueryUsed` 回溯原始查询获取 `externalId`，最多一跳
- ✅ 设计约束：`createNewQueryFromCached` 时 `cachedQueryUsed: existing.cachedQueryUsed || existing.id`，确保永远指向原始
- ✅ 优点：一次批量查询即可拿到所有原始文档，O(n) 时间复杂度
- ⚠️ 并发场景：取消时可能遇到原始查询已完成，此时 `cancelQuery` 会抛出"查询已完成"异常，通过独立 try-catch 隔离不影响其他取消

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
| **结果分块** | `models/SqlResultChunkModel.ts` | `createFromResults`, `addResultsToQueries` |
| **编解码算法** | `shared/src/sql.ts` | `encodeSQLResults`(4MB分块:202-260), `decodeSQLResults` |

---

## 七、关键偏差修正总结

| 先前理解 | 实际实现 | 代码位置 |
|---------|---------|---------|
| BigQuery 有应用层客户端池 | 每次 `getClient()` 都 `new BigQuery()`，连接复用在底层 | `BigQuery.ts:47-60` |
| 结果分块按大小触发 | 按 `result === rawResult` 引用相等触发，有 `process` 函数时永不分块 | `QueryModel.ts:143-156` |
| 取消时直接用 externalId | 需要通过 `cachedQueryUsed` 回溯到原始查询文档获取，最多一跳 | `QueryRunner.ts:576-606` |
| 1秒刷新只是防抖 | 核心目的是规避 DAG 持久化竞态，防止查询永久卡在 queued | `QueryRunner.ts:314-331` 注释 |
| onQueryFinish 只是启动定时器 | `startAnalysis` 末尾的调用是兜底机制，确保竞态发生后能恢复 | `QueryRunner.ts:376` |
