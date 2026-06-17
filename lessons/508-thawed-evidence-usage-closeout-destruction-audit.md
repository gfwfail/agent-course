# 508 - Agent 解冻证据使用关闭与销毁审计闸门

> ThawPermit 只说明冷证据可以被解冻一次，不说明它已经被正确使用、没有越权复制、用完后真的销毁。成熟 Agent 要把解冻后的使用、派生产物、缓存清理和销毁证明也做成闭环。

上一讲讲了 Cold Evidence Rehydration：冷证据要能从冷归档安全解冻到隔离热工作区，并用 purpose-bound 的 ThawPermit 控制用途、TTL、操作者和数据范围。

今天继续往后走一步：**拿到解冻许可，不等于审计闭环完成**。

常见事故是：

- 审计员只请求 20 条 decision，却把整个 bundle 导出；
- regression replay 过程中生成了新的 fixture、diff、embedding cache，但没有登记；
- 临时 workspace TTL 到了，主目录删除了，旁路缓存还在；
- 用途是 incident review，结果被拿去训练 prompt 或调阈值；
- 销毁脚本跑了，但没有 hash、日志和外部存储探针证明。

所以需要一个 **Thawed Evidence Usage Closeout Gate**：证据解冻后，必须证明“读了什么、派生了什么、清了什么、销毁了什么、还有没有残留”。

---

## 核心模型

```
PurposeBoundThawPermit
        ↓
ThawedWorkspaceUsageLog
        ↓
DerivedEvidenceInventory
        ↓
UsageScopeReconciliation
        ↓
DestructionExecutionReceipt
        ↓
ResidualDataProbe
        ↓
ThawedEvidenceCloseoutReceipt
```

关键判断：

- permit 是否一次性消费，actor/purpose/scope/TTL 是否匹配；
- usage log 是否覆盖查询、导出、回放、解释重建等动作；
- 派生产物是否都有 parent evidence hash、retention class 和 redaction profile；
- 实际使用是否越过 permit 的 decisionIds、fields、purpose；
- workspace、object store、queue payload、embedding cache、debug log 是否都清理；
- residual probe 是否证明没有残留原始证据。

一句话：**解冻不是把证据拿出来就结束，而是要把证据从拿出到用完再销毁的每一步都留下可复核收据。**

---

## learn-claude-code：使用关闭判定器

教学版继续写纯函数。它不真的删文件，只判断能不能关闭这次解冻使用。

```python
from dataclasses import dataclass
from typing import Literal

Decision = Literal[
    "close_thawed_usage",
    "purge_residual_data",
    "open_usage_violation_case",
    "extend_destruction_observation",
    "manual_review",
]

@dataclass
class ThawPermit:
    consumed_once: bool
    actor_matched: bool
    purpose: Literal["audit", "incident_review", "regression_replay", "legal_hold"]
    allowed_decisions: set[str]
    allowed_fields: set[str]
    ttl_expired: bool

@dataclass
class UsageLog:
    complete: bool
    actor_id: str
    purpose_used: str
    decisions_read: set[str]
    fields_read: set[str]
    raw_exports: int
    replay_runs: int

@dataclass
class DerivedInventory:
    all_artifacts_registered: bool
    all_artifacts_have_parent_hash: bool
    raw_evidence_copied: bool
    retention_class_assigned: bool
    redaction_profile_preserved: bool

@dataclass
class DestructionReceipt:
    workspace_deleted: bool
    object_store_deleted: bool
    queue_payloads_deleted: bool
    embedding_cache_deleted: bool
    debug_logs_redacted: bool
    destruction_hash_recorded: bool

@dataclass
class ResidualProbe:
    raw_evidence_found: bool
    derived_without_receipt_found: bool
    probe_age_minutes: int

def decide_thawed_evidence_closeout(
    permit: ThawPermit,
    usage: UsageLog,
    derived: DerivedInventory,
    destruction: DestructionReceipt,
    residual: ResidualProbe,
) -> Decision:
    if not permit.consumed_once or not permit.actor_matched:
        return "open_usage_violation_case"

    if not usage.complete:
        return "manual_review"

    if usage.purpose_used != permit.purpose:
        return "open_usage_violation_case"

    if not usage.decisions_read.issubset(permit.allowed_decisions):
        return "open_usage_violation_case"

    if not usage.fields_read.issubset(permit.allowed_fields):
        return "open_usage_violation_case"

    if usage.raw_exports > 0 and permit.purpose != "legal_hold":
        return "open_usage_violation_case"

    if derived.raw_evidence_copied:
        return "open_usage_violation_case"

    if not derived.all_artifacts_registered or not derived.all_artifacts_have_parent_hash:
        return "manual_review"

    if not derived.retention_class_assigned or not derived.redaction_profile_preserved:
        return "manual_review"

    destruction_complete = all([
        destruction.workspace_deleted,
        destruction.object_store_deleted,
        destruction.queue_payloads_deleted,
        destruction.embedding_cache_deleted,
        destruction.debug_logs_redacted,
        destruction.destruction_hash_recorded,
    ])

    if not destruction_complete:
        return "purge_residual_data"

    if residual.raw_evidence_found or residual.derived_without_receipt_found:
        return "purge_residual_data"

    if residual.probe_age_minutes > 30:
        return "extend_destruction_observation"

    if not permit.ttl_expired:
        return "extend_destruction_observation"

    return "close_thawed_usage"
```

最该测的不是 happy path，而是“只多读了一点点”的越权。

```python
def test_blocks_field_overread_after_thaw():
    permit = ThawPermit(
        consumed_once=True,
        actor_matched=True,
        purpose="audit",
        allowed_decisions={"d1", "d2"},
        allowed_fields={"decision_id", "reason", "receipt_hash"},
        ttl_expired=True,
    )
    usage = UsageLog(
        complete=True,
        actor_id="auditor-7",
        purpose_used="audit",
        decisions_read={"d1", "d2"},
        fields_read={"decision_id", "reason", "receipt_hash", "raw_prompt"},
        raw_exports=0,
        replay_runs=0,
    )

    decision = decide_thawed_evidence_closeout(
        permit,
        usage,
        DerivedInventory(True, True, False, True, True),
        DestructionReceipt(True, True, True, True, True, True),
        ResidualProbe(False, False, 5),
    )

    assert decision == "open_usage_violation_case"
```

这个测试的意义很大：冷证据治理最容易输在“多读一个字段”。`raw_prompt`、`tool_args`、`user_context` 这类字段通常包含敏感信息，必须按 permit 明确授权。

---

## pi-mono：Thawed Evidence Closeout Worker

生产版可以把它放在 thaw workspace TTL 到期后的关闭 worker 里。注意它不是只删目录，而是先对账，再销毁，再探测，再写 closeout receipt。

```typescript
type CloseoutDecision =
  | "close_thawed_usage"
  | "purge_residual_data"
  | "open_usage_violation_case"
  | "extend_destruction_observation"
  | "manual_review";

interface ThawedEvidenceCloseoutJob {
  thawPermitId: string;
  workspaceId: string;
  actorId: string;
}

class ThawedEvidenceCloseoutWorker {
  constructor(
    private readonly permits: ThawPermitStore,
    private readonly usageLogs: UsageLogStore,
    private readonly derived: DerivedEvidenceStore,
    private readonly destroyer: EphemeralWorkspaceDestroyer,
    private readonly probes: ResidualDataProbeRunner,
    private readonly receipts: ReceiptStore,
    private readonly outbox: Outbox,
  ) {}

  async run(job: ThawedEvidenceCloseoutJob) {
    const permit = await this.permits.getConsumed(job.thawPermitId);
    const usage = await this.usageLogs.collectForWorkspace(job.workspaceId);
    const inventory = await this.derived.inventoryForWorkspace(job.workspaceId);

    const destruction = await this.destroyer.destroyAll({
      workspaceId: job.workspaceId,
      includeObjectStore: true,
      includeQueuePayloads: true,
      includeEmbeddingCache: true,
      redactDebugLogs: true,
    });

    const residual = await this.probes.scan({
      workspaceId: job.workspaceId,
      parentPermitId: job.thawPermitId,
      signatures: permit.evidenceSignatures,
    });

    const decision = decideThawedEvidenceCloseout(
      permit,
      usage,
      inventory,
      destruction,
      residual,
    );

    return this.receipts.transaction(async (tx) => {
      await tx.saveThawedEvidenceCloseout({
        permitId: job.thawPermitId,
        workspaceId: job.workspaceId,
        decision,
        usageHash: usage.hash,
        derivedInventoryHash: inventory.hash,
        destructionHash: destruction.hash,
        residualProbeHash: residual.hash,
      });

      if (decision === "open_usage_violation_case") {
        await this.outbox.publish(tx, {
          type: "security.thawed_evidence_violation",
          permitId: job.thawPermitId,
          workspaceId: job.workspaceId,
        });
      }

      if (decision === "purge_residual_data") {
        await this.outbox.publish(tx, {
          type: "evidence.residual_purge_requested",
          permitId: job.thawPermitId,
          workspaceId: job.workspaceId,
        });
      }

      return decision;
    });
  }
}
```

这里有三个实现要点：

- `destroyAll()` 必须覆盖所有临时面：workspace、object store、queue、cache、debug log；
- `ResidualDataProbeRunner` 不能只看路径，要用 evidence signature/hash 搜索残留；
- closeout receipt 要保存 usage、derived、destruction、probe 四个 hash，方便未来复盘。

---

## OpenClaw 课程 Cron 的类比

这条课程流水线也有类似闭环：

- 任务拿到“本次授课许可”：cron payload；
- 读取已讲内容，避免重复；
- 生成 lesson 文件，这是派生产物；
- 更新 README 和 TOOLS，这是索引和长期记忆；
- git commit/push，这是外部持久化收据；
- Telegram 发送，这是交付收据。

如果只写了 lesson，没有更新 README，未来索引就漂移；如果发了 Telegram，没有 commit，仓库证据就缺；如果 commit 了，没有更新 TOOLS，下一次就可能重复。

这和冷证据解冻一样：**一次动作跨多个存储面时，关闭条件必须覆盖所有面，而不是只看主路径成功。**

---

## 实战检查清单

设计 thaw closeout 时，至少问 8 个问题：

1. ThawPermit 是否被 CAS 一次性消费？
2. 使用日志是否能还原 actor、purpose、decisionIds、fields？
3. 是否阻断了超范围 decision 和 field？
4. 派生产物是否都有 parent hash？
5. raw evidence 是否禁止复制，legal hold 是否例外登记？
6. workspace 外的 object、queue、cache、log 是否也被销毁？
7. residual probe 是否用 hash/signature 搜索残留？
8. closeout receipt 是否能让未来审计员不接触原始证据也能复核？

---

## 小结

Cold Evidence Rehydration 解决的是“冷证据能不能安全拿出来”。

Thawed Evidence Usage Closeout 解决的是“拿出来以后有没有按许可使用，并且用完真的消失”。

成熟 Agent 的证据系统不是一个 archive bucket，而是一条完整证据链：保留、取回、解冻、使用、派生、销毁、残留探测、关闭收据。只有这样，历史证据才既可用，又不会变成新的风险源。
