# Dashboard 查询执行流程解析

本文档深入解析 GrowthBook 中 Dashboard 编辑保存后，查询是如何真正执行的完整流程。

## 一、整体架构概览

Dashboard 查询执行涉及三个核心环节的衔接：

```
前端可视化配置 → 配置序列化存储 → 查询拼装 → 数据源调度 → 结果存储与展示
```

---

## 二、可视化配置的序列化过程

### 2.1 Dashboard Block 类型系统

Dashboard 由多个 Block 组成，每种 Block 对应不同的查询类型：

| Block 类型 | 用途 | 关联查询类型 |
|-----------|------|-------------|
| `sql-explorer` | 自定义 SQL 查询 | SavedQuery |
| `metric-explorer` | 指标探索（已废弃） | MetricAnalysis |
| `metric-exploration` | 产品分析指标探索 | ProductAnalyticsExploration |
| `fact-table-exploration` | 事实表探索 | ProductAnalyticsExploration |
| `data-source-exploration` | 数据源探索 | ProductAnalyticsExploration |
| `experiment-metric` | 实验指标结果 | ExperimentSnapshot |
| `experiment-dimension` | 实验维度结果 | ExperimentSnapshot |
| `experiment-time-series` | 实验时间序列 | ExperimentSnapshot |
| `experiment-traffic` | 实验流量 | ExperimentSnapshot |
| `markdown` | Markdown 文本 | 无 |

**代码位置**：`packages/front-end/enterprise/components/Dashboards/DashboardEditor/index.tsx:64-113`

### 2.2 可视化配置结构（以 SqlExplorer 为例）

SqlExplorer Block 的配置包含两部分：

1. **Block 配置**（存储在 Dashboard 的 blocks 数组中）：
```typescript
interface SqlExplorerBlockInterface {
  type: "sql-explorer";
  id: string;           // Block ID (dshblk_ 前缀)
  uid: string;          // UUID
  savedQueryId: string; // 关联的 SavedQuery ID
  blockConfig: string[]; // 配置项 ID 列表，如 ["results-table", "chart-1"]
  dataVizConfigIndex?: number; // 向后兼容字段
}
```

2. **SavedQuery 配置**（独立存储在 savedQueries 集合中）：
```typescript
interface SavedQuery {
  id: string;
  datasourceId: string;    // 数据源 ID
  sql: string;             // SQL 语句
  name: string;            // 查询名称
  dateLastRan: Date;       // 最后执行时间
  dataVizConfig: DataVizConfig[]; // 可视化配置数组
  results: QueryExecutionResult;  // 查询结果
  linkedDashboardIds: string[];   // 关联的 Dashboard ID
}
```

**代码位置**：`packages/shared/src/validators/saved-queries.ts:261-275`

### 2.3 DataVizConfig 可视化配置详解

`dataVizConfig` 是可视化配置的核心，支持多种图表类型：

```typescript
// 基础配置
interface BaseChartConfig {
  id?: string;           // 可视化项 ID
  title?: string;        // 图表标题
  yAxis: YAxisConfig[];  // Y 轴配置
  filters?: FilterConfig[]; // 筛选条件
}

// 图表类型（通过 chartType 区分）
type DataVizConfig = 
  | BarChart      // 柱状图
  | LineChart     // 折线图
  | AreaChart     // 面积图
  | ScatterChart  // 散点图
  | BigValueChart // 大数值
  | PivotTable;   // 透视表
```

**代码位置**：`packages/shared/src/validators/saved-queries.ts:148-229`

### 2.4 序列化流程

**前端 → 后端 保存流程**：

1. 前端编辑器（`DashboardEditor/index.tsx`）收集所有 Block 配置
2. 调用 API 发送到后端：`PUT /api/dashboards/:id`
3. 后端 `processApiUpdateBody` 处理请求体：
   - 调用 `fromBlockApiInterface` 转换 API 格式到内部格式
   - 调用 `migrateBlock` 处理版本迁移
   - 为新 Block 生成 ID：`generateDashboardBlockIds()`

**代码位置**：
- `packages/back-end/src/enterprise/models/DashboardModel.ts:442-460`
- `packages/back-end/src/routers/dashboards/dashboards.controller.ts:131-168`

---

## 三、查询拼装逻辑

### 3.1 查询触发时机

查询可以通过两种方式触发：

| 触发方式 | 入口代码 | 说明 |
|---------|---------|------|
| 手动刷新 | `dashboards.controller.ts:180` `refreshDashboardData` | 用户点击刷新按钮 |
| 定时更新 | `jobs/updateDashboards.ts` | 每 10 分钟检查一次 `nextUpdate` 到期的 Dashboard |

### 3.2 SQL Explorer 查询流程

**核心执行链路**：

```
refreshDashboardData()
  → updateDashboardSavedQueries()
    → executeAndSaveQuery()
      → runFreeFormQuery()
        → integration.getFreeFormQuery()
        → integration.runTestQuery()
          → integration.runQuery()
```

#### 3.2.1 `updateDashboardSavedQueries` - 批量更新 Dashboard 中的 SavedQuery

```typescript
async function updateDashboardSavedQueries(context, blocks) {
  // 1. 从 blocks 中提取所有 savedQueryId
  const savedQueryIds = blocks
    .filter(block => blockHasFieldOfType(block, "savedQueryId", isString))
    .map(block => block.savedQueryId);
  
  // 2. 获取 SavedQuery 对象
  const savedQueries = await context.models.savedQueries.getByIds(savedQueryIds);
  
  // 3. 获取关联的数据源
  const datasources = await getDataSourcesByIds(context, datasourceIds);
  
  // 4. 并行执行所有查询
  await Promise.all(
    savedQueries.map(savedQuery => 
      executeAndSaveQuery(context, savedQuery, datasource)
    )
  );
}
```

**代码位置**：`packages/back-end/src/enterprise/services/dashboards.ts:379-414`

#### 3.2.2 `executeAndSaveQuery` - 执行并保存查询

```typescript
async function executeAndSaveQuery(context, savedQuery, datasource, limit = 1000) {
  // 1. 执行查询
  const { results, sql, duration, error } = await runFreeFormQuery(
    context,
    datasource,
    savedQuery.sql,
    limit
  );

  // 2. 无错误才保存结果
  if (!error && results) {
    await context.models.savedQueries.update(savedQuery, {
      results: { results, error, duration, sql },
      dateLastRan: new Date(),
    });
  }
}
```

**代码位置**：`packages/back-end/src/routers/saved-queries/saved-queries.controller.ts:249-281`

#### 3.2.3 `runFreeFormQuery` - 执行自由格式 SQL

```typescript
async function runFreeFormQuery(context, datasource, query, limit?) {
  // 1. 权限检查
  if (!context.permissions.canRunSqlExplorerQueries(datasource)) {
    throw new Error("Permission denied");
  }

  // 2. 安全检查：只允许只读查询
  if (!isReadOnlySQL(query)) {
    throw new Error("Only SELECT queries are allowed.");
  }

  // 3. 获取数据源集成对象
  const integration = getSourceIntegrationObject(context, datasource);

  // 4. 拼装 SQL（添加 LIMIT，格式化）
  const sql = integration.getFreeFormQuery(query, limit);
  
  // 5. 执行查询
  const { results, duration, columns } = await integration.runTestQuery(
    sql,
    ["timestamp"],
    "freeFormQuery"
  );

  // 6. 推断列类型
  const typeMap = determineColumnTypes(results, columns);
  
  return { results, duration, sql, columns: finalColumns };
}
```

**代码位置**：`packages/back-end/src/services/datasource.ts:152-219`

#### 3.2.4 `getFreeFormQuery` - SQL 拼装

```typescript
function getFreeFormQuery(dialect, sql, limit?) {
  // 1. 确保有 LIMIT 子句（默认 1000 行）
  const limitedQuery = ensureLimit(sql, limit ?? SQL_ROW_LIMIT);
  
  // 2. 格式化 SQL（美化）
  return format(limitedQuery, dialect.formatDialect);
}
```

**代码位置**：`packages/back-end/src/integrations/sql/queries/free-form-query.ts:4-11`

### 3.3 其他 Block 类型的查询流程

#### 3.3.1 Metric Analysis（指标分析）

```
updateDashboardMetricAnalyses()
  → createMetricAnalysis()
    → 构建 Metric Analysis SQL
    → 执行查询
    → 更新 block.metricAnalysisId
```

**代码位置**：`packages/back-end/src/enterprise/services/dashboards.ts:253-320`

#### 3.3.2 Product Analytics Exploration（产品分析探索）

```
updateDashboardExplorations()
  → runProductAnalyticsExploration()
    → generateProductAnalyticsSQL()  // 动态生成 SQL
    → 执行查询
    → 更新 block.explorerAnalysisId
```

**代码位置**：`packages/back-end/src/enterprise/services/dashboards.ts:348-377`

#### 3.3.3 Experiment Snapshot（实验快照）

```
refreshDashboardData()  // 实验 Dashboard
  → planExperimentSnapshot()
  → createExperimentSnapshotFromPlan()
    → 构建实验分析 SQL
    → 执行查询
    → 更新 block.snapshotId
```

**代码位置**：`packages/back-end/src/routers/dashboards/dashboards.controller.ts:188-273`

---

## 四、数据源调度衔接

### 4.1 数据源集成架构

GrowthBook 使用**工厂模式 + 策略模式**管理多种数据源：

```
DataSourceInterface (配置)
       ↓
getIntegrationObj()  // 工厂函数
       ↓
具体数据源类 (ClickHouse / Snowflake / BigQuery 等)
       ↓
继承 SqlIntegration 抽象基类
```

#### 4.1.1 工厂函数 `getIntegrationObj`

```typescript
function getIntegrationObj(context, datasource) {
  switch (datasource.type) {
    case "growthbook_clickhouse":
    case "clickhouse":
      return new ClickHouse(context, datasource);
    case "snowflake":
      return new Snowflake(context, datasource);
    case "bigquery":
      return new BigQuery(context, datasource);
    case "postgres":
      return new Postgres(context, datasource);
    // ... 其他 10+ 种数据源
    default:
      throw new Error("Unknown data source type");
  }
}
```

**代码位置**：`packages/back-end/src/services/datasource.ts:71-105`

#### 4.1.2 SqlIntegration 抽象基类

定义了所有 SQL 数据源的通用行为：

```typescript
abstract class SqlIntegration {
  abstract setParams(encryptedParams: string): void;
  abstract runQuery(sql: string): Promise<QueryResponse>;
  abstract getSensitiveParamKeys(): string[];
  abstract getSqlDialect(): SqlDialect;
  
  // 通用方法
  async testConnection() { ... }
  getFreeFormQuery(sql, limit) { ... }
  async runTestQuery(sql, timestampCols, queryType) { ... }
  // ... 20+ 种查询构建方法
}
```

**代码位置**：`packages/back-end/src/integrations/SqlIntegration.ts:178-199`

#### 4.1.3 具体数据源实现（以 ClickHouse 为例）

```typescript
class ClickHouse extends SqlIntegration {
  setParams(encryptedParams: string) {
    this.params = decryptDataSourceParams<ClickHouseConnectionParams>(encryptedParams);
  }

  getSqlDialect(): SqlDialect {
    return clickHouseDialect;
  }

  async runQuery(sql: string): Promise<QueryResponse> {
    const client = createClient({
      url: getHost(this.params.url, this.params.port),
      username: this.params.username,
      password: this.params.password,
      database: this.params.database,
      // ... 其他配置
    });
    
    const results = await client.query({ query: sql, format: "JSON" });
    const data = await results.json();
    
    return {
      rows: data.data || [],
      statistics: {
        executionDurationMs: data.statistics.elapsed,
        rowsProcessed: data.statistics.rows_read,
        bytesProcessed: data.statistics.bytes_read,
      },
    };
  }
}
```

**代码位置**：`packages/back-end/src/integrations/ClickHouse.ts:19-81`

### 4.2 定时任务调度

#### 4.2.1 调度入口

使用 Agenda（MongoDB  backed 的任务调度库）：

```typescript
// 每 10 分钟执行一次，检查需要更新的 Dashboard
agenda.define(QUEUE_DASHBOARD_UPDATES, async () => {
  const dashboards = await DashboardModel.getDashboardsToUpdate();
  
  for (const dash of dashboards) {
    await queueDashboardUpdate(dash.organization, dash.id);
  }
});

// 启动定时任务
async function startUpdateJob() {
  const job = agenda.create(QUEUE_DASHBOARD_UPDATES, {});
  job.unique({});
  job.repeatEvery("10 minutes");
  await job.save();
}
```

**代码位置**：`packages/back-end/src/jobs/updateDashboards.ts:9-33`

#### 4.2.2 待更新 Dashboard 查询逻辑

```typescript
static async getDashboardsToUpdate() {
  return getCollection(COLLECTION_NAME)
    .find({
      isDeleted: false,
      isDefault: false,
      enableAutoUpdates: true,
      experimentId: null,  // 只处理非实验 Dashboard
      $or: [
        { nextUpdate: { $exists: true, $lte: new Date() } },  // 已到更新时间
        { nextUpdate: { $exists: false } },                   // 从未更新过
      ],
    })
    .project({ id: true, organization: true })
    .limit(100)
    .sort({ nextUpdate: 1 })
    .toArray();
}
```

**代码位置**：`packages/back-end/src/enterprise/models/DashboardModel.ts:113-147`

#### 4.2.3 单个 Dashboard 更新

```typescript
const updateSingleDashboard = async (job) => {
  const { dashboardId, organization } = job.attrs.data;
  
  const context = await getContextForAgendaJobByOrgId(organization);
  const dashboard = await context.models.dashboards.getById(dashboardId);
  
  try {
    await updateNonExperimentDashboard(context, dashboard);
  } catch (e) {
    // 更新失败，关闭自动更新
    await context.models.dashboards.dangerousUpdateByIdBypassPermission(
      dashboardId,
      { enableAutoUpdates: false }
    );
  }
};
```

**代码位置**：`packages/back-end/src/jobs/updateDashboards.ts:53-82`

### 4.3 下次更新时间计算

`nextUpdate` 字段通过 `determineNextDate` 函数计算：

```typescript
// 在 beforeUpdate 钩子中自动计算
protected async beforeUpdate(existing, updates, _newDoc) {
  if (shouldRecalculateNextUpdate(updates, existing)) {
    const schedule = updates.updateSchedule ?? existing.updateSchedule;
    updates.nextUpdate = schedule
      ? determineNextDate(schedule) ?? undefined
      : undefined;
  }
}
```

**代码位置**：`packages/back-end/src/enterprise/models/DashboardModel.ts:379-395`

---

## 五、完整调用链路

### 5.1 手动刷新 Dashboard 完整链路

```
前端点击"刷新"按钮
  ↓
PUT /api/dashboards/:id/refresh
  ↓
refreshDashboardData() [dashboards.controller.ts:180]
  ├─ 实验 Dashboard
  │   ├─ planExperimentSnapshot()
  │   ├─ createExperimentSnapshotFromPlan()
  │   └─ 更新 block.snapshotId
  └─ 通用 Dashboard
      └─ updateNonExperimentDashboard() [dashboards.ts:235]
          ├─ updateDashboardMetricAnalyses()
          │   └─ createMetricAnalysis()
          ├─ updateDashboardSavedQueries()
          │   └─ executeAndSaveQuery() [saved-queries.controller.ts:249]
          │       └─ runFreeFormQuery() [datasource.ts:152]
          │           ├─ getSourceIntegrationObject()
          │           ├─ integration.getFreeFormQuery()
          │           │   └─ ensureLimit() + format()
          │           └─ integration.runTestQuery()
          │               └─ integration.runQuery()  // 具体数据源实现
          └─ updateDashboardExplorations()
              └─ runProductAnalyticsExploration()
```

### 5.2 定时更新完整链路

```
Agenda 定时任务 (每 10 分钟)
  ↓
QUEUE_DASHBOARD_UPDATES 任务
  ↓
DashboardModel.getDashboardsToUpdate()
  ↓
为每个待更新 Dashboard 创建 UPDATE_SINGLE_DASH 任务
  ↓
updateSingleDashboard() [updateDashboards.ts:53]
  ↓
updateNonExperimentDashboard() [dashboards.ts:235]
  └─ (同上手动刷新的通用 Dashboard 流程)
```

---

## 六、关键数据结构

### 6.1 Dashboard 数据结构

```typescript
interface DashboardInterface {
  id: string;                    // dash_ 前缀
  organization: string;
  title: string;
  userId: string;                // 创建者
  editLevel: "private" | "published";
  shareLevel: "private" | "public" | "organization";
  enableAutoUpdates: boolean;
  updateSchedule?: UpdateSchedule;
  nextUpdate?: Date;             // 下次更新时间
  lastUpdated?: Date;            // 上次更新时间
  experimentId?: string;         // 关联的实验 ID（实验 Dashboard）
  projects: string[];
  blocks: DashboardBlockInterface[];  // Block 数组
  dateCreated: Date;
  dateUpdated: Date;
}
```

### 6.2 QueryExecutionResult 查询结果

```typescript
interface QueryExecutionResult {
  results: Record<string, any>[];  // 行数据
  error: string | null;
  duration?: number;               // 执行耗时（毫秒）
  sql?: string;                    // 实际执行的 SQL
  columns?: QueryResponseColumnData[];  // 列信息
}
```

### 6.3 数据流转

```
配置阶段:
  前端可视化配置 → API JSON → Zod 验证 → MongoDB 存储

执行阶段:
  MongoDB 读取配置 → 构建查询参数 → 生成 SQL → 数据源执行
  → 结果解析 → 类型推断 → 保存到 SavedQuery.results

展示阶段:
  前端请求 Dashboard → 获取 blocks 和关联的 savedQueries
  → 根据 dataVizConfig 渲染图表
```

---

## 七、关键文件索引

| 功能模块 | 文件路径 |
|---------|---------|
| Dashboard 模型 | `packages/back-end/src/enterprise/models/DashboardModel.ts` |
| Dashboard 服务 | `packages/back-end/src/enterprise/services/dashboards.ts` |
| Dashboard 控制器 | `packages/back-end/src/routers/dashboards/dashboards.controller.ts` |
| 数据源服务 | `packages/back-end/src/services/datasource.ts` |
| SavedQuery 控制器 | `packages/back-end/src/routers/saved-queries/saved-queries.controller.ts` |
| 定时更新任务 | `packages/back-end/src/jobs/updateDashboards.ts` |
| SQL 集成基类 | `packages/back-end/src/integrations/SqlIntegration.ts` |
| ClickHouse 实现 | `packages/back-end/src/integrations/ClickHouse.ts` |
| SQL 工具函数 | `packages/shared/src/sql.ts` |
| 可视化配置验证 | `packages/shared/src/validators/saved-queries.ts` |
| 前端 Dashboard 编辑器 | `packages/front-end/enterprise/components/Dashboards/DashboardEditor/index.tsx` |
| SqlExplorer Block | `packages/front-end/enterprise/components/Dashboards/DashboardEditor/DashboardBlock/SqlExplorerBlock.tsx` |
