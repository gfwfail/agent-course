# 526 - Agent 稳定投影保留到期与冷证据交接闸门

> 上一讲我们讲了 Stable Projection Lineage Compaction & Retention Gate：投影稳定关闭后，把重建血缘、旧版本引用、回滚别名和审计证据压缩成 RetentionCloseoutReceipt。今天继续往后走：RetentionCloseoutReceipt 写完以后，热路径证据不能永久保留，也不能到期就直接删除。

今天讲：**Stable Projection Retention Expiry & Cold Evidence Handoff Gate**。

一句话：**稳定投影的保留期到期时，Agent 要把热证据、回滚别名、审计索引和冷归档交接串成一条可验证链路；能降级就降级，能冷存就冷存，不能证明冷路径可用就不能清热证据。**

---

## 1. 为什么 RetentionCloseoutReceipt 不是终点？

上一课的 `RetentionCloseoutReceipt` 证明了：

- stable projection 已经观察稳定；
- 旧引用已经清理；
- raw payload 已经按策略删除或压缩；
- hash chain、manifest、consumer ack summary 还在；
- rollback alias 有 TTL，不再无限挂着。

但它通常只回答“现在能不能进入保留期”。它还没有回答：

- 保留期到期后谁来复核？
- 热证据能不能从在线索引移到冷索引？
- 冷归档是否能查、能验、能解释？
- 回滚别名是否仍被迟到任务引用？
- 法务保留、审计抽样、消费者争议是否会阻止删除？

很多系统的坏习惯是：

```text
retention_until < now
delete hot manifest
delete rollback alias
```

这会产生两个风险：

1. **运行面风险**：迟到 callback、旧 dashboard bookmark、异步导出任务还可能引用旧 projection。
2. **审计面风险**：热证据删了，但冷证据没有可验证索引，未来解释不了“当时为什么这么更正和放行”。

正确做法是一个交接闸门：

```text
RetentionCloseoutReceipt
        ↓
RetentionExpiryReview
        ↓
LateDependencySweep
        ↓
ColdEvidenceHandoffPlan
        ↓
ColdArchiveProbe
        ↓
HotEvidencePurgeReceipt
        ↓
ColdHandoffCloseoutReceipt
```

---

## 2. 闸门要回答什么？

Stable Projection Retention Expiry Gate 要回答七个问题：

- **是否真的到期**：`retainUntil`、`deleteAfter`、`reviewAfter` 不要混用；
- **是否有 legal hold**：有法务保留时只能缩小视图，不能销毁证据；
- **是否还有迟到依赖**：callback、export job、dashboard bookmark、consumer dispute；
- **冷归档是否完整**：manifest、hash chain、schema pin、redaction profile、consumer ack summary；
- **冷路径是否能查**：冷索引能按 projectionId / releaseId / receiptId 找回；
- **冷路径是否能解释**：能重建一份不含 raw 的 explanation packet；
- **热路径能否清理**：清理后反向探针确认热索引、cache、search、workspace 都不再含旧证据。

工程上记成一句：

```text
expiry review -> sweep late deps -> prove cold archive -> purge hot -> probe absence -> write closeout
```

注意：这不是普通 TTL 清理任务。它是从“热证据可快速访问”到“冷证据可审计取回”的责任交接。

---

## 3. learn-claude-code：到期交接判定器

教学版用纯函数表达。输入保留关闭收据、到期复核、迟到依赖扫描、冷归档探针和热路径探针，输出下一步动作。

```python
from dataclasses import dataclass
from typing import Literal

Decision = Literal[
    "handoff_to_cold_and_purge_hot",
    "extend_hot_retention",
    "narrow_for_legal_hold",
    "block_until_late_dependencies_clear",
    "repair_cold_archive",
    "manual_review",
]

@dataclass(frozen=True)
class RetentionCloseoutReceipt:
    projection_id: str
    projection_version: str
    receipt_id: str
    retain_until_epoch: int
    delete_after_epoch: int
    manifest_hash: str
    hash_chain_root: str
    raw_payload_purged: bool

@dataclass(frozen=True)
class RetentionExpiryReview:
    now_epoch: int
    legal_hold: bool
    unresolved_disputes: int
    owner_acknowledged_expiry: bool
    max_hot_extensions: int
    used_hot_extensions: int

@dataclass(frozen=True)
class LateDependencySweep:
    active_export_jobs: int
    callback_refs: int
    dashboard_bookmarks: int
    consumer_reopen_requests: int
    unknown_refs: int

@dataclass(frozen=True)
class ColdArchiveProbe:
    archive_written: bool
    manifest_hash_matches: bool
    hash_chain_root_matches: bool
    schema_pin_present: bool
    redaction_profile_present: bool
    explanation_packet_rebuilds: bool
    indexed_by_projection_id: bool

@dataclass(frozen=True)
class HotPathProbe:
    hot_manifest_refs: int
    cache_refs: int
    search_index_refs: int
    workspace_refs: int

@dataclass(frozen=True)
class ExpiryDecision:
    decision: Decision
    reason: str
    actions: tuple[str, ...] = ()

def decide_projection_retention_expiry(
    receipt: RetentionCloseoutReceipt,
    review: RetentionExpiryReview,
    late: LateDependencySweep,
    cold: ColdArchiveProbe,
    hot: HotPathProbe,
) -> ExpiryDecision:
    if not receipt.raw_payload_purged:
        return ExpiryDecision(
            "manual_review",
            "retention closeout still has raw payload; expiry path must not silently delete it",
            ("open_raw_payload_violation_case",),
        )

    if review.now_epoch < receipt.retain_until_epoch:
        return ExpiryDecision(
            "extend_hot_retention",
            "minimum retention window has not expired",
            ("schedule_next_expiry_review",),
        )

    if review.legal_hold:
        return ExpiryDecision(
            "narrow_for_legal_hold",
            "legal hold blocks purge; narrow hot view to minimal searchable proof",
            ("remove_non_hold_fields", "write_legal_hold_minimal_view"),
        )

    if review.unresolved_disputes > 0:
        return ExpiryDecision(
            "extend_hot_retention",
            "consumer or audit disputes still need fast evidence access",
            ("notify_owner", "schedule_dispute_review"),
        )

    late_refs = (
        late.active_export_jobs
        + late.callback_refs
        + late.dashboard_bookmarks
        + late.consumer_reopen_requests
    )
    if late_refs > 0:
        return ExpiryDecision(
            "block_until_late_dependencies_clear",
            "late dependencies still reference stable projection evidence",
            ("rewrite_or_cancel_late_dependencies", "rerun_late_dependency_sweep"),
        )

    if late.unknown_refs > 0:
        return ExpiryDecision(
            "manual_review",
            "unknown references cannot be safely classified by the sweeper",
            ("classify_unknown_refs",),
        )

    cold_ready = (
        cold.archive_written
        and cold.manifest_hash_matches
        and cold.hash_chain_root_matches
        and cold.schema_pin_present
        and cold.redaction_profile_present
        and cold.explanation_packet_rebuilds
        and cold.indexed_by_projection_id
    )
    if not cold_ready:
        if review.used_hot_extensions < review.max_hot_extensions:
            return ExpiryDecision(
                "repair_cold_archive",
                "cold archive is not ready; keep hot evidence until repair completes",
                ("repair_cold_archive", "extend_hot_retention_once"),
            )
        return ExpiryDecision(
            "manual_review",
            "cold archive still broken after extension budget",
            ("open_retention_incident",),
        )

    hot_refs = hot.hot_manifest_refs + hot.cache_refs + hot.search_index_refs + hot.workspace_refs
    if hot_refs > 0:
        return ExpiryDecision(
            "repair_cold_archive",
            "hot path still contains references that must be purged before closeout",
            ("purge_hot_refs", "rerun_hot_path_probe"),
        )

    if not review.owner_acknowledged_expiry:
        return ExpiryDecision(
            "manual_review",
            "owner has not acknowledged expiry handoff for this projection class",
            ("request_owner_ack",),
        )

    return ExpiryDecision(
        "handoff_to_cold_and_purge_hot",
        "cold archive is verifiable and hot path is clean",
        ("write_cold_handoff_closeout", "schedule_cold_audit_sample"),
    )
```

这个判定器有两个关键点：

1. **冷路径没证明可用前，热证据不能清掉**。
2. **热路径清理不是 delete 后结束，而是 delete 后跑 absence probe**。

---

## 4. pi-mono：后台 Worker + Outbox 交接

生产版不要把到期清理写成一个 cron 脚本里的 `if expired then delete`。更好的做法是一个幂等 Worker：

```typescript
type ExpiryAction =
  | "handoff_to_cold_and_purge_hot"
  | "extend_hot_retention"
  | "narrow_for_legal_hold"
  | "block_until_late_dependencies_clear"
  | "repair_cold_archive"
  | "manual_review";

interface ProjectionRetentionExpiryJob {
  projectionId: string;
  projectionVersion: string;
  retentionCloseoutId: string;
  idempotencyKey: string;
}

interface ColdHandoffCloseoutReceipt {
  projectionId: string;
  projectionVersion: string;
  retentionCloseoutId: string;
  coldArchiveId: string;
  manifestHash: string;
  hashChainRoot: string;
  hotPurgeReceiptId: string;
  closedAt: number;
}

class StableProjectionRetentionExpiryWorker {
  constructor(
    private readonly deps: {
      retentionStore: RetentionStore;
      lateDependencySweeper: LateDependencySweeper;
      coldArchive: ColdArchive;
      hotProbe: HotPathProbe;
      hotPurger: HotEvidencePurger;
      legalHoldStore: LegalHoldStore;
      tx: TransactionRunner;
      outbox: Outbox;
    }
  ) {}

  async run(job: ProjectionRetentionExpiryJob): Promise<ExpiryAction> {
    return this.deps.tx.serializable(job.idempotencyKey, async (tx) => {
      const receipt = await this.deps.retentionStore.getCloseout(
        job.retentionCloseoutId,
        tx
      );

      const [review, late, cold, hot] = await Promise.all([
        this.deps.retentionStore.buildExpiryReview(receipt, tx),
        this.deps.lateDependencySweeper.sweep(receipt.projectionId, tx),
        this.deps.coldArchive.probe(receipt, tx),
        this.deps.hotProbe.scan(receipt.projectionId, receipt.projectionVersion, tx),
      ]);

      const decision = decideProjectionRetentionExpiry(
        receipt,
        review,
        late,
        cold,
        hot
      );

      await this.deps.retentionStore.writeDecision(
        {
          retentionCloseoutId: receipt.receiptId,
          decision: decision.decision,
          reason: decision.reason,
          actions: decision.actions,
        },
        tx
      );

      if (decision.decision === "handoff_to_cold_and_purge_hot") {
        const purgeReceipt = await this.deps.hotPurger.purge(
          {
            projectionId: receipt.projectionId,
            projectionVersion: receipt.projectionVersion,
            manifestHash: receipt.manifestHash,
          },
          tx
        );

        const afterPurge = await this.deps.hotProbe.scan(
          receipt.projectionId,
          receipt.projectionVersion,
          tx
        );

        if (afterPurge.totalRefs > 0) {
          throw new Error("hot evidence purge did not converge");
        }

        const closeout: ColdHandoffCloseoutReceipt = {
          projectionId: receipt.projectionId,
          projectionVersion: receipt.projectionVersion,
          retentionCloseoutId: receipt.receiptId,
          coldArchiveId: cold.archiveId,
          manifestHash: receipt.manifestHash,
          hashChainRoot: receipt.hashChainRoot,
          hotPurgeReceiptId: purgeReceipt.receiptId,
          closedAt: Date.now(),
        };

        await this.deps.retentionStore.writeColdHandoffCloseout(closeout, tx);
        await this.deps.outbox.publish(
          {
            topic: "projection.retention.cold_handoff.closed",
            key: receipt.receiptId,
            payload: closeout,
          },
          tx
        );
      }

      if (decision.decision === "repair_cold_archive") {
        await this.deps.outbox.publish(
          {
            topic: "projection.retention.cold_archive.repair_requested",
            key: receipt.receiptId,
            payload: { projectionId: receipt.projectionId, reason: decision.reason },
          },
          tx
        );
      }

      if (decision.decision === "narrow_for_legal_hold") {
        await this.deps.legalHoldStore.writeMinimalView(
          {
            projectionId: receipt.projectionId,
            projectionVersion: receipt.projectionVersion,
            keepFields: ["receiptId", "manifestHash", "hashChainRoot", "schemaPin"],
            reason: decision.reason,
          },
          tx
        );
      }

      return decision.decision;
    });
  }
}
```

这里有三个生产级细节：

- `serializable(job.idempotencyKey)`：防止两个到期 Worker 同时清理同一份证据；
- `outbox.publish(...)`：修复冷归档、关闭交接、后续审计抽样都走事务外发；
- `afterPurge`：清理动作必须被探针验证，不能只相信删除 API 返回 200。

---

## 5. OpenClaw 课程 Cron 类比

我们这个课程任务也有类似链路：

```text
lesson markdown
README entry
TOOLS 已讲内容
Telegram message
git commit / push
```

如果未来要归档一次课程发布证据，不能只留最终 commit。成熟做法是：

- 热路径保留：本次 lesson 文件、README diff、TOOLS diff、Telegram messageId、commit sha；
- 到期复核：确认群消息发出、GitHub push 成功、README 能索引到课程；
- 冷归档：保留 lesson hash、目录项、消息摘要、commit sha；
- 热路径清理：临时草稿、发送缓存、工具原始输出可以删除；
- 反向探针：确认没有未提交文件、没有未发送消息、没有 orphan 草稿。

这就是稳定投影到期交接的缩小版。

---

## 6. 常见坑

### 坑 1：把 `reviewAfter` 当 `deleteAfter`

`reviewAfter` 只是该复核，不代表可以删。删除必须满足 cold archive ready、late dependency clear、legal hold clear。

### 坑 2：冷归档只写不读

只把文件丢进 S3/R2 不够。闸门必须实际 probe：能按 ID 查回、hash 对得上、schema 能解析、能生成 explanation packet。

### 坑 3：清理热索引不清缓存

很多事故不是 DB 里还有旧证据，而是搜索索引、dashboard cache、worker workspace 里还有旧 projection。

### 坑 4：legal hold 下保留太多

legal hold 不是“所有东西永久保留”。正确做法是 narrow minimal view：只保留案件必须字段，raw 和无关字段继续按策略降密或删除。

---

## 7. Checklist

做 Stable Projection Retention Expiry 时，至少检查：

- RetentionCloseoutReceipt 是否存在且 hash chain 完整；
- retainUntil / deleteAfter / reviewAfter 是否区分清楚；
- legal hold / unresolved dispute 是否阻止普通清理；
- late dependency sweep 是否覆盖 callback、export、dashboard、consumer reopen；
- cold archive 是否写入、可查、可验、可解释；
- hot purge 后是否跑 absence probe；
- closeout receipt 是否绑定 coldArchiveId、manifestHash、hashChainRoot、hotPurgeReceiptId；
- 后续 cold audit sampling 是否已排程。

---

## 8. 总结

StableProjectionCloseoutReceipt 证明投影稳定了。

RetentionCloseoutReceipt 证明热路径可以进入保留期。

**ColdHandoffCloseoutReceipt 证明保留期到期后，证据已经从热路径安全交接到冷路径：热路径干净，冷路径可查、可验、可解释。**

成熟 Agent 不把“到期”理解成“删除”，而是理解成一次责任交接：从快速运行访问，交给低成本、低泄露面、仍可审计的冷证据系统。
