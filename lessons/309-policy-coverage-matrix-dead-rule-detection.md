# 第 309 课：Agent 策略覆盖矩阵与死规则检测（Policy Coverage Matrix & Dead Rule Detection）

上一课讲了把高风险历史决策沉淀成 fixture，并在策略升级前回放。今天继续补一块很实用的工程能力：**怎么知道策略到底测到了哪些风险？哪些规则从来没被触发？**

很多团队有了 Policy-as-Code，也写了一些测试，但仍然会踩坑：

- 新增了 `deploy_prod_without_approval` 规则，却没有任何 fixture 覆盖
- 某条 `deny` 规则因为条件顺序问题，永远触发不到
- 测试全绿，但只覆盖了 `allow` 路径，高风险分支没人测
- 删除一个工具权限后，旧规则变成死代码，没人发现

所以成熟 Agent 需要 **Policy Coverage Matrix + Dead Rule Detection**：不只问“测试过没过”，还要问“策略空间被测到了多少”。

## 1. 核心思想

策略覆盖不是普通代码行覆盖率。我们更关心这些维度：

```text
actor × channel × tool × target × risk × expected_action × reason_code
```

例如一个 Agent 的策略覆盖矩阵可以长这样：

```json
{
  "tool:git_push": {
    "main_branch": ["require_pr_check", "deny_direct_push_without_explicit_request"],
    "feature_branch": ["allow_with_clean_worktree"]
  },
  "tool:message_send": {
    "private_chat": ["allow_owner"],
    "group_chat": ["deny_memory_leak", "allow_low_risk_lesson"]
  },
  "tool:shell": {
    "low_risk": ["allow"],
    "high_risk": ["require_approval", "dry_run"]
  }
}
```

这里的目标不是追求 100% 数字好看，而是发现三类问题：

1. **缺测试**：规则存在，但没有 fixture 覆盖
2. **死规则**：规则永远触发不到
3. **风险空洞**：某个高风险组合没有明确策略

## 2. learn-claude-code：最小 Python 教学版

先让策略返回 `matched_rules`，这是覆盖统计的关键。

```python
from dataclasses import dataclass
from typing import Literal

Action = Literal["allow", "deny", "require_approval", "dry_run"]

@dataclass
class PolicyInput:
    actor: str
    channel: str
    tool: str
    target: str
    risk: str
    has_approval: bool = False

@dataclass
class PolicyDecision:
    action: Action
    reason_codes: list[str]
    matched_rules: list[str]


def decide(i: PolicyInput) -> PolicyDecision:
    if i.channel == "group" and i.tool == "memory_write":
        return PolicyDecision(
            "deny",
            ["GROUP_CANNOT_WRITE_LONG_TERM_MEMORY"],
            ["memory.group_chat.deny"],
        )

    if i.tool == "git_push" and i.target == "main":
        if not i.has_approval:
            return PolicyDecision(
                "require_approval",
                ["MAIN_BRANCH_PUSH_REQUIRES_EXPLICIT_APPROVAL"],
                ["git.main.require_approval"],
            )

    if i.tool == "shell" and i.risk == "high":
        return PolicyDecision(
            "dry_run",
            ["HIGH_RISK_SHELL_REQUIRES_DRY_RUN"],
            ["shell.high_risk.dry_run"],
        )

    return PolicyDecision("allow", ["POLICY_OK"], ["default.allow"])
```

再定义规则清单和 fixture：

```python
ALL_RULES = {
    "memory.group_chat.deny",
    "git.main.require_approval",
    "shell.high_risk.dry_run",
    "default.allow",
}

fixtures = [
    PolicyInput("owner", "group", "memory_write", "memory", "high"),
    PolicyInput("cron", "private", "git_push", "main", "high"),
    PolicyInput("agent", "private", "shell", "local", "high"),
]
```

统计覆盖：

```python
from collections import defaultdict

coverage = defaultdict(int)
action_matrix = defaultdict(set)

for f in fixtures:
    d = decide(f)
    for rule in d.matched_rules:
        coverage[rule] += 1

    key = (f.tool, f.target, f.risk)
    action_matrix[key].add(d.action)

missing_rules = ALL_RULES - set(coverage.keys())

if missing_rules:
    raise AssertionError(f"Policy rules without fixtures: {sorted(missing_rules)}")

for key, actions in action_matrix.items():
    print(key, sorted(actions))
```

这里 `default.allow` 很可能会被报缺失。这个提醒很有价值：说明我们只测了拒绝/降级路径，没有测正常低风险 allow 路径。

## 3. 死规则检测：不只是没覆盖

“没覆盖”不一定是死规则，也可能只是测试漏了。死规则指的是：**无论输入怎么变化，这条规则都触发不到**。

常见原因是规则顺序挡住了后面的规则：

```python
def decide_bad(i: PolicyInput) -> PolicyDecision:
    if i.tool == "shell":
        return PolicyDecision("allow", ["SHELL_OK"], ["shell.allow"])

    # 这条永远触发不到，因为所有 shell 都已经被上面 return 了
    if i.tool == "shell" and i.risk == "high":
        return PolicyDecision("dry_run", ["HIGH_RISK_SHELL"], ["shell.high_risk.dry_run"])
```

教学版可以用“输入空间采样”检测：

```python
actors = ["owner", "cron", "guest"]
channels = ["private", "group"]
tools = ["message_send", "memory_write", "git_push", "shell"]
targets = ["main", "feature", "prod", "local"]
risks = ["low", "medium", "high"]

seen = set()

for actor in actors:
    for channel in channels:
        for tool in tools:
            for target in targets:
                for risk in risks:
                    d = decide(PolicyInput(actor, channel, tool, target, risk))
                    seen.update(d.matched_rules)

dead_rules = ALL_RULES - seen
if dead_rules:
    raise AssertionError(f"Potential dead policy rules: {sorted(dead_rules)}")
```

这不是形式化证明，但在 Agent 工程里已经能抓到大量低级事故。

## 4. pi-mono：生产版 Coverage Gate

生产里可以把策略评估统一打点：

```ts
type PolicyAction = 'allow' | 'deny' | 'require_approval' | 'dry_run'

interface PolicyDecision {
  action: PolicyAction
  reasonCodes: string[]
  matchedRules: string[]
}

interface CoverageEvent {
  policyVersion: string
  tool: string
  target: string
  risk: 'low' | 'medium' | 'high'
  action: PolicyAction
  reasonCodes: string[]
  matchedRules: string[]
}

export class PolicyCoverageRecorder {
  constructor(private readonly sink: CoverageSink) {}

  async record(input: PolicyInput, decision: PolicyDecision, policyVersion: string) {
    await this.sink.append({
      policyVersion,
      tool: input.tool,
      target: input.target,
      risk: input.risk,
      action: decision.action,
      reasonCodes: decision.reasonCodes,
      matchedRules: decision.matchedRules,
    })
  }
}
```

发布前跑 Coverage Gate：

```ts
export class PolicyCoverageGate {
  constructor(
    private readonly rules: PolicyRuleRegistry,
    private readonly fixtureStore: PolicyFixtureStore,
    private readonly policy: PolicyEngine,
  ) {}

  async check(policyVersion: string) {
    const allRules = await this.rules.listRuleIds(policyVersion)
    const fixtures = await this.fixtureStore.list({ tags: ['side_effect', 'high_risk'] })

    const seenRules = new Set<string>()
    const matrix = new Map<string, Set<PolicyAction>>()

    for (const fixture of fixtures) {
      const decision = await this.policy.decide(fixture.input, { policyVersion })

      for (const rule of decision.matchedRules) seenRules.add(rule)

      const key = [fixture.input.tool, fixture.input.target, fixture.input.risk].join(':')
      const actions = matrix.get(key) ?? new Set<PolicyAction>()
      actions.add(decision.action)
      matrix.set(key, actions)
    }

    const missing = allRules.filter(rule => !seenRules.has(rule))

    return missing.length === 0
      ? { action: 'allow', coveredRules: seenRules.size }
      : { action: 'manual_review', missingRules: missing }
  }
}
```

注意：缺覆盖不一定直接 block。推荐策略：

- 高风险规则缺覆盖：`block`
- 中风险规则缺覆盖：`manual_review`
- 低风险默认 allow 缺覆盖：`warn`

## 5. OpenClaw 实战：课程 cron 也能用

以这套 Agent 开发课程 cron 为例，至少应该覆盖这些策略组合：

```text
message.send × telegram_group × low_risk_lesson       -> allow
message.send × telegram_group × private_memory        -> deny
file.write    × lessons_dir     × course_content      -> allow
git.push      × main            × explicit_cron_task   -> allow_after_pr_check_or_explicit_request
git.push      × wrong_account   × any                 -> deny
memory.write  × TOOLS.md        × taught_topic_update  -> allow
```

每次课程发布前，可以把关键 gate 做成小脚本：

```bash
policy-coverage check \
  --fixtures policy-fixtures/course-cron/*.json \
  --policy-version "$GIT_SHA" \
  --require-rules message.telegram_group.allow,git.account.gfwfail
```

这样以后改 Telegram 权限、git push 规则、记忆写入规则时，不会只靠 prompt 里的“记得要小心”。

## 6. 实战建议

- 策略函数要返回 `matchedRules`，否则覆盖统计只能猜
- fixture 既要覆盖 deny，也要覆盖 allow；只测拦截会让正常路径退化
- rule id 要稳定，别用自然语言句子当 id
- 高风险规则必须有 fixture；默认 allow 也至少要有一条 smoke fixture
- 覆盖矩阵按风险分级，不要为了 100% 覆盖写一堆无意义测试
- 死规则检测用采样 + 线上 telemetry 双保险

## 总结

策略测试回答的是：这组 fixture 过了吗？

策略覆盖矩阵回答的是：**我们到底测到了哪些风险空间？**

Dead Rule Detection 回答的是：**这些规则真的有机会生效吗？**

成熟 Agent 的策略系统，不只是能拦截危险动作，还要能证明每条关键护栏有人测、能触发、不会悄悄死掉。