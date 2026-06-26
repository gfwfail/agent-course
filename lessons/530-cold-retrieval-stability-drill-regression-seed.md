# 530. Agent 冷取回稳定关闭后的演练调度与回归种子闸门

> ColdRetrievalStabilityReceipt 证明冷路径修复后已经稳定，但它不应该只是一个归档收据。成熟 Agent 会把这次事故里的失败样本、lookup key、runtime 指纹和 SLA 阈值转成回归种子与定期演练计划，防止同类冷路径问题悄悄回来。

上一讲讲了 **Post-Remediation Observation & SLA Rebaseline Gate**：修复完成后，要用真实读取样本、降级摘要 TTL、runtime 指纹和 SLA 再基线判断能不能关闭。

今天继续往后走：**关闭以后，怎么让这次修复变成未来不会退化的测试资产？**

常见问题是：

- 事故关闭时有稳定收据，但没有把 missing lookup key 变成回归用例；
- 冷索引换了 schema 后，旧审计查询没人再演练；
- SLA rebaseline 通过了，但新的 P95 阈值没有进入监控和 synthetic drill；
- 修复时发现的 bundle hash / tenant scope / lease policy 没有被 pin 到后续检查；
- 下一次归档迁移时，冷路径又坏了，团队只能重新排查。

所以要加一层 **Cold Retrieval Stability Drill Scheduling & Regression Seed Gate**：稳定关闭不是把证据放进冷库就忘，而是把事故样本转成可复现、可调度、可审计的回归资产。

---

## 1. 核心链路

```text
ColdRetrievalStabilityReceipt
        ↓
RegressionSeedInventory
        ↓
DrillSchedulePlan
        ↓
SeedReplayReadinessReview
        ↓
DrillActivationReceipt
```

每一环回答一个问题：

- stability receipt：冷路径是否已经通过观察窗口；
- regression seed inventory：哪些真实失败样本必须保留下来做回归；
- drill schedule plan：多久演练一次，覆盖哪些租户、bundle、索引和 lease policy；
- seed replay readiness：回归种子能否安全重放，是否需要脱敏或范围收窄；
- drill activation receipt：演练是否已经被调度、监控、告警和审计系统接管。

一句话：**稳定收据是事故结束，回归种子是未来防线；不把事故样本产品化，Agent 只是在重复交学费。**

---

## 2. learn-claude-code：演练调度判定器

教学版先写纯函数。它不负责跑 cron，只负责决定稳定关闭后下一步该做什么。

```python
from dataclasses import dataclass
from typing import Literal

Decision = Literal[
    "activate_drill_schedule",
    "add_regression_seeds",
    "narrow_seed_scope",
    "extend_stability_observation",
    "reopen_remediation",
    "manual_review",
]

@dataclass
class ColdRetrievalStabilityReceipt:
    cold_path_id: str
    stable: bool
    observed_samples: int
    p95_ms: int
    error_rate: float
    runtime_fingerprint: str
    hot_absence_verified: bool
    sla_rebaseline_approved: bool

@dataclass
class RegressionSeedInventory:
    missing_lookup_key_samples: int
    bundle_mismatch_samples: int
    lease_denial_samples: int
    redacted_seed_ratio: float
    raw_sensitive_seed_count: int
    has_runtime_fingerprint_seed: bool

@dataclass
class DrillSchedulePlan:
    interval_hours: int
    covers_latest_runtime: bool
    covers_oldest_supported_bundle: bool
    covers_rebaseline_sla: bool
    owner_oncall_route: str | None
    alert_enabled: bool

def decide_cold_retrieval_drill_gate(
    stability: ColdRetrievalStabilityReceipt,
    seeds: RegressionSeedInventory,
    plan: DrillSchedulePlan,
) -> Decision:
    if not stability.stable:
        return "reopen_remediation"

    if not stability.hot_absence_verified:
        return "manual_review"

    if stability.observed_samples < 100 or stability.error_rate > 0.01:
        return "extend_stability_observation"

    if seeds.raw_sensitive_seed_count > 0:
        return "narrow_seed_scope"

    useful_seed_count = (
        seeds.missing_lookup_key_samples
        + seeds.bundle_mismatch_samples
        + seeds.lease_denial_samples
    )
    if useful_seed_count == 0 or not seeds.has_runtime_fingerprint_seed:
        return "add_regression_seeds"

    if not plan.covers_latest_runtime:
        return "add_regression_seeds"

    if not plan.covers_oldest_supported_bundle:
        return "add_regression_seeds"

    if stability.sla_rebaseline_approved and not plan.covers_rebaseline_sla:
        return "add_regression_seeds"

    if plan.interval_hours <= 0 or plan.interval_hours > 168:
        return "manual_review"

    if not plan.owner_oncall_route or not plan.alert_enabled:
        return "manual_review"

    return "activate_drill_schedule"
```

这个判定器刻意把几个问题分开：

1. 冷路径没稳定，不准谈演练；
2. hot absence 没验证，不准把临时热路径包装成成功；
3. 种子里有 raw sensitive payload，要先收窄或脱敏；
4. 没有覆盖 runtime fingerprint、bundle 边界和新 SLA，就不能激活演练；
5. 没有 oncall route 和 alert，演练失败没人接，等于没演练。

对应测试可以这样写：

```python
def test_requires_seed_scope_narrowing_when_raw_sensitive_seed_exists():
    stability = ColdRetrievalStabilityReceipt(
        cold_path_id="cold-audit",
        stable=True,
        observed_samples=500,
        p95_ms=900,
        error_rate=0.0,
        runtime_fingerprint="cold-v18",
        hot_absence_verified=True,
        sla_rebaseline_approved=False,
    )
    seeds = RegressionSeedInventory(
        missing_lookup_key_samples=8,
        bundle_mismatch_samples=2,
        lease_denial_samples=1,
        redacted_seed_ratio=0.9,
        raw_sensitive_seed_count=1,
        has_runtime_fingerprint_seed=True,
    )
    plan = DrillSchedulePlan(
        interval_hours=24,
        covers_latest_runtime=True,
        covers_oldest_supported_bundle=True,
        covers_rebaseline_sla=True,
        owner_oncall_route="cold-path-oncall",
        alert_enabled=True,
    )

    assert decide_cold_retrieval_drill_gate(stability, seeds, plan) == "narrow_seed_scope"
```

重点不是“定时跑个脚本”，而是只有在种子安全、覆盖足够、告警闭环存在时，才允许把演练接入生产调度。

---

## 3. pi-mono：DrillActivationWorker

生产版建议把它做成稳定关闭后的后置 worker。它消费 `ColdRetrievalStabilityReceipt`，生成回归种子和调度收据。

```typescript
type DrillDecision =
  | "activate_drill_schedule"
  | "add_regression_seeds"
  | "narrow_seed_scope"
  | "extend_stability_observation"
  | "reopen_remediation"
  | "manual_review";

interface StabilityDrillJob {
  coldPathId: string;
  stabilityReceiptId: string;
}

class ColdRetrievalDrillActivationWorker {
  constructor(
    private readonly receipts: ReceiptStore,
    private readonly seedStore: RegressionSeedStore,
    private readonly drillScheduler: DrillScheduler,
    private readonly policy: ColdRetrievalPolicy,
    private readonly audit: AuditLog,
  ) {}

  async run(job: StabilityDrillJob) {
    const stability = await this.receipts.getColdRetrievalStability(
      job.stabilityReceiptId,
    );

    const seeds = await this.seedStore.buildInventoryFromIncident({
      coldPathId: job.coldPathId,
      incidentId: stability.incidentId,
      redactionProfile: stability.redactionProfile,
    });

    const plan = await this.drillScheduler.previewPlan({
      coldPathId: job.coldPathId,
      intervalHours: this.policy.defaultDrillIntervalHours,
      includeOldestSupportedBundle: true,
      includeLatestRuntimeFingerprint: true,
      slaProfile: stability.slaProfile,
    });

    const decision = decideColdRetrievalDrillGate(stability, seeds, plan);

    await this.audit.append("cold_retrieval_drill_gate_decided", {
      coldPathId: job.coldPathId,
      stabilityReceiptId: job.stabilityReceiptId,
      decision,
      seedCounts: seeds.counts,
      runtimeFingerprint: stability.runtimeFingerprint,
    });

    if (decision === "activate_drill_schedule") {
      const schedule = await this.drillScheduler.activate(plan);
      return this.receipts.writeDrillActivationReceipt({
        coldPathId: job.coldPathId,
        stabilityReceiptId: job.stabilityReceiptId,
        scheduleId: schedule.id,
        seedInventoryHash: seeds.inventoryHash,
        intervalHours: plan.intervalHours,
        alertRoute: plan.ownerOncallRoute,
      });
    }

    if (decision === "narrow_seed_scope") {
      return this.seedStore.openSeedRedactionTask({
        coldPathId: job.coldPathId,
        reason: "raw sensitive material in regression seed inventory",
      });
    }

    if (decision === "add_regression_seeds") {
      return this.seedStore.openSeedCoverageTask({
        coldPathId: job.coldPathId,
        requiredCoverage: [
          "missing_lookup_key",
          "bundle_mismatch",
          "lease_denial",
          "runtime_fingerprint",
        ],
      });
    }

    return this.receipts.writeManualReviewReceipt({
      coldPathId: job.coldPathId,
      stabilityReceiptId: job.stabilityReceiptId,
      decision,
    });
  }
}
```

生产里最重要的是两个 hash：

- `seedInventoryHash`：证明这次演练用的是哪批事故种子；
- `runtimeFingerprint`：证明演练不是在一个已经过期的 runtime 上自嗨。

如果未来演练失败，Agent 可以直接定位：是冷索引坏了、bundle 格式变了、lease policy 变了，还是 runtime 没收敛。

---

## 4. OpenClaw 课程 Cron 类比

我们这个课程 cron 其实也有同样问题。

每次课程任务完成后，不只是发 Telegram、写 lesson、更新 README、更新 TOOLS、git push。还应该留下能回放的“回归种子”：

- 本次 lesson 文件名和编号；
- README 是否追加了目录项；
- TOOLS 是否追加了已讲主题；
- Telegram 是否发送成功；
- commit hash 是否已 push；
- 下次选题是否能从 TOOLS 去重。

如果某次 cron 失败，下一次不能只说“重新跑”。它应该读取上一轮收据，把失败点变成回归检查：

```text
LessonPublishedReceipt
        ↓
RegressionSeed: lesson_number + topic_slug + commit_hash + telegram_message_id
        ↓
NextCronPreflight
        ↓
detect_duplicate_topic | detect_missing_readme_entry | detect_unpushed_commit
```

这就是今天主题在 OpenClaw 里的落地：**任务做完后，把真实失败和真实成功都变成下一轮自动检查的种子。**

---

## 5. 设计检查清单

实现 Cold Retrieval Drill Gate 时，可以用这个 checklist：

- 稳定收据是否验证了 hot absence；
- 回归种子是否来自真实事故样本，而不只是 synthetic happy path；
- 种子是否脱敏、收窄、绑定用途；
- drill 是否覆盖最新 runtime 和最老仍支持 bundle；
- SLA rebaseline 是否进入 drill 断言；
- drill 是否有 owner、告警路由、失败升级；
- drill activation 是否写入不可变收据；
- 下次演练失败时，是否能直接关联到原始 stability receipt。

成熟 Agent 的冷路径治理，不是“修好了就关单”，而是把修复经验转成自动演练资产。这样下一次系统变更时，旧事故会自己站出来提醒你。

