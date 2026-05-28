# 425. Agent 护栏修复 Shadow 结果回填与晋级证据闸门（Guard Pattern Repair Shadow Outcome Backfill & Promotion Evidence Gate）

上一课讲了 **Guard Pattern Backstop Repair Hypothesis & Counterfactual Replay Gate**：Backstop 仲裁决定 open_repair 后，先写 Repair Hypothesis，再用 active/candidate 反事实回放证明目标失败被修复、known good 没回归、证据读取没越界。

今天继续下一步：candidate 通过 replay 后进入 **shadow**。

一句话：**Shadow 不是“看起来没事就上 canary”。它要把 candidate 在真实流量中的只读决策、active 真实决策、最终 outcome、证据新鲜度和样本覆盖率回填成 Promotion Evidence Bundle，再由晋级闸门决定 promote_to_canary、extend_shadow、tighten_scope、rollback_candidate 或 manual_review。**

---

## 1. 为什么 replay 通过还要 shadow outcome backfill

离线 replay 解决的是“历史样本里能不能修”。Shadow 解决的是“真实流量里会不会产生新问题”。

两者差别很大：

1. replay case 是挑过的，真实流量会出现没覆盖的新 actionClass；
2. replay 的 evidence snapshot 是固定的，生产 evidence 会过期、缺失、延迟；
3. replay 只比较 decision，真实世界还要看 outcome；
4. candidate 在 shadow 里不能影响用户，但可以记录“如果它掌权会怎样”。

所以 shadow 期最重要的不是统计 candidate 和 active 有多少差异，而是把差异和结果对上账：candidate 多挡的请求，后续是否真的有风险？candidate 放过的请求，active 挡住后是否证明 active 是对的？candidate 读的证据，是否稳定、新鲜、在预算内？

---

## 2. Shadow Outcome 的最小数据模型

可以把每个真实决策记录成一条 paired observation。

~~~text
GuardRepairShadowObservation:
  observationId
  repairId
  patternId
  activeVersion
  candidateVersion
  runtimeGroup
  tenant
  riskSurface
  actionClass
  active:
    decision
    reasonCode
    evidenceHash
    evidenceFreshnessSec
  candidate:
    decision
    reasonCode
    evidenceHash
    evidenceFreshnessSec
    extraEvidenceReads[]
  outcome:
    status:
      confirmed_safe |
      confirmed_risky |
      reversed_by_human |
      tool_failed |
      pending |
      unavailable
    observedAt
  shadowClass:
    same_decision |
    candidate_fixed_false_allow |
    candidate_fixed_false_block |
    candidate_too_strict |
    candidate_too_loose |
    evidence_drift |
    unknown
~~~

注意两点：

- active 是生产真实执行结果；
- candidate 只是只读旁路结果，不能触发外部副作用。

这样做的好处是，晋级时不靠“worker 说应该可以了”，而是靠一组可审计的 observation。

---

## 3. learn-claude-code：Shadow 差异分类器

教学版先把 paired observation 分类写成纯函数。

~~~py
from dataclasses import dataclass
from typing import Literal


Decision = Literal["allow", "warn", "dry_run", "block"]
OutcomeStatus = Literal[
    "confirmed_safe",
    "confirmed_risky",
    "reversed_by_human",
    "tool_failed",
    "pending",
    "unavailable",
]
ShadowClass = Literal[
    "same_decision",
    "candidate_fixed_false_allow",
    "candidate_fixed_false_block",
    "candidate_too_strict",
    "candidate_too_loose",
    "evidence_drift",
    "unknown",
]


@dataclass(frozen=True)
class DecisionView:
    decision: Decision
    reason_code: str
    evidence_hash: str
    evidence_freshness_sec: int
    extra_evidence_reads: tuple[str, ...] = ()


@dataclass(frozen=True)
class ShadowObservation:
    active: DecisionView
    candidate: DecisionView
    outcome_status: OutcomeStatus
    max_evidence_freshness_sec: int


def classify_shadow_observation(obs: ShadowObservation) -> ShadowClass:
    if obs.candidate.evidence_freshness_sec > obs.max_evidence_freshness_sec:
        return "evidence_drift"

    if obs.outcome_status in ("pending", "unavailable", "tool_failed"):
        return "unknown"

    if obs.active.decision == obs.candidate.decision:
        return "same_decision"

    active_blocks = obs.active.decision in ("dry_run", "block")
    candidate_blocks = obs.candidate.decision in ("dry_run", "block")

    if candidate_blocks and not active_blocks:
        if obs.outcome_status in ("confirmed_risky", "reversed_by_human"):
            return "candidate_fixed_false_allow"
        return "candidate_too_strict"

    if active_blocks and not candidate_blocks:
        if obs.outcome_status == "confirmed_safe":
            return "candidate_fixed_false_block"
        return "candidate_too_loose"

    return "unknown"
~~~

这个分类器故意把 evidence freshness 放在最前面。candidate 如果依赖已经过期的证据，即使决策方向看起来对，也不能算晋级证据。

---

## 4. learn-claude-code：Promotion Evidence Gate

Shadow 观察要汇总成晋级证据包。这里不要只看“差异率”，而要同时看覆盖率、unknown rate、false allow 修复、false block 修复和 evidence drift。

~~~py
from collections import Counter
from dataclasses import dataclass
from typing import Literal


PromotionDecision = Literal[
    "promote_to_canary",
    "extend_shadow",
    "tighten_scope",
    "rollback_candidate",
    "manual_review",
]


@dataclass(frozen=True)
class PromotionEvidence:
    total_observations: int
    covered_tenants: int
    covered_action_classes: int
    required_tenants: int
    required_action_classes: int
    classes: list[str]


def decide_shadow_promotion(e: PromotionEvidence) -> tuple[PromotionDecision, list[str]]:
    counts = Counter(e.classes)

    if e.total_observations < 100:
        return "extend_shadow", ["insufficient_shadow_sample"]

    if e.covered_tenants < e.required_tenants or e.covered_action_classes < e.required_action_classes:
        return "extend_shadow", ["insufficient_scope_coverage"]

    if counts["evidence_drift"] > 0:
        return "tighten_scope", ["candidate_evidence_drift"]

    if counts["candidate_too_loose"] > 0:
        return "rollback_candidate", ["candidate_would_false_allow"]

    if counts["candidate_too_strict"] > 2:
        return "tighten_scope", ["candidate_would_false_block_too_often"]

    unknown_rate = counts["unknown"] / e.total_observations
    if unknown_rate > 0.05:
        return "manual_review", ["shadow_unknown_rate_too_high"]

    fixed = counts["candidate_fixed_false_allow"] + counts["candidate_fixed_false_block"]
    if fixed == 0:
        return "extend_shadow", ["no_positive_shadow_evidence"]

    return "promote_to_canary", ["positive_shadow_evidence_with_bounded_risk"]
~~~

这里有一个实用规则：如果 candidate 没有任何正向证据，只是“没出事”，不要晋级。Shadow 的目标不是证明安静，而是证明它在真实流量里确实修了某类错误。

---

## 5. pi-mono：ShadowOutcomeBackfillWorker

生产版里，Shadow worker 可以订阅 guard decision 事件，旁路跑 candidate，然后等 outcome 回填后生成 evidence bundle。

~~~ts
type GuardDecision = "allow" | "warn" | "dry_run" | "block"

type GuardDecisionEvent = {
  type: "guard.decision"
  decisionId: string
  tenant: string
  riskSurface: string
  actionClass: string
  runtimeGroup: string
  activeVersion: string
  activeDecision: GuardDecision
  activeReasonCode: string
  activeEvidenceHash: string
  activeEvidenceFreshnessSec: number
}

type CandidateDecision = {
  candidateVersion: string
  decision: GuardDecision
  reasonCode: string
  evidenceHash: string
  evidenceFreshnessSec: number
  extraEvidenceReads: string[]
}

type OutcomeEvent = {
  type: "guard.outcome"
  decisionId: string
  status:
    | "confirmed_safe"
    | "confirmed_risky"
    | "reversed_by_human"
    | "tool_failed"
    | "pending"
    | "unavailable"
  observedAt: string
}

type ShadowObservation = {
  type: "guard_pattern.repair_shadow_observation"
  observationId: string
  repairId: string
  patternId: string
  activeVersion: string
  candidateVersion: string
  tenant: string
  riskSurface: string
  actionClass: string
  runtimeGroup: string
  activeDecision: GuardDecision
  candidateDecision: GuardDecision
  outcomeStatus: OutcomeEvent["status"]
  shadowClass: string
  createdAt: string
}

class ShadowOutcomeBackfillWorker {
  constructor(
    private readonly candidateRunner: {
      evaluate(event: GuardDecisionEvent): Promise<CandidateDecision>
    },
    private readonly outcomes: {
      waitFor(decisionId: string): Promise<OutcomeEvent>
    },
    private readonly eventBus: {
      publish(event: ShadowObservation): Promise<void>
    },
  ) {}

  async handleDecision(event: GuardDecisionEvent, repair: {
    repairId: string
    patternId: string
    candidateVersion: string
    maxEvidenceFreshnessSec: number
  }) {
    const candidate = await this.candidateRunner.evaluate(event)
    const outcome = await this.outcomes.waitFor(event.decisionId)

    const shadowClass = this.classify(event, candidate, outcome, repair.maxEvidenceFreshnessSec)

    await this.eventBus.publish({
      type: "guard_pattern.repair_shadow_observation",
      observationId: repair.repairId + ":" + event.decisionId,
      repairId: repair.repairId,
      patternId: repair.patternId,
      activeVersion: event.activeVersion,
      candidateVersion: candidate.candidateVersion,
      tenant: event.tenant,
      riskSurface: event.riskSurface,
      actionClass: event.actionClass,
      runtimeGroup: event.runtimeGroup,
      activeDecision: event.activeDecision,
      candidateDecision: candidate.decision,
      outcomeStatus: outcome.status,
      shadowClass,
      createdAt: new Date().toISOString(),
    })
  }

  private classify(
    active: GuardDecisionEvent,
    candidate: CandidateDecision,
    outcome: OutcomeEvent,
    maxFreshnessSec: number,
  ) {
    if (candidate.evidenceFreshnessSec > maxFreshnessSec) return "evidence_drift"
    if (["pending", "unavailable", "tool_failed"].includes(outcome.status)) return "unknown"
    if (active.activeDecision === candidate.decision) return "same_decision"

    const activeBlocks = ["dry_run", "block"].includes(active.activeDecision)
    const candidateBlocks = ["dry_run", "block"].includes(candidate.decision)

    if (candidateBlocks && !activeBlocks) {
      return ["confirmed_risky", "reversed_by_human"].includes(outcome.status)
        ? "candidate_fixed_false_allow"
        : "candidate_too_strict"
    }

    if (activeBlocks && !candidateBlocks) {
      return outcome.status === "confirmed_safe"
        ? "candidate_fixed_false_block"
        : "candidate_too_loose"
    }

    return "unknown"
  }
}
~~~

这段代码的关键是：candidate 只读运行，outcome 延迟回填，最后产出 observation，而不是直接改变 active 决策。

---

## 6. pi-mono：PromotionEvidenceBundleWorker

当 shadow observation 足够多时，再聚合成晋级证据包。

~~~ts
type PromotionEvidenceBundle = {
  type: "guard_pattern.repair_promotion_evidence"
  repairId: string
  patternId: string
  candidateVersion: string
  totalObservations: number
  coveredTenants: number
  coveredActionClasses: number
  counts: Record<string, number>
  decision:
    | "promote_to_canary"
    | "extend_shadow"
    | "tighten_scope"
    | "rollback_candidate"
    | "manual_review"
  reasons: string[]
  evidenceRefs: string[]
  createdAt: string
}

class PromotionEvidenceBundleWorker {
  constructor(
    private readonly store: {
      loadObservations(repairId: string): Promise<ShadowObservation[]>
    },
    private readonly eventBus: {
      publish(event: PromotionEvidenceBundle): Promise<void>
    },
  ) {}

  async build(repair: {
    repairId: string
    patternId: string
    candidateVersion: string
    requiredTenants: number
    requiredActionClasses: number
  }) {
    const observations = await this.store.loadObservations(repair.repairId)
    const counts = observations.reduce<Record<string, number>>((acc, item) => {
      acc[item.shadowClass] = (acc[item.shadowClass] ?? 0) + 1
      return acc
    }, {})

    const coveredTenants = new Set(observations.map((item) => item.tenant)).size
    const coveredActionClasses = new Set(observations.map((item) => item.actionClass)).size
    const decision = this.decide({
      total: observations.length,
      counts,
      coveredTenants,
      coveredActionClasses,
      requiredTenants: repair.requiredTenants,
      requiredActionClasses: repair.requiredActionClasses,
    })

    await this.eventBus.publish({
      type: "guard_pattern.repair_promotion_evidence",
      repairId: repair.repairId,
      patternId: repair.patternId,
      candidateVersion: repair.candidateVersion,
      totalObservations: observations.length,
      coveredTenants,
      coveredActionClasses,
      counts,
      decision: decision.decision,
      reasons: decision.reasons,
      evidenceRefs: observations.slice(0, 20).map((item) => "shadow:" + item.observationId),
      createdAt: new Date().toISOString(),
    })
  }

  private decide(input: {
    total: number
    counts: Record<string, number>
    coveredTenants: number
    coveredActionClasses: number
    requiredTenants: number
    requiredActionClasses: number
  }) {
    if (input.total < 100) {
      return { decision: "extend_shadow" as const, reasons: ["insufficient_shadow_sample"] }
    }
    if (
      input.coveredTenants < input.requiredTenants ||
      input.coveredActionClasses < input.requiredActionClasses
    ) {
      return { decision: "extend_shadow" as const, reasons: ["insufficient_scope_coverage"] }
    }
    if (input.counts.evidence_drift) {
      return { decision: "tighten_scope" as const, reasons: ["candidate_evidence_drift"] }
    }
    if (input.counts.candidate_too_loose) {
      return { decision: "rollback_candidate" as const, reasons: ["candidate_would_false_allow"] }
    }
    if ((input.counts.candidate_too_strict ?? 0) > 2) {
      return { decision: "tighten_scope" as const, reasons: ["candidate_would_false_block_too_often"] }
    }

    const unknownRate = (input.counts.unknown ?? 0) / input.total
    if (unknownRate > 0.05) {
      return { decision: "manual_review" as const, reasons: ["shadow_unknown_rate_too_high"] }
    }

    const fixed =
      (input.counts.candidate_fixed_false_allow ?? 0) +
      (input.counts.candidate_fixed_false_block ?? 0)
    if (fixed === 0) {
      return { decision: "extend_shadow" as const, reasons: ["no_positive_shadow_evidence"] }
    }

    return {
      decision: "promote_to_canary" as const,
      reasons: ["positive_shadow_evidence_with_bounded_risk"],
    }
  }
}
~~~

生产上这个 bundle 就是 canary 的准入材料：没有 bundle，不允许改 traffic share；bundle 决策不是 promote，就不能绕过。

---

## 7. OpenClaw：课程 Cron 的同构实践

这套课程 cron 也有类似 shadow outcome backfill。

每次选新主题时，我会先做只读检查：

1. 读取 TOOLS.md 已讲内容；
2. 读取 agent-course/README.md 最近目录；
3. 用 rg 搜索相近标题和关键词；
4. 生成 candidate 课程；
5. 写入 lesson/README/TOOLS；
6. 发送 Telegram；
7. commit/push；
8. 用 messageId、commit sha 和远端 main 指针回填 memory。

这里的 shadow observation 可以类比成：

~~~text
activeDecision: "上一课已经发布 424"
candidateDecision: "425 主题与 424 相邻但不重复"
outcomeStatus: "Telegram message delivered + git push verified"
shadowClass: "same_scope_new_angle"
evidenceRefs:
  - "README entry"
  - "TOOLS 已讲内容"
  - "Telegram messageId"
  - "git commit sha"
~~~

如果 rg 发现主题已经讲过，就应该 reject candidate；如果 Telegram 发出但 git push 没成功，就不是完整 outcome；如果 TOOLS 没更新，下次 cron 就可能重复选题。

---

## 8. 实战落地清单

要把这个模式落地，先做五件事：

1. **Shadow 必须只读**：candidate 只能记录 would_decide，不能触发工具、副作用或用户可见变化。
2. **Outcome 延迟回填**：不要在 decision 当场判断修复成功，要等真实 outcome、人工复核或后续 probe。
3. **统计 coverage，不只统计 count**：样本数够但只覆盖一个 tenant 或 actionClass，不能晋级。
4. **positive evidence 必须存在**：candidate 至少要在真实流量里修复过 false_allow 或 false_block，不能只靠“没出事”晋级。
5. **晋级决策写成 bundle**：canary controller 只接受 Promotion Evidence Bundle，不接受口头判断。

---

## 9. 常见坑

- **把 shadow 当灰度**：shadow 不该影响真实决策；一旦影响用户，就已经是 canary。
- **只看 diff rate**：candidate 和 active 差异少，不代表 candidate 正确；关键是差异与 outcome 是否对齐。
- **pending outcome 太多还晋级**：unknown rate 高说明证据不足，不是系统稳定。
- **coverage 假充分**：1000 条样本都来自同一个低风险 actionClass，对高风险面没有证明力。
- **没有 evidence freshness**：candidate 依赖过期证据时，shadow 结果可能只是偶然正确。

---

## 10. 今日作业

给你自己的 Agent guard 修复设计一份 Promotion Evidence Bundle：

- 需要多少 shadow observations？
- 至少覆盖几个 tenant / actionClass / riskSurface？
- 哪些 shadowClass 直接 rollback candidate？
- unknown rate 超过多少要 manual review？
- candidate 必须证明修复了哪类 false_allow 或 false_block？

成熟 Agent 的护栏修复，不是 replay 通过后就上生产。它要在真实流量里只读证明：**新版确实修了错判，没制造新漏挡或误伤，证据仍然新鲜，覆盖面足够，然后才有资格进入 canary。**
