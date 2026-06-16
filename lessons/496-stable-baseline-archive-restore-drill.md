# Agent 稳定基线归档后的恢复演练闸门

> Stable Baseline Archive Restore Drill Gate

第 495 课讲了 Baseline Stable Closeout：baseline 通过观察期后，要把证据归档、压缩 lineage、确认 rollback debt 清零，再解钉旧 LKG / rollback baseline。

今天补下一环：**归档成功不等于未来可用**。很多系统的事故恢复失败，不是因为没有备份，而是因为备份从来没演练过：schema 变了、依赖工具版本变了、证据被脱密后无法重放、旧配置可以读但不能重新激活。

所以第 496 课是 **Stable Baseline Archive Restore Drill Gate**：稳定 baseline 归档后，定期抽样做恢复演练，证明归档包可以被读取、校验、在隔离环境恢复、跑过 smoke replay，并且不会误触生产副作用。

一句话：**成熟 Agent 的归档不是“存起来”，而是能定期证明自己真的能恢复。**

## 核心模型

把恢复演练拆成 5 个对象：

~~~text
StableBaselineArchiveReceipt
  -> ArchiveRestoreDrillPlan
  -> IsolatedRestoreRun
  -> RestoreReplayVerification
  -> RestoreDrillCloseoutReceipt
~~~

分工：

- StableBaselineArchiveReceipt：第 495 课的归档收据，记录 stableVersion、lineageHash、evidenceRefs、retained rollback metadata；
- ArchiveRestoreDrillPlan：演练计划，指定抽样范围、隔离环境、允许恢复的字段、禁止副作用清单；
- IsolatedRestoreRun：真实执行恢复，但只能在 sandbox / staging / replay runner 里跑；
- RestoreReplayVerification：对恢复后的 baseline 做 smoke replay、runtime fingerprint、tool contract 和 evidence hash 校验；
- RestoreDrillCloseoutReceipt：关闭收据，决定 archive_verified、refresh_archive、rebuild_restore_path、reopen_baseline_case 或 manual_review。

关键点：**恢复演练必须隔离副作用。** 你可以恢复配置、重放决策、跑工具 mock，但不能在演练里真的发消息、推送部署、扣款或改生产状态。

## learn-claude-code：恢复演练判定纯函数

教学版先做纯函数。输入归档收据、演练运行结果和回放验证结果，输出下一步动作。

~~~python
# learn_claude_code/stable_baseline_restore_drill_gate.py
from dataclasses import dataclass
from typing import Literal

RestoreDrillDecision = Literal[
    "archive_verified",
    "refresh_archive",
    "rebuild_restore_path",
    "reopen_baseline_case",
    "manual_review",
]


@dataclass
class StableBaselineArchiveReceipt:
    case_id: str
    stable_version: str
    archive_hash: str
    lineage_hash: str
    evidence_refs: list[str]
    rollback_metadata_retained: bool


@dataclass
class IsolatedRestoreRun:
    case_id: str
    stable_version: str
    archive_hash: str
    sandbox_id: str
    restored: bool
    schema_migrations_applied: int
    forbidden_side_effects_seen: int
    missing_evidence_refs: list[str]


@dataclass
class RestoreReplayVerification:
    case_id: str
    stable_version: str
    archive_hash: str
    runtime_fingerprint_match: bool
    tool_contract_match: bool
    smoke_replay_pass_rate: float
    min_smoke_replay_pass_rate: float
    evidence_hash_match: bool
    migration_warnings: int


def decide_restore_drill(
    archive: StableBaselineArchiveReceipt,
    run: IsolatedRestoreRun,
    verification: RestoreReplayVerification,
) -> RestoreDrillDecision:
    same_archive = (
        archive.case_id == run.case_id == verification.case_id
        and archive.stable_version == run.stable_version == verification.stable_version
        and archive.archive_hash == run.archive_hash == verification.archive_hash
    )
    if not same_archive:
        return "manual_review"

    if not archive.rollback_metadata_retained:
        return "refresh_archive"

    if run.forbidden_side_effects_seen > 0:
        return "reopen_baseline_case"

    if not run.restored:
        return "rebuild_restore_path"

    if run.missing_evidence_refs:
        return "refresh_archive"

    if not verification.evidence_hash_match:
        return "refresh_archive"

    if not verification.runtime_fingerprint_match:
        return "rebuild_restore_path"

    if not verification.tool_contract_match:
        return "rebuild_restore_path"

    if verification.smoke_replay_pass_rate < verification.min_smoke_replay_pass_rate:
        return "reopen_baseline_case"

    if verification.migration_warnings > 0 or run.schema_migrations_applied > 0:
        return "refresh_archive"

    return "archive_verified"
~~~

这里的判断顺序很重要：

- 身份对不上，manual_review；
- rollback metadata 不完整，先 refresh archive；
- 演练触发了禁用副作用，直接 reopen case；
- 恢复不起来，修 restore path；
- 证据缺失或 hash 不一致，刷新归档；
- runtime / tool contract 漂移，重建恢复路径；
- smoke replay 不达标，说明 archive 代表的稳定性已经失效。

## pi-mono：RestoreDrillWorker

生产版要把恢复演练做成独立 worker：它只读生产归档，写演练收据，所有恢复动作都在隔离 runner 里执行。

~~~ts
// packages/agent-runtime/src/ops/RestoreDrillWorker.ts
export type RestoreDrillDecision =
  | "archive_verified"
  | "refresh_archive"
  | "rebuild_restore_path"
  | "reopen_baseline_case"
  | "manual_review";

export interface StableBaselineArchiveReceipt {
  caseId: string;
  stableVersion: string;
  archiveHash: string;
  lineageHash: string;
  evidenceRefs: string[];
  rollbackMetadataRetained: boolean;
}

export interface IsolatedRestoreRun {
  caseId: string;
  stableVersion: string;
  archiveHash: string;
  sandboxId: string;
  restored: boolean;
  schemaMigrationsApplied: number;
  forbiddenSideEffectsSeen: number;
  missingEvidenceRefs: string[];
}

export interface RestoreReplayVerification {
  caseId: string;
  stableVersion: string;
  archiveHash: string;
  runtimeFingerprintMatch: boolean;
  toolContractMatch: boolean;
  smokeReplayPassRate: number;
  minSmokeReplayPassRate: number;
  evidenceHashMatch: boolean;
  migrationWarnings: number;
}

export class RestoreDrillWorker {
  constructor(
    private readonly store: BaselineArchiveStore,
    private readonly runner: IsolatedReplayRunner,
    private readonly outbox: OpsOutbox,
  ) {}

  decide(
    archive: StableBaselineArchiveReceipt,
    run: IsolatedRestoreRun,
    verification: RestoreReplayVerification,
  ): RestoreDrillDecision {
    const sameArchive =
      archive.caseId === run.caseId &&
      archive.caseId === verification.caseId &&
      archive.stableVersion === run.stableVersion &&
      archive.stableVersion === verification.stableVersion &&
      archive.archiveHash === run.archiveHash &&
      archive.archiveHash === verification.archiveHash;

    if (!sameArchive) return "manual_review";
    if (!archive.rollbackMetadataRetained) return "refresh_archive";
    if (run.forbiddenSideEffectsSeen > 0) return "reopen_baseline_case";
    if (!run.restored) return "rebuild_restore_path";
    if (run.missingEvidenceRefs.length > 0) return "refresh_archive";
    if (!verification.evidenceHashMatch) return "refresh_archive";
    if (!verification.runtimeFingerprintMatch) return "rebuild_restore_path";
    if (!verification.toolContractMatch) return "rebuild_restore_path";
    if (
      verification.smokeReplayPassRate <
      verification.minSmokeReplayPassRate
    ) {
      return "reopen_baseline_case";
    }
    if (verification.migrationWarnings > 0 || run.schemaMigrationsApplied > 0) {
      return "refresh_archive";
    }
    return "archive_verified";
  }

  async handle(caseId: string) {
    const archive = await this.store.getStableBaselineArchive(caseId);

    const run = await this.runner.restoreBaselineArchive({
      caseId,
      archiveHash: archive.archiveHash,
      mode: "isolated",
      sideEffectPolicy: "deny_all",
    });

    const verification = await this.runner.verifyRestoredBaseline({
      sandboxId: run.sandboxId,
      stableVersion: archive.stableVersion,
      smokeSuite: "baseline-restore-smoke",
    });

    const decision = this.decide(archive, run, verification);

    return this.store.transaction(async (tx) => {
      const receipt = await tx.writeRestoreDrillCloseoutReceipt({
        caseId,
        stableVersion: archive.stableVersion,
        archiveHash: archive.archiveHash,
        sandboxId: run.sandboxId,
        decision,
        checkedAt: new Date().toISOString(),
      });

      if (decision !== "archive_verified") {
        await this.outbox.enqueue({
          type: "baseline_restore_drill_attention_required",
          caseId,
          decision,
          receiptId: receipt.id,
        });
      }

      return receipt;
    });
  }
}
~~~

注意这里没有“直接修复生产 baseline”。RestoreDrillWorker 只产生证据和任务：

- refresh_archive：重新打包归档证据；
- rebuild_restore_path：修恢复脚本、schema migration、runtime adapter；
- reopen_baseline_case：恢复演练说明旧稳定判断不可信，要重新开 case；
- manual_review：身份链不一致，人工复核。

## OpenClaw：Cron 做归档恢复演练

OpenClaw 这类 always-on agent 很适合把恢复演练做成低频 cron，例如每天抽 1 个 stable archive。

~~~yaml
cron:
  name: baseline-archive-restore-drill
  schedule: "0 3 * * *"
  timezone: "UTC"
  payload:
    task: "pick one stable baseline archive and run isolated restore drill"
    constraints:
      side_effect_policy: "deny_all"
      write_receipt: true
      notify_on:
        - refresh_archive
        - rebuild_restore_path
        - reopen_baseline_case
        - manual_review
~~~

实战建议：

- 演练 runner 默认断网，外部工具全部走 replay cassette；
- 所有消息、部署、支付、写库工具都替换成 DryRun adapter；
- 恢复后必须比对 runtime fingerprint，避免“能读旧 JSON，但不能跑旧逻辑”；
- 演练结果要写 receipt，不要只写日志；
- 每次 refresh_archive 都要保留 oldHash -> newHash 的迁移链。

## 常见坑

1. **只校验文件存在**
   文件存在不代表能恢复。至少要跑一次 isolated restore + smoke replay。

2. **演练环境偷用生产工具**
   如果 restore drill 能发真实消息或改真实库，这不是演练，是事故预演。演练环境必须 deny_all side effects。

3. **忽略 schema migration**
   旧 archive 能通过 migration 恢复，不代表长期健康。出现 migration warning 时要 refresh archive，把可恢复状态重新封存。

4. **只归档配置，不归档证据**
   baseline 能恢复但证据不能验证，未来审计时仍然说不清为什么这个 baseline 曾经稳定。

5. **没有关闭收据**
   cron 运行成功不等于演练通过。必须把 decision、sandboxId、archiveHash、verification summary 写成 closeout receipt。

## 关键 takeaway

稳定归档的质量，不看“存了多少”，看“多久没演练仍然能恢复”。

工程上要记住三条：

- archive 是恢复资产，不是日志垃圾桶；
- restore drill 必须隔离副作用；
- 每次恢复演练都要留下 closeout receipt。

第 495 课把稳定 baseline 干净归档，第 496 课证明这些归档未来真的能救命。这样 Agent 的事故恢复链路才不是“希望能回滚”，而是“定期证明能回滚”。
