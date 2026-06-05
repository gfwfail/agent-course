# Agent 常态运维基线重校准与漂移哨兵

> Normal Ops Baseline Recalibration & Drift Sentinel

第 490 课讲了 Stable Promotion Closeout：观察期通过以后，要对账 rollback permit、LKG pin、临时阈值、release freeze 和长期 regression seeds，确认事故恢复真的关闭。

但关闭事故不等于系统回到“正常”。事故期间我们经常会临时调高采样、放宽超时、降低并发、切换备用模型、增加人工审批、压低告警阈值。如果这些运行参数被长期保留，Agent 会慢慢进入一个奇怪状态：看起来安全，实际上成本更高、吞吐更低、告警更多，而且未来容量规划全部基于事故模式。

所以第 491 课补上 **Normal Ops Baseline Recalibration & Drift Sentinel**：稳定晋级后，重新计算常态基线，把 incident mode 的临时指标、阈值和采样策略转换成 normal mode 的可验证运行契约，并挂一个短期漂移哨兵，防止系统刚恢复就再次偏离。

## 核心模型

把常态基线重建拆成 5 个对象：

~~~text
StablePromotionCloseoutReceipt
  -> NormalOpsBaselineSnapshot
  -> BaselineRecalibrationPlan
  -> BaselineActivationReceipt
  -> BaselineDriftSentinelLease
~~~

分工：

- StablePromotionCloseoutReceipt：上一课的稳定关闭收据，证明事故已经进入 stable 或带债关闭；
- NormalOpsBaselineSnapshot：收集事故前、事故中、观察期后的延迟、成本、错误率、工具成功率、采样率；
- BaselineRecalibrationPlan：决定哪些阈值恢复旧基线，哪些吸收新现实，哪些继续保留短期保护；
- BaselineActivationReceipt：原子写入新基线，绑定版本、回滚点和适用 scope；
- BaselineDriftSentinelLease：短期哨兵，观察新基线是否马上失效，必要时 rollback_baseline 或 reopen_observation。

一句话：**事故关闭后不要让系统永远活在事故参数里；要把“临时恢复策略”转成“新的正常基线”，并用哨兵证明它真的稳定。**

## learn-claude-code：基线重校准纯函数

教学版先写一个纯函数。输入是稳定关闭收据、指标快照和保护债务，输出一个清晰决策。

~~~python
# learn_claude_code/normal_ops_baseline_gate.py
from dataclasses import dataclass
from typing import Literal

BaselineDecision = Literal[
    "activate_normal_baseline",
    "activate_with_sentinel",
    "extend_observation",
    "keep_incident_mode",
    "manual_review",
]


@dataclass
class StablePromotionCloseoutReceipt:
    case_id: str
    decision: str  # promote_stable | close_with_debt | extend_observation | rollback_patch
    commit_sha: str
    debt_owner: str | None


@dataclass
class NormalOpsBaselineSnapshot:
    case_id: str
    pre_incident_error_rate: float
    post_recovery_error_rate: float
    pre_incident_p95_ms: int
    post_recovery_p95_ms: int
    pre_incident_cost_per_run: float
    post_recovery_cost_per_run: float
    sample_count: int


@dataclass
class BaselineRecalibrationPlan:
    case_id: str
    min_sample_count: int
    max_error_rate_multiplier: float
    max_latency_multiplier: float
    max_cost_multiplier: float
    temporary_protections_remaining: int
    owner_ack: bool
    rollback_point_ready: bool


def decide_baseline_recalibration(
    closeout: StablePromotionCloseoutReceipt,
    snapshot: NormalOpsBaselineSnapshot,
    plan: BaselineRecalibrationPlan,
) -> BaselineDecision:
    if closeout.case_id != snapshot.case_id or closeout.case_id != plan.case_id:
        return "manual_review"

    if closeout.decision == "rollback_patch":
        return "keep_incident_mode"

    if closeout.decision == "extend_observation":
        return "extend_observation"

    if closeout.decision not in {"promote_stable", "close_with_debt"}:
        return "manual_review"

    if snapshot.sample_count < plan.min_sample_count:
        return "extend_observation"

    if not plan.owner_ack or not plan.rollback_point_ready:
        return "manual_review"

    error_ok = snapshot.post_recovery_error_rate <= (
        snapshot.pre_incident_error_rate * plan.max_error_rate_multiplier
    )
    latency_ok = snapshot.post_recovery_p95_ms <= (
        snapshot.pre_incident_p95_ms * plan.max_latency_multiplier
    )
    cost_ok = snapshot.post_recovery_cost_per_run <= (
        snapshot.pre_incident_cost_per_run * plan.max_cost_multiplier
    )

    if not (error_ok and latency_ok and cost_ok):
        return "keep_incident_mode"

    if plan.temporary_protections_remaining > 0 or closeout.debt_owner:
        return "activate_with_sentinel"

    return "activate_normal_baseline"
~~~

这里的关键不是“恢复事故前所有数字”。有些事故修复会让真实延迟略升、成本略升，或者为了安全保留更高采样率。重校准的目标是：**把变化显式化、版本化、可回滚，而不是让临时参数悄悄变成永久默认值。**

## pi-mono：BaselineRecalibrationWorker

生产版可以让 worker 消费 `StablePromotionCloseoutReceipt`，读取 metrics store、config registry 和 release ledger，生成一份可审计的 baseline activation receipt。

~~~ts
// packages/agent-runtime/src/ops/BaselineRecalibrationWorker.ts
export type BaselineDecision =
  | "activate_normal_baseline"
  | "activate_with_sentinel"
  | "extend_observation"
  | "keep_incident_mode"
  | "manual_review";

export interface StablePromotionCloseoutReceipt {
  caseId: string;
  decision:
    | "promote_stable"
    | "close_with_debt"
    | "extend_observation"
    | "rollback_patch";
  commitSha: string;
  debtOwner?: string;
}

export interface NormalOpsBaselineSnapshot {
  caseId: string;
  preIncidentErrorRate: number;
  postRecoveryErrorRate: number;
  preIncidentP95Ms: number;
  postRecoveryP95Ms: number;
  preIncidentCostPerRun: number;
  postRecoveryCostPerRun: number;
  sampleCount: number;
}

export interface BaselineRecalibrationPlan {
  caseId: string;
  minSampleCount: number;
  maxErrorRateMultiplier: number;
  maxLatencyMultiplier: number;
  maxCostMultiplier: number;
  temporaryProtectionsRemaining: number;
  ownerAck: boolean;
  rollbackPointReady: boolean;
}

export interface BaselineActivationReceipt {
  receiptId: string;
  caseId: string;
  baselineVersion: string;
  decision: BaselineDecision;
  commitSha: string;
  rollbackBaselineVersion?: string;
  sentinelLeaseId?: string;
  reason: string;
  createdAt: string;
}

export class BaselineRecalibrationWorker {
  constructor(private readonly store: OpsBaselineStore) {}

  decide(
    closeout: StablePromotionCloseoutReceipt,
    snapshot: NormalOpsBaselineSnapshot,
    plan: BaselineRecalibrationPlan,
  ): BaselineDecision {
    if (
      closeout.caseId !== snapshot.caseId ||
      closeout.caseId !== plan.caseId
    ) {
      return "manual_review";
    }

    if (closeout.decision === "rollback_patch") return "keep_incident_mode";
    if (closeout.decision === "extend_observation") return "extend_observation";

    if (
      closeout.decision !== "promote_stable" &&
      closeout.decision !== "close_with_debt"
    ) {
      return "manual_review";
    }

    if (snapshot.sampleCount < plan.minSampleCount) return "extend_observation";
    if (!plan.ownerAck || !plan.rollbackPointReady) return "manual_review";

    const errorOk =
      snapshot.postRecoveryErrorRate <=
      snapshot.preIncidentErrorRate * plan.maxErrorRateMultiplier;

    const latencyOk =
      snapshot.postRecoveryP95Ms <=
      snapshot.preIncidentP95Ms * plan.maxLatencyMultiplier;

    const costOk =
      snapshot.postRecoveryCostPerRun <=
      snapshot.preIncidentCostPerRun * plan.maxCostMultiplier;

    if (!errorOk || !latencyOk || !costOk) return "keep_incident_mode";

    if (plan.temporaryProtectionsRemaining > 0 || closeout.debtOwner) {
      return "activate_with_sentinel";
    }

    return "activate_normal_baseline";
  }

  async run(closeout: StablePromotionCloseoutReceipt): Promise<BaselineActivationReceipt> {
    const snapshot = await this.store.collectBaselineSnapshot(closeout.caseId);
    const plan = await this.store.loadRecalibrationPlan(closeout.caseId);
    const decision = this.decide(closeout, snapshot, plan);

    return this.store.transaction(async tx => {
      const rollbackBaseline = await tx.getActiveBaselineVersion();
      const baselineVersion =
        decision === "activate_normal_baseline" ||
        decision === "activate_with_sentinel"
          ? await tx.publishBaseline({
              caseId: closeout.caseId,
              snapshot,
              plan,
              previousVersion: rollbackBaseline,
            })
          : rollbackBaseline;

      const sentinelLeaseId =
        decision === "activate_with_sentinel"
          ? await tx.createDriftSentinelLease({
              caseId: closeout.caseId,
              baselineVersion,
              rollbackBaselineVersion: rollbackBaseline,
              expiresInHours: 24,
            })
          : undefined;

      return tx.writeBaselineActivationReceipt({
        receiptId: crypto.randomUUID(),
        caseId: closeout.caseId,
        baselineVersion,
        decision,
        commitSha: closeout.commitSha,
        rollbackBaselineVersion: rollbackBaseline,
        sentinelLeaseId,
        reason: this.reason(decision),
        createdAt: new Date().toISOString(),
      });
    });
  }

  private reason(decision: BaselineDecision): string {
    switch (decision) {
      case "activate_normal_baseline":
        return "post-recovery metrics fit normal baseline budget";
      case "activate_with_sentinel":
        return "baseline accepted with remaining temporary protections";
      case "extend_observation":
        return "insufficient stable samples for baseline recalibration";
      case "keep_incident_mode":
        return "post-recovery metrics still exceed normal baseline budget";
      case "manual_review":
        return "baseline evidence or owner acknowledgement is incomplete";
    }
  }
}
~~~

几个生产细节：

- baseline 写入必须有 `rollbackBaselineVersion`，否则新基线坏了没法快速退回；
- `activate_with_sentinel` 不等于失败，它适合还有短期保护债务但已经可以恢复常态运行的场景；
- sentinel lease 要有明确 TTL，避免“短期观察”变成永久高频采样；
- cost baseline 要和 latency/error 一起看，因为事故后最容易留下的隐形问题就是“多花钱保稳定”。

## OpenClaw 课程 Cron 实战

这门课的自动发布 cron 本身就适合套这个模型。

一次成功发布后，不能只看 `git push` 通过：

- Git 基线：当前分支、远端 commit、README 目录和 lesson 文件一致；
- 消息基线：Telegram messageId 已记录，内容没有重复主题；
- 记忆基线：TOOLS.md 的已讲内容已更新；
- 运行基线：下一次 cron 不应该因为上次半完成状态重复发课；
- 哨兵租约：下一轮 heartbeat 或 cron 可以检查 repo、TOOLS 和群消息是否仍然对齐。

这就是 OpenClaw 的常态基线：**文件、Git、外部消息、记忆和调度状态都要收敛**。只要其中一个没对齐，下次就不应该盲目“继续讲下一课”，而是先对账。

## 常见坑

1. **把事故期间的阈值当成新常态**
   临时调高 timeout、采样率、审批等级很常见，但必须有 TTL 和 owner。

2. **只看错误率，不看成本**
   错误率降了但成本翻倍，说明系统可能靠过量重试或更贵模型撑住。

3. **没有 rollback baseline**
   新基线一旦造成告警失明或过度告警，没有旧版本可回滚，只能手动猜。

4. **样本量太小就重校准**
   低流量窗口的“稳定”不可靠，至少要满足 sampleCount 和时间窗口。

5. **哨兵没有退出条件**
   Sentinel 必须能 close、extend、rollback 或 reopen，不能永远挂着。

## 小结

Stable closeout 是事故关闭，baseline recalibration 是常态恢复。

成熟 Agent 的事故恢复不是“问题解决了就回去睡觉”，而是把事故期间的临时参数逐项清账：该恢复的恢复，该吸收的新现实版本化，该继续观察的挂短期哨兵，该回滚的保留回滚点。这样系统才不会在每次事故后都悄悄变重、变慢、变贵。

