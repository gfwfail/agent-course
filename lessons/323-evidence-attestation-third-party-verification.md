# 323. Agent 证据签发与第三方验证（Evidence Attestation & Third-Party Verification）

前几课我们把证据链做得越来越完整：新鲜度、作用域、防复用、撤销与失效传播。

今天补一个更适合生产系统的能力：**证据不只要 Agent 自己说“我检查过”，还要能让外部验证者证明“这份证据确实由可信组件签发”。**

这叫 Evidence Attestation。

典型场景：

- CI 系统签发 `tests_passed` 证明，Agent 不能自己伪造
- 审批服务签发 `human_approved` 证明，执行器只认审批服务签名
- 部署平台签发 `deploy_succeeded` 回执，审计系统可离线验签
- OpenClaw cron 任务完成后，把 lesson 文件 hash、Telegram message id、git commit 组成可验证 attestation

一句话：**Evidence 记录事实，Attestation 证明事实是谁签的、有没有被改过。**

## 1. 为什么只存 hash 不够？

很多系统会给证据加 `payloadHash`：

```json
{
  "evidenceId": "ev_ci_123",
  "kind": "tests_passed",
  "payloadHash": "sha256:..."
}
```

这能证明 payload 没变，但不能证明：

1. 谁生成了这条证据？
2. 生成者有没有权限签这种证据？
3. 审计系统离线拿到证据后，能不能独立验证？
4. 攻击者能不能伪造一条同样格式的 JSON？

所以生产系统要把 Evidence 升级成：

```text
EvidenceEnvelope + Attestation(signature, issuer, keyId, claims)
```

## 2. Attestation 的核心结构

最小结构可以这样设计：

```json
{
  "evidenceId": "ev_ci_123",
  "kind": "tests_passed",
  "payload": {
    "repo": "gfwfail/agent-course",
    "commit": "abc123",
    "workflow": "ci",
    "status": "passed"
  },
  "attestation": {
    "issuer": "ci.github-actions",
    "keyId": "github-actions-2026-05",
    "alg": "HMAC-SHA256",
    "issuedAt": "2026-05-15T12:28:00+11:00",
    "claims": {
      "canSignKinds": ["tests_passed"],
      "repo": "gfwfail/agent-course"
    },
    "signature": "base64url(...)"
  }
}
```

注意：`claims` 不是装饰字段，它决定这把 key 能证明什么。

比如 CI key 只能签 `tests_passed`，不能签 `human_approved`；审批服务 key 只能签 `approval_granted`，不能签 `deploy_succeeded`。

## 3. learn-claude-code：最小可验证签发器

教学版用 HMAC，生产可以换成 Ed25519 / KMS / Sigstore。

```python
# learn-claude-code: evidence_attestation.py
import base64
import hashlib
import hmac
import json
from dataclasses import dataclass
from datetime import datetime, timezone
from typing import Any


def canonical_json(value: Any) -> bytes:
    return json.dumps(value, sort_keys=True, separators=(",", ":")).encode()


def b64url(data: bytes) -> str:
    return base64.urlsafe_b64encode(data).decode().rstrip("=")


@dataclass(frozen=True)
class Attestation:
    issuer: str
    key_id: str
    alg: str
    issued_at: str
    claims: dict[str, Any]
    signature: str


class Attestor:
    def __init__(self, issuer: str, key_id: str, secret: bytes):
        self.issuer = issuer
        self.key_id = key_id
        self.secret = secret

    def sign(self, evidence: dict[str, Any], claims: dict[str, Any]) -> dict[str, Any]:
        body = {
            "evidenceId": evidence["evidenceId"],
            "kind": evidence["kind"],
            "payloadHash": evidence["payloadHash"],
            "issuer": self.issuer,
            "keyId": self.key_id,
            "claims": claims,
        }
        sig = hmac.new(self.secret, canonical_json(body), hashlib.sha256).digest()
        return {
            **evidence,
            "attestation": {
                "issuer": self.issuer,
                "keyId": self.key_id,
                "alg": "HMAC-SHA256",
                "issuedAt": datetime.now(timezone.utc).isoformat(),
                "claims": claims,
                "signature": b64url(sig),
            },
        }
```

签发时只签 canonical body，不签任意字符串，避免字段顺序、空格、重复字段造成验证歧义。

## 4. 验证器：签名正确还不够，还要校验授权

很多系统只做 `verify(signature)`，这是不够的。

还要检查：

- `issuer` 是否在信任列表
- `keyId` 是否存在且未吊销
- `kind` 是否在 `canSignKinds` 内
- claims 是否绑定 repo / env / tenant
- `payloadHash` 是否匹配真实 payload

```python
class AttestationVerifier:
    def __init__(self, keys: dict[str, bytes]):
        self.keys = keys

    def verify(self, envelope: dict[str, Any]) -> tuple[bool, list[str]]:
        errors: list[str] = []
        att = envelope.get("attestation", {})
        key_id = att.get("keyId")
        secret = self.keys.get(key_id)
        if not secret:
            return False, [f"unknown key: {key_id}"]

        payload_hash = "sha256:" + hashlib.sha256(
            canonical_json(envelope["payload"])
        ).hexdigest()
        if payload_hash != envelope["payloadHash"]:
            errors.append("payloadHash mismatch")

        body = {
            "evidenceId": envelope["evidenceId"],
            "kind": envelope["kind"],
            "payloadHash": envelope["payloadHash"],
            "issuer": att["issuer"],
            "keyId": key_id,
            "claims": att["claims"],
        }
        expected = b64url(hmac.new(secret, canonical_json(body), hashlib.sha256).digest())
        if not hmac.compare_digest(expected, att.get("signature", "")):
            errors.append("bad signature")

        allowed_kinds = set(att.get("claims", {}).get("canSignKinds", []))
        if envelope["kind"] not in allowed_kinds:
            errors.append(f"issuer cannot sign kind: {envelope['kind']}")

        repo_claim = att.get("claims", {}).get("repo")
        if repo_claim and envelope["payload"].get("repo") != repo_claim:
            errors.append("repo claim mismatch")

        return len(errors) == 0, errors
```

这就是生产里常见的“签名 + 授权声明 + payload hash”三件套。

## 5. pi-mono：EvidenceAttestationMiddleware

在 pi-mono 里，不要让业务 tool 自己散落验签逻辑。

更好的方式是放进中间件：

```ts
// pi-mono: EvidenceAttestationMiddleware.ts
type EvidenceKind = 'tests_passed' | 'human_approved' | 'deploy_succeeded'

type Attestation = {
  issuer: string
  keyId: string
  alg: 'ed25519' | 'hmac-sha256'
  issuedAt: string
  claims: {
    canSignKinds: EvidenceKind[]
    repo?: string
    env?: 'dev' | 'staging' | 'prod'
    tenantId?: string
  }
  signature: string
}

type EvidenceEnvelope = {
  evidenceId: string
  kind: EvidenceKind
  payload: Record<string, unknown>
  payloadHash: string
  attestation?: Attestation
}

export class EvidenceAttestationMiddleware {
  constructor(private verifier: AttestationVerifier) {}

  async beforeSideEffect(ctx: ToolContext, next: Next) {
    const required = ctx.policy.requiredEvidence ?? []

    for (const requirement of required) {
      const evidence = ctx.proofBag.find(
        (item: EvidenceEnvelope) => item.kind === requirement.kind,
      )

      if (!evidence) {
        throw new PolicyError(`missing evidence: ${requirement.kind}`)
      }

      const result = await this.verifier.verify(evidence)
      if (!result.ok) {
        throw new PolicyError('invalid evidence attestation', {
          evidenceId: evidence.evidenceId,
          errors: result.errors,
        })
      }

      if (requirement.repo && evidence.attestation?.claims.repo !== requirement.repo) {
        throw new PolicyError('evidence repo claim mismatch')
      }
    }

    return next()
  }
}
```

这样所有高风险动作都能统一要求：

```ts
policy.requiredEvidence = [
  { kind: 'tests_passed', repo: 'gfwfail/agent-course' },
  { kind: 'human_approved' },
]
```

业务层只提交 proof bag，中间件负责验签、验权限、验作用域。

## 6. OpenClaw 实战：课程 Cron 的可验证完成证明

以这个课程 Cron 为例，一次完成证明可以包含：

```json
{
  "evidenceId": "ev_lesson_323_complete",
  "kind": "lesson_published",
  "payload": {
    "lesson": "323-evidence-attestation-third-party-verification.md",
    "lessonHash": "sha256:<file-hash>",
    "telegramGroup": "-5115329245",
    "messageId": "<telegram-message-id>",
    "gitCommit": "<commit-sha>",
    "repo": "gfwfail/agent-course"
  },
  "payloadHash": "sha256:<canonical-payload-hash>",
  "attestation": {
    "issuer": "openclaw.cron.agent-course",
    "keyId": "openclaw-local-2026-05",
    "claims": {
      "canSignKinds": ["lesson_published"],
      "repo": "gfwfail/agent-course"
    },
    "signature": "..."
  }
}
```

以后如果老板问“第 323 课到底有没有发、发的是不是这个文件、GitHub 上是不是同一个 commit”，不需要靠聊天记忆，直接验这份 attestation。

## 7. 常见坑

### 坑 1：签了 payload，但没签 scope

如果签名没有覆盖 repo/env/target/action，攻击者可能把 staging 的证明拿去 prod 用。

### 坑 2：只有签名，没有 key policy

签名正确只能说明“某把 key 签过”，不能说明“这把 key 有资格签这种证据”。

### 坑 3：签发者和执行者是同一个无边界组件

如果执行器能随便给自己签 approval，那 attestation 就失去意义。高风险证据最好由独立组件签发。

### 坑 4：key 轮换后旧证据无法审计

Key registry 要保留历史 public key / key metadata，状态区分 `active` / `retired` / `revoked`。`retired` 可验旧证据，`revoked` 要触发上一课的失效传播。

## 8. 一句话总结

**Evidence Attestation = 证据内容 + 签发者身份 + 授权声明 + 可验证签名。**

成熟 Agent 不只会说“我有证据”，还要让任何审计者都能独立验证：这份证据是谁签的、签了什么、有没有权限签、现在还能不能信。
