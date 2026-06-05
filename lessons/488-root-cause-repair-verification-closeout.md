# Agent 根因修复验证与回归套件解除阻塞闸门

> Root-Cause Repair Verification & Regression Suite Unblock Gate

第 487 课讲了 Root-Cause Case 的 owner 路由和修复 SLA：长期回归套件失败聚类以后，不能只知道“谁坏了”，还要把 case 派给正确 owner，绑定 ack、repair deadline 和升级路径。

但 owner 说“修好了”还不是闭环。真正危险的是：补丁合了，失败 seed 暂时绿了，但 release blocking lane 仍然带着旧的 freeze、quarantine、override 或 stale failure cluster。结果要么系统迟迟不能发布，要么更糟，手动解除阻塞后同类失败又回来。

所以第 488 课补上 **Root-Cause Repair Verification & Regression Suite Unblock Gate**：根因修复完成后，必须把修复补丁、影响 seed、重放结果、发布阻塞状态和关闭收据串起来，证明“这个根因 case 可以关闭，并且长期回归套件可以解除对应阻塞”。

## 核心模型

把关闭流程拆成 5 个对象：

~~~text
RootCauseCase
  -> RepairPatchReceipt
  -> RegressionReplayVerification
  -> SuiteUnblockReadinessReview
  -> RootCauseCloseoutReceipt
~~~

分工：

- RootCauseCase：来自上一课的根因案件，记录 cluster、owner、SLA 和 release blocking 范围；
- RepairPatchReceipt：证明修复已经落地，包含 commit、变更范围、测试证据和风险面；
- RegressionReplayVerification：对 affected seeds、adjacent seeds 和 smoke seeds 做重放验证；
- SuiteUnblockReadinessReview：判断能不能解除 quarantine、freeze 或 manual override；
- RootCauseCloseoutReceipt：最终收据，说明 close_case、extend_replay、keep_blocked、reopen_case 还是 manual_review。

一句话：**根因修复的终点不是 PR merge，而是失败 cluster 被证据关闭，发布阻塞被精确解除。**

## learn-claude-code：解除阻塞判定纯函数

教学版先写一个纯函数。它只关心输入证据是否足够，不碰数据库、不发通知，方便单测覆盖各种边界。

~~~python
# learn_claude_code/root_cause_unblock_gate.py
from dataclasses import dataclass
from typing import Literal

BlockerKind = Literal["release_freeze", "seed_quarantine", "manual_override"]
CloseoutDecision = Literal[
    "close_case_unblock_suite",
    "extend_replay",
    "keep_blocked",
    "reopen_case",
    "manual_review",
]


@dataclass
class RootCauseCase:
    case_id: str
    cluster_id: str
    owner: str
    blocker_kind: BlockerKind
    release_blocking: bool
    affected_seed_ids: list[str]
    adjacent_seed_ids: list[str]
    repeated_failure_count: int


@dataclass
class RepairPatchReceipt:
    case_id: str
    commit_sha: str | None
    changed_risk_surfaces: list[str]
    owner_ack: bool
    unit_tests_passed: bool
    migration_required: bool
    migration_verified: bool
    rollback_plan_ref: str | None


@dataclass
class RegressionReplayVerification:
    case_id: str
    affected_seed_pass_rate: float
    adjacent_seed_pass_rate: float
    smoke_seed_pass_rate: float
    flake_rate: float
    runtime_contract_hash_changed: bool
    side_effect_mismatches: int
    replay_runs: int


def decide_root_cause_closeout(
    case: RootCauseCase,
    patch: RepairPatchReceipt,
    verification: RegressionReplayVerification,
) -> CloseoutDecision:
    if case.case_id != patch.case_id or case.case_id != verification.case_id:
        return "manual_review"

    if not patch.commit_sha or not patch.owner_ack:
        return "keep_blocked"

    if not patch.unit_tests_passed:
        return "reopen_case"

    if patch.migration_required and not patch.migration_verified:
        return "keep_blocked"

    if not patch.rollback_plan_ref and case.release_blocking:
        return "manual_review"

    expected_runs = max(10, len(case.affected_seed_ids) + len(case.adjacent_seed_ids))
    if verification.replay_runs < expected_runs:
        return "extend_replay"

    if verification.side_effect_mismatches > 0:
        return "reopen_case"

    if verification.runtime_contract_hash_changed:
        return "extend_replay"

    if verification.affected_seed_pass_rate < 1.0:
        return "reopen_case"

    if verification.adjacent_seed_pass_rate < 0.98:
        return "extend_replay"

    if verification.smoke_seed_pass_rate < 0.99:
        return "keep_blocked"

    if verification.flake_rate > 0.02:
        return "extend_replay"

    if case.repeated_failure_count >= 3 and len(patch.changed_risk_surfaces) == 0:
        return "manual_review"

    return "close_case_unblock_suite"
~~~

这里有几个关键点：

- case/patch/verification 必须同一个 case_id，避免拿错证据关闭案件；
- affected seed 必须 100% 通过，因为它们就是本次 root cause 的直接证据；
- adjacent seed 可以给一点点容忍，但不通过时只能 extend replay，不能直接 close；
- runtime contract 变了要延长验证，因为你不知道是修复生效，还是 runner 语义换了；
- release blocking case 必须有 rollback plan，不然解除阻塞后无法快速回退。

## pi-mono：Closeout Worker + 事务化 unblock

生产版不要让修复 worker 自己解除发布阻塞。更稳的做法是让 `RootCauseCloseoutWorker` 订阅 repair 和 replay 事件，然后在一个事务里同时写 closeout receipt、更新 case、释放 blocker。

~~~ts
// packages/agent-runtime/src/regression/RootCauseCloseoutWorker.ts
export type BlockerKind = "release_freeze" | "seed_quarantine" | "manual_override";

export type CloseoutDecision =
  | "close_case_unblock_suite"
  | "extend_replay"
  | "keep_blocked"
  | "reopen_case"
  | "manual_review";

export interface RootCauseCase {
  caseId: string;
  clusterId: string;
  owner: string;
  blockerKind: BlockerKind;
  releaseBlocking: boolean;
  affectedSeedIds: string[];
  adjacentSeedIds: string[];
  repeatedFailureCount: number;
}

export interface RepairPatchReceipt {
  caseId: string;
  commitSha?: string;
  changedRiskSurfaces: string[];
  ownerAck: boolean;
  unitTestsPassed: boolean;
  migrationRequired: boolean;
  migrationVerified: boolean;
  rollbackPlanRef?: string;
}

export interface RegressionReplayVerification {
  caseId: string;
  affectedSeedPassRate: number;
  adjacentSeedPassRate: number;
  smokeSeedPassRate: number;
  flakeRate: number;
  runtimeContractHashChanged: boolean;
  sideEffectMismatches: number;
  replayRuns: number;
}

export interface RootCauseCloseoutReceipt {
  receiptId: string;
  caseId: string;
  clusterId: string;
  decision: CloseoutDecision;
  releasedBlocker?: BlockerKind;
  commitSha?: string;
  reason: string;
  createdAt: string;
}

export class RootCauseCloseoutWorker {
  constructor(private readonly store: RootCauseStore) {}

  decide(
    rootCauseCase: RootCauseCase,
    patch: RepairPatchReceipt,
    verification: RegressionReplayVerification,
  ): CloseoutDecision {
    if (
      rootCauseCase.caseId !== patch.caseId ||
      rootCauseCase.caseId !== verification.caseId
    ) {
      return "manual_review";
    }

    if (!patch.commitSha || !patch.ownerAck) return "keep_blocked";
    if (!patch.unitTestsPassed) return "reopen_case";
    if (patch.migrationRequired && !patch.migrationVerified) return "keep_blocked";
    if (!patch.rollbackPlanRef && rootCauseCase.releaseBlocking) {
      return "manual_review";
    }

    const expectedRuns = Math.max(
      10,
      rootCauseCase.affectedSeedIds.length + rootCauseCase.adjacentSeedIds.length,
    );

    if (verification.replayRuns < expectedRuns) return "extend_replay";
    if (verification.sideEffectMismatches > 0) return "reopen_case";
    if (verification.runtimeContractHashChanged) return "extend_replay";
    if (verification.affectedSeedPassRate < 1.0) return "reopen_case";
    if (verification.adjacentSeedPassRate < 0.98) return "extend_replay";
    if (verification.smokeSeedPassRate < 0.99) return "keep_blocked";
    if (verification.flakeRate > 0.02) return "extend_replay";

    if (
      rootCauseCase.repeatedFailureCount >= 3 &&
      patch.changedRiskSurfaces.length === 0
    ) {
      return "manual_review";
    }

    return "close_case_unblock_suite";
  }

  async closeout(caseId: string): Promise<RootCauseCloseoutReceipt> {
    const rootCauseCase = await this.store.getCase(caseId);
    const patch = await this.store.getLatestPatchReceipt(caseId);
    const verification = await this.store.getLatestReplayVerification(caseId);
    const decision = this.decide(rootCauseCase, patch, verification);

    return this.store.transaction(async (tx) => {
      const receipt: RootCauseCloseoutReceipt = {
        receiptId: `root_cause_closeout_${caseId}_${Date.now()}`,
        caseId,
        clusterId: rootCauseCase.clusterId,
        decision,
        releasedBlocker:
          decision === "close_case_unblock_suite"
            ? rootCauseCase.blockerKind
            : undefined,
        commitSha: patch.commitSha,
        reason: this.reason(decision),
        createdAt: new Date().toISOString(),
      };

      await tx.writeCloseoutReceipt(receipt);

      if (decision === "close_case_unblock_suite") {
        await tx.markCaseClosed(caseId, receipt.receiptId);
        await tx.releaseRegressionBlocker(rootCauseCase.clusterId, rootCauseCase.blockerKind);
        await tx.recordSuiteUnblock(rootCauseCase.affectedSeedIds, receipt.receiptId);
      } else if (decision === "reopen_case") {
        await tx.reopenCase(caseId, receipt.receiptId);
      } else if (decision === "extend_replay") {
        await tx.enqueueAdditionalReplay(caseId, rootCauseCase.adjacentSeedIds);
      }

      return receipt;
    });
  }

  private reason(decision: CloseoutDecision): string {
    return {
      close_case_unblock_suite: "repair verified and blocker can be released",
      extend_replay: "verification needs more replay coverage",
      keep_blocked: "repair evidence is incomplete",
      reopen_case: "repair failed verification",
      manual_review: "evidence is inconsistent or high risk",
    }[decision];
  }
}
~~~

这里的工程重点是事务化：

- 写 closeout receipt 和 release blocker 必须一起成功；
- 如果 decision 是 extend_replay，只追加验证任务，不动 blocker；
- 如果 decision 是 reopen_case，case 回到修复队列，保留失败验证证据；
- `recordSuiteUnblock` 绑定 seed 和 receipt，后续审计能查到“为什么这个阻塞被解除”。

## OpenClaw：课程 Cron 的类比

这个课程 Cron 其实也有类似闭环：

~~~text
lesson file created
  -> README updated
  -> TOOLS taught-list updated
  -> git commit pushed
  -> Telegram message sent
  -> final report
~~~

如果只写了 lesson 文件，却没更新 README，读者找不到目录；如果只发了 Telegram，却没 git push，课程资产丢在本机；如果只 commit 没更新 TOOLS，下次 cron 可能重复选题。

所以 OpenClaw 的自动课程任务也应该有 closeout receipt：

~~~json
{
  "job": "agent-course-cron",
  "lesson": "488-root-cause-repair-verification-closeout",
  "artifacts": [
    "lessons/488-root-cause-repair-verification-closeout.md",
    "README.md",
    "../TOOLS.md"
  ],
  "checks": {
    "readmeUpdated": true,
    "taughtListUpdated": true,
    "gitPushed": true,
    "telegramSent": true
  },
  "decision": "close_job"
}
~~~

这和回归套件 unblock 是同一个原则：**完成动作不等于完成任务，必须把证据、索引、副作用和关闭状态对齐。**

## 实战清单

给长期回归套件加 root cause closeout，可以按 8 步落地：

1. 每个 failure cluster 创建 RootCauseCase，并记录 release blocker；
2. 修复 PR merge 后只写 RepairPatchReceipt，不直接关闭 case；
3. replay runner 生成 RegressionReplayVerification，覆盖 affected、adjacent、smoke 三类 seed；
4. Closeout Worker 用纯函数输出 close_case、extend_replay、keep_blocked、reopen 或 manual_review；
5. close_case 时在同一事务里写 receipt、关 case、释放 blocker、更新 suite index；
6. extend_replay 时补跑 adjacent seeds，不提前放行；
7. reopen_case 时把失败验证证据带回修复队列；
8. 每次 unblock 都保留 receipt_id，方便发布审计追溯。

## 小结

根因修复不是“owner 修了一个 bug”，而是一个带证据的关闭流程：

- RootCauseCase 定义失败范围；
- RepairPatchReceipt 证明修复落地；
- RegressionReplayVerification 证明失败不再复现；
- SuiteUnblockReadinessReview 决定能否解除阻塞；
- RootCauseCloseoutReceipt 让关闭和 unblock 可审计、可回放。

> 成熟 Agent 的长期回归系统，不会因为 PR merge 就放行发布；它会等根因修复、重放验证、阻塞释放和关闭收据都对齐以后，才把发布通道重新打开。
