# 288. Agent 爆炸半径限制与渐进式自主（Blast Radius Limiting & Progressive Autonomy）

> Agent 出错不可怕，可怕的是一次错误影响所有用户、所有租户、所有任务。生产级 Agent 不应该一上来就“全自动全量执行”，而应该从小范围、低风险、可回滚的自主权开始，逐步放大。

上一课讲了 Kill Switch：事故发生时怎么一键停手。今天往前移一层：**事故还没发生前，如何限制每一次自主执行的最大影响范围**。

核心思想：

> 每个 Agent run 都必须带一个 Blast Radius Budget：最多能影响多少对象、多少租户、多少金额、多少条消息、多少次外部写操作；超过预算就自动降级为 dry-run、分批执行或要求人工审批。

---

## 1. 为什么需要爆炸半径限制？

传统后端服务通常有权限系统：这个 token 能不能调用这个 API。

但 Agent 的风险不是“有没有权限”这么简单，而是：

- 它可能把一个操作循环执行 1000 次；
- 它可能把测试租户的操作套到生产租户；
- 它可能把“通知一个人”理解成“通知全群”；
- 它可能在缓存/工具返回异常时扩大错误决策；
- 它可能让多个 sub-agent 并发执行同一种副作用。

所以除了权限，还要限制**影响范围**。

一句话：

> 权限回答“能不能做”，Blast Radius 回答“最多能做多大”。

---

## 2. Blast Radius Budget 长什么样？

一个实用的预算可以这样定义：

```json
{
  "runId": "run_20260511_0330",
  "autonomyLevel": "canary",
  "maxTenants": 1,
  "maxUsers": 5,
  "maxSideEffects": 10,
  "maxMoneyCents": 0,
  "allowedTools": ["message.send", "git.commit", "git.push"],
  "blockedTools": ["db.delete", "payment.refund"],
  "batchSize": 3,
  "requireApprovalAboveRisk": "medium",
  "stopOnFirstFailure": true
}
```

建议的自主等级：

- `observe`：只读观察，不做副作用；
- `dry_run`：生成计划和 diff，不真正写入；
- `canary`：小批量执行，例如 1 个租户 / 5 个用户；
- `limited`：扩大范围，但仍有预算；
- `full`：全量自动化，但保留 Kill Switch 和审计。

成熟系统里，Agent 应该从 `observe -> dry_run -> canary -> limited -> full` 渐进升级，而不是直接 full auto。

---

## 3. learn-claude-code：最小预算守卫

教学版可以先做一个纯 Python Guard：每次工具调用前扣预算，超限就抛错。

```python
# learn_claude_code/blast_radius.py
from __future__ import annotations

from dataclasses import dataclass, field
from typing import Literal

Risk = Literal["low", "medium", "high", "critical"]
AutonomyLevel = Literal["observe", "dry_run", "canary", "limited", "full"]


@dataclass
class ToolCall:
    name: str
    risk: Risk
    side_effect: bool
    affected_tenants: set[str] = field(default_factory=set)
    affected_users: set[str] = field(default_factory=set)
    money_cents: int = 0


@dataclass
class BlastRadiusBudget:
    autonomy_level: AutonomyLevel
    max_tenants: int
    max_users: int
    max_side_effects: int
    max_money_cents: int
    allowed_tools: set[str]
    blocked_tools: set[str]
    require_approval_above_risk: Risk
    stop_on_first_failure: bool = True


RISK_ORDER = ["low", "medium", "high", "critical"]


class BlastRadiusGuard:
    def __init__(self, budget: BlastRadiusBudget):
        self.budget = budget
        self.tenants: set[str] = set()
        self.users: set[str] = set()
        self.side_effects = 0
        self.money_cents = 0

    def preflight(self, call: ToolCall) -> Literal["allow", "dry_run", "approval_required"]:
        if call.name in self.budget.blocked_tools:
            raise PermissionError(f"tool_blocked_by_blast_radius: {call.name}")

        if self.budget.allowed_tools and call.name not in self.budget.allowed_tools:
            raise PermissionError(f"tool_not_in_budget: {call.name}")

        if self.budget.autonomy_level == "observe" and call.side_effect:
            return "dry_run"

        if self.budget.autonomy_level == "dry_run" and call.side_effect:
            return "dry_run"

        approval_threshold = RISK_ORDER.index(self.budget.require_approval_above_risk)
        if RISK_ORDER.index(call.risk) > approval_threshold:
            return "approval_required"

        next_tenants = self.tenants | call.affected_tenants
        next_users = self.users | call.affected_users
        next_side_effects = self.side_effects + (1 if call.side_effect else 0)
        next_money = self.money_cents + call.money_cents

        if len(next_tenants) > self.budget.max_tenants:
            raise PermissionError("blast_radius_exceeded: max_tenants")
        if len(next_users) > self.budget.max_users:
            raise PermissionError("blast_radius_exceeded: max_users")
        if next_side_effects > self.budget.max_side_effects:
            raise PermissionError("blast_radius_exceeded: max_side_effects")
        if next_money > self.budget.max_money_cents:
            raise PermissionError("blast_radius_exceeded: max_money_cents")

        return "allow"

    def commit(self, call: ToolCall) -> None:
        self.tenants |= call.affected_tenants
        self.users |= call.affected_users
        self.side_effects += 1 if call.side_effect else 0
        self.money_cents += call.money_cents
```

使用方式：

```python
budget = BlastRadiusBudget(
    autonomy_level="canary",
    max_tenants=1,
    max_users=5,
    max_side_effects=10,
    max_money_cents=0,
    allowed_tools={"message.send", "git.commit", "git.push"},
    blocked_tools={"db.delete", "payment.refund"},
    require_approval_above_risk="medium",
)

guard = BlastRadiusGuard(budget)
call = ToolCall(
    name="message.send",
    risk="low",
    side_effect=True,
    affected_users={"telegram:-5115329245"},
)

decision = guard.preflight(call)
if decision == "allow":
    # execute tool here
    guard.commit(call)
elif decision == "dry_run":
    print("would send message, but autonomy is dry_run")
else:
    print("approval required")
```

注意：`commit()` 必须在工具真实成功后调用，不能 preflight 通过就扣账。否则失败重试时预算会被错误消耗。

---

## 4. pi-mono：把预算做成 Tool Middleware

生产版建议把预算放到 `RunContext`，所有工具统一通过 Middleware。

```ts
// pi-mono/packages/agent-runtime/src/safety/blast-radius-middleware.ts
export type Risk = "low" | "medium" | "high" | "critical";
export type AutonomyLevel = "observe" | "dry_run" | "canary" | "limited" | "full";

export interface ToolMeta {
  name: string;
  risk: Risk;
  sideEffect: boolean;
}

export interface ToolImpact {
  tenants?: string[];
  users?: string[];
  moneyCents?: number;
}

export interface BlastRadiusBudget {
  autonomyLevel: AutonomyLevel;
  maxTenants: number;
  maxUsers: number;
  maxSideEffects: number;
  maxMoneyCents: number;
  allowedTools: string[];
  blockedTools: string[];
  requireApprovalAboveRisk: Risk;
  batchSize: number;
  stopOnFirstFailure: boolean;
}

export interface BlastRadiusUsage {
  tenants: Set<string>;
  users: Set<string>;
  sideEffects: number;
  moneyCents: number;
}

const riskRank: Record<Risk, number> = {
  low: 1,
  medium: 2,
  high: 3,
  critical: 4,
};

export class BlastRadiusExceeded extends Error {
  constructor(public readonly reason: string) {
    super(`blast_radius_exceeded: ${reason}`);
  }
}

export function createBlastRadiusMiddleware() {
  return async function blastRadiusMiddleware(ctx: {
    runId: string;
    tool: ToolMeta;
    impact: ToolImpact;
    budget: BlastRadiusBudget;
    usage: BlastRadiusUsage;
    markDryRun: () => void;
    requireApproval: (reason: string) => Promise<void>;
  }, next: () => Promise<unknown>) {
    const { tool, budget, usage, impact } = ctx;

    if (budget.blockedTools.includes(tool.name)) {
      throw new BlastRadiusExceeded(`blocked_tool:${tool.name}`);
    }

    if (budget.allowedTools.length > 0 && !budget.allowedTools.includes(tool.name)) {
      throw new BlastRadiusExceeded(`tool_not_allowed:${tool.name}`);
    }

    if ((budget.autonomyLevel === "observe" || budget.autonomyLevel === "dry_run") && tool.sideEffect) {
      ctx.markDryRun();
      return next();
    }

    if (riskRank[tool.risk] > riskRank[budget.requireApprovalAboveRisk]) {
      await ctx.requireApproval(`risk ${tool.risk} exceeds ${budget.requireApprovalAboveRisk}`);
    }

    const nextTenants = new Set([...usage.tenants, ...(impact.tenants ?? [])]);
    const nextUsers = new Set([...usage.users, ...(impact.users ?? [])]);
    const nextSideEffects = usage.sideEffects + (tool.sideEffect ? 1 : 0);
    const nextMoney = usage.moneyCents + (impact.moneyCents ?? 0);

    if (nextTenants.size > budget.maxTenants) throw new BlastRadiusExceeded("max_tenants");
    if (nextUsers.size > budget.maxUsers) throw new BlastRadiusExceeded("max_users");
    if (nextSideEffects > budget.maxSideEffects) throw new BlastRadiusExceeded("max_side_effects");
    if (nextMoney > budget.maxMoneyCents) throw new BlastRadiusExceeded("max_money_cents");

    const result = await next();

    // 只有工具成功后才更新 usage，保证失败重试不会污染预算。
    for (const tenant of impact.tenants ?? []) usage.tenants.add(tenant);
    for (const user of impact.users ?? []) usage.users.add(user);
    usage.sideEffects = nextSideEffects;
    usage.moneyCents = nextMoney;

    return result;
  };
}
```

这层和上一课的 Kill Switch 是互补关系：

- Kill Switch：全局或局部紧急刹车；
- Blast Radius：每个 run 的最大影响范围；
- Approval：超过预算或风险阈值时升级人工。

---

## 5. OpenClaw 落地：给 Cron/Sub-agent 加“自主等级”

OpenClaw 这种 always-on 环境，最适合用 Blast Radius 控制定时任务和子任务。

例如课程 cron 可以有一个很小的预算：

```json
{
  "job": "agent-course-cron",
  "autonomyLevel": "limited",
  "maxTenants": 1,
  "maxUsers": 1,
  "maxSideEffects": 4,
  "allowedTools": [
    "file.write",
    "file.edit",
    "message.send",
    "git.commit",
    "git.push"
  ],
  "blockedTools": ["db.delete", "payment.refund", "server.restart"],
  "requireApprovalAboveRisk": "medium",
  "stopOnFirstFailure": true
}
```

对应这次课程任务，副作用其实只有四类：

1. 写 lesson 文件；
2. 更新 README / TOOLS；
3. 发 Telegram 群消息；
4. git commit + push。

如果某次 Agent 想额外执行部署、删文件、批量通知多个群，就应该被预算挡住。

---

## 6. 批量任务：先 canary，再扩大

很多 Agent 任务天然是批量的：

- 给 500 个用户发通知；
- 给 100 个仓库升级依赖；
- 给多个租户跑配置修复；
- 对一批订单执行补偿。

不要直接全量跑。推荐流程：

```text
observe    -> 统计影响对象，生成计划
 dry_run   -> 输出预计 diff / message / SQL，不写入
 canary    -> 先执行 1~5 个对象
 verify    -> 检查错误率、用户反馈、指标
 limited   -> 分批扩大，每批后验证
 full      -> 只有证据稳定才全量
```

关键是每一批都要有：

- batch id；
- affected objects；
- success / failed / skipped；
- rollback 或 compensation plan；
- 是否允许进入下一批的 gate。

---

## 7. 常见坑

**坑 1：只限制工具，不限制对象数量**

`message.send` 本身是低风险，但循环发 10 万次就是事故。

**坑 2：只在计划阶段检查预算**

计划阶段说影响 5 个用户，执行阶段工具返回 500 个用户，必须重新检查。

**坑 3：sub-agent 各自有预算，但总预算不共享**

10 个 sub-agent 每个允许 10 次写入，总体就是 100 次。父任务要有全局预算，子任务只能领取子预算。

**坑 4：失败重试不算预算**

重复失败的副作用也会影响外部系统。建议记录 attempt budget 和 successful side-effect budget 两套指标。

---

## 8. 实战 Checklist

给 Agent 加自主执行能力时，至少问 8 个问题：

- 这次 run 的 `autonomyLevel` 是什么？
- 最多影响几个 tenant / user / repo / order？
- 最多允许几次外部写操作？
- 是否涉及金额、删除、部署、权限变更？
- 超过哪个风险等级必须人工审批？
- 批量任务的 canary 批次多大？
- sub-agent 是否共享父任务总预算？
- 最终证据里有没有记录实际影响范围？

---

## 9. 一句话总结

> 生产级 Agent 的自主性不是“放开手让它跑”，而是给它一个明确、可审计、可逐步放大的行动半径。

权限控制让 Agent 不越权；Kill Switch 让 Agent 能停下；Blast Radius Budget 让 Agent 即使犯错，也只能在小范围内犯错。
