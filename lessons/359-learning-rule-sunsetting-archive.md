# 359. Agent 学习规则退场与归档（Learning Rule Sunsetting & Archive）

上一课讲了 Learning Rule Version Lineage：学习规则不能只是一句记忆，而要有稳定 ID、不可变版本、parentVersion、ruleHash 和 dependencyLock。

今天继续补最后一块生命周期：**学习规则什么时候应该退场，退场后怎么归档，怎么保证它不再影响决策但仍然可审计。**

很多 Agent 系统一开始会拼命“记住更多东西”。问题是，经验会过期：

- 工具 schema 改了，旧规矩已经不适用；
- 用户偏好变了，旧偏好继续生效会误伤；
- 事故修复已经被产品逻辑消灭，旧 guardrail 只剩噪音；
- 某条学习规则长期没有命中过，但还在占 token 和影响排序；
- 两条新旧规则都 active，旧规则在低置信场景里抢了决策权。

一句话：**成熟 Agent 不只是会学习，还要会让学习安全退场。**

## 1. 不要直接删除学习规则

看一个危险做法：

~~~json
{
  "learningId": "learn.git.push.pr-state-check",
  "deleted": true
}
~~~

这会丢掉几个关键问题的答案：

- 这条规则为什么退场？
- 是被新版本 supersede，还是证明无效，还是误伤太多？
- 退场前影响过哪些 run？
- 如果同类事故复发，要从哪条历史规则恢复？
- 退场后是否还有 prompt、cache、vector index 残留副本？

学习规则退场不是 delete，而是一次有证据的状态迁移。

建议状态机：

~~~text
draft -> shadow -> canary -> active
active -> deprecated -> archived
active -> retired -> archived
active -> rolled_back -> repair -> shadow
~~~

几个状态要分清：

- deprecated：仍可被检索做参考，但不能直接影响自动决策；
- retired：不再进入 prompt / policy / retrieval，只保留审计；
- archived：进入冷存储，只能按 evidence / incident / replay 手动查；
- rolled_back：不是正常退役，是出现误伤或事故后撤回，必须进入 repair。

## 2. Sunsetting Decision：退场必须有理由

一个最小的退场决策包可以这样设计：

~~~json
{
  "learningId": "learn.git.push.pr-state-check",
  "version": 4,
  "fromStage": "active",
  "toStage": "deprecated",
  "reason": "superseded",
  "supersededBy": {
    "learningId": "learn.git.push.preflight-bundle",
    "version": 2
  },
  "metricsWindow": {
    "from": "2026-05-01T00:00:00Z",
    "to": "2026-05-20T00:00:00Z",
    "retrieved": 81,
    "decisionInfluenced": 12,
    "preventedFailures": 0,
    "falsePositiveBlocks": 2
  },
  "requiredCleanup": [
    "remove_from_prompt_pack",
    "remove_from_vector_index",
    "invalidate_policy_cache",
    "write_archive_manifest"
  ],
  "evidenceRefs": [
    "learning-metrics://learn.git.push.pr-state-check/v4/2026-05",
    "replay://git-push-safety@v9",
    "policy://git-safety@2026-05-20"
  ]
}
~~~

这里最重要的是 reason。常见理由可以枚举化：

- superseded：被更精确的新规则替代；
- expired：超过 expiresAt / reviewAfter 后没有重新验证；
- low_value：长期没有实际影响决策；
- harmful：误伤或 false positive 超过阈值；
- dependency_invalid：依赖工具、证据、策略已经失效；
- scope_removed：相关产品、功能、渠道不存在了。

## 3. learn-claude-code：纯函数退场闸门

教学版先写成一个可测试的纯函数。它不负责真正删除，只负责根据指标和依赖状态给出退场建议。

~~~python
from dataclasses import dataclass
from typing import Literal

Stage = Literal["shadow", "canary", "active", "deprecated", "retired", "archived"]
SunsetAction = Literal["keep", "deprecate", "retire", "archive", "rollback"]

@dataclass(frozen=True)
class LearningMetrics:
    retrieved: int
    injected: int
    decision_influenced: int
    prevented_failures: int
    false_positive_blocks: int
    days_since_last_positive_influence: int

@dataclass(frozen=True)
class DependencyStatus:
    policy_valid: bool
    tool_schema_valid: bool
    evidence_available: bool
    replay_pack_current: bool

@dataclass(frozen=True)
class SunsetDecision:
    action: SunsetAction
    reason: str
    next_stage: Stage
    required_cleanup: list[str]

def decide_learning_sunset(
    stage: Stage,
    metrics: LearningMetrics,
    deps: DependencyStatus,
    expires_in_days: int,
    superseded_by: str | None = None,
) -> SunsetDecision:
    if stage not in ("active", "deprecated", "retired"):
        return SunsetDecision("keep", "not_terminal_stage", stage, [])

    if metrics.false_positive_blocks >= 2:
        return SunsetDecision(
            action="rollback",
            reason="harmful_false_positive_blocks",
            next_stage="deprecated",
            required_cleanup=["remove_from_prompt_pack", "open_repair_ticket"],
        )

    if not all([deps.policy_valid, deps.tool_schema_valid, deps.evidence_available]):
        return SunsetDecision(
            action="deprecate",
            reason="dependency_invalid",
            next_stage="deprecated",
            required_cleanup=["remove_from_prompt_pack", "rerun_replay_or_refresh_evidence"],
        )

    if superseded_by:
        return SunsetDecision(
            action="deprecate",
            reason=f"superseded_by:{superseded_by}",
            next_stage="deprecated",
            required_cleanup=["remove_from_prompt_pack", "link_successor"],
        )

    if expires_in_days <= 0 and metrics.prevented_failures == 0:
        return SunsetDecision(
            action="retire",
            reason="expired_without_positive_influence",
            next_stage="retired",
            required_cleanup=["remove_from_retrieval_index", "write_archive_manifest"],
        )

    if (
        metrics.decision_influenced == 0
        and metrics.days_since_last_positive_influence > 30
    ):
        return SunsetDecision(
            action="archive",
            reason="low_value_stale_rule",
            next_stage="archived",
            required_cleanup=["remove_from_retrieval_index", "cold_archive"],
        )

    return SunsetDecision("keep", "still_useful", stage, [])
~~~

注意：这个函数输出的是 decision，不是 side effect。真正的清理动作要由执行器按 required_cleanup 做，并写审计日志。

learn-claude-code 的 s12_worktree_task_isolation.py 里，worktree remove / keep 会写生命周期事件，比如 worktree.remove.before、worktree.remove.after、worktree.keep。学习规则也应该一样：状态迁移前后都写事件，不能静默消失。

## 4. Archive Manifest：归档不是扔进角落

退场后至少要留下一个归档清单：

~~~json
{
  "archiveId": "archive.learn.git.push.pr-state-check.v4",
  "learningId": "learn.git.push.pr-state-check",
  "version": 4,
  "finalStage": "retired",
  "reason": "expired_without_positive_influence",
  "archivedAt": "2026-05-20T17:30:00Z",
  "successor": null,
  "lastRuleHash": "sha256:8a91...",
  "dependencyLock": {
    "policyVersion": "policy.git-safety@2026-05-01",
    "replayPack": "regression-pack://git-push-safety@v8"
  },
  "impactSummary": {
    "lifetimeRetrieved": 420,
    "lifetimeDecisionInfluenced": 37,
    "lifetimePreventedFailures": 4,
    "lifetimeFalsePositiveBlocks": 2
  },
  "resurrectionPolicy": {
    "allowed": true,
    "requires": [
      "new_counterfactual_replay",
      "current_dependency_lock",
      "canary_release"
    ]
  }
}
~~~

这个 manifest 的价值是：半年后如果同类问题又出现，Agent 可以知道“这条旧规则曾经存在，但不能直接复活；必须用当前依赖重新 replay，再走 canary”。

## 5. pi-mono：用 Agent.subscribe 观察影响力，再触发退场

pi-mono 的 Agent 有 subscribe(fn)，mom 里也用 session.subscribe 监听 tool_execution_start 等事件。这类事件流很适合做 learning influence ledger：不侵入主 Agent Loop，只在外层记录某条 learning 是否被检索、注入、影响决策、阻断了副作用。

概念代码：

~~~ts
type LearningInfluenceEvent = {
  runId: string;
  learningId: string;
  version: number;
  event:
    | "retrieved"
    | "injected"
    | "decision_influenced"
    | "prevented_failure"
    | "false_positive_block";
  at: number;
};

type LearningSunsetDecision = {
  learningId: string;
  version: number;
  action: "keep" | "deprecate" | "retire" | "archive" | "rollback";
  reason: string;
  cleanup: string[];
};

class LearningSunsetMonitor {
  private events: LearningInfluenceEvent[] = [];

  record(event: LearningInfluenceEvent) {
    this.events.push(event);
  }

  evaluate(learningId: string, version: number): LearningSunsetDecision {
    const rows = this.events.filter(
      (e) => e.learningId === learningId && e.version === version,
    );

    const influenced = rows.filter((e) => e.event === "decision_influenced").length;
    const prevented = rows.filter((e) => e.event === "prevented_failure").length;
    const falsePositives = rows.filter((e) => e.event === "false_positive_block").length;

    if (falsePositives >= 2) {
      return {
        learningId,
        version,
        action: "rollback",
        reason: "false_positive_threshold_exceeded",
        cleanup: ["remove_from_prompt_pack", "open_repair_ticket"],
      };
    }

    if (influenced === 0 && prevented === 0) {
      return {
        learningId,
        version,
        action: "retire",
        reason: "no_positive_influence_in_window",
        cleanup: ["remove_from_retrieval_index", "write_archive_manifest"],
      };
    }

    return { learningId, version, action: "keep", reason: "useful", cleanup: [] };
  }
}

const sunsetMonitor = new LearningSunsetMonitor();

agent.subscribe((event) => {
  if (event.type !== "learning_influence") return;

  sunsetMonitor.record({
    runId: event.runId,
    learningId: event.learningId,
    version: event.version,
    event: event.influenceType,
    at: Date.now(),
  });
});
~~~

重点不是这段代码能直接粘贴进当前 pi-mono，而是架构位置：**学习规则生命周期治理应该挂在事件流外层，而不是塞进每次 LLM prompt 里临场判断。**

## 6. OpenClaw 实战：课程 Cron 的去重规则也要退场

OpenClaw 课程 Cron 现在每 3 小时会检查 TOOLS.md 的已讲内容，避免重复。这类规则如果一直增长，会遇到两个问题：

- 已讲内容越来越长，检索和 prompt 成本增加；
- 旧课程主题可能被新版大纲覆盖，旧标题不能永久阻止新主题。

更成熟的做法：

~~~json
{
  "rule": "avoid_duplicate_agent_course_topics",
  "activeIndex": "topics.active.json",
  "archiveIndex": "topics.archive.json",
  "freshWindowDays": 90,
  "dedupePolicy": {
    "sameTitle": "block",
    "sameConceptFresh": "block",
    "sameConceptArchived": "allow_if_new_angle_and_refs"
  }
}
~~~

也就是说：

- 近 90 天讲过的主题，严格 block；
- 很久以前讲过但架构已经演进的主题，可以用新角度重讲；
- 被归档的主题不直接进 prompt，只在选题冲突时按需查询；
- 每次允许“旧题新讲”，必须写清新证据和新差异。

这就是学习规则退场的核心价值：不是忘掉历史，而是不让历史无限占用当前决策路径。

## 7. 实战检查清单

给生产 Agent 加 learning sunset，可以从这 6 个检查开始：

1. 每条 learning 是否有 stage、version、expiresAt、reviewAfter？
2. 是否记录 retrieved / injected / decision_influenced / prevented_failure / false_positive？
3. 过期规则是否自动从 prompt pack 和 retrieval index 移除？
4. 退场是否写 Sunsetting Decision 和 Archive Manifest？
5. superseded 规则是否链接 successor，避免旧规则和新规则同时抢决策？
6. archived 规则是否禁止直接复活，必须重新 replay + canary？

最后总结一句：**学习规则的生命周期应该像代码发布一样，有创建、验证、发布、观测、回滚、退场和归档。只会写记忆的 Agent 会越来越重；会安全忘记的 Agent 才能长期运行。**
