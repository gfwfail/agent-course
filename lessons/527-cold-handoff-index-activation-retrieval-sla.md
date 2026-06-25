# 527 - Agent 冷证据交接后的索引激活与取回 SLA 闸门

> 上一讲我们讲了 Stable Projection Retention Expiry & Cold Evidence Handoff Gate：保留期到期时，先证明冷归档完整、迟到依赖清空、热路径反向探针通过，才能清理热证据。今天继续往后走：热证据清了以后，冷证据系统不能只是“文件已经在冷库里”，它必须正式接管审计读取路径。

今天讲：**Cold Handoff Index Activation & Retrieval SLA Gate**。

一句话：**ColdHandoffCloseoutReceipt 写完以后，Agent 要把冷索引激活、访问租约、取回演练、SLA 监控和热路径缺失哨兵串起来；冷证据必须可找、可验、可解释、可限权读取，否则热路径清理只是把问题藏进了冷库。**

---

## 1. 为什么冷归档完成还不够？

上一课的 `ColdHandoffCloseoutReceipt` 证明了：

- 冷归档对象写入成功；
- manifest hash、hash chain、schema pin 都能对上；
- 热路径 cache/search/workspace 已经清理；
- legal hold 和迟到依赖已经处理；
- hot purge 有收据。

但它通常只证明“交接那一刻完成了”。生产系统还需要回答：

- 审计员以后按 `projectionId` 能不能找到冷证据？
- 找到以后能不能确认它没有损坏？
- 取回是否需要租约、purpose、字段范围和 TTL？
- 冷证据是否能重建不含 raw payload 的解释包？
- 冷路径查询慢了、索引坏了、对象丢了，谁来告警？
- 热路径被清理后，是否有 worker 又把旧证据写回热缓存？

很多团队会把 cold archive 当成对象存储目录：

```text
s3://archive/projection/p-123/receipt.json
```

这只能解决“存了”。Agent 系统要解决的是“未来在正确权限下能复盘”。因此要有一个冷路径激活闸门：

```text
ColdHandoffCloseoutReceipt
        ↓
ColdIndexActivationPlan
        ↓
ColdRetrievalLeasePolicy
        ↓
SyntheticRetrievalDrill
        ↓
RetrievalSlaMonitor
        ↓
HotAbsenceSentinel
        ↓
ColdPathActivationReceipt
```

核心区别：

- **archive** 是存储动作；
- **index activation** 是让审计读取路径接管；
- **retrieval SLA** 是持续证明冷路径可用；
- **hot absence sentinel** 是持续证明旧证据没有回流热路径。

---

## 2. 闸门要回答什么？

Cold Handoff Index Activation Gate 要回答八个问题：

- **索引是否激活**：`projectionId`、`releaseId`、`receiptId`、`consumerId` 是否都能定位；
- **对象是否可验**：manifest hash、bundle hash、hash chain root 是否匹配；
- **读取是否受控**：必须通过 `ColdRetrievalLease`，绑定 actor、purpose、field scope、TTL；
- **解释是否可重建**：能生成 declassified explanation packet，而不是把 raw payload 解冻出来；
- **SLA 是否有基线**：P95 取回时间、失败率、冷索引缺失率要被监控；
- **失败是否能修复**：索引丢失可 rebuild，对象损坏要 quarantine，租约越界要开 violation case；
- **热路径是否保持空**：热 cache/search/workspace 不应重新出现旧 projection evidence；
- **是否有关闭收据**：激活成功后写 `ColdPathActivationReceipt`，作为后续审计入口。

工程上记成一句：

```text
activate index -> bind lease policy -> run synthetic retrieval -> monitor SLA -> watch hot absence
```

注意：这不是数据备份课。它是 Agent 证据链从热运行面切到冷审计面的接管协议。

---

## 3. learn-claude-code：冷路径激活判定器

教学版用纯函数表达。输入冷交接收据、索引探针、租约策略、取回演练、SLA 快照和热路径哨兵，输出下一步动作。

```python
from dataclasses import dataclass
from typing import Literal

Decision = Literal[
    "activate_cold_path",
    "rebuild_cold_index",
    "repair_archive_bundle",
    "tighten_retrieval_lease_policy",
    "extend_hot_absence_observation",
    "quarantine_cold_path",
    "manual_review",
]

@dataclass(frozen=True)
class ColdHandoffCloseoutReceipt:
    projection_id: str
    release_id: str
    receipt_id: str
    archive_bundle_hash: str
    manifest_hash: str
    hash_chain_root: str
    hot_purge_receipt_id: str
    closed_epoch: int

@dataclass(frozen=True)
class ColdIndexProbe:
    indexed_by_projection_id: bool
    indexed_by_release_id: bool
    indexed_by_receipt_id: bool
    indexed_by_consumer_id: bool
    manifest_hash_matches: bool
    bundle_hash_matches: bool
    hash_chain_root_matches: bool

@dataclass(frozen=True)
class ColdRetrievalLeasePolicy:
    requires_actor: bool
    requires_purpose: bool
    requires_field_scope: bool
    max_ttl_seconds: int
    allows_raw_payload: bool
    writes_usage_receipt: bool

@dataclass(frozen=True)
class SyntheticRetrievalDrill:
    retrieval_succeeded: bool
    p95_latency_ms: int
    explanation_packet_rebuilds: bool
    raw_payload_exposed: bool
    usage_receipt_written: bool

@dataclass(frozen=True)
class RetrievalSlaSnapshot:
    max_p95_latency_ms: int
    error_rate: float
    max_error_rate: float
    missing_index_count: int
    corrupted_bundle_count: int

@dataclass(frozen=True)
class HotAbsenceSentinel:
    hot_cache_refs: int
    hot_search_refs: int
    hot_workspace_refs: int
    unexpected_rehydrations: int

@dataclass(frozen=True)
class ActivationDecision:
    decision: Decision
    reason: str
    actions: tuple[str, ...] = ()

def decide_cold_path_activation(
    receipt: ColdHandoffCloseoutReceipt,
    index: ColdIndexProbe,
    lease: ColdRetrievalLeasePolicy,
    drill: SyntheticRetrievalDrill,
    sla: RetrievalSlaSnapshot,
    hot: HotAbsenceSentinel,
) -> ActivationDecision:
    index_complete = (
        index.indexed_by_projection_id
        and index.indexed_by_release_id
        and index.indexed_by_receipt_id
        and index.indexed_by_consumer_id
    )
    if not index_complete or sla.missing_index_count > 0:
        return ActivationDecision(
            "rebuild_cold_index",
            "cold archive exists but lookup paths are incomplete",
            ("rebuild_secondary_indexes", "rerun_cold_index_probe"),
        )

    archive_verified = (
        index.manifest_hash_matches
        and index.bundle_hash_matches
        and index.hash_chain_root_matches
        and sla.corrupted_bundle_count == 0
    )
    if not archive_verified:
        return ActivationDecision(
            "repair_archive_bundle",
            "cold bundle cannot be verified against handoff receipt",
            ("quarantine_bundle", "restore_from_redundant_copy", "write_repair_receipt"),
        )

    lease_safe = (
        lease.requires_actor
        and lease.requires_purpose
        and lease.requires_field_scope
        and lease.max_ttl_seconds <= 3600
        and not lease.allows_raw_payload
        and lease.writes_usage_receipt
    )
    if not lease_safe:
        return ActivationDecision(
            "tighten_retrieval_lease_policy",
            "cold evidence retrieval must be purpose-bound, scoped, short-lived, and receipted",
            ("disable_unscoped_reads", "require_usage_reconciliation"),
        )

    if not drill.retrieval_succeeded or not drill.explanation_packet_rebuilds:
        return ActivationDecision(
            "manual_review",
            "synthetic retrieval cannot rebuild a usable explanation packet",
            ("open_cold_retrieval_failure_case",),
        )

    if drill.raw_payload_exposed:
        return ActivationDecision(
            "quarantine_cold_path",
            "cold retrieval exposed raw payload outside an explicit thaw permit",
            ("revoke_recent_leases", "purge_retrieval_cache", "open_privacy_violation_case"),
        )

    if not drill.usage_receipt_written:
        return ActivationDecision(
            "tighten_retrieval_lease_policy",
            "retrieval succeeded without durable usage receipt",
            ("block_retrieval_until_receipts_enabled",),
        )

    if drill.p95_latency_ms > sla.max_p95_latency_ms or sla.error_rate > sla.max_error_rate:
        return ActivationDecision(
            "extend_hot_absence_observation",
            "cold path works but SLA is not stable enough for unattended audit reads",
            ("increase_cold_index_replication", "schedule_sla_recheck"),
        )

    hot_refs = hot.hot_cache_refs + hot.hot_search_refs + hot.hot_workspace_refs
    if hot_refs > 0 or hot.unexpected_rehydrations > 0:
        return ActivationDecision(
            "extend_hot_absence_observation",
            "old evidence reappeared in hot path after purge",
            ("purge_hot_refs_again", "identify_rehydration_source", "rerun_hot_absence_sentinel"),
        )

    return ActivationDecision(
        "activate_cold_path",
        "cold evidence is indexed, verifiable, lease-protected, retrievable within SLA, and absent from hot path",
        ("write_cold_path_activation_receipt", "schedule_periodic_retrieval_drill"),
    )
```

重点是顺序：

1. 先证明能找到；
2. 再证明没坏；
3. 再证明不能随便读；
4. 再证明能解释；
5. 最后才让冷路径成为正式审计入口。

---

## 4. pi-mono：把冷路径变成 Worker 和收据

生产版不要把这个逻辑塞进一个 cron 脚本里。更好的结构是：

- `ColdEvidenceIndexProjector`：从 handoff closeout 事件构建冷索引；
- `ColdRetrievalLeaseService`：签发 purpose-bound 读取租约；
- `SyntheticRetrievalDrillWorker`：定期抽样取回并重建解释包；
- `HotAbsenceSentinelWorker`：反查热 cache/search/workspace；
- `ColdPathActivationGate`：聚合所有探针，写激活收据。

TypeScript 结构可以这样设计：

```ts
type ColdPathDecision =
  | "activate_cold_path"
  | "rebuild_cold_index"
  | "repair_archive_bundle"
  | "tighten_retrieval_lease_policy"
  | "extend_hot_absence_observation"
  | "quarantine_cold_path"
  | "manual_review";

interface ColdPathActivationInput {
  handoffReceipt: {
    projectionId: string;
    releaseId: string;
    receiptId: string;
    archiveBundleHash: string;
    manifestHash: string;
    hashChainRoot: string;
  };
  indexProbe: {
    lookupKeys: Array<"projectionId" | "releaseId" | "receiptId" | "consumerId">;
    manifestHashMatches: boolean;
    bundleHashMatches: boolean;
    hashChainRootMatches: boolean;
  };
  leasePolicy: {
    requiresActor: boolean;
    requiresPurpose: boolean;
    requiresFieldScope: boolean;
    maxTtlSeconds: number;
    allowsRawPayload: boolean;
    writesUsageReceipt: boolean;
  };
  retrievalDrill: {
    ok: boolean;
    p95LatencyMs: number;
    explanationPacketRebuilds: boolean;
    rawPayloadExposed: boolean;
    usageReceiptId?: string;
  };
  sla: {
    maxP95LatencyMs: number;
    maxErrorRate: number;
    errorRate: number;
    missingIndexCount: number;
    corruptedBundleCount: number;
  };
  hotAbsence: {
    hotRefs: number;
    unexpectedRehydrations: number;
  };
}

interface ColdPathActivationResult {
  decision: ColdPathDecision;
  reason: string;
  nextActions: string[];
}

export function decideColdPathActivation(
  input: ColdPathActivationInput,
): ColdPathActivationResult {
  const requiredKeys = ["projectionId", "releaseId", "receiptId", "consumerId"] as const;
  const hasAllKeys = requiredKeys.every((key) => input.indexProbe.lookupKeys.includes(key));

  if (!hasAllKeys || input.sla.missingIndexCount > 0) {
    return {
      decision: "rebuild_cold_index",
      reason: "cold lookup keys are incomplete",
      nextActions: ["rebuildSecondaryIndexes", "rerunIndexProbe"],
    };
  }

  const archiveVerified =
    input.indexProbe.manifestHashMatches &&
    input.indexProbe.bundleHashMatches &&
    input.indexProbe.hashChainRootMatches &&
    input.sla.corruptedBundleCount === 0;

  if (!archiveVerified) {
    return {
      decision: "repair_archive_bundle",
      reason: "cold bundle verification failed",
      nextActions: ["quarantineBundle", "restoreRedundantCopy"],
    };
  }

  const leaseSafe =
    input.leasePolicy.requiresActor &&
    input.leasePolicy.requiresPurpose &&
    input.leasePolicy.requiresFieldScope &&
    input.leasePolicy.maxTtlSeconds <= 3600 &&
    !input.leasePolicy.allowsRawPayload &&
    input.leasePolicy.writesUsageReceipt;

  if (!leaseSafe) {
    return {
      decision: "tighten_retrieval_lease_policy",
      reason: "retrieval lease policy is too broad",
      nextActions: ["disableUnscopedReads", "requireUsageReceipt"],
    };
  }

  if (input.retrievalDrill.rawPayloadExposed) {
    return {
      decision: "quarantine_cold_path",
      reason: "synthetic retrieval exposed raw payload",
      nextActions: ["revokeRecentLeases", "purgeRetrievalCache", "openViolationCase"],
    };
  }

  if (!input.retrievalDrill.ok || !input.retrievalDrill.explanationPacketRebuilds) {
    return {
      decision: "manual_review",
      reason: "synthetic retrieval failed to rebuild explanation packet",
      nextActions: ["openColdRetrievalFailureCase"],
    };
  }

  if (!input.retrievalDrill.usageReceiptId) {
    return {
      decision: "tighten_retrieval_lease_policy",
      reason: "retrieval succeeded without usage receipt",
      nextActions: ["blockRetrievalUntilReceiptsEnabled"],
    };
  }

  const slaHealthy =
    input.retrievalDrill.p95LatencyMs <= input.sla.maxP95LatencyMs &&
    input.sla.errorRate <= input.sla.maxErrorRate;

  if (!slaHealthy || input.hotAbsence.hotRefs > 0 || input.hotAbsence.unexpectedRehydrations > 0) {
    return {
      decision: "extend_hot_absence_observation",
      reason: "cold path or hot absence is not stable yet",
      nextActions: ["rerunSlaMonitor", "rerunHotAbsenceSentinel"],
    };
  }

  return {
    decision: "activate_cold_path",
    reason: "cold path is ready to become the audit entrypoint",
    nextActions: ["writeActivationReceipt", "schedulePeriodicRetrievalDrill"],
  };
}
```

Worker 侧要注意两点。

第一，冷索引必须是可重建的 projection，而不是唯一真相：

```text
source of truth = immutable archive bundle + handoff receipt
secondary index = projectionId/releaseId/receiptId/consumerId lookup
```

索引坏了可以 rebuild；bundle hash 对不上才是严重事件。

第二，租约不是普通 API token。它应该绑定：

```json
{
  "leaseId": "clr_01J...",
  "actor": "audit-worker",
  "purpose": "rebuild-explanation-packet",
  "projectionId": "proj_123",
  "fieldScope": ["manifest", "hashChain", "consumerAckSummary"],
  "expiresAt": "2026-06-26T01:00:00Z",
  "allowRawPayload": false
}
```

没有 purpose 和 field scope 的冷证据读取，等于把热路径删掉后又开了一个更隐蔽的后门。

---

## 5. OpenClaw：课程 Cron 本身也是冷路径激活类比

OpenClaw 的课程自动化也有类似链路：

```text
lesson.md 写入
README.md 目录更新
TOOLS.md 已讲内容更新
Telegram 发送
git commit/push
```

如果把每次课程看成一个 projection，那么：

- `lessons/527-*.md` 是不可变 lesson bundle；
- `README.md` 是热索引，方便人快速找到；
- `TOOLS.md` 是已讲内容索引，防止重复；
- Telegram message 是外部消费者投影；
- git commit 是证据收据。

冷路径激活类比就是：

- 未来按 lesson number 能找到文件；
- README 和 TOOLS 都能定位这节课；
- Telegram 消息有对应内容；
- git commit 能证明哪次 cron 生成；
- 如果 README 索引坏了，可以从 lessons 目录重建；
- 如果 lesson 文件 hash 对不上，就不能悄悄覆盖，必须开修复 case。

一个 OpenClaw 风格的检查清单：

```ts
type CourseColdPathProbe = {
  lessonFileExists: boolean;
  readmeHasEntry: boolean;
  toolsHasTopic: boolean;
  telegramDelivered: boolean;
  gitCommitPushed: boolean;
};

function decideCourseCloseout(probe: CourseColdPathProbe) {
  if (!probe.lessonFileExists) return "rewrite_lesson_bundle";
  if (!probe.readmeHasEntry) return "rebuild_readme_index";
  if (!probe.toolsHasTopic) return "repair_topic_registry";
  if (!probe.telegramDelivered) return "retry_delivery";
  if (!probe.gitCommitPushed) return "push_commit";
  return "closeout_verified";
}
```

这就是同一个思想：**不要只做副作用，要给副作用留下可查、可验、可恢复的索引和收据。**

---

## 6. 常见坑

**坑 1：只有对象存储，没有查询索引。**

几年后审计只知道 projectionId，不知道对象路径，冷证据等于找不到。索引至少覆盖 projectionId、releaseId、receiptId、consumerId。

**坑 2：冷取回默认返回 raw。**

冷证据默认应该返回解释包、manifest、hash proof、summary。raw payload 只能走单独 thaw permit。

**坑 3：租约没有 usage receipt。**

谁读了、为什么读、读了哪些字段、读完有没有派生产物，都要有记录。否则冷库会变成绕过热路径权限的地方。

**坑 4：只测写入，不测读取。**

归档成功不代表未来能读。必须有 synthetic retrieval drill，定期抽样取回、验 hash、重建 explanation packet。

**坑 5：热路径清理后没有哨兵。**

旧 worker、缓存刷新、搜索重建都可能把旧证据重新写回热路径。HotAbsenceSentinel 要持续跑一段观察窗口。

---

## 7. 实战落地建议

最小可用版本只需要五张表/集合：

```text
cold_handoff_closeouts
cold_archive_bundles
cold_archive_indexes
cold_retrieval_leases
cold_path_activation_receipts
```

最小 Worker：

```text
ColdIndexProjector
SyntheticRetrievalDrillWorker
HotAbsenceSentinelWorker
ColdPathActivationGate
```

最低限度指标：

```text
cold_index_missing_total
cold_bundle_verify_fail_total
cold_retrieval_p95_ms
cold_retrieval_error_rate
cold_raw_exposure_total
hot_absence_violation_total
```

生产判断规则：

- 索引缺失：rebuild index；
- hash 不匹配：quarantine bundle；
- 租约过宽：block reads；
- raw 曝光：open privacy violation；
- SLA 不达标：extend observation；
- 热路径回流：purge + find writer；
- 全部通过：write activation receipt。

---

## 8. 今日总结

ColdHandoffCloseoutReceipt 不是“证据治理结束”，而是“冷路径开始负责”的交接点。

成熟 Agent 的冷证据系统要做到：

- 存得下；
- 找得到；
- 验得过；
- 读得受控；
- 解释能重建；
- SLA 可监控；
- 热路径不回流；
- 每次激活都有收据。

一句话收尾：

> 成熟 Agent 不把冷归档当仓库，而把它当一条受租约保护、有 SLA、有演练、有收据的审计读取路径。

