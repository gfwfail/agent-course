# 503. Agent 切流后的稳定观察与主路径排空闸门（Post-Cutover Stabilization & Primary Drain Gate）

上一课讲的是 **Switchover Permit Consumption & Live Cutover Audit Gate**：拿到一次性切换许可后，必须通过 CAS 消费、真实切流、运行时一致性审计、健康采样和回滚 probe，才能证明切换动作本身没有失控。

这一课继续往后走：**切流成功不等于可以立刻关掉旧主路径。**

很多生产事故不是发生在切换按钮按下的那一秒，而是发生在切流后 10 分钟、1 小时或第二天：

- 新 recovery path 看起来健康，但还有老 primary worker 在消费队列；
- 主路径 outbox 里还有未投递副作用；
- cache / scheduler / webhook receiver 还有旧路由；
- 用户可见状态已经在 recovery 更新，但 primary 迟到写入又把它覆盖；
- 团队看到“切换成功”就解除了保护，结果没法回滚。

所以第 503 课是 **Post-Cutover Stabilization & Primary Drain Gate**：切流后不要急着宣布胜利，而是进入稳定观察窗口，排空旧主路径，确认迟到副作用、双写冲突、队列残留和回滚点都处理干净，再决定关闭、延长观察、回滚或隔离。

## 1. 为什么切流后还要观察

`LiveCutoverCloseoutReceipt` 只能证明某个时间点：

- permit 被正确消费；
- control plane 指向了目标路径；
- runtime 大体收敛；
- 健康采样没有明显越界；
- 回滚 probe 仍然可用。

但它不证明：

- 旧 primary 已经完全排空；
- 所有迟到任务都被幂等处理；
- outbox 没有遗留副作用；
- recovery 写入不会被 primary 旧写覆盖；
- 用户可见状态和审计投影持续一致；
- 可以安全解钉 LKG primary。

成熟 Agent 的 cutover 不是“指针切过去就结束”，而是：

```text
LiveCutoverCloseoutReceipt
  -> PostCutoverObservationLease
  -> PrimaryDrainReport
  -> LateEffectReconciliation
  -> RollbackReadinessSnapshot
  -> PostCutoverStabilizationReceipt
```

核心原则：**新路径稳定、旧路径排空、迟到副作用已对账，三者同时成立，才算切换真的稳定。**

## 2. 最小数据模型

```python
from dataclasses import dataclass
from typing import Literal

Decision = Literal[
    "close_cutover_stable",
    "continue_observation",
    "rollback_to_primary",
    "quarantine_primary_path",
    "enter_degraded_mode",
    "manual_review",
]


@dataclass(frozen=True)
class LiveCutoverCloseoutReceipt:
    receipt_id: str
    permit_id: str
    target_path: Literal["recovery", "primary"]
    archive_hash: str
    restore_path_version: str
    closed_as_stable: bool
    created_at_epoch: int


@dataclass(frozen=True)
class PostCutoverObservationLease:
    lease_id: str
    receipt_id: str
    min_observation_seconds: int
    started_at_epoch: int
    expires_at_epoch: int
    required_sample_count: int
    max_error_rate_percent: float
    max_late_effect_count: int


@dataclass(frozen=True)
class PrimaryDrainReport:
    receipt_id: str
    primary_workers_active: int
    primary_queue_depth: int
    primary_outbox_pending: int
    scheduler_old_route_count: int
    webhook_old_route_count: int
    drain_checkpoint_written: bool


@dataclass(frozen=True)
class LateEffectReconciliation:
    receipt_id: str
    late_primary_writes: int
    overwritten_recovery_writes: int
    duplicate_side_effects: int
    unreconciled_user_visible_mismatches: int
    idempotency_keys_verified: bool
    audit_projection_consistent: bool


@dataclass(frozen=True)
class RollbackReadinessSnapshot:
    receipt_id: str
    lkg_primary_pinned: bool
    rollback_probe_passed: bool
    recovery_state_exported: bool
    primary_replay_cursor_safe: bool
    rollback_cost_within_budget: bool
```

这里最容易漏的是 `LateEffectReconciliation`。切流后 primary 的迟到写入不一定是 bug，可能是旧任务正常完成；真正危险的是这些迟到结果没有通过幂等键、版本号或 compare-and-swap 被正确吸收。

## 3. learn-claude-code：纯函数闸门

教学版先写纯函数，把复杂状态压成可测试的判定器。

```python
def decide_post_cutover_stabilization(
    receipt: LiveCutoverCloseoutReceipt,
    lease: PostCutoverObservationLease,
    drain: PrimaryDrainReport,
    reconciliation: LateEffectReconciliation,
    rollback: RollbackReadinessSnapshot,
    *,
    now_epoch: int,
    observed_sample_count: int,
    recovery_error_rate_percent: float,
) -> Decision:
    if not receipt.closed_as_stable:
        return "manual_review"

    if lease.receipt_id != receipt.receipt_id:
        return "manual_review"

    if drain.receipt_id != receipt.receipt_id:
        return "manual_review"

    if reconciliation.receipt_id != receipt.receipt_id:
        return "manual_review"

    if rollback.receipt_id != receipt.receipt_id:
        return "manual_review"

    observed_seconds = now_epoch - lease.started_at_epoch
    if observed_seconds < lease.min_observation_seconds:
        return "continue_observation"

    if observed_sample_count < lease.required_sample_count:
        return "continue_observation"

    if recovery_error_rate_percent > lease.max_error_rate_percent:
        if rollback.rollback_probe_passed and rollback.lkg_primary_pinned:
            return "rollback_to_primary"
        return "enter_degraded_mode"

    if drain.primary_workers_active > 0:
        return "continue_observation"

    if drain.primary_queue_depth > 0:
        return "continue_observation"

    if drain.primary_outbox_pending > 0:
        return "continue_observation"

    if drain.scheduler_old_route_count > 0 or drain.webhook_old_route_count > 0:
        return "quarantine_primary_path"

    if not drain.drain_checkpoint_written:
        return "manual_review"

    late_effects = (
        reconciliation.late_primary_writes
        + reconciliation.overwritten_recovery_writes
        + reconciliation.duplicate_side_effects
        + reconciliation.unreconciled_user_visible_mismatches
    )
    if late_effects > lease.max_late_effect_count:
        return "rollback_to_primary"

    if reconciliation.overwritten_recovery_writes > 0:
        return "rollback_to_primary"

    if reconciliation.duplicate_side_effects > 0:
        return "quarantine_primary_path"

    if reconciliation.unreconciled_user_visible_mismatches > 0:
        return "enter_degraded_mode"

    if not reconciliation.idempotency_keys_verified:
        return "manual_review"

    if not reconciliation.audit_projection_consistent:
        return "manual_review"

    if not rollback.lkg_primary_pinned:
        return "manual_review"

    if not rollback.rollback_probe_passed:
        return "continue_observation"

    if not rollback.recovery_state_exported:
        return "continue_observation"

    if not rollback.primary_replay_cursor_safe:
        return "manual_review"

    if not rollback.rollback_cost_within_budget:
        return "manual_review"

    if now_epoch > lease.expires_at_epoch:
        return "close_cutover_stable"

    return "continue_observation"
```

这个函数的重点不是“条件很多”，而是把工程判断拆成几个可解释的阶段：

1. 证据是否属于同一个 receipt；
2. 观察窗口是否足够；
3. recovery path 是否持续健康；
4. primary path 是否真正排空；
5. late effects 是否完成对账；
6. rollback path 是否仍然存在。

## 4. pi-mono：Worker 不直接关 primary

生产版不要让 cutover worker 自己删除旧主路径。更稳的做法是把它拆成两个 worker：

- `PostCutoverObservationWorker`：采样新路径健康、队列和迟到副作用；
- `PrimaryDrainCloseoutWorker`：只在闸门通过后写稳定关闭收据，后续再由单独归档任务解钉旧主路径。

```ts
type PostCutoverDecision =
  | "close_cutover_stable"
  | "continue_observation"
  | "rollback_to_primary"
  | "quarantine_primary_path"
  | "enter_degraded_mode"
  | "manual_review";

interface PostCutoverEvidence {
  receiptId: string;
  leaseId: string;
  observedSampleCount: number;
  recoveryErrorRatePercent: number;
  primaryWorkersActive: number;
  primaryQueueDepth: number;
  primaryOutboxPending: number;
  oldRouteCount: number;
  latePrimaryWrites: number;
  overwrittenRecoveryWrites: number;
  duplicateSideEffects: number;
  userVisibleMismatches: number;
  idempotencyKeysVerified: boolean;
  auditProjectionConsistent: boolean;
  rollbackProbePassed: boolean;
  lkgPrimaryPinned: boolean;
}

class PostCutoverObservationWorker {
  constructor(
    private readonly store: CutoverStore,
    private readonly router: RuntimeRouter,
    private readonly queues: QueueInspector,
    private readonly outbox: OutboxInspector,
    private readonly projector: AuditProjector,
  ) {}

  async tick(receiptId: string): Promise<PostCutoverDecision> {
    const receipt = await this.store.getLiveCutoverCloseout(receiptId);
    const lease = await this.store.getObservationLease(receiptId);

    const evidence: PostCutoverEvidence = {
      receiptId,
      leaseId: lease.id,
      observedSampleCount: await this.router.sampleCount(receipt.targetPath),
      recoveryErrorRatePercent: await this.router.errorRate(receipt.targetPath),
      primaryWorkersActive: await this.router.activeWorkers("primary"),
      primaryQueueDepth: await this.queues.depth("primary"),
      primaryOutboxPending: await this.outbox.pending("primary"),
      oldRouteCount: await this.router.oldRouteCount("primary"),
      latePrimaryWrites: await this.projector.countLatePrimaryWrites(receiptId),
      overwrittenRecoveryWrites: await this.projector.countOverwrites(receiptId),
      duplicateSideEffects: await this.outbox.countDuplicateSideEffects(receiptId),
      userVisibleMismatches: await this.projector.countUserVisibleMismatches(receiptId),
      idempotencyKeysVerified: await this.outbox.verifyIdempotencyKeys(receiptId),
      auditProjectionConsistent: await this.projector.isConsistent(receiptId),
      rollbackProbePassed: await this.router.rollbackProbe("primary"),
      lkgPrimaryPinned: await this.store.isLkgPinned("primary", receipt.archiveHash),
    };

    const decision = decidePostCutover(receipt, lease, evidence, Date.now());
    await this.store.appendPostCutoverEvidence(evidence, decision);

    if (decision === "rollback_to_primary") {
      await this.store.requestRollback({
        receiptId,
        reason: "post_cutover_stabilization_failed",
      });
    }

    if (decision === "quarantine_primary_path") {
      await this.store.quarantinePath({
        path: "primary",
        receiptId,
        reason: "primary_drain_or_late_effect_violation",
      });
    }

    if (decision === "close_cutover_stable") {
      await this.store.casClosePostCutover({
        receiptId,
        expectedState: "observing",
        nextState: "stable",
        evidenceHash: hashEvidence(evidence),
      });
    }

    return decision;
  }
}
```

这里的关键是 `casClosePostCutover`。稳定关闭必须是一次事务状态变更：从 `observing` 到 `stable`，绑定 evidence hash，避免两个 worker 同时关闭或一个 worker 用旧证据关闭。

## 5. OpenClaw 课程 Cron 的类比

这门课程自己的 cron 也有类似问题：

1. 课程写入 lesson 文件；
2. README 和 TOOLS 更新；
3. Telegram 发送成功；
4. git commit 成功；
5. git push 成功。

如果 Telegram 已发送但 git push 失败，不能说“任务成功”；如果 git push 成功但 TOOLS 没更新，下一次可能重复选题；如果 README 更新了但 lesson 文件没进 commit，目录会指向不存在的课。

所以这类自动化也需要 post-cutover 思维：

- `messageId` 是外部副作用收据；
- commit sha 是仓库状态收据；
- remote branch head 是传播收据；
- README / TOOLS / lesson 文件是本地投影；
- 最后要对账四者是否指向同一个 lesson slug。

这就是 Agent 工程里最常见的经验：**只要任务跨了多个现实系统，就不要把“某一步成功”当成“整个任务稳定”。**

## 6. 实战检查清单

设计 Post-Cutover Stabilization Gate 时，可以用这 8 个问题自检：

1. 切流成功收据是否有唯一 `receipt_id`？
2. 观察窗口有没有最小时长和最小样本数？
3. 旧 primary worker 是否全部停止消费？
4. primary queue 和 outbox 是否归零或被迁移？
5. 迟到写入是否通过版本号 / CAS / idempotency key 对账？
6. 新路径是否持续满足错误率、延迟和成本预算？
7. 回滚点是否仍然 pinned，rollback probe 是否仍然通过？
8. 稳定关闭是否写入不可变 receipt，而不是只改一个状态字段？

## 7. 常见坑

- **只看 control plane 指针**：控制面显示 recovery，不代表所有 runtime 都看见了 recovery。
- **过早解钉 primary**：旧主路径一旦删掉，迟到副作用和 rollback 都会变成手工考古。
- **忽略 outbox**：最危险的不是队列里还有任务，而是 outbox 里还有会影响现实世界的副作用。
- **没有 late effect ledger**：迟到写入不入账，后面就分不清是正常延迟还是数据回潮。
- **关闭没有 evidence hash**：关闭收据不绑定证据，审计时无法证明当时为什么认为稳定。

## 8. 总结

第 503 课的核心：

> 切流成功只是进入新现实，稳定关闭才证明旧现实已经排空。

用一条证据链表达：

```text
LiveCutoverCloseoutReceipt
  -> PostCutoverObservationLease
  -> PrimaryDrainReport
  -> LateEffectReconciliation
  -> RollbackReadinessSnapshot
  -> PostCutoverStabilizationReceipt
```

成熟 Agent 做生产切换时，不会在“按钮按下成功”后马上庆祝。它会继续观察新路径、排空旧路径、对账迟到副作用、保留可回滚点，最后用一张稳定关闭收据证明：现在真的可以把这次 cutover 当成完成。
