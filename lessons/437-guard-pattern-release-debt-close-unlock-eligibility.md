# 437. Agent 护栏发布债务的验收关闭与解锁资格闸门

> **核心思想：ReleaseDebtLedger 不是待办列表，而是重新获得发布资格前的证据闸门。成熟 Agent 必须让每一项 debt 通过可验证的 close proof，生成 DebtCloseReceipt，再由 UnlockEligibilityGate 判断 family lock 是否可以进入下一轮 unlock review。**

---

## 1. 为什么 debt 不能靠口头关闭

上一课讲 RestoreException 被 revoke 后，要恢复 family lock、冻结 release lane、回填 affected decisions，并创建 ReleaseDebtLedger。

这里最危险的偷懒是：owner 看一眼说“处理完了”，然后把 debt.status 改成 verified。

这会留下三个问题：

1. compensation 是否真的影响了外部系统，没有 proof；
2. replay extension 是否覆盖了失败签名和相邻风险，没有 coverage；
3. cache invalidation 是否真的传到 runtime，没有 runtime fingerprint。

所以 release debt 的关闭必须分两层：

- DebtCloseReceipt：单个 debt item 为什么可以关闭；
- UnlockEligibilityGate：所有阻断性 debt 关闭后，是否允许重新提交 unlock review。

注意：通过 UnlockEligibilityGate 不等于解锁成功，只是允许进入下一轮 UnlockReviewRequest。它不是绿灯，是重新排队资格。

---

## 2. 最小数据模型

~~~text
ReleaseDebtItem:
  debtId
  familyLockId
  sourceExceptionId
  kind:
    outcome_backfill |
    compensation |
    replay_extension |
    evidence_boundary_review |
    owner_review |
    cache_invalidation_probe
  status: open | close_requested | verified | rejected | waived
  blocksUnlock: true | false

DebtCloseProof:
  debtId
  proofType:
    outcome_snapshot |
    compensation_receipt |
    replay_coverage |
    evidence_boundary_report |
    owner_approval |
    runtime_cache_probe
  evidenceRefs[]
  coverage:
    affectedDecisionCount
    verifiedDecisionCount
    residualUnknownCount
  residualRisk: none | low | medium | high
  verifier

DebtCloseReceipt:
  debtId
  closedAt
  decision: verified | rejected | waived
  reason
  proofHash
  residualRisk
  followUpDebtIds[]

UnlockEligibilityReceipt:
  familyLockId
  checkedAt
  eligible: true | false
  blockingDebtIds[]
  residualRiskSummary
  nextAllowedAction:
    submit_unlock_review |
    continue_debt_settlement |
    escalate_owner_review
~~~

关键点：每次关闭都带 proofHash。未来如果 evidence 被撤销、补偿失败、runtime 漂移，可以反查是哪张 close receipt 放行了后续动作。

---

## 3. 每种 debt 的关闭标准

outcome_backfill：
所有 affectedDecisionIds 都要有 outcome snapshot。允许 residualUnknownCount > 0，但这时只能 close_with_residual_risk，并且必须生成 follow-up owner_review。

compensation：
每个外部副作用都要有 compensation receipt，例如消息已编辑、权限已撤销、任务已重跑、账本已修正。只记录“已入队”不能关闭，只能保持 open。

replay_extension：
必须证明新的 replay suite 覆盖原 forbidden signature、locked regression seeds 和相邻风险样本。只修一个失败样本不够。

evidence_boundary_review：
要证明修复没有扩大 raw evidence 读取范围，没有绕过 purpose/scope/tenant 约束。这里最好由 evidence access log 支撑。

owner_review：
owner approval 要绑定具体 debtId、risk summary、residual risk 和 expiry。不要用一句“approved”当永久通行证。

cache_invalidation_probe：
要有 runtime group 的 cache fingerprint。控制面 invalidated 不等于执行面已经收敛。

---

## 4. learn-claude-code：纯函数 close gate

教学版先把 close gate 写成纯函数。输入 debt item 和 proof，输出 close receipt 或 rejected reason。

~~~py
from dataclasses import dataclass
from typing import Literal


DebtKind = Literal[
    "outcome_backfill",
    "compensation",
    "replay_extension",
    "evidence_boundary_review",
    "owner_review",
    "cache_invalidation_probe",
]

DebtStatus = Literal["open", "close_requested", "verified", "rejected", "waived"]
ResidualRisk = Literal["none", "low", "medium", "high"]
CloseDecision = Literal["verified", "rejected", "waived"]


@dataclass(frozen=True)
class ReleaseDebtItem:
    debt_id: str
    family_lock_id: str
    kind: DebtKind
    blocks_unlock: bool
    status: DebtStatus


@dataclass(frozen=True)
class DebtCloseProof:
    debt_id: str
    proof_type: str
    affected_decision_count: int
    verified_decision_count: int
    residual_unknown_count: int
    compensation_applied_count: int
    compensation_required_count: int
    replay_covers_forbidden_signature: bool
    replay_covers_adjacent_risk: bool
    evidence_boundary_expanded: bool
    owner_approved: bool
    runtime_groups_converged: bool
    residual_risk: ResidualRisk


@dataclass(frozen=True)
class DebtCloseReceipt:
    debt_id: str
    decision: CloseDecision
    reason: str
    residual_risk: ResidualRisk
    blocks_unlock_after_close: bool


def evaluate_debt_close(
    item: ReleaseDebtItem,
    proof: DebtCloseProof,
) -> DebtCloseReceipt:
    if item.debt_id != proof.debt_id:
        return DebtCloseReceipt(
            item.debt_id,
            "rejected",
            "proof_debt_id_mismatch",
            "high",
            item.blocks_unlock,
        )

    if item.status not in {"open", "close_requested"}:
        return DebtCloseReceipt(
            item.debt_id,
            "rejected",
            f"invalid_status_{item.status}",
            "medium",
            item.blocks_unlock,
        )

    if item.kind == "outcome_backfill":
        all_known = (
            proof.affected_decision_count == proof.verified_decision_count
            and proof.residual_unknown_count == 0
        )
        if all_known:
            return DebtCloseReceipt(item.debt_id, "verified", "all_outcomes_backfilled", "none", False)
        if proof.owner_approved and proof.residual_risk in {"low", "medium"}:
            return DebtCloseReceipt(item.debt_id, "verified", "closed_with_residual_risk", proof.residual_risk, False)
        return DebtCloseReceipt(item.debt_id, "rejected", "outcome_gap_remains", "high", True)

    if item.kind == "compensation":
        if proof.compensation_required_count == proof.compensation_applied_count:
            return DebtCloseReceipt(item.debt_id, "verified", "all_compensations_applied", proof.residual_risk, False)
        return DebtCloseReceipt(item.debt_id, "rejected", "compensation_not_fully_applied", "high", True)

    if item.kind == "replay_extension":
        if proof.replay_covers_forbidden_signature and proof.replay_covers_adjacent_risk:
            return DebtCloseReceipt(item.debt_id, "verified", "replay_coverage_sufficient", proof.residual_risk, False)
        return DebtCloseReceipt(item.debt_id, "rejected", "replay_coverage_insufficient", "high", True)

    if item.kind == "evidence_boundary_review":
        if not proof.evidence_boundary_expanded:
            return DebtCloseReceipt(item.debt_id, "verified", "evidence_boundary_preserved", proof.residual_risk, False)
        return DebtCloseReceipt(item.debt_id, "rejected", "evidence_boundary_expanded", "high", True)

    if item.kind == "owner_review":
        if proof.owner_approved and proof.residual_risk in {"none", "low", "medium"}:
            return DebtCloseReceipt(item.debt_id, "verified", "owner_approved_bounded_risk", proof.residual_risk, False)
        return DebtCloseReceipt(item.debt_id, "rejected", "owner_approval_missing_or_risk_high", "high", True)

    if item.kind == "cache_invalidation_probe":
        if proof.runtime_groups_converged:
            return DebtCloseReceipt(item.debt_id, "verified", "runtime_cache_converged", "none", False)
        return DebtCloseReceipt(item.debt_id, "rejected", "runtime_cache_not_converged", "high", True)

    return DebtCloseReceipt(item.debt_id, "rejected", "unknown_debt_kind", "high", True)
~~~

这段代码的价值在于：关闭 debt 不是 UI 操作，而是可重放、可测试、可审计的判断。

---

## 5. pi-mono：DebtCloseWorker + UnlockEligibilityGate

生产里建议把单项关闭和整体资格分开。DebtCloseWorker 只处理某个 debt，UnlockEligibilityGate 只看 ledger 的整体状态。

~~~ts
type DebtKind =
  | "outcome_backfill"
  | "compensation"
  | "replay_extension"
  | "evidence_boundary_review"
  | "owner_review"
  | "cache_invalidation_probe";

type ReleaseDebtItem = {
  debtId: string;
  familyLockId: string;
  kind: DebtKind;
  blocksUnlock: boolean;
  status: "open" | "close_requested" | "verified" | "rejected" | "waived";
};

type DebtCloseProof = {
  debtId: string;
  proofHash: string;
  evidenceRefs: string[];
  residualRisk: "none" | "low" | "medium" | "high";
};

type DebtCloseReceipt = {
  debtId: string;
  familyLockId: string;
  decision: "verified" | "rejected" | "waived";
  proofHash: string;
  residualRisk: "none" | "low" | "medium" | "high";
  followUpDebtIds: string[];
};

class DebtCloseWorker {
  constructor(
    private readonly ledger: ReleaseDebtLedger,
    private readonly proofLoader: DebtProofLoader,
    private readonly receipts: DebtCloseReceiptStore,
  ) {}

  async handle(command: { debtId: string }) {
    const debt = await this.ledger.load(command.debtId);
    const proof = await this.proofLoader.loadForDebt(command.debtId);

    const receipt = evaluateDebtClose(debt, proof);

    await this.receipts.append({
      debtId: debt.debtId,
      familyLockId: debt.familyLockId,
      decision: receipt.decision,
      proofHash: proof.proofHash,
      residualRisk: receipt.residualRisk,
      followUpDebtIds: receipt.followUpDebtIds,
    });

    await this.ledger.applyCloseReceipt(receipt);
  }
}

class UnlockEligibilityGate {
  constructor(private readonly ledger: ReleaseDebtLedger) {}

  async check(familyLockId: string) {
    const debts = await this.ledger.listByFamilyLock(familyLockId);

    const blocking = debts.filter((debt) =>
      debt.blocksUnlock &&
      debt.status !== "verified" &&
      debt.status !== "waived"
    );

    const highResidualRisk = debts.some((debt) =>
      debt.status === "verified" &&
      debt.residualRisk === "high"
    );

    if (blocking.length > 0 || highResidualRisk) {
      return {
        eligible: false,
        blockingDebtIds: blocking.map((debt) => debt.debtId),
        nextAllowedAction: "continue_debt_settlement" as const,
      };
    }

    return {
      eligible: true,
      blockingDebtIds: [],
      nextAllowedAction: "submit_unlock_review" as const,
    };
  }
}
~~~

这里有一个工程细节：waived 也必须有 receipt，而且只能由明确策略或 owner approval 产生。没有 proof 的 waived，本质上就是绕过闸门。

---

## 6. OpenClaw 实战：课程 Cron 的 debt close

拿这个课程 Cron 举例，假设某次选题重复并且已经发到了 Telegram：

- outcome_backfill close proof：
  - lesson 文件路径；
  - README 目录项；
  - TOOLS 已讲内容；
  - Telegram messageId；
  - Git commit hash；
- compensation close proof：
  - 如果要修正 Telegram，记录 edit/delete 后的 messageId；
  - 如果要修正 Git，记录 follow-up commit；
- replay_extension close proof：
  - 新增的去重签名；
  - 最近 20 个课程标题回放结果；
  - TOOLS 已讲内容命中结果；
- cache_invalidation_probe close proof：
  - 当前 cron run 读取到最新 README / TOOLS / remote main；
  - git ls-remote 确认远端 commit；
- owner_review close proof：
  - 老板明确允许 residual risk 时，记录 approval scope 和过期时间。

只有这些 proof 都能对上，下一次 cron 才能说：这个 family lock 可以进入 unlock review。否则继续清债，不继续发新课。

---

## 7. 实战检查清单

- 每个 ReleaseDebtItem 是否都有独立 close proof？
- close proof 是否绑定 debtId，而不是泛泛说“已处理”？
- compensation 是否证明外部状态已经改变，而不只是任务入队？
- replay extension 是否覆盖 forbidden signature 和相邻风险？
- evidence boundary review 是否证明没有扩大读取范围？
- cache invalidation 是否覆盖所有 runtime group？
- waived 是否有策略依据、owner approval 和 expiry？
- UnlockEligibilityGate 是否只授予“申请解锁资格”，而不是直接解锁？

---

## 8. 一句话总结

Release debt 的关闭不是把状态改成 done，而是给每一项现实影响补上可验证证据。成熟 Agent 只有在所有阻断性 debt 都被 proof 关闭、残余风险被明确记录后，才允许 candidate family 重新进入 unlock review。
