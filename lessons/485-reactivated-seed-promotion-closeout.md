# Agent 再激活种子的晋级关闭与长期套件接回闸门

> Reactivated Seed Promotion Closeout & Long-term Suite Re-entry Gate

第 484 课讲了 burn-in 失败后的修复回路：seed 失败后不能一句“已修复”就回归长期套件，而是要经过 triage、repair plan、verification report 和短期 ReactivationPermit，先重新进入 burn-in lane。

但这里还有最后一公里：重新 burn-in 通过，也不代表可以直接塞回 blocking lane。因为修复后的 seed 可能只是在短窗口里看起来稳定，长期运行仍然可能有成本漂移、断言信号下降、runner 版本不一致、owner 未接回等问题。

所以第 485 课补上 **Reactivated Seed Promotion Closeout & Long-term Suite Re-entry Gate**：再激活 seed 必须用 burn-in 结果、permit 消耗记录、长期套件接回计划和关闭收据证明自己可以回到长期 blocking lane。

## 核心模型

把再激活后的接回流程拆成 5 个对象：

~~~text
ReactivationPermit
  -> ReactivatedBurnInRunSummary
  -> LongTermReentryPlan
  -> SuiteReentryReadinessReview
  -> ReentryCloseoutReceipt
~~~

分工：

- ReactivationPermit：上一课签发的短期许可，只允许 seed 回 burn-in；
- ReactivatedBurnInRunSummary：记录再激活后的真实 runner 结果；
- LongTermReentryPlan：声明要接回哪个 suite、blocking 级别、owner 和退役条件；
- SuiteReentryReadinessReview：判断是否满足长期套件准入；
- ReentryCloseoutReceipt：最终收据，说明 seed 是 promoted、compressed、kept_in_burn_in、quarantined 还是 manual_review。

一句话：**再激活通过 burn-in 只是“可以被重新考虑”，不是“自动恢复长期 blocking 权力”。**

## learn-claude-code：接回判定纯函数

教学版先写一个纯函数。它把 permit、burn-in summary 和 reentry plan 合在一起，输出是否允许接回长期套件。

~~~python
# learn_claude_code/reactivated_seed_reentry_gate.py
from dataclasses import dataclass
from typing import Literal

SuiteLane = Literal["nightly", "release_blocking", "post_deploy_sentinel"]
ReentryDecision = Literal[
    "promote_long_term",
    "promote_compressed",
    "keep_burn_in",
    "quarantine_seed",
    "manual_review",
]


@dataclass
class ReactivationPermit:
    permit_id: str
    seed_id: str
    lane: Literal["burn_in"]
    max_runs: int
    consumed_runs: int
    expires_at_epoch_ms: int


@dataclass
class ReactivatedBurnInRunSummary:
    seed_id: str
    runs: int
    pass_rate: float
    flake_rate: float
    assertion_signal_rate: float
    p95_duration_ms: int
    average_cost_cents: int
    runner_contract_hash: str
    side_effect_mismatches: int
    completed_at_epoch_ms: int


@dataclass
class LongTermReentryPlan:
    seed_id: str
    target_lane: SuiteLane
    expected_runner_contract_hash: str
    owner: str | None
    retirement_condition: str | None
    allow_compressed_entry: bool


def decide_reentry(
    permit: ReactivationPermit,
    summary: ReactivatedBurnInRunSummary,
    plan: LongTermReentryPlan,
) -> ReentryDecision:
    if permit.seed_id != summary.seed_id or summary.seed_id != plan.seed_id:
        return "manual_review"

    if permit.lane != "burn_in":
        return "manual_review"

    if summary.completed_at_epoch_ms > permit.expires_at_epoch_ms:
        return "keep_burn_in"

    if summary.runs < min(permit.max_runs, 10):
        return "keep_burn_in"

    if permit.consumed_runs > permit.max_runs:
        return "manual_review"

    if not plan.owner or not plan.retirement_condition:
        return "manual_review"

    if summary.runner_contract_hash != plan.expected_runner_contract_hash:
        return "keep_burn_in"

    if summary.side_effect_mismatches > 0:
        return "quarantine_seed"

    if summary.pass_rate < 0.98:
        return "keep_burn_in"

    if summary.flake_rate > 0.02:
        return "keep_burn_in"

    if summary.assertion_signal_rate < 0.92:
        return "keep_burn_in"

    if summary.p95_duration_ms > 45_000 or summary.average_cost_cents > 150:
        if plan.allow_compressed_entry and summary.assertion_signal_rate >= 0.95:
            return "promote_compressed"
        return "keep_burn_in"

    return "promote_long_term"
~~~

这里的重点是四个防线：

- permit 必须没有越权：只能来自 burn_in lane，不能拿别的许可混用；
- 样本数必须够：短跑几次通过不算长期稳定；
- runner contract 必须一致：避免本地修好了，真实 blocking runner 仍然不匹配；
- owner 和 retirement condition 必须存在：长期套件不是垃圾桶，每个 seed 都要有人负责、知道什么时候退役。

## pi-mono：Re-entry Worker + 事务化接回

生产版建议由 worker 统一消费 burn-in summary。它不只改一个状态，而是同时写 suite index、关闭 permit、记录 closeout receipt，避免 seed 半接回。

~~~ts
// packages/agent-runtime/src/regression/ReactivatedSeedReentryWorker.ts
export type SuiteLane = "nightly" | "release_blocking" | "post_deploy_sentinel";

export type ReentryDecision =
  | "promote_long_term"
  | "promote_compressed"
  | "keep_burn_in"
  | "quarantine_seed"
  | "manual_review";

export interface ReactivationPermit {
  permitId: string;
  seedId: string;
  lane: "burn_in";
  maxRuns: number;
  consumedRuns: number;
  expiresAtEpochMs: number;
}

export interface ReactivatedBurnInRunSummary {
  seedId: string;
  runs: number;
  passRate: number;
  flakeRate: number;
  assertionSignalRate: number;
  p95DurationMs: number;
  averageCostCents: number;
  runnerContractHash: string;
  sideEffectMismatches: number;
  completedAtEpochMs: number;
}

export interface LongTermReentryPlan {
  seedId: string;
  targetLane: SuiteLane;
  expectedRunnerContractHash: string;
  owner?: string;
  retirementCondition?: string;
  allowCompressedEntry: boolean;
}

export interface ReentryCloseoutReceipt {
  receiptId: string;
  seedId: string;
  decision: ReentryDecision;
  targetLane?: SuiteLane;
  suiteMode?: "full" | "compressed";
  reason: string;
  createdAt: string;
}

export class ReactivatedSeedReentryWorker {
  constructor(private readonly store: RegressionSeedStore) {}

  decide(
    permit: ReactivationPermit,
    summary: ReactivatedBurnInRunSummary,
    plan: LongTermReentryPlan,
  ): ReentryDecision {
    if (permit.seedId !== summary.seedId || summary.seedId !== plan.seedId) {
      return "manual_review";
    }

    if (permit.lane !== "burn_in") return "manual_review";
    if (summary.completedAtEpochMs > permit.expiresAtEpochMs) return "keep_burn_in";
    if (summary.runs < Math.min(permit.maxRuns, 10)) return "keep_burn_in";
    if (permit.consumedRuns > permit.maxRuns) return "manual_review";
    if (!plan.owner || !plan.retirementCondition) return "manual_review";
    if (summary.runnerContractHash !== plan.expectedRunnerContractHash) {
      return "keep_burn_in";
    }
    if (summary.sideEffectMismatches > 0) return "quarantine_seed";
    if (summary.passRate < 0.98) return "keep_burn_in";
    if (summary.flakeRate > 0.02) return "keep_burn_in";
    if (summary.assertionSignalRate < 0.92) return "keep_burn_in";

    const tooExpensive =
      summary.p95DurationMs > 45_000 || summary.averageCostCents > 150;
    if (tooExpensive) {
      return plan.allowCompressedEntry && summary.assertionSignalRate >= 0.95
        ? "promote_compressed"
        : "keep_burn_in";
    }

    return "promote_long_term";
  }

  async closeout(seedId: string): Promise<ReentryCloseoutReceipt> {
    const [permit, summary, plan] = await Promise.all([
      this.store.getActiveReactivationPermit(seedId),
      this.store.getLatestReactivatedBurnInSummary(seedId),
      this.store.getLongTermReentryPlan(seedId),
    ]);

    const decision = this.decide(permit, summary, plan);

    return this.store.transaction(async (tx) => {
      if (decision === "promote_long_term" || decision === "promote_compressed") {
        await tx.upsertSuiteIndex({
          seedId,
          lane: plan.targetLane,
          mode: decision === "promote_compressed" ? "compressed" : "full",
          owner: plan.owner!,
          retirementCondition: plan.retirementCondition!,
        });
        await tx.closePermit(permit.permitId, "consumed");
      }

      if (decision === "quarantine_seed") {
        await tx.moveSeedToQuarantine(seedId, "side_effect_mismatch_after_reactivation");
        await tx.closePermit(permit.permitId, "quarantined");
      }

      if (decision === "keep_burn_in") {
        await tx.extendBurnIn(seedId, { reason: "reentry_not_ready" });
      }

      return tx.writeReentryCloseoutReceipt({
        receiptId: crypto.randomUUID(),
        seedId,
        decision,
        targetLane: plan.targetLane,
        suiteMode:
          decision === "promote_compressed"
            ? "compressed"
            : decision === "promote_long_term"
              ? "full"
              : undefined,
        reason: `reactivated seed reentry decision: ${decision}`,
        createdAt: new Date().toISOString(),
      });
    });
  }
}
~~~

生产实现有两个关键点：

- `closePermit` 和 `upsertSuiteIndex` 必须在同一个事务里，防止 permit 已消费但 suite 没写进去；
- `quarantine_seed` 要关闭 permit，防止同一个有副作用 mismatch 的 seed 被重复 reentry。

## OpenClaw：课程 Cron 的类比

这个课程 cron 也有类似问题：

1. 上一轮生成 lesson，相当于 seed 通过了一次 burn-in；
2. `README.md` 目录更新，相当于 suite index 接回；
3. `TOOLS.md` 已讲内容更新，相当于 owner/retirement metadata；
4. Telegram 发群，相当于真实 side effect；
5. git commit + push，相当于 closeout receipt。

如果只写了 lesson 文件，没更新 README，就像 seed 通过了 burn-in 但没进入 suite index；如果发了 Telegram 但 git push 失败，就像真实副作用发生了但没有关闭收据。成熟的自动化要把这些动作串成一个可对账的 closeout。

## 实战清单

做 reactivated seed re-entry 时，至少检查：

- permit 是否仍有效，且没有超出 maxRuns；
- burn-in 是否跑够样本数，而不是只跑 1-2 次；
- pass rate、flake rate、assertion signal 是否同时达标；
- runner contract hash 是否和目标长期套件一致；
- 是否存在 side-effect mismatch；
- seed 是否有 owner、target lane 和 retirement condition；
- 接回 suite index、关闭 permit、写 closeout receipt 是否事务化；
- 成本过高时，是否能压缩进入，而不是简单丢弃有价值 seed。

## 小结

第 484 课解决“失败 seed 怎么修完再回来”，第 485 课解决“回来以后怎么重新拿到长期 blocking 权力”。

成熟 Agent 的回归套件不是只会增加测试，而是让每个 seed 都经过：候选、验证、燃尽、修复、再激活、接回、维护、退役。每一步都有证据，每一次状态变化都有收据。
