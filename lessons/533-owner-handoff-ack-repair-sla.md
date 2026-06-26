# 533. Agent Owner 交接确认与修复 SLA 闸门

> OwnerHandoffReceipt 只能证明失败演练已经被交给某个负责人，不证明负责人真的接手、修复时限已经开始、逾期会升级、修复证据会回流。成熟 Agent 会把交接后的 ack、RepairSlaLease、进度心跳、逾期升级和 closeout 串成闭环，防止失败任务躺在工单系统里无人处理。

上一讲讲了 **Drill Failure Root Cause Packet & Owner Handoff Gate**：演练失败后，要把证据、复现命令、影响范围和验收条件打包给正确 owner。

今天继续往后走：**交给 owner 以后，怎么证明不是“已派单”等于“已解决”？**

常见问题是：

- 工单创建了，但 owner 没确认接手；
- SLA 从创建工单开始算，结果 owner 半天后才看到，责任边界不清；
- 修复过程没有 heartbeat，直到 deadline 过了才发现卡住；
- 高风险失败没有逐级升级，只在群里反复 @；
- owner 修复了，但没有把 repair evidence 回写到后续验证闸门。

所以要加一层 **Owner Handoff Acknowledgement & Repair SLA Gate**：交接不是终点，而是修复责任的起点。

---

## 1. 核心链路

```text
OwnerHandoffReceipt
        ↓
OwnerAcknowledgement
        ↓
RepairSlaLease
        ↓
RepairProgressHeartbeat
        ↓
EscalationDecision
        ↓
RepairSlaCloseoutReceipt
```

每一环回答一个问题：

- handoff receipt：失败是否已经交给候选 owner；
- owner acknowledgement：owner 是否确认接手，是否接受范围；
- repair SLA lease：修复时限、升级路径、暂停条件是否明确；
- progress heartbeat：修复是否持续推进，是否卡在外部依赖；
- escalation decision：未 ack、逾期、范围争议或高风险是否升级；
- closeout receipt：修复责任是否进入下一道验证闸门。

一句话：**handoff 是通知，ack + SLA 才是责任。没有 RepairSlaLease 的 owner 交接，只是把问题从 Agent 系统移动到了人的待办列表。**

---

## 2. 什么时候必须升级？

Owner 交接后至少要处理 5 类异常：

```text
异常                      处理动作
────────────────────────────────────────────────────────
未确认接手                 page owner / reroute backup owner
拒绝负责或范围争议           split case / escalate incident commander
SLA 即将到期但无进度          page owner + raise priority
SLA 已逾期                  escalate manager / block release
修复声称完成但无证据          request repair evidence
```

这里的重点是：**升级不是情绪动作，而是状态机动作。**

不要在群里反复喊“谁看一下”，而是写出清晰的 lease：

```text
repairSlaLease
  owner: cold-path-oncall
  ackDeadline: 2026-06-27T04:00:00Z
  repairDeadline: 2026-06-27T10:00:00Z
  severity: high
  escalationRoute: incident-commander
  acceptanceCriteria:
    - seed-cold-17 passes
    - lookup key archive:tenant-a:2026-06 returns within 1200ms
    - repair evidence linked to root cause case
```

---

## 3. learn-claude-code：ACK 与 SLA 判定器

教学版依旧先写纯函数。它不发消息、不改工单，只根据交接、确认、心跳和当前时间判断下一步动作。

```python
from dataclasses import dataclass
from datetime import datetime
from typing import Literal

Severity = Literal["low", "medium", "high", "critical"]

Decision = Literal[
    "wait_for_ack",
    "issue_repair_sla_lease",
    "continue_monitoring",
    "page_owner",
    "reroute_backup_owner",
    "escalate_incident_commander",
    "block_release",
    "request_repair_evidence",
    "close_sla_ready_for_verification",
    "manual_review",
]

@dataclass
class OwnerHandoffReceipt:
    packet_id: str
    owner: str
    backup_owner: str | None
    severity: Severity
    handoff_at: datetime
    ack_deadline: datetime
    repair_deadline: datetime
    release_blocking: bool

@dataclass
class OwnerAcknowledgement:
    acknowledged: bool
    accepted_scope: bool
    acknowledged_at: datetime | None
    dispute_reason: str | None

@dataclass
class RepairProgressHeartbeat:
    last_seen_at: datetime | None
    status: Literal["not_started", "in_progress", "blocked", "claimed_fixed"]
    blocker: str | None
    repair_evidence_refs: list[str]
    verification_requested: bool

def decide_owner_ack_sla_gate(
    handoff: OwnerHandoffReceipt,
    ack: OwnerAcknowledgement,
    heartbeat: RepairProgressHeartbeat,
    now: datetime,
) -> Decision:
    if not ack.acknowledged:
        if now < handoff.ack_deadline:
            return "wait_for_ack"
        if handoff.backup_owner:
            return "reroute_backup_owner"
        return "page_owner"

    if not ack.accepted_scope:
        if handoff.severity in ("high", "critical"):
            return "escalate_incident_commander"
        return "manual_review"

    if heartbeat.status == "claimed_fixed":
        if not heartbeat.repair_evidence_refs:
            return "request_repair_evidence"
        if heartbeat.verification_requested:
            return "close_sla_ready_for_verification"
        return "request_repair_evidence"

    if now >= handoff.repair_deadline:
        if handoff.release_blocking or handoff.severity == "critical":
            return "block_release"
        return "escalate_incident_commander"

    if heartbeat.last_seen_at is None:
        return "issue_repair_sla_lease"

    minutes_since_heartbeat = (now - heartbeat.last_seen_at).total_seconds() / 60
    if minutes_since_heartbeat > 120 and handoff.severity in ("high", "critical"):
        return "page_owner"

    if heartbeat.status == "blocked":
        return "escalate_incident_commander"

    return "continue_monitoring"
```

对应测试：

```python
from datetime import datetime, timedelta, timezone

def test_blocks_release_when_repair_deadline_passed_for_blocking_case():
    now = datetime(2026, 6, 27, 10, 30, tzinfo=timezone.utc)
    handoff = OwnerHandoffReceipt(
        packet_id="rcp-532",
        owner="cold-path-oncall",
        backup_owner="platform-oncall",
        severity="high",
        handoff_at=now - timedelta(hours=8),
        ack_deadline=now - timedelta(hours=7),
        repair_deadline=now - timedelta(minutes=30),
        release_blocking=True,
    )
    ack = OwnerAcknowledgement(
        acknowledged=True,
        accepted_scope=True,
        acknowledged_at=now - timedelta(hours=7, minutes=30),
        dispute_reason=None,
    )
    heartbeat = RepairProgressHeartbeat(
        last_seen_at=now - timedelta(hours=3),
        status="in_progress",
        blocker=None,
        repair_evidence_refs=[],
        verification_requested=False,
    )

    assert decide_owner_ack_sla_gate(handoff, ack, heartbeat, now) == "block_release"
```

这个判定器故意把 `claimed_fixed` 和 `close_sla_ready_for_verification` 分开：owner 说修好了，只能进入验证闸门，不能直接关闭。

---

## 4. pi-mono：RepairSlaCoordinator

生产版可以把它做成 RootCauseHandoffWorker 之后的协调器：消费 OwnerHandoffReceipt，创建 SLA lease，监听 owner ack 与 heartbeat，逾期时写 outbox 事件。

```typescript
type RepairSlaDecision =
  | "wait_for_ack"
  | "issue_repair_sla_lease"
  | "continue_monitoring"
  | "page_owner"
  | "reroute_backup_owner"
  | "escalate_incident_commander"
  | "block_release"
  | "request_repair_evidence"
  | "close_sla_ready_for_verification"
  | "manual_review";

interface RepairSlaLease {
  leaseId: string;
  packetId: string;
  owner: string;
  ackDeadline: string;
  repairDeadline: string;
  severity: "low" | "medium" | "high" | "critical";
  releaseBlocking: boolean;
  escalationRoute: string;
  fencingToken: number;
}

interface RepairSlaCloseoutReceipt {
  receiptId: string;
  leaseId: string;
  decision: RepairSlaDecision;
  evidenceRefs: string[];
  nextGate: "repair_verification" | "escalation" | "release_blocker" | "manual_review";
  createdAt: string;
}

class RepairSlaCoordinator {
  constructor(
    private readonly leases: RepairSlaLeaseStore,
    private readonly ownerAck: OwnerAckStore,
    private readonly heartbeat: RepairHeartbeatStore,
    private readonly decisionEngine: RepairSlaDecisionEngine,
    private readonly outbox: Outbox,
    private readonly audit: AuditLog,
  ) {}

  async tick(packetId: string, now: Date): Promise<void> {
    const lease = await this.leases.getOrCreateForPacket(packetId);
    const ack = await this.ownerAck.get(packetId);
    const progress = await this.heartbeat.latest(packetId);

    const decision = this.decisionEngine.decide({ lease, ack, progress, now });

    await this.audit.append({
      type: "repair_sla_decision",
      packetId,
      leaseId: lease.leaseId,
      decision,
      at: now.toISOString(),
      fencingToken: lease.fencingToken,
    });

    switch (decision) {
      case "page_owner":
        await this.outbox.publish("owner.page", { packetId, owner: lease.owner });
        break;

      case "reroute_backup_owner":
        await this.outbox.publish("owner.reroute", { packetId, from: lease.owner });
        break;

      case "escalate_incident_commander":
        await this.outbox.publish("incident.escalate", { packetId, route: lease.escalationRoute });
        break;

      case "block_release":
        await this.outbox.publish("release.block", { packetId, reason: "repair_sla_expired" });
        break;

      case "close_sla_ready_for_verification":
        await this.outbox.publish("repair.verification.requested", {
          packetId,
          leaseId: lease.leaseId,
          evidenceRefs: progress.repairEvidenceRefs,
        });
        break;
    }
  }
}
```

生产实现有三个关键细节：

1. **getOrCreateForPacket 必须幂等**：同一个 root cause packet 只能有一个 active SLA lease；
2. **fencingToken 防并发 worker 双重升级**：只有 token 最新的 worker 能发 page/block 事件；
3. **所有外部动作走 outbox**：发 Telegram、建 Jira、阻塞 release 都先写事件，再由 sender 投递，避免半成功状态。

---

## 5. OpenClaw 场景：Cron 自动授课失败怎么处理？

拿 OpenClaw 的课程 Cron 做类比：

```text
cron run 失败
  ↓
DrillCloseoutReceipt: git push failed
  ↓
RootCausePacket: gh auth switched to wrong user
  ↓
OwnerHandoffReceipt: owner = agent-course-maintainer
  ↓
RepairSlaLease:
  ackDeadline = 15 min
  repairDeadline = 2 h
  acceptanceCriteria:
    - gh auth switch --user gfwfail succeeds
    - git push reaches origin/main
    - Telegram lesson message delivered
```

如果 15 分钟内没人 ack，Agent 可以 page owner；如果 2 小时内还没修复，暂停下一次课程发布，避免连续生成未推送的本地课程文件。

这就是 Always-on Agent 的真实问题：不是“失败会不会发生”，而是失败后责任是否能自动收敛。

---

## 6. 设计要点

### ACK 和 SLA 分开

ACK 表示“我接手并认可范围”；SLA 表示“我承诺在某个时间前交付修复证据”。二者不能混成一个 `assigned = true`。

### 修复完成不等于验证通过

Owner 提交 PR、刷新索引或修复配置后，只能进入 Repair Verification Gate。验证要重新跑 affected seed、adjacent seed、smoke replay 和生产探针。

### 高风险逾期要能阻塞发布

如果 root cause 影响 release blocking suite，SLA 逾期不能只是发提醒。要写 release blocker，让发布系统看到硬状态。

### 心跳不是聊天消息

RepairProgressHeartbeat 应该是结构化状态：`in_progress / blocked / claimed_fixed`、blocker、evidence refs、更新时间。聊天可以作为输入，但系统里必须落结构化记录。

---

## 7. 常见坑

### 坑 1：只创建工单，不等 owner ack

工单创建成功不代表 owner 看见了。必须有明确 ack deadline。

### 坑 2：SLA 没有暂停条件

如果卡在第三方、权限审批或外部 incident，SLA 可以暂停，但暂停本身也要有 receipt 和新的 deadline。

### 坑 3：逾期升级没有幂等键

没有 `packetId + leaseId + decision` 这种幂等键，多个 worker 可能重复 page、重复创建 blocker。

### 坑 4：修复证据只存在 PR 描述里

PR 是证据之一，但 RepairSlaCloseoutReceipt 里要记录 evidence refs，后续验证闸门才能自动取用。

---

## 8. 总结

Owner Handoff Acknowledgement & Repair SLA Gate 的核心不是“催人干活”，而是把人的修复责任也纳入 Agent 的状态机：

- handoff 之后必须 ack；
- ack 之后必须签 SLA lease；
- repair 期间必须有 heartbeat；
- 逾期必须自动升级或阻塞发布；
- claimed fixed 必须带证据进入验证闸门；
- closeout 必须写 receipt，供下一环消费。

成熟 Agent 不把“已派单”当成功。它会证明：正确的人已经接手、承诺了期限、持续推进、逾期有后果、修复证据能进入下一道验证。

下一讲可以继续：**Root-Cause Repair Verification & Regression Suite Unblock Gate**，也就是 owner 说“修好了”以后，Agent 如何用回归重放和生产探针决定能不能解除阻塞。
