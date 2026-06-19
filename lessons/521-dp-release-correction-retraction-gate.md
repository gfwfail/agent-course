# 521 - Agent 差分隐私发布更正与撤回闸门

> 上一讲我们把一次性审计许可消费、隔离复现、敏感输出扫描和 seed 销毁串成关闭链路。但如果复现发现 mismatch，或者审计输出泄露了敏感材料，系统不能只把 release 标成 `frozen` 就停在那里。成熟 Agent 还要决定：更正、撤回、保留冻结、重发通知，还是打开事故案件。

上一讲讲的是 **DP Audit Permit Consumption & Reproducibility Closeout Gate**。今天继续往下走一步：**DP Release Correction & Retraction Gate**。

一句话：**DP release 出问题后，不能悄悄改数字，也不能只发一句“请忽略”；要用证据决定更正、撤回和下游通知。**

---

## 1. 为什么 frozen release 不是终态？

很多数据系统遇到统计发布错误时会做两件事：

```text
status = frozen
note = "audit mismatch, pending review"
```

这只是暂停，不是关闭。它没有回答：

- 已经读过这个 noisy value 的 dashboard、报告、缓存和下游任务怎么办；
- mismatch 是 seed 复现问题、raw aggregate 问题、贡献边界问题，还是发布管道写错；
- 能不能重新发布一个 corrected noisy value；
- 如果更正需要消耗新的 epsilon，预算从哪里来；
- 撤回通知是否送达所有 consumer；
- 原 release 的引用是否还能继续展示。

所以 DP 更正不是“改一行数值”，而是一条收据链：

```text
FrozenDpRelease
        ↓
CorrectionInvestigationReport
        ↓
CorrectionDecision
        ↓
CorrectionOrRetractionExecution
        ↓
ConsumerNotificationReceipt
        ↓
CorrectionCloseoutReceipt
```

关键是：原始错误 release 不能被覆盖，要保留不可变历史；新的 corrected release 必须有自己的预算、噪声、收据和通知闭环。

---

## 2. 闸门要检查什么？

DP Release Correction Gate 要回答六个问题：

- freeze 原因是否明确：`reproduction_mismatch`、`sensitive_output`、`budget_error`、`contribution_error`；
- 影响范围是否完整：dashboard、cache、report、API consumer、derived artifact；
- 更正是否需要新 epsilon，预算是否可用；
- 原 release 是否必须撤回，还是可以发布 superseding release；
- consumer 是否收到撤回或更正通知，并带上 replacement release id；
- closeout 是否禁止覆盖原 release，只能追加 correction lineage。

工程上可以把它记成：

```text
investigate -> decide correction/retraction -> notify downstream -> close lineage
```

---

## 3. learn-claude-code：更正决策纯函数

教学版先写成纯函数，输入 frozen release、调查报告、预算和下游影响，输出下一步动作。

```python
from dataclasses import dataclass
from typing import Literal

Decision = Literal[
    "publish_superseding_release",
    "retract_release_only",
    "keep_frozen_waiting_investigation",
    "deny_correction_budget_exhausted",
    "purge_sensitive_outputs",
    "open_dp_release_incident",
]

FreezeReason = Literal[
    "reproduction_mismatch",
    "sensitive_output",
    "budget_error",
    "contribution_error",
    "unknown",
]

@dataclass(frozen=True)
class FrozenDpRelease:
    release_id: str
    status: str
    freeze_reason: FreezeReason
    published_to_consumers: bool

@dataclass(frozen=True)
class InvestigationReport:
    root_cause_known: bool
    raw_aggregate_valid: bool
    contribution_bounding_valid: bool
    original_noise_invalid: bool
    sensitive_material_leaked: bool

@dataclass(frozen=True)
class PrivacyBudget:
    epsilon_remaining: float
    correction_epsilon_cost: float

@dataclass(frozen=True)
class DownstreamImpact:
    consumer_count: int
    all_consumers_addressable: bool
    derived_artifacts_exist: bool

@dataclass(frozen=True)
class CorrectionDecision:
    decision: Decision
    reason: str
    require_consumer_notice: bool = False
    replacement_allowed: bool = False

def decide_dp_release_correction(
    release: FrozenDpRelease,
    report: InvestigationReport,
    budget: PrivacyBudget,
    impact: DownstreamImpact,
) -> CorrectionDecision:
    if release.status != "frozen":
        return CorrectionDecision(
            "open_dp_release_incident",
            "correction gate only handles frozen releases",
        )

    if report.sensitive_material_leaked:
        return CorrectionDecision(
            "purge_sensitive_outputs",
            "audit or release output contains seed, noise, raw value, or equivalent material",
            require_consumer_notice=release.published_to_consumers,
        )

    if not report.root_cause_known or release.freeze_reason == "unknown":
        return CorrectionDecision(
            "keep_frozen_waiting_investigation",
            "release remains frozen until the mismatch root cause is known",
        )

    if not report.raw_aggregate_valid or not report.contribution_bounding_valid:
        return CorrectionDecision(
            "open_dp_release_incident",
            "underlying aggregate or contribution boundary is invalid; correction needs incident handling",
            require_consumer_notice=release.published_to_consumers,
        )

    if report.original_noise_invalid:
        if budget.epsilon_remaining < budget.correction_epsilon_cost:
            return CorrectionDecision(
                "deny_correction_budget_exhausted",
                "a superseding noisy release would overspend the privacy budget",
                require_consumer_notice=release.published_to_consumers,
            )
        return CorrectionDecision(
            "publish_superseding_release",
            "raw aggregate is valid; publish a new noisy release with fresh budget and lineage",
            require_consumer_notice=impact.consumer_count > 0,
            replacement_allowed=True,
        )

    if impact.derived_artifacts_exist and not impact.all_consumers_addressable:
        return CorrectionDecision(
            "open_dp_release_incident",
            "release reached derived artifacts that cannot all be notified",
            require_consumer_notice=True,
        )

    return CorrectionDecision(
        "retract_release_only",
        "published value should be withdrawn without replacement",
        require_consumer_notice=release.published_to_consumers,
    )
```

最小测试：

```python
def test_noise_mismatch_can_publish_superseding_release():
    decision = decide_dp_release_correction(
        FrozenDpRelease("rel1", "frozen", "reproduction_mismatch", True),
        InvestigationReport(True, True, True, True, False),
        PrivacyBudget(epsilon_remaining=0.4, correction_epsilon_cost=0.1),
        DownstreamImpact(3, True, False),
    )
    assert decision.decision == "publish_superseding_release"
    assert decision.replacement_allowed is True
    assert decision.require_consumer_notice is True

def test_budget_exhaustion_blocks_correction():
    decision = decide_dp_release_correction(
        FrozenDpRelease("rel2", "frozen", "reproduction_mismatch", True),
        InvestigationReport(True, True, True, True, False),
        PrivacyBudget(epsilon_remaining=0.01, correction_epsilon_cost=0.1),
        DownstreamImpact(2, True, False),
    )
    assert decision.decision == "deny_correction_budget_exhausted"

def test_sensitive_leak_forces_purge():
    decision = decide_dp_release_correction(
        FrozenDpRelease("rel3", "frozen", "sensitive_output", True),
        InvestigationReport(True, True, True, False, True),
        PrivacyBudget(epsilon_remaining=1.0, correction_epsilon_cost=0.1),
        DownstreamImpact(5, True, True),
    )
    assert decision.decision == "purge_sensitive_outputs"
```

这个函数的重点是：mismatch 不一定都能“重新算一次”。只有 raw aggregate 和 contribution boundary 都有效，且预算允许，才可以发布 superseding release。

---

## 4. pi-mono：DpReleaseCorrectionWorker

生产版要把更正做成追加式 lineage，绝不能原地覆盖原 release。

```ts
type FrozenRelease = {
  releaseId: string;
  datasetId: string;
  status: "frozen";
  freezeReason: string;
  epsilonSpent: number;
  publishedAt: string;
};

type CorrectionPlan = {
  originalReleaseId: string;
  action: "supersede" | "retract" | "purge_sensitive" | "incident";
  replacementReleaseId?: string;
  noticeRequired: boolean;
  reason: string;
};

class DpReleaseCorrectionWorker {
  constructor(
    private readonly releases: DpReleaseStore,
    private readonly investigations: CorrectionInvestigationStore,
    private readonly budgets: PrivacyBudgetLedger,
    private readonly notifier: ConsumerNotificationService,
    private readonly closeouts: CorrectionCloseoutStore,
    private readonly outbox: EventOutbox,
  ) {}

  async handleFrozenRelease(input: {
    releaseId: string;
    investigationId: string;
  }): Promise<CorrectionCloseout> {
    const release = await this.releases.getFrozen(input.releaseId);
    const report = await this.investigations.get(input.investigationId);
    const impact = await this.releases.getDownstreamImpact(input.releaseId);

    const plan = await this.buildPlan(release, report, impact);

    if (plan.action === "supersede") {
      const budgetReceipt = await this.budgets.reserveOnce({
        datasetId: release.datasetId,
        purpose: "dp_release_correction",
        parentReleaseId: release.releaseId,
        epsilon: report.correctionEpsilonCost,
      });

      const replacement = await this.releases.insertSupersedingRelease({
        parentReleaseId: release.releaseId,
        datasetId: release.datasetId,
        noisyValue: report.correctedNoisyValue,
        epsilonReceiptId: budgetReceipt.receiptId,
        lineageReason: plan.reason,
      });

      plan.replacementReleaseId = replacement.releaseId;
      await this.releases.markSuperseded(release.releaseId, replacement.releaseId);
    }

    if (plan.action === "retract") {
      await this.releases.markRetracted(release.releaseId, plan.reason);
    }

    if (plan.action === "purge_sensitive") {
      await this.releases.purgeSensitiveProjections(release.releaseId);
      await this.releases.markRetracted(release.releaseId, "sensitive_material_purged");
    }

    const notice = plan.noticeRequired
      ? await this.notifier.notifyAllConsumers({
          originalReleaseId: release.releaseId,
          replacementReleaseId: plan.replacementReleaseId,
          action: plan.action,
          reason: plan.reason,
        })
      : { delivered: 0, failed: 0, receiptId: "notice_not_required" };

    const closeout = await this.closeouts.insertOnce({
      originalReleaseId: release.releaseId,
      action: plan.action,
      replacementReleaseId: plan.replacementReleaseId,
      noticeReceiptId: notice.receiptId,
      consumerFailures: notice.failed,
      closedAt: new Date().toISOString(),
    });

    await this.outbox.publish("dp.release_correction_closed", closeout);
    return closeout;
  }

  private async buildPlan(
    release: FrozenRelease,
    report: CorrectionInvestigation,
    impact: DownstreamImpact,
  ): Promise<CorrectionPlan> {
    if (report.sensitiveMaterialLeaked) {
      return {
        originalReleaseId: release.releaseId,
        action: "purge_sensitive",
        noticeRequired: impact.consumerCount > 0,
        reason: "sensitive_material_leaked",
      };
    }

    if (!report.rawAggregateValid || !report.contributionBoundingValid) {
      return {
        originalReleaseId: release.releaseId,
        action: "incident",
        noticeRequired: impact.consumerCount > 0,
        reason: "invalid_aggregate_or_contribution_boundary",
      };
    }

    if (report.originalNoiseInvalid) {
      return {
        originalReleaseId: release.releaseId,
        action: "supersede",
        noticeRequired: impact.consumerCount > 0,
        reason: "recomputed_noisy_release_required",
      };
    }

    return {
      originalReleaseId: release.releaseId,
      action: "retract",
      noticeRequired: impact.consumerCount > 0,
      reason: "release_withdrawn_without_replacement",
    };
  }
}
```

几个生产细节：

- `insertSupersedingRelease` 要写新 release id，原 release 只改状态，不改值；
- `reserveOnce` 要用 parent release 做幂等键，避免更正重试重复消耗 epsilon；
- consumer 通知要有 outbox 和重试队列，否则 correction closeout 不能算完整；
- `incident` 不是失败，而是进入更高等级治理：冻结下游、保留证据、人工或自动修复。

---

## 5. OpenClaw 实战类比

OpenClaw 课程 cron 如果 Git push 成功但 Telegram 发送失败，不能直接改历史说“没发过”。正确做法类似：

```text
原始 commit 保留
        ↓
发现 delivery mismatch
        ↓
补发 Telegram 或发更正说明
        ↓
记录 message id
        ↓
最终 closeout 对齐 lesson/README/TOOLS/Git/message
```

DP release 更正也是一样：

- 原 release 是不可变历史；
- corrected release 是新事实；
- retract notice 是下游收敛动作；
- closeout receipt 证明所有 consumer 都知道旧值不能再用。

不要把“覆盖错误”当修复。Agent 系统里，现实世界已经看见过的输出，都必须用追加式更正收敛。

---

## 6. 设计 Checklist

落地这个闸门时，可以直接检查：

- frozen release 是否记录 `freezeReason`、`frozenAt`、`auditReceiptId`；
- investigation 是否区分 noise error、aggregate error、contribution bounding error；
- correction 是否消耗新的 epsilon，并写预算收据；
- superseding release 是否追加 lineage，而不是覆盖原值；
- retraction 是否通知所有 dashboard/API/report/cache consumer；
- derived artifacts 是否被标记 stale 或重新生成；
- sensitive output 是否先 purge，再允许 closeout；
- closeout 是否记录 original/replacement release id、notice receipt 和失败 consumer 数。

---

## 7. 今日要点

DP 发布出错后，最危险的动作是“悄悄改对”。这会让审计、预算和下游解释全部失真。

记住这个公式：

```text
DP Correction = immutable original + investigation + budgeted replacement/retraction + consumer notice + closeout
```

成熟 Agent 不只会发现 DP 发布不可信，还会带着预算、血缘和通知，把错误发布收敛成可解释、可审计、不会继续传播的终态。
