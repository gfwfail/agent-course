# Agent 基线再晋级后的观察与回退闸门

> Post-Repromotion Baseline Observation & Rollback Gate

第 493 课讲了 Baseline Remediation Candidate Re-promotion：漂移修复后不能直接把配置改回 active，而是要把修复后的 baseline 作为 candidate，经过 shadow 验证、回归回放、隐藏信号检查，再决定 promote_candidate、extend_shadow、tighten_patch、reject_candidate 或 manual_review。

今天补上 promote 之后的闭环：候选基线被重新晋级为 active，不代表事情结束。因为 shadow 模式只能证明“旁路计算看起来没问题”，不能证明真实生产路径、真实队列、真实工具失败、真实成本和真实告警都稳定。

所以第 494 课是 **Post-Repromotion Baseline Observation & Rollback Gate**：基线 candidate 晋级后，必须创建短期观察租约，采样真实运行结果，对比晋级前的 shadow 预期和 rollback point。如果出现重复漂移、隐藏信号回补、成本失控或工具失败率上升，就自动 rollback_repromotion 或 reopen_baseline_case。

一句话：**基线重新上线不是终点，而是进入真实流量观察期。promote 是控制面动作，observe 才证明运行面真的接住了。**

## 核心模型

把再晋级后的生产观察拆成 5 个对象：

~~~text
BaselineRepromotionReceipt
  -> PostRepromotionObservationLease
  -> RuntimeBaselineSignalSnapshot
  -> RepromotionRollbackReadinessReview
  -> PostRepromotionCloseoutReceipt
~~~

分工：

- BaselineRepromotionReceipt：上一课写下的再晋级收据，包含 candidateVersion、rollbackBaselineVersion、patchHash；
- PostRepromotionObservationLease：观察租约，定义观察窗口、样本数、阈值、退出条件和自动回退策略；
- RuntimeBaselineSignalSnapshot：真实 active 路径的运行信号，包括 error、latency、cost、tool failure、alert delivery、sampling coverage；
- RepromotionRollbackReadinessReview：判断是否继续观察、稳定关闭、回退 candidate、重开 baseline case；
- PostRepromotionCloseoutReceipt：关闭收据，记录最终 stable、rolled_back、reopened 或 manual_review 的结果。

关键点：这里看的是 **active 后的真实运行信号**，不是 shadow 报告本身。

## learn-claude-code：观察期判定纯函数

教学版继续用纯函数表达。输入再晋级收据、观察租约和真实信号快照，输出下一步动作。

~~~python
# learn_claude_code/post_repromotion_observation_gate.py
from dataclasses import dataclass
from typing import Literal

ObservationDecision = Literal[
    "close_stable",
    "continue_observation",
    "rollback_repromotion",
    "reopen_baseline_case",
    "manual_review",
]


@dataclass
class BaselineRepromotionReceipt:
    case_id: str
    candidate_version: str
    rollback_baseline_version: str
    decision: str  # promote_candidate
    patch_hash: str


@dataclass
class PostRepromotionObservationLease:
    case_id: str
    baseline_version: str
    min_windows: int
    max_windows: int
    max_error_rate_delta: float
    max_p95_delta_ms: int
    max_cost_delta: float
    max_tool_failure_delta: float
    min_sampling_coverage: float
    expires_at: int


@dataclass
class RuntimeBaselineSignalSnapshot:
    case_id: str
    baseline_version: str
    observed_windows: int
    error_rate_delta: float
    p95_delta_ms: int
    cost_delta: float
    tool_failure_delta: float
    sampling_coverage: float
    repeated_breach_signature: bool
    hidden_signal_backfill_count: int
    rollback_probe_passed: bool
    now: int


def decide_post_repromotion_observation(
    receipt: BaselineRepromotionReceipt,
    lease: PostRepromotionObservationLease,
    snapshot: RuntimeBaselineSignalSnapshot,
) -> ObservationDecision:
    same_case = (
        receipt.case_id == lease.case_id == snapshot.case_id
        and receipt.candidate_version == lease.baseline_version
        and receipt.candidate_version == snapshot.baseline_version
        and receipt.decision == "promote_candidate"
    )
    if not same_case:
        return "manual_review"

    if snapshot.now > lease.expires_at:
        return "manual_review"

    if snapshot.sampling_coverage < lease.min_sampling_coverage:
        return "continue_observation"

    if snapshot.hidden_signal_backfill_count > 0:
        return "manual_review"

    if snapshot.repeated_breach_signature and snapshot.rollback_probe_passed:
        return "rollback_repromotion"

    runtime_worse = (
        snapshot.error_rate_delta > lease.max_error_rate_delta
        or snapshot.p95_delta_ms > lease.max_p95_delta_ms
        or snapshot.cost_delta > lease.max_cost_delta
        or snapshot.tool_failure_delta > lease.max_tool_failure_delta
    )

    if runtime_worse and snapshot.rollback_probe_passed:
        return "rollback_repromotion"

    if runtime_worse:
        return "reopen_baseline_case"

    if snapshot.observed_windows < lease.min_windows:
        return "continue_observation"

    if snapshot.observed_windows > lease.max_windows:
        return "manual_review"

    return "close_stable"
~~~

这里有 4 个硬门槛：

- `sampling_coverage` 不够时不能关闭，因为稳定可能只是没采到；
- `hidden_signal_backfill_count` 大于 0 时要人工复核，因为 shadow 阶段可能漏了关键观测；
- `repeated_breach_signature` 复现原漂移签名时优先 rollback；
- `rollback_probe_passed` 是自动回退前的最后检查，确认旧 baseline 仍可加载、可运行、可收敛。

## pi-mono：PostRepromotionObservationWorker

生产版 worker 不应该只发告警，而要在同一事务里写观察收据、决定动作、必要时切回 rollback baseline，并安排后续 case。

~~~ts
// packages/agent-runtime/src/ops/PostRepromotionObservationWorker.ts
export type ObservationDecision =
  | "close_stable"
  | "continue_observation"
  | "rollback_repromotion"
  | "reopen_baseline_case"
  | "manual_review";

export interface BaselineRepromotionReceipt {
  caseId: string;
  candidateVersion: string;
  rollbackBaselineVersion: string;
  decision: "promote_candidate";
  patchHash: string;
}

export interface PostRepromotionObservationLease {
  caseId: string;
  baselineVersion: string;
  minWindows: number;
  maxWindows: number;
  maxErrorRateDelta: number;
  maxP95DeltaMs: number;
  maxCostDelta: number;
  maxToolFailureDelta: number;
  minSamplingCoverage: number;
  expiresAt: number;
}

export interface RuntimeBaselineSignalSnapshot {
  caseId: string;
  baselineVersion: string;
  observedWindows: number;
  errorRateDelta: number;
  p95DeltaMs: number;
  costDelta: number;
  toolFailureDelta: number;
  samplingCoverage: number;
  repeatedBreachSignature: boolean;
  hiddenSignalBackfillCount: number;
  rollbackProbePassed: boolean;
  now: number;
}

export class PostRepromotionObservationWorker {
  constructor(private readonly store: BaselineOpsStore) {}

  decide(
    receipt: BaselineRepromotionReceipt,
    lease: PostRepromotionObservationLease,
    snapshot: RuntimeBaselineSignalSnapshot,
  ): ObservationDecision {
    const sameCase =
      receipt.caseId === lease.caseId &&
      receipt.caseId === snapshot.caseId &&
      receipt.candidateVersion === lease.baselineVersion &&
      receipt.candidateVersion === snapshot.baselineVersion &&
      receipt.decision === "promote_candidate";

    if (!sameCase) return "manual_review";
    if (snapshot.now > lease.expiresAt) return "manual_review";
    if (snapshot.samplingCoverage < lease.minSamplingCoverage) {
      return "continue_observation";
    }
    if (snapshot.hiddenSignalBackfillCount > 0) return "manual_review";

    if (snapshot.repeatedBreachSignature && snapshot.rollbackProbePassed) {
      return "rollback_repromotion";
    }

    const runtimeWorse =
      snapshot.errorRateDelta > lease.maxErrorRateDelta ||
      snapshot.p95DeltaMs > lease.maxP95DeltaMs ||
      snapshot.costDelta > lease.maxCostDelta ||
      snapshot.toolFailureDelta > lease.maxToolFailureDelta;

    if (runtimeWorse && snapshot.rollbackProbePassed) {
      return "rollback_repromotion";
    }
    if (runtimeWorse) return "reopen_baseline_case";
    if (snapshot.observedWindows < lease.minWindows) {
      return "continue_observation";
    }
    if (snapshot.observedWindows > lease.maxWindows) return "manual_review";

    return "close_stable";
  }

  async handle(snapshot: RuntimeBaselineSignalSnapshot) {
    return this.store.transaction(async (tx) => {
      const receipt = await tx.getRepromotionReceipt(snapshot.caseId);
      const lease = await tx.getObservationLease(snapshot.caseId);
      const decision = this.decide(receipt, lease, snapshot);

      const closeout = await tx.writePostRepromotionCloseout({
        caseId: snapshot.caseId,
        baselineVersion: snapshot.baselineVersion,
        decision,
        snapshot,
        decidedAt: Date.now(),
      });

      if (decision === "rollback_repromotion") {
        await tx.activateBaselinePointer({
          version: receipt.rollbackBaselineVersion,
          expectedCurrent: receipt.candidateVersion,
          reason: "post_repromotion_observation_failed",
          receiptId: closeout.id,
        });

        await tx.openBaselineCase({
          parentCaseId: snapshot.caseId,
          baselineVersion: receipt.candidateVersion,
          rollbackVersion: receipt.rollbackBaselineVersion,
          reason: "candidate_failed_after_promotion",
        });
      }

      if (decision === "reopen_baseline_case") {
        await tx.openBaselineCase({
          parentCaseId: snapshot.caseId,
          baselineVersion: snapshot.baselineVersion,
          reason: "runtime_worse_without_safe_rollback",
        });
      }

      if (decision === "continue_observation") {
        await tx.extendObservationLease({
          caseId: snapshot.caseId,
          baselineVersion: snapshot.baselineVersion,
          reason: "insufficient_or_incomplete_runtime_signal",
        });
      }

      return closeout;
    });
  }
}
~~~

生产里最容易漏的是 `expectedCurrent`。没有它，旧 worker 可能在 stale 状态下把 baseline 指针切回去，覆盖了另一个合法发布。所有 control-plane pointer 切换都应该带 CAS 条件。

## OpenClaw 实战：课程 Cron 的类比

这套逻辑可以直接类比当前课程自动化：

1. 第 493 课相当于把候选 lesson 写入 README、TOOLS、Git，并发到 Telegram；
2. 第 494 课相当于发完后还要观察：Git push 是否成功、README 是否指向正确文件、TOOLS 是否追加去重、Telegram 是否真的送达；
3. 如果 Git 成功但 Telegram 失败，不能说课程已完成，要 reopen delivery case；
4. 如果 Telegram 发错主题，就要 correction / rollback / closeout，而不是下一轮继续假装没事。

OpenClaw 可以用一个轻量文件式观察收据：

~~~json
{
  "caseId": "agent-course-494",
  "baselineVersion": "lesson-494",
  "signals": {
    "gitPushed": true,
    "readmeLinked": true,
    "toolsUpdated": true,
    "telegramDelivered": true
  },
  "decision": "close_stable"
}
~~~

这个文件不一定每次都要保留，但这个思路很重要：**外部副作用完成后要有观察和关闭，不要把工具返回值当最终现实。**

## 常见坑

第一，shadow 报告通过就删掉 rollback point。错。rollback point 至少要保留到 post-promotion observation close_stable。

第二，只看错误率，不看采样覆盖。采样掉了、日志漏了、告警没发，错误率当然好看。

第三，自动回退不做 probe。旧 baseline 可能已经和新工具 schema 不兼容，盲目回退会制造更大的事故。

第四，观察期无限延长。观察租约要有 maxWindows 和 expiresAt，超过就 manual_review，避免系统一直处在半发布状态。

## 记忆口诀

**候选晋级看 shadow，晋级之后看 reality；回退之前跑 probe，关闭之前写 receipt。**

成熟 Agent 的运维基线发布链路应该是：

~~~text
recalibrate
  -> activate with sentinel
  -> detect breach
  -> remediate candidate
  -> shadow repromotion
  -> active observation
  -> close stable or rollback
~~~

到这里，事故后的基线重校准才算真正闭环：不是“配置改好了”，而是“真实运行证明它稳定，失败时也知道怎么退”。
