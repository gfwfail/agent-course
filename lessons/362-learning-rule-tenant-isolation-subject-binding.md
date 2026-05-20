# 362. Agent 学习规则租户隔离与主体绑定（Learning Rule Tenant Isolation & Subject Binding）

上一课讲了学习规则的权限边界：Learning Rule 只能收窄行为，不能自动放大权限。

今天继续补一块很容易被忽略的生产问题：**学习规则必须绑定它从哪里学来、能影响谁，不能跨用户、跨租户、跨频道乱生效。**

很多 Agent 的长期记忆事故，不是因为“记错了”，而是因为“用错地方了”。

比如在一个 Telegram 群里学到：

~~~text
老板喜欢课程内容简洁，讲完直接发群，不要多问。
~~~

这条规则只适用于特定老板、特定课程任务、特定 Telegram 群。如果系统把它变成全局规则：

~~~json
{
  "rule": "send directly without asking",
  "scope": "*"
}
~~~

那就危险了：它可能影响别的客户、别的群、别的外部副作用，甚至把某个用户的偏好泄漏给另一个用户。

一句话：**学习规则不是公共常识，除非它被明确证明可以公共化。**

## 1. 为什么 Subject Binding 比普通 scope 更严格

普通 scope 常写工具、repo、渠道：

- repo: gfwfail/agent-course；
- capability: telegram_send；
- channel: telegram:-5115329245；
- risk: external_side_effect。

但这些还不够。因为同一个工具、同一个 repo、同一个群，也可能有不同主体：

- 谁授权了这次操作；
- 学习来自哪个用户或租户；
- 规则可以影响哪个 actor；
- 规则能不能被共享到团队层；
- 是否包含私有偏好或个人上下文。

所以 Learning Rule 需要显式 subjectBinding：

~~~json
{
  "learningId": "learn.agent-course.concise-style",
  "stage": "active",
  "sourceSubject": {
    "tenantId": "owner-space",
    "actorId": "telegram:67431246",
    "channelId": "telegram:-5115329245",
    "sessionClass": "scheduled_course_cron"
  },
  "effectiveSubject": {
    "tenantIds": ["owner-space"],
    "actorIds": ["telegram:67431246", "telegram:7594549600"],
    "channelIds": ["telegram:-5115329245"],
    "sessionClasses": ["scheduled_course_cron"]
  },
  "shareability": "private",
  "containsPersonalPreference": true,
  "decisionImpact": {
    "canInjectPromptContext": true,
    "canAffectExternalSideEffect": true,
    "canCrossTenant": false
  }
}
~~~

这里有三条底线：

1. sourceSubject 记录“从哪里学来”；
2. effectiveSubject 记录“能影响哪里”；
3. private 学习默认不能跨租户、跨用户、跨频道复用。

## 2. 学习规则分三类：private、team、global

生产系统里可以把学习规则按共享级别分层：

- private：个人偏好、账号权限、私有项目上下文，只能用于绑定主体；
- team：团队 runbook、项目约定、共享事故经验，只能用于同 tenant / team；
- global：通用工程原则、公开 API 行为、框架事实，可以跨主体复用。

关键点：**global 不是默认值，而是审核后的结果。**

比如：

~~~text
git push 前先检查 PR 状态
~~~

看起来像通用规则，但真正落地时仍要拆开：

- “push 前需要检查远端状态”可以是 global engineering principle；
- “必须切换 gfwfail 账号”只能绑定到特定 repo / owner；
- “发到 Rust 学习小组”只能绑定到特定 Telegram channel；
- “老板喜欢简洁中文”是 private preference。

如果不拆，Agent 会把局部经验包装成全局真理。

## 3. learn-claude-code：Subject Gate 教学版

教学版可以用一个纯函数，在注入学习规则前检查当前运行主体是否被允许。

~~~python
from dataclasses import dataclass
from enum import Enum

class Shareability(str, Enum):
    PRIVATE = "private"
    TEAM = "team"
    GLOBAL = "global"

@dataclass(frozen=True)
class Subject:
    tenant_id: str
    actor_id: str
    channel_id: str
    session_class: str

@dataclass(frozen=True)
class EffectiveSubject:
    tenant_ids: set[str]
    actor_ids: set[str]
    channel_ids: set[str]
    session_classes: set[str]

@dataclass(frozen=True)
class LearningRule:
    learning_id: str
    shareability: Shareability
    source_subject: Subject
    effective_subject: EffectiveSubject
    contains_personal_preference: bool
    can_cross_tenant: bool

def matches(value: str, allowed: set[str]) -> bool:
    return value in allowed or "*" in allowed

def can_apply_learning_rule(rule: LearningRule, runtime: Subject) -> tuple[bool, str]:
    if rule.shareability == Shareability.PRIVATE:
        if runtime.tenant_id != rule.source_subject.tenant_id:
            return False, "private_learning_tenant_mismatch"
        if not matches(runtime.actor_id, rule.effective_subject.actor_ids):
            return False, "private_learning_actor_mismatch"
        if not matches(runtime.channel_id, rule.effective_subject.channel_ids):
            return False, "private_learning_channel_mismatch"

    if rule.shareability == Shareability.TEAM:
        if runtime.tenant_id not in rule.effective_subject.tenant_ids:
            return False, "team_learning_tenant_mismatch"

    if rule.shareability == Shareability.GLOBAL:
        if rule.contains_personal_preference:
            return False, "personal_preference_cannot_be_global"

    if not matches(runtime.session_class, rule.effective_subject.session_classes):
        return False, "learning_session_class_mismatch"

    if runtime.tenant_id != rule.source_subject.tenant_id and not rule.can_cross_tenant:
        return False, "cross_tenant_learning_forbidden"

    return True, "ok"
~~~

这个函数不是为了“少学”，而是为了让每条学习都有明确作用域。学习可以很多，但注入必须很窄。

## 4. pi-mono：Prompt Pack 注入前做主体过滤

pi-mono 里不要等 tool dispatch 才发现规则不该用。更好的位置是在组装 prompt pack 或 policy context 前，先过滤学习规则。

简化成 TypeScript：

~~~ts
type Shareability = "private" | "team" | "global";

type Subject = {
  tenantId: string;
  actorId: string;
  channelId: string;
  sessionClass: string;
};

type LearningRule = {
  learningId: string;
  shareability: Shareability;
  sourceSubject: Subject;
  effectiveSubject: {
    tenantIds: string[];
    actorIds: string[];
    channelIds: string[];
    sessionClasses: string[];
  };
  containsPersonalPreference: boolean;
  canCrossTenant: boolean;
  text: string;
};

type FilteredLearning = {
  inject: LearningRule[];
  excluded: Array<{ learningId: string; reason: string }>;
};

function includes(value: string, allowed: string[]): boolean {
  return allowed.includes(value) || allowed.includes("*");
}

function subjectDecision(rule: LearningRule, runtime: Subject): "allow" | string {
  if (rule.shareability === "global" && rule.containsPersonalPreference) {
    return "personal_preference_cannot_be_global";
  }

  if (!rule.canCrossTenant && runtime.tenantId !== rule.sourceSubject.tenantId) {
    return "cross_tenant_learning_forbidden";
  }

  if (!includes(runtime.sessionClass, rule.effectiveSubject.sessionClasses)) {
    return "session_class_mismatch";
  }

  if (rule.shareability === "private") {
    if (!includes(runtime.actorId, rule.effectiveSubject.actorIds)) {
      return "actor_mismatch";
    }
    if (!includes(runtime.channelId, rule.effectiveSubject.channelIds)) {
      return "channel_mismatch";
    }
  }

  if (rule.shareability === "team") {
    if (!includes(runtime.tenantId, rule.effectiveSubject.tenantIds)) {
      return "tenant_mismatch";
    }
  }

  return "allow";
}

function filterLearningForPrompt(
  rules: LearningRule[],
  runtime: Subject,
): FilteredLearning {
  const inject: LearningRule[] = [];
  const excluded: Array<{ learningId: string; reason: string }> = [];

  for (const rule of rules) {
    const decision = subjectDecision(rule, runtime);
    if (decision === "allow") {
      inject.push(rule);
    } else {
      excluded.push({ learningId: rule.learningId, reason: decision });
    }
  }

  return { inject, excluded };
}
~~~

然后 prompt builder 只接收过滤后的规则：

~~~ts
async function buildAgentPrompt(input: {
  runtime: Subject;
  retrievedLearning: LearningRule[];
}) {
  const filtered = filterLearningForPrompt(
    input.retrievedLearning,
    input.runtime,
  );

  await recordLearningInjectionAudit({
    runtime: input.runtime,
    injectedIds: filtered.inject.map((rule) => rule.learningId),
    excluded: filtered.excluded,
  });

  return [
    "You are an agent.",
    ...filtered.inject.map((rule) => "Learning: " + rule.text),
  ].join("\\n");
}
~~~

注意这里要记录 excluded evidence。否则排查问题时只知道“Agent 没想起某条规则”，不知道是检索没命中，还是主体隔离挡住了。

## 5. OpenClaw：课程 Cron 的真实例子

OpenClaw 课程 Cron 里有很多私有上下文：

- Telegram 群组 id；
- GitHub 账号 gfwfail；
- 老板偏好的中文授课风格；
- TOOLS.md 已讲内容；
- agent-course 本地路径；
- cron 的提交和推送流程。

这些规则都很有用，但多数不该变成全局 Agent 行为。

正确做法是把它们绑定到具体 subject：

~~~json
{
  "learningId": "learn.openclaw.agent-course.delivery",
  "shareability": "private",
  "sourceSubject": {
    "tenantId": "bot001-workspace",
    "actorId": "telegram:67431246",
    "channelId": "telegram:-5115329245",
    "sessionClass": "agent_course_cron"
  },
  "effectiveSubject": {
    "tenantIds": ["bot001-workspace"],
    "actorIds": ["telegram:67431246", "telegram:7594549600"],
    "channelIds": ["telegram:-5115329245"],
    "sessionClasses": ["agent_course_cron"]
  },
  "rule": "For each scheduled course, avoid repeated topics, write lesson markdown, update README and TOOLS, then commit and push as gfwfail."
}
~~~

这样未来同一个 Agent 去处理别的群、别的客户、别的仓库时，不会把“Rust 学习小组课程发布流程”误当成默认行为。

## 6. 最小落地清单

把 Subject Binding 落地，至少要做五件事：

1. Learning Rule schema 增加 sourceSubject、effectiveSubject、shareability；
2. 检索后、注入 prompt 前，先跑 Subject Gate；
3. private 学习默认禁止 cross-tenant；
4. personal preference 禁止自动升级成 global；
5. 注入和排除都写 audit evidence，方便解释“为什么用了 / 为什么没用”。

真正成熟的学习系统，不是记得越多越好，而是知道每条经验属于谁、能影响谁、什么时候必须闭嘴。

## 7. 课后练习

把你现有的 Agent memory 分成三类：

- private preference：某个人喜欢什么、某个账号怎么用；
- team runbook：某个团队约定的流程；
- global fact：通用技术事实。

然后给每条规则补上 tenantId、actorId、channelId、sessionClass。你会立刻发现：很多“看起来通用”的经验，其实只是某个上下文里的局部真理。
