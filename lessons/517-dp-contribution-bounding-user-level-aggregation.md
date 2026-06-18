# 517 - Agent 差分隐私贡献边界与用户级聚合闸门

> DP 预算和噪声账本只能保护“发布动作”，但如果原始聚合前没有限制单个用户的贡献，一个高活跃用户仍然可能把统计值拉出明显形状。成熟 Agent 做 DP 发布前，要先做 contribution bounding：限制每个用户、设备、会话或租户在一个统计窗口里的最大贡献，再进入 user-level aggregation 和加噪。

上一讲讲了 DP 预算续期：epsilon 用完后怎么复用、续租、降精度或拒绝发布。今天补上 DP 管道里更靠前、也更容易漏掉的一层：**Contribution Bounding & User-Level Aggregation Gate**。

一句话：**不要直接给 event count 加噪，先把 event 压成有边界的 user contribution**。

---

## 1. 为什么只加噪不够？

很多 Agent 指标都是事件级别的：

- 某个工具一天调用了多少次；
- 某类错误在多少会话中出现；
- 某个推荐被点击多少次；
- 某个自动化任务生成了多少输出。

如果直接对 event count 加噪，会有一个问题：少数用户可能贡献大量事件。

例子：

```text
user_a: 1 次 tool_call
user_b: 2 次 tool_call
user_c: 640 次 tool_call
```

即使用 DP 噪声保护总数，`user_c` 的行为仍然主导统计。一旦攻击者知道这个用户是否在窗口里，就可能通过差分查询推断他的活动。

所以发布前要先定义：

- 统计窗口：`day`、`week`、`release_window`；
- privacy unit：通常是 `user_id`，有时是 `tenant_id` 或 `device_id`；
- contribution cap：每个 privacy unit 最多贡献多少；
- clipping 策略：超过 cap 是截断、采样，还是分桶粗化；
- receipt：证明这次发布用的 cap、窗口和 hash。

---

## 2. 管道模型

```
RawEventBatch
        ↓
ContributionBoundaryPolicy
        ↓
UserContributionSummary
        ↓
BoundedAggregate
        ↓
DifferentialPrivacyReleaseReceipt
        ↓
ContributionBoundingReceipt
```

闸门要回答几个问题：

- 是否存在未绑定 privacy unit 的事件；
- 单个用户贡献是否超过 cap；
- cap 是否和发布目的匹配，不能发布前临时调大；
- 被裁剪的贡献比例是否异常高；
- 是否有小群体 bucket 需要 suppress；
- DP release receipt 是否绑定了 contribution receipt。

---

## 3. learn-claude-code：贡献边界纯函数

教学版先写纯函数，把原始事件压成 bounded aggregate。重点不是 DP 数学，而是把“谁贡献了多少、裁剪了多少”变成可测试的状态。

```python
from dataclasses import dataclass
from typing import Literal

Decision = Literal[
    "release_bounded_aggregate",
    "suppress_small_bucket",
    "lower_contribution_cap",
    "reject_unbounded_events",
    "manual_review_high_clipping",
]

@dataclass(frozen=True)
class RawEvent:
    user_id: str | None
    bucket: str
    value: int

@dataclass(frozen=True)
class ContributionPolicy:
    max_per_user_per_bucket: int
    min_users_per_bucket: int
    max_clipped_ratio: float

@dataclass(frozen=True)
class BoundedAggregate:
    bucket: str
    bounded_total: int
    distinct_users: int
    clipped_events: int
    raw_total: int

@dataclass(frozen=True)
class BoundaryDecision:
    decision: Decision
    aggregate: BoundedAggregate | None
    reason: str

def bound_contributions(
    events: list[RawEvent],
    policy: ContributionPolicy,
    bucket: str,
) -> BoundaryDecision:
    if any(event.user_id is None for event in events):
        return BoundaryDecision(
            "reject_unbounded_events",
            None,
            "event without privacy unit cannot enter DP release",
        )

    per_user: dict[str, int] = {}
    raw_total = 0
    for event in events:
        if event.bucket != bucket:
            continue
        raw_total += event.value
        per_user[event.user_id or ""] = per_user.get(event.user_id or "", 0) + event.value

    if len(per_user) < policy.min_users_per_bucket:
        return BoundaryDecision(
            "suppress_small_bucket",
            None,
            "bucket does not meet minimum user threshold",
        )

    bounded_total = 0
    clipped_events = 0
    for value in per_user.values():
        clipped = min(value, policy.max_per_user_per_bucket)
        bounded_total += clipped
        clipped_events += max(0, value - clipped)

    clipped_ratio = clipped_events / raw_total if raw_total else 0.0
    aggregate = BoundedAggregate(
        bucket=bucket,
        bounded_total=bounded_total,
        distinct_users=len(per_user),
        clipped_events=clipped_events,
        raw_total=raw_total,
    )

    if clipped_ratio > policy.max_clipped_ratio:
        return BoundaryDecision(
            "manual_review_high_clipping",
            aggregate,
            "too much traffic was clipped; bucket may be dominated by few users",
        )

    return BoundaryDecision(
        "release_bounded_aggregate",
        aggregate,
        "bounded aggregate is ready for DP noise",
    )
```

测试要覆盖三类危险情况：

```python
def test_rejects_events_without_user_id():
    decision = bound_contributions(
        [RawEvent(None, "tool_calls", 1)],
        ContributionPolicy(max_per_user_per_bucket=10, min_users_per_bucket=2, max_clipped_ratio=0.2),
        "tool_calls",
    )
    assert decision.decision == "reject_unbounded_events"

def test_suppresses_small_bucket():
    decision = bound_contributions(
        [RawEvent("u1", "errors", 3)],
        ContributionPolicy(max_per_user_per_bucket=10, min_users_per_bucket=3, max_clipped_ratio=0.2),
        "errors",
    )
    assert decision.decision == "suppress_small_bucket"

def test_high_clipping_requires_review():
    events = [
        RawEvent("u1", "tool_calls", 200),
        RawEvent("u2", "tool_calls", 1),
        RawEvent("u3", "tool_calls", 1),
    ]
    decision = bound_contributions(
        events,
        ContributionPolicy(max_per_user_per_bucket=10, min_users_per_bucket=3, max_clipped_ratio=0.2),
        "tool_calls",
    )
    assert decision.decision == "manual_review_high_clipping"
```

关键点：DP release worker 只能接收 `BoundedAggregate`，不能接收 raw event batch。

---

## 4. pi-mono：UserLevelAggregationWorker

生产版建议把 contribution bounding 做成 DP release 的前置 worker，并且用 receipt 把后续噪声发布绑住。

```ts
type RawMetricEvent = {
  userId?: string;
  tenantId: string;
  bucket: string;
  value: number;
  occurredAt: string;
};

type ContributionBoundaryPolicy = {
  privacyUnit: "user" | "tenant" | "device";
  maxPerUnitPerBucket: number;
  minUnitsPerBucket: number;
  maxClippedRatio: number;
  policyVersion: string;
};

type ContributionBoundingReceipt = {
  receiptId: string;
  bucket: string;
  policyVersion: string;
  boundedTotal: number;
  distinctUnits: number;
  clippedEvents: number;
  sourceBatchHash: string;
};

class UserLevelAggregationWorker {
  async aggregate(
    events: RawMetricEvent[],
    policy: ContributionBoundaryPolicy,
    bucket: string,
  ): Promise<ContributionBoundingReceipt> {
    const missingUnit = events.some((event) => !event.userId);
    if (missingUnit) {
      throw new Error("reject_unbounded_events");
    }

    const perUnit = new Map<string, number>();
    for (const event of events) {
      if (event.bucket !== bucket) continue;
      const unit = event.userId!;
      perUnit.set(unit, (perUnit.get(unit) ?? 0) + event.value);
    }

    if (perUnit.size < policy.minUnitsPerBucket) {
      throw new Error("suppress_small_bucket");
    }

    let boundedTotal = 0;
    let clippedEvents = 0;
    for (const value of perUnit.values()) {
      const clipped = Math.min(value, policy.maxPerUnitPerBucket);
      boundedTotal += clipped;
      clippedEvents += Math.max(0, value - clipped);
    }

    return {
      receiptId: crypto.randomUUID(),
      bucket,
      policyVersion: policy.policyVersion,
      boundedTotal,
      distinctUnits: perUnit.size,
      clippedEvents,
      sourceBatchHash: await hashCanonical(events),
    };
  }
}
```

DP release worker 接口应该像这样：

```ts
type DpReleaseInput = {
  contributionReceiptId: string;
  boundedTotal: number;
  epsilonCost: number;
  dimensionsFingerprint: string;
};
```

也就是说，release worker 不再有机会“顺手读 raw events”。它只能对已裁剪、已收据化的 bounded total 加噪。

---

## 5. OpenClaw 实战：课程 Cron 的指标发布

拿这套课程 Cron 举例：如果想发布“最近 30 天课程自动化成功率”，不要直接数所有 cron run 事件。先定义边界：

```json
{
  "metric": "agent_course_cron_success_rate",
  "window": "30d",
  "privacyUnit": "chat_or_repo",
  "maxPerUnitPerBucket": 3,
  "minUnitsPerBucket": 5,
  "policyVersion": "course-metrics-v1"
}
```

执行顺序：

1. 从 raw run log 生成 `sourceBatchHash`；
2. 按 `chat_or_repo` 聚合，每个 unit 最多贡献 3 次；
3. 小于 5 个 unit 的 bucket 直接 suppress；
4. 生成 `ContributionBoundingReceipt`；
5. DP release worker 只读取 receipt 和 bounded total；
6. 发布结果附带 `dpReceiptId`，内部审计可追到 contribution receipt。

简化 receipt：

```json
{
  "receiptId": "cb_20260619_course_success_001",
  "bucket": "agent_course_cron_success_rate",
  "policyVersion": "course-metrics-v1",
  "boundedTotal": 84,
  "distinctUnits": 31,
  "clippedEvents": 12,
  "sourceBatchHash": "sha256:abc123"
}
```

这样即使某个 repo 或群组特别活跃，也不会让它在聚合统计里无限放大。

---

## 6. 常见坑

- **直接对 raw event count 加噪**：事件级 DP 不等于用户级 DP；
- **发布前临时调高 cap**：攻击者可以用多次 cap 差分反推高活跃用户；
- **只记录 noisy release receipt**：后面无法证明 raw 数据有没有先裁剪；
- **小 bucket 不 suppress**：再强的噪声也挡不住“一两个人”的统计形状；
- **LLM 自己选择 cap**：cap 是策略，不是推理结果，应由 policy 配置或审批决定；
- **裁剪比例异常还继续发布**：说明 bucket 被少数 unit 主导，应该人工复核或粗化维度。

---

## 7. 记忆口诀

DP 发布顺序不是：

```text
raw events -> noise -> dashboard
```

而是：

```text
raw events -> privacy unit -> contribution cap -> bounded aggregate -> noise -> receipt -> dashboard
```

成熟 Agent 不只会“给数字加噪”，还会证明每个进入发布管道的数字，已经先被压到用户级贡献边界之内。
