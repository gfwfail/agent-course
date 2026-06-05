# Agent 观察期后的稳定晋级与回滚债务关闭闸门

> Post-Observation Stable Promotion & Rollback Debt Closeout Gate

第 489 课讲了 Post-Unblock Observation：解除 release blocker 后，不要马上相信修复已经稳定，而是用真实生产信号观察 repeated failures、tool error、latency、cost 和 user impact。

但观察期通过以后也不能只写一句“稳定了”。很多团队会在事故恢复后留下隐形债务：临时 rollback permit 没过期、LKG pin 没解、额外采样没关闭、告警阈值还停在事故模式、回归 seed 没接回长期套件。系统表面恢复了，控制面却还处在半紧急状态。

所以第 490 课补上 **Post-Observation Stable Promotion & Rollback Debt Closeout Gate**：观察期结束后，要把生产稳定证据、回滚准备状态、临时保护、长期测试资产和 release lane 一次性对账，再决定 `promote_stable`、`extend_observation`、`close_with_debt`、`rollback_patch` 或 `manual_review`。

## 核心模型

把观察期关闭拆成 5 个对象：

~~~text
PostUnblockCloseoutReceipt
  -> StabilityPromotionReview
  -> RollbackDebtInventory
  -> LongTermGuardReentryPlan
  -> StablePromotionCloseoutReceipt
~~~

分工：

- PostUnblockCloseoutReceipt：上一课的观察期收据，记录观察窗口的结果；
- StabilityPromotionReview：稳定晋级评审，确认 failures、latency、cost、tool error 都在预算内；
- RollbackDebtInventory：回滚债务清单，检查 rollback permit、LKG pin、临时阈值、人工冻结是否还存在；
- LongTermGuardReentryPlan：长期保护接回计划，决定 regression seed、guard pattern、release lane 何时恢复正常；
- StablePromotionCloseoutReceipt：最终关闭收据，说明是稳定晋级、延长观察、带债关闭、回滚还是人工处理。

一句话：**观察期通过不是“事故结束”，而是进入最后一次控制面对账：哪些临时保护要解除，哪些证据要长期化，哪些债务必须留 owner。**

## learn-claude-code：稳定晋级判定纯函数

教学版先写一个纯函数，把观察期结果和债务清单转成明确决策。它没有 I/O，适合单测覆盖各种边界。

~~~python
# learn_claude_code/stable_promotion_gate.py
from dataclasses import dataclass
from typing import Literal

StablePromotionDecision = Literal[
    "promote_stable",
    "extend_observation",
    "close_with_debt",
    "rollback_patch",
    "manual_review",
]


@dataclass
class PostUnblockCloseoutReceipt:
    case_id: str
    decision: str
    commit_sha: str
    observed_minutes: int
    repeated_failures: int
    rollback_permit_id: str | None


@dataclass
class StabilityPromotionReview:
    case_id: str
    min_stable_minutes: int
    max_failure_count: int
    cost_budget_ok: bool
    latency_budget_ok: bool
    tool_error_budget_ok: bool
    owner_ack: bool


@dataclass
class RollbackDebtInventory:
    case_id: str
    active_rollback_permits: int
    lkg_pins: int
    temporary_thresholds: int
    release_freezes: int
    debt_owner: str | None


def decide_stable_promotion(
    closeout: PostUnblockCloseoutReceipt,
    review: StabilityPromotionReview,
    debt: RollbackDebtInventory,
) -> StablePromotionDecision:
    if closeout.case_id != review.case_id or closeout.case_id != debt.case_id:
        return "manual_review"

    if closeout.decision == "rollback_patch":
        return "rollback_patch"

    if closeout.decision not in {"promote_stable", "continue_observation"}:
        return "manual_review"

    if closeout.repeated_failures > review.max_failure_count:
        return "extend_observation"

    if closeout.observed_minutes < review.min_stable_minutes:
        return "extend_observation"

    if not (
        review.cost_budget_ok
        and review.latency_budget_ok
        and review.tool_error_budget_ok
    ):
        return "extend_observation"

    if not review.owner_ack:
        return "manual_review"

    unresolved_debt = (
        debt.active_rollback_permits
        + debt.lkg_pins
        + debt.temporary_thresholds
        + debt.release_freezes
    )

    if unresolved_debt == 0:
        return "promote_stable"

    if debt.debt_owner:
        return "close_with_debt"

    return "manual_review"
~~~

这个函数故意把 `close_with_debt` 留出来。现实系统里不是所有临时保护都能马上解除，比如外部 API 供应商还在降级，或者 release lane 还要等下一个窗口恢复。关键不是“必须零债务”，而是**每一笔债务都有 owner、TTL 和退出条件**。

## pi-mono：StablePromotionCloseoutWorker

生产版可以让 worker 消费 `PostUnblockCloseoutReceipt`，读取指标面板、rollback permit 表、release lock 表和 regression suite 索引，然后事务化写关闭收据。

~~~ts
// packages/agent-runtime/src/regression/StablePromotionCloseoutWorker.ts
export type StablePromotionDecision =
  | "promote_stable"
  | "extend_observation"
  | "close_with_debt"
  | "rollback_patch"
  | "manual_review";

export interface PostUnblockCloseoutReceipt {
  caseId: string;
  decision: "promote_stable" | "continue_observation" | "rollback_patch" | "reblock_suite" | "manual_review";
  commitSha: string;
  observedMinutes: number;
  repeatedFailures: number;
  rollbackPermitId?: string;
}

export interface StabilityPromotionReview {
  caseId: string;
  minStableMinutes: number;
  maxFailureCount: number;
  costBudgetOk: boolean;
  latencyBudgetOk: boolean;
  toolErrorBudgetOk: boolean;
  ownerAck: boolean;
}

export interface RollbackDebtInventory {
  caseId: string;
  activeRollbackPermits: number;
  lkgPins: number;
  temporaryThresholds: number;
  releaseFreezes: number;
  debtOwner?: string;
}

export interface StablePromotionCloseoutReceipt {
  receiptId: string;
  caseId: string;
  commitSha: string;
  decision: StablePromotionDecision;
  debtOwner?: string;
  reason: string;
  createdAt: string;
}

export class StablePromotionCloseoutWorker {
  constructor(private readonly store: RegressionStore) {}

  decide(
    closeout: PostUnblockCloseoutReceipt,
    review: StabilityPromotionReview,
    debt: RollbackDebtInventory,
  ): StablePromotionDecision {
    if (
      closeout.caseId !== review.caseId ||
      closeout.caseId !== debt.caseId
    ) {
      return "manual_review";
    }

    if (closeout.decision === "rollback_patch") {
      return "rollback_patch";
    }

    if (
      closeout.decision !== "promote_stable" &&
      closeout.decision !== "continue_observation"
    ) {
      return "manual_review";
    }

    if (closeout.repeatedFailures > review.maxFailureCount) {
      return "extend_observation";
    }

    if (closeout.observedMinutes < review.minStableMinutes) {
      return "extend_observation";
    }

    if (
      !review.costBudgetOk ||
      !review.latencyBudgetOk ||
      !review.toolErrorBudgetOk
    ) {
      return "extend_observation";
    }

    if (!review.ownerAck) {
      return "manual_review";
    }

    const unresolvedDebt =
      debt.activeRollbackPermits +
      debt.lkgPins +
      debt.temporaryThresholds +
      debt.releaseFreezes;

    if (unresolvedDebt === 0) {
      return "promote_stable";
    }

    return debt.debtOwner ? "close_with_debt" : "manual_review";
  }

  async close(input: {
    closeout: PostUnblockCloseoutReceipt;
    review: StabilityPromotionReview;
    debt: RollbackDebtInventory;
  }): Promise<StablePromotionCloseoutReceipt> {
    const decision = this.decide(input.closeout, input.review, input.debt);

    return this.store.transaction(async (tx) => {
      if (decision === "promote_stable") {
        await tx.clearRollbackPermits(input.closeout.caseId);
        await tx.unpinLkg(input.closeout.caseId);
        await tx.restoreReleaseLane(input.closeout.caseId);
        await tx.promoteRegressionSeeds(input.closeout.caseId);
      }

      if (decision === "extend_observation") {
        await tx.extendObservationWindow(input.closeout.caseId, {
          reason: "stable promotion budgets are not fully satisfied",
        });
      }

      if (decision === "close_with_debt") {
        await tx.createDebtLease(input.closeout.caseId, {
          owner: input.debt.debtOwner!,
          exitCriteria: "clear rollback debt and restore normal release lane",
        });
      }

      if (decision === "rollback_patch") {
        await tx.enqueueRollback(input.closeout.caseId, {
          permitId: input.closeout.rollbackPermitId,
          commitSha: input.closeout.commitSha,
        });
      }

      return tx.writeStablePromotionCloseout({
        receiptId: crypto.randomUUID(),
        caseId: input.closeout.caseId,
        commitSha: input.closeout.commitSha,
        decision,
        debtOwner: input.debt.debtOwner,
        reason: this.reasonFor(decision),
        createdAt: new Date().toISOString(),
      });
    });
  }

  private reasonFor(decision: StablePromotionDecision): string {
    switch (decision) {
      case "promote_stable":
        return "observation passed and rollback debt is cleared";
      case "extend_observation":
        return "stability budgets need more production evidence";
      case "close_with_debt":
        return "stable enough to close, but temporary controls remain leased";
      case "rollback_patch":
        return "observation closeout requested rollback";
      case "manual_review":
        return "case state is inconsistent or lacks accountable owner ack";
    }
  }
}
~~~

注意这里的 `promote_stable` 会同时做四件事：

- 清掉 rollback permit；
- 解开 LKG pin；
- 恢复 release lane；
- 把相关 regression seeds 接回长期套件。

如果只做第一件，系统可能会出现“没有回滚了，但发布仍然冻结”的半关闭状态。成熟 Agent 的恢复流程要同时看数据面、控制面和测试面。

## OpenClaw：课程 Cron 的类比

把 OpenClaw 的三小时课程 cron 当成一个小型 release pipeline：

~~~text
lesson markdown 写入
  -> README 目录更新
  -> TOOLS.md 已讲内容更新
  -> Telegram 群发送
  -> git commit/push
  -> 最终回执
~~~

如果 Telegram 已发送，但 git push 失败，这就是一种“外部副作用已发生、控制面未关闭”的债务。下一轮 cron 不能假装没发生过，否则会重复发课；也不能只看 git 状态，否则会漏掉群里已经发布的事实。

更稳的做法是给 cron 写一个 closeout receipt：

~~~json
{
  "job": "agent-course-cron",
  "lesson": 490,
  "telegramSent": true,
  "readmeUpdated": true,
  "toolsUpdated": true,
  "gitCommit": "abc123",
  "gitPushed": true,
  "rollbackDebt": [],
  "decision": "promote_stable"
}
~~~

如果 `telegramSent=true` 但 `gitPushed=false`，下一轮就应该进入 `close_with_debt` 或 `manual_review`：补提交、补 README、补 TOOLS，而不是重新讲同一课。

## 实战要点

1. 观察期结束后，不要只看生产指标，还要清点控制面债务。
2. `promote_stable` 必须同时恢复 release lane、清 permit、解 LKG、接回 regression seeds。
3. 允许 `close_with_debt`，但债务必须有 owner、TTL、退出条件和追踪收据。
4. 如果 closeout state、review state、debt state 的 case_id 对不上，直接 manual review。
5. 外部副作用场景尤其需要 closeout receipt，否则重试会变成重复通知、重复扣款、重复发版。

## 小结

第 489 课解决的是“修复上线后是否真的稳定”。第 490 课解决的是“确认稳定后，系统是否真的回到正常运行形态”。

成熟 Agent 的事故恢复，不是测试绿了、观察过了、群里说一句“恢复了”就结束；它要把临时保护、回滚权限、发布车道、长期测试和最终收据全部对齐。这样下一次发布、下一次 cron、下一次回归运行，才不会踩在上一次事故留下的隐形债务上。
