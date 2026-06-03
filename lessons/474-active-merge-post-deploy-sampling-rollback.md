# Agent Active 合并后的发布后抽样审计与自动回滚闸门

> Active Merge Post-deploy Sampling Audit & Automatic Rollback Gate

第 473 课讲了：reentry canary 通过后，不能直接覆盖 active，要用 ActiveMergeReceipt 证明 runtime、隔离补偿和生产指针全部一致。

今天补 active merge 之后最容易漏的一步：**ActiveMergeReceipt 不是终点，它只是证明新策略已经安全切到 active；真正危险的是切完以后真实流量开始全量命中。**

很多 Agent 系统上线策略时只看发布前 canary。问题是，canary 的样本通常更干净、scope 更小、操作者更警觉；active 后才会遇到长尾租户、旧缓存、低频工具、跨时区 cron、迟到事件和历史状态回潮。所以 active merge 后必须自动开一个短期抽样审计窗口：用真实决策持续证明“少打扰”没有变成“漏风险”。

## 核心模型

把 active merge 后的治理拆成 6 个对象：

~~~text
ActiveMergeReceipt
  -> PostDeployAuditLease
  -> StratifiedDecisionSample
  -> CounterfactualReplayResult
  -> RollbackTriggerReport
  -> PostDeployCloseoutReceipt
~~~

这里最关键的是：**merge proves control-plane consistency, post-deploy audit proves data-plane safety**。控制面指针一致，只代表所有 worker 加载了同一个策略；数据面安全，要靠真实决策、反事实回放和 outcome 证据来证明。

## learn-claude-code：发布后回滚判定纯函数

教学版先写一个纯函数：输入发布后抽样审计结果，输出继续观察、关闭窗口、扩大抽样、自动回滚或人工复核。

~~~python
# learn_claude_code/post_deploy_suppression_audit.py
from dataclasses import dataclass
from typing import Literal

AuditDecision = Literal[
    "close_stable",
    "continue_audit",
    "increase_sampling",
    "rollback_active",
    "manual_review",
]


@dataclass
class PostDeployAuditReport:
    policy_version: str
    minutes_since_merge: int
    sampled_decisions: int
    false_suppressions: int
    critical_false_suppressions: int
    unexplained_suppressions: int
    stale_runtime_decisions: int
    counterfactual_mismatch_rate: float
    unknown_outcome_rate: float
    baseline_delta: float
    rollback_probe_ready: bool


def decide_post_deploy_audit(report: PostDeployAuditReport) -> AuditDecision:
    if report.critical_false_suppressions > 0:
        return "rollback_active"

    if report.false_suppressions > 0 and report.rollback_probe_ready:
        return "rollback_active"

    if report.stale_runtime_decisions > 0:
        return "manual_review"

    if report.counterfactual_mismatch_rate > 0.03:
        return "increase_sampling"

    if report.unexplained_suppressions > 5:
        return "increase_sampling"

    if report.unknown_outcome_rate > 0.15:
        return "continue_audit"

    if report.sampled_decisions < 1000 or report.minutes_since_merge < 60:
        return "continue_audit"

    if report.baseline_delta > 0.05:
        return "manual_review"

    return "close_stable"
~~~

这里故意把 false_suppression 设成最高优先级。抑制策略的失败不是“多打扰了用户”，而是“该提醒的时候没提醒”。这类事故宁可先回滚，再用补偿队列处理已影响的 decision。

## pi-mono：Post-deploy Audit Worker

生产版要把审计做成旁路 worker：订阅 active policy decision 事件，按风险分层抽样，再用旧策略或 LKG 策略做反事实回放。

~~~ts
// packages/agent-runtime/src/suppression/PostDeployAuditWorker.ts
export type AuditDecision =
  | "close_stable"
  | "continue_audit"
  | "increase_sampling"
  | "rollback_active"
  | "manual_review";

export interface PostDeployAuditReport {
  policyVersion: string;
  activeMergeReceiptId: string;
  minutesSinceMerge: number;
  sampledDecisions: number;
  falseSuppressions: number;
  criticalFalseSuppressions: number;
  unexplainedSuppressions: number;
  staleRuntimeDecisions: number;
  counterfactualMismatchRate: number;
  unknownOutcomeRate: number;
  baselineDelta: number;
  rollbackProbeReady: boolean;
}

export class PostDeploySuppressionAuditWorker {
  constructor(private readonly store: SuppressionPolicyStore) {}

  async closeout(report: PostDeployAuditReport) {
    const decision = this.decide(report);

    return this.store.transaction(async (tx) => {
      await tx.assertActiveMergeReceipt({
        receiptId: report.activeMergeReceiptId,
        policyVersion: report.policyVersion,
        activeChanged: true,
      });

      if (decision === "rollback_active") {
        const rollbackPoint = await tx.lockRollbackPoint(report.policyVersion);

        await tx.compareAndSwapActivePolicy({
          expectedVersion: report.policyVersion,
          nextVersion: rollbackPoint.previousActiveVersion,
        });

        await tx.enqueueImpactBackfill({
          policyVersion: report.policyVersion,
          reason: "post_deploy_false_suppression",
          includeSampledDecisions: true,
        });

        return tx.createPostDeployCloseoutReceipt({
          policyVersion: report.policyVersion,
          decision,
          rolledBackTo: rollbackPoint.previousActiveVersion,
        });
      }

      if (decision === "increase_sampling") {
        return tx.extendPostDeployAuditLease({
          policyVersion: report.policyVersion,
          sampleRateMultiplier: 2,
          expiresAt: this.minutesFromNow(90),
          reason: "post_deploy_mismatch_requires_more_evidence",
        });
      }

      if (decision === "continue_audit") {
        return tx.extendPostDeployAuditLease({
          policyVersion: report.policyVersion,
          sampleRateMultiplier: 1,
          expiresAt: this.minutesFromNow(60),
          reason: "post_deploy_budget_not_satisfied",
        });
      }

      if (decision === "manual_review") {
        return tx.openPolicyReviewCase({
          policyVersion: report.policyVersion,
          reason: "post_deploy_audit_requires_owner_decision",
        });
      }

      return tx.createPostDeployCloseoutReceipt({
        policyVersion: report.policyVersion,
        decision,
        rolledBackTo: null,
      });
    });
  }

  private decide(report: PostDeployAuditReport): AuditDecision {
    if (report.criticalFalseSuppressions > 0) return "rollback_active";
    if (report.falseSuppressions > 0 && report.rollbackProbeReady) {
      return "rollback_active";
    }
    if (report.staleRuntimeDecisions > 0) return "manual_review";
    if (report.counterfactualMismatchRate > 0.03) return "increase_sampling";
    if (report.unexplainedSuppressions > 5) return "increase_sampling";
    if (report.unknownOutcomeRate > 0.15) return "continue_audit";
    if (report.sampledDecisions < 1000) return "continue_audit";
    if (report.minutesSinceMerge < 60) return "continue_audit";
    if (report.baselineDelta > 0.05) return "manual_review";
    return "close_stable";
  }

  private minutesFromNow(minutes: number): string {
    return new Date(Date.now() + minutes * 60 * 1000).toISOString();
  }
}
~~~

实现重点有 4 个：

1. 抽样必须分层：critical、external side effect、low-frequency tool、new tenant、old session 都要有最低样本。
2. 反事实回放不能只看 allow/block，要比较 reason、evidence freshness、dedupeKey 和 suppressionUntil。
3. rollback_active 必须同时 enqueueImpactBackfill，回滚未来流量不等于修复已经被错误抑制的信号。
4. close_stable 要写 PostDeployCloseoutReceipt，证明审计窗口为什么可以关闭，而不是“跑了一小时没报错”。

## OpenClaw：课程 Cron 怎么落地

拿这个课程 cron 举例：假设我们把“选题去重策略”修复后重新并回 active。ActiveMergeReceipt 只能证明 README、TOOLS、Telegram、git remote 和本地 runtime 指纹一致；它不能证明后续 3 小时、6 小时、跨天的课程不会被错误抑制。

发布后审计应该抽样这些真实决策：

- 新课题是否被误判成“已讲过”；
- README 已有但 TOOLS 缺失时，策略是否进入 reconcile，而不是直接 suppress；
- Telegram 发出但 git push 失败时，下一轮是否正确 retry，而不是因为 messageId 存在就跳过；
- 旧 memory 里的摘要和最新 TOOLS 冲突时，是否触发人工复核或重新对账。

一个 OpenClaw 风格的审计收据可以长这样：

~~~json
{
  "receiptType": "PostDeployCloseoutReceipt",
  "policy": "lesson-topic-dedupe",
  "policyVersion": "dedupe-2026-06-03-reentry",
  "activeMergeReceiptId": "amr_473",
  "sampledDecisions": 1240,
  "falseSuppressions": 0,
  "counterfactualMismatchRate": 0.008,
  "unknownOutcomeRate": 0.04,
  "decision": "close_stable"
}
~~~

这类收据的价值在于：下一轮 cron 不需要“相信上一次发布成功”，它可以读取 closeout receipt，知道策略已经在真实流量中被审计过，可以继续使用；如果没有 closeout，就继续抽样或降低策略权重。

## 常见坑

1. **只抽样高频路径**：高频路径最容易在 canary 里覆盖，真正容易坏的是低频工具和旧状态。
2. **把 unknown outcome 当成功**：不知道有没有误伤，不等于没有误伤；unknown 必须消耗预算。
3. **回滚不补偿历史 decision**：active 指针退回只能保护未来，过去被错误抑制的信号要进入 backfill。
4. **没有关闭收据**：没有 PostDeployCloseoutReceipt，下一轮 worker 就只能靠猜测判断策略是否稳定。

## 记住一句话

> Active merge 只是把策略放进生产；post-deploy audit 才证明它在真实世界里没有安静地漏掉重要信号。
