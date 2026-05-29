# 434. Agent 护栏候选族锁定后的解锁复核与分阶段恢复

> **核心思想：CandidateFamilyLock 不是永久封印，也不是管理员点一下“解锁”。成熟 Agent 要把解锁做成一次小型发布：先提交 UnlockReviewRequest，证明 root cause 已修复、regression seed 已通过、evidence contract 没越界，再按 shadow -> canary -> active 分阶段恢复候选族资格，每一步都留下 RestoreReceipt。**

---

## 1. 为什么 family lock 不能手动一键解除

上一课讲了 ResealCase Closeout：复封案例关闭前，要扩展 tombstone、锁住 candidate family、注册 recurrence sentinel，并把 release gate 接到新的 forbidden signature。

问题是：锁住以后怎么办？

如果永远不解锁，系统会把某一类修复路径永久废掉，后续正确修复也无法发布。如果随手解锁，之前复封挡住的坏候选又可能换个名字回来。

所以 CandidateFamilyLock 后面必须接一个 **Unlock Review & Staged Restore**：

1. 先证明这次解锁不是绕过 tombstone；
2. 再证明 sibling candidate 的共同 root cause 已经消失；
3. 最后只恢复一个小 scope，逐步让 shadow、canary、active 重新获得资格。

这本质上是“恢复发布权”的发布流程，而不是权限开关。

---

## 2. 最小数据模型

~~~text
UnlockReviewRequest:
  requestId
  familyLockId
  requestedBy
  candidateFamilyKey
  targetVersions[]
  proposedFix:
    repairTicketId
    rootCauseFixed
    predicateDiffHash
    evidenceContractDiffHash
    dependencyLockHash
  proofBundle:
    regressionSeedIds[]
    counterfactualReplayId
    shadowComparisonId
    ownerApprovalId?
  requestedRestoreScope:
    stages: shadow | canary | active
    tenants[]
    riskSurfaces[]
    actionClasses[]

UnlockReviewDecision:
  decisionId
  familyLockId
  decision:
    reject_unlock |
    require_more_replay |
    restore_shadow |
    restore_canary |
    restore_active |
    manual_review
  allowedScope
  restoreBudget
  reasonCodes[]

FamilyRestoreReceipt:
  receiptId
  familyLockId
  stage
  restoredVersions[]
  allowedScope
  budget
  rollbackTrigger
  expiresAt
~~~

关键点：解锁决策不要直接删除 family lock。更稳的做法是保留 lock，然后在 lock 上追加一组 **restore exception**。这样 release gate 可以同时知道：

- 哪些 signature 仍然禁止；
- 哪些版本在什么 scope 内临时恢复；
- 恢复窗口过期后是否自动回到 locked。

---

## 3. 解锁复核的四个硬条件

第一，root cause repair proof。
如果当初锁定原因是 repair_overfit，就要证明修复不是只让一个失败样本变绿；如果是 evidence_contract_broken，就要证明证据边界被收窄或重新声明。

第二，regression seed replay。
Family lock 关联的 failed seed 必须全部回放。只跑新样本不够，因为这把锁就是从历史失败里来的。

第三，scope diff 不能变宽。
解锁请求可以缩小 tenant、riskSurface、actionClass，但不能趁解锁扩大影响面。要扩大 scope，必须走新的 canary ramp。

第四，restore budget 必须有限。
恢复不能是永久 active。至少要绑定 maxDecisions、maxBadDiff、maxEvidenceExpansion、expiresAt 和 rollbackTrigger。

---

## 4. learn-claude-code：解锁复核纯函数

教学版可以把 UnlockReview 写成纯函数。输入是当前 family lock、修复证明、回放结果和请求 scope；输出不是 boolean，而是 staged restore decision。

~~~py
from dataclasses import dataclass
from typing import Literal


LockReason = Literal[
    "old_failure_recurred",
    "repair_overfit",
    "evidence_contract_broken",
    "runtime_dependency_drift",
]

Decision = Literal[
    "reject_unlock",
    "require_more_replay",
    "restore_shadow",
    "restore_canary",
    "restore_active",
    "manual_review",
]


@dataclass(frozen=True)
class CandidateFamilyLock:
    lock_id: str
    family_key: str
    reason: LockReason
    locked_versions: tuple[str, ...]
    blocked_scopes: tuple[str, ...]
    failed_regression_seed_ids: tuple[str, ...]
    permanent: bool


@dataclass(frozen=True)
class RepairProof:
    root_cause_fixed: bool
    predicate_diff_hash: str | None
    evidence_contract_diff_hash: str | None
    dependency_lock_hash: str | None
    owner_approval_id: str | None


@dataclass(frozen=True)
class ReplayResult:
    covered_seed_ids: tuple[str, ...]
    failed_seed_ids: tuple[str, ...]
    false_allow_count: int
    false_block_count: int
    evidence_expansion_count: int
    unknown_count: int


@dataclass(frozen=True)
class RequestedScope:
    stage: Literal["shadow", "canary", "active"]
    tenants: tuple[str, ...]
    risk_surfaces: tuple[str, ...]
    action_classes: tuple[str, ...]
    max_decisions: int


@dataclass(frozen=True)
class UnlockReviewDecision:
    lock_id: str
    decision: Decision
    allowed_stage: str | None
    max_decisions: int
    reason_codes: tuple[str, ...]


def all_locked_seeds_covered(lock: CandidateFamilyLock, replay: ReplayResult) -> bool:
    return set(lock.failed_regression_seed_ids).issubset(set(replay.covered_seed_ids))


def decide_family_unlock(
    lock: CandidateFamilyLock,
    proof: RepairProof,
    replay: ReplayResult,
    requested_scope: RequestedScope,
) -> UnlockReviewDecision:
    reasons: list[str] = []

    if lock.permanent and not proof.owner_approval_id:
        reasons.append("permanent_lock_requires_owner_approval")

    if not proof.root_cause_fixed:
        reasons.append("root_cause_not_fixed")

    if not all_locked_seeds_covered(lock, replay):
        reasons.append("missing_locked_regression_seed_replay")

    if replay.failed_seed_ids:
        reasons.append("regression_seed_still_failing")

    if replay.evidence_expansion_count > 0:
        reasons.append("evidence_boundary_expanded")

    if replay.false_allow_count > 0:
        reasons.append("false_allow_not_zero")

    if reasons:
        decision: Decision = "require_more_replay"
        if "root_cause_not_fixed" in reasons or "false_allow_not_zero" in reasons:
            decision = "reject_unlock"
        return UnlockReviewDecision(
            lock_id=lock.lock_id,
            decision=decision,
            allowed_stage=None,
            max_decisions=0,
            reason_codes=tuple(reasons),
        )

    if requested_scope.stage == "active":
        return UnlockReviewDecision(
            lock_id=lock.lock_id,
            decision="restore_canary",
            allowed_stage="canary",
            max_decisions=min(requested_scope.max_decisions, 100),
            reason_codes=("active_requires_canary_restore_first",),
        )

    if requested_scope.stage == "canary":
        return UnlockReviewDecision(
            lock_id=lock.lock_id,
            decision="restore_canary",
            allowed_stage="canary",
            max_decisions=min(requested_scope.max_decisions, 100),
            reason_codes=("bounded_canary_restore",),
        )

    return UnlockReviewDecision(
        lock_id=lock.lock_id,
        decision="restore_shadow",
        allowed_stage="shadow",
        max_decisions=min(requested_scope.max_decisions, 500),
        reason_codes=("shadow_restore_only_readonly",),
    )
~~~

这个函数有个很重要的态度：即使请求 active，也不会直接恢复 active，而是降级成 canary restore。因为 family lock 来自复封失败，恢复时要默认保守。

---

## 5. pi-mono：Release Gate 上的 restore exception

生产版不要把解锁做成删除 lock，而是让 release gate 在判断时读取 lock + exceptions。

~~~ts
type RestoreException = {
  exceptionId: string;
  familyLockId: string;
  version: string;
  stage: "shadow" | "canary" | "active";
  allowedTenants: string[];
  allowedRiskSurfaces: string[];
  maxDecisions: number;
  expiresAt: string;
  rollbackTrigger: {
    maxBadDiff: number;
    maxEvidenceExpansion: number;
    maxUnknownOutcome: number;
  };
};

class FamilyLockReleaseGate {
  constructor(
    private readonly lockRegistry: CandidateFamilyLockRegistry,
    private readonly restoreRegistry: RestoreExceptionRegistry,
  ) {}

  async check(input: {
    familyKey: string;
    version: string;
    stage: "shadow" | "canary" | "active";
    tenant: string;
    riskSurface: string;
  }): Promise<"allow" | "block" | "allow_with_restore_budget"> {
    const lock = await this.lockRegistry.findActiveLock(input.familyKey);
    if (!lock) return "allow";

    const exception = await this.restoreRegistry.findActiveException({
      familyLockId: lock.lockId,
      version: input.version,
      stage: input.stage,
      tenant: input.tenant,
      riskSurface: input.riskSurface,
    });

    if (!exception) return "block";

    if (exception.remainingDecisionBudget <= 0) return "block";
    if (new Date(exception.expiresAt).getTime() <= Date.now()) return "block";

    return "allow_with_restore_budget";
  }
}
~~~

这样做的好处是：lock 仍然是默认 deny，exception 只是限时、限域、限预算的恢复窗口。后续 canary 如果再次触发 bad diff，只需要撤销 exception，不需要重新构造整套 tombstone。

---

## 6. OpenClaw 实战：课程 Cron 的“题目族锁”也要能恢复

这套模式放到课程 Cron，就是“不要重复讲旧题，但也不要把一个方向永久封死”。

比如我们已经连续讲了 tombstone、unseal、reseal、family lock。下一次如果还想讲相关主题，就要证明不是换标题重复：

~~~text
TopicFamilyUnlockReview:
  family: guard-pattern-reseal
  previousLessons:
    - 430 tombstone hit unseal
    - 431 scoped trial reseal
    - 432 reseal forensics routing
    - 433 reseal closeout family lock
  proposedLesson: family lock unlock review staged restore
  noveltyProof:
    newLifecycleStage: unlock review after family lock
    newDecisionObject: RestoreException
    newRisk: one-click unlock bypasses tombstone
  decision: allow_shadow_publish
~~~

这就是把选题去重从“字符串不一样”升级成“生命周期阶段不一样”。长期课程最怕旧主题换皮，TopicFamilyUnlockReview 能让下一课明确站在上一课之后，而不是绕回去重讲。

---

## 7. 常见坑

不要删除 family lock 来表示解锁。
删除 lock 会丢掉事故背景，也让 release gate 不知道这次恢复其实带风险。追加 restore exception 更可审计。

不要跳过 shadow restore。
如果一个 family 曾经被复封，说明它在真实路径里失败过。恢复时先 shadow 只读比较，再 canary 小流量，最后 active。

不要让 owner approval 替代 replay。
人工批准只能说明有人愿意承担风险，不能证明 regression seed 已经通过。

不要把 expiresAt 当装饰字段。
Restore exception 过期后必须自动回到 locked，否则临时恢复会变成永久放行。

---

## 8. 小结

Family Lock Unlock Review 解决的是“被锁候选族如何安全恢复发布资格”。

最小闭环是：

1. submit unlock review request；
2. verify root cause repair proof；
3. replay locked regression seeds；
4. issue staged restore decision；
5. append restore exception；
6. release gate 按 exception 限域放行；
7. bad diff 或过期时撤销 exception，回到 locked。

成熟 Agent 不只会踩刹车，也会知道什么时候、以多大范围、带着哪些证据慢慢松开刹车。
