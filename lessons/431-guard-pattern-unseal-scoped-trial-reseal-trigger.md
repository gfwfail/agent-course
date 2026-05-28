# 431. Agent 护栏解封后的限域试运行与复封触发（Guard Pattern Unseal Scoped Trial & Reseal Trigger）

上一课讲了 **Guard Pattern Tombstone Hit Unseal Gate & Release Blocker**：candidate 命中 Tombstone 后不能靠 warning 放行，必须生成 ReleaseBlocker；只有提交 Repair Hypothesis、Regression Seed proof、predicate diff、evidence boundary proof、scope diff 和必要 owner approval，才能写 UnsealReceipt。

今天继续讲下一步：**解封以后不能直接当成正常版本发布。**

一句话：**UnsealReceipt 只代表“允许进入一个受限试运行窗口”，不是洗白历史墓碑。系统要把 candidate 放进 Scoped Trial，按 scope、stage、样本量、时间和差异预算重新 shadow/canary；一旦 regression seed 失败、现实 outcome 变坏、证据边界扩大或 owner 条件失效，就自动 Reseal，把版本重新封回 Tombstone 阻断流。**

---

## 1. 为什么 Unseal 之后还要试运行

很多发布系统会把“解封”理解成“解除阻断”。这在普通功能里问题不大，但 Agent 护栏的 tombstone 是事故记忆：它记录的是一个候选护栏曾经真实制造过 false allow、false block、越界取证或 runtime divergence。

所以 UnsealReceipt 不能直接升级成 active permission。它只能说明：

1. 这次 candidate 和失败版本已经有可解释差异；
2. 历史 regression seed 在离线回放里通过；
3. 证据边界和作用范围已经被收窄或证明合理；
4. 责任 owner 接受剩余风险。

这些都还不是生产证明。真正要证明的是：**在当前真实流量、当前 runtime、当前工具输出下，它没有再次变坏。**

因此 Unseal 后应该进入一个单独状态：**Scoped Trial**。

---

## 2. 最小数据模型

~~~text
ScopedTrial:
  trialId
  unsealReceiptId
  tombstoneId
  patternId
  candidateVersion
  allowedStages:
    shadow | canary
  scope:
    tenantIds[]
    riskSurfaces[]
    actionClasses[]
    maxDecisionCount
    expiresAt
  budgets:
    regressionSeedFailures: 0
    badDiffs: 0
    evidenceBoundaryExpansions: 0
    unknownOutcomes: 3
    latencyP95Ms
  status:
    running | promoted | resealed | expired | manual_review

TrialObservation:
  trialId
  decisionId
  stage
  activeDecision
  candidateDecision
  diffClass
  outcomeClass
  evidenceBoundaryDelta
  regressionSeedRef
  observedAt

ResealReceipt:
  receiptId
  trialId
  tombstoneId
  candidateVersion
  trigger:
    regression_seed_failed |
    bad_diff_budget_exceeded |
    evidence_boundary_expanded |
    scope_violation |
    owner_approval_expired |
    runtime_divergence
  affectedDecisionIds[]
  newForbiddenPredicateHashes[]
  nextAction:
    reject_candidate |
    reopen_repair |
    extend_replay |
    escalate_owner
~~~

重点：**Tombstone 仍然 active**。ScopedTrial 只是给某个 candidate 在某个有限 scope 里开一条临时通道。

---

## 3. Scoped Trial 的四条硬规则

第一，scope 必须比原事故更窄，或者有明确证明为什么同等范围安全。比如只允许低风险 tenant、只读 shadow、有限 actionClass、最大 decision 数。

第二，trial 必须有到期时间。没有 expiresAt 的 unseal 等于永久豁免，最后会变成绕过 tombstone 的后门。

第三，预算必须偏保守。对 tombstone 版本来说，regressionSeedFailures 和 evidenceBoundaryExpansions 通常应该是 0 容忍。

第四，reseal 必须自动化。发现坏 diff 后靠人工“记得关”不够，发布状态机要直接把 candidate 重新拉回 blocked，并生成 ResealReceipt。

---

## 4. learn-claude-code：创建限域试运行

教学版先写纯函数：从 UnsealReceipt 生成 ScopedTrial。这里故意只允许 shadow / canary，不允许直接 active。

~~~py
from dataclasses import dataclass
from datetime import datetime, timedelta
from typing import Literal


Stage = Literal["shadow", "canary", "active"]
TrialStatus = Literal["running", "promoted", "resealed", "expired", "manual_review"]


@dataclass(frozen=True)
class UnsealReceipt:
    receipt_id: str
    tombstone_id: str
    pattern_id: str
    candidate_version: str
    allowed_stage: Stage
    scope_risk_surfaces: tuple[str, ...]
    scope_action_classes: tuple[str, ...]
    owner_approval_expires_at: datetime


@dataclass(frozen=True)
class TrialBudgets:
    max_decisions: int
    max_unknown_outcomes: int
    max_latency_p95_ms: int
    max_bad_diffs: int = 0
    max_regression_seed_failures: int = 0
    max_evidence_boundary_expansions: int = 0


@dataclass(frozen=True)
class ScopedTrial:
    trial_id: str
    unseal_receipt_id: str
    tombstone_id: str
    pattern_id: str
    candidate_version: str
    allowed_stages: tuple[Stage, ...]
    risk_surfaces: tuple[str, ...]
    action_classes: tuple[str, ...]
    expires_at: datetime
    budgets: TrialBudgets
    status: TrialStatus


def create_scoped_trial(receipt: UnsealReceipt, now: datetime) -> ScopedTrial:
    if receipt.allowed_stage == "active":
        allowed_stages: tuple[Stage, ...] = ("shadow", "canary")
    elif receipt.allowed_stage == "canary":
        allowed_stages = ("shadow", "canary")
    else:
        allowed_stages = ("shadow",)

    expires_at = min(now + timedelta(hours=24), receipt.owner_approval_expires_at)

    return ScopedTrial(
        trial_id=f"trial:{receipt.receipt_id}",
        unseal_receipt_id=receipt.receipt_id,
        tombstone_id=receipt.tombstone_id,
        pattern_id=receipt.pattern_id,
        candidate_version=receipt.candidate_version,
        allowed_stages=allowed_stages,
        risk_surfaces=receipt.scope_risk_surfaces,
        action_classes=receipt.scope_action_classes,
        expires_at=expires_at,
        budgets=TrialBudgets(
            max_decisions=200 if "high_risk_tool" not in receipt.scope_action_classes else 25,
            max_unknown_outcomes=3,
            max_latency_p95_ms=1500,
        ),
        status="running",
    )
~~~

这里的保守点在于：即使 UnsealGate 决定 allow_active，也要先落到 shadow/canary trial。历史墓碑命中过的版本不应该跳过生产证明。

---

## 5. learn-claude-code：复封触发器

接着写一个评估函数：汇总 trial observations，决定继续、晋级、复封或人工审查。

~~~py
from dataclasses import dataclass
from typing import Literal


DiffClass = Literal[
    "same",
    "candidate_safer",
    "candidate_more_strict",
    "candidate_more_lenient",
    "bad_diff",
    "reason_changed",
]
OutcomeClass = Literal["good", "bad", "unknown"]
TrialDecision = Literal["continue_trial", "promote_candidate", "reseal_candidate", "manual_review"]


@dataclass(frozen=True)
class TrialObservation:
    decision_id: str
    diff_class: DiffClass
    outcome_class: OutcomeClass
    regression_seed_failed: bool
    evidence_boundary_expanded: bool
    scope_violated: bool
    latency_ms: int


@dataclass(frozen=True)
class TrialSummary:
    decision_count: int
    bad_diffs: int
    unknown_outcomes: int
    regression_seed_failures: int
    evidence_boundary_expansions: int
    scope_violations: int
    latency_p95_ms: int


@dataclass(frozen=True)
class TrialEvaluation:
    decision: TrialDecision
    trigger: str | None
    affected_decision_ids: tuple[str, ...]


def summarize_trial(observations: tuple[TrialObservation, ...]) -> TrialSummary:
    latencies = sorted(ob.latency_ms for ob in observations)
    p95_index = max(0, int(len(latencies) * 0.95) - 1)

    return TrialSummary(
        decision_count=len(observations),
        bad_diffs=sum(1 for ob in observations if ob.diff_class == "bad_diff"),
        unknown_outcomes=sum(1 for ob in observations if ob.outcome_class == "unknown"),
        regression_seed_failures=sum(1 for ob in observations if ob.regression_seed_failed),
        evidence_boundary_expansions=sum(1 for ob in observations if ob.evidence_boundary_expanded),
        scope_violations=sum(1 for ob in observations if ob.scope_violated),
        latency_p95_ms=latencies[p95_index] if latencies else 0,
    )


def evaluate_trial(
    trial: ScopedTrial,
    observations: tuple[TrialObservation, ...],
    now: datetime,
) -> TrialEvaluation:
    summary = summarize_trial(observations)

    if now >= trial.expires_at:
        return TrialEvaluation("manual_review", "trial_expired", ())

    reseal_checks = [
        ("regression_seed_failed", summary.regression_seed_failures > trial.budgets.max_regression_seed_failures),
        ("bad_diff_budget_exceeded", summary.bad_diffs > trial.budgets.max_bad_diffs),
        ("evidence_boundary_expanded", summary.evidence_boundary_expansions > trial.budgets.max_evidence_boundary_expansions),
        ("scope_violation", summary.scope_violations > 0),
    ]

    for trigger, failed in reseal_checks:
        if failed:
            affected = tuple(
                ob.decision_id
                for ob in observations
                if ob.regression_seed_failed
                or ob.diff_class == "bad_diff"
                or ob.evidence_boundary_expanded
                or ob.scope_violated
            )
            return TrialEvaluation("reseal_candidate", trigger, affected)

    if summary.unknown_outcomes > trial.budgets.max_unknown_outcomes:
        return TrialEvaluation("manual_review", "unknown_outcome_budget_exceeded", ())

    if summary.latency_p95_ms > trial.budgets.max_latency_p95_ms:
        return TrialEvaluation("manual_review", "latency_budget_exceeded", ())

    if summary.decision_count >= trial.budgets.max_decisions:
        return TrialEvaluation("promote_candidate", None, ())

    return TrialEvaluation("continue_trial", None, ())
~~~

这段逻辑的核心不是复杂，而是可审计：任何复封都能说清楚是哪条预算触发、影响了哪些 decisionId、下一步是 reject 还是 reopen repair。

---

## 6. pi-mono：把 Trial 放进事件流 Worker

生产版不要让 trial 只存在内存里。它应该由事件驱动：

~~~ts
type TrialEvent =
  | { type: "UnsealReceiptWritten"; receiptId: string; candidateVersion: string }
  | { type: "DecisionObserved"; trialId: string; decisionId: string }
  | { type: "TrialEvaluationCompleted"; trialId: string; decision: "continue_trial" | "promote_candidate" | "reseal_candidate" | "manual_review" }
  | { type: "ResealReceiptWritten"; trialId: string; trigger: string };

class ScopedTrialWorker {
  constructor(
    private readonly trialStore: TrialStore,
    private readonly decisionStore: DecisionStore,
    private readonly patternRegistry: PatternRegistry,
    private readonly eventBus: EventBus,
  ) {}

  async onUnseal(receipt: UnsealReceipt) {
    const trial = createScopedTrial(receipt, new Date());
    await this.trialStore.save(trial);

    await this.patternRegistry.allowOnlyWithinScope({
      patternId: trial.patternId,
      candidateVersion: trial.candidateVersion,
      trialId: trial.trialId,
      stages: trial.allowedStages,
      riskSurfaces: trial.riskSurfaces,
      actionClasses: trial.actionClasses,
      expiresAt: trial.expiresAt,
    });
  }

  async evaluate(trialId: string) {
    const trial = await this.trialStore.get(trialId);
    const observations = await this.decisionStore.listTrialObservations(trialId);
    const result = evaluateTrial(trial, observations, new Date());

    if (result.decision === "reseal_candidate") {
      await this.patternRegistry.blockCandidate({
        tombstoneId: trial.tombstoneId,
        candidateVersion: trial.candidateVersion,
        trigger: result.trigger!,
      });

      await this.eventBus.publish({
        type: "ResealReceiptWritten",
        trialId,
        trigger: result.trigger!,
        affectedDecisionIds: result.affectedDecisionIds,
      });
      return;
    }

    await this.eventBus.publish({
      type: "TrialEvaluationCompleted",
      trialId,
      decision: result.decision,
    });
  }
}
~~~

注意这个 worker 做了两件事：先把 registry 限域放行，再持续评估是否要复封。这样 UnsealReceipt 不会变成永久白名单。

---

## 7. OpenClaw：Cron / Telegram / Git 的实战映射

拿这门课程 cron 举例，最小落地是：

1. **UnsealReceipt**：如果某次课程选题命中“已讲内容”相似主题，但确实有新角度，必须写明新 angle、diff proof 和 scope；
2. **ScopedTrial**：只允许本次 cron 生成一篇 lesson、一次 Telegram 消息、一次 git commit，不允许扩大到别的仓库或别的群；
3. **TrialObservation**：发布前检查 README、TOOLS、lesson 文件、Telegram 内容是否一致；
4. **ResealTrigger**：如果发现 lesson 主题重复、README 没更新、TOOLS 没写入、git push 失败或消息没发出，就把本次候选内容标记为 blocked，不能假装完成；
5. **ResealReceipt**：记录失败原因、受影响文件和下一步动作，下一次 cron 必须先处理这个阻断。

这就是为什么长期自主 Agent 不能只靠“我记得”。每一次放行、试运行、复封都要有文件和事件能证明。

---

## 8. 常见坑

**坑 1：Unseal 后直接 active。**  
这等于让一个事故版本绕过了真实流量证明。至少要 shadow；高风险动作必须 canary。

**坑 2：Trial 没有 scope。**  
没有 tenant / riskSurface / actionClass / maxDecisionCount / expiresAt 的 trial，本质是永久豁免。

**坑 3：只统计 bad diff，不看 evidence boundary。**  
护栏 candidate 即使决策结果正确，只要读取了不该读的证据，也可能是 privacy regression。

**坑 4：Reseal 只改状态不写 receipt。**  
没有 ResealReceipt，后续 repair 不知道为什么被封，也无法生成新的 regression seed。

---

## 9. 总结

Tombstone 命中后的 Unseal 不是“赦免”，只是“受控试运行许可”。

成熟的 Agent 护栏发布链应该是：

**TombstoneHit -> ReleaseBlocker -> UnsealReceipt -> ScopedTrial -> TrialObservation -> Promote 或 ResealReceipt**

这样系统既不会永远封死合理修复，也不会让历史事故版本靠一次人工批准重新混进生产。

