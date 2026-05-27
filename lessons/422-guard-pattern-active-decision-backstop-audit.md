# 422. Agent 护栏模式激活后的决策兜底审计（Guard Pattern Active Decision Backstop Audit）

上一课讲了 **Guard Pattern Active Promotion & Runtime Convergence Gate**：canary 全部通过后，也要冻结窗口、校验 LKG、runtime readiness、dependency fingerprint 和 rollback probe，再用 CAS 切 active/LKG，并证明所有 runtime group 收敛。

今天继续往后走一步：**active 已经收敛以后，怎么证明真实决策没有悄悄偏离，并且在发现异常时能先兜底、再回滚、再修复。**

一句话：**Guard Pattern active 后不能只靠 runtime convergence 安心；要给高风险决策加 Backstop Audit，旁路重算 expected decision，按 false_allow / false_block / unexplained_diff / evidence_gap 分类，先触发 bounded mitigation，再决定 rollback_active、quarantine_scope、open_repair 或 keep_active。**

---

## 1. 为什么收敛以后还要兜底审计

Convergence probe 证明的是：所有 runtime group 加载了同一个 pattern 版本，并且关键 fingerprint 一致。

但它不能证明三件事：

- 真实输入分布是否已经超出 canary 见过的范围；
- pattern 在组合场景里是否被其他 guard、policy、memory 或 tool schema 影响；
- 某次 allow/block 是否真的符合业务后果。

所以 active 后要补一个轻量的旁路审计：

1. 主路径照常执行 active pattern；
2. 对高风险或抽样命中的决策，旁路重算 expected decision；
3. 把 primary decision、expected decision、证据覆盖和真实 outcome 绑定起来；
4. 发现异常时先做有限兜底动作，避免扩大影响；
5. 再进入回滚、隔离、修复或继续观察。

这不是把生产流量再跑一遍完整 replay，而是只对关键决策做可解释、可预算、可追责的 backstop。

---

## 2. Backstop Audit 的最小输入

一个 backstop audit event 至少要包含这些字段：

~~~text
GuardPatternDecisionEvent:
  decisionId
  patternId
  activeVersion
  tenantId
  riskSurface
  actionClass
  primaryDecision
  primaryReasons[]
  evidenceRefs[]
  runtimeFingerprint
  sampledBecause[]
  outcomeRef?

BackstopPolicy:
  sampleRateByRiskSurface
  alwaysAuditActionClasses[]
  maxEvidenceAgeSeconds
  allowedDecisionDelta
  mitigationBudget:
    maxAutoBlocksPerTenant
    maxAutoDryRunsPerHour
    maxQuarantineScopes
~~~

关键点是 sampledBecause。以后排查时要知道这条审计是因为高风险动作、随机抽样、近期 drift、用户投诉，还是某个 pattern 刚 active 后的观察窗口。

---

## 3. learn-claude-code：审计差异分类器

教学版先写一个纯函数，把 primary 与 backstop 的差异压成稳定分类。

~~~py
from dataclasses import dataclass
from typing import Literal


Decision = Literal["allow", "warn", "dry_run", "block"]
AuditClass = Literal[
    "match",
    "false_allow",
    "false_block",
    "unexplained_diff",
    "evidence_gap",
    "stale_evidence",
]


@dataclass(frozen=True)
class BackstopInput:
    primary_decision: Decision
    expected_decision: Decision
    primary_reasons: list[str]
    expected_reasons: list[str]
    evidence_refs: list[str]
    oldest_evidence_age_seconds: int
    max_evidence_age_seconds: int
    explanation_overlap: float


def classify_backstop(input: BackstopInput) -> tuple[AuditClass, list[str]]:
    if not input.evidence_refs:
        return "evidence_gap", ["missing_evidence_refs"]

    if input.oldest_evidence_age_seconds > input.max_evidence_age_seconds:
        return "stale_evidence", ["evidence_too_old"]

    if input.primary_decision == input.expected_decision:
        if input.explanation_overlap < 0.5:
            return "unexplained_diff", ["same_decision_different_reasoning"]
        return "match", ["decision_and_reasoning_match"]

    if input.primary_decision in ("allow", "warn") and input.expected_decision in ("dry_run", "block"):
        return "false_allow", [
            f"primary:{input.primary_decision}",
            f"expected:{input.expected_decision}",
        ]

    if input.primary_decision in ("dry_run", "block") and input.expected_decision in ("allow", "warn"):
        return "false_block", [
            f"primary:{input.primary_decision}",
            f"expected:{input.expected_decision}",
        ]

    return "unexplained_diff", [
        f"primary:{input.primary_decision}",
        f"expected:{input.expected_decision}",
    ]
~~~

这里不要只看 allow/block 是否一致。理由差异也要记录，因为很多事故一开始表现为“结果一样但理由开始漂移”，再过几小时才变成真实误伤或漏挡。

---

## 4. learn-claude-code：兜底动作闸门

审计发现异常后，不能马上全局回滚。先按风险和预算做 bounded mitigation。

~~~py
from dataclasses import dataclass
from typing import Literal


MitigationAction = Literal[
    "record_only",
    "force_dry_run_for_scope",
    "temporary_block_action_class",
    "quarantine_scope",
    "rollback_active_pattern",
    "manual_review",
]


@dataclass(frozen=True)
class MitigationInput:
    audit_class: str
    risk_surface: str
    action_class: str
    tenant_recent_false_allows: int
    global_recent_false_allows: int
    scope_quarantines_active: int
    max_scope_quarantines: int
    action_reversible: bool


def decide_mitigation(input: MitigationInput) -> tuple[MitigationAction, list[str]]:
    if input.audit_class == "match":
        return "record_only", ["audit_match"]

    if input.audit_class in ("evidence_gap", "stale_evidence"):
        return "force_dry_run_for_scope", [input.audit_class]

    if input.audit_class == "false_block":
        return "manual_review", ["possible_user_impact_false_block"]

    if input.audit_class == "false_allow":
        if not input.action_reversible:
            return "temporary_block_action_class", ["irreversible_false_allow"]

        if input.global_recent_false_allows >= 3:
            return "rollback_active_pattern", ["false_allow_budget_exhausted"]

        if input.scope_quarantines_active < input.max_scope_quarantines:
            return "quarantine_scope", ["bounded_scope_quarantine"]

        return "manual_review", ["quarantine_budget_exhausted"]

    return "manual_review", ["unexplained_decision_delta"]
~~~

这里的原则是：漏挡比误挡更危险，但误挡也不能忽略。false_allow 对不可逆动作要更保守；false_block 更偏向人工复核和用户影响评估。

---

## 5. pi-mono：ActiveDecisionBackstopWorker

生产版可以把 backstop 做成事件流旁路 worker，不阻塞主路径，但能快速触发兜底。

~~~ts
type GuardDecision = "allow" | "warn" | "dry_run" | "block"
type BackstopClass =
  | "match"
  | "false_allow"
  | "false_block"
  | "unexplained_diff"
  | "evidence_gap"
  | "stale_evidence"

type BackstopAuditReceipt = {
  type: "guard_pattern.backstop_audit_receipt"
  decisionId: string
  patternId: string
  activeVersion: number
  primaryDecision: GuardDecision
  expectedDecision: GuardDecision
  auditClass: BackstopClass
  mitigationAction: string
  sampledBecause: string[]
  evidenceRefs: string[]
  runtimeFingerprint: string
  createdAt: string
}

class ActiveDecisionBackstopWorker {
  constructor(
    private readonly eventBus: EventBus,
    private readonly backstopEvaluator: BackstopEvaluator,
    private readonly mitigationGate: MitigationGate,
    private readonly receiptStore: ReceiptStore,
  ) {}

  async handle(event: GuardPatternDecisionEvent) {
    if (!this.shouldAudit(event)) {
      return
    }

    const expected = await this.backstopEvaluator.recompute({
      patternId: event.patternId,
      activeVersion: event.activeVersion,
      inputRef: event.inputRef,
      evidenceRefs: event.evidenceRefs,
      runtimeFingerprint: event.runtimeFingerprint,
    })

    const auditClass = classifyBackstop({
      primaryDecision: event.primaryDecision,
      expectedDecision: expected.decision,
      primaryReasons: event.reasons,
      expectedReasons: expected.reasons,
      evidenceRefs: event.evidenceRefs,
      oldestEvidenceAgeSeconds: expected.oldestEvidenceAgeSeconds,
      maxEvidenceAgeSeconds: expected.maxEvidenceAgeSeconds,
      explanationOverlap: expected.explanationOverlap,
    })

    const mitigation = await this.mitigationGate.decide({
      event,
      expected,
      auditClass,
    })

    const receipt: BackstopAuditReceipt = {
      type: "guard_pattern.backstop_audit_receipt",
      decisionId: event.decisionId,
      patternId: event.patternId,
      activeVersion: event.activeVersion,
      primaryDecision: event.primaryDecision,
      expectedDecision: expected.decision,
      auditClass,
      mitigationAction: mitigation.action,
      sampledBecause: event.sampledBecause,
      evidenceRefs: event.evidenceRefs,
      runtimeFingerprint: event.runtimeFingerprint,
      createdAt: new Date().toISOString(),
    }

    await this.receiptStore.append(receipt)
    await this.eventBus.publish("guard_pattern.backstop_audited", receipt)
  }

  private shouldAudit(event: GuardPatternDecisionEvent) {
    return (
      event.riskSurface === "external_side_effect" ||
      event.actionClass === "irreversible_write" ||
      event.sampledBecause.includes("post_active_observation")
    )
  }
}
~~~

注意：这个 worker 不应该拿完整原始隐私数据去重算。输入最好是 inputRef + 最小 evidenceRefs，通过 evidence budget 决定可读到 hash、summary、redacted text 还是 raw。

---

## 6. OpenClaw 课程 Cron 的对应实践

这套课程 cron 本身就是一个小型 active decision 系统：

- **primary decision**：选题、写课、发 Telegram、改 README/TOOLS、commit/push；
- **backstop audit**：检查 TOOLS.md 已讲内容、README 最新编号、git diff、远端 commit；
- **evidence refs**：lesson path、Telegram messageId、commit sha、memory 日志；
- **mitigation**：如果发现重复选题、push 失败、消息未发出，不能假装完成，要 hold、补发、重试或记录阻塞。

一个实用收据可以长这样：

~~~json
{
  "type": "course.backstop_audit_receipt",
  "lesson": 422,
  "primaryDecision": "publish",
  "checks": {
    "topicDeduped": true,
    "lessonFileExists": true,
    "readmeUpdated": true,
    "toolsUpdated": true,
    "telegramSent": true,
    "gitPushed": true
  },
  "mitigation": "record_only"
}
~~~

这不是形式主义。长期自动化任务最怕“看起来跑完了，实际上某个副作用没成功”。Backstop audit 就是在自动任务完成后再问一句：现实世界真的和我的完成状态一致吗？

---

## 7. 工程建议

- active 后至少保留一个 post-active observation window，不要 promotion 成功就撤掉观察；
- 对不可逆副作用、资金、权限、公开发布类动作设置 always audit；
- backstop evaluator 要版本化，避免审计器自己漂移后制造假信号；
- mitigation 必须有预算，避免一个异常把全局系统冻结；
- receipt 要能反查原始 decision、expected decision、证据 refs、runtime fingerprint 和采取过的兜底动作；
- false_allow、false_block 要进入不同修复队列，前者优先停损，后者优先评估用户影响和误伤范围。

---

## 8. 记忆点

**Runtime convergence 证明 active pattern 被一致加载；Backstop Audit 证明 active pattern 在真实决策里仍然可信。**

成熟 Agent 的护栏不是发布后就相信自己，而是在真实流量里持续抽样、旁路重算、有限兜底，并把每一次异常变成可修复、可回放、可关闭的工程证据。
