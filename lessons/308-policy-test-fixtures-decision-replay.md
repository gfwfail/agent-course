# 第 308 课：Agent 策略测试夹具与决策回放（Policy Test Fixtures & Decision Replay）

上一课讲了 `PolicyDecision` 不能只返回 allow/deny，还要带可审计理由。今天继续往前走一步：**策略要能被测试、被回放、被证明没有悄悄变危险**。

很多 Agent 事故不是工具坏了，而是策略改了一行之后：

- 原来 `require_approval` 的部署变成了 `allow`
- 原来群聊低权限不能写长期记忆，现在误判成高权限
- 原来高风险 shell 命令要 dry-run，现在直接执行

所以成熟 Agent 需要一套 **Policy Test Fixtures + Decision Replay**：把历史真实请求、上下文、证据、期望决策保存下来，策略升级前批量回放。

## 1. 核心思想

把策略判断看成一个纯函数：

```text
PolicyInput + PolicyVersion -> PolicyDecision
```

每次线上遇到关键决策，都可以沉淀成 fixture：

```json
{
  "id": "deploy-prod-needs-approval",
  "input": {
    "actor": "cron-agent",
    "channel": "telegram",
    "tool": "deploy",
    "target": "prod",
    "risk": "high",
    "hasApproval": false
  },
  "expected": {
    "action": "require_approval",
    "reasonCodes": ["HIGH_RISK_TARGET", "MISSING_APPROVAL"]
  }
}
```

策略变更时，不是靠感觉 review，而是跑：

```bash
policy-replay fixtures/*.json --policy ./policy.py
```

只要出现危险放宽，就阻断发布。

## 2. learn-claude-code：最小 Python 教学版

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
    has_approval: bool

@dataclass
class PolicyDecision:
    action: Action
    reason_codes: list[str]


def decide(i: PolicyInput) -> PolicyDecision:
    reasons: list[str] = []

    if i.channel == "group" and i.tool == "memory_write":
        return PolicyDecision("deny", ["GROUP_CANNOT_WRITE_LONG_TERM_MEMORY"])

    if i.target == "prod" and i.risk == "high":
        reasons.append("HIGH_RISK_TARGET")
        if not i.has_approval:
            reasons.append("MISSING_APPROVAL")
            return PolicyDecision("require_approval", reasons)

    if i.tool == "shell" and i.risk != "low":
        return PolicyDecision("dry_run", ["SHELL_SIDE_EFFECT_REQUIRES_DRY_RUN"])

    return PolicyDecision("allow", ["POLICY_OK"])
```

回放测试：

```python
import json
from pathlib import Path

RANK = {
    "deny": 0,
    "require_approval": 1,
    "dry_run": 2,
    "allow": 3,
}


def dangerous_relaxation(old: str, new: str) -> bool:
    return RANK[new] > RANK[old]


def load_input(raw: dict) -> PolicyInput:
    return PolicyInput(
        actor=raw["actor"],
        channel=raw["channel"],
        tool=raw["tool"],
        target=raw["target"],
        risk=raw["risk"],
        has_approval=raw.get("hasApproval", False),
    )


def replay_fixture(path: Path) -> None:
    fixture = json.loads(path.read_text())
    actual = decide(load_input(fixture["input"]))
    expected = fixture["expected"]

    if dangerous_relaxation(expected["action"], actual.action):
        raise AssertionError(
            f"{fixture['id']} became less safe: "
            f"{expected['action']} -> {actual.action}"
        )

    missing_reasons = set(expected.get("reasonCodes", [])) - set(actual.reason_codes)
    if missing_reasons:
        raise AssertionError(f"{fixture['id']} lost reasons: {missing_reasons}")


for p in Path("fixtures").glob("*.json"):
    replay_fixture(p)
```

这里的重点不是“结果必须一字不差”，而是：

- 不能从 `deny/approval/dry_run` 危险放宽到 `allow`
- 关键 reason code 不能丢
- 低权限来源不能被策略升级误判成高权限

## 3. pi-mono：生产版中间件

生产里不要把 replay 写成散落脚本，可以做成 `PolicyReplayGate`。

```ts
type PolicyAction = 'deny' | 'require_approval' | 'dry_run' | 'allow'

interface PolicyFixture {
  id: string
  tags: string[]
  input: PolicyInput
  expected: {
    action: PolicyAction
    reasonCodes: string[]
  }
}

const rank: Record<PolicyAction, number> = {
  deny: 0,
  require_approval: 1,
  dry_run: 2,
  allow: 3,
}

function isDangerousRelaxation(expected: PolicyAction, actual: PolicyAction) {
  return rank[actual] > rank[expected]
}

export class PolicyReplayGate {
  constructor(
    private readonly policy: PolicyEngine,
    private readonly store: FixtureStore,
  ) {}

  async checkChangedPolicy(policyVersion: string) {
    const fixtures = await this.store.list({ tags: ['side_effect', 'memory', 'prod'] })
    const failures = []

    for (const fixture of fixtures) {
      const actual = await this.policy.decide(fixture.input, { policyVersion })

      if (isDangerousRelaxation(fixture.expected.action, actual.action)) {
        failures.push({
          id: fixture.id,
          type: 'dangerous_relaxation',
          expected: fixture.expected.action,
          actual: actual.action,
        })
      }

      const lostReasons = fixture.expected.reasonCodes
        .filter(code => !actual.reasonCodes.includes(code))

      if (lostReasons.length > 0) {
        failures.push({ id: fixture.id, type: 'lost_reason_codes', lostReasons })
      }
    }

    return failures.length === 0
      ? { action: 'allow', replayed: fixtures.length }
      : { action: 'block', replayed: fixtures.length, failures }
  }
}
```

接入发布门控：

```ts
const replay = await policyReplayGate.checkChangedPolicy(newPolicyVersion)

if (replay.action === 'block') {
  throw new Error(`Policy replay failed: ${JSON.stringify(replay.failures)}`)
}
```

这样策略升级就不再是“我觉得没问题”，而是“历史高风险案例都回放通过”。

## 4. OpenClaw 实战：把真实 cron 决策变成 fixture

OpenClaw 这类 Always-on Agent 很适合沉淀 fixture：

- Telegram 发群消息前：记录 channel、target、message type、权限来源
- git push 前：记录 repo、branch、PR 状态、是否 main、是否要求 direct push
- MEMORY.md 写入前：记录会话类型、操作者、source authority
- deploy 前：记录环境、风险、审批状态、回滚证据

一个文件式目录即可：

```text
policy-fixtures/
  telegram-send-group-low-risk.json
  git-push-main-requires-pr-check.json
  memory-write-group-chat-deny.json
  deploy-prod-high-risk-requires-approval.json
```

高价值 fixture 来源：

1. 真实事故复盘
2. 老板明确要求的铁律
3. 高风险工具调用前的审批案例
4. 曾经差点做错的边界场景

## 5. 实战建议

- fixture 要小：只保留策略判断需要的字段
- fixture 要分类：`prod` / `memory` / `group_chat` / `shell` / `deploy`
- replay 要进 CI：策略、工具权限、系统提示词变更都跑
- 危险放宽默认阻断；过度收紧可以人工 review
- reason code 也要测，因为它影响审计、修复建议和用户解释

## 总结

策略系统真正成熟的标志不是“规则很多”，而是：

> 每次改规则，都能回放历史关键场景，证明没有把 Agent 放得更危险。

Policy Fixture 是策略的单元测试；Decision Replay 是策略升级的回归门控。没有 replay 的策略升级，本质上是在拿生产流量做实验。
