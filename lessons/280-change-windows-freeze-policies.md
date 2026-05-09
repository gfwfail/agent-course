# 280. Agent 变更窗口与冻结期策略（Change Windows & Freeze Policies）

> 核心思想：Agent 不是任何时间都应该动生产。把“什么时候能改、什么时候只能看、什么时候必须审批”做成策略层，能显著降低半夜自动化误操作风险。

很多 Agent 系统一开始只做权限判断：这个用户能不能 deploy？这个工具能不能写数据库？

但生产里还缺一个维度：**时间与业务状态**。

同一个 `deploy`：

- 周二下午低峰期：可以自动 dry-run + 执行
- 大促开始前 30 分钟：只能读状态，禁止变更
- 线上事故中：允许紧急修复，但必须 break-glass 审批
- 凌晨无人值守：低风险操作可自动执行，高风险操作排队到变更窗口

这就是 **Change Window / Freeze Policy**。

---

## 1. 设计一个变更窗口策略

一个实用策略至少包含 5 个字段：

```json
{
  "name": "prod-default",
  "timezone": "Australia/Sydney",
  "allowedWindows": [
    { "days": ["mon", "tue", "wed", "thu"], "start": "10:00", "end": "17:00" }
  ],
  "freezeWindows": [
    { "reason": "promo", "start": "2026-05-10T18:00:00+10:00", "end": "2026-05-11T02:00:00+10:00" }
  ],
  "breakGlass": {
    "allowed": true,
    "requiresApproval": true,
    "maxTtlMinutes": 30
  }
}
```

关键点：

- `allowedWindows`：普通变更允许时间
- `freezeWindows`：冻结期，优先级最高
- `timezone`：不要用服务器本地时间猜
- `breakGlass`：紧急通道必须短 TTL、必须留原因
- 策略返回的不是 boolean，而是决策：`allow / defer / require_approval / deny`

---

## 2. learn-claude-code：Python 教学版

先用一个纯函数实现策略判断，方便测试：

```python
from dataclasses import dataclass
from datetime import datetime, time
from zoneinfo import ZoneInfo

@dataclass
class ChangeRequest:
    actor: str
    environment: str
    action: str
    risk: str          # low / medium / high / critical
    emergency: bool = False
    reason: str | None = None

@dataclass
class Decision:
    outcome: str       # allow / defer / require_approval / deny
    reason: str
    earliest_at: datetime | None = None

class ChangeWindowPolicy:
    def __init__(self, tz: str = "Australia/Sydney"):
        self.tz = ZoneInfo(tz)

    def decide(self, req: ChangeRequest, now: datetime) -> Decision:
        local_now = now.astimezone(self.tz)

        if self._in_freeze(local_now):
            if req.emergency and req.reason and req.risk in {"high", "critical"}:
                return Decision("require_approval", "freeze_window_break_glass_required")
            return Decision("deny", "change_freeze_active")

        if self._in_allowed_window(local_now):
            if req.risk in {"high", "critical"}:
                return Decision("require_approval", "high_risk_change_requires_approval")
            return Decision("allow", "inside_change_window")

        if req.risk == "low":
            return Decision("defer", "outside_window_defer_low_risk", self._next_window(local_now))

        return Decision("require_approval", "outside_window_manual_approval_required")

    def _in_allowed_window(self, now: datetime) -> bool:
        return now.weekday() in range(0, 4) and time(10, 0) <= now.time() <= time(17, 0)

    def _in_freeze(self, now: datetime) -> bool:
        # 教学版写死；生产里从 DB / config / incident system 读取
        return False

    def _next_window(self, now: datetime) -> datetime:
        candidate = now.replace(hour=10, minute=0, second=0, microsecond=0)
        if now.time() >= time(17, 0):
            candidate = candidate.replace(day=candidate.day + 1)
        return candidate
```

这个函数的价值是：**LLM 不负责记规则，工具层负责执行规则**。

---

## 3. pi-mono：TypeScript 中间件生产版

生产版更适合放在 Tool Middleware 里，在工具真正执行前拦截：

```ts
type Risk = "low" | "medium" | "high" | "critical";
type ChangeDecision =
  | { outcome: "allow"; reason: string }
  | { outcome: "defer"; reason: string; earliestAt: string }
  | { outcome: "require_approval"; reason: string; approvalTtlSec: number }
  | { outcome: "deny"; reason: string };

type ToolCall = {
  toolName: string;
  args: Record<string, unknown>;
  metadata: {
    environment?: "dev" | "staging" | "prod";
    risk: Risk;
    writes: boolean;
    actor: string;
    emergency?: boolean;
    reason?: string;
  };
};

class ChangeWindowMiddleware {
  constructor(private policy: ChangeWindowPolicy) {}

  async beforeExecute(call: ToolCall): Promise<ToolCall> {
    if (!call.metadata.writes) return call;
    if (call.metadata.environment !== "prod") return call;

    const decision = await this.policy.decide({
      actor: call.metadata.actor,
      action: call.toolName,
      risk: call.metadata.risk,
      emergency: call.metadata.emergency,
      reason: call.metadata.reason,
    });

    switch (decision.outcome) {
      case "allow":
        return call;

      case "defer":
        throw new DeferredToolCallError({
          toolName: call.toolName,
          earliestAt: decision.earliestAt,
          reason: decision.reason,
        });

      case "require_approval":
        throw new ApprovalRequiredError({
          toolName: call.toolName,
          reason: decision.reason,
          ttlSec: decision.approvalTtlSec,
          canonicalArgs: canonicalize(call.args),
        });

      case "deny":
        throw new PolicyDeniedError(decision.reason);
    }
  }
}
```

注意这个中间件只拦截：

- `writes: true`
- `environment: prod`

只读诊断不应该被阻断，否则事故排查时 Agent 反而帮不上忙。

---

## 4. OpenClaw 实战：Cron / Heartbeat / Git Push 前都适用

OpenClaw 这种 Always-on Agent 最容易遇到“时间风险”：

- Cron 半夜自动跑
- Heartbeat 主动巡检
- Sub-agent 在后台完成后继续 push
- Telegram 审批可能过期

所以在执行副作用前做一个轻量 preflight：

```ts
async function preflightExternalWrite(action: ExternalAction) {
  const snapshot = await collectObservationSnapshot({
    gitSha: true,
    currentTime: true,
    channel: true,
    targetEnvironment: action.environment,
  });

  const decision = await changeWindowPolicy.decide({
    actor: action.actor,
    action: action.name,
    risk: action.risk,
    emergency: action.emergency,
    reason: action.reason,
    now: snapshot.currentTime,
  });

  await auditLog.append({
    type: "change_window_decision",
    action: action.name,
    decision,
    snapshot,
  });

  if (decision.outcome === "allow") return;
  if (decision.outcome === "defer") return scheduleAt(decision.earliestAt, action);
  if (decision.outcome === "require_approval") return requestApproval(action, decision);

  throw new Error(`blocked_by_change_policy: ${decision.reason}`);
}
```

这和前面几课的关系：

- 和 **Plan Signing** 结合：审批绑定计划 hash
- 和 **Observation Snapshot** 结合：执行前重新确认时间/环境
- 和 **Effect Journal** 结合：defer 后可恢复执行
- 和 **Policy-as-Code** 结合：策略可测试、可审计

---

## 5. 最小测试用例

```python
def test_prod_write_outside_window_requires_approval():
    policy = ChangeWindowPolicy()
    req = ChangeRequest(
        actor="cron-agent",
        environment="prod",
        action="deploy",
        risk="high",
    )

    now = datetime(2026, 5, 10, 3, 30, tzinfo=ZoneInfo("Australia/Sydney"))
    decision = policy.decide(req, now)

    assert decision.outcome == "require_approval"
    assert decision.reason == "outside_window_manual_approval_required"


def test_freeze_window_blocks_non_emergency():
    policy = PromoFreezePolicy()
    req = ChangeRequest(
        actor="agent",
        environment="prod",
        action="restart_service",
        risk="medium",
    )

    decision = policy.decide(req, promo_time())

    assert decision.outcome == "deny"
```

Agent 系统里，策略代码必须比 prompt 更可测试。

---

## 6. 常见坑

1. **只按权限，不按时间**  
   “能做”不代表“现在该做”。

2. **冻结期阻断所有工具**  
   错。只读诊断必须保留，否则事故时 Agent 失明。

3. **Break-glass 没 TTL**  
   紧急权限如果不自动过期，就会变成永久后门。

4. **审批后不复核时间**  
   审批发生在 16:59，执行发生在 18:30，就应该重新判断。

5. **用 prompt 写规则**  
   “不要在周末部署”写在 system prompt 里不可靠，要写在 Tool Middleware 里。

---

## 总结

成熟 Agent 的生产变更应该遵守一句话：

> 权限决定“谁能做”，策略决定“什么时候能做”，审批决定“例外怎么做”。

Change Window 不是传统运维流程的包袱，而是让自主 Agent 能安全长期运行的刹车系统。
