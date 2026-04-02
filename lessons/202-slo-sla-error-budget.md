# 202 - Agent SLO/SLA 管理与错误预算（Error Budget & SLO Management）

> **一句话总结**：SLO 是你对用户的承诺，Error Budget 是你敢创新的底气——用光了就该老老实实修稳定性，还有剩就大胆上新功能。

---

## 为什么 Agent 需要 SLO？

普通 Web 服务的 SLO 很直接：HTTP 200 率 > 99.9%，P99 延迟 < 500ms。

但 **Agent 的 SLO 更复杂**：

| 维度 | Web 服务 | Agent 系统 |
|------|---------|-----------|
| 成功定义 | HTTP 2xx | 任务完成？工具调用成功？用户满意？ |
| 延迟测量 | 单次请求 | 多轮对话总耗时？单步工具调用？ |
| 错误分类 | 5xx | LLM 幻觉？工具失败？任务漂移？ |
| 依赖链 | DB/缓存 | LLM API + N 个外部工具 |

---

## 核心概念

### SLI（Service Level Indicator）—— 你在测什么

```
SLI = good_events / total_events
```

Agent 常用 SLI：

```typescript
// 1. 任务完成率
taskCompletionRate = successfulTasks / totalTasks

// 2. 工具调用成功率
toolSuccessRate = successfulToolCalls / totalToolCalls

// 3. P95 端到端延迟
p95Latency = percentile(taskDurations, 95)

// 4. 幻觉率（需要 LLM 评分）
hallucinationRate = detectedHallucinations / totalOutputs

// 5. 用户满意度（负反馈率）
dissatisfactionRate = negativeFeedbacks / totalSessions
```

### SLO（Service Level Objective）—— 你的承诺

```
SLO = SLI 目标值，例如 99.5% 任务完成率 over 30天滑动窗口
```

### Error Budget —— 你还能出多少故障

```
Error Budget = 1 - SLO
// 99.5% SLO → 0.5% 错误预算
// 30天内 = 30 * 24 * 60 * 0.005 = 216 分钟可以出故障
```

---

## 实战实现

### Step 1：SLI 数据收集中间件

```typescript
// slo-middleware.ts
interface SLIEvent {
  timestamp: number;
  taskId: string;
  type: 'task' | 'tool_call' | 'llm_call';
  success: boolean;
  durationMs: number;
  errorType?: 'tool_fail' | 'llm_error' | 'timeout' | 'hallucination';
  metadata?: Record<string, unknown>;
}

class SLICollector {
  private events: SLIEvent[] = [];
  private redis: RedisClient;

  async recordEvent(event: SLIEvent): Promise<void> {
    // 写入 Redis Sorted Set，score = timestamp（便于时间窗口查询）
    await this.redis.zadd(
      'sli:events',
      event.timestamp,
      JSON.stringify(event)
    );
    
    // 实时更新计数器（高性能路径）
    const minute = Math.floor(event.timestamp / 60000);
    const key = `sli:${event.type}:${minute}`;
    await this.redis.hincrby(key, event.success ? 'good' : 'bad', 1);
    await this.redis.expire(key, 86400 * 31); // 保留 31 天
  }
}

// 工具调用中间件
function sloToolMiddleware(collector: SLICollector) {
  return async (toolName: string, params: unknown, next: () => Promise<unknown>) => {
    const start = Date.now();
    let success = false;
    let errorType: SLIEvent['errorType'];
    
    try {
      const result = await next();
      success = true;
      return result;
    } catch (err) {
      errorType = classifyError(err);
      throw err;
    } finally {
      await collector.recordEvent({
        timestamp: Date.now(),
        taskId: AsyncLocalStorage.getStore()?.taskId ?? 'unknown',
        type: 'tool_call',
        success,
        durationMs: Date.now() - start,
        errorType,
        metadata: { toolName },
      });
    }
  };
}
```

### Step 2：Error Budget 计算器

```typescript
// error-budget.ts
interface SLOConfig {
  name: string;
  target: number;           // e.g. 0.995 for 99.5%
  windowDays: number;       // 滑动窗口天数
  sliType: 'task' | 'tool_call' | 'llm_call';
}

class ErrorBudgetManager {
  constructor(
    private redis: RedisClient,
    private slos: SLOConfig[]
  ) {}

  async getErrorBudget(slo: SLOConfig): Promise<{
    budgetTotal: number;    // 总错误预算（分钟）
    budgetUsed: number;     // 已消耗（分钟）
    budgetRemaining: number; // 剩余（分钟）
    burnRate: number;       // 消耗速率（> 1 表示超速）
    exhaustsAt?: Date;      // 预计耗尽时间
  }> {
    const windowMs = slo.windowDays * 24 * 60 * 60 * 1000;
    const now = Date.now();
    const since = now - windowMs;
    
    // 从 Redis Sorted Set 查询时间窗口内的事件
    const rawEvents = await this.redis.zrangebyscore(
      'sli:events',
      since,
      now
    );
    
    const events = rawEvents
      .map(e => JSON.parse(e) as SLIEvent)
      .filter(e => e.type === slo.sliType);
    
    const total = events.length;
    const good = events.filter(e => e.success).length;
    const bad = total - good;
    
    const actualSLI = total > 0 ? good / total : 1;
    const errorRate = 1 - actualSLI;
    
    // Error Budget 以时间计量更直观
    const windowMinutes = slo.windowDays * 24 * 60;
    const budgetTotal = windowMinutes * (1 - slo.target);
    const budgetUsed = windowMinutes * errorRate;
    const budgetRemaining = budgetTotal - budgetUsed;
    
    // 消耗速率：1.0 = 刚好按预算消耗，> 1 = 超速
    const burnRate = errorRate / (1 - slo.target);
    
    // 预计耗尽时间
    let exhaustsAt: Date | undefined;
    if (burnRate > 1 && budgetRemaining > 0) {
      const remainingMs = (budgetRemaining / (errorRate * windowMinutes)) * windowMs;
      exhaustsAt = new Date(now + remainingMs);
    }
    
    return {
      budgetTotal,
      budgetUsed,
      budgetRemaining,
      burnRate,
      exhaustsAt,
    };
  }

  async shouldFreezeDeploy(slo: SLOConfig): Promise<boolean> {
    const budget = await this.getErrorBudget(slo);
    // 错误预算消耗超过 50% → 冻结部署
    return budget.budgetUsed > budget.budgetTotal * 0.5;
  }
}
```

### Step 3：Burn Rate 告警（比单纯阈值更强）

```typescript
// burn-rate-alerting.ts
// 参考 Google SRE Book 的多窗口 Burn Rate 告警

interface BurnRateAlert {
  severity: 'page' | 'ticket';
  longWindowHours: number;
  shortWindowHours: number;
  burnRateThreshold: number;
}

// 经典双窗口配置（1小时+5分钟/6小时+30分钟/3天+6小时）
const ALERT_CONFIGS: BurnRateAlert[] = [
  {
    severity: 'page',         // 立即呼人
    longWindowHours: 1,
    shortWindowHours: 1/12,   // 5分钟
    burnRateThreshold: 14.4,  // 消耗速率超过 14.4x
  },
  {
    severity: 'page',
    longWindowHours: 6,
    shortWindowHours: 0.5,    // 30分钟
    burnRateThreshold: 6,
  },
  {
    severity: 'ticket',       // 创建工单
    longWindowHours: 72,      // 3天
    shortWindowHours: 6,
    burnRateThreshold: 1,
  },
];

class BurnRateAlerter {
  async checkAlerts(sloConfig: SLOConfig): Promise<void> {
    for (const alertConfig of ALERT_CONFIGS) {
      const [longBurn, shortBurn] = await Promise.all([
        this.getBurnRate(sloConfig, alertConfig.longWindowHours),
        this.getBurnRate(sloConfig, alertConfig.shortWindowHours),
      ]);
      
      // 双窗口：长窗口确认趋势，短窗口确认当前
      if (
        longBurn > alertConfig.burnRateThreshold &&
        shortBurn > alertConfig.burnRateThreshold
      ) {
        await this.fireAlert({
          severity: alertConfig.severity,
          sloName: sloConfig.name,
          burnRate: longBurn,
          message: `SLO "${sloConfig.name}" burn rate ${longBurn.toFixed(1)}x`,
        });
      }
    }
  }

  private async getBurnRate(
    slo: SLOConfig,
    windowHours: number
  ): Promise<number> {
    // 计算指定时间窗口内的消耗速率
    const windowMs = windowHours * 3600 * 1000;
    const now = Date.now();
    // ... (同上查询逻辑)
    return burnRate;
  }
}
```

### Step 4：Error Budget 驱动发布决策

```typescript
// deploy-gate.ts
class AgentDeployGate {
  async canDeploy(agentName: string): Promise<{
    allowed: boolean;
    reason: string;
    budgetStatus: Record<string, unknown>;
  }> {
    const slos = await this.getSLOsForAgent(agentName);
    const budgets = await Promise.all(
      slos.map(slo => this.errorBudgetManager.getErrorBudget(slo))
    );
    
    // 任一 SLO 错误预算告急 → 阻止发布
    for (let i = 0; i < slos.length; i++) {
      const slo = slos[i];
      const budget = budgets[i];
      
      if (budget.budgetRemaining < 0) {
        return {
          allowed: false,
          reason: `SLO "${slo.name}" 错误预算已耗尽（消耗速率 ${budget.burnRate.toFixed(1)}x）。先修复稳定性问题再发布。`,
          budgetStatus: budget,
        };
      }
      
      if (budget.budgetUsed > budget.budgetTotal * 0.8) {
        return {
          allowed: false,
          reason: `SLO "${slo.name}" 错误预算仅剩 ${((1 - budget.budgetUsed/budget.budgetTotal)*100).toFixed(1)}%，请谨慎发布`,
          budgetStatus: budget,
        };
      }
    }
    
    const minRemaining = Math.min(...budgets.map(b => b.budgetRemaining));
    return {
      allowed: true,
      reason: `错误预算充足，剩余 ${minRemaining.toFixed(0)} 分钟`,
      budgetStatus: budgets,
    };
  }
}
```

### Step 5：Cron 驱动的 SLO 报告

```typescript
// OpenClaw Cron 每日 SLO 报告
// payload: { kind: "agentTurn", message: "生成昨日 SLO 报告并发送" }

async function generateSLOReport() {
  const slos = [
    { name: '任务完成率', sliType: 'task', target: 0.99, windowDays: 30 },
    { name: '工具调用成功率', sliType: 'tool_call', target: 0.999, windowDays: 7 },
    { name: '响应延迟 P95 < 30s', sliType: 'task', target: 0.95, windowDays: 7 },
  ];
  
  const lines: string[] = ['📊 **Agent SLO 日报**\n'];
  
  for (const slo of slos) {
    const budget = await errorBudgetManager.getErrorBudget(slo);
    const statusEmoji = budget.burnRate > 2 ? '🔴' : budget.burnRate > 1 ? '🟡' : '🟢';
    
    lines.push(`${statusEmoji} **${slo.name}**`);
    lines.push(`  目标: ${(slo.target * 100).toFixed(2)}%`);
    lines.push(`  预算剩余: ${budget.budgetRemaining.toFixed(0)} min`);
    lines.push(`  消耗速率: ${budget.burnRate.toFixed(2)}x`);
    if (budget.exhaustsAt) {
      lines.push(`  ⚠️ 预计耗尽: ${budget.exhaustsAt.toLocaleDateString()}`);
    }
    lines.push('');
  }
  
  return lines.join('\n');
}
```

---

## 和 OpenClaw 的结合

OpenClaw 本身的可观测性工具可以作为 SLI 数据源：

```typescript
// 从 session_history 提取 SLI
// 工具结果中有 success/error 状态
// exec 工具的 exitCode 可判断成功

// 在 HEARTBEAT.md 中加入 SLO 检查：
// - 检查最近 1 小时工具失败率是否 > 1%
// - 检查 Cron 任务失败记录
// - 汇报当前 Error Budget 状态
```

---

## 关键指标参考

| Agent 类型 | 任务完成率 SLO | 工具调用成功率 | P95 延迟 |
|-----------|-------------|-------------|--------|
| 生产环境 API Agent | 99.5% | 99.9% | < 30s |
| Cron 后台 Agent | 99.0% | 99.5% | < 5min |
| 交互式聊天 Agent | 98.0% | 99.0% | < 10s |
| 开发/测试环境 | 95.0% | 95.0% | 无限制 |

---

## 三条铁律

1. **先定义 SLI，再谈 SLO**：你测不了的就承诺不了
2. **Error Budget 是政策工具**：预算充足→大胆创新，预算告急→冻结发布修稳定性
3. **Burn Rate 比阈值告警更有价值**：当前错误率 0.5% 不可怕，但 14x 消耗速率意味着 2 小时后预算耗尽

---

## 与其他课的关系

| 课程 | 关系 |
|------|------|
| [123-realtime-monitoring-alerting](123-realtime-monitoring-alerting.md) | 四大黄金信号是 SLI 的输入 |
| [108-health-check-auto-restart](108-health-check-auto-restart.md) | 健康探针数据可作为 SLI |
| [158-agent-cicd-pipeline](158-agent-cicd-pipeline.md) | Error Budget → 发布门控集成 |
| [146-chaos-engineering-resilience-testing](146-chaos-engineering-resilience-testing.md) | 混沌测试主动消耗 Error Budget 换取韧性验证 |

---

## 小结

```
SLI → 测量现实
SLO → 承诺目标  
Error Budget → 量化风险容忍度
Burn Rate → 预警速度

Error Budget 用光 → 停止新功能，全力修稳定性
Error Budget 充足 → 放心创新，快速迭代
```

**SLO 不是为了汇报，是为了做决策。**
