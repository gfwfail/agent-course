# 532. Agent 演练失败根因包与 Owner 交接闸门

> DrillCloseoutReceipt 只能证明演练结果被消费了，不证明失败已经被交给正确的人、带着足够证据进入修复闭环。成熟 Agent 会把失败演练压缩成 Root Cause Packet，绑定 owner、证据、复现命令、影响范围和验收条件，再用交接收据证明修复责任没有丢在聊天记录里。

上一讲讲了 **Drill Execution Result, Escalation & Closeout Gate**：演练跑完以后，要把 pass/fail/flaky/skipped 变成可升级、可关闭、可学习的结果。

今天继续往后走：**演练失败已经升级了，怎么保证接手修复的人不用重新考古？**

常见问题是：

- drill 报警只写了 "missing lookup key"，owner 还要翻日志、找 seed、问上下文；
- 失败被派给了错误团队，冷路径、索引、权限、归档各自甩锅；
- 修复任务没有复现命令，回归时只能凭感觉验证；
- drill failure 被关闭了，但 root cause 没有固化成后续 guard、seed 或 runbook；
- Telegram/Slack 里说了很多，代码仓库和任务系统里没有可审计交接。

所以要加一层 **Drill Failure Root Cause Packet & Owner Handoff Gate**：失败不是转发一条报警，而是生成一个足够小、足够完整、可验收的交接包。

---

## 1. 核心链路

```text
DrillCloseoutReceipt
        ↓
FailureEvidenceBundle
        ↓
RootCausePacket
        ↓
OwnerHandoffDecision
        ↓
OwnerHandoffReceipt
```

每一环回答一个问题：

- closeout receipt：演练失败是否已经被确认需要处理；
- evidence bundle：失败证据是否齐全，能不能复现；
- root cause packet：根因假设、影响范围、修复入口和验收条件是否明确；
- handoff decision：交给哪个 owner，是否需要拆分、补证据或升级；
- handoff receipt：owner 是否确认接手，后续 task/incident/PR 是否绑定。

一句话：**报警通知人，根因包交接责任。没有 OwnerHandoffReceipt 的失败演练，只是把问题从系统里推到了人脑里。**

---

## 2. 根因包应该包含什么

一个 Root Cause Packet 不需要写成长篇复盘，但必须包含 7 个字段：

```text
rootCausePacket
  packetId
  drillId / runId / seedIds
  failureClass
  suspectedOwner
  evidenceRefs
  reproductionCommand
  acceptanceCriteria
```

字段解释：

- `failureClass`：是 retrieval regression、index stale、lease policy deny、archive missing、harness bug，还是 unknown；
- `suspectedOwner`：先按证据推断 owner，不要让 oncall 猜；
- `evidenceRefs`：日志片段、bundle hash、lookup key、runtime fingerprint、commit、messageId；
- `reproductionCommand`：一条能本地或 staging 复现的命令；
- `acceptanceCriteria`：修复关闭条件，比如 seed 全部 pass、P95 回到阈值、旧 bundle 可取回。

这里的关键是：**根因包不是最终真相，而是可行动的第一版假设。** 它允许后续 owner 修正 root cause，但不能允许空手接锅。

---

## 3. learn-claude-code：根因包交接判定器

教学版先写纯函数。它不创建工单，只判断这个失败能不能交接、交给谁、还缺什么。

```python
from dataclasses import dataclass
from typing import Literal

FailureClass = Literal[
    "retrieval_regression",
    "index_stale",
    "lease_policy_denied",
    "archive_bundle_missing",
    "drill_harness_bug",
    "unknown",
]

Decision = Literal[
    "handoff_to_owner",
    "split_multi_owner_handoff",
    "request_more_evidence",
    "escalate_incident_commander",
    "route_to_harness_repair",
    "manual_triage",
]

@dataclass
class DrillCloseoutReceipt:
    drill_id: str
    run_id: str
    decision: str
    severity: Literal["low", "medium", "high", "critical"]
    acknowledged: bool

@dataclass
class FailureEvidenceBundle:
    seed_ids: list[str]
    runtime_fingerprint: str | None
    bundle_hash: str | None
    lookup_keys: list[str]
    log_excerpt_ref: str | None
    reproduction_command: str | None
    affected_tenants: list[str]
    harness_error: bool

@dataclass
class RootCausePacket:
    packet_id: str
    failure_class: FailureClass
    suspected_owners: list[str]
    evidence_refs: list[str]
    acceptance_criteria: list[str]
    confidence: float

def decide_owner_handoff_gate(
    closeout: DrillCloseoutReceipt,
    evidence: FailureEvidenceBundle,
    packet: RootCausePacket,
) -> Decision:
    if not closeout.acknowledged:
        return "manual_triage"

    if closeout.decision not in (
        "escalate_retrieval_regression",
        "repair_drill_harness",
        "open_seed_refresh_task",
    ):
        return "manual_triage"

    if evidence.harness_error or packet.failure_class == "drill_harness_bug":
        if evidence.log_excerpt_ref and evidence.reproduction_command:
            return "route_to_harness_repair"
        return "request_more_evidence"

    missing_repro = not evidence.reproduction_command
    missing_runtime = not evidence.runtime_fingerprint
    missing_evidence = not packet.evidence_refs or not evidence.log_excerpt_ref
    missing_acceptance = not packet.acceptance_criteria

    if missing_repro or missing_runtime or missing_evidence or missing_acceptance:
        return "request_more_evidence"

    if closeout.severity == "critical" and len(evidence.affected_tenants) > 1:
        return "escalate_incident_commander"

    if len(packet.suspected_owners) > 1:
        return "split_multi_owner_handoff"

    if len(packet.suspected_owners) == 1 and packet.confidence >= 0.65:
        return "handoff_to_owner"

    return "manual_triage"
```

对应测试：

```python
def test_requests_more_evidence_when_reproduction_command_missing():
    closeout = DrillCloseoutReceipt(
        drill_id="cold-retrieval-daily",
        run_id="run-532",
        decision="escalate_retrieval_regression",
        severity="high",
        acknowledged=True,
    )
    evidence = FailureEvidenceBundle(
        seed_ids=["seed-cold-17"],
        runtime_fingerprint="cold-v27",
        bundle_hash="sha256:abc",
        lookup_keys=["archive:tenant-a:2026-06"],
        log_excerpt_ref="logs/run-532#L120-L160",
        reproduction_command=None,
        affected_tenants=["tenant-a"],
        harness_error=False,
    )
    packet = RootCausePacket(
        packet_id="rcp-532",
        failure_class="index_stale",
        suspected_owners=["search-index"],
        evidence_refs=["logs/run-532#L120-L160", "bundle:sha256:abc"],
        acceptance_criteria=["seed-cold-17 passes", "P95 < 1200ms"],
        confidence=0.82,
    )

    assert decide_owner_handoff_gate(closeout, evidence, packet) == "request_more_evidence"
```

注意这个判定器的态度很硬：没有复现命令、运行时指纹、证据引用或验收条件，就不要交给 owner。否则只是把排查成本甩给下游。

---

## 4. pi-mono：RootCauseHandoffWorker

生产版可以把它做成 drill closeout 后的 worker。它消费 closeout receipt，聚合证据，生成根因包，创建任务，并写 OwnerHandoffReceipt。

```typescript
type HandoffDecision =
  | "handoff_to_owner"
  | "split_multi_owner_handoff"
  | "request_more_evidence"
  | "escalate_incident_commander"
  | "route_to_harness_repair"
  | "manual_triage";

interface DrillClosedEvent {
  drillId: string;
  runId: string;
  closeoutReceiptId: string;
}

interface OwnerHandoffReceipt {
  receiptId: string;
  packetId: string;
  decision: HandoffDecision;
  ownerTasks: string[];
  evidenceHash: string;
  createdAt: string;
}

class RootCauseHandoffWorker {
  constructor(
    private readonly receipts: ReceiptStore,
    private readonly evidence: DrillEvidenceStore,
    private readonly rootCause: RootCausePacketBuilder,
    private readonly owners: OwnerDirectory,
    private readonly tasks: TaskQueue,
    private readonly incidents: IncidentQueue,
    private readonly audit: AuditLog,
  ) {}

  async run(event: DrillClosedEvent): Promise<OwnerHandoffReceipt> {
    const closeout = await this.receipts.getDrillCloseout(
      event.closeoutReceiptId,
    );

    const bundle = await this.evidence.collectFailureBundle({
      drillId: event.drillId,
      runId: event.runId,
      seedIds: closeout.seedIds,
    });

    const packet = await this.rootCause.build({
      closeout,
      evidence: bundle,
      ownerHints: await this.owners.findByFailureClass(closeout.failureClass),
    });

    const decision = decideOwnerHandoffGate(closeout, bundle, packet);
    const ownerTasks: string[] = [];

    if (decision === "handoff_to_owner") {
      ownerTasks.push(
        await this.tasks.create({
          owner: packet.suspectedOwners[0],
          title: `[drill] ${packet.failureClass} in ${event.drillId}`,
          body: renderRootCausePacket(packet),
          acceptanceCriteria: packet.acceptanceCriteria,
          evidenceRefs: packet.evidenceRefs,
        }),
      );
    }

    if (decision === "split_multi_owner_handoff") {
      for (const owner of packet.suspectedOwners) {
        ownerTasks.push(
          await this.tasks.create({
            owner,
            title: `[drill] scoped handoff for ${packet.packetId}`,
            body: renderOwnerScopedPacket(packet, owner),
            acceptanceCriteria: packet.acceptanceCriteria,
            evidenceRefs: packet.evidenceRefs,
          }),
        );
      }
    }

    if (decision === "escalate_incident_commander") {
      ownerTasks.push(
        await this.incidents.open({
          severity: closeout.severity,
          summary: `Critical drill regression: ${event.drillId}`,
          packet,
        }),
      );
    }

    const receipt = await this.receipts.writeOwnerHandoff({
      packetId: packet.packetId,
      decision,
      ownerTasks,
      evidenceHash: bundle.hash,
    });

    await this.audit.append("drill.owner_handoff", {
      drillId: event.drillId,
      runId: event.runId,
      receiptId: receipt.receiptId,
      decision,
      ownerTasks,
    });

    return receipt;
  }
}
```

这里可以直接借 pi-mono 里的两个生产模式：

- `EventBus`：drill closeout 之后发布 `DrillClosedEvent`，handoff worker 异步消费；
- `TaskQueue` / 队列串行化：每个 owner 的修复任务独立创建，但都绑定同一个 packetId 和 evidenceHash。

这比在群里 @ 某个人强很多，因为 owner 接到的是一个带证据和验收条件的工程对象。

---

## 5. OpenClaw：课程 Cron 的现实类比

拿 OpenClaw 课程 Cron 举例：

```text
cron run
  ↓
生成 lesson/README/TOOLS
  ↓
git commit/push
  ↓
Telegram 发布
  ↓
写 closeout
```

如果其中一步失败，比如：

- lesson 写好了但 push 失败；
- Telegram 发了但 README 没更新；
- README 更新了但 TOOLS 已讲列表没收敛；
- commit 成功但远端 rejected；

不能只说“cron 失败了”。应该生成一个小型 Root Cause Packet：

```json
{
  "packetId": "course-cron-2026-06-27-0030",
  "failureClass": "git_push_rejected",
  "suspectedOwner": "course-automation",
  "evidenceRefs": [
    "commit:abc123",
    "remote:origin/main",
    "lesson:lessons/532-drill-failure-root-cause-owner-handoff.md"
  ],
  "reproductionCommand": "gh auth switch --user gfwfail && git push",
  "acceptanceCriteria": [
    "origin/main contains commit abc123",
    "README links lesson 532",
    "TOOLS.md contains lesson 532 topic",
    "Telegram group -5115329245 received the message"
  ]
}
```

这样下一轮 Agent 接手时，不需要翻整段对话，只要读 packet 就知道：哪里失败、证据在哪、怎么复现、怎么算修好。

---

## 6. 设计细节：不要把根因包写成复盘文

Root Cause Packet 的目标不是总结得漂亮，而是让修复更快、更可靠。建议遵守四条规则：

### 规则一：证据引用必须可取回

不要写：

```text
日志里有报错
```

要写：

```text
logs/run-532#L120-L160
bundle:sha256:abc
seed:seed-cold-17
runtime:cold-v27
```

证据不可取回，owner 就会重新搜一遍。

### 规则二：复现命令必须能被机器执行

不要写：

```text
重新跑一下冷路径演练
```

要写：

```bash
openclaw drill run cold-retrieval-daily --seed seed-cold-17 --runtime cold-v27
```

哪怕命令只是伪代码，也要让系统设计时朝这个方向靠。

### 规则三：验收条件必须可判定

不要写：

```text
确认问题修好
```

要写：

```text
seed-cold-17 passes
missing_lookup_keys == 0
p95_ms < 1200
lease_denials == 0
```

验收条件模糊，closeout 就会变成主观判断。

### 规则四：owner 可以修正，但不能空接

RootCause Packet 可以是低置信度假设，但它必须带：

- suspected owner；
- confidence；
- why this owner；
- fallback owner 或 incident commander。

如果 confidence 太低，就进入 `manual_triage` 或 `request_more_evidence`，不要随机派单。

---

## 7. 常见坑

### 坑一：把报警文本当根因包

报警适合短、快、醒目；根因包适合完整、可追踪、可复现。两者可以互相链接，但不要合并成一条长消息。

### 坑二：多 owner 问题只派给一个人

比如冷路径失败可能同时涉及 archive bundle、search index、lease policy。证据显示多个 owner 时，应生成 `split_multi_owner_handoff`，每个任务只负责自己的边界，同时由同一个 packetId 关联。

### 坑三：只绑定工单，不绑定 evidence hash

没有 evidence hash，后续证据被覆盖或刷新后，很难证明 owner 当时接手的是什么。OwnerHandoffReceipt 里必须记录 evidenceHash。

### 坑四：关闭修复任务时不回写 drill seed

如果修复过程中发现了新的失败样本，要回写 Regression Seed Inventory。否则这次事故没有变成未来自动检测能力。

---

## 8. 实战检查清单

给你的 Agent 加 Root Cause Handoff Gate，可以按这个顺序落地：

1. 先定义 `FailureEvidenceBundle`：日志、seed、runtime、bundle、lookup key、复现命令；
2. 再定义 `RootCausePacket`：failureClass、owner、证据、验收条件、置信度；
3. 写纯函数 `decideOwnerHandoffGate`，强制缺证据不能交接；
4. 在生产系统里做 `RootCauseHandoffWorker`，消费 drill closeout event；
5. 任务系统里绑定 `packetId`、`evidenceHash`、`acceptanceCriteria`；
6. 修复关闭时要求回填 `OwnerHandoffReceipt` 和 seed 更新结果。

最小可用版本只需要三件事：

```text
failure_class + reproduction_command + acceptance_criteria
```

只要这三件事齐了，owner 接手效率就会明显提升。

---

## 9. 一句话总结

**Drill failure 的目标不是把错误喊出去，而是把可修复的责任交出去。Root Cause Packet 把“有人看到报警”升级成“正确 owner 拿到证据、复现方式和验收标准”。**

下一讲可以继续讲：Owner 接手以后，如何做 **Repair Attempt Tracking & Verification Gate**，证明修复尝试不是只改了一次配置，而是被 drill seed、现实探针和 closeout receipt 验证通过。
