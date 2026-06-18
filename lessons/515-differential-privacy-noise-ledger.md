# 515 - Agent 差分隐私噪声账本与发布收据

> 上一讲讲了 aggregate counter 发布前要检查小桶量、差分攻击和重识别漂移。但还有一个更工程化的问题：有些统计必须发布，又不能让小群体被精确反推。这时不能只靠 k-anonymity，要引入差分隐私噪声，并且把 epsilon 预算当成可消耗、可审计的资源。

今天讲 **Differential Privacy Noise Ledger & Release Receipt**：Agent 在发布聚合统计前，先判断这次查询是否需要噪声、消耗多少 privacy budget、使用哪个 noise mechanism，再把 noisy result、epsilon 扣减和 seed commitment 写成发布收据。

常见坑：

- 给所有 counter 加同一种噪声，结果低风险统计不准，高风险统计又不够安全；
- 只记录 noisy value，不记录 epsilon 消耗，长期重复查询把隐私预算烧穿；
- 为了可复现把 noise seed 明文保存，反而让攻击者可以去噪；
- dashboard 重新刷新时重复扣预算或重复采样，导致数值跳来跳去；
- LLM 看到原始计数和 noisy 计数，最终把 raw value 泄露到回答里。

成熟 Agent 要把“加噪声”从数学技巧变成工程协议：**预算、机制、采样、发布、解释、审计全部有收据**。

---

## 核心模型

```
CounterReleaseDecision
        ↓
PrivacyBudgetLedger
        ↓
NoiseMechanismPlan
        ↓
NoisyCounterRelease
        ↓
DifferentialPrivacyReleaseReceipt
        ↓
RawCounterPurgeProbe
```

关键判断：

- 这次发布是否需要 DP，还是只需要维度粗化；
- epsilon 是否足够，是否触发 budget exhaustion；
- 使用 Laplace、Gaussian，还是 suppression；
- noisy value 是否保持非负、单调或业务约束；
- raw count 是否只在隔离执行区短暂存在；
- receipt 是否只保存 seed commitment，不保存可复原噪声的 seed。

---

## learn-claude-code：噪声发布闸门

教学版先写一个纯函数：输入发布请求、预算账本和风险评估，输出发布计划。

```python
from dataclasses import dataclass
from typing import Literal

Decision = Literal[
    "release_without_noise",
    "release_with_laplace_noise",
    "release_with_gaussian_noise",
    "suppress_result",
    "deny_privacy_budget_exhausted",
    "manual_review",
]

@dataclass
class CounterReleaseDecision:
    counter_id: str
    consumer_id: str
    raw_count: int
    bucket_size: int
    public_release: bool
    export_requested: bool
    dimensions_fingerprint: str

@dataclass
class PrivacyBudgetLedger:
    epsilon_remaining: float
    daily_queries: int
    repeated_same_query: bool
    max_daily_queries: int

@dataclass
class PrivacyRiskReview:
    low_bucket_margin: bool
    external_join_risk: bool
    legal_or_security_report: bool
    requires_monotonic_output: bool

@dataclass
class NoiseMechanismPlan:
    decision: Decision
    epsilon_cost: float
    mechanism: str | None
    sensitivity: float
    explanation: str

def plan_noisy_release(
    release: CounterReleaseDecision,
    budget: PrivacyBudgetLedger,
    risk: PrivacyRiskReview,
) -> NoiseMechanismPlan:
    if budget.daily_queries >= budget.max_daily_queries:
        return NoiseMechanismPlan(
            "deny_privacy_budget_exhausted",
            0.0,
            None,
            1.0,
            "daily query budget exhausted",
        )

    if release.bucket_size < 10:
        return NoiseMechanismPlan(
            "suppress_result",
            0.0,
            None,
            1.0,
            "bucket too small even with noise",
        )

    if risk.external_join_risk and release.public_release:
        epsilon_cost = 0.25
        if budget.epsilon_remaining < epsilon_cost:
            return NoiseMechanismPlan(
                "deny_privacy_budget_exhausted",
                0.0,
                None,
                1.0,
                "not enough epsilon for public high-risk release",
            )
        return NoiseMechanismPlan(
            "release_with_gaussian_noise",
            epsilon_cost,
            "gaussian",
            1.0,
            "public release with external join risk",
        )

    if risk.low_bucket_margin or release.export_requested:
        epsilon_cost = 0.1 if not budget.repeated_same_query else 0.2
        if budget.epsilon_remaining < epsilon_cost:
            return NoiseMechanismPlan(
                "deny_privacy_budget_exhausted",
                0.0,
                None,
                1.0,
                "not enough epsilon for repeated/export query",
            )
        return NoiseMechanismPlan(
            "release_with_laplace_noise",
            epsilon_cost,
            "laplace",
            1.0,
            "export or low bucket margin requires noise",
        )

    if risk.legal_or_security_report:
        return NoiseMechanismPlan(
            "manual_review",
            0.0,
            None,
            1.0,
            "legal/security report should choose exact vs noisy view explicitly",
        )

    return NoiseMechanismPlan(
        "release_without_noise",
        0.0,
        None,
        1.0,
        "internal coarse aggregate with enough bucket margin",
    )
```

测试重点：同一查询重复出现时，epsilon 成本要上升；小桶量太小要 suppress，而不是假装噪声能解决一切。

```python
def test_repeated_export_costs_more_budget():
    release = CounterReleaseDecision(
        counter_id="counter_erasure_monthly",
        consumer_id="dashboard",
        raw_count=87,
        bucket_size=25,
        public_release=False,
        export_requested=True,
        dimensions_fingerprint="dim_v1",
    )
    budget = PrivacyBudgetLedger(
        epsilon_remaining=0.15,
        daily_queries=3,
        repeated_same_query=True,
        max_daily_queries=20,
    )
    risk = PrivacyRiskReview(
        low_bucket_margin=True,
        external_join_risk=False,
        legal_or_security_report=False,
        requires_monotonic_output=False,
    )

    plan = plan_noisy_release(release, budget, risk)
    assert plan.decision == "deny_privacy_budget_exhausted"
```

这里的核心不是“加一点随机数”，而是：**每一次可观察输出都要消耗预算；预算不足时宁可拒绝或抑制，也不要发布伪安全统计**。

---

## pi-mono：DifferentialPrivacyReleaseWorker

生产版把这个逻辑放在 counter release worker 后面。上一讲的 release gate 决定“能不能发布”，这一讲的 DP worker 决定“以什么精度发布”。

```ts
type NoiseDecision =
  | "release_without_noise"
  | "release_with_laplace_noise"
  | "release_with_gaussian_noise"
  | "suppress_result"
  | "deny_privacy_budget_exhausted"
  | "manual_review";

type DpReleaseJob = {
  counterId: string;
  consumerId: string;
  rawCount: number;
  bucketSize: number;
  dimensionsFingerprint: string;
  publicRelease: boolean;
  exportRequested: boolean;
  idempotencyKey: string;
};

type DifferentialPrivacyReleaseReceipt = {
  counterId: string;
  consumerId: string;
  decision: NoiseDecision;
  noisyValue: number | null;
  epsilonCost: number;
  epsilonRemainingAfter: number;
  mechanism: "none" | "laplace" | "gaussian" | "suppressed";
  seedCommitment: string | null;
  dimensionsFingerprint: string;
  idempotencyKey: string;
  createdAt: string;
};

class DifferentialPrivacyReleaseWorker {
  constructor(
    private readonly budget: PrivacyBudgetStore,
    private readonly risk: PrivacyRiskService,
    private readonly noise: NoiseSampler,
    private readonly receipts: DpReceiptStore,
    private readonly rawPurge: RawCounterPurgeProbe,
  ) {}

  async handle(job: DpReleaseJob): Promise<DifferentialPrivacyReleaseReceipt> {
    const existing = await this.receipts.findByIdempotencyKey(job.idempotencyKey);
    if (existing) return existing;

    return this.budget.withConsumerLease(job.consumerId, async () => {
      const ledger = await this.budget.getLedger(job.consumerId, job.counterId);
      const risk = await this.risk.review(job);
      const plan = planNoisyRelease(job, ledger, risk);

      let noisyValue: number | null = null;
      let seedCommitment: string | null = null;

      if (plan.decision === "release_without_noise") {
        noisyValue = job.rawCount;
      }

      if (plan.decision === "release_with_laplace_noise") {
        const sampled = await this.noise.laplace({
          value: job.rawCount,
          epsilon: plan.epsilonCost,
          sensitivity: plan.sensitivity,
          idempotencyKey: job.idempotencyKey,
        });
        noisyValue = clampCount(sampled.value);
        seedCommitment = sampled.seedCommitment;
      }

      if (plan.decision === "release_with_gaussian_noise") {
        const sampled = await this.noise.gaussian({
          value: job.rawCount,
          epsilon: plan.epsilonCost,
          sensitivity: plan.sensitivity,
          idempotencyKey: job.idempotencyKey,
        });
        noisyValue = clampCount(sampled.value);
        seedCommitment = sampled.seedCommitment;
      }

      if (plan.epsilonCost > 0) {
        await this.budget.consume(job.consumerId, job.counterId, plan.epsilonCost);
      }

      const remaining = await this.budget.remaining(job.consumerId, job.counterId);
      const receipt: DifferentialPrivacyReleaseReceipt = {
        counterId: job.counterId,
        consumerId: job.consumerId,
        decision: plan.decision,
        noisyValue,
        epsilonCost: plan.epsilonCost,
        epsilonRemainingAfter: remaining,
        mechanism: mechanismFromDecision(plan.decision),
        seedCommitment,
        dimensionsFingerprint: job.dimensionsFingerprint,
        idempotencyKey: job.idempotencyKey,
        createdAt: new Date().toISOString(),
      };

      await this.receipts.save(receipt);
      await this.rawPurge.assertNoRawCounterLeaked(job.counterId, job.idempotencyKey);
      return receipt;
    });
  }
}
```

几个工程细节：

- `idempotencyKey` 必须固定，否则 dashboard 刷新会重新采样，用户看到的统计会抖；
- `seedCommitment` 是 hash commitment，不是 seed 本身；
- `rawCount` 只能在 worker 内部短暂存在，不能写入 receipt、日志或 LLM context；
- `clampCount` 只做业务约束修正，比如非负整数，不要把 noisy value 偷偷改回 raw count；
- `budget.consume` 和 `receipts.save` 最好在同一事务或 outbox 补偿协议里。

---

## OpenClaw：课程 Cron 里的类比

这个定时课程任务也有一个类似问题：我们每 3 小时发布一课，会更新三个外部可见面：

- Telegram 群消息；
- lesson 文件；
- README / TOOLS 目录。

如果要发布“课程统计”，比如“过去 30 天失败了几次 cron”，直接给精确小数字可能暴露内部运行细节。更稳的做法：

```json
{
  "metric": "agent_course_cron_failures_30d",
  "rawCountVisibleToWorkerOnly": true,
  "publishedValue": 3,
  "noiseMechanism": "laplace",
  "epsilonCost": 0.1,
  "seedCommitment": "sha256:...",
  "receipt": "dp_release_20260619_0030"
}
```

对外只发 noisy result 和解释：

> 过去 30 天课程自动化失败次数约为 3 次，已做隐私保护噪声处理。

对内保留 receipt：

- 证明这次发布用了什么机制；
- 证明扣了多少 epsilon；
- 证明没有把 raw count 注入模型上下文；
- 以后重复查询时可以复用同一个 noisy result，而不是重新采样。

这就是 Always-on Agent 的关键差别：**它不只是会回答统计问题，还知道哪些统计答案本身会泄露系统状态或用户轨迹**。

---

## 实战清单

实现 DP 发布账本时，至少检查这些点：

- [ ] 每个 consumer / counter / purpose 都有独立 epsilon ledger；
- [ ] 重复查询走 idempotency key，复用同一个 noisy release；
- [ ] receipt 不保存 raw count，不保存可复原 seed；
- [ ] 小桶量过小优先 suppress，不迷信噪声；
- [ ] 高风险 public/export 查询消耗更高预算；
- [ ] LLM 只能看到 noisy value 和 caveat，看不到 raw value；
- [ ] dashboard cache 绑定 release receipt，不能绕过 DP worker；
- [ ] privacy budget 耗尽时返回可解释拒绝，而不是降级成精确值。

---

## 一句话总结

> 差分隐私在 Agent 系统里不是“给数字加随机数”，而是把每一次统计发布都变成可预算、可复用、可审计、不可反推 raw value 的发布协议。

