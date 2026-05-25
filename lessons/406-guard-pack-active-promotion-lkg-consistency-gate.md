# 406. Agent 回归护栏包全量激活与 LKG 一致性闸门（Guard Pack Active Promotion & LKG Consistency Gate）

上一课讲了 **Guard Pack Canary Blast Radius Ramp**：Guard Pack 从 shadow 进入 canary 后，不能直接全量接管，要按 tenant、risk surface、action class 和 traffic percentage 分阶段放量，并用 blast radius budget 决定 promote、hold、rollback 或 abort。

今天继续讲 canary 之后的最后一公里：**canary 全部通过，也不能马上把新包写成 Last Known Good。**

一句话：**Guard Pack active promotion 要先证明新版本已在所有运行时一致生效、旧版本可回滚、LKG 更新可撤销，并经过 post-activation freeze window；成熟 Agent 不是“100% 流量没炸就结束”，而是能证明全量激活真的一致、可回滚、可审计。**

---

## 1. 为什么 100% canary 不等于 active 完成

很多系统会把发布想成这样：

~~~text
shadow 通过 -> canary 10% -> canary 50% -> canary 100% -> done
~~~

对普通 feature flag 来说，这已经不错。对 Agent Guard Pack 来说还不够，因为 Guard Pack 是安全契约：

- 多个 agent runtime 可能还缓存着旧版本；
- worker、cron、sub-agent、interactive session 的加载时机不同；
- 有些高风险 action 在 canary 窗口里没有真实样本；
- 回滚指针可能还指向更旧的包，而不是本次晋级前的 Last Known Good；
- LKG 一旦被覆盖，后面发现问题时可能失去可信回退点；
- 运行时日志显示 100% 流量命中，不代表每个风险面都用同一套 guard executor 解释规则。

所以 active promotion 至少要回答四个问题：

1. **一致性**：所有应该使用 v43 的 runtime 是否真的都在用 v43？
2. **覆盖性**：关键 risk surface 是否都产生了足够样本或明确豁免？
3. **回滚性**：v42 是否仍可作为 Last Known Good 立即恢复？
4. **稳定性**：全量激活后一段 freeze window 内，是否没有出现延迟、误挡、漏挡、人工复核积压或证据缺口？

这就是 **LKG Consistency Gate**。

---

## 2. 最小模型：ActivePromotionPlan

Active promotion 不是一个布尔值，而是一张发布收据。

~~~ts
type ActivePromotionPlan = {
  planId: string;
  candidateVersion: number;
  previousLkgVersion: number;
  requiredRuntimeGroups: RuntimeGroup[];
  requiredRiskSurfaces: RiskSurface[];
  freezeWindowMinutes: number;
  consistencyBudget: ConsistencyBudget;
  promoteIf: PromotionRule[];
  rollbackIf: RollbackRule[];
};

type RuntimeGroup =
  | "interactive_session"
  | "cron_worker"
  | "subagent_worker"
  | "tool_executor"
  | "message_egress"
  | "git_executor";

type RiskSurface =
  | "message_egress"
  | "git_push"
  | "memory_write"
  | "credential_access"
  | "deployment"
  | "external_side_effect";

type ConsistencyBudget = {
  minRuntimeCoveragePercent: number;
  maxVersionSkewSeconds: number;
  maxUnknownRuntimeCount: number;
  maxPolicyInterpretationMismatch: number;
  maxFalsePositiveCount: number;
  maxFalseNegativeCount: number;
  maxManualReviewQueueDepth: number;
  maxDecisionLatencyP95Ms: number;
};
~~~

注意这里有两个容易漏掉的字段：

- **maxVersionSkewSeconds**：允许少量 runtime 延迟加载，但必须有边界；
- **maxPolicyInterpretationMismatch**：同一个 Guard Pack 被不同 executor 解释出不同 decision，比版本没同步更危险。

Active promotion 的重点不是“把配置切到 active”，而是确认“所有执行者对 active 的理解一致”。

---

## 3. learn-claude-code：纯函数激活闸门

教学版可以先写纯函数：输入 promotion 观察结果，输出 promote_lkg、hold、rollback 或 abort。

~~~py
from dataclasses import dataclass
from typing import Literal


Decision = Literal["promote_lkg", "hold", "rollback", "abort"]


@dataclass(frozen=True)
class ConsistencyBudget:
    min_runtime_coverage_percent: float
    max_version_skew_seconds: int
    max_unknown_runtime_count: int
    max_policy_interpretation_mismatch: int
    max_false_positive_count: int
    max_false_negative_count: int
    max_manual_review_queue_depth: int
    max_decision_latency_p95_ms: int


@dataclass(frozen=True)
class ActivePromotionMetrics:
    runtime_coverage_percent: float
    version_skew_seconds: int
    unknown_runtime_count: int
    policy_interpretation_mismatch: int
    risk_surfaces_without_samples: list[str]
    approved_sample_exemptions: list[str]
    false_positive_count: int
    false_negative_count: int
    manual_review_queue_depth: int
    decision_latency_p95_ms: int
    rollback_probe_passed: bool
    lkg_write_is_atomic: bool
    freeze_window_elapsed: bool


def decide_active_promotion(
    metrics: ActivePromotionMetrics,
    budget: ConsistencyBudget,
) -> dict:
    hard_failures = []

    if metrics.false_negative_count > budget.max_false_negative_count:
        hard_failures.append("false_negative_budget_exceeded")
    if not metrics.rollback_probe_passed:
        hard_failures.append("rollback_probe_failed")
    if not metrics.lkg_write_is_atomic:
        hard_failures.append("lkg_write_not_atomic")
    if metrics.policy_interpretation_mismatch > budget.max_policy_interpretation_mismatch:
        hard_failures.append("policy_interpretation_mismatch")

    if hard_failures:
        return {
            "decision": "rollback",
            "reason": "active promotion failed a hard safety invariant",
            "failures": hard_failures,
        }

    missing_surfaces = [
        surface
        for surface in metrics.risk_surfaces_without_samples
        if surface not in metrics.approved_sample_exemptions
    ]

    soft_failures = []
    if metrics.runtime_coverage_percent < budget.min_runtime_coverage_percent:
        soft_failures.append("runtime_coverage_too_low")
    if metrics.version_skew_seconds > budget.max_version_skew_seconds:
        soft_failures.append("version_skew_too_high")
    if metrics.unknown_runtime_count > budget.max_unknown_runtime_count:
        soft_failures.append("unknown_runtime_count_too_high")
    if metrics.false_positive_count > budget.max_false_positive_count:
        soft_failures.append("false_positive_budget_exceeded")
    if metrics.manual_review_queue_depth > budget.max_manual_review_queue_depth:
        soft_failures.append("manual_review_queue_over_capacity")
    if metrics.decision_latency_p95_ms > budget.max_decision_latency_p95_ms:
        soft_failures.append("decision_latency_budget_exceeded")
    if missing_surfaces:
        soft_failures.append("risk_surface_sample_gap")
    if not metrics.freeze_window_elapsed:
        soft_failures.append("freeze_window_not_elapsed")

    if soft_failures:
        return {
            "decision": "hold",
            "reason": "active promotion needs more stability evidence",
            "failures": soft_failures,
            "missingRiskSurfaces": missing_surfaces,
        }

    return {
        "decision": "promote_lkg",
        "reason": "candidate is globally consistent and stable",
    }
~~~

这里的关键是：**LKG 更新是最后一步，不是开始步骤。**

只有当全局一致性、回滚探针、freeze window 都通过，才允许把 previousLkgVersion 从 v42 更新到 v43。

---

## 4. pi-mono：把 LKG 更新做成事务

生产系统里，Guard Pack active promotion 至少要写三类事件：

~~~ts
type GuardPackActiveObserved = {
  type: "guard_pack.active.observed";
  planId: string;
  candidateVersion: number;
  runtimeCoveragePercent: number;
  versionSkewSeconds: number;
  unknownRuntimeCount: number;
  policyInterpretationMismatch: number;
  riskSurfacesWithoutSamples: string[];
  rollbackProbePassed: boolean;
  freezeWindowElapsed: boolean;
};

type GuardPackLkgPromotionReceipt = {
  type: "guard_pack.lkg.promotion_receipt";
  planId: string;
  promotedVersion: number;
  previousLkgVersion: number;
  decision: "promote_lkg" | "hold" | "rollback" | "abort";
  reason: string;
  evidenceRefs: string[];
  decidedAt: string;
};

type GuardPackLkgPointerUpdated = {
  type: "guard_pack.lkg.pointer_updated";
  planId: string;
  oldLkgVersion: number;
  newLkgVersion: number;
  compareAndSwapToken: string;
  updatedAt: string;
};
~~~

执行器可以长这样：

~~~ts
class GuardPackActivePromotionController {
  constructor(
    private readonly eventBus: EventBus,
    private readonly metrics: GuardRuntimeMetrics,
    private readonly lkgStore: GuardPackLkgStore,
    private readonly runtimeRegistry: RuntimeRegistry,
  ) {}

  async evaluate(plan: ActivePromotionPlan) {
    const observed = await this.metrics.readActivePromotion(plan.planId);
    const decision = decideActivePromotion(observed, plan.consistencyBudget);

    await this.eventBus.publish({
      type: "guard_pack.lkg.promotion_receipt",
      planId: plan.planId,
      promotedVersion: plan.candidateVersion,
      previousLkgVersion: plan.previousLkgVersion,
      decision: decision.decision,
      reason: decision.reason,
      evidenceRefs: observed.evidenceRefs,
      decidedAt: new Date().toISOString(),
    });

    if (decision.decision === "rollback") {
      await this.runtimeRegistry.activateGuardPack(plan.previousLkgVersion, {
        sourcePlanId: plan.planId,
        reason: decision.reason,
      });
      return decision;
    }

    if (decision.decision !== "promote_lkg") {
      return decision;
    }

    const current = await this.lkgStore.readPointer();
    if (current.version !== plan.previousLkgVersion) {
      await this.eventBus.publish({
        type: "guard_pack.lkg.promotion_receipt",
        planId: plan.planId,
        promotedVersion: plan.candidateVersion,
        previousLkgVersion: plan.previousLkgVersion,
        decision: "abort",
        reason: "LKG pointer changed during active promotion",
        evidenceRefs: [current.evidenceRef],
        decidedAt: new Date().toISOString(),
      });
      return {
        decision: "abort",
        reason: "LKG pointer changed during active promotion",
      };
    }

    await this.lkgStore.compareAndSwap({
      expectedVersion: plan.previousLkgVersion,
      nextVersion: plan.candidateVersion,
      reason: decision.reason,
      planId: plan.planId,
    });

    await this.eventBus.publish({
      type: "guard_pack.lkg.pointer_updated",
      planId: plan.planId,
      oldLkgVersion: plan.previousLkgVersion,
      newLkgVersion: plan.candidateVersion,
      compareAndSwapToken: current.casToken,
      updatedAt: new Date().toISOString(),
    });

    return decision;
  }
}
~~~

这里有三个生产级细节：

- **compare-and-swap**：防止两个发布流程同时覆盖 LKG；
- **rollback probe**：不是口头说能回滚，而是主动验证旧包仍可加载、可执行、可解释；
- **pointer_updated 事件**：LKG 变化本身也要可审计，后续事故取证需要知道哪个流程改了它。

---

## 5. OpenClaw 实战：课程 Cron 的 LKG 思维

OpenClaw 课程 Cron 虽然只是发一节课，但它也有类似 LKG 的概念：

~~~text
旧可信状态：
- lessons/405-xxx.md 已存在
- README.md 目录包含 405
- TOOLS.md 已讲内容包含 405
- 远端 main 指向 commit A

候选状态：
- 新增 lessons/406-xxx.md
- README.md 加 406
- TOOLS.md 加 406
- 本地 commit B

激活动作：
- git push origin main
- Telegram 发群消息

LKG 验证：
- git ls-remote origin main 包含 commit B
- git status clean
- Telegram messageId 已记录
- memory/YYYY-MM-DD.md 写入发布收据
~~~

这套流程已经在每天的课程发布里自然出现了：

1. **先 pull --rebase --autostash**，保证候选基于最新可信状态；
2. **git diff --check**，先做低成本一致性检查；
3. **commit 前后检查 staged diff**，确认写入范围；
4. **push 后 ls-remote 验证远端指针**，确认 active 状态真的生效；
5. **记录 messageId 和 commit hash**，留下发布收据；
6. **TOOLS.md 更新已讲内容**，避免下一次重复选题。

这其实就是小型 active promotion：不只是“我写了文件”，而是证明“课程目录、群消息、远端仓库、长期去重记忆”四个运行时都一致。

---

## 6. 常见坑

### 坑 1：把 LKG 太早更新

错误做法：

~~~text
canary 100% 刚开始 -> 立刻把 v43 标成 LKG
~~~

一旦 v43 后面出现问题，系统会把有问题的版本当可信回退点。

正确做法：

~~~text
canary 100% -> active freeze window -> consistency gate -> atomic LKG update
~~~

### 坑 2：只检查入口流量，不检查 runtime group

“100% 请求使用 v43”不代表 cron worker、sub-agent worker、message egress、git executor 都真的加载了 v43。

Guard Pack 这种安全包必须按 runtime group 统计覆盖率。

### 坑 3：忽略策略解释差异

同一条规则：

~~~text
external_write + missing evidence -> block
~~~

如果 tool_executor 判 block，但 message_egress 判 manual_review，这就是 policy interpretation mismatch。

版本一致但解释不一致，仍然不能晋级 LKG。

### 坑 4：样本缺口没有显式豁免

有些风险面在 freeze window 里没有样本是正常的，比如 deployment 低频发生。

但它必须变成明确的 exemption：

~~~json
{
  "riskSurface": "deployment",
  "reason": "no production deployment during freeze window",
  "compensatingEvidence": "offline replay pack passed",
  "approvedBy": "release-gate",
  "expiresAt": "2026-05-27T00:00:00Z"
}
~~~

没有样本、也没有豁免，就是 blind spot。

---

## 7. 实战检查清单

做 Guard Pack active promotion 前，至少检查：

- **Runtime coverage**：所有 runtime group 的 active version 覆盖率达标；
- **Version skew**：旧版本残留时间在预算内；
- **Interpretation parity**：同一输入在各 executor 的 decision 一致；
- **Risk surface evidence**：关键风险面有样本，或有带过期时间的豁免；
- **Rollback probe**：previous LKG 仍可加载、可执行、可恢复；
- **Freeze window**：全量激活后稳定观察窗口已结束；
- **Atomic LKG update**：用 compare-and-swap 更新 LKG 指针；
- **Promotion receipt**：记录 planId、old/new version、证据引用、决策原因；
- **Post-promotion monitor**：LKG 更新后继续观察一段短窗口，防止迟到信号。

---

## 8. 总结

Guard Pack 的发布链路可以拆成：

~~~text
Regression Candidate
  -> Guard Pack shadow
  -> canary blast radius ramp
  -> active consistency gate
  -> LKG atomic promotion
  -> post-promotion monitor
~~~

今天这课补上的是 **active consistency gate**。

核心原则：

- canary 100% 不是结束，只是进入全量一致性验证；
- LKG 是可信回退点，不能被未经稳定验证的版本覆盖；
- active promotion 必须检查 runtime coverage、version skew、policy interpretation、risk surface samples；
- LKG 更新要用事务和 compare-and-swap；
- 每次晋级都要留下 promotion receipt，方便回滚、取证和审计。

成熟 Agent 的安全发布，不是“新版本已经上线”，而是能回答：**谁在用它、是否一致、能不能退、为什么现在可以把它变成新的可信基线。**
