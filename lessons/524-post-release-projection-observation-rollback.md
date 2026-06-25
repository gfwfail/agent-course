# 524 - Agent 投影重新放行后的观察窗口与回滚闸门

> 上一讲我们讲了被隔离的 dashboard、cache、report、derived artifact 如何重建、重验并重新放行。今天继续往后走：重新放行不等于完全结束，投影重新进入服务路径后，还要证明旧缓存不会回流、消费者读到的新值稳定、必要时可以自动回滚。

上一讲是 **Quarantined Projection Rebuild, Revalidation & Release Gate**。今天讲：**Post-Release Projection Observation & Rollback Gate**。

一句话：**ProjectionReleaseReceipt 只能证明放行那一刻证据通过；成熟 Agent 还要创建短观察窗口，持续采样读路径、缓存层、下游消费者和回滚能力，最后写 StableProjectionCloseoutReceipt 或 RollbackReceipt。**

---

## 1. 为什么 release 后还要观察？

很多系统把投影状态改回 active 就结束：

```text
projection.status = active
releasedBy = ProjectionReleaseReceipt
```

问题是，投影的真实读路径通常不止一个：

- CDN / edge cache 可能还保留旧 response；
- dashboard 前端可能有 localStorage 或 query cache；
- 下游 report worker 可能还引用旧 projection version；
- search / vector / OLAP projection 可能异步更新；
- consumer 可能通过老链接、老 cursor 或老 signed URL 访问旧产物；
- 回滚别名可能指向错误版本。

上一讲证明“重建结果可以放行”，这一讲证明“放行后真实世界继续稳定”。

正确流程是：

```text
ProjectionReleaseReceipt
        ↓
PostReleaseObservationLease
        ↓
ReadPathSample
        ↓
CacheInvalidationProbe
        ↓
DownstreamConsumerProbe
        ↓
RollbackReadinessReview
        ↓
StableProjectionCloseoutReceipt | ProjectionRollbackReceipt
```

这一步的核心不是监控面板有没有红，而是把真实读取路径抽样成证据，明确遇到 stale、mismatch、leak、rollback broken 时应该继续观察、回滚、重新隔离还是升级人工。

---

## 2. 闸门要检查什么？

Post-Release Projection Gate 要回答六个问题：

- **读路径是否一致**：API、dashboard、export、search index 返回的是同一个 projection version；
- **旧 release 是否回流**：缓存、异步 job、前端 storage 是否还引用 original release；
- **敏感 material 是否复现**：重建后是否有 residual sensitive payload；
- **消费者是否收敛**：关键 consumer 是否确认新值或撤回标记；
- **回滚是否可用**：如果发现问题，能否回到 quarantine 或 last known good；
- **观察窗口是否足够**：是否覆盖至少一个刷新周期、一个异步投影周期、一个 cache TTL。

工程上可以记成：

```text
release -> observe reads -> probe caches -> sample consumers -> verify rollback -> close or rollback
```

注意：这不是“多等一会”。观察窗口本身要有 lease、TTL、采样计划和关闭收据，否则它会变成永远挂着的待办。

---

## 3. learn-claude-code：放行后观察判定器

教学版用纯函数表达。输入放行收据、观察窗口采样、缓存探针、消费者探针和回滚复核，输出下一步动作。

```python
from dataclasses import dataclass
from typing import Literal

Decision = Literal[
    "close_stable",
    "continue_observation",
    "rollback_projection",
    "requarantine_projection",
    "open_release_violation",
]

@dataclass(frozen=True)
class ProjectionReleaseReceipt:
    projection_id: str
    projection_version: str
    release_receipt_id: str
    original_release_id: str
    replacement_release_id: str | None
    released_at_epoch: int

@dataclass(frozen=True)
class PostReleaseObservationLease:
    lease_id: str
    min_samples: int
    min_age_seconds: int
    observed_age_seconds: int
    critical_path_coverage: float

@dataclass(frozen=True)
class ReadPathSample:
    total_reads: int
    stale_release_reads: int
    version_mismatch_reads: int
    sensitive_material_hits: int
    unknown_reads: int

@dataclass(frozen=True)
class CacheInvalidationProbe:
    edge_cache_stale: bool
    app_cache_stale: bool
    search_index_stale: bool
    cache_purge_failed: bool

@dataclass(frozen=True)
class DownstreamConsumerProbe:
    sampled_consumers: int
    unacknowledged_consumers: int
    mismatch_consumers: int
    stale_export_consumers: int

@dataclass(frozen=True)
class RollbackReadinessReview:
    lkg_available: bool
    quarantine_path_available: bool
    rollback_probe_passed: bool
    rollback_would_reintroduce_sensitive_material: bool

@dataclass(frozen=True)
class ObservationDecision:
    decision: Decision
    reason: str
    required_actions: tuple[str, ...] = ()

def decide_post_release_projection(
    release: ProjectionReleaseReceipt,
    lease: PostReleaseObservationLease,
    reads: ReadPathSample,
    cache: CacheInvalidationProbe,
    consumers: DownstreamConsumerProbe,
    rollback: RollbackReadinessReview,
) -> ObservationDecision:
    if reads.sensitive_material_hits > 0:
        return ObservationDecision(
            "requarantine_projection",
            "sensitive material appeared after release",
            ("freeze_projection", "purge_projection_cache", "open_incident"),
        )

    if reads.stale_release_reads > 0 or consumers.stale_export_consumers > 0:
        if rollback.quarantine_path_available:
            return ObservationDecision(
                "requarantine_projection",
                "old release reference reappeared in read path",
                ("freeze_reads", "restore_quarantine_status", "rerun_rebuild_validation"),
            )
        return ObservationDecision(
            "open_release_violation",
            "old release returned but quarantine path is unavailable",
            ("block_exports", "manual_reconcile"),
        )

    cache_stale = cache.edge_cache_stale or cache.app_cache_stale or cache.search_index_stale
    if cache.cache_purge_failed:
        return ObservationDecision(
            "continue_observation",
            "cache purge failed; observation must continue after retry",
            ("retry_cache_purge", "increase_sampling"),
        )

    if cache_stale:
        return ObservationDecision(
            "continue_observation",
            "some cache layers still point at stale projection",
            ("invalidate_stale_cache_layers", "resample_read_paths"),
        )

    if reads.version_mismatch_reads > 0 or consumers.mismatch_consumers > 0:
        if (
            rollback.lkg_available
            and rollback.rollback_probe_passed
            and not rollback.rollback_would_reintroduce_sensitive_material
        ):
            return ObservationDecision(
                "rollback_projection",
                "released projection returns mismatched versions",
                ("rollback_to_lkg", "notify_consumers", "write_rollback_receipt"),
            )
        return ObservationDecision(
            "open_release_violation",
            "version mismatch found but rollback is not safe",
            ("block_downstream_reads", "manual_review"),
        )

    if consumers.unacknowledged_consumers > 0:
        return ObservationDecision(
            "continue_observation",
            "not all critical consumers have acknowledged released projection",
            ("retry_consumer_ack", "keep_observation_lease"),
        )

    enough_samples = reads.total_reads >= lease.min_samples
    enough_time = lease.observed_age_seconds >= lease.min_age_seconds
    enough_coverage = lease.critical_path_coverage >= 0.95
    unknown_rate = reads.unknown_reads / max(reads.total_reads, 1)

    if not enough_samples or not enough_time or not enough_coverage:
        return ObservationDecision(
            "continue_observation",
            "observation window has not met sample, age, or coverage requirements",
            ("collect_more_samples",),
        )

    if unknown_rate > 0.02:
        return ObservationDecision(
            "continue_observation",
            "unknown read outcome rate is above threshold",
            ("instrument_unknown_paths", "resample_unknown_reads"),
        )

    return ObservationDecision(
        "close_stable",
        f"projection {release.projection_id}@{release.projection_version} stayed stable after release",
        ("write_stable_projection_closeout_receipt",),
    )
```

最小测试：

```python
def test_close_stable_after_clean_observation():
    decision = decide_post_release_projection(
        ProjectionReleaseReceipt("dash1", "v2", "r1", "rel1", "rel2", 100),
        PostReleaseObservationLease("lease1", 100, 3600, 4000, 0.99),
        ReadPathSample(120, 0, 0, 0, 1),
        CacheInvalidationProbe(False, False, False, False),
        DownstreamConsumerProbe(5, 0, 0, 0),
        RollbackReadinessReview(True, True, True, False),
    )
    assert decision.decision == "close_stable"

def test_sensitive_material_requarantines_immediately():
    decision = decide_post_release_projection(
        ProjectionReleaseReceipt("report1", "v8", "r2", "rel_old", None, 200),
        PostReleaseObservationLease("lease2", 50, 1800, 120, 0.5),
        ReadPathSample(10, 0, 0, 1, 0),
        CacheInvalidationProbe(False, False, False, False),
        DownstreamConsumerProbe(1, 0, 0, 0),
        RollbackReadinessReview(True, True, True, False),
    )
    assert decision.decision == "requarantine_projection"

def test_version_mismatch_rolls_back_when_lkg_safe():
    decision = decide_post_release_projection(
        ProjectionReleaseReceipt("dash2", "v3", "r3", "rel1", "rel2", 300),
        PostReleaseObservationLease("lease3", 100, 3600, 1000, 0.7),
        ReadPathSample(40, 0, 2, 0, 0),
        CacheInvalidationProbe(False, False, False, False),
        DownstreamConsumerProbe(3, 0, 1, 0),
        RollbackReadinessReview(True, True, True, False),
    )
    assert decision.decision == "rollback_projection"
```

这个纯函数的价值在于：它把“上线后看看”变成可执行闸门。只要出现 sensitive material，立即重新隔离；出现 version mismatch，先看回滚是否安全；样本、时间、覆盖率不足，就不能关闭。

---

## 4. pi-mono：PostReleaseProjectionObserver Worker

生产版适合做成一个后台 Worker。它不直接信任 release job 的成功状态，而是从真实读路径取样：

```typescript
type ObservationDecision =
  | "close_stable"
  | "continue_observation"
  | "rollback_projection"
  | "requarantine_projection"
  | "open_release_violation";

interface ProjectionReleaseReceipt {
  projectionId: string;
  projectionVersion: string;
  releaseReceiptId: string;
  originalReleaseId: string;
  replacementReleaseId?: string;
}

interface ProjectionObserverDeps {
  releaseStore: ProjectionReleaseStore;
  readSampler: ProjectionReadSampler;
  cacheProbe: CacheProbe;
  consumerProbe: ConsumerProbe;
  rollbackProbe: RollbackProbe;
  projectionWriter: ProjectionWriter;
  receiptStore: ReceiptStore;
  outbox: Outbox;
}

export class PostReleaseProjectionObserver {
  constructor(private readonly deps: ProjectionObserverDeps) {}

  async observe(receiptId: string): Promise<ObservationDecision> {
    const release = await this.deps.releaseStore.getReleaseReceipt(receiptId);
    const lease = await this.deps.releaseStore.getObservationLease(receiptId);

    const [reads, cache, consumers, rollback] = await Promise.all([
      this.deps.readSampler.sample({
        projectionId: release.projectionId,
        projectionVersion: release.projectionVersion,
        minSamples: lease.minSamples,
      }),
      this.deps.cacheProbe.probe(release.projectionId),
      this.deps.consumerProbe.probe(release.projectionId),
      this.deps.rollbackProbe.review(release.projectionId),
    ]);

    const decision = decidePostReleaseProjection({
      release,
      lease,
      reads,
      cache,
      consumers,
      rollback,
    });

    await this.applyDecision(release, decision);
    return decision.decision;
  }

  private async applyDecision(
    release: ProjectionReleaseReceipt,
    decision: { decision: ObservationDecision; reason: string; requiredActions: string[] },
  ) {
    await this.deps.receiptStore.transaction(async (tx) => {
      await tx.append("ProjectionObservationDecision", {
        projectionId: release.projectionId,
        releaseReceiptId: release.releaseReceiptId,
        decision,
      });

      if (decision.decision === "close_stable") {
        await tx.append("StableProjectionCloseoutReceipt", {
          projectionId: release.projectionId,
          projectionVersion: release.projectionVersion,
          closedAt: new Date().toISOString(),
        });
        await this.deps.projectionWriter.markStable(release.projectionId, release.projectionVersion, tx);
      }

      if (decision.decision === "requarantine_projection") {
        await this.deps.projectionWriter.markQuarantined(release.projectionId, decision.reason, tx);
        await this.deps.outbox.enqueue(tx, {
          topic: "projection.requarantined",
          payload: { projectionId: release.projectionId, reason: decision.reason },
        });
      }

      if (decision.decision === "rollback_projection") {
        await this.deps.projectionWriter.rollbackToLastKnownGood(release.projectionId, tx);
        await tx.append("ProjectionRollbackReceipt", {
          projectionId: release.projectionId,
          releaseReceiptId: release.releaseReceiptId,
          reason: decision.reason,
        });
      }
    });
  }
}
```

这里有三个生产细节：

1. **readSampler 必须打真实读路径**，不要只读数据库状态；
2. **receiptStore.transaction 包住状态变更和收据**，避免状态改了但证据没写；
3. **requarantine 和 rollback 都走 outbox**，通知下游 consumer 重新对账。

---

## 5. OpenClaw 实战：Cron 课程发布也是投影观察

这套模式不只适用于 DP dashboard。我们这个课程 cron 每 3 小时也有类似链路：

```text
lesson file
README projection
TOOLS.md projection
Telegram message
git commit / remote branch
```

如果我写完课、更新 README、发群、git push，看起来任务完成了。但成熟的收口还要验证：

- README 是否真的包含新课程链接；
- TOOLS.md 是否记录了新知识点；
- Telegram 是否送达目标群；
- git remote head 是否包含本次 commit；
- 如果某一步失败，是否能重试而不重复发错消息。

对应到今天的课：

```text
ProjectionReleaseReceipt = README/TOOLS/message/git 都写出
ReadPathSample = grep README/TOOLS + git log + message receipt
CacheInvalidationProbe = remote head / local status
DownstreamConsumerProbe = Telegram group delivery result
RollbackReadinessReview = 如果 push 失败是否能 amend/retry，不重复发群
StableProjectionCloseoutReceipt = final cron summary
```

所以不要把“文件写了”当结束。只要有多个下游视图，就要把它们当投影，放行后再观察一轮。

---

## 6. 什么时候需要这套闸门？

适合使用：

- 隔离后重新放行 dashboard、report、cache、feature store；
- DP / privacy correction 后恢复消费者可见投影；
- search index、vector index、OLAP cube 的异步重建；
- 多渠道发布：README、网页、Telegram、邮件、API cache；
- 高风险修复后进入真实读路径。

不一定需要：

- 单机临时脚本、没有下游 consumer；
- 完全同步、无缓存、无副作用的本地计算；
- 已经有强事务保证且没有异步投影的内部状态。

判断标准很简单：**如果 release 后还有多个地方可能读到旧版本，就需要观察窗口。**

---

## 7. 常见坑

1. **只监控写路径，不采样读路径**

   写入成功不代表用户读到成功。读路径才是投影是否稳定的真相。

2. **观察窗口没有退出条件**

   `observe forever` 不是安全，是债务。必须定义 minSamples、minAge、coverage 和 closeout。

3. **发现 mismatch 直接覆盖修复**

   覆盖会破坏证据链。应该先写 ObservationDecision，再 rollback / requarantine / manual review。

4. **回滚路径未验证**

   发现问题时才测试 rollback，通常太晚。RollbackReadinessReview 应该是观察窗口的一部分。

5. **把 consumer ack 当成全部消费者收敛**

   ack 只是控制面确认，仍要抽样真实读路径，尤其是 export、cache、signed URL、search index。

---

## 8. 和前几课的关系

- **#522 Consumer Ack & Stale Projection Quarantine**：发现过期投影并隔离；
- **#523 Projection Rebuild & Release**：重建、重验并重新放行；
- **#524 Post-Release Observation & Rollback**：放行后证明真实读路径稳定；
- **#320 Post-Deploy Verification & Auto-Rollback**：同样思想在部署层的应用；
- **#503 Post-Cutover Stabilization**：同样思想在生产切流层的应用。

---

## 总结

投影重新放行不是终点，而是进入真实读路径的开始。

成熟 Agent 要把 `ProjectionReleaseReceipt` 后面的世界继续纳入证据链：用 `PostReleaseObservationLease` 采样读路径，用 `CacheInvalidationProbe` 找旧缓存，用 `DownstreamConsumerProbe` 检查消费者收敛，用 `RollbackReadinessReview` 确认问题可回退，最后写 `StableProjectionCloseoutReceipt` 或 `ProjectionRollbackReceipt`。

一句话：**放行只说明可以重新服务；观察关闭才说明它已经稳定服务。**

---

下一课可以继续讲：**Stable Projection Closeout 后的证据降温与采样审计**，也就是投影稳定后如何从热观察转成低成本长期审计。
