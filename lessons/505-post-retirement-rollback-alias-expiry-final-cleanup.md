# 505 - Agent 退役后的回滚别名过期与最终清理闸门

> 旧 primary 退役后，保留 rollback alias 是保险；但保险不能永远挂在生产路径上。成熟 Agent 要让回滚别名按证据过期，并在最终清理前再证明没有迟到依赖。

上一讲讲了 Post-Cutover Primary Retirement：切流稳定后，旧 primary 不能直接删除，要先冻结证据、分阶段停用，并保留短期 rollback alias。今天继续往后走一步：**rollback alias 到期时，如何安全摘掉最后一条旧路径**。

很多系统会在这里踩坑：

- rollback alias 永久保留，后续依赖误连到旧路径
- 清理太早，事故复盘或补偿任务还需要旧 runtime fingerprint
- 外部回调延迟到达，却找不到旧 route 的解释和归档
- 删除旧配置后，审计链出现断点

所以最终清理不是 `delete alias`，而是一个到期复核、依赖扫描、证据封存、清理收据的闸门。

---

## 核心模型

```
PrimaryRetirementCloseoutReceipt
        ↓
RollbackAliasLease
        ↓
AliasExpiryReadinessReview
        ↓
LateDependencyScan
        ↓
FinalCleanupPlan
        ↓
CleanupExecutionReceipt
        ↓
PostCleanupAuditProbe
        ↓
FinalRetirementReceipt
```

关键判断：

- rollback alias 的 lease 是否真的到期
- 最近是否还有 rollback probe 失败
- 是否存在迟到 callback、补偿任务、审计查询依赖
- alias 是否被新任务误用
- 证据 bundle 是否已封存并可按 hash 找回
- 清理后是否还能解释旧决策

---

## learn-claude-code：过期闸门纯函数

先把最终清理决策写成纯函数，避免清理逻辑散落在脚本里。

```python
from dataclasses import dataclass
from typing import Literal

Decision = Literal[
    "cleanup_alias",
    "extend_alias_lease",
    "freeze_more_evidence",
    "block_cleanup",
    "manual_review",
]

@dataclass
class RollbackAliasLease:
    expired: bool
    kept_days: int
    rollback_probe_ok: bool
    last_probe_hours_ago: int
    alias_hit_count_24h: int

@dataclass
class CleanupReadiness:
    late_callbacks_open: int
    compensation_jobs_open: int
    audit_queries_using_alias: int
    evidence_bundle_frozen: bool
    evidence_hash_verified: bool
    runtime_fingerprint_archived: bool

def decide_final_cleanup(
    lease: RollbackAliasLease,
    readiness: CleanupReadiness,
) -> Decision:
    if not lease.expired:
        return "extend_alias_lease"

    if not lease.rollback_probe_ok or lease.last_probe_hours_ago > 24:
        return "manual_review"

    if lease.alias_hit_count_24h > 0:
        return "block_cleanup"

    if readiness.late_callbacks_open > 0 or readiness.compensation_jobs_open > 0:
        return "block_cleanup"

    if readiness.audit_queries_using_alias > 0:
        return "extend_alias_lease"

    evidence_ready = (
        readiness.evidence_bundle_frozen
        and readiness.evidence_hash_verified
        and readiness.runtime_fingerprint_archived
    )

    if not evidence_ready:
        return "freeze_more_evidence"

    return "cleanup_alias"
```

这里最重要的是 `alias_hit_count_24h`。如果旧 alias 还在被访问，说明系统里有隐藏依赖或误路由，不能把“没人报错”当成“没人使用”。

测试要覆盖最危险的误删：

```python
def test_blocks_cleanup_when_alias_still_receives_traffic():
    lease = RollbackAliasLease(
        expired=True,
        kept_days=7,
        rollback_probe_ok=True,
        last_probe_hours_ago=2,
        alias_hit_count_24h=3,
    )
    readiness = CleanupReadiness(
        late_callbacks_open=0,
        compensation_jobs_open=0,
        audit_queries_using_alias=0,
        evidence_bundle_frozen=True,
        evidence_hash_verified=True,
        runtime_fingerprint_archived=True,
    )

    assert decide_final_cleanup(lease, readiness) == "block_cleanup"
```

---

## pi-mono：Final Cleanup Worker

生产里建议把最终清理做成 worker，并且每一步都写 receipt。

```typescript
type CleanupDecision =
  | "cleanup_alias"
  | "extend_alias_lease"
  | "freeze_more_evidence"
  | "block_cleanup"
  | "manual_review";

interface FinalCleanupJob {
  cutoverId: string;
  retiredPath: string;
  rollbackAlias: string;
  retirementReceiptId: string;
}

class PostRetirementCleanupWorker {
  constructor(
    private readonly receipts: ReceiptStore,
    private readonly runtime: RuntimeControlPlane,
    private readonly evidence: EvidenceStore,
    private readonly audit: AuditLog,
  ) {}

  async run(job: FinalCleanupJob) {
    const lease = await this.runtime.getRollbackAliasLease(job.rollbackAlias);
    const readiness = await this.evidence.reviewCleanupReadiness(job.cutoverId);
    const decision = decideFinalCleanup(lease, readiness);

    await this.audit.append({
      type: "rollback_alias_cleanup_decision",
      cutoverId: job.cutoverId,
      rollbackAlias: job.rollbackAlias,
      retiredPath: job.retiredPath,
      decision,
    });

    if (decision === "freeze_more_evidence") {
      await this.evidence.freezeBundle({
        cutoverId: job.cutoverId,
        include: ["runtime_fingerprint", "route_table", "audit_index"],
      });
      return { status: "evidence_frozen_retry_later" };
    }

    if (decision !== "cleanup_alias") {
      return { status: decision };
    }

    await this.runtime.disableRollbackAlias(job.rollbackAlias);

    const probe = await this.runtime.probePostCleanup({
      oldAlias: job.rollbackAlias,
      retiredPath: job.retiredPath,
    });

    if (!probe.auditReadable || probe.aliasStillRoutes) {
      await this.runtime.restoreRollbackAlias(job.rollbackAlias);
      return { status: "manual_review", reason: "post_cleanup_probe_failed" };
    }

    return this.receipts.writeFinalRetirement({
      cutoverId: job.cutoverId,
      retiredPath: job.retiredPath,
      rollbackAliasRemoved: true,
      evidenceHash: readiness.evidenceHash,
      cleanedAt: new Date().toISOString(),
    });
  }
}
```

注意：清理不是先删再看，而是**禁用 alias → 运行 post-cleanup probe → 失败则恢复 alias**。这让最终清理也具备回滚点。

---

## OpenClaw：课程 Cron 的类比

这门课的自动发布链路也有类似问题：

- Telegram 发出去了
- README 更新了
- TOOLS 已讲列表更新了
- git commit/push 成功了

如果后续要清理某个旧发布路径，比如旧脚本、旧目录、旧分支，不能只看“新路径已经跑通”。还要确认：

- 旧路径最近没有被 cron 调用
- 历史课程还能通过 README 链接打开
- Telegram messageId / commit hash / lesson file 还能对账
- 旧路径删除后不会影响回放和审计

可以把它抽象成一个清理收据：

```json
{
  "type": "final_retirement_receipt",
  "target": "old-course-publisher",
  "aliasRemoved": true,
  "lastHitAt": null,
  "evidenceHash": "sha256:...",
  "postCleanupProbe": {
    "readmeLinksOk": true,
    "lessonFilesOk": true,
    "gitRemoteOk": true
  }
}
```

成熟 Agent 的自动化不是一直加新流程，也要会安全地下线旧流程。

---

## 实战建议

1. 给所有 rollback alias 加 lease，不要永久保留。
2. 清理前统计 alias hit count，命中过就先查隐藏依赖。
3. 证据 bundle 要在删除 runtime 前冻结，不要事后从日志里捞。
4. 清理动作要写 receipt，并保留 post-cleanup probe。
5. 清理失败时恢复 alias，而不是让系统停在半删除状态。

一句话总结：**旧路径退役后，最后一步不是删除，而是证明它已经无人依赖、证据可查、删除可回滚。**
