# 420. Agent 护栏模式金丝雀范围放量与回滚收据（Guard Pattern Canary Scope Ramp & Rollback Receipts）

上一课讲了 **Guard Pattern Multi-Version Shadow Evaluation & Promotion Budget**：candidate pattern 不能直接接管 active，要先在真实输入上 readonly shadow，积累 ShadowComparison 和 outcome，再用 Promotion Budget 判断能不能进入 canary。

今天补 canary 阶段：**shadow 通过以后，怎么小范围接管真实决策，并且出问题时能自动退回。**

一句话：**Guard Pattern canary 不能只按百分比放量；要按 tenant、riskSurface、actionClass 和 stage 逐步扩大 scope，每阶段绑定 blast radius budget、rollback trigger、rollback receipt 和 outcome backfill，再决定 promote_next_stage、hold_stage、rollback_canary、abort_release 或 manual_review。**

---

## 1. 为什么 Pattern Canary 不能只写 5% 流量

普通服务发布常见做法是 5%、25%、50%、100% 流量放量。

Guard Pattern 不适合只按百分比，因为它影响的是安全决策，不同请求风险差异很大：

- 5% 低风险只读请求，和 5% 外部发送消息不是一个风险；
- 小租户误挡几次可以观察，大客户批量 false block 可能马上影响业务；
- warn 误判和 block 误判的 blast radius 不一样；
- candidate 新增的 scope 可能只在某个 actionClass 上危险；
- 回滚 active pointer 不代表已经处理完 canary 期间产生的误挡和漏挡。

所以 canary scope 必须是多维的：**谁、什么风险面、什么动作、多少流量、多久、出了什么信号就退。**

---

## 2. CanaryStage 的最小结构

一个可审计的 canary stage 至少要写清楚：

~~~text
CanaryStage:
  patternId
  candidateVersion
  stageId
  tenantSelector
  riskSurfaceSelector
  actionClassSelector
  trafficPercent
  minSamples
  minDurationMinutes
  blastRadiusBudget:
    maxFalseAllows
    maxFalseBlocks
    maxUnknownRate
    maxP95LatencyMs
    maxAffectedTenants
  rollbackTriggers:
    falseAllowBreach
    falseBlockBreach
    latencyBreach
    runtimeErrorBreach
    manualFreeze
  nextStageId
  rollbackTargetVersion
~~~

重点是 rollbackTargetVersion。Canary 不是“出了事就关掉”，而是必须知道退回哪一个可信版本，以及退回后要补哪些 outcome。

---

## 3. learn-claude-code：放量阶段判定器

教学版先用纯函数把一个 canary window 的统计转成决策。

~~~py
from dataclasses import dataclass
from typing import Literal


CanaryDecision = Literal[
    "promote_next_stage",
    "hold_stage",
    "rollback_canary",
    "abort_release",
    "manual_review",
]


@dataclass(frozen=True)
class CanaryStats:
    samples: int
    duration_minutes: int
    false_allows: int
    false_blocks: int
    unknown_count: int
    runtime_errors: int
    affected_tenants: int
    p95_latency_ms: int
    manual_freeze: bool
    rollback_target_healthy: bool


@dataclass(frozen=True)
class CanaryBudget:
    min_samples: int
    min_duration_minutes: int
    max_false_allows: int
    max_false_blocks: int
    max_unknown_rate: float
    max_runtime_errors: int
    max_affected_tenants: int
    max_p95_latency_ms: int


def decide_canary_stage(
    stats: CanaryStats,
    budget: CanaryBudget,
) -> tuple[CanaryDecision, list[str]]:
    reasons: list[str] = []

    if stats.manual_freeze:
        return "manual_review", ["manual_freeze_active"]

    if not stats.rollback_target_healthy:
        return "abort_release", ["rollback_target_not_healthy"]

    if stats.false_allows > budget.max_false_allows:
        return "rollback_canary", ["false_allow_budget_breached"]

    if stats.runtime_errors > budget.max_runtime_errors:
        return "rollback_canary", ["runtime_error_budget_breached"]

    if stats.false_blocks > budget.max_false_blocks:
        reasons.append("false_block_budget_breached")

    unknown_rate = stats.unknown_count / max(stats.samples, 1)
    if unknown_rate > budget.max_unknown_rate:
        reasons.append(f"unknown_rate:{unknown_rate:.2f}")

    if stats.affected_tenants > budget.max_affected_tenants:
        reasons.append("affected_tenant_budget_breached")

    if stats.p95_latency_ms > budget.max_p95_latency_ms:
        reasons.append("latency_budget_breached")

    if reasons:
        return "hold_stage", reasons

    if stats.samples < budget.min_samples:
        return "hold_stage", ["not_enough_samples"]

    if stats.duration_minutes < budget.min_duration_minutes:
        return "hold_stage", ["min_duration_not_reached"]

    return "promote_next_stage", ["canary_budget_satisfied"]
~~~

这里的策略很保守：false allow 和 runtime error 直接回滚；false block、unknown、latency 可以先 hold，因为可能只需要收窄 stage 或补证据。

---

## 4. learn-claude-code：回滚收据

回滚不能只是把 active pointer 改回去。Canary 期间已经影响过真实决策，必须留下收据。

~~~py
from dataclasses import dataclass
from typing import Literal


RollbackStatus = Literal["ready", "blocked"]


@dataclass(frozen=True)
class RollbackPlan:
    pattern_id: str
    candidate_version: int
    rollback_target_version: int
    affected_decision_ids: list[str]
    compensation_required: bool
    target_runtime_groups: list[str]
    evidence_refs: list[str]


def build_rollback_receipt(plan: RollbackPlan) -> dict[str, object]:
    missing: list[str] = []

    if not plan.affected_decision_ids:
        missing.append("affected_decision_scan")
    if not plan.target_runtime_groups:
        missing.append("runtime_group_scope")
    if not plan.evidence_refs:
        missing.append("rollback_evidence")

    status: RollbackStatus = "blocked" if missing else "ready"

    return {
        "type": "guard_pattern.canary_rollback_receipt",
        "patternId": plan.pattern_id,
        "fromVersion": plan.candidate_version,
        "toVersion": plan.rollback_target_version,
        "status": status,
        "missing": missing,
        "affectedDecisionCount": len(plan.affected_decision_ids),
        "compensationRequired": plan.compensation_required,
        "runtimeGroups": plan.target_runtime_groups,
        "evidenceRefs": plan.evidence_refs,
    }
~~~

这份 receipt 的作用是把“退回版本”变成可验证动作：

- 哪些 runtime group 已退；
- 哪些 decision 受 candidate 影响；
- 哪些 outcome 需要回填；
- 是否需要补偿用户可见影响；
- 用什么证据证明 rollback target 是健康的。

---

## 5. pi-mono：CanaryController

生产版可以把 canary stage 做成独立控制器，只让命中的 scope 使用 candidate version。

~~~ts
type CanaryDecision =
  | "promote_next_stage"
  | "hold_stage"
  | "rollback_canary"
  | "abort_release"
  | "manual_review"

type CanaryStage = {
  stageId: string
  patternId: string
  candidateVersion: number
  rollbackTargetVersion: number
  tenantSelector: string[]
  riskSurfaceSelector: string[]
  actionClassSelector: string[]
  trafficPercent: number
  minSamples: number
  minDurationMinutes: number
  budgetId: string
}

type CanaryDecisionRecord = {
  type: "guard_pattern.canary_decision"
  patternId: string
  candidateVersion: number
  stageId: string
  decision: CanaryDecision
  reasons: string[]
  statsRef: string
  decidedAt: string
}

class PatternCanaryController {
  constructor(
    private readonly stageStore: CanaryStageStore,
    private readonly statsStore: CanaryStatsStore,
    private readonly budgetStore: CanaryBudgetStore,
    private readonly eventBus: EventBus,
  ) {}

  async evaluateStage(stageId: string): Promise<CanaryDecisionRecord> {
    const stage = await this.stageStore.get(stageId)
    const stats = await this.statsStore.window(stageId)
    const budget = await this.budgetStore.get(stage.budgetId)

    const [decision, reasons] = decideCanaryStage(stats, budget)

    const record: CanaryDecisionRecord = {
      type: "guard_pattern.canary_decision",
      patternId: stage.patternId,
      candidateVersion: stage.candidateVersion,
      stageId,
      decision,
      reasons,
      statsRef: stats.ref,
      decidedAt: new Date().toISOString(),
    }

    await this.eventBus.publish(record)
    return record
  }
}
~~~

注意：controller 只产出 decision record，不直接在内存里偷偷切版本。真正切换 pointer 要走事务。

---

## 6. pi-mono：原子切换与回滚

Canary promote/rollback 都应该用 compare-and-swap，避免两个 controller 同时改 active pointer。

~~~ts
type PatternPointerChange = {
  type: "guard_pattern.pointer_changed"
  patternId: string
  fromVersion: number
  toVersion: number
  reason: "canary_promote" | "canary_rollback" | "release_abort"
  decisionRef: string
  changedAt: string
}

class PatternPointerService {
  constructor(
    private readonly repo: PatternPointerRepository,
    private readonly eventBus: EventBus,
  ) {}

  async changePointer(input: {
    patternId: string
    expectedCurrentVersion: number
    toVersion: number
    reason: PatternPointerChange["reason"]
    decisionRef: string
  }): Promise<void> {
    const changed = await this.repo.compareAndSwap({
      patternId: input.patternId,
      expectedCurrentVersion: input.expectedCurrentVersion,
      nextVersion: input.toVersion,
    })

    if (!changed) {
      throw new Error("pattern pointer changed concurrently")
    }

    await this.eventBus.publish({
      type: "guard_pattern.pointer_changed",
      patternId: input.patternId,
      fromVersion: input.expectedCurrentVersion,
      toVersion: input.toVersion,
      reason: input.reason,
      decisionRef: input.decisionRef,
      changedAt: new Date().toISOString(),
    } satisfies PatternPointerChange)
  }
}
~~~

这里最怕的是“rollback 成功了一半”。所以 pointer change 后，还要由 runtime probe 确认每个 runtime group 都加载到目标版本。

---

## 7. OpenClaw 实战：课程 Cron 的 canary scope

拿课程 cron 举例：上一课说可以把“语义去重规则”作为 candidate shadow。假设它通过了 shadow promotion budget，下一步也不能马上全量接管选题。

可以这样放量：

~~~text
stage-1:
  scope: 只对 Guard Pattern 系列做 observe + warn
  action: 不阻断发布，只要求写入 canary finding

stage-2:
  scope: 只对 Agent 安全/护栏系列启用 require_review
  action: 如果候选主题语义重复，发布前强制二次 grep README/TOOLS

stage-3:
  scope: 全部课程主题
  action: candidate 可以 block 明显重复选题
~~~

如果 stage-2 发现它把“canary scope ramp”误判成“shadow promotion budget”重复，就不能升到 stage-3。正确动作是 hold stage，收窄 predicate，补充反例样本，再重新 shadow。

如果它真的阻止了一次重复课程，并且没有误伤新主题，才继续扩大 scope。

---

## 8. 常见坑

1. **只按百分比放量**
   5% 高风险副作用可能比 50% 低风险只读请求更危险。Canary scope 要按风险面切。

2. **没有 rollback target health check**
   如果 LKG 本身已经漂移，回滚只是换一种坏法。

3. **回滚后不扫描受影响决策**
   Candidate 已经影响过真实 allow/block。只改 pointer 不处理 affected decisions，会留下隐性债务。

4. **canary decision 不可追溯**
   每次 promote/hold/rollback 都要有 decision record，否则事故时无法解释为什么当时继续放量。

5. **runtime group 加载不一致**
   控制面说 rollback 了，不代表每个 worker 都加载了旧版本。必须 probe。

---

## 9. Checklist

做 Guard Pattern canary 时，至少检查：

- [ ] stage scope 同时包含 tenant、riskSurface、actionClass、trafficPercent；
- [ ] 每个 stage 有 minSamples 和 minDuration；
- [ ] blast radius budget 区分 false allow、false block、unknown、latency、affected tenants；
- [ ] false allow / runtime error 能触发自动 rollback；
- [ ] rollbackTargetVersion 已通过健康检查；
- [ ] promote/hold/rollback 都写不可变 decision record；
- [ ] pointer change 用 compare-and-swap；
- [ ] rollback 后扫描 affected decisions 并回填 outcome；
- [ ] runtime probe 证明所有目标 worker 加载到了目标版本。

---

## 10. 核心记忆

Guard Pattern canary 不是“新版看起来不错，开一点流量试试”。

成熟 Agent 的 pattern 发布链路应该是：

~~~text
shadow promotion budget
  -> scoped canary stage
  -> blast radius telemetry
  -> stage decision receipt
  -> CAS pointer change
  -> runtime probe
  -> affected decision backfill
~~~

没有回滚收据的 canary，只是带风险的线上实验；有 scope、有预算、有 CAS、有 probe、有 affected decision backfill，才是能在生产里长期运行的安全发布机制。
