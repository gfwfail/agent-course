# 453. Agent 退役护栏归档披露争议与 Legal Hold 仲裁（Retired Guard Archive Disclosure Dispute & Legal Hold Arbitration）

> 关键词：disclosure dispute、legal hold arbitration、erasure exception、obligation conflict、dispute receipt

上一课讲了外部披露撤回后，Agent 要禁用访问、通知接收方、收集删除证明，并用 RevocationReconciliationReceipt 做最终对账。

今天继续补上一个容易被忽略的现实问题：**接收方不一定会乖乖删除。它可能说自己有 legal hold、监管留存、合同义务、派生报告依赖，或者说你的撤回范围不清楚。**

很多系统会把这类情况丢给人工：

~~~text
recipient says: cannot delete, legal hold
agent says: manual_review
ticket sits for 3 weeks
~~~

这不是治理闭环，只是把风险转移给人。成熟 Agent 应该把争议也做成工程对象：

~~~text
RevocationReconciliationReceipt
  -> DisclosureDisputeCase
  -> HoldClaimPacket
  -> ObligationConflictReview
  -> ArbitrationDecisionReceipt
  -> ExceptionLease / ErasureOrder / ViolationCase
~~~

一句话：**撤回失败不等于流程失败；没有结构化仲裁，才是真失败。**

---

## 1. 为什么需要争议仲裁层

Revocation workflow 处理的是“应该删除并证明删除”。但实际会出现几类冲突：

- 接收方说有 legal hold，不能删除；
- 接收方删除了原始 export，但保留了派生报告；
- 你的 ExportManifest 写的是字段级脱密，对方理解成整包可留存；
- 接收方已下载多次，水印扫描发现疑似二次分发；
- 合同要求 30 天删除，监管要求 7 年保留；
- 同一份证据被多个案件、多个审计任务引用。

如果 Agent 只会输出 manual_review，系统就失去了自动化治理的价值。正确做法是把争议拆成可判定、可租约、可追责的对象。

---

## 2. 核心对象

建议建五类对象：

- **DisclosureDisputeCase**：争议来源、exportId、recipientId、争议类型、当前阻塞点；
- **HoldClaimPacket**：接收方声称不能删除的依据、范围、到期时间、签名和证据 hash；
- **ObligationConflictReview**：把 ExportManifest、RecipientObligationReceipt、法律保留、删除策略放在一起比对；
- **ArbitrationDecisionReceipt**：仲裁结果，必须一次性、可审计、带理由；
- **ExceptionLease**：如果允许临时保留，必须有 scope、expiresAt、monitoringSignals 和 revokeOnBreach。

状态机可以这样写：

~~~text
revocation_blocked
  -> dispute_opened
  -> hold_claim_collected
  -> obligation_conflict_reviewed
  -> arbitration_decided
  -> exception_leased
  -> erasure_ordered
  -> violation_opened
  -> dispute_closed
~~~

注意：legal hold 不是永久豁免。它最多是一个带范围、期限和复核条件的 ExceptionLease。

---

## 3. learn-claude-code：争议仲裁纯函数

教学版先写一个纯函数：输入撤回对账结果、接收方 hold claim、义务条款和观测信号，输出仲裁动作。

~~~python
# learn_claude_code/guard_patterns/disclosure_dispute.py
from dataclasses import dataclass
from typing import Literal

Decision = Literal[
    "grant_exception_lease",
    "order_erasure",
    "open_violation_case",
    "request_more_evidence",
    "manual_legal_review",
]

DisputeType = Literal[
    "legal_hold_claim",
    "derived_artifact_retention",
    "scope_disagreement",
    "watermark_leak",
    "attestation_conflict",
]


@dataclass(frozen=True)
class DisclosureDisputeCase:
    case_id: str
    export_id: str
    recipient_id: str
    dispute_type: DisputeType
    opened_at_epoch: int
    urgent: bool


@dataclass(frozen=True)
class HoldClaimPacket:
    recipient_id: str
    claim_id: str | None
    legal_basis: str | None
    scope_fields: tuple[str, ...]
    expires_at_epoch: int | None
    signed: bool
    evidence_hash: str | None


@dataclass(frozen=True)
class RecipientObligation:
    recipient_id: str
    allowed_retention_fields: tuple[str, ...]
    must_delete_by_epoch: int
    allows_derived_artifacts: bool
    requires_erasure_attestation: bool


@dataclass(frozen=True)
class DisputeObservation:
    watermark_seen_elsewhere: bool
    downloads_after_revocation: int
    attestation_state: str
    derived_artifacts_count: int


@dataclass(frozen=True)
class ArbitrationDecision:
    decision: Decision
    reason: str
    required_actions: tuple[str, ...] = ()
    lease_expires_at_epoch: int | None = None


def arbitrate_disclosure_dispute(
    case: DisclosureDisputeCase,
    hold: HoldClaimPacket | None,
    obligation: RecipientObligation,
    observation: DisputeObservation,
    now_epoch: int,
    max_exception_days: int = 30,
) -> ArbitrationDecision:
    if observation.watermark_seen_elsewhere or observation.downloads_after_revocation > 0:
        return ArbitrationDecision(
            "open_violation_case",
            "post_revocation_distribution_or_access_detected",
            ("freeze_related_exports", "notify_owner", "preserve_forensics"),
        )

    if case.dispute_type == "attestation_conflict":
        return ArbitrationDecision(
            "request_more_evidence",
            "recipient_attestation_conflicts_with_local_observation",
            ("request_signed_scope_statement", "run_watermark_rescan"),
        )

    if hold is None:
        return ArbitrationDecision(
            "request_more_evidence",
            "hold_claim_missing",
            ("request_hold_claim_packet",),
        )

    if not hold.signed or not hold.evidence_hash or not hold.legal_basis:
        return ArbitrationDecision(
            "request_more_evidence",
            "hold_claim_not_verifiable",
            ("request_signed_hold_claim_with_evidence_hash",),
        )

    if hold.expires_at_epoch is None:
        return ArbitrationDecision(
            "manual_legal_review",
            "hold_claim_has_no_expiry",
            ("legal_review_required",),
        )

    if hold.expires_at_epoch <= now_epoch:
        return ArbitrationDecision(
            "order_erasure",
            "hold_claim_expired",
            ("send_erasure_order", "require_erasure_attestation"),
        )

    disallowed_fields = tuple(
        field for field in hold.scope_fields
        if field not in obligation.allowed_retention_fields
    )
    if disallowed_fields:
        return ArbitrationDecision(
            "manual_legal_review",
            "hold_scope_exceeds_recipient_obligation",
            ("review_field_scope", "minimize_retained_packet"),
        )

    if observation.derived_artifacts_count > 0 and not obligation.allows_derived_artifacts:
        return ArbitrationDecision(
            "order_erasure",
            "derived_artifacts_not_allowed_by_obligation",
            ("delete_derived_artifacts", "submit_erasure_attestation"),
        )

    max_lease_seconds = max_exception_days * 24 * 60 * 60
    lease_expires_at = min(hold.expires_at_epoch, now_epoch + max_lease_seconds)
    return ArbitrationDecision(
        "grant_exception_lease",
        "verifiable_hold_claim_within_obligation_scope",
        ("write_exception_lease", "schedule_lease_renewal_review"),
        lease_expires_at,
    )
~~~

这个函数的重点：

- 水印外泄和撤回后下载优先升级 violation；
- legal hold 必须有签名、依据、证据 hash、字段范围和到期时间；
- hold 范围不能超过 RecipientObligation；
- 允许保留也只能生成 ExceptionLease，不能把争议直接 close。

---

## 4. pi-mono：Dispute Worker + Exception Lease

生产版可以把争议处理做成独立 worker。RevocationReconciliationWorker 发现 blocked 后，只负责开 case；DisputeWorker 再做仲裁。

~~~ts
// packages/agent-governance/src/disclosure/dispute-worker.ts
type ArbitrationOutcome =
  | "exception_leased"
  | "erasure_ordered"
  | "violation_opened"
  | "more_evidence_requested"
  | "manual_legal_review";

interface DisputeJob {
  caseId: string;
  requestedBy: string;
}

class DisclosureDisputeWorker {
  constructor(
    private readonly repo: DisclosureRepository,
    private readonly policy: DisclosurePolicyEngine,
    private readonly outbox: Outbox,
  ) {}

  async run(job: DisputeJob): Promise<{ outcome: ArbitrationOutcome }> {
    return this.repo.transaction(async (tx) => {
      const dispute = await tx.disputes.lockForUpdate(job.caseId);
      const obligation = await tx.obligations.getForRecipient(
        dispute.exportId,
        dispute.recipientId,
      );
      const hold = await tx.holdClaims.latestForDispute(dispute.id);
      const observation = await tx.disputeObservations.latest(dispute.id);

      const decision = this.policy.arbitrateDispute({
        dispute,
        obligation,
        hold,
        observation,
      });

      const receipt = await tx.arbitrationReceipts.insert({
        disputeId: dispute.id,
        decision: decision.type,
        reason: decision.reason,
        decidedAt: new Date(),
        decidedBy: job.requestedBy,
        inputHash: decision.inputHash,
      });

      if (decision.type === "grant_exception_lease") {
        await tx.exceptionLeases.insert({
          disputeId: dispute.id,
          recipientId: dispute.recipientId,
          exportId: dispute.exportId,
          scope: decision.scope,
          expiresAt: decision.expiresAt,
          receiptId: receipt.id,
          status: "active",
        });
        await tx.disputes.update(dispute.id, { status: "exception_leased" });
        return { outcome: "exception_leased" };
      }

      if (decision.type === "order_erasure") {
        await this.outbox.enqueue(tx, {
          type: "disclosure.erasure_order",
          dedupeKey: `${dispute.id}:erasure_order:${receipt.id}`,
          payload: {
            disputeId: dispute.id,
            recipientId: dispute.recipientId,
            receiptId: receipt.id,
            requiredBy: decision.requiredBy,
          },
        });
        await tx.disputes.update(dispute.id, { status: "erasure_ordered" });
        return { outcome: "erasure_ordered" };
      }

      if (decision.type === "open_violation_case") {
        await tx.violationCases.insert({
          exportId: dispute.exportId,
          recipientId: dispute.recipientId,
          sourceDisputeId: dispute.id,
          arbitrationReceiptId: receipt.id,
          severity: decision.severity,
        });
        await tx.disputes.update(dispute.id, { status: "violation_opened" });
        return { outcome: "violation_opened" };
      }

      await tx.disputes.update(dispute.id, { status: decision.type });
      return { outcome: decision.type };
    });
  }
}
~~~

这里的关键是把 ExceptionLease 当成活对象：

- 有 expiresAt；
- 有 scope；
- 有 receiptId；
- 有 renewal review；
- breach 后能自动转 violation。

不要把 legal hold 写成 export.status = retained_forever。

---

## 5. OpenClaw：cron 里的争议处理

在 OpenClaw 课程 cron 这种多副作用任务里，也能看到同样模式：

~~~text
lesson published
  -> Telegram sent
  -> README updated
  -> TOOLS updated
  -> git pushed
  -> receipt checked
~~~

如果其中某步出现外部状态冲突，比如 Telegram 已发送但 git push 失败，就不能简单“重跑全部”。要开一个 structured case：

~~~text
CoursePublishDisputeCase
  export: lesson topic + messageId + commit
  conflict: message_sent_but_repo_not_pushed
  arbitration:
    - retry_push
    - amend_receipt
    - send_correction
    - manual_review
~~~

这和 disclosure dispute 是同一个思想：**外部现实和本地账本不一致时，先结构化争议，再做最小补偿。**

---

## 6. 实战检查清单

给披露治理加争议仲裁时，至少检查 8 件事：

- RevocationReconciliationReceipt 能不能自动开 DisclosureDisputeCase？
- legal hold claim 是否必须签名、带证据 hash、字段范围和到期时间？
- RecipientObligationReceipt 是否能约束 hold 的最大保留范围？
- 派生产物是否和原始 export 分开处理？
- ExceptionLease 是否有 expiresAt、scope、renewal review 和 breach action？
- 水印外泄是否绕过普通争议，直接开 violation？
- 仲裁 decision 是否写入不可变 ArbitrationDecisionReceipt？
- 关闭 dispute 前，是否证明 erasure order、exception lease 或 violation case 已进入终态？

---

## 7. 反模式

三个常见坑：

1. **把 legal hold 当免死金牌**
   Legal hold 只能说明暂时不能删某些东西，不代表可以无限期、无限范围保留全部导出包。

2. **让人工 review 吞掉状态机**
   人工可以做判断，但判断结果必须回写 ArbitrationDecisionReceipt，否则系统无法审计。

3. **派生产物不建模**
   很多泄露不是原始文件没删，而是报告、截图、缓存、索引、embedding 还在。

---

## 8. 一句话总结

**撤回后的争议不是流程异常，而是治理系统的正常分支。成熟 Agent 会把 legal hold、派生产物、义务冲突和水印观测放进同一个仲裁收据链里，让“不能删”也变成有范围、有期限、有证据的工程状态。**
