# 501. Agent 恢复路径故障切换演练与切换许可闸门（Restore Path Failover Drill & Switchover Permit Gate）

上一课讲的是 **Active Restore Path Drift Revalidation Gate**：恢复路径已经收敛后，仍然要周期复核 runner、index、cache、standby、alert route 有没有漂移。

这一课继续往后走：**复核通过只能证明恢复路径“看起来还在”，不能证明事故真的来时它能接管主路径。**

恢复系统最危险的错觉是：probe 都是绿的，但真正需要从 primary 切到 recovery path 时，lease 抢不到、队列没 drain、幂等键冲突、下游只读模式没打开、owner 通知没到，最后恢复路径卡在“理论可用”。

所以第 501 课是 **Restore Path Failover Drill & Switchover Permit Gate**：把恢复路径从“周期探测可用”推进到“可演练、可预授权、可真实切换、可回退”的生产切换能力。

## 1. 为什么 probe 通过还不够

`RestorePathDriftCloseoutReceipt` 证明的是：

- archive hash 没漂移；
- restore path version 没漂移；
- standby restore probe 通过；
- alert route 还可用；
- runtime 没拿旧缓存。

但它没有证明：

- primary path 失效时，恢复路径能不能拿到独占切换锁；
- 正在执行的任务能不能 drain 或 fencing；
- 外部副作用能不能进入 read-only / dry-run / idempotent retry；
- recovery worker 是否真的能消费积压队列；
- 切换后用户可见状态是否一致；
- 切换失败时能不能退回 LKG primary 或保持安全降级。

成熟 Agent 的恢复系统不能只会“检查备用轮胎还在”，还要定期证明“真的能换上去开一段，并且能换回来”。

```text
RestorePathDriftCloseoutReceipt
  -> FailoverDrillPlan
  -> IsolatedFailoverDrillRun
  -> SwitchoverReadinessReview
  -> SwitchoverPermit
  -> FailoverDrillCloseoutReceipt
```

关键思想：**probe 验证存在性，failover drill 验证接管能力。**

## 2. 最小数据模型

```python
from dataclasses import dataclass
from typing import Literal

Decision = Literal[
    "issue_switchover_permit",
    "extend_drill",
    "repair_restore_path",
    "quarantine_failover_lane",
    "manual_review",
]


@dataclass(frozen=True)
class RestorePathDriftCloseoutReceipt:
    receipt_id: str
    convergence_receipt_id: str
    lease_id: str
    decision: Literal["renew_audit_lease", "refresh_probe_only"]
    expected_archive_hash: str
    observed_archive_hash: str
    expected_restore_path_version: str
    observed_restore_path_version: str


@dataclass(frozen=True)
class FailoverDrillPlan:
    plan_id: str
    drift_closeout_receipt_id: str
    archive_hash: str
    restore_path_version: str
    scenario: Literal["primary_timeout", "queue_stall", "region_loss", "policy_freeze"]
    max_side_effects: int
    require_read_only_downstream: bool
    expires_at_epoch: int


@dataclass(frozen=True)
class IsolatedFailoverDrillRun:
    plan_id: str
    acquired_switchover_lock: bool
    primary_fenced: bool
    queue_replayed: bool
    idempotency_preserved: bool
    downstream_read_only: bool
    user_visible_state_matched: bool
    rollback_probe_passed: bool
    side_effects_attempted: int
    runtime_fingerprint_matched: bool
    alert_delivered: bool


@dataclass(frozen=True)
class SwitchoverReadinessReview:
    plan_id: str
    owner_ack: bool
    no_open_restore_incident: bool
    no_policy_freeze: bool
    lkg_primary_available: bool
    capacity_headroom_percent: int
```

这里故意把 `SwitchoverPermit` 从 `FailoverDrillPlan` 拆开：演练计划不是切换许可。只有演练结果、人工/策略复核、容量和回滚点都满足后，系统才发一次性 permit。

## 3. learn-claude-code：纯函数闸门

教学版先写成纯函数，方便单元测试。它不发消息、不切流量，只决定下一步动作。

```python
def decide_failover_drill_switchover_permit(
    closeout: RestorePathDriftCloseoutReceipt,
    plan: FailoverDrillPlan,
    drill: IsolatedFailoverDrillRun,
    review: SwitchoverReadinessReview,
    now_epoch: int,
) -> Decision:
    if plan.drift_closeout_receipt_id != closeout.receipt_id:
        return "manual_review"

    if drill.plan_id != plan.plan_id or review.plan_id != plan.plan_id:
        return "manual_review"

    if closeout.expected_archive_hash != closeout.observed_archive_hash:
        return "repair_restore_path"

    if closeout.expected_restore_path_version != closeout.observed_restore_path_version:
        return "quarantine_failover_lane"

    if plan.archive_hash != closeout.expected_archive_hash:
        return "manual_review"

    if plan.restore_path_version != closeout.expected_restore_path_version:
        return "manual_review"

    if now_epoch >= plan.expires_at_epoch:
        return "extend_drill"

    if drill.side_effects_attempted > plan.max_side_effects:
        return "quarantine_failover_lane"

    if plan.require_read_only_downstream and not drill.downstream_read_only:
        return "quarantine_failover_lane"

    if not drill.acquired_switchover_lock:
        return "extend_drill"

    if not drill.primary_fenced:
        return "quarantine_failover_lane"

    if not drill.queue_replayed:
        return "repair_restore_path"

    if not drill.idempotency_preserved:
        return "quarantine_failover_lane"

    if not drill.user_visible_state_matched:
        return "repair_restore_path"

    if not drill.rollback_probe_passed:
        return "manual_review"

    if not drill.runtime_fingerprint_matched:
        return "repair_restore_path"

    if not drill.alert_delivered:
        return "extend_drill"

    if not review.no_open_restore_incident:
        return "manual_review"

    if not review.no_policy_freeze:
        return "manual_review"

    if not review.lkg_primary_available:
        return "manual_review"

    if review.capacity_headroom_percent < 30:
        return "extend_drill"

    if not review.owner_ack:
        return "manual_review"

    return "issue_switchover_permit"
```

几个关键顺序：

1. 先验证所有证据绑定同一个 closeout 和 plan，避免拿旧 probe 给新切换背书。
2. archive hash 不一致先修恢复路径，path version 不一致先隔离切换车道。
3. 副作用超预算、下游没只读、primary 没 fencing，都不能发 permit。
4. rollback probe 没通过不自动修，因为切换和回退都可能有风险，要人工复核。
5. capacity 不足不是失败，是继续演练或等待扩容。

## 4. pi-mono：Failover Drill Worker

生产版建议把它做成独立 worker：读取 drift closeout，创建演练计划，在隔离环境里跑切换，然后只在闸门通过时签发一次性 `SwitchoverPermit`。

```ts
type FailoverDecision =
  | "issue_switchover_permit"
  | "extend_drill"
  | "repair_restore_path"
  | "quarantine_failover_lane"
  | "manual_review";

interface SwitchoverPermit {
  permitId: string;
  planId: string;
  archiveHash: string;
  restorePathVersion: string;
  scenario: string;
  maxTrafficPercent: number;
  expiresAt: string;
  consumedAt?: string;
}

interface FailoverDrillCloseoutReceipt {
  receiptId: string;
  driftCloseoutReceiptId: string;
  planId: string;
  decision: FailoverDecision;
  permitId?: string;
  reasons: string[];
  createdAt: string;
}

class RestoreFailoverDrillWorker {
  constructor(
    private readonly store: RestoreOpsStore,
    private readonly drillRunner: IsolatedFailoverRunner,
    private readonly permits: SwitchoverPermitStore,
    private readonly quarantine: RestoreLaneQuarantine,
    private readonly outbox: Outbox,
  ) {}

  async handle(driftCloseoutReceiptId: string): Promise<FailoverDrillCloseoutReceipt> {
    const closeout = await this.store.getDriftCloseout(driftCloseoutReceiptId);
    const plan = await this.store.createFailoverDrillPlan({
      driftCloseoutReceiptId,
      archiveHash: closeout.expectedArchiveHash,
      restorePathVersion: closeout.expectedRestorePathVersion,
      scenario: "primary_timeout",
      maxSideEffects: 0,
      requireReadOnlyDownstream: true,
    });

    const drill = await this.drillRunner.run(plan);
    const review = await this.store.buildSwitchoverReadinessReview(plan.planId);
    const decision = decideFailoverDrillSwitchoverPermit(closeout, plan, drill, review);

    let permit: SwitchoverPermit | undefined;

    if (decision === "issue_switchover_permit") {
      permit = await this.permits.createOnce({
        planId: plan.planId,
        archiveHash: plan.archiveHash,
        restorePathVersion: plan.restorePathVersion,
        scenario: plan.scenario,
        maxTrafficPercent: 10,
        expiresAt: addMinutes(new Date(), 30).toISOString(),
      });
    }

    if (decision === "quarantine_failover_lane") {
      await this.quarantine.block({
        lane: plan.scenario,
        reason: "failover_drill_failed_safety_gate",
        planId: plan.planId,
      });
    }

    const receipt = await this.store.writeFailoverDrillCloseoutReceipt({
      driftCloseoutReceiptId,
      planId: plan.planId,
      decision,
      permitId: permit?.permitId,
      reasons: explainFailoverDecision(decision, drill, review),
    });

    await this.outbox.publish("restore.failover_drill.closed", receipt);
    return receipt;
  }
}
```

注意这里的 `SwitchoverPermitStore.createOnce` 要做唯一约束：

```text
unique(planId, archiveHash, restorePathVersion, scenario)
```

同一份演练结果只能生成一张 permit，真实切换 worker 消费 permit 时还要原子写 `consumedAt`，防止两个 worker 同时切流量。

## 5. OpenClaw 实战：课程 Cron 的“切换许可”类比

这套逻辑可以直接类比到我们的课程 cron：

- primary path：本轮 agent 负责写 lesson、README、TOOLS、git push、Telegram 发送；
- recovery path：如果本轮中断，下一轮 cron 或接管 worker 从 git status / README / TOOLS / Telegram messageId 继续；
- failover drill：在 dry-run 里模拟“写完文件但还没发 Telegram”“发了 Telegram 但 git push 失败”“commit 成功但 TOOLS 没更新”；
- switchover permit：只有确认幂等键、文件状态、远端 commit、消息投递状态都能对账，才允许 recovery worker 继续。

最小运行记录可以长这样：

```json
{
  "runId": "agent-course-2026-06-17T06:30:00+11:00",
  "lesson": 501,
  "restorePath": "next-cron-or-manual-takeover",
  "drill": {
    "scenario": "telegram_sent_git_push_failed",
    "idempotencyKey": "lesson-501-restore-path-failover",
    "fileReplayOk": true,
    "messageReplayBlocked": true,
    "gitReplayOk": true
  },
  "permit": {
    "decision": "issue_switchover_permit",
    "maxTrafficPercent": 10,
    "expiresInMinutes": 30
  }
}
```

这不是为了把课程 cron 复杂化，而是训练一个生产习惯：**每个长期自动化都要知道自己失败到一半时，谁能接管、从哪接管、哪些动作不能重复。**

## 6. 测试清单

`learn-claude-code` 里至少测这些 fixture：

```python
def test_issue_permit_when_drill_and_review_are_clean():
    assert decision == "issue_switchover_permit"


def test_quarantine_when_primary_not_fenced():
    assert decision == "quarantine_failover_lane"


def test_repair_when_queue_replay_fails():
    assert decision == "repair_restore_path"


def test_manual_review_when_rollback_probe_fails():
    assert decision == "manual_review"


def test_extend_when_capacity_is_low():
    assert decision == "extend_drill"


def test_reject_mismatched_plan_binding():
    assert decision == "manual_review"
```

`pi-mono` 里再补集成测试：

- 同一个 plan 只能创建一张 permit；
- permit 过期后不能被消费；
- quarantine 车道后真实切换 worker 必须拒绝执行；
- drill runner 禁止真实外部副作用；
- outbox 事件失败可重试但不重复签发 permit；
- owner ack 缺失时只写 closeout，不写 permit。

## 7. 常见坑

1. **把演练通过当成永久许可**
   Permit 必须短 TTL、一次性消费、绑定 archive hash 和 restore path version。

2. **只测恢复，不测回退**
   真正危险的是切过去以后发现更坏，却没有 rollback probe。

3. **演练环境太假**
   可以隔离副作用，但 runtime fingerprint、queue shape、policy state、tool schema 必须接近真实。

4. **没有 fencing primary**
   recovery path 接管时，如果 primary 还能继续写外部副作用，就会制造双写和重复通知。

5. **没有容量门槛**
   恢复路径能跑 1 个测试任务，不代表能吃下真实 backlog。

## 8. 记忆句

Probe 证明恢复路径还活着。

Drill 证明恢复路径能接管。

Permit 证明这次接管被授权、限域、可回退。

一句话：**成熟 Agent 的恢复能力，不是“备用路径存在”，而是备用路径定期演练过、接管前有许可、接管中有 fencing、接管后能回退。**
