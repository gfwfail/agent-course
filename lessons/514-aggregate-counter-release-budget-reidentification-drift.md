# 514 - Agent 聚合计数发布预算与重识别漂移闸门

> 上一讲把 hash-only audit stub 降级成 aggregate counter，避免单条删除证明长期可查。但聚合统计也不是天然安全：同一个 counter 如果被反复按不同维度导出、被 dashboard 缓存、被小桶量切片组合，仍然可能把单个 subject 重新拼出来。

今天讲 **Aggregate Counter Release Budget & Re-identification Drift Gate**：聚合计数可以保留，但每次发布、查询、导出都要扣预算、做小桶量检查、做重识别漂移探针，并留下 release receipt。

常见坑：

- 单条 stub 已删除，但 aggregate counter 支持 `tenant + day + region + outcome` 任意切片；
- dashboard 每天导出一个小差分，外部观察者可以用 yesterday/today 的差值反推出单个事件；
- 一次查询安全，多次查询组合后不安全；
- counter 的维度后来被新增，旧的 k-anonymity 评估没有重跑；
- 没有 release ledger，无法解释某个统计值已经被哪些消费者看过、还剩多少预算。

成熟 Agent 要把 aggregate counter 当成“仍有隐私边界的统计资产”，而不是“脱敏后随便用的数据”。

---

## 核心模型

```
AggregateCounterCloseoutReceipt
        ↓
CounterReleaseRequest
        ↓
ReleaseBudgetLedger
        ↓
ReidentificationDriftProbe
        ↓
CounterReleaseDecision
        ↓
AggregateCounterReleaseReceipt
```

关键判断：

- 当前 bucket 是否满足最小桶量，例如 `k >= 20`；
- 请求维度是否比 closeout 时更细；
- 同一 consumer 的累计查询是否耗尽 release budget；
- 时间窗口差分是否能反推出单条事件；
- dashboard/search/cache/export 是否存在未登记副本；
- 发布收据是否只记录 counter id、维度指纹、预算扣减和不可逆 proof hash，不写回 subject/receipt 级字段。

---

## learn-claude-code：发布闸门纯函数

教学版先把判断写成纯函数。输入聚合计数关闭收据、发布请求、预算账本和漂移探针，输出下一步动作。

```python
from dataclasses import dataclass
from typing import Literal

Decision = Literal[
    "release_counter",
    "coarsen_dimensions",
    "deny_budget_exhausted",
    "suppress_small_bucket",
    "block_differencing_attack",
    "purge_unregistered_projection",
    "open_reidentification_incident",
    "manual_review",
]

@dataclass
class AggregateCounterCloseoutReceipt:
    counter_id: str
    proof_hash: str
    allowed_dimensions: set[str]
    min_bucket_size: int
    release_budget: int
    schema_version: str

@dataclass
class CounterReleaseRequest:
    consumer_id: str
    requested_dimensions: set[str]
    bucket_size: int
    window_seconds: int
    export_requested: bool
    public_release: bool

@dataclass
class ReleaseBudgetLedger:
    used_budget: int
    same_consumer_recent_queries: int
    last_release_window_seconds: int | None
    unregistered_dashboard_copies: int
    unregistered_export_copies: int

@dataclass
class ReidentificationDriftProbe:
    can_diff_against_previous_release: bool
    newly_added_dimension: bool
    small_delta_bucket_detected: bool
    joins_with_external_keyspace: bool

def decide_counter_release(
    receipt: AggregateCounterCloseoutReceipt,
    request: CounterReleaseRequest,
    ledger: ReleaseBudgetLedger,
    probe: ReidentificationDriftProbe,
) -> Decision:
    if ledger.unregistered_dashboard_copies + ledger.unregistered_export_copies > 0:
        return "purge_unregistered_projection"

    if probe.joins_with_external_keyspace:
        return "open_reidentification_incident"

    if request.requested_dimensions - receipt.allowed_dimensions:
        return "coarsen_dimensions"

    if probe.newly_added_dimension:
        return "coarsen_dimensions"

    if request.bucket_size < receipt.min_bucket_size or probe.small_delta_bucket_detected:
        return "suppress_small_bucket"

    if ledger.used_budget >= receipt.release_budget:
        return "deny_budget_exhausted"

    if (
        probe.can_diff_against_previous_release
        and ledger.last_release_window_seconds is not None
        and request.window_seconds <= ledger.last_release_window_seconds
        and ledger.same_consumer_recent_queries >= 2
    ):
        return "block_differencing_attack"

    if request.public_release and request.export_requested and request.bucket_size < receipt.min_bucket_size * 3:
        return "manual_review"

    return "release_counter"
```

测试重点不是“能不能查”，而是“多次查会不会把小桶量拼回来”。

```python
def test_blocks_repeated_window_differencing():
    receipt = AggregateCounterCloseoutReceipt(
        counter_id="counter_delete_monthly",
        proof_hash="hash_abc",
        allowed_dimensions={"period", "request_class", "outcome"},
        min_bucket_size=20,
        release_budget=5,
        schema_version="v1",
    )
    request = CounterReleaseRequest(
        consumer_id="dashboard",
        requested_dimensions={"period", "request_class", "outcome"},
        bucket_size=42,
        window_seconds=86_400,
        export_requested=False,
        public_release=False,
    )
    ledger = ReleaseBudgetLedger(
        used_budget=2,
        same_consumer_recent_queries=3,
        last_release_window_seconds=86_400,
        unregistered_dashboard_copies=0,
        unregistered_export_copies=0,
    )
    probe = ReidentificationDriftProbe(
        can_diff_against_previous_release=True,
        newly_added_dimension=False,
        small_delta_bucket_detected=False,
        joins_with_external_keyspace=False,
    )

    assert decide_counter_release(receipt, request, ledger, probe) == "block_differencing_attack"
```

这里的设计原则是：**隐私风险不是单次查询的属性，而是查询历史、维度组合、外部 join 和时间差分共同产生的属性**。

---

## pi-mono：AggregateCounterReleaseWorker

生产版可以把它放在统计导出、dashboard 查询、合规报表生成之前。每次 release 都走同一个 worker，先拿预算锁，再做漂移探针，最后写 release receipt。

```ts
type CounterReleaseDecision =
  | "release_counter"
  | "coarsen_dimensions"
  | "deny_budget_exhausted"
  | "suppress_small_bucket"
  | "block_differencing_attack"
  | "purge_unregistered_projection"
  | "open_reidentification_incident"
  | "manual_review";

type CounterReleaseJob = {
  counterId: string;
  consumerId: string;
  requestedDimensions: string[];
  windowSeconds: number;
  exportRequested: boolean;
  publicRelease: boolean;
};

type AggregateCounterReleaseReceipt = {
  counterId: string;
  consumerId: string;
  decision: CounterReleaseDecision;
  releasedDimensions: string[];
  schemaVersion: string;
  budgetBefore: number;
  budgetAfter: number;
  dimensionFingerprint: string;
  proofHash: string;
  createdAt: string;
};

class AggregateCounterReleaseWorker {
  constructor(
    private readonly counters: CounterRepository,
    private readonly ledger: ReleaseBudgetLedgerStore,
    private readonly probes: ReidentificationProbeService,
    private readonly receipts: CounterReleaseReceiptStore,
    private readonly outbox: Outbox,
  ) {}

  async handle(job: CounterReleaseJob): Promise<AggregateCounterReleaseReceipt> {
    return this.ledger.withCounterLease(job.counterId, async () => {
      const counter = await this.counters.getCloseoutReceipt(job.counterId);
      const budget = await this.ledger.getBudget(job.counterId, job.consumerId);
      const probe = await this.probes.inspect({
        counterId: job.counterId,
        consumerId: job.consumerId,
        requestedDimensions: job.requestedDimensions,
        windowSeconds: job.windowSeconds,
      });

      const decision = decideCounterRelease(counter, job, budget, probe);
      const releasedDimensions =
        decision === "release_counter"
          ? job.requestedDimensions
          : coarsenDimensions(job.requestedDimensions, counter.allowedDimensions);

      const receipt: AggregateCounterReleaseReceipt = {
        counterId: job.counterId,
        consumerId: job.consumerId,
        decision,
        releasedDimensions,
        schemaVersion: counter.schemaVersion,
        budgetBefore: budget.remaining,
        budgetAfter: decision === "release_counter" ? budget.remaining - 1 : budget.remaining,
        dimensionFingerprint: fingerprintDimensions(releasedDimensions),
        proofHash: counter.proofHash,
        createdAt: new Date().toISOString(),
      };

      if (decision === "release_counter") {
        await this.ledger.consume(job.counterId, job.consumerId, 1);
        await this.outbox.publish("counter.release.approved", receipt);
      }

      if (decision === "open_reidentification_incident") {
        await this.outbox.publish("privacy.incident.opened", receipt);
      }

      await this.receipts.append(receipt);
      return receipt;
    });
  }
}
```

几个实现细节：

- `withCounterLease` 防止两个导出请求同时看到同一份预算；
- `dimensionFingerprint` 记录维度形状，不记录原始 subject；
- `probe.inspect` 要检查历史 release、dashboard projection、缓存副本和外部 join 风险；
- 被拒绝的请求也要写 receipt，因为拒绝本身是合规证据。

---

## OpenClaw：课程 Cron 的类比

这个课程自动化每 3 小时发一次 Agent 课，也有类似风险：单次 lesson 看起来没重复，但如果不看 README 和 TOOLS 的历史列表，就可能在长期运行中反复讲同一类主题。

对应关系：

- `AggregateCounterCloseoutReceipt`：上一讲把单条 hash stub 降级成统计资产；
- `CounterReleaseRequest`：本次要把某个统计口径发给 dashboard、审计员或公开报表；
- `ReleaseBudgetLedger`：记录某个 consumer 已经看过多少次、还能看多少次；
- `ReidentificationDriftProbe`：检查维度变细、时间窗口差分、外部 join、小桶量；
- `AggregateCounterReleaseReceipt`：像本次 lesson 的 commit + README + TOOLS 更新，证明“发了什么、为什么能发、如何避免重复/越界”。

简单说：**去重不是只看标题，隐私也不是只删原文。长期自动化系统要记录历史、扣预算、检查组合风险，再决定是否继续发布。**

---

## 落地建议

1. 所有 aggregate counter 查询都经过同一个 release gate，禁止 dashboard 直连底表。
2. 给每个 consumer 维护 release budget，而不是只给 counter 设全局 TTL。
3. 维度新增、schema 变更、bucket size 下降时，强制重跑 reidentification probe。
4. 对时间窗口查询做 differencing 检查，避免通过相邻窗口差值反推出单条事件。
5. 拒绝、降维、抑制小桶量、批准发布都写 release receipt。

今天的要点：**聚合计数不是终点，而是进入了另一个治理阶段。成熟 Agent 不只证明单条证据已删除，还要证明剩下的统计结果不会在未来被组合回个人轨迹。**
