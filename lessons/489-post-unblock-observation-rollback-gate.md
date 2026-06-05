# Agent 解除阻塞后的观察期与回滚闸门

> Post-Unblock Observation & Rollback Gate

第 488 课讲了 Root-Cause Repair Verification：修复合并以后，要用 affected/adjacent/smoke seed 的重放证据来关闭 root cause case，并精确解除 release blocker。

但解除阻塞不是故事结束。真正上线后，Agent 会遇到离线 replay 覆盖不到的环境差异：真实用户输入、真实工具延迟、真实权限状态、真实外部 API 抖动。一个修复在回归套件里绿了，不代表生产流量里一定稳。

所以第 489 课补上 **Post-Unblock Observation & Rollback Gate**：解除阻塞后先进入短观察期，用真实生产信号验证修复是否稳定；如果失败复发、成本异常、工具错误回升或用户影响扩大，就自动触发 rollback、reblock 或 manual review。

## 核心模型

把解除阻塞后的闭环拆成 5 个对象：

~~~text
RootCauseCloseoutReceipt
  -> PostUnblockObservationWindow
  -> ProductionSignalSnapshot
  -> RollbackReadinessReview
  -> PostUnblockCloseoutReceipt
~~~

分工：

- RootCauseCloseoutReceipt：上一课的关闭收据，证明 blocker 已解除；
- PostUnblockObservationWindow：观察窗口，定义监控多久、看哪些 risk surface、阈值是什么；
- ProductionSignalSnapshot：生产信号快照，包含 failure recurrence、tool error、latency、cost、user impact；
- RollbackReadinessReview：判断继续观察、正式关闭、重新阻塞还是回滚；
- PostUnblockCloseoutReceipt：最终收据，记录 promote_stable、continue_observation、reblock_suite、rollback_patch 或 manual_review。

一句话：**解除阻塞以后，不要马上相信系统已经好了；先让真实流量给修复打一段时间的分。**

## learn-claude-code：观察期判定纯函数

教学版先写纯函数，只把 closeout receipt 和生产信号转成决策，方便单测。

~~~python
# learn_claude_code/post_unblock_gate.py
from dataclasses import dataclass
from typing import Literal

ObservationDecision = Literal[
    "promote_stable",
    "continue_observation",
    "reblock_suite",
    "rollback_patch",
    "manual_review",
]


@dataclass
class RootCauseCloseoutReceipt:
    case_id: str
    cluster_id: str
    commit_sha: str
    released_blocker: str
    released_at_unix: int
    rollback_plan_ref: str | None


@dataclass
class PostUnblockObservationWindow:
    case_id: str
    min_observation_minutes: int
    max_observation_minutes: int
    failure_budget: int
    max_tool_error_rate: float
    max_cost_multiplier: float
    max_p95_latency_ms: int


@dataclass
class ProductionSignalSnapshot:
    case_id: str
    observed_minutes: int
    repeated_failures: int
    affected_user_count: int
    tool_error_rate: float
    cost_multiplier: float
    p95_latency_ms: int
    rollback_probe_passed: bool


def decide_post_unblock(
    closeout: RootCauseCloseoutReceipt,
    window: PostUnblockObservationWindow,
    signal: ProductionSignalSnapshot,
) -> ObservationDecision:
    if closeout.case_id != window.case_id or closeout.case_id != signal.case_id:
        return "manual_review"

    if signal.repeated_failures > window.failure_budget:
        if closeout.rollback_plan_ref and signal.rollback_probe_passed:
            return "rollback_patch"
        return "reblock_suite"

    if signal.affected_user_count >= 10 and signal.repeated_failures > 0:
        return "reblock_suite"

    if signal.tool_error_rate > window.max_tool_error_rate:
        return "continue_observation"

    if signal.cost_multiplier > window.max_cost_multiplier:
        return "manual_review"

    if signal.p95_latency_ms > window.max_p95_latency_ms:
        return "continue_observation"

    if signal.observed_minutes < window.min_observation_minutes:
        return "continue_observation"

    if signal.observed_minutes > window.max_observation_minutes:
        return "manual_review"

    return "promote_stable"
~~~

这里的判断顺序很重要：

- 失败复发优先级最高，超过 failure budget 就不能继续假装稳定；
- 有 rollback plan 且 rollback probe 通过，才能自动 rollback；
- 用户影响扩大时，即使失败次数不多，也要先 reblock suite；
- 成本异常通常不是单纯测试能解释的，进入 manual review 更稳；
- 延迟和工具错误可能是外部抖动，先 continue observation，避免误回滚。

## pi-mono：Observation Worker + rollback permit

生产版可以让 `PostUnblockObservationWorker` 订阅 closeout receipt，然后开一个短 TTL 的 observation window。它不直接删 case，而是写稳定收据或生成 rollback permit。

~~~ts
// packages/agent-runtime/src/regression/PostUnblockObservationWorker.ts
export type ObservationDecision =
  | "promote_stable"
  | "continue_observation"
  | "reblock_suite"
  | "rollback_patch"
  | "manual_review";

export interface RootCauseCloseoutReceipt {
  caseId: string;
  clusterId: string;
  commitSha: string;
  releasedBlocker: string;
  releasedAtUnix: number;
  rollbackPlanRef?: string;
}

export interface PostUnblockObservationWindow {
  caseId: string;
  minObservationMinutes: number;
  maxObservationMinutes: number;
  failureBudget: number;
  maxToolErrorRate: number;
  maxCostMultiplier: number;
  maxP95LatencyMs: number;
}

export interface ProductionSignalSnapshot {
  caseId: string;
  observedMinutes: number;
  repeatedFailures: number;
  affectedUserCount: number;
  toolErrorRate: number;
  costMultiplier: number;
  p95LatencyMs: number;
  rollbackProbePassed: boolean;
}

export interface PostUnblockCloseoutReceipt {
  receiptId: string;
  caseId: string;
  clusterId: string;
  decision: ObservationDecision;
  commitSha: string;
  rollbackPermitId?: string;
  reason: string;
  createdAt: string;
}

export class PostUnblockObservationWorker {
  constructor(private readonly store: RegressionStore) {}

  decide(
    closeout: RootCauseCloseoutReceipt,
    window: PostUnblockObservationWindow,
    signal: ProductionSignalSnapshot,
  ): ObservationDecision {
    if (
      closeout.caseId !== window.caseId ||
      closeout.caseId !== signal.caseId
    ) {
      return "manual_review";
    }

    if (signal.repeatedFailures > window.failureBudget) {
      return closeout.rollbackPlanRef && signal.rollbackProbePassed
        ? "rollback_patch"
        : "reblock_suite";
    }

    if (signal.affectedUserCount >= 10 && signal.repeatedFailures > 0) {
      return "reblock_suite";
    }

    if (signal.toolErrorRate > window.maxToolErrorRate) {
      return "continue_observation";
    }

    if (signal.costMultiplier > window.maxCostMultiplier) {
      return "manual_review";
    }

    if (signal.p95LatencyMs > window.maxP95LatencyMs) {
      return "continue_observation";
    }

    if (signal.observedMinutes < window.minObservationMinutes) {
      return "continue_observation";
    }

    if (signal.observedMinutes > window.maxObservationMinutes) {
      return "manual_review";
    }

    return "promote_stable";
  }

  async closeObservation(caseId: string): Promise<PostUnblockCloseoutReceipt> {
    return this.store.transaction(async (tx) => {
      const closeout = await tx.getRootCauseCloseout(caseId);
      const window = await tx.getObservationWindow(caseId);
      const signal = await tx.collectProductionSignal(caseId);
      const decision = this.decide(closeout, window, signal);

      const rollbackPermitId =
        decision === "rollback_patch"
          ? await tx.createRollbackPermit({
              caseId,
              commitSha: closeout.commitSha,
              rollbackPlanRef: closeout.rollbackPlanRef!,
              expiresInMinutes: 30,
            })
          : undefined;

      if (decision === "reblock_suite") {
        await tx.reblockRegressionSuite({
          caseId,
          clusterId: closeout.clusterId,
          reason: "post-unblock production recurrence",
        });
      }

      const receipt = await tx.writePostUnblockCloseout({
        caseId,
        clusterId: closeout.clusterId,
        commitSha: closeout.commitSha,
        decision,
        rollbackPermitId,
        reason: this.reason(decision, signal),
      });

      await tx.emitEvent("regression.post_unblock.closed", receipt);
      return receipt;
    });
  }

  private reason(decision: ObservationDecision, signal: ProductionSignalSnapshot): string {
    return [
      decision,
      `failures=${signal.repeatedFailures}`,
      `users=${signal.affectedUserCount}`,
      `toolError=${signal.toolErrorRate}`,
      `costMultiplier=${signal.costMultiplier}`,
      `p95=${signal.p95LatencyMs}`,
    ].join(" ");
  }
}
~~~

生产版的关键点：

- rollback permit 必须短 TTL，避免旧 permit 被误用；
- reblock suite 和写 closeout receipt 要在同一个事务里，避免状态分裂；
- observation closeout 要发事件，让 release dashboard、通知系统、审计日志都能同步；
- `collectProductionSignal` 应该从 trace、tool audit、cost attribution 和 user-impact 事件聚合，不要只看单一 metric。

## OpenClaw：课程 Cron 的类比

这个课程 Cron 也有类似逻辑：

1. 写 lesson 文件；
2. 更新 README 和 TOOLS 已讲列表；
3. git commit/push；
4. 发 Telegram；
5. 最后看发送、提交、推送是否都成功。

如果只做到第 3 步就说“完成”，其实还不够。因为课程的真实交付目标是群里收到内容，仓库也有记录。更稳的做法是把 Telegram delivery、git push、TOOLS 去重都看成 post-unblock observation 信号。

## 实战建议

- 给每个 release blocker 解除动作自动创建 observation window；
- observation window 要短，但不能为 0，常见是 30 分钟、1 小时、24 小时三档；
- rollback 不要只靠人工记忆，必须有 rollback plan ref 和 rollback probe；
- reblock 不是失败，是保护机制，说明修复证据和生产信号不一致；
- closeout receipt 要写清楚为什么稳定，后续事故复盘才能追证据。

## 小结

成熟 Agent 的回归系统不是“修复通过测试就永久放行”，而是：

~~~text
修复验证 -> 解除阻塞 -> 生产观察 -> 稳定关闭 / 重新阻塞 / 自动回滚
~~~

**解除阻塞只是把系统交回真实流量；观察期和回滚闸门，才是防止同一类事故第二次扩大。**
