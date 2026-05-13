# 第 310 课：Agent 策略变异测试与护栏强度验证（Policy Mutation Testing & Guardrail Strength）

上一课讲了策略覆盖矩阵：证明每条关键规则“被测到”。但覆盖率有一个经典盲区：**测到了，不代表测得有用**。

比如 fixture 确实跑到了 `git.main.require_approval`，但测试只断言 `action != "deny"`；这时哪怕把规则改坏成 `allow`，测试也可能继续绿。

所以今天讲一招特别实用的策略工程方法：**Policy Mutation Testing**。

它的目标不是再写更多 happy path 测试，而是故意把策略规则“改坏”，看现有测试能不能把坏策略杀死。

## 1. 核心思想

普通测试问：

> 当前策略对 fixture 的输出是否符合预期？

变异测试问：

> 如果我把关键护栏偷偷削弱，测试会不会失败？

典型策略变异包括：

- `deny` 变成 `allow`
- `require_approval` 变成 `allow`
- 删除一个 `reason_code`
- 把 `risk == "high"` 改成 `risk == "critical"`
- 把 `channel == "group"` 改成 `channel != "group"`
- 跳过 `has_approval` 检查

如果变异后的策略还能通过测试，说明测试没有真正保护这条规则，这个变异叫 **survived mutant**。

成熟 Agent 的策略测试，不只追求 test pass，还要追求：

```text
高风险护栏的 mutant 必须被 fixture 杀死
```

## 2. learn-claude-code：最小 Python 教学版

先定义一个策略和 fixture：

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


def decide(i: PolicyInput) -> PolicyDecision:
    if i.channel == "group" and i.tool == "memory_write":
        return PolicyDecision("deny", ["GROUP_MEMORY_WRITE_DENIED"])

    if i.tool == "git_push" and i.target == "main" and not i.has_approval:
        return PolicyDecision("require_approval", ["MAIN_PUSH_REQUIRES_APPROVAL"])

    if i.tool == "shell" and i.risk == "high":
        return PolicyDecision("dry_run", ["HIGH_RISK_SHELL_DRY_RUN"])

    return PolicyDecision("allow", ["POLICY_OK"])
```

fixture 不要只断言 action，最好同时断言 reason code：

```python
fixtures = [
    (
        PolicyInput("agent", "group", "memory_write", "MEMORY.md", "high"),
        PolicyDecision("deny", ["GROUP_MEMORY_WRITE_DENIED"]),
    ),
    (
        PolicyInput("cron", "private", "git_push", "main", "high"),
        PolicyDecision("require_approval", ["MAIN_PUSH_REQUIRES_APPROVAL"]),
    ),
    (
        PolicyInput("agent", "private", "shell", "prod", "high"),
        PolicyDecision("dry_run", ["HIGH_RISK_SHELL_DRY_RUN"]),
    ),
]


def run_fixtures(policy_fn):
    failures = []
    for input_, expected in fixtures:
        actual = policy_fn(input_)
        if actual.action != expected.action:
            failures.append((input_, "action", expected.action, actual.action))
        if set(expected.reason_codes) - set(actual.reason_codes):
            failures.append((input_, "reason_codes", expected.reason_codes, actual.reason_codes))
    return failures
```

再手写几个“坏策略变异”：

```python
def mutant_main_push_allows(i: PolicyInput) -> PolicyDecision:
    if i.channel == "group" and i.tool == "memory_write":
        return PolicyDecision("deny", ["GROUP_MEMORY_WRITE_DENIED"])

    # 变异：main push 不再要求审批，危险放宽
    if i.tool == "git_push" and i.target == "main" and not i.has_approval:
        return PolicyDecision("allow", ["POLICY_OK"])

    if i.tool == "shell" and i.risk == "high":
        return PolicyDecision("dry_run", ["HIGH_RISK_SHELL_DRY_RUN"])

    return PolicyDecision("allow", ["POLICY_OK"])


def mutant_group_memory_allows(i: PolicyInput) -> PolicyDecision:
    # 变异：删除 group memory_write deny 护栏
    if i.tool == "git_push" and i.target == "main" and not i.has_approval:
        return PolicyDecision("require_approval", ["MAIN_PUSH_REQUIRES_APPROVAL"])

    if i.tool == "shell" and i.risk == "high":
        return PolicyDecision("dry_run", ["HIGH_RISK_SHELL_DRY_RUN"])

    return PolicyDecision("allow", ["POLICY_OK"])
```

变异测试 runner：

```python
mutants = {
    "main_push_allows": mutant_main_push_allows,
    "group_memory_allows": mutant_group_memory_allows,
}

survived = []

for name, mutant in mutants.items():
    failures = run_fixtures(mutant)
    if not failures:
        survived.append(name)

if survived:
    raise AssertionError(f"Policy mutants survived: {survived}")
```

如果 `main_push_allows` 没被杀死，说明你的测试虽然覆盖了 main push 场景，但没有强断言“必须 require approval”。

## 3. 自动生成变异：从规则元数据下手

生产里不一定真的改源码。更稳的方式是让策略规则声明可变异字段：

```python
rules = [
    {
        "id": "git.main.require_approval",
        "when": {"tool": "git_push", "target": "main", "has_approval": False},
        "then": {"action": "require_approval", "reason": "MAIN_PUSH_REQUIRES_APPROVAL"},
        "risk": "high",
        "mutations": ["weaken_action", "drop_condition", "drop_reason"],
    },
    {
        "id": "memory.group.deny",
        "when": {"channel": "group", "tool": "memory_write"},
        "then": {"action": "deny", "reason": "GROUP_MEMORY_WRITE_DENIED"},
        "risk": "high",
        "mutations": ["weaken_action", "invert_condition"],
    },
]
```

变异器可以这样做：

```python
def mutate_rule(rule):
    if "weaken_action" in rule["mutations"]:
        weakened = dict(rule)
        weakened["then"] = dict(rule["then"], action="allow")
        yield rule["id"] + ":weaken_action", weakened

    if "drop_reason" in rule["mutations"]:
        changed = dict(rule)
        changed["then"] = dict(rule["then"], reason="POLICY_OK")
        yield rule["id"] + ":drop_reason", changed
```

关键点：**高风险 deny / require_approval / dry_run 规则，至少要有一个 fixture 能杀死对应 weaken_action 变异。**

## 4. pi-mono：生产版 Mutation Gate

在 pi-mono 里可以把它做成发布门控，而不是单独脚本。

```ts
type PolicyAction = 'allow' | 'deny' | 'require_approval' | 'dry_run'

type MutationKind =
  | 'weaken_action'
  | 'drop_condition'
  | 'invert_condition'
  | 'drop_reason'

interface PolicyRule {
  id: string
  risk: 'low' | 'medium' | 'high'
  action: PolicyAction
  reasonCode: string
  mutationKinds: MutationKind[]
}

interface MutationResult {
  ruleId: string
  mutationKind: MutationKind
  killed: boolean
  killedByFixtureIds: string[]
}

export class PolicyMutationGate {
  constructor(
    private readonly policyFactory: PolicyFactory,
    private readonly fixtureStore: PolicyFixtureStore,
  ) {}

  async run(policyVersion: string): Promise<MutationResult[]> {
    const rules = await this.policyFactory.rules(policyVersion)
    const fixtures = await this.fixtureStore.loadHighRiskFixtures()
    const results: MutationResult[] = []

    for (const rule of rules.filter(r => r.risk === 'high')) {
      for (const kind of rule.mutationKinds) {
        const mutant = await this.policyFactory.withMutation(policyVersion, rule.id, kind)
        const killedBy: string[] = []

        for (const fixture of fixtures) {
          const actual = await mutant.decide(fixture.input)
          if (!fixture.expected.matches(actual)) {
            killedBy.push(fixture.id)
          }
        }

        results.push({
          ruleId: rule.id,
          mutationKind: kind,
          killed: killedBy.length > 0,
          killedByFixtureIds: killedBy,
        })
      }
    }

    return results
  }

  assertPass(results: MutationResult[]) {
    const survived = results.filter(r => !r.killed)
    if (survived.length > 0) {
      throw new Error(
        `Policy mutation gate failed: ${survived.map(r => `${r.ruleId}:${r.mutationKind}`).join(', ')}`,
      )
    }
  }
}
```

这类 gate 最适合放在策略发布、工具权限变更、审批逻辑变更之前。

## 5. OpenClaw 实战：课程 Cron 的护栏怎么测

以这个课程 Cron 为例，关键策略不是“能不能写 Markdown”，而是这些护栏：

- 不能重复已讲主题
- 群消息只能发课程内容，不能泄漏私有记忆或密钥
- push 前必须更新 lesson、README、TOOLS
- git push 必须使用正确账号 `gfwfail`
- 完成前必须有 messageId / commit / push 证据

可以为这些护栏写 fixture：

```json
[
  {
    "id": "telegram_group_no_memory_leak",
    "input": {
      "channel": "telegram_group",
      "tool": "message.send",
      "contentTags": ["lesson", "private_memory"]
    },
    "expected": {
      "action": "deny",
      "reasonCodes": ["GROUP_MESSAGE_CANNOT_INCLUDE_PRIVATE_MEMORY"]
    }
  },
  {
    "id": "push_requires_gfwfail_account",
    "input": {
      "tool": "git.push",
      "repo": "gfwfail/agent-course",
      "ghUser": "wrong-user"
    },
    "expected": {
      "action": "deny",
      "reasonCodes": ["REPO_PUSH_REQUIRES_EXPECTED_GH_USER"]
    }
  }
]
```

然后变异：

- 把 `GROUP_MESSAGE_CANNOT_INCLUDE_PRIVATE_MEMORY` 改成 `allow`
- 把 `ghUser == gfwfail` 条件删除
- 把 `README_UPDATED_REQUIRED` reason 删除

如果这些 mutant 还能通过测试，说明课程 Cron 的验收闸门只是形式主义。

## 6. 实战建议

不要一开始就追求全量 mutation testing，成本会爆炸。建议分三层：

1. **只测 high risk 规则**：deny / require_approval / dry_run
2. **只测危险放宽变异**：block → allow、approval → allow、dry_run → allow
3. **只在发布前跑完整 gate**：日常开发跑 smoke mutation，合并前跑 full mutation

一个简单门槛：

```text
high_risk_mutation_score = killed_high_risk_mutants / total_high_risk_mutants
要求 >= 0.95
任何 side_effect allow 变异 survived，直接阻断发布
```

## 7. 小结

策略覆盖矩阵告诉你：规则有没有被测到。

策略变异测试告诉你：测试有没有能力发现规则被削弱。

Agent 越自主，策略越不能只靠“看起来写了规则”。真正可靠的护栏，必须能经受住故意破坏：

```text
能被测试杀死的坏策略，才说明好策略真的被保护住了。
```
