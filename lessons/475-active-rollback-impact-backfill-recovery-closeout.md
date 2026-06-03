# Agent 自动回滚后的影响回填与恢复关闭闸门

> Active Rollback Impact Backfill & Recovery Closeout Gate

第 474 课讲了：active merge 后要做发布后抽样审计，一旦发现 critical false suppression、counterfactual mismatch 或 stale runtime，就触发 rollback_active。

今天补 rollback_active 后面最容易被低估的一步：**回滚不是把 active 指针改回去就结束。回滚只保护未来流量；已经被错误策略影响过的 decision，要回填、补偿、复查，然后才能关闭事故。**

生产 Agent 里，最危险的回滚幻觉是“系统已经恢复了”。实际上它可能只是未来决策恢复了，过去被吞掉的信号还躺在 SuppressedSignalLedger 里，用户没有收到提醒，队列没有重新入队，审计样本没有 outcome，甚至还有旧 worker 持有过期策略继续跑。

## 核心模型

把 active rollback 后的闭环拆成 6 个对象：

~~~text
RollbackTriggerReport
  -> ActiveRollbackReceipt
  -> ImpactBackfillPlan
  -> BackfillExecutionReceipt
  -> RuntimeRecoveryProbe
  -> RollbackRecoveryCloseoutReceipt
~~~

这里有一个关键分工：

- ActiveRollbackReceipt 证明生产指针已经退回 Last Known Good；
- ImpactBackfillPlan 说明哪些历史 decision 被错误策略影响；
- BackfillExecutionReceipt 证明补偿动作已经执行或明确豁免；
- RuntimeRecoveryProbe 证明所有 runtime 真的加载了 LKG；
- RollbackRecoveryCloseoutReceipt 才允许关闭这次 rollback incident。

不要把这几件事压成一个 status。指针恢复、历史补偿、运行时一致和事故关闭，是四个不同事实。

## learn-claude-code：回滚关闭判定纯函数

教学版先写一个纯函数：输入回滚后的影响回填报告，输出继续回填、扩大扫描、隔离 runtime、人工复核或关闭。

~~~python
# learn_claude_code/active_rollback_recovery_closeout.py
from dataclasses import dataclass
from typing import Literal

CloseoutDecision = Literal[
    "close_recovered",
    "continue_backfill",
    "expand_impact_scan",
    "quarantine_runtime",
    "manual_review",
]


@dataclass
class RollbackRecoveryReport:
    rollback_receipt_id: str
    affected_decisions: int
    backfilled_decisions: int
    uncompensated_critical: int
    missing_outcomes: int
    stale_runtime_count: int
    lkg_runtime_match_rate: float
    reopened_cases: int
    unknown_impact_rate: float
    compensation_error_rate: float
    minutes_since_rollback: int


def decide_rollback_recovery(report: RollbackRecoveryReport) -> CloseoutDecision:
    if report.stale_runtime_count > 0:
        return "quarantine_runtime"

    if report.uncompensated_critical > 0:
        return "continue_backfill"

    if report.compensation_error_rate > 0.02:
        return "manual_review"

    if report.unknown_impact_rate > 0.10:
        return "expand_impact_scan"

    if report.missing_outcomes > 0:
        return "continue_backfill"

    if report.backfilled_decisions < report.affected_decisions:
        return "continue_backfill"

    if report.lkg_runtime_match_rate < 1.0:
        return "quarantine_runtime"

    if report.minutes_since_rollback < 30:
        return "continue_backfill"

    return "close_recovered"
~~~

这里最重要的是优先级：stale runtime 比 backfill 更紧急。因为只要还有 worker 继续用坏策略，补偿队列会一边修一边漏，永远清不干净。

## pi-mono：Rollback Recovery Worker

生产版要把回滚做成事务化控制器，而不是简单更新 policy pointer。

~~~ts
// packages/agent-runtime/src/suppression/ActiveRollbackRecoveryWorker.ts
export type CloseoutDecision =
  | "close_recovered"
  | "continue_backfill"
  | "expand_impact_scan"
  | "quarantine_runtime"
  | "manual_review";

export interface ActiveRollbackReceipt {
  id: string;
  policyId: string;
  badPolicyVersion: string;
  restoredLkgVersion: string;
  triggerReportId: string;
  rollbackPointId: string;
  pointerCasToken: string;
  createdAt: string;
}

export interface ImpactBackfillPlan {
  rollbackReceiptId: string;
  scanWindowStart: string;
  scanWindowEnd: string;
  affectedDecisionIds: string[];
  criticalDecisionIds: string[];
  unknownImpactDecisionIds: string[];
}

export interface BackfillExecutionReceipt {
  rollbackReceiptId: string;
  decisionId: string;
  action:
    | "re_emit_signal"
    | "reopen_case"
    | "notify_owner"
    | "mark_noop"
    | "manual_review";
  outcomeRef?: string;
  reason: string;
}

export class ActiveRollbackRecoveryWorker {
  constructor(private readonly store: SuppressionPolicyStore) {}

  async rollbackAndBackfill(input: {
    triggerReportId: string;
    policyId: string;
    badPolicyVersion: string;
    lkgVersion: string;
  }): Promise<ActiveRollbackReceipt> {
    return this.store.transaction(async (tx) => {
      const rollbackPoint = await tx.requireRollbackPoint({
        policyId: input.policyId,
        targetVersion: input.lkgVersion,
      });

      const receipt = await tx.compareAndSwapActivePolicy({
        policyId: input.policyId,
        expectedVersion: input.badPolicyVersion,
        nextVersion: input.lkgVersion,
        reason: "post_deploy_audit_triggered_rollback",
        triggerReportId: input.triggerReportId,
        rollbackPointId: rollbackPoint.id,
      });

      await tx.enqueueImpactBackfill({
        rollbackReceiptId: receipt.id,
        policyId: input.policyId,
        badPolicyVersion: input.badPolicyVersion,
        scanWindowStart: rollbackPoint.createdAt,
        scanWindowEnd: receipt.createdAt,
      });

      await tx.enqueueRuntimeRecoveryProbe({
        rollbackReceiptId: receipt.id,
        expectedRuntimeVersion: input.lkgVersion,
      });

      return receipt;
    });
  }

  decideCloseout(report: {
    affectedDecisions: number;
    backfilledDecisions: number;
    uncompensatedCritical: number;
    missingOutcomes: number;
    staleRuntimeCount: number;
    lkgRuntimeMatchRate: number;
    unknownImpactRate: number;
    compensationErrorRate: number;
  }): CloseoutDecision {
    if (report.staleRuntimeCount > 0) return "quarantine_runtime";
    if (report.uncompensatedCritical > 0) return "continue_backfill";
    if (report.compensationErrorRate > 0.02) return "manual_review";
    if (report.unknownImpactRate > 0.10) return "expand_impact_scan";
    if (report.missingOutcomes > 0) return "continue_backfill";
    if (report.backfilledDecisions < report.affectedDecisions) {
      return "continue_backfill";
    }
    if (report.lkgRuntimeMatchRate < 1) return "quarantine_runtime";
    return "close_recovered";
  }
}
~~~

实现重点有 5 个：

1. rollback 必须用 compare-and-swap，防止旧 worker 把新策略覆盖回去。
2. backfill 的扫描窗口从 RollbackPoint 到 ActiveRollbackReceipt，而不是只扫最近几分钟。
3. critical decision 要优先补偿，普通 decision 可以批量队列化。
4. RuntimeRecoveryProbe 要覆盖 API worker、cron worker、subagent runtime 和缓存层。
5. closeout 前必须确认 affectedDecisions 全部有 BackfillExecutionReceipt 或明确豁免。

## OpenClaw：课程 Cron 怎么落地

拿这个课程 cron 举例：假设“选题去重策略”active 后误把一个新主题判成已讲过，导致 cron 没发课。

第 474 课的 post-deploy audit 会触发 rollback_active；但这只让下一轮恢复旧策略。当前这一轮被跳过的课程，还需要影响回填：

- 扫描 memory、README、TOOLS 和 Telegram messageId，确认哪一轮被错误 suppress；
- 对没有发出的课程重新创建 lesson draft；
- 如果 Telegram 没发、git 没 commit，就重新执行完整发布；
- 如果 Telegram 已发但 git 未 push，就只补 git/README/TOOLS；
- 给每个受影响 cron run 写 BackfillExecutionReceipt；
- 最后检查 runtime 是否都回到 LKG dedupe policy。

一个 OpenClaw 风格的关闭收据可以长这样：

~~~json
{
  "receiptType": "RollbackRecoveryCloseoutReceipt",
  "policy": "lesson-topic-dedupe",
  "badPolicyVersion": "dedupe-2026-06-03-active",
  "restoredLkgVersion": "dedupe-2026-06-02-lkg",
  "rollbackReceiptId": "arr_475",
  "affectedDecisions": 2,
  "backfilledDecisions": 2,
  "uncompensatedCritical": 0,
  "staleRuntimeCount": 0,
  "decision": "close_recovered"
}
~~~

这张收据的意义是：以后再看到同一个 rollback incident，不需要重复补偿；系统可以根据 receipt 判断哪些 run 已经恢复，哪些仍要继续 backfill。

## 常见坑

1. **只改 active 指针**：这只能保护未来，不能修复已经漏掉的提醒、任务或消息。
2. **不扫 unknown impact**：不知道有没有影响，不等于没有影响；unknown 要扩大扫描或人工复核。
3. **回填没有幂等键**：补偿动作也可能重复发消息、重复开 case、重复 push。
4. **runtime probe 只查控制面**：数据库指针是 LKG，不代表每个 worker 缓存都刷新了。
5. **没有 closeout receipt**：没有关闭收据，下一轮自动化会在“已补偿”和“未补偿”之间反复横跳。

## 记住一句话

> 回滚只让系统停止继续犯错；影响回填和恢复关闭收据，才证明已经把犯过的错补回来。
