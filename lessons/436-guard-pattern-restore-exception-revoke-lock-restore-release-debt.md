# 436. Agent 护栏恢复例外收回后的锁恢复与发布债务清算

> **核心思想：RestoreException 被 revoke 不是删除一条白名单记录就完事。成熟 Agent 要把 family lock 恢复为阻断态，冻结受影响 release lane，回填例外期间的真实决策，把未关闭的补证据、补偿、重放和 owner review 记录成 Release Debt，直到债务清零才能再次申请解锁。**

---

## 1. 为什么 revoke 后还要做 debt settlement

上一课讲 RestoreException 要像 lease 一样过期、再验证、自动收回。

这里最容易犯的错是：检测到 bad diff 或 recurrence signal 后，只把 exception.status 改成 revoked。

这会留下四类隐患：

1. release gate 可能已经缓存过 exception，继续放行同一族 candidate；
2. exception 生效期间已经影响了一批 shadow / canary / active 决策；
3. family lock 的 forbidden signature 可能还没扩展到这次新失败；
4. owner 只看到“已 revoke”，但不知道还有多少补偿、重放、审计没有关闭。

所以 revoke 必须生成一个 LockRestoreReceipt 和 ReleaseDebtLedger：

- LockRestoreReceipt 证明 family lock 已恢复阻断；
- ReleaseDebtLedger 记录 revoke 后必须清算的工程债；
- release gate 必须拒绝带未清 debt 的同族 candidate 再次解锁。

---

## 2. 最小数据模型

~~~text
RestoreExceptionRevocation:
  exceptionId
  familyLockId
  candidateVersion
  revokedAt
  reason:
    lease_expired |
    bad_diff_budget_exceeded |
    evidence_expansion_budget_exceeded |
    dependency_fingerprint_changed |
    recurrence_signal_hit |
    locked_regression_seed_failed
  affectedDecisionIds[]
  latestForbiddenSignature

LockRestoreReceipt:
  familyLockId
  restoredAt
  blockedCandidateVersions[]
  releaseLanesFrozen[]
  gateCacheInvalidated: true | false
  forbiddenSignatureExtended: true | false
  recurrenceSentinelArmed: true | false

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
  status: open | verified | waived
  blocksUnlock: true | false

ReleaseDebtLedger:
  familyLockId
  openDebtItems[]
  unlockBlocked: true | false
~~~

这套模型把 revoke 从“状态字段变化”升级成“发布系统重新回到可信阻断态”的收据链。

---

## 3. revoke 后必须完成的五件事

第一，恢复 family lock 阻断态。
把 revoked exception 从 allow path 移出，并确保同一 candidate family 回到 locked。这里要用原子更新或 CAS，避免一个 worker revoke，另一个 worker 同时 promote。

第二，冻结相关 release lane。
只冻结相同 familyLockId / riskSurface / actionClass 的 lane，不要把全站发布都停了。冻结范围要小，但要可证明。

第三，失效 release gate 缓存。
如果 gate 有本地缓存、runtime cache 或 prompt pack cache，LockRestoreReceipt 必须记录 cache invalidation probe。否则控制面 revoke 了，执行面还在用旧例外。

第四，回填 affected decisions。
exception 生效期间所有真实影响过的 decisionId 要进入 outcome backfill。若已经产生外部副作用，再进入 compensation queue。

第五，创建 release debt。
revoke 后还没完成的 replay、evidence review、owner review、cache probe，都要变成 debt item。下次 unlock review 先看 debt ledger，不清零不允许恢复。

---

## 4. learn-claude-code：revoke 后债务分类纯函数

教学版先写一个纯函数：输入 revocation、lock restore 状态和 affected decision outcome，输出还欠哪些 release debt。

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


@dataclass(frozen=True)
class RestoreExceptionRevocation:
    exception_id: str
    family_lock_id: str
    candidate_version: str
    reason: str
    affected_decision_ids: tuple[str, ...]
    forbidden_signature_extended: bool


@dataclass(frozen=True)
class LockRestoreState:
    lock_restored: bool
    release_lanes_frozen: bool
    gate_cache_invalidated: bool
    recurrence_sentinel_armed: bool


@dataclass(frozen=True)
class AffectedDecisionOutcome:
    decision_id: str
    outcome_known: bool
    external_side_effect: bool
    compensation_verified: bool


@dataclass(frozen=True)
class ReleaseDebtItem:
    kind: DebtKind
    reason: str
    blocks_unlock: bool


def build_release_debt(
    revocation: RestoreExceptionRevocation,
    restore: LockRestoreState,
    outcomes: tuple[AffectedDecisionOutcome, ...],
) -> tuple[ReleaseDebtItem, ...]:
    debts: list[ReleaseDebtItem] = []

    if not restore.lock_restored or not restore.release_lanes_frozen:
        debts.append(ReleaseDebtItem(
            "owner_review",
            "lock_or_lane_not_fully_restored",
            True,
        ))

    if not restore.gate_cache_invalidated:
        debts.append(ReleaseDebtItem(
            "cache_invalidation_probe",
            "release_gate_cache_may_still_allow_exception",
            True,
        ))

    if not restore.recurrence_sentinel_armed:
        debts.append(ReleaseDebtItem(
            "replay_extension",
            "recurrence_sentinel_missing",
            True,
        ))

    if not revocation.forbidden_signature_extended:
        debts.append(ReleaseDebtItem(
            "replay_extension",
            "forbidden_signature_not_extended",
            True,
        ))

    unknown = [o for o in outcomes if not o.outcome_known]
    if unknown:
        debts.append(ReleaseDebtItem(
            "outcome_backfill",
            f"{len(unknown)}_affected_decisions_missing_outcome",
            True,
        ))

    uncompensated = [
        o for o in outcomes
        if o.external_side_effect and not o.compensation_verified
    ]
    if uncompensated:
        debts.append(ReleaseDebtItem(
            "compensation",
            f"{len(uncompensated)}_side_effects_need_compensation",
            True,
        ))

    if revocation.reason in {
        "evidence_expansion_budget_exceeded",
        "dependency_fingerprint_changed",
    }:
        debts.append(ReleaseDebtItem(
            "evidence_boundary_review",
            f"revoked_by_{revocation.reason}",
            True,
        ))

    if revocation.reason in {
        "recurrence_signal_hit",
        "locked_regression_seed_failed",
    }:
        debts.append(ReleaseDebtItem(
            "owner_review",
            f"high_risk_revocation_{revocation.reason}",
            True,
        ))

    return tuple(debts)
~~~

这里的重点不是分类有多复杂，而是 revoke 后“是否还能解锁”变成可计算条件。

---

## 5. pi-mono：用事件流恢复锁并写 ReleaseDebtLedger

生产实现里不要让 release gate 自己顺手做所有事。更稳的做法是：RestoreExceptionRevoked 事件进入一个 worker，worker 只负责收据和债务账本。

~~~ts
type RestoreExceptionRevoked = {
  type: "guard.restore_exception.revoked";
  exceptionId: string;
  familyLockId: string;
  candidateVersion: string;
  reason: string;
  affectedDecisionIds: string[];
  forbiddenSignature: string;
};

type ReleaseDebtKind =
  | "outcome_backfill"
  | "compensation"
  | "replay_extension"
  | "evidence_boundary_review"
  | "owner_review"
  | "cache_invalidation_probe";

type ReleaseDebtItem = {
  debtId: string;
  familyLockId: string;
  sourceExceptionId: string;
  kind: ReleaseDebtKind;
  blocksUnlock: boolean;
  status: "open" | "verified" | "waived";
};

class RestoreExceptionRevocationWorker {
  constructor(
    private readonly familyLocks: FamilyLockRegistry,
    private readonly releaseLanes: ReleaseLaneRegistry,
    private readonly gateCache: ReleaseGateCache,
    private readonly outcomes: DecisionOutcomeStore,
    private readonly debtLedger: ReleaseDebtLedger,
  ) {}

  async handle(event: RestoreExceptionRevoked) {
    await this.familyLocks.restoreBlockedState({
      familyLockId: event.familyLockId,
      candidateVersion: event.candidateVersion,
      forbiddenSignature: event.forbiddenSignature,
    });

    await this.releaseLanes.freezeScoped({
      familyLockId: event.familyLockId,
      reason: event.reason,
    });

    const cacheProbe = await this.gateCache.invalidateAndProbe({
      familyLockId: event.familyLockId,
      exceptionId: event.exceptionId,
    });

    const affected = await this.outcomes.loadMany(event.affectedDecisionIds);
    const debts = classifyReleaseDebt(event, affected, cacheProbe).map((d) => ({
      ...d,
      familyLockId: event.familyLockId,
      sourceExceptionId: event.exceptionId,
      status: "open" as const,
    }));

    await this.debtLedger.append({
      familyLockId: event.familyLockId,
      sourceExceptionId: event.exceptionId,
      items: debts,
      unlockBlocked: debts.some((d) => d.blocksUnlock),
    });
  }
}
~~~

这段逻辑要做成幂等：同一个 exceptionId 的 revoke 事件重复投递时，只能更新同一组 debt item，不能重复冻结 lane 或重复补偿。

---

## 6. OpenClaw 实战：课程 Cron 的 release debt

拿这个课程 Cron 举例：

- RestoreExceptionRevoked：发现临时允许的题目族又命中重复主题；
- affectedDecisionIds：已经写入的 lesson 文件、README 目录项、TOOLS 已讲内容、Telegram messageId、Git commit；
- LockRestoreReceipt：后续选题 gate 恢复阻断同类主题；
- ReleaseDebtLedger：
  - README 是否撤回或修正；
  - Telegram 是否需要补充说明；
  - TOOLS 已讲列表是否记录新 forbidden signature；
  - Git commit 是否需要 revert / follow-up；
  - memory 是否记录这次重复原因。

关键点是：外部副作用一旦发生，revoke 不能只改本地状态。Telegram、Git、README、TOOLS、memory 都是现实世界的一部分，必须进入 outcome reconciliation。

---

## 7. 实战检查清单

- RestoreException revoke 后是否写 LockRestoreReceipt？
- family lock 是否恢复阻断，而不是只删除 exception？
- release gate / runtime / prompt pack cache 是否失效并探测？
- affected decisions 是否进入 outcome backfill？
- 有外部副作用的 decision 是否进入 compensation queue？
- forbidden signature / recurrence sentinel 是否扩展？
- ReleaseDebtLedger 是否阻止同族 candidate 再次 unlock？
- debt item 是否可验证关闭，而不是靠人工口头确认？

---

## 8. 一句话总结

RestoreException revoke 是恢复控制权的开始，不是结束。成熟 Agent 要把“临时例外失效”落实成锁恢复、缓存失效、影响回填、补偿队列和 release debt 清算；只有债务清零，下一次解锁才不是带病发布。
