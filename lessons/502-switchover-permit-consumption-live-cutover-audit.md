# 502. Agent 切换许可消耗与真实切流审计闸门（Switchover Permit Consumption & Live Cutover Audit Gate）

上一课讲的是 **Restore Path Failover Drill & Switchover Permit Gate**：恢复路径 probe 通过以后，还要通过隔离故障切换演练、fencing、队列重放、只读下游、回滚 probe、容量和 owner ack，才能签发一次性 `SwitchoverPermit`。

这一课继续往后走：**拿到 permit 不是已经切换成功，只是允许进入真实切流动作。**

生产事故里很常见的危险状态是：

- permit 发出来了，但被重复消费；
- control plane 说切到 recovery，runtime 里还有一半 worker 在 primary；
- queue cursor 移动了，但 outbox 没 drain；
- 下游只读模式开启了，但某些工具仍然能产生副作用；
- 切换失败后没有稳定回退点，只能靠人工猜。

所以第 502 课是 **Switchover Permit Consumption & Live Cutover Audit Gate**：把“一次性切换许可”变成“真实切流、证据审计、运行时一致、可关闭或可回退”的完整闭环。

## 1. 为什么 permit 还要被审计

`SwitchoverPermit` 只证明：

- 演练通过；
- 容量够；
- owner 同意；
- LKG primary 或 fallback 还可用；
- permit 在有效期内。

但它不证明：

- permit 只被消费一次；
- 所有 runtime 都加载了同一个 target path；
- task queue、outbox、scheduler、cache 都完成 cutover；
- 用户可见状态和审计日志一致；
- 切换后错误率、延迟、成本没有越界；
- 失败时可以自动 rollback 或进入受控 degraded mode。

成熟 Agent 的切换系统不能停在“批准切换”，要能证明“切换已经按许可发生，并且每个入口都看见了同一个现实”。

```text
SwitchoverPermit
  -> PermitConsumptionReceipt
  -> LiveCutoverExecution
  -> RuntimeCutoverAudit
  -> CutoverHealthSnapshot
  -> LiveCutoverCloseoutReceipt
```

核心原则：**permit 是授权，consumption 是原子状态变更，audit 是事实对账。**

## 2. 最小数据模型

```python
from dataclasses import dataclass
from typing import Literal

Decision = Literal[
    "close_cutover_stable",
    "continue_cutover_observation",
    "rollback_to_primary",
    "enter_degraded_mode",
    "quarantine_cutover",
    "manual_review",
]


@dataclass(frozen=True)
class SwitchoverPermit:
    permit_id: str
    plan_id: str
    archive_hash: str
    restore_path_version: str
    target_path: Literal["recovery", "primary"]
    max_traffic_percent: int
    expires_at_epoch: int
    consumed_at_epoch: int | None


@dataclass(frozen=True)
class PermitConsumptionReceipt:
    receipt_id: str
    permit_id: str
    consumed_once: bool
    consumed_by_worker_id: str
    compare_and_swap_succeeded: bool
    traffic_percent_requested: int
    created_at_epoch: int


@dataclass(frozen=True)
class LiveCutoverExecution:
    permit_id: str
    target_path: Literal["recovery", "primary"]
    control_plane_pointer_updated: bool
    scheduler_paused: bool
    queue_cursor_frozen: bool
    outbox_drained: bool
    runtime_rollout_percent: int
    downstream_write_gate_closed: bool
    rollback_point_pinned: bool


@dataclass(frozen=True)
class RuntimeCutoverAudit:
    permit_id: str
    expected_archive_hash: str
    observed_archive_hashes: set[str]
    expected_restore_path_version: str
    observed_restore_path_versions: set[str]
    stale_worker_count: int
    cache_invalidated: bool
    active_queue_cursor: str
    audited_queue_cursor: str
    forbidden_side_effects: int
    audit_log_complete: bool


@dataclass(frozen=True)
class CutoverHealthSnapshot:
    permit_id: str
    sample_count: int
    error_rate_percent: float
    p95_latency_ms: int
    cost_multiplier: float
    user_visible_mismatch_count: int
    rollback_probe_passed: bool
    alert_route_confirmed: bool
```

这里最重要的对象是 `PermitConsumptionReceipt`。真实系统里，permit 必须通过 CAS 或事务锁从 `available` 变成 `consumed`，不能靠 worker 自己说“我只用了一次”。

## 3. learn-claude-code：纯函数闸门

教学版仍然先写纯函数。输入是 permit、消费收据、执行记录、运行时审计和健康快照，输出下一步动作。

```python
def decide_live_cutover_closeout(
    permit: SwitchoverPermit,
    consumption: PermitConsumptionReceipt,
    execution: LiveCutoverExecution,
    audit: RuntimeCutoverAudit,
    health: CutoverHealthSnapshot,
    now_epoch: int,
) -> Decision:
    if now_epoch >= permit.expires_at_epoch and permit.consumed_at_epoch is None:
        return "manual_review"

    if consumption.permit_id != permit.permit_id:
        return "manual_review"

    if execution.permit_id != permit.permit_id:
        return "manual_review"

    if audit.permit_id != permit.permit_id:
        return "manual_review"

    if health.permit_id != permit.permit_id:
        return "manual_review"

    if not consumption.consumed_once:
        return "quarantine_cutover"

    if not consumption.compare_and_swap_succeeded:
        return "quarantine_cutover"

    if consumption.traffic_percent_requested > permit.max_traffic_percent:
        return "rollback_to_primary"

    if execution.target_path != permit.target_path:
        return "manual_review"

    if not execution.scheduler_paused:
        return "enter_degraded_mode"

    if not execution.queue_cursor_frozen:
        return "enter_degraded_mode"

    if not execution.outbox_drained:
        return "continue_cutover_observation"

    if not execution.control_plane_pointer_updated:
        return "manual_review"

    if execution.runtime_rollout_percent < consumption.traffic_percent_requested:
        return "continue_cutover_observation"

    if not execution.downstream_write_gate_closed:
        return "quarantine_cutover"

    if not execution.rollback_point_pinned:
        return "manual_review"

    if audit.observed_archive_hashes != {audit.expected_archive_hash}:
        return "rollback_to_primary"

    if audit.observed_restore_path_versions != {audit.expected_restore_path_version}:
        return "rollback_to_primary"

    if audit.expected_archive_hash != permit.archive_hash:
        return "manual_review"

    if audit.expected_restore_path_version != permit.restore_path_version:
        return "manual_review"

    if audit.stale_worker_count > 0:
        return "continue_cutover_observation"

    if not audit.cache_invalidated:
        return "continue_cutover_observation"

    if audit.active_queue_cursor != audit.audited_queue_cursor:
        return "enter_degraded_mode"

    if audit.forbidden_side_effects > 0:
        return "quarantine_cutover"

    if not audit.audit_log_complete:
        return "manual_review"

    if health.sample_count < 100:
        return "continue_cutover_observation"

    if health.user_visible_mismatch_count > 0:
        return "rollback_to_primary"

    if health.error_rate_percent > 2.0:
        return "rollback_to_primary"

    if health.p95_latency_ms > 5000:
        return "enter_degraded_mode"

    if health.cost_multiplier > 2.0:
        return "continue_cutover_observation"

    if not health.rollback_probe_passed:
        return "manual_review"

    if not health.alert_route_confirmed:
        return "continue_cutover_observation"

    return "close_cutover_stable"
```

几个判断顺序值得注意：

1. 先验证所有对象绑定同一个 permit，避免用旧审计给新切换背书。
2. permit 重复消费或 CAS 失败直接隔离，因为这代表控制面状态已经不可信。
3. 流量超过许可上限要回退，不让恢复路径在事故中被过载打穿。
4. queue cursor 不一致进入 degraded mode，因为继续正常写入可能扩大不一致。
5. 运行时 hash/version 漂移直接回 primary，说明 recovery path 没有统一。
6. rollback probe 没通过时不自动回滚，因为回滚路径本身也可能坏了。

## 4. pi-mono：Live Cutover Worker

生产版建议把切换执行做成一个 worker，入口只接受 `permitId` 和 `requestedTrafficPercent`。worker 第一件事不是切流，而是事务化消费 permit。

```ts
type CutoverDecision =
  | "close_cutover_stable"
  | "continue_cutover_observation"
  | "rollback_to_primary"
  | "enter_degraded_mode"
  | "quarantine_cutover"
  | "manual_review";

interface SwitchoverPermitRecord {
  permitId: string;
  status: "available" | "consumed" | "expired" | "revoked";
  archiveHash: string;
  restorePathVersion: string;
  targetPath: "recovery" | "primary";
  maxTrafficPercent: number;
  expiresAt: Date;
  consumedAt?: Date;
}

interface PermitConsumptionReceipt {
  receiptId: string;
  permitId: string;
  consumedOnce: boolean;
  compareAndSwapSucceeded: boolean;
  trafficPercentRequested: number;
  createdAt: Date;
}

class LiveCutoverWorker {
  constructor(
    private readonly permits: SwitchoverPermitStore,
    private readonly controlPlane: ControlPlane,
    private readonly runtimeAudit: RuntimeCutoverAuditor,
    private readonly healthProbe: CutoverHealthProbe,
    private readonly receipts: CutoverReceiptStore,
    private readonly outbox: Outbox,
  ) {}

  async run(input: {
    permitId: string;
    requestedTrafficPercent: number;
    workerId: string;
  }): Promise<CutoverDecision> {
    const permit = await this.permits.get(input.permitId);

    const consumption = await this.permits.consumeOnce({
      permitId: input.permitId,
      workerId: input.workerId,
      requestedTrafficPercent: input.requestedTrafficPercent,
    });

    if (!consumption.compareAndSwapSucceeded || !consumption.consumedOnce) {
      await this.outbox.publish({
        type: "CUTOVER_QUARANTINED",
        permitId: input.permitId,
        reason: "permit_consumption_not_unique",
      });
      return "quarantine_cutover";
    }

    const execution = await this.controlPlane.executeCutover({
      targetPath: permit.targetPath,
      archiveHash: permit.archiveHash,
      restorePathVersion: permit.restorePathVersion,
      trafficPercent: input.requestedTrafficPercent,
    });

    const audit = await this.runtimeAudit.collect({
      permitId: permit.permitId,
      expectedArchiveHash: permit.archiveHash,
      expectedRestorePathVersion: permit.restorePathVersion,
    });

    const health = await this.healthProbe.sample({
      permitId: permit.permitId,
      targetPath: permit.targetPath,
    });

    const decision = decideLiveCutoverCloseout(
      toDomainPermit(permit),
      toDomainConsumption(consumption),
      execution,
      audit,
      health,
      Date.now(),
    );

    await this.receipts.writeCloseout({
      permitId: permit.permitId,
      decision,
      execution,
      audit,
      health,
    });

    if (decision === "rollback_to_primary") {
      await this.controlPlane.rollbackToPrimary({
        permitId: permit.permitId,
        rollbackPointRequired: true,
      });
    }

    if (decision === "enter_degraded_mode") {
      await this.controlPlane.enterDegradedMode({
        permitId: permit.permitId,
        allowReads: true,
        allowWrites: false,
      });
    }

    return decision;
  }
}
```

这里有三个生产细节：

- `consumeOnce` 必须是数据库事务、Redis Lua、etcd txn 或其他原子 CAS，不能分成 read then write。
- `executeCutover` 要先暂停 scheduler、冻结 queue cursor、drain outbox，再移动 control plane pointer。
- `runtimeAudit.collect` 要从真实 worker、cache、scheduler、queue consumer 采样，不只读控制面配置。

## 5. OpenClaw 课程 Cron：真实切流类比

我们这个课程 Cron 也有类似模式，只是规模小很多：

```text
选题许可：TOOLS.md 已讲内容确认没有重复
消费许可：创建 lessons/502-...md
执行切换：README.md 目录追加新课
运行时审计：git diff 检查 lesson、README、TOOLS 三处一致
关闭收据：git commit && git push
对外送达：Telegram 群组消息
```

如果把它做得更工程化，可以给每次课程生成一个 `LessonDeliveryReceipt`：

```json
{
  "lessonNumber": 502,
  "topic": "Switchover Permit Consumption & Live Cutover Audit Gate",
  "dedupeChecked": true,
  "lessonFileCreated": true,
  "readmeUpdated": true,
  "toolsUpdated": true,
  "gitPushed": true,
  "telegramDelivered": true
}
```

这样下一次 Cron 不只靠“看起来已经写了”，而是能用 receipt 判断哪些动作已完成、哪些动作需要补偿。

## 6. 常见坑

**坑 1：permit 可重复使用。**  
一次性 permit 没有原子消费，就可能两个 worker 同时切流。解决方式是 CAS 消费，并把 `PermitConsumptionReceipt` 写成审计证据。

**坑 2：只切 control plane，不查 runtime。**  
配置中心显示 recovery，不代表 worker 都加载了 recovery。必须采样 runtime fingerprint。

**坑 3：队列 cursor 没冻结。**  
切换期间如果 primary 和 recovery 同时推进 cursor，后面很难证明到底谁处理了哪些任务。

**坑 4：outbox 未 drain。**  
旧路径的副作用事件还没发完，新路径已经开始写，会产生重复通知、重复扣费、重复任务。

**坑 5：回滚点没 pin。**  
切换失败才发现 primary 已经被清理或覆盖，是恢复系统最糟糕的事故形态之一。

## 7. 实战检查清单

设计真实切流时，至少问这 8 个问题：

- permit 是否只能被 CAS 消费一次？
- 切流流量是否小于 permit 许可上限？
- scheduler 是否先暂停？
- queue cursor 是否冻结并审计？
- outbox 是否 drain？
- 下游写入 gate 是否关闭？
- runtime worker 是否全部加载同一 archive hash 和 path version？
- rollback probe 是否在切流后仍然通过？

## 8. 记忆点

> Switchover permit 不是切换结果，而是切换授权。

第 501 课解决的是“能不能签发切换许可”。  
第 502 课解决的是“许可被消费后，真实生产路径有没有一致地切过去，并且能不能被证据关闭或回退”。

成熟 Agent 的生产切换不是一个按钮，而是一条证据链：许可、消费、执行、审计、健康、关闭。缺一环，就不要假装切换已经完成。
