# 513 - Agent 哈希存根到期降级与聚合计数器关闭闸门

> Proof-of-forgetting 证明“原始数据已经被忘记”，但 hash-only audit stub 也不能永久留在热审计路径里。成熟 Agent 要在 stub 到期后，把可关联的单条证明降级成不可反查、不可串联、仍可统计的 aggregate counter，并留下关闭收据。

上一讲讲了 Post-Erasure Hash Stub Verification：删除后只保留不可逆、带 pepper、可过期的 hash-only audit stub，用 challenge 证明 raw/object/search/vector/cache/workspace 都无残留。

今天继续补收尾动作：**stub 到期不是简单 delete**。有些系统还需要保留合规统计，例如“本季度完成了多少次 deletion request”“多少次因 legal hold 延期”“多少次 proof challenge 通过”。这些统计有价值，但不能保留能反推出某个 subject 的单条轨迹。

常见坑：

- hash stub 过期后仍按 subjectId / tenantId / receiptId 可查；
- 把单条 stub 删了，但 audit dashboard 还缓存了 subject 维度；
- aggregate counter 太细，组合 tenant + day + request type 后仍可重识别；
- 没有记录“从单条 stub 降级到聚合计数”的收据；
- 删除 stub 后，未来审计只能相信口头说明，无法证明当时做过降级和反向探针。

所以需要一个 **Hash Stub Expiry, Aggregate Counter & Closeout Gate**：到期时先验证 stub 没有残留依赖，再把它压缩成低基数统计，最后删除单条索引并写 closeout receipt。

---

## 核心模型

```
HashOnlyAuditStub
        ↓
StubExpiryReview
        ↓
AggregateCounterPlan
        ↓
CounterAggregationReceipt
        ↓
SingleStubPurgeProbe
        ↓
HashStubExpiryCloseoutReceipt
```

关键判断：

- stub 是否已经过 retention deadline 或 max retention age；
- 是否存在 legal hold、open dispute、active proof challenge；
- aggregate 维度是否足够粗，避免单 subject / 单 receipt 反查；
- counter 是否只保留 period、class、outcome、risk tier 等低基数字段；
- 单条 stub、搜索索引、缓存、dashboard projection 是否都已清理；
- closeout receipt 是否只引用 irreversible proof hash，而不重新引入可关联字段。

---

## learn-claude-code：到期降级闸门纯函数

教学版可以把它写成一个纯函数：输入 stub、到期复核和聚合策略，输出下一步动作。

```python
from dataclasses import dataclass
from typing import Literal

Decision = Literal[
    "aggregate_and_purge_stub",
    "keep_stub_under_hold",
    "coarsen_aggregate_plan",
    "purge_dashboard_projection",
    "rerun_single_stub_purge_probe",
    "delete_without_aggregate",
    "open_expiry_violation_case",
    "manual_review",
]

@dataclass
class HashOnlyAuditStub:
    stub_id: str
    proof_hash: str
    expires_at_epoch: int
    created_at_epoch: int
    has_plain_identifier: bool
    has_receipt_lookup_index: bool
    open_legal_hold: bool
    open_dispute: bool
    active_challenge: bool

@dataclass
class StubExpiryReview:
    now_epoch: int
    max_retention_seconds: int
    dashboard_projection_hits: int
    search_index_hits: int
    cache_hits: int
    can_lookup_by_subject: bool
    can_lookup_by_receipt: bool

@dataclass
class AggregateCounterPlan:
    enabled: bool
    period_seconds: int
    dimensions: set[str]
    min_bucket_size: int
    includes_subject_dimension: bool
    includes_receipt_dimension: bool
    includes_tenant_raw_id: bool

def decide_hash_stub_expiry(
    stub: HashOnlyAuditStub,
    review: StubExpiryReview,
    plan: AggregateCounterPlan,
) -> Decision:
    if stub.has_plain_identifier:
        return "open_expiry_violation_case"

    if stub.open_legal_hold or stub.open_dispute or stub.active_challenge:
        return "keep_stub_under_hold"

    expired_by_deadline = review.now_epoch >= stub.expires_at_epoch
    expired_by_age = review.now_epoch - stub.created_at_epoch > review.max_retention_seconds
    if not expired_by_deadline and not expired_by_age:
        return "manual_review"

    if review.dashboard_projection_hits > 0:
        return "purge_dashboard_projection"

    if review.search_index_hits + review.cache_hits > 0:
        return "rerun_single_stub_purge_probe"

    if review.can_lookup_by_subject:
        return "open_expiry_violation_case"

    if not plan.enabled:
        return "delete_without_aggregate"

    unsafe_dimensions = (
        plan.includes_subject_dimension
        or plan.includes_receipt_dimension
        or plan.includes_tenant_raw_id
        or plan.min_bucket_size < 20
        or plan.period_seconds < 86_400
    )
    if unsafe_dimensions:
        return "coarsen_aggregate_plan"

    if stub.has_receipt_lookup_index or review.can_lookup_by_receipt:
        return "rerun_single_stub_purge_probe"

    return "aggregate_and_purge_stub"
```

一个关键测试：如果聚合维度太细，即使没有 raw data，也不能关闭。

```python
def test_rejects_too_fine_aggregate_bucket():
    stub = HashOnlyAuditStub(
        stub_id="stub_1",
        proof_hash="proof_hash",
        expires_at_epoch=1_000,
        created_at_epoch=100,
        has_plain_identifier=False,
        has_receipt_lookup_index=False,
        open_legal_hold=False,
        open_dispute=False,
        active_challenge=False,
    )
    review = StubExpiryReview(
        now_epoch=2_000,
        max_retention_seconds=1_000,
        dashboard_projection_hits=0,
        search_index_hits=0,
        cache_hits=0,
        can_lookup_by_subject=False,
        can_lookup_by_receipt=False,
    )
    plan = AggregateCounterPlan(
        enabled=True,
        period_seconds=3_600,
        dimensions={"tenant", "day", "request_type"},
        min_bucket_size=3,
        includes_subject_dimension=False,
        includes_receipt_dimension=False,
        includes_tenant_raw_id=False,
    )

    assert decide_hash_stub_expiry(stub, review, plan) == "coarsen_aggregate_plan"
```

这里的重点是：**聚合不是把敏感字段换个表名保存，而是让单条删除事件离开可查询世界**。

---

## pi-mono：HashStubExpiryWorker

生产版可以做成周期 worker：扫描到期 stub，生成聚合计划，执行计数写入，然后删除单条索引并做反向探针。

```ts
type ExpiryDecision =
  | "aggregate_and_purge_stub"
  | "keep_stub_under_hold"
  | "coarsen_aggregate_plan"
  | "purge_dashboard_projection"
  | "rerun_single_stub_purge_probe"
  | "delete_without_aggregate"
  | "open_expiry_violation_case"
  | "manual_review";

type HashStubExpiryJob = {
  stubId: string;
  proofHash: string;
  expiresAt: number;
  createdAt: number;
  openLegalHold: boolean;
  openDispute: boolean;
  activeChallenge: boolean;
};

type AggregateCounterPlan = {
  period: "day" | "week" | "month";
  dimensions: Array<"requestClass" | "outcome" | "riskTier" | "region">;
  minBucketSize: number;
};

type HashStubExpiryCloseoutReceipt = {
  stubId: string;
  proofHash: string;
  decision: ExpiryDecision;
  aggregateCounterId?: string;
  purgedLookupIndexes: string[];
  reverseProbePassed: boolean;
  closedAt: number;
};

async function runHashStubExpiryWorker(job: HashStubExpiryJob) {
  const review = await auditStore.reviewStubExpiry(job.stubId);
  const plan = await counterPlanner.planFor(job.stubId);
  const decision = decideHashStubExpiry(job, review, plan);

  if (decision === "aggregate_and_purge_stub") {
    const counter = await aggregateCounters.increment({
      period: plan.period,
      dimensions: plan.dimensions,
      minBucketSize: plan.minBucketSize,
      sourceProofHash: job.proofHash,
    });

    await auditStore.purgeSingleStubIndexes(job.stubId, {
      subjectLookup: true,
      receiptLookup: true,
      searchProjection: true,
      dashboardProjection: true,
    });

    const probe = await auditStore.reverseProbe(job.stubId);
    return writeCloseoutReceipt({
      stubId: job.stubId,
      proofHash: job.proofHash,
      decision,
      aggregateCounterId: counter.counterId,
      purgedLookupIndexes: probe.checkedIndexes,
      reverseProbePassed: probe.hits === 0,
      closedAt: Date.now(),
    });
  }

  if (decision === "keep_stub_under_hold") {
    return holdLedger.extend(job.stubId, "expiry_blocked_by_hold_or_challenge");
  }

  if (decision === "coarsen_aggregate_plan") {
    return reviewQueue.enqueue("aggregate_plan_too_specific", { stubId: job.stubId, plan });
  }

  return remediationQueue.enqueue(decision, { stubId: job.stubId, review });
}
```

注意两个边界：

- counter 写入用 `sourceProofHash` 做幂等，不提供反查 API；
- closeout receipt 保留 `proofHash` 和 `aggregateCounterId`，但不能保留 subject、receipt raw id、原始 digest input。

---

## OpenClaw 实战类比

我们的课程 Cron 也有类似需求：

- lesson 文件、README、TOOLS 是热路径证据；
- Telegram messageId、git commit 是单次发布证据；
- 长期看更有价值的是“第几课、主题族、是否送达、是否提交成功”的聚合状态；
- 如果未来要清理旧 delivery raw log，应该先确认 README/TOOLS/commit 还能证明课程存在，再把单次 raw 发送细节降级成不可反查统计。

也就是说，Agent 的审计系统应该分三层：

- **热证据**：单次事件可查，用于刚发生的纠错和对账；
- **冷证据**：按租约取回，用于合规和复盘；
- **聚合证据**：只回答趋势、数量、SLO，不允许反查个人或单条事件。

---

## 设计原则

1. **先证明无依赖，再降级**

   到期前先扫 dashboard、search、cache、receipt lookup。还有下游依赖就不能直接删。

2. **聚合维度要有最小桶**

   `tenant + day + rare request type` 很容易重新识别。高风险统计要设置 `minBucketSize`、更粗 period 或更少 dimensions。

3. **关闭收据不能复活敏感字段**

   closeout receipt 是证明，不是备份。只保留 proof hash、counter id、probe summary 和 policy version。

4. **删除不是失去审计能力**

   单条 stub 删除后，系统仍能回答“这个时期有多少删除请求完成、多少被 hold、多少挑战通过”，但不能回答“某个 subject 的那条记录在哪里”。

---

## 小结

成熟 Agent 的可证明遗忘，不是把 raw data 删了、hash stub 永久留着。

真正成熟的做法是：

- raw data 删除后，用 hash-only stub 做短期证明；
- stub 到期后，先做反向探针和依赖扫描；
- 需要统计时降级成低基数 aggregate counter；
- 删除单条 lookup/index/projection；
- 留下不可反查的 closeout receipt。

一句话：**遗忘不是审计能力的终点，而是把单条可关联证据安全降级成不可反查统计的过程。**
