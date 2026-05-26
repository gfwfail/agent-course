# 416. Agent 护栏模式知识库的效果遥测与退役闸门（Guard Pattern Effectiveness Telemetry & Retirement Gate）

上一课讲了 **Guard Recurrence Pattern Mining & Knowledge Base**：复发 case 关闭后，不要只写一段复盘，而要抽取 Recurrence Pattern，进入 Pattern Knowledge Base，让后续 finding、replay、release、escalation 都能先查旧模式。

今天继续补上一个关键闭环：**pattern 入库以后，也不能永远相信它。**

一句话：**Guard Pattern KB 要持续记录 pattern match 的效果遥测，用 outcome 证明它是否真的缩短 triage、补对 replay、减少复发；一旦误导、过期或成本过高，就要收紧 scope、降级到观察或退役。**

---

## 1. 为什么 Pattern 也会变坏

Pattern KB 很容易从“经验资产”变成“旧经验负债”。

典型变化有四类：

1. 工具 schema 改了，旧 predicate 还能匹配，但语义已经不一样；
2. guard pack 升级后，旧 recommendedAction 过于保守，频繁误伤；
3. replay matrix 已补齐，旧 pattern 还在要求人工复核；
4. 某个 risk surface 已经下线，pattern 仍然污染选例和派单。

所以 Pattern KB 不是只增不减的 wiki。它是运行时决策的一部分，就必须有 telemetry、health check 和 retirement gate。

---

## 2. Pattern Effectiveness 的最小指标

一个 pattern 每次被命中，都应该回答三件事：

~~~text
这次为什么命中？
它建议了什么动作？
后来事实证明这个建议有没有帮助？
~~~

最小指标可以这样设计：

~~~text
PatternEffectivenessSnapshot:
  patternId
  activeVersion
  windowStartedAt
  windowEndedAt
  matchCount                 命中次数
  actionAppliedCount          建议动作被采用次数
  helpedTriageCount           缩短排查或正确路由次数
  preventedRecurrenceCount    防住复发次数
  falseMatchCount             命中了但不该命中
  harmfulActionCount          推荐动作造成误伤或延误
  staleEvidenceCount          依赖证据过期次数
  avgAddedReplayCases         平均额外增加多少 replay case
  avgDecisionLatencyMs        平均增加多少决策延迟
~~~

重点是：**matchCount 不等于 usefulness。**

一个 pattern 一周命中 80 次，如果其中 50 次都是 false match，它不是“高价值模式”，而是在把旧事故的阴影投到新任务上。

---

## 3. learn-claude-code：纯函数退役闸门

教学版先用纯函数把一个窗口内的 pattern telemetry 转成生命周期动作。

~~~py
from dataclasses import dataclass
from typing import Literal


LifecycleDecision = Literal[
    "keep_active",
    "tighten_scope",
    "demote_observe",
    "refresh_predicates",
    "retire_pattern",
    "manual_review",
]


@dataclass(frozen=True)
class PatternSnapshot:
    match_count: int
    action_applied_count: int
    helped_triage_count: int
    prevented_recurrence_count: int
    false_match_count: int
    harmful_action_count: int
    stale_evidence_count: int
    avg_added_replay_cases: float
    avg_decision_latency_ms: int
    days_since_last_helpful_outcome: int
    code_path_exists: bool
    predicates_changed: bool


def safe_rate(numerator: int, denominator: int) -> float:
    if denominator <= 0:
        return 0.0
    return numerator / denominator


def decide_pattern_lifecycle(
    snapshot: PatternSnapshot,
) -> tuple[LifecycleDecision, list[str]]:
    if not snapshot.code_path_exists:
        return "retire_pattern", ["risk_surface_removed"]

    if snapshot.match_count == 0:
        if snapshot.days_since_last_helpful_outcome >= 90:
            return "retire_pattern", ["no_match_and_no_helpful_outcome_for_90_days"]
        return "keep_active", ["no_recent_match_but_still_within_ttl"]

    false_match_rate = safe_rate(snapshot.false_match_count, snapshot.match_count)
    harmful_rate = safe_rate(snapshot.harmful_action_count, snapshot.action_applied_count)
    stale_rate = safe_rate(snapshot.stale_evidence_count, snapshot.match_count)
    helpful_rate = safe_rate(
        snapshot.helped_triage_count + snapshot.prevented_recurrence_count,
        snapshot.match_count,
    )

    if harmful_rate > 0.05:
        return "demote_observe", [f"harmful_rate:{harmful_rate:.2f}"]

    if stale_rate > 0.25:
        return "refresh_predicates", [f"stale_evidence_rate:{stale_rate:.2f}"]

    if false_match_rate > 0.20:
        return "tighten_scope", [f"false_match_rate:{false_match_rate:.2f}"]

    if snapshot.predicates_changed:
        return "manual_review", ["predicate_dependency_changed"]

    if (
        helpful_rate < 0.03
        and snapshot.avg_added_replay_cases >= 5
        and snapshot.days_since_last_helpful_outcome >= 30
    ):
        return "demote_observe", [
            f"helpful_rate:{helpful_rate:.2f}",
            f"avg_added_replay_cases:{snapshot.avg_added_replay_cases:.1f}",
        ]

    if snapshot.avg_decision_latency_ms > 1200 and helpful_rate < 0.05:
        return "demote_observe", [
            f"avg_decision_latency_ms:{snapshot.avg_decision_latency_ms}",
            f"helpful_rate:{helpful_rate:.2f}",
        ]

    return "keep_active", [
        f"helpful_rate:{helpful_rate:.2f}",
        f"false_match_rate:{false_match_rate:.2f}",
    ]
~~~

这里故意把退役拆成多种动作：

- **tighten_scope**：pattern 有用，但匹配条件太宽；
- **refresh_predicates**：证据或依赖变了，需要重写 predicate；
- **demote_observe**：只做提示，不再影响 release / replay / routing；
- **retire_pattern**：风险面消失或长期无价值，归档退出热路径。

不要把所有问题都处理成删除。很多 pattern 是“还值得保留，但不能继续阻断”。

---

## 4. pi-mono：记录 PatternMatch 和 PatternOutcome

生产版里，Pattern KB 每次参与决策都要产出两类事件。

~~~ts
type PatternMatchEvent = {
  type: "guard_pattern.matched";
  eventId: string;
  runId: string;
  patternId: string;
  patternVersion: number;
  riskSurface: string;
  source:
    | "audit_finding"
    | "replay_selection"
    | "guard_pack_release"
    | "runtime_drift"
    | "human_escalation";
  matchReasonCodes: string[];
  recommendedAction:
    | "auto_route_repair"
    | "extend_replay_matrix"
    | "tighten_guard_scope"
    | "require_human_review"
    | "observe_only";
  actionApplied: boolean;
  evidenceRefs: string[];
  latencyMs: number;
  createdAt: Date;
};

type PatternOutcomeEvent = {
  type: "guard_pattern.outcome_recorded";
  eventId: string;
  matchEventId: string;
  patternId: string;
  outcome:
    | "helped_triage"
    | "prevented_recurrence"
    | "false_match"
    | "harmful_action"
    | "stale_evidence"
    | "neutral";
  decidedBy: "system" | "human" | "replay" | "incident_closeout";
  evidenceRefs: string[];
  createdAt: Date;
};
~~~

PatternMatchEvent 记录“旧经验当时怎么影响了当前 run”；PatternOutcomeEvent 记录“后来证明它有没有帮忙”。

没有 outcome，KB 只知道自己被查询过，不知道自己有没有价值。

---

## 5. pi-mono：PatternLifecycleWorker

可以把生命周期判断做成一个周期性 worker。它不直接改业务状态，而是产出可审计的 lifecycle decision。

~~~ts
type PatternLifecycleDecision = {
  patternId: string;
  patternVersion: number;
  decision:
    | "keep_active"
    | "tighten_scope"
    | "demote_observe"
    | "refresh_predicates"
    | "retire_pattern"
    | "manual_review";
  reasons: string[];
  windowStartedAt: Date;
  windowEndedAt: Date;
  snapshot: PatternEffectivenessSnapshot;
};

class PatternLifecycleWorker {
  constructor(
    private readonly store: PatternStore,
    private readonly telemetry: PatternTelemetryStore,
    private readonly eventBus: EventBus,
  ) {}

  async run(window: { startedAt: Date; endedAt: Date }) {
    const activePatterns = await this.store.listActivePatterns();

    for (const pattern of activePatterns) {
      const snapshot = await this.telemetry.buildSnapshot(pattern.patternId, window);
      const decision = decidePatternLifecycle(snapshot);

      await this.store.appendLifecycleDecision({
        patternId: pattern.patternId,
        patternVersion: pattern.version,
        decision: decision.kind,
        reasons: decision.reasons,
        windowStartedAt: window.startedAt,
        windowEndedAt: window.endedAt,
        snapshot,
      });

      await this.eventBus.publish({
        type: "guard_pattern.lifecycle_decided",
        patternId: pattern.patternId,
        patternVersion: pattern.version,
        decision: decision.kind,
        reasons: decision.reasons,
      });
    }
  }
}
~~~

真正应用 decision 时，再经过一个单独的 controller：

~~~ts
class PatternLifecycleController {
  async apply(decision: PatternLifecycleDecision) {
    if (decision.decision === "keep_active") return;

    if (decision.decision === "manual_review") {
      return this.openReviewCase(decision);
    }

    if (decision.decision === "tighten_scope") {
      return this.createScopeRepairTicket(decision);
    }

    if (decision.decision === "refresh_predicates") {
      return this.createPredicateRefreshTicket(decision);
    }

    if (decision.decision === "demote_observe") {
      return this.store.setPatternMode(decision.patternId, "observe");
    }

    if (decision.decision === "retire_pattern") {
      return this.store.retirePattern(decision.patternId, {
        retiredBy: "pattern_lifecycle_worker",
        reasons: decision.reasons,
        snapshotRef: decision.snapshot.snapshotId,
      });
    }
  }
}
~~~

这样做的好处是：**判断和执行分开**。判断可以每天跑，执行可以按风险走审批、PR 或自动变更。

---

## 6. OpenClaw 课程 Cron 实战映射

放到本课程 cron，Pattern KB 可以记住这些旧事故模式：

~~~text
pattern:course_publish:outcome_mismatch:remote-main-missing-commit
  match:
    lesson file exists
    README contains lesson id
    local commit exists
    remote main does not contain commit
  recommendedAction:
    block final response; run git push and ls-remote verify
~~~

它很有用，但也可能过期：

1. 如果课程仓库改成 PR 模式，remote main 不立刻包含 commit 就不再是错误；
2. 如果 README 改成自动生成，README 缺链接可能只是生成步骤还没跑；
3. 如果 Telegram 发布改成 outbox，messageId 不是即时条件；
4. 如果 git push 经常因为网络失败重试，pattern 应该要求 retry receipt，而不是直接升级人工。

所以每次 cron 完成后可以写一个轻量 outcome：

~~~text
PatternMatch:
  patternId = course_publish_remote_gate
  recommendedAction = verify_remote_commit
  actionApplied = true

PatternOutcome:
  outcome = helped_triage
  evidenceRefs = [git_ls_remote_contains_commit]
~~~

如果连续 30 天这个 pattern 都没有发现问题，但每次都增加大量延迟，可以降级到 observe。它仍记录信号，但不再阻塞发布。

---

## 7. 常见坑

**坑 1：只统计命中，不统计结果**

这会把“经常出现”误当成“很有用”。Pattern 必须有 outcome。

**坑 2：Pattern 一过期就删除**

删除会丢掉历史证据。正确做法是 retire + archive，保留 source case、retired reason 和最后一份 snapshot。

**坑 3：让语义相似度直接决定生命周期**

向量相似度可以辅助发现 false match，但不能单独决定退役或阻断。生命周期动作要绑定结构化 telemetry。

**坑 4：把退役当失败**

Pattern 能安全退役，说明系统知道旧经验什么时候不再适用。这是成熟，不是损失。

---

## 8. 小结

今天的核心：

~~~text
Pattern KB 不是只增不减；
每次 pattern 命中都要记录 match event；
每次后续事实确认都要记录 outcome event；
Lifecycle Worker 用 telemetry 决定 keep / tighten / refresh / observe / retire；
退役要归档证据，不要直接删除。
~~~

成熟 Agent 的知识库，不是记得越多越可靠，而是能持续证明哪些经验仍然有效，哪些经验应该收窄、观察或退出热路径。
