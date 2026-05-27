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

**依赖图管理：**
- 每个查询通过 `dependencies` 字段声明依赖（其他查询的 ID）
- `startReadyQueries()` 遍历查询，仅当所有依赖状态为 `succeeded` 时才执行

**`runAtEnd` 收尾依赖**（`QueryRunner.ts:431`）：

```typescript
if (query.runAtEnd) {
  const pendingQueries = this.model.queries.filter(
    (q) =>
      !queryMap.get(q.name)?.runAtEnd &&          // 排除其他 runAtEnd 查询
      (q.status === "queued" || q.status === "running"),  // 尚未完成
  );
  if (pendingQueries.length) {
    logger.debug(`${query.id}: "Run at end query" waiting...`);
    return;  // ⚠️ 直接退出整个 startReadyQueries 方法
  }
}
```

**等待条件：** 所有**非 `runAtEnd`** 的查询中，没有任何一个处于 `queued` 或 `running` 状态。注意使用 `queryMap.get(q.name)` 来获取 `runAtEnd` 标记（而非 `q.runAtEnd`），因为 `queryMap` 是从数据库加载的最新状态。

**关键设计 — `return` 而非 `continue`：**
- 当某个 `runAtEnd` 查询需要等待时，`startReadyQueries()` 方法**立即返回**，跳过当前循环中剩余的所有 queued 查询
- 这意味着 `runAtEnd` 查询具有"门闩"效应：只要第一个 `runAtEnd` 遇到阻塞，所有后续 queued 查询（包括其他 `runAtEnd` 和普通查询）都被跳过
- 后续查询的启动只能依赖下一次 `onQueryFinish()` 事件触发新一轮 `startReadyQueries()`

**执行顺序保证：**
- 普通查询：按 queued 数组中的顺序逐个检查依赖，满足即执行
- `runAtEnd` 查询：必须等所有普通查询完成，且在队列中靠前的 `runAtEnd` 优先执行

**并发控制：**
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

**查询缓存：**
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

