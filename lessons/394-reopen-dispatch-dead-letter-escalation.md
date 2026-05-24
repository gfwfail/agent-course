# 394. Agent 重开调度的死信队列与人工升级（Reopen Dispatch Dead Letter Queue & Escalation）

上一课讲了 **Reopen Dispatch 的 lock、lease、heartbeat 和 fence token**：谁拿到最新租约，谁才有资格写回。

今天继续补下一层：**如果一个重开任务反复超时、反复失败、反复被抢占，不能无限重试。**

一句话：**恢复调度必须有 Dead Letter Queue，把自动修不动的 Reopen Ticket 变成结构化 Escalation Case，并冻结相关下游。**

---

## 1. 无限重试是恢复系统里的隐性事故

普通后台 job 失败后可以 retry 3 次。但 Reopen Dispatch 背后往往是：

- 外部副作用已经发生；
- 补偿账本处在 unknown 或 failed_safe；
- 用户、仓库、群消息、部署、账单等真实世界状态可能正在漂移；
- 下游 release gate 还在等这个 ticket 的 terminal outcome。

如果 worker 一直失败还继续自动重试，会出现两个问题：

1. 自动系统把同一个坏动作重复放大；
2. 人类看不到“这里已经超出自动恢复能力”。

所以重开调度不能只有 queued / claimed / completed，还要有 **dead_lettered** 和 **escalated**。

---

## 2. DLQ 不是垃圾桶，是可恢复案件

Dead Letter Queue 里不能只存 error message。它应该保留足够信息，让人工或更高权限 worker 接手。

~~~ts
type ReopenDispatchTicket = {
  ticketId: string;
  caseId: string;
  route: "reconcile" | "compensation" | "incident" | "manual_review";
  lockKey: string;
  priorityScore: number;
  attempts: number;
  maxAttempts: number;
  leaseTimeouts: number;
  lastError?: string;
  status: "queued" | "claimed" | "completed" | "dead_lettered";
};

type DeadLetterRecord = {
  dlqId: string;
  ticketId: string;
  caseId: string;
  lockKey: string;
  reason:
    | "max_attempts_exceeded"
    | "lease_timeout_budget_exceeded"
    | "unsafe_error"
    | "missing_required_evidence"
    | "manual_review_required";
  frozenDownstream: string[];
  evidenceRefs: string[];
  lastFenceToken: number;
  createdAt: string;
  escalationId?: string;
};

type EscalationCase = {
  escalationId: string;
  dlqId: string;
  severity: "low" | "medium" | "high" | "critical";
  owner: "human" | "senior_worker" | "incident_lead";
  requiredDecision:
    | "retry_with_override"
    | "forward_fix"
    | "accept_residual_risk"
    | "rollback"
    | "close_as_duplicate";
  status: "open" | "acknowledged" | "resolved";
};
~~~

关键点：DLQ 是 **可审计案件入口**，不是“失败日志目录”。

---

## 3. learn-claude-code：先用纯函数判断是否进 DLQ

教学版可以先写一个纯函数，把自动重试边界讲清楚。

~~~python
from dataclasses import dataclass
from typing import Literal

DlqReason = Literal[
    "max_attempts_exceeded",
    "lease_timeout_budget_exceeded",
    "unsafe_error",
    "missing_required_evidence",
    "manual_review_required",
]

Decision = Literal["retry", "dead_letter"]

@dataclass(frozen=True)
class FailureSignal:
    attempts: int
    max_attempts: int
    lease_timeouts: int
    max_lease_timeouts: int
    error_kind: str
    missing_evidence: bool
    requires_human: bool

def classify_failure(signal: FailureSignal) -> tuple[Decision, DlqReason | None]:
    if signal.requires_human:
        return ("dead_letter", "manual_review_required")

    if signal.missing_evidence:
        return ("dead_letter", "missing_required_evidence")

    if signal.error_kind in {"unsafe_side_effect", "policy_conflict", "fence_token_mismatch"}:
        return ("dead_letter", "unsafe_error")

    if signal.lease_timeouts >= signal.max_lease_timeouts:
        return ("dead_letter", "lease_timeout_budget_exceeded")

    if signal.attempts >= signal.max_attempts:
        return ("dead_letter", "max_attempts_exceeded")

    return ("retry", None)
~~~

这个函数的价值不是复杂，而是把规则固定下来：

- 缺关键证据不要重试；
- policy conflict 不要重试；
- fence token mismatch 不要靠重试掩盖；
- timeout 有预算，预算耗尽就升级；
- 自动恢复只能在明确安全的范围内继续。

---

## 4. pi-mono：失败写回时原子转 DLQ

生产版里，worker 失败时不能只更新 ticket.attempts。它要在同一个事务里完成三件事：

1. 记录 failure event；
2. 判断是否还能 retry；
3. 如不能 retry，创建 DeadLetterRecord 并冻结 downstream gate。

~~~ts
function classifyDispatchFailure(signal: {
  attempts: number;
  maxAttempts: number;
  leaseTimeouts: number;
  maxLeaseTimeouts: number;
  errorKind: string;
  missingEvidence: boolean;
  requiresHuman: boolean;
}): { action: "retry" | "dead_letter"; reason?: string } {
  if (signal.requiresHuman) return { action: "dead_letter", reason: "manual_review_required" };
  if (signal.missingEvidence) return { action: "dead_letter", reason: "missing_required_evidence" };
  if (["unsafe_side_effect", "policy_conflict", "fence_token_mismatch"].includes(signal.errorKind)) {
    return { action: "dead_letter", reason: "unsafe_error" };
  }
  if (signal.leaseTimeouts >= signal.maxLeaseTimeouts) {
    return { action: "dead_letter", reason: "lease_timeout_budget_exceeded" };
  }
  if (signal.attempts >= signal.maxAttempts) {
    return { action: "dead_letter", reason: "max_attempts_exceeded" };
  }
  return { action: "retry" };
}

async function recordDispatchFailure(
  db: Db,
  input: {
    ticketId: string;
    workerId: string;
    leaseId: string;
    fenceToken: number;
    errorKind: string;
    errorMessage: string;
    evidenceRefs: string[];
  },
  now: Date,
) {
  return db.transaction(async (tx) => {
    const ticket = await tx.reopenDispatchTicket.findUniqueOrThrow({
      where: { ticketId: input.ticketId },
      lock: "for update",
    });

    if (
      ticket.claimedBy !== input.workerId ||
      ticket.leaseId !== input.leaseId ||
      ticket.fenceToken !== input.fenceToken
    ) {
      throw new Error("stale worker cannot record dispatch failure");
    }

    const attempts = ticket.attempts + 1;
    const decision: { action: "retry" | "dead_letter"; reason?: string } =
      classifyDispatchFailure({
      attempts,
      maxAttempts: ticket.maxAttempts,
      leaseTimeouts: ticket.leaseTimeouts,
      maxLeaseTimeouts: ticket.maxLeaseTimeouts,
      errorKind: input.errorKind,
      missingEvidence: input.evidenceRefs.length === 0,
      requiresHuman: ticket.route === "manual_review",
    });

    await tx.dispatchFailureEvent.create({
      data: {
        ticketId: ticket.ticketId,
        workerId: input.workerId,
        leaseId: input.leaseId,
        fenceToken: input.fenceToken,
        errorKind: input.errorKind,
        errorMessage: input.errorMessage.slice(0, 2000),
        evidenceRefs: input.evidenceRefs,
        createdAt: now,
      },
    });

    if (decision.action === "retry") {
      return tx.reopenDispatchTicket.update({
        where: { ticketId: ticket.ticketId },
        data: {
          status: "queued",
          attempts,
          claimedBy: null,
          leaseId: null,
          leaseExpiresAt: null,
          lastError: input.errorKind,
        },
      });
    }

    const dlq = await tx.deadLetterRecord.create({
      data: {
        ticketId: ticket.ticketId,
        caseId: ticket.caseId,
        lockKey: ticket.lockKey,
        reason: decision.reason,
        frozenDownstream: ticket.downstreamGateIds,
        evidenceRefs: input.evidenceRefs,
        lastFenceToken: input.fenceToken,
        createdAt: now,
      },
    });

    await tx.downstreamGate.updateMany({
      where: { gateId: { in: ticket.downstreamGateIds } },
      data: {
        status: "frozen",
        frozenBy: "dispatch_dlq",
        frozenReason: decision.reason,
      },
    });

    return tx.reopenDispatchTicket.update({
      where: { ticketId: ticket.ticketId },
      data: {
        status: "dead_lettered",
        attempts,
        deadLetterId: dlq.dlqId,
        lastError: input.errorKind,
      },
    });
  });
}
~~~

这里还有一个重要细节：**stale worker 不能写 failure**。如果旧 worker 的 lease 已经过期，它连“我失败了”都不能覆盖新 holder 的状态，只能写旁路日志，不能改 ticket 终态。

---

## 5. Escalation Case 要问一个明确决策

很多升级系统的问题是只发一条“失败了”的告警。真正有用的升级应该问一个明确问题：

~~~ts
function buildEscalationCase(dlq: DeadLetterRecord): EscalationCase {
  const criticalReasons = new Set([
    "unsafe_error",
    "missing_required_evidence",
  ]);

  return {
    escalationId: crypto.randomUUID(),
    dlqId: dlq.dlqId,
    severity: criticalReasons.has(dlq.reason) ? "critical" : "high",
    owner: criticalReasons.has(dlq.reason) ? "incident_lead" : "senior_worker",
    requiredDecision:
      dlq.reason === "missing_required_evidence"
        ? "retry_with_override"
        : dlq.reason === "unsafe_error"
          ? "forward_fix"
          : "accept_residual_risk",
    status: "open",
  };
}
~~~

升级不是“请看一下”，而是：

- 是否允许带 override 重试；
- 是否改用 forward fix；
- 是否接受残余风险；
- 是否 rollback；
- 是否 close as duplicate。

这样人工接手不会从零分析，而是在结构化上下文里做决策。

---

## 6. OpenClaw 课程 Cron 怎么用

这套课程 cron 本身就适合当例子。

假设发课流程里 Telegram 已发成功，但 git push 一直失败。自动恢复可以 retry，但不能无限 retry：

- attempts < 3：重新 pull/rebase/push；
- 缺 messageId 或 commit hash：进 DLQ，因为证据缺口不能靠重试解决；
- GitHub token 权限冲突：进 DLQ，升级给 human/senior worker；
- Telegram 已发但 README 没推上去：冻结“完成汇报”，直到 Git reality matched；
- 人工决定 forward fix 后，再生成新的 DispatchTicket，而不是复活旧 worker 的 lease。

也就是说，DLQ 让 Agent 可以诚实地说：**这件事自动系统已经尽力到边界了，现在需要一个带上下文的决策。**

---

## 7. 实战建议

实现 Reopen Dispatch DLQ 时，至少加 5 条规则：

1. 每个 ticket 有 maxAttempts 和 maxLeaseTimeouts；
2. unsafe_error / policy_conflict / missing evidence 直接 DLQ；
3. DLQ 创建时冻结相关 downstream gates；
4. escalation case 必须包含 requiredDecision；
5. 从 DLQ 恢复时创建新 ticket，继承 evidenceRefs 和 parentDlqId，不复用旧 lease。

最后这条很关键：**DLQ 恢复不是把旧任务改回 queued，而是创建一个带人工决策证据的新任务。**

---

## 8. 小结

Reopen Dispatch 的成熟度分三层：

- priority scheduling：先处理最危险的；
- lock + lease：证明谁能处理；
- DLQ + escalation：证明自动系统什么时候必须停手。

成熟 Agent 不是永远自动修，而是知道自动恢复的边界在哪里，并把越界任务升级成可审计、可决策、可继续恢复的案件。
