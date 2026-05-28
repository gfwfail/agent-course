# 429. Agent 护栏回滚关闭后的候选版本墓碑与回归种子（Guard Pattern Rollback Closeout Candidate Tombstone & Regression Seed）

上一课讲了 **Guard Pattern Compensation Outcome Reconciliation & Close Gate**：补偿任务 applied 不等于事故 close，必须把外部状态、本地投影、用户可见结果和审计链全部对账，才能 close_verified。

今天继续往后走一步：**事故关闭以后，怎么防止同一个坏候选版本以后又被“整理代码”“重新跑一次发布”偷偷带回来**。

一句话：**rollback incident close_verified 后，要给失败的 candidate pattern version 写 Candidate Tombstone，并把这次失败沉淀成 Regression Seed。Tombstone 阻止坏版本复活，Regression Seed 保证后续任何 repair、shadow、canary、active promotion 都必须重新证明它不会再犯同一种错。**

---

## 1. 为什么 close_verified 之后还要写墓碑

很多团队在事故关闭后只留下 incident report。问题是：报告是给人看的，发布系统不一定会读。

在 Agent 护栏系统里，这会带来几个真实风险：

1. 某个失败 candidate 分支后来被 rebase，又重新进入 shadow；
2. 另一个 repair 基于同一段 predicate 复制粘贴，绕过了原事故；
3. canary 回滚后，版本 registry 只知道“当前不用它”，不知道“这个版本不能再被用”；
4. 历史 replay pack 没更新，后续候选版本仍然通过旧测试；
5. 人类记得“别这么改”，但自动发布管道没有可执行约束。

所以 close_verified 不是结束，它应该触发两个长期资产：

- **Candidate Tombstone**：记录失败版本、失败原因、禁止复活条件和解除条件；
- **Regression Seed**：把失败决策压缩成后续发布必须跑的最小回归样本。

前者防复活，后者防复发。

---

## 2. 最小数据模型

~~~text
CandidateTombstone:
  tombstoneId
  patternId
  failedVersion
  sourceRollbackReceiptId
  closeBundleId
  failureClass:
    false_allow |
    false_block |
    evidence_overread |
    stale_predicate |
    outcome_mismatch |
    runtime_divergence
  forbiddenReuse:
    predicateHash[]
    evidenceScope[]
    actionClass[]
    riskSurface[]
  unsealRequirements:
    newRepairHypothesisRequired: true
    counterfactualReplayRequired: true
    regressionSeedMustPass: true
    ownerApprovalRequired: true | false
  status:
    active | superseded | unsealed

RegressionSeed:
  seedId
  sourceTombstoneId
  decisionId
  inputFingerprint
  expectedDecision
  mustNotDecision
  evidenceBoundary
  outcomeRef
  minimalReplayPackRef
  addedTo:
    replayMatrix |
    shadowGate |
    canaryDiffBudget |
    activeBackstopAudit

RollbackCloseoutBackfill:
  closeBundleId
  tombstones[]
  regressionSeeds[]
  releaseBlockers[]
  nextAllowedActions[]
~~~

关键点：**Tombstone 不是删除版本，而是给版本 registry 加一个不可忽略的负向事实**。它应该被发布管道、repair gate、shadow evaluator、canary controller 都读取。

---

## 3. learn-claude-code：生成候选版本墓碑

教学版先写纯函数：给 rollback close bundle 和失败证据，决定要不要 tombstone。

~~~py
from dataclasses import dataclass
from typing import Literal


CloseDecision = Literal["close_verified", "close_with_residual_risk", "keep_observing", "reopen_incident"]
FailureClass = Literal[
    "false_allow",
    "false_block",
    "evidence_overread",
    "stale_predicate",
    "outcome_mismatch",
    "runtime_divergence",
]


@dataclass(frozen=True)
class RollbackCloseBundle:
    close_bundle_id: str
    pattern_id: str
    failed_version: str
    close_decision: CloseDecision
    rollback_receipt_id: str
    accepted_residual_risk: str


@dataclass(frozen=True)
class FailureEvidence:
    failure_class: FailureClass
    predicate_hash: str
    evidence_scope: str
    action_class: str
    risk_surface: str
    affected_decision_count: int


@dataclass(frozen=True)
class CandidateTombstone:
    pattern_id: str
    failed_version: str
    source_close_bundle_id: str
    source_rollback_receipt_id: str
    failure_class: FailureClass
    forbidden_predicate_hash: str
    forbidden_evidence_scope: str
    forbidden_action_class: str
    forbidden_risk_surface: str
    owner_approval_required: bool


def build_candidate_tombstone(
    close_bundle: RollbackCloseBundle,
    evidence: FailureEvidence,
) -> CandidateTombstone | None:
    if close_bundle.close_decision not in ("close_verified", "close_with_residual_risk"):
        return None

    owner_approval_required = (
        evidence.failure_class in ("false_allow", "evidence_overread")
        or evidence.affected_decision_count >= 5
        or close_bundle.accepted_residual_risk in ("medium", "high")
    )

    return CandidateTombstone(
        pattern_id=close_bundle.pattern_id,
        failed_version=close_bundle.failed_version,
        source_close_bundle_id=close_bundle.close_bundle_id,
        source_rollback_receipt_id=close_bundle.rollback_receipt_id,
        failure_class=evidence.failure_class,
        forbidden_predicate_hash=evidence.predicate_hash,
        forbidden_evidence_scope=evidence.evidence_scope,
        forbidden_action_class=evidence.action_class,
        forbidden_risk_surface=evidence.risk_surface,
        owner_approval_required=owner_approval_required,
    )
~~~

这里故意要求 close_verified 或 close_with_residual_risk 后才写 tombstone：如果 incident 还在 reopen / observing，事实还没收敛，过早写入可能把错误根因固化。

---

## 4. learn-claude-code：把失败压成 Regression Seed

Regression Seed 不需要保存完整事故材料。它要保存的是“未来必须能重放这个风险”的最小证据。

~~~py
from dataclasses import dataclass
from typing import Literal


Decision = Literal["allow", "warn", "dry_run", "block"]


@dataclass(frozen=True)
class FailedDecisionTrace:
    decision_id: str
    input_fingerprint: str
    observed_bad_decision: Decision
    expected_safe_decision: Decision
    evidence_boundary: str
    outcome_ref: str
    minimal_replay_pack_ref: str


@dataclass(frozen=True)
class RegressionSeed:
    seed_id: str
    source_tombstone_id: str
    decision_id: str
    input_fingerprint: str
    expected_decision: Decision
    must_not_decision: Decision
    evidence_boundary: str
    outcome_ref: str
    minimal_replay_pack_ref: str


def build_regression_seed(
    source_tombstone_id: str,
    trace: FailedDecisionTrace,
) -> RegressionSeed:
    return RegressionSeed(
        seed_id=f"seed:{trace.decision_id}",
        source_tombstone_id=source_tombstone_id,
        decision_id=trace.decision_id,
        input_fingerprint=trace.input_fingerprint,
        expected_decision=trace.expected_safe_decision,
        must_not_decision=trace.observed_bad_decision,
        evidence_boundary=trace.evidence_boundary,
        outcome_ref=trace.outcome_ref,
        minimal_replay_pack_ref=trace.minimal_replay_pack_ref,
    )
~~~

注意字段 `must_not_decision`：它比“expected 是什么”更重要。很多护栏修复不一定只有唯一正确动作，但一定有某个坏动作不能再出现。

---

## 5. pi-mono：发布前 Tombstone Gate

生产版要把 tombstone 接到发布管道。任何 candidate 进入 shadow/canary 前，都先查它是否命中历史墓碑。

~~~ts
type ReleaseStage = "shadow" | "canary" | "active"
type GateDecision = "allow" | "block" | "require_repair_hypothesis" | "manual_review"

type CandidatePatternVersion = {
  patternId: string
  version: string
  predicateHash: string
  evidenceScopes: string[]
  actionClasses: string[]
  riskSurfaces: string[]
  repairHypothesisId?: string
  replayProofRefs: string[]
}

type CandidateTombstone = {
  tombstoneId: string
  patternId: string
  failedVersion: string
  forbiddenReuse: {
    predicateHashes: string[]
    evidenceScopes: string[]
    actionClasses: string[]
    riskSurfaces: string[]
  }
  ownerApprovalRequired: boolean
  status: "active" | "superseded" | "unsealed"
}

type TombstoneGateResult = {
  decision: GateDecision
  matchedTombstoneIds: string[]
  reasons: string[]
}

function intersects(a: string[], b: string[]) {
  return a.some((item) => b.includes(item))
}

export function evaluateTombstoneGate(
  candidate: CandidatePatternVersion,
  tombstones: CandidateTombstone[],
  targetStage: ReleaseStage,
): TombstoneGateResult {
  const activeMatches = tombstones.filter((tombstone) => {
    if (tombstone.status !== "active") return false
    if (tombstone.patternId !== candidate.patternId) return false

    const samePredicate = tombstone.forbiddenReuse.predicateHashes.includes(candidate.predicateHash)
    const sameScope = intersects(candidate.evidenceScopes, tombstone.forbiddenReuse.evidenceScopes)
    const sameAction = intersects(candidate.actionClasses, tombstone.forbiddenReuse.actionClasses)
    const sameRisk = intersects(candidate.riskSurfaces, tombstone.forbiddenReuse.riskSurfaces)

    return samePredicate || (sameScope && sameAction && sameRisk)
  })

  if (activeMatches.length === 0) {
    return { decision: "allow", matchedTombstoneIds: [], reasons: ["no_active_tombstone_match"] }
  }

  if (!candidate.repairHypothesisId) {
    return {
      decision: "require_repair_hypothesis",
      matchedTombstoneIds: activeMatches.map((t) => t.tombstoneId),
      reasons: ["candidate_matches_failed_history_without_repair_hypothesis"],
    }
  }

  const missingReplayProof = activeMatches.some((tombstone) => {
    return !candidate.replayProofRefs.includes(`tombstone:${tombstone.tombstoneId}`)
  })

  if (missingReplayProof) {
    return {
      decision: "block",
      matchedTombstoneIds: activeMatches.map((t) => t.tombstoneId),
      reasons: ["missing_tombstone_regression_replay_proof"],
    }
  }

  if (targetStage === "active" && activeMatches.some((t) => t.ownerApprovalRequired)) {
    return {
      decision: "manual_review",
      matchedTombstoneIds: activeMatches.map((t) => t.tombstoneId),
      reasons: ["owner_approval_required_before_active_promotion"],
    }
  }

  return {
    decision: "allow",
    matchedTombstoneIds: activeMatches.map((t) => t.tombstoneId),
    reasons: ["tombstone_replay_proof_satisfied"],
  }
}
~~~

这个 gate 的本质不是“禁止所有相似修改”，而是要求相似修改必须带着 repair hypothesis 和 regression replay proof 回来。

---

## 6. OpenClaw：课程 cron 本身也能套这个模式

这套机制不只适用于安全护栏，也适用于 Always-on Agent 的长期自动化。

以课程 cron 为例：

1. 如果某次课程发布到 Telegram 成功，但 Git push 失败，后续补偿对账会把 incident 关掉；
2. close_verified 后，要记录这次失败的结构化原因，比如 `git_remote_rejected_after_message_sent`；
3. 下次课程发布前，cron 先跑 tombstone gate：如果仍在同一分支、同一远端状态、同一发布路径，就必须先 `git pull --rebase --autostash` 和 `gh pr/list remote check`；
4. regression seed 可以是一个最小检查：`local HEAD 是否落后 origin/main`、`README 是否包含 lesson entry`、`TOOLS 是否包含已讲内容`、`Telegram messageId 是否已记录`；
5. 只有 seed 通过，才允许进入真实外部副作用。

也就是说，Agent 的“记住教训”不要只写在长文本记忆里，还要进入运行前 gate。

---

## 7. 实战检查清单

实现 Candidate Tombstone + Regression Seed 时，至少检查这些点：

- **写入时机**：只在 rollback incident close_verified / close_with_residual_risk 后写；
- **阻断范围**：不要只按 version 匹配，也要按 predicateHash、evidenceScope、actionClass、riskSurface 匹配；
- **解除条件**：必须有新的 repair hypothesis、回归重放证据，必要时要 owner approval；
- **最小回归样本**：seed 保存 input fingerprint、expected decision、must-not decision、evidence boundary；
- **发布集成**：shadow、canary、active promotion 前都要读取 tombstone registry；
- **审计闭环**：每次 gate 放行都要记录 matched tombstone 和 replay proof。

---

## 8. 这节课的核心

成熟 Agent 的 rollback closeout，不只是把事故关掉。

它要留下两个可执行资产：

1. **Candidate Tombstone**：失败版本不能靠改名、rebase、复制谓词悄悄复活；
2. **Regression Seed**：下一版候选必须证明不会再犯同一个错误。

一句话收尾：**事故报告让人记住，Tombstone 和 Regression Seed 让系统记住。**
