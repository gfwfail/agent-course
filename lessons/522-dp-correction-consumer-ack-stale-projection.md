# 522 - Agent 差分隐私更正后的消费者确认与过期投影隔离闸门

> 上一讲我们讲了 DP release 出问题后，如何用 investigation、预算、更正或撤回、consumer notice 和 closeout 把错误发布收敛成可审计终态。今天再往下走一步：通知发出去了，不代表所有下游都真的停止使用旧值。

上一讲讲的是 **DP Release Correction & Retraction Gate**。今天继续讲：**DP Correction Consumer Acknowledgement & Stale Projection Quarantine Gate**。

一句话：**DP release 更正后，系统不能只记录“通知已发送”；还要证明 dashboard、cache、report、API consumer 和派生投影都已确认、刷新或隔离。**

---

## 1. 为什么通知送达不是关闭？

很多系统会把更正流程停在这里：

```text
notice.delivered = true
release.status = superseded
```

这还不够。因为下游可能已经把旧 noisy value 复制到很多地方：

- dashboard materialized view；
- BI 报表快照；
- API cache；
- feature store；
- notebook export；
- 派生聚合表；
- webhook consumer 本地数据库。

如果只证明“通知发出”，没有证明“旧值不再被读”，系统仍然可能继续传播错误 DP release。

所以更正后的关闭链路应该是：

```text
CorrectionCloseoutReceipt
        ↓
ConsumerAcknowledgementLedger
        ↓
ProjectionFreshnessScan
        ↓
StaleProjectionQuarantineDecision
        ↓
ProjectionRefreshOrQuarantineReceipt
        ↓
ConsumerAckCloseoutReceipt
```

这一步的关键不是再算一次 DP，而是治理下游状态：谁收到了，谁确认了，谁仍在读旧值，哪些投影必须隔离。

---

## 2. 闸门要检查什么？

DP Correction Consumer Ack Gate 要回答五个问题：

- 所有 consumer 是否都能定位：dashboard、cache、API client、report、derived artifact；
- consumer 是否 ack 了原 release 的撤回或 replacement release；
- materialized projection 是否仍引用旧 release id；
- 未 ack consumer 是否超过 SLA；
- 旧投影是可以 refresh，还是必须 quarantine 或 purge。

工程上可以把它记成：

```text
notify sent -> consumer ack -> projection scan -> refresh/quarantine -> close
```

注意：ack 不是一句 “OK”。它必须绑定 `originalReleaseId`、`replacementReleaseId`、`consumerId`、`projectionId` 和 `observedAt`，否则无法审计。

---

## 3. learn-claude-code：消费者确认闸门纯函数

教学版先写成纯函数：输入更正收据、消费者确认、投影扫描结果，输出下一步动作。

```python
from dataclasses import dataclass
from typing import Literal

Decision = Literal[
    "close_all_consumers_converged",
    "refresh_stale_projections",
    "quarantine_stale_projections",
    "retry_consumer_notice",
    "open_consumer_ack_violation",
]

@dataclass(frozen=True)
class CorrectionCloseout:
    original_release_id: str
    replacement_release_id: str | None
    action: Literal["supersede", "retract", "purge_sensitive"]
    notice_required: bool

@dataclass(frozen=True)
class ConsumerAckSummary:
    expected_consumers: int
    acknowledged_consumers: int
    notice_failures: int
    ack_sla_expired: bool

@dataclass(frozen=True)
class ProjectionScan:
    total_projections: int
    stale_projection_count: int
    stale_sensitive_projection_count: int
    refreshable_stale_count: int

@dataclass(frozen=True)
class AckGateDecision:
    decision: Decision
    reason: str
    require_followup_scan: bool = False

def decide_consumer_ack_closeout(
    closeout: CorrectionCloseout,
    ack: ConsumerAckSummary,
    scan: ProjectionScan,
) -> AckGateDecision:
    if closeout.notice_required and ack.notice_failures > 0:
        return AckGateDecision(
            "retry_consumer_notice",
            "some consumers did not receive correction notice",
            require_followup_scan=True,
        )

    if closeout.notice_required and ack.ack_sla_expired:
        missing = ack.expected_consumers - ack.acknowledged_consumers
        if missing > 0:
            return AckGateDecision(
                "open_consumer_ack_violation",
                f"{missing} consumers did not acknowledge before SLA expiry",
                require_followup_scan=True,
            )

    if scan.stale_sensitive_projection_count > 0:
        return AckGateDecision(
            "quarantine_stale_projections",
            "stale projection still exposes sensitive or purged release material",
            require_followup_scan=True,
        )

    if scan.stale_projection_count > 0:
        if scan.refreshable_stale_count == scan.stale_projection_count:
            return AckGateDecision(
                "refresh_stale_projections",
                "all stale projections can be refreshed to replacement or retraction marker",
                require_followup_scan=True,
            )
        return AckGateDecision(
            "quarantine_stale_projections",
            "some stale projections cannot be safely refreshed",
            require_followup_scan=True,
        )

    if ack.acknowledged_consumers < ack.expected_consumers and closeout.notice_required:
        return AckGateDecision(
            "retry_consumer_notice",
            "waiting for remaining consumer acknowledgements",
            require_followup_scan=True,
        )

    return AckGateDecision(
        "close_all_consumers_converged",
        "all consumers acknowledged and no stale projection references the old release",
    )
```

最小测试：

```python
def test_refresh_when_all_stale_projections_are_refreshable():
    decision = decide_consumer_ack_closeout(
        CorrectionCloseout("rel1", "rel2", "supersede", True),
        ConsumerAckSummary(3, 3, 0, False),
        ProjectionScan(10, 2, 0, 2),
    )
    assert decision.decision == "refresh_stale_projections"

def test_sensitive_stale_projection_forces_quarantine():
    decision = decide_consumer_ack_closeout(
        CorrectionCloseout("rel1", None, "purge_sensitive", True),
        ConsumerAckSummary(5, 5, 0, False),
        ProjectionScan(8, 1, 1, 1),
    )
    assert decision.decision == "quarantine_stale_projections"

def test_all_converged_can_close():
    decision = decide_consumer_ack_closeout(
        CorrectionCloseout("rel1", "rel2", "supersede", True),
        ConsumerAckSummary(4, 4, 0, False),
        ProjectionScan(12, 0, 0, 0),
    )
    assert decision.decision == "close_all_consumers_converged"
```

这个纯函数强调一件事：consumer ack 和 projection freshness 是两条不同证据。人或系统确认收到了通知，不代表缓存、视图和导出已经刷新。

---

## 4. pi-mono：ConsumerAckCloseoutWorker

生产版要把 ack、扫描、刷新、隔离和关闭做成事务化 worker。

```ts
type CorrectionCloseout = {
  closeoutId: string;
  originalReleaseId: string;
  replacementReleaseId?: string;
  action: "supersede" | "retract" | "purge_sensitive";
};

type ProjectionRefreshPlan = {
  projectionId: string;
  action: "refresh_to_replacement" | "write_retraction_marker" | "quarantine";
  reason: string;
};

class ConsumerAckCloseoutWorker {
  constructor(
    private readonly closeouts: CorrectionCloseoutStore,
    private readonly consumers: ConsumerRegistry,
    private readonly ackLedger: ConsumerAckLedger,
    private readonly projections: ProjectionIndex,
    private readonly refresher: ProjectionRefreshService,
    private readonly quarantine: ProjectionQuarantineStore,
    private readonly receipts: ConsumerAckCloseoutStore,
    private readonly outbox: EventOutbox,
  ) {}

  async handle(input: { correctionCloseoutId: string }): Promise<ConsumerAckCloseout> {
    const closeout = await this.closeouts.get(input.correctionCloseoutId);
    const expected = await this.consumers.listForRelease(closeout.originalReleaseId);
    const ackSummary = await this.ackLedger.summarize({
      originalReleaseId: closeout.originalReleaseId,
      expectedConsumerIds: expected.map((c) => c.consumerId),
    });

    const stale = await this.projections.findReferences({
      releaseId: closeout.originalReleaseId,
      includeCaches: true,
      includeMaterializedViews: true,
      includeDerivedArtifacts: true,
    });

    const plans = stale.map((projection): ProjectionRefreshPlan => {
      if (projection.containsSensitiveMaterial || closeout.action === "purge_sensitive") {
        return {
          projectionId: projection.projectionId,
          action: "quarantine",
          reason: "stale projection contains purged or sensitive material",
        };
      }

      if (closeout.replacementReleaseId && projection.canRefresh) {
        return {
          projectionId: projection.projectionId,
          action: "refresh_to_replacement",
          reason: "replace old release reference with superseding release",
        };
      }

      if (projection.canRefresh) {
        return {
          projectionId: projection.projectionId,
          action: "write_retraction_marker",
          reason: "release was retracted without replacement",
        };
      }

      return {
        projectionId: projection.projectionId,
        action: "quarantine",
        reason: "projection cannot be refreshed deterministically",
      };
    });

    for (const plan of plans) {
      if (plan.action === "quarantine") {
        await this.quarantine.insertOnce({
          projectionId: plan.projectionId,
          releaseId: closeout.originalReleaseId,
          reason: plan.reason,
        });
      } else {
        await this.refresher.applyOnce({
          projectionId: plan.projectionId,
          originalReleaseId: closeout.originalReleaseId,
          replacementReleaseId: closeout.replacementReleaseId,
          action: plan.action,
        });
      }
    }

    const followupScan = await this.projections.findReferences({
      releaseId: closeout.originalReleaseId,
      includeCaches: true,
      includeMaterializedViews: true,
      includeDerivedArtifacts: true,
      excludeQuarantined: true,
    });

    const receipt = await this.receipts.insertOnce({
      correctionCloseoutId: closeout.closeoutId,
      expectedConsumers: expected.length,
      acknowledgedConsumers: ackSummary.acknowledged,
      staleBefore: stale.length,
      staleAfter: followupScan.length,
      quarantined: plans.filter((p) => p.action === "quarantine").length,
      closed: ackSummary.acknowledged === expected.length && followupScan.length === 0,
    });

    await this.outbox.publish("dp.consumer_ack_closeout_recorded", receipt);
    return receipt;
  }
}
```

几个生产细节：

- `applyOnce` 要用 `projectionId + originalReleaseId + action` 做幂等键；
- quarantine 不是删除，而是让投影从服务路径下线，并保留审计指针；
- follow-up scan 必须排除已隔离投影，否则会把已处理的旧引用反复报警；
- consumer ack 要绑定 release id，不能只按 consumer 维度写一个 “received”。

---

## 5. OpenClaw 实战类比

OpenClaw 课程 cron 里也有类似问题。假设 Git commit 已 push，Telegram 也补发了，但 README、TOOLS 和消息内容的某个链接仍指向旧 lesson：

```text
message delivered = true
```

这并不代表交付收敛。还要检查：

- README 是否引用最新 lesson；
- TOOLS 已讲列表是否更新；
- Telegram 消息是否包含新主题；
- Git remote head 是否包含本次 commit；
- 旧草稿或错误链接是否还在可见路径。

DP correction consumer ack 也是同一个模式：消息送达只是副作用成功，状态收敛要靠下游投影扫描和关闭收据证明。

---

## 6. 设计 Checklist

落地这个闸门时，可以直接检查：

- correction closeout 是否列出 expected consumers；
- consumer ack 是否绑定 original/replacement release id；
- ack 是否有 SLA，超时是否升级 violation；
- projection index 是否覆盖 cache、materialized view、report、derived artifact；
- stale projection 是否区分 refreshable 和 non-refreshable；
- sensitive stale projection 是否默认 quarantine；
- refresh 后是否做 follow-up scan；
- closeout 是否记录 staleBefore、staleAfter、quarantined、unacked consumers。

---

## 7. 今日要点

DP release 更正不是把通知发出去就结束。真正的终态是：所有消费者知道旧值不能用，所有投影不再服务旧 release，无法安全刷新的投影被隔离。

记住这个公式：

```text
Consumer Ack Closeout = correction notice + ack ledger + projection scan + refresh/quarantine + follow-up scan
```

成熟 Agent 不只会更正错误输出，还会证明错误输出已经从所有下游读取路径里收敛或隔离。
