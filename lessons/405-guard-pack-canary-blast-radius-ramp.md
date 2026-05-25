# 405. Agent 回归护栏包的金丝雀放量与爆炸半径限制（Guard Pack Canary Blast Radius Ramp）

上一课讲了 **Guard Pack Repair Replay Matrix & Shadow Re-entry Gate**：修复候选先用离线回放矩阵证明没有放宽安全，再进入只读 shadow，用真实流量差异预算决定能不能回到 canary。

今天继续讲 canary 之后最容易被忽略的一步：**即使 shadow 通过，也不能一下子让新 Guard Pack 接管全部生产流量。**

一句话：**Guard Pack canary 要按 tenant、risk surface、action class 和 traffic percentage 分阶段放量，每一阶段都绑定爆炸半径预算、自动回退触发器和晋级收据；成熟 Agent 不是“上线了再观察”，而是每一步都知道最多能影响谁、影响多少、什么时候必须停。**

---

## 1. 为什么 shadow 通过还不够

Shadow Re-entry 是只读跟跑：

~~~text
真实决策：旧 Guard Pack v42 生效
候选决策：新 Guard Pack v43 旁路计算
影响：不改变用户真实结果
~~~

它能发现很多差异，但不能覆盖所有真实副作用：

- 真实 block 会不会造成用户工单积压；
- 真实 allow 会不会放过某个外部副作用；
- 某些租户的权限模型和样本分布在 shadow 期间不够多；
- 下游系统对 guard decision 的处理可能和 shadow 记录不一致；
- 高峰期 latency、队列积压和人工复核容量会被真实流量放大。

所以 shadow 通过只代表“可以尝试接管一小部分真实流量”，不是“可以全量替换”。

这就是 **Canary Blast Radius Ramp**。

---

## 2. 最小模型：CanaryPlan

CanaryPlan 的核心不是百分比，而是 **blast radius**。

~~~ts
type CanaryPlan = {
  planId: string;
  packVersion: number;
  fromVersion: number;
  stages: CanaryStage[];
  globalAbort: AbortRule[];
};

type CanaryStage = {
  stageId: string;
  trafficPercent: number;
  tenantSelector: "internal" | "low_risk" | "allowlist" | "all";
  riskSurfaces: RiskSurface[];
  actionClasses: ActionClass[];
  durationMinutes: number;
  blastRadiusBudget: BlastRadiusBudget;
  promoteIf: PromoteRule[];
  rollbackIf: RollbackRule[];
};

type RiskSurface =
  | "message_egress"
  | "git_push"
  | "memory_write"
  | "credential_access"
  | "deployment"
  | "external_side_effect";

type ActionClass =
  | "readonly"
  | "dry_run"
  | "reversible_write"
  | "external_write"
  | "destructive_or_irreversible";

type BlastRadiusBudget = {
  maxAffectedTenants: number;
  maxBlockedActions: number;
  maxAllowedHighRiskActions: number;
  maxManualReviewsQueued: number;
  maxDecisionLatencyP95Ms: number;
};
~~~

几个关键点：

- **tenantSelector**：先 internal/low_risk，再 allowlist，最后 all；
- **riskSurfaces**：不要第一阶段就覆盖所有风险面；
- **actionClasses**：readonly/dry_run 可以先放，external_write 后放，irreversible 最后放；
- **blastRadiusBudget**：每一阶段必须知道最大可接受影响；
- **rollbackIf**：触发后自动回退，不等人肉发现。

如果一个 canary plan 只有“10% -> 50% -> 100%”，那它其实没有安全边界，只是在赌。

---

## 3. learn-claude-code：纯函数放量闸门

教学版可以先写一个纯函数：输入当前阶段配置和观测指标，输出 promote、hold、rollback 或 abort。

~~~py
from dataclasses import dataclass
from typing import Literal


Decision = Literal["promote", "hold", "rollback", "abort"]


@dataclass(frozen=True)
class BlastRadiusBudget:
    max_affected_tenants: int
    max_blocked_actions: int
    max_allowed_high_risk_actions: int
    max_manual_reviews_queued: int
    max_decision_latency_p95_ms: int


@dataclass(frozen=True)
class CanaryMetrics:
    affected_tenants: int
    blocked_actions: int
    allowed_high_risk_actions: int
    manual_reviews_queued: int
    decision_latency_p95_ms: int
    false_negative_count: int
    evidence_gap_count: int
    sample_size: int
    min_sample_size: int


def decide_canary_stage(metrics: CanaryMetrics, budget: BlastRadiusBudget) -> dict:
    hard_failures = []

    if metrics.false_negative_count > 0:
        hard_failures.append("false_negative_detected")
    if metrics.allowed_high_risk_actions > budget.max_allowed_high_risk_actions:
        hard_failures.append("high_risk_allow_budget_exceeded")
    if metrics.affected_tenants > budget.max_affected_tenants:
        hard_failures.append("tenant_blast_radius_exceeded")

    if hard_failures:
        return {
            "decision": "rollback",
            "reason": "canary exceeded hard safety budget",
            "failures": hard_failures,
        }

    soft_failures = []
    if metrics.blocked_actions > budget.max_blocked_actions:
        soft_failures.append("blocked_action_budget_exceeded")
    if metrics.manual_reviews_queued > budget.max_manual_reviews_queued:
        soft_failures.append("manual_review_capacity_exceeded")
    if metrics.decision_latency_p95_ms > budget.max_decision_latency_p95_ms:
        soft_failures.append("latency_budget_exceeded")
    if metrics.evidence_gap_count > 0:
        soft_failures.append("evidence_gap_detected")

    if soft_failures:
        return {
            "decision": "hold",
            "reason": "canary needs more observation or scope tightening",
            "failures": soft_failures,
        }

    if metrics.sample_size < metrics.min_sample_size:
        return {
            "decision": "hold",
            "reason": "not enough production samples yet",
            "sampleSize": metrics.sample_size,
            "required": metrics.min_sample_size,
        }

    return {
        "decision": "promote",
        "reason": "canary stayed within blast radius budget",
    }
~~~

这里的规则故意区分 hard failure 和 soft failure：

- false negative、放过高风险动作、影响租户超预算：直接 rollback；
- manual_review 积压、latency 超标、证据缺口：先 hold，收紧 scope 或延长窗口；
- 样本不足：不能晋级，只能继续观察。

成熟的 Agent 发布系统里，**“没有事故”不等于“可以晋级”，样本量也必须够。**

---

## 4. pi-mono：把 canary 放量接进事件流

生产实现里，Guard Pack 的放量不应该散落在 cron、脚本和人工笔记里，而应该变成事件流中的状态机。

~~~ts
type GuardPackCanaryStarted = {
  type: "guard_pack.canary.started";
  planId: string;
  stageId: string;
  packVersion: number;
  fromVersion: number;
  tenantSelector: string;
  riskSurfaces: string[];
  actionClasses: string[];
  trafficPercent: number;
};

type GuardPackCanaryObserved = {
  type: "guard_pack.canary.observed";
  planId: string;
  stageId: string;
  sampleSize: number;
  affectedTenants: number;
  blockedActions: number;
  allowedHighRiskActions: number;
  manualReviewsQueued: number;
  decisionLatencyP95Ms: number;
  falseNegativeCount: number;
  evidenceGapCount: number;
};

type GuardPackCanaryReceipt = {
  type: "guard_pack.canary.receipt";
  planId: string;
  stageId: string;
  decision: "promote" | "hold" | "rollback" | "abort";
  nextStageId?: string;
  rollbackToVersion?: number;
  reason: string;
  evidenceRefs: string[];
  decidedAt: string;
};
~~~

执行器可以长这样：

~~~ts
class GuardPackCanaryController {
  constructor(
    private readonly eventBus: EventBus,
    private readonly metrics: GuardMetricsReader,
    private readonly guardPackStore: GuardPackStore,
  ) {}

  async evaluateStage(plan: CanaryPlan, stage: CanaryStage) {
    const observed = await this.metrics.readCanaryStage(plan.planId, stage.stageId);
    const receipt = decideCanaryStage(observed, stage.blastRadiusBudget);

    await this.eventBus.publish({
      type: "guard_pack.canary.receipt",
      planId: plan.planId,
      stageId: stage.stageId,
      decision: receipt.decision,
      reason: receipt.reason,
      evidenceRefs: observed.evidenceRefs,
      decidedAt: new Date().toISOString(),
    });

    if (receipt.decision === "rollback") {
      await this.guardPackStore.activate(plan.fromVersion, {
        reason: receipt.reason,
        sourcePlanId: plan.planId,
      });
    }

    if (receipt.decision === "promote") {
      await this.guardPackStore.advanceCanary(plan.planId, stage.stageId);
    }
  }
}
~~~

这段代码的重点不是类名，而是三个约束：

1. 观测来自统一 metrics reader，不靠人眼看日志；
2. 每次 promote/hold/rollback 都有 receipt；
3. rollback 激活的是明确的 fromVersion，不是“找一个旧版本试试”。

---

## 5. OpenClaw 实战：课程 Cron 的 Guard Pack 放量

拿我们这个课程 Cron 举例，涉及三个外部副作用：

- 写 lesson 文件；
- 发 Telegram 群消息；
- git commit/push。

如果我们给它上一个新的 Guard Pack，第一阶段不应该直接覆盖全部动作，而可以这样放量：

~~~yaml
planId: guardpack-course-cron-v405
fromVersion: 44
packVersion: 45
stages:
  - stageId: internal-readonly
    tenantSelector: internal
    trafficPercent: 100
    riskSurfaces: [memory_write]
    actionClasses: [readonly, dry_run]
    durationMinutes: 60
    blastRadiusBudget:
      maxAffectedTenants: 1
      maxBlockedActions: 2
      maxAllowedHighRiskActions: 0
      maxManualReviewsQueued: 1
      maxDecisionLatencyP95Ms: 250

  - stageId: telegram-dry-run
    tenantSelector: allowlist
    trafficPercent: 25
    riskSurfaces: [message_egress]
    actionClasses: [dry_run]
    durationMinutes: 180
    blastRadiusBudget:
      maxAffectedTenants: 1
      maxBlockedActions: 3
      maxAllowedHighRiskActions: 0
      maxManualReviewsQueued: 2
      maxDecisionLatencyP95Ms: 300

  - stageId: git-push-reversible
    tenantSelector: allowlist
    trafficPercent: 25
    riskSurfaces: [git_push]
    actionClasses: [reversible_write]
    durationMinutes: 180
    blastRadiusBudget:
      maxAffectedTenants: 1
      maxBlockedActions: 3
      maxAllowedHighRiskActions: 0
      maxManualReviewsQueued: 2
      maxDecisionLatencyP95Ms: 350
~~~

注意：Telegram 真发和 git push 都是外部副作用，所以 canary receipt 里必须记录 messageId、commitSha、remoteSha、policyDecision 和 rollbackToVersion。

---

## 6. 常见坑

**坑 1：只按百分比放量。**

10% 的流量如果刚好包含所有高风险部署动作，爆炸半径可能比 50% readonly 还大。放量维度必须包含 risk surface 和 action class。

**坑 2：rollback 触发器只看错误率。**

Guard Pack 的事故经常不是 500，而是错误 allow、错误 block、manual_review 风暴和证据缺口。只看 error rate 会漏掉真正的问题。

**坑 3：没有晋级收据。**

几天后没人知道为什么从 25% 到 50%。没有 receipt，就无法复盘，也无法证明当时的样本量和预算足够。

**坑 4：把 irreversible 动作放太早。**

删除资源、公开发布、转账、生产部署这类动作应该最后接入，并且通常需要额外人工审批或双护栏。

---

## 7. 记住这句话

**Guard Pack canary 不是把新护栏“慢慢打开”，而是把真实世界的最大损失先关进预算里。**

Shadow 证明候选包看起来合理；canary 证明它在小范围真实副作用里仍然可靠；blast radius budget 证明即使它错了，损失也被限制住。

成熟 Agent 的安全发布，不靠胆子大，而靠每一步都有边界、有样本、有收据、有回退。

