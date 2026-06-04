# Agent 长期回归套件的覆盖缺口雷达与种子补齐闸门

> Long-term Regression Suite Coverage Gap Radar & Seed Backfill Gate

第 480 课讲了长期 regression seed 的维护、压缩、刷新和退役。这里还有一个容易被忽略的问题：**当旧 seed 被压缩或退役以后，长期套件可能会悄悄出现覆盖缺口。**

成熟 Agent 不能只问“哪些测试该删”，还要问：

- 哪些 risk surface 最近有生产事件，但没有长期 seed 覆盖；
- 哪些 tool contract / runtime schema 已经变化，但没有新回放样本；
- 哪些高风险 action 只有单一路径测试，没有失败路径、取消路径或重试路径；
- 哪些 seed 被 retire 后，replacement coverage 只是“看起来类似”，但没有覆盖关键断言。

所以长期套件维护后，要再跑一个 **Coverage Gap Radar**：把真实生产风险、事故复发信号、release gate、tool schema 变化和现有 suite coverage 对齐，发现缺口后进入种子补齐闸门。

## 核心模型

可以拆成 5 个对象：

~~~text
SuiteMaintenanceReceipt
  -> RiskSurfaceInventory
  -> CoverageGapReport
  -> SeedBackfillCandidate
  -> SeedBackfillReceipt
~~~

分工：

- SuiteMaintenanceReceipt 说明哪些 seed 被 keep、compress、quarantine 或 retire；
- RiskSurfaceInventory 汇总当前 Agent 的高风险面：外部副作用、权限、支付、消息、Git、部署、记忆写入等；
- CoverageGapReport 对比风险面和现有长期 seed，标出 missing / weak / stale / overfit；
- SeedBackfillCandidate 把真实事件、shadow diff、audit finding 或 synthetic case 转成候选 seed；
- SeedBackfillReceipt 记录补齐结果，后续 nightly、release gate 和事故复盘都能追溯。

一句话：**长期套件不是一张静态清单，而是一张会随生产风险变化自动补洞的网。**

## learn-claude-code：覆盖缺口判定纯函数

教学版先写纯函数：输入风险面覆盖状态，输出是否需要补 seed。

~~~python
# learn_claude_code/regression_coverage_gap_gate.py
from dataclasses import dataclass
from typing import Literal

GapDecision = Literal[
    "coverage_ok",
    "backfill_seed",
    "backfill_compressed_seed",
    "refresh_existing_seed",
    "manual_review",
]


@dataclass
class RiskSurfaceCoverage:
    surface_id: str
    criticality: Literal["low", "medium", "high", "critical"]
    active_seed_count: int
    has_failure_path_seed: bool
    has_cancel_retry_seed: bool
    last_real_incident_days_ago: int | None
    tool_contract_changed: bool
    runtime_schema_changed: bool
    replacement_coverage_score: float
    existing_seed_stale: bool
    candidate_has_pii: bool
    deterministic_replay_rate: float
    estimated_cost_cents: int


def decide_coverage_gap(surface: RiskSurfaceCoverage) -> GapDecision:
    if surface.candidate_has_pii:
        return "manual_review"

    if surface.existing_seed_stale:
        return "refresh_existing_seed"

    if surface.tool_contract_changed or surface.runtime_schema_changed:
        if surface.replacement_coverage_score < 0.95:
            return "backfill_seed"

    if surface.criticality in ("high", "critical") and surface.active_seed_count == 0:
        return "backfill_seed"

    if surface.criticality == "critical" and not surface.has_failure_path_seed:
        return "backfill_seed"

    if surface.criticality == "critical" and not surface.has_cancel_retry_seed:
        return "backfill_compressed_seed"

    if (
        surface.last_real_incident_days_ago is not None
        and surface.last_real_incident_days_ago <= 30
        and surface.replacement_coverage_score < 0.90
    ):
        return "backfill_seed"

    if surface.deterministic_replay_rate < 0.98:
        return "manual_review"

    if surface.estimated_cost_cents > 50:
        return "backfill_compressed_seed"

    return "coverage_ok"
~~~

这里的重点不是追求测试数量，而是按风险补齐：

- critical risk surface 至少要有成功路径、失败路径、取消/重试路径；
- tool contract 或 runtime schema 变了，要检查旧 seed 是否还能代表新现实；
- 最近 30 天有真实事故的风险面，不能只靠相邻 seed 安慰自己；
- 含 PII 或不可稳定回放的候选样本，先 manual review，不直接进长期套件。

## pi-mono：Coverage Gap Radar Worker

生产版可以把 radar 做成 nightly worker，也可以在 suite maintenance 之后触发。

~~~ts
// packages/agent-runtime/src/regression/CoverageGapRadarWorker.ts
export type GapDecision =
  | "coverage_ok"
  | "backfill_seed"
  | "backfill_compressed_seed"
  | "refresh_existing_seed"
  | "manual_review";

export interface RiskSurfaceCoverage {
  surfaceId: string;
  criticality: "low" | "medium" | "high" | "critical";
  activeSeedCount: number;
  hasFailurePathSeed: boolean;
  hasCancelRetrySeed: boolean;
  lastRealIncidentDaysAgo?: number;
  toolContractChanged: boolean;
  runtimeSchemaChanged: boolean;
  replacementCoverageScore: number;
  existingSeedStale: boolean;
  candidateHasPii: boolean;
  deterministicReplayRate: number;
  estimatedCostCents: number;
}

export interface SeedBackfillReceipt {
  id: string;
  surfaceId: string;
  decision: GapDecision;
  candidateSeedId?: string;
  source: "incident" | "audit_finding" | "shadow_diff" | "synthetic";
  reason: string;
  createdAt: string;
}

export class CoverageGapRadarWorker {
  constructor(private readonly store: RegressionStore) {}

  decide(surface: RiskSurfaceCoverage): GapDecision {
    if (surface.candidateHasPii) return "manual_review";
    if (surface.existingSeedStale) return "refresh_existing_seed";

    if (
      (surface.toolContractChanged || surface.runtimeSchemaChanged) &&
      surface.replacementCoverageScore < 0.95
    ) {
      return "backfill_seed";
    }

    if (
      (surface.criticality === "high" || surface.criticality === "critical") &&
      surface.activeSeedCount === 0
    ) {
      return "backfill_seed";
    }

    if (surface.criticality === "critical" && !surface.hasFailurePathSeed) {
      return "backfill_seed";
    }

    if (surface.criticality === "critical" && !surface.hasCancelRetrySeed) {
      return "backfill_compressed_seed";
    }

    if (
      surface.lastRealIncidentDaysAgo !== undefined &&
      surface.lastRealIncidentDaysAgo <= 30 &&
      surface.replacementCoverageScore < 0.90
    ) {
      return "backfill_seed";
    }

    if (surface.deterministicReplayRate < 0.98) return "manual_review";
    if (surface.estimatedCostCents > 50) return "backfill_compressed_seed";

    return "coverage_ok";
  }

  async scan(contractId: string): Promise<SeedBackfillReceipt[]> {
    return this.store.transaction(async (tx) => {
      const inventory = await tx.buildRiskSurfaceInventory({
        includeExternalEffects: true,
        includePermissionedTools: true,
        includeRecentIncidentsDays: 30,
      });

      const coverageRows = await tx.compareSuiteCoverage({
        contractId,
        inventory,
      });

      const receipts: SeedBackfillReceipt[] = [];

      for (const surface of coverageRows) {
        const decision = this.decide(surface);

        if (decision === "coverage_ok") {
          receipts.push(
            await tx.writeSeedBackfillReceipt({
              surfaceId: surface.surfaceId,
              decision,
              source: "synthetic",
              reason: "active long-term seeds cover required paths",
            }),
          );
          continue;
        }

        if (decision === "refresh_existing_seed") {
          const refreshed = await tx.refreshSeedForSurface(surface.surfaceId);
          receipts.push(
            await tx.writeSeedBackfillReceipt({
              surfaceId: surface.surfaceId,
              decision,
              candidateSeedId: refreshed.seedId,
              source: "audit_finding",
              reason: "existing seed was stale after contract drift",
            }),
          );
          continue;
        }

        if (decision === "backfill_seed" || decision === "backfill_compressed_seed") {
          const candidate = await tx.createBackfillSeedCandidate({
            surfaceId: surface.surfaceId,
            compressed: decision === "backfill_compressed_seed",
            preferSources: ["incident", "shadow_diff", "audit_finding", "synthetic"],
          });

          receipts.push(
            await tx.writeSeedBackfillReceipt({
              surfaceId: surface.surfaceId,
              decision,
              candidateSeedId: candidate.seedId,
              source: candidate.source,
              reason: "coverage gap detected by long-term suite radar",
            }),
          );
          continue;
        }

        receipts.push(
          await tx.writeSeedBackfillReceipt({
            surfaceId: surface.surfaceId,
            decision: "manual_review",
            source: "incident",
            reason: "candidate requires human review before long-term promotion",
          }),
        );
      }

      return receipts;
    });
  }
}
~~~

生产实现里要注意两点：

1. radar 只负责发现和生成候选，不应该直接把 seed 塞进 release gate；
2. backfill receipt 要进入第 479 课讲过的 promotion gate，再决定是否进长期套件。

这样可以避免“发现缺口就乱补测试”，也避免“维护时删掉旧 seed，却没有人负责补新覆盖”。

## OpenClaw：课程 Cron 的现实例子

OpenClaw 的 Agent 课程 cron 就是一个很好的例子。长期运行后，我们已经有 Telegram 发送、lesson 文件、README、TOOLS、Git commit/push、远端 main 验证这些固定步骤。

如果某次维护决定退役一个旧 seed，比如“不再检查 README 是否追加目录”，coverage radar 必须立刻问：

- README 目录更新是不是仍有别的 seed 覆盖？
- Telegram 已发送但 Git push 失败的取消/重试路径有没有覆盖？
- TOOLS.md 已更新但 commit 未推送的 partial effect 有没有覆盖？
- gh 账号切换失败、远端已落后、本地有未提交变更，这些关键路径有没有覆盖？

一个实用的 backfill receipt 可以长这样：

~~~json
{
  "id": "sbr_481",
  "surfaceId": "course_cron_git_publish",
  "decision": "backfill_seed",
  "source": "synthetic",
  "reason": "README directory seed retired; git publish path needs replacement coverage",
  "requiredAssertions": [
    "telegram_message_id_recorded",
    "lesson_file_exists",
    "readme_contains_lesson",
    "tools_contains_topic",
    "remote_main_matches_commit"
  ]
}
~~~

这条 receipt 不是测试本身，而是“为什么要补测试”的证据。后面再用 promotion gate 检查 deterministic replay、成本、PII 和 owner，最后才进入长期套件。

## 落地建议

长期回归套件维护后，加一个固定顺序：

1. 先跑 maintenance gate，决定 keep / refresh / compress / quarantine / retire；
2. 再跑 coverage gap radar，检查 risk surface 有没有被删出缺口；
3. 对 high / critical 缺口生成 SeedBackfillCandidate；
4. 用 promotion gate 审核候选 seed；
5. 所有动作写 receipt，release gate 只相信 receipt，不相信口头说明。

这套机制的价值是：**测试套件可以变轻，但不能变瞎。**

成熟 Agent 的回归体系，不是越跑越多，也不是越删越快，而是每次删、压缩、刷新之后，都能证明关键风险仍然被看见。
