# 293. Agent 运行健康 SLO 与错误预算（Runtime Health SLO & Error Budget）

> 核心观点：Agent 不能只看「有没有回复」。生产级 Agent 要给每类能力定义 SLO，并用错误预算决定：继续自动化、降级、暂停，还是升级人工。

前面我们讲过 ORR、Game Day、Kill Switch 和交接包。今天补上中间最容易缺的一层：**Runtime Health SLO**。

很多 Agent Demo 的健康检查只有一个：进程活着。但真正线上会坏在更细的地方：

- LLM 首 token 很慢，但最终还能回；
- 工具调用 5xx 增多，但不是全挂；
- Sub-agent 积压越来越多；
- 用户请求能完成，但完成证据缺失；
- 发消息成功率下降，导致「做了但没通知」。

所以成熟 Agent 要把「健康」拆成可度量的 SLI/SLO。

---

## 1. SLI / SLO / Error Budget

- **SLI（Service Level Indicator）**：实际观测指标，例如 `turn_success_rate`、`tool_p95_ms`、`message_delivery_rate`。
- **SLO（Service Level Objective）**：目标，例如 30 分钟内 Agent turn 成功率 ≥ 99%。
- **Error Budget**：允许失败额度，例如 1% 失败预算。预算烧完后，Agent 不应该继续扩大副作用。

Agent 常用 SLI：

```yaml
sli:
  turn_success_rate:
    good: completed && !blocked && final_reply_present
    total: all_runs
  tool_success_rate:
    good: tool.status == "ok"
    total: all_tool_calls
  side_effect_verified_rate:
    good: side_effect.committed && side_effect.verified
    total: side_effect.attempted
  delivery_receipt_rate:
    good: message.provider_id != null
    total: message.send_attempted
  latency_p95_ms:
    value: run.completed_at - run.started_at
```

关键不是指标多，而是每个指标都能驱动动作。

---

## 2. learn-claude-code：最小错误预算守卫

教学版可以用一个 JSONL 事件文件做滑动窗口统计。

```python
# learn_claude_code/health_slo.py
from dataclasses import dataclass
from time import time

@dataclass
class SloTarget:
    name: str
    window_sec: int
    min_success_rate: float
    min_total: int = 20

class ErrorBudgetGuard:
    def __init__(self, events):
        self.events = events

    def evaluate(self, target: SloTarget):
        cutoff = time() - target.window_sec
        xs = [e for e in self.events if e["slo"] == target.name and e["ts"] >= cutoff]
        total = len(xs)
        good = sum(1 for e in xs if e["good"])

        if total < target.min_total:
            return {"decision": "insufficient_data", "total": total}

        success_rate = good / total
        budget_burned = max(0, target.min_success_rate - success_rate)

        if success_rate >= target.min_success_rate:
            return {"decision": "allow", "successRate": success_rate}

        if budget_burned < 0.02:
            return {"decision": "degrade", "successRate": success_rate}

        return {"decision": "freeze_side_effects", "successRate": success_rate}
```

在 Agent Loop 里，不要等最终失败才记录；每次工具调用、消息发送、完成闸门都写事件：

```python
health.record({
    "ts": time(),
    "slo": "side_effect_verified_rate",
    "good": effect.status == "verified",
    "runId": run.id,
    "tool": effect.tool,
})

decision = budget_guard.evaluate(SloTarget(
    name="side_effect_verified_rate",
    window_sec=30 * 60,
    min_success_rate=0.995,
))

if decision["decision"] == "freeze_side_effects":
    run.mode = "read_only"
    notify_oncall("Side-effect verification SLO burned; switched to read-only")
```

这就是最小闭环：**观测 → 预算 → 降级动作**。

---

## 3. pi-mono：HealthSloMiddleware

生产版建议放在 middleware 层，让每个 run 自动携带 health decision。

```ts
// pi-mono/src/agent/health/HealthSloMiddleware.ts
type SloDecision = "allow" | "degrade" | "read_only" | "manual_review";

interface SloSnapshot {
  name: string;
  windowSec: number;
  successRate: number;
  target: number;
  total: number;
  decision: SloDecision;
  reasons: string[];
}

export class HealthSloMiddleware {
  constructor(
    private readonly metrics: HealthMetricStore,
    private readonly policy: HealthSloPolicy,
  ) {}

  async beforeRun(ctx: AgentRunContext) {
    const snapshots = await this.metrics.evaluateAll({
      tenantId: ctx.tenantId,
      agentId: ctx.agentId,
      now: ctx.clock.now(),
    });

    const decision = this.policy.combine(snapshots);

    ctx.health = { snapshots, decision };

    if (decision === "read_only") {
      ctx.capabilities = ctx.capabilities.withoutSideEffects();
    }

    if (decision === "manual_review") {
      ctx.requiresApproval = true;
    }
  }

  async afterTool(ctx: AgentRunContext, result: ToolResult) {
    await this.metrics.record({
      tenantId: ctx.tenantId,
      agentId: ctx.agentId,
      runId: ctx.runId,
      sli: `tool.${result.toolName}.success_rate`,
      good: result.status === "ok",
      latencyMs: result.latencyMs,
      observedAt: ctx.clock.now(),
    });
  }
}
```

策略合并要保守：

```ts
export class HealthSloPolicy {
  combine(xs: SloSnapshot[]): SloDecision {
    if (xs.some(x => x.name.includes("side_effect") && x.decision === "manual_review")) {
      return "manual_review";
    }

    if (xs.some(x => x.decision === "read_only")) return "read_only";
    if (xs.some(x => x.decision === "degrade")) return "degrade";
    return "allow";
  }
}
```

重点：SLO 不是 dashboard 上的装饰，而是参与能力裁剪。

---

## 4. OpenClaw 实战：Cron 课程任务怎么用

以这个课程 cron 为例，可以定义四个 SLO：

```json
{
  "agentCourseCron": {
    "turn_success_rate_24h": 0.99,
    "git_push_success_rate_24h": 0.995,
    "telegram_delivery_rate_24h": 0.995,
    "completion_gate_pass_rate_24h": 1.0
  }
}
```

执行前检查：

```ts
const health = await healthSlo.evaluate("agent-course-cron");

if (health.deliveryRate < 0.995) {
  // 还可以写课程、提交 git，但不要假装群里已经收到了
  mode.disable("telegram_send");
  mode.requireFinalReport("delivery degraded; message not sent automatically");
}

if (health.completionGatePassRate < 1.0) {
  mode.requireManualReview("recent course runs missed a completion gate");
}
```

最终交付时，证据包里要带健康快照：

```json
{
  "runId": "cron:agent-course:293",
  "healthDecision": "allow",
  "slo": {
    "git_push_success_rate_24h": 1,
    "telegram_delivery_rate_24h": 1,
    "completion_gate_pass_rate_24h": 1
  },
  "evidence": {
    "lessonFile": "lessons/293-runtime-health-slo-error-budget.md",
    "telegramMessageId": 116xx,
    "commit": "..."
  }
}
```

---

## 5. 设计建议

1. **按能力定义 SLO，不要只按进程定义**  
   Agent turn、工具调用、消息投递、外部副作用验证要分开算。

2. **高风险能力用更严格预算**  
   读文件失败可以重试；转账、部署、删数据这类副作用，一旦验证率下降就应该 read-only。

3. **错误预算烧完后自动降级**  
   不要只报警。成熟系统应该自动减少动作半径：`full → limited → dry_run → read_only → manual_review`。

4. **SLO 事件必须可审计**  
   每条 good/bad 事件要能追到 runId、tool、tenant、evidence，不然事故复盘无法判断预算为什么被烧掉。

---

## 6. 一句话总结

> 健康检查告诉你 Agent 还活着；SLO 和错误预算告诉你 Agent 现在还配不配继续自动化。

生产 Agent 的目标不是永远不失败，而是在失败率开始失控时，能自动收缩权限、保住副作用边界。