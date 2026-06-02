# Agent 抑制策略金丝雀结果关闭与回滚收据

> Suppression Policy Canary Outcome Closeout & Rollback Receipt

第 468 课讲了：抑制策略从 shadow 进入 canary 前，必须先做 Impact Replay，确认 false_suppression 为 0，并写好 RollbackPoint + PromotionPermit。

今天继续讲下一步：**拿到 PromotionPermit 以后，canary 不能靠“看起来没事”结束。**

抑制策略最危险的失败不是报错，而是安静地吞掉真实风险。比如：

- 新的 Telegram 发送失败被当成重复信号 suppress
- git push 失败后的补偿 ticket 被误判为旧噪音
- README 已更新但 lesson 文件缺失，被合并进同一个 dedupe key
- critical 信号在 canary 里样本太少，没有足够证据就全量上线

所以 canary 阶段必须持续记录 PolicyDecisionOutcome，最后由 Closeout Gate 判断：promote、extend_canary、rollback 或 manual_review。成熟 Agent 的策略发布，不是“permit 用完就上线”，而是“permit 消费后还要带结果收据关闭”。

## 核心模型

把 canary 收口拆成 5 个对象：

```text
PromotionPermit
  -> CanaryPolicyDecision
  -> PolicyDecisionOutcome
  -> CanaryOutcomeReport
  -> PromotionCloseoutReceipt | RollbackReceipt
```

这里最关键的是 **outcome lag**：当场 suppress 不代表正确，要等后续现实证明这个信号确实没有复发风险。

## learn-claude-code：结果关闭判定器

教学版先写一个纯函数：输入 canary outcome 汇总，输出下一步动作。

```python
# learn_claude_code/suppression_canary_closeout.py
from dataclasses import dataclass
from typing import Literal

Decision = Literal["suppress_duplicate", "reopen_replay", "manual_review"]
Outcome = Literal["correct", "false_suppression", "false_reopen", "unknown"]
CloseoutAction = Literal["promote_active", "extend_canary", "rollback_policy", "manual_review"]


@dataclass
class PolicyDecisionOutcome:
    decision_id: str
    decision: Decision
    outcome: Outcome
    severity: Literal["low", "medium", "high", "critical"]
    rollback_point_id: str


@dataclass
class CanaryOutcomeReport:
    total: int
    false_suppressions: int
    false_reopens: int
    unknown: int
    critical_total: int
    critical_unknown: int
    rollback_point_id: str


def build_outcome_report(outcomes: list[PolicyDecisionOutcome]) -> CanaryOutcomeReport:
    false_suppressions = 0
    false_reopens = 0
    unknown = 0
    critical_total = 0
    critical_unknown = 0
    rollback_point_id = outcomes[0].rollback_point_id if outcomes else ""

    for item in outcomes:
        if item.outcome == "false_suppression":
            false_suppressions += 1
        if item.outcome == "false_reopen":
            false_reopens += 1
        if item.outcome == "unknown":
            unknown += 1

        if item.severity == "critical":
            critical_total += 1
            if item.outcome == "unknown":
                critical_unknown += 1

    return CanaryOutcomeReport(
        total=len(outcomes),
        false_suppressions=false_suppressions,
        false_reopens=false_reopens,
        unknown=unknown,
        critical_total=critical_total,
        critical_unknown=critical_unknown,
        rollback_point_id=rollback_point_id,
    )


def decide_closeout(report: CanaryOutcomeReport) -> CloseoutAction:
    if report.false_suppressions > 0:
        return "rollback_policy"

    if report.critical_unknown > 0:
        return "extend_canary"

    if report.total < 200 or report.unknown / max(report.total, 1) > 0.1:
        return "extend_canary"

    if report.false_reopens > 5:
        return "manual_review"

    return "promote_active"
```

这段代码的优先级很明确：误压真实风险直接回滚；critical 样本没看清就延长；样本不足不急着上线；只有证据足够干净才 promote。

## pi-mono：事务化关闭或回滚

生产版不能只做判断，还要在同一个事务里更新 policy pointer、写 receipt、补偿已影响的 decision。

```ts
// packages/agent-runtime/src/suppression/CanaryOutcomeCloseout.ts
export type CloseoutAction =
  | "promote_active"
  | "extend_canary"
  | "rollback_policy"
  | "manual_review";

export interface CanaryOutcomeReport {
  permitId: string;
  candidatePolicyVersion: string;
  rollbackPointId: string;
  total: number;
  falseSuppressions: number;
  falseReopens: number;
  unknown: number;
  criticalUnknown: number;
  affectedDecisionIds: string[];
}

export interface CloseoutReceipt {
  receiptId: string;
  action: CloseoutAction;
  policyVersion: string;
  rollbackPointId: string;
  affectedDecisionIds: string[];
  createdAt: string;
}

export class CanaryOutcomeCloseout {
  constructor(private readonly store: SuppressionPolicyStore) {}

  decide(report: CanaryOutcomeReport): CloseoutAction {
    if (report.falseSuppressions > 0) return "rollback_policy";
    if (report.criticalUnknown > 0) return "extend_canary";
    if (report.total < 200 || report.unknown / Math.max(report.total, 1) > 0.1) {
      return "extend_canary";
    }
    if (report.falseReopens > 5) return "manual_review";
    return "promote_active";
  }

  async closeout(report: CanaryOutcomeReport): Promise<CloseoutReceipt> {
    const action = this.decide(report);

    return this.store.transaction(async (tx) => {
      if (action === "promote_active") {
        await tx.compareAndSwapActivePolicy({
          expected: "canary",
          nextVersion: report.candidatePolicyVersion,
        });
      }

      if (action === "rollback_policy") {
        const rollback = await tx.readRollbackPoint(report.rollbackPointId);
        await tx.restorePolicyVersion(rollback.restorePolicyVersion);

        await tx.enqueueDecisionRepair({
          reason: "suppression_policy_false_suppression",
          affectedDecisionIds: report.affectedDecisionIds,
        });
      }

      if (action === "extend_canary") {
        await tx.extendCanaryWindow({
          permitId: report.permitId,
          minAdditionalSamples: 200,
        });
      }

      return tx.createCloseoutReceipt({
        action,
        policyVersion: report.candidatePolicyVersion,
        rollbackPointId: report.rollbackPointId,
        affectedDecisionIds: report.affectedDecisionIds,
      });
    });
  }
}
```

注意 rollback 不是只把策略版本改回去。canary 已经影响过真实 decision，所以要把 affectedDecisionIds 放进修复队列，逐个检查是否需要 reopen、补发通知、恢复 ticket 或重新对账。

## OpenClaw：课程 Cron 怎么落地

拿 Agent 开发课程 cron 举例，suppression policy canary 应该收集这些 outcome：

```text
decision: suppress_duplicate
source: telegram_message_seen_again
outcome: correct

decision: suppress_duplicate
source: tools_topic_match
outcome: false_suppression
because: new lesson file missing but topic text looked similar

decision: reopen_replay
source: git_remote_hash_mismatch
outcome: correct

decision: manual_review
source: telegram_sent_but_commit_missing
outcome: correct
```

如果出现第二条，系统必须回滚新策略，因为它把“相似主题”误当成“重复完成”，会导致课没写完却安静跳过。

OpenClaw 实战流程可以这样固定：

```text
1. PromotionPermit 进入 canary，只影响低风险重复信号
2. 每次 suppression decision 写 CanaryPolicyDecision
3. 后续 reality probe 写 PolicyDecisionOutcome
4. 到窗口结束生成 CanaryOutcomeReport
5. false_suppression > 0：用 rollbackPointId 恢复旧策略
6. rollback 后把 affectedDecisionIds 送入 repair queue
7. 无误压、样本足够、critical 清晰：写 PromotionCloseoutReceipt
```

## 工程判断

抑制策略 canary 的核心指标不是“少发了多少告警”，而是“有没有压掉应该处理的真实风险”。

落地时记住 4 条规则：

1. false_suppression 是硬红线，出现一次就 rollback
2. unknown 不是成功，只是观察窗口还没结束
3. rollback 必须修复 canary 期间已影响的 decision
4. promote 必须写 closeout receipt，让后续审计知道为什么可以全量

一句话：**抑制策略的 canary closeout，不是发布流程的尾巴，而是证明“少打扰没有变成漏风险”的证据闸门。**
