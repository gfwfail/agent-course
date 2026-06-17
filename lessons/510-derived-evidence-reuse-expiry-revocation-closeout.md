# 510 - Agent 派生证据复用到期撤回与关闭审计闸门

> DerivedEvidenceReusePermit 允许派生证据在新用途里短期复用，但许可到期不是自然消失。成熟系统要证明下游读者停用了、缓存清了、派生产物没有越界再传播，并留下关闭收据。

上一讲讲了 Derived Evidence Reuse Permit：原始冷证据解冻并销毁后，派生产物如果要进入新用途，必须重新拿许可，绑定 purpose、consumer、TTL、maxReads 和写入范围。

今天继续往后走一步：**派生证据被允许复用，不代表它可以无限留在下游系统里**。

常见事故是：

- 许可 TTL 到了，但下游 feature store 还在 serving；
- `maxReads` 用完后，缓存层仍命中旧 proof view；
- 接收方把派生证据写进共享 dataset，超出了 permit 的 scope；
- 复用链路里生成了二次派生产物，却没有 parent hash 和用途 lineage；
- revoke 只改了控制面，没有验证日志、队列、索引、临时文件和模型上下文缓存。

所以需要一个 **Derived Reuse Expiry & Revocation Closeout Gate**：许可到期或撤回后，系统必须把所有下游使用面收口，并证明没有残留。

---

## 核心模型

```
DerivedEvidenceReusePermit
        ↓
ReuseExecutionReceipt
        ↓
ReuseExpiryReview
        ↓
DownstreamDependencyInventory
        ↓
RevocationExecutionReceipt
        ↓
ResidualDerivedDataProbe
        ↓
DerivedReuseCloseoutReceipt
```

关键判断：

- permit 是否已过期、读次数耗尽、用途结束或被主动撤回；
- 下游 consumer 是否都 ack 停用；
- feature store、cache、queue、log、temporary workspace 是否已清理；
- 二次派生产物是否全部登记 parent hash；
- legal hold 是否只保留最小 proof view，而不是 raw derived data；
- 残留探针是否覆盖查询、索引、对象存储和运行时上下文。

---

## learn-claude-code：复用关闭判定器

教学版先写纯函数。它不连接数据库，只判断一个派生证据复用许可是否可以关闭。

```python
from dataclasses import dataclass
from typing import Literal

Decision = Literal[
    "close_derived_reuse",
    "execute_revocation",
    "purge_residual_derived_data",
    "extend_legal_hold_minimal_view",
    "open_reuse_violation_case",
    "manual_review",
]

@dataclass
class ReusePermit:
    permit_id: str
    expired: bool
    revoked: bool
    reads_remaining: int
    legal_hold: bool
    allowed_consumers: set[str]

@dataclass
class DownstreamInventory:
    active_consumers: set[str]
    undeclared_consumers: set[str]
    secondary_derivatives_without_parent: int
    shared_dataset_writes: int

@dataclass
class RevocationReceipt:
    control_plane_disabled: bool
    consumers_acked: bool
    feature_store_purged: bool
    cache_purged: bool
    queues_drained: bool
    temp_workspace_destroyed: bool

@dataclass
class ResidualProbe:
    query_hits: int
    index_hits: int
    object_store_hits: int
    runtime_context_hits: int
    raw_derived_exports: int

def decide_derived_reuse_closeout(
    permit: ReusePermit,
    inventory: DownstreamInventory,
    receipt: RevocationReceipt,
    probe: ResidualProbe,
) -> Decision:
    should_close = permit.expired or permit.revoked or permit.reads_remaining <= 0
    if not should_close:
        return "manual_review"

    if inventory.undeclared_consumers:
        return "open_reuse_violation_case"

    if inventory.secondary_derivatives_without_parent > 0:
        return "open_reuse_violation_case"

    if inventory.shared_dataset_writes > 0 and not permit.legal_hold:
        return "open_reuse_violation_case"

    if not receipt.control_plane_disabled or permit.allowed_consumers & inventory.active_consumers:
        return "execute_revocation"

    if not receipt.consumers_acked or not receipt.queues_drained:
        return "execute_revocation"

    if not receipt.feature_store_purged or not receipt.cache_purged:
        return "purge_residual_derived_data"

    if not receipt.temp_workspace_destroyed:
        return "purge_residual_derived_data"

    residual_hits = (
        probe.query_hits
        + probe.index_hits
        + probe.object_store_hits
        + probe.runtime_context_hits
    )
    if probe.raw_derived_exports > 0:
        return "open_reuse_violation_case"

    if residual_hits > 0:
        return "purge_residual_derived_data"

    if permit.legal_hold:
        return "extend_legal_hold_minimal_view"

    return "close_derived_reuse"
```

最重要的测试不是 happy path，而是“控制面禁用了，但缓存还在命中”：

```python
def test_requires_purge_when_cache_still_contains_derived_evidence():
    permit = ReusePermit(
        permit_id="reuse_123",
        expired=True,
        revoked=False,
        reads_remaining=0,
        legal_hold=False,
        allowed_consumers={"audit-dashboard"},
    )
    inventory = DownstreamInventory(
        active_consumers=set(),
        undeclared_consumers=set(),
        secondary_derivatives_without_parent=0,
        shared_dataset_writes=0,
    )
    receipt = RevocationReceipt(
        control_plane_disabled=True,
        consumers_acked=True,
        feature_store_purged=True,
        cache_purged=False,
        queues_drained=True,
        temp_workspace_destroyed=True,
    )
    probe = ResidualProbe(
        query_hits=0,
        index_hits=0,
        object_store_hits=0,
        runtime_context_hits=0,
        raw_derived_exports=0,
    )

    assert decide_derived_reuse_closeout(permit, inventory, receipt, probe) == "purge_residual_derived_data"
```

这个测试很贴近真实生产：很多系统以为 revoke 就是删一条 ACL，但 Agent 证据链里最危险的残留往往在缓存、队列和临时 workspace。

---

## pi-mono：Derived Reuse Closeout Worker

生产版可以把关闭流程做成 worker：扫描即将到期或已撤回的 permit，生成依赖清单，执行撤回，再做残留探针。

```typescript
type DerivedReuseCloseoutDecision =
  | "close_derived_reuse"
  | "execute_revocation"
  | "purge_residual_derived_data"
  | "extend_legal_hold_minimal_view"
  | "open_reuse_violation_case"
  | "manual_review";

interface DerivedReuseCloseoutJob {
  permitId: string;
  reason: "expired" | "revoked" | "reads_exhausted" | "purpose_completed";
}

class DerivedReuseCloseoutWorker {
  constructor(
    private readonly permits: DerivedReusePermitStore,
    private readonly inventory: DownstreamInventoryScanner,
    private readonly revoker: DerivedEvidenceRevoker,
    private readonly residualProbe: ResidualDerivedDataProbe,
    private readonly receipts: ReceiptStore,
    private readonly incidents: IncidentStore,
  ) {}

  async run(job: DerivedReuseCloseoutJob) {
    const permit = await this.permits.get(job.permitId);
    const downstream = await this.inventory.scan({
      parentDerivedEvidenceId: permit.derivedEvidenceId,
      allowedConsumers: permit.allowedConsumers,
    });

    let revocation = await this.revoker.currentReceipt(job.permitId);
    if (!revocation?.controlPlaneDisabled || downstream.activeConsumers.length > 0) {
      revocation = await this.revoker.revoke({
        permitId: job.permitId,
        consumers: permit.allowedConsumers,
        purgeFeatureStore: true,
        purgeCaches: true,
        drainQueues: true,
        destroyTemporaryWorkspaces: true,
      });
    }

    const probe = await this.residualProbe.scan({
      derivedEvidenceId: permit.derivedEvidenceId,
      consumers: permit.allowedConsumers,
      includeRuntimeContextCache: true,
    });

    const decision = decideDerivedReuseCloseout(
      permit,
      downstream,
      revocation,
      probe,
    );

    if (decision === "close_derived_reuse") {
      return this.receipts.writeDerivedReuseCloseout({
        permitId: job.permitId,
        revocationReceiptId: revocation.id,
        probeHash: probe.hash,
        closedAt: new Date().toISOString(),
      });
    }

    if (decision === "open_reuse_violation_case") {
      return this.incidents.open({
        type: "derived_evidence_reuse_violation",
        permitId: job.permitId,
        evidence: { downstream, probe },
      });
    }

    return this.receipts.writeCloseoutPending({
      permitId: job.permitId,
      decision,
      revocationReceiptId: revocation?.id,
      probeHash: probe.hash,
    });
  }
}
```

这里有两个生产细节：

- `scan()` 要扫运行面，不只扫数据库表。包括 Redis、对象存储、搜索索引、队列、workspace、LLM context cache。
- `writeDerivedReuseCloseout()` 要是幂等的。Cron 可能重跑，worker 可能并发，closeout receipt 必须用 `permitId + revocationReceiptId + probeHash` 做唯一键。

---

## OpenClaw 实战类比

这次课程 cron 自己就是一个小型 closeout：

1. 先看 TOOLS.md 已讲内容，避免重复主题；
2. 写 lesson 文件，更新 README 和 TOOLS；
3. git commit/push，把本地状态推进远端；
4. Telegram 发课，拿到 message delivery 结果；
5. 最后用 git status 确认工作区干净。

如果把“课程内容”看成派生证据，那么 Telegram、README、TOOLS、Git remote 都是下游 consumer。任务结束时不能只说“我发了”，而要对账：

- README 是否有目录；
- TOOLS 是否有已讲记录；
- Git remote 是否包含 commit；
- Telegram 是否成功送达；
- 本地 workspace 是否没有未提交残留。

这就是 Derived Reuse Closeout 的日常版本：**一个副作用扇出到多个下游时，关闭动作必须逐个证明，而不是凭印象结束。**

---

## 设计要点

- Permit 到期后先禁控制面，再扫运行面；
- 下游 consumer 要显式 ack，不能靠“没报错”当停用证明；
- 缓存、队列、索引、临时 workspace 都算证据残留；
- 二次派生产物必须保留 parent hash 和 purpose lineage；
- legal hold 只保留最小 proof view，不能把 raw derived data 当长期证据；
- closeout receipt 要能回答：谁用过、何时停用、清了哪里、残留探针结果是什么。

一句话总结：

> 派生证据的复用许可不是“给出去就完了”，而是到期后能带证据收回来。成熟 Agent 不只管理访问开始，也管理访问结束。

