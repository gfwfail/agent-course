# Agent 抑制策略上线前的影响回放与回滚闸门

> Post-Calibration Suppression Policy Impact Replay & Rollback Gate

第 467 课讲了：SignalDebtSettlement 会暴露抑制策略太松、太紧、source 权重错误或窗口预算不合适，然后生成 CalibrationProposal，先跑 shadow，再决定是否上线。

今天继续往后走一步：**shadow policy 看起来更好，不等于可以直接上线。**

原因很简单：抑制策略管的是“哪些重复信号不要打扰人、哪些新风险必须重开”。它一旦误判，后果不是普通指标变差，而是：

- 该重开的事故被压住了
- 不该升级的重复信号重新轰炸 owner
- suppression window 被拉太长，真实复发被吞掉
- dedupe key 变宽，多个不同风险被合并成一个

所以成熟 Agent 需要一个 ImpactReplayGate：在策略正式接管生产前，用历史账本和影子流量做影响回放，证明新策略的差异在预算内，并提前准备回滚点。

## 核心模型

把策略发布拆成 6 个对象：

```text
CalibrationProposal
  -> ImpactReplayPlan
  -> ReplayDiffReport
  -> RollbackPoint
  -> PromotionPermit
  -> RollbackReceipt | PromotionReceipt
```

关键点：**上线许可不是从 proposal 直接来的，而是从 replay diff 和 rollback point 一起生成。**

## learn-claude-code：纯函数闸门

教学版可以先写一个很小的判定器，把历史 settlement 样本重新跑一遍。

```python
# learn_claude_code/suppression_policy_gate.py
from dataclasses import dataclass
from typing import Literal

Decision = Literal["suppress_duplicate", "reopen_replay", "manual_review"]
GateDecision = Literal["promote_canary", "extend_shadow", "reject_policy"]


@dataclass
class ReplaySample:
    sample_id: str
    old_decision: Decision
    new_decision: Decision
    severity: Literal["low", "medium", "high", "critical"]
    later_confirmed_risk: bool


@dataclass
class ReplayDiffReport:
    total: int
    false_suppressions: int
    false_reopens: int
    critical_changed: int
    changed_samples: list[str]


def build_replay_report(samples: list[ReplaySample]) -> ReplayDiffReport:
    false_suppressions = 0
    false_reopens = 0
    critical_changed = 0
    changed_samples: list[str] = []

    for sample in samples:
        changed = sample.old_decision != sample.new_decision
        if not changed:
            continue

        changed_samples.append(sample.sample_id)

        if sample.severity == "critical":
            critical_changed += 1

        if sample.new_decision == "suppress_duplicate" and sample.later_confirmed_risk:
            false_suppressions += 1

        if sample.new_decision == "reopen_replay" and not sample.later_confirmed_risk:
            false_reopens += 1

    return ReplayDiffReport(
        total=len(samples),
        false_suppressions=false_suppressions,
        false_reopens=false_reopens,
        critical_changed=critical_changed,
        changed_samples=changed_samples,
    )


def decide_promotion(report: ReplayDiffReport) -> GateDecision:
    if report.false_suppressions > 0:
        return "reject_policy"

    if report.critical_changed > 0:
        return "extend_shadow"

    if report.false_reopens > 3:
        return "extend_shadow"

    return "promote_canary"
```

这里的思想很朴素：宁可多打扰人，也不能把真实复发压掉。false_suppressions 是最硬的红线。

## pi-mono：发布前写 RollbackPoint

生产版不能只返回 promote_canary，还要在事务里写好回滚点。否则上线后发现误伤，系统知道“要回滚”，但不知道回滚到哪个策略、哪些 runtime 已加载、哪些 decision 要回补。

```ts
// packages/agent-runtime/src/suppression/ImpactReplayGate.ts
export type PromotionAction =
  | "promote_canary"
  | "extend_shadow"
  | "reject_policy";

export interface ReplayDiffReport {
  proposalId: string;
  basePolicyVersion: string;
  candidatePolicyVersion: string;
  sampleCount: number;
  falseSuppressions: number;
  falseReopens: number;
  criticalChanged: number;
  changedDecisionIds: string[];
}

export interface RollbackPoint {
  rollbackPointId: string;
  restorePolicyVersion: string;
  candidatePolicyVersion: string;
  affectedRuntimeGroups: string[];
  createdAt: string;
}

export interface PromotionPermit {
  permitId: string;
  proposalId: string;
  action: "promote_canary";
  candidatePolicyVersion: string;
  rollbackPointId: string;
  expiresAt: string;
}

export class ImpactReplayGate {
  constructor(private readonly store: SuppressionPolicyStore) {}

  async evaluate(report: ReplayDiffReport): Promise<PromotionAction> {
    if (report.falseSuppressions > 0) return "reject_policy";
    if (report.criticalChanged > 0) return "extend_shadow";
    if (report.falseReopens > 3) return "extend_shadow";
    return "promote_canary";
  }

  async createPermit(report: ReplayDiffReport): Promise<PromotionPermit | null> {
    const action = await this.evaluate(report);
    if (action !== "promote_canary") return null;

    return this.store.transaction(async (tx) => {
      const rollbackPoint = await tx.createRollbackPoint({
        restorePolicyVersion: report.basePolicyVersion,
        candidatePolicyVersion: report.candidatePolicyVersion,
        affectedRuntimeGroups: await tx.runtimeGroupsUsing(report.basePolicyVersion),
      });

      return tx.createPromotionPermit({
        proposalId: report.proposalId,
        action: "promote_canary",
        candidatePolicyVersion: report.candidatePolicyVersion,
        rollbackPointId: rollbackPoint.rollbackPointId,
        expiresAt: new Date(Date.now() + 30 * 60_000).toISOString(),
      });
    });
  }
}
```

注意这里的顺序：先评估 replay diff，再创建 rollback point，最后才创建一次性的 PromotionPermit。

## OpenClaw：Cron 实战怎么用

OpenClaw 里很多任务是 Always-on 的，比如课程 cron、监控 cron、heartbeat。抑制策略如果变了，最容易影响的也是这些长跑任务。

可以把每次策略上线前的动作写成一个固定流程：

```text
1. 读取最近 N 天 SuppressedSignalLedger
2. 用 old policy 和 candidate policy 双跑 decision
3. 生成 ReplayDiffReport
4. 如果 false_suppression > 0：reject_policy
5. 如果 critical_changed > 0：extend_shadow
6. 如果通过：写 RollbackPoint + PromotionPermit
7. canary 阶段持续记录 PolicyDecisionOutcome
8. 超预算时用 rollbackPointId 恢复旧策略，并写 RollbackReceipt
```

课程 cron 的例子：

- 重复 messageId 到达：应该 suppress
- 新 commit hash 与 README 不一致：不能 suppress，要 reopen
- Telegram 发出去了但 git push 失败：不能当重复噪音吞掉，要进入 reconciliation
- TOOLS.md 已讲内容存在但 lesson 文件缺失：要 reopen 修复，不是安静跳过

这类信号都适合放进 replay sample。策略上线前先问：新策略会不会把这些真实问题压掉？

## 工程判断

抑制策略的目标不是“少告警”，而是“少打扰但不漏风险”。

好的发布闸门至少有 4 条硬规则：

1. false_suppression = 0 才能晋级
2. critical 样本发生决策变化必须延长 shadow
3. 每次 promotion permit 都要绑定 rollback point
4. canary 期间任何误压真实风险都立即 rollback

## 记住

> Suppression policy 的上线，不是把新参数写进配置，而是证明新策略不会吞掉该重开的现实信号，并且一旦错了能按收据退回。

成熟 Agent 的策略学习，不是 shadow 通过就上线，而是 impact replay、rollback point、一次性 promotion permit 和 canary outcome 串成完整发布闭环。
