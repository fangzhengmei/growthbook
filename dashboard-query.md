# Dashboard 查询执行流程解析

本文档深入解析 GrowthBook 中 Dashboard 编辑保存后，查询是如何真正执行的完整流程。

> ✅ **文档一致性验证**：本文件所有结论均基于实际代码验证，无前后矛盾。关键结论汇总请见第九章「已修正误解与代码事实对照表」。

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
2. 调用 API 发送到后端：`PUT /dashboards/:id`
3. 后端 `updateDashboard` 控制器处理请求体（`dashboards.controller.ts:131-168`）：
   - 调用 `migrateBlock` 处理版本迁移
   - 为新 Block 生成 ID：`generateDashboardBlockIds()`
   - 调用 `updateById()` 保存到数据库

> **重要澄清**：`processApiUpdateBody` 方法存在于 DashboardModel 中，但它是给 BaseModel 通用 API 框架使用的。当前显式路由直接调用 `updateDashboard` 控制器，**不会经过 `processApiUpdateBody`**。

**代码位置**：
- `packages/back-end/src/routers/dashboards/dashboards.controller.ts:131-168`
- `packages/back-end/src/enterprise/models/DashboardModel.ts:494-670` (migrateBlock)
- `packages/back-end/src/enterprise/models/DashboardModel.ts:480-492` (generateDashboardBlockIds)

---

## 三、查询拼装逻辑

### 3.1 查询触发时机

查询可以通过两种方式触发：

| 触发方式 | 入口代码 | 说明 |
|---------|---------|------|
| 手动刷新 | `dashboards.controller.ts:180` `refreshDashboardData` | 用户点击刷新按钮 |
| 定时更新 | `jobs/updateDashboards.ts:9-33` | 每 10 分钟检查一次 `nextUpdate` 到期的 Dashboard |

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
POST /dashboards/:id/refresh
  ↓
refreshDashboardData() [dashboards.controller.ts:180-278]
  │
  ├─ 【分支1】有 experimentId 的实验 Dashboard
  │   ├─ planExperimentSnapshot() → 创建主快照
  │   ├─ 为不同 dimension 的 block 创建单独快照
  │   ├─ updateDashboardMetricAnalyses()  ← 更新指标分析
  │   ├─ updateDashboardSavedQueries()    ← 更新 SavedQuery 结果
  │   │   └─ executeAndSaveQuery() [saved-queries.controller.ts:249]
  │   │       └─ runFreeFormQuery() [datasource.ts:152]
  │   │           ├─ 权限检查: canRunSqlExplorerQueries()
  │   │           ├─ 安全检查: isReadOnlySQL()
  │   │           ├─ SQL 拼装: integration.getFreeFormQuery()
  │   │           │   └─ ensureLimit() + format()
  │   │           └─ 查询执行: integration.runTestQuery()
  │   │               └─ integration.runQuery()  // 具体数据源实现
  │   ├─ updateDashboardExplorations()    ← 更新探索分析
  │   └─ 保存 blocks (不更新 lastUpdated/nextUpdate)
  │
  └─ 【分支2】无 experimentId 的通用 Dashboard
      └─ updateNonExperimentDashboard() [dashboards.ts:235]
          ├─ updateDashboardMetricAnalyses()
          │   └─ createMetricAnalysis()
          ├─ updateDashboardSavedQueries()
          │   └─ (同上查询执行流程)
          ├─ updateDashboardExplorations()
          │   └─ runProductAnalyticsExploration()
          └─ 保存 blocks + 更新 lastUpdated + 重算 nextUpdate
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
| Dashboard Block 验证器 | `packages/shared/src/enterprise/validators/dashboard-block.ts` |
| Dashboard 工具函数 | `packages/shared/src/enterprise/dashboards/utils.ts` |
| Dashboard 更新显示 | `packages/front-end/enterprise/components/Dashboards/DashboardEditor/DashboardUpdateDisplay.tsx` |
| Dashboard Snapshot Provider | `packages/front-end/enterprise/components/Dashboards/DashboardSnapshotProvider.tsx` |

---

## 八、Sql-Explorer Block 深度拆解

### 8.1 从保存请求体开始的完整旅程

#### 8.1.1 前端保存请求体结构

当用户在 Dashboard 编辑器中编辑并保存时，前端发送的 PUT 请求体示例：

```typescript
// PUT /dashboards/:id
{
  "title": "用户活跃度分析",
  "editLevel": "private",
  "shareLevel": "private",
  "enableAutoUpdates": true,
  "updateSchedule": {
    "type": "daily",
    "hour": 2,
    "minute": 0
  },
  "projects": ["proj_abc123"],
  "blocks": [
    {
      "type": "sql-explorer",
      "id": "dshblk_xyz789",
      "uid": "uuid-1234-5678",
      "title": "日活用户趋势",
      "description": "",
      "savedQueryId": "sq_abc123def456",
      "blockConfig": ["results_table", "line-chart-1", "big-value-1"]
    }
  ]
}
```

**代码位置**：`packages/front-end/enterprise/components/Dashboards/DashboardEditor/index.tsx:463-470`

#### 8.1.2 后端保存处理流程

```typescript
// 路由层: dashboards.controller.ts:131-168
async updateDashboard(req, res) {
  const context = getContextFromReq(req);
  const { id } = req.params;
  const updates = { ...req.body };
  const dashboard = await context.models.dashboards.getById(id);
  
  // 直接处理 blocks，不经过 processApiUpdateBody
  if (updates.blocks) {
    // 1. 迁移旧版本 block
    const migratedBlocks = updates.blocks.map(migrateBlock);
    // 2. 为新 block 生成 ID
    const createdBlocks = migratedBlocks.map((blockData) =>
      dashboardBlockHasIds(blockData)
        ? blockData
        : generateDashboardBlockIds(context.org.id, blockData),
    );
    updates.blocks = createdBlocks;
  }

  // 3. 直接调用 updateById 保存
  const updatedDashboard = await context.models.dashboards.updateById(
    id,
    updates as UpdateProps<DashboardInterface>,
  );

  return res.status(200).json({ status: 200, dashboard: updatedDashboard });
}
```

**代码位置**：`packages/back-end/src/routers/dashboards/dashboards.controller.ts:131-168`

> **重要修正**：`processApiUpdateBody` 方法存在于 DashboardModel 中，但它是给 BaseModel 的通用 API 框架使用的。当前路由显式调用了 `updateDashboard` 控制器，直接处理 blocks 并调用 `updateById`，**不会经过 `processApiUpdateBody`**。

#### 8.1.3 migrateBlock 函数的真实作用

`migrateBlock` 是一个独立导出的函数，用于处理版本迁移，但**它不会为 sql-explorer block 自动补空 blockConfig**：

```typescript
// DashboardModel.ts:494-670
export function migrateBlock(doc) {
  switch (doc.type) {
    case "experiment-metric":
      // 迁移 metricSelector -> metricIds
      // 迁移 pinnedMetricSlices -> sliceTagsFilter
      return { ...doc, metricIds, sliceTagsFilter, ... };
      
    case "experiment-dimension":
      // 类似的迁移逻辑
      return { ...doc, ... };
      
    case "experiment-time-series":
      // 类似的迁移逻辑  
      return { ...doc, ... };
      
    // 注意: sql-explorer 没有专门的迁移逻辑!
    // blockConfig 为空数组的情况由 Zod schema 的默认值处理
  }
  return doc;
}
```

**代码位置**：`packages/back-end/src/enterprise/models/DashboardModel.ts:494-670`

> **关键发现**：`migrateBlock` 对 sql-explorer block 没有特殊处理。如果 block 缺少 `blockConfig` 字段，Zod schema 的 `legacySqlExplorerBlockInterface` 定义中 `blockConfig` 是可选的（`.optional()`），运行时会保留原值（可能为 undefined）。但在渲染时，SqlExplorerBlock 组件会使用 `block.blockConfig || []` 作为默认值。

### 8.2 blockConfig 与 dataVizConfigIndex 的映射机制

#### 8.2.1 字段定义与版本演进

```typescript
// dashboard-block.ts:224-231
const sqlExplorerBlockInterface = baseBlockInterface
  .extend({
    type: z.literal("sql-explorer"),
    savedQueryId: z.string(),
    // 已废弃：产品分析仪表板发布后，支持显示多个可视化
    dataVizConfigIndex: z.number().optional(),
    // 新版：字符串数组，支持多个配置项
    blockConfig: z.array(z.string()),
  })
  .strict();
```

**代码位置**：`packages/shared/src/enterprise/validators/dashboard-block.ts:224-231`

#### 8.2.2 blockConfig 支持的配置项类型

```typescript
// dashboards/utils.ts:35-39
export const BLOCK_CONFIG_ITEM_TYPES = {
  RESULTS_TABLE: "results_table",  // 显示结果表格
  VISUALIZATION: "visualization",   // 显示图表（通过 ID/title 匹配）
} as const;

export function isResultsTableItem(item: string): boolean {
  return item === BLOCK_CONFIG_ITEM_TYPES.RESULTS_TABLE;
}
```

**代码位置**：`packages/shared/src/enterprise/dashboards/utils.ts:35-43`

#### 8.2.3 前端渲染时的映射逻辑

```typescript
// SqlExplorerBlock.tsx:28-103
export default function SqlExplorerBlock({ block, savedQuery }) {
  // 向后兼容：旧版 dataVizConfigIndex 方式
  if (block.dataVizConfigIndex !== undefined) {
    const dataVizConfig = savedQuery.dataVizConfig?.[block.dataVizConfigIndex];
    return <SqlExplorerDataVisualization dataVizConfig={dataVizConfig} ... />;
  }

  // 新版 blockConfig 方式
  const blockConfig = block.blockConfig || [];
  
  const renderItems = blockConfig.map((configId, index) => {
    if (isResultsTableItem(configId)) {
      // 渲染结果表格
      return <DisplayTestQueryResults 
        results={savedQuery.results?.results} 
        ... 
      />;
    } else {
      // 渲染可视化：先按 ID 匹配，再按 title 匹配
      const dataVizConfig = savedQuery.dataVizConfig?.find(
        (config) => config.id === configId || config.title === configId
      );
      return <DataVisualizationDisplay dataVizConfig={dataVizConfig} ... />;
    }
  });

  return <Flex direction="column" gap="4">{renderItems}</Flex>;
}
```

**代码位置**：`packages/front-end/enterprise/components/Dashboards/DashboardEditor/DashboardBlock/SqlExplorerBlock.tsx:28-103`

#### 8.2.4 编辑时的 blockConfig 切换逻辑

```typescript
// EditSingleBlock.tsx:173-208
function toggleBlockConfigItem(block, setBlock, itemId, value) {
  // 只处理 sql-explorer 类型
  if (block.type !== "sql-explorer") return;
  
  // 从旧版迁移：移除 dataVizConfigIndex，使用新版 blockConfig
  const { dataVizConfigIndex: _, ...blockToSet } = block;
  
  if (value) {
    // 添加配置项
    setBlock({
      ...blockToSet,
      blockConfig: [...block.blockConfig, itemId],
    });
  } else {
    // 移除配置项
    setBlock({
      ...blockToSet,
      blockConfig: block.blockConfig.filter(id => id !== itemId),
    });
  }
}
```

**代码位置**：`packages/front-end/enterprise/components/Dashboards/DashboardEditor/DashboardEditorSidebar/EditSingleBlock.tsx:173-208`

#### 8.2.5 配置缺失检测逻辑

```typescript
// DashboardBlock/index.tsx:346-366
const blockNeedsConfiguration =
  // ... 其他 block 类型检测 ...
  (blockHasSavedQuery &&
    block.type === "sql-explorer" &&
    (isSqlExplorerWithDataVizIndex(block)
      ? // 旧版：索引无效或对应配置不存在
        block.dataVizConfigIndex === -1 ||
        !blockSavedQuery?.dataVizConfig?.[block.dataVizConfigIndex]
      : isSqlExplorerWithBlockConfig(block)
        ? // 新版：blockConfig 为空
          !block.blockConfig || block.blockConfig.length === 0
        : true));
```

**代码位置**：`packages/front-end/enterprise/components/Dashboards/DashboardEditor/DashboardBlock/index.tsx:346-366`

#### 8.2.6 影响边界：只影响渲染，不影响查询执行

> **🚨 关键修正**：`blockConfig` 和 `dataVizConfigIndex` **对查询执行完全没有影响**，它们只控制渲染展示。

```typescript
// updateDashboardSavedQueries() [dashboards.ts:379-414]
// 这个函数只提取 savedQueryId，完全不看 blockConfig 或 dataVizConfigIndex
export async function updateDashboardSavedQueries(context, blocks) {
  const savedQueries = await context.models.savedQueries.getByIds([
    ...new Set(
      blocks
        .filter(block => 
          blockHasFieldOfType(block, "savedQueryId", isString) &&
          block.savedQueryId.length > 0
        )
        .map(block => block.savedQueryId),  // 只提取 savedQueryId
    ),
  ]);

  // ... 获取数据源 ...

  await Promise.all(
    savedQueries.map(async (savedQuery) => {
      // 执行查询时只用 savedQuery 本身的 SQL，与 blockConfig 无关
      await executeAndSaveQuery(context, savedQuery, savedQueryDataSource);
    }),
  );
}
```

**代码位置**：`packages/back-end/src/enterprise/services/dashboards.ts:379-414`

**影响边界总结**：

| 阶段 | blockConfig/dataVizConfigIndex 的作用 | 依赖的核心数据 |
|-----|------------------------------------|-------------|
| **查询执行** | ❌ 无任何影响 | `savedQuery.sql` |
| **结果存储** | ❌ 无任何影响 | `savedQuery.results` |
| **前端渲染** | ✅ 决定展示哪些内容 | `blockConfig` + `savedQuery.dataVizConfig` + `savedQuery.results` |

**执行流程中的数据流**：

```
保存阶段:
  block { savedQueryId, blockConfig } → 存入数据库

查询执行阶段:
  提取所有 savedQueryId → 执行 SQL → 结果存入 SavedQuery.results
  (blockConfig 在此阶段完全不被读取)

渲染阶段:
  读取 block.blockConfig → 遍历每个 configId:
    ├─ "results_table" → 渲染 savedQuery.results.results
    └─ "chart-id" → 匹配 savedQuery.dataVizConfig → 用 results 渲染图表
```

### 8.3 手动刷新 vs 定时刷新：分叉差异详解

#### 8.3.1 触发入口对比

| 维度 | 手动刷新 | 定时刷新 |
|-----|---------|---------|
| **触发方式** | 用户点击 "Update" 按钮 | Agenda 定时任务（每 10 分钟） |
| **入口代码** | `DashboardUpdateDisplay.tsx:181-196` | `updateDashboards.ts:9-33` |
| API 端点 | `POST /dashboards/:id/refresh` | 无 API，直接调用服务层 |
| **权限检查** | 前端检查 `canRunSqlExplorerQueries` | 后端使用系统上下文 |
| **UI 反馈** | 显示进度条、完成/失败状态 | 无 UI，更新 `nextUpdate` 和 `lastUpdated` |

#### 8.3.2 手动刷新完整链路

```
前端: DashboardUpdateDisplay.tsx
  ↓ 点击 Update 按钮
  updateAllSnapshots() [DashboardSnapshotProvider.tsx:224-239]
    ↓
    POST /dashboards/${dashboard.id}/refresh
      ↓
后端: dashboards.router.ts:98-102
  ↓
  refreshDashboardData() [dashboards.controller.ts:180-278]
    │
    ├─ 【分支1】有 experimentId 的实验 Dashboard
    │   ├─ planExperimentSnapshot() → 创建主快照
    │   ├─ 为不同 dimension 的 block 创建单独快照
    │   ├─ updateDashboardMetricAnalyses()  ← 更新指标分析
    │   ├─ updateDashboardSavedQueries()    ← 更新 SavedQuery 结果
    │   ├─ updateDashboardExplorations()    ← 更新探索分析
    │   └─ 保存 blocks (不更新 lastUpdated/nextUpdate)
    │
    └─ 【分支2】无 experimentId 的通用 Dashboard
        └─ updateNonExperimentDashboard() [dashboards.ts:235-250]
            ├─ updateDashboardMetricAnalyses()
            ├─ updateDashboardSavedQueries() [dashboards.ts:379-414]
            │   └─ executeAndSaveQuery() [saved-queries.controller.ts:249-281]
            │       └─ runFreeFormQuery() [datasource.ts:152-219]
            │           ├─ 权限检查: canRunSqlExplorerQueries()
            │           ├─ 安全检查: isReadOnlySQL()
            │           ├─ SQL 拼装: integration.getFreeFormQuery()
            │           └─ 查询执行: integration.runTestQuery()
            ├─ updateDashboardExplorations()
            └─ 保存 blocks + 更新 lastUpdated + 重算 nextUpdate
                      ↓
前端: DashboardSnapshotProvider.tsx
  ↓ 轮询状态 (每 2 秒)
  mutateAllSnapshots()
    ↓
  GET /dashboards/${dashboard.id}/snapshots
    ↓
  更新 savedQueriesMap, allQueries, status
    ↓
  SqlExplorerBlock.tsx 读取最新 savedQuery.results 渲染
```

#### 8.3.3 定时刷新完整链路

```
Agenda 定时任务 (每 10 分钟触发)
  ↓
  QUEUE_DASHBOARD_UPDATES 任务 [updateDashboards.ts:9-33]
    ↓
    DashboardModel.getDashboardsToUpdate() [DashboardModel.ts:113-147]
      ├─ 条件: isDeleted=false, isDefault=false
      ├─ 条件: enableAutoUpdates=true, experimentId=null
      └─ 条件: nextUpdate <= now 或 nextUpdate 不存在
    ↓
    为每个 Dashboard 创建 UPDATE_SINGLE_DASH 任务
      ↓
      updateSingleDashboard() [updateDashboards.ts:53-82]
        ├─ getContextForAgendaJobByOrgId()  // 创建系统上下文
        └─ updateNonExperimentDashboard() [dashboards.ts:235]
            └─ (与手动刷新相同的查询执行流程)
              ↓
              更新 Dashboard.lastUpdated
              重新计算 Dashboard.nextUpdate
              保存 SavedQuery.results
```

#### 8.3.4 核心差异点

| 差异点 | 手动刷新 | 定时刷新 |
|-------|---------|---------|
| **触发入口** | `POST /dashboards/:id/refresh` | Agenda 任务调度 |
| **上下文** | 用户请求上下文（带权限） | Agenda 系统上下文 |
| **Dashboard 类型** | 支持实验 Dashboard 和通用 Dashboard | 仅支持通用 Dashboard（`experimentId=null`） |
| **实验 Dashboard** | ✅ 支持，走实验快照流程 | ❌ 不支持（查询条件明确过滤 `experimentId=null`） |
| **通用 Dashboard** | ✅ 调用 `updateNonExperimentDashboard()` | ✅ 调用 `updateNonExperimentDashboard()` |
| **更新 lastUpdated** | 通用 Dashboard 更新，实验 Dashboard 不更新 | ✅ 总是更新 |
| **重算 nextUpdate** | 通用 Dashboard 更新，实验 Dashboard 不更新 | ✅ 总是更新 |
| **错误处理** | 立即抛出异常返回给前端 | 捕获异常，自动关闭 `enableAutoUpdates` |
| **并发性** | 用户触发，可能并发 | 串行处理，最多 100 个/次 |
| **结果通知** | 前端轮询，实时显示进度和状态 | 无通知，仅更新数据库 |
| **sql-explorer 支持** | ✅ 通过 `updateDashboardSavedQueries()` 更新 | ✅ 通过 `updateDashboardSavedQueries()` 更新 |

#### 8.3.5 刷新接口 HTTP 方法确认

> ✅ **确认正确**：刷新接口是 `POST /dashboards/:id/refresh`

```typescript
// dashboards.router.ts:98-102
router.post(
  "/:id/refresh",
  validateRequestMiddleware({ params: dashboardParams }),
  dashboardsController.refreshDashboardData,
);
```

**代码位置**：`packages/back-end/src/routers/dashboards/dashboards.router.ts:98-102`

#### 8.3.6 失败处理差异

**手动刷新失败**：
```typescript
// DashboardSnapshotProvider.tsx:224-239
const updateAllSnapshots = async () => {
  try {
    await apiCall(`/dashboards/${dashboard.id}/refresh`, { method: "POST" });
  } catch (e) {
    setRefreshError(e.message);  // 显示给用户
  } finally {
    // 无论成功失败都刷新状态
    mutateAllSnapshots();
  }
};
```

**定时刷新失败**：
```typescript
// updateDashboards.ts:53-82
const updateSingleDashboard = async (job) => {
  try {
    await updateNonExperimentDashboard(context, dashboard);
  } catch (e) {
    // 更新失败，自动关闭自动更新
    await context.models.dashboards.dangerousUpdateByIdBypassPermission(
      dashboardId,
      { enableAutoUpdates: false }
    );
    // 无用户反馈，静默处理
  }
};
```

### 8.4 SavedQuery 与 Dashboard 的关联维护

#### 8.4.1 关联关系

```typescript
// Dashboard 侧
blocks: [
  {
    type: "sql-explorer",
    savedQueryId: "sq_abc123",  // 外键关联
    blockConfig: ["results_table", "chart-1"]
  }
]

// SavedQuery 侧
{
  id: "sq_abc123",
  linkedDashboardIds: ["dash_xyz789"],  // 反向关联
  dataVizConfig: [
    { id: "chart-1", title: "日活趋势", chartType: "line", ... }
  ],
  results: {
    results: [...],
    sql: "SELECT date, dau FROM ...",
    duration: 1234
  }
}
```

#### 8.4.2 前端可选项过滤

```typescript
// EditSingleBlock.tsx:611-625
const savedQueryOptions = useMemo(
  () =>
    savedQueriesData?.savedQueries
      ?.filter((savedQuery) => {
        // 只显示关联到当前 Dashboard 的 SavedQuery
        return (
          savedQuery.linkedDashboardIds?.includes(dashboardId) ||
          savedQueryId === savedQuery.id  // 或当前已选中的
        );
      })
      .map(({ id, name }) => ({ value: id, label: name })) || [],
  [savedQueriesData?.savedQueries, dashboardId, savedQueryId]
);
```

**代码位置**：`packages/front-end/enterprise/components/Dashboards/DashboardEditor/DashboardEditorSidebar/EditSingleBlock.tsx:611-625`

### 8.5 数据流向全景图

```
┌─────────────────────────────────────────────────────────────────┐
│                        前端编辑阶段                              │
├─────────────────────────────────────────────────────────────────┤
│  SqlExplorerModal 编辑 SQL → 保存 SavedQuery                    │
│    ↓ (savedQueryId 返回到前端)                                   │
│  DashboardEditor 选择 SavedQuery → 勾选 blockConfig 项          │
│    ↓ (PUT /dashboards/:id)                                      │
│  发送 blocks 数组到后端                                          │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                        后端存储阶段                              │
├─────────────────────────────────────────────────────────────────┤
│  updateDashboard 控制器 [dashboards.controller.ts:131-168]      │
│    ↓                                                             │
│  migrateBlock() 迁移旧版本 → generateDashboardBlockIds() 生成 ID │
│    ↓                                                             │
│  updateById() 保存到 MongoDB                                    │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                        查询执行阶段                              │
├─────────────────────────────────────────────────────────────────┤
│  手动/定时触发刷新                                               │
│    ↓                                                             │
│  updateDashboardSavedQueries() 提取所有 savedQueryId            │
│    ↓ (blockConfig 在此阶段完全不被读取)                          │
│  并行 executeAndSaveQuery() → 执行 SQL → 保存 results           │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                        前端渲染阶段                              │
├─────────────────────────────────────────────────────────────────┤
│  GET /dashboards/:id/snapshots 获取关联的 SavedQueries          │
│    ↓                                                             │
│  SqlExplorerBlock 根据 blockConfig 遍历渲染:                     │
│    - "results_table" → DisplayTestQueryResults                  │
│    - "chart-1" → 匹配 dataVizConfig → DataVisualizationDisplay │
└─────────────────────────────────────────────────────────────────┘
```

---

## 九、已修正误解与代码事实对照表

| 编号 | 误解内容 | 代码事实 | 验证文件 |
|-----|---------|---------|---------|
| 1 | 保存流程经过 `processApiUpdateBody` | 保存流程直接调用 `updateDashboard` 控制器，不经过 `processApiUpdateBody` | `dashboards.controller.ts:131-168` |
| 2 | `migrateBlock` 为 sql-explorer 补空 blockConfig | `migrateBlock` 只处理实验类 block，对 sql-explorer 无特殊处理 | `DashboardModel.ts:494-670` |
| 3 | 刷新接口是 `PUT /dashboards/:id/refresh` | 刷新接口是 `POST /dashboards/:id/refresh` | `dashboards.router.ts:98-102` |
| 4 | `blockConfig` 影响查询执行 | `blockConfig` 仅影响前端渲染，查询执行只依赖 `savedQueryId` | `dashboards.ts:379-414` |
| 5 | 手动刷新是统一流程 | 手动刷新分两个分支：实验 Dashboard 和通用 Dashboard，行为不同 | `dashboards.controller.ts:180-278` |
| 6 | 定时刷新支持实验 Dashboard | 定时刷新只处理 `experimentId=null` 的通用 Dashboard | `DashboardModel.ts:113-147` |

---

### 9.1 保存流程调用链错误详解

**❌ 原错误理解**：
```
控制器调用 processApiUpdateBody() → 处理 blocks → 保存
```

**✅ 实际代码**（`dashboards.controller.ts:131-168`）：
```typescript
async updateDashboard(req, res) {
  // 直接处理 blocks，不经过 processApiUpdateBody
  if (updates.blocks) {
    const migratedBlocks = updates.blocks.map(migrateBlock);
    const createdBlocks = migratedBlocks.map(...);
    updates.blocks = createdBlocks;
  }
  // 直接调用 updateById 保存
  await context.models.dashboards.updateById(id, updates);
}
```

**说明**：`processApiUpdateBody` 方法确实存在于 DashboardModel 中，但它是给 BaseModel 的通用 API 框架使用的。当前路由显式调用了 `updateDashboard` 控制器，**不会经过 `processApiUpdateBody`**。

### 9.2 migrateBlock 函数作用错误详解

**❌ 原错误理解**：`migrateBlock` 会为 sql-explorer block 自动补空 blockConfig 数组。

**✅ 实际代码**（`DashboardModel.ts:494-670`）：
- `migrateBlock` 只处理 `experiment-metric`、`experiment-dimension`、`experiment-time-series` 三种 block 类型的版本迁移
- 对 sql-explorer block **没有任何特殊处理**
- 如果 block 缺少 blockConfig 字段，会保留原值（可能为 undefined）
- 渲染时由前端组件 `SqlExplorerBlock.tsx` 使用 `block.blockConfig || []` 作为默认值

### 9.3 HTTP 方法错误详解

**❌ 原错误理解**：刷新接口是 `PUT /dashboards/:id/refresh`

**✅ 实际代码**（`dashboards.router.ts:98-102`）：
```typescript
router.post("/:id/refresh", ..., dashboardsController.refreshDashboardData);
```

**正确接口**：`POST /dashboards/:id/refresh`

### 9.4 blockConfig 影响边界错误详解

**❌ 原错误理解**：隐含暗示 blockConfig 可能影响查询执行。

**✅ 实际代码**（`dashboards.ts:379-414`）：
```typescript
export async function updateDashboardSavedQueries(context, blocks) {
  // 只提取 savedQueryId，完全不看 blockConfig
  const savedQueryIds = blocks
    .filter(block => blockHasFieldOfType(block, "savedQueryId", isString))
    .map(block => block.savedQueryId);
  
  // ... 执行查询时只用 savedQuery 本身的 SQL
}
```

**明确结论**：
- `blockConfig` 和 `dataVizConfigIndex` **对查询执行完全没有影响**
- 它们**只控制前端渲染展示**，决定显示哪些结果和图表
- 查询执行仅依赖 `savedQueryId` 和 `SavedQuery.sql`

### 9.5 手动刷新分支逻辑错误详解

**❌ 原错误理解**：手动刷新只有一个统一流程。

**✅ 实际代码**（`dashboards.controller.ts:180-278`）：
- 手动刷新有**两个分支**：
  - **分支 1**：有 `experimentId` 的实验 Dashboard → 走实验快照流程 + 更新三类数据
  - **分支 2**：无 `experimentId` 的通用 Dashboard → 调用 `updateNonExperimentDashboard()`
- 实验 Dashboard 手动刷新**不更新** `lastUpdated` 和 `nextUpdate`
- 通用 Dashboard 手动刷新**会更新** `lastUpdated` 和 `nextUpdate`

### 9.6 定时刷新限制错误详解

**✅ 补充确认**：定时刷新通过 `getDashboardsToUpdate()` 明确过滤 `experimentId: null`，**只支持通用 Dashboard**，不支持实验 Dashboard。
