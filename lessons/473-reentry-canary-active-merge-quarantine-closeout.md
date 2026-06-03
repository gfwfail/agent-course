# Agent 抑制策略重入后的金丝雀并回与隔离清账闸门

> Reentry Canary Active Merge & Quarantine Closeout Gate

第 472 课讲了：suppression policy 重入以后，要用观察窗口决定继续、延长、退出隔离、回滚或推进 canary。

今天补下一步：**canary 通过不等于可以直接覆盖 active，更不等于 quarantine case 可以关掉。**

很多 Agent 系统一看到 reentry canary 指标恢复，就急着把策略设回 active。问题是，降级期间可能已经产生了补偿任务、隔离状态、人工复核 case、缓存里的旧 policy fingerprint、以及被临时放开的 reopen signal。真正的合并动作要同时证明两件事：新策略可以重新成为 active；旧隔离状态可以被有序清账。

## 核心模型

把重入后的最终恢复拆成 6 个对象：

~~~text
ReentryPromotionReceipt
  -> CanaryCloseoutReport
  -> ActiveMergePlan
  -> RuntimeConsistencyProbe
  -> QuarantineCloseoutReceipt
  -> ActiveMergeReceipt
~~~

这里最关键的是：**promotion proves candidate health, merge proves system consistency**。canary 只证明候选策略在一小段流量里表现正常，active merge 还要证明 runtime、缓存、隔离补偿和下游状态都能一起收敛。

## learn-claude-code：并回闸门纯函数

教学版先写一个纯函数：输入 canary closeout 和隔离状态，输出是否允许并回 active。

~~~python
# learn_claude_code/reentry_active_merge_gate.py
from dataclasses import dataclass
from typing import Literal

MergeDecision = Literal[
    "merge_active",
    "extend_canary",
    "block_merge",
    "close_quarantine_only",
    "manual_review",
]


@dataclass
class ReentryCanaryCloseout:
    policy_version: str
    sample_count: int
    false_suppressions: int
    critical_coverage: float
    unknown_rate: float
    runtime_fingerprint_match: bool
    compensation_open: int
    quarantine_cases_open: int
    stale_runtime_count: int
    baseline_delta: float


def decide_active_merge(report: ReentryCanaryCloseout) -> MergeDecision:
    if report.false_suppressions > 0:
        return "block_merge"

    if report.critical_coverage < 0.99:
        return "block_merge"

    if not report.runtime_fingerprint_match or report.stale_runtime_count > 0:
        return "extend_canary"

    if report.unknown_rate > 0.10:
        return "extend_canary"

    if report.sample_count < 500:
        return "extend_canary"

    if report.compensation_open > 0:
        return "manual_review"

    if report.quarantine_cases_open > 0 and report.baseline_delta <= 0.02:
        return "close_quarantine_only"

    if report.baseline_delta > 0.05:
        return "manual_review"

    return "merge_active"
~~~

注意这里故意把 close_quarantine_only 和 merge_active 分开。因为有些情况下，canary 还没到 active 样本量，但旧 quarantine case 已经没有开放补偿，可以先关隔离账，不要让降级状态无限挂着。

## pi-mono：Active Merge Worker

生产版要在一个事务里锁住 policy 指针、runtime group 和 quarantine case，避免两个 worker 同时把不同版本写成 active。

~~~ts
// packages/agent-runtime/src/suppression/ReentryActiveMergeWorker.ts
export type MergeDecision =
  | "merge_active"
  | "extend_canary"
  | "block_merge"
  | "close_quarantine_only"
  | "manual_review";

export interface ReentryCanaryCloseout {
  policyVersion: string;
  promotionReceiptId: string;
  sampleCount: number;
  falseSuppressions: number;
  criticalCoverage: number;
  unknownRate: number;
  runtimeFingerprintMatch: boolean;
  compensationOpen: number;
  quarantineCasesOpen: number;
  staleRuntimeCount: number;
  baselineDelta: number;
}

export class ReentryActiveMergeWorker {
  constructor(private readonly store: SuppressionPolicyStore) {}

  async closeout(report: ReentryCanaryCloseout) {
    const decision = this.decide(report);

    return this.store.transaction(async (tx) => {
      const current = await tx.lockSuppressionPolicyPointer("active");
      const candidate = await tx.lockPolicyCandidate(report.policyVersion);

      await tx.assertPromotionReceipt({
        receiptId: report.promotionReceiptId,
        policyVersion: report.policyVersion,
        mode: "canary",
      });

      if (decision === "block_merge") {
        await tx.disablePolicyCandidate({
          policyVersion: report.policyVersion,
          reason: "reentry_canary_failed_merge_gate",
        });

        return tx.createActiveMergeReceipt({
          policyVersion: report.policyVersion,
          previousActiveVersion: current.policyVersion,
          decision,
          activeChanged: false,
        });
      }

      if (decision === "extend_canary") {
        return tx.extendCanaryLease({
          policyVersion: report.policyVersion,
          trafficPercent: candidate.trafficPercent,
          expiresAt: this.hoursFromNow(3),
          reason: "merge_gate_requires_more_consistency",
        });
      }

      if (decision === "close_quarantine_only") {
        await tx.closeQuarantineCases({
          policyVersion: report.policyVersion,
          reason: "reentry_candidate_stable_but_not_active_ready",
        });

        return tx.createQuarantineCloseoutReceipt({
          policyVersion: report.policyVersion,
          activeChanged: false,
        });
      }

      if (decision === "manual_review") {
        return tx.openPolicyReviewCase({
          policyVersion: report.policyVersion,
          reason: "reentry_merge_requires_human_review",
        });
      }

      await tx.assertNoOpenCompensation(report.policyVersion);
      await tx.assertRuntimeGroupsConsistent(report.policyVersion);

      await tx.compareAndSwapActivePolicy({
        expectedVersion: current.policyVersion,
        nextVersion: report.policyVersion,
      });

      await tx.closeQuarantineCases({
        policyVersion: report.policyVersion,
        reason: "reentry_policy_merged_active",
      });

      return tx.createActiveMergeReceipt({
        policyVersion: report.policyVersion,
        previousActiveVersion: current.policyVersion,
        decision,
        activeChanged: true,
      });
    });
  }

  private decide(report: ReentryCanaryCloseout): MergeDecision {
    if (report.falseSuppressions > 0) return "block_merge";
    if (report.criticalCoverage < 0.99) return "block_merge";
    if (!report.runtimeFingerprintMatch) return "extend_canary";
    if (report.staleRuntimeCount > 0) return "extend_canary";
    if (report.unknownRate > 0.10) return "extend_canary";
    if (report.sampleCount < 500) return "extend_canary";
    if (report.compensationOpen > 0) return "manual_review";
    if (report.quarantineCasesOpen > 0 && report.baselineDelta <= 0.02) {
      return "close_quarantine_only";
    }
    if (report.baselineDelta > 0.05) return "manual_review";
    return "merge_active";
  }

  private hoursFromNow(hours: number): string {
    return new Date(Date.now() + hours * 60 * 60 * 1000).toISOString();
  }
}
~~~

这里有 4 个实现要点：

1. compareAndSwapActivePolicy 必须用当前 active version 做 CAS，避免并发 closeout 覆盖新版本。
2. assertRuntimeGroupsConsistent 要检查热更新、缓存、tool metadata 和 executor build fingerprint，不只检查数据库指针。
3. assertNoOpenCompensation 必须在 merge 前跑，避免旧补偿任务在新 active 策略下继续写回。
4. QuarantineCloseoutReceipt 和 ActiveMergeReceipt 要分开，前者清隔离账，后者改生产指针。

## OpenClaw：课程 Cron 怎么落地

拿这套课程 cron 举例：假设我们之前发现“选题去重策略”误伤，把它降级到 quarantine，然后修复后进入 canary。

并回 active 前不能只看“连续几次没重复”：

- README 目录、TOOLS 已讲内容、Telegram messageId、git commit hash 都要对账；
- 本地 runtime 和远端 repo 都要确认使用同一个去重规则版本；
- quarantine 期间打开的补偿任务必须关闭或明确转人工；
- active 指针切换后，要写 ActiveMergeReceipt，记录 previousVersion、nextVersion、runtimeFingerprint 和 closeout case。

一个 OpenClaw 风格的 closeout 摘要可以长这样：

~~~json
{
  "receiptType": "ActiveMergeReceipt",
  "policy": "lesson-topic-dedupe",
  "previousVersion": "dedupe-2026-06-02",
  "nextVersion": "dedupe-2026-06-03-reentry",
  "runtimeFingerprint": "agent-course-cron@c444838",
  "telegramChecked": true,
  "readmeChecked": true,
  "toolsChecked": true,
  "openCompensation": 0,
  "decision": "merge_active"
}
~~~

这不是形式主义。课程 cron 这种任务同时写文件、发群、提交 git、更新记忆；任何一个状态没对齐，都可能让下一轮以为“没讲过”或“已经讲过但没推送”。并回 active 的核心就是：**让所有状态持有者一起承认同一个现实。**

## 常见坑

1. **只看 canary 成功率**：suppression policy 的核心风险是漏告警，不是报错率。
2. **先改 active 再清 quarantine**：旧补偿 worker 可能会用旧上下文写回，制造二次误伤。
3. **没有 runtime fingerprint**：数据库指针变了，不代表所有 worker、缓存和热加载策略都变了。
4. **把 close quarantine 当 merge active**：清隔离账和改生产指针是两个动作，收据必须分开。

## 记住一句话

> 重入策略回到 active，不是“canary 没炸就上线”，而是 policy 指针、runtime 指纹、补偿任务、隔离 case 和现实结果全部对齐后，才写下可回滚的合并收据。
