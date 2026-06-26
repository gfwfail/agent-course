# 531. Agent 演练执行结果升级与关闭闸门

> DrillActivationReceipt 只证明演练已经被调度，不证明演练真的跑过、失败有人接、成功结果被写成可审计收据。成熟 Agent 会把每次演练执行结果、失败升级、修复工单、下次种子调整和关闭收据串成一条闭环。

上一讲讲了 **Cold Retrieval Stability Drill Scheduling & Regression Seed Gate**：冷路径稳定关闭后，要把真实事故样本变成回归种子，并把演练计划接入调度、监控和告警。

今天继续往后走：**演练被调度以后，怎么证明它不是一个没人看的定时任务？**

常见问题是：

- synthetic drill 已经创建，但连续失败只在日志里留下 error；
- 演练成功了，却没有记录覆盖了哪些 seed、bundle、runtime 和 SLA；
- 失败后 oncall 修了索引，但没有把新失败样本补回回归库；
- flaky drill 被静默跳过，慢慢变成“绿色但没意义”的检查；
- 下游只看 cron status，不看演练输出是否被消费、确认和关闭。

所以要加一层 **Drill Execution Result, Escalation & Closeout Gate**：演练不是跑完就算，必须把执行结果变成可路由、可升级、可关闭、可学习的证据。

---

## 1. 核心链路

```text
DrillActivationReceipt
        ↓
DrillExecutionRun
        ↓
DrillResultReview
        ↓
EscalationOrCloseoutDecision
        ↓
DrillCloseoutReceipt
```

每一环回答一个问题：

- activation receipt：演练计划是否已经正式激活；
- execution run：本轮实际跑了哪些 seed、runtime、bundle 和租户范围；
- result review：结果是 pass、fail、flaky、skipped 还是 stale；
- escalation decision：失败要不要升级成 incident、修复任务或种子更新；
- closeout receipt：本轮演练是否已经被确认、归档，并影响下一轮计划。

一句话：**调度是入口，关闭才是交代；没有 closeout receipt 的演练，只是一个会定时产生日志的脚本。**

---

## 2. learn-claude-code：演练结果判定器

教学版先写纯函数。它不负责跑演练，只判断一次执行结果应该如何处理。

```python
from dataclasses import dataclass
from typing import Literal

Decision = Literal[
    "close_drill_passed",
    "escalate_retrieval_regression",
    "open_seed_refresh_task",
    "rerun_flaky_drill",
    "repair_drill_harness",
    "manual_review",
]

@dataclass
class DrillActivationReceipt:
    drill_id: str
    active: bool
    expected_seed_count: int
    expected_runtime_fingerprint: str
    owner_oncall_route: str | None
    alert_enabled: bool

@dataclass
class DrillExecutionRun:
    run_id: str
    status: Literal["passed", "failed", "flaky", "skipped"]
    executed_seed_count: int
    runtime_fingerprint: str
    oldest_bundle_checked: bool
    p95_ms: int
    error_rate: float
    missing_lookup_keys: int
    lease_denials: int
    harness_error: bool

@dataclass
class DrillResultReview:
    result_acknowledged: bool
    consecutive_flaky_runs: int
    consecutive_skipped_runs: int
    new_failure_seed_count: int
    sla_p95_ms: int
    max_error_rate: float

def decide_drill_execution_gate(
    activation: DrillActivationReceipt,
    run: DrillExecutionRun,
    review: DrillResultReview,
) -> Decision:
    if not activation.active:
        return "manual_review"

    if not activation.owner_oncall_route or not activation.alert_enabled:
        return "manual_review"

    if run.harness_error:
        return "repair_drill_harness"

    if run.status == "skipped":
        if review.consecutive_skipped_runs >= 2:
            return "repair_drill_harness"
        return "rerun_flaky_drill"

    if run.runtime_fingerprint != activation.expected_runtime_fingerprint:
        return "open_seed_refresh_task"

    if run.executed_seed_count < activation.expected_seed_count:
        return "open_seed_refresh_task"

    if not run.oldest_bundle_checked:
        return "open_seed_refresh_task"

    if run.status == "flaky":
        if review.consecutive_flaky_runs >= 3:
            return "repair_drill_harness"
        return "rerun_flaky_drill"

    regression_detected = (
        run.status == "failed"
        or run.p95_ms > review.sla_p95_ms
        or run.error_rate > review.max_error_rate
        or run.missing_lookup_keys > 0
        or run.lease_denials > 0
    )
    if regression_detected:
        return "escalate_retrieval_regression"

    if review.new_failure_seed_count > 0:
        return "open_seed_refresh_task"

    if not review.result_acknowledged:
        return "manual_review"

    return "close_drill_passed"
```

这个判定器刻意把三类失败分开：

1. **真实回归**：missing lookup key、lease denial、P95 超标、error rate 超标，要升级；
2. **演练自身坏了**：harness error、连续 skipped、连续 flaky，要修演练工具；
3. **覆盖变旧了**：runtime fingerprint 或 seed count 不匹配，要刷新回归种子。

对应测试可以这样写：

```python
def test_escalates_when_drill_finds_missing_lookup_keys():
    activation = DrillActivationReceipt(
        drill_id="cold-audit-daily",
        active=True,
        expected_seed_count=20,
        expected_runtime_fingerprint="cold-v19",
        owner_oncall_route="cold-path-oncall",
        alert_enabled=True,
    )
    run = DrillExecutionRun(
        run_id="run-42",
        status="failed",
        executed_seed_count=20,
        runtime_fingerprint="cold-v19",
        oldest_bundle_checked=True,
        p95_ms=850,
        error_rate=0.0,
        missing_lookup_keys=1,
        lease_denials=0,
        harness_error=False,
    )
    review = DrillResultReview(
        result_acknowledged=True,
        consecutive_flaky_runs=0,
        consecutive_skipped_runs=0,
        new_failure_seed_count=1,
        sla_p95_ms=1200,
        max_error_rate=0.01,
    )

    assert decide_drill_execution_gate(activation, run, review) == "escalate_retrieval_regression"
```

重点不是“失败就报警”，而是分清：是系统退化、演练退化，还是测试样本退化。三种问题的 owner 和修复路径完全不同。

---

## 3. pi-mono：DrillResultCloseoutWorker

生产版建议把它做成每次 synthetic drill 之后的 closeout worker。它消费执行结果，写审计收据，并把失败路由到正确队列。

```typescript
type DrillDecision =
  | "close_drill_passed"
  | "escalate_retrieval_regression"
  | "open_seed_refresh_task"
  | "rerun_flaky_drill"
  | "repair_drill_harness"
  | "manual_review";

interface DrillRunCompletedEvent {
  drillId: string;
  runId: string;
  activationReceiptId: string;
}

class DrillResultCloseoutWorker {
  constructor(
    private readonly receipts: ReceiptStore,
    private readonly drillRuns: DrillRunStore,
    private readonly decisionEngine: DrillDecisionEngine,
    private readonly incidents: IncidentQueue,
    private readonly seedRefresh: SeedRefreshQueue,
    private readonly harnessRepair: HarnessRepairQueue,
    private readonly audit: AuditLog,
  ) {}

  async run(event: DrillRunCompletedEvent) {
    const activation = await this.receipts.getDrillActivation(
      event.activationReceiptId,
    );

    const execution = await this.drillRuns.getExecution(event.runId);
    const review = await this.drillRuns.reviewResult({
      drillId: event.drillId,
      runId: event.runId,
      expectedSeedCount: activation.expectedSeedCount,
      expectedRuntimeFingerprint: activation.expectedRuntimeFingerprint,
    });

    const decision = this.decisionEngine.decide({
      activation,
      execution,
      review,
    });

    await this.routeDecision(decision, {
      drillId: event.drillId,
      runId: event.runId,
      activationReceiptId: event.activationReceiptId,
      ownerRoute: activation.ownerOncallRoute,
      regressionSignals: review.regressionSignals,
      newFailureSeeds: review.newFailureSeeds,
    });

    const closeout = await this.receipts.createDrillCloseout({
      drillId: event.drillId,
      runId: event.runId,
      activationReceiptId: event.activationReceiptId,
      decision,
      executedSeedCount: execution.executedSeedCount,
      runtimeFingerprint: execution.runtimeFingerprint,
      p95Ms: execution.p95Ms,
      errorRate: execution.errorRate,
      regressionSignalHash: review.regressionSignalHash,
      createdAt: new Date(),
    });

    await this.audit.append({
      type: "drill.closeout",
      subject: event.drillId,
      receiptId: closeout.id,
      decision,
    });

    return closeout;
  }

  private async routeDecision(
    decision: DrillDecision,
    context: {
      drillId: string;
      runId: string;
      activationReceiptId: string;
      ownerRoute: string;
      regressionSignals: string[];
      newFailureSeeds: string[];
    },
  ) {
    if (decision === "escalate_retrieval_regression") {
      await this.incidents.enqueue({
        kind: "cold_retrieval_regression",
        ownerRoute: context.ownerRoute,
        drillId: context.drillId,
        runId: context.runId,
        signals: context.regressionSignals,
      });
      return;
    }

    if (decision === "open_seed_refresh_task") {
      await this.seedRefresh.enqueue({
        drillId: context.drillId,
        runId: context.runId,
        activationReceiptId: context.activationReceiptId,
        newFailureSeeds: context.newFailureSeeds,
      });
      return;
    }

    if (decision === "repair_drill_harness") {
      await this.harnessRepair.enqueue({
        drillId: context.drillId,
        runId: context.runId,
        ownerRoute: context.ownerRoute,
      });
    }
  }
}
```

这里有几个工程要点：

- `DrillCloseoutReceipt` 必须追加写，不能覆盖上一次演练结果；
- incident、seed refresh、harness repair 是三个队列，不要混在一个“失败处理”里；
- closeout 要记录 `regressionSignalHash`，避免把敏感 seed 原文塞进审计日志；
- worker 要幂等：同一个 `runId` 重放时返回同一个 closeout，不能重复开 incident。

---

## 4. OpenClaw 课程 Cron 类比

这套课程 cron 也有类似闭环：

```text
课程触发
  ↓
生成 lesson
  ↓
更新 README / TOOLS
  ↓
发送 Telegram
  ↓
git commit + push
  ↓
最终汇报
```

如果只做到“写了 lesson 文件”，但没发 Telegram、没更新目录、没提交推送，那就像 synthetic drill 只跑了脚本没有 closeout。

更好的做法是每次都留下可检查证据：

- lesson 文件：本轮实际产物；
- README 条目：目录投影已更新；
- TOOLS 已讲内容：去重索引已更新；
- Telegram 发送结果：下游消费者已收到；
- git commit hash：版本化关闭收据。

Agent 系统里同理：**一次演练真正结束，应该能回答“跑了什么、覆盖了什么、谁确认了、失败去哪了、下一轮怎么变得更好”。**

---

## 5. 实战检查清单

设计演练关闭闸门时，可以问 8 个问题：

1. 每次 drill run 有没有唯一 `runId` 和幂等 closeout？
2. pass/fail/flaky/skipped/stale 有没有分开建模？
3. 真实系统回归和演练工具故障有没有分开 owner？
4. seed 覆盖不足时，是刷新种子，而不是假装演练成功？
5. runtime fingerprint、bundle version、SLA 阈值有没有进入 closeout？
6. 失败信号有没有脱敏后写入审计，而不是 raw payload 进日志？
7. 连续 skipped / flaky 有没有升级，而不是无限重试？
8. closeout 有没有反向影响下一轮 seed、schedule、alert 和 oncall route？

---

## 6. 小结

今天的关键点：

- **演练调度不等于演练完成**：必须看 execution run 和 result review；
- **失败要分类**：系统回归、演练故障、覆盖变旧是三种不同问题；
- **关闭要有收据**：每轮演练都应该追加写 DrillCloseoutReceipt；
- **收据要影响未来**：失败样本、runtime 变化和 SLA 漂移要反向更新下一轮演练；
- **成熟 Agent 不只会跑检查，还会证明检查结果被消费、升级或关闭。**

下一讲可以继续讲：**演练失败升级后的修复验证与种子回灌闸门**，也就是 incident 修完以后，如何证明修复有效，并把新失败样本安全加入回归库。
