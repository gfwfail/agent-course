# 428. Agent 护栏补偿执行后的结果对账与关闭闸门（Guard Pattern Compensation Outcome Reconciliation & Close Gate）

上一课讲了 **Guard Pattern Repair Rollback Impact Backfill & Compensation Queue**：repair canary 回滚后，不能只把 pointer 切回 active，还要把 canary 期间受 candidate 影响的 decisionId 回填成 ImpactBackfillRecord，并按 diffClass / outcome / side effect 生成补偿任务。

今天继续往后走一步：**补偿任务执行完，怎么证明这件事真的可以关掉**。

一句话：**补偿执行不是 close。每个 rerun、revoke、unblock、notify、audit correction 都要产生 CompensationReceipt，再由 Outcome Reconciliation 对账目标状态、外部副作用、用户可见结果和审计记录；只有通过 Close Gate，rollback incident 才能从 mitigated 变成 closed_verified。**

---

## 1. 为什么补偿完成不等于事件关闭

很多系统看到队列任务状态变成 applied，就把事故关了。这对 Agent 护栏非常危险。

原因是补偿动作本身也可能失败、半成功，甚至制造新的副作用：

1. rerun_action 可能重跑成功，但写入了重复记录；
2. revoke_action 可能撤销了外部 API 状态，但没有同步本地投影；
3. unblock_action 可能解除了误挡，但用户已经超时放弃；
4. notify_user 可能发出通知，但内容和真实修复状态不一致；
5. append_audit_correction 可能补了审计说明，却没有关联原 decisionId。

所以 close gate 要问的不是“任务有没有跑完”，而是：

- **目标状态是否达成**：补偿后的事实是否等于 expected final state？
- **副作用是否闭环**：外部系统、本地状态、用户可见结果是否一致？
- **审计是否可追溯**：原决策、rollback、补偿和最终 outcome 是否串起来？
- **残余风险是否可接受**：无法补偿的部分是否有人工确认或长期观察计划？

补偿队列解决的是“做什么”，结果对账解决的是“做完以后世界是不是对的”。

---

## 2. 最小数据模型

把补偿关闭拆成三层对象：

~~~text
CompensationReceipt:
  receiptId
  rollbackReceiptId
  decisionId
  compensationPlan
  idempotencyKey
  startedAt
  finishedAt
  executor
  executionStatus:
    applied | skipped | blocked | failed
  externalEffects[]
  localStatePatch[]
  userVisibleEvent[]
  auditCorrectionId?

OutcomeReconciliation:
  reconciliationId
  receiptId
  expectedFinalState
  observedFinalState
  externalStateMatch: true | false | unknown
  localProjectionMatch: true | false | unknown
  userVisibleMatch: true | false | unknown
  auditTrailMatch: true | false | unknown
  residualRisk:
    none | low | medium | high
  mismatches[]
  recommendedAction:
    close_verified |
    retry_compensation |
    append_audit_correction |
    notify_user |
    manual_review |
    reopen_incident

CompensationCloseBundle:
  rollbackReceiptId
  repairedPatternVersion
  receipts[]
  reconciliations[]
  openMismatches[]
  closeDecision:
    close_verified |
    close_with_residual_risk |
    keep_observing |
    reopen_incident
~~~

注意：**Receipt 记录执行事实，Reconciliation 记录对账判断，CloseBundle 记录是否可以关闭整次 rollback incident**。不要把这三件事混成一个 status 字段。

---

## 3. learn-claude-code：单条补偿结果对账

教学版先写纯函数：给一条 receipt 和 observed state，判断下一步动作。

~~~py
from dataclasses import dataclass
from typing import Literal


ExecutionStatus = Literal["applied", "skipped", "blocked", "failed"]
MatchState = Literal["match", "mismatch", "unknown"]
ResidualRisk = Literal["none", "low", "medium", "high"]
RecommendedAction = Literal[
    "close_verified",
    "retry_compensation",
    "append_audit_correction",
    "notify_user",
    "manual_review",
    "reopen_incident",
]


@dataclass(frozen=True)
class CompensationReceipt:
    decision_id: str
    plan: str
    execution_status: ExecutionStatus
    external_effect_count: int
    user_visible_count: int
    audit_correction_id: str | None


@dataclass(frozen=True)
class ObservedState:
    external_state: MatchState
    local_projection: MatchState
    user_visible_state: MatchState
    audit_trail: MatchState
    residual_risk: ResidualRisk


def reconcile_compensation(
    receipt: CompensationReceipt,
    observed: ObservedState,
) -> RecommendedAction:
    if receipt.execution_status == "failed":
        return "retry_compensation"

    if receipt.execution_status == "blocked":
        return "manual_review"

    if observed.external_state == "mismatch" or observed.local_projection == "mismatch":
        return "reopen_incident"

    if observed.user_visible_state == "mismatch":
        return "notify_user"

    if observed.audit_trail == "mismatch":
        return "append_audit_correction"

    if "unknown" in (
        observed.external_state,
        observed.local_projection,
        observed.user_visible_state,
        observed.audit_trail,
    ):
        return "manual_review"

    if observed.residual_risk in ("medium", "high"):
        return "manual_review"

    return "close_verified"
~~~

这里的设计原则很硬：只要外部状态或本地投影不一致，就不要轻易 close。因为这代表系统和现实世界已经分叉。

---

## 4. learn-claude-code：整次 rollback 的关闭闸门

一条补偿通过，不代表整次 incident 可以关。Close Gate 要聚合所有 affectedDecisionIds。

~~~py
from dataclasses import dataclass
from typing import Literal


Action = Literal[
    "close_verified",
    "retry_compensation",
    "append_audit_correction",
    "notify_user",
    "manual_review",
    "reopen_incident",
]

CloseDecision = Literal[
    "close_verified",
    "close_with_residual_risk",
    "keep_observing",
    "reopen_incident",
]


@dataclass(frozen=True)
class ReconciliationResult:
    decision_id: str
    action: Action
    residual_risk: str


def decide_compensation_close(
    results: list[ReconciliationResult],
    all_affected_decisions_count: int,
) -> tuple[CloseDecision, list[str]]:
    reasons: list[str] = []

    if len(results) < all_affected_decisions_count:
        return "keep_observing", ["missing_reconciliation_results"]

    if any(r.action == "reopen_incident" for r in results):
        return "reopen_incident", ["state_mismatch_after_compensation"]

    if any(r.action in ("retry_compensation", "manual_review") for r in results):
        return "keep_observing", ["compensation_not_fully_resolved"]

    if any(r.action in ("notify_user", "append_audit_correction") for r in results):
        return "keep_observing", ["followup_side_effect_required"]

    medium_risk = [r for r in results if r.residual_risk == "medium"]
    if medium_risk:
        reasons.append("medium_residual_risk_accepted_with_owner")
        return "close_with_residual_risk", reasons

    return "close_verified", ["all_compensations_reconciled"]
~~~

这个函数故意保守：只要还有通知、审计修正、重试、人工确认没完成，就保持 observing，不急着关闭。

---

## 5. pi-mono：CompensationReconciliationWorker

生产版可以把执行和对账拆开，避免 worker 自己宣布自己成功。

~~~ts
type ExecutionStatus = "applied" | "skipped" | "blocked" | "failed"
type MatchState = "match" | "mismatch" | "unknown"
type ReconciliationAction =
  | "close_verified"
  | "retry_compensation"
  | "append_audit_correction"
  | "notify_user"
  | "manual_review"
  | "reopen_incident"

type CompensationReceipt = {
  receiptId: string
  rollbackReceiptId: string
  decisionId: string
  plan: string
  idempotencyKey: string
  executionStatus: ExecutionStatus
  externalEffectRefs: string[]
  localPatchRefs: string[]
  userVisibleEventRefs: string[]
  auditCorrectionId?: string
}

type ObservedFinalState = {
  externalState: MatchState
  localProjection: MatchState
  userVisibleState: MatchState
  auditTrail: MatchState
  residualRisk: "none" | "low" | "medium" | "high"
}

type OutcomeReconciliation = {
  reconciliationId: string
  receiptId: string
  decisionId: string
  observed: ObservedFinalState
  action: ReconciliationAction
  reasons: string[]
}

class CompensationReconciliationWorker {
  constructor(
    private readonly receipts: CompensationReceiptStore,
    private readonly stateProbe: FinalStateProbe,
    private readonly outbox: EventOutbox,
  ) {}

  async run(receiptId: string): Promise<OutcomeReconciliation> {
    const receipt = await this.receipts.get(receiptId)
    const observed = await this.stateProbe.observe({
      decisionId: receipt.decisionId,
      externalEffectRefs: receipt.externalEffectRefs,
      localPatchRefs: receipt.localPatchRefs,
      userVisibleEventRefs: receipt.userVisibleEventRefs,
      auditCorrectionId: receipt.auditCorrectionId,
    })

    const reconciliation = this.decide(receipt, observed)
    await this.receipts.saveReconciliation(reconciliation)

    await this.outbox.publish("guard.compensation.reconciled", {
      rollbackReceiptId: receipt.rollbackReceiptId,
      decisionId: receipt.decisionId,
      action: reconciliation.action,
      reasons: reconciliation.reasons,
    })

    return reconciliation
  }

  private decide(
    receipt: CompensationReceipt,
    observed: ObservedFinalState,
  ): OutcomeReconciliation {
    const reasons: string[] = []
    let action: ReconciliationAction = "close_verified"

    if (receipt.executionStatus === "failed") {
      action = "retry_compensation"
      reasons.push("execution_failed")
    } else if (receipt.executionStatus === "blocked") {
      action = "manual_review"
      reasons.push("execution_blocked")
    } else if (
      observed.externalState === "mismatch" ||
      observed.localProjection === "mismatch"
    ) {
      action = "reopen_incident"
      reasons.push("final_state_mismatch")
    } else if (observed.userVisibleState === "mismatch") {
      action = "notify_user"
      reasons.push("user_visible_state_mismatch")
    } else if (observed.auditTrail === "mismatch") {
      action = "append_audit_correction"
      reasons.push("audit_trail_mismatch")
    } else if (Object.values(observed).includes("unknown")) {
      action = "manual_review"
      reasons.push("unknown_final_state")
    } else if (observed.residualRisk === "medium" || observed.residualRisk === "high") {
      action = "manual_review"
      reasons.push("residual_risk_requires_owner")
    } else {
      reasons.push("final_state_reconciled")
    }

    return {
      reconciliationId: `recon_${receipt.receiptId}`,
      receiptId: receipt.receiptId,
      decisionId: receipt.decisionId,
      observed,
      action,
      reasons,
    }
  }
}
~~~

要点：CompensationWorker 只负责执行补偿，ReconciliationWorker 负责观察最终状态。不要让同一个执行器既当运动员又当裁判。

---

## 6. pi-mono：RollbackCloseGate

等所有 reconciliation 都写入后，再聚合成关闭决策。

~~~ts
type CloseDecision =
  | "close_verified"
  | "close_with_residual_risk"
  | "keep_observing"
  | "reopen_incident"

type RollbackCloseBundle = {
  rollbackReceiptId: string
  affectedDecisionCount: number
  reconciledDecisionCount: number
  closeDecision: CloseDecision
  reasons: string[]
  openActions: ReconciliationAction[]
}

class RollbackCloseGate {
  decide(input: {
    rollbackReceiptId: string
    affectedDecisionCount: number
    reconciliations: OutcomeReconciliation[]
  }): RollbackCloseBundle {
    const actions = input.reconciliations.map((r) => r.action)
    const reasons: string[] = []

    let closeDecision: CloseDecision = "close_verified"

    if (input.reconciliations.length < input.affectedDecisionCount) {
      closeDecision = "keep_observing"
      reasons.push("missing_reconciliation_results")
    } else if (actions.includes("reopen_incident")) {
      closeDecision = "reopen_incident"
      reasons.push("state_mismatch_after_compensation")
    } else if (
      actions.includes("retry_compensation") ||
      actions.includes("manual_review")
    ) {
      closeDecision = "keep_observing"
      reasons.push("compensation_not_fully_resolved")
    } else if (
      actions.includes("notify_user") ||
      actions.includes("append_audit_correction")
    ) {
      closeDecision = "keep_observing"
      reasons.push("followup_side_effect_required")
    } else if (
      input.reconciliations.some(
        (r) => r.observed.residualRisk === "medium",
      )
    ) {
      closeDecision = "close_with_residual_risk"
      reasons.push("medium_residual_risk_accepted")
    } else {
      reasons.push("all_compensations_reconciled")
    }

    return {
      rollbackReceiptId: input.rollbackReceiptId,
      affectedDecisionCount: input.affectedDecisionCount,
      reconciledDecisionCount: input.reconciliations.length,
      closeDecision,
      reasons,
      openActions: [...new Set(actions.filter((a) => a !== "close_verified"))],
    }
  }
}
~~~

这层 gate 输出的是事件级结果，不是单条任务结果。它适合接告警、incident close、postmortem archive 和 learning backfill。

---

## 7. OpenClaw 实战：Cron 任务的外部副作用对账

拿我们这个课程 Cron 举例，每次发课会产生一串外部副作用：

1. Telegram 群消息；
2. lessons 文件；
3. README 目录；
4. TOOLS.md 已讲内容；
5. Git commit；
6. git push 到远端 main；
7. daily memory 记录。

如果某次课程内容发错，rollback 后不能只改本地文件。补偿关闭前要对账：

~~~text
expectedFinalState:
  telegram:
    wrongMessage: deleted_or_corrected
    correctionMessage: sent_if_needed
  git:
    fixCommit: pushed
    remoteMainContainsFix: true
  docs:
    lessonFile: corrected
    readmeEntry: corrected
    toolsTopicList: corrected
  memory:
    dailyNoteContainsReceipt: true

observedFinalState:
  telegram:
    messageId: 12984
    correctionMessageId: 12985
  git:
    localHead: abc123
    remoteMain: abc123
  docs:
    diffCheck: clean
  memory:
    entryWritten: true
~~~

只有这些都对上，才能说这次外部副作用已经闭环。否则只是“我执行过补偿”，不是“补偿已经生效”。

---

## 8. 工程落地检查清单

做 Agent 护栏补偿 close gate 时，至少检查：

- 每个 affectedDecisionId 都有 CompensationReceipt；
- 每个 receipt 都有 idempotencyKey 和执行状态；
- external effect 有可查询引用，不只是一段日志；
- local projection 和 external state 都被 probe 过；
- 用户可见通知有 messageId / eventId / delivery receipt；
- audit correction 关联原 decisionId、rollbackReceiptId 和 compensationReceiptId；
- unknown 状态不能自动 close；
- close_with_residual_risk 必须有 owner、expiry 和后续观察窗口；
- close bundle 要能被后续 learning backfill / recurrence detection 引用。

真正成熟的 Agent rollback close，不是“队列清空了”，而是能把每个受影响决策从错误状态一路证明到最终正确状态。

---

## 9. 小结

这一课的核心：

- **CompensationReceipt** 记录补偿执行事实；
- **OutcomeReconciliation** 证明补偿后的世界是否符合 expected final state；
- **RollbackCloseGate** 聚合所有受影响决策，决定 close、继续观察、人工复核或 reopen；
- **OpenClaw Cron** 这种会发消息、改文件、推 Git 的任务，也要把 Telegram messageId、commit hash、README/TOOLS/memory 状态纳入关闭证据。

一句话记住：**补偿执行只是动作，对账通过才是事实，Close Gate 通过才是事故结束。**
