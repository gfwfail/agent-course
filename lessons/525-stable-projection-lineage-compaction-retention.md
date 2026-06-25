# 525 - Agent 稳定投影关闭后的血缘压缩与保留策略闸门

> 上一讲我们讲了 Projection 重新放行后的观察窗口：真实读路径稳定、缓存不回流、消费者收敛、回滚可用以后，才能写 StableProjectionCloseoutReceipt。今天继续往后走：稳定关闭不等于可以把所有中间证据、旧版本和回滚指针一删了之。

上一讲是 **Post-Release Projection Observation & Rollback Gate**。今天讲：**Stable Projection Lineage Compaction & Retention Gate**。

一句话：**StableProjectionCloseoutReceipt 证明投影已经稳定，但成熟 Agent 还要把重建血缘、旧版本、回滚别名、审计证据和保留期限收敛成一份 RetentionCloseoutReceipt，既降低运行负担，又不破坏未来审计、解释和回滚能力。**

---

## 1. 为什么稳定后还要做血缘压缩？

很多系统在观察期通过后会这么做：

```text
projection.status = stable
delete old_projection
delete quarantine_workspace
```

这很危险。因为 DP 更正、投影重建、消费者确认、放行观察这些步骤都产生了证据链：

- 原始错误 release 为什么被冻结；
- 哪个 replacement / retraction source 用于重建；
- 哪些 dashboard、cache、report、search index 被刷新；
- 哪些 consumer 已确认收敛；
- 观察期采样过哪些读路径；
- 如果未来发现问题，能回滚到哪个 Last Known Good。

如果直接删除旧路径，短期省空间，长期失去审计和解释能力。

正确做法不是“全保留”或“全删除”，而是分层压缩：

```text
StableProjectionCloseoutReceipt
        ↓
LineageCompactionPlan
        ↓
RollbackAliasReview
        ↓
EvidenceRetentionPolicy
        ↓
CompactionExecutionReceipt
        ↓
RetentionCloseoutReceipt
```

运行面可以瘦身，证据面要可追溯。

---

## 2. 闸门要回答什么？

Stable Projection Lineage Gate 要回答六个问题：

- **哪些证据必须保留**：release receipt、consumer ack、rollback receipt、hash、schema version；
- **哪些原始材料必须删除**：raw payload、临时 workspace、敏感缓存、未脱敏导出；
- **旧版本是否还被引用**：dashboard、signed URL、cursor、async job、consumer bookmark；
- **回滚别名是否还需要保留**：观察窗口刚结束时通常要短 TTL 保留；
- **血缘能否压缩**：把大量 event 压成 manifest + hash + sample summary；
- **删除是否可证明**：清理后要跑反向探针，证明热路径不再读旧版本。

工程上记成一句：

```text
stable closeout -> compact lineage -> lease rollback alias -> retain proof -> purge raw -> verify no old reads
```

注意：血缘压缩不是丢失历史，而是把“每一步的完整原文”压成“足够审计、解释、重放的证明包”。

---

## 3. learn-claude-code：血缘压缩判定器

教学版用纯函数表达。输入稳定关闭收据、旧引用扫描、回滚复核、证据保留策略和敏感数据探针，输出下一步动作。

```python
from dataclasses import dataclass
from typing import Literal

Decision = Literal[
    "compact_and_retain",
    "compact_keep_rollback_alias",
    "extend_observation",
    "block_compaction",
    "manual_review",
]

@dataclass(frozen=True)
class StableProjectionCloseoutReceipt:
    projection_id: str
    stable_version: str
    closeout_receipt_id: str
    observed_read_samples: int
    critical_path_coverage: float
    sensitive_hits: int

@dataclass(frozen=True)
class LineageCompactionPlan:
    keeps_manifest: bool
    keeps_hash_chain: bool
    keeps_consumer_ack_summary: bool
    keeps_replay_seed_ref: bool
    deletes_raw_payload: bool
    deletes_temp_workspace: bool

@dataclass(frozen=True)
class OldReferenceScan:
    dashboard_refs: int
    signed_url_refs: int
    async_job_refs: int
    cache_refs: int
    unknown_refs: int

@dataclass(frozen=True)
class RollbackAliasReview:
    alias_exists: bool
    alias_ttl_seconds: int
    rollback_probe_passed: bool
    lkg_hash_pinned: bool

@dataclass(frozen=True)
class EvidenceRetentionPolicy:
    min_retention_days: int
    allows_raw_retention: bool
    requires_hash_only_stub: bool
    audit_manifest_signed: bool

@dataclass(frozen=True)
class CompactionDecision:
    decision: Decision
    reason: str
    required_actions: tuple[str, ...] = ()

def decide_stable_projection_compaction(
    stable: StableProjectionCloseoutReceipt,
    plan: LineageCompactionPlan,
    refs: OldReferenceScan,
    rollback: RollbackAliasReview,
    policy: EvidenceRetentionPolicy,
) -> CompactionDecision:
    if stable.sensitive_hits > 0:
        return CompactionDecision(
            "block_compaction",
            "stable closeout still contains sensitive read hits",
            ("reopen_projection_case", "purge_sensitive_paths"),
        )

    if stable.critical_path_coverage < 0.95 or stable.observed_read_samples < 100:
        return CompactionDecision(
            "extend_observation",
            "stable closeout lacks enough read-path evidence",
            ("extend_observation_lease", "increase_sampling"),
        )

    old_refs = refs.dashboard_refs + refs.signed_url_refs + refs.async_job_refs + refs.cache_refs
    if old_refs > 0:
        return CompactionDecision(
            "block_compaction",
            "old projection version is still referenced",
            ("rewrite_old_refs", "invalidate_caches", "rerun_reference_scan"),
        )

    if refs.unknown_refs > 0:
        return CompactionDecision(
            "manual_review",
            "unknown old references require human review",
            ("classify_unknown_refs",),
        )

    proof_complete = (
        plan.keeps_manifest
        and plan.keeps_hash_chain
        and plan.keeps_consumer_ack_summary
        and policy.audit_manifest_signed
    )
    if not proof_complete:
        return CompactionDecision(
            "block_compaction",
            "compaction would lose audit proof",
            ("write_signed_manifest", "preserve_hash_chain"),
        )

    if not policy.allows_raw_retention and not plan.deletes_raw_payload:
        return CompactionDecision(
            "block_compaction",
            "raw payload must be removed under retention policy",
            ("purge_raw_payload", "write_hash_only_stub"),
        )

    if not plan.deletes_temp_workspace:
        return CompactionDecision(
            "block_compaction",
            "temporary rebuild workspace must be destroyed",
            ("destroy_temp_workspace", "run_residual_probe"),
        )

    if rollback.alias_exists:
        if not rollback.rollback_probe_passed or not rollback.lkg_hash_pinned:
            return CompactionDecision(
                "block_compaction",
                "rollback alias exists but is not provably usable",
                ("repair_rollback_alias", "pin_lkg_hash"),
            )
        if rollback.alias_ttl_seconds > 0:
            return CompactionDecision(
                "compact_keep_rollback_alias",
                "compact lineage but keep short-lived rollback alias",
                ("schedule_alias_expiry_review", "write_compaction_receipt"),
            )

    return CompactionDecision(
        "compact_and_retain",
        "stable projection can be compacted with retained audit proof",
        ("write_retention_closeout", "run_post_compaction_probe"),
    )
```

这个函数的重点不是复杂，而是顺序：先确认稳定证据，再清理旧引用，再保留审计证明，最后才删除 raw 和临时工作区。

---

## 4. pi-mono：事件流里的压缩 Worker

生产实现通常不在请求线程里做压缩，而是由 Worker 消费稳定关闭事件。

```typescript
type ProjectionEvent =
  | { type: "projection.stable_closeout"; projectionId: string; version: string; receiptId: string }
  | { type: "projection.old_ref_found"; projectionId: string; refType: string; count: number }
  | { type: "projection.compaction_done"; projectionId: string; receiptId: string };

interface ProjectionStore {
  loadStableCloseout(projectionId: string): Promise<StableProjectionCloseoutReceipt>;
  buildCompactionPlan(projectionId: string): Promise<LineageCompactionPlan>;
  scanOldReferences(projectionId: string): Promise<OldReferenceScan>;
  loadRollbackAlias(projectionId: string): Promise<RollbackAliasReview>;
  loadRetentionPolicy(projectionId: string): Promise<EvidenceRetentionPolicy>;
  writeCompactionReceipt(input: {
    projectionId: string;
    stableVersion: string;
    decision: string;
    manifestHash: string;
    retainedUntil: string;
    rollbackAliasExpiresAt?: string;
  }): Promise<string>;
}

class StableProjectionCompactionWorker {
  constructor(
    private readonly store: ProjectionStore,
    private readonly bus: { publish(event: ProjectionEvent): Promise<void> },
  ) {}

  async handle(event: ProjectionEvent) {
    if (event.type !== "projection.stable_closeout") return;

    const [stable, plan, refs, rollback, policy] = await Promise.all([
      this.store.loadStableCloseout(event.projectionId),
      this.store.buildCompactionPlan(event.projectionId),
      this.store.scanOldReferences(event.projectionId),
      this.store.loadRollbackAlias(event.projectionId),
      this.store.loadRetentionPolicy(event.projectionId),
    ]);

    const decision = decideStableProjectionCompaction(
      stable,
      plan,
      refs,
      rollback,
      policy,
    );

    if (decision.decision === "block_compaction") {
      await this.bus.publish({
        type: "projection.old_ref_found",
        projectionId: event.projectionId,
        refType: decision.reason,
        count: 1,
      });
      return;
    }

    if (
      decision.decision === "compact_and_retain" ||
      decision.decision === "compact_keep_rollback_alias"
    ) {
      const receiptId = await this.store.writeCompactionReceipt({
        projectionId: event.projectionId,
        stableVersion: event.version,
        decision: decision.decision,
        manifestHash: await hashProjectionManifest(event.projectionId),
        retainedUntil: retentionDate(policy.min_retention_days),
        rollbackAliasExpiresAt:
          decision.decision === "compact_keep_rollback_alias"
            ? new Date(Date.now() + rollback.alias_ttl_seconds * 1000).toISOString()
            : undefined,
      });

      await this.bus.publish({
        type: "projection.compaction_done",
        projectionId: event.projectionId,
        receiptId,
      });
    }
  }
}
```

这里有两个生产细节：

- `Promise.all` 并行读取证据，但写 receipt 必须串行；
- `compaction_done` 只是控制面事件，后面还要有 post-compaction probe 验证旧路径确实不再被读。

---

## 5. OpenClaw 实战：课程 Cron 的证据收敛

这套模式在 OpenClaw 课程 Cron 里也能类比：

```text
lesson.md 写入
README.md 更新
TOOLS.md 已讲内容更新
Telegram messageId
Git commit SHA
        ↓
Stable closeout
        ↓
压缩成一次课程发布收据
```

一次课程发布完成后，不需要永远保留所有中间草稿，但必须保留：

- lesson 文件路径；
- README 目录项；
- TOOLS 已讲内容项；
- Telegram 发送结果或失败原因；
- git commit SHA；
- 本次 topic 的去重依据。

可以用一个简单 JSONL receipt：

```json
{
  "type": "agent_course_release_receipt",
  "lesson": "525-stable-projection-lineage-compaction-retention.md",
  "topic": "Stable Projection Lineage Compaction & Retention Gate",
  "telegramChatId": "-5115329245",
  "commitSha": "pending",
  "retainedEvidence": [
    "lesson_file",
    "readme_entry",
    "tools_topic_entry",
    "telegram_message_id",
    "git_commit_sha"
  ],
  "discardedEvidence": [
    "draft_notes",
    "temporary_prompt_fragments"
  ]
}
```

这就是同一个思想：**副作用完成以后，要把散落证据收敛成可审计、可解释、可关闭的 proof bundle。**

---

## 6. 常见坑

### 坑 1：把 stable 当成 delete 许可

稳定只说明当前读路径健康，不说明所有历史证据都可以删除。删除必须由 Retention Policy 决定。

### 坑 2：只保留 raw，不保留 manifest

raw 很重、很敏感，而且常常不适合长期保存。更好的做法是保留签名 manifest、hash chain、sample summary 和必要的最小证据。

### 坑 3：回滚别名永久存在

rollback alias 是恢复工具，不是长期入口。它应该有 TTL、命中统计、过期复核和最终清理。

### 坑 4：压缩后不跑反向探针

compaction 写完 receipt 之后，要重新扫 dashboard、cache、search、job queue，证明旧版本不再被热路径引用。

---

## 7. 实战清单

给你的 Agent 加 Stable Projection Lineage Gate，可以从这 6 步开始：

1. 为每个 projection release 写 `StableProjectionCloseoutReceipt`。
2. 扫描旧版本引用：dashboard、cache、signed URL、async job、consumer bookmark。
3. 把重建事件压成 signed manifest + hash chain。
4. raw payload 默认删除，只保留 hash-only stub 或 redacted summary。
5. rollback alias 设置短 TTL，并安排 expiry review。
6. 压缩后跑 post-compaction probe，确认旧版本不再被读。

---

## 总结

今天这讲的核心：

> 成熟 Agent 的稳定关闭，不是“系统看起来好了”，而是把运行面、回滚面、证据面和保留策略都收敛成可审计终态。

`StableProjectionCloseoutReceipt` 证明投影稳定；`RetentionCloseoutReceipt` 证明稳定后的证据该留的留、该删的删、该回滚的还能回滚、该解释的还能解释。

下一次你做任何自动修复、缓存刷新、报表重建、DP release 更正，都可以问自己一句：**我关闭的是任务，还是也关闭了证据链？**
