# 325. Agent 证据规范化与哈希稳定性（Evidence Canonicalization & Hash Stability）

上一课讲了可信时间戳：证据要能证明“什么时候存在、顺序是否正确”。

但在生产里还有一个更底层、也更容易踩坑的问题：**同一份证据，必须在不同语言、不同服务、不同时间算出同一个 hash。**

这叫 Evidence Canonicalization。

典型事故：

- Python 写入 `payloadHash`，TypeScript 验签时 hash 不一致
- JSON 字段顺序变化导致签名失效
- `1`、`1.0`、`"1"` 被不同服务序列化成不同字节
- 时间字段有的带毫秒，有的不带毫秒
- Unicode、空格、换行、浮点数精度导致审计重放失败

一句话：**签名、时间戳、Merkle root、证据链之前，先把“要签的东西”规范化。否则你签的不是事实，而是某个运行时碰巧吐出来的一串字节。**

## 1. 为什么普通 `json.dumps` / `JSON.stringify` 不够？

很多教学代码会这样写：

```python
hashlib.sha256(json.dumps(payload, sort_keys=True).encode()).hexdigest()
```

这已经比直接 `str(payload)` 好，但还不够。

生产里至少要明确这些规则：

1. 字段排序：object key 必须稳定排序
2. 空值策略：`null` 是否保留，还是省略
3. 时间格式：统一 UTC RFC3339，精度固定
4. 数字格式：禁止 float 进入证据 payload，金额用 string/int minor unit
5. Unicode：统一 NFC 规范化
6. 二进制：统一 base64url，不允许原始 bytes 混入 JSON
7. 未知字段：签名域是否允许额外字段
8. Hash 域：hash 不能把自己的 `payloadHash` 算进去

所以 Evidence 系统要把 canonicalization 当成一等公民：

```text
raw payload -> normalize -> canonical bytes -> payloadHash -> signature/timestamp/chain
```

## 2. EvidenceEnvelope 增加 canonicalization 元数据

不要只存 hash，要存“这个 hash 是按什么规则算的”。

```json
{
  "evidenceId": "ev_tool_result_123",
  "kind": "tool_result",
  "canonicalization": {
    "version": "evidence-c14n-v1",
    "hashAlg": "sha256",
    "encoding": "utf-8",
    "unicode": "NFC",
    "time": "RFC3339_UTC_MILLIS",
    "numberPolicy": "no_float"
  },
  "payloadHash": "sha256:...",
  "payload": {
    "tool": "git.push",
    "target": "gfwfail/agent-course",
    "commit": "abc123",
    "observedAt": "2026-05-15T08:30:00.000Z"
  }
}
```

以后审计者看到 `payloadHash`，不是猜“当时怎么序列化的”，而是按 `evidence-c14n-v1` 复算。

## 3. learn-claude-code：最小 canonicalizer

教学版可以先实现一个严格的 JSON canonicalizer。重点不是功能多，而是**拒绝不稳定输入**。

```python
# learn-claude-code: evidence_canonicalizer.py
import base64
import hashlib
import json
import math
import unicodedata
from datetime import datetime, timezone
from typing import Any


C14N_VERSION = "evidence-c14n-v1"


def b64url(data: bytes) -> str:
    return base64.urlsafe_b64encode(data).decode().rstrip("=")


def normalize_time(value: datetime) -> str:
    # 固定 UTC + 毫秒，避免 2026-05-15T08:30:00Z 和 .000Z 两种写法混用
    utc = value.astimezone(timezone.utc)
    return utc.isoformat(timespec="milliseconds").replace("+00:00", "Z")


def normalize(value: Any) -> Any:
    if value is None or isinstance(value, bool):
        return value

    if isinstance(value, str):
        return unicodedata.normalize("NFC", value)

    if isinstance(value, int):
        return value

    if isinstance(value, float):
        if not math.isfinite(value):
            raise ValueError("float NaN/Infinity is not canonical")
        # 生产建议：证据 payload 禁止 float，金额用 cents/int 或 decimal string
        raise ValueError("float is not allowed in evidence payload; use string or integer minor units")

    if isinstance(value, datetime):
        return normalize_time(value)

    if isinstance(value, bytes):
        return {"$bytes_base64url": b64url(value)}

    if isinstance(value, list):
        return [normalize(item) for item in value]

    if isinstance(value, dict):
        normalized: dict[str, Any] = {}
        for key, item in value.items():
            if not isinstance(key, str):
                raise ValueError("object keys must be strings")
            if key in {"payloadHash", "signature", "timestampProof"}:
                # 这些是 envelope 派生字段，不能参与 payloadHash
                continue
            normalized[unicodedata.normalize("NFC", key)] = normalize(item)
        return {key: normalized[key] for key in sorted(normalized)}

    raise TypeError(f"unsupported evidence value: {type(value).__name__}")


def canonical_bytes(payload: dict[str, Any]) -> bytes:
    normalized = normalize(payload)
    return json.dumps(
        normalized,
        ensure_ascii=False,
        sort_keys=True,
        separators=(",", ":"),
    ).encode("utf-8")


def payload_hash(payload: dict[str, Any]) -> str:
    return "sha256:" + hashlib.sha256(canonical_bytes(payload)).hexdigest()


if __name__ == "__main__":
    payload = {
        "tool": "git.push",
        "observedAt": datetime(2026, 5, 15, 8, 30, tzinfo=timezone.utc),
        "target": "gfwfail/agent-course",
        "args": {"branch": "main", "force": False},
    }
    print(canonical_bytes(payload).decode())
    print(payload_hash(payload))
```

几个故意严格的点：

- `float` 直接拒绝
- bytes 转成明确 tagged object
- 时间统一 UTC 毫秒
- Unicode 做 NFC
- 派生字段不参与 payloadHash

这会让系统早失败，而不是审计时才发现 hash 对不上。

## 4. pi-mono：把 canonicalization 放进 Evidence Middleware

在生产框架里，不要让每个 tool 自己算 hash。应该由统一中间件负责。

```ts
// pi-mono: EvidenceCanonicalizationMiddleware.ts
type Canonicalization = {
  version: 'evidence-c14n-v1'
  hashAlg: 'sha256'
  encoding: 'utf-8'
  unicode: 'NFC'
  time: 'RFC3339_UTC_MILLIS'
  numberPolicy: 'no_float'
}

type EvidenceEnvelope = {
  evidenceId: string
  kind: string
  payload: unknown
  canonicalization: Canonicalization
  payloadHash: string
}

const C14N: Canonicalization = {
  version: 'evidence-c14n-v1',
  hashAlg: 'sha256',
  encoding: 'utf-8',
  unicode: 'NFC',
  time: 'RFC3339_UTC_MILLIS',
  numberPolicy: 'no_float',
}

function normalize(value: unknown): unknown {
  if (value === null || typeof value === 'boolean' || typeof value === 'string') {
    return typeof value === 'string' ? value.normalize('NFC') : value
  }

  if (typeof value === 'number') {
    if (!Number.isInteger(value)) {
      throw new Error('float is not allowed in evidence payload')
    }
    return value
  }

  if (value instanceof Date) {
    return value.toISOString().replace(/\.\d{3}Z$/, (ms) => ms) // 固定毫秒格式
  }

  if (Array.isArray(value)) return value.map(normalize)

  if (typeof value === 'object') {
    const input = value as Record<string, unknown>
    const out: Record<string, unknown> = {}
    for (const key of Object.keys(input).sort()) {
      if (['payloadHash', 'signature', 'timestampProof'].includes(key)) continue
      out[key.normalize('NFC')] = normalize(input[key])
    }
    return out
  }

  throw new Error(`unsupported evidence value: ${typeof value}`)
}

function canonicalJson(value: unknown): string {
  return JSON.stringify(normalize(value))
}

async function sha256Hex(text: string): Promise<string> {
  const bytes = new TextEncoder().encode(text)
  const digest = await crypto.subtle.digest('SHA-256', bytes)
  return 'sha256:' + [...new Uint8Array(digest)]
    .map((b) => b.toString(16).padStart(2, '0'))
    .join('')
}

export async function createEvidence(kind: string, payload: unknown): Promise<EvidenceEnvelope> {
  const canonical = canonicalJson(payload)
  return {
    evidenceId: crypto.randomUUID(),
    kind,
    payload: normalize(payload),
    canonicalization: C14N,
    payloadHash: await sha256Hex(canonical),
  }
}
```

注意：真实生产里建议直接采用成熟规范，比如 JCS（RFC 8785）或 CBOR canonical encoding，并把规则版本写死进 `canonicalization.version`。不要让业务代码随手 `JSON.stringify`。

## 5. OpenClaw：副作用前复算 proof hash

对 OpenClaw 这类 always-on Agent，canonicalization 最实用的位置是副作用前闸门：发消息、push、部署之前，重新复算证据包 hash。

```python
# OpenClaw preflight: verify_evidence_hash.py
from evidence_canonicalizer import payload_hash


def verify_proof_hashes(proof: dict) -> list[str]:
    errors: list[str] = []
    for evidence in proof.get("evidence", []):
        expected_version = "evidence-c14n-v1"
        got_version = evidence.get("canonicalization", {}).get("version")
        if got_version != expected_version:
            errors.append(f"{evidence['evidenceId']}: unsupported c14n {got_version}")
            continue

        actual = payload_hash(evidence["payload"])
        if actual != evidence.get("payloadHash"):
            errors.append(f"{evidence['evidenceId']}: payloadHash mismatch")

    return errors


def gate_side_effect(proof: dict, action: str) -> None:
    errors = verify_proof_hashes(proof)
    if errors:
        raise RuntimeError(f"block {action}: evidence hash verification failed: {errors}")
```

这能防三类问题：

- 证据 payload 被手工改过，但 hash 没更新
- 不同服务 canonicalization 版本混用
- 审计包里混入无法复算的“神秘证据”

## 6. 常见设计坑

### 坑 1：把整个 envelope 都拿去 hash

不要这样：

```text
hash(envelope including payloadHash/signature)
```

因为 `payloadHash` 自己依赖 hash，会形成循环。通常拆成：

```text
payloadHash = hash(canonical(payload))
envelopeHash = hash(canonical(metadata + payloadHash + attestations))
```

### 坑 2：证据 payload 里放 float

金额、概率、评分都不要直接放 float：

```json
{ "amountCents": 1299 }
{ "confidenceBps": 8750 }
{ "price": "12.99" }
```

Agent 系统跨语言很多，float 是审计重放的毒药。

### 坑 3：canonicalization 版本不可升级

规范迟早要升级，所以 proof gate 要允许多版本验证：

```text
evidence-c14n-v1 -> verifier_v1
evidence-c14n-v2 -> verifier_v2
unknown          -> block/manual_review
```

旧证据不能因为新版本上线就无法验证。

## 7. 实战清单

给 Agent Evidence 系统加 hash 稳定性，最小落地可以这样做：

1. 定义 `canonicalization.version`
2. 所有 EvidenceEnvelope 必须存 `payloadHash + canonicalization`
3. 禁止 tool 自己算 hash，统一中间件生成
4. 禁止 float、非字符串 key、隐式 bytes
5. 时间统一 UTC RFC3339 毫秒
6. 签名/时间戳/证据链都基于 canonical bytes
7. 副作用前复算 hash，不通过就 block
8. 升级 canonicalization 时保留旧 verifier

## 8. 一句话总结

**Evidence 的可信度，从稳定字节开始。**

没有 canonicalization，签名、时间戳、Merkle root 都可能只是“这台机器当时这样序列化”的幻觉。

成熟 Agent 不只会保存证据，还要保证证据在任何地方、任何时间都能被复算、验签和重放。
