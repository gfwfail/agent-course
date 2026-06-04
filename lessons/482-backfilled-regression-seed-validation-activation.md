# Agent 补齐回归种子的验证、去重与激活闸门

> Backfilled Regression Seed Validation, Dedup & Activation Gate

第 481 课讲了长期回归套件的 Coverage Gap Radar：当 seed 被压缩、退役，或者 tool/runtime 发生漂移后，要主动发现哪些风险面缺覆盖，并生成 SeedBackfillCandidate。

但这里还有一个坑：**候选 seed 不等于可用 seed。**

如果 radar 一发现缺口就直接把候选塞进长期套件，很容易出现几类问题：

- seed 其实不可稳定重放，nightly 变成噪音；
- seed 和已有样本重复，只是换了一个 incident id；
- seed 带了 raw PII、secret、外部 message body，长期保留不合规；
- seed 没有 owner 和退役条件，几年后没人知道为什么存在；
- seed 只覆盖了表面路径，没有覆盖真正的断言。

所以第 482 课补上 **Backfilled Regression Seed Validation, Dedup & Activation Gate**：补齐候选必须先经过验证、去重、降密、owner 绑定和激活收据，才能进入长期套件。

## 核心模型

把补齐后的激活拆成 6 个对象：

~~~text
SeedBackfillCandidate
  -> CandidateValidationReport
  -> SeedDedupeReview
  -> SeedActivationPlan
  -> BackfilledSeedActivationReceipt
  -> SuiteIndexUpdateReceipt
~~~

分工：

- SeedBackfillCandidate：从事故、审计发现、shadow diff 或 synthetic case 生成的候选；
- CandidateValidationReport：证明候选可重放、断言明确、证据已降密；
- SeedDedupeReview：判断它是 new coverage、duplicate、adjacent coverage 还是 supersedes old seed；
- SeedActivationPlan：绑定 suite、risk surface、owner、TTL、成本预算和退役条件；
- BackfilledSeedActivationReceipt：记录激活动作真的完成；
- SuiteIndexUpdateReceipt：证明索引、README、release gate 和 nightly runner 都能看到它。

一句话：**补齐 seed 的目标不是“多一个测试”，而是补上一个可稳定、可追溯、可退役的风险传感器。**

## learn-claude-code：验证与激活判定纯函数

教学版可以先把 gate 写成纯函数。输入候选 seed 的验证结果，输出是否允许激活。

~~~python
# learn_claude_code/backfilled_seed_activation_gate.py
from dataclasses import dataclass
from typing import Literal

ActivationDecision = Literal[
    "activate_seed",
    "activate_compressed_seed",
    "merge_with_existing_seed",
    "quarantine_candidate",
    "reject_candidate",
    "manual_review",
]


@dataclass
class BackfilledSeedCandidate:
    candidate_id: str
    risk_surface_id: str
    criticality: Literal["low", "medium", "high", "critical"]
    deterministic_replay_rate: float
    assertion_count: int
    has_failure_assertion: bool
    has_external_side_effect: bool
    redaction_complete: bool
    contains_secret_or_pii: bool
    duplicate_score: float
    adjacent_coverage_score: float
    supersedes_existing_seed: bool
    owner_assigned: bool
    retirement_condition_defined: bool
    estimated_cost_cents: int
    source_trust: Literal["incident", "audit_finding", "shadow_diff", "synthetic"]


def decide_backfilled_seed_activation(seed: BackfilledSeedCandidate) -> ActivationDecision:
    if seed.contains_secret_or_pii and not seed.redaction_complete:
        return "manual_review"

    if seed.deterministic_replay_rate < 0.95:
        return "quarantine_candidate"

    if seed.assertion_count == 0:
        return "reject_candidate"

    if seed.criticality in ("high", "critical") and not seed.has_failure_assertion:
        return "quarantine_candidate"

    if not seed.owner_assigned or not seed.retirement_condition_defined:
        return "manual_review"

    if seed.duplicate_score >= 0.98:
        return "merge_with_existing_seed"

    if seed.adjacent_coverage_score >= 0.95 and not seed.supersedes_existing_seed:
        return "merge_with_existing_seed"

    if seed.has_external_side_effect and seed.estimated_cost_cents > 50:
        return "activate_compressed_seed"

    if seed.estimated_cost_cents > 100:
        return "activate_compressed_seed"

    return "activate_seed"
~~~

这个 gate 有几个故意很硬的规则：

- 不稳定的 seed 先 quarantine，不进长期套件污染 nightly；
- 没有断言的 replay 只是录像，不是 regression seed；
- critical seed 必须覆盖失败断言，否则不能证明它守住了风险；
- 没 owner、没退役条件的 seed 不能长期存在；
- 高重复候选不要新增，只合并到已有 seed 的 coverage metadata。

## pi-mono：BackfilledSeedActivationWorker

生产版要做两件事：一是验证候选，二是在事务里激活并更新索引。否则容易出现 seed 文件创建了，但 release gate 看不到；或者 README 有目录项，但 nightly runner 没注册。

~~~ts
// packages/agent-runtime/src/regression/BackfilledSeedActivationWorker.ts
export type ActivationDecision =
  | "activate_seed"
  | "activate_compressed_seed"
  | "merge_with_existing_seed"
  | "quarantine_candidate"
  | "reject_candidate"
  | "manual_review";

export interface CandidateValidationReport {
  candidateId: string;
  riskSurfaceId: string;
  criticality: "low" | "medium" | "high" | "critical";
  deterministicReplayRate: number;
  assertionCount: number;
  hasFailureAssertion: boolean;
  hasExternalSideEffect: boolean;
  redactionComplete: boolean;
  containsSecretOrPii: boolean;
  duplicateScore: number;
  adjacentCoverageScore: number;
  supersedesExistingSeed: boolean;
  ownerAssigned: boolean;
  retirementConditionDefined: boolean;
  estimatedCostCents: number;
  sourceTrust: "incident" | "audit_finding" | "shadow_diff" | "synthetic";
}

export interface BackfilledSeedActivationReceipt {
  receiptId: string;
  candidateId: string;
  suiteId: string;
  decision: ActivationDecision;
  activatedSeedId?: string;
  mergedIntoSeedId?: string;
  owner?: string;
  reason: string;
  createdAt: string;
}

export class BackfilledSeedActivationWorker {
  constructor(private readonly store: RegressionSeedStore) {}

  decide(report: CandidateValidationReport): ActivationDecision {
    if (report.containsSecretOrPii && !report.redactionComplete) {
      return "manual_review";
    }

    if (report.deterministicReplayRate < 0.95) return "quarantine_candidate";
    if (report.assertionCount === 0) return "reject_candidate";

    if (
      (report.criticality === "high" || report.criticality === "critical") &&
      !report.hasFailureAssertion
    ) {
      return "quarantine_candidate";
    }

    if (!report.ownerAssigned || !report.retirementConditionDefined) {
      return "manual_review";
    }

    if (report.duplicateScore >= 0.98) return "merge_with_existing_seed";

    if (
      report.adjacentCoverageScore >= 0.95 &&
      !report.supersedesExistingSeed
    ) {
      return "merge_with_existing_seed";
    }

    if (report.hasExternalSideEffect && report.estimatedCostCents > 50) {
      return "activate_compressed_seed";
    }

    if (report.estimatedCostCents > 100) return "activate_compressed_seed";

    return "activate_seed";
  }

  async activate(candidateId: string, suiteId: string): Promise<BackfilledSeedActivationReceipt> {
    const report = await this.store.validateCandidate(candidateId);
    const decision = this.decide(report);

    return this.store.transaction(async (tx) => {
      if (decision === "merge_with_existing_seed") {
        const target = await tx.findBestExistingSeed(report.riskSurfaceId, {
          candidateId,
          duplicateScore: report.duplicateScore,
          adjacentCoverageScore: report.adjacentCoverageScore,
        });

        await tx.attachCandidateEvidence(target.seedId, candidateId);
        await tx.updateCoverageMetadata(target.seedId, {
          riskSurfaceId: report.riskSurfaceId,
          sourceTrust: report.sourceTrust,
        });

        return tx.writeBackfilledSeedActivationReceipt({
          candidateId,
          suiteId,
          decision,
          mergedIntoSeedId: target.seedId,
          reason: "candidate coverage was already represented by an existing seed",
        });
      }

      if (decision === "activate_seed" || decision === "activate_compressed_seed") {
        const seed = await tx.createLongTermSeed({
          candidateId,
          suiteId,
          compressed: decision === "activate_compressed_seed",
          requireOwner: true,
          requireRetirementCondition: true,
        });

        await tx.registerSeedInNightlyRunner(seed.seedId);
        await tx.registerSeedInReleaseGate(seed.seedId);
        await tx.updateSuiteIndex({
          suiteId,
          seedId: seed.seedId,
          riskSurfaceId: report.riskSurfaceId,
        });

        return tx.writeBackfilledSeedActivationReceipt({
          candidateId,
          suiteId,
          decision,
          activatedSeedId: seed.seedId,
          owner: seed.owner,
          reason: "validated backfill candidate was activated into long-term suite",
        });
      }

      await tx.markCandidateNonActive(candidateId, decision);

      return tx.writeBackfilledSeedActivationReceipt({
        candidateId,
        suiteId,
        decision,
        reason: "candidate did not meet long-term activation requirements",
      });
    });
  }
}
~~~

这里最关键的是事务边界：

1. 创建或合并 seed；
2. 更新 coverage metadata；
3. 注册 nightly runner；
4. 注册 release gate；
5. 写 activation receipt。

这几步必须一起成功。只成功一半，就会出现“以为有覆盖，实际发布闸门没跑”的假安全感。

## OpenClaw 实战：课程 Cron 的 seed 怎么补齐

用这个课程 cron 举例，Coverage Gap Radar 可能发现：

- Telegram 发送成功但 git push 失败的半完成路径没有 seed；
- README 更新成功但 TOOLS.md 漏写的长期记忆路径没有 seed；
- gh auth 切错账号导致 push 权限失败，没有 release gate seed；
- lesson 文件存在但 README 目录缺项，只有人工发现；
- messageId 已发送但重复 cron 重新运行，缺少幂等 replay seed。

这些都可以生成 SeedBackfillCandidate，但不能直接激活。激活前至少跑：

~~~text
1. replay 是否稳定：同一输入重放 20 次，关键 decision 是否一致
2. assertion 是否明确：检查 lesson、README、TOOLS、Telegram、git commit 五个状态
3. redaction 是否完成：群消息、token、remote URL 只保留 hash 或 summary
4. dedupe 是否通过：是否已经有类似课程 cron seed
5. owner 和 retirement 是否存在：谁负责，什么条件下退役
6. runner 是否注册：nightly 和 release gate 是否都能跑到
~~~

如果只是为了证明“README 缺目录项会被发现”，可以用 compressed seed：保留文件 hash、期望目录项和模拟 git status，不保留完整聊天上下文。

## 工程判断

好的 backfilled seed 有 5 个标准：

1. **能稳定重放**：不是靠时间、随机数或真实外部副作用碰运气；
2. **断言对准风险**：不是 replay 成功就算过，而是检查关键安全属性；
3. **去重后再新增**：能合并就合并，不让长期套件膨胀成噪音；
4. **降密后长期留存**：raw evidence 不进长期热路径；
5. **有 owner 和退役条件**：seed 生命周期必须有人和规则负责。

## 记住

> Backfill candidate 只是补洞线索，不是长期测试资产。

成熟 Agent 的长期回归套件，不是发现缺口就疯狂加测试，而是把每个补齐 seed 都验证成稳定、去重、降密、可运行、可退役的工程资产。这样测试套件才会越补越准，而不是越补越吵。
