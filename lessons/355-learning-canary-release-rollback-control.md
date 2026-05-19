# 355. Agent 学习 Canary 发布与自动回滚（Learning Canary Release & Rollback Control）

上一课讲了 Learning Counterfactual Evaluation：学习候选不能只靠“听起来合理”，要拿历史真实决策做反事实回放，证明它能减少坏决策，并且不会制造大量误伤。

今天继续往上线链路推进：**反事实评估通过，也不要直接 active；学习规则应该像代码一样 canary 发布，并且必须有自动回滚条件。**

成熟链路是：

~~~text
Learning Candidate
  -> Quality Gate
  -> Conflict Detection
  -> Counterfactual Evaluation
  -> Shadow
  -> Canary
  -> Active
  -> Rollback / Retire
~~~

核心原则：**学习规则一旦生效，就会改变未来 Agent 的动作；所以它的发布必须有流量范围、观察指标、回滚目标和证据包。**

## 1. 为什么学习也要 Canary

很多团队会把“学习”当成文档更新：

~~~text
事故复盘 -> 写入 MEMORY / Runbook / Policy -> 下次自动使用
~~~

这很危险。因为学习不是静态知识，它会进入决策路径：

- 让某些工具调用从 allow 变成 require_evidence；
- 让某些环境从自动执行变成人工复核；
- 让某些用户请求被路由到不同的 Agent；
- 让某些历史经验覆盖默认策略。

比如一条学习：

~~~json
{
  "id": "learn.git-push.require-pr-state-check",
  "rule": "git push 前必须检查 gh pr list，确认没有往已 merge 分支继续推",
  "decisionIfMissing": "require_evidence"
}
~~~

它在课程 cron 里很有价值。但如果直接全局 active，可能误伤这些场景：

- 非 GitHub 仓库，没有 gh CLI；
- 本地教学 demo，只是 dry-run；
- bot 没有 GitHub 权限，但仍要提交本地 patch；
- 紧急修复已经处于 break-glass 流程。

所以学习规则要先从小范围跑起来，观察真实影响，再逐步扩大。

## 2. Learning Release Spec

不要只存一段 rule 文本。上线前要生成一个发布规格：

~~~json
{
  "releaseId": "lrnrel-2026-05-19-355",
  "learningId": "learn.git-push.require-pr-state-check",
  "version": 3,
  "stage": "canary",
  "scope": {
    "agents": ["course-cron"],
    "repos": ["gfwfail/agent-course"],
    "operations": ["git_push"],
    "channels": ["telegram:-5115329245"]
  },
  "traffic": {
    "mode": "deterministic_hash",
    "percentage": 10
  },
  "rollbackTarget": {
    "learningId": "learn.git-push.require-pr-state-check",
    "version": 2,
    "stage": "shadow"
  },
  "slo": {
    "maxFalsePositiveRate": 0.03,
    "maxBlockedGoodRuns": 1,
    "minPreventedBadRuns": 1
  }
}
~~~

这里最重要的是四件事：

- **scope**：只在哪些 Agent、仓库、操作、渠道生效。
- **traffic**：只影响多少真实请求。
- **rollbackTarget**：出问题退回哪个版本和阶段。
- **slo**：用什么指标判断继续晋级还是回滚。

没有 rollback target 的学习规则，不允许进入 canary。

## 3. learn-claude-code：最小 Canary 发布控制器

教学版可以用纯函数做：给定 release spec 和一次决策输入，判断学习规则是否参与决策；再根据观察事件决定 promote / hold / rollback。

~~~python
from dataclasses import dataclass, field
from enum import Enum
from hashlib import sha256

class Stage(str, Enum):
    SHADOW = "shadow"
    CANARY = "canary"
    ACTIVE = "active"
    ROLLED_BACK = "rolled_back"

class ReleaseDecision(str, Enum):
    PROMOTE = "promote"
    HOLD = "hold"
    ROLLBACK = "rollback"

@dataclass
class Scope:
    agents: set[str]
    repos: set[str]
    operations: set[str]
    channels: set[str]

@dataclass
class Traffic:
    percentage: int

@dataclass
class RollbackTarget:
    learning_id: str
    version: int
    stage: Stage

@dataclass
class LearningReleaseSpec:
    release_id: str
    learning_id: str
    version: int
    stage: Stage
    scope: Scope
    traffic: Traffic
    rollback_target: RollbackTarget
    max_false_positive_rate: float
    max_blocked_good_runs: int
    min_prevented_bad_runs: int

@dataclass
class DecisionContext:
    run_id: str
    agent: str
    repo: str
    operation: str
    channel: str

@dataclass
class LearningObservation:
    release_id: str
    influenced: bool
    prevented_bad_run: bool = False
    blocked_good_run: bool = False
    false_positive: bool = False

@dataclass
class ReleaseEvaluation:
    decision: ReleaseDecision
    reasons: list[str]

def in_scope(spec: LearningReleaseSpec, ctx: DecisionContext) -> bool:
    return (
        ctx.agent in spec.scope.agents
        and ctx.repo in spec.scope.repos
        and ctx.operation in spec.scope.operations
        and ctx.channel in spec.scope.channels
    )

def selected_by_canary(spec: LearningReleaseSpec, ctx: DecisionContext) -> bool:
    key = f"{spec.release_id}:{ctx.run_id}".encode()
    bucket = int(sha256(key).hexdigest()[:8], 16) % 100
    return bucket < spec.traffic.percentage

def should_apply_learning(spec: LearningReleaseSpec, ctx: DecisionContext) -> bool:
    if not in_scope(spec, ctx):
        return False

    if spec.stage == Stage.ACTIVE:
        return True

    if spec.stage == Stage.CANARY:
        return selected_by_canary(spec, ctx)

    # shadow 只记录“如果应用会怎样”，不改变真实动作。
    return False

def evaluate_release(
    spec: LearningReleaseSpec,
    observations: list[LearningObservation],
    min_influenced_runs: int = 20,
) -> ReleaseEvaluation:
    relevant = [
        item for item in observations
        if item.release_id == spec.release_id and item.influenced
    ]

    influenced = len(relevant)
    false_positive = sum(1 for item in relevant if item.false_positive)
    blocked_good = sum(1 for item in relevant if item.blocked_good_run)
    prevented_bad = sum(1 for item in relevant if item.prevented_bad_run)
    false_positive_rate = false_positive / influenced if influenced else 0.0

    if blocked_good > spec.max_blocked_good_runs:
        return ReleaseEvaluation(
            ReleaseDecision.ROLLBACK,
            [f"blocked good runs exceeded: {blocked_good}"],
        )

    if false_positive_rate > spec.max_false_positive_rate:
        return ReleaseEvaluation(
            ReleaseDecision.ROLLBACK,
            [f"false positive rate too high: {false_positive_rate:.2%}"],
        )

    if influenced < min_influenced_runs:
        return ReleaseEvaluation(
            ReleaseDecision.HOLD,
            [f"not enough influenced runs: {influenced}/{min_influenced_runs}"],
        )

    if prevented_bad >= spec.min_prevented_bad_runs:
        return ReleaseEvaluation(
            ReleaseDecision.PROMOTE,
            [f"prevented {prevented_bad} bad runs within SLO"],
        )

    return ReleaseEvaluation(
        ReleaseDecision.HOLD,
        ["no harm detected, but benefit is not proven yet"],
    )
~~~

这个控制器故意很小，但已经把关键边界表达清楚：

- shadow 不改真实动作；
- canary 用稳定 hash 选流量，避免同一个 run 抖动；
- active 全量生效；
- false positive 和 blocked good run 触发 rollback；
- 证据不足时 hold，不靠感觉晋级。

## 4. pi-mono：把学习发布做成中间件

生产实现不要散落在各个工具里。更合理的位置是 Agent 决策链外层的 middleware：

~~~ts
type LearningStage = "shadow" | "canary" | "active" | "rolled_back";
type Decision = "allow" | "require_evidence" | "manual_review" | "block";

interface LearningRelease {
  releaseId: string;
  learningId: string;
  version: number;
  stage: LearningStage;
  scope: {
    agents: string[];
    repos: string[];
    operations: string[];
    channels: string[];
  };
  traffic: {
    percentage: number;
  };
  rollbackTarget: {
    learningId: string;
    version: number;
    stage: LearningStage;
  };
}

interface AgentDecisionContext {
  runId: string;
  agentId: string;
  repo?: string;
  operation: string;
  channel?: string;
  decision: Decision;
  evidenceRefs: string[];
}

interface LearningInfluenceEvent {
  type: "learning.influence";
  releaseId: string;
  learningId: string;
  version: number;
  stage: LearningStage;
  runId: string;
  originalDecision: Decision;
  effectiveDecision: Decision;
  applied: boolean;
  evidenceRefs: string[];
}

function isScoped(release: LearningRelease, ctx: AgentDecisionContext): boolean {
  return release.scope.agents.includes(ctx.agentId)
    && release.scope.operations.includes(ctx.operation)
    && (!ctx.repo || release.scope.repos.includes(ctx.repo))
    && (!ctx.channel || release.scope.channels.includes(ctx.channel));
}

function bucket(input: string): number {
  let hash = 0;
  for (const char of input) {
    hash = (hash * 31 + char.charCodeAt(0)) >>> 0;
  }
  return hash % 100;
}

function shouldApply(release: LearningRelease, ctx: AgentDecisionContext): boolean {
  if (!isScoped(release, ctx)) return false;
  if (release.stage === "active") return true;
  if (release.stage !== "canary") return false;

  return bucket(`${release.releaseId}:${ctx.runId}`) < release.traffic.percentage;
}

function applyGitPushLearning(ctx: AgentDecisionContext): Decision {
  const hasPrStateEvidence = ctx.evidenceRefs.some((ref) =>
    ref.startsWith("evidence:gh-pr-list:")
  );

  if (ctx.operation === "git_push" && !hasPrStateEvidence) {
    return "require_evidence";
  }

  return ctx.decision;
}

export async function learningReleaseMiddleware(
  ctx: AgentDecisionContext,
  release: LearningRelease,
  emit: (event: LearningInfluenceEvent) => Promise<void>,
): Promise<AgentDecisionContext> {
  if (!isScoped(release, ctx)) return ctx;

  const candidateDecision = applyGitPushLearning(ctx);
  const applied = shouldApply(release, ctx);
  const effectiveDecision = applied ? candidateDecision : ctx.decision;

  await emit({
    type: "learning.influence",
    releaseId: release.releaseId,
    learningId: release.learningId,
    version: release.version,
    stage: release.stage,
    runId: ctx.runId,
    originalDecision: ctx.decision,
    effectiveDecision,
    applied,
    evidenceRefs: ctx.evidenceRefs,
  });

  return {
    ...ctx,
    decision: effectiveDecision,
  };
}
~~~

注意这里即使没有 applied，也要 emit influence event。因为 shadow 阶段最重要的证据就是：

> 如果这条学习当时生效，它会不会改变决策？

这和上一课的 counterfactual replay 是互补的：counterfactual 看历史样本，shadow/canary 看真实线上流量。

## 5. OpenClaw 实战：课程 Cron 的学习发布

以课程 cron 为例，假设我们把“push 前必须检查 PR 状态”做成学习规则。OpenClaw 的执行证据可以这样落地：

~~~text
memory/YYYY-MM-DD.md
  - 本次 run 是否触发 learning release
  - 是否检查 gh pr list
  - 是否执行 git diff --cached --check
  - Telegram messageId
  - commit sha

agent-course/.openclaw/evidence/learning-releases/
  lrnrel-2026-05-19-355.json
  lrnrel-2026-05-19-355-events.jsonl
  lrnrel-2026-05-19-355-evaluation.json
~~~

一个 release spec 可以长这样：

~~~json
{
  "releaseId": "lrnrel-agent-course-git-push-pr-check-v3",
  "learningId": "learn.git-push.require-pr-state-check",
  "version": 3,
  "stage": "canary",
  "scope": {
    "agents": ["agent-course-cron"],
    "repos": ["gfwfail/agent-course"],
    "operations": ["git_push"],
    "channels": ["telegram:-5115329245"]
  },
  "traffic": {
    "percentage": 25
  },
  "rollbackTarget": {
    "learningId": "learn.git-push.require-pr-state-check",
    "version": 2,
    "stage": "shadow"
  }
}
~~~

每次课程 cron 完成后写一条观察事件：

~~~json
{
  "type": "learning.influence.observed",
  "releaseId": "lrnrel-agent-course-git-push-pr-check-v3",
  "runId": "cron:3eba6ee3-2026-05-19T05:30:00Z",
  "applied": true,
  "originalDecision": "allow",
  "effectiveDecision": "require_evidence",
  "evidenceSatisfied": true,
  "evidenceRefs": [
    "evidence:gh-pr-list:2026-05-19T05:33:10Z",
    "evidence:git-diff-check:2026-05-19T05:34:02Z",
    "evidence:telegram-message:12217",
    "evidence:commit:abc1234"
  ],
  "outcome": "good_allow_after_evidence"
}
~~~

这条事件说明：学习确实影响了流程，但没有阻断正常发布，因为证据补齐后继续执行。

## 6. 自动回滚条件

学习回滚不能靠“感觉不太对”。至少要有这些硬条件：

~~~text
rollback if:
  false_positive_rate > 3%
  blocked_good_runs > 1
  missing_evidence_rate > 10%
  learning_engine_error_rate > 1%
  operator_override_count > 2
~~~

回滚动作也要结构化：

~~~json
{
  "action": "rollback_learning_release",
  "releaseId": "lrnrel-agent-course-git-push-pr-check-v3",
  "from": {
    "learningId": "learn.git-push.require-pr-state-check",
    "version": 3,
    "stage": "canary"
  },
  "to": {
    "learningId": "learn.git-push.require-pr-state-check",
    "version": 2,
    "stage": "shadow"
  },
  "reasonCodes": ["false_positive_rate_exceeded"],
  "evidenceRefs": ["evidence:learning-release-eval:2026-05-19T06:00:00Z"]
}
~~~

关键点：回滚不是删除学习，而是把它退回 shadow，继续收集证据或等待修订。

## 7. 常见坑

### 坑 1：Canary 只按随机数选流量

随机选择会导致同一 run 重试时命中状态变化。用稳定 hash：

~~~text
bucket = hash(releaseId + runId) % 100
~~~

这样重试、回放和审计都可复现。

### 坑 2：只记录最终决策，不记录 original decision

没有 original decision，就不知道学习到底改变了什么。Influence event 必须同时记录：

- originalDecision；
- candidateDecision；
- effectiveDecision；
- applied；
- evidenceRefs。

### 坑 3：Canary scope 太宽

学习规则先绑定一个小 scope，例如：

~~~text
agent-course-cron + gfwfail/agent-course + git_push
~~~

不要一开始就全局影响所有 Agent、所有 repo、所有副作用工具。

### 坑 4：没有负向指标

只看 prevented_bad_runs 会让规则越收越紧。必须同时看：

- false_positive；
- blocked_good_run；
- manual_review 增量；
- operator_override；
- missing_evidence_rate。

学习的目标不是“更保守”，而是“更正确”。

## 8. 一句话总结

Learning Canary Release 的重点不是“慢慢上线”，而是把学习规则从文本变成可发布、可观测、可回滚的生产资产。

成熟 Agent 不只会从事故里学到规则，还会像发布代码一样发布学习：先 shadow，再 canary，看真实影响，证据达标才 active，出问题能自动退回。
