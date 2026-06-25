# 523 - Agent 隔离投影的重建、重验与重新放行闸门

> 上一讲我们讲了 DP release 更正后，如何让 consumer ack、projection scan、refresh 和 quarantine 形成关闭收据。今天继续往后走：投影被隔离后，不能永远躺在隔离区，也不能修一下就直接重新上线。

上一讲讲的是 **DP Correction Consumer Acknowledgement & Stale Projection Quarantine Gate**。今天继续讲：**Quarantined Projection Rebuild, Revalidation & Release Gate**。

一句话：**被隔离的 dashboard、cache、report、derived artifact 要重新进入服务路径，必须先证明它已经用正确 release 重建、重新验证、通过消费者抽样确认，并写入 release receipt。**

---

## 1. 为什么隔离不是终点？

很多系统做完 quarantine 后就停了：

```text
projection.status = quarantined
reason = stale release reference
```

这能止血，但还没有恢复服务。更危险的是，有人为了恢复 dashboard，会手动把状态改回 active：

```text
projection.status = active
```

这等于绕过了上一讲的证据链。隔离后的正确流程应该是：

```text
ProjectionQuarantineReceipt
        ↓
ProjectionRebuildPlan
        ↓
RebuildExecutionReceipt
        ↓
ProjectionRevalidationReport
        ↓
ConsumerSampleConfirmation
        ↓
ProjectionReleaseReceipt
```

这一步的关键不是“把缓存刷一下”，而是证明重建后的投影不再引用旧 release、不泄露被 purge 的 material、消费者看到的是 replacement 或 retraction marker。

---

## 2. 闸门要检查什么？

Projection Rebuild Gate 要回答六个问题：

- 隔离原因是否已经关闭：旧 release reference、敏感 material、不可刷新状态；
- 重建源是否可信：replacement release、retraction marker、或重新跑过的 bounded aggregate；
- 重建过程是否幂等：同一个 `projectionId + quarantineId` 不能重复写出不同结果；
- validation 是否覆盖 raw reference scan、schema hash、redaction profile 和 DP release lineage；
- consumer sample 是否看到新值或撤回标记；
- release 后是否还需要短观察窗口，防止旧 cache 回流。

工程上可以记成：

```text
quarantine -> rebuild plan -> isolated rebuild -> validate -> sample confirm -> release
```

注意：release 不是删除 quarantine 记录，而是把 quarantine 记录关闭，并保留 `releasedByReceiptId`。以后审计时必须能从 active projection 反查它为什么曾经被隔离、如何重建、谁确认了重新放行。

---

## 3. learn-claude-code：重建放行闸门纯函数

教学版先写成纯函数：输入隔离收据、重建结果、验证报告和消费者抽样，输出下一步动作。

```python
from dataclasses import dataclass
from typing import Literal

Decision = Literal[
    "release_projection",
    "rerun_rebuild",
    "keep_quarantined",
    "purge_projection",
    "open_rebuild_violation",
]

@dataclass(frozen=True)
class ProjectionQuarantineReceipt:
    projection_id: str
    quarantine_id: str
    reason: Literal["stale_release", "sensitive_material", "non_refreshable"]
    original_release_id: str
    replacement_release_id: str | None

@dataclass(frozen=True)
class RebuildExecutionReceipt:
    rebuilt: bool
    source_release_id: str | None
    wrote_retraction_marker: bool
    idempotency_conflict: bool

@dataclass(frozen=True)
class ProjectionRevalidationReport:
    references_original_release: bool
    contains_sensitive_material: bool
    schema_hash_matches: bool
    redaction_profile_matches: bool
    lineage_matches_replacement: bool

@dataclass(frozen=True)
class ConsumerSampleConfirmation:
    sampled_consumers: int
    mismatched_consumers: int
    stale_cache_hits: int

@dataclass(frozen=True)
class ReleaseDecision:
    decision: Decision
    reason: str
    require_observation: bool = False

def decide_projection_release(
    quarantine: ProjectionQuarantineReceipt,
    rebuild: RebuildExecutionReceipt,
    validation: ProjectionRevalidationReport,
    sample: ConsumerSampleConfirmation,
) -> ReleaseDecision:
    if rebuild.idempotency_conflict:
        return ReleaseDecision(
            "open_rebuild_violation",
            "same quarantine rebuild produced conflicting outputs",
            require_observation=True,
        )

    if not rebuild.rebuilt:
        if quarantine.reason == "sensitive_material":
            return ReleaseDecision(
                "purge_projection",
                "sensitive projection cannot be rebuilt safely",
            )
        return ReleaseDecision("rerun_rebuild", "projection rebuild did not complete")

    if validation.references_original_release:
        return ReleaseDecision(
            "keep_quarantined",
            "rebuilt projection still references the original release",
            require_observation=True,
        )

    if validation.contains_sensitive_material:
        return ReleaseDecision(
            "purge_projection",
            "rebuilt projection still contains purged or sensitive material",
        )

    if not validation.schema_hash_matches or not validation.redaction_profile_matches:
        return ReleaseDecision(
            "rerun_rebuild",
            "projection schema or redaction profile drifted during rebuild",
        )

    if quarantine.replacement_release_id:
        if rebuild.source_release_id != quarantine.replacement_release_id:
            return ReleaseDecision(
                "keep_quarantined",
                "rebuild did not use the expected replacement release",
            )
        if not validation.lineage_matches_replacement:
            return ReleaseDecision(
                "keep_quarantined",
                "validation lineage does not match replacement release",
            )
    elif not rebuild.wrote_retraction_marker:
        return ReleaseDecision(
            "keep_quarantined",
            "retracted release requires a visible retraction marker",
        )

    if sample.mismatched_consumers > 0 or sample.stale_cache_hits > 0:
        return ReleaseDecision(
            "keep_quarantined",
            "consumer sampling still observes stale or mismatched values",
            require_observation=True,
        )

    return ReleaseDecision(
        "release_projection",
        "projection rebuilt, validated, sampled, and no longer references the old release",
        require_observation=True,
    )
```

最小测试：

```python
def test_release_after_clean_rebuild_to_replacement():
    decision = decide_projection_release(
        ProjectionQuarantineReceipt("dash1", "q1", "stale_release", "rel1", "rel2"),
        RebuildExecutionReceipt(True, "rel2", False, False),
        ProjectionRevalidationReport(False, False, True, True, True),
        ConsumerSampleConfirmation(3, 0, 0),
    )
    assert decision.decision == "release_projection"

def test_old_release_reference_keeps_projection_quarantined():
    decision = decide_projection_release(
        ProjectionQuarantineReceipt("dash1", "q1", "stale_release", "rel1", "rel2"),
        RebuildExecutionReceipt(True, "rel2", False, False),
        ProjectionRevalidationReport(True, False, True, True, True),
        ConsumerSampleConfirmation(2, 0, 0),
    )
    assert decision.decision == "keep_quarantined"

def test_sensitive_material_after_rebuild_forces_purge():
    decision = decide_projection_release(
        ProjectionQuarantineReceipt("report1", "q2", "sensitive_material", "rel1", None),
        RebuildExecutionReceipt(True, None, True, False),
        ProjectionRevalidationReport(False, True, True, True, False),
        ConsumerSampleConfirmation(1, 0, 0),
    )
    assert decision.decision == "purge_projection"
```

这个纯函数强调一件事：重新放行必须同时满足 rebuild、validation、consumer sample 三类证据，不能只看 job 成功。

---

## 4. pi-mono：ProjectionReleaseWorker

生产版要把隔离投影当成一个有生命周期的实体：隔离、重建、验证、抽样、放行或清理。

```ts
type QuarantinedProjection = {
  quarantineId: string;
  projectionId: string;
  originalReleaseId: string;
  replacementReleaseId?: string;
  reason: "stale_release" | "sensitive_material" | "non_refreshable";
};

type ProjectionReleaseReceipt = {
  quarantineId: string;
  projectionId: string;
  decision: "released" | "kept_quarantined" | "purged";
  rebuildReceiptId: string;
  validationReportId: string;
  consumerSampleId: string;
};

class ProjectionReleaseWorker {
  constructor(
    private readonly quarantine: ProjectionQuarantineStore,
    private readonly rebuilder: IsolatedProjectionRebuilder,
    private readonly validator: ProjectionValidator,
    private readonly sampler: ConsumerProjectionSampler,
    private readonly activeIndex: ActiveProjectionIndex,
    private readonly receipts: ProjectionReleaseReceiptStore,
    private readonly outbox: EventOutbox,
  ) {}

  async handle(input: { quarantineId: string }): Promise<ProjectionReleaseReceipt> {
    const q = await this.quarantine.getOpen(input.quarantineId);

    const rebuild = await this.rebuilder.rebuildOnce({
      idempotencyKey: `${q.quarantineId}:${q.projectionId}`,
      projectionId: q.projectionId,
      originalReleaseId: q.originalReleaseId,
      replacementReleaseId: q.replacementReleaseId,
      mode: q.replacementReleaseId ? "replacement" : "retraction_marker",
      isolated: true,
    });

    const validation = await this.validator.validate({
      projectionId: q.projectionId,
      rebuiltArtifactId: rebuild.artifactId,
      forbiddenReleaseId: q.originalReleaseId,
      expectedReleaseId: q.replacementReleaseId,
      requireRedactionProfile: true,
      requireSchemaHash: true,
    });

    const sample = await this.sampler.sampleConsumers({
      projectionId: q.projectionId,
      artifactId: rebuild.artifactId,
      expectedReleaseId: q.replacementReleaseId,
      expectedMarker: q.replacementReleaseId ? undefined : "retracted",
    });

    if (validation.containsSensitiveMaterial) {
      await this.activeIndex.removeIfPresent(q.projectionId);
      await this.quarantine.closeAsPurged(q.quarantineId, validation.reportId);
      const receipt = await this.receipts.insertOnce({
        quarantineId: q.quarantineId,
        projectionId: q.projectionId,
        decision: "purged",
        rebuildReceiptId: rebuild.receiptId,
        validationReportId: validation.reportId,
        consumerSampleId: sample.sampleId,
      });
      await this.outbox.publish("projection.release.purged", receipt);
      return receipt;
    }

    if (
      validation.referencesForbiddenRelease ||
      !validation.schemaHashMatches ||
      !validation.redactionProfileMatches ||
      sample.staleCacheHits > 0 ||
      sample.mismatchedConsumers > 0
    ) {
      const receipt = await this.receipts.insertOnce({
        quarantineId: q.quarantineId,
        projectionId: q.projectionId,
        decision: "kept_quarantined",
        rebuildReceiptId: rebuild.receiptId,
        validationReportId: validation.reportId,
        consumerSampleId: sample.sampleId,
      });
      await this.outbox.publish("projection.release.blocked", receipt);
      return receipt;
    }

    await this.activeIndex.swapArtifact({
      projectionId: q.projectionId,
      artifactId: rebuild.artifactId,
      expectedCurrentStatus: "quarantined",
      releaseReceiptRef: rebuild.receiptId,
    });

    const receipt = await this.receipts.insertOnce({
      quarantineId: q.quarantineId,
      projectionId: q.projectionId,
      decision: "released",
      rebuildReceiptId: rebuild.receiptId,
      validationReportId: validation.reportId,
      consumerSampleId: sample.sampleId,
    });

    await this.quarantine.closeAsReleased(q.quarantineId, receipt);
    await this.outbox.publish("projection.release.released", receipt);
    return receipt;
  }
}
```

几个生产细节：

- rebuild 必须在 isolated artifact 里完成，验证通过后再 `swapArtifact`；
- `swapArtifact` 要用 CAS，避免人工或旧 worker 在 quarantine 期间抢先恢复；
- validation 要扫描 forbidden release id，而不是只看新 artifact 是否存在；
- consumer sample 要打真实读取路径，不能只读构建产物；
- release 后保留 quarantine receipt，方便审计和后续观察窗口定位来源。

---

## 5. OpenClaw 实战类比

OpenClaw 课程 cron 也有同样的模式。假设上一轮课程消息发错链接，先把错误消息视为 quarantined projection：

```text
wrong Telegram lesson link -> quarantine / stop reusing
```

恢复时不能只重新发一条消息，还要验证：

- lesson 文件存在；
- README 已指向新文件；
- TOOLS 已讲内容包含新主题；
- git remote head 包含 commit；
- Telegram 群里看到的是新主题和正确链接；
- 旧错误链接不再作为最新入口。

这就是 projection rebuild：先隔离错误入口，再从正确源重建入口，最后通过真实消费者路径确认后重新放行。

---

## 6. 设计 Checklist

落地这个闸门时，可以直接检查：

- quarantine receipt 是否包含 `originalReleaseId` 和隔离原因；
- rebuild plan 是否指定 replacement release 或 retraction marker；
- rebuild 是否有 `projectionId + quarantineId` 幂等键；
- rebuilt artifact 是否先进入隔离区，而不是直接覆盖 active；
- validation 是否扫描 forbidden release id；
- validation 是否校验 schema hash、redaction profile、lineage；
- consumer sample 是否走真实读路径；
- release 是否用 CAS swap；
- quarantine 是否以 released / purged / kept 状态关闭；
- release 后是否创建短观察窗口。

---

## 7. 今日要点

隔离投影只是止血，重建放行才是恢复。成熟 Agent 不会把 quarantine 当成垃圾桶，也不会把 rebuild success 当成可上线证明。

记住这个公式：

```text
Projection Release = quarantine receipt + isolated rebuild + forbidden-reference validation + consumer sample + CAS release receipt
```

成熟 Agent 不只会把错误投影隔离起来，还会证明它何时、为何、如何被安全重建并重新进入服务路径。
