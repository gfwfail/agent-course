# 396. Agent 人工升级决策的执行闸门与恢复收据（Escalation Decision Enforcement & Resumption Gate）

上一课讲了 **Escalation Decision SLA & Owner Routing**：DLQ 不能只是写一句 manual review required，而要有 owner、ackBy、decisionDueAt、expiresAt 和结构化 Decision Record。

今天补下一步：**人类给了决策以后，Agent 不能直接“继续跑”。**

一句话：**人工决策只是恢复执行的输入；真正放行动作的是 Resumption Gate。它要校验决策有效期、证据覆盖、allowedActions、风险边界、租约/fenceToken，并写下 Resumption Receipt。**

---

## 1. “老板说可以”不能直接变成执行

很多系统的人工升级链路会这样写：

> 人类回复 approved，然后 worker 继续重试。

这很危险。因为 approved 没有说明：

- 允许哪个动作？
- 允许对哪个 target 做？
- 允许用哪份证据继续？
- 决策什么时候过期？
- 如果恢复执行失败，能不能再试？
- 旧 worker 的 lease 是否还有效？

恢复系统里最容易出事故的地方，是 **人工批准和自动执行之间的缝隙**。

所以人工决策不能直接解冻队列。它必须先经过一个恢复闸门。

---

## 2. Decision Record 要变成 Resumption Permit

建议把人类决策分成两层：

1. **EscalationDecision**：人类做出的判断；
2. **ResumptionPermit**：系统基于当前现实生成的一次性恢复许可。

~~~ts
type EscalationDecision = {
  decisionId: string;
  escalationId: string;
  decidedBy: string;
  decision:
    | "retry_with_override"
    | "forward_fix"
    | "accept_residual_risk"
    | "close_as_duplicate"
    | "open_incident";
  allowedActions: string[];
  deniedActions: string[];
  allowedTargets: string[];
  evidenceRefs: string[];
  constraints: {
    maxAttempts?: number;
    requireDryRun?: boolean;
    requireFreshSnapshot?: boolean;
    maxRiskTier?: "low" | "medium" | "high" | "critical";
  };
  validUntil: string;
};

type ResumptionPermit = {
  permitId: string;
  decisionId: string;
  caseId: string;
  action: string;
  target: string;
  evidenceRefs: string[];
  leaseId: string;
  fenceToken: number;
  expiresAt: string;
  singleUse: true;
};
~~~

这里的关键点是：**Decision Record 可以覆盖一个案件，Permit 只覆盖一次具体恢复动作。**

这能避免一个人工决策被多个 worker、多个 target、多个时间窗口重复滥用。

---

## 3. learn-claude-code：用纯函数判断能不能恢复

教学版先写纯函数，不碰数据库、不发消息，只回答一个问题：当前 action 是否能被这条 decision 放行。

~~~python
from dataclasses import dataclass
from datetime import datetime, timezone
from typing import Literal

Decision = Literal[
    "retry_with_override",
    "forward_fix",
    "accept_residual_risk",
    "close_as_duplicate",
    "open_incident",
]

@dataclass(frozen=True)
class EscalationDecision:
    decision_id: str
    decision: Decision
    allowed_actions: set[str]
    denied_actions: set[str]
    allowed_targets: set[str]
    evidence_refs: set[str]
    valid_until: datetime
    require_fresh_snapshot: bool

@dataclass(frozen=True)
class ResumeRequest:
    case_id: str
    action: str
    target: str
    required_evidence_refs: set[str]
    has_fresh_snapshot: bool
    now: datetime

@dataclass(frozen=True)
class ResumeGateResult:
    allowed: bool
    reason: str

def evaluate_resume_gate(
    decision: EscalationDecision,
    request: ResumeRequest,
) -> ResumeGateResult:
    if request.now >= decision.valid_until:
        return ResumeGateResult(False, "decision_expired")

    if request.action in decision.denied_actions:
        return ResumeGateResult(False, "action_explicitly_denied")

    if request.action not in decision.allowed_actions:
        return ResumeGateResult(False, "action_not_allowed")

    if request.target not in decision.allowed_targets:
        return ResumeGateResult(False, "target_not_allowed")

    missing = request.required_evidence_refs - decision.evidence_refs
    if missing:
        return ResumeGateResult(False, f"missing_decision_evidence:{sorted(missing)}")

    if decision.require_fresh_snapshot and not request.has_fresh_snapshot:
        return ResumeGateResult(False, "fresh_snapshot_required")

    return ResumeGateResult(True, "resume_permitted")
~~~

这个函数的价值不是复杂，而是把“人类批准到底批准了什么”变成可测试规则。

---

## 4. pi-mono：恢复许可必须在事务里抢新 lease

生产版不要让原来的 dead worker 继续拿旧 lease 跑。人工恢复应该重新 claim 一次：

1. 锁住 Escalation Case；
2. 校验 Decision Record；
3. 校验当前 Reality Snapshot；
4. 创建新的 lease / fenceToken；
5. 创建 single-use Resumption Permit；
6. 写 outbox 让 worker 继续执行。

~~~ts
async function createResumptionPermit(
  db: Db,
  input: {
    escalationId: string;
    decisionId: string;
    action: string;
    target: string;
    workerId: string;
    now: Date;
  },
) {
  return db.transaction(async (tx) => {
    const escalation = await tx.escalationCase.findUniqueOrThrow({
      where: { escalationId: input.escalationId },
      lock: "for update",
    });

    const decision = await tx.escalationDecision.findUniqueOrThrow({
      where: { decisionId: input.decisionId },
    });

    if (decision.escalationId !== escalation.escalationId) {
      throw new Error("decision_case_mismatch");
    }

    const snapshot = await tx.realitySnapshot.findFirst({
      where: { caseId: escalation.caseId },
      orderBy: { observedAt: "desc" },
    });

    const gate = evaluateResumeGate({
      decision,
      action: input.action,
      target: input.target,
      now: input.now,
      hasFreshSnapshot:
        snapshot != null &&
        input.now.getTime() - snapshot.observedAt.getTime() < 60_000,
      requiredEvidenceRefs: escalation.requiredEvidenceRefs,
    });

    if (!gate.allowed) {
      await tx.resumeDeniedAudit.create({
        data: {
          escalationId: escalation.escalationId,
          decisionId: decision.decisionId,
          action: input.action,
          target: input.target,
          reason: gate.reason,
        },
      });
      throw new Error(gate.reason);
    }

    const lease = await tx.workerLease.create({
      data: {
        caseId: escalation.caseId,
        workerId: input.workerId,
        purpose: "post_escalation_resumption",
        expiresAt: new Date(input.now.getTime() + 5 * 60_000),
      },
    });

    const permit = await tx.resumptionPermit.create({
      data: {
        decisionId: decision.decisionId,
        caseId: escalation.caseId,
        action: input.action,
        target: input.target,
        evidenceRefs: decision.evidenceRefs,
        leaseId: lease.leaseId,
        fenceToken: lease.fenceToken,
        expiresAt: lease.expiresAt,
        singleUse: true,
        usedAt: null,
      },
    });

    await tx.resumeOutbox.create({
      data: {
        kind: "resume_case",
        dedupeKey: `resume:${permit.permitId}`,
        payload: {
          permitId: permit.permitId,
          caseId: escalation.caseId,
          action: input.action,
          target: input.target,
          leaseId: lease.leaseId,
          fenceToken: lease.fenceToken,
        },
      },
    });

    return permit;
  });
}
~~~

注意这里不是“人类批准后直接调用工具”，而是生成一个有范围、有时效、有 fenceToken 的恢复许可。

---

## 5. Worker 执行前要消费 permit

Permit 最好是 single-use。worker 真正执行前，必须原子消费它：

~~~ts
async function consumePermitBeforeAction(
  tx: Tx,
  input: {
    permitId: string;
    workerId: string;
    leaseId: string;
    fenceToken: number;
    now: Date;
  },
) {
  const permit = await tx.resumptionPermit.findUniqueOrThrow({
    where: { permitId: input.permitId },
    lock: "for update",
  });

  if (permit.usedAt) throw new Error("permit_already_used");
  if (permit.expiresAt <= input.now) throw new Error("permit_expired");
  if (permit.leaseId !== input.leaseId) throw new Error("lease_mismatch");
  if (permit.fenceToken !== input.fenceToken) throw new Error("stale_fence_token");

  const lease = await tx.workerLease.findUniqueOrThrow({
    where: { leaseId: input.leaseId },
  });

  if (lease.workerId !== input.workerId) throw new Error("worker_mismatch");
  if (lease.fenceToken !== input.fenceToken) throw new Error("lease_fence_mismatch");
  if (lease.expiresAt <= input.now) throw new Error("lease_expired");

  await tx.resumptionPermit.update({
    where: { permitId: permit.permitId },
    data: { usedAt: input.now },
  });

  return permit;
}
~~~

这一步挡住三个常见事故：

- 旧 worker 醒来后继续写；
- 同一条人工决策被重复消费；
- permit 过期后仍被队列延迟执行。

---

## 6. OpenClaw 课程 Cron 里的实战映射

拿这个课程 cron 举例，外部副作用包括：

- 发 Telegram 课程；
- 写 lesson 文件；
- 更新 README；
- 更新 TOOLS；
- git commit/push。

如果某次 push 卡进 DLQ，人工决策不应该是“继续推”。更好的结构是：

~~~json
{
  "decision": "retry_with_override",
  "allowedActions": ["git_push", "telegram_edit_status"],
  "deniedActions": ["delete_lesson", "force_push"],
  "allowedTargets": ["gfwfail/agent-course:main"],
  "evidenceRefs": [
    "git_status_clean_before_retry",
    "remote_main_checked",
    "lesson_file_hash",
    "readme_entry_hash"
  ],
  "validUntil": "2026-05-24T11:45:00Z",
  "constraints": {
    "requireFreshSnapshot": true,
    "maxAttempts": 1
  }
}
~~~

然后 Resumption Gate 重新检查：

- 当前分支还是 main；
- 远端 main 没被别人推进到冲突状态；
- lesson 文件 hash 和 README entry hash 没变；
- action 是 git_push，不是 force_push；
- permit 还没过期且没被消费。

这样人工批准才不会变成无限权力。

---

## 7. 最后要写 Resumption Receipt

恢复执行后不要只把 case 标成 done。要写一张收据：

~~~ts
type ResumptionReceipt = {
  receiptId: string;
  permitId: string;
  decisionId: string;
  caseId: string;
  action: string;
  target: string;
  startedAt: string;
  finishedAt: string;
  outcome: "succeeded" | "failed_safe" | "failed_unsafe" | "needs_reopen";
  toolCallRefs: string[];
  postCheckEvidenceRefs: string[];
  nextState: "closed" | "reopened" | "manual_review" | "monitoring";
};
~~~

Receipt 的作用是把“人工批准后系统做了什么”重新接回证据链。否则审计时只能看到一个批准，看不到批准后的实际动作是否越界。

---

## 8. 记住这个判断

人工升级链路要分清四件事：

- **Escalation Case**：系统不知道怎么安全继续；
- **Escalation Decision**：人类给出恢复方向；
- **Resumption Permit**：系统确认当前现实允许这一次动作；
- **Resumption Receipt**：动作执行后留下可审计结果。

成熟 Agent 不把人工批准当万能钥匙，而是把它变成一次可验证、可过期、可消费、可追责的恢复许可。
