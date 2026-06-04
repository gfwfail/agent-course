# Agent 长期回归套件的漂移维护与退役闸门

> Long-term Regression Suite Drift Maintenance & Retirement Gate

第 479 课讲了 regression seed 如何从事故样本晋级成长期套件：有 owner、有成本预算、有退役条件，才能从一次性 smoke replay 变成长期传感器。

今天继续往后讲：**seed 进了长期套件以后，也不能永久不管。成熟 Agent 要定期检查长期回归套件是否还在测真实风险，而不是变成 CI 里的历史包袱。**

最常见的坏味道有 4 个：

- seed 长期 green，但真实风险面已经被新护栏覆盖，继续跑只是浪费；
- seed 因依赖 API、模型版本、工具 schema 漂移变成 flaky，CI 假红越来越多；
- seed 的期望输出还停留在旧行为，开始阻止合理重构；
- seed 没有 owner，失败后没人知道该修测试、修产品还是修 Agent 策略。

所以长期套件要有一个明确的 **RegressionSuiteMaintenanceGate**：周期性复核每个 seed 的覆盖价值、稳定性、成本和退役条件，最后决定 keep、refresh、compress、quarantine 或 retire。

## 核心模型

把长期套件维护拆成 5 个对象：

~~~text
SuitePromotionReceipt
  -> LongTermSuiteRunSummary
  -> SeedDriftReview
  -> RetirementReadinessReview
  -> SuiteMaintenanceReceipt
~~~

分工要清楚：

- SuitePromotionReceipt 说明 seed 当初为什么进入长期套件；
- LongTermSuiteRunSummary 汇总最近一段时间的 pass rate、flaky rate、成本、耗时和失败分布；
- SeedDriftReview 判断 seed 是否因为 schema、runtime、tool、model 或业务行为漂移而失真；
- RetirementReadinessReview 判断退役条件是否真的满足，是否已有替代覆盖；
- SuiteMaintenanceReceipt 写下维护结果，后续 CI、nightly 和 release gate 都按这个 receipt 调整。

关键原则：**回归测试不是越多越安全，而是每个长期 seed 都要持续证明自己还值得运行。**

## learn-claude-code：维护判定纯函数

教学版先写纯函数：输入一个长期 seed 的运行摘要和漂移检查结果，输出维护动作。

~~~python
# learn_claude_code/regression_suite_maintenance_gate.py
from dataclasses import dataclass
from typing import Literal

MaintenanceDecision = Literal[
    "keep_seed",
    "refresh_expected_trace",
    "compress_seed",
    "quarantine_seed",
    "retire_seed",
    "manual_review",
]


@dataclass
class LongTermSeedHealth:
    seed_id: str
    source_promotion_receipt_id: str
    owner_active: bool
    last_30d_run_count: int
    pass_rate: float
    flaky_rate: float
    median_latency_ms: int
    cost_cents_per_run: int
    runtime_schema_changed: bool
    tool_contract_changed: bool
    expected_trace_matches_current_contract: bool
    recurrence_detected_in_last_90d: bool
    replacement_coverage_score: float
    retirement_condition_met: bool
    failure_has_clear_owner: bool


def decide_suite_maintenance(seed: LongTermSeedHealth) -> MaintenanceDecision:
    if not seed.owner_active:
        return "manual_review"

    if seed.last_30d_run_count < 5:
        return "manual_review"

    if seed.flaky_rate > 0.05:
        return "quarantine_seed"

    if seed.pass_rate < 0.95 and not seed.failure_has_clear_owner:
        return "manual_review"

    if seed.runtime_schema_changed or seed.tool_contract_changed:
        if seed.expected_trace_matches_current_contract:
            return "keep_seed"
        return "refresh_expected_trace"

    if seed.cost_cents_per_run > 50 or seed.median_latency_ms > 30_000:
        return "compress_seed"

    if (
        seed.retirement_condition_met
        and not seed.recurrence_detected_in_last_90d
        and seed.replacement_coverage_score >= 0.95
    ):
        return "retire_seed"

    return "keep_seed"
~~~

这里故意把 retire 放到很靠后：退役不是因为“它很久没失败”，而是因为：

1. 原始退役条件已经满足；
2. 最近 90 天没有同类复发；
3. 新护栏、新 replay case 或更小 seed 已经覆盖同一风险；
4. seed 本身不是因为 flaky、owner 缺失或 contract 漂移而被迫下线。

## pi-mono：Regression Suite Maintenance Worker

生产版要把维护做成周期 worker。它读取长期套件合同、最近运行记录、schema/tool fingerprint 和替代覆盖索引，然后写维护收据。

~~~ts
// packages/agent-runtime/src/regression/RegressionSuiteMaintenanceWorker.ts
export type MaintenanceDecision =
  | "keep_seed"
  | "refresh_expected_trace"
  | "compress_seed"
  | "quarantine_seed"
  | "retire_seed"
  | "manual_review";

export interface LongTermSeedHealth {
  seedId: string;
  contractId: string;
  sourcePromotionReceiptId: string;
  ownerActive: boolean;
  last30dRunCount: number;
  passRate: number;
  flakyRate: number;
  medianLatencyMs: number;
  costCentsPerRun: number;
  runtimeSchemaChanged: boolean;
  toolContractChanged: boolean;
  expectedTraceMatchesCurrentContract: boolean;
  recurrenceDetectedInLast90d: boolean;
  replacementCoverageScore: number;
  retirementConditionMet: boolean;
  failureHasClearOwner: boolean;
}

export interface SuiteMaintenanceReceipt {
  id: string;
  seedId: string;
  contractId: string;
  decision: MaintenanceDecision;
  reason: string;
  previousTraceHash?: string;
  newTraceHash?: string;
  createdAt: string;
}

export class RegressionSuiteMaintenanceWorker {
  constructor(private readonly store: RegressionStore) {}

  decide(seed: LongTermSeedHealth): MaintenanceDecision {
    if (!seed.ownerActive) return "manual_review";
    if (seed.last30dRunCount < 5) return "manual_review";
    if (seed.flakyRate > 0.05) return "quarantine_seed";

    if (seed.passRate < 0.95 && !seed.failureHasClearOwner) {
      return "manual_review";
    }

    if (seed.runtimeSchemaChanged || seed.toolContractChanged) {
      return seed.expectedTraceMatchesCurrentContract
        ? "keep_seed"
        : "refresh_expected_trace";
    }

    if (seed.costCentsPerRun > 50 || seed.medianLatencyMs > 30_000) {
      return "compress_seed";
    }

    if (
      seed.retirementConditionMet &&
      !seed.recurrenceDetectedInLast90d &&
      seed.replacementCoverageScore >= 0.95
    ) {
      return "retire_seed";
    }

    return "keep_seed";
  }

  async maintainSuite(contractId: string): Promise<SuiteMaintenanceReceipt[]> {
    return this.store.transaction(async (tx) => {
      const contract = await tx.getLongTermSuiteContract(contractId);
      const healthRows = await tx.buildSeedHealthReport({
        contractId: contract.id,
        lookbackDays: 30,
        recurrenceLookbackDays: 90,
      });

      const receipts: SuiteMaintenanceReceipt[] = [];

      for (const health of healthRows) {
        const decision = this.decide(health);

        if (decision === "refresh_expected_trace") {
          const refreshed = await tx.refreshExpectedTrace({
            seedId: health.seedId,
            reason: "runtime_or_tool_contract_changed",
          });

          receipts.push(
            await tx.writeSuiteMaintenanceReceipt({
              seedId: health.seedId,
              contractId,
              decision,
              reason: "expected trace refreshed after contract drift review",
              previousTraceHash: refreshed.previousTraceHash,
              newTraceHash: refreshed.newTraceHash,
            }),
          );
          continue;
        }

        if (decision === "compress_seed") {
          const compressed = await tx.compressSeed({
            seedId: health.seedId,
            keepAssertions: ["decision", "tool_sequence", "risk_signature"],
            mockExternalDependencies: true,
          });

          receipts.push(
            await tx.writeSuiteMaintenanceReceipt({
              seedId: compressed.seedId,
              contractId,
              decision,
              reason: "seed kept but compressed to stay inside suite budget",
              newTraceHash: compressed.traceHash,
            }),
          );
          continue;
        }

        if (decision === "quarantine_seed") {
          await tx.quarantineSeed({
            seedId: health.seedId,
            reason: "flaky_long_term_seed",
          });
        }

        if (decision === "retire_seed") {
          await tx.retireSeed({
            seedId: health.seedId,
            contractId,
            replacementCoverageScore: health.replacementCoverageScore,
          });
        }

        receipts.push(
          await tx.writeSuiteMaintenanceReceipt({
            seedId: health.seedId,
            contractId,
            decision,
            reason: this.reasonFor(decision),
          }),
        );
      }

      return receipts;
    });
  }

  private reasonFor(decision: MaintenanceDecision): string {
    const reasons: Record<MaintenanceDecision, string> = {
      keep_seed: "seed remains stable and valuable",
      refresh_expected_trace: "handled in refresh branch",
      compress_seed: "handled in compression branch",
      quarantine_seed: "seed is too flaky for release gates",
      retire_seed: "retirement condition met with replacement coverage",
      manual_review: "owner, sample size, or failure routing is unclear",
    };
    return reasons[decision];
  }
}
~~~

注意两个细节：

- quarantine 和 retire 都必须写 receipt，不能静默从套件里删除；
- refresh expected trace 只能在 contract drift 已解释清楚时做，不能把真实回归“刷新”成新基线。

## OpenClaw 实战：课程 Cron 的长期套件维护

拿我们的 Agent 课程 cron 举例，lesson/README/TOOLS/Telegram/Git 这一串已经跑了几百次。它天然可以沉淀长期回归 seed：

- seed A：新增 lesson 后 README 必须有对应目录行；
- seed B：TOOLS.md 已讲内容必须追加同主题，避免下次重复选题；
- seed C：Telegram 发送成功后 memory 里要记录 messageId；
- seed D：git push 后远端 main 必须等于本地提交。

这些 seed 进入长期套件后，还要定期维护：

~~~ts
const courseCronSeedHealth: LongTermSeedHealth = {
  seedId: "agent-course-readme-tools-telegram-git-closeout",
  contractId: "release-gate-agent-course-cron",
  sourcePromotionReceiptId: "suite-promotion-479",
  ownerActive: true,
  last30dRunCount: 240,
  passRate: 0.996,
  flakyRate: 0.004,
  medianLatencyMs: 4_200,
  costCentsPerRun: 3,
  runtimeSchemaChanged: false,
  toolContractChanged: false,
  expectedTraceMatchesCurrentContract: true,
  recurrenceDetectedInLast90d: false,
  replacementCoverageScore: 0.72,
  retirementConditionMet: false,
  failureHasClearOwner: true,
};

const decision = worker.decide(courseCronSeedHealth);
// keep_seed
~~~

为什么不能退役？因为 replacementCoverageScore 只有 0.72，说明虽然它很稳定，但还没有更小、更便宜、覆盖足够完整的替代 seed。稳定不等于无用。

如果以后 OpenClaw message tool 的返回结构改了，toolContractChanged=true，并且旧 expected trace 不再匹配当前 contract，这个 seed 应该走 refresh_expected_trace；如果它突然 flaky_rate > 5%，则先 quarantine，不让 release gate 被假红拖死。

## 工程落地清单

长期回归套件维护至少要落 7 个字段：

- seed_id：被维护的长期 seed；
- source_promotion_receipt_id：当初为什么进套件；
- run_window：本次复核统计窗口；
- drift_fingerprint：runtime/tool/schema/model 的版本指纹；
- replacement_coverage_score：替代覆盖是否足够；
- decision：keep / refresh / compress / quarantine / retire / manual_review；
- receipt_id：维护收据，供 CI 和后续 worker 引用。

这里最容易犯的错是把“测试很久没失败”当作退役理由。对 Agent 来说，长期回归测试的价值经常就是长时间不失败，因为它证明某类事故没有复发。真正能退役的条件是：风险被更强的机制覆盖了，并且这件事有证据、有 owner、有收据。

## 小结

长期回归 seed 晋级只是开始，不是终点。

成熟 Agent 的回归套件要会自我维护：稳定的继续跑，漂移的刷新，昂贵的压缩，flaky 的隔离，已经被替代覆盖的才退役。

一句话：**长期测试资产也要有生命周期；没有维护闸门的回归套件，最后会从安全网变成噪音源。**
