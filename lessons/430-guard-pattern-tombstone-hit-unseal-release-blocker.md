# 430. Agent 护栏墓碑命中后的解封闸门与发布阻断（Guard Pattern Tombstone Hit Unseal Gate & Release Blocker）

上一课讲了 **Guard Pattern Rollback Closeout Candidate Tombstone & Regression Seed**：rollback incident 关闭后，要给失败 candidate 写 Tombstone，把失败决策压成 Regression Seed，防止坏版本靠改名、rebase、复制 predicate 悄悄复活。

今天继续讲下一步：**发布管道真的命中 Tombstone 时，系统应该怎么处理**。

一句话：**Tombstone hit 不能只是打印 warning。它必须生成 Release Blocker，把 candidate 卡在 shadow/canary/active 之前；只有提交新的 Repair Hypothesis、通过 tombstone regression seed、证明 scope / predicate / evidence boundary 已经变化，才能写 Unseal Receipt，让候选版本继续进入发布流。**

---

## 1. 为什么 warning 不够

很多团队会把“历史坏版本”做成一个 checklist：

1. 发布前查一下有没有历史事故；
2. 如果命中，CI 打一个 warning；
3. reviewer 看到了就自己判断；
4. 赶时间时先 merge，后面再补证据。

这套对普通功能也许还能靠人兜住，对 Agent 护栏不够。因为护栏本身在控制副作用：发消息、执行工具、切模型、读证据、阻断用户操作。一个坏 predicate 重新上线，可能不是 UI 小 bug，而是系统重新放行 false allow 或重新制造 false block。

所以 Tombstone hit 要变成三个硬对象：

- **TombstoneHit**：证明 candidate 和哪条墓碑发生了什么级别的命中；
- **ReleaseBlocker**：在发布状态机里阻断 shadow / canary / active；
- **UnsealReceipt**：如果确实要继续发布，必须证明这不是旧坏版本复活。

没有 UnsealReceipt 的 hit，默认阻断。

---

## 2. 最小数据模型

~~~text
TombstoneHit:
  hitId
  tombstoneId
  candidateVersion
  hitKind:
    exact_version |
    predicate_hash |
    evidence_scope |
    action_surface |
    regression_seed_fail |
    lineage_overlap
  severity:
    block |
    require_unseal |
    manual_review
  matchedFields[]
  stage:
    shadow | canary | active

ReleaseBlocker:
  blockerId
  hitId
  patternId
  candidateVersion
  blockedStage
  reason
  requiredUnseal:
    repairHypothesisId
    regressionSeedProof
    predicateDiffProof
    evidenceBoundaryProof
    scopeDiffProof
    ownerApproval
  status:
    active | unsealed | superseded

UnsealReceipt:
  receiptId
  blockerId
  candidateVersion
  repairHypothesisId
  proofRefs[]
  remainingRisk
  decision:
    allow_shadow |
    allow_canary |
    allow_active |
    reject_candidate |
    require_manual_review
~~~

注意：**Unseal 不是删除 Tombstone**。Tombstone 仍然保留，UnsealReceipt 只允许某个 candidate 在某个 scope、某个 stage 继续走。

---

## 3. 命中类型要分级

Tombstone hit 不是只有“相等/不相等”。生产系统里至少要分五类：

1. **exact_version**：candidate version 就是失败版本，直接 block；
2. **predicate_hash**：predicate 内容一致但版本名不同，直接 block；
3. **lineage_overlap**：candidate 基于失败版本派生，要求 unseal；
4. **evidence_scope / action_surface**：证据边界或动作面仍覆盖事故范围，要求 unseal；
5. **regression_seed_fail**：历史 seed 回放失败，直接 block，且通常要 reopen repair。

最危险的是第二类：版本名变了，但 predicateHash 没变。很多“我重构了一下”其实没有改变坏决策逻辑。

---

## 4. learn-claude-code：Tombstone Hit 判定器

教学版先做一个纯函数。输入 candidate 和 tombstone，输出命中结果。

~~~py
from dataclasses import dataclass
from typing import Literal


Stage = Literal["shadow", "canary", "active"]
HitKind = Literal[
    "exact_version",
    "predicate_hash",
    "evidence_scope",
    "action_surface",
    "lineage_overlap",
    "no_hit",
]
Severity = Literal["block", "require_unseal", "manual_review", "allow"]


@dataclass(frozen=True)
class CandidateVersion:
    pattern_id: str
    version: str
    parent_versions: tuple[str, ...]
    predicate_hash: str
    evidence_scopes: tuple[str, ...]
    action_surfaces: tuple[str, ...]
    target_stage: Stage
    repair_hypothesis_id: str | None


@dataclass(frozen=True)
class CandidateTombstone:
    tombstone_id: str
    pattern_id: str
    failed_version: str
    forbidden_predicate_hashes: tuple[str, ...]
    forbidden_evidence_scopes: tuple[str, ...]
    forbidden_action_surfaces: tuple[str, ...]
    owner_approval_required: bool


@dataclass(frozen=True)
class TombstoneHit:
    tombstone_id: str
    candidate_version: str
    hit_kind: HitKind
    severity: Severity
    matched_fields: tuple[str, ...]


def intersects(left: tuple[str, ...], right: tuple[str, ...]) -> bool:
    return bool(set(left) & set(right))


def classify_tombstone_hit(
    candidate: CandidateVersion,
    tombstone: CandidateTombstone,
) -> TombstoneHit:
    if candidate.pattern_id != tombstone.pattern_id:
        return TombstoneHit(tombstone.tombstone_id, candidate.version, "no_hit", "allow", ())

    if candidate.version == tombstone.failed_version:
        return TombstoneHit(
            tombstone.tombstone_id,
            candidate.version,
            "exact_version",
            "block",
            ("version",),
        )

    if candidate.predicate_hash in tombstone.forbidden_predicate_hashes:
        return TombstoneHit(
            tombstone.tombstone_id,
            candidate.version,
            "predicate_hash",
            "block",
            ("predicate_hash",),
        )

    if tombstone.failed_version in candidate.parent_versions:
        return TombstoneHit(
            tombstone.tombstone_id,
            candidate.version,
            "lineage_overlap",
            "require_unseal",
            ("parent_versions",),
        )

    if intersects(candidate.evidence_scopes, tombstone.forbidden_evidence_scopes):
        return TombstoneHit(
            tombstone.tombstone_id,
            candidate.version,
            "evidence_scope",
            "require_unseal",
            ("evidence_scopes",),
        )

    if intersects(candidate.action_surfaces, tombstone.forbidden_action_surfaces):
        return TombstoneHit(
            tombstone.tombstone_id,
            candidate.version,
            "action_surface",
            "manual_review" if tombstone.owner_approval_required else "require_unseal",
            ("action_surfaces",),
        )

    return TombstoneHit(tombstone.tombstone_id, candidate.version, "no_hit", "allow", ())
~~~

这个函数不要直接去“聪明地放行”。它只负责分类。是否放行交给后面的 Release Blocker / Unseal Gate。

---

## 5. learn-claude-code：Release Blocker 和 Unseal Gate

命中后要创建 blocker。然后用证据判断能不能解封。

~~~py
from dataclasses import dataclass
from typing import Literal


UnsealDecision = Literal[
    "allow_shadow",
    "allow_canary",
    "allow_active",
    "reject_candidate",
    "require_manual_review",
]


@dataclass(frozen=True)
class ReleaseBlocker:
    blocker_id: str
    tombstone_id: str
    candidate_version: str
    blocked_stage: Stage
    hit_kind: HitKind
    severity: Severity


@dataclass(frozen=True)
class UnsealProof:
    repair_hypothesis_id: str | None
    regression_seed_passed: bool
    predicate_changed: bool
    evidence_boundary_respected: bool
    scope_reduced_or_justified: bool
    owner_approved: bool


def build_release_blocker(candidate: CandidateVersion, hit: TombstoneHit) -> ReleaseBlocker | None:
    if hit.severity == "allow":
        return None

    return ReleaseBlocker(
        blocker_id=f"block:{hit.tombstone_id}:{candidate.version}:{candidate.target_stage}",
        tombstone_id=hit.tombstone_id,
        candidate_version=candidate.version,
        blocked_stage=candidate.target_stage,
        hit_kind=hit.hit_kind,
        severity=hit.severity,
    )


def decide_unseal(blocker: ReleaseBlocker, proof: UnsealProof) -> UnsealDecision:
    if blocker.severity == "block" and blocker.hit_kind in ("exact_version", "predicate_hash"):
        return "reject_candidate"

    if not proof.repair_hypothesis_id:
        return "require_manual_review"

    required = [
        proof.regression_seed_passed,
        proof.predicate_changed,
        proof.evidence_boundary_respected,
        proof.scope_reduced_or_justified,
    ]
    if not all(required):
        return "reject_candidate"

    if blocker.severity == "manual_review" and not proof.owner_approved:
        return "require_manual_review"

    if blocker.blocked_stage == "shadow":
        return "allow_shadow"
    if blocker.blocked_stage == "canary":
        return "allow_canary"
    return "allow_active" if proof.owner_approved else "require_manual_review"
~~~

这里有一个保守原则：**exact_version 和 predicate_hash 命中不解封，直接要求新 candidate**。如果代码逻辑完全没变，所谓“补证据”很容易变成绕过墓碑。

---

## 6. pi-mono：发布管道里的 TombstoneReleaseBlocker

生产版建议把 Tombstone Gate 放在 release pipeline 的早期，位于 replay gate 之后、shadow/canary 之前。

~~~ts
type ReleaseStage = "shadow" | "canary" | "active"
type BlockerStatus = "active" | "unsealed" | "superseded"

type CandidatePatternVersion = {
  patternId: string
  version: string
  parentVersions: string[]
  predicateHash: string
  evidenceScopes: string[]
  actionSurfaces: string[]
  targetStage: ReleaseStage
  repairHypothesisId?: string
}

type CandidateTombstone = {
  tombstoneId: string
  patternId: string
  failedVersion: string
  forbiddenPredicateHashes: string[]
  forbiddenEvidenceScopes: string[]
  forbiddenActionSurfaces: string[]
  ownerApprovalRequired: boolean
}

type TombstoneHit = {
  tombstoneId: string
  hitKind:
    | "exact_version"
    | "predicate_hash"
    | "evidence_scope"
    | "action_surface"
    | "lineage_overlap"
  severity: "block" | "require_unseal" | "manual_review"
  matchedFields: string[]
}

type ReleaseBlocker = {
  blockerId: string
  tombstoneId: string
  patternId: string
  candidateVersion: string
  blockedStage: ReleaseStage
  status: BlockerStatus
  hit: TombstoneHit
}

class TombstoneReleaseBlocker {
  constructor(
    private readonly tombstones: TombstoneStore,
    private readonly blockers: ReleaseBlockerStore,
    private readonly outbox: EventOutbox,
  ) {}

  async preflight(candidate: CandidatePatternVersion): Promise<ReleaseBlocker[]> {
    const activeTombstones = await this.tombstones.findActiveByPattern(candidate.patternId)
    const blockers: ReleaseBlocker[] = []

    for (const tombstone of activeTombstones) {
      const hit = classifyHit(candidate, tombstone)
      if (!hit) continue

      const blocker: ReleaseBlocker = {
        blockerId: [
          "guard-tombstone",
          tombstone.tombstoneId,
          candidate.version,
          candidate.targetStage,
        ].join(":"),
        tombstoneId: tombstone.tombstoneId,
        patternId: candidate.patternId,
        candidateVersion: candidate.version,
        blockedStage: candidate.targetStage,
        status: "active",
        hit,
      }

      await this.blockers.upsertActive(blocker)
      await this.outbox.publish("guard_pattern.release_blocked", blocker)
      blockers.push(blocker)
    }

    return blockers
  }
}
~~~

重点是 upsertActive。同一个 candidate 重跑 pipeline 时不能不断创建新 blocker；它应该更新同一个阻断事实，保持幂等。

---

## 7. pi-mono：UnsealReceipt 只能缩小例外范围

解封 worker 不应该把 blocker 直接删掉，而是写收据并把 blocker 标为 unsealed。

~~~ts
type UnsealProofBundle = {
  repairHypothesisId?: string
  regressionSeedProofRef?: string
  regressionSeedPassed: boolean
  predicateDiffProofRef?: string
  predicateChanged: boolean
  evidenceBoundaryProofRef?: string
  evidenceBoundaryRespected: boolean
  scopeDiffProofRef?: string
  scopeReducedOrJustified: boolean
  ownerApprovalRef?: string
}

type UnsealReceipt = {
  receiptId: string
  blockerId: string
  candidateVersion: string
  allowedStage: ReleaseStage
  proofRefs: string[]
  decision: "allow_shadow" | "allow_canary" | "allow_active"
  expiresAt: string
}

class TombstoneUnsealWorker {
  constructor(
    private readonly blockers: ReleaseBlockerStore,
    private readonly receipts: UnsealReceiptStore,
    private readonly outbox: EventOutbox,
  ) {}

  async unseal(blockerId: string, proof: UnsealProofBundle) {
    const blocker = await this.blockers.getActive(blockerId)
    if (!blocker) throw new Error("active blocker not found")

    const decision = decideUnseal(blocker, proof)
    if (!decision.startsWith("allow_")) {
      await this.outbox.publish("guard_pattern.unseal_rejected", {
        blockerId,
        candidateVersion: blocker.candidateVersion,
        decision,
      })
      return { decision }
    }

    const receipt: UnsealReceipt = {
      receiptId: "unseal:" + blockerId,
      blockerId,
      candidateVersion: blocker.candidateVersion,
      allowedStage: blocker.blockedStage,
      proofRefs: [
        proof.regressionSeedProofRef,
        proof.predicateDiffProofRef,
        proof.evidenceBoundaryProofRef,
        proof.scopeDiffProofRef,
        proof.ownerApprovalRef,
      ].filter((ref): ref is string => Boolean(ref)),
      decision,
      expiresAt: new Date(Date.now() + 7 * 24 * 3600 * 1000).toISOString(),
    }

    await this.receipts.putIfAbsent(receipt)
    await this.blockers.markUnsealed(blockerId, receipt.receiptId)
    await this.outbox.publish("guard_pattern.tombstone_unsealed", receipt)
    return { decision, receipt }
  }
}
~~~

给 receipt 加 expiresAt 很重要。解封是针对当前证据窗口的例外，不应该无限期有效。

---

## 8. OpenClaw 实战：课程 Cron 的发布阻断

用我们的课程 cron 举例。上一课的 Tombstone 可以这样落地：

~~~text
tombstone:
  patternId: agent-course-publish-guard
  failedVersion: lesson-publish-v429-bad
  forbiddenPredicateHashes:
    - skipped-remote-main-check
  forbiddenActionSurfaces:
    - telegram_send
    - git_push
  forbiddenEvidenceScopes:
    - README.md
    - TOOLS.md
    - lessons/**
~~~

今天的 Release Blocker 则负责拦住这种情况：

1. 新课程文件已写；
2. README 已更新；
3. Telegram 已发送；
4. 但 git ls-remote 没证明远端 main 包含当前 commit；
5. candidate 命中 skipped-remote-main-check tombstone；
6. 发布状态不能 close，只能 active blocker；
7. 补跑 remote proof 后写 UnsealReceipt；
8. 最后再更新 memory / TOOLS 说本轮完成。

这不是为了形式主义，而是避免“本地看起来完成，远端其实没更新”的旧事故复活。

---

## 9. 常见坑

第一，把 Tombstone 当 lint warning。
护栏事故的历史失败不是代码风格，它应该进入发布状态机，默认阻断。

第二，Unseal 后删除 Tombstone。
这是错的。Tombstone 是长期负向事实；UnsealReceipt 只是某个 candidate 的阶段性例外。

第三，只比较 version，不比较 predicateHash。
坏逻辑换个名字仍然是坏逻辑。predicateHash / evidence boundary / action surface 都要比较。

第四，Regression Seed 没进解封条件。
如果历史失败样本都没过，说明 candidate 还没资格回到 shadow。

第五，解封没有过期时间。
证据会变旧，scope 会变，runtime 会漂移。UnsealReceipt 应该有 TTL 或 stage 绑定。

---

## 10. 落地检查清单

今天要给 Agent 护栏发布系统加 Tombstone Release Blocker，至少做 7 件事：

- [ ] candidate 进入 shadow/canary/active 前查询 active tombstones；
- [ ] hit 分类覆盖 exact_version、predicate_hash、lineage、evidence_scope、action_surface、regression_seed_fail；
- [ ] hit 生成幂等 ReleaseBlocker，而不是只打印 warning；
- [ ] exact_version / predicate_hash 命中默认 reject candidate；
- [ ] require_unseal 必须提交 Repair Hypothesis、Regression Seed proof、predicate diff、evidence boundary proof；
- [ ] UnsealReceipt 绑定 candidate、stage、scope、proofRefs 和 expiresAt；
- [ ] Tombstone 永不因一次 unseal 被删除，只能 superseded / archived。

---

## 11. 记住

上一课让发布系统“记住坏版本”。今天这课让发布系统“真的执行这段记忆”。

成熟 Agent 的事故记忆，不是复盘文档里一句“以后注意”，而是发布管道里的可执行阻断：命中 Tombstone 就停，证据足够才解封，解封也只对当前 candidate 和当前阶段有效。
