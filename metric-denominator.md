# 分母校验与护栏机制配合流程分析

## 1. 分母定义 (Denominator Definition)

### 1.1 Metric 中的分母配置

在 `packages/shared/types/metric.d.ts:63` 中，`MetricInterface` 定义了分母字段：

```typescript
export interface MetricInterface {
  // ...
  denominator?: string;  // 分母度量的 ID
  // ...
}
```

在 `packages/shared/src/validators/metrics.ts:117` 中，API 验证器定义了 `denominatorMetricId`：

```typescript
sql: z
  .object({
    identifierTypes: z.array(z.string()),
    conversionSQL: z.string(),
    userAggregationSQL: z.string(),
    denominatorMetricId: z.string(),  // 分母度量 ID
  })
  .optional(),
```

### 1.2 分母度量的用途

分母度量主要用于：
- **比率度量 (Ratio Metrics)**：如 "每个用户的平均收入" (ARPU)
- **漏斗度量 (Funnel Metrics)**：计算转化率
- **标准化指标**：将分子数据按分母标准化，消除样本量偏差

### 1.3 Safe Rollout 中的分母处理

在 `packages/back-end/src/services/safeRolloutSnapshots.ts:210-215` 中，分母度量被特殊处理：

```typescript
const denominatorMetrics = allExperimentMetrics
  .filter((m) => m && !isFactMetric(m) && m.denominator)
  .map((m: ExperimentMetricInterface) =>
    metricMap.get(m.denominator as string),
  )
  .filter(Boolean) as MetricInterface[];
```

在 `getMetricForSafeRolloutSnapshot` 函数 (第89行) 中，分母被传递到度量设置中：

```typescript
denominator: (!isFactMetric(metric) && metric.denominator) || undefined,
```

## 2. 护栏阈值设置 (Guardrail Threshold Settings)

### 2.1 核心阈值常量

在 `packages/shared/src/constants.ts` 中定义了默认阈值：

| 常量名称 | 默认值 | 说明 |
|---------|-------|------|
| `DEFAULT_SRM_THRESHOLD` | 0.001 | SRM (样本比率不匹配) p-value 阈值 |
| `DEFAULT_SRM_MINIMINUM_COUNT_PER_VARIATION` | 8 | 每个变体最小用户数 |
| `DEFAULT_MULTIPLE_EXPOSURES_THRESHOLD` | 0.01 | 多重暴露百分比阈值 (1%) |
| `DEFAULT_MULTIPLE_EXPOSURES_ENOUGH_DATA_THRESHOLD` | 10 | 多重暴露数据量阈值 |
| `DEFAULT_P_VALUE_THRESHOLD` | 0.05 | 统计显著性 p-value 阈值 |
| `DEFAULT_GUARDRAIL_ALPHA` | 0.05 | 护栏度量显著性水平 |

### 2.2 组织级别设置

在 `packages/shared/src/enterprise/decision-criteria/decisionCriteria.ts:245-261` 中，`getHealthSettings` 函数合并组织设置与默认值：

```typescript
export function getHealthSettings(
  settings?: OrganizationSettings,
  hasDecisionFramework?: boolean,
): ExperimentHealthSettings {
  return {
    decisionFrameworkEnabled:
      (settings?.decisionFrameworkEnabled ?? DEFAULT_DECISION_FRAMEWORK_ENABLED) &&
      !!hasDecisionFramework,
    experimentMinLengthDays:
      settings?.experimentMinLengthDays ?? DEFAULT_EXPERIMENT_MIN_LENGTH_DAYS,
    srmThreshold: settings?.srmThreshold ?? DEFAULT_SRM_THRESHOLD,
    multipleExposureMinPercent:
      settings?.multipleExposureMinPercent ?? DEFAULT_MULTIPLE_EXPOSURES_THRESHOLD,
  };
}
```

### 2.3 度量级别阈值

在 `packages/shared/types/metric.d.ts:74-79` 中，每个度量可以配置独立的阈值：

```typescript
export interface MetricInterface {
  // ...
  winRisk?: number;           // 获胜风险阈值
  loseRisk?: number;          // 失败风险阈值
  maxPercentChange?: number;  // 最大变化百分比阈值
  minPercentChange?: number;  // 最小变化百分比阈值
  minSampleSize?: number;     // 最小样本量
  targetMDE?: number;         // 目标最小可检测效应
  // ...
}
```

## 3. 异常拦截机制 (Anomaly Interception Mechanism)

### 3.1 SRM (Sample Ratio Mismatch) 样本比率不匹配检测

**核心逻辑** 在 `packages/shared/src/health/health.ts:74-96`：

```typescript
export function getSRMHealthData({
  srm,                    // 计算出的 SRM p-value
  numOfVariations,        // 变体数量
  totalUsersCount,        // 总用户数
  srmThreshold,           // SRM 阈值 (默认 0.001)
  minUsersPerVariation,   // 每个变体最小用户数
}: {
  srm: number;
  numOfVariations: number;
  totalUsersCount: number;
  srmThreshold: number;
  minUsersPerVariation: number;
}): SRMHealthStatus {
  const minUsersCount = numOfVariations * minUsersPerVariation;

  if (totalUsersCount < minUsersCount) {
    return "not-enough-traffic";  // 流量不足，不做判断
  } else if (srm < srmThreshold) {
    return "unhealthy";           // p-value < 阈值，标记为不健康
  } else {
    return "healthy";             // 健康
  }
}
```

**SRM 值获取** 在 `packages/shared/src/health/health.ts:98-155`：
- 从健康查询结果中获取 `snapshot.health?.traffic?.overall?.srm`
- 如无健康查询结果，回退到主分析结果 `snapshot.analyses?.[0]?.results?.[0]?.srm`

### 3.2 多重暴露检测

**核心逻辑** 在 `packages/shared/src/health/health.ts:25-62`：

```typescript
export function getMultipleExposureHealthData({
  multipleExposuresCount,    // 多重暴露用户数
  totalUsersCount,           // 总用户数
  minCountThreshold,         // 最小数据量阈值 (默认 10)
  minPercentThreshold,       // 最小百分比阈值 (默认 0.01)
}: {
  multipleExposuresCount: number;
  totalUsersCount: number;
  minCountThreshold: number;
  minPercentThreshold: number;
}): MultipleExposureHealthData {
  const multipleExposureDecimal = multipleExposuresCount / totalUsersCount;
  const hasEnoughData = totalUsersCount >= minCountThreshold;
  const isUnhealthy = multipleExposureDecimal >= minPercentThreshold;

  // 返回状态: not-enough-traffic / unhealthy / healthy
}
```

### 3.3 护栏度量异常拦截

在 `packages/shared/src/enterprise/decision-criteria/decisionCriteria.ts:648-665` 中，Safe Rollout 使用专门的决策标准：

```typescript
const ROLLBACK_SAFE_ROLLOUT_DECISION_CRITERIA: DecisionCriteriaData = {
  id: "gbdeccrit_rollback_safe_rollout",
  name: "Rollback Safe Rollout",
  rules: [
    {
      conditions: [
        {
          match: "any",              // 任意一个护栏度量满足条件
          metrics: "guardrails",     // 检查护栏度量
          direction: "statsigLoser", // 状态为 "lost" (统计显著下降)
        },
      ],
      action: "rollback",           // 执行回滚操作
    },
  ],
  defaultAction: "review",
};
```

**状态评估逻辑** 在 `evaluateDecisionRuleOnVariation` 函数 (第40-108行)：
- 遍历所有护栏度量
- 检查度量状态是否为 "lost" (显著下降)
- 如果任意护栏度量状态为 "lost"，触发回滚

## 4. 整体配合流程 (Overall Coordination Flow)

### 4.1 Safe Rollout 健康评估流程

**入口函数** `getSafeRolloutResultStatus` (`decisionCriteria.ts:597-717`)

```
流程图示:

1. 数据收集阶段
   ↓
2. SRM 检测
   ├─ 流量检查: totalUsers >= 2 * 8 (每个变体至少8个用户)
   ├─ SRM p-value 计算
   └─ 判断: srm < 0.001 ?  unhealthy : healthy
   ↓
3. 多重暴露检测
   ├─ 计算: multipleExposures / totalUsers
   ├─ 流量检查: totalUsers >= 10
   └─ 判断: 比例 >= 0.01 ?  unhealthy : healthy
   ↓
4. 护栏度量评估
   ├─ 对每个护栏度量进行统计检验
   ├─ 使用顺序检验 (Sequential Testing)
   └─ 检查度量状态是否为 "lost" (显著下降)
   ↓
5. 决策阶段
   ├─ 如有 SRM 或 多重暴露异常 → status: unhealthy
   ├─ 如任意护栏度量显著下降 → status: rollback-now
   ├─ 如监测期未结束 → status: days-left
   └─ 如监测期结束且无异常 → status: ship-now
```

### 4.2 关键决策点详解

#### 4.2.1 "unhealthy" 状态触发条件

在 `decisionCriteria.ts:678-683`：

```typescript
if (unhealthyData.srm || unhealthyData.multipleExposures) {
  return {
    status: "unhealthy",
    unhealthyData,
  };
}
```

**触发条件**：
- SRM p-value < 0.001 (样本分配严重不均)
- 或 多重暴露用户占比 >= 1%

**影响**：标记为不健康，可能触发人工审查

#### 4.2.2 "rollback-now" 状态触发条件

在 `decisionCriteria.ts:686-693`：

```typescript
if (decisionStatus?.status === "rollback-now") {
  return {
    status: "rollback-now",
    variations: decisionStatus.variations,
    sequentialUsed: true,
    powerReached: false,
  };
}
```

**触发条件**：
- 任意护栏度量统计显著下降 (status === "lost")
- 使用顺序检验 (Sequential Testing) 确保结果可靠性

**影响**：如果开启了 Auto Rollback，系统会自动回滚发布

#### 4.2.3 "ship-now" 状态触发条件

在 `decisionCriteria.ts:703-716`：

```typescript
if (daysLeft <= 0 && resultsStatus) {
  return {
    status: "ship-now",
    variations: [
      {
        variationId: "1",
        decidingRule: null,
      },
    ],
    sequentialUsed: true,
    powerReached: false,
  };
}
```

**触发条件**：
- 监测期已结束 (daysLeft <= 0)
- 无 SRM 异常
- 无多重暴露异常
- 无护栏度量显著下降

**影响**：建议全量发布

### 4.3 自动回滚机制

在 `packages/back-end/src/services/safeRolloutSnapshots.ts:683-700` 中，通知系统会根据状态触发事件：

```typescript
if (safeRolloutStatus?.status === "rollback-now") {
  const rollbackNotificationSent = await memoizeSafeRolloutNotification({
    context,
    types: ["rollback"],
    safeRollout: updatedSafeRollout,
    dispatch: () =>
      dispatchSafeRolloutEvent({
        context,
        feature,
        environment: notificationData.environment,
        event: "saferollout.rollback",
        data: { object: notificationData },
      }),
  });
  notificationSent = notificationSent || rollbackNotificationSent;
}
```

**Auto Rollback 配置**（前端 `SafeRolloutFields.tsx:472-481`）：
```typescript
<Checkbox
  id="autoRollback"
  value={form.watch("safeRolloutFields.autoRollback")}
  setValue={(v) => form.setValue("safeRolloutFields.autoRollback", v)}
  label="Auto Rollback"
  description="Automatically rollback when unhealthy or a guardrail fails"
/>
```

## 5. 数据流转图示

```
+---------------------+     +---------------------+     +---------------------+
|  分母度量定义       |     |  护栏度量选择       |     |  阈值配置           |
|  (Metric.denominator)|     |  (guardrailMetricIds)|     |  (组织/度量级别)    |
+----------+----------+     +----------+----------+     +----------+----------+
           |                           |                           |
           v                           v                           v
+-----------------------------------------------------------------------------+
|                                                                             |
|                    Safe Rollout Snapshot 分析流程                           |
|                                                                             |
|  +----------------+   +----------------+   +----------------+               |
|  |  数据查询      |   |  SRM 计算      |   |  多重暴露计算   |               |
|  |  (SQL Query)   |-->|  (Chi-square)  |-->|  (用户去重)     |               |
|  +----------------+   +----------------+   +----------------+               |
|                                    |                                         |
|                                    v                                         |
|  +----------------+   +----------------+   +----------------+               |
|  |  护栏度量分析  |<--+  分母标准化    |   |  健康状态评估   |               |
|  |  (统计检验)    |   |  (比率计算)    |   |  (阈值比较)     |               |
|  +--------+-------+   +----------------+   +--------+-------+               |
|           |                                    |                           |
|           v                                    v                           |
|  +----------------+                  +----------------+                    |
|  |  度量状态判断  |                  |  最终决策      |                    |
|  |  (won/lost/...)|----------------->|  (rollback/ship)|                    |
|  +----------------+                  +----------------+                    |
+-----------------------------------------------------------------------------+
                                    |
                                    v
                          +-------------------+
                          |  事件通知/自动操作 |
                          |  (Event Bus)       |
                          +-------------------+
```

## 6. 关键文件索引

| 文件路径 | 核心功能 |
|---------|---------|
| `packages/shared/types/metric.d.ts` | Metric 类型定义，包含 denominator 字段 |
| `packages/shared/src/constants.ts` | 阈值常量定义 |
| `packages/shared/src/health/health.ts` | SRM 和多重暴露健康检测 |
| `packages/shared/src/enterprise/decision-criteria/decisionCriteria.ts` | 决策框架与护栏评估逻辑 |
| `packages/back-end/src/services/safeRolloutSnapshots.ts` | Safe Rollout 快照服务 |
| `packages/front-end/components/Features/RuleModal/SafeRolloutFields.tsx` | Safe Rollout 配置表单 |
