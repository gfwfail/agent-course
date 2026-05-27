# 419. Agent 护栏模式多版本影子评估与晋级预算（Guard Pattern Multi-Version Shadow Evaluation & Promotion Budget）

上一课讲了 **Guard Pattern Version Lineage & Dependency Lock**：Guard Pattern 不能原地覆盖，要拆成稳定 patternId 和不可变 PatternVersion Manifest，绑定 dependency lock、replay proof、activation scope 和 rollback target。

今天补上版本发布链路的下一步：**新版本进入 shadow 后，怎么证明它比旧版本更值得接管。**

一句话：**Guard Pattern 新版本不能只靠离线 replay 通过就 active；要让 active/candidate 多版本并跑，记录 ShadowComparison，用 false allow、false block、unknown、latency 和 scope delta 计算 Promotion Budget，再决定 promote_to_canary、extend_shadow、tighten_scope、rollback_candidate 或 manual_review。**

---

## 1. 为什么多版本 Shadow 不能只看“是否一致”

很多团队做 shadow evaluation 时只看一件事：新旧版本决策是否一样。

这不够。

Guard Pattern 的目标不是永远和旧版本一致，而是修复旧版本的盲点，同时不制造更大的误伤。比如：

- 旧版漏挡了重复外发，新版应该更严格；
- 旧版误伤了正常自动化，新版应该更精确；
- 旧版 unknown 太多，新版应该更能解释；
- 新版 latency 变高，也可能拖垮热路径；
- 新版扩大了 scope，必须证明新增范围有证据支撑。

所以 shadow comparison 不是 diff viewer，而是一个发布闸门：它要回答“新版在哪些场景比旧版好，在哪些场景风险更大，风险预算是否允许继续晋级”。

---

## 2. ShadowComparison 的最小结构

每次真实 guard evaluation 都可以同时运行 active version 和 candidate version，但只有 active version 真正影响决策。candidate 只产出 shadow result。

~~~text
ShadowComparison:
  patternId
  activeVersion
  candidateVersion
  decisionId
  riskSurface
  actionClass
  activeDecision:
    action: allow | warn | require_approval | block | unknown
    reasonHash
  candidateDecision:
    action: allow | warn | require_approval | block | unknown
    reasonHash
  outcome:
    laterLabel: true_positive | false_positive | true_negative | false_negative | unknown
  diffClass:
    same
    candidate_tighter
    candidate_looser
    candidate_more_precise
    candidate_more_unknown
    candidate_error
  latencyMs:
    active
    candidate
  evidenceRefs
~~~

关键点：candidate 不要执行副作用，不要改队列，不要改变用户可见行为。它只读输入、证据和 runtime deps，写审计事件。

---

## 3. learn-claude-code：影子差异分类器

教学版先用纯函数做 diff classification。它不需要 LLM，只需要把两个版本的决策按严格程度比较。

~~~py
from dataclasses import dataclass
from typing import Literal

Action = Literal["allow", "warn", "require_approval", "block", "unknown"]
DiffClass = Literal[
    "same",
    "candidate_tighter",
    "candidate_looser",
    "candidate_more_precise",
    "candidate_more_unknown",
    "candidate_error",
]

STRICTNESS: dict[Action, int] = {
    "allow": 0,
    "warn": 1,
    "require_approval": 2,
    "block": 3,
    "unknown": 2,
}


@dataclass(frozen=True)
class PatternDecision:
    action: Action
    reason_hash: str
    error: str | None = None


def classify_shadow_diff(
    active: PatternDecision,
    candidate: PatternDecision,
) -> DiffClass:
    if candidate.error:
        return "candidate_error"
    if active.action == candidate.action and active.reason_hash == candidate.reason_hash:
        return "same"
    if active.action == "unknown" and candidate.action != "unknown":
        return "candidate_more_precise"
    if active.action != "unknown" and candidate.action == "unknown":
        return "candidate_more_unknown"

    active_level = STRICTNESS[active.action]
    candidate_level = STRICTNESS[candidate.action]

    if candidate_level > active_level:
        return "candidate_tighter"
    if candidate_level < active_level:
        return "candidate_looser"
    return "candidate_more_precise"
~~~

注意这里把 unknown 当成中间严格度，但单独分类。因为 unknown 不是安全策略，它是证据不足；unknown 变少通常是好事，但 unknown 变成 allow/block 也必须等 outcome 验证。

---

## 4. learn-claude-code：晋级预算判定

Shadow 跑一段窗口后，把 comparisons 聚合成 Promotion Budget。

~~~py
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
class ShadowStats:
    samples: int
    candidate_errors: int
    candidate_looser_high_risk: int
    candidate_false_allows: int
    candidate_false_blocks: int
    candidate_more_unknown: int
    candidate_fixed_active_failures: int
    p95_latency_ms: int
    scope_delta_count: int


@dataclass(frozen=True)
class PromotionBudget:
    min_samples: int
    max_candidate_errors: int
    max_looser_high_risk: int
    max_false_allows: int
    max_false_blocks: int
    max_more_unknown_rate: float
    max_p95_latency_ms: int
    max_scope_delta_without_review: int
    min_fixed_active_failures: int


def decide_pattern_promotion(
    stats: ShadowStats,
    budget: PromotionBudget,
) -> tuple[PromotionDecision, list[str]]:
    reasons: list[str] = []

    if stats.samples < budget.min_samples:
        reasons.append("not_enough_shadow_samples")
    if stats.candidate_errors > budget.max_candidate_errors:
        reasons.append("candidate_runtime_errors")
    if stats.candidate_looser_high_risk > budget.max_looser_high_risk:
        reasons.append("candidate_looser_on_high_risk")
    if stats.candidate_false_allows > budget.max_false_allows:
        reasons.append("candidate_false_allows")
    if stats.candidate_false_blocks > budget.max_false_blocks:
        reasons.append("candidate_false_blocks")
    if stats.p95_latency_ms > budget.max_p95_latency_ms:
        reasons.append("candidate_latency_too_high")

    unknown_rate = stats.candidate_more_unknown / max(stats.samples, 1)
    if unknown_rate > budget.max_more_unknown_rate:
        reasons.append("candidate_more_unknown_rate_too_high")

    if stats.scope_delta_count > budget.max_scope_delta_without_review:
        reasons.append("scope_delta_requires_review")

    if stats.candidate_fixed_active_failures < budget.min_fixed_active_failures:
        reasons.append("candidate_has_not_proven_value")

    if "candidate_false_allows" in reasons or "candidate_looser_on_high_risk" in reasons:
        return "rollback_candidate", reasons
    if "scope_delta_requires_review" in reasons:
        return "manual_review", reasons
    if reasons:
        if "candidate_false_blocks" in reasons or "candidate_more_unknown_rate_too_high" in reasons:
            return "tighten_scope", reasons
        return "extend_shadow", reasons

    return "promote_to_canary", ["promotion_budget_satisfied"]
~~~

这段逻辑故意保守：false allow 和高风险放宽是硬伤；false block 可以先收窄 scope；样本不足则继续 shadow。

---

## 5. pi-mono：多版本 Pattern Evaluator

生产版可以把 active 和 candidate evaluation 放在同一个事件边界里，保证输入一致、证据一致、依赖一致。

~~~ts
type PatternAction = "allow" | "warn" | "require_approval" | "block" | "unknown"

type PatternEvaluation = {
  patternId: string
  version: number
  action: PatternAction
  reasonHash: string
  latencyMs: number
  error?: string
}

type ShadowComparison = {
  type: "guard_pattern.shadow_comparison"
  patternId: string
  activeVersion: number
  candidateVersion: number
  decisionId: string
  riskSurface: string
  actionClass: string
  active: PatternEvaluation
  candidate: PatternEvaluation
  diffClass:
    | "same"
    | "candidate_tighter"
    | "candidate_looser"
    | "candidate_more_precise"
    | "candidate_more_unknown"
    | "candidate_error"
  evidenceRefs: string[]
  createdAt: string
}

class MultiVersionPatternEvaluator {
  constructor(
    private readonly patternStore: PatternStore,
    private readonly eventBus: EventBus,
  ) {}

  async evaluate(input: GuardInput): Promise<PatternEvaluation[]> {
    const active = await this.patternStore.activeVersion(input.patternId)
    const candidates = await this.patternStore.shadowCandidates(input.patternId)

    const activeResult = await this.runPattern(active, input)
    const candidateResults = await Promise.all(
      candidates.map((version) => this.runPattern(version, input)),
    )

    for (const candidateResult of candidateResults) {
      await this.eventBus.publish({
        type: "guard_pattern.shadow_comparison",
        patternId: input.patternId,
        activeVersion: active.version,
        candidateVersion: candidateResult.version,
        decisionId: input.decisionId,
        riskSurface: input.riskSurface,
        actionClass: input.actionClass,
        active: activeResult,
        candidate: candidateResult,
        diffClass: classifyShadowDiff(activeResult, candidateResult),
        evidenceRefs: input.evidenceRefs,
        createdAt: new Date().toISOString(),
      } satisfies ShadowComparison)
    }

    return [activeResult]
  }

  private async runPattern(
    version: PatternVersion,
    input: GuardInput,
  ): Promise<PatternEvaluation> {
    const started = Date.now()
    try {
      const result = await version.evaluateReadonly(input)
      return {
        patternId: version.patternId,
        version: version.version,
        action: result.action,
        reasonHash: result.reasonHash,
        latencyMs: Date.now() - started,
      }
    } catch (error) {
      return {
        patternId: version.patternId,
        version: version.version,
        action: "unknown",
        reasonHash: "candidate-error",
        latencyMs: Date.now() - started,
        error: String(error),
      }
    }
  }
}
~~~

这里最重要的是 evaluateReadonly。candidate version 只能读，不能触发 side effect，也不能写 active decision。

---

## 6. pi-mono：PromotionBudgetWorker

Shadow comparison 事件进入聚合 worker，按版本生成晋级决定。

~~~ts
type PatternPromotionDecision =
  | "promote_to_canary"
  | "extend_shadow"
  | "tighten_scope"
  | "rollback_candidate"
  | "manual_review"

type PatternPromotionRecord = {
  type: "guard_pattern.promotion_decision"
  patternId: string
  candidateVersion: number
  decision: PatternPromotionDecision
  reasons: string[]
  stats: {
    samples: number
    candidateErrors: number
    candidateLooserHighRisk: number
    candidateFalseAllows: number
    candidateFalseBlocks: number
    candidateMoreUnknown: number
    candidateFixedActiveFailures: number
    p95LatencyMs: number
    scopeDeltaCount: number
  }
  budgetId: string
  decidedAt: string
}

class PatternPromotionBudgetWorker {
  constructor(
    private readonly comparisonStore: ShadowComparisonStore,
    private readonly budgetStore: PromotionBudgetStore,
    private readonly eventBus: EventBus,
  ) {}

  async run(patternId: string, candidateVersion: number): Promise<void> {
    const comparisons = await this.comparisonStore.window({
      patternId,
      candidateVersion,
      sinceHours: 24,
    })
    const stats = aggregateShadowStats(comparisons)
    const budget = await this.budgetStore.forPattern(patternId)

    const { decision, reasons } = decidePatternPromotion(stats, budget)

    await this.eventBus.publish({
      type: "guard_pattern.promotion_decision",
      patternId,
      candidateVersion,
      decision,
      reasons,
      stats,
      budgetId: budget.id,
      decidedAt: new Date().toISOString(),
    } satisfies PatternPromotionRecord)
  }
}
~~~

PromotionDecision 也要不可变记录。后续 canary 放量、rollback、manual review 都引用这个 decisionId，而不是重新解释一遍旧数据。

---

## 7. OpenClaw 实战：课程 Cron 的多版本影子评估

拿这个课程 cron 类比，active 规则是“发课前检查 TOOLS.md 已讲内容、README 目录、lessons 文件，避免重复”。如果我想升级选题规则，比如让它能自动检测“语义重复”而不只是字符串重复，就不能直接替换 active 流程。

安全做法：

1. active 规则照常决定本次课程能不能发布；
2. candidate 规则只读运行，判断它会不会选同一个主题、会不会误判为重复；
3. 记录 shadow comparison：active allow，candidate block，原因是“语义相似”；
4. 课程发出后，用结果回填 outcome：是否真的重复、是否被群里反馈、README/TOOLS 是否一致；
5. 24 小时或 N 次样本后再算 promotion budget。

如果 candidate 经常把新主题误判为重复，它就不能晋级；如果它抓住了 active 没抓住的重复风险，并且 false block 很低，才进入 canary：比如先只对“Guard Pattern”系列启用，再逐步扩大到全部选题。

这就是多版本 shadow 的价值：**让新规则在真实流量里证明自己，但先不让它控制现实世界。**

---

## 8. 常见坑

1. **candidate 偷偷写状态**
   Shadow 版本只要写了队列、记忆、缓存或外部消息，就不再是 shadow，而是隐形 canary。

2. **只看 diff 数量，不看 diff 方向**
   candidate_tighter 和 candidate_looser 风险完全不同。高风险场景放宽一次，可能比收紧一百次更危险。

3. **没有 outcome 回填**
   没有 laterLabel，就只能知道“新旧不同”，不能知道谁对谁错。

4. **scope delta 没单独预算**
   新版扩大租户、风险面或 action class，必须额外 review。样本再漂亮，也不能自动扩大到没见过的范围。

5. **latency 不进预算**
   Guard Pattern 常在热路径里运行。一个更聪明但慢十倍的 pattern，可能让整个 Agent loop 卡住。

---

## 9. Checklist

做 Guard Pattern 多版本影子评估时，至少检查：

- [ ] active 和 candidate 使用同一份输入、证据和 dependency lock snapshot；
- [ ] candidate 只能 readonly evaluate，不能产生副作用；
- [ ] ShadowComparison 记录 activeVersion/candidateVersion/decisionId；
- [ ] diffClass 区分 tighter、looser、unknown、error；
- [ ] outcome laterLabel 能回填；
- [ ] Promotion Budget 明确 false allow、false block、unknown、latency、scope delta 上限；
- [ ] promote/extend/tighten/rollback/manual_review 都写不可变 decision record；
- [ ] canary 放量引用 promotion decision，而不是重新凭感觉判断。

---

## 10. 核心记忆

Guard Pattern 新版本进入 shadow，不是为了假装已经上线，而是为了在真实输入上积累可归因证据。

成熟 Agent 的 pattern 发布链路应该是：

~~~text
candidate version
  -> readonly shadow comparison
  -> outcome backfill
  -> promotion budget
  -> canary scope
  -> active pointer
  -> runtime drift monitor
~~~

没有预算的 shadow 只是日志；有版本、有 outcome、有预算、有收据的 shadow，才是生产级安全发布。
