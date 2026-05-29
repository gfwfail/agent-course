# 440. Agent 护栏阶段激活后的观察窗口与晋级回滚闸门

> **核心思想：StageActivationReceipt 只能证明某个恢复阶段已经被正确激活，不能证明它已经稳定。成熟 Agent 要为每次 shadow/canary/active 恢复绑定 ObservationWindow，用真实运行信号决定 promote_next_stage、extend_window、rollback_activation 或 reopen_unlock_review。**

---

## 1. 为什么激活后还要观察窗口

上一课讲 RestorePermit 的原子消费：worker 消费 permit、抢 lease、写 StageActivationReceipt。

这一步解决的是“恢复动作有没有被正确执行”。但生产里的另一个问题更危险：恢复动作执行正确，不代表恢复结果稳定。

常见事故有四类：

1. shadow 阶段没有副作用，但真实决策差异已经变坏；
2. canary 阶段样本太少就晋级 active，漏掉低频风险面；
3. runtime 没漂移，但用户可见投诉、队列积压或下游 side effect 开始异常；
4. 观察期发现问题，却没有绑定 rollback point，最后只能手工翻日志。

所以 StageActivationReceipt 后面必须接 ObservationWindow。它不是监控图表，而是一张有边界、有预算、有动作的工程合约。

链路应该是：

- StageActivationReceipt：这次恢复阶段怎么激活；
- ObservationWindow：接下来观察什么、观察多久、预算多少；
- ObservationSample：每个周期采样到的事实；
- ObservationDecision：继续、延长、回滚、重开；
- PromotionPermit 或 RollbackReceipt：下一步动作的单次许可或回滚收据。

---

## 2. 最小数据模型

~~~text
ObservationWindow:
  windowId
  activationReceiptId
  stage: shadow | canary | active
  candidateVersion
  scope:
    tenants[]
    riskSurfaces[]
    actionClasses[]
  budgets:
    maxDecisionCount
    maxFalseAllowRate
    maxFalseBlockRate
    maxUnknownDiffRate
    maxLatencyP95Ms
    maxUserComplaintCount
    maxQueueBacklogDelta
  startedAt
  expiresAt
  rollbackPointRef
  nextStageRequest?

ObservationSample:
  sampleId
  windowId
  observedDecisionCount
  falseAllowCount
  falseBlockCount
  unknownDiffCount
  latencyP95Ms
  userComplaintCount
  queueBacklogDelta
  sideEffectMismatchCount
  runtimeFingerprint
  sampledAt

ObservationDecision:
  decisionId
  windowId
  decision:
    promote_next_stage
    close_stable
    extend_window
    rollback_activation
    reopen_unlock_review
    manual_review
  reason
  evidenceRefs[]
  nextPermitId?
  rollbackReceiptId?
~~~

这里最重要的是 rollbackPointRef。观察窗口不是“看一看”，而是“看坏了能退”。没有 rollback point 的观察，只能叫告警，不能叫恢复闸门。

---

## 3. 观察什么信号

第一，观察决策质量。

false_allow、false_block、unknown_diff 是核心信号。false_allow 通常要 fail closed，false_block 可以按业务风险决定是 rollback 还是 extend。

第二，观察 scope 边界。

如果 canary 只允许 payment_refund，却采样到了 account_delete，这不是样本异常，而是 scope violation。正确动作是 rollback_activation 或 reopen_unlock_review。

第三，观察 runtime 一致性。

StageActivationReceipt 里的 runtimeFingerprint 要和观察时一致。fingerprint 变了，说明正在观察的系统不再是激活时那套系统，必须重新复核。

第四，观察用户可见副作用。

Agent 护栏不是只服务内部指标。用户投诉、消息发送失败、订单状态异常、外部 effect mismatch 都要进入窗口预算。

第五，观察时间和样本量。

窗口到期但样本量不够，不应该 promote，也不应该直接 rollback。正确结果通常是 extend_window，并说明还差哪些样本。

---

## 4. learn-claude-code：纯函数观察闸门

教学版先把观察决策写成纯函数。输入是窗口定义、当前采样聚合和运行时 fingerprint，输出下一步动作。

~~~py
from dataclasses import dataclass
from typing import Literal


Stage = Literal["shadow", "canary", "active"]
Decision = Literal[
    "promote_next_stage",
    "close_stable",
    "extend_window",
    "rollback_activation",
    "reopen_unlock_review",
    "manual_review",
]


@dataclass(frozen=True)
class ObservationBudgets:
    min_decision_count: int
    max_false_allow_rate: float
    max_false_block_rate: float
    max_unknown_diff_rate: float
    max_latency_p95_ms: int
    max_user_complaints: int
    max_queue_backlog_delta: int


@dataclass(frozen=True)
class ObservationWindow:
    window_id: str
    stage: Stage
    runtime_fingerprint: str
    expires_at_epoch: int
    rollback_point_ref: str | None
    budgets: ObservationBudgets


@dataclass(frozen=True)
class ObservationAggregate:
    decision_count: int
    false_allow_count: int
    false_block_count: int
    unknown_diff_count: int
    latency_p95_ms: int
    user_complaint_count: int
    queue_backlog_delta: int
    side_effect_mismatch_count: int
    runtime_fingerprint: str
    now_epoch: int


@dataclass(frozen=True)
class ObservationGateResult:
    decision: Decision
    reason: str
    needs_new_unlock_review: bool


def rate(count: int, total: int) -> float:
    if total <= 0:
        return 0.0
    return count / total


def evaluate_observation(
    window: ObservationWindow,
    aggregate: ObservationAggregate,
) -> ObservationGateResult:
    if window.runtime_fingerprint != aggregate.runtime_fingerprint:
        return ObservationGateResult(
            "reopen_unlock_review",
            "runtime_fingerprint_changed",
            True,
        )

    if not window.rollback_point_ref:
        return ObservationGateResult(
            "manual_review",
            "missing_rollback_point",
            True,
        )

    if aggregate.side_effect_mismatch_count > 0:
        return ObservationGateResult(
            "rollback_activation",
            "side_effect_mismatch_detected",
            False,
        )

    false_allow_rate = rate(aggregate.false_allow_count, aggregate.decision_count)
    if false_allow_rate > window.budgets.max_false_allow_rate:
        return ObservationGateResult(
            "rollback_activation",
            "false_allow_budget_exceeded",
            False,
        )

    false_block_rate = rate(aggregate.false_block_count, aggregate.decision_count)
    if false_block_rate > window.budgets.max_false_block_rate:
        return ObservationGateResult(
            "extend_window",
            "false_block_budget_exceeded_needs_more_evidence",
            False,
        )

    unknown_diff_rate = rate(aggregate.unknown_diff_count, aggregate.decision_count)
    if unknown_diff_rate > window.budgets.max_unknown_diff_rate:
        return ObservationGateResult(
            "extend_window",
            "unknown_diff_budget_exceeded",
            False,
        )

    if aggregate.latency_p95_ms > window.budgets.max_latency_p95_ms:
        return ObservationGateResult("extend_window", "latency_budget_exceeded", False)

    if aggregate.user_complaint_count > window.budgets.max_user_complaints:
        return ObservationGateResult("manual_review", "user_complaint_budget_exceeded", True)

    if aggregate.queue_backlog_delta > window.budgets.max_queue_backlog_delta:
        return ObservationGateResult("extend_window", "queue_backlog_budget_exceeded", False)

    if aggregate.decision_count < window.budgets.min_decision_count:
        if aggregate.now_epoch >= window.expires_at_epoch:
            return ObservationGateResult("extend_window", "insufficient_sample_count", False)
        return ObservationGateResult("extend_window", "window_still_collecting_samples", False)

    if window.stage in ("shadow", "canary"):
        return ObservationGateResult("promote_next_stage", "window_stable", False)

    return ObservationGateResult("close_stable", "active_window_stable", False)
~~~

这段代码的关键不是阈值，而是动作分层：能自动回滚的直接回滚，证据不足的延长窗口，环境变了的重开 unlock review，active 稳定后才 close_stable。

---

## 5. pi-mono：事件流里的 ObservationWorker

生产版不要让 API 请求线程同步等观察结果。StageActivationReceipt 写入后，发一个事件，后台 ObservationWorker 按窗口聚合事实。

~~~ts
type ObservationDecision =
  | "promote_next_stage"
  | "close_stable"
  | "extend_window"
  | "rollback_activation"
  | "reopen_unlock_review"
  | "manual_review";

type ObservationGateResult = {
  decision: ObservationDecision;
  reason: string;
  needsNewUnlockReview: boolean;
};

export class ObservationWorker {
  constructor(
    private readonly windows: ObservationWindowStore,
    private readonly samples: ObservationSampleStore,
    private readonly decisions: ObservationDecisionStore,
    private readonly permits: PromotionPermitStore,
    private readonly rollback: RollbackOrchestrator,
    private readonly outbox: Outbox,
  ) {}

  async evaluate(windowId: string, runId: string): Promise<void> {
    await this.windows.transaction(async (tx) => {
      const window = await tx.windows.lockById(windowId);
      if (!window || window.status !== "observing") return;

      const aggregate = await tx.samples.aggregate(window.windowId);
      const result = evaluateObservation(window, aggregate);

      const decision = await tx.decisions.insert({
        windowId: window.windowId,
        activationReceiptId: window.activationReceiptId,
        decision: result.decision,
        reason: result.reason,
        evidenceRefs: aggregate.evidenceRefs,
        decidedByRunId: runId,
      });

      if (result.decision === "promote_next_stage") {
        const permit = await tx.permits.createSingleUse({
          sourceDecisionId: decision.decisionId,
          candidateVersion: window.candidateVersion,
          allowedStage: nextStage(window.stage),
          allowedScope: window.scope,
          dependencyFingerprint: aggregate.runtimeFingerprint,
          expiresAt: addMinutes(new Date(), 30),
        });

        await tx.windows.markClosed(window.windowId, "promoted", decision.decisionId);
        await tx.outbox.publish("guard.promotion_permit.created", {
          permitId: permit.permitId,
          windowId: window.windowId,
        });
        return;
      }

      if (result.decision === "rollback_activation") {
        const receipt = await this.rollback.rollbackToPoint(tx, {
          rollbackPointRef: window.rollbackPointRef,
          reason: result.reason,
          sourceDecisionId: decision.decisionId,
        });

        await tx.windows.markClosed(window.windowId, "rolled_back", decision.decisionId);
        await tx.outbox.publish("guard.activation.rolled_back", {
          rollbackReceiptId: receipt.receiptId,
          windowId: window.windowId,
        });
        return;
      }

      if (result.decision === "reopen_unlock_review") {
        await tx.windows.markClosed(window.windowId, "reopen_required", decision.decisionId);
        await tx.outbox.publish("guard.unlock_review.reopen_requested", {
          windowId: window.windowId,
          reason: result.reason,
        });
        return;
      }

      if (result.decision === "extend_window") {
        await tx.windows.extend(window.windowId, {
          reason: result.reason,
          extraSampleBudget: 200,
          extraMinutes: 30,
        });
        return;
      }

      await tx.windows.markClosed(window.windowId, "stable", decision.decisionId);
    });
  }
}
~~~

这里有两个生产细节：

- promote_next_stage 不直接改 active 状态，而是创建下一张 PromotionPermit，让下一阶段仍然走一次性许可；
- rollback_activation 要在同一个事务里关闭窗口、写 decision、发 outbox，避免重复回滚或漏发事件。

---

## 6. OpenClaw 实战：课程 Cron 也要有观察窗口

拿这个课程任务举例，发课不是 message sent 就结束。

一次课程 cron 的 StageActivationReceipt 可以是：

~~~json
{
  "activationReceiptId": "lesson-440-publish",
  "stage": "active",
  "scope": {
    "channel": "telegram:-5115329245",
    "repo": "gfwfail/agent-course",
    "files": [
      "lessons/440-guard-pattern-post-activation-observation-window-promotion-rollback.md",
      "README.md",
      "TOOLS.md"
    ]
  },
  "rollbackPointRef": "git:main@2b45744",
  "observationWindowId": "lesson-440-post-publish"
}
~~~

观察窗口要检查：

1. Telegram messageId 是否返回；
2. lesson 文件是否存在、README 是否指向它；
3. TOOLS.md 是否追加已讲内容，避免下次重复；
4. git push 后远端 main 是否包含新 commit；
5. markdown 检查是否没有 trailing whitespace；
6. 今天 memory 是否记录 messageId、commit 和主题。

如果 Telegram 发送成功但 git push 失败，窗口不能 close_stable。正确动作是 extend_window 或 manual_review，因为外部世界和 repo 状态不一致。

---

## 7. 常见坑

第一，把观察窗口做成 Prometheus 告警。

Prometheus 只能告诉你指标变坏，不能自动证明这次恢复应该 promote、rollback 还是 reopen。观察窗口必须有 decision 和 receipt。

第二，样本不足也晋级。

“没有坏样本”不等于“稳定”。低风险面可以缩短窗口，但不能跳过 min_decision_count 或明确豁免。

第三，false_allow 和 false_block 用同一个动作处理。

false_allow 通常表示漏挡危险动作，要更倾向 rollback。false_block 可能是误伤，可以先 extend 或 manual_review。

第四，active 稳定后没有 close receipt。

active 阶段 close_stable 后，要把窗口、样本、decision、rollback point、最终 guard version 打包归档。否则以后出了相似事故，无法复盘“当时为什么认为它稳定”。

---

## 8. 记忆口诀

StageActivationReceipt 证明“我恢复过”。

ObservationWindow 证明“恢复后我看过”。

ObservationDecision 证明“我知道下一步该怎么做”。

PromotionPermit / RollbackReceipt 证明“下一步动作仍然受控”。

成熟 Agent 的恢复不是一键解锁，而是一段带观察窗口、证据预算和可回滚路径的受控放量。
