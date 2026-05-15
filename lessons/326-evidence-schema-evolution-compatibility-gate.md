# 326. Agent 证据 Schema 演进与兼容闸门（Evidence Schema Evolution & Compatibility Gate）

上一课讲了证据规范化：同一份 Evidence 必须在 Python、TypeScript、审计重放里算出同一个 hash。

但生产系统还有一个更长期的问题：**证据格式会变。**

今天你的 `tool_result` 只有 `tool/target/commit`，明天你想加 `policyVersion`、`envFingerprint`、`attestation`；如果没有 Schema 演进机制，就会出现这些事故：

- 新版 Agent 写入 v2 Evidence，旧版审计器读不懂，直接跳过
- 字段改名后，历史证据无法验证
- 新字段没有参与 canonical hash，导致签名覆盖不完整
- 多个子 Agent 版本不同，证据链里混着 v1/v2/v3，验证器行为不一致
- 为了兼容旧数据临时写 if/else，越写越像考古现场

一句话：**Evidence 是长期资产，不是当前版本的临时 JSON。Schema 变更必须像数据库迁移一样受控。**

## 1. 核心原则：append-only 优先，breaking change 走迁移

证据 Schema 演进建议分三类：

1. `add_optional`：新增可选字段，旧读者可以忽略
2. `add_required`：新增必填字段，需要 writer/reader 同时升级，通常要 gate
3. `rename/remove/type_change`：破坏性变更，必须提供 migration 或 adapter

一个 EvidenceEnvelope 不只带 `kind`，还要带 schema 身份：

```json
{
  "evidenceId": "ev_git_push_123",
  "kind": "tool_result",
  "schema": {
    "name": "openclaw.evidence.tool_result",
    "version": 2,
    "compat": "backward_readable"
  },
  "payload": {
    "tool": "git.push",
    "target": "gfwfail/agent-course",
    "commit": "abc123",
    "policyVersion": "policy-2026-05-15"
  },
  "payloadHash": "sha256:..."
}
```

这里的关键不是 version 数字，而是：

- writer 明确自己写的是哪个 schema
- reader 明确自己支持哪些 schema
- verifier 能判断是否需要 upgrade/downcast
- breaking change 不能静默通过

## 2. learn-claude-code：最小 Schema Registry + Upgrader

教学版可以用一个纯 Python registry 管住 schema 版本和迁移函数。

```python
# learn-claude-code: evidence_schema_registry.py
from dataclasses import dataclass
from typing import Any, Callable


@dataclass(frozen=True)
class SchemaId:
    name: str
    version: int


@dataclass
class EvidenceEnvelope:
    evidence_id: str
    kind: str
    schema: SchemaId
    payload: dict[str, Any]
    payload_hash: str | None = None


Upgrade = Callable[[dict[str, Any]], dict[str, Any]]


class SchemaRegistry:
    def __init__(self) -> None:
        self.latest: dict[str, int] = {}
        self.upgrades: dict[tuple[str, int, int], Upgrade] = {}
        self.required_fields: dict[tuple[str, int], set[str]] = {}

    def register(self, name: str, version: int, required: set[str]) -> None:
        self.latest[name] = max(self.latest.get(name, 0), version)
        self.required_fields[(name, version)] = required

    def upgrade(self, name: str, from_version: int, to_version: int):
        def decorator(fn: Upgrade) -> Upgrade:
            self.upgrades[(name, from_version, to_version)] = fn
            return fn
        return decorator

    def normalize_to_latest(self, ev: EvidenceEnvelope) -> EvidenceEnvelope:
        latest = self.latest[ev.schema.name]
        payload = dict(ev.payload)
        version = ev.schema.version

        while version < latest:
            step = self.upgrades.get((ev.schema.name, version, version + 1))
            if not step:
                raise ValueError(f"missing schema upgrade {ev.schema.name} v{version}->v{version + 1}")
            payload = step(payload)
            version += 1

        required = self.required_fields[(ev.schema.name, latest)]
        missing = sorted(required - payload.keys())
        if missing:
            raise ValueError(f"schema {ev.schema.name} v{latest} missing required fields: {missing}")

        return EvidenceEnvelope(
            evidence_id=ev.evidence_id,
            kind=ev.kind,
            schema=SchemaId(ev.schema.name, latest),
            payload=payload,
            payload_hash=ev.payload_hash,
        )


registry = SchemaRegistry()
registry.register("openclaw.evidence.tool_result", 1, {"tool", "target", "ok"})
registry.register("openclaw.evidence.tool_result", 2, {"tool", "target", "ok", "policyVersion"})


@registry.upgrade("openclaw.evidence.tool_result", 1, 2)
def tool_result_v1_to_v2(payload: dict[str, Any]) -> dict[str, Any]:
    # 迁移必须显式标注默认值来源，不能假装旧证据本来就有新字段
    return {
        **payload,
        "policyVersion": payload.get("policyVersion", "unknown:legacy-v1"),
        "migrationNote": "v1 evidence upgraded for verification; policyVersion was not captured originally",
    }


legacy = EvidenceEnvelope(
    evidence_id="ev_1",
    kind="tool_result",
    schema=SchemaId("openclaw.evidence.tool_result", 1),
    payload={"tool": "git.push", "target": "gfwfail/agent-course", "ok": True},
)

print(registry.normalize_to_latest(legacy))
```

注意这里的 `unknown:legacy-v1` 很重要：

- 它让旧证据可读
- 但不会伪造一个不存在的 `policyVersion`
- 高风险验证器可以看到 `unknown` 后降级为 `manual_review`

兼容不是撒谎。兼容是把“旧世界缺什么”明确暴露出来。

## 3. 兼容闸门：读得懂，不代表能用于高风险动作

Schema Registry 只能解决“能不能解析”。真正执行前还要看用途。

建议按 purpose 分级：

| Purpose | 允许旧 Schema？ | 策略 |
|---|---:|---|
| answer / summary | 可以 | upgrade 后带 caveat |
| planning | 可以 | 标记 legacy assumptions |
| audit_query | 可以 | 展示 migrationNote |
| external_side_effect | 谨慎 | required fields 缺失就 block/manual_review |
| security_decision | 不建议 | 必须 latest + no synthetic critical fields |

也就是说，旧证据可以帮助理解历史，但不能随便拿来证明“现在可以 push / deploy / 发钱”。

## 4. pi-mono：EvidenceSchemaMiddleware

生产框架里，Schema 演进应该放在 Evidence 读取路径，而不是散落在业务代码里。

```ts
// pi-mono: EvidenceSchemaMiddleware.ts
type Purpose = 'answer' | 'planning' | 'audit_query' | 'external_side_effect' | 'security_decision'

type SchemaRef = {
  name: string
  version: number
}

type EvidenceEnvelope<T = unknown> = {
  evidenceId: string
  kind: string
  schema: SchemaRef
  payload: T
  payloadHash: string
  migrations?: Array<{ from: number; to: number; note: string; syntheticFields: string[] }>
}

type UpgradeFn = (payload: any) => {
  payload: any
  syntheticFields: string[]
  note: string
}

class EvidenceSchemaRegistry {
  private latest = new Map<string, number>()
  private upgrades = new Map<string, UpgradeFn>()

  register(name: string, version: number) {
    this.latest.set(name, Math.max(this.latest.get(name) ?? 0, version))
  }

  addUpgrade(name: string, from: number, to: number, fn: UpgradeFn) {
    this.upgrades.set(`${name}:${from}->${to}`, fn)
  }

  normalize(ev: EvidenceEnvelope): EvidenceEnvelope {
    const latest = this.latest.get(ev.schema.name)
    if (!latest) throw new Error(`unknown evidence schema: ${ev.schema.name}`)

    let version = ev.schema.version
    let payload: any = ev.payload
    const migrations: EvidenceEnvelope['migrations'] = [...(ev.migrations ?? [])]

    while (version < latest) {
      const fn = this.upgrades.get(`${ev.schema.name}:${version}->${version + 1}`)
      if (!fn) throw new Error(`missing migration ${ev.schema.name} ${version}->${version + 1}`)
      const result = fn(payload)
      payload = result.payload
      migrations.push({
        from: version,
        to: version + 1,
        note: result.note,
        syntheticFields: result.syntheticFields,
      })
      version += 1
    }

    return { ...ev, schema: { name: ev.schema.name, version }, payload, migrations }
  }
}

function gateEvidenceForPurpose(ev: EvidenceEnvelope, purpose: Purpose) {
  const synthetic = new Set(ev.migrations?.flatMap(m => m.syntheticFields) ?? [])

  if (purpose === 'security_decision' && synthetic.size > 0) {
    return {
      action: 'block',
      reason: 'schema_migration_synthetic_fields_not_allowed_for_security_decision',
      syntheticFields: [...synthetic],
    }
  }

  if (purpose === 'external_side_effect' && synthetic.has('policyVersion')) {
    return {
      action: 'manual_review',
      reason: 'legacy_evidence_missing_policy_version',
    }
  }

  return { action: 'allow' }
}
```

这段 middleware 的重点：

- 所有 Evidence 先统一 normalize
- migration 过程进入 envelope，可审计
- synthetic fields 不等于真实历史字段
- 高风险 purpose 可以拒绝依赖 synthetic critical fields

## 5. OpenClaw 实战：课程 Cron 的证据版本闸门

拿这个课程 Cron 举例，每 3 小时会做这些事：

1. 检查已讲内容，避免重复
2. 写 lesson 文件
3. 更新 README/TOOLS
4. 发 Telegram
5. git commit + push
6. 记录 messageId / commit hash

如果我们给每个步骤留 Evidence，后面 schema 升级时可以这样处理：

```json
{
  "kind": "course_publish_proof",
  "schema": { "name": "openclaw.course.publish_proof", "version": 3 },
  "payload": {
    "lesson": 326,
    "topic": "Evidence Schema Evolution & Compatibility Gate",
    "filesChanged": [
      "lessons/326-evidence-schema-evolution-compatibility-gate.md",
      "README.md",
      "../TOOLS.md"
    ],
    "telegram": { "chatId": "-5115329245", "messageId": "..." },
    "git": { "repo": "gfwfail/agent-course", "commit": "...", "remoteVerified": true },
    "schemaGate": { "latestOnlyForPush": true, "legacySyntheticCriticalFields": [] }
  }
}
```

如果旧版 proof 缺 `remoteVerified`，可以用于课程目录展示；但用于“证明这次已经 push 成功”时，就必须 live check 远端 main。

这就是兼容闸门的价值：**旧数据能读，但高风险动作必须用足够新的证据。**

## 6. Schema 变更检查清单

每次改 Evidence Schema 前，至少问 8 个问题：

1. 这是新增可选字段，还是破坏性变更？
2. 旧 reader 看到新字段会忽略还是报错？
3. 新 reader 看到旧证据时如何升级？
4. 新增字段是否参与 canonical hash？
5. synthetic/default 字段是否会误导安全判断？
6. 哪些 purpose 可以接受 legacy 证据？
7. migration 本身有没有审计记录？
8. 回滚后，新写入的 schema 旧版本是否能读取？

成熟 Agent 的 Evidence 系统不是“JSON 能 parse 就行”，而是：

```text
schema declared
-> compatibility checked
-> migration audited
-> purpose gate enforced
-> high-risk proof uses latest trustworthy fields
```

## 7. 小结

今天这课的核心：**Evidence Schema 是长期契约。**

- 证据要带 schema name/version
- Schema Registry 负责集中迁移
- migration 不能伪造历史，只能显式标注 synthetic/default
- 读得懂不代表能用于高风险动作
- external_side_effect / security_decision 要有更严格的 latest schema gate

成熟 Agent 不只会留下证据，还要保证几年后系统升级了，这些证据仍然能被正确、诚实、带边界地理解。
