# 322. Agent 证据撤销与失效传播（Evidence Revocation & Invalidation Propagation）

上一课讲了 Evidence Scope Binding：证据必须绑定 action / target / args / env，不能跨作用域复用。

今天继续补一层生产级护栏：**证据即使没过期、作用域也匹配，也可能已经被撤销。**

典型场景：

- 用户刚刚批准了部署，但 2 分钟后说“先别发”
- GitHub token 被轮换，旧的 `gh auth status` 证明不能再用
- PR 已经被 force-push，之前的 test pass 证据不再对应当前 commit
- 监控健康检查通过后，Incident Command Center 把服务切到 `freeze`
- 一条审批被发现发错目标群，需要立刻让下游所有 proof 失效

这就是 Evidence Revocation：**证据不是只会自然过期，还要支持主动撤销，并把撤销影响传播到依赖它的证据链。**

## 1. 为什么 freshness 不够？

Freshness Gate 问的是：

> 这条证据是不是足够新？

Scope Gate 问的是：

> 这条证据是不是证明当前动作？

Revocation Gate 还要问：

> 这条证据在刚才到现在之间，有没有被主动作废？

如果没有撤销机制，Agent 很容易出现“拿着旧世界的证明，去执行新世界已经禁止的动作”。

危险例子：

```text
09:00 approval: 允许 push main
09:01 policy: allow
09:02 user: 暂停发布，先别 push
09:03 agent: approval 还没过期，scope 也匹配，于是继续 git push
```

这里问题不在证据缺失，也不在证据过期，而是 **撤销事件没有进入执行闸门**。

## 2. Revocation 的核心模型

不要直接改历史证据内容；历史应该保持不可变。

正确做法是追加一条 revocation event：

```json
{
  "revocationId": "rev_20260515_0930_001",
  "evidenceId": "ev_approval_123",
  "reasonCode": "user_cancelled",
  "scope": {
    "target": "repo:gfwfail/agent-course",
    "action": "git.push"
  },
  "revokedBy": "user:67431246",
  "revokedAt": "2026-05-15T09:02:00+11:00",
  "propagate": true
}
```

这样做有三个好处：

1. 原始证据仍然可审计
2. 撤销本身也是证据
3. 下游可以根据 `evidenceId` / `scope` / `propagate` 判断是否失效

## 3. learn-claude-code：最小撤销注册表

教学版先做一个内存 registry。

```python
# learn-claude-code: evidence_revocation.py
from dataclasses import dataclass, field
from datetime import datetime, timezone
from typing import Literal

ReasonCode = Literal[
    "user_cancelled",
    "credential_rotated",
    "target_changed",
    "policy_changed",
    "incident_freeze",
    "superseded",
]

@dataclass(frozen=True)
class RevocationEvent:
    revocation_id: str
    evidence_id: str
    reason_code: ReasonCode
    revoked_by: str
    revoked_at: datetime
    propagate: bool = True
    note: str = ""

@dataclass
class EvidenceEnvelope:
    id: str
    kind: str
    parent_ids: list[str] = field(default_factory=list)
    payload_hash: str = ""

class RevocationRegistry:
    def __init__(self):
        self.by_evidence: dict[str, RevocationEvent] = {}

    def revoke(self, event: RevocationEvent):
        # 幂等：同一 evidence 重复撤销，只保留第一条事实
        self.by_evidence.setdefault(event.evidence_id, event)

    def get(self, evidence_id: str) -> RevocationEvent | None:
        return self.by_evidence.get(evidence_id)
```

执行前检查当前 proof bag：

```python
def check_revocation(
    proof: list[EvidenceEnvelope],
    registry: RevocationRegistry,
) -> tuple[bool, list[str]]:
    errors: list[str] = []

    for ev in proof:
        revoked = registry.get(ev.id)
        if revoked:
            errors.append(
                f"{ev.id} revoked: {revoked.reason_code} by {revoked.revoked_by}"
            )

    return len(errors) == 0, errors
```

这只能检查“直接被撤销”的证据。

生产里更重要的是：**如果父证据被撤销，子证据也可能失效。**

## 4. 失效传播：从点撤销到链撤销

假设证据链是这样：

```text
approval(ev_a)
  -> policy_decision(ev_b)
      -> tool_precheck(ev_c)
          -> action_receipt(ev_d)
```

如果 `ev_a` 被撤销，`ev_b/ev_c/ev_d` 不能继续被当作有效证明。

可以用一个简单的 DAG 遍历做传播：

```python
def build_children_index(evidence: list[EvidenceEnvelope]) -> dict[str, list[str]]:
    children: dict[str, list[str]] = {}
    for ev in evidence:
        for parent_id in ev.parent_ids:
            children.setdefault(parent_id, []).append(ev.id)
    return children


def invalidated_ids(
    evidence: list[EvidenceEnvelope],
    registry: RevocationRegistry,
) -> set[str]:
    children = build_children_index(evidence)
    invalid: set[str] = set()
    queue: list[str] = []

    for ev in evidence:
        event = registry.get(ev.id)
        if event:
            invalid.add(ev.id)
            if event.propagate:
                queue.append(ev.id)

    while queue:
        parent = queue.pop(0)
        for child in children.get(parent, []):
            if child not in invalid:
                invalid.add(child)
                queue.append(child)

    return invalid
```

执行前 gate：

```python
def require_no_invalidated_evidence(
    proof: list[EvidenceEnvelope],
    registry: RevocationRegistry,
):
    bad = invalidated_ids(proof, registry)
    if bad:
        raise RuntimeError(f"proof contains invalidated evidence: {sorted(bad)}")
```

注意：撤销传播不等于删除证据。

它只是告诉执行器：

> 这些证据仍然存在，但不能再用于证明当前动作可以执行。

## 5. pi-mono：EvidenceRevocationMiddleware

在 pi-mono 里，撤销检查适合放在 Tool Gateway 的 proof 校验阶段，顺序建议是：

1. proof completeness
2. freshness
3. scope binding
4. revocation
5. consumption ledger

TypeScript 结构可以这样写：

```ts
// pi-mono: EvidenceRevocationMiddleware.ts
type RevocationReason =
  | 'user_cancelled'
  | 'credential_rotated'
  | 'target_changed'
  | 'policy_changed'
  | 'incident_freeze'
  | 'superseded'

type Evidence = {
  id: string
  kind: string
  parentIds: string[]
  payloadHash: string
}

type RevocationEvent = {
  evidenceId: string
  reasonCode: RevocationReason
  revokedBy: string
  revokedAt: string
  propagate: boolean
}

type EvidenceStore = {
  getRevocations(ids: string[]): Promise<RevocationEvent[]>
}

export class EvidenceRevocationMiddleware {
  constructor(private readonly store: EvidenceStore) {}

  async beforeToolCall(input: {
    toolName: string
    proof: Evidence[]
  }) {
    const ids = input.proof.map(e => e.id)
    const revoked = await this.store.getRevocations(ids)

    const revokedIds = new Set(revoked.map(r => r.evidenceId))
    const children = new Map<string, string[]>()

    for (const ev of input.proof) {
      for (const parentId of ev.parentIds) {
        const list = children.get(parentId) ?? []
        list.push(ev.id)
        children.set(parentId, list)
      }
    }

    const invalid = new Set<string>(revokedIds)
    const queue = revoked
      .filter(r => r.propagate)
      .map(r => r.evidenceId)

    while (queue.length > 0) {
      const parent = queue.shift()!
      for (const child of children.get(parent) ?? []) {
        if (!invalid.has(child)) {
          invalid.add(child)
          queue.push(child)
        }
      }
    }

    if (invalid.size > 0) {
      return {
        decision: 'block' as const,
        reasonCode: 'evidence_revoked',
        message: `Proof contains revoked/invalidated evidence: ${[...invalid].join(', ')}`,
      }
    }

    return { decision: 'allow' as const }
  }
}
```

重点不是代码复杂，而是放置位置：**必须在副作用工具执行前检查。**

如果工具已经执行了，再发现 evidence 被撤销，只能进入补偿和事故流程。

## 6. OpenClaw 实战：课程 Cron 的撤销点

拿这个课程 Cron 举例，一次正常发布会产生这些证据：

- `gh_auth_status`: 当前 GitHub 账号是 `gfwfail`
- `git_pull`: 已基于远端 main 最新代码
- `lesson_file_written`: lesson 文件 hash
- `readme_updated`: README hash
- `message_sent`: Telegram message id
- `git_push`: remote commit hash

哪些事情应该触发撤销？

| 事件 | 撤销对象 | 后果 |
|---|---|---|
| 老板说“这课别发” | `message_approval` | 阻断 Telegram send / git push |
| `gh auth switch` 后账号不是 gfwfail | `gh_auth_status` | 阻断 push |
| `git pull` 后远端又更新 | `git_pull` | 需要重新 pull / rebase |
| 课程文件修改后 hash 变了 | `lesson_file_written` | README / commit proof 要重算 |
| Telegram 发送失败或发错群 | `message_sent` | 阻断“已发布”归档 |

一个很实用的规则：

> 任何会改变 action 前提条件的事件，都应该撤销依赖该前提的 proof。

比如 `git pull` 证据不是永久有效的；一旦远端 main 变了，它就应被 `superseded` 撤销。

## 7. 实战检查清单

给生产 Agent 加 Revocation Gate 时，至少确认：

- [ ] Evidence 是不可变 append-only，撤销用 RevocationEvent 表达
- [ ] RevocationEvent 自己也有 hash、actor、time、reason
- [ ] 副作用工具执行前检查 revoked evidence
- [ ] 支持 parent -> child 的 invalidation propagation
- [ ] credential / policy / incident freeze 能写入撤销事件
- [ ] no_reuse 证据撤销后不能被重新消费
- [ ] 被撤销的 proof 不删除，只从 “可用于证明” 集合移除
- [ ] Gate 失败时返回明确 reasonCode，方便审计和修复

## 8. 小结

这一课的核心：

> Evidence 不只是会过期，还会被现实世界主动推翻。

Freshness 解决“是不是太旧”。

Scope Binding 解决“是不是证明这件事”。

Revocation 解决“这条证明是否已经被作废”。

成熟 Agent 的执行前 proof gate，不能只看证据是否存在，而要确认：

1. 完整
2. 新鲜
3. 作用域匹配
4. 未被撤销
5. 未被重复消费

做到这一层，Agent 才不会拿着一张已经被取消、替换或污染的证明，继续做高风险动作。
