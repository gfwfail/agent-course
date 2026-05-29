# 438. Agent 护栏解锁复核包与决策收据

> **核心思想：DebtCloseReceipt 只证明债务项可以关闭，不能自动解除 family lock。成熟 Agent 要把关闭收据、回归证据、范围差异和 owner quorum 组装成 UnlockReviewPacket，再由一次性 UnlockDecisionReceipt 决定是 staged_restore、hold、reject 还是 escalate。**

---

## 1. 为什么 debt 清零后还不能直接解锁

上一课讲 ReleaseDebtLedger 的关闭：每个 debt item 都要有 close proof，最后由 UnlockEligibilityGate 判断是否具备提交 unlock review 的资格。

这里最容易犯的错误是：看到 blockingDebtIds 为空，就把 CandidateFamilyLock 改成 unlocked。

这会绕过三个关键判断：

1. close receipt 是否仍然新鲜，证据有没有过期或被撤销；
2. 这次 unlock request 和上一次失败候选相比，scope、predicate、dependency 有没有偷偷变宽；
3. owner approval 是否覆盖当前风险，而不是沿用旧批准。

所以 unlock 必须是一张复核包，而不是一个布尔字段。

可以把流程拆成四层：

- DebtCloseReceipt：单项债务关闭证明；
- UnlockEligibilityReceipt：允许提交复核；
- UnlockReviewPacket：本次复核证据包；
- UnlockDecisionReceipt：最终放行、搁置、拒绝或升级的不可变决策。

---

## 2. 最小数据模型

~~~text
UnlockReviewPacket:
  requestId
  familyLockId
  candidateFamilyId
  candidateVersion
  basedOnEligibilityReceiptId
  debtCloseReceiptIds[]
  regressionReplayRef
  scopeDiffRef
  dependencyFingerprint
  ownerQuorum:
    requiredRoles[]
    approvals[]
  receiptFreshness:
    maxAgeHours
    staleReceiptIds[]
  requestedRestoreStage:
    shadow | canary | active

UnlockDecisionReceipt:
  requestId
  decidedAt
  decision:
    staged_restore |
    hold_for_fresh_evidence |
    reject_unlock |
    escalate_owner_review
  allowedStage:
    none | shadow | canary | active
  reason
  consumedReceiptIds[]
  residualRisk
  nextReviewAfter
  restorePermitId?
~~~

关键点：UnlockDecisionReceipt 要消费 receipt。被消费过的 debt close receipt 不能被复制到另一个 candidate family 里继续解锁。否则一次补证据会变成多个发布路径的通行证。

---

## 3. 复核包必须检查什么

第一，检查资格链路。

UnlockReviewPacket 必须引用同一个 familyLockId 的 UnlockEligibilityReceipt，而且 eligibility 本身仍然有效。不要允许手动拼接不同 lock、不同 candidate family 的 receipt。

第二，检查关闭收据的新鲜度。

compensation receipt、runtime cache probe、owner approval 都有时间属性。过期不是失败，但只能 hold_for_fresh_evidence，不能 staged_restore。

第三，检查 scope diff。

candidateVersion 的 riskSurface、actionClass、tenant scope、tool capability 必须是等宽或更窄。任何变宽都需要新 review，不能继承旧 tombstone/unseal/restore 证据。

第四，检查 regression replay。

locked regression seeds、forbidden signatures、相邻风险样本都要重新跑。通过率不够时，reject_unlock；证据缺失时，hold。

第五，检查 owner quorum。

高风险恢复不能只有一个泛泛 approval。approval 要绑定 requestId、allowedStage、expiresAt 和 residualRisk。批准 shadow 不等于批准 active。

---

## 4. learn-claude-code：纯函数 unlock review gate

教学版先写成纯函数。它不做网络请求，只判断当前 packet 是否可以生成 restore permit。

~~~py
from dataclasses import dataclass
from typing import Literal


Decision = Literal[
    "staged_restore",
    "hold_for_fresh_evidence",
    "reject_unlock",
    "escalate_owner_review",
]

Stage = Literal["none", "shadow", "canary", "active"]
Risk = Literal["none", "low", "medium", "high"]


@dataclass(frozen=True)
class UnlockReviewPacket:
    request_id: str
    family_lock_id: str
    eligibility_family_lock_id: str
    eligibility_valid: bool
    stale_receipt_count: int
    missing_debt_receipt_count: int
    scope_widened: bool
    dependency_changed: bool
    regression_required_count: int
    regression_passed_count: int
    owner_roles_required: set[str]
    owner_roles_approved: set[str]
    requested_stage: Stage
    residual_risk: Risk


@dataclass(frozen=True)
class UnlockDecisionReceipt:
    request_id: str
    decision: Decision
    allowed_stage: Stage
    reason: str
    residual_risk: Risk
    restore_permit_id: str | None


def evaluate_unlock_review(packet: UnlockReviewPacket) -> UnlockDecisionReceipt:
    if packet.family_lock_id != packet.eligibility_family_lock_id:
        return UnlockDecisionReceipt(
            packet.request_id,
            "reject_unlock",
            "eligibility_receipt_family_lock_mismatch",
            "none",
            "high",
            None,
        )

    if not packet.eligibility_valid:
        return UnlockDecisionReceipt(
            packet.request_id,
            "hold_for_fresh_evidence",
            "eligibility_receipt_expired_or_revoked",
            "none",
            "medium",
            None,
        )

    if packet.missing_debt_receipt_count > 0:
        return UnlockDecisionReceipt(
            packet.request_id,
            "hold_for_fresh_evidence",
            "blocking_debt_receipts_missing",
            "none",
            "high",
            None,
        )

    if packet.stale_receipt_count > 0 or packet.dependency_changed:
        return UnlockDecisionReceipt(
            packet.request_id,
            "hold_for_fresh_evidence",
            "fresh_probe_required_before_unlock",
            "none",
            "medium",
            None,
        )

    if packet.scope_widened:
        return UnlockDecisionReceipt(
            packet.request_id,
            "reject_unlock",
            "candidate_scope_widened_since_lock",
            "none",
            "high",
            None,
        )

    if packet.regression_passed_count < packet.regression_required_count:
        return UnlockDecisionReceipt(
            packet.request_id,
            "reject_unlock",
            "locked_regression_replay_failed",
            "none",
            "high",
            None,
        )

    missing_roles = packet.owner_roles_required - packet.owner_roles_approved
    if missing_roles:
        return UnlockDecisionReceipt(
            packet.request_id,
            "escalate_owner_review",
            "owner_quorum_incomplete",
            "none",
            "medium",
            None,
        )

    if packet.requested_stage == "active" and packet.residual_risk in {"medium", "high"}:
        return UnlockDecisionReceipt(
            packet.request_id,
            "staged_restore",
            "active_request_downgraded_to_canary_due_to_residual_risk",
            "canary",
            packet.residual_risk,
            f"restore-permit:{packet.request_id}:canary",
        )

    return UnlockDecisionReceipt(
        packet.request_id,
        "staged_restore",
        "unlock_review_passed",
        packet.requested_stage,
        packet.residual_risk,
        f"restore-permit:{packet.request_id}:{packet.requested_stage}",
    )
~~~

这段代码的重点不是复杂，而是顺序：先证明资格链路，再证明证据新鲜，再证明范围没变宽，最后才看 owner quorum 和恢复阶段。

---

## 5. pi-mono：UnlockReviewWorker + 一次性 RestorePermit

生产系统里建议把 restore permit 当成一次性能力令牌。发布 worker 只能消费 permit，不能重新解释 review packet。

~~~ts
type Stage = "none" | "shadow" | "canary" | "active";

type UnlockReviewPacket = {
  requestId: string;
  familyLockId: string;
  candidateVersion: string;
  eligibilityReceiptId: string;
  debtCloseReceiptIds: string[];
  regressionReplayRef: string;
  scopeDiffRef: string;
  dependencyFingerprint: string;
  requestedStage: Exclude<Stage, "none">;
};

type UnlockDecisionReceipt = {
  requestId: string;
  decision:
    | "staged_restore"
    | "hold_for_fresh_evidence"
    | "reject_unlock"
    | "escalate_owner_review";
  allowedStage: Stage;
  reason: string;
  consumedReceiptIds: string[];
  restorePermitId?: string;
};

type RestorePermit = {
  permitId: string;
  requestId: string;
  familyLockId: string;
  candidateVersion: string;
  allowedStage: Exclude<Stage, "none">;
  expiresAt: string;
  consumed: boolean;
};

class UnlockReviewWorker {
  constructor(
    private receipts: ReceiptStore,
    private permits: RestorePermitStore,
    private bus: EventBus,
  ) {}

  async review(packet: UnlockReviewPacket): Promise<UnlockDecisionReceipt> {
    return this.receipts.transaction(async (tx) => {
      const eligibility = await tx.getEligibility(packet.eligibilityReceiptId);
      const debtReceipts = await tx.getDebtCloseReceipts(packet.debtCloseReceiptIds);

      const receiptIds = [
        packet.eligibilityReceiptId,
        ...packet.debtCloseReceiptIds,
      ];

      if (await tx.anyConsumed(receiptIds)) {
        return this.emit(packet, {
          requestId: packet.requestId,
          decision: "reject_unlock",
          allowedStage: "none",
          reason: "unlock_receipt_already_consumed",
          consumedReceiptIds: [],
        });
      }

      const gate = evaluateUnlockPacket({
        packet,
        eligibility,
        debtReceipts,
        replay: await tx.getReplay(packet.regressionReplayRef),
        scopeDiff: await tx.getScopeDiff(packet.scopeDiffRef),
      });

      if (gate.decision !== "staged_restore") {
        return this.emit(packet, {
          requestId: packet.requestId,
          decision: gate.decision,
          allowedStage: "none",
          reason: gate.reason,
          consumedReceiptIds: [],
        });
      }

      await tx.markConsumed(receiptIds, packet.requestId);

      const permit = await this.permits.create({
        requestId: packet.requestId,
        familyLockId: packet.familyLockId,
        candidateVersion: packet.candidateVersion,
        allowedStage: gate.allowedStage,
        expiresAt: new Date(Date.now() + 2 * 60 * 60 * 1000).toISOString(),
        consumed: false,
      });

      return this.emit(packet, {
        requestId: packet.requestId,
        decision: "staged_restore",
        allowedStage: gate.allowedStage,
        reason: gate.reason,
        consumedReceiptIds: receiptIds,
        restorePermitId: permit.permitId,
      });
    });
  }

  private async emit(
    packet: UnlockReviewPacket,
    receipt: UnlockDecisionReceipt,
  ): Promise<UnlockDecisionReceipt> {
    await this.bus.publish("guard.unlock_review.decided", {
      familyLockId: packet.familyLockId,
      candidateVersion: packet.candidateVersion,
      receipt,
    });
    return receipt;
  }
}
~~~

这里的关键是 transaction：

- 检查 receipt 是否已被消费；
- gate 通过后原子标记 consumed；
- 同事务创建 restore permit；
- 下游只认 permitId，不重新拼证据。

---

## 6. OpenClaw 课程 Cron 的实战映射

这条课程流水线本身也有类似结构：

1. 上一课完成后有 messageId、commit、README、TOOLS 更新；
2. 本课开始前检查已讲内容和仓库状态；
3. 写 lesson、更新目录、更新 TOOLS；
4. 发送 Telegram；
5. git add/commit/push；
6. 写入 memory 作为本次发布收据。

如果第 5 步失败，不应该因为第 4 步已发群就把课程算完成。正确做法是生成一个 release debt：Telegram 已外发但 Git 未对账。等补齐 commit/push 后，才允许关闭 debt。

而今天这个主题继续往前一步：即使 release debt 都关了，也不要自动允许下一轮发布复用旧证据。下一轮发布要重新组装 UnlockReviewPacket：引用上一轮的收据、确认它们没过期、确认主题没有变宽、确认 Git/Telegram/TOOLS 三个现实面一致，再进入新一课的发布流程。

---

## 7. 工程落地 checklist

- 不要把 family lock 直接改成 unlocked；只允许 UnlockDecisionReceipt 产生 RestorePermit。
- debt close receipt 要一次性消费，避免同一批证据解锁多个 candidate。
- owner approval 必须绑定 requestId、allowedStage、expiresAt 和 residualRisk。
- active 恢复要比 shadow/canary 更严格；中高残余风险时自动降级到 canary。
- publish worker 只消费 permit，不重新解释 review packet。
- permit 必须有 expiresAt 和 consumed 字段，避免旧 permit 被后续发布复用。
- 所有 hold/reject/escalate 都要有 reason，方便继续补证据，而不是重新猜原因。

---

## 8. 一句话总结

> **Debt 关闭只是“可以申请复核”，UnlockReviewPacket 才是“这次为什么能恢复”的完整证据；成熟 Agent 用一次性 RestorePermit 把复核结论和发布执行分开，避免旧证据变成永久通行证。**
