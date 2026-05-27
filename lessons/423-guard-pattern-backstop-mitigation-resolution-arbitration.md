# 423. Agent 护栏兜底缓解后的处置仲裁（Guard Pattern Backstop Mitigation Resolution Arbitration）

上一课讲了 **Guard Pattern Active Decision Backstop Audit**：Guard Pattern active 后，对高风险决策旁路重算 expected decision，按 false_allow / false_block / unexplained_diff / evidence_gap 分类，并先触发 bounded mitigation。

今天继续往后走一步：**兜底动作已经执行了，系统怎么判断这次异常到底该关闭、继续隔离、回滚 active pattern，还是进入修复队列。**

一句话：**Backstop mitigation 不是最终状态；每个 force_dry_run、temporary_block、quarantine 或 rollback 都要生成 Mitigation Receipt，回填 outcome / blast radius / evidence freshness / recurrence signal，再由 Resolution Arbitration 决定 close_observed、extend_mitigation、rollback_active、open_repair 或 escalate_incident。**

---

## 1. 为什么兜底后还要仲裁

Backstop audit 发现异常时，第一反应是停损，比如：

- false_allow 命中不可逆动作，临时 block action class；
- evidence_gap 命中高风险租户，强制 dry-run；
- 某个 scope 连续异常，先 quarantine；
- false_allow 预算耗尽，回滚 active pattern。

这些动作都只是 **临时控制面变化**。如果没有后续仲裁，会出现几个问题：

1. 临时 block 变成永久 block，没人知道什么时候能解除；
2. scope quarantine 越积越多，系统越来越保守；
3. rollback 后没有把新失败沉淀成 repair ticket；
4. false_block 的用户影响没有统计，误伤被当成“安全”；
5. 审计器发现的证据缺口没有补齐，下次还会重复触发。

所以 mitigation 后必须有 resolution arbitration：把临时缓解动作变成可关闭、可延长、可回滚、可修复的工程状态。

---

## 2. Mitigation Receipt 的最小结构

兜底动作执行后，至少写一份收据。

~~~text
MitigationReceipt:
  mitigationId
  sourceAuditReceiptId
  patternId
  activeVersion
  action:
    force_dry_run_for_scope |
    temporary_block_action_class |
    quarantine_scope |
    rollback_active_pattern |
    manual_review
  scope:
    tenantId?
    riskSurface
    actionClass?
    runtimeGroup?
  startedAt
  expiresAt?
  blastRadius:
    affectedTenants
    affectedActions
    estimatedBlockedGoodRuns
    preventedRiskyRuns
  evidence:
    evidenceRefs[]
    outcomeRefs[]
    missingEvidence[]
    oldestEvidenceAgeSeconds
  obligations:
    requiresOutcomeBackfill
    requiresRepairTicket
    requiresUserImpactReview
    requiresRegressionCase
~~~

重点是 obligations。兜底动作不能只记录“我 block 了”，还要明确后续欠了哪些账。

---

## 3. learn-claude-code：处置仲裁纯函数

教学版先写一个纯函数，把 mitigation 的状态压成下一步动作。

~~~py
from dataclasses import dataclass
from typing import Literal


MitigationAction = Literal[
    "force_dry_run_for_scope",
    "temporary_block_action_class",
    "quarantine_scope",
    "rollback_active_pattern",
    "manual_review",
]

Resolution = Literal[
    "close_observed",
    "extend_mitigation",
    "rollback_active",
    "open_repair_ticket",
    "escalate_incident",
    "manual_review",
]


@dataclass(frozen=True)
class ResolutionInput:
    mitigation_action: MitigationAction
    audit_class: str
    mitigation_age_minutes: int
    expires_in_minutes: int | None
    outcome_backfilled: bool
    repair_ticket_opened: bool
    regression_case_added: bool
    recent_recurrences: int
    prevented_risky_runs: int
    estimated_blocked_good_runs: int
    missing_evidence_count: int
    false_allow_budget_exhausted: bool
    user_impact_review_done: bool


def arbitrate_resolution(input: ResolutionInput) -> tuple[Resolution, list[str]]:
    if input.recent_recurrences >= 3:
        return "escalate_incident", ["recurrence_budget_exhausted"]

    if input.false_allow_budget_exhausted:
        return "rollback_active", ["false_allow_budget_exhausted"]

    if input.missing_evidence_count > 0:
        return "extend_mitigation", ["evidence_backfill_incomplete"]

    if not input.outcome_backfilled:
        return "extend_mitigation", ["outcome_backfill_required"]

    if input.audit_class == "false_allow":
        if not input.repair_ticket_opened or not input.regression_case_added:
            return "open_repair_ticket", ["false_allow_requires_repair_and_replay"]
        return "close_observed", ["risk_contained_and_repair_tracked"]

    if input.audit_class == "false_block":
        if input.estimated_blocked_good_runs > 0 and not input.user_impact_review_done:
            return "manual_review", ["false_block_user_impact_unknown"]
        return "close_observed", ["false_block_impact_reviewed"]

    if input.audit_class in ("evidence_gap", "stale_evidence"):
        if input.mitigation_action == "force_dry_run_for_scope":
            return "close_observed", ["evidence_refreshed_under_dry_run"]
        return "manual_review", ["evidence_issue_under_stronger_mitigation"]

    return "close_observed", ["mitigation_observed_no_open_obligations"]
~~~

这个函数的核心不是复杂，而是明确顺序：

1. 先看复发和预算是否已经必须升级；
2. 再看证据和 outcome 欠账是否补齐；
3. 再按 false_allow / false_block / evidence_gap 分不同处置路径；
4. 最后才能关闭。

---

## 4. learn-claude-code：缓解收据关闭闸门

仲裁结果是 close_observed 时，也不要直接删掉 mitigation。先过一个关闭闸门。

~~~py
from dataclasses import dataclass


@dataclass(frozen=True)
class CloseGateInput:
    resolution: str
    active_mitigation_removed: bool
    receipt_archived: bool
    outcome_refs_count: int
    decision_replay_refs_count: int
    next_watch_window_minutes: int


def can_close_mitigation(input: CloseGateInput) -> tuple[bool, list[str]]:
    reasons: list[str] = []

    if input.resolution != "close_observed":
        reasons.append("resolution_not_close")

    if not input.active_mitigation_removed:
        reasons.append("mitigation_still_active")

    if not input.receipt_archived:
        reasons.append("receipt_not_archived")

    if input.outcome_refs_count == 0:
        reasons.append("missing_outcome_refs")

    if input.decision_replay_refs_count == 0:
        reasons.append("missing_replay_refs")

    if input.next_watch_window_minutes < 30:
        reasons.append("watch_window_too_short")

    return (len(reasons) == 0, reasons or ["close_gate_passed"])
~~~

关闭不是“没有报警”。关闭是：临时控制面已经解除或正式转正、证据归档、outcome 已回填、replay case 能覆盖，并且后续观察窗口已经安排。

---

## 5. pi-mono：BackstopMitigationResolutionWorker

生产版可以把仲裁做成 event worker，订阅 mitigation receipt 和 outcome backfill。

~~~ts
type ResolutionDecision =
  | "close_observed"
  | "extend_mitigation"
  | "rollback_active"
  | "open_repair_ticket"
  | "escalate_incident"
  | "manual_review"

type MitigationResolutionReceipt = {
  type: "guard_pattern.mitigation_resolution_receipt"
  mitigationId: string
  sourceAuditReceiptId: string
  patternId: string
  activeVersion: number
  resolution: ResolutionDecision
  reasons: string[]
  obligationsClosed: string[]
  obligationsOpen: string[]
  evidenceRefs: string[]
  outcomeRefs: string[]
  createdAt: string
}

class BackstopMitigationResolutionWorker {
  constructor(
    private readonly mitigationStore: MitigationStore,
    private readonly outcomeStore: OutcomeStore,
    private readonly recurrenceIndex: RecurrenceIndex,
    private readonly repairQueue: RepairQueue,
    private readonly patternPointer: PatternPointerService,
    private readonly receiptStore: ReceiptStore,
    private readonly eventBus: EventBus,
  ) {}

  async handle(mitigationId: string) {
    const mitigation = await this.mitigationStore.get(mitigationId)
    const outcomes = await this.outcomeStore.findByMitigation(mitigationId)
    const recurrences = await this.recurrenceIndex.countRecent({
      patternId: mitigation.patternId,
      activeVersion: mitigation.activeVersion,
      riskSurface: mitigation.scope.riskSurface,
      actionClass: mitigation.scope.actionClass,
    })

    const decision = arbitrateResolution({
      mitigationAction: mitigation.action,
      auditClass: mitigation.auditClass,
      mitigationAgeMinutes: mitigation.ageMinutes(),
      expiresInMinutes: mitigation.expiresInMinutes(),
      outcomeBackfilled: outcomes.length > 0,
      repairTicketOpened: mitigation.obligations.repairTicketId != null,
      regressionCaseAdded: mitigation.obligations.regressionCaseId != null,
      recentRecurrences: recurrences,
      preventedRiskyRuns: mitigation.blastRadius.preventedRiskyRuns,
      estimatedBlockedGoodRuns: mitigation.blastRadius.estimatedBlockedGoodRuns,
      missingEvidenceCount: mitigation.evidence.missingEvidence.length,
      falseAllowBudgetExhausted: mitigation.budget.falseAllowExhausted,
      userImpactReviewDone: mitigation.obligations.userImpactReviewDone,
    })

    if (decision.resolution === "rollback_active") {
      await this.patternPointer.rollback({
        patternId: mitigation.patternId,
        fromVersion: mitigation.activeVersion,
        reason: "backstop_mitigation_resolution",
        sourceReceiptId: mitigation.sourceAuditReceiptId,
      })
    }

    if (decision.resolution === "open_repair_ticket") {
      await this.repairQueue.enqueue({
        source: "guard_pattern_backstop_mitigation",
        patternId: mitigation.patternId,
        activeVersion: mitigation.activeVersion,
        scope: mitigation.scope,
        evidenceRefs: mitigation.evidence.evidenceRefs,
        outcomeRefs: outcomes.map((outcome) => outcome.id),
      })
    }

    const receipt: MitigationResolutionReceipt = {
      type: "guard_pattern.mitigation_resolution_receipt",
      mitigationId,
      sourceAuditReceiptId: mitigation.sourceAuditReceiptId,
      patternId: mitigation.patternId,
      activeVersion: mitigation.activeVersion,
      resolution: decision.resolution,
      reasons: decision.reasons,
      obligationsClosed: decision.obligationsClosed,
      obligationsOpen: decision.obligationsOpen,
      evidenceRefs: mitigation.evidence.evidenceRefs,
      outcomeRefs: outcomes.map((outcome) => outcome.id),
      createdAt: new Date().toISOString(),
    }

    await this.receiptStore.append(receipt)
    await this.eventBus.publish("guard_pattern.mitigation_resolved", receipt)
  }
}
~~~

注意两点：

- rollback_active 只能通过 pointer service 做 CAS 和 rollback receipt，不能让 worker 直接改数据库字段；
- open_repair_ticket 要带 writeScope / forbiddenScope / regression requirements，避免修复 worker 乱改护栏之外的东西。

---

## 6. OpenClaw 课程 Cron 的对应实践

这套课程 cron 也有 mitigation resolution：

- 如果 Telegram 发出但 git push 失败，mitigation 是“保留 messageId，补 commit/push”，仲裁结果不能 close；
- 如果 lesson 文件写了但 README 没更新，mitigation 是“hold publish 或补目录”，仲裁结果是 extend_mitigation；
- 如果发现选题重复，mitigation 是“停止发送，重选题”，仲裁结果是 open_repair_ticket 或 manual_review；
- 如果 push 成功但远端 main 没包含 commit，mitigation 是“remote reconciliation”，仲裁结果不能 close_observed；
- 只有 lesson path、README、TOOLS、Telegram messageId、commit sha、远端验证都对齐，才能 close。

一个课程 cron 的 resolution receipt 可以这样写：

~~~json
{
  "type": "course.mitigation_resolution_receipt",
  "lesson": 423,
  "sourceAudit": "topic_dedup_and_publish_backstop",
  "mitigation": "record_only",
  "resolution": "close_observed",
  "evidenceRefs": [
    "lessons/423-guard-pattern-backstop-mitigation-resolution-arbitration.md",
    "README.md",
    "TOOLS.md",
    "telegram:12970",
    "git:abc1234"
  ],
  "obligationsOpen": []
}
~~~

这就是自动化任务可靠性的关键：每个外部副作用都要能被现实世界反查，而不是只相信本地“命令执行完了”。

---

## 7. 工程建议

- 每个 mitigation 都必须有 expiresAt 或 reviewAfter，避免临时状态永久化；
- false_allow 优先停损和修复，false_block 优先用户影响评估和补偿路径；
- rollback_active 不是失败，是系统知道当前 active 证据不够安全；
- quarantine scope 要有上限，超过上限说明不是局部问题，要升级为 incident；
- close gate 要要求 outcome backfill 和 replay refs，否则只是把问题从热路径扫到冷角落；
- resolution receipt 要可审计：谁触发、因为什么、关了哪些 obligation、还欠什么。

---

## 8. 记忆点

**Backstop Audit 负责发现异常并先停损；Mitigation Resolution 负责证明停损之后该关闭、延长、回滚还是修复。**

成熟 Agent 的护栏系统不是“发现问题就 block 一下”，而是让每个临时兜底动作都有收据、有期限、有 outcome、有仲裁结果，最后能被安全关闭或进入可追踪的修复闭环。
