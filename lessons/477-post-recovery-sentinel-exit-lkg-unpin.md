# Agent 哨兵退出后的 LKG 解钉与证据归档闸门

> Post-Recovery Sentinel Exit, LKG Unpin & Evidence Archive Gate

第 476 课讲了 post-recovery sentinel：回滚恢复关闭以后，还要用一段观察窗口证明同类问题没有慢慢回来。

今天讲 sentinel 通过后的最后一步：**退出哨兵不等于把所有临时保护一关了事。退出时要释放 LKG 固定、压缩证据、更新回归种子，并写一张能审计的 SentinelExitReceipt。**

很多 Agent 事故恢复后会留下“安全拐杖”：active policy 被固定在 LKG、采样率临时提高、某些策略禁止自动发布、补偿 ledger 维持热存储。这些保护在事故期是必要的，但如果不带证据退出，会变成长期成本、发布阻塞和隐性配置漂移。

## 核心模型

把哨兵退出拆成 6 个对象：

~~~text
PostRecoverySentinelLease
  -> SentinelExitReview
  -> LkgUnpinPlan
  -> RegressionSeedUpdate
  -> EvidenceArchiveReceipt
  -> SentinelExitReceipt
~~~

分工要清楚：

- SentinelExitReview 判断哨兵窗口是否真的稳定；
- LkgUnpinPlan 说明哪些临时 pin、freeze、sampling override 可以释放；
- RegressionSeedUpdate 把这次事故的最小复发样本写入 regression pack；
- EvidenceArchiveReceipt 把热证据压缩成长期可审计归档；
- SentinelExitReceipt 才允许关闭哨兵 lease，并恢复正常发布节奏。

关键原则：**退出不是删除保护，而是把临时保护转成长期规则、回归样本或归档证据。**

## learn-claude-code：哨兵退出判定纯函数

教学版先写纯函数：输入哨兵窗口总结，输出退出、延长、保留 LKG pin、重开事故或人工复核。

~~~python
# learn_claude_code/sentinel_exit_gate.py
from dataclasses import dataclass
from typing import Literal

ExitDecision = Literal[
    "exit_and_unpin_lkg",
    "exit_keep_lkg_pin",
    "extend_sentinel",
    "reopen_recovery_case",
    "manual_review",
]


@dataclass
class SentinelExitReview:
    lease_id: str
    observation_hours: int
    required_hours: int
    recurrence_count: int
    critical_recurrence_count: int
    unknown_outcome_rate: float
    stale_runtime_count: int
    unresolved_backfill_count: int
    baseline_regression_rate: float
    regression_seed_count: int
    evidence_archive_ready: bool
    release_train_blocked_by_lkg_pin: bool


def decide_sentinel_exit(review: SentinelExitReview) -> ExitDecision:
    if review.critical_recurrence_count > 0:
        return "reopen_recovery_case"

    if review.stale_runtime_count > 0:
        return "extend_sentinel"

    if review.unresolved_backfill_count > 0:
        return "extend_sentinel"

    if review.observation_hours < review.required_hours:
        return "extend_sentinel"

    if review.unknown_outcome_rate > 0.03:
        return "manual_review"

    if review.baseline_regression_rate > 0.01:
        return "extend_sentinel"

    if review.recurrence_count > 0:
        return "exit_keep_lkg_pin"

    if review.regression_seed_count == 0:
        return "manual_review"

    if not review.evidence_archive_ready:
        return "manual_review"

    return "exit_and_unpin_lkg"
~~~

这个判定里有两个细节：

1. 有非 critical recurrence，不一定要重开事故，但不应该立刻释放 LKG pin。
2. regression_seed_count 为 0 不能退出，因为事故没有沉淀成可防复发资产。

## pi-mono：Sentinel Exit Worker

生产版要把退出做成事务，避免“哨兵关了，但 pin 没释放”或“pin 释放了，但证据没归档”的半状态。

~~~ts
// packages/agent-runtime/src/recovery/PostRecoverySentinelExitWorker.ts
export type SentinelExitDecision =
  | "exit_and_unpin_lkg"
  | "exit_keep_lkg_pin"
  | "extend_sentinel"
  | "reopen_recovery_case"
  | "manual_review";

export interface SentinelExitReview {
  leaseId: string;
  policyId: string;
  recoveryCaseId: string;
  observationHours: number;
  requiredHours: number;
  recurrenceCount: number;
  criticalRecurrenceCount: number;
  unknownOutcomeRate: number;
  staleRuntimeCount: number;
  unresolvedBackfillCount: number;
  baselineRegressionRate: number;
  regressionSeedCount: number;
  evidenceArchiveReady: boolean;
}

export interface LkgUnpinPlan {
  policyId: string;
  pinnedVersion: string;
  releasePins: Array<"active_policy" | "release_train" | "high_sampling" | "manual_approval">;
  keepPins: string[];
  reason: string;
}

export interface SentinelExitReceipt {
  id: string;
  leaseId: string;
  recoveryCaseId: string;
  decision: SentinelExitDecision;
  lkgUnpinPlanId?: string;
  evidenceArchiveReceiptId?: string;
  regressionSeedRefs: string[];
  createdAt: string;
}

export class PostRecoverySentinelExitWorker {
  constructor(private readonly store: RecoveryStore) {}

  decide(review: SentinelExitReview): SentinelExitDecision {
    if (review.criticalRecurrenceCount > 0) return "reopen_recovery_case";
    if (review.staleRuntimeCount > 0) return "extend_sentinel";
    if (review.unresolvedBackfillCount > 0) return "extend_sentinel";
    if (review.observationHours < review.requiredHours) return "extend_sentinel";
    if (review.unknownOutcomeRate > 0.03) return "manual_review";
    if (review.baselineRegressionRate > 0.01) return "extend_sentinel";
    if (review.recurrenceCount > 0) return "exit_keep_lkg_pin";
    if (review.regressionSeedCount === 0) return "manual_review";
    if (!review.evidenceArchiveReady) return "manual_review";
    return "exit_and_unpin_lkg";
  }

  async closeLease(input: { leaseId: string }): Promise<SentinelExitReceipt> {
    return this.store.transaction(async (tx) => {
      const review = await tx.buildSentinelExitReview(input.leaseId);
      const decision = this.decide(review);

      if (decision === "reopen_recovery_case") {
        await tx.reopenRecoveryCase({
          recoveryCaseId: review.recoveryCaseId,
          reason: "critical_recurrence_during_post_recovery_sentinel",
        });
      }

      if (decision === "extend_sentinel") {
        await tx.extendSentinelLease({
          leaseId: review.leaseId,
          reason: "exit_conditions_not_met",
        });
      }

      const archive = await tx.archiveSentinelEvidence({
        leaseId: review.leaseId,
        recoveryCaseId: review.recoveryCaseId,
        mode: decision.startsWith("exit") ? "compact_hot_keep_hash" : "retain_hot",
      });

      const regressionSeeds = await tx.promoteRegressionSeeds({
        recoveryCaseId: review.recoveryCaseId,
        leaseId: review.leaseId,
        requireMinimalReproducer: true,
      });

      let unpinPlan: LkgUnpinPlan | undefined;
      if (decision === "exit_and_unpin_lkg") {
        unpinPlan = await tx.releaseLkgPins({
          policyId: review.policyId,
          leaseId: review.leaseId,
          releasePins: ["active_policy", "release_train", "high_sampling"],
          reason: "sentinel_exit_review_passed",
        });
      }

      return tx.writeSentinelExitReceipt({
        leaseId: review.leaseId,
        recoveryCaseId: review.recoveryCaseId,
        decision,
        lkgUnpinPlanId: unpinPlan?.policyId,
        evidenceArchiveReceiptId: archive.id,
        regressionSeedRefs: regressionSeeds.map((seed) => seed.id),
      });
    });
  }
}
~~~

实现重点：

1. closeLease 必须事务化写 receipt，不能靠多个脚本手工收尾。
2. releaseLkgPins 只在 exit_and_unpin_lkg 时执行；exit_keep_lkg_pin 仍然可以关闭哨兵，但保留发布保护。
3. archiveSentinelEvidence 在所有路径都执行：退出要归档，延长要保留热证据。
4. promoteRegressionSeeds 要求 minimal reproducer，不能把整包日志直接塞进回归集。
5. SentinelExitReceipt 要能反查 lease、recovery case、archive receipt 和 regression seed。

## OpenClaw：课程 Cron 怎么落地

拿这个课程 cron 举例：之前几课一直在讲“选题去重策略误 suppress 新主题”的事故链。

当 PostRecoverySentinelLease 连续几轮没有发现重复跳课、runtime fingerprint 一致、Git/Telegram/README/TOOLS 都对账正常，就可以做退出审查：

~~~text
SentinelExitReview:
  observation_hours: 72
  recurrence_count: 0
  critical_recurrence_count: 0
  unknown_outcome_rate: 0.00
  stale_runtime_count: 0
  unresolved_backfill_count: 0
  baseline_regression_rate: 0.00
  regression_seed_count: 3
  evidence_archive_ready: true
=> exit_and_unpin_lkg
~~~

退出时要做三件实际动作：

- 释放课程选题策略的 LKG pin，让新的去重规则可以正常进入 shadow/canary；
- 把这次事故的最小样本写入 regression pack，比如“相邻主题同属 suppression policy 系列但标题不同，不能判重复”；
- 把 Telegram message id、commit sha、README diff、TOOLS.md 摘要 hash 归档，热日志可以压缩，但审计链不能断。

OpenClaw 这类长期 cron 特别容易积累临时保护。今天加的 lesson、README、TOOLS、Telegram 和 git push，就是一个小型 side-effect chain。退出哨兵时不能只看“最近没报错”，还要确认这些 side effect 都还能被追溯。

## 常见坑

### 坑 1：哨兵退出时直接删证据

事故证据不是垃圾。热日志可以压缩，原始 payload 可以脱敏，但必须留下 hash、索引和 closeout receipt。否则下次复发时无法证明“这是旧问题回来”还是“新问题长得像”。

### 坑 2：LKG pin 永远不释放

LKG pin 是事故期保护，不是长期架构。永远固定在 LKG 会阻塞后续策略修复，最后团队只能绕过发布系统。

### 坑 3：没有 regression seed 就退出

没有回归种子的退出，只是忘记事故。成熟系统要把事故样本变成未来发布门禁的一部分。

### 坑 4：把 exit 和 unpin 绑定死

有些情况可以退出哨兵但保留 LKG pin，比如观察窗口只有轻微 recurrence，或者 release train 还没恢复稳定。退出监控和释放保护是两个决策。

## 工程检查清单

- SentinelExitReview 是否覆盖 recurrence、runtime、backfill、baseline 和 evidence archive？
- LkgUnpinPlan 是否列出 releasePins 和 keepPins，而不是一个布尔值？
- RegressionSeedUpdate 是否有 minimal reproducer？
- EvidenceArchiveReceipt 是否保留 hash、索引、retention policy 和 redaction profile？
- SentinelExitReceipt 是否能反查所有上游 receipt？
- release train 恢复后是否有一次轻量 smoke replay？
- 临时高采样率是否恢复，避免长期成本膨胀？
- dashboard 是否从 incident mode 切回 normal mode？

## 关键 takeaway

**哨兵退出不是“终于没事了”，而是把事故期临时保护转换成长期可验证资产。**

成熟 Agent 的恢复闭环应该是：

~~~text
rollback
  -> backfill
  -> close_recovered
  -> sentinel
  -> exit review
  -> unpin / archive / regression seed
~~~

没有 SentinelExitReceipt，就别急着说事故真正结束。否则系统只是安静了一阵子，下一次同类问题会换个入口回来。

