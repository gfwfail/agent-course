# 426. Agent 护栏修复 Canary 差异预算与自动回滚触发（Guard Pattern Repair Canary Diff Budget & Rollback Trigger）

上一课讲了 **Guard Pattern Repair Shadow Outcome Backfill & Promotion Evidence Gate**：repair candidate 先在 shadow 里只读比较 active/candidate 决策，把真实 outcome、证据新鲜度和覆盖率回填成 Promotion Evidence Bundle，再决定能不能进 canary。

今天继续下一步：candidate 进入 **canary**，开始真实影响一小部分生产决策。

一句话：**修复版护栏的 canary 不能只看错误率和流量百分比。它必须维护 Decision Diff Budget，记录 candidate 相比 active 多挡、少挡、改 reason、读新证据带来的真实影响；一旦 false_allow、false_block、unknown outcome 或证据漂移超过预算，就自动 rollback，并把已影响的决策转入回填与补偿队列。**

---

## 1. 为什么修复 canary 要单独做 diff budget

普通 feature canary 常看 latency、error rate、成功率。护栏修复 canary 还要看一个更关键的东西：**决策差异是否朝预期方向发生**。

修复候选进 canary 后，风险主要有四类：

1. 修复 false_allow 时，candidate 可能挡住更多危险动作，但也可能误伤正常动作；
2. 修复 false_block 时，candidate 可能放行更多正常动作，但也可能漏掉危险动作；
3. reasonCode 改了，用户或审计系统看到的解释可能不再符合策略；
4. candidate 读取了新证据，证据源延迟、漂移或权限边界会变成生产风险。

所以 canary 不是“给 5% 流量试试”。更靠谱的做法是给每类差异一个预算：

- candidate_more_strict：candidate 比 active 更严格；
- candidate_more_lenient：candidate 比 active 更宽松；
- reason_changed：决策没变但解释变了；
- evidence_expanded：candidate 多读了证据；
- unknown_outcome：真实结果还没回来；
- confirmed_bad_diff：已经证明 candidate 方向错了。

预算耗尽就回滚，不等全量事故。

---

## 2. Canary Diff 的最小数据模型

Canary 期需要同时保存 active baseline 和 candidate 实际执行结果。即使 active 没有真正执行，也要在旁路里重算 baseline，方便事后证明“candidate 改变了什么”。

~~~text
GuardRepairCanaryDecision:
  decisionId
  repairId
  patternId
  stageId
  tenant
  riskSurface
  actionClass
  activeBaseline:
    decision
    reasonCode
    evidenceHash
  candidateApplied:
    decision
    reasonCode
    evidenceHash
    extraEvidenceReads[]
  diffClass:
    same_decision |
    candidate_more_strict |
    candidate_more_lenient |
    reason_changed |
    evidence_expanded |
    unknown
  outcome:
    pending |
    confirmed_safe |
    confirmed_risky |
    reversed_by_human |
    side_effect_repaired |
    unavailable
~~~

这里最关键的是 **candidateApplied**。Shadow 期 candidate 不影响现实；canary 期 candidate 已经影响了一小部分真实动作，所以每条 diff 都必须能追到 outcome 和补偿状态。

---

## 3. learn-claude-code：Canary diff 分类器

教学版先写一个纯函数，把 active baseline 和 candidate applied 分类。

~~~py
from dataclasses import dataclass
from typing import Literal


Decision = Literal["allow", "warn", "dry_run", "block"]
DiffClass = Literal[
    "same_decision",
    "candidate_more_strict",
    "candidate_more_lenient",
    "reason_changed",
    "evidence_expanded",
    "unknown",
]


@dataclass(frozen=True)
class GuardDecisionView:
    decision: Decision
    reason_code: str
    evidence_hash: str
    extra_evidence_reads: tuple[str, ...] = ()


STRICTNESS = {
    "allow": 0,
    "warn": 1,
    "dry_run": 2,
    "block": 3,
}


def classify_canary_diff(
    active: GuardDecisionView,
    candidate: GuardDecisionView,
) -> DiffClass:
    if candidate.extra_evidence_reads:
        return "evidence_expanded"

    active_level = STRICTNESS[active.decision]
    candidate_level = STRICTNESS[candidate.decision]

    if candidate_level > active_level:
        return "candidate_more_strict"

    if candidate_level < active_level:
        return "candidate_more_lenient"

    if active.reason_code != candidate.reason_code:
        return "reason_changed"

    if active.evidence_hash != candidate.evidence_hash:
        return "evidence_expanded"

    return "same_decision"
~~~

注意：这里把 evidence_expanded 放得很靠前。因为 candidate 一旦多读证据，哪怕决策方向没变，也会带来权限、成本、延迟和证据新鲜度风险。

---

## 4. learn-claude-code：差异预算闸门

下一步，把 canary 阶段内的 diff 和 outcome 汇总成预算消耗。

~~~py
from collections import Counter
from dataclasses import dataclass
from typing import Literal


CanaryDecision = Literal[
    "promote_next_stage",
    "hold_stage",
    "rollback_canary",
    "tighten_scope",
    "manual_review",
]


@dataclass(frozen=True)
class CanaryBudget:
    max_more_strict: int
    max_more_lenient: int
    max_reason_changed: int
    max_evidence_expanded: int
    max_unknown_rate: float
    max_bad_diff: int


@dataclass(frozen=True)
class CanaryWindow:
    total: int
    diff_classes: list[str]
    bad_diff_count: int
    unknown_outcome_count: int
    min_required_total: int


def decide_canary_window(
    window: CanaryWindow,
    budget: CanaryBudget,
) -> tuple[CanaryDecision, list[str]]:
    if window.total < window.min_required_total:
        return "hold_stage", ["insufficient_canary_sample"]

    counts = Counter(window.diff_classes)

    if window.bad_diff_count > budget.max_bad_diff:
        return "rollback_canary", ["confirmed_bad_diff_budget_exceeded"]

    unknown_rate = window.unknown_outcome_count / window.total
    if unknown_rate > budget.max_unknown_rate:
        return "hold_stage", ["outcome_lag_too_high"]

    if counts["candidate_more_lenient"] > budget.max_more_lenient:
        return "rollback_canary", ["lenient_diff_budget_exceeded"]

    if counts["candidate_more_strict"] > budget.max_more_strict:
        return "tighten_scope", ["strict_diff_budget_exceeded"]

    if counts["evidence_expanded"] > budget.max_evidence_expanded:
        return "manual_review", ["evidence_expansion_budget_exceeded"]

    if counts["reason_changed"] > budget.max_reason_changed:
        return "hold_stage", ["explanation_diff_budget_exceeded"]

    return "promote_next_stage", ["diff_budget_within_limits"]
~~~

这里的策略故意不对称：

- candidate_more_lenient 更危险，因为它可能放过原本该挡的动作，所以直接 rollback；
- candidate_more_strict 可能是修复 false_allow 的目标行为，但超过预算会误伤，所以先 tighten_scope；
- evidence_expanded 涉及证据边界，交给 manual_review；
- outcome lag 太高时不晋级，因为没有结果就无法证明安全。

---

## 5. pi-mono：RepairCanaryController

生产版可以把 canary 做成一个 stage controller。它订阅真实 decision/outcome 事件，实时更新预算，超过阈值就原子回滚 pointer，并把已影响决策写入 repair queue。

~~~ts
type GuardDecision = "allow" | "warn" | "dry_run" | "block"

type CanaryStage = {
  repairId: string
  patternId: string
  candidateVersion: string
  activeBaselineVersion: string
  stageId: string
  scope: {
    tenants: string[]
    riskSurfaces: string[]
    actionClasses: string[]
    trafficPercent: number
  }
  budget: {
    maxMoreStrict: number
    maxMoreLenient: number
    maxReasonChanged: number
    maxEvidenceExpanded: number
    maxUnknownRate: number
    maxBadDiff: number
  }
}

type CanaryDiffEvent = {
  type: "guard.repair_canary.diff"
  decisionId: string
  repairId: string
  stageId: string
  tenant: string
  riskSurface: string
  actionClass: string
  activeDecision: GuardDecision
  candidateDecision: GuardDecision
  diffClass:
    | "same_decision"
    | "candidate_more_strict"
    | "candidate_more_lenient"
    | "reason_changed"
    | "evidence_expanded"
    | "unknown"
  outcomeStatus: "pending" | "confirmed_safe" | "confirmed_risky" | "reversed_by_human" | "unavailable"
}

type CanaryAction =
  | { kind: "promote_next_stage"; receiptId: string }
  | { kind: "hold_stage"; reasons: string[] }
  | { kind: "tighten_scope"; reasons: string[]; nextScope: CanaryStage["scope"] }
  | { kind: "rollback_canary"; reasons: string[]; affectedDecisionIds: string[] }
  | { kind: "manual_review"; reasons: string[] }

type CanaryWindow = {
  total: number
  diffClasses: CanaryDiffEvent["diffClass"][]
  badDiffCount: number
  unknownOutcomeCount: number
  minRequiredTotal: number
}

function decideCanaryWindow(
  window: CanaryWindow,
  stage: CanaryStage,
): CanaryAction {
  if (window.total < window.minRequiredTotal) {
    return { kind: "hold_stage", reasons: ["insufficient_canary_sample"] }
  }

  const count = (diffClass: CanaryDiffEvent["diffClass"]) =>
    window.diffClasses.filter((value) => value === diffClass).length

  if (window.badDiffCount > stage.budget.maxBadDiff) {
    return {
      kind: "rollback_canary",
      reasons: ["confirmed_bad_diff_budget_exceeded"],
      affectedDecisionIds: [],
    }
  }

  if (window.unknownOutcomeCount / window.total > stage.budget.maxUnknownRate) {
    return { kind: "hold_stage", reasons: ["outcome_lag_too_high"] }
  }

  if (count("candidate_more_lenient") > stage.budget.maxMoreLenient) {
    return {
      kind: "rollback_canary",
      reasons: ["lenient_diff_budget_exceeded"],
      affectedDecisionIds: [],
    }
  }

  if (count("candidate_more_strict") > stage.budget.maxMoreStrict) {
    return {
      kind: "tighten_scope",
      reasons: ["strict_diff_budget_exceeded"],
      nextScope: {
        ...stage.scope,
        trafficPercent: Math.max(1, Math.floor(stage.scope.trafficPercent / 2)),
      },
    }
  }

  if (count("evidence_expanded") > stage.budget.maxEvidenceExpanded) {
    return { kind: "manual_review", reasons: ["evidence_expansion_budget_exceeded"] }
  }

  if (count("reason_changed") > stage.budget.maxReasonChanged) {
    return { kind: "hold_stage", reasons: ["explanation_diff_budget_exceeded"] }
  }

  return { kind: "promote_next_stage", receiptId: "pending_receipt" }
}

class RepairCanaryController {
  constructor(
    private readonly store: {
      loadStage(stageId: string): Promise<CanaryStage>
      listWindow(stageId: string): Promise<CanaryDiffEvent[]>
      rollbackToActive(stageId: string, reason: string): Promise<string>
      enqueueAffectedDecisions(decisionIds: string[], rollbackReceiptId: string): Promise<void>
      writeReceipt(stageId: string, action: CanaryAction): Promise<void>
    },
  ) {}

  async evaluate(stageId: string): Promise<CanaryAction> {
    const stage = await this.store.loadStage(stageId)
    const events = await this.store.listWindow(stageId)

    const badDiffs = events.filter((event) => {
      if (event.diffClass === "candidate_more_lenient") {
        return event.outcomeStatus === "confirmed_risky"
      }

      if (event.diffClass === "candidate_more_strict") {
        return event.outcomeStatus === "confirmed_safe"
      }

      return false
    })

    const unknownCount = events.filter((event) =>
      event.outcomeStatus === "pending" || event.outcomeStatus === "unavailable"
    ).length

    const decision = decideCanaryWindow({
      total: events.length,
      diffClasses: events.map((event) => event.diffClass),
      badDiffCount: badDiffs.length,
      unknownOutcomeCount: unknownCount,
      minRequiredTotal: 100,
    }, stage)

    if (decision.kind === "rollback_canary") {
      const receiptId = await this.store.rollbackToActive(stageId, decision.reasons.join(","))
      const affectedDecisionIds = events
        .filter((event) => event.diffClass !== "same_decision")
        .map((event) => event.decisionId)

      await this.store.enqueueAffectedDecisions(affectedDecisionIds, receiptId)

      const action: CanaryAction = {
        kind: "rollback_canary",
        reasons: decision.reasons,
        affectedDecisionIds,
      }
      await this.store.writeReceipt(stageId, action)
      return action
    }

    await this.store.writeReceipt(stageId, decision)
    return decision
  }
}
~~~

这里有两个生产细节：

1. rollback pointer 和 enqueue affected decisions 要在同一个事务边界里完成；
2. rollback 不是结束，所有 canary 期间被 candidate 影响过的决策都要进入 outcome backfill / compensation queue。

---

## 6. OpenClaw：课程 Cron 的类比实战

拿这条课程 Cron 做类比：

- shadow：先生成课程草稿、README diff、TOOLS diff，但不发群、不推 repo；
- canary：先发一个小范围渠道或执行 dry-run push，观察 messageId、commit、remote main、README 链接是否一致；
- diff budget：如果正文主题偏离已讲链路、README 漏目录、TOOLS 未更新、远端 commit 不一致，就消耗预算；
- rollback trigger：如果消息已发但 git push 失败，就不能假装成功，要补 push、补记忆、必要时发更正说明；
- affected decisions：这次 cron 影响过的 Telegram messageId、commit hash、TOOLS 条目都要能对账。

也就是说，Agent 只要开始真实影响外部世界，就要有差异账本。不是为了形式主义，而是为了出问题时知道影响范围、回滚点和补偿路径。

---

## 7. 工程落地清单

修复版护栏进入 canary 前，至少准备这些东西：

1. **Stage Spec**：candidateVersion、activeBaselineVersion、scope、trafficPercent；
2. **Diff Budget**：more_strict、more_lenient、reason_changed、evidence_expanded、unknown、bad_diff；
3. **Rollback Pointer**：能原子切回 active baseline；
4. **Affected Decision Queue**：回滚后能找回所有被 candidate 影响过的 decisionId；
5. **Outcome Backfill**：pending outcome 不足时 hold，不允许盲目晋级；
6. **Receipt**：每次 promote、hold、tighten、rollback 都要写收据。

一个好用的判断标准：

> Canary 结束时，你不只要知道“候选版有没有炸”，还要知道“它具体改变了哪些决策、这些改变有没有被真实 outcome 证明、如果回滚要修哪些现实影响”。

---

## 8. 小结

成熟 Agent 的护栏修复，不是 shadow 通过后直接放量，而是用 Decision Diff Budget 控制真实差异：多挡、少挡、改解释、多读证据、未知 outcome 都有预算；预算耗尽自动回滚，并把已影响决策转入回填和补偿。

这套机制的核心不是保守，而是可证明：**每一次 canary 晋级都能解释为什么风险还在预算内；每一次回滚都能解释影响过谁、怎么补回来。**
