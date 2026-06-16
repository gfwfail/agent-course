# Agent 基线稳定关闭后的血缘压缩与回滚点解钉闸门

> Baseline Stable Closeout, Lineage Compaction & Rollback Unpin Gate

第 494 课讲了 Post-Repromotion Baseline Observation：candidate baseline 重新 promote 成 active 后，必须进入真实流量观察期，用 error、latency、cost、tool failure、sampling coverage 和 rollback probe 决定 close_stable、continue_observation、rollback_repromotion、reopen_baseline_case 或 manual_review。

今天继续补最后一环：观察期 close_stable 之后，不代表可以什么都不做。因为系统里还残留很多“事故模式”的临时资产：rollback baseline 被钉住、LKG 指针长期保留、临时阈值还在、case 证据分散在多个表里、baseline lineage 越堆越长。

所以第 495 课是 **Baseline Stable Closeout, Lineage Compaction & Rollback Unpin Gate**：当再晋级后的 baseline 通过观察期，要先归档证据、确认没有 open rollback debt，再解钉旧回滚点、压缩 baseline lineage、恢复正常发布策略。

一句话：**close_stable 不是“删掉保护”，而是把临时保护转换成可审计的长期资产，然后让生产路径回到干净状态。**

## 核心模型

把稳定关闭拆成 5 个对象：

~~~text
PostRepromotionCloseoutReceipt
  -> StableBaselineCloseoutReview
  -> LineageCompactionPlan
  -> RollbackUnpinPermit
  -> StableBaselineArchiveReceipt
~~~

分工：

- PostRepromotionCloseoutReceipt：上一课的观察关闭收据，必须是 close_stable；
- StableBaselineCloseoutReview：复核观察窗口、rollback debt、临时阈值、owner ack 和证据完整性；
- LineageCompactionPlan：决定哪些 baseline version 保留、哪些压缩、哪些只归档；
- RollbackUnpinPermit：允许解除 rollback baseline / LKG pin 的一次性许可；
- StableBaselineArchiveReceipt：最终归档收据，记录 stableVersion、retainedVersions、archivedEvidence、unpinResult。

关键点：**稳定关闭要先 review，再 unpin。** 如果直接删旧版本，后面发现证据缺口或隐藏信号回补，就失去了可验证回退点。

## learn-claude-code：稳定关闭判定纯函数

教学版继续用纯函数。输入观察关闭收据、关闭复核和血缘计划，输出下一步动作。

~~~python
# learn_claude_code/baseline_stable_closeout_gate.py
from dataclasses import dataclass
from typing import Literal

StableCloseoutDecision = Literal[
    "archive_and_unpin",
    "archive_keep_rollback_pin",
    "extend_observation",
    "reopen_baseline_case",
    "manual_review",
]


@dataclass
class PostRepromotionCloseoutReceipt:
    case_id: str
    stable_version: str
    decision: str  # close_stable
    observed_windows: int
    sampling_coverage: float
    rollback_probe_passed: bool


@dataclass
class StableBaselineCloseoutReview:
    case_id: str
    stable_version: str
    min_observed_windows: int
    min_sampling_coverage: float
    open_rollback_debt: int
    temporary_thresholds_active: int
    hidden_signal_backfill_open: bool
    owner_ack: bool
    archive_evidence_complete: bool


@dataclass
class LineageCompactionPlan:
    case_id: str
    stable_version: str
    rollback_version: str
    retained_versions: list[str]
    versions_to_archive: list[str]
    lkg_unpin_probe_passed: bool
    lineage_hash: str


def decide_stable_baseline_closeout(
    receipt: PostRepromotionCloseoutReceipt,
    review: StableBaselineCloseoutReview,
    plan: LineageCompactionPlan,
) -> StableCloseoutDecision:
    same_case = (
        receipt.case_id == review.case_id == plan.case_id
        and receipt.stable_version == review.stable_version == plan.stable_version
        and receipt.decision == "close_stable"
    )
    if not same_case:
        return "manual_review"

    if not review.archive_evidence_complete:
        return "manual_review"

    if review.hidden_signal_backfill_open:
        return "reopen_baseline_case"

    if receipt.observed_windows < review.min_observed_windows:
        return "extend_observation"

    if receipt.sampling_coverage < review.min_sampling_coverage:
        return "extend_observation"

    has_release_debt = (
        review.open_rollback_debt > 0
        or review.temporary_thresholds_active > 0
        or not review.owner_ack
    )
    if has_release_debt:
        return "archive_keep_rollback_pin"

    if not receipt.rollback_probe_passed:
        return "archive_keep_rollback_pin"

    if not plan.lkg_unpin_probe_passed:
        return "archive_keep_rollback_pin"

    if plan.stable_version not in plan.retained_versions:
        return "manual_review"

    if plan.rollback_version in plan.versions_to_archive and len(plan.retained_versions) < 2:
        return "manual_review"

    return "archive_and_unpin"
~~~

这里的重点不是“代码复杂”，而是把关闭动作拆清楚：

- 证据不完整，不关闭；
- 隐藏信号还在回补，重开 case；
- 观察窗口或采样不足，继续观察；
- 有 rollback debt 或临时阈值，归档但保留 pin；
- 只有证据、owner、probe、lineage 都对齐，才 archive_and_unpin。

## pi-mono：StableBaselineCloseoutWorker

生产版 worker 要把 archive、unpin、lineage compaction 放在一个事务里，并且所有指针操作都带 CAS 条件。

~~~ts
// packages/agent-runtime/src/ops/StableBaselineCloseoutWorker.ts
export type StableCloseoutDecision =
  | "archive_and_unpin"
  | "archive_keep_rollback_pin"
  | "extend_observation"
  | "reopen_baseline_case"
  | "manual_review";

export interface PostRepromotionCloseoutReceipt {
  caseId: string;
  stableVersion: string;
  decision: "close_stable";
  observedWindows: number;
  samplingCoverage: number;
  rollbackProbePassed: boolean;
}

export interface StableBaselineCloseoutReview {
  caseId: string;
  stableVersion: string;
  minObservedWindows: number;
  minSamplingCoverage: number;
  openRollbackDebt: number;
  temporaryThresholdsActive: number;
  hiddenSignalBackfillOpen: boolean;
  ownerAck: boolean;
  archiveEvidenceComplete: boolean;
}

export interface LineageCompactionPlan {
  caseId: string;
  stableVersion: string;
  rollbackVersion: string;
  retainedVersions: string[];
  versionsToArchive: string[];
  lkgUnpinProbePassed: boolean;
  lineageHash: string;
}

export class StableBaselineCloseoutWorker {
  constructor(private readonly store: BaselineOpsStore) {}

  decide(
    receipt: PostRepromotionCloseoutReceipt,
    review: StableBaselineCloseoutReview,
    plan: LineageCompactionPlan,
  ): StableCloseoutDecision {
    const sameCase =
      receipt.caseId === review.caseId &&
      receipt.caseId === plan.caseId &&
      receipt.stableVersion === review.stableVersion &&
      receipt.stableVersion === plan.stableVersion &&
      receipt.decision === "close_stable";

    if (!sameCase) return "manual_review";
    if (!review.archiveEvidenceComplete) return "manual_review";
    if (review.hiddenSignalBackfillOpen) return "reopen_baseline_case";
    if (receipt.observedWindows < review.minObservedWindows) {
      return "extend_observation";
    }
    if (receipt.samplingCoverage < review.minSamplingCoverage) {
      return "extend_observation";
    }

    const hasReleaseDebt =
      review.openRollbackDebt > 0 ||
      review.temporaryThresholdsActive > 0 ||
      !review.ownerAck;

    if (hasReleaseDebt) return "archive_keep_rollback_pin";
    if (!receipt.rollbackProbePassed) return "archive_keep_rollback_pin";
    if (!plan.lkgUnpinProbePassed) return "archive_keep_rollback_pin";
    if (!plan.retainedVersions.includes(plan.stableVersion)) {
      return "manual_review";
    }
    if (
      plan.versionsToArchive.includes(plan.rollbackVersion) &&
      plan.retainedVersions.length < 2
    ) {
      return "manual_review";
    }

    return "archive_and_unpin";
  }

  async handle(caseId: string) {
    return this.store.transaction(async (tx) => {
      const receipt = await tx.getPostRepromotionCloseout(caseId);
      const review = await tx.buildStableCloseoutReview(caseId);
      const plan = await tx.buildLineageCompactionPlan(caseId);
      const decision = this.decide(receipt, review, plan);

      const archive = await tx.writeStableBaselineArchiveReceipt({
        caseId,
        stableVersion: receipt.stableVersion,
        decision,
        lineageHash: plan.lineageHash,
        retainedVersions: plan.retainedVersions,
        archivedVersions: plan.versionsToArchive,
        decidedAt: Date.now(),
      });

      if (decision === "archive_and_unpin") {
        await tx.compactBaselineLineage({
          stableVersion: receipt.stableVersion,
          retainedVersions: plan.retainedVersions,
          archiveReceiptId: archive.id,
          expectedLineageHash: plan.lineageHash,
        });

        await tx.unpinRollbackBaseline({
          stableVersion: receipt.stableVersion,
          rollbackVersion: plan.rollbackVersion,
          expectedCurrent: receipt.stableVersion,
          archiveReceiptId: archive.id,
        });
      }

      if (decision === "archive_keep_rollback_pin") {
        await tx.keepRollbackPin({
          caseId,
          rollbackVersion: plan.rollbackVersion,
          reason: "stable_but_release_debt_or_probe_incomplete",
          archiveReceiptId: archive.id,
        });
      }

      if (decision === "extend_observation") {
        await tx.extendObservationLease({
          caseId,
          stableVersion: receipt.stableVersion,
          reason: "stable_closeout_requirements_not_met",
        });
      }

      if (decision === "reopen_baseline_case") {
        await tx.openBaselineCase({
          parentCaseId: caseId,
          baselineVersion: receipt.stableVersion,
          reason: "hidden_signal_backfill_during_stable_closeout",
        });
      }

      return archive;
    });
  }
}
~~~

注意两个工程细节：

第一，`compactBaselineLineage` 必须带 `expectedLineageHash`。如果另一个发布同时改了 baseline lineage，当前 closeout 不能覆盖它。

第二，`unpinRollbackBaseline` 必须带 `expectedCurrent`。解钉旧回滚点时，要确认 active 仍然是刚刚通过观察的 stableVersion。

## OpenClaw 实战：课程 Cron 的类比

这套机制可以直接类比 Agent 课程自动化：

1. 第 494 课确认 lesson 已写入、README 已更新、TOOLS 已追加、Telegram 已送达、Git 已 push；
2. 第 495 课要做的是收尾：确认没有未提交文件、没有重复目录项、没有 Telegram 发送失败补偿、没有临时草稿文件；
3. 如果一切稳定，把这次课程的 evidence 归档成 commit hash + messageId + lesson path；
4. 如果还有债务，比如 TOOLS 更新了但 repo push 失败，就不能算 closeout，只能 archive_keep_rollback_pin 或 reopen delivery case。

一个轻量 closeout receipt 可以长这样：

~~~json
{
  "caseId": "agent-course-495",
  "stableVersion": "lesson-495",
  "signals": {
    "lessonFile": "lessons/495-baseline-stable-closeout-lineage-compaction.md",
    "readmeLinked": true,
    "toolsUpdated": true,
    "gitPushed": true,
    "telegramDelivered": true
  },
  "decision": "archive_and_unpin"
}
~~~

课程 Cron 不一定要永久保存这个 JSON，但思维方式要保留：**完成一次外部副作用后，要检查工作区、交付通道和长期目录是否都回到稳定状态。**

## 常见坑

第一，close_stable 后立刻删除旧 baseline。错。旧 baseline 至少要等 evidence archive 完整、rollback debt 清零、unpin probe 通过。

第二，把 lineage compaction 当清理任务随便跑。错。它是控制面变更，必须带版本 hash 和收据。

第三，临时阈值忘记恢复。事故期间放宽 timeout、降低采样、关闭告警，如果稳定后不清掉，就会变成永久盲区。

第四，只归档日志，不归档决策原因。以后复盘时最重要的是“为什么可以 unpin”，不是“当时有哪些日志”。

## 记忆口诀

**观察通过不等于保护可删；证据归档、债务清零、probe 通过，才解钉回滚点。**

完整链路到这里变成：

~~~text
detect baseline drift
  -> remediate candidate
  -> shadow repromotion
  -> active observation
  -> stable closeout
  -> lineage compaction
  -> rollback unpin
~~~

成熟 Agent 的事故恢复，不只是知道什么时候回退，也知道什么时候可以安全地移除临时保护，让系统重新回到干净、可发布、可审计的正常形态。
