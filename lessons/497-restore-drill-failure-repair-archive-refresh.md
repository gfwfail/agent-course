# Agent 恢复演练失败后的修复计划与归档刷新闸门

> Restore Drill Failure Repair & Archive Refresh Gate

第 496 课讲了 Stable Baseline Archive Restore Drill：归档后的 baseline 要定期在隔离环境里恢复、重放、校验 runtime fingerprint / tool contract / evidence hash，证明“真的能恢复”。

今天补下一环：**恢复演练失败以后，不能只发告警，也不能直接重跑。** 失败可能来自四类完全不同的根因：

- 归档内容坏了：证据缺失、hash 不一致、rollback metadata 不完整；
- 恢复路径坏了：schema migration、runner、tool contract 或 runtime fingerprint 漂移；
- 旧 baseline 本身失效：smoke replay 不达标，说明过去的“稳定”已不再稳定；
- 演练安全边界坏了：sandbox 里出现 forbidden side effect。

这些失败不能用同一个 retry 解决。所以第 497 课是 **Restore Drill Failure Repair & Archive Refresh Gate**：把恢复演练失败结果转成结构化 Repair Plan，区分 refresh archive、rebuild restore path、reopen baseline case 和 quarantine drill runner，再用刷新后的归档重新演练，直到拿到新的关闭收据。

一句话：**成熟 Agent 的恢复演练失败不是“再跑一次”，而是先证明该修归档、修路径、重开案件，还是隔离演练环境。**

## 核心模型

把失败修复拆成 5 个对象：

~~~text
RestoreDrillCloseoutReceipt
  -> RestoreDrillFailureTriage
  -> RestoreRepairPlan
  -> ArchiveRefreshExecution
  -> RestoreRepairCloseoutReceipt
~~~

分工：

- RestoreDrillCloseoutReceipt：第 496 课输出的演练关闭收据，包含 decision 和失败证据；
- RestoreDrillFailureTriage：把失败分类成 archive_corrupt、restore_path_broken、baseline_invalid、sandbox_violation、ambiguous；
- RestoreRepairPlan：声明修复目标、允许动作、禁止副作用、需要重新验证的证据；
- ArchiveRefreshExecution：真正刷新 archive / rebuild path / reopen case / quarantine runner 的执行结果；
- RestoreRepairCloseoutReceipt：决定 repair_verified、rerun_restore_drill、reopen_baseline_case、quarantine_runner 或 manual_review。

关键点：**先分类，再执行。** 如果 forbidden side effect 是 sandbox runner 的问题，重建 archive 没用；如果 evidence hash 不一致，修 runner 也没用。

## learn-claude-code：失败分类与修复闸门纯函数

教学版先写纯函数：输入上次演练关闭收据、失败 triage 和修复执行结果，输出下一步动作。

~~~python
# learn_claude_code/restore_drill_failure_repair_gate.py
from dataclasses import dataclass
from typing import Literal

DrillDecision = Literal[
    "archive_verified",
    "refresh_archive",
    "rebuild_restore_path",
    "reopen_baseline_case",
    "manual_review",
]

FailureClass = Literal[
    "archive_corrupt",
    "restore_path_broken",
    "baseline_invalid",
    "sandbox_violation",
    "ambiguous",
]

RepairDecision = Literal[
    "repair_verified",
    "rerun_restore_drill",
    "reopen_baseline_case",
    "quarantine_runner",
    "manual_review",
]


@dataclass
class RestoreDrillCloseoutReceipt:
    case_id: str
    stable_version: str
    archive_hash: str
    decision: DrillDecision
    forbidden_side_effects_seen: int
    missing_evidence_count: int
    runtime_fingerprint_match: bool
    tool_contract_match: bool
    smoke_replay_pass_rate: float
    min_smoke_replay_pass_rate: float


@dataclass
class RestoreDrillFailureTriage:
    case_id: str
    stable_version: str
    archive_hash: str
    failure_class: FailureClass
    confidence: float
    evidence_refs: list[str]


@dataclass
class ArchiveRefreshExecution:
    case_id: str
    stable_version: str
    old_archive_hash: str
    new_archive_hash: str | None
    archive_refreshed: bool
    restore_path_rebuilt: bool
    runner_quarantined: bool
    baseline_case_reopened: bool
    verification_rerun_scheduled: bool


def classify_failure(receipt: RestoreDrillCloseoutReceipt) -> FailureClass:
    if receipt.forbidden_side_effects_seen > 0:
        return "sandbox_violation"
    if receipt.missing_evidence_count > 0 or receipt.decision == "refresh_archive":
        return "archive_corrupt"
    if (
        not receipt.runtime_fingerprint_match
        or not receipt.tool_contract_match
        or receipt.decision == "rebuild_restore_path"
    ):
        return "restore_path_broken"
    if (
        receipt.smoke_replay_pass_rate < receipt.min_smoke_replay_pass_rate
        or receipt.decision == "reopen_baseline_case"
    ):
        return "baseline_invalid"
    return "ambiguous"


def decide_repair_closeout(
    receipt: RestoreDrillCloseoutReceipt,
    triage: RestoreDrillFailureTriage,
    execution: ArchiveRefreshExecution,
) -> RepairDecision:
    same_case = (
        receipt.case_id == triage.case_id == execution.case_id
        and receipt.stable_version == triage.stable_version == execution.stable_version
        and receipt.archive_hash == triage.archive_hash == execution.old_archive_hash
    )
    if not same_case:
        return "manual_review"

    if receipt.decision == "archive_verified":
        return "repair_verified"

    if triage.confidence < 0.75 or not triage.evidence_refs:
        return "manual_review"

    if triage.failure_class == "sandbox_violation":
        if execution.runner_quarantined and execution.baseline_case_reopened:
            return "reopen_baseline_case"
        if execution.runner_quarantined:
            return "quarantine_runner"
        return "manual_review"

    if triage.failure_class == "archive_corrupt":
        if execution.archive_refreshed and execution.new_archive_hash:
            return "rerun_restore_drill"
        return "manual_review"

    if triage.failure_class == "restore_path_broken":
        if execution.restore_path_rebuilt and execution.verification_rerun_scheduled:
            return "rerun_restore_drill"
        return "manual_review"

    if triage.failure_class == "baseline_invalid":
        if execution.baseline_case_reopened:
            return "reopen_baseline_case"
        return "manual_review"

    return "manual_review"
~~~

这里有一个实战经验：`archive_corrupt` 和 `restore_path_broken` 都可以通过 rerun drill 重新证明；但 `baseline_invalid` 不应该靠刷新归档掩盖，必须重开 baseline case，因为旧稳定结论已经失效。

## pi-mono：FailureRepairWorker

生产版建议把 triage、repair 和 rerun permit 做成同一个事务化 worker，避免“修了一半，演练又开始跑旧 archive”。

~~~ts
// packages/agent-runtime/src/ops/RestoreDrillFailureRepairWorker.ts
export type FailureClass =
  | "archive_corrupt"
  | "restore_path_broken"
  | "baseline_invalid"
  | "sandbox_violation"
  | "ambiguous";

export type RepairDecision =
  | "repair_verified"
  | "rerun_restore_drill"
  | "reopen_baseline_case"
  | "quarantine_runner"
  | "manual_review";

export interface RestoreDrillCloseoutReceipt {
  caseId: string;
  stableVersion: string;
  archiveHash: string;
  decision:
    | "archive_verified"
    | "refresh_archive"
    | "rebuild_restore_path"
    | "reopen_baseline_case"
    | "manual_review";
  forbiddenSideEffectsSeen: number;
  missingEvidenceCount: number;
  runtimeFingerprintMatch: boolean;
  toolContractMatch: boolean;
  smokeReplayPassRate: number;
  minSmokeReplayPassRate: number;
}

export interface RestoreDrillFailureTriage {
  caseId: string;
  stableVersion: string;
  archiveHash: string;
  failureClass: FailureClass;
  confidence: number;
  evidenceRefs: string[];
}

export interface ArchiveRefreshExecution {
  caseId: string;
  stableVersion: string;
  oldArchiveHash: string;
  newArchiveHash?: string;
  archiveRefreshed: boolean;
  restorePathRebuilt: boolean;
  runnerQuarantined: boolean;
  baselineCaseReopened: boolean;
  verificationRerunScheduled: boolean;
}

export class RestoreDrillFailureRepairWorker {
  constructor(
    private readonly store: RestoreArchiveStore,
    private readonly planner: RestoreRepairPlanner,
    private readonly outbox: OpsOutbox,
  ) {}

  classify(receipt: RestoreDrillCloseoutReceipt): FailureClass {
    if (receipt.forbiddenSideEffectsSeen > 0) return "sandbox_violation";
    if (receipt.missingEvidenceCount > 0 || receipt.decision === "refresh_archive") {
      return "archive_corrupt";
    }
    if (
      !receipt.runtimeFingerprintMatch ||
      !receipt.toolContractMatch ||
      receipt.decision === "rebuild_restore_path"
    ) {
      return "restore_path_broken";
    }
    if (
      receipt.smokeReplayPassRate < receipt.minSmokeReplayPassRate ||
      receipt.decision === "reopen_baseline_case"
    ) {
      return "baseline_invalid";
    }
    return "ambiguous";
  }

  decide(
    receipt: RestoreDrillCloseoutReceipt,
    triage: RestoreDrillFailureTriage,
    execution: ArchiveRefreshExecution,
  ): RepairDecision {
    const sameCase =
      receipt.caseId === triage.caseId &&
      receipt.caseId === execution.caseId &&
      receipt.stableVersion === triage.stableVersion &&
      receipt.stableVersion === execution.stableVersion &&
      receipt.archiveHash === triage.archiveHash &&
      receipt.archiveHash === execution.oldArchiveHash;

    if (!sameCase) return "manual_review";
    if (receipt.decision === "archive_verified") return "repair_verified";
    if (triage.confidence < 0.75 || triage.evidenceRefs.length === 0) {
      return "manual_review";
    }

    switch (triage.failureClass) {
      case "sandbox_violation":
        if (execution.runnerQuarantined && execution.baselineCaseReopened) {
          return "reopen_baseline_case";
        }
        return execution.runnerQuarantined ? "quarantine_runner" : "manual_review";
      case "archive_corrupt":
        return execution.archiveRefreshed && execution.newArchiveHash
          ? "rerun_restore_drill"
          : "manual_review";
      case "restore_path_broken":
        return execution.restorePathRebuilt && execution.verificationRerunScheduled
          ? "rerun_restore_drill"
          : "manual_review";
      case "baseline_invalid":
        return execution.baselineCaseReopened
          ? "reopen_baseline_case"
          : "manual_review";
      default:
        return "manual_review";
    }
  }

  async repair(receipt: RestoreDrillCloseoutReceipt): Promise<RepairDecision> {
    return this.store.transaction(async (tx) => {
      const triage = await this.planner.triage(tx, {
        ...receipt,
        failureClass: this.classify(receipt),
      });

      const execution = await this.planner.executeRepair(tx, triage, {
        forbidProductionSideEffects: true,
        requireFenceToken: true,
      });

      const decision = this.decide(receipt, triage, execution);
      await tx.writeRepairCloseout({ receipt, triage, execution, decision });

      if (decision === "rerun_restore_drill") {
        await tx.createSingleUseRerunPermit({
          caseId: receipt.caseId,
          archiveHash: execution.newArchiveHash ?? receipt.archiveHash,
        });
      }

      await this.outbox.enqueue("restore_drill_repair_closed", {
        caseId: receipt.caseId,
        stableVersion: receipt.stableVersion,
        decision,
      });

      return decision;
    });
  }
}
~~~

生产实现里最容易踩的坑：

- rerun permit 必须一次性消费，防止旧 worker 拿旧 archive 重跑；
- refresh archive 后 archiveHash 必须变化，否则等于没有新证据；
- sandbox violation 要先 quarantine runner，再决定是否重开 baseline case；
- baseline_invalid 不允许被 archive refresh “洗白”。

## OpenClaw：课程 Cron 的类比

这套机制在 OpenClaw 课程自动化里也能类比：

- 如果 lesson 文件写错，是 archive_corrupt：修文件、更新 README，再重跑 git diff；
- 如果 Telegram 工具失败，是 restore_path_broken：修发送路径，不能改课程内容糊弄；
- 如果选题重复，是 baseline_invalid：重开选题 case，不能靠改标题继续推；
- 如果 git push 推到了错误账号，就是 sandbox_violation：先隔离凭据/账号上下文，再决定是否回滚。

自动化任务长期运行时，最怕的是“失败后盲目重试”。正确做法是：每次失败都先归类，再选择最小修复动作，并留下 closeout receipt。

## 落地检查清单

- Drill failure 是否有结构化 failureClass，而不是只有错误字符串？
- Archive refresh 是否产生新的 archiveHash 和 evidence hash？
- Restore path rebuild 是否绑定 runtime fingerprint / tool contract 重新验证？
- Baseline invalid 是否强制 reopen case，而不是静默刷新归档？
- Sandbox violation 是否隔离 runner，并检查是否有真实副作用泄露？
- Rerun restore drill 是否通过 single-use permit 原子触发？

## 小结

第 496 课证明归档能不能恢复；第 497 课证明恢复失败以后系统会不会乱修。

核心原则：

- 失败先分类；
- 分类决定修复路径；
- 修复后必须重新演练；
- 演练环境出问题先隔离；
- baseline 失效必须重开案件；
- 所有动作都用收据闭环。

成熟 Agent 的恢复体系，不是“备份 + 重试”，而是“演练 + 分类 + 修复 + 再验证 + 收据”。
