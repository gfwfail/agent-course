# 432. Agent 护栏复封后的取证隔离与修复分流

> **核心思想：ResealReceipt 不是终点，它只是把坏候选从发布路径上拉下来。成熟 Agent 还要立刻生成 Reseal Case，把 candidate version 隔离、冻结相关 scope、抽取失败证据、分类根因，并把后续动作分流到 reject、repair、extend replay 或 incident escalation。否则复封只是“停止继续坏”，没有把这次失败变成下一版修复的可靠输入。**

---

## 1. 为什么复封后还要单独建 Case

上一课讲了 Unseal 后的 Scoped Trial：candidate 只能在受限 shadow/canary 窗口里证明自己。一旦命中 regression seed、bad diff、证据边界扩大、scope violation 或 runtime divergence，就自动写 ResealReceipt。

但 ResealReceipt 本身只回答一个问题：**为什么现在必须重新封住它。**

它还没回答：

1. 哪些决策被这个 trial 影响过；
2. 失败是旧问题复发、修复过拟合，还是新依赖漂移；
3. candidate 是否应该永久 reject，还是还能进入 repair；
4. 是否需要扩大 tombstone 的 forbidden predicate；
5. 是否要冻结相邻 scope，防止同类候选继续试运行。

所以复封后应该创建一个独立对象：**Reseal Case**。

它的职责不是发布控制，而是事故取证和修复分流。

---

## 2. 最小数据模型

~~~text
ResealCase:
  caseId
  resealReceiptId
  tombstoneId
  patternId
  candidateVersion
  trialId
  trigger:
    regression_seed_failed |
    bad_diff_budget_exceeded |
    evidence_boundary_expanded |
    scope_violation |
    owner_approval_expired |
    runtime_divergence
  affectedDecisionIds[]
  quarantine:
    candidateVersionBlocked: true
    freezeSiblingCandidates: boolean
    frozenScopes[]
  forensicBundle:
    failedRegressionSeeds[]
    badDiffDecisionIds[]
    evidenceBoundaryDelta
    runtimeFingerprintDelta
    dependencyDelta
  rootCauseClass:
    old_failure_recurred |
    repair_overfit |
    predicate_too_broad |
    predicate_too_loose |
    evidence_contract_broken |
    runtime_dependency_drift |
    unknown
  routingDecision:
    reject_candidate |
    open_repair |
    extend_replay |
    escalate_incident |
    manual_review
  status:
    opened | routed | repaired | rejected | escalated | closed
~~~

这里有一个关键细节：**quarantine 和 routing 是两件事**。

Quarantine 先阻断污染，routing 再决定下一步。不要等 root cause 分析完成才隔离 candidate，因为分析期间它可能已经被其他 release worker 或 sub-agent 当成“可继续修”的版本继续推进。

---

## 3. 复封后的四个硬动作

第一，冻结 candidate version。
被 Reseal 的版本不能再进入 shadow、canary、active，也不能被复制成新版本绕过 trial history。

第二，回填 affected decisions。
如果 trial 进入过 canary，就要把真实影响过的 decisionId 全部进入 impact backfill。即使只是 shadow，也要保留 comparison evidence，供 repair replay 使用。

第三，扩展 tombstone 证据。
如果这次失败不是原 seed，而是相邻风险面暴露出来的新失败，要把新的 predicate hash、evidence boundary delta、runtime fingerprint delta 加进 tombstone extension，而不是只写在日志里。

第四，分流修复路径。
旧问题复发通常 reject candidate；修复过拟合进入 repair；依赖漂移进入 replay/dependency lock；scope violation 通常升级 incident 或 owner review。

---

## 4. learn-claude-code：从 ResealReceipt 创建 Case

教学版先写纯函数，把复封收据和 trial observation 转成 ResealCase。重点是：生成 case 时就把 candidate 隔离。

~~~py
from dataclasses import dataclass
from typing import Literal


ResealTrigger = Literal[
    "regression_seed_failed",
    "bad_diff_budget_exceeded",
    "evidence_boundary_expanded",
    "scope_violation",
    "owner_approval_expired",
    "runtime_divergence",
]

RootCauseClass = Literal[
    "old_failure_recurred",
    "repair_overfit",
    "predicate_too_broad",
    "predicate_too_loose",
    "evidence_contract_broken",
    "runtime_dependency_drift",
    "unknown",
]

RoutingDecision = Literal[
    "reject_candidate",
    "open_repair",
    "extend_replay",
    "escalate_incident",
    "manual_review",
]


@dataclass(frozen=True)
class ResealReceipt:
    receipt_id: str
    trial_id: str
    tombstone_id: str
    pattern_id: str
    candidate_version: str
    trigger: ResealTrigger
    affected_decision_ids: tuple[str, ...]


@dataclass(frozen=True)
class TrialFailureEvidence:
    failed_regression_seed_ids: tuple[str, ...]
    bad_diff_decision_ids: tuple[str, ...]
    evidence_boundary_delta: int
    runtime_fingerprint_changed: bool
    dependency_changed: bool
    scope_violated: bool


@dataclass(frozen=True)
class CandidateQuarantine:
    candidate_version_blocked: bool
    freeze_sibling_candidates: bool
    frozen_scopes: tuple[str, ...]


@dataclass(frozen=True)
class ResealCase:
    case_id: str
    reseal_receipt_id: str
    tombstone_id: str
    pattern_id: str
    candidate_version: str
    trigger: ResealTrigger
    affected_decision_ids: tuple[str, ...]
    quarantine: CandidateQuarantine
    root_cause_class: RootCauseClass
    routing_decision: RoutingDecision


def classify_root_cause(
    receipt: ResealReceipt,
    evidence: TrialFailureEvidence,
) -> RootCauseClass:
    if evidence.failed_regression_seed_ids:
        return "old_failure_recurred"
    if receipt.trigger == "evidence_boundary_expanded":
        return "evidence_contract_broken"
    if evidence.runtime_fingerprint_changed or evidence.dependency_changed:
        return "runtime_dependency_drift"
    if receipt.trigger == "scope_violation" or evidence.scope_violated:
        return "predicate_too_broad"
    if receipt.trigger == "bad_diff_budget_exceeded":
        return "repair_overfit"
    return "unknown"


def route_reseal_case(root_cause: RootCauseClass) -> RoutingDecision:
    if root_cause == "old_failure_recurred":
        return "reject_candidate"
    if root_cause in {"repair_overfit", "predicate_too_broad", "predicate_too_loose"}:
        return "open_repair"
    if root_cause == "runtime_dependency_drift":
        return "extend_replay"
    if root_cause == "evidence_contract_broken":
        return "escalate_incident"
    return "manual_review"


def open_reseal_case(
    receipt: ResealReceipt,
    evidence: TrialFailureEvidence,
) -> ResealCase:
    root_cause = classify_root_cause(receipt, evidence)

    quarantine = CandidateQuarantine(
        candidate_version_blocked=True,
        freeze_sibling_candidates=root_cause in {
            "old_failure_recurred",
            "evidence_contract_broken",
            "runtime_dependency_drift",
        },
        frozen_scopes=("trial_scope",) if evidence.scope_violated else (),
    )

    return ResealCase(
        case_id=f"reseal-case:{receipt.receipt_id}",
        reseal_receipt_id=receipt.receipt_id,
        tombstone_id=receipt.tombstone_id,
        pattern_id=receipt.pattern_id,
        candidate_version=receipt.candidate_version,
        trigger=receipt.trigger,
        affected_decision_ids=receipt.affected_decision_ids,
        quarantine=quarantine,
        root_cause_class=root_cause,
        routing_decision=route_reseal_case(root_cause),
    )
~~~

这个函数有意不调用 LLM。复封后的第一步应该是确定性控制面动作：隔离、分类、分流。LLM 可以参与后续 repair hypothesis，但不应该决定是否先把 candidate 停掉。

---

## 5. pi-mono：事件驱动的 ResealCaseWorker

生产版可以把 trial worker 发出的事件接到一个 worker，按 eventId 幂等创建 case，并同步写 Quarantine Registry。

~~~ts
type ResealTrigger =
  | "regression_seed_failed"
  | "bad_diff_budget_exceeded"
  | "evidence_boundary_expanded"
  | "scope_violation"
  | "owner_approval_expired"
  | "runtime_divergence";

type ResealEvent = {
  eventId: string;
  trialId: string;
  resealReceiptId: string;
  tombstoneId: string;
  patternId: string;
  candidateVersion: string;
  trigger: ResealTrigger;
  affectedDecisionIds: string[];
  evidence: {
    failedRegressionSeedIds: string[];
    badDiffDecisionIds: string[];
    evidenceBoundaryDelta: number;
    runtimeFingerprintChanged: boolean;
    dependencyChanged: boolean;
    scopeViolated: boolean;
  };
};

type ResealRoute =
  | "reject_candidate"
  | "open_repair"
  | "extend_replay"
  | "escalate_incident"
  | "manual_review";

export class ResealCaseWorker {
  constructor(
    private readonly cases: ResealCaseStore,
    private readonly quarantine: QuarantineRegistry,
    private readonly repairQueue: RepairQueue,
    private readonly incidentQueue: IncidentQueue,
  ) {}

  async handle(event: ResealEvent) {
    const existing = await this.cases.findBySourceEvent(event.eventId);
    if (existing) return existing;

    const rootCause = classifyRootCause(event);
    const route = routeReseal(rootCause);

    await this.quarantine.blockVersion({
      patternId: event.patternId,
      candidateVersion: event.candidateVersion,
      tombstoneId: event.tombstoneId,
      reason: rootCause,
      sourceEventId: event.eventId,
      freezeSiblingCandidates: shouldFreezeSiblings(rootCause),
    });

    const resealCase = await this.cases.create({
      sourceEventId: event.eventId,
      resealReceiptId: event.resealReceiptId,
      tombstoneId: event.tombstoneId,
      patternId: event.patternId,
      candidateVersion: event.candidateVersion,
      trigger: event.trigger,
      affectedDecisionIds: event.affectedDecisionIds,
      forensicBundle: event.evidence,
      rootCause,
      route,
      status: "routed",
    });

    if (route === "open_repair" || route === "extend_replay") {
      await this.repairQueue.enqueue({
        caseId: resealCase.caseId,
        patternId: event.patternId,
        blockedVersion: event.candidateVersion,
        requiredReplaySeeds: event.evidence.failedRegressionSeedIds,
        affectedDecisionIds: event.affectedDecisionIds,
      });
    }

    if (route === "escalate_incident") {
      await this.incidentQueue.enqueue({
        caseId: resealCase.caseId,
        severity: "high",
        reason: "reseal_evidence_contract_broken",
      });
    }

    return resealCase;
  }
}

function routeReseal(rootCause: string): ResealRoute {
  switch (rootCause) {
    case "old_failure_recurred":
      return "reject_candidate";
    case "repair_overfit":
    case "predicate_too_broad":
    case "predicate_too_loose":
      return "open_repair";
    case "runtime_dependency_drift":
      return "extend_replay";
    case "evidence_contract_broken":
      return "escalate_incident";
    default:
      return "manual_review";
  }
}
~~~

这里有两个工程点：

1. QuarantineRegistry.blockVersion 必须早于 repairQueue enqueue；
2. worker 必须按 eventId 幂等，否则同一个 reseal event 重放会生成多个 repair ticket。

---

## 6. OpenClaw：Cron 课程里的实际类比

我们这个课程 cron 也有类似流程：

1. 先检查 TOOLS.md 已讲内容，避免重复；
2. 发现主题冲突时，不是硬讲，而是换到相邻未讲主题；
3. lesson、README、TOOLS、Telegram messageId、Git commit 都要对账；
4. 如果 push 后发现 markdown trailing whitespace，就追加修复提交并记录；
5. 下次 cron 读取当天 memory，知道上次已经修过什么。

把它映射到 Reseal Case：

- TOOLS.md 已讲内容是 tombstone / regression seed；
- 新 lesson 的候选主题是 candidate version；
- 去重失败就是 trial bad diff；
- 修复提交和 memory 记录就是 forensic bundle；
- 下次选题避开同坑就是 quarantine + repair routing。

课程 cron 看起来只是发课，实质也是一个小型 Agent 发布系统：它要证明自己没有重复、没有漏写、没有把外部副作用当成本地成功。

---

## 7. 实战 Checklist

- Reseal 后立刻创建 ResealCase，不要只写日志。
- Candidate version 必须进入 Quarantine Registry。
- affectedDecisionIds 必须进入 impact backfill 或 shadow evidence archive。
- rootCauseClass 要结构化，不能只写自由文本。
- repairQueue 只能消费已隔离的 candidate。
- 旧 regression seed 再失败，默认 reject candidate。
- evidence boundary 或 runtime dependency 出问题，优先升级 incident 或 extend replay。
- ResealCase 关闭前，要证明 tombstone extension、repair ticket、incident ticket 或 reject receipt 至少一个已经落地。

---

## 8. 一句话总结

**Reseal 是刹车，Reseal Case 是事故处理器。好的 Agent 护栏系统不会只把坏候选停掉，还会把它隔离、取证、分类，并把下一步修复路径变成可追踪的工程对象。**
