# 324. Agent 可信时间戳与时序证明（Evidence Trusted Timestamping & Ordering Proof）

上一课我们讲了 Evidence Attestation：证据要能证明“是谁签发的”。

今天补另一个生产里很容易被忽略的点：**证据还要能证明“什么时候存在、先后顺序是什么”。**

这叫 Trusted Timestamping。

典型场景：

- 审批必须发生在部署之前，而不是部署后补一张 approval
- CI 通过证明必须早于 git push / release
- 用户撤销授权之后，旧 approval 不能继续被拿来执行
- OpenClaw cron 课程要证明：写 lesson → 发 Telegram → commit/push 的顺序可审计

一句话：**Attestation 证明证据是谁签的，Timestamping 证明证据在某个时间点已经存在，并且顺序没有被倒放。**

## 1. 为什么 `createdAt` 不够？

很多 Evidence 都有字段：

```json
{
  "evidenceId": "ev_approval_123",
  "kind": "human_approved",
  "createdAt": "2026-05-15T15:20:00+11:00"
}
```

问题是：

1. `createdAt` 可能是本机时钟写的，本机时间可能漂移
2. 攻击者可以补写一条更早的时间
3. 不同服务之间时间不一致，难以判断先后
4. 审计时只能相信字段，不能独立验证

所以生产系统要把“时间”从普通字段升级为可验证对象：

```text
EvidenceEnvelope + TimestampProof(source, observedAt, sequence, signature)
```

## 2. TimestampProof 的核心结构

最小结构可以这样设计：

```json
{
  "evidenceId": "ev_approval_123",
  "kind": "human_approved",
  "payloadHash": "sha256:...",
  "timestampProof": {
    "source": "time.audit-log",
    "observedAt": "2026-05-15T05:20:03Z",
    "sequence": 884201,
    "previousHash": "sha256:prev...",
    "entryHash": "sha256:this...",
    "signature": "base64url(...)"
  }
}
```

这里有三层含义：

- `observedAt`：可信时间源看到这条证据的时间
- `sequence`：append-only 日志里的递增序号
- `previousHash + entryHash`：把证据串进 hash chain，防止中间插入、删除、倒序

重点：**不是让业务服务自己声明时间，而是让独立的时间戳服务观察并签名。**

## 3. learn-claude-code：最小可信时间戳日志

教学版用文件式 append-only log + HMAC。生产可以换成数据库事务、KMS 签名、RFC3161 TSA、Sigstore/Rekor 或云审计日志。

```python
# learn-claude-code: trusted_timestamp_log.py
import base64
import hashlib
import hmac
import json
from dataclasses import dataclass
from datetime import datetime, timezone
from pathlib import Path
from typing import Any


def canonical_json(value: Any) -> bytes:
    return json.dumps(value, sort_keys=True, separators=(",", ":")).encode()


def sha256_hex(value: bytes) -> str:
    return "sha256:" + hashlib.sha256(value).hexdigest()


def b64url(data: bytes) -> str:
    return base64.urlsafe_b64encode(data).decode().rstrip("=")


@dataclass(frozen=True)
class TimestampProof:
    source: str
    observed_at: str
    sequence: int
    previous_hash: str | None
    entry_hash: str
    signature: str


class TrustedTimestampLog:
    def __init__(self, path: Path, secret: bytes, source: str = "time.audit-log"):
        self.path = path
        self.secret = secret
        self.source = source
        self.path.parent.mkdir(parents=True, exist_ok=True)
        self.path.touch(exist_ok=True)

    def _last_entry(self) -> dict[str, Any] | None:
        lines = [line for line in self.path.read_text().splitlines() if line.strip()]
        return json.loads(lines[-1]) if lines else None

    def stamp(self, evidence: dict[str, Any]) -> dict[str, Any]:
        last = self._last_entry()
        sequence = 1 if last is None else last["timestampProof"]["sequence"] + 1
        previous_hash = None if last is None else last["timestampProof"]["entryHash"]

        observed_at = datetime.now(timezone.utc).isoformat()
        unsigned = {
            "source": self.source,
            "observedAt": observed_at,
            "sequence": sequence,
            "previousHash": previous_hash,
            "evidenceId": evidence["evidenceId"],
            "kind": evidence["kind"],
            "payloadHash": evidence["payloadHash"],
        }
        entry_hash = sha256_hex(canonical_json(unsigned))
        signature = b64url(hmac.new(self.secret, canonical_json({
            **unsigned,
            "entryHash": entry_hash,
        }), hashlib.sha256).digest())

        stamped = {
            **evidence,
            "timestampProof": {
                "source": self.source,
                "observedAt": observed_at,
                "sequence": sequence,
                "previousHash": previous_hash,
                "entryHash": entry_hash,
                "signature": signature,
            },
        }
        self.path.write_text(self.path.read_text() + json.dumps(stamped, ensure_ascii=False) + "\n")
        return stamped
```

这个实现很朴素，但已经抓住了关键：

- 时间戳由独立 log 写入
- sequence 单调递增
- 每条 entry 绑定 previousHash
- 签名覆盖 evidenceId/kind/payloadHash/sequence/time/hash

## 4. 验证：不仅验单条，还要验顺序

单条签名正确，只能证明“这条看起来没坏”。

Timestamping 真正的价值是：检查一组证据的相对顺序。

```python
class TimestampVerifier:
    def __init__(self, secret: bytes, trusted_source: str):
        self.secret = secret
        self.trusted_source = trusted_source

    def verify_entry(self, evidence: dict[str, Any]) -> tuple[bool, list[str]]:
        errors: list[str] = []
        proof = evidence.get("timestampProof", {})
        if proof.get("source") != self.trusted_source:
            errors.append("untrusted timestamp source")

        unsigned = {
            "source": proof["source"],
            "observedAt": proof["observedAt"],
            "sequence": proof["sequence"],
            "previousHash": proof.get("previousHash"),
            "evidenceId": evidence["evidenceId"],
            "kind": evidence["kind"],
            "payloadHash": evidence["payloadHash"],
        }
        entry_hash = sha256_hex(canonical_json(unsigned))
        if entry_hash != proof.get("entryHash"):
            errors.append("entryHash mismatch")

        expected = b64url(hmac.new(self.secret, canonical_json({
            **unsigned,
            "entryHash": entry_hash,
        }), hashlib.sha256).digest())
        if not hmac.compare_digest(expected, proof.get("signature", "")):
            errors.append("bad timestamp signature")

        return len(errors) == 0, errors

    def require_before(self, earlier: dict[str, Any], later: dict[str, Any]) -> None:
        a = earlier["timestampProof"]["sequence"]
        b = later["timestampProof"]["sequence"]
        if a >= b:
            raise ValueError(f"ordering violation: {earlier['kind']} must happen before {later['kind']}")
```

比如 release gate 可以要求：

```python
verifier.require_before(approval_evidence, deploy_evidence)
verifier.require_before(ci_passed_evidence, deploy_evidence)
verifier.require_before(policy_allow_evidence, tool_execute_evidence)
```

这样就能挡住“先执行，后补证据”的假合规。

## 5. pi-mono：EvidenceTimestampMiddleware

在 pi-mono 里，时间戳不要散落在每个 tool 内部，而应作为 Evidence Pipeline 的中间件。

```ts
// pi-mono: EvidenceTimestampMiddleware.ts
type TimestampProof = {
  source: 'time.audit-log'
  observedAt: string
  sequence: number
  previousHash?: string
  entryHash: string
  signature: string
}

type EvidenceEnvelope = {
  evidenceId: string
  kind: 'policy_allow' | 'human_approved' | 'tests_passed' | 'tool_executed'
  payloadHash: string
  payload: unknown
  timestampProof?: TimestampProof
}

type TimestampLog = {
  append(input: Pick<EvidenceEnvelope, 'evidenceId' | 'kind' | 'payloadHash'>): Promise<TimestampProof>
  verify(proof: TimestampProof, evidence: EvidenceEnvelope): Promise<boolean>
}

export class EvidenceTimestampMiddleware {
  constructor(private readonly log: TimestampLog) {}

  async record(evidence: EvidenceEnvelope): Promise<EvidenceEnvelope> {
    if (evidence.timestampProof) return evidence

    const proof = await this.log.append({
      evidenceId: evidence.evidenceId,
      kind: evidence.kind,
      payloadHash: evidence.payloadHash,
    })

    return { ...evidence, timestampProof: proof }
  }

  async assertBefore(earlier: EvidenceEnvelope, later: EvidenceEnvelope): Promise<void> {
    if (!earlier.timestampProof || !later.timestampProof) {
      throw new Error('missing timestamp proof')
    }
    if (earlier.timestampProof.sequence >= later.timestampProof.sequence) {
      throw new Error(`${earlier.kind} must happen before ${later.kind}`)
    }
  }
}
```

生产建议：

- 写 Evidence 时自动 stamp
- 高风险副作用前检查 required ordering
- timestamp log 独立于业务数据库，避免业务 DB 被改后连时间证据一起被改
- sequence 和 signature 进入审计包，便于离线复核

## 6. OpenClaw 实战：副作用前的时序闸门

OpenClaw 这类 always-on Agent 最需要防的是：长任务跑久了，世界变了，但 Agent 还拿旧证据继续执行。

课程 cron 的顺序可以这样建模：

```json
{
  "requiredOrdering": [
    ["lesson_file_written", "telegram_message_sent"],
    ["telegram_message_sent", "git_commit_created"],
    ["git_commit_created", "git_push_verified"]
  ]
}
```

执行完成前检查：

```text
if lesson_file_written.sequence >= telegram_message_sent.sequence: block
if telegram_message_sent.sequence >= git_commit_created.sequence: block
if git_commit_created.sequence >= git_push_verified.sequence: block
```

这不是形式主义。它能防：

- 先 push，后来补 README
- Telegram 发了旧内容，commit 是新内容
- commit 成功但 push 失败，却误报完成
- 审批/检查发生在副作用之后

## 7. 常见坑

**坑 1：只相信本机时间**

可以记录本机时间，但高风险 proof 要有独立时间源签名。

**坑 2：只验时间，不验 payloadHash**

时间戳必须绑定 payloadHash，否则可以把旧时间戳挪到新证据上。

**坑 3：只验单条，不验顺序**

可信时间戳的核心价值是 ordering proof，不只是 createdAt 好看。

**坑 4：允许补写历史 sequence**

append-only log 必须单调追加，不能让业务服务指定 sequence。

## 8. 总结

Agent 的证据系统要回答三个问题：

1. **谁签的？** —— Attestation
2. **什么时候存在？** —— Trusted Timestamp
3. **先后顺序对不对？** —— Ordering Proof

成熟 Agent 不只会说“我有证据”，还要能证明：

> 这份证据在副作用之前已经存在，并且没有被后补、倒序或篡改。

这就是 Evidence Trusted Timestamping 的价值。
