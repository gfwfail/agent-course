# 321. Agent 证据作用域绑定与防复用（Evidence Scope Binding & Replay Prevention）

上一课讲了 Evidence Freshness：证据不能过期，副作用前要重新确认。

今天再补一个很容易被忽略的坑：**证据新鲜，不代表能拿来证明任何动作。**

比如：

- 用户批准“发一条课程消息”，不能拿来证明“删除一批消息”也被批准
- `git status clean` 是在 `agent-course` 仓库查的，不能拿去证明另一个仓库能 push
- 某次部署健康检查通过，不能拿来证明另一个环境也健康
- `gh pr list` 检查的是 `gfwfail/agent-course`，不能复用到 `meta-draw/mysterybox`

这就是 Evidence Scope Binding：**每条证据都要绑定它证明的 actor / action / target / args / env / run，不允许跨作用域复用。**

## 1. 为什么要防“证据复用”？

很多 Agent 审计链看起来很完整：

1. 有用户意图
2. 有计划
3. 有审批
4. 有工具检查
5. 有执行结果

但细看会发现：这些证据可能不是为同一个动作产生的。

危险例子：

```text
approval: 用户批准发送 Telegram 消息
precheck: git status clean
action: git push origin main
```

这三块证据都是真的，也都没过期，但它们证明的事情不一样。

成熟 Agent 的证据校验不能只问：

- 证据是否存在？
- 证据是否新鲜？
- 证据链是否完整？

还必须问：

- 这条证据是不是为当前 action 生成的？
- target 是否一致？
- 参数 hash 是否一致？
- 环境和账号是否一致？
- 是否允许跨 run 复用？

## 2. Scope Binding 的核心字段

一条可用于执行前证明的 Evidence，至少应该绑定这些字段：

| 字段 | 说明 | 示例 |
|---|---|---|
| `actorId` | 谁在执行 | `bot001` / `user:67431246` |
| `runId` | 哪次 Agent run | `cron:3eba...` |
| `action` | 要证明的动作类型 | `telegram.send` / `git.push` |
| `target` | 动作目标 | `group:-5115329245` / `repo:gfwfail/agent-course` |
| `argsHash` | 规范化参数 hash | 消息正文、branch、remote 等 |
| `env` | 环境/账号/区域 | `gh:gfwfail` / `prod` |
| `risk` | 风险等级 | `low` / `high` |
| `reusePolicy` | 是否允许复用 | `same_action_only` / `same_run_only` |

重点是 `argsHash`：不要直接比较原始字符串，要先 canonicalize。

例如 Telegram 消息要 hash：

- channel id
- normalized text
- media/file hash
- reply target
- silent / document 等投递选项

Git push 要 hash：

- repo
- remote
- branch
- expected HEAD
- working tree clean proof id
- GitHub account

## 3. learn-claude-code：最小 Scope Gate

先定义证据作用域。

```python
# learn-claude-code: evidence_scope.py
from dataclasses import dataclass
from typing import Literal

ReusePolicy = Literal[
    "same_action_only",
    "same_target_only",
    "same_run_only",
    "no_reuse",
]

@dataclass(frozen=True)
class EvidenceScope:
    actor_id: str
    run_id: str
    action: str
    target: str
    args_hash: str
    env: str
    risk: str
    reuse_policy: ReusePolicy = "same_action_only"

@dataclass
class EvidenceEnvelope:
    id: str
    kind: str
    scope: EvidenceScope
    payload_hash: str
```

然后把“当前即将执行的动作”也规范化成 scope。

```python
@dataclass(frozen=True)
class ActionScope:
    actor_id: str
    run_id: str
    action: str
    target: str
    args_hash: str
    env: str
    risk: str


def check_scope(evidence: EvidenceEnvelope, current: ActionScope) -> tuple[bool, str]:
    s = evidence.scope

    if s.actor_id != current.actor_id:
        return False, "actor mismatch"

    if s.env != current.env:
        return False, "environment/account mismatch"

    if s.reuse_policy == "no_reuse":
        # no_reuse 证据必须由当前 action exact match 消费，并且消费后标记 used
        if s.run_id != current.run_id:
            return False, "no_reuse evidence from another run"

    if s.reuse_policy == "same_run_only" and s.run_id != current.run_id:
        return False, "run mismatch"

    if s.action != current.action:
        return False, "action mismatch"

    if s.target != current.target:
        return False, "target mismatch"

    if s.args_hash != current.args_hash:
        return False, "args hash mismatch"

    return True, "scope matched"
```

这个版本故意保守：必须 action / target / args 全匹配。

实际生产可以按证据类型放宽：

- `repo_status_clean`：允许同 run 内复用，但必须 repo + branch + HEAD 一致
- `approval_receipt`：通常 `no_reuse`，一次审批只消费一次
- `policy_decision`：必须 action + argsHash 一致
- `health_check`：允许同 target + env 下短时间复用
- `delivery_receipt`：可以跨 run 引用，但只能证明“已投递”，不能证明“可再次投递”

## 4. 防 replay：证据消费账本

`no_reuse` 证据需要消费账本，否则同一张 approval 可以被重复刷。

```python
class EvidenceConsumptionLedger:
    def __init__(self):
        self.used: set[str] = set()

    def consume_once(self, evidence: EvidenceEnvelope, current: ActionScope):
        ok, reason = check_scope(evidence, current)
        if not ok:
            raise ValueError(f"scope check failed: {reason}")

        if evidence.scope.reuse_policy == "no_reuse":
            if evidence.id in self.used:
                raise ValueError(f"evidence already consumed: {evidence.id}")
            self.used.add(evidence.id)

        return True
```

教学版用内存 set；生产版要用 Redis / DB 的唯一索引：

```sql
CREATE UNIQUE INDEX evidence_consumption_once
ON evidence_consumptions(evidence_id)
WHERE reuse_policy = 'no_reuse';
```

这能防止并发 Agent 同时拿同一张审批去执行两次。

## 5. pi-mono：ProofScopeMiddleware

在 pi-mono 里，Scope Gate 适合放在 Tool Gateway 的 proof 校验阶段。

```ts
// pi-mono: ProofScopeMiddleware.ts
type ReusePolicy = 'same_action_only' | 'same_target_only' | 'same_run_only' | 'no_reuse'

type EvidenceScope = {
  actorId: string
  runId: string
  action: string
  target: string
  argsHash: string
  env: string
  risk: 'low' | 'medium' | 'high' | 'critical'
  reusePolicy: ReusePolicy
}

type EvidenceEnvelope = {
  id: string
  kind: string
  scope: EvidenceScope
  payloadHash: string
}

export class ProofScopeMiddleware {
  constructor(
    private proofStore: ProofStore,
    private ledger: EvidenceConsumptionLedger,
    private audit: AuditLog,
  ) {}

  async beforeToolCall(ctx: ToolCallContext) {
    const current: EvidenceScope = {
      actorId: ctx.actor.id,
      runId: ctx.runId,
      action: ctx.tool.name,
      target: canonicalTarget(ctx.tool.name, ctx.args),
      argsHash: hashCanonicalArgs(ctx.tool.name, ctx.args),
      env: ctx.environment.fingerprint,
      risk: ctx.risk.level,
      reusePolicy: 'same_action_only',
    }

    const proof = await this.proofStore.load(ctx.proofId)

    for (const evidence of proof.items) {
      const decision = matchEvidenceScope(evidence.scope, current, evidence.kind)

      await this.audit.append({
        type: 'evidence_scope_check',
        runId: ctx.runId,
        tool: ctx.tool.name,
        evidenceId: evidence.id,
        decision,
      })

      if (!decision.ok) {
        throw new Error(`Evidence ${evidence.id} cannot prove this action: ${decision.reason}`)
      }

      if (evidence.scope.reusePolicy === 'no_reuse') {
        await this.ledger.consumeOnce({
          evidenceId: evidence.id,
          runId: ctx.runId,
          action: ctx.tool.name,
          argsHash: current.argsHash,
        })
      }
    }
  }
}
```

注意：Scope Gate 不应该相信 LLM 传来的 `target` 和 `argsHash`，必须由工具层自己 canonicalize。

## 6. OpenClaw 实战：课程 Cron 的两个 proof

以这次课程 Cron 为例，至少有两个外部副作用：

1. 发 Telegram 群消息
2. git push 课程仓库

它们的 proof 不能混用。

### Telegram proof scope

```json
{
  "kind": "delivery_intent",
  "scope": {
    "actorId": "bot001",
    "runId": "cron:agent-course:2026-05-15T06:30:00+11:00",
    "action": "telegram.send",
    "target": "telegram:-5115329245",
    "argsHash": "sha256(normalized lesson text + no media)",
    "env": "telegram:main",
    "risk": "medium",
    "reusePolicy": "no_reuse"
  }
}
```

### Git push proof scope

```json
{
  "kind": "git_preflight",
  "scope": {
    "actorId": "bot001",
    "runId": "cron:agent-course:2026-05-15T06:30:00+11:00",
    "action": "git.push",
    "target": "repo:gfwfail/agent-course:main",
    "argsHash": "sha256(remote + branch + expectedHead + ghAccount)",
    "env": "mac:BOT001|gh:gfwfail",
    "risk": "high",
    "reusePolicy": "same_action_only"
  }
}
```

如果有人试图拿 Telegram 的 proof 去证明 git push，Scope Gate 会直接挡住：`action mismatch`。

如果有人改了消息正文但复用旧 approval，Scope Gate 会挡住：`args hash mismatch`。

如果 GitHub 账号从 `gfwfail` 变成 `metaclaw01`，Scope Gate 会挡住：`environment/account mismatch`。

## 7. 实战 checklist

给 Agent 证据系统加 Scope Binding 时，先落地这 7 条：

- [ ] 所有可执行前 proof 都带 `EvidenceScope`
- [ ] `target` 和 `argsHash` 由工具层 canonicalize，不由 LLM 手写
- [ ] 高风险审批默认 `reusePolicy = no_reuse`
- [ ] `no_reuse` 证据写消费账本，DB 唯一索引防并发 replay
- [ ] `env` 包含账号、环境、区域、仓库等关键执行上下文
- [ ] scope mismatch 要进入 audit log，方便追查被挡住的复用尝试
- [ ] delivery receipt / commit hash 这类历史回执只能证明“已发生”，不能证明“可再次执行”

## 总结

Evidence Scope Binding 的核心不是“多存几个字段”，而是把证据从一句泛泛的“我检查过”升级成：

> 谁，在什么 run、什么环境下，为哪个动作、哪个目标、哪组参数生成了这条证据。

成熟 Agent 不只要有证据、新鲜证据、完整证据链，还要保证：**证据只能证明它本来要证明的那件事。**
