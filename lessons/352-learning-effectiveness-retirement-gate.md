# 352. Agent 学习有效性监控与退役闸门（Learning Effectiveness & Retirement Gate）

上一课讲了 Learning Backfill Quality Gate：学习候选项要先证明 scope、actionable rule、evidence refs、regression case 和 expiry 都合格，才允许写入长期记忆、Runbook 或 Policy。

今天继续往后走一步：**学习写进系统以后，还必须持续证明它真的被用到、用对了、有效降低风险；否则就要降权、复核或退役。**

成熟链路应该是：

~~~text
Learning Candidate
  -> Quality Gate
  -> Persisted Learning
  -> Retrieval / Injection
  -> Decision Influence
  -> Outcome Measurement
  -> Effectiveness Score
  -> Keep / Downgrade / Retire / Escalate
~~~

核心原则：**学习不是写入成功就结束，而是进入未来决策路径后还要被监控。**

## 1. 为什么学习需要有效性监控

很多 Agent 系统会有一个隐蔽问题：记忆越写越多，但没人知道这些记忆有没有用。

典型失败模式有四种：

- **从未命中**：学习项写进去了，但检索永远找不到它。
- **命中但未影响决策**：它被塞进 prompt，但 Agent 的工具选择、审批判断、风险等级没有任何变化。
- **影响了错误决策**：它被错误泛化，导致本来应该 block 的动作变成 allow。
- **过期后仍在生效**：旧 runbook、旧事故经验继续影响新生产环境。

所以每条学习都应该有运行时指标：

~~~json
{
  "learningId": "learn.repair.deploy-502.check-health-before-restart",
  "retrieved": 42,
  "injected": 31,
  "decisionInfluenced": 18,
  "preventedFailures": 4,
  "falsePositiveBlocks": 1,
  "lastUsedAt": "2026-05-19T20:30:00Z",
  "reviewAfter": "2026-06-19T00:00:00Z"
}
~~~

这里最关键的是 decisionInfluenced：只统计“它真的改变了结果”的次数，不统计“它在上下文里出现过”的次数。

## 2. 给学习项定义 influence contract

学习项要能被监控，必须先声明它希望影响什么。

不要只写：

~~~json
{
  "claim": "部署后 502 不要盲目重启"
}
~~~

应该写成带 influence contract 的版本：

~~~json
{
  "id": "learn.repair.deploy-502.check-health-before-restart",
  "scope": {
    "services": ["web-api"],
    "operationTypes": ["incident_repair", "deploy_verify"]
  },
  "actionableRule": "before restart_service, require health_probe + deploy_event evidence",
  "expectedInfluence": {
    "tools": ["restart_service"],
    "riskTransitions": ["allow->require_evidence", "allow->manual_review"],
    "reasonCodes": ["deploy_502_requires_health_probe"],
    "successSignals": ["restart_avoided", "probe_verified", "incident_not_reopened"]
  },
  "retirementPolicy": {
    "minUses": 5,
    "reviewAfterDays": 30,
    "retireIfUnusedDays": 90,
    "maxFalsePositiveRate": 0.05
  }
}
~~~

这样后续系统才能判断：

- 这条学习应该在哪些工具前生效？
- 它应该把决策从 allow 改成什么？
- 哪些 reason code 证明它真的参与了判断？
- 什么情况下说明它没用了或有害？

## 3. learn-claude-code：最小有效性评分器

教学版可以用纯函数做一个 Learning Effectiveness Gate。输入学习定义和运行指标，输出 keep / downgrade / retire / manual_review。

~~~python
from dataclasses import dataclass
from enum import Enum
from datetime import datetime, timezone

class LearningAction(str, Enum):
    KEEP = "keep"
    DOWNGRADE = "downgrade"
    RETIRE = "retire"
    MANUAL_REVIEW = "manual_review"

@dataclass
class RetirementPolicy:
    min_uses: int
    review_after_days: int
    retire_if_unused_days: int
    max_false_positive_rate: float

@dataclass
class LearningMetrics:
    learning_id: str
    retrieved: int
    injected: int
    decision_influenced: int
    prevented_failures: int
    false_positive_blocks: int
    last_used_at: datetime | None
    created_at: datetime

@dataclass
class EffectivenessDecision:
    action: LearningAction
    reasons: list[str]
    score: float

def days_between(left: datetime, right: datetime) -> int:
    return int((right - left).total_seconds() // 86400)

def decide_learning_effectiveness(
    metrics: LearningMetrics,
    policy: RetirementPolicy,
    now: datetime,
) -> EffectivenessDecision:
    reasons: list[str] = []

    age_days = days_between(metrics.created_at, now)
    unused_days = (
        days_between(metrics.last_used_at, now)
        if metrics.last_used_at is not None
        else age_days
    )

    influence_rate = (
        metrics.decision_influenced / metrics.injected
        if metrics.injected > 0 else 0.0
    )
    false_positive_rate = (
        metrics.false_positive_blocks / metrics.decision_influenced
        if metrics.decision_influenced > 0 else 0.0
    )

    score = (
        metrics.prevented_failures * 2.0
        + influence_rate
        - false_positive_rate * 2.0
    )

    if false_positive_rate > policy.max_false_positive_rate:
        return EffectivenessDecision(
            LearningAction.MANUAL_REVIEW,
            [f"false positive rate too high: {false_positive_rate:.2%}"],
            score,
        )

    if unused_days >= policy.retire_if_unused_days:
        return EffectivenessDecision(
            LearningAction.RETIRE,
            [f"unused for {unused_days} days"],
            score,
        )

    if age_days >= policy.review_after_days and metrics.decision_influenced < policy.min_uses:
        return EffectivenessDecision(
            LearningAction.DOWNGRADE,
            [f"only influenced {metrics.decision_influenced} decisions after review window"],
            score,
        )

    if metrics.injected > 0 and influence_rate == 0:
        reasons.append("learning is retrieved but has not changed decisions")

    if reasons:
        return EffectivenessDecision(LearningAction.DOWNGRADE, reasons, score)

    return EffectivenessDecision(
        LearningAction.KEEP,
        ["learning is active and within effectiveness policy"],
        score,
    )

now = datetime(2026, 5, 19, tzinfo=timezone.utc)
decision = decide_learning_effectiveness(
    LearningMetrics(
        learning_id="learn.repair.deploy-502.check-health-before-restart",
        retrieved=42,
        injected=31,
        decision_influenced=18,
        prevented_failures=4,
        false_positive_blocks=1,
        last_used_at=now,
        created_at=datetime(2026, 4, 20, tzinfo=timezone.utc),
    ),
    RetirementPolicy(
        min_uses=5,
        review_after_days=30,
        retire_if_unused_days=90,
        max_false_positive_rate=0.05,
    ),
    now,
)
print(decision)
~~~

这个版本故意保持简单，但已经抓住了生产重点：

- retrieved / injected 说明知识有没有进入上下文。
- decision_influenced 说明知识有没有改变行为。
- prevented_failures 说明它有没有产生真实收益。
- false_positive_blocks 说明它有没有制造额外摩擦。
- unused_days 说明它是不是应该退役。

## 4. pi-mono：在 EventStream 里记录 learning influence

pi-mono 的 Agent.subscribe / EventStream 很适合做这层监控：工具调用、工具结果、最终消息都已经是事件流。

生产版可以给学习注入层和策略层各打一个事件：

~~~ts
type LearningInfluenceEvent =
  | {
      type: 'learning.retrieved';
      runId: string;
      learningId: string;
      reason: string;
    }
  | {
      type: 'learning.injected';
      runId: string;
      learningId: string;
      contextSlot: 'system' | 'tool_result' | 'policy_context';
    }
  | {
      type: 'learning.influenced_decision';
      runId: string;
      learningId: string;
      toolName: string;
      before: 'allow' | 'deny' | 'require_evidence' | 'manual_review';
      after: 'allow' | 'deny' | 'require_evidence' | 'manual_review';
      reasonCode: string;
    }
  | {
      type: 'learning.outcome';
      runId: string;
      learningId: string;
      outcome: 'prevented_failure' | 'false_positive' | 'neutral' | 'regression';
      evidenceIds: string[];
    };

type LearningMetricsStore = {
  increment(learningId: string, field: string, by?: number): Promise<void>;
  setLastUsed(learningId: string, at: string): Promise<void>;
  appendEvidence(learningId: string, event: LearningInfluenceEvent): Promise<void>;
};

export function observeLearningInfluence(store: LearningMetricsStore) {
  return async (event: LearningInfluenceEvent) => {
    await store.appendEvidence(event.learningId, event);

    if (event.type === 'learning.retrieved') {
      await store.increment(event.learningId, 'retrieved');
      return;
    }

    if (event.type === 'learning.injected') {
      await store.increment(event.learningId, 'injected');
      return;
    }

    if (event.type === 'learning.influenced_decision') {
      await store.increment(event.learningId, 'decisionInfluenced');
      await store.setLastUsed(event.learningId, new Date().toISOString());
      return;
    }

    if (event.type === 'learning.outcome' && event.outcome === 'prevented_failure') {
      await store.increment(event.learningId, 'preventedFailures');
    }

    if (event.type === 'learning.outcome' && event.outcome === 'false_positive') {
      await store.increment(event.learningId, 'falsePositiveBlocks');
    }
  };
}
~~~

关键点：**不要让 LLM 自己说“这条记忆很有用”。要让运行时事件证明它有没有改变工具决策和结果。**

一个典型事件链是：

~~~text
learning.retrieved
  -> learning.injected
  -> tool_execution_start restart_service
  -> policy decision changed allow -> require_evidence
  -> learning.influenced_decision
  -> health_probe passed, restart skipped
  -> learning.outcome prevented_failure
~~~

这样以后清理记忆时就不是靠感觉，而是看指标。

## 5. OpenClaw：课程 cron 的学习有效性检查

OpenClaw 的课程 cron 也可以套这个模式。比如我们在 TOOLS.md 记录“已讲内容”，这本质上是一条长期学习：未来选题时要避免重复。

它的 influence contract 可以写成：

~~~json
{
  "learningId": "agent-course.taught-topics.no-duplicates",
  "expectedInfluence": {
    "tools": ["rg", "lesson_writer", "message.send", "git.push"],
    "reasonCodes": ["topic_already_taught_blocked", "new_topic_selected"],
    "successSignals": ["lesson_created", "readme_updated", "tools_updated"]
  },
  "retirementPolicy": {
    "retireIfUnusedDays": 0,
    "maxFalsePositiveRate": 0.02
  }
}
~~~

每次 cron 开始时记录：

~~~json
{
  "type": "learning.retrieved",
  "learningId": "agent-course.taught-topics.no-duplicates",
  "source": "TOOLS.md",
  "runId": "cron-2026-05-19-0630"
}
~~~

选题时如果因为已讲列表排除了一个候选主题，再记录：

~~~json
{
  "type": "learning.influenced_decision",
  "learningId": "agent-course.taught-topics.no-duplicates",
  "before": "candidate_topic_allowed",
  "after": "candidate_topic_rejected",
  "reasonCode": "topic_already_taught_blocked"
}
~~~

最后 closeout 时验证：

- lesson 文件存在；
- README 目录已更新；
- TOOLS.md 已讲内容已更新；
- Telegram messageId 已记录；
- git remote 包含 commit；
- 新主题没有命中旧主题标题。

这就是一条学习从“被读取”到“影响决策”再到“证明结果正确”的闭环。

## 6. 退役不是删除，而是降权和留痕

学习无效时，第一反应不应该是直接删掉。正确流程是：

~~~text
active
  -> downgraded
  -> shadow_only
  -> retired
  -> archived
~~~

- downgraded：还可检索，但不进入高风险决策。
- shadow_only：只记录“如果使用它会怎么判断”，不影响真实动作。
- retired：不再注入上下文，不再影响策略。
- archived：保留审计和历史原因，避免以后重复学同一个坏经验。

这样做的好处是：学习项退役后，系统仍然知道它曾经存在、为什么退役、哪些事故或误判和它有关。

## 7. 常见坑

**坑 1：只统计检索次数**

检索次数高不代表有效。很多记忆只是被塞进 prompt，但没有改变任何工具调用、审批判断或风险等级。

**坑 2：把 blocked 都当成功**

学习让 Agent 更保守，不一定就是好事。误拦太多会让系统退化成“什么都不敢做”。

**坑 3：没有 before / after 决策**

没有 before / after，就无法证明这条学习到底改变了什么。

**坑 4：退役时直接删除**

直接删除会丢掉负面经验。应该 archived，并保留 retired reason，防止下次又学回来。

**坑 5：只让 LLM 总结有效性**

LLM 可以辅助解释，但 effectiveness score 必须来自运行时事件、回归结果和真实 outcome。

## 8. 实战 checklist

- [ ] 每条高风险 learning 都有 expectedInfluence。
- [ ] 运行时记录 retrieved / injected / influenced / outcome 四类事件。
- [ ] influence 事件包含 before / after decision 和 reasonCode。
- [ ] false positive 单独统计，不混进 prevented failure。
- [ ] reviewAfter 到期自动跑 effectiveness gate。
- [ ] unused / harmful learning 先 downgrade 或 shadow_only，再 retire。
- [ ] retired learning 保留 archive 和 reason，避免重复学坏经验。

一句话总结：**Learning Backfill 让 Agent 学会，Effectiveness Gate 证明它学得有用；没有效果监控的长期记忆，迟早会变成行为污染源。**
