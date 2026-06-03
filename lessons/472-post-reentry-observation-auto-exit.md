# Agent 抑制策略重入后的观察窗口与自动退出闸门

> Post-Reentry Observation Window & Auto-Exit Gate

第 471 课讲了：suppression policy 被降级以后，要先 quarantine、补偿被吞信号，再用 ReentryPermit 控制它能不能重新进入 shadow/canary。

今天继续补最后一段：**重入不是恢复，重入只是带限制的试运行。**

很多系统拿到 reentry permit 后会犯一个错误：shadow/canary 一开始没报错，就默认修好了。可 suppression policy 最危险的失败不是显性 crash，而是静悄悄地把关键 reopen signal 吞掉。重入以后必须有短期观察窗口，持续比较 old baseline、repaired policy 和真实 outcome，一旦出现同类误伤、unknown 暴涨或 critical coverage 掉线，就自动退出。

## 核心模型

把重入后的闭环拆成 5 个对象：

```text
ReentryPermit
  -> ReentryObservationLease
  -> ReentryPolicyDecisionSample
  -> ReentryOutcomeProbe
  -> ReentryExitReceipt | ReentryPromotionReceipt
```

这里最关键的是：**permit controls entry, observation controls survival**。许可只证明它可以开始试跑，观察窗口才证明它是否值得继续留在生产路径里。

## learn-claude-code：重入观察判定纯函数

教学版先写一个纯函数：输入观察窗口内的样本和结果，输出下一步动作。

```python
# learn_claude_code/reentry_observation_gate.py
from dataclasses import dataclass
from typing import Literal

ReentryDecision = Literal[
    "continue_observation",
    "extend_observation",
    "exit_to_quarantine",
    "rollback_reentry",
    "promote_canary",
    "manual_review",
]


@dataclass
class ReentryObservation:
    mode: Literal["shadow", "canary"]
    observed_decisions: int
    false_suppressions: int
    same_signature_failures: int
    unknown_rate: float
    critical_coverage: float
    baseline_delta: float
    permit_expired: bool


def decide_reentry_survival(obs: ReentryObservation) -> ReentryDecision:
    if obs.permit_expired:
        return "rollback_reentry"

    if obs.false_suppressions > 0 or obs.same_signature_failures > 0:
        return "exit_to_quarantine"

    if obs.critical_coverage < 0.98:
        return "rollback_reentry"

    if obs.unknown_rate > 0.15:
        return "extend_observation"

    if obs.observed_decisions < 100:
        return "continue_observation"

    if obs.baseline_delta > 0.05:
        return "manual_review"

    if obs.mode == "shadow":
        return "promote_canary"

    return "continue_observation"
```

这里不要只看 pass rate。suppression policy 的风险核心是 **false suppression**：少发一个告警、少重开一个 case、少提醒一次 owner，表面都像“系统更安静了”，实际可能是漏风险。

## pi-mono：重入观察 Worker

生产版要把 observation lease、decision sample、outcome probe 和 exit receipt 串成一条可审计链路。

```ts
// packages/agent-runtime/src/suppression/ReentryObservationWorker.ts
export type ReentryDecision =
  | "continue_observation"
  | "extend_observation"
  | "exit_to_quarantine"
  | "rollback_reentry"
  | "promote_canary"
  | "manual_review";

export interface ReentryObservationLease {
  leaseId: string;
  permitId: string;
  policyVersion: string;
  mode: "shadow" | "canary";
  startedAt: string;
  expiresAt: string;
  fenceToken: string;
}

export interface ReentryObservationReport {
  leaseId: string;
  policyVersion: string;
  observedDecisions: number;
  falseSuppressions: number;
  sameSignatureFailures: number;
  unknownRate: number;
  criticalCoverage: number;
  baselineDelta: number;
  permitExpired: boolean;
}

export class ReentryObservationWorker {
  constructor(private readonly store: SuppressionPolicyStore) {}

  async evaluate(report: ReentryObservationReport) {
    const decision = this.decide(report);

    return this.store.transaction(async (tx) => {
      const lease = await tx.lockReentryObservationLease(report.leaseId);

      await tx.assertFenceToken({
        leaseId: lease.leaseId,
        fenceToken: lease.fenceToken,
      });

      if (decision === "exit_to_quarantine") {
        await tx.disablePolicyReentry({
          policyVersion: report.policyVersion,
          reason: "false_suppression_during_reentry",
        });

        return tx.createReentryExitReceipt({
          leaseId: report.leaseId,
          policyVersion: report.policyVersion,
          decision,
          reopenQuarantine: true,
        });
      }

      if (decision === "rollback_reentry") {
        await tx.rollbackToLastKnownGoodSuppressionPolicy({
          policyVersion: report.policyVersion,
          sourceLeaseId: report.leaseId,
        });

        return tx.createReentryExitReceipt({
          leaseId: report.leaseId,
          policyVersion: report.policyVersion,
          decision,
          reopenQuarantine: false,
        });
      }

      if (decision === "promote_canary") {
        return tx.createReentryPromotionReceipt({
          leaseId: report.leaseId,
          policyVersion: report.policyVersion,
          nextMode: "canary",
          trafficPercent: 2,
          expiresAt: this.hoursFromNow(2),
        });
      }

      if (decision === "extend_observation") {
        return tx.extendObservationLease({
          leaseId: report.leaseId,
          expiresAt: this.hoursFromNow(3),
          reason: "unknown_rate_above_budget",
        });
      }

      if (decision === "manual_review") {
        return tx.openPolicyReviewCase({
          policyVersion: report.policyVersion,
          sourceLeaseId: report.leaseId,
          reason: "reentry_baseline_delta_requires_review",
        });
      }

      return tx.createObservationHeartbeat({
        leaseId: report.leaseId,
        policyVersion: report.policyVersion,
        decision,
      });
    });
  }

  private decide(report: ReentryObservationReport): ReentryDecision {
    if (report.permitExpired) return "rollback_reentry";
    if (report.falseSuppressions > 0) return "exit_to_quarantine";
    if (report.sameSignatureFailures > 0) return "exit_to_quarantine";
    if (report.criticalCoverage < 0.98) return "rollback_reentry";
    if (report.unknownRate > 0.15) return "extend_observation";
    if (report.observedDecisions < 100) return "continue_observation";
    if (report.baselineDelta > 0.05) return "manual_review";
    if (report.mode === "shadow") return "promote_canary";
    return "continue_observation";
  }

  private hoursFromNow(hours: number): string {
    return new Date(Date.now() + hours * 60 * 60 * 1000).toISOString();
  }
}
```

这里有 3 个实现要点：

1. `fenceToken` 必须校验，避免旧 observation worker 在 lease 过期后继续写 promotion。
2. `exit_to_quarantine` 和 `rollback_reentry` 要分开：前者说明修复仍会误伤同类信号，后者可能只是样本不足、许可过期或覆盖率掉线。
3. `promote_canary` 也只能给很小流量和短 TTL，不能直接回 active。

## OpenClaw：课程 Cron 怎么落地

拿 Agent 开发课程 cron 举例，上一课如果给了一个短期 `ReentryPermit`：

```text
policyVersion: suppression-v471-fixed
mode: shadow
expiresAt: 2026-06-03T15:30:00+10:00
```

重入观察窗口要采集这些信号：

```text
decision sample:
  - README title 相似但 lesson path 不同，是否被误判重复
  - TOOLS.md 已讲内容相似但主题不同，是否被误 suppress
  - Telegram messageId 已存在但 git commit 未推送，是否还能触发补偿

outcome probe:
  - git remote main 是否包含本次 commit
  - Telegram 群是否实际收到课程
  - README / lessons / TOOLS 三处是否一致
  - memory 记录是否写入对应 messageId 和 commit
```

如果 shadow 期间发现“README 已有相似标题，所以跳过新课”，这不是安静，这是 false suppression，必须立刻 `exit_to_quarantine`。

如果只是 `unknown_rate` 高，比如 Telegram API 一时读不到 messageId，可以 `extend_observation`，但不能升 canary。

## 实战检查清单

- ReentryPermit 是否有 TTL、mode、trafficPercent 和 source quarantine case？
- ObservationLease 是否有 fenceToken，能防旧 worker 写回？
- 是否单独统计 false_suppression，而不是只看错误率？
- 是否采集 outcome probe，证明被放行/被抑制的决策在现实里真的正确？
- 是否能自动 exit_to_quarantine，而不是等人看 dashboard？

一句话总结：**Agent 的策略重入不是“修好了再上”，而是“带许可进入、带观察存活、带收据退出”。**
