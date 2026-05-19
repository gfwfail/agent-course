# 361. Agent 学习规则权限边界与作用域收敛（Learning Rule Permission Boundary & Scope Narrowing）

上一课讲了 Learning Rule Schema 迁移：旧规则可以被读懂，但迁移出来的 synthetic 字段不能直接支撑高风险决策。

今天继续往前走一步：**学习规则不能因为“学到了”就自动获得权限。**

很多 Agent 事故不是模型不会总结，而是总结之后边界变宽了。比如一次复盘写下：

~~~text
以后遇到 git push 前，要先检查 PR 状态。
~~~

这是好规则。但如果系统把它转换成：

~~~json
{
  "condition": "before_any_git_operation",
  "action": "allow_if_recent_pr_check_passed"
}
~~~

边界就变危险了：

- 原来只是 push 前检查，被扩大成所有 git 操作；
- 原来只是要求检查，被误用成允许条件；
- 原来只适用于某个 repo，被全局注入；
- 原来只该影响提醒，被拿去影响外部副作用。

一句话：**学习规则只能收窄行为边界，不能自动放宽权限边界。**

## 1. 学习系统最容易犯的权限错误

学习规则进入生产后，常见有四种越权：

1. scope 扩张：从单 repo、单工具、单渠道，变成全局规则；
2. action 升级：从 remind / require_check，变成 allow / execute；
3. risk 降级：把外部副作用当成普通 answer；
4. subject 混淆：从某个用户、某个租户、某个会话学到的偏好，影响了其他人。

这和传统权限系统一样：**没有显式授予，就不能默认拥有。**

Learning Rule 应该被看成“行为约束候选”，不是 capability token。它可以告诉 Agent：

- 多检查一步；
- 少用一个工具；
- 要求审批；
- 降级成 dry-run；
- 缩小目标路径；
- 提醒用户确认。

但它不应该单独告诉 Agent：

- 现在可以发外部消息；
- 现在可以删除资源；
- 现在可以访问更高敏感度证据；
- 现在可以绕过 policy；
- 现在可以替用户授权子 Agent。

## 2. Scope Narrowing：学习默认只能收窄

可以给 Learning Rule 增加一个 permissionBoundary：

~~~json
{
  "learningId": "learn.git.push.pr-state-check",
  "stage": "active",
  "scope": {
    "actors": ["owner:67431246"],
    "repos": ["gfwfail/agent-course"],
    "capabilities": ["git_push"],
    "channels": ["telegram:-5115329245"],
    "riskSurface": ["source_control", "external_side_effect"]
  },
  "decisionImpact": {
    "canAllow": false,
    "canDeny": true,
    "canRequireApproval": true,
    "canNarrowScope": true,
    "canEscalateCapability": false
  },
  "permissionBoundary": {
    "maxSensitivity": "internal",
    "allowedEffects": ["require_check", "deny", "require_approval", "dry_run"],
    "forbiddenEffects": ["allow", "grant_capability", "bypass_policy"],
    "mustBeSubsetOfRuntimeGrant": true
  }
}
~~~

这里最关键的是三条：

- canAllow: false：学习不能单独放行动作；
- canEscalateCapability: false：学习不能给 Agent 新工具权限；
- mustBeSubsetOfRuntimeGrant: true：学习规则作用域必须是当前运行授权的子集。

这会让学习系统变成“刹车和方向盘”，而不是“万能钥匙”。

## 3. learn-claude-code：Scope Gate 教学版

教学版用纯函数做交集检查。规则很简单：runtime grant 是当前用户 / 会话 / 任务真正授予的能力；learning rule 只能在它的子集内生效。

~~~python
from dataclasses import dataclass

@dataclass(frozen=True)
class RuntimeGrant:
    actor: str
    repos: set[str]
    capabilities: set[str]
    channels: set[str]
    max_sensitivity: str

@dataclass(frozen=True)
class LearningBoundary:
    repos: set[str]
    capabilities: set[str]
    channels: set[str]
    can_allow: bool
    can_deny: bool
    can_require_approval: bool
    can_escalate_capability: bool
    max_sensitivity: str

SENSITIVITY_ORDER = ["public", "internal", "confidential", "secret"]

def sensitivity_lte(left: str, right: str) -> bool:
    return SENSITIVITY_ORDER.index(left) <= SENSITIVITY_ORDER.index(right)

def is_subset(child: set[str], parent: set[str]) -> bool:
    return child.issubset(parent) or "*" in parent

def evaluate_learning_boundary(
    runtime: RuntimeGrant,
    boundary: LearningBoundary,
    *,
    requested_effect: str,
) -> tuple[bool, str]:
    if not is_subset(boundary.repos, runtime.repos):
        return False, "learning_scope_escapes_runtime_repos"

    if not is_subset(boundary.capabilities, runtime.capabilities):
        return False, "learning_scope_escapes_runtime_capabilities"

    if not is_subset(boundary.channels, runtime.channels):
        return False, "learning_scope_escapes_runtime_channels"

    if not sensitivity_lte(boundary.max_sensitivity, runtime.max_sensitivity):
        return False, "learning_sensitivity_exceeds_runtime_grant"

    if requested_effect == "allow" and not boundary.can_allow:
        return False, "learning_rule_cannot_allow"

    if requested_effect == "grant_capability":
        return False, "learning_rule_cannot_grant_capability"

    if requested_effect == "require_approval" and boundary.can_require_approval:
        return True, "ok"

    if requested_effect == "deny" and boundary.can_deny:
        return True, "ok"

    if requested_effect in {"dry_run", "narrow_scope"}:
        return True, "ok"

    return False, "effect_not_allowed"
~~~

这个 Gate 的意义不是让 Agent 更保守，而是让规则的权力来源清楚：

- runtime grant 决定“最多能做什么”；
- policy 决定“这次能不能做”；
- learning rule 只能在里面增加约束；
- 学习不能凭空创造权限。

## 4. pi-mono：放在 Tool Dispatch 前，而不是 prompt 里

pi-mono 的 agent loop 在工具执行前会找到 tool、校验参数，然后调用 tool.execute。生产里的学习边界闸门应该插在这个位置之前，而不是只写进 system prompt。

简化成 TypeScript：

~~~ts
type RuntimeGrant = {
  actor: string;
  repos: string[];
  capabilities: string[];
  channels: string[];
  maxSensitivity: "public" | "internal" | "confidential" | "secret";
};

type LearningRule = {
  learningId: string;
  scope: {
    repos: string[];
    capabilities: string[];
    channels: string[];
  };
  decisionImpact: {
    canAllow: boolean;
    canDeny: boolean;
    canRequireApproval: boolean;
    canNarrowScope: boolean;
    canEscalateCapability: boolean;
  };
};

type BoundaryDecision =
  | { kind: "ok"; constraints: string[] }
  | { kind: "deny"; reason: string };

function assertLearningBoundary(
  runtime: RuntimeGrant,
  rule: LearningRule,
  requestedEffect: "allow" | "deny" | "require_approval" | "narrow_scope",
): BoundaryDecision {
  const subset = (child: string[], parent: string[]) =>
    child.every((value) => parent.includes(value) || parent.includes("*"));

  if (!subset(rule.scope.repos, runtime.repos)) {
    return { kind: "deny", reason: "learning_repo_scope_escape" };
  }

  if (!subset(rule.scope.capabilities, runtime.capabilities)) {
    return { kind: "deny", reason: "learning_capability_scope_escape" };
  }

  if (!subset(rule.scope.channels, runtime.channels)) {
    return { kind: "deny", reason: "learning_channel_scope_escape" };
  }

  if (requestedEffect === "allow" && !rule.decisionImpact.canAllow) {
    return { kind: "deny", reason: "learning_cannot_allow" };
  }

  if (rule.decisionImpact.canEscalateCapability) {
    return { kind: "deny", reason: "learning_capability_escalation_forbidden" };
  }

  return {
    kind: "ok",
    constraints: rule.decisionImpact.canNarrowScope ? ["apply_scope_narrowing"] : [],
  };
}
~~~

然后在工具执行前组合：

~~~ts
async function executeWithLearningBoundary(input: {
  runtime: RuntimeGrant;
  toolName: string;
  learningRules: LearningRule[];
  execute: () => Promise<unknown>;
}) {
  for (const rule of input.learningRules) {
    const decision = assertLearningBoundary(input.runtime, rule, "narrow_scope");
    if (decision.kind === "deny") {
      throw new Error(`Learning boundary violation: ${decision.reason}`);
    }
  }

  return input.execute();
}
~~~

注意：这不是替代 policy。它是在 policy 之外再加一层学习专用的边界约束，防止“记忆”和“权限”混在一起。

## 5. OpenClaw 实战：Cron 课程为什么适合这个模式

这条 Agent 课程 Cron 本身就是一个好例子。它每 3 小时会做：

- 检查已讲内容；
- 写 lessons/XX-topic.md；
- 更新 README.md；
- 发 Telegram；
- git commit / push；
- 更新 TOOLS.md。

这次任务有明确 runtime grant：只针对 agent-course 仓库、Rust 学习小组、课程文件和 TOOLS.md 已讲列表。

如果从历史记忆学到“老板允许课程 cron push main”，这条学习最多只能作用于：

- repo = gfwfail/agent-course；
- files = lessons/*、README.md；
- workspace note = TOOLS.md 已讲内容；
- channel = Telegram group -5115329245；
- action = commit/push 本课程内容。

它不能自动扩张成：

- 所有 repo 都能 push main；
- 所有 Telegram 群都能发布；
- 所有记忆都能外发；
- 所有外部副作用都不用检查。

所以 OpenClaw 可以把课程 Cron 的学习规则写成 scoped learning：

~~~json
{
  "learningId": "learn.agent-course.cron.direct-main-push",
  "source": "memory://TOOLS.md#Agent开发课程",
  "scope": {
    "taskName": "agent-course-cron",
    "repos": ["gfwfail/agent-course"],
    "paths": ["lessons/*.md", "README.md"],
    "workspacePaths": ["TOOLS.md"],
    "channels": ["telegram:-5115329245"],
    "capabilities": ["message.send", "git_commit", "git_push"]
  },
  "decisionImpact": {
    "canAllow": false,
    "canDeny": true,
    "canRequireApproval": true,
    "canNarrowScope": true,
    "canEscalateCapability": false
  }
}
~~~

这里 canAllow 仍然是 false。真正允许来自本次 cron 指令和运行时授权；学习只负责提醒边界、检查去重、阻止跑错仓库或发错群。

## 6. 实战 Checklist

设计 Learning Rule Permission Boundary 时，问 7 个问题：

1. 这条学习来自哪个 actor / tenant / channel；
2. 它适用哪些 repo / path / tool / capability；
3. 它最多能影响什么 decision effect；
4. 它能不能 allow，如果能，依据是什么；
5. 它是否只能作为 deny / require_approval / narrow_scope；
6. 它是否必须是 runtime grant 的子集；
7. 它触发外部副作用前是否仍需 policy + evidence 双重检查。

默认建议：

- preference learning：只能影响回答风格和排序；
- safety learning：可以 deny / require_approval；
- workflow learning：可以 narrow_scope / require_check；
- permission learning：必须转成 policy 或 capability，由人或系统显式授权，不能只留在 memory。

## 7. 小结

成熟 Agent 的学习系统，不是“越学越自由”，而是“越学越知道边界”。

Learning Rule Permission Boundary 的核心原则：

- 学习规则不是权限；
- 学习规则默认只能收窄，不能放宽；
- 学习作用域必须是 runtime grant 的子集；
- allow / grant_capability / bypass_policy 不能由 memory 单独决定；
- 所有外部副作用仍然要过 policy、evidence、freshness 和 audit。

这样 Agent 才能持续学习，又不会因为学到一句宽泛经验，就把生产权限边界悄悄扩大。
