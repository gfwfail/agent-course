# Agent 燃尽失败后的修复回路与再激活闸门

> Burn-in Failure Repair Loop & Reactivation Gate

第 483 课讲了补齐 seed 的首轮燃尽：新 seed 先进入 burn-in lane，用真实 runner 的 flake、duration、cost、assertion signal、owner ack 和首次失败 triage 来决定 promote、quarantine、compress、repair 或 manual_review。

但 burn-in 失败以后还有一个坑：很多团队会把 seed 修一下就重新打开，结果同一个 flaky / 低信号 / 高成本问题反复回到 nightly，慢慢把 release gate 变成噪音源。

所以第 484 课补上 **Burn-in Failure Repair Loop & Reactivation Gate**：燃尽失败的 seed 必须带着失败分类、修复计划、验证证据和再激活许可回到 burn-in，不能靠一句“已修复”直接进入长期套件。

## 核心模型

把失败后的回路拆成 5 个对象：

~~~text
BurnInCloseoutReceipt
  -> SeedFailureTriage
  -> SeedRepairPlan
  -> RepairVerificationReport
  -> ReactivationPermit
~~~

分工：

- BurnInCloseoutReceipt：上一课的关闭收据，说明 seed 为什么失败；
- SeedFailureTriage：把失败分类成 flake、low_signal、runner_mismatch、high_cost、external_dependency 或 assertion_bug；
- SeedRepairPlan：声明修复动作，比如 stabilize_clock、mock_external_call、tighten_assertion、compress_trace、update_runner_contract；
- RepairVerificationReport：离线和真实 runner 的验证结果，证明修复有效；
- ReactivationPermit：短期许可，只允许 seed 回到 burn-in lane，不允许跳过燃尽直接进 blocking lane。

一句话：**燃尽失败不是修完就上线，而是先证明“修的是正确原因”，再带许可重新燃尽。**

## learn-claude-code：再激活判定纯函数

教学版先写一个纯函数。输入失败分类、修复计划和验证报告，输出是否允许重新进入 burn-in。

~~~python
# learn_claude_code/burn_in_reactivation_gate.py
from dataclasses import dataclass
from typing import Literal

FailureKind = Literal[
    "flake",
    "low_signal",
    "runner_mismatch",
    "high_cost",
    "external_dependency",
    "assertion_bug",
]

RepairAction = Literal[
    "stabilize_clock",
    "mock_external_call",
    "tighten_assertion",
    "compress_trace",
    "update_runner_contract",
]

ReactivationDecision = Literal[
    "reactivate_burn_in",
    "continue_repair",
    "quarantine_seed",
    "reject_seed",
    "manual_review",
]


@dataclass
class SeedFailureTriage:
    seed_id: str
    failure_kind: FailureKind
    repeated_failures: int
    criticality: Literal["low", "medium", "high", "critical"]
    owner_acknowledged_root_cause: bool


@dataclass
class SeedRepairPlan:
    seed_id: str
    actions: list[RepairAction]
    expected_failure_kind: FailureKind
    expires_at_epoch_ms: int


@dataclass
class RepairVerificationReport:
    seed_id: str
    offline_replay_passed: bool
    runner_replay_passed: bool
    flake_rate_after_repair: float
    assertion_signal_rate_after_repair: float
    p95_duration_ms_after_repair: int
    average_cost_cents_after_repair: int
    runner_contract_matched: bool
    verified_at_epoch_ms: int


def decide_reactivation(
    triage: SeedFailureTriage,
    plan: SeedRepairPlan,
    report: RepairVerificationReport,
) -> ReactivationDecision:
    if triage.seed_id != plan.seed_id or plan.seed_id != report.seed_id:
        return "manual_review"

    if plan.expected_failure_kind != triage.failure_kind:
        return "manual_review"

    if report.verified_at_epoch_ms > plan.expires_at_epoch_ms:
        return "continue_repair"

    if not triage.owner_acknowledged_root_cause:
        return "manual_review"

    if triage.repeated_failures >= 3 and triage.criticality in ("high", "critical"):
        return "quarantine_seed"

    if not report.offline_replay_passed or not report.runner_replay_passed:
        return "continue_repair"

    if not report.runner_contract_matched:
        return "continue_repair"

    if report.flake_rate_after_repair > 0.05:
        return "continue_repair"

    if report.assertion_signal_rate_after_repair < 0.9:
        return "continue_repair"

    if report.p95_duration_ms_after_repair > 30_000:
        return "continue_repair"

    if report.average_cost_cents_after_repair > 100:
        return "continue_repair"

    return "reactivate_burn_in"
~~~

这里最重要的是防止“错因修复”：

- triage 和 repair plan 的 failure kind 必须一致；
- repair plan 过期后不能拿旧验证报告复用；
- owner 必须确认 root cause，避免无人负责的 seed 反复回流；
- 高风险 seed 连续失败 3 次，直接 quarantine，别继续污染 runner；
- offline replay 和 runner replay 都要过，说明本地修复和真实环境一致；
- 再激活只回 burn-in，不直接进长期 blocking lane。

## pi-mono：Repair Worker + Reactivation Permit

生产版建议把“修复”和“再激活”拆成事务。Worker 只在证据满足时写短期 permit，并把 seed lane 从 repair_required 移到 burn_in。

~~~ts
// packages/agent-runtime/src/regression/BurnInRepairReactivationWorker.ts
export type FailureKind =
  | "flake"
  | "low_signal"
  | "runner_mismatch"
  | "high_cost"
  | "external_dependency"
  | "assertion_bug";

export type ReactivationDecision =
  | "reactivate_burn_in"
  | "continue_repair"
  | "quarantine_seed"
  | "reject_seed"
  | "manual_review";

export interface SeedFailureTriage {
  seedId: string;
  failureKind: FailureKind;
  repeatedFailures: number;
  criticality: "low" | "medium" | "high" | "critical";
  ownerAcknowledgedRootCause: boolean;
}

export interface SeedRepairPlan {
  seedId: string;
  actions: Array<
    | "stabilize_clock"
    | "mock_external_call"
    | "tighten_assertion"
    | "compress_trace"
    | "update_runner_contract"
  >;
  expectedFailureKind: FailureKind;
  expiresAtEpochMs: number;
}

export interface RepairVerificationReport {
  seedId: string;
  offlineReplayPassed: boolean;
  runnerReplayPassed: boolean;
  flakeRateAfterRepair: number;
  assertionSignalRateAfterRepair: number;
  p95DurationMsAfterRepair: number;
  averageCostCentsAfterRepair: number;
  runnerContractMatched: boolean;
  verifiedAtEpochMs: number;
}

export interface ReactivationPermit {
  permitId: string;
  seedId: string;
  lane: "burn_in";
  maxRuns: number;
  expiresAt: string;
  reason: string;
}

export class BurnInRepairReactivationWorker {
  constructor(private readonly store: RegressionSeedStore) {}

  decide(
    triage: SeedFailureTriage,
    plan: SeedRepairPlan,
    report: RepairVerificationReport,
  ): ReactivationDecision {
    if (triage.seedId !== plan.seedId || plan.seedId !== report.seedId) {
      return "manual_review";
    }

    if (plan.expectedFailureKind !== triage.failureKind) {
      return "manual_review";
    }

    if (report.verifiedAtEpochMs > plan.expiresAtEpochMs) {
      return "continue_repair";
    }

    if (!triage.ownerAcknowledgedRootCause) return "manual_review";

    if (
      triage.repeatedFailures >= 3 &&
      (triage.criticality === "high" || triage.criticality === "critical")
    ) {
      return "quarantine_seed";
    }

    if (!report.offlineReplayPassed || !report.runnerReplayPassed) {
      return "continue_repair";
    }

    if (!report.runnerContractMatched) return "continue_repair";
    if (report.flakeRateAfterRepair > 0.05) return "continue_repair";
    if (report.assertionSignalRateAfterRepair < 0.9) return "continue_repair";
    if (report.p95DurationMsAfterRepair > 30_000) return "continue_repair";
    if (report.averageCostCentsAfterRepair > 100) return "continue_repair";

    return "reactivate_burn_in";
  }

  async closeRepair(seedId: string): Promise<ReactivationPermit | null> {
    const triage = await this.store.loadFailureTriage(seedId);
    const plan = await this.store.loadRepairPlan(seedId);
    const report = await this.store.loadRepairVerification(seedId);
    const decision = this.decide(triage, plan, report);

    return this.store.transaction(async (tx) => {
      await tx.appendRepairDecision(seedId, {
        decision,
        failureKind: triage.failureKind,
        actions: plan.actions,
        verifiedAtEpochMs: report.verifiedAtEpochMs,
      });

      if (decision === "quarantine_seed") {
        await tx.moveSeedLane(seedId, "repair_required", "quarantine");
        await tx.openRepairTicket(seedId, {
          reason: "repeated_burn_in_failure",
          repeatedFailures: triage.repeatedFailures,
        });
        return null;
      }

      if (decision !== "reactivate_burn_in") {
        await tx.moveSeedLane(seedId, "repair_required", "repair_required");
        return null;
      }

      const permit = await tx.createReactivationPermit(seedId, {
        lane: "burn_in",
        maxRuns: 3,
        ttlMs: 7 * 24 * 60 * 60 * 1000,
        reason: `repair_verified:${triage.failureKind}`,
      });

      await tx.moveSeedLane(seedId, "repair_required", "burn_in");
      await tx.enqueueBurnIn(seedId, { permitId: permit.permitId });

      return permit;
    });
  }
}
~~~

几个工程细节：

- permit 有 TTL 和 maxRuns，防止修复许可被长期复用；
- lane 转换和 permit 创建在同一个事务里，避免 seed 半激活；
- repeatedFailures 进入 triage，不让同一个 seed 无限修复；
- appendRepairDecision 形成审计链，后面 drift / retirement 可以追溯；
- enqueueBurnIn 必须带 permitId，runner 只消费有许可的回流 seed。

## OpenClaw：课程 Cron 的实战类比

这个课程 Cron 本身也适合套这个模型：

- 如果 Telegram 发送失败，先分类是 message tool、网络、权限还是内容过长；
- 修复计划要对应失败原因，比如拆消息、换目标、刷新 session 或缩短内容；
- 修复后先发一条短验证消息或 dry-run，不要直接把大段课程重复发 3 次；
- 再继续写 lesson / README / TOOLS / git push，确保外部副作用和 repo 状态一致；
- 如果同类失败连续出现，应该 quarantine 这个自动发布链路，让人工检查凭证或平台限制。

所以成熟 Agent 的“重试”不是 while True，而是：

~~~text
失败分类 -> 修复计划 -> 验证报告 -> 短期许可 -> 回到观察 lane
~~~

## 落地建议

1. 所有 burn-in failure 都必须先写 SeedFailureTriage，不允许直接 reopen seed。
2. RepairPlan 的 expectedFailureKind 要和 triage 对齐，避免“看起来修了，实际修错”。
3. ReactivationPermit 只能回 burn-in lane，不能直接 promote 到长期套件。
4. repeatedFailures 达到阈值后 quarantine，不要让高风险 seed 无限占用 runner。
5. 把 permitId 写入 runner event，方便后续定位“这次运行为什么被允许”。

成熟 Agent 的测试资产不是越多越好，而是每个失败过的资产都有修复证据、回流边界和退出条件。
