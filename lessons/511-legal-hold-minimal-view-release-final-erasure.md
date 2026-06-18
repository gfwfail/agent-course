# 511 - Agent 法务保留最小视图释放与最终删除闸门

> Legal hold 不是把派生证据永久冻住。成熟 Agent 要把保留原因、最小视图、释放许可、删除执行和删除后审计串成证据链，证明该留的仍可查，不该留的已经彻底退出运行面和数据面。

上一讲讲了 Derived Evidence Reuse Expiry：派生证据复用许可到期或撤回后，不能只禁控制面，还要清 feature store、cache、queue、workspace，并处理 legal hold 最小视图。

今天继续往后走一步：**legal hold 最小视图也要有释放条件，不能变成无限期的数据坟场**。

常见事故是：

- 复用许可已关闭，但 legal hold snapshot 没有到期时间；
- 法务保留只需要 proof view，却把 raw derived data 一起留下；
- hold release 只删对象存储，没有清搜索索引、审计投影和向量库；
- 删除完成后没有做反向查询，未来事故调查时说不清哪些数据被删了；
- 多租户系统里，一个租户的 hold 阻塞了不相关租户的证据删除。

所以需要一个 **Legal Hold Minimal View Release & Final Erasure Gate**：当保留原因结束或范围缩小时，用证据驱动释放最小视图，并生成最终删除收据。

---

## 核心模型

```
DerivedReuseCloseoutReceipt
        ↓
LegalHoldMinimalView
        ↓
HoldReleaseReadinessReview
        ↓
FinalErasurePlan
        ↓
ErasureExecutionReceipt
        ↓
PostErasureAuditProbe
        ↓
LegalHoldReleaseCloseoutReceipt
```

关键判断：

- legal hold 是否仍有有效 case、期限、owner 和 reason；
- 最小视图是否只包含 proof fields，而不是 raw derived payload；
- release 是否拿到了法务或合规 owner 的确认；
- 删除计划是否覆盖对象存储、索引、缓存、向量库、临时 workspace 和审计投影；
- 删除后探针是否还能命中 raw data 或过期最小视图；
- 如需长期审计，是否只保留不可逆 hash、receipt id、schema pin 和 decision summary。

---

## learn-claude-code：释放闸门纯函数

教学版先把判断写成纯函数。核心原则：**没有释放许可不删，有 raw 残留不关，有审计需要只留不可逆证明**。

```python
from dataclasses import dataclass
from typing import Literal

Decision = Literal[
    "release_and_erase",
    "keep_legal_hold",
    "narrow_to_minimal_view",
    "purge_residual_raw_data",
    "retain_hash_only_audit_stub",
    "open_hold_violation_case",
    "manual_review",
]

@dataclass
class LegalHoldMinimalView:
    hold_id: str
    active_case: bool
    expired: bool
    owner_approved_release: bool
    contains_raw_payload: bool
    contains_only_proof_fields: bool
    affected_tenant_ids: set[str]

@dataclass
class FinalErasurePlan:
    target_tenant_ids: set[str]
    object_store: bool
    search_index: bool
    vector_index: bool
    cache: bool
    temp_workspace: bool
    audit_projection: bool
    keep_hash_stub: bool

@dataclass
class ErasureExecutionReceipt:
    object_store_deleted: bool
    search_index_deleted: bool
    vector_index_deleted: bool
    cache_deleted: bool
    temp_workspace_destroyed: bool
    audit_projection_redacted: bool
    hash_stub_written: bool

@dataclass
class PostErasureAuditProbe:
    raw_object_hits: int
    search_hits: int
    vector_hits: int
    cache_hits: int
    workspace_hits: int
    cross_tenant_hits: int

def decide_legal_hold_release(
    hold: LegalHoldMinimalView,
    plan: FinalErasurePlan,
    receipt: ErasureExecutionReceipt,
    probe: PostErasureAuditProbe,
) -> Decision:
    if hold.contains_raw_payload:
        return "narrow_to_minimal_view"

    if not hold.contains_only_proof_fields:
        return "open_hold_violation_case"

    if hold.active_case and not hold.expired:
        return "keep_legal_hold"

    if not hold.owner_approved_release:
        return "manual_review"

    if not plan.target_tenant_ids <= hold.affected_tenant_ids:
        return "open_hold_violation_case"

    plan_complete = all([
        plan.object_store,
        plan.search_index,
        plan.vector_index,
        plan.cache,
        plan.temp_workspace,
        plan.audit_projection,
    ])
    if not plan_complete:
        return "manual_review"

    receipt_complete = all([
        receipt.object_store_deleted,
        receipt.search_index_deleted,
        receipt.vector_index_deleted,
        receipt.cache_deleted,
        receipt.temp_workspace_destroyed,
        receipt.audit_projection_redacted,
    ])
    if not receipt_complete:
        return "purge_residual_raw_data"

    residual_hits = (
        probe.raw_object_hits
        + probe.search_hits
        + probe.vector_hits
        + probe.cache_hits
        + probe.workspace_hits
    )
    if probe.cross_tenant_hits > 0:
        return "open_hold_violation_case"

    if residual_hits > 0:
        return "purge_residual_raw_data"

    if plan.keep_hash_stub and not receipt.hash_stub_written:
        return "retain_hash_only_audit_stub"

    return "release_and_erase"
```

一个必须覆盖的测试：legal hold 已到期，但最小视图里还混着 raw payload，不能直接删，也不能关闭。

```python
def test_narrows_hold_before_release_when_raw_payload_is_present():
    hold = LegalHoldMinimalView(
        hold_id="hold_42",
        active_case=False,
        expired=True,
        owner_approved_release=True,
        contains_raw_payload=True,
        contains_only_proof_fields=False,
        affected_tenant_ids={"tenant_a"},
    )
    plan = FinalErasurePlan(
        target_tenant_ids={"tenant_a"},
        object_store=True,
        search_index=True,
        vector_index=True,
        cache=True,
        temp_workspace=True,
        audit_projection=True,
        keep_hash_stub=True,
    )
    receipt = ErasureExecutionReceipt(
        object_store_deleted=False,
        search_index_deleted=False,
        vector_index_deleted=False,
        cache_deleted=False,
        temp_workspace_destroyed=False,
        audit_projection_redacted=False,
        hash_stub_written=False,
    )
    probe = PostErasureAuditProbe(
        raw_object_hits=0,
        search_hits=0,
        vector_hits=0,
        cache_hits=0,
        workspace_hits=0,
        cross_tenant_hits=0,
    )

    assert decide_legal_hold_release(hold, plan, receipt, probe) == "narrow_to_minimal_view"
```

这个测试很重要：删除不是第一步。先把 legal hold 收窄成最小 proof view，避免把 raw data 在“等待删除”的过程中继续暴露给查询、索引或下游任务。

---

## pi-mono：Legal Hold Release Worker

生产版可以做成一个 worker：扫描可释放的 hold，生成删除计划，执行删除，再跑反向探针。

```typescript
type HoldReleaseDecision =
  | "release_and_erase"
  | "keep_legal_hold"
  | "narrow_to_minimal_view"
  | "purge_residual_raw_data"
  | "retain_hash_only_audit_stub"
  | "open_hold_violation_case"
  | "manual_review";

interface LegalHoldReleaseJob {
  holdId: string;
  reason: "expired" | "case_closed" | "scope_reduced" | "owner_requested";
}

class LegalHoldReleaseWorker {
  constructor(
    private readonly holds: LegalHoldStore,
    private readonly planner: ErasurePlanBuilder,
    private readonly eraser: EvidenceEraser,
    private readonly probe: PostErasureAuditProbe,
    private readonly receipts: ReceiptStore,
    private readonly incidents: IncidentStore,
  ) {}

  async run(job: LegalHoldReleaseJob) {
    const hold = await this.holds.getMinimalView(job.holdId);

    if (hold.containsRawPayload) {
      await this.holds.narrowToProofFields({
        holdId: job.holdId,
        keepFields: ["receiptId", "schemaHash", "decisionSummary", "evidenceHash"],
      });
      return { decision: "narrow_to_minimal_view" as const };
    }

    if (hold.activeCase && !hold.expired) {
      return { decision: "keep_legal_hold" as const };
    }

    if (!hold.ownerApprovedRelease) {
      await this.incidents.openManualReview({
        kind: "legal_hold_release_missing_owner_approval",
        holdId: job.holdId,
      });
      return { decision: "manual_review" as const };
    }

    const plan = await this.planner.build({
      holdId: job.holdId,
      tenantIds: hold.affectedTenantIds,
      eraseObjectStore: true,
      eraseSearchIndex: true,
      eraseVectorIndex: true,
      eraseCaches: true,
      destroyTemporaryWorkspaces: true,
      redactAuditProjection: true,
      keepHashOnlyAuditStub: true,
    });

    const erasure = await this.eraser.execute(plan, {
      idempotencyKey: `legal-hold-release:${job.holdId}:${plan.planHash}`,
    });

    const auditProbe = await this.probe.scan({
      holdId: job.holdId,
      tenantIds: hold.affectedTenantIds,
      includeVectorIndex: true,
      includeRuntimeCaches: true,
    });

    const decision = decideLegalHoldRelease(hold, plan, erasure, auditProbe);

    if (decision === "release_and_erase") {
      await this.receipts.write({
        type: "LegalHoldReleaseCloseoutReceipt",
        holdId: job.holdId,
        planHash: plan.planHash,
        erasureReceiptId: erasure.receiptId,
        residualProbeHash: auditProbe.probeHash,
      });
    }

    return { decision };
  }
}
```

这里的关键不是 `deleteMany()`，而是三个工程约束：

- 删除计划必须绑定 tenant scope，避免跨租户误删；
- 删除执行必须带 idempotency key，worker 重试不会重复扩大删除面；
- closeout receipt 必须包含 plan hash、erasure receipt 和 probe hash，之后才能解释“为什么删、删了什么、如何证明删完”。

---

## OpenClaw 实战类比

OpenClaw 的课程 cron 每 3 小时会做一条完整链路：

1. 检查 `TOOLS.md` 已讲内容，避免重复；
2. 写 lesson 文件；
3. 更新 `README.md`；
4. 发 Telegram；
5. `git commit && git push`；
6. 再更新本地课程索引。

如果把这条链路当成证据生命周期，`README.md` 和 `TOOLS.md` 就是可查询的 proof view，Telegram messageId 和 commit SHA 是审计 stub。假设未来要清理草稿、临时日志或中间素材，不能删掉最终可审计线索；但也不应该把所有 raw scratch 永远留下。

Agent 系统里的 legal hold 释放也是这个逻辑：**保留证明，不保留不必要的原始数据；删除数据，也留下删除本身的证明**。

---

## 落地建议

- 给所有 legal hold 加 `expiresAt`、`owner`、`reason`、`tenantScope` 和 `releaseCriteria`；
- 把 raw evidence 和 minimal proof view 分开存储，权限也分开；
- release 前先跑 `narrow_to_minimal_view`，再进入 erasure；
- 删除计划覆盖 object store、search index、vector index、cache、workspace、audit projection；
- 删除后必须反向查询，命中任何 raw data 都不能 closeout；
- 长期只保留 hash-only audit stub，避免“审计需要”变成无限保留 raw data 的借口。

一句话：**成熟 Agent 不只会保存证据，也会在证据使命结束时，安全、可审计、可解释地删除证据。**
