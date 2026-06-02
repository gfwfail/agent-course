# Agent 主动抑制策略漂移监控与自动降级

> Active Suppression Policy Drift Monitor & Automatic Demotion

第 469 课讲了：抑制策略 canary 结束时，要用 CanaryOutcomeReport 证明没有 false_suppression，才能 promote_active。

今天继续讲 promote 之后的一步：**active 不代表永久正确。**

抑制策略最容易随时间漂移。昨天有效的 dedupe key，今天可能因为业务流程变了、消息模板变了、工具 receipt schema 变了，开始把真实风险压掉。比如：

- Telegram 发送失败的错误文案更新，旧策略仍把它当成重复噪音
- git push 的 remote hash 变了，但 suppression key 只看 commit message
- lesson 文件和 README 的对账规则升级，旧策略还按旧 receipt 判断完成
- critical 信号样本突然变少，不是系统稳定，而是被策略吞掉了

所以 promote_active 后还要有 Drift Monitor：持续采集 active decision 的现实结果，和 canary 期的基线比较，触发 keep_active、demote_canary、demote_shadow、rollback_policy 或 manual_review。

## 核心模型

把 active 后的监控拆成 5 个对象：

```text
PromotionCloseoutReceipt
  -> ActivePolicyDecision
  -> DecisionOutcomeProbe
  -> PolicyDriftReport
  -> DemotionReceipt | ActiveStabilityReceipt
```

这里最关键的是 **baseline drift**：不是看到一个坏结果才算漂移，而是 unknown ratio、false suppression、critical coverage、repair queue lag 相对 canary 基线持续变坏。

## learn-claude-code：漂移判定纯函数

教学版先写一个纯函数：输入 active 观察窗口和 canary 基线，输出策略动作。

```python
# learn_claude_code/active_suppression_drift.py
from dataclasses import dataclass
from typing import Literal

DriftAction = Literal[
    "keep_active",
    "demote_canary",
    "demote_shadow",
    "rollback_policy",
    "manual_review",
]


@dataclass
class PolicyBaseline:
    policy_version: str
    false_suppression_rate: float
    unknown_rate: float
    critical_coverage: float
    repair_lag_seconds: int


@dataclass
class ActiveObservationWindow:
    policy_version: str
    total: int
    false_suppressions: int
    unknown: int
    critical_total: int
    critical_outcomes_known: int
    repair_lag_seconds: int
    external_schema_changed: bool


def decide_demotion(
    baseline: PolicyBaseline,
    window: ActiveObservationWindow,
) -> DriftAction:
    if window.total < 50:
        return "keep_active"

    false_suppression_rate = window.false_suppressions / max(window.total, 1)
    unknown_rate = window.unknown / max(window.total, 1)
    critical_coverage = window.critical_outcomes_known / max(window.critical_total, 1)

    if window.false_suppressions > 0:
        return "rollback_policy"

    if window.external_schema_changed and unknown_rate > baseline.unknown_rate:
        return "demote_shadow"

    if critical_coverage < min(0.95, baseline.critical_coverage - 0.05):
        return "demote_canary"

    if unknown_rate > baseline.unknown_rate * 2 and unknown_rate > 0.15:
        return "demote_canary"

    if window.repair_lag_seconds > baseline.repair_lag_seconds * 3:
        return "manual_review"

    if false_suppression_rate > baseline.false_suppression_rate:
        return "demote_canary"

    return "keep_active"
```

这段代码有一个重要顺序：false_suppression 直接 rollback；schema 变了先降到 shadow；critical 看不清降到 canary；repair lag 暴涨交给人工判断。

## pi-mono：主动监控 Worker

生产版要把 drift action 写成一次原子降级，不允许多个 worker 同时把策略指针改乱。

```ts
// packages/agent-runtime/src/suppression/ActivePolicyDriftMonitor.ts
export type DriftAction =
  | "keep_active"
  | "demote_canary"
  | "demote_shadow"
  | "rollback_policy"
  | "manual_review";

export interface PolicyDriftReport {
  policyVersion: string;
  baselineReceiptId: string;
  windowId: string;
  total: number;
  falseSuppressions: number;
  unknownRate: number;
  criticalCoverage: number;
  repairLagSeconds: number;
  externalSchemaChanged: boolean;
  affectedDecisionIds: string[];
}

export interface DemotionReceipt {
  receiptId: string;
  action: DriftAction;
  fromPolicyVersion: string;
  nextMode: "active" | "canary" | "shadow" | "rollback" | "manual_review";
  affectedDecisionIds: string[];
  createdAt: string;
}

export class ActivePolicyDriftMonitor {
  constructor(private readonly store: SuppressionPolicyStore) {}

  async closeWindow(report: PolicyDriftReport): Promise<DemotionReceipt> {
    const action = this.decide(report);

    return this.store.transaction(async (tx) => {
      await tx.lockPolicyVersion(report.policyVersion);

      if (action === "rollback_policy") {
        const rollbackPoint = await tx.readRollbackPointFor(report.policyVersion);
        await tx.restorePolicyVersion(rollbackPoint.restorePolicyVersion);
        await tx.enqueueDecisionRepair({
          reason: "active_policy_false_suppression",
          affectedDecisionIds: report.affectedDecisionIds,
        });
      }

      if (action === "demote_canary") {
        await tx.setPolicyMode({
          policyVersion: report.policyVersion,
          mode: "canary",
          trafficPercent: 10,
        });
      }

      if (action === "demote_shadow") {
        await tx.setPolicyMode({
          policyVersion: report.policyVersion,
          mode: "shadow",
          trafficPercent: 0,
        });
      }

      if (action === "manual_review") {
        await tx.openPolicyReviewCase({
          policyVersion: report.policyVersion,
          reportId: report.windowId,
        });
      }

      return tx.createDemotionReceipt({
        action,
        fromPolicyVersion: report.policyVersion,
        nextMode: this.nextMode(action),
        affectedDecisionIds: report.affectedDecisionIds,
      });
    });
  }

  private decide(report: PolicyDriftReport): DriftAction {
    if (report.falseSuppressions > 0) return "rollback_policy";
    if (report.externalSchemaChanged && report.unknownRate > 0.1) return "demote_shadow";
    if (report.criticalCoverage < 0.95) return "demote_canary";
    if (report.unknownRate > 0.15) return "demote_canary";
    if (report.repairLagSeconds > 1800) return "manual_review";
    return "keep_active";
  }

  private nextMode(action: DriftAction): DemotionReceipt["nextMode"] {
    if (action === "rollback_policy") return "rollback";
    if (action === "demote_canary") return "canary";
    if (action === "demote_shadow") return "shadow";
    if (action === "manual_review") return "manual_review";
    return "active";
  }
}
```

注意这里的降级不是“改个配置”。rollback 要修复已影响 decision；demote_canary 要限制真实影响范围；demote_shadow 要停止真实压制，只保留影子对比；manual_review 要打开 case，不能只发日志。

## OpenClaw：课程 Cron 怎么落地

拿 Agent 开发课程 cron 举例，active suppression policy 每个窗口至少观察这些信号：

```text
active_decision: suppress_duplicate
source: telegram_message_seen_again
outcome_probe: correct

active_decision: suppress_duplicate
source: readme_entry_similar
outcome_probe: unknown
because: lesson file path changed but README title looked repeated

active_decision: suppress_duplicate
source: git_push_retry
outcome_probe: false_suppression
because: remote main did not contain current commit

active_decision: reopen_replay
source: tools_topic_missing
outcome_probe: correct
```

如果第三条出现，策略必须 rollback，并把 affectedDecisionIds 送进 repair queue，重新检查 Telegram message、lesson 文件、README、TOOLS.md 和 remote main。因为这不是“少打一条告警”，而是把没完成的外部副作用压成了安静成功。

OpenClaw 实战流程可以固定为：

```text
1. PromotionCloseoutReceipt 成为 active baseline
2. 每个 active decision 都写 ActivePolicyDecision
3. 后续 reality probe 写 DecisionOutcomeProbe
4. 每个窗口生成 PolicyDriftReport
5. false_suppression > 0：rollback + repair queue
6. schema_changed + unknown 上升：demote_shadow
7. critical coverage 下降：demote_canary
8. 稳定窗口写 ActiveStabilityReceipt
```

## 工程判断

抑制策略 active 后，最怕的不是它偶尔吵，而是它太安静。

落地时记住 4 条规则：

1. active policy 也必须持续写 outcome probe
2. 漂移判断要和 canary baseline 比，不只看绝对值
3. 降级要有 receipt，rollback 要有 repair queue
4. schema、模板、receipt 格式变化时，优先怀疑 suppression key 失效

一句话：**抑制策略上线不是结束，而是进入持续审计；成熟 Agent 要能发现自己的安静是否变成了漏风险。**
