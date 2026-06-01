# 459. Agent 取消后的冻结快照关闭闸门与保留清理（Cancelled Run Snapshot Closeout & Retention Cleanup Gate）

上一课讲了取消发生时要先写 EvidenceFreezeSnapshot：冻结 tool phase、effect refs、artifact refs、payloadHash 和 redaction profile，再去 cleanup、reaper、side-effect reconciliation。

今天继续补完整个闭环：**冻结快照不能永远躺在热路径里，也不能在对账没结束前被清理。**

很多系统做完 cancel 后会留下两种坏状态：

~~~text
1. 快照一直 active，后续 worker 每次都重复扫描，热存储越来越肥；
2. 快照太早删除，orphan reaper / side-effect reconciliation 还没证明终态。
~~~

所以取消恢复链路里需要一个专门的关闭闸门：

~~~text
EvidenceFreezeSnapshot
  -> cleanup / orphan reaper / side-effect reconciliation
  -> SnapshotCloseoutGate
  -> RetentionCleanupReceipt
~~~

一句话：**冻结快照是为了判断真相；真相判断完以后，要用关闭收据证明哪些证据该留、哪些该压缩、哪些该清理。**

---

## 1. 为什么冻结快照需要 closeout

EvidenceFreezeSnapshot 解决的是“取消那一刻有什么证据”。但它不是最终状态。

后面至少还有几条异步链路会继续跑：

- cleanup worker：清理临时文件、关闭 stream、释放 lease；
- orphan reaper：停止或接管子进程、sub-agent、外部 job；
- side-effect reconciler：对账 Telegram/GitHub/支付/部署等外部副作用；
- compensation worker：对不可逆副作用做 forward-fix；
- user-facing closeout：告诉用户这次取消最终停在什么状态。

如果没有 closeout gate，系统很容易犯两个错误。

第一种是**把 frozen 当 closed**：

~~~text
snapshot exists
=> 看起来有证据
=> 标记 cancellation closed
~~~

但 snapshot 只能证明“当时拍了照”，不能证明“后面所有对象都收口了”。

第二种是**把 cleanup 当 closed**：

~~~text
temp files deleted
process killed
=> 标记 cancellation closed
~~~

但 cleanup 只证明本地干净了，不能证明外部现实已经一致。

正确的关闭条件应该更硬：

~~~text
snapshotHash verified
all required cleanup receipts present
all descendants terminal or transferred
all effects matched/repaired/accepted residual risk
retention decision written
cleanup receipt written once
~~~

---

## 2. 最小数据模型

建议把关闭闸门拆成两个对象。

第一个是 SnapshotCloseoutDecision：

~~~json
{
  "snapshotId": "freeze_run_123_cancel_9",
  "decision": "close_and_compact",
  "requiredReceipts": [
    "cleanup:run_123",
    "reaper:run_123",
    "effect_reconcile:run_123"
  ],
  "residualRisks": [],
  "retainRefs": ["summary:freeze_run_123_cancel_9", "hash:freeze_run_123_cancel_9"],
  "purgeRefs": ["stdout_tail:tool_8", "temp_manifest:patch_17"],
  "reasonCodes": ["all_effects_matched", "all_descendants_terminal"]
}
~~~

第二个是 RetentionCleanupReceipt：

~~~json
{
  "receiptId": "snapshot-retention:freeze_run_123_cancel_9",
  "snapshotId": "freeze_run_123_cancel_9",
  "snapshotHash": "sha256:...",
  "closedAt": 1780321800000,
  "retentionMode": "compact_summary_keep_hash",
  "retainedRefs": ["summary:freeze_run_123_cancel_9"],
  "purgedRefs": ["stdout_tail:tool_8"],
  "tombstoneRef": "tombstone:stdout_tail:tool_8",
  "nextReviewAt": 1780926600000
}
~~~

注意这里不是“删掉就完事”。每个被清理的热证据都应该留下 tombstone 或 purge proof，后续审计能回答三个问题：

- 为什么当时可以删；
- 删的是哪份证据；
- 是否还有 summary/hash 可以证明历史存在过。

---

## 3. learn-claude-code：教学版关闭闸门

learn-claude-code 的教学实现里，任务系统把状态写到文件，BackgroundManager 用 notification queue 把后台结果重新注入 agent loop。这个思想可以直接套到取消快照关闭上：把每条异步收据看成外部状态文件，closeout gate 只做纯判定。

~~~python
# learn_claude_code/runtime/cancel_snapshot_closeout.py
from dataclasses import dataclass
from typing import Literal

ReceiptStatus = Literal["missing", "pending", "terminal_ok", "terminal_risk"]
Decision = Literal[
    "hold_for_receipts",
    "hold_for_reconcile",
    "close_and_compact",
    "close_with_residual_risk",
    "manual_review",
]


@dataclass(frozen=True)
class RequiredReceipt:
    kind: str
    status: ReceiptStatus
    evidence_ref: str | None = None


@dataclass(frozen=True)
class SnapshotCloseoutInput:
    snapshot_id: str
    snapshot_hash_valid: bool
    required_receipts: list[RequiredReceipt]
    residual_risk_count: int
    contains_raw_secret: bool
    legal_hold: bool


@dataclass(frozen=True)
class SnapshotCloseoutDecision:
    decision: Decision
    retain_mode: str
    reason_codes: list[str]


def decide_snapshot_closeout(case: SnapshotCloseoutInput) -> SnapshotCloseoutDecision:
    if not case.snapshot_hash_valid:
        return SnapshotCloseoutDecision(
            "manual_review",
            "retain_raw_until_review",
            ["snapshot_hash_mismatch"],
        )

    if case.contains_raw_secret:
        return SnapshotCloseoutDecision(
            "manual_review",
            "quarantine_then_reclassify",
            ["snapshot_contains_raw_secret"],
        )

    missing = [r.kind for r in case.required_receipts if r.status == "missing"]
    if missing:
        return SnapshotCloseoutDecision(
            "hold_for_receipts",
            "retain_hot",
            [f"missing_{kind}_receipt" for kind in missing],
        )

    pending = [r.kind for r in case.required_receipts if r.status == "pending"]
    if pending:
        return SnapshotCloseoutDecision(
            "hold_for_reconcile",
            "retain_hot",
            [f"pending_{kind}_receipt" for kind in pending],
        )

    risky = [r.kind for r in case.required_receipts if r.status == "terminal_risk"]
    if risky or case.residual_risk_count > 0:
        return SnapshotCloseoutDecision(
            "close_with_residual_risk",
            "compact_summary_keep_refs",
            [f"residual_{kind}" for kind in risky] or ["residual_risk_registered"],
        )

    if case.legal_hold:
        return SnapshotCloseoutDecision(
            "close_and_compact",
            "compact_summary_keep_under_legal_hold",
            ["legal_hold_blocks_purge"],
        )

    return SnapshotCloseoutDecision(
        "close_and_compact",
        "compact_summary_keep_hash_purge_raw",
        ["all_receipts_terminal_ok"],
    )
~~~

这个函数的重点不是代码复杂，而是边界清楚：

- hash 不对不能关闭；
- raw secret 不能正常进入保留清理；
- receipt missing/pending 都不能删热证据；
- terminal risk 可以关闭，但必须登记残余风险；
- legal hold 允许关闭执行链路，但不允许物理清理；
- 全部 terminal_ok 才能压缩 summary、保留 hash、清理 raw。

---

## 4. pi-mono：生产版 SnapshotCloseoutWorker

pi-mono 的 agent-loop 已经把 AgentEvent 流式发出来，coding-agent 也有 EventBus 这种轻量发布订阅接口。生产实现里，可以让 cancel 链路各 worker 产出 receipt event，再由 SnapshotCloseoutWorker 聚合判断。

~~~ts
type ReceiptStatus = "missing" | "pending" | "terminal_ok" | "terminal_risk";

type CancellationReceiptKind =
  | "cleanup"
  | "orphan_reaper"
  | "side_effect_reconcile"
  | "compensation"
  | "user_notice";

type SnapshotCloseoutReceipt = {
  receiptId: string;
  snapshotId: string;
  snapshotHash: string;
  decision:
    | "hold_for_receipts"
    | "hold_for_reconcile"
    | "close_and_compact"
    | "close_with_residual_risk"
    | "manual_review";
  retentionMode:
    | "retain_hot"
    | "compact_summary_keep_hash_purge_raw"
    | "compact_summary_keep_refs"
    | "retain_raw_until_review";
  reasonCodes: string[];
  retainedRefs: string[];
  purgedRefs: string[];
  closedAt?: number;
};

class SnapshotCloseoutWorker {
  constructor(
    private readonly snapshots: EvidenceSnapshotStore,
    private readonly receipts: CancellationReceiptStore,
    private readonly retention: EvidenceRetentionStore,
    private readonly outbox: Outbox,
    private readonly hasher: SnapshotHasher,
  ) {}

  async evaluate(snapshotId: string): Promise<SnapshotCloseoutReceipt> {
    const snapshot = await this.snapshots.get(snapshotId);
    const currentHash = this.hasher.hash(snapshot.withoutHash());

    if (currentHash !== snapshot.snapshotHash) {
      return this.writeReceipt(snapshot, {
        decision: "manual_review",
        retentionMode: "retain_raw_until_review",
        reasonCodes: ["snapshot_hash_mismatch"],
      });
    }

    const required: CancellationReceiptKind[] = [
      "cleanup",
      "orphan_reaper",
      "side_effect_reconcile",
      "user_notice",
    ];

    const receiptStatuses = await Promise.all(
      required.map(async (kind) => ({
        kind,
        status: await this.receipts.statusFor(snapshot.runId, kind),
      })),
    );

    const missingOrPending = receiptStatuses.filter(
      (r) => r.status === "missing" || r.status === "pending",
    );

    if (missingOrPending.length > 0) {
      return this.writeReceipt(snapshot, {
        decision: missingOrPending.some((r) => r.status === "missing")
          ? "hold_for_receipts"
          : "hold_for_reconcile",
        retentionMode: "retain_hot",
        reasonCodes: missingOrPending.map((r) => `${r.status}_${r.kind}`),
      });
    }

    const risky = receiptStatuses.filter((r) => r.status === "terminal_risk");

    const retainedRefs = await this.retention.compactSnapshot(snapshot, {
      keepSummary: true,
      keepHash: true,
      keepEffectRefs: risky.length > 0,
      purgeRawArtifactTails: risky.length === 0 && !snapshot.legalHold,
    });

    const receipt = await this.writeReceipt(snapshot, {
      decision: risky.length > 0 ? "close_with_residual_risk" : "close_and_compact",
      retentionMode:
        risky.length > 0
          ? "compact_summary_keep_refs"
          : "compact_summary_keep_hash_purge_raw",
      reasonCodes: risky.length > 0
        ? risky.map((r) => `residual_${r.kind}`)
        : ["all_receipts_terminal_ok"],
      retainedRefs,
      purgedRefs: retainedRefs.purgedRefs,
      closedAt: Date.now(),
    });

    await this.outbox.enqueue("cancel.snapshot.closeout", receipt);
    return receipt;
  }

  private async writeReceipt(
    snapshot: EvidenceFreezeSnapshot,
    partial: Partial<SnapshotCloseoutReceipt>,
  ): Promise<SnapshotCloseoutReceipt> {
    return this.receipts.writeOnceSnapshotCloseout({
      receiptId: "snapshot-closeout:" + snapshot.snapshotId,
      snapshotId: snapshot.snapshotId,
      snapshotHash: snapshot.snapshotHash,
      decision: partial.decision ?? "hold_for_reconcile",
      retentionMode: partial.retentionMode ?? "retain_hot",
      reasonCodes: partial.reasonCodes ?? [],
      retainedRefs: partial.retainedRefs ?? snapshot.artifactRefs,
      purgedRefs: partial.purgedRefs ?? [],
      closedAt: partial.closedAt,
    });
  }
}
~~~

这里有几个工程点：

- **writeOnceSnapshotCloseout**：worker 重跑不会制造多份互相冲突的关闭收据；
- **statusFor(runId, kind)**：不要扫日志猜状态，要读结构化 receipt；
- **compactSnapshot**：清理动作本身也要返回 retained/purged refs；
- **outbox.enqueue**：关闭结果是事件，后续可以触发用户通知、指标、归档；
- **manual_review**：hash mismatch 和 raw secret 不走自动清理。

---

## 5. OpenClaw：课程 Cron 里的真实映射

拿这个 Agent 开发课程 cron 举例，一次任务会产生一串外部副作用：

~~~text
lesson file written
README updated
TOOLS updated
Telegram message sent
git commit created
git push completed
memory updated
~~~

如果这次 run 在中间被取消，EvidenceFreezeSnapshot 会记录“取消那一刻”：

~~~yaml
snapshot:
  runId: agent-course-2026-06-01-2130
  artifactRefs:
    - lessons/459-cancelled-run-snapshot-closeout-retention-cleanup.md
    - README.md
    - TOOLS.md
  effectRefs:
    - telegram:messageId?
    - git:commit?
    - git:push?
  phaseVector:
    - write_lesson: acknowledged
    - send_telegram: dispatched
    - git_push: prepared
~~~

但最终能不能清理这份 snapshot，要等 closeout gate 看完：

~~~yaml
requiredReceipts:
  cleanup: terminal_ok
  orphan_reaper: terminal_ok
  side_effect_reconcile: terminal_ok
  user_notice: terminal_ok

decision: close_and_compact
retention:
  keep:
    - lesson path
    - README entry
    - TOOLS topic
    - Telegram messageId
    - git commit hash
  purge:
    - command stdout tail
    - temporary diff buffer
    - draft prompt scratch
~~~

这就是自动化 cron 稳定运行的关键：不是“成功了就忘”，而是把成功、取消、恢复、清理都变成可审计状态。

---

## 6. 常见坑

**坑 1：只按时间 TTL 清理 snapshot。**

TTL 只能说明“时间到了”，不能说明“对账结束”。snapshot 的清理条件必须包含 required receipts。

**坑 2：side effect unknown 时仍然 purge raw refs。**

如果外部副作用还是 unknown_needs_reconcile，raw/tail evidence 可能是唯一能查 remoteRef 的证据。不要提前删。

**坑 3：legal hold 和 closeout 混在一起。**

legal hold 可以阻止 purge，但不应该阻止执行链路 closeout。正确状态是 close_and_compact_under_hold 或 close_with_hold。

**坑 4：closeout receipt 可覆盖。**

关闭收据要 write-once 或 append-only。后续发现错误应该写 correction receipt，不要改旧 receipt。

**坑 5：清理动作没有 tombstone。**

没有 tombstone 的删除在审计里等于“证据突然消失”。哪怕只保留 hash，也要能证明删的是哪一份。

---

## 7. 落地检查清单

给取消恢复系统加 snapshot closeout 时，至少检查：

- EvidenceFreezeSnapshot 是否有 snapshotHash？
- cleanup / orphan reaper / side-effect reconciliation 是否都有结构化 receipt？
- closeout gate 是否把 missing 和 pending 分开？
- unknown side effect 是否会阻止 raw evidence purge？
- legal hold 是否只阻止 purge，不阻止关闭执行链？
- retention cleanup 是否返回 retainedRefs 和 purgedRefs？
- purgedRefs 是否写 tombstone？
- closeout receipt 是否 write-once？
- 后续发现错误时是否写 correction receipt，而不是改旧收据？

---

## 8. 总结

取消链路不能停在“我已经拍照了”。冻结快照只是第一步，后续必须证明 cleanup、reaper、side-effect reconciliation 都进入可信终态，然后再决定证据保留策略。

成熟 Agent 的取消恢复闭环应该是：

~~~text
freeze evidence
-> reconcile reality
-> close with receipts
-> compact evidence
-> purge raw with tombstones
~~~

这样系统既不会因为太早清理丢证据，也不会因为永远保留 raw 快照把热路径拖垮。
