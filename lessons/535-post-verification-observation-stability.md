# 535. Agent 验证通过后的观察窗口与稳定关闭闸门

> VerificationCloseoutReceipt 证明修复在验证时刻通过了测试、回归重放和现实探针，但不证明接下来真实流量、缓存、异步任务和下游消费者都会持续稳定。成熟 Agent 会把 VerificationCloseoutReceipt、ObservationLease、RuntimeSample、RegressionSignal、DownstreamProbe 和 StabilityCloseoutReceipt 串起来，防止“刚验证通过”变成“过几小时又复发”。

上一讲讲了 **Repair Evidence Verification & Regression Closeout Gate**：owner 说修好了以后，Agent 要验证修复证据、重放原始 seed、探测真实 runtime，确认修复确实命中根因。

今天继续往后走：**验证通过以后，Agent 能不能马上关单？**

答案是：高风险系统里不应该马上永久关闭。因为还有几类问题经常滞后出现：

- 异步队列还有旧版本 worker 在处理旧任务；
- CDN、Redis、浏览器缓存或本地投影仍然回流旧数据；
- 真实流量覆盖了测试没覆盖的输入组合；
- 下游服务还在读旧 schema、旧索引或旧配置；
- 演练 seed 通过了，但同类 failure signature 在生产样本里复发。

所以需要一个 **Post-Verification Observation & Stability Closeout Gate**：验证通过后先进入观察窗口，持续采样真实路径，最后用稳定关闭收据证明可以退出事故流程。

---

## 1. 核心链路

```text
VerificationCloseoutReceipt
        ↓
ObservationLease
        ↓
RuntimeSample
        ↓
RegressionSignal
        ↓
DownstreamProbe
        ↓
StabilityCloseoutReceipt
```

每一环回答一个问题：

- VerificationCloseoutReceipt：修复是否已经通过验证入口；
- ObservationLease：观察窗口多久、采样哪些路径、谁负责续租；
- RuntimeSample：真实 runtime 是否持续加载正确版本；
- RegressionSignal：同类 failure signature 是否复发；
- DownstreamProbe：缓存、投影、消费者是否已经收敛；
- StabilityCloseoutReceipt：最终是关闭、延长观察、回滚，还是重新打开修复流程。

一句话：**验证通过是进入观察窗口，不是直接宣布长期稳定。**

---

## 2. ObservationLease 应该包含什么？

观察窗口不是“等一等看看”，而是一张结构化租约：

```text
observationLease
  verificationCloseoutId: vco-534
  repairCaseId: repair-cold-17
  window:
    startedAt: 2026-07-01T03:30:00+11:00
    minDurationMinutes: 60
    maxDurationMinutes: 180
  requiredRuntime:
    gitSha: 9f2a1c7
    configHash: cfg-44ab
  watchedSignals:
    - missing_lookup_key
    - hot_fallback_observed
    - stale_cache_hit
    - downstream_schema_mismatch
  samplePlan:
    minRuntimeSamples: 30
    minDownstreamProbes: 5
    maxRegressionSignals: 0
```

这里的关键不是时间长短，而是把“怎么证明稳定”写清楚：样本数、信号类型、版本指纹、下游探针都必须绑定同一张 lease。

---

## 3. learn-claude-code：稳定关闭判定器

教学版先写纯函数。输入观察窗口里的样本和信号，输出下一步决策。

```python
from dataclasses import dataclass
from typing import Literal

Decision = Literal[
    "continue_observation",
    "extend_observation",
    "rollback_repair",
    "reopen_repair_case",
    "repair_downstream_projection",
    "close_stable",
    "manual_review",
]

@dataclass
class ObservationLease:
    required_git_sha: str
    required_config_hash: str
    min_runtime_samples: int
    min_downstream_probes: int
    max_regression_signals: int
    expired: bool

@dataclass
class RuntimeSample:
    git_sha: str
    config_hash: str
    healthy: bool
    stale_cache_hit: bool

@dataclass
class RegressionSignal:
    signature: str
    severity: Literal["low", "medium", "high", "critical"]
    matched_original_failure: bool

@dataclass
class DownstreamProbe:
    consumer: str
    converged: bool
    schema_mismatch: bool
    stale_projection: bool

def decide_stability_closeout(
    lease: ObservationLease,
    runtime_samples: list[RuntimeSample],
    regression_signals: list[RegressionSignal],
    downstream_probes: list[DownstreamProbe],
) -> Decision:
    if len(runtime_samples) < lease.min_runtime_samples:
        return "continue_observation"

    if len(downstream_probes) < lease.min_downstream_probes:
        return "continue_observation"

    for sample in runtime_samples:
        if sample.git_sha != lease.required_git_sha:
            return "extend_observation"
        if sample.config_hash != lease.required_config_hash:
            return "extend_observation"
        if not sample.healthy:
            return "manual_review"
        if sample.stale_cache_hit:
            return "extend_observation"

    critical_regression = [
        signal for signal in regression_signals
        if signal.severity == "critical" or signal.matched_original_failure
    ]
    if critical_regression:
        return "rollback_repair"

    if len(regression_signals) > lease.max_regression_signals:
        return "reopen_repair_case"

    broken_downstream = [
        probe for probe in downstream_probes
        if probe.schema_mismatch or probe.stale_projection
    ]
    if broken_downstream:
        return "repair_downstream_projection"

    if any(not probe.converged for probe in downstream_probes):
        return "extend_observation"

    if not lease.expired:
        return "continue_observation"

    return "close_stable"
```

对应测试：

```python
def test_does_not_close_when_downstream_projection_is_stale():
    lease = ObservationLease(
        required_git_sha="9f2a1c7",
        required_config_hash="cfg-44ab",
        min_runtime_samples=2,
        min_downstream_probes=1,
        max_regression_signals=0,
        expired=True,
    )
    samples = [
        RuntimeSample("9f2a1c7", "cfg-44ab", True, False),
        RuntimeSample("9f2a1c7", "cfg-44ab", True, False),
    ]
    probes = [
        DownstreamProbe(
            consumer="dashboard-cache",
            converged=False,
            schema_mismatch=False,
            stale_projection=True,
        )
    ]

    assert decide_stability_closeout(
        lease,
        samples,
        regression_signals=[],
        downstream_probes=probes,
    ) == "repair_downstream_projection"
```

这个测试的重点是：**修复服务本身稳定，不代表所有读取路径都稳定。** Agent 关闭事故前要检查消费者、缓存和投影。

---

## 4. pi-mono：PostVerificationObserver

生产版可以放在 `RepairVerificationWorker` 后面。它消费 `VerificationCloseoutReceipt`，创建观察租约，周期性采样 runtime、查询回归信号、探测下游消费者，最后写稳定关闭收据。

```typescript
type StabilityDecision =
  | "continue_observation"
  | "extend_observation"
  | "rollback_repair"
  | "reopen_repair_case"
  | "repair_downstream_projection"
  | "close_stable"
  | "manual_review";

type ObservationLease = {
  id: string;
  verificationCloseoutId: string;
  requiredGitSha: string;
  requiredConfigHash: string;
  minRuntimeSamples: number;
  minDownstreamProbes: number;
  maxRegressionSignals: number;
  expiresAt: Date;
};

export class PostVerificationObserver {
  constructor(
    private readonly leaseStore: ObservationLeaseStore,
    private readonly runtimeSampler: RuntimeSampler,
    private readonly regressionFeed: RegressionSignalFeed,
    private readonly downstreamProbe: DownstreamProbeRunner,
    private readonly receiptStore: ReceiptStore,
    private readonly outbox: Outbox,
  ) {}

  async tick(leaseId: string): Promise<void> {
    const lease = await this.leaseStore.get(leaseId);

    const [runtimeSamples, regressionSignals, downstreamProbes] =
      await Promise.all([
        this.runtimeSampler.collect(lease.id),
        this.regressionFeed.findByCase(lease.verificationCloseoutId),
        this.downstreamProbe.collect(lease.id),
      ]);

    const decision = decideStabilityCloseout({
      lease,
      runtimeSamples,
      regressionSignals,
      downstreamProbes,
      now: new Date(),
    });

    await this.applyDecision(lease, decision, {
      runtimeSamples,
      regressionSignals,
      downstreamProbes,
    });
  }

  private async applyDecision(
    lease: ObservationLease,
    decision: StabilityDecision,
    evidence: unknown,
  ) {
    switch (decision) {
      case "continue_observation":
        await this.outbox.enqueue("observation.tick", {
          leaseId: lease.id,
          delaySeconds: 300,
        });
        return;

      case "extend_observation":
        await this.leaseStore.extend(lease.id, { minutes: 30, reason: "stability_not_proven" });
        await this.outbox.enqueue("observation.tick", { leaseId: lease.id, delaySeconds: 300 });
        return;

      case "repair_downstream_projection":
        await this.outbox.enqueue("projection.repair.requested", {
          leaseId: lease.id,
          evidence,
        });
        return;

      case "rollback_repair":
        await this.outbox.enqueue("repair.rollback.requested", {
          verificationCloseoutId: lease.verificationCloseoutId,
          evidence,
        });
        return;

      case "reopen_repair_case":
        await this.outbox.enqueue("repair.case.reopened", {
          verificationCloseoutId: lease.verificationCloseoutId,
          evidence,
        });
        return;

      case "manual_review":
        await this.outbox.enqueue("incident.manual_review", {
          leaseId: lease.id,
          evidence,
        });
        return;

      case "close_stable":
        await this.receiptStore.append({
          type: "StabilityCloseoutReceipt",
          leaseId: lease.id,
          verificationCloseoutId: lease.verificationCloseoutId,
          closedAt: new Date().toISOString(),
          evidence,
        });
        return;
    }
  }
}
```

这里有两个生产细节：

1. `tick()` 必须幂等，重复执行不能生成多张互相冲突的 closeout receipt；
2. `close_stable` 前最好用数据库唯一键约束 `(leaseId, receiptType)`，避免并发 worker 同时关闭。

---

## 5. OpenClaw 课程 Cron 类比

这套机制可以直接套到我们这个课程自动发布任务：

```text
VerificationCloseoutReceipt
  - lesson file exists
  - README updated
  - TOOLS.md updated
  - git commit created
  - git push succeeded
  - Telegram send returned message id

ObservationLease
  - after push, re-check remote branch contains commit
  - after send, verify Telegram delivery receipt
  - after docs update, verify README link resolves

RegressionSignal
  - duplicate lesson number
  - missing TOOLS.md topic
  - git push rejected
  - Telegram delivery failed

StabilityCloseoutReceipt
  - all side effects observed in the outside world
  - no duplicate or stale repo state
```

这就是成熟 Agent 和普通脚本的差别：普通脚本执行完命令就退出；成熟 Agent 会在关键副作用后观察现实是否真的收敛。

---

## 6. 实战建议

- 观察窗口要短而明确：常见是 30-180 分钟，不要无限观察；
- 采样要覆盖真实读写路径，不只看服务健康检查；
- 复发信号要绑定原始 failure signature，否则容易把无关噪声当复发；
- 下游缓存、投影、报表、搜索索引都要有探针；
- 稳定关闭必须写 receipt，后续审计才能知道为什么当时允许关单；
- 如果观察窗口多次延长，要升级人工或拆成新的 repair case。

---

## 总结

**Post-Verification Observation & Stability Closeout Gate** 把“验证通过”升级成“持续稳定可证明”。

核心不是多等一会儿，而是：

1. 用 ObservationLease 定义观察范围、时间和样本要求；
2. 用 RuntimeSample 确认真实服务持续加载正确版本；
3. 用 RegressionSignal 捕获同类 failure 是否复发；
4. 用 DownstreamProbe 检查缓存、投影和消费者是否收敛；
5. 用 StabilityCloseoutReceipt 证明可以安全退出事故流程。

成熟 Agent 不把通过测试当终点，而是用观察窗口证明现实世界也跟着稳定了。
