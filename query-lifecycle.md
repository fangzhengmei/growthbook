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

BigQuery 采用客户端池 + Job 异步模式：
```typescript
private getClient() {
  if (!IS_CLOUD && this.params.authType === "auto") {
    return new bq.BigQuery();  // 从环境或元数据服务器自动获取凭证
  }
  return new bq.BigQuery({
    projectId: this.params.projectId,
    credentials: {
      client_email: this.params.clientEmail,
      private_key: this.params.privateKey,
    },
  });
}

async runQuery(sql: string, setExternalId?: ExternalIdCallback) {
  const client = this.getClient();
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

### 1.5 连接层设计要点

| 设计决策 | 实现方式 | 代码位置 |
|---------|---------|---------|
| **参数加密** | `decryptDataSourceParams` 解密敏感参数 | `services/datasource.ts` |
| **查询审计** | `queryTag` 注入用户ID、查询类型等元数据 | `util/integration.ts` |
| **超时控制** | 分场景设置超时（30s 测试 / 10min 常规） | `services/snowflake.ts:136` |
| **资源清理** | `try/finally` 确保连接销毁 | `services/snowflake.ts:224-226` |
| **取消支持** | 持久化 `externalId`，通过 `cancelQuery` 终止 | `QueryRunner.ts:721-725` |

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

```typescript
public async startAnalysis(params: Params): Promise<Model> {
  // 1. 生成查询DAG（由子类实现）
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

  // 3. 持久化状态
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
    this.onQueryFinish();  // 启动状态刷新定时器
  }

  return newModel;
}
```

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

```typescript
export async function updateQuery(
  context: ReqContext,
  query: QueryInterface,
  changes: Partial<QueryInterface>,
): Promise<QueryInterface> {
  // ========== 大结果分块逻辑 ==========
  if (
    changes.result &&
    changes.result === changes.rawResult &&
    changes.rawResult.length > 0
  ) {
    // 结果较大，存入分块集合
    await context.models.sqlResultChunks.createFromResults(
      query.id,
      changes.rawResult,
    );
    // 主文档只保留标记，移除大字段
    changes = omit(changes, ["result", "rawResult"]);
    changes.hasChunkedResults = true;
  }

  await QueryModel.updateOne(
    { organization: query.organization, id: query.id },
    { $set: changes },
  );

  return { ...query, ...changes };
}
```

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

**选择**：列式存储 + 4MB 分块
- ✅ 优点：相同类型数据连续存储，压缩率更高；单字段访问无需加载全量
- ❌ 缺点：编码/解码有CPU开销，单行随机访问困难

### 5.3 调度模式：事件驱动 + 轮询刷新

**选择**：查询完成后 1 秒延迟刷新，而非主动轮询
- ✅ 优点：低空闲消耗，响应及时
- ❌ 缺点：存在 1 秒延迟窗口，极端情况需额外定时补偿

### 5.4 缓存粒度：完整SQL精确匹配

**选择**：按 SQL 文本精确匹配 + TTL
- ✅ 优点：实现简单，无一致性问题
- ❌ 缺点：SQL 微小差异（如空格、参数顺序）导致缓存失效

---

## 六、核心代码速查表

| 功能模块 | 主要文件 | 关键函数 |
|---------|---------|---------|
| **连接基类** | `integrations/SqlIntegration.ts` | `constructor`, `wrapRunQuery`, `runQuery` |
| **Snowflake连接** | `services/snowflake.ts` | `buildSnowflakeConnection`, `runSnowflakeQuery`, `destroySnowflakeConnection` |
| **BigQuery连接** | `integrations/BigQuery.ts` | `getClient`, `runQuery` |
| **查询编排器** | `queryRunners/QueryRunner.ts` | `startAnalysis`, `startQuery`, `executeQuery`, `startReadyQueries`, `refreshQueryStatuses` |
| **DAG生成示例** | `queryRunners/ExperimentResultsQueryRunner.ts` | `startExperimentResultQueries` |
| **查询主模型** | `models/QueryModel.ts` | `createNewQuery`, `updateQuery`, `getRecentQuery`, `getStaleQueries`, `countRunningQueries` |
| **结果分块** | `models/SqlResultChunkModel.ts` | `createFromResults`, `addResultsToQueries` |
| **编解码算法** | `shared/src/sql.ts` | `encodeSQLResults`, `decodeSQLResults` |
