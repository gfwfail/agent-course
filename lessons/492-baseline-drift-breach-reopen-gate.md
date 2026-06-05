# Agent 常态基线漂移违约与重开闸门

> Normal Ops Baseline Drift Breach & Reopen Gate

第 491 课讲了 Normal Ops Baseline Recalibration：事故稳定关闭后，把 incident mode 期间临时调整过的 timeout、采样率、并发、成本阈值、告警阈值重新收敛成可版本、可回滚、可观察的新常态基线，并挂一个短期 drift sentinel。

今天补上哨兵真正报警后的动作。很多系统会在这里犯错：看到漂移就立刻全量回滚，或者只发一条告警然后继续跑。前者容易制造二次事故，后者会让“新常态”悄悄变成坏常态。

所以第 492 课是 **Normal Ops Baseline Drift Breach & Reopen Gate**：当新常态基线被激活后，如果 Drift Sentinel 发现错误率、延迟、成本、工具失败率、审批绕行或采样缺口超出契约，要把 breach 先变成可审计证据，再按影响面决定 rollback_baseline、reopen_observation、open_regression_case、extend_sentinel 或 ignore_noise。

## 核心模型

把漂移违约处理拆成 5 个对象：

~~~text
BaselineActivationReceipt
  -> BaselineDriftSentinelLease
  -> BaselineDriftBreachReport
  -> BaselineReopenReadinessReview
  -> BaselineDriftCloseoutReceipt
~~~

分工：

- BaselineActivationReceipt：上一课写入的新常态基线，包含 baselineVersion、rollbackBaselineVersion、scope；
- BaselineDriftSentinelLease：哨兵租约，定义观察窗口、阈值、样本数、退出条件；
- BaselineDriftBreachReport：真实漂移证据，不是单点告警，要包含连续窗口、影响面、是否可复现；
- BaselineReopenReadinessReview：判断是否需要回滚基线、重开观察期、开根因案件，还是只是延长哨兵；
- BaselineDriftCloseoutReceipt：关闭收据，记录采取了什么动作、谁负责、下一次检查何时发生。

一句话：**基线漂移不是“报警”这么简单，而是一次小型事故 intake。要先确认 breach，再选择最小但可回滚的控制面动作。**

## learn-claude-code：漂移违约判定纯函数

教学版用纯函数表达核心判断。它不写数据库、不发告警，只把输入证据变成决策，方便单元测试。

~~~python
# learn_claude_code/baseline_drift_breach_gate.py
from dataclasses import dataclass
from typing import Literal

DriftDecision = Literal[
    "rollback_baseline",
    "reopen_observation",
    "open_regression_case",
    "extend_sentinel",
    "ignore_noise",
    "manual_review",
]


@dataclass
class BaselineActivationReceipt:
    case_id: str
    baseline_version: str
    rollback_baseline_version: str | None
    decision: str  # activate_normal_baseline | activate_with_sentinel
    scope: str


@dataclass
class BaselineDriftSentinelLease:
    case_id: str
    baseline_version: str
    min_windows: int
    max_error_rate: float
    max_p95_ms: int
    max_cost_per_run: float
    max_tool_failure_rate: float
    expires_at: int


@dataclass
class BaselineDriftBreachReport:
    case_id: str
    baseline_version: str
    observed_windows: int
    error_rate: float
    p95_ms: int
    cost_per_run: float
    tool_failure_rate: float
    affected_users: int
    release_blocking: bool
    reproduced_in_replay: bool
    now: int


def decide_baseline_drift_breach(
    activation: BaselineActivationReceipt,
    lease: BaselineDriftSentinelLease,
    report: BaselineDriftBreachReport,
) -> DriftDecision:
    if (
        activation.case_id != lease.case_id
        or activation.case_id != report.case_id
        or activation.baseline_version != lease.baseline_version
        or activation.baseline_version != report.baseline_version
    ):
        return "manual_review"

    if report.now > lease.expires_at:
        return "manual_review"

    if report.observed_windows < lease.min_windows:
        return "extend_sentinel"

    breached = (
        report.error_rate > lease.max_error_rate
        or report.p95_ms > lease.max_p95_ms
        or report.cost_per_run > lease.max_cost_per_run
        or report.tool_failure_rate > lease.max_tool_failure_rate
    )

    if not breached:
        return "ignore_noise"

    if report.release_blocking and activation.rollback_baseline_version:
        return "rollback_baseline"

    if report.affected_users > 0 and report.reproduced_in_replay:
        return "open_regression_case"

    if report.affected_users > 0:
        return "reopen_observation"

    return "extend_sentinel"
~~~

这里最重要的是 `observed_windows` 和 `reproduced_in_replay`。单个尖刺不一定要回滚，连续窗口才说明新基线失效；能被 replay 复现的漂移，才值得开根因案件，否则先延长哨兵，避免被偶发噪声拖进事故流程。

## pi-mono：BaselineDriftBreachWorker

生产版 worker 可以消费 sentinel 采样结果，事务化生成 closeout receipt，并把动作发给 config registry、regression queue 或 observation scheduler。

~~~ts
// packages/agent-runtime/src/ops/BaselineDriftBreachWorker.ts
export type DriftDecision =
  | "rollback_baseline"
  | "reopen_observation"
  | "open_regression_case"
  | "extend_sentinel"
  | "ignore_noise"
  | "manual_review";

export interface BaselineActivationReceipt {
  caseId: string;
  baselineVersion: string;
  rollbackBaselineVersion?: string;
  decision: "activate_normal_baseline" | "activate_with_sentinel";
  scope: string;
}

export interface BaselineDriftSentinelLease {
  caseId: string;
  baselineVersion: string;
  minWindows: number;
  maxErrorRate: number;
  maxP95Ms: number;
  maxCostPerRun: number;
  maxToolFailureRate: number;
  expiresAt: number;
}

export interface BaselineDriftBreachReport {
  caseId: string;
  baselineVersion: string;
  observedWindows: number;
  errorRate: number;
  p95Ms: number;
  costPerRun: number;
  toolFailureRate: number;
  affectedUsers: number;
  releaseBlocking: boolean;
  reproducedInReplay: boolean;
  now: number;
}

export class BaselineDriftBreachWorker {
  constructor(private readonly store: BaselineOpsStore) {}

  decide(
    activation: BaselineActivationReceipt,
    lease: BaselineDriftSentinelLease,
    report: BaselineDriftBreachReport,
  ): DriftDecision {
    const sameCase =
      activation.caseId === lease.caseId &&
      activation.caseId === report.caseId &&
      activation.baselineVersion === lease.baselineVersion &&
      activation.baselineVersion === report.baselineVersion;

    if (!sameCase) return "manual_review";
    if (report.now > lease.expiresAt) return "manual_review";
    if (report.observedWindows < lease.minWindows) return "extend_sentinel";

    const breached =
      report.errorRate > lease.maxErrorRate ||
      report.p95Ms > lease.maxP95Ms ||
      report.costPerRun > lease.maxCostPerRun ||
      report.toolFailureRate > lease.maxToolFailureRate;

    if (!breached) return "ignore_noise";

    if (report.releaseBlocking && activation.rollbackBaselineVersion) {
      return "rollback_baseline";
    }

    if (report.affectedUsers > 0 && report.reproducedInReplay) {
      return "open_regression_case";
    }

    if (report.affectedUsers > 0) return "reopen_observation";

    return "extend_sentinel";
  }

  async handle(report: BaselineDriftBreachReport) {
    return this.store.transaction(async (tx) => {
      const activation = await tx.getActivation(report.caseId);
      const lease = await tx.getSentinelLease(report.caseId);
      const decision = this.decide(activation, lease, report);

      const receipt = await tx.createDriftCloseout({
        caseId: report.caseId,
        baselineVersion: report.baselineVersion,
        decision,
        reason: `${decision}: windows=${report.observedWindows}, p95=${report.p95Ms}`,
      });

      if (decision === "rollback_baseline") {
        await tx.activateBaseline(activation.rollbackBaselineVersion!, {
          reason: "sentinel_breach",
          receiptId: receipt.receiptId,
        });
      }

      if (decision === "open_regression_case") {
        await tx.enqueueRegressionCase({
          caseId: report.caseId,
          baselineVersion: report.baselineVersion,
          evidenceReceiptId: receipt.receiptId,
        });
      }

      if (decision === "reopen_observation") {
        await tx.reopenObservationWindow({
          caseId: report.caseId,
          receiptId: receipt.receiptId,
        });
      }

      return receipt;
    });
  }
}
~~~

注意 `handle` 里所有动作都在事务里绑定 receipt。否则最危险的情况是：基线已经回滚了，但没有 closeout receipt；或者 regression case 已经开了，但证据找不到。生产 Agent 的控制面动作必须能反查“为什么发生”。

## OpenClaw：Cron 哨兵怎么落地

OpenClaw 里可以把 Drift Sentinel 做成一个短周期 Cron：

~~~text
every 15 minutes:
  1. 读取最新 BaselineActivationReceipt
  2. 拉取过去 N 个窗口的 tool error / p95 / cost / affected user
  3. 生成 BaselineDriftBreachReport
  4. 调用 BaselineDriftBreachWorker
  5. 如果 decision 不是 ignore_noise，把 receipt 写入 ops log / Telegram 摘要
~~~

实际工程建议：

- 哨兵只监控刚激活的新基线，不要永久跑成另一个全局告警系统；
- 回滚只回滚 baseline/config，不要顺手回滚业务代码，除非 release blocker 明确指向代码变更；
- open_regression_case 要带 replay 证据，避免把不可复现的尖刺塞进长期修复队列；
- extend_sentinel 要设置上限，连续延长 3 次还不能判断，就转 manual_review。

## 常见坑

1. **把瞬时尖刺当基线失败**：没有连续窗口就回滚，会让系统反复震荡。
2. **只告警不落 receipt**：事后没人知道为什么回滚、谁触发、影响范围多大。
3. **把成本漂移当低优先级**：Agent 系统里成本飙升常常意味着循环、重试风暴或缓存失效，不能只看错误率。
4. **哨兵永不过期**：短期保护变成长期噪音，团队最后会忽略它。

## 记忆点

事故恢复链路不是：

~~~text
修复 -> 观察 -> 稳定 -> 结束
~~~

而是：

~~~text
修复 -> 观察 -> 稳定 -> 新基线 -> 漂移哨兵 -> 违约处理 -> 关闭收据
~~~

成熟 Agent 的运维系统不会因为“新常态已激活”就停止怀疑。它会给新基线一个试用期，发现漂移时用证据驱动最小动作：能延长观察就不回滚，能开根因案件就不乱改配置，必须回滚时也要带 receipt。
