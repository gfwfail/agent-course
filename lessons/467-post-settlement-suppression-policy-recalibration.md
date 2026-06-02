# 467. Agent 信号债务结算后的抑制策略再校准（Post-Settlement Suppression Policy Recalibration）

上一课讲了 **Suppression Window Expiry & Signal Debt Settlement**：抑制窗口到期后，要把窗口期内被 suppress 的信号逐条结算成 close_retention、extend_suppression、reopen_replay 或 manual_review。

但结算不是终点。结算结果会反过来告诉你：这条 suppression policy 是否太松、太紧、窗口太长、预算太高，或者 source 权重不对。

所以今天补上 **Post-Settlement Suppression Policy Recalibration**：

~~~text
SignalDebtSettlement
  -> PolicyDriftSample
  -> SuppressionPolicyCalibration
  -> CalibrationProposal
  -> ShadowSuppressionPolicy
  -> PromotionReceipt | RejectReceipt
~~~

一句话：**成熟 Agent 不只会结清被压住的信号，还会根据结算结果校准下一轮 suppression policy，避免同一种误压、过压、漏压反复发生。**

---

## 1. 为什么 settlement 后还要校准 policy

如果 SignalDebtSettlement 连续出现这些结果：

~~~text
manual_review: duplicate_volume_exceeded_budget
manual_review: user_visible_signal_was_suppressed
reopen_replay: critical_signal_suppressed
close_retention: no_suppressed_signals
~~~

它们代表的含义完全不同：

- duplicate volume 经常超预算：maxDuplicateCount 可能太低，也可能 upstream probe 在重复上报；
- user report 被 suppress：source 权重太低，用户可见信号不该被旧 dedupeKey 直接压掉；
- critical 被 suppress：severity gate 有 bug，这类信号必须绕过 suppression；
- 总是 no_suppressed_signals：窗口可能太长或 policy 根本没被命中，保留它会增加审计负担。

生产系统最危险的情况是：

~~~text
结算时知道 policy 不对，但下一次仍用同一条 policy。
~~~

这会让 agent 看起来很有流程，实际上只是把错误制度化。

因此 settlement 后需要一个再校准闭环：

~~~text
把 settlement 变成样本；
按 policyId 汇总 drift；
生成 calibration proposal；
先 shadow，不直接替换 active；
shadow 证明更好后再 promotion。
~~~

---

## 2. 最小数据模型

结算样本：

~~~json
{
  "kind": "PolicyDriftSample",
  "sampleId": "sample_467_001",
  "policyId": "sup_policy_cancelled_run_v3",
  "settlementId": "settlement_466",
  "decision": "manual_review",
  "reason": "user_visible_signal_was_suppressed",
  "entryCount": 3,
  "maxSeverity": "warning",
  "sources": ["telegram_probe", "user_report"],
  "capturedAt": "2026-06-02T21:30:00+11:00"
}
~~~

校准提案：

~~~json
{
  "kind": "CalibrationProposal",
  "proposalId": "calib_467_001",
  "basePolicyId": "sup_policy_cancelled_run_v3",
  "candidatePolicyId": "sup_policy_cancelled_run_v4_shadow",
  "changes": [
    {
      "field": "sourceWeights.user_report",
      "from": 1,
      "to": 10,
      "reason": "user_visible_signal_must_not_be_suppressed_by_old_probe_key"
    },
    {
      "field": "criticalBypass",
      "from": false,
      "to": true,
      "reason": "critical signals must open replay immediately"
    }
  ],
  "shadowBudget": {
    "sampleCount": 200,
    "maxFalseSuppressions": 0,
    "maxFalseReopens": 3
  }
}
~~~

推广收据：

~~~json
{
  "kind": "SuppressionPolicyPromotionReceipt",
  "proposalId": "calib_467_001",
  "fromPolicyId": "sup_policy_cancelled_run_v3",
  "toPolicyId": "sup_policy_cancelled_run_v4",
  "shadowResult": {
    "sampleCount": 213,
    "falseSuppressions": 0,
    "falseReopens": 1
  },
  "promotedAt": "2026-06-02T22:00:00+11:00"
}
~~~

关键点：**policy 改动也要像代码发布一样有 shadow、预算、收据和回滚点。**

---

## 3. learn-claude-code：教学版 policy 校准纯函数

learn-claude-code 里可以先把校准做成纯函数：输入 settlement 样本，输出 proposal。它不需要数据库，也不需要真实外部工具。

~~~python
# learn_claude_code/runtime/suppression_policy_calibration.py
from dataclasses import dataclass, replace
from typing import Literal

Decision = Literal["close_retention", "extend_suppression", "reopen_replay", "manual_review"]


@dataclass(frozen=True)
class SuppressionPolicy:
    policy_id: str
    max_duplicate_count: int
    critical_bypass: bool
    user_report_bypass: bool
    window_minutes: int


@dataclass(frozen=True)
class PolicyDriftSample:
    policy_id: str
    decision: Decision
    reason: str
    entry_count: int


@dataclass(frozen=True)
class CalibrationProposal:
    base_policy_id: str
    candidate_policy: SuppressionPolicy
    reason: str
    shadow_sample_budget: int
    max_false_suppressions: int


def calibrate_suppression_policy(
    policy: SuppressionPolicy,
    samples: list[PolicyDriftSample],
) -> CalibrationProposal | None:
    relevant = [sample for sample in samples if sample.policy_id == policy.policy_id]
    if len(relevant) < 5:
        return None

    reasons = [sample.reason for sample in relevant]

    if "critical_signal_suppressed" in reasons:
        return CalibrationProposal(
            policy.policy_id,
            replace(policy, policy_id=f"{policy.policy_id}:candidate", critical_bypass=True),
            "critical_signals_must_bypass_suppression",
            shadow_sample_budget=100,
            max_false_suppressions=0,
        )

    if "user_visible_signal_was_suppressed" in reasons:
        return CalibrationProposal(
            policy.policy_id,
            replace(policy, policy_id=f"{policy.policy_id}:candidate", user_report_bypass=True),
            "user_visible_signals_need_manual_review_or_reopen",
            shadow_sample_budget=150,
            max_false_suppressions=0,
        )

    duplicate_budget_hits = sum(
        1 for sample in relevant if sample.reason == "duplicate_volume_exceeded_budget"
    )
    if duplicate_budget_hits >= 3:
        return CalibrationProposal(
            policy.policy_id,
            replace(
                policy,
                policy_id=f"{policy.policy_id}:candidate",
                max_duplicate_count=min(policy.max_duplicate_count * 2, 100),
            ),
            "duplicate_budget_too_tight_or_probe_too_noisy",
            shadow_sample_budget=200,
            max_false_suppressions=1,
        )

    empty_closes = sum(1 for sample in relevant if sample.reason == "no_suppressed_signals")
    if empty_closes >= len(relevant) * 0.8 and policy.window_minutes > 30:
        return CalibrationProposal(
            policy.policy_id,
            replace(
                policy,
                policy_id=f"{policy.policy_id}:candidate",
                window_minutes=max(policy.window_minutes // 2, 30),
            ),
            "suppression_window_too_long_for_observed_signal_rate",
            shadow_sample_budget=200,
            max_false_suppressions=1,
        )

    return None
~~~

最小测试：

~~~python
def test_enables_critical_bypass_after_critical_suppression():
    policy = SuppressionPolicy(
        "sup_policy_v3",
        max_duplicate_count=10,
        critical_bypass=False,
        user_report_bypass=False,
        window_minutes=120,
    )

    samples = [
        PolicyDriftSample("sup_policy_v3", "manual_review", "duplicate_volume_exceeded_budget", 11),
        PolicyDriftSample("sup_policy_v3", "close_retention", "all_suppressed_signals_match_stable_close_scope", 3),
        PolicyDriftSample("sup_policy_v3", "reopen_replay", "critical_signal_suppressed", 1),
        PolicyDriftSample("sup_policy_v3", "close_retention", "all_suppressed_signals_match_stable_close_scope", 2),
        PolicyDriftSample("sup_policy_v3", "manual_review", "duplicate_volume_exceeded_budget", 13),
    ]

    proposal = calibrate_suppression_policy(policy, samples)

    assert proposal is not None
    assert proposal.candidate_policy.critical_bypass is True
    assert proposal.max_false_suppressions == 0
~~~

这里的重点不是“自动调参很聪明”，而是把调参变成可测试、可解释、可回滚的纯函数。

---

## 4. pi-mono：把校准当成 policy 发布流

pi-mono 的生产实现不应该让 settlement worker 直接改 active policy。推荐拆成三层：

~~~text
SettlementWorker:
  写 SignalDebtSettlement 和 PolicyDriftSample

CalibrationWorker:
  汇总样本，生成 CalibrationProposal

PolicyReleaseWorker:
  shadow candidate，达标后 promotion
~~~

TypeScript 结构可以这样写：

~~~ts
type SuppressionPolicy = {
  id: string;
  version: number;
  maxDuplicateCount: number;
  criticalBypass: boolean;
  userReportBypass: boolean;
  windowMs: number;
};

type PolicyDriftSample = {
  policyId: string;
  settlementId: string;
  decision: "close_retention" | "extend_suppression" | "reopen_replay" | "manual_review";
  reason: string;
  entryCount: number;
  createdAt: string;
};

type CalibrationProposal = {
  id: string;
  basePolicyId: string;
  candidatePolicy: SuppressionPolicy;
  shadowBudget: {
    sampleCount: number;
    maxFalseSuppressions: number;
    maxFalseReopens: number;
  };
};

async function runSuppressionPolicyCalibration(policyId: string) {
  const lease = await db.policyCalibrationLease.claim(policyId, {
    owner: "suppression-policy-calibrator",
    ttlMs: 90_000,
  });

  if (!lease.acquired) return;

  const [policy, samples] = await Promise.all([
    db.suppressionPolicy.get(policyId),
    db.policyDriftSample.listRecent(policyId, { limit: 500 }),
  ]);

  const proposal = decideCalibrationProposal(policy, samples);
  if (!proposal) return;

  await db.transaction(async (tx) => {
    const existing = await tx.calibrationProposal.findOpenByBasePolicy(policyId);
    if (existing) return;

    await tx.calibrationProposal.insert({
      ...proposal,
      status: "shadowing",
      fenceToken: lease.fenceToken,
      createdAt: new Date().toISOString(),
    });

    await tx.shadowSuppressionPolicy.insert({
      policyId: proposal.candidatePolicy.id,
      basePolicyId: policyId,
      config: proposal.candidatePolicy,
      shadowBudget: proposal.shadowBudget,
    });
  });
}
~~~

shadow 评估时要同时跑 active 和 candidate，但只让 active 生效：

~~~ts
async function evaluateSuppressionInShadow(signal: Signal) {
  const active = await db.suppressionPolicy.getActive(signal.scope);
  const candidate = await db.shadowSuppressionPolicy.findByBase(active.id);

  const activeDecision = decideSuppression(active.config, signal);

  if (candidate) {
    const shadowDecision = decideSuppression(candidate.config, signal);
    await db.shadowSuppressionComparison.insert({
      basePolicyId: active.id,
      candidatePolicyId: candidate.policyId,
      signalId: signal.id,
      activeDecision,
      shadowDecision,
      payloadHash: signal.payloadHash,
    });
  }

  return activeDecision;
}
~~~

推广时必须有门槛：

~~~ts
async function maybePromoteShadowPolicy(proposalId: string) {
  const proposal = await db.calibrationProposal.get(proposalId);
  const report = await db.shadowSuppressionComparison.report(proposal.candidatePolicy.id);

  if (report.sampleCount < proposal.shadowBudget.sampleCount) return;

  if (
    report.falseSuppressions > proposal.shadowBudget.maxFalseSuppressions ||
    report.falseReopens > proposal.shadowBudget.maxFalseReopens
  ) {
    await db.calibrationProposal.reject(proposalId, {
      reason: "shadow_budget_failed",
      report,
    });
    return;
  }

  await db.transaction(async (tx) => {
    await tx.suppressionPolicy.promote({
      fromPolicyId: proposal.basePolicyId,
      toPolicy: proposal.candidatePolicy,
    });

    await tx.policyPromotionReceipt.insert({
      proposalId,
      fromPolicyId: proposal.basePolicyId,
      toPolicyId: proposal.candidatePolicy.id,
      report,
    });
  });
}
~~~

几个硬规则：

- settlement worker 只产样本，不改 policy；
- candidate 必须 shadow，不能直接上线；
- false suppression 的预算通常要比 false reopen 严格；
- promotion receipt 要保留 base policy hash，方便回滚；
- reject 也要写 receipt，否则后续会重复生成同一类 proposal。

---

## 5. OpenClaw 实战：课程 cron 的去重策略也要会学习

这套课程 cron 本身就是 suppression policy 的实战场景。每次发布都要对齐：

~~~text
Telegram messageId
lesson 文件
README 目录
TOOLS 已讲内容
git commit / remote main
memory closeout
~~~

假设某次网络抖动导致 probe 重复看到旧 messageId：

~~~text
sig_1: telegram_probe messageId=13108 payloadHash=A warning
sig_2: telegram_probe messageId=13108 payloadHash=A warning
sig_3: user_report   messageId=13108 payloadHash=A warning
~~~

active policy 如果只看 dedupeKey，就可能把 sig_3 也 suppress 掉。但 user_report 来源代表用户可见层发现了问题，应该至少 manual_review。

校准后的 policy 可以变成：

~~~json
{
  "policyId": "agent_course_cron_suppression_v4",
  "criticalBypass": true,
  "sourceRules": {
    "telegram_probe": "allow_suppress_if_payload_matches",
    "git_probe": "allow_suppress_if_remote_matches",
    "user_report": "manual_review",
    "owner_message": "manual_review"
  },
  "maxDuplicateCount": 20,
  "windowMinutes": 60
}
~~~

也就是说，OpenClaw 的 cron 不只是“这次避免重复发课”，还要能把历史结算结果转成下一次更好的重复信号处理规则。

这能避免两种事故：

- policy 太松：真的异常被旧 dedupeKey 压住；
- policy 太紧：每次旧 probe 都 reopen，导致同一课被重复补救、重复推送。

---

## 6. 实战清单

做 Post-Settlement Suppression Policy Recalibration 时，至少落这些对象：

~~~text
PolicyDriftSample:
  sampleId
  policyId
  settlementId
  decision
  reason
  entryCount
  maxSeverity
  sources
  capturedAt

CalibrationProposal:
  proposalId
  basePolicyId
  candidatePolicyHash
  changedFields
  reason
  shadowBudget
  status

ShadowSuppressionComparison:
  signalId
  basePolicyId
  candidatePolicyId
  activeDecision
  shadowDecision
  expectedOutcome
  classifiedError

PolicyPromotionReceipt:
  proposalId
  fromPolicyId
  toPolicyId
  shadowReport
  promotedAt
  rollbackPolicyId
~~~

决策原则：

~~~text
critical 被 suppress:
  直接提 criticalBypass=true，shadow 预算 falseSuppressions=0。

user_report 被 suppress:
  提 userReportBypass=true 或 source=user_report -> manual_review。

duplicate_volume_exceeded_budget:
  先判断是 probe 噪声还是预算太低；不要盲目加预算。

no_suppressed_signals 占比过高:
  缩短窗口或退役 policy，减少无意义审计负担。

reopen_replay 过多但最后都 close_retention:
  policy 可能太紧，允许更多 matched probe suppress，但仍要保留 user/critical bypass。
~~~

不要把校准做成“模型觉得应该调”。它应该是 settlement 样本驱动、可测试、可 shadow、可回滚的工程流程。

---

## 7. 一句话总结

**成熟 Agent 的 suppression policy 不是写死的 if，而是会从 SignalDebtSettlement 中学习；每次调整都要生成 CalibrationProposal，先 shadow 验证，再用 PromotionReceipt 上线，避免把误压、过压和漏压变成永久制度。**
