# 442. Agent 护栏全量恢复后的定版收据与族锁释放闸门

> **核心思想：active 阶段观察通过只说明恢复版本在当前窗口内稳定，不等于可以直接删除 CandidateFamilyLock。成熟 Agent 要先生成 RestoreFinalizationReceipt，证明 active 运行、回归种子、债务、例外、运行时收敛都已关闭，再用 FamilyLockReleaseGate 原子释放族锁。**

---

## 1. 为什么 active 稳定后还不能直接删锁

上一课讲了 CrossStageHandoffReceipt：每次从 shadow 到 canary、从 canary 到 active，都要把 scope、预算、rollback chain 和 runtime fingerprint 交接清楚。

但恢复链路还有最后一个坑：active 观察窗口通过后，很多系统会直接把 family lock 删除，认为“已经恢复了”。

这会留下三个风险：

1. active 只是一个时间窗口内稳定，不代表历史债务都关闭；
2. restore exception、promotion permit、activation lease 等临时通行证可能还在热路径里；
3. family lock 一删，后续 sibling candidate 可能绕过本次恢复证据直接发布。

所以 active close_stable 之后，不能立刻 unlock family。要先生成 RestoreFinalizationReceipt，再经过 FamilyLockReleaseGate。它们回答的是两件事：

- RestoreFinalizationReceipt：这次恢复是否完整定版；
- FamilyLockReleaseReceipt：族锁是否可以被原子释放，释放后还有什么残留约束。

一句话：active 稳定是结果信号，族锁释放是治理动作，中间必须有定版收据。

---

## 2. 最小数据模型

~~~text
RestoreFinalizationReceipt:
  receiptId
  familyLockId
  candidateVersion
  activeObservationDecisionId
  activeWindowId
  stableScopeHash
  runtimeConvergenceHash
  regressionSeedProofRefs[]
  releaseDebtCloseReceiptRefs[]
  expiredRestoreExceptionRefs[]
  revokedTemporaryPermitRefs[]
  rollbackChainArchiveRef
  residualConstraints[]
  finalizedAt

FamilyLockReleaseGate:
  gateId
  familyLockId
  finalizationReceiptId
  requiredOwners[]
  releaseMode:
    full_release | scoped_release | keep_locked
  residualBlockers[]
  nextAuditAt
  decision:
    release_family_lock
    release_scoped_with_constraints
    keep_locked
    reopen_unlock_review

FamilyLockReleaseReceipt:
  receiptId
  familyLockId
  finalizationReceiptId
  releasedScope
  residualConstraints[]
  tombstoneRefs[]
  recurrenceSentinelRefs[]
  releasedAt
~~~

这里最重要的是 residualConstraints。族锁释放不一定是“全解锁”。如果只证明 payment_refund 风险面稳定，就只能 release_scoped_with_constraints；account_delete、external_email_send 之类相邻风险面仍然可以继续 locked。

另外，tombstone 和 recurrence sentinel 不能随锁删除。它们是事故记忆，族锁释放只表示当前 candidate family 可以重新走正常发布流程，不表示历史失败证据失效。

---

## 3. learn-claude-code：定版闸门纯函数

教学版先写一个纯函数，输入 active 观察结果、债务状态和临时许可状态，输出是否允许释放族锁。

~~~py
from dataclasses import dataclass
from typing import Literal


Decision = Literal[
    "release_family_lock",
    "release_scoped_with_constraints",
    "keep_locked",
    "reopen_unlock_review",
]


@dataclass(frozen=True)
class Scope:
    tenants: set[str]
    risk_surfaces: set[str]
    action_classes: set[str]


@dataclass(frozen=True)
class FinalizationEvidence:
    active_close_stable: bool
    runtime_converged: bool
    regression_seed_replayed: bool
    release_debts_closed: bool
    restore_exceptions_expired: bool
    temporary_permits_revoked: bool
    rollback_chain_archived: bool
    requested_release_scope: Scope
    proven_stable_scope: Scope
    residual_constraints: list[str]


@dataclass(frozen=True)
class ReleaseGateResult:
    decision: Decision
    reason: str
    residual_constraints: list[str]


def scope_within(child: Scope, parent: Scope) -> bool:
    return (
        child.tenants.issubset(parent.tenants)
        and child.risk_surfaces.issubset(parent.risk_surfaces)
        and child.action_classes.issubset(parent.action_classes)
    )


def evaluate_family_lock_release(
    evidence: FinalizationEvidence,
) -> ReleaseGateResult:
    if not evidence.active_close_stable:
        return ReleaseGateResult("keep_locked", "active_window_not_stable", [])

    if not evidence.runtime_converged:
        return ReleaseGateResult("reopen_unlock_review", "runtime_not_converged", [])

    if not evidence.regression_seed_replayed:
        return ReleaseGateResult("keep_locked", "missing_regression_seed_replay", [])

    if not evidence.release_debts_closed:
        return ReleaseGateResult("keep_locked", "release_debts_still_open", [])

    if not evidence.restore_exceptions_expired:
        return ReleaseGateResult("keep_locked", "restore_exceptions_still_active", [])

    if not evidence.temporary_permits_revoked:
        return ReleaseGateResult("keep_locked", "temporary_permits_still_valid", [])

    if not evidence.rollback_chain_archived:
        return ReleaseGateResult("keep_locked", "rollback_chain_not_archived", [])

    if not scope_within(evidence.requested_release_scope, evidence.proven_stable_scope):
        return ReleaseGateResult(
            "release_scoped_with_constraints",
            "requested_scope_exceeds_stable_scope",
            evidence.residual_constraints,
        )

    if evidence.residual_constraints:
        return ReleaseGateResult(
            "release_scoped_with_constraints",
            "stable_with_residual_constraints",
            evidence.residual_constraints,
        )

    return ReleaseGateResult(
        "release_family_lock",
        "finalization_evidence_complete",
        [],
    )
~~~

这段函数的重点不是复杂，而是把“能不能删锁”变成可测试规则。以后事故复盘时，可以拿同一份 evidence 重放，证明当时为什么 full release、scoped release 或 keep locked。

---

## 4. pi-mono：原子释放族锁的 worker

生产版要把 release gate 和锁状态更新放在同一个事务里。否则两个 worker 同时跑，一个释放 full scope，一个释放 scoped scope，最后状态会互相覆盖。

~~~ts
type ReleaseMode = "full_release" | "scoped_release" | "keep_locked";

type FamilyLock = {
  familyLockId: string;
  status: "locked" | "released" | "scoped_released";
  lockedScope: {
    tenants: string[];
    riskSurfaces: string[];
    actionClasses: string[];
  };
  version: number;
};

type RestoreFinalizationReceipt = {
  receiptId: string;
  familyLockId: string;
  candidateVersion: string;
  activeObservationDecisionId: string;
  stableScopeHash: string;
  runtimeConvergenceHash: string;
  regressionSeedProofRefs: string[];
  releaseDebtCloseReceiptRefs: string[];
  expiredRestoreExceptionRefs: string[];
  revokedTemporaryPermitRefs: string[];
  rollbackChainArchiveRef: string;
  residualConstraints: string[];
  finalizedAt: string;
};

type FamilyLockReleaseReceipt = {
  receiptId: string;
  familyLockId: string;
  finalizationReceiptId: string;
  releaseMode: ReleaseMode;
  releasedScopeHash: string;
  residualConstraints: string[];
  tombstoneRefs: string[];
  recurrenceSentinelRefs: string[];
  releasedAt: string;
};

class FamilyLockStore {
  async releaseWithCas(args: {
    familyLockId: string;
    expectedVersion: number;
    releaseMode: ReleaseMode;
    receipt: FamilyLockReleaseReceipt;
  }): Promise<void> {
    // 真实实现应在 DB transaction 中做：
    // 1. select lock for update
    // 2. 校验 version/status/finalizationReceiptId
    // 3. update lock status + residual constraints
    // 4. insert immutable release receipt
    // 5. emit FamilyLockReleased event
    throw new Error("implement with transaction and compare-and-swap");
  }
}

async function finalizeFamilyLockRelease(params: {
  lock: FamilyLock;
  finalization: RestoreFinalizationReceipt;
  releaseMode: ReleaseMode;
  releasedScopeHash: string;
  tombstoneRefs: string[];
  recurrenceSentinelRefs: string[];
  store: FamilyLockStore;
}) {
  if (params.releaseMode === "keep_locked") {
    return;
  }

  const receipt: FamilyLockReleaseReceipt = {
    receiptId: crypto.randomUUID(),
    familyLockId: params.lock.familyLockId,
    finalizationReceiptId: params.finalization.receiptId,
    releaseMode: params.releaseMode,
    releasedScopeHash: params.releasedScopeHash,
    residualConstraints: params.finalization.residualConstraints,
    tombstoneRefs: params.tombstoneRefs,
    recurrenceSentinelRefs: params.recurrenceSentinelRefs,
    releasedAt: new Date().toISOString(),
  };

  await params.store.releaseWithCas({
    familyLockId: params.lock.familyLockId,
    expectedVersion: params.lock.version,
    releaseMode: params.releaseMode,
    receipt,
  });
}
~~~

这里的 receipt 必须 immutable。族锁状态可以从 locked 变成 scoped_released 或 released，但 release receipt 不能被覆盖。以后发现释放错了，要新增 ReopenLockReceipt，而不是修改旧收据。

---

## 5. OpenClaw 课程 Cron 实战

这门课本身也有一个小型恢复链路：

1. 先检查 TOOLS.md 已讲内容，避免重复选题；
2. 写 lesson 文件和 README 目录；
3. 更新 TOOLS.md，把本课加入长期已讲列表；
4. 发送 Telegram 课程；
5. git add、commit、push；
6. 把 messageId、commit、主题写回当天 memory。

这对应 RestoreFinalizationReceipt 的思路：课程发布不是“消息发出去了”就结束，而是 Telegram、文件、README、TOOLS、Git、memory 都对齐后，才算本轮课程定版。

如果中途只发了 Telegram 但 git push 失败，下一轮不能假装已经完成；要按 messageId 和文件状态补齐，或者写一个失败收据说明哪些下游没有完成。

---

## 6. 工程落地要点

- active close_stable 只能触发 finalization，不直接删除锁；
- RestoreFinalizationReceipt 要绑定 active observation、runtime convergence、regression seed、debt close、temporary permit cleanup；
- FamilyLockReleaseGate 要支持 scoped release，别把一个风险面稳定误解成全族安全；
- tombstone / recurrence sentinel 要继续保留，它们不是 family lock 的附属品；
- release receipt 要不可变，释放错了用新收据 reopen，不改历史；
- 锁状态更新要 CAS 或事务化，防止并发 worker 互相覆盖。

---

## 7. 一句话总结

> Agent 护栏恢复的终点不是 active 观察窗口变绿，而是生成可重放的定版收据，并用原子释放收据证明族锁为什么、在什么范围、带着哪些残留约束被解除。

