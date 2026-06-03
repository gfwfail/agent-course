# Agent 回滚恢复关闭后的长期哨兵与再发预算

> Post-Recovery Sentinel & Recurrence Budget

第 475 课讲了：active rollback 之后不能只恢复 LKG 指针，还要做影响回填、运行时恢复探针和 RollbackRecoveryCloseoutReceipt，证明已经把犯过的错补回来。

今天继续补关闭之后的一步：**close_recovered 不是永久平安，只是事故闭环可以关闭。真正成熟的 Agent 会在恢复关闭后保留一个轻量长期哨兵，用再发预算判断是噪声、复发，还是需要重新打开事故。**

很多生产系统在 rollback incident 关闭后就停止采样。问题是，复发经常不是马上出现：旧 session 延迟恢复、cron 跨天触发、缓存 TTL 到期、低频工具重新出现、某个 subagent 还带着旧 prompt fragment。短期 closeout 证明当前证据齐了，长期 sentinel 证明同类错误没有在现实世界慢慢回来。

## 核心模型

把 recovery closeout 后的长期监控拆成 5 个对象：

~~~text
RollbackRecoveryCloseoutReceipt
  -> PostRecoverySentinelLease
  -> RecurrenceSignalSample
  -> RecurrenceBudgetReport
  -> SentinelExitReceipt | ReopenRollbackCase
~~~

关键点是：**closeout 关闭事故，sentinel 监控复发债务**。不要把长期哨兵做成永远打开的事故 ticket；它应该有采样范围、预算、退出条件和重开条件。

## learn-claude-code：再发预算判定纯函数

教学版先写一个纯函数：输入 sentinel 采样报告，输出继续观察、扩大采样、重开 rollback case、降级策略或退出哨兵。

~~~python
# learn_claude_code/post_recovery_sentinel.py
from dataclasses import dataclass
from typing import Literal

SentinelDecision = Literal[
    "exit_sentinel",
    "continue_sentinel",
    "increase_sampling",
    "reopen_rollback_case",
    "demote_policy",
    "manual_review",
]


@dataclass
class RecurrenceBudgetReport:
    closeout_receipt_id: str
    hours_since_closeout: int
    sampled_decisions: int
    recurrence_signals: int
    critical_recurrences: int
    same_signature_recurrences: int
    stale_runtime_hits: int
    unresolved_backfill_refs: int
    unknown_outcome_rate: float
    baseline_regression: float
    recurrence_budget_remaining: int


def decide_post_recovery_sentinel(report: RecurrenceBudgetReport) -> SentinelDecision:
    if report.stale_runtime_hits > 0:
        return "demote_policy"

    if report.critical_recurrences > 0:
        return "reopen_rollback_case"

    if report.unresolved_backfill_refs > 0:
        return "reopen_rollback_case"

    if report.same_signature_recurrences >= 2:
        return "reopen_rollback_case"

    if report.recurrence_budget_remaining < 0:
        return "manual_review"

    if report.unknown_outcome_rate > 0.12:
        return "increase_sampling"

    if report.baseline_regression > 0.04:
        return "increase_sampling"

    if report.hours_since_closeout < 72:
        return "continue_sentinel"

    if report.sampled_decisions < 500:
        return "continue_sentinel"

    return "exit_sentinel"
~~~

这里的重点不是阈值本身，而是优先级：

1. stale runtime 是执行层问题，优先降级或隔离；
2. critical recurrence 和 unresolved backfill 说明恢复关闭可能误关，直接重开；
3. same signature recurrence 说明根因没修干净；
4. unknown outcome 和 baseline regression 先扩大采样，不急着误报。

## pi-mono：PostRecoverySentinelWorker

生产版把 sentinel 做成轻量 lease。它不阻塞正常流量，只旁路订阅 decision/outcome/backfill 事件，并在预算耗尽或复发命中时升级。

~~~ts
// packages/agent-runtime/src/suppression/PostRecoverySentinelWorker.ts
export type SentinelDecision =
  | "exit_sentinel"
  | "continue_sentinel"
  | "increase_sampling"
  | "reopen_rollback_case"
  | "demote_policy"
  | "manual_review";

export interface PostRecoverySentinelLease {
  id: string;
  closeoutReceiptId: string;
  policyId: string;
  restoredLkgVersion: string;
  watchSignatures: string[];
  recurrenceBudget: number;
  sampleRate: number;
  expiresAt: string;
}

export interface RecurrenceBudgetReport {
  closeoutReceiptId: string;
  hoursSinceCloseout: number;
  sampledDecisions: number;
  recurrenceSignals: number;
  criticalRecurrences: number;
  sameSignatureRecurrences: number;
  staleRuntimeHits: number;
  unresolvedBackfillRefs: number;
  unknownOutcomeRate: number;
  baselineRegression: number;
  recurrenceBudgetRemaining: number;
}

export class PostRecoverySentinelWorker {
  constructor(private readonly store: SuppressionPolicyStore) {}

  async evaluate(report: RecurrenceBudgetReport) {
    const decision = this.decide(report);

    return this.store.transaction(async (tx) => {
      await tx.assertRollbackRecoveryCloseout({
        receiptId: report.closeoutReceiptId,
        status: "close_recovered",
      });

      if (decision === "reopen_rollback_case") {
        return tx.reopenRollbackCase({
          closeoutReceiptId: report.closeoutReceiptId,
          reason: "post_recovery_recurrence_detected",
          recurrenceSignals: report.recurrenceSignals,
          sameSignatureRecurrences: report.sameSignatureRecurrences,
        });
      }

      if (decision === "demote_policy") {
        return tx.demoteActivePolicy({
          closeoutReceiptId: report.closeoutReceiptId,
          reason: "post_recovery_stale_runtime_hit",
          nextMode: "canary",
        });
      }

      if (decision === "increase_sampling") {
        return tx.extendSentinelLease({
          closeoutReceiptId: report.closeoutReceiptId,
          sampleRateMultiplier: 2,
          expiresAt: this.hoursFromNow(48),
          reason: "recurrence_budget_needs_more_evidence",
        });
      }

      if (decision === "continue_sentinel") {
        return tx.extendSentinelLease({
          closeoutReceiptId: report.closeoutReceiptId,
          sampleRateMultiplier: 1,
          expiresAt: this.hoursFromNow(24),
          reason: "minimum_observation_window_not_met",
        });
      }

      if (decision === "manual_review") {
        return tx.openPolicyReviewCase({
          closeoutReceiptId: report.closeoutReceiptId,
          reason: "recurrence_budget_exhausted_without_clear_signature",
        });
      }

      return tx.createSentinelExitReceipt({
        closeoutReceiptId: report.closeoutReceiptId,
        sampledDecisions: report.sampledDecisions,
        recurrenceSignals: report.recurrenceSignals,
        reason: "recurrence_budget_stable",
      });
    });
  }

  private decide(report: RecurrenceBudgetReport): SentinelDecision {
    if (report.staleRuntimeHits > 0) return "demote_policy";
    if (report.criticalRecurrences > 0) return "reopen_rollback_case";
    if (report.unresolvedBackfillRefs > 0) return "reopen_rollback_case";
    if (report.sameSignatureRecurrences >= 2) return "reopen_rollback_case";
    if (report.recurrenceBudgetRemaining < 0) return "manual_review";
    if (report.unknownOutcomeRate > 0.12) return "increase_sampling";
    if (report.baselineRegression > 0.04) return "increase_sampling";
    if (report.hoursSinceCloseout < 72) return "continue_sentinel";
    if (report.sampledDecisions < 500) return "continue_sentinel";
    return "exit_sentinel";
  }

  private hoursFromNow(hours: number): string {
    return new Date(Date.now() + hours * 60 * 60 * 1000).toISOString();
  }
}
~~~

实现重点有 5 个：

1. sentinel 订阅的是真实 decision/outcome/backfill 事件，不靠人工复盘。
2. watchSignatures 要绑定事故根因签名，比如 policyVersion、dedupeKey、toolName、riskSurface。
3. recurrenceBudget 是允许出现的轻微信号预算，不是错误次数清零的幻想。
4. critical recurrence 不消费预算，直接 reopen。
5. exit_sentinel 必须写 SentinelExitReceipt，证明为什么可以停止长期监控。

## OpenClaw：课程 Cron 怎么落地

拿这个课程 cron 举例：第 475 课说如果选题去重策略误伤，rollback 后要补发漏掉的课程、补 README、补 TOOLS、补 git push。第 476 课要问的是：补完以后，未来 72 小时还会不会复发？

可以给课程 cron 建一个轻量 sentinel：

- 观察 README、TOOLS、memory 和 Telegram messageId 是否再次出现不一致；
- 采样“新主题被判已讲过”的 decision；
- 检查跨天 memory 文件里有没有同一个 cron id 被错误 suppress；
- 如果 git push 成功但 README 缺目录，算 recurrence signal；
- 如果 Telegram 未发送但 TOOLS 已记录“已讲”，直接 reopen rollback case；
- 72 小时内样本足够且 recurrenceSignals 为 0，写 SentinelExitReceipt。

一个 OpenClaw 风格的退出收据可以长这样：

~~~json
{
  "receiptType": "SentinelExitReceipt",
  "policy": "lesson-topic-dedupe",
  "closeoutReceiptId": "rrc_475",
  "watchWindowHours": 72,
  "sampledDecisions": 18,
  "recurrenceSignals": 0,
  "criticalRecurrences": 0,
  "sameSignatureRecurrences": 0,
  "unknownOutcomeRate": 0.03,
  "baselineRegression": 0,
  "decision": "exit_sentinel"
}
~~~

如果出现复发，重开收据应该带上同源签名：

~~~json
{
  "receiptType": "ReopenRollbackCase",
  "policy": "lesson-topic-dedupe",
  "closeoutReceiptId": "rrc_475",
  "signature": "telegram_sent_git_missing_tools_recorded",
  "sameSignatureRecurrences": 2,
  "decision": "reopen_rollback_case"
}
~~~

## 设计原则

Post-recovery sentinel 有 3 条硬规则：

1. **轻量**：它不是重新开一个重型事故流程，而是低成本旁路采样。
2. **有预算**：小噪声可以被记录和消耗预算，关键复发直接升级。
3. **可退出**：没有 SentinelExitReceipt 的 sentinel 会变成永久后台负担。

成熟 Agent 的事故关闭，不是“关掉就忘”。它会在恢复后保留一段有限、可解释、可退出的长期哨兵，确认同类问题没有慢慢回到生产路径。
