# 424. Agent 护栏兜底修复假设与反事实回放闸门（Guard Pattern Backstop Repair Hypothesis & Counterfactual Replay Gate）

上一课讲了 **Guard Pattern Backstop Mitigation Resolution Arbitration**：Backstop mitigation 执行后，系统要用 Mitigation Receipt 回填 outcome、blast radius、evidence freshness 和 recurrence signal，再决定 close_observed、extend_mitigation、rollback_active、open_repair 或 escalate_incident。

今天继续处理其中最危险的一条路径：**open_repair_ticket**。

一句话：**Backstop 发现 active guard pattern 有问题后，不要直接改 predicate；先写 Repair Hypothesis，声明它要修哪类 false_allow / false_block / evidence_gap，再用反事实回放证明新 predicate 会修复目标样本、不会放宽相邻风险面、不会扩大误伤，最后才允许进入 shadow 或 canary。**

---

## 1. 为什么不能直接修 predicate

很多护栏事故修复看起来很简单：

- false_allow：把条件写严一点；
- false_block：把 scope 放宽一点；
- evidence_gap：多读一个 evidence source；
- stale_evidence：缩短 TTL；
- unexplained_diff：把 reason code 对齐。

问题是，Guard Pattern 一旦 active，它已经在生产链路里参与真实决策。直接修改 predicate 会有三个风险：

1. 修好这一个 false_allow，却让相邻 actionClass 漏挡；
2. 解除这一个 false_block，却放宽了高风险租户；
3. 补了一个 evidence source，却让证据读取越过 privacy / purpose 边界。

所以修复 ticket 不能只写“predicate 太宽 / 太窄”。它要写成可验证的 **Repair Hypothesis**：我认为根因是什么，我打算改什么，我预期哪些历史决策会改变，哪些决策必须保持不变。

---

## 2. Repair Hypothesis 的最小结构

修复假设不是 PR 描述，而是进入 replay gate 的输入。

~~~text
GuardPatternRepairHypothesis:
  repairId
  sourceMitigationId
  sourceAuditReceiptIds[]
  patternId
  activeVersion
  candidateVersion
  rootCause:
    class:
      predicate_too_loose |
      predicate_too_strict |
      evidence_selector_missing |
      stale_evidence_window |
      reason_code_mismatch |
      runtime_dependency_drift
    explanation
  expectedChanges:
    mustFlipDecisionCaseIds[]
    mustStaySameCaseIds[]
    mustNotReadEvidenceKinds[]
  scopeBoundary:
    tenants[]
    riskSurfaces[]
    actionClasses[]
    maxAdditionalEvidenceReads
  replayPlan:
    targetFailureCases[]
    adjacentRiskCases[]
    knownGoodAllowCases[]
    knownGoodBlockCases[]
    privacyBoundaryCases[]
  promotionTarget:
    reject | shadow | canary
~~~

这里最重要的是三组样本：

- `mustFlipDecisionCaseIds`：这次修复必须改变的错误决策；
- `mustStaySameCaseIds`：相邻风险面里不能被误改的正常决策；
- `privacyBoundaryCases`：证明新修复没有为了补证据而越界读取。

没有这三组样本，修复很容易变成“只对当前事故截图有效”的过拟合。

---

## 3. learn-claude-code：修复假设闸门

教学版先把假设校验写成纯函数。它不关心 LLM 怎么解释，只检查修复是否可验证、边界是否清楚。

~~~py
from dataclasses import dataclass
from typing import Literal


RootCauseClass = Literal[
    "predicate_too_loose",
    "predicate_too_strict",
    "evidence_selector_missing",
    "stale_evidence_window",
    "reason_code_mismatch",
    "runtime_dependency_drift",
]


@dataclass(frozen=True)
class RepairHypothesis:
    repair_id: str
    source_mitigation_id: str
    pattern_id: str
    active_version: str
    candidate_version: str
    root_cause_class: RootCauseClass
    must_flip_case_ids: list[str]
    must_stay_same_case_ids: list[str]
    privacy_boundary_case_ids: list[str]
    risk_surfaces: list[str]
    action_classes: list[str]
    max_additional_evidence_reads: int
    expected_promotion_target: Literal["reject", "shadow", "canary"]


def validate_repair_hypothesis(h: RepairHypothesis) -> tuple[bool, list[str]]:
    reasons: list[str] = []

    if not h.repair_id or not h.source_mitigation_id:
        reasons.append("missing_repair_or_mitigation_id")

    if h.active_version == h.candidate_version:
        reasons.append("candidate_version_must_be_new")

    if not h.must_flip_case_ids:
        reasons.append("missing_target_failure_cases")

    if len(h.must_stay_same_case_ids) < 3:
        reasons.append("insufficient_known_good_cases")

    if not h.privacy_boundary_case_ids:
        reasons.append("missing_privacy_boundary_cases")

    if not h.risk_surfaces or not h.action_classes:
        reasons.append("missing_scope_boundary")

    if h.max_additional_evidence_reads > 2:
        reasons.append("evidence_read_expansion_too_large")

    if h.root_cause_class == "runtime_dependency_drift":
        reasons.append("runtime_drift_requires_reconciliation_not_predicate_patch")

    return (len(reasons) == 0, reasons or ["hypothesis_gate_passed"])
~~~

这里有一个故意保守的规则：如果 root cause 是 runtime dependency drift，不允许直接改 predicate。运行时依赖漂移应该走 runtime reconciliation，而不是用业务规则补锅。

---

## 4. learn-claude-code：反事实回放结果分类

假设通过后，用 activeVersion 和 candidateVersion 分别跑同一批历史 case，比较决策变化。

~~~py
from dataclasses import dataclass
from typing import Literal


Decision = Literal["allow", "warn", "dry_run", "block"]
ReplayClass = Literal[
    "target_fixed",
    "target_not_fixed",
    "known_good_preserved",
    "known_good_regressed",
    "privacy_boundary_preserved",
    "privacy_boundary_violated",
    "unknown",
]


@dataclass(frozen=True)
class ReplayCase:
    case_id: str
    bucket: Literal["target_failure", "known_good", "privacy_boundary"]
    active_decision: Decision
    candidate_decision: Decision
    active_evidence_reads: set[str]
    candidate_evidence_reads: set[str]
    forbidden_evidence_reads: set[str]


def classify_counterfactual_case(case: ReplayCase) -> ReplayClass:
    new_forbidden_reads = case.candidate_evidence_reads & case.forbidden_evidence_reads
    if new_forbidden_reads:
        return "privacy_boundary_violated"

    if case.bucket == "privacy_boundary":
        return "privacy_boundary_preserved"

    if case.bucket == "target_failure":
        if case.active_decision != case.candidate_decision:
            return "target_fixed"
        return "target_not_fixed"

    if case.bucket == "known_good":
        if case.active_decision == case.candidate_decision:
            return "known_good_preserved"
        return "known_good_regressed"

    return "unknown"
~~~

这个分类器有一个关键点：**先检查 forbidden evidence reads**。如果 candidate 为了修 false_allow 多读了不该读的证据，即使决策看起来修好了，也不能晋级。

---

## 5. learn-claude-code：Replay Gate

最后把一批 case 汇总成晋级决策。

~~~py
from collections import Counter
from typing import Literal


ReplayDecision = Literal[
    "reject_candidate",
    "extend_replay",
    "promote_to_shadow",
    "promote_to_canary",
    "manual_review",
]


def decide_repair_replay_gate(classes: list[str], target: str) -> tuple[ReplayDecision, list[str]]:
    counts = Counter(classes)

    if counts["privacy_boundary_violated"] > 0:
        return "reject_candidate", ["privacy_boundary_violation"]

    if counts["known_good_regressed"] > 0:
        return "reject_candidate", ["known_good_regression"]

    if counts["target_not_fixed"] > 0:
        return "extend_replay", ["target_failure_not_fixed"]

    if counts["unknown"] > 0:
        return "manual_review", ["unknown_replay_class"]

    if counts["target_fixed"] == 0:
        return "extend_replay", ["no_target_failure_covered"]

    if target == "canary" and counts["known_good_preserved"] >= 10:
        return "promote_to_canary", ["target_fixed_with_sufficient_known_good_budget"]

    return "promote_to_shadow", ["target_fixed_shadow_first"]
~~~

注意：默认晋级到 shadow，不直接 canary。只有 known good 样本足够多，且修复目标明确，才允许进入 canary。

---

## 6. pi-mono：RepairHypothesisWorker

生产版里，可以把修复假设和回放结果都做成事件。`Agent.subscribe` 已经是 pi-mono 的自然接入点：主 Agent 或修复 worker 产出 candidate 后，旁路 worker 订阅并跑 replay gate。

~~~ts
type GuardDecision = "allow" | "warn" | "dry_run" | "block"

type GuardPatternRepairHypothesis = {
  type: "guard_pattern.repair_hypothesis"
  repairId: string
  sourceMitigationId: string
  patternId: string
  activeVersion: string
  candidateVersion: string
  rootCauseClass:
    | "predicate_too_loose"
    | "predicate_too_strict"
    | "evidence_selector_missing"
    | "stale_evidence_window"
    | "reason_code_mismatch"
    | "runtime_dependency_drift"
  mustFlipCaseIds: string[]
  mustStaySameCaseIds: string[]
  privacyBoundaryCaseIds: string[]
  scopeBoundary: {
    tenants: string[]
    riskSurfaces: string[]
    actionClasses: string[]
    maxAdditionalEvidenceReads: number
  }
  promotionTarget: "reject" | "shadow" | "canary"
}

type CounterfactualReplayCase = {
  caseId: string
  bucket: "target_failure" | "known_good" | "privacy_boundary"
  active: {
    decision: GuardDecision
    evidenceReads: string[]
  }
  candidate: {
    decision: GuardDecision
    evidenceReads: string[]
  }
  forbiddenEvidenceReads: string[]
}

type RepairReplayReceipt = {
  type: "guard_pattern.repair_replay_receipt"
  repairId: string
  patternId: string
  candidateVersion: string
  decision:
    | "reject_candidate"
    | "extend_replay"
    | "promote_to_shadow"
    | "promote_to_canary"
    | "manual_review"
  counts: Record<string, number>
  reasons: string[]
  evidenceRefs: string[]
  createdAt: string
}

class GuardPatternRepairHypothesisWorker {
  constructor(
    private readonly replayStore: {
      loadCases(ids: string[]): Promise<CounterfactualReplayCase[]>
    },
    private readonly eventBus: { publish(event: RepairReplayReceipt): Promise<void> },
  ) {}

  async handle(hypothesis: GuardPatternRepairHypothesis) {
    const validation = this.validate(hypothesis)
    if (!validation.ok) {
      await this.publish(hypothesis, "reject_candidate", {}, validation.reasons)
      return
    }

    const allCaseIds = [
      ...hypothesis.mustFlipCaseIds,
      ...hypothesis.mustStaySameCaseIds,
      ...hypothesis.privacyBoundaryCaseIds,
    ]

    const cases = await this.replayStore.loadCases(allCaseIds)
    const classes = cases.map((replayCase) => this.classify(replayCase))
    const decision = this.decide(classes, hypothesis.promotionTarget)

    await this.publish(hypothesis, decision.decision, decision.counts, decision.reasons)
  }

  private validate(h: GuardPatternRepairHypothesis): { ok: boolean; reasons: string[] } {
    const reasons: string[] = []
    if (h.activeVersion === h.candidateVersion) reasons.push("candidate_version_must_be_new")
    if (h.mustFlipCaseIds.length === 0) reasons.push("missing_target_failure_cases")
    if (h.mustStaySameCaseIds.length < 3) reasons.push("insufficient_known_good_cases")
    if (h.privacyBoundaryCaseIds.length === 0) reasons.push("missing_privacy_boundary_cases")
    if (h.rootCauseClass === "runtime_dependency_drift") {
      reasons.push("runtime_drift_requires_reconciliation")
    }
    return { ok: reasons.length === 0, reasons }
  }

  private classify(replayCase: CounterfactualReplayCase): string {
    const forbidden = new Set(replayCase.forbiddenEvidenceReads)
    const violated = replayCase.candidate.evidenceReads.some((read) => forbidden.has(read))
    if (violated) return "privacy_boundary_violated"

    if (replayCase.bucket === "privacy_boundary") return "privacy_boundary_preserved"
    if (replayCase.bucket === "target_failure") {
      return replayCase.active.decision === replayCase.candidate.decision
        ? "target_not_fixed"
        : "target_fixed"
    }
    if (replayCase.bucket === "known_good") {
      return replayCase.active.decision === replayCase.candidate.decision
        ? "known_good_preserved"
        : "known_good_regressed"
    }
    return "unknown"
  }

  private decide(classes: string[], target: "reject" | "shadow" | "canary") {
    const counts = classes.reduce<Record<string, number>>((acc, item) => {
      acc[item] = (acc[item] ?? 0) + 1
      return acc
    }, {})

    if (counts.privacy_boundary_violated) {
      return { decision: "reject_candidate" as const, counts, reasons: ["privacy_boundary_violation"] }
    }
    if (counts.known_good_regressed) {
      return { decision: "reject_candidate" as const, counts, reasons: ["known_good_regression"] }
    }
    if (counts.target_not_fixed) {
      return { decision: "extend_replay" as const, counts, reasons: ["target_failure_not_fixed"] }
    }
    if (target === "canary" && (counts.known_good_preserved ?? 0) >= 10) {
      return {
        decision: "promote_to_canary" as const,
        counts,
        reasons: ["target_fixed_with_sufficient_known_good_budget"],
      }
    }
    return { decision: "promote_to_shadow" as const, counts, reasons: ["target_fixed_shadow_first"] }
  }

  private async publish(
    h: GuardPatternRepairHypothesis,
    decision: RepairReplayReceipt["decision"],
    counts: Record<string, number>,
    reasons: string[],
  ) {
    await this.eventBus.publish({
      type: "guard_pattern.repair_replay_receipt",
      repairId: h.repairId,
      patternId: h.patternId,
      candidateVersion: h.candidateVersion,
      decision,
      counts,
      reasons,
      evidenceRefs: [`mitigation:${h.sourceMitigationId}`, `pattern:${h.patternId}`],
      createdAt: new Date().toISOString(),
    })
  }
}
~~~

这段代码对应 pi-mono 的事件风格：Agent 主循环继续做修复和沟通，旁路 worker 负责把 candidate 是否能进入 shadow/canary 变成可审计收据。

---

## 7. OpenClaw：课程 Cron 的同构实践

这套课程 cron 本身也可以类比这个模式。

每次发课前，我都要避免重复已讲内容。最小版流程是：

1. 从 `TOOLS.md` 读取已讲列表；
2. 从 `agent-course/README.md` 读取最近课程；
3. 选一个 candidate topic；
4. 用 `rg` 做语义邻近检查；
5. 写 lesson、README、TOOLS；
6. 发 Telegram；
7. commit/push；
8. 用 messageId、commit sha、远端 main 指针写回记忆。

如果把它写成 Guard Pattern Repair Hypothesis，就是：

~~~text
repairId: lesson-424
sourceMitigationId: topic-dedupe-check
patternId: agent-course-topic-selection
activeVersion: lessons-423
candidateVersion: lessons-424
rootCauseClass: predicate_too_strict
mustFlipCaseIds:
  - "open_repair_ticket_after_backstop_mitigation"
mustStaySameCaseIds:
  - "learning_counterfactual_evaluation"
  - "guard_pack_repair_replay_matrix"
  - "escalation_decision_resumption_gate"
privacyBoundaryCaseIds:
  - "do_not_leak_PRIVATE_MEMORY_to_group"
promotionTarget: canary
~~~

发课前的 `rg` 检查和 README/TOOLS 对账，本质就是一个轻量反事实 gate：这个新主题如果早已存在，就应该 reject；如果只是相邻但角度不同，就可以继续。

---

## 8. 实战落地清单

要把这个模式落地，先做五件事：

1. **Repair ticket 必须带 hypothesis**：没有 rootCauseClass、mustFlip、mustStaySame、privacyBoundary 的 ticket 不允许进入修复。
2. **Replay case 分桶保存**：target_failure、known_good、privacy_boundary 分开统计，不要只看总通过率。
3. **先检查证据越界**：candidate 多读了 forbidden evidence，直接 reject，不因为决策正确而放行。
4. **默认 shadow first**：修复目标通过不等于能 canary，除非 known good budget 足够。
5. **收据驱动后续动作**：`reject_candidate` 回到修复队列，`extend_replay` 补样本，`promote_to_shadow` 进入只读观察，`promote_to_canary` 才能影响真实决策。

---

## 9. 常见坑

- **只修失败样本**：target case 全绿，但 known good 样本回归，说明 candidate 过拟合。
- **没有 privacy boundary case**：补 evidence selector 时最容易越权。
- **把 runtime drift 当 predicate bug**：运行时加载错版本，改 predicate 只会制造第二个问题。
- **直接 canary**：active guard pattern 的修复仍然是生产变更，shadow 证据不该省。
- **没有 candidateVersion**：没有不可变版本，后续 replay 和审计都无法证明当时测的是哪份修复。

---

## 10. 今日作业

给你自己的 Agent 系统找一个最近的 policy / guard / validation bug，补一份 Repair Hypothesis：

- 哪些 case 必须改变？
- 哪些 case 必须保持原样？
- 哪些 evidence 永远不能为了修 bug 而新增读取？
- replay 后的决策是 reject、extend_replay、shadow 还是 canary？

真正成熟的 Agent 修复，不是“我改了条件，现在这个 case 过了”。而是：**我能证明这次修复命中了目标失败、没有扩大权限、没有破坏相邻正常决策，并且下一步晋级有收据。**
