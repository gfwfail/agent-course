# 284. Agent 运维 Runbook 与受控执行（Operational Runbooks & Guarded Execution）

> 很多线上事故不是因为 Agent 不会操作，而是因为它“太会操作”：知道命令、能调用工具、也有权限，但缺少一份可复用、可审计、可暂停、可回滚的操作剧本。

前几课我们讲了发布验证、证据包、缓存 DR。今天把这些能力收束到一个更通用的生产模式：**Agent 运维 Runbook**。

核心思想：

> 不要让 Agent 临场发挥生产操作；把操作流程写成结构化 Runbook，让执行器负责 dry-run、审批、幂等、证据和回滚。

---

## 1. 为什么 Agent 需要 Runbook？

传统脚本通常只关心“怎么做”：

```bash
kubectl rollout restart deploy/main
curl /health
```

但 Agent 生产操作还要回答：

1. **为什么做**：来自用户请求、告警、cron 还是自动恢复？
2. **能不能做**：当前时间是否在变更窗口？风险等级是否需要审批？
3. **做过没有**：重复触发时能否幂等跳过？
4. **做到哪一步**：失败后能否从 checkpoint 恢复？
5. **怎么证明**：执行前后证据、工具输出、外部回执在哪里？
6. **怎么撤回**：失败时走 rollback、compensate 还是 manual review？

所以 Runbook 不是一篇文档，而是 Agent 可以执行的结构化工作流。

---

## 2. 一个好的 Runbook 至少包含什么？

建议每个生产 Runbook 都有这些字段：

```yaml
id: restart-agent-workers
risk: medium
owner: platform
idempotencyKey: "restart-agent-workers:${cluster}:${deploy}"
preconditions:
  - kind: change_window
    policy: normal-prod-change
  - kind: health_check
    target: api
steps:
  - id: snapshot
    action: collect_evidence
  - id: restart
    action: tool_call
    tool: kubectl.rollout_restart
    requiresApproval: true
  - id: verify
    action: probe
    slo: agent_turn_success_rate >= 0.99
rollback:
  - action: kubectl.rollout_undo
onFailure: manual_review
```

重点不是 YAML 格式，而是四个约束：

- **步骤有名字**：日志和证据可以精确定位。
- **动作有类型**：执行器知道哪些需要 dry-run / approval。
- **前置条件可测试**：不是写在 prompt 里靠模型记住。
- **失败路径显式**：rollback / retry / pause / manual_review 不能临时猜。

---

## 3. learn-claude-code：最小 Runbook Executor

教学版先实现一个纯 Python 执行器。它不直接执行危险操作，而是统一返回 `allowed / blocked / approval_required`。

```python
# learn_claude_code/runbook_executor.py
from __future__ import annotations

import json
import time
from dataclasses import dataclass, field
from typing import Any, Literal

Decision = Literal["allowed", "blocked", "approval_required"]
StepStatus = Literal["pending", "skipped", "done", "failed", "waiting_approval"]


@dataclass
class RunbookStep:
    id: str
    action: str
    args: dict[str, Any] = field(default_factory=dict)
    requires_approval: bool = False
    rollback_action: str | None = None


@dataclass
class Runbook:
    id: str
    risk: Literal["low", "medium", "high"]
    idempotency_key: str
    steps: list[RunbookStep]


@dataclass
class StepResult:
    step_id: str
    status: StepStatus
    decision: Decision
    evidence: dict[str, Any]


class RunbookExecutor:
    def __init__(self, ledger_path: str):
        self.ledger_path = ledger_path
        self.completed_keys: set[str] = set()

    def execute(self, runbook: Runbook, *, dry_run: bool = True) -> list[StepResult]:
        if runbook.idempotency_key in self.completed_keys:
            return [StepResult(
                step_id="idempotency",
                status="skipped",
                decision="allowed",
                evidence={"reason": "already_completed", "key": runbook.idempotency_key},
            )]

        results: list[StepResult] = []

        for step in runbook.steps:
            decision = self._policy_decision(runbook, step, dry_run=dry_run)

            if decision == "blocked":
                result = StepResult(step.id, "failed", decision, {"reason": "policy_blocked"})
                results.append(result)
                self._append_ledger(runbook, result)
                break

            if decision == "approval_required":
                result = StepResult(step.id, "waiting_approval", decision, {
                    "reason": "approval_required",
                    "action": step.action,
                    "args": step.args,
                })
                results.append(result)
                self._append_ledger(runbook, result)
                break

            # 教学版：真实系统这里会 dispatch 到工具层
            output = self._dispatch(step, dry_run=dry_run)
            result = StepResult(step.id, "done", decision, output)
            results.append(result)
            self._append_ledger(runbook, result)

        if all(r.status in ("done", "skipped") for r in results):
            self.completed_keys.add(runbook.idempotency_key)

        return results

    def _policy_decision(self, runbook: Runbook, step: RunbookStep, *, dry_run: bool) -> Decision:
        if runbook.risk == "high" and not dry_run:
            return "approval_required"
        if step.requires_approval and not dry_run:
            return "approval_required"
        if step.action.startswith("delete_") and dry_run:
            return "blocked"
        return "allowed"

    def _dispatch(self, step: RunbookStep, *, dry_run: bool) -> dict[str, Any]:
        return {
            "dry_run": dry_run,
            "action": step.action,
            "args": step.args,
            "observed_at": time.time(),
        }

    def _append_ledger(self, runbook: Runbook, result: StepResult) -> None:
        event = {
            "runbook_id": runbook.id,
            "idempotency_key": runbook.idempotency_key,
            "step_id": result.step_id,
            "status": result.status,
            "decision": result.decision,
            "evidence": result.evidence,
            "ts": time.time(),
        }
        with open(self.ledger_path, "a", encoding="utf-8") as f:
            f.write(json.dumps(event, ensure_ascii=False) + "\n")
```

这段代码故意简单，但已经有生产骨架：

- `idempotency_key` 防止重复执行。
- `dry_run` 默认开启。
- 高风险 / 审批步骤不会偷偷执行。
- 每一步都写 ledger，后面能恢复、审计、复盘。

---

## 4. pi-mono：生产版 RunbookRunner

在 pi-mono 这种 TypeScript 生产架构里，Runbook 应该接到 Tool Middleware / Policy / Evidence 系统，而不是自己绕过工具层。

```ts
// pi-mono/packages/agent-runtime/src/runbooks/runbook-runner.ts
export type Risk = "low" | "medium" | "high";
export type StepDecision = "allow" | "deny" | "require_approval" | "dry_run_only";

export interface RunbookStep {
  id: string;
  tool: string;
  args: Record<string, unknown>;
  risk?: Risk;
  requiresApproval?: boolean;
  rollback?: { tool: string; args: Record<string, unknown> };
  verify?: { probe: string; expect: string };
}

export interface RunbookSpec {
  id: string;
  owner: string;
  risk: Risk;
  idempotencyKey: string;
  preconditions: Array<{ kind: string; value: string }>;
  steps: RunbookStep[];
}

export interface ToolGateway {
  call(input: {
    tool: string;
    args: Record<string, unknown>;
    dryRun: boolean;
    purpose: string;
    risk: Risk;
  }): Promise<{ ok: boolean; output: unknown; receipt?: string }>;
}

export interface PolicyEngine {
  decide(input: {
    actor: string;
    tool: string;
    args: Record<string, unknown>;
    risk: Risk;
    purpose: string;
  }): Promise<{ decision: StepDecision; reasons: string[] }>;
}

export interface EvidenceRecorder {
  record(event: Record<string, unknown>): Promise<void>;
}

export class RunbookRunner {
  constructor(
    private readonly tools: ToolGateway,
    private readonly policy: PolicyEngine,
    private readonly evidence: EvidenceRecorder,
    private readonly idempotency: { seen(key: string): Promise<boolean>; mark(key: string): Promise<void> },
  ) {}

  async run(spec: RunbookSpec, actor: string, opts: { dryRun: boolean }) {
    if (await this.idempotency.seen(spec.idempotencyKey)) {
      return { status: "skipped", reason: "idempotency_key_seen" } as const;
    }

    const completed: RunbookStep[] = [];

    for (const step of spec.steps) {
      const risk = step.risk ?? spec.risk;
      const decision = await this.policy.decide({
        actor,
        tool: step.tool,
        args: step.args,
        risk,
        purpose: `runbook:${spec.id}:${step.id}`,
      });

      await this.evidence.record({
        type: "runbook.step.policy",
        runbookId: spec.id,
        stepId: step.id,
        decision,
      });

      if (decision.decision === "deny") {
        await this.rollback(spec, completed, actor);
        return { status: "blocked", step: step.id, reasons: decision.reasons } as const;
      }

      if (decision.decision === "require_approval" || step.requiresApproval) {
        return { status: "waiting_approval", step: step.id, reasons: decision.reasons } as const;
      }

      const dryRun = opts.dryRun || decision.decision === "dry_run_only";
      const receipt = await this.tools.call({
        tool: step.tool,
        args: step.args,
        dryRun,
        risk,
        purpose: `runbook:${spec.id}:${step.id}`,
      });

      await this.evidence.record({
        type: "runbook.step.result",
        runbookId: spec.id,
        stepId: step.id,
        dryRun,
        receipt,
      });

      if (!receipt.ok) {
        await this.rollback(spec, completed, actor);
        return { status: "failed", step: step.id, receipt } as const;
      }

      completed.push(step);
    }

    if (!opts.dryRun) {
      await this.idempotency.mark(spec.idempotencyKey);
    }

    return { status: "completed", dryRun: opts.dryRun } as const;
  }

  private async rollback(spec: RunbookSpec, completed: RunbookStep[], actor: string) {
    for (const step of completed.reverse()) {
      if (!step.rollback) continue;
      await this.tools.call({
        tool: step.rollback.tool,
        args: step.rollback.args,
        dryRun: false,
        risk: "medium",
        purpose: `runbook:${spec.id}:${step.id}:rollback:${actor}`,
      });
    }
  }
}
```

生产版关键点：

- Runbook 不直接 shell out，所有动作都走 `ToolGateway`。
- PolicyEngine 统一处理权限、审批、变更窗口、risk。
- EvidenceRecorder 记录每个 step 的 policy decision 和 tool receipt。
- rollback 只执行已完成步骤的补偿动作，避免乱回滚。

---

## 5. OpenClaw 实战：把 cron 任务也当 Runbook

比如课程 cron、账单巡检、发布验证，都可以用同一套结构：

```json
{
  "id": "agent-course-publish",
  "owner": "openclaw-main",
  "risk": "medium",
  "idempotencyKey": "agent-course:2026-05-10T05:30Z",
  "preconditions": [
    { "kind": "gh_user", "value": "gfwfail" },
    { "kind": "git_clean_or_autostash", "value": "agent-course" }
  ],
  "steps": [
    { "id": "select-topic", "tool": "workspace.search_previous_lessons", "args": {} },
    { "id": "write-lesson", "tool": "workspace.write", "args": { "path": "lessons/284-...md" } },
    { "id": "send-telegram", "tool": "message.send", "risk": "medium", "args": { "target": "-5115329245" } },
    { "id": "git-push", "tool": "git.push", "risk": "medium", "requiresApproval": false }
  ]
}
```

这样做的好处：

- cron 重试不会重复发课：靠 `idempotencyKey`。
- Telegram 发送、git push 都有 evidence receipt。
- 如果 GitHub 账号不对，precondition 直接失败。
- 如果 README 更新成功但 push 失败，下次可以从 checkpoint 继续。

---

## 6. 常见坑

1. **Runbook 只写自然语言**  
   自然语言适合人读，不适合机器可靠恢复。关键步骤要结构化。

2. **每一步都让 LLM 决策**  
   LLM 负责解释和补全上下文，执行权限必须交给 Policy / Tool Gateway。

3. **没有 idempotency key**  
   cron、webhook、retry 最容易重复触发。没有幂等键，迟早重复发消息、重复部署、重复扣费。

4. **rollback 写成“失败后让 Agent 想办法”**  
   这不是 rollback，是事故现场即兴创作。补偿动作要提前声明。

5. **证据只记成功结果**  
   被 deny、等待审批、dry-run 输出也要记录。这些才是安全系统真正有价值的证据。

---

## 7. 记忆口诀

> Runbook 是 Agent 的操作 SOP；Policy 决定能不能做，ToolGateway 决定怎么做，Evidence 证明做了什么，Idempotency 保证不会重复做。

成熟 Agent 的生产操作，不是“模型觉得可以”，而是：

1. 有剧本；
2. 有护栏；
3. 有证据；
4. 有回滚；
5. 重试也不会炸。
