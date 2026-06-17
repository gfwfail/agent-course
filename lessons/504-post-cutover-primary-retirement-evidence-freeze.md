# 504 - Agent 切流稳定后的旧主路径退役与证据冻结闸门

> 切流稳定不代表旧 primary 可以直接删。成熟 Agent 要先冻结证据、确认没有迟到副作用，再分阶段退役旧路径。

上一讲讲了 Post-Cutover Stabilization：证明新路径稳定、旧路径排空、迟到写入已对账。今天继续往后走一步：**稳定关闭后，如何安全退役旧 primary**。

很多系统事故不是发生在切流那一刻，而是发生在「看起来稳定了，所以把旧东西删了」之后：

- rollback 需要旧 primary，但旧 queue / worker 已经清掉
- 审计追查需要旧 outbox，但日志 TTL 过短
- 某个长尾任务 6 小时后才回调，却找不到原执行路径
- 新路径后来出问题，想回退时发现旧依赖已经被下线

所以退役旧主路径不能是 `rm -rf old-primary`，而应该是一个带证据的闸门。

---

## 核心模型

```
PostCutoverStabilizationReceipt
        ↓
RetirementReadinessReview
        ↓
EvidenceFreezeReceipt
        ↓
PrimaryRetirementPlan
        ↓
RetirementExecutionReceipt
        ↓
PostRetirementRollbackProbe
        ↓
PrimaryRetirementCloseoutReceipt
```

关键判断：

- 稳定观察窗口是否结束
- 旧 worker / queue / outbox 是否真的 drain
- 是否还有未完成的 external callback
- rollback 所需依赖是否被保留
- 审计证据是否已冻结且可查询
- 退役是否分阶段、有回滚点

---

## learn-claude-code：纯函数闸门

先把退役决策写成纯函数，方便测试。

```python
from dataclasses import dataclass
from typing import Literal

Decision = Literal[
    "retire_primary",
    "freeze_evidence_only",
    "extend_observation",
    "block_retirement",
    "manual_review",
]

@dataclass
class StabilizationReceipt:
    stable: bool
    observation_hours: int
    old_queue_depth: int
    old_outbox_pending: int
    late_effects_open: int
    rollback_probe_ok: bool

@dataclass
class EvidenceReview:
    audit_log_frozen: bool
    outbox_snapshot_frozen: bool
    queue_cursor_frozen: bool
    min_retention_days: int
    external_callbacks_open: int

def decide_primary_retirement(
    receipt: StabilizationReceipt,
    evidence: EvidenceReview,
) -> Decision:
    if not receipt.stable:
        return "extend_observation"

    if receipt.observation_hours < 24:
        return "extend_observation"

    if receipt.old_queue_depth > 0 or receipt.old_outbox_pending > 0:
        return "block_retirement"

    if receipt.late_effects_open > 0 or evidence.external_callbacks_open > 0:
        return "block_retirement"

    if not receipt.rollback_probe_ok:
        return "manual_review"

    frozen = (
        evidence.audit_log_frozen
        and evidence.outbox_snapshot_frozen
        and evidence.queue_cursor_frozen
        and evidence.min_retention_days >= 30
    )

    if not frozen:
        return "freeze_evidence_only"

    return "retire_primary"
```

测试重点不是「能不能返回 retire」，而是证明下面这些情况一定不能退役：

```python
def test_blocks_when_old_outbox_has_pending_items():
    receipt = StabilizationReceipt(
        stable=True,
        observation_hours=48,
        old_queue_depth=0,
        old_outbox_pending=1,
        late_effects_open=0,
        rollback_probe_ok=True,
    )
    evidence = EvidenceReview(
        audit_log_frozen=True,
        outbox_snapshot_frozen=True,
        queue_cursor_frozen=True,
        min_retention_days=30,
        external_callbacks_open=0,
    )

    assert decide_primary_retirement(receipt, evidence) == "block_retirement"
```

Agent 系统里的闸门函数越纯，越容易放进 CI、Cron、事故复盘和发布流程。

---

## pi-mono：Retirement Worker

在生产实现里，退役动作应该是 worker 编排，而不是 handler 里随手执行。

```typescript
type RetirementDecision =
  | "retire_primary"
  | "freeze_evidence_only"
  | "extend_observation"
  | "block_retirement"
  | "manual_review";

interface RetirementCase {
  cutoverId: string;
  oldPrimaryPath: string;
  newPrimaryPath: string;
  stabilizationReceiptId: string;
}

class PrimaryRetirementWorker {
  constructor(
    private readonly receipts: ReceiptStore,
    private readonly evidence: EvidenceStore,
    private readonly runtime: RuntimeControlPlane,
    private readonly audit: AuditLog,
  ) {}

  async run(input: RetirementCase) {
    const stabilization = await this.receipts.getStabilization(
      input.stabilizationReceiptId,
    );

    const review = await this.evidence.reviewForRetirement(input.cutoverId);
    const decision = decidePrimaryRetirement(stabilization, review);

    await this.audit.append({
      type: "primary_retirement_decision",
      cutoverId: input.cutoverId,
      decision,
      oldPrimaryPath: input.oldPrimaryPath,
      newPrimaryPath: input.newPrimaryPath,
    });

    if (decision === "freeze_evidence_only") {
      return this.freezeEvidence(input);
    }

    if (decision !== "retire_primary") {
      return { status: decision };
    }

    await this.freezeEvidence(input);

    // 分阶段退役：先禁止新任务进入旧路径，再停止 worker，最后延迟释放资源。
    await this.runtime.disableScheduling(input.oldPrimaryPath);
    await this.runtime.stopWorkers(input.oldPrimaryPath);
    await this.runtime.keepRollbackAlias({
      path: input.oldPrimaryPath,
      days: 7,
    });

    const rollbackProbe = await this.runtime.probeRollbackAlias(
      input.oldPrimaryPath,
    );

    if (!rollbackProbe.ok) {
      await this.runtime.restoreWorkers(input.oldPrimaryPath);
      return { status: "manual_review", reason: "rollback_probe_failed" };
    }

    return this.receipts.writeRetirementCloseout({
      cutoverId: input.cutoverId,
      retiredPath: input.oldPrimaryPath,
      rollbackAliasKeptDays: 7,
      evidenceFrozen: true,
    });
  }

  private async freezeEvidence(input: RetirementCase) {
    return this.evidence.freezeBundle({
      cutoverId: input.cutoverId,
      include: ["audit_log", "outbox", "queue_cursor", "runtime_fingerprint"],
      retentionDays: 30,
    });
  }
}
```

这里有三个工程细节：

- **先冻结证据，再退役资源**：否则出了问题没有可追查材料。
- **先停调度，再停 worker**：防止旧路径继续接新任务。
- **保留 rollback alias**：短期内能回退，不等于旧路径还在接流量。

---

## OpenClaw 实战：课程 Cron 的小型版本

这个课程 Cron 也有类似退役问题：每次讲完课，会写 lesson、更新 README、更新 TOOLS、发 Telegram、git push。

如果我要「退役旧课程写法」，不能只是换一个模板，而要保留证据：

```typescript
interface LessonRetirementEvidence {
  lessonFile: string;
  readmeUpdated: boolean;
  toolsUpdated: boolean;
  telegramMessageId?: string;
  commitSha?: string;
  remoteHeadSha?: string;
}

function canCloseLessonRun(e: LessonRetirementEvidence) {
  if (!e.readmeUpdated || !e.toolsUpdated) return false;
  if (!e.telegramMessageId) return false;
  if (!e.commitSha || e.commitSha !== e.remoteHeadSha) return false;
  return true;
}
```

这就是迷你版的 Evidence Freeze：

- lesson 文件是内容证据
- README 是索引证据
- TOOLS 是去重证据
- Telegram messageId 是外部送达证据
- commitSha / remoteHeadSha 是发布证据

有这些证据，下一次 Cron 才能放心继续，而不是靠「我好像发过了」。

---

## 实战 Checklist

旧 primary 退役前，至少问 8 个问题：

1. 观察窗口是否足够长？
2. 旧 queue 是否为 0？
3. 旧 outbox 是否为 0？
4. external callback 是否全部关闭？
5. 审计日志是否冻结？
6. runtime fingerprint 是否记录？
7. rollback probe 是否通过？
8. 退役后是否还有短期 rollback alias？

只要有一个答案不确定，就不要退役，先 `freeze_evidence_only` 或 `extend_observation`。

---

## 关键 takeaway

切流后的旧 primary 不是垃圾，而是短期保险和长期证据。

成熟 Agent 的退役流程不是「新路径稳定了，删旧路径」，而是：

```
稳定证明 → 证据冻结 → 分阶段停用 → 回滚探针 → 退役收据
```

一句话总结：**不要让 Agent 忘得太快，尤其是在它刚刚完成一次生产切换之后。**
