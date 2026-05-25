# 402. Agent 回归护栏包发布回滚与安全恢复（Guard Pack Rollout Rollback & Safe Recovery）

上一课讲了 **Guard Pack Schema Compatibility Gate**：护栏包升级 schema 时，旧 guard 要能迁移、标记 degradedCapabilities，并在高风险动作前通过兼容闸门。

今天继续往生产发布链路走一步：**兼容闸门通过，不代表新 Guard Pack 上线以后一定正确。**

一句话：**Guard Pack 发布必须有 Last Known Good、回滚触发器、影响窗口扫描和恢复收据；发现误伤或漏挡时，不能只把版本号改回去，还要证明运行时已经退回可信护栏，并处理新包影响过的决策。**

---

## 1. 为什么 Guard Pack 回滚不能只是 git revert

很多团队会把 guard pack 当配置文件：

~~~json
{
  "guardPackVersion": 42,
  "guards": [
    {"guardId": "block_duplicate_push", "tier": "active"}
  ]
}
~~~

然后上线出问题就想：

> 把 config 改回 v41 就好了。

这在 Agent 系统里不够。

因为 Guard Pack 参与的是运行时安全决策。v42 如果有问题，可能已经造成几类影响：

1. **误伤**：本来应该 allow 的任务被 block，队列里堆了很多 pending；
2. **漏挡**：本来应该 block 的外部副作用被 allow，比如发消息、push、部署；
3. **决策漂移**：不同 worker / session 看到不同 packVersion；
4. **证据污染**：v42 生成的 decision record 被下游当成可信依据；
5. **人工升级噪音**：错误 guard 把大量任务打进 manual_review。

所以成熟的回滚要分两层：

- **control-plane rollback**：把 active pack 指针退回 Last Known Good；
- **data-plane recovery**：扫描 v42 影响窗口内的决策，修复队列、证据和副作用。

只改版本号是止血，不是恢复。

---

## 2. 最小模型：GuardPackRollout

Guard Pack 发布时先写 rollout record，而不是直接覆盖 current。

~~~ts
type GuardPackRollout = {
  rolloutId: string;
  fromPackVersion: number;
  toPackVersion: number;
  strategy: "shadow" | "canary" | "active";
  trafficShare: number;
  startedAt: string;
  status: "running" | "promoted" | "rollback_requested" | "rolled_back" | "failed";
  lastKnownGood: {
    packId: string;
    version: number;
    hash: string;
    activatedAt: string;
  };
  rollbackTriggers: {
    falsePositiveRate: number;
    falseNegativeCount: number;
    unknownRate: number;
    latencyP95Ms: number;
  };
};

type GuardDecisionRecord = {
  decisionId: string;
  rolloutId: string;
  packVersion: number;
  guardIds: string[];
  actionId: string;
  actionKind: "read" | "write" | "message_send" | "git_push" | "deploy";
  decision: "allow" | "warn" | "dry_run" | "block" | "manual_review";
  evidenceRefs: string[];
  createdAt: string;
};
~~~

这里的关键字段是：

- rolloutId：把一次发布期间的所有 decision 绑起来；
- lastKnownGood：回滚不是猜版本，而是有 hash 的可信版本；
- rollbackTriggers：什么信号触发自动回滚；
- decision record：后面恢复扫描必须知道哪些动作被新包影响过。

没有 rolloutId，就只能靠日志搜索；没有 Last Known Good，就只能靠人肉判断。

---

## 3. learn-claude-code：纯函数回滚判定器

教学版可以把回滚判定做成纯函数。输入 rollout 指标，输出 promote / hold / rollback / manual_review。

~~~py
from dataclasses import dataclass


@dataclass(frozen=True)
class RolloutMetrics:
    sample_size: int
    false_positive_rate: float
    false_negative_count: int
    unknown_rate: float
    latency_p95_ms: int
    runtime_version_drift: bool


def evaluate_guard_pack_rollout(metrics: RolloutMetrics) -> dict:
    if metrics.runtime_version_drift:
        return {
            "decision": "rollback",
            "reason": "runtime_pack_version_drift",
            "freezeSideEffects": True,
        }

    if metrics.false_negative_count > 0:
        return {
            "decision": "rollback",
            "reason": "guard_pack_allowed_forbidden_action",
            "freezeSideEffects": True,
        }

    if metrics.false_positive_rate > 0.03 and metrics.sample_size >= 100:
        return {
            "decision": "rollback",
            "reason": "false_positive_budget_exceeded",
            "freezeSideEffects": False,
        }

    if metrics.unknown_rate > 0.10:
        return {
            "decision": "manual_review",
            "reason": "too_many_unknown_guard_evaluations",
        }

    if metrics.latency_p95_ms > 250:
        return {
            "decision": "hold",
            "reason": "guard_pack_latency_budget_exceeded",
        }

    if metrics.sample_size < 100:
        return {
            "decision": "hold",
            "reason": "insufficient_rollout_sample",
        }

    return {"decision": "promote", "reason": "rollout_within_budget"}
~~~

注意这里把 false negative 看得比 false positive 更重：

- 误伤会影响效率，可以回滚后重放队列；
- 漏挡可能已经造成外部副作用，要立即 freezeSideEffects；
- runtime drift 说明不同组件看到不同护栏，也要回滚，因为安全决策已经不可解释。

成熟系统的回滚判定不是“感觉不对”，而是预算 + 风险分级。

---

## 4. 回滚不是结束：还要扫描影响窗口

回滚执行后，要扫描从 rollout startedAt 到 rollback completedAt 的所有 decision record。

~~~py
def classify_impacted_decision(record: dict) -> dict:
    external = record["actionKind"] in ("message_send", "git_push", "deploy")

    if record["decision"] in ("block", "manual_review"):
        return {
            "impact": "possible_false_positive",
            "recovery": "requeue_after_lkg_recheck",
            "decisionId": record["decisionId"],
        }

    if external and record["decision"] in ("allow", "dry_run"):
        return {
            "impact": "possible_false_negative",
            "recovery": "reconcile_external_side_effect",
            "decisionId": record["decisionId"],
            "requiresBarrier": True,
        }

    if record["decision"] == "warn":
        return {
            "impact": "low_risk_warning_drift",
            "recovery": "archive_with_rollout_context",
            "decisionId": record["decisionId"],
        }

    return {
        "impact": "no_recovery_needed",
        "recovery": "none",
        "decisionId": record["decisionId"],
    }
~~~

恢复动作可以分三类：

- false positive：用 Last Known Good 重新评估，能过就 requeue；
- false negative：进入 external side-effect reconciliation，对账远端现实；
- warning drift：归档上下文，避免以后审计时误读。

这一步很重要：**回滚只能影响未来决策，影响窗口扫描负责处理已经发生的过去。**

---

## 5. pi-mono：Rollout Controller + Event Stream

生产版可以把 guard pack 发布做成 controller，而不是散落在配置加载器里。

~~~ts
type RollbackDecision =
  | { decision: "promote"; reason: string }
  | { decision: "hold"; reason: string }
  | { decision: "manual_review"; reason: string }
  | { decision: "rollback"; reason: string; freezeSideEffects: boolean };

class GuardPackRolloutController {
  constructor(
    private readonly store: GuardPackStore,
    private readonly metrics: GuardPackMetrics,
    private readonly decisions: GuardDecisionLedger,
    private readonly recovery: GuardPackRecoveryPlanner,
    private readonly events: EventBus,
  ) {}

  async evaluate(rolloutId: string): Promise<void> {
    const rollout = await this.store.getRollout(rolloutId);
    const snapshot = await this.metrics.snapshot(rolloutId);
    const decision = evaluateRollout(snapshot, rollout.rollbackTriggers);

    this.events.emit("guard_pack.rollout_evaluated", {
      rolloutId,
      toPackVersion: rollout.toPackVersion,
      decision: decision.decision,
      reason: decision.reason,
      snapshot,
    });

    if (decision.decision !== "rollback") {
      await this.store.recordRolloutDecision(rolloutId, decision);
      return;
    }

    await this.store.transaction(async (tx) => {
      await tx.guardPacks.activate(rollout.lastKnownGood);
      await tx.rollouts.markRolledBack({
        rolloutId,
        reason: decision.reason,
        rolledBackTo: rollout.lastKnownGood.version,
      });

      if (decision.freezeSideEffects) {
        await tx.policy.freezeExternalSideEffects({
          rolloutId,
          reason: decision.reason,
        });
      }
    });

    const impacted = await this.decisions.findByRolloutWindow(rolloutId);
    const plan = this.recovery.plan(impacted);

    await this.store.saveRecoveryPlan({
      rolloutId,
      fromPackVersion: rollout.toPackVersion,
      restoredPackVersion: rollout.lastKnownGood.version,
      plan,
    });

    this.events.emit("guard_pack.rollback_completed", {
      rolloutId,
      restoredPackVersion: rollout.lastKnownGood.version,
      impactedDecisionCount: impacted.length,
      recoveryPlanId: plan.planId,
    });
  }
}
~~~

这段代码的工程重点：

1. active pack 指针和 rollout 状态在一个事务里更新；
2. false negative / runtime drift 可以同时冻结外部副作用；
3. 回滚完成后立即生成 recovery plan；
4. 事件流里记录 restoredPackVersion 和 impactedDecisionCount，方便观察窗口接管。

和 pi-mono 的 EventStream 思路一样：不要只返回一个最终值，要把每个关键状态变化变成可订阅、可审计的事件。

---

## 6. OpenClaw 实战：课程 Cron 的 Guard Pack 回滚类比

拿这个课程 Cron 当例子。

一次发课涉及多个 guard：

- 选题不能重复；
- 必须写 lesson 文件；
- README 要更新目录；
- TOOLS.md 要更新已讲内容；
- Telegram 要发出去；
- git push 前要检查远端状态。

假设我们上线了一个新的 Guard Pack v42，其中“选题去重 guard”误判，把所有包含 rollback 的课程都当重复，于是课程任务被 block。

正确处理不是删掉 guard 然后继续跑，而是：

1. 记录 v42 rolloutId 和所有被 block 的 lesson decision；
2. 触发 false_positive_budget_exceeded，active 指针退回 v41；
3. 用 v41 重新评估被 block 的课程任务；
4. 重新放行后继续写 lesson、发 Telegram、提交 Git；
5. 生成 Rollback Recovery Receipt，写清楚 v42 影响了哪些 decision、哪些已重放、哪些无需处理。

如果反过来，新 Guard Pack 漏挡，导致重复课程已经发到群里，那就不能只回滚 guard：

- 要用 Release Ledger 找到 Telegram messageId；
- 判断 edit/delete/notice 哪个符合通道策略；
- Git commit 如果已 push，要走 revert 或 forward-fix；
- TOOLS.md / README 里的目录也要对账修复；
- 最后关闭 recovery plan。

这就是为什么 guard pack 回滚必须连接前面几课讲过的 side-effect reconciliation、release ledger、outcome barrier。

---

## 7. Rollback Recovery Receipt

回滚完成后要写收据，证明系统真的回到可信状态。

~~~json
{
  "receiptType": "guard_pack_rollback_recovery",
  "rolloutId": "gpr_2026_05_25_1530",
  "rolledBackFrom": 42,
  "restoredTo": {
    "packVersion": 41,
    "hash": "sha256:7b9..."
  },
  "reason": "false_positive_budget_exceeded",
  "sideEffectsFrozen": false,
  "impactedDecisions": {
    "total": 18,
    "requeued": 17,
    "reconciled": 0,
    "manualReview": 1
  },
  "runtimeConsistency": {
    "checkedComponents": ["scheduler", "worker", "tool_dispatch", "message_outbox"],
    "allObservedPackVersion": 41
  },
  "closedAt": "2026-05-25T05:30:00Z"
}
~~~

这个 receipt 回答四个问题：

- 退回了哪个 Last Known Good；
- 为什么退；
- v42 影响过的决策怎么处理；
- 当前运行时是不是都看到 v41。

没有 receipt 的回滚，只能算“尝试修了一下”。

---

## 8. 设计检查清单

做 Guard Pack 发布回滚时，至少检查这些点：

- 发布前是否记录 Last Known Good packId/version/hash；
- 每个 guard decision 是否带 rolloutId 和 packVersion；
- rollback trigger 是否区分 false positive / false negative / unknown / latency / runtime drift；
- 回滚 active 指针和 rollout 状态是否原子更新；
- 漏挡或版本漂移时是否冻结外部副作用；
- 回滚后是否扫描影响窗口；
- 被误伤的任务是否用 Last Known Good 重新评估后再放行；
- 已发生的外部副作用是否进入 reconciliation；
- 是否生成 Rollback Recovery Receipt；
- 是否做 runtime consistency check，确认所有 worker 都看到 restored version。

---

## 9. 小结

Guard Pack 是 Agent 的运行时安全系统，不是普通配置。

今天的核心：

- 发布 Guard Pack 要有 rollout record 和 Last Known Good；
- 回滚触发器要按误伤、漏挡、unknown、延迟、运行时漂移分级；
- 回滚不是只改版本号，还要扫描影响窗口；
- 外部副作用要进入 reconciliation，队列误伤要重新评估；
- 最后用 Rollback Recovery Receipt 证明恢复完成。

成熟 Agent 的护栏发布能力，不是“能上线新规则”，而是新规则出问题时，能快速退回可信版本，并把已经影响过的现实世界补回一致状态。
