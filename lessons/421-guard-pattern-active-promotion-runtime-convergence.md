# 421. Agent 护栏模式全量激活与运行时收敛闸门（Guard Pattern Active Promotion & Runtime Convergence Gate）

上一课讲了 **Guard Pattern Canary Scope Ramp & Rollback Receipts**：candidate pattern 通过 shadow 后，要按 tenant、riskSurface、actionClass 和 stage 分段接管真实决策，并且每一步都有 blast radius budget 与 rollback receipt。

今天补最后一公里：**canary 全部通过以后，怎么把 pattern 升为 active，并证明所有 runtime 真的收敛到了同一个可信版本。**

一句话：**Guard Pattern active promotion 不是把 pointer 改成 candidate；要先冻结 promotion window，校验 canary 证据、LKG 健康、runtime group readiness、dependency fingerprint 和 rollback probe，再用 CAS 原子切 active/LKG 指针，最后通过 convergence probe 和 promotion receipt 证明所有执行面一致。**

---

## 1. 为什么 canary 通过还不是 active

很多系统会把 canary 通过理解成可以全量。对 Guard Pattern 来说，这还不够。

因为 pattern 影响的是决策路径，而不是只影响 UI 或普通业务逻辑：

- 控制面 pointer 改了，不代表所有 worker 都加载到了新版本；
- 某些 runtime 可能缓存旧 pattern；
- 子 Agent、cron、长会话可能带着旧 prompt pack 或旧 guard index；
- rollback target 可能在 canary 期间被清理或依赖漂移；
- canary 只覆盖了 stage scope，active 会覆盖完整 scope；
- active 后还要把新版本登记为 Last Known Good，否则下次回滚没有可信基线。

所以 active promotion 要回答五个问题：

1. candidate 真的通过了所有 canary stage 吗？
2. 它的依赖和 shadow/canary 时一致吗？
3. LKG 和 rollback probe 仍然健康吗？
4. 所有 runtime group 是否能加载 candidate？
5. 指针切换后，是否真的收敛到了同一个版本？

---

## 2. ActivePromotionGate 的最小输入

一个 promotion gate 至少需要这些字段：

~~~text
ActivePromotionRequest:
  patternId
  candidateVersion
  expectedCurrentLkgVersion
  canaryStageReceipts[]
  dependencyFingerprint
  runtimeGroups[]
  requiredRiskSurfaces[]
  rollbackProbeRef
  freezeWindowRef
  promotionPolicy:
    requireAllStagesPassed
    requireRuntimeConvergence
    maxDependencyDrift
    minRollbackProbeFreshnessSeconds
    convergenceTimeoutSeconds
~~~

这里的 expectedCurrentLkgVersion 很关键。Promotion 必须是 compare-and-swap：只有当前 LKG 仍然等于你验证过的版本，才能切到 candidate。否则说明有人已经动过基线，需要重新评估。

---

## 3. learn-claude-code：Promotion Gate 纯函数

教学版先把输入证据压成一个明确决策。

~~~py
from dataclasses import dataclass
from typing import Literal


PromotionDecision = Literal[
    "promote_active",
    "hold_for_more_evidence",
    "refresh_dependencies",
    "block_promotion",
    "manual_review",
]


@dataclass(frozen=True)
class CanaryReceipt:
    stage_id: str
    decision: str
    samples: int
    risk_surfaces: set[str]


@dataclass(frozen=True)
class RuntimeReadiness:
    group_id: str
    can_load_candidate: bool
    current_version: int
    dependency_fingerprint: str


@dataclass(frozen=True)
class PromotionInput:
    candidate_version: int
    expected_lkg_version: int
    actual_lkg_version: int
    canary_receipts: list[CanaryReceipt]
    required_risk_surfaces: set[str]
    runtime_readiness: list[RuntimeReadiness]
    expected_dependency_fingerprint: str
    rollback_probe_fresh_seconds: int
    freeze_window_open: bool


def decide_active_promotion(input: PromotionInput) -> tuple[PromotionDecision, list[str]]:
    reasons: list[str] = []

    if not input.freeze_window_open:
        return "manual_review", ["promotion_window_closed"]

    if input.actual_lkg_version != input.expected_lkg_version:
        return "block_promotion", ["lkg_pointer_changed"]

    failed_stages = [
        receipt.stage_id
        for receipt in input.canary_receipts
        if receipt.decision != "promote_next_stage"
    ]
    if failed_stages:
        return "hold_for_more_evidence", [f"canary_not_passed:{','.join(failed_stages)}"]

    covered_surfaces: set[str] = set()
    for receipt in input.canary_receipts:
        covered_surfaces |= receipt.risk_surfaces

    missing_surfaces = input.required_risk_surfaces - covered_surfaces
    if missing_surfaces:
        reasons.append("missing_risk_surface:" + ",".join(sorted(missing_surfaces)))

    if input.rollback_probe_fresh_seconds > 300:
        reasons.append("rollback_probe_stale")

    not_ready = [
        runtime.group_id
        for runtime in input.runtime_readiness
        if not runtime.can_load_candidate
    ]
    if not_ready:
        return "block_promotion", [f"runtime_not_ready:{','.join(not_ready)}"]

    drifted = [
        runtime.group_id
        for runtime in input.runtime_readiness
        if runtime.dependency_fingerprint != input.expected_dependency_fingerprint
    ]
    if drifted:
        return "refresh_dependencies", [f"dependency_drift:{','.join(drifted)}"]

    if reasons:
        return "hold_for_more_evidence", reasons

    return "promote_active", ["promotion_gate_satisfied"]
~~~

这个函数的态度很明确：

- LKG pointer 变了：直接 block；
- runtime 不能加载 candidate：直接 block；
- dependency drift：先刷新依赖再说；
- 覆盖面或 rollback probe 不足：hold，补证据；
- 全部证据一致：才 promote。

---

## 4. learn-claude-code：收敛探针

Promotion 之后还要做 convergence probe。控制面写成功不等于执行面一致。

~~~py
from dataclasses import dataclass
from typing import Literal


ConvergenceDecision = Literal[
    "converged",
    "wait_and_probe_again",
    "quarantine_runtime_group",
    "rollback_active",
]


@dataclass(frozen=True)
class RuntimeProbe:
    group_id: str
    observed_version: int
    expected_version: int
    decision_fingerprint: str
    expected_decision_fingerprint: str
    probe_age_seconds: int
    healthy: bool


def decide_convergence(
    probes: list[RuntimeProbe],
    max_probe_age_seconds: int,
) -> tuple[ConvergenceDecision, list[str]]:
    stale = [p.group_id for p in probes if p.probe_age_seconds > max_probe_age_seconds]
    if stale:
        return "wait_and_probe_again", [f"stale_probe:{','.join(stale)}"]

    unhealthy = [p.group_id for p in probes if not p.healthy]
    if unhealthy:
        return "quarantine_runtime_group", [f"unhealthy_runtime:{','.join(unhealthy)}"]

    wrong_version = [
        p.group_id for p in probes
        if p.observed_version != p.expected_version
    ]
    if wrong_version:
        return "rollback_active", [f"version_mismatch:{','.join(wrong_version)}"]

    wrong_decision = [
        p.group_id for p in probes
        if p.decision_fingerprint != p.expected_decision_fingerprint
    ]
    if wrong_decision:
        return "rollback_active", [f"decision_fingerprint_mismatch:{','.join(wrong_decision)}"]

    return "converged", ["all_runtime_groups_converged"]
~~~

这里不仅看 observed_version，还看 decision_fingerprint。因为有些坏情况是版本号一致，但依赖、prompt pack、tool schema 或 evidence index 不一致，最终决策仍然不同。

---

## 5. pi-mono：ActivePromotionController

生产版可以把 promotion 做成三阶段事务：

1. preflight gate；
2. CAS 切 active/LKG pointer；
3. convergence probe + receipt。

~~~ts
type ActivePromotionDecision =
  | "promote_active"
  | "hold_for_more_evidence"
  | "refresh_dependencies"
  | "block_promotion"
  | "manual_review"

type PromotionReceipt = {
  type: "guard_pattern.active_promotion_receipt"
  patternId: string
  fromLkgVersion: number
  toActiveVersion: number
  decision: ActivePromotionDecision
  reasons: string[]
  canaryReceiptRefs: string[]
  dependencyFingerprint: string
  pointerChangeRef?: string
  convergenceRef?: string
  createdAt: string
}

class GuardPatternActivePromotionController {
  constructor(
    private readonly evidence: PromotionEvidenceStore,
    private readonly pointerRepo: PatternPointerRepository,
    private readonly runtimeProbe: RuntimeConvergenceProbe,
    private readonly receipts: PromotionReceiptStore,
    private readonly eventBus: EventBus,
  ) {}

  async promote(input: {
    patternId: string
    candidateVersion: number
    expectedLkgVersion: number
  }): Promise<PromotionReceipt> {
    const evidence = await this.evidence.load(input.patternId, input.candidateVersion)
    const [decision, reasons] = decideActivePromotion(evidence)

    if (decision !== "promote_active") {
      return this.saveReceipt(input, decision, reasons, evidence)
    }

    const pointerChange = await this.pointerRepo.compareAndSwapLkg({
      patternId: input.patternId,
      expectedCurrentVersion: input.expectedLkgVersion,
      nextVersion: input.candidateVersion,
    })

    if (!pointerChange.changed) {
      return this.saveReceipt(input, "block_promotion", ["lkg_pointer_changed"], evidence)
    }

    await this.eventBus.publish({
      type: "guard_pattern.active_pointer_changed",
      patternId: input.patternId,
      fromVersion: input.expectedLkgVersion,
      toVersion: input.candidateVersion,
      pointerChangeRef: pointerChange.ref,
      changedAt: new Date().toISOString(),
    })

    const convergence = await this.runtimeProbe.waitForConvergence({
      patternId: input.patternId,
      expectedVersion: input.candidateVersion,
      timeoutSeconds: evidence.policy.convergenceTimeoutSeconds,
    })

    return this.saveReceipt(
      input,
      convergence.decision === "converged" ? "promote_active" : "block_promotion",
      convergence.reasons,
      evidence,
      pointerChange.ref,
      convergence.ref,
    )
  }

  private async saveReceipt(
    input: { patternId: string; candidateVersion: number; expectedLkgVersion: number },
    decision: ActivePromotionDecision,
    reasons: string[],
    evidence: PromotionEvidence,
    pointerChangeRef?: string,
    convergenceRef?: string,
  ): Promise<PromotionReceipt> {
    const receipt: PromotionReceipt = {
      type: "guard_pattern.active_promotion_receipt",
      patternId: input.patternId,
      fromLkgVersion: input.expectedLkgVersion,
      toActiveVersion: input.candidateVersion,
      decision,
      reasons,
      canaryReceiptRefs: evidence.canaryReceiptRefs,
      dependencyFingerprint: evidence.dependencyFingerprint,
      pointerChangeRef,
      convergenceRef,
      createdAt: new Date().toISOString(),
    }

    await this.receipts.save(receipt)
    return receipt
  }
}
~~~

关键点：CAS 成功只是中间步骤。最终 receipt 必须包含 convergenceRef，否则只能说明控制面改了，不能说明运行时已一致。

---

## 6. pi-mono：runtime convergence probe

Runtime probe 不应该只查配置中心。它要在每个 runtime group 上跑一个只读决策样本，确认同一个输入得到同一个 guard trace。

~~~ts
type RuntimePatternProbe = {
  runtimeGroupId: string
  observedPatternVersion: number
  observedDependencyFingerprint: string
  decisionFingerprint: string
  healthy: boolean
  observedAt: string
}

class RuntimeConvergenceProbe {
  constructor(
    private readonly runtimeRegistry: RuntimeRegistry,
    private readonly probeClient: GuardProbeClient,
    private readonly eventBus: EventBus,
  ) {}

  async collect(input: {
    patternId: string
    expectedVersion: number
    probeCaseId: string
  }): Promise<RuntimePatternProbe[]> {
    const groups = await this.runtimeRegistry.groupsForPattern(input.patternId)

    const probes = await Promise.all(groups.map(async group => {
      const result = await this.probeClient.evaluateReadonly(group.id, {
        patternId: input.patternId,
        probeCaseId: input.probeCaseId,
      })

      return {
        runtimeGroupId: group.id,
        observedPatternVersion: result.patternVersion,
        observedDependencyFingerprint: result.dependencyFingerprint,
        decisionFingerprint: result.decisionFingerprint,
        healthy: result.ok,
        observedAt: new Date().toISOString(),
      } satisfies RuntimePatternProbe
    }))

    await this.eventBus.publish({
      type: "guard_pattern.runtime_convergence_probe",
      patternId: input.patternId,
      expectedVersion: input.expectedVersion,
      probes,
    })

    return probes
  }
}
~~~

这个 probe 的价值是把所有 worker 都加载了 v43 变成事实证据，而不是控制面的假设。

---

## 7. OpenClaw 实战：课程 Cron 的 active promotion

拿课程 cron 举例：假设我们做了一个新的已讲内容语义去重 pattern。

它已经经历：

~~~text
v1 active: 只看精确标题和 README slug
v2 candidate: 标题 + 语义相似 + 系列上下文
shadow: 对真实课程选题只读比较
canary stage 1: 只 warn
canary stage 2: 对 Guard Pattern 系列 require_review
canary stage 3: 全部课程可 block 明显重复
~~~

进入 active 前，不能只说 stage 3 没出事。要生成 Promotion Receipt：

~~~text
patternId: lesson-topic-dedup
fromLkgVersion: 1
toActiveVersion: 2
canaryReceiptRefs:
  - stage-1-warn-receipt
  - stage-2-review-receipt
  - stage-3-block-receipt
runtimeGroups:
  - cron-agent-course
  - main-session-topic-picker
dependencyFingerprint:
  README.md hash
  TOOLS.md 已讲内容 hash
  lessons/ directory index hash
rollbackProbe:
  v1 can still reject exact duplicate topic
convergenceProbe:
  cron-agent-course -> v2, decisionFingerprint ok
  main-session-topic-picker -> v2, decisionFingerprint ok
~~~

如果 main-session-topic-picker 还在用旧 README hash，就不能 active。正确动作是 refresh dependency，重新 probe，再切 pointer。

---

## 8. 常见坑

1. **把 canary 通过当 active**
   Canary 只能证明 stage scope 没坏，不能证明完整 active scope 和所有 runtime 都一致。

2. **只切 active pointer，不切 LKG**
   下次 rollback 不知道哪个版本是可信基线。Active promotion 要同步更新 LKG 或明确记录 LKG 策略。

3. **没有 CAS**
   两个 controller 同时 promote，会让 receipt 和真实 pointer 对不上。

4. **probe 只看版本号**
   版本号一致但依赖 hash 不一致，决策仍然可能不同。要看 decision fingerprint。

5. **没有 freeze window**
   依赖还在变化时 promote，证据很快过期。Promotion window 要短、可冻结、可审计。

6. **promotion receipt 不含 rollback proof**
   不能退的 active promotion，是把系统推进了更危险的状态。

---

## 9. Checklist

做 Guard Pattern active promotion 时，至少检查：

- [ ] 所有 canary stage 都有 pass receipt；
- [ ] required risk surfaces 已覆盖或有明确豁免；
- [ ] 当前 LKG pointer 等于 expectedCurrentLkgVersion；
- [ ] rollback target 健康，rollback probe 新鲜；
- [ ] dependency fingerprint 和 shadow/canary 证据一致；
- [ ] 所有 runtime group 都能加载 candidate；
- [ ] pointer change 使用 compare-and-swap；
- [ ] 切换后运行 convergence probe；
- [ ] probe 校验 version、dependency fingerprint 和 decision fingerprint；
- [ ] promotion receipt 记录 pointerChangeRef、convergenceRef、canaryReceiptRefs；
- [ ] 如果收敛失败，能 quarantine runtime group 或 rollback active。

---

## 10. 核心记忆

Guard Pattern active promotion 是安全发布链路里的基线确认。

成熟 Agent 的路径应该是：

~~~text
shadow comparison
  -> canary stage receipts
  -> active promotion preflight
  -> CAS pointer/LKG change
  -> runtime convergence probe
  -> promotion receipt
  -> active drift monitor
~~~

Canary 证明 candidate 在小范围内值得信任；active promotion 证明整个运行时已经一致地使用这份可信规则。没有 convergence receipt 的 active，只是控制面的一次写入，不是生产安全基线。
