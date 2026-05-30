# 441. Agent 护栏观察窗口后的晋级许可与跨阶段交接收据

> **核心思想：ObservationWindow 通过只说明当前阶段稳定，不等于下一阶段已经可以自动生效。成熟 Agent 要把 ObservationDecision 转成一次性的 PromotionPermit，再用 CrossStageHandoffReceipt 证明 scope、预算、rollback chain、runtime fingerprint 和发布租约都被正确交接。**

---

## 1. 为什么不能观察通过就直接晋级

上一课讲了 Post-Activation ObservationWindow：shadow、canary、active 恢复都要先观察，再决定 promote、extend、rollback 或 reopen。

但很多系统会在这里犯一个隐蔽错误：观察窗口显示 green，就直接把 candidate 从 shadow 推到 canary，或从 canary 推到 active。

这会丢掉四类关键证据：

1. 当前阶段观察通过时的证据是否被完整绑定到下一阶段；
2. 下一阶段 scope 是否被偷偷放宽；
3. 下一阶段预算是否继承、收紧或重新声明；
4. rollback point 是否形成连续链，而不是每阶段各自为政。

所以 ObservationDecision 不能直接改 runtime pointer。它只能生成 PromotionPermit。真正进入下一阶段时，发布 worker 必须消费这张 permit，并写 CrossStageHandoffReceipt。

正确链路是：

- ObservationDecision：当前窗口是否稳定；
- PromotionPermit：允许一次具体晋级；
- CrossStageHandoffPlan：下一阶段要如何接手；
- CrossStageHandoffReceipt：交接已经按计划完成；
- StageActivationReceipt：下一阶段正式激活；
- ObservationWindow：下一阶段继续观察。

一句话：观察通过是“可以申请晋级”，不是“已经晋级”。

---

## 2. 最小数据模型

~~~text
PromotionPermit:
  permitId
  sourceObservationDecisionId
  candidateVersion
  fromStage: shadow | canary
  toStage: canary | active
  allowedScope:
    tenants[]
    riskSurfaces[]
    actionClasses[]
  requiredEvidenceRefs[]
  inheritedRollbackChain[]
  maxTrafficShare
  expiresAt
  consumedAt?

CrossStageHandoffPlan:
  planId
  permitId
  nextStage
  nextScope
  nextBudgets
  runtimeFingerprint
  activationLeaseKey
  rollbackPointRef
  observationTemplateRef

CrossStageHandoffReceipt:
  receiptId
  permitId
  planId
  candidateVersion
  fromStage
  toStage
  scopeHash
  budgetHash
  runtimeFingerprint
  rollbackChainHash
  activationLeaseId
  nextObservationWindowId
  emittedAt
~~~

这里有两个重点。

第一，allowedScope 是上限，不是建议。下一阶段的 nextScope 必须是 allowedScope 的子集。canary 到 active 可以扩大流量份额，但不能扩大风险面，除非重新走 unlock review。

第二，inheritedRollbackChain 要串起来。shadow 的 rollback 证据、canary 的 rollback point、active 的 emergency fallback 不是三套孤立机制，而是一条可以审计的恢复链。

---

## 3. learn-claude-code：纯函数交接闸门

教学版先把交接规则写成纯函数。输入 permit 和 handoff plan，输出 accept 或 reject reason。

~~~py
from dataclasses import dataclass
from typing import Literal


Stage = Literal["shadow", "canary", "active"]
Decision = Literal["accept_handoff", "reject_handoff", "reopen_unlock_review"]


@dataclass(frozen=True)
class Scope:
    tenants: set[str]
    risk_surfaces: set[str]
    action_classes: set[str]


@dataclass(frozen=True)
class PromotionPermit:
    permit_id: str
    candidate_version: str
    from_stage: Stage
    to_stage: Stage
    allowed_scope: Scope
    inherited_rollback_chain: list[str]
    max_traffic_share: float
    expires_at_epoch: int
    consumed: bool


@dataclass(frozen=True)
class HandoffPlan:
    candidate_version: str
    from_stage: Stage
    to_stage: Stage
    next_scope: Scope
    next_traffic_share: float
    runtime_fingerprint: str
    expected_runtime_fingerprint: str
    rollback_point_ref: str | None
    now_epoch: int


@dataclass(frozen=True)
class HandoffGateResult:
    decision: Decision
    reason: str


def is_subset(child: set[str], parent: set[str]) -> bool:
    return child.issubset(parent)


def scope_within(plan: Scope, allowed: Scope) -> bool:
    return (
        is_subset(plan.tenants, allowed.tenants)
        and is_subset(plan.risk_surfaces, allowed.risk_surfaces)
        and is_subset(plan.action_classes, allowed.action_classes)
    )


def evaluate_handoff(
    permit: PromotionPermit,
    plan: HandoffPlan,
) -> HandoffGateResult:
    if permit.consumed:
        return HandoffGateResult("reject_handoff", "permit_already_consumed")

    if plan.now_epoch > permit.expires_at_epoch:
        return HandoffGateResult("reopen_unlock_review", "permit_expired")

    if permit.candidate_version != plan.candidate_version:
        return HandoffGateResult("reject_handoff", "candidate_version_mismatch")

    if (permit.from_stage, permit.to_stage) != (plan.from_stage, plan.to_stage):
        return HandoffGateResult("reject_handoff", "stage_transition_mismatch")

    if not scope_within(plan.next_scope, permit.allowed_scope):
        return HandoffGateResult("reopen_unlock_review", "next_scope_exceeds_permit")

    if plan.next_traffic_share > permit.max_traffic_share:
        return HandoffGateResult("reject_handoff", "traffic_share_exceeds_permit")

    if plan.runtime_fingerprint != plan.expected_runtime_fingerprint:
        return HandoffGateResult("reopen_unlock_review", "runtime_fingerprint_changed")

    if not plan.rollback_point_ref:
        return HandoffGateResult("reject_handoff", "missing_next_stage_rollback_point")

    if not permit.inherited_rollback_chain:
        return HandoffGateResult("reject_handoff", "missing_inherited_rollback_chain")

    return HandoffGateResult("accept_handoff", "handoff_constraints_satisfied")
~~~

这段代码刻意没有写数据库、队列或 LLM。核心规则必须先能被纯函数复现，否则生产问题来了很难判断是模型错、worker 错，还是状态机错。

---

## 4. pi-mono：事件驱动的晋级 worker

生产版可以把 PromotionPermit 当成一次性事件许可，由 worker 原子消费。

~~~ts
type Stage = "shadow" | "canary" | "active";

type Scope = {
  tenants: string[];
  riskSurfaces: string[];
  actionClasses: string[];
};

type PromotionPermit = {
  permitId: string;
  sourceObservationDecisionId: string;
  candidateVersion: string;
  fromStage: Stage;
  toStage: Stage;
  allowedScope: Scope;
  inheritedRollbackChain: string[];
  maxTrafficShare: number;
  expiresAt: string;
  consumedAt?: string;
};

type CrossStageHandoffReceipt = {
  receiptId: string;
  permitId: string;
  candidateVersion: string;
  fromStage: Stage;
  toStage: Stage;
  scopeHash: string;
  budgetHash: string;
  runtimeFingerprint: string;
  rollbackChainHash: string;
  activationLeaseId: string;
  nextObservationWindowId: string;
  emittedAt: string;
};

class PromotionPermitStore {
  async consumeOnce(permitId: string): Promise<PromotionPermit> {
    // 真实实现应使用 DB transaction:
    // update permits set consumed_at = now()
    // where permit_id = ? and consumed_at is null
    // returning *
    throw new Error("implement with transactional compare-and-set");
  }
}

class CrossStagePromotionWorker {
  constructor(
    private readonly permits: PromotionPermitStore,
    private readonly runtime: RuntimeFingerprintReader,
    private readonly leaseStore: ActivationLeaseStore,
    private readonly outbox: EventOutbox,
  ) {}

  async promote(permitId: string, requestedScope: Scope) {
    const permit = await this.permits.consumeOnce(permitId);
    const fingerprint = await this.runtime.current();

    const plan = buildHandoffPlan({
      permit,
      requestedScope,
      runtimeFingerprint: fingerprint,
    });

    const gate = evaluateHandoff(permit, plan);
    if (gate.decision !== "accept_handoff") {
      await this.outbox.publish("guard.handoff.rejected", {
        permitId,
        reason: gate.reason,
        decision: gate.decision,
      });
      return;
    }

    const lease = await this.leaseStore.acquire({
      key: `guard-restore:${permit.candidateVersion}:${permit.toStage}`,
      ttlSeconds: 900,
    });

    const nextWindow = createObservationWindow({
      activationLeaseId: lease.leaseId,
      stage: permit.toStage,
      candidateVersion: permit.candidateVersion,
      scope: requestedScope,
      rollbackChain: permit.inheritedRollbackChain,
    });

    const receipt: CrossStageHandoffReceipt = {
      receiptId: crypto.randomUUID(),
      permitId: permit.permitId,
      candidateVersion: permit.candidateVersion,
      fromStage: permit.fromStage,
      toStage: permit.toStage,
      scopeHash: hashJson(requestedScope),
      budgetHash: hashJson(nextWindow.budgets),
      runtimeFingerprint: fingerprint,
      rollbackChainHash: hashJson(permit.inheritedRollbackChain),
      activationLeaseId: lease.leaseId,
      nextObservationWindowId: nextWindow.windowId,
      emittedAt: new Date().toISOString(),
    };

    await this.outbox.publish("guard.handoff.receipt.created", receipt);
    await this.outbox.publish("guard.stage.activation.requested", {
      receiptId: receipt.receiptId,
      nextObservationWindowId: nextWindow.windowId,
    });
  }
}
~~~

注意这个 worker 不直接“相信”上一阶段结果。它只相信一次性 permit，而且必须重新检查 scope、fingerprint、rollback chain 和 lease。

---

## 5. OpenClaw 实战：课程 cron 也需要交接收据

OpenClaw 课程 cron 本身就是一个很好的类比：

- lesson 文件写完，不代表课程完成；
- README 更新，不代表群消息已经发出；
- Telegram messageId 存在，不代表 Git 已经推上远端；
- git commit 成功，不代表 TOOLS.md 的已讲内容已经同步。

所以每次发课都应该形成一个小型 handoff receipt：

~~~text
CoursePublishHandoffReceipt:
  lessonFile
  readmeEntry
  telegramMessageId
  toolsTopicEntry
  gitCommit
  remoteHead
  emittedAt
~~~

这个 receipt 能回答一个最实际的问题：如果 cron 中途重跑，它应该从哪里继续，哪些副作用不能重复？

对应到护栏恢复系统里也是一样：

- PromotionPermit 防止同一个观察结果重复晋级；
- HandoffReceipt 防止“以为交接了但没人能证明”；
- nextObservationWindowId 防止下一阶段上线后没人继续观察；
- remote/runtime head 防止交接时依赖已经漂移。

成熟 Agent 的副作用链路，不能只记录“做了什么”，还要记录“做完以后谁接手继续证明”。

---

## 6. 工程检查清单

做跨阶段晋级时，可以用这 8 条检查：

1. ObservationDecision 是否只生成 permit，没有直接改 runtime pointer；
2. PromotionPermit 是否一次性消费，重复消费会被拒绝；
3. nextScope 是否严格包含在 allowedScope 内；
4. trafficShare 是否不超过 permit 上限；
5. runtimeFingerprint 是否和 permit 生成时一致；
6. rollbackChain 是否继承上一阶段；
7. 下一阶段 ObservationWindow 是否在激活前创建；
8. HandoffReceipt 是否进入 outbox，后续 projector 可以恢复状态。

这套机制的价值不是让发布流程更慢，而是让每次晋级都有“可暂停、可重试、可审计、可回滚”的明确边界。

---

## 7. 小结

第 440 课解决的是：恢复阶段激活后，如何通过观察窗口判断稳定性。

这一课补上下一步：观察通过以后，不能直接晋级，而要生成一次性 PromotionPermit，并通过 CrossStageHandoffReceipt 完成跨阶段交接。

成熟 Agent 的护栏恢复不是一串 boolean 状态，而是一条收据链：permit、activation receipt、observation decision、promotion permit、handoff receipt、next activation receipt。链条越完整，系统越能在事故、重试和并发发布中保持可解释、可恢复、可控。
