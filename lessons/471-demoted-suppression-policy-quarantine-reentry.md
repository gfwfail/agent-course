# Agent 降级抑制策略的隔离补偿与重入闸门

> Demoted Suppression Policy Quarantine, Compensation & Re-entry Gate

第 470 课讲了：active suppression policy 上线后还要持续监控漂移，一旦出现 false_suppression、unknown ratio 暴涨、critical coverage 下降，就要 demote 或 rollback。

今天继续讲降级之后的一步：**降级不是终点，隔离和重入才是闭环。**

很多系统做到 demote_canary 或 demote_shadow 就停了，但这会留下 3 个坑：

- 旧策略已经吞掉的信号没有补回来
- 新策略虽然降级了，但运行时缓存还可能继续执行旧版本
- 修复后的策略重新进入 shadow/canary 时，没有证明它不会再吞同类信号

所以 active suppression policy 被降级后，要创建 Quarantine Case：冻结旧策略的真实压制能力，扫描 affected decisions，补偿被吞掉的信号，然后通过 Re-entry Gate 控制它能不能重新进 shadow/canary。

## 核心模型

把降级后的恢复拆成 6 个对象：

```text
DemotionReceipt
  -> PolicyQuarantineCase
  -> AffectedDecisionScan
  -> SuppressionCompensationReceipt
  -> ReentryReadinessReport
  -> ReentryPermit | QuarantineCloseReceipt
```

这里最关键的是 **quarantine is stronger than demotion**：demotion 是策略模式变化，quarantine 是运行时约束。只要 quarantine 没关闭，任何 executor 都不能继续用这个 policy version 做真实 suppress。

## learn-claude-code：重入闸门纯函数

教学版先写一个纯函数：输入降级收据、补偿扫描结果和修复验证结果，输出下一步动作。

```python
# learn_claude_code/suppression_reentry_gate.py
from dataclasses import dataclass
from typing import Literal

ReentryAction = Literal[
    "keep_quarantined",
    "open_compensation",
    "allow_shadow_reentry",
    "allow_canary_reentry",
    "manual_review",
]


@dataclass
class DemotionReceipt:
    policy_version: str
    action: str
    false_suppressions: int
    affected_decision_count: int


@dataclass
class CompensationScan:
    scanned: int
    compensated: int
    unresolved: int
    runtime_cache_cleared: bool
    downstream_repaired: bool


@dataclass
class RepairValidation:
    replay_total: int
    replay_passed: int
    same_signature_failures: int
    shadow_unknown_rate: float
    critical_coverage: float


def decide_reentry(
    demotion: DemotionReceipt,
    scan: CompensationScan,
    validation: RepairValidation,
) -> ReentryAction:
    if not scan.runtime_cache_cleared:
        return "keep_quarantined"

    if scan.scanned < demotion.affected_decision_count:
        return "open_compensation"

    if scan.unresolved > 0 or not scan.downstream_repaired:
        return "open_compensation"

    if validation.same_signature_failures > 0:
        return "keep_quarantined"

    if validation.replay_total < 30:
        return "manual_review"

    pass_rate = validation.replay_passed / max(validation.replay_total, 1)

    if pass_rate < 0.98:
        return "keep_quarantined"

    if validation.shadow_unknown_rate > 0.10:
        return "allow_shadow_reentry"

    if validation.critical_coverage >= 0.98:
        return "allow_canary_reentry"

    return "allow_shadow_reentry"
```

这段代码故意把 runtime cache 放在第一位。因为策略指针降级了，但 worker 还拿着旧 policy snapshot，等于现实里没有真正隔离。

## pi-mono：隔离与重入 Worker

生产版要把 quarantine、compensation、reentry permit 做成事务，避免一个 worker 清缓存，另一个 worker 又把旧策略放回 canary。

```ts
// packages/agent-runtime/src/suppression/DemotedPolicyQuarantineWorker.ts
export type ReentryAction =
  | "keep_quarantined"
  | "open_compensation"
  | "allow_shadow_reentry"
  | "allow_canary_reentry"
  | "manual_review";

export interface PolicyQuarantineCase {
  caseId: string;
  policyVersion: string;
  sourceDemotionReceiptId: string;
  affectedDecisionIds: string[];
  quarantineToken: string;
}

export interface ReentryReadinessReport {
  caseId: string;
  policyVersion: string;
  scanned: number;
  compensated: number;
  unresolved: number;
  runtimeCacheCleared: boolean;
  downstreamRepaired: boolean;
  replayPassRate: number;
  sameSignatureFailures: number;
  shadowUnknownRate: number;
  criticalCoverage: number;
}

export interface ReentryPermit {
  permitId: string;
  policyVersion: string;
  mode: "shadow" | "canary";
  trafficPercent: number;
  expiresAt: string;
}

export class DemotedPolicyQuarantineWorker {
  constructor(private readonly store: SuppressionPolicyStore) {}

  async closeQuarantineWindow(report: ReentryReadinessReport) {
    const action = this.decide(report);

    return this.store.transaction(async (tx) => {
      const quarantine = await tx.lockQuarantineCase(report.caseId);

      await tx.assertPolicyStillQuarantined({
        policyVersion: report.policyVersion,
        quarantineToken: quarantine.quarantineToken,
      });

      if (action === "open_compensation") {
        await tx.enqueueSuppressedSignalCompensation({
          policyVersion: report.policyVersion,
          unresolved: report.unresolved,
          sourceCaseId: report.caseId,
        });
      }

      if (action === "allow_shadow_reentry") {
        return tx.createReentryPermit({
          policyVersion: report.policyVersion,
          mode: "shadow",
          trafficPercent: 0,
          expiresAt: this.hoursFromNow(6),
        });
      }

      if (action === "allow_canary_reentry") {
        return tx.createReentryPermit({
          policyVersion: report.policyVersion,
          mode: "canary",
          trafficPercent: 5,
          expiresAt: this.hoursFromNow(2),
        });
      }

      if (action === "manual_review") {
        await tx.openPolicyReviewCase({
          policyVersion: report.policyVersion,
          sourceCaseId: report.caseId,
          reason: "quarantine_reentry_needs_human_decision",
        });
      }

      return tx.createQuarantineCloseReceipt({
        caseId: report.caseId,
        policyVersion: report.policyVersion,
        action,
      });
    });
  }

  private decide(report: ReentryReadinessReport): ReentryAction {
    if (!report.runtimeCacheCleared) return "keep_quarantined";
    if (!report.downstreamRepaired || report.unresolved > 0) return "open_compensation";
    if (report.sameSignatureFailures > 0) return "keep_quarantined";
    if (report.replayPassRate < 0.98) return "keep_quarantined";
    if (report.shadowUnknownRate > 0.10) return "allow_shadow_reentry";
    if (report.criticalCoverage >= 0.98) return "allow_canary_reentry";
    return "manual_review";
  }

  private hoursFromNow(hours: number): string {
    return new Date(Date.now() + hours * 60 * 60 * 1000).toISOString();
  }
}
```

注意两个细节：

1. `assertPolicyStillQuarantined` 要带 quarantineToken，防止旧 worker 用过期状态写回。
2. reentry permit 必须短期、低流量、可过期，不能因为修了一次 replay 就直接恢复 active。

## OpenClaw：课程 Cron 怎么落地

拿 Agent 开发课程 cron 举例，假设上一课的 drift monitor 发现：

```text
policyVersion: suppression-v470
false_suppression: git_push_retry
because: remote main did not contain current commit
action: rollback_policy
```

降级后不能只把策略改回旧版，还要补 4 件事：

```text
1. PolicyQuarantineCase
   - 禁止 suppression-v470 继续真实 suppress git/telegram/readme/tools 信号

2. AffectedDecisionScan
   - 扫描被 suppression-v470 压掉的 git_push_retry、readme_entry_similar、telegram_message_seen_again

3. SuppressionCompensationReceipt
   - 对 remote main 重新 probe
   - 对 Telegram messageId 重新确认
   - 对 lessons/README/TOOLS.md 重新对账

4. ReentryReadinessReport
   - replay 同类 git push retry 样本
   - shadow 观察 README title 相似但 path 不同的样本
   - critical coverage 必须恢复到基线
```

只有这些证据齐了，策略才允许带 ReentryPermit 回到 shadow 或 5% canary。它不能直接回 active，因为 active 是生产压制权，不是修复后的默认状态。

## 工程判断

抑制策略被降级后，真正危险的是“以为已经安全了”。

落地时记住 5 条规则：

1. demotion 改模式，quarantine 管运行时
2. 降级后要扫描 affected decisions，不能只修新流量
3. 补偿收据要覆盖外部现实：消息、git、部署、文件、队列
4. 重入从 shadow/canary 开始，permit 要短期、低流量、可撤回
5. 同类 signature 只要 replay 还失败，就继续 quarantine
