# Agent 事故保护解除后的发布列车恢复与烟测闸门

> Post-Incident Release Train Resumption & Smoke Replay Gate

第 477 课讲了 sentinel 退出：释放 LKG pin、归档证据、沉淀 regression seed。

今天讲下一步：**临时保护解除后，不能立刻把发布系统切回满速自动驾驶。要先恢复 release train，并用 smoke replay 证明“正常发布节奏”真的能跑通。**

很多 Agent 事故恢复后会出现一个反直觉问题：事故处理链路都绿了，但正常发布链路已经生锈了。临时 pin、manual approval、high sampling、freeze window、shadow override 关掉以后，release worker、队列租约、runtime reload、通知 side effect 和 Git/README/TOOLS 对账可能已经和事故前不一样。

所以成熟系统要有一个明确的 **ReleaseTrainResumptionGate**：它不再判断事故是否恢复，而是判断系统能不能安全回到日常发布节奏。

## 核心模型

把恢复正常发布拆成 5 个对象：

~~~text
SentinelExitReceipt
  -> ReleaseTrainResumptionPlan
  -> SmokeReplayRun
  -> RuntimeConvergenceCheck
  -> ReleaseTrainResumptionReceipt
~~~

分工要清楚：

- SentinelExitReceipt 证明事故哨兵已经退出；
- ReleaseTrainResumptionPlan 说明哪些发布限制要恢复、哪些还要保留；
- SmokeReplayRun 用小流量或 dry-run 重新走一遍正常发布路径；
- RuntimeConvergenceCheck 确认所有 runtime 都加载了恢复后的发布配置；
- ReleaseTrainResumptionReceipt 才允许 release train 回到 normal cadence。

关键原则：**解除事故保护不是恢复正常。只有正常发布链路被重新跑通，才算恢复正常。**

## learn-claude-code：发布恢复判定纯函数

教学版先写一个纯函数：输入发布恢复审查，输出恢复、继续烟测、保留限速、重新打开事故或人工复核。

~~~python
# learn_claude_code/release_train_resumption_gate.py
from dataclasses import dataclass
from typing import Literal

ResumptionDecision = Literal[
    "resume_normal_train",
    "resume_with_rate_limit",
    "extend_smoke_replay",
    "reopen_incident",
    "manual_review",
]


@dataclass
class ReleaseTrainResumptionReview:
    sentinel_exit_receipt_id: str
    lkg_unpinned: bool
    smoke_replay_runs: int
    required_smoke_runs: int
    smoke_failures: int
    side_effect_mismatches: int
    runtime_convergence_rate: float
    queue_backlog_age_minutes: int
    stale_release_lock_count: int
    regression_seed_pass_rate: float
    normal_cadence_blocked: bool
    retained_manual_approval: bool


def decide_release_train_resumption(
    review: ReleaseTrainResumptionReview,
) -> ResumptionDecision:
    if review.smoke_failures > 0 and review.side_effect_mismatches > 0:
        return "reopen_incident"

    if review.stale_release_lock_count > 0:
        return "manual_review"

    if review.runtime_convergence_rate < 1.0:
        return "extend_smoke_replay"

    if review.queue_backlog_age_minutes > 30:
        return "resume_with_rate_limit"

    if review.smoke_replay_runs < review.required_smoke_runs:
        return "extend_smoke_replay"

    if review.side_effect_mismatches > 0:
        return "manual_review"

    if review.regression_seed_pass_rate < 1.0:
        return "extend_smoke_replay"

    if review.normal_cadence_blocked and review.retained_manual_approval:
        return "resume_with_rate_limit"

    if not review.lkg_unpinned:
        return "resume_with_rate_limit"

    return "resume_normal_train"
~~~

这里有三个细节：

1. sentinel_exit_receipt_id 是前置证据，不能绕过上一课的退出收据。
2. runtime convergence 必须是 100%，否则有些 worker 还在事故配置里跑。
3. LKG 没解钉也可以恢复限速发布，但不能宣称回到 normal train。

## pi-mono：Release Train Resumption Worker

生产版要把恢复发布做成一个幂等 worker。它不是“把开关设回 normal”，而是带 smoke replay、runtime convergence 和 receipt 的状态机。

~~~ts
// packages/agent-runtime/src/release/ReleaseTrainResumptionWorker.ts
export type ResumptionDecision =
  | "resume_normal_train"
  | "resume_with_rate_limit"
  | "extend_smoke_replay"
  | "reopen_incident"
  | "manual_review";

export interface ReleaseTrainResumptionPlan {
  incidentId: string;
  policyId: string;
  sentinelExitReceiptId: string;
  targetCadence: "normal" | "rate_limited";
  smokeReplayBudget: number;
  releaseLane: "shadow" | "canary" | "active";
  keepControls: string[];
  releaseControls: string[];
}

export interface SmokeReplayRun {
  id: string;
  planId: string;
  sampledDecisionIds: string[];
  sideEffectMode: "dry_run" | "idempotent_probe";
  failureCount: number;
  sideEffectMismatchCount: number;
  regressionSeedPassRate: number;
}

export interface RuntimeConvergenceCheck {
  planId: string;
  expectedConfigHash: string;
  runtimeCount: number;
  convergedRuntimeCount: number;
  staleRuntimeIds: string[];
}

export interface ReleaseTrainResumptionReceipt {
  id: string;
  incidentId: string;
  planId: string;
  decision: ResumptionDecision;
  smokeReplayRunIds: string[];
  runtimeConfigHash: string;
  targetCadence: "normal" | "rate_limited" | "blocked";
  createdAt: string;
}

export class ReleaseTrainResumptionWorker {
  constructor(private readonly store: ReleaseStore) {}

  decide(input: {
    plan: ReleaseTrainResumptionPlan;
    smokeRuns: SmokeReplayRun[];
    convergence: RuntimeConvergenceCheck;
    queueBacklogAgeMinutes: number;
    staleReleaseLockCount: number;
    lkgUnpinned: boolean;
  }): ResumptionDecision {
    const failures = input.smokeRuns.reduce((sum, run) => sum + run.failureCount, 0);
    const mismatches = input.smokeRuns.reduce(
      (sum, run) => sum + run.sideEffectMismatchCount,
      0,
    );
    const minSeedPassRate = Math.min(
      ...input.smokeRuns.map((run) => run.regressionSeedPassRate),
    );
    const convergenceRate =
      input.convergence.convergedRuntimeCount / input.convergence.runtimeCount;

    if (failures > 0 && mismatches > 0) return "reopen_incident";
    if (input.staleReleaseLockCount > 0) return "manual_review";
    if (convergenceRate < 1) return "extend_smoke_replay";
    if (input.queueBacklogAgeMinutes > 30) return "resume_with_rate_limit";
    if (input.smokeRuns.length < input.plan.smokeReplayBudget) {
      return "extend_smoke_replay";
    }
    if (mismatches > 0) return "manual_review";
    if (minSeedPassRate < 1) return "extend_smoke_replay";
    if (!input.lkgUnpinned) return "resume_with_rate_limit";
    return input.plan.targetCadence === "normal"
      ? "resume_normal_train"
      : "resume_with_rate_limit";
  }

  async run(planId: string): Promise<ReleaseTrainResumptionReceipt> {
    return this.store.transaction(async (tx) => {
      const plan = await tx.getResumptionPlan(planId);
      await tx.assertSentinelExitReceipt(plan.sentinelExitReceiptId);

      const smokeRuns = await tx.runMissingSmokeReplays({
        planId,
        budget: plan.smokeReplayBudget,
        lane: plan.releaseLane,
        sideEffectMode: "idempotent_probe",
      });

      const convergence = await tx.checkRuntimeConvergence({
        planId,
        policyId: plan.policyId,
      });

      const decision = this.decide({
        plan,
        smokeRuns,
        convergence,
        queueBacklogAgeMinutes: await tx.getReleaseQueueBacklogAgeMinutes(),
        staleReleaseLockCount: await tx.countStaleReleaseLocks(plan.policyId),
        lkgUnpinned: await tx.isLkgUnpinned(plan.policyId),
      });

      if (decision === "reopen_incident") {
        await tx.reopenIncident({
          incidentId: plan.incidentId,
          reason: "release_train_smoke_replay_failed",
        });
      }

      if (decision === "resume_normal_train") {
        await tx.setReleaseCadence({
          policyId: plan.policyId,
          cadence: "normal",
          releaseControls: plan.releaseControls,
        });
      }

      if (decision === "resume_with_rate_limit") {
        await tx.setReleaseCadence({
          policyId: plan.policyId,
          cadence: "rate_limited",
          keepControls: plan.keepControls,
        });
      }

      return tx.writeReleaseTrainResumptionReceipt({
        incidentId: plan.incidentId,
        planId,
        decision,
        smokeReplayRunIds: smokeRuns.map((run) => run.id),
        runtimeConfigHash: convergence.expectedConfigHash,
        targetCadence:
          decision === "resume_normal_train"
            ? "normal"
            : decision === "resume_with_rate_limit"
              ? "rate_limited"
              : "blocked",
      });
    });
  }
}
~~~

实现重点：

1. assertSentinelExitReceipt 防止跳过事故关闭链路。
2. smoke replay 要走真实 release path，但 side effect 必须是 dry-run 或 idempotent probe。
3. runtime convergence 要看配置 hash，不只看 worker 在线。
4. resume_with_rate_limit 是合法中间态，不要把它当失败。
5. Receipt 要记录 smoke run、runtime config hash 和最终 cadence，方便以后复盘。

## OpenClaw：课程 Cron 怎么落地

拿这个课程 cron 举例：第 477 课释放了 LKG pin 后，不应该马上恢复“每 3 小时无条件发布”的全速策略。应该先跑一次小型 smoke replay：

~~~text
ReleaseTrainResumptionPlan:
  incident_id: course-topic-suppression-2026-06
  target_cadence: normal
  smoke_replay_budget: 2
  release_lane: active
  release_controls:
    - lkg_pin
    - forced_manual_topic_review
  keep_controls:
    - duplicate_topic_regression_seed
    - git_readme_tools_diff_check
~~~

Smoke replay 不需要再发 Telegram。它可以 dry-run 下面几件事：

- 生成下一课标题，确认不会被误判为重复；
- 检查 lessons 文件名、README 条目和 TOOLS 已讲内容是否一致；
- 验证 git diff 只包含本轮课程文件和索引；
- 模拟 Telegram payload，确认消息长度、中文标题、代码块格式都正常；
- 确认所有 runtime 都加载了新的去重策略 hash。

只有这些 smoke replay 通过，课程 cron 才能写：

~~~text
ReleaseTrainResumptionReceipt:
  decision: resume_normal_train
  target_cadence: normal
  smoke_replay_run_ids: [smoke-1, smoke-2]
  runtime_config_hash: topic-policy-v478
~~~

如果 Git 队列积压、runtime hash 不一致或 TOOLS/README 对账失败，那就不要宣布恢复正常。可以先 resume_with_rate_limit，比如下一轮仍保留额外 diff check。

## 常见坑

### 坑 1：把 LKG 解钉当作恢复发布

LKG 解钉只说明事故保护可以释放，不说明正常 release worker 没坏。发布链路必须重新跑一遍。

### 坑 2：smoke replay 真的发外部副作用

烟测要接近真实路径，但不能重复发 Telegram、重复 push、重复部署。副作用应该走 dry-run、idempotency key 或 reality probe。

### 坑 3：只检查主 runtime

Agent 系统常有 cron worker、subagent runner、tool dispatcher、message sender 多个 runtime。只看一个进程的配置 hash，会漏掉旧策略残留。

### 坑 4：恢复正常没有 receipt

“我把开关改回去了”不是证据。必须写 ReleaseTrainResumptionReceipt，记录为什么恢复、恢复到什么 cadence、基于哪些 smoke replay。

## 工程检查清单

- 是否要求 SentinelExitReceipt 作为恢复发布前置条件？
- ReleaseTrainResumptionPlan 是否区分 releaseControls 和 keepControls？
- SmokeReplayRun 是否覆盖决策、文件、队列、外部副作用 payload？
- side effect 是否使用 dry-run、idempotency key 或 probe？
- RuntimeConvergenceCheck 是否基于 config hash？
- stale release lock 是否会阻断自动恢复？
- queue backlog 过大时是否进入 rate-limited 而不是 full normal？
- Receipt 是否记录 cadence、runtime hash 和 smoke run ids？

## 关键 takeaway

**事故恢复结束，不等于发布系统恢复正常。**

成熟 Agent 的完整闭环应该是：

~~~text
rollback
  -> backfill
  -> close_recovered
  -> sentinel
  -> exit review
  -> unpin / archive / regression seed
  -> release train resumption
  -> smoke replay
  -> normal cadence receipt
~~~

没有 ReleaseTrainResumptionReceipt，就不要急着把系统切回全速。否则下一次事故不一定来自旧 bug，可能来自已经很久没真正跑过的“正常发布路径”。
