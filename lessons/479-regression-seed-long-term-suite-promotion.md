# Agent 回归种子的长期套件晋级闸门

> Regression Seed Long-term Suite Promotion Gate

第 478 课讲了事故保护解除后的 release train 恢复：用 smoke replay 证明正常发布链路重新跑通。

今天继续往后讲：**smoke replay 里跑过的事故样本，不能只作为一次性证据用完就丢。成熟 Agent 要把有价值的 regression seed 晋级成长期回归套件，让每次 nightly、CI、策略发布和运行时升级都能复测同类风险。**

很多团队事故后会留下几个“回放脚本”，但三个月后没人知道它们覆盖什么、为什么存在、什么时候可以删除。结果不是测试越堆越慢，就是关键事故样本在重构时被删掉。

所以要有一个明确的 **RegressionSeedPromotionGate**：把临时事故样本变成可维护、可分层、可退役的长期测试资产。

## 核心模型

把回归种子长期化拆成 5 个对象：

~~~text
ReleaseTrainResumptionReceipt
  -> RegressionSeedCandidate
  -> LongTermSuiteContract
  -> SuitePromotionReview
  -> SuitePromotionReceipt
~~~

分工要清楚：

- ReleaseTrainResumptionReceipt 证明这些 seed 刚刚在恢复发布时跑通过；
- RegressionSeedCandidate 描述 seed 的来源、风险面、输入、期望输出和脱敏状态；
- LongTermSuiteContract 说明它进入哪个长期套件、运行频率、owner、成本预算和退役条件；
- SuitePromotionReview 决定 promote、quarantine、compress、manual_review 或 reject；
- SuitePromotionReceipt 写下长期化结果，后续 CI/nightly/cron 都引用这个 receipt。

关键原则：**事故样本不是纪念品，是未来发布的传感器。**

## learn-claude-code：长期化判定纯函数

教学版先写一个纯函数：输入候选种子的质量与成本，输出是否晋级长期套件。

~~~python
# learn_claude_code/regression_seed_promotion_gate.py
from dataclasses import dataclass
from typing import Literal

PromotionDecision = Literal[
    "promote_to_long_term_suite",
    "promote_compressed_seed",
    "quarantine_seed",
    "reject_seed",
    "manual_review",
]


@dataclass
class RegressionSeedCandidate:
    seed_id: str
    source_receipt_id: str
    risk_surface: str
    reproduces_original_failure: bool
    deterministic_replay_rate: float
    contains_raw_pii: bool
    expected_output_stable: bool
    runtime_dependencies_fresh: bool
    estimated_cost_cents: int
    median_latency_ms: int
    duplicate_coverage_score: float
    owner_assigned: bool
    retirement_condition_defined: bool


def decide_seed_promotion(seed: RegressionSeedCandidate) -> PromotionDecision:
    if seed.contains_raw_pii:
        return "quarantine_seed"

    if not seed.reproduces_original_failure:
        return "reject_seed"

    if not seed.owner_assigned or not seed.retirement_condition_defined:
        return "manual_review"

    if seed.deterministic_replay_rate < 0.98:
        return "quarantine_seed"

    if not seed.expected_output_stable:
        return "quarantine_seed"

    if not seed.runtime_dependencies_fresh:
        return "manual_review"

    if seed.duplicate_coverage_score >= 0.90:
        return "reject_seed"

    if seed.estimated_cost_cents > 50 or seed.median_latency_ms > 30_000:
        return "promote_compressed_seed"

    return "promote_to_long_term_suite"
~~~

这里有 4 个工程点：

1. reproduces_original_failure 必须为真，否则它不是回归种子，只是普通样本。
2. deterministic_replay_rate 太低不能进长期套件，否则 CI 会被 flaky test 污染。
3. PII 不允许靠“大家小心点”进入长期资产，必须先隔离脱敏。
4. 成本高不一定拒绝，可以压缩成更小输入、mock 外部依赖、降低运行频率。

## pi-mono：Regression Seed Promotion Worker

生产版要把晋级做成幂等 worker。它从 release train 的 smoke replay receipt 中提取候选 seed，做脱敏、去重、成本评估，然后写入长期套件合同。

~~~ts
// packages/agent-runtime/src/regression/RegressionSeedPromotionWorker.ts
export type PromotionDecision =
  | "promote_to_long_term_suite"
  | "promote_compressed_seed"
  | "quarantine_seed"
  | "reject_seed"
  | "manual_review";

export interface RegressionSeedCandidate {
  seedId: string;
  sourceReceiptId: string;
  riskSurface: string;
  inputHash: string;
  expectedTraceHash: string;
  reproducesOriginalFailure: boolean;
  deterministicReplayRate: number;
  containsRawPii: boolean;
  expectedOutputStable: boolean;
  runtimeDependenciesFresh: boolean;
  estimatedCostCents: number;
  medianLatencyMs: number;
  duplicateCoverageScore: number;
  ownerAssigned: boolean;
  retirementConditionDefined: boolean;
}

export interface LongTermSuiteContract {
  id: string;
  suiteName: "nightly" | "release_gate" | "runtime_upgrade" | "policy_canary";
  riskSurface: string;
  ownerTeam: string;
  runCadence: "per_pr" | "nightly" | "weekly" | "before_policy_promotion";
  maxCostCentsPerRun: number;
  maxLatencyMsPerSeed: number;
  retirementCondition: string;
  sourceReceiptIds: string[];
}

export interface SuitePromotionReceipt {
  id: string;
  seedId: string;
  contractId?: string;
  decision: PromotionDecision;
  reason: string;
  promotedTraceHash?: string;
  createdAt: string;
}

export class RegressionSeedPromotionWorker {
  constructor(private readonly store: RegressionStore) {}

  decide(seed: RegressionSeedCandidate): PromotionDecision {
    if (seed.containsRawPii) return "quarantine_seed";
    if (!seed.reproducesOriginalFailure) return "reject_seed";
    if (!seed.ownerAssigned || !seed.retirementConditionDefined) {
      return "manual_review";
    }
    if (seed.deterministicReplayRate < 0.98) return "quarantine_seed";
    if (!seed.expectedOutputStable) return "quarantine_seed";
    if (!seed.runtimeDependenciesFresh) return "manual_review";
    if (seed.duplicateCoverageScore >= 0.9) return "reject_seed";
    if (seed.estimatedCostCents > 50 || seed.medianLatencyMs > 30_000) {
      return "promote_compressed_seed";
    }
    return "promote_to_long_term_suite";
  }

  async promoteFromReleaseReceipt(
    releaseReceiptId: string,
  ): Promise<SuitePromotionReceipt[]> {
    return this.store.transaction(async (tx) => {
      const releaseReceipt = await tx.getReleaseTrainResumptionReceipt(
        releaseReceiptId,
      );
      const candidates = await tx.extractRegressionSeedCandidates(
        releaseReceipt.smokeReplayRunIds,
      );

      const receipts: SuitePromotionReceipt[] = [];

      for (const candidate of candidates) {
        const decision = this.decide(candidate);

        if (decision === "quarantine_seed") {
          await tx.quarantineSeed({
            seedId: candidate.seedId,
            reason: "pii_or_flaky_or_unstable_expected_output",
          });
        }

        if (decision === "promote_compressed_seed") {
          const compressed = await tx.compressSeed({
            seedId: candidate.seedId,
            keepAssertions: ["decision", "tool_sequence", "side_effect_digest"],
            mockExternalDependencies: true,
          });

          const contract = await tx.upsertLongTermSuiteContract({
            suiteName: "nightly",
            riskSurface: candidate.riskSurface,
            ownerTeam: "agent-runtime",
            runCadence: "nightly",
            maxCostCentsPerRun: 20,
            maxLatencyMsPerSeed: 10_000,
            retirementCondition:
              "retire after 90 days without recurrence and replacement guard coverage >= 0.95",
            sourceReceiptIds: [candidate.sourceReceiptId],
          });

          receipts.push(
            await tx.writeSuitePromotionReceipt({
              seedId: compressed.seedId,
              contractId: contract.id,
              decision,
              reason: "seed promoted after compression",
              promotedTraceHash: compressed.traceHash,
            }),
          );
          continue;
        }

        if (decision === "promote_to_long_term_suite") {
          const contract = await tx.upsertLongTermSuiteContract({
            suiteName: "release_gate",
            riskSurface: candidate.riskSurface,
            ownerTeam: "agent-runtime",
            runCadence: "before_policy_promotion",
            maxCostCentsPerRun: 50,
            maxLatencyMsPerSeed: 30_000,
            retirementCondition:
              "retire only after sunset review proves duplicate long-term coverage",
            sourceReceiptIds: [candidate.sourceReceiptId],
          });

          receipts.push(
            await tx.writeSuitePromotionReceipt({
              seedId: candidate.seedId,
              contractId: contract.id,
              decision,
              reason: "deterministic low-cost seed with assigned owner",
              promotedTraceHash: candidate.expectedTraceHash,
            }),
          );
          continue;
        }

        receipts.push(
          await tx.writeSuitePromotionReceipt({
            seedId: candidate.seedId,
            decision,
            reason: "seed did not meet long-term promotion criteria",
          }),
        );
      }

      return receipts;
    });
  }
}
~~~

注意这里的 upsertLongTermSuiteContract 很重要：同一个 risk surface 的多个 seed 可以挂到同一份 suite contract 下，避免每次事故都新增一个没人维护的测试目录。

## OpenClaw：课程 cron 的真实例子

这门课本身就是一个 Agent 自动化任务：每 3 小时生成课程、发 Telegram、写 lesson、更新 README、更新 TOOLS、commit、push。

第 478 课的 smoke replay 可以对应这些检查：

- Telegram dry-run：消息格式、长度、目标群组是否正确；
- Git dry-run：git diff --check、远端 main 是否同步；
- README/TOOLS 对账：课程编号、目录条目、已讲内容是否一致；
- memory receipt：把 messageId、commit hash、lesson path 写入当天记忆。

第 479 课讲的长期化，就是把这些一次性检查晋级为长期套件：

~~~yaml
# .agent-regression/course-cron-suite.yml
suite: agent-course-cron-release-gate
owner: agent-runtime
cadence: before_policy_promotion
seeds:
  - id: course-number-monotonic
    sourceReceipt: release-train-resumption-478
    assertion: "new lesson number must equal max(existing lessons) + 1"
  - id: readme-tools-consistency
    sourceReceipt: release-train-resumption-478
    assertion: "README entry and TOOLS taught-topic entry must describe same lesson"
  - id: telegram-side-effect-digest
    sourceReceipt: release-train-resumption-478
    assertion: "message target, title, and code blocks match committed lesson"
  - id: git-identity-and-remote-main
    sourceReceipt: release-train-resumption-478
    assertion: "gh user is gfwfail and pushed commit is remote main HEAD"
retirement:
  condition: "replace only when a broader course-publish contract covers all assertions"
~~~

这样下一次改课程 cron、换 Git 账号、改 README 生成器、改消息发送器时，不需要靠人肉记忆。长期套件会告诉你：这个自动化工作流以前出过什么事，现在是否还守得住。

## 实战建议

1. 每个事故 seed 必须有 source receipt，没有来源就不能进长期资产。
2. 长期套件必须有 owner 和 retirement condition，否则测试只会越积越慢。
3. 先去重再晋级，重复覆盖超过 90% 的 seed 直接拒绝。
4. 高成本 seed 不要硬塞进 PR CI，压缩后放 nightly 或 release gate。
5. seed 的期望结果要用 trace hash / side effect digest 表达，避免 fragile string snapshot。

一句话总结：**smoke replay 证明这次恢复能过；长期回归套件证明未来不会悄悄把同一个坑挖回来。**
