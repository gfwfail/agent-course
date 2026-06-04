# Agent 补齐回归种子的首轮运行与燃尽监控闸门

> Backfilled Seed First-Run Burn-in & Flake Budget Gate

第 482 课讲了补齐 seed 的验证、去重与激活：candidate 只有通过 replay 稳定性、断言质量、降密、owner、退役条件和 runner 注册，才允许进入长期回归套件。

但激活不是终点。一个 seed 第一次进入 nightly / release gate 后，最容易暴露三类问题：

- 离线 replay 稳定，真实 runner 环境却因为时间、权限、缓存、工具版本不同而 flaky；
- seed 断言太宽，跑过了却没有真的保护风险面；
- seed 成本、耗时或外部依赖超预算，拖慢整条 release train；
- 激活索引写成功了，但首次运行结果没有被 owner、suite 和 risk surface 正确关联；
- 第一次失败被当作普通测试失败，没有进入隔离、修复或压缩路径。

所以第 483 课补上 **Backfilled Seed First-Run Burn-in & Flake Budget Gate**：新补齐 seed 激活后必须进入一个短暂的燃尽窗口，先证明它在真实 runner 里稳定、有信号、成本可控，再转成长期 active seed。

## 核心模型

把首轮运行拆成 5 个对象：

~~~text
BackfilledSeedActivationReceipt
  -> FirstRunEnrollment
  -> BurnInRunSummary
  -> FlakeBudgetReview
  -> BurnInCloseoutReceipt
~~~

分工：

- BackfilledSeedActivationReceipt：上一课的激活收据，证明 seed 是合法进入套件的；
- FirstRunEnrollment：把 seed 放入 burn-in lane，绑定 runner、owner、riskSurface、观察次数和预算；
- BurnInRunSummary：记录首轮/多轮真实执行的 pass、fail、flake、duration、cost 和 assertion signal；
- FlakeBudgetReview：判断 flake 是否可接受、是否需要隔离或压缩；
- BurnInCloseoutReceipt：关闭燃尽窗口，决定 promote_long_term、quarantine_seed、compress_seed、repair_seed 或 manual_review。

一句话：**补齐 seed 不是激活就完，而是要在真实运行环境里先烧一轮，证明它不是噪音。**

## learn-claude-code：首轮燃尽判定纯函数

教学版先写成纯函数。输入 seed 的燃尽统计，输出关闭动作。

~~~python
# learn_claude_code/backfilled_seed_burn_in_gate.py
from dataclasses import dataclass
from typing import Literal

BurnInDecision = Literal[
    "promote_long_term",
    "quarantine_seed",
    "compress_seed",
    "repair_seed",
    "manual_review",
]


@dataclass
class BurnInRunSummary:
    seed_id: str
    risk_surface_id: str
    criticality: Literal["low", "medium", "high", "critical"]
    enrolled_runs: int
    passed_runs: int
    failed_runs: int
    flaky_runs: int
    assertion_signal_rate: float
    p95_duration_ms: int
    average_cost_cents: int
    runner_contract_matched: bool
    owner_acknowledged: bool
    first_failure_triaged: bool
    has_external_dependency: bool


def decide_burn_in(summary: BurnInRunSummary) -> BurnInDecision:
    if not summary.runner_contract_matched:
        return "manual_review"

    if not summary.owner_acknowledged:
        return "manual_review"

    if summary.enrolled_runs < 3:
        return "repair_seed"

    flake_rate = summary.flaky_runs / summary.enrolled_runs
    fail_rate = summary.failed_runs / summary.enrolled_runs

    if summary.criticality in ("high", "critical") and summary.assertion_signal_rate < 0.9:
        return "repair_seed"

    if fail_rate > 0 and not summary.first_failure_triaged:
        return "manual_review"

    if flake_rate >= 0.2:
        return "quarantine_seed"

    if summary.has_external_dependency and summary.average_cost_cents > 50:
        return "compress_seed"

    if summary.p95_duration_ms > 30_000 or summary.average_cost_cents > 100:
        return "compress_seed"

    return "promote_long_term"
~~~

这里的关键不是阈值，而是状态分类：

- runner contract 不匹配，说明激活索引和真实执行环境对不上，不能自动放行；
- owner 没确认，长期 seed 以后没人管，必须停住；
- enrolled_runs 太少，样本不足，不要急着转长期；
- high / critical seed 断言信号太低，说明它可能只是“跑了流程”，没守住风险；
- 首次失败没 triage，不允许被 nightly 噪音吞掉；
- flake 超预算先 quarantine，避免污染 release gate；
- 外部依赖高成本 seed 优先压缩成 synthetic / cassette replay。

## pi-mono：BurnIn Worker

生产版可以把新激活的 seed 放进单独的 burn-in lane。它参加 nightly，但默认不阻断主 release；只有燃尽通过后，才进入长期套件的 blocking lane。

~~~ts
// packages/agent-runtime/src/regression/BackfilledSeedBurnInWorker.ts
export type BurnInDecision =
  | "promote_long_term"
  | "quarantine_seed"
  | "compress_seed"
  | "repair_seed"
  | "manual_review";

export interface BurnInRunSummary {
  seedId: string;
  riskSurfaceId: string;
  criticality: "low" | "medium" | "high" | "critical";
  enrolledRuns: number;
  passedRuns: number;
  failedRuns: number;
  flakyRuns: number;
  assertionSignalRate: number;
  p95DurationMs: number;
  averageCostCents: number;
  runnerContractMatched: boolean;
  ownerAcknowledged: boolean;
  firstFailureTriaged: boolean;
  hasExternalDependency: boolean;
}

export interface BurnInCloseoutReceipt {
  receiptId: string;
  seedId: string;
  decision: BurnInDecision;
  burnInRuns: number;
  flakeRate: number;
  reason: string;
  createdAt: string;
}

export class BackfilledSeedBurnInWorker {
  constructor(private readonly store: RegressionSeedStore) {}

  decide(summary: BurnInRunSummary): BurnInDecision {
    if (!summary.runnerContractMatched) return "manual_review";
    if (!summary.ownerAcknowledged) return "manual_review";
    if (summary.enrolledRuns < 3) return "repair_seed";

    const flakeRate = summary.flakyRuns / summary.enrolledRuns;
    const failRate = summary.failedRuns / summary.enrolledRuns;

    if (
      (summary.criticality === "high" || summary.criticality === "critical") &&
      summary.assertionSignalRate < 0.9
    ) {
      return "repair_seed";
    }

    if (failRate > 0 && !summary.firstFailureTriaged) return "manual_review";
    if (flakeRate >= 0.2) return "quarantine_seed";

    if (summary.hasExternalDependency && summary.averageCostCents > 50) {
      return "compress_seed";
    }

    if (summary.p95DurationMs > 30_000 || summary.averageCostCents > 100) {
      return "compress_seed";
    }

    return "promote_long_term";
  }

  async closeBurnIn(seedId: string): Promise<BurnInCloseoutReceipt> {
    const summary = await this.store.loadBurnInSummary(seedId);
    const decision = this.decide(summary);
    const flakeRate = summary.flakyRuns / Math.max(summary.enrolledRuns, 1);

    return this.store.transaction(async (tx) => {
      if (decision === "promote_long_term") {
        await tx.moveSeedLane(seedId, "burn_in", "blocking_long_term");
      }

      if (decision === "quarantine_seed") {
        await tx.moveSeedLane(seedId, "burn_in", "quarantine");
        await tx.openRepairTicket(seedId, {
          reason: "flake_budget_exceeded",
          flakeRate,
          riskSurfaceId: summary.riskSurfaceId,
        });
      }

      if (decision === "compress_seed") {
        await tx.moveSeedLane(seedId, "burn_in", "non_blocking_compression");
        await tx.enqueueCompressionPlan(seedId, {
          p95DurationMs: summary.p95DurationMs,
          averageCostCents: summary.averageCostCents,
        });
      }

      if (decision === "repair_seed") {
        await tx.moveSeedLane(seedId, "burn_in", "repair_required");
        await tx.openRepairTicket(seedId, {
          reason: "insufficient_signal_or_runs",
          assertionSignalRate: summary.assertionSignalRate,
        });
      }

      const receipt = await tx.writeBurnInCloseoutReceipt({
        seedId,
        decision,
        burnInRuns: summary.enrolledRuns,
        flakeRate,
        reason: this.reason(summary, decision),
      });

      await tx.publishEvent("regression.seed.burn_in.closed", receipt);
      return receipt;
    });
  }

  private reason(summary: BurnInRunSummary, decision: BurnInDecision): string {
    return `${decision}: runs=${summary.enrolledRuns}, flaky=${summary.flakyRuns}, signal=${summary.assertionSignalRate}`;
  }
}
~~~

这个 Worker 的重点是 **lane 转换**：

- burn_in：新 seed 先真实运行，但不直接阻断主发布；
- blocking_long_term：燃尽通过，进入长期阻断套件；
- quarantine：flake 超预算，先隔离，避免污染 release；
- non_blocking_compression：有价值但成本高，转压缩计划；
- repair_required：断言弱或样本不足，回到 seed 修复队列。

## OpenClaw：课程 Cron 的类比

这个课程 Cron 本身也能类比 burn-in：

1. 新 lesson 写入 `lessons/483-...md`，相当于 seed activation；
2. README 目录更新，证明索引能看到它；
3. Telegram 发出去，证明外部副作用完成；
4. git commit + push，证明远端状态持久化；
5. TOOLS.md 已讲内容更新，证明去重索引闭环。

如果中间任何一步失败，正确做法不是“重跑整个 cron”，而是先检查已经完成的副作用，再从缺口继续。这和 backfilled seed 的 burn-in 一样：**新资产要先证明自己在真实运行链路里被看见、被执行、被记录、被关闭。**

## 实战建议

- 新补齐 seed 不要直接进入 blocking lane，先 burn-in 3 到 5 次；
- burn-in lane 要记录 flake、duration、cost、assertion signal，不只记录 pass/fail；
- 首次失败必须 triage，否则 release gate 会被噪音腐蚀；
- high / critical seed 的断言信号阈值要比普通 seed 更高；
- 有外部依赖的 seed 优先转 cassette replay 或 synthetic compressed seed；
- BurnInCloseoutReceipt 要写入审计日志，后续 seed 退役、压缩、修复都引用它。

## 小结

长期回归套件的质量，不取决于 seed 数量，而取决于每个 seed 是否稳定、有信号、可维护。

Backfilled Seed First-Run Burn-in & Flake Budget Gate 把“刚激活的新 seed”放进一个短期真实运行窗口：先观察、计量、隔离噪音，再决定长期阻断、压缩、修复或人工复核。

成熟 Agent 的测试资产不是一入库就可信，而是先在真实 runner 里烧出第一张稳定收据。
