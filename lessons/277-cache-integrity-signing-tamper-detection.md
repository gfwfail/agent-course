# 277. Agent 缓存完整性签名与防篡改（Cache Integrity Signing & Tamper Detection）

> 上一课我们讲了缓存校验器：命中以后要验证 envelope、schema、hash、freshness。今天再往安全边界推进一步：**缓存不只要校验“像不像对的”，还要证明“是不是系统自己写的、有没有被改过”。**

一句话：**hash 只能发现 value 变了；signature / MAC 才能证明这份缓存来自可信写入路径。**

## 1. 为什么 valueHash 不够？

很多缓存系统会给 value 存一个 `sha256(value)`。这能发现 value 被改过，但挡不住一个更隐蔽的问题：

```text
攻击者/误操作：
1. 改 value
2. 顺手重算 valueHash
3. 缓存校验器看到 hash 对得上，于是放行
```

所以生产级 Agent 缓存至少要区分三层完整性：

- **content hash**：value 有没有变化；
- **envelope hash**：metadata 有没有变化，比如 tenant、purpose、risk；
- **signature / MAC**：这份 envelope 是否由可信写入者签发。

如果缓存会参与外部副作用、权限判断、账单分析、部署决策，就不能只靠普通 hash。

## 2. 签什么？不要只签 value

错误做法：

```json
{
  "value": { "balance": 100 },
  "valueHash": "sha256:...",
  "signature": "sign(valueHash)"
}
```

问题是：`tenantId`、`purpose`、`risk`、`sourceVersion` 被改了，value 本身不变，签名仍然通过。

正确做法是签一份 canonical envelope：

```json
{
  "key": "agent-cache:v7:billing:tenant_metadraw:...",
  "tenantId": "metadraw",
  "actorId": "cron:aws-billing",
  "purpose": "answer",
  "risk": "medium",
  "schemaVersion": 3,
  "sourceVersion": "aws-ce:2026-05-08",
  "observedAt": "2026-05-09T08:30:00Z",
  "expiresAt": "2026-05-09T08:45:00Z",
  "valueHash": "sha256:..."
}
```

注意：签名输入里通常不直接塞大 value，而是塞 `valueHash`，然后把关键 metadata 一起签。

## 3. 签名策略：HMAC vs Ed25519

常见两种选择：

### 3.1 HMAC：适合单服务 / 内部缓存

- 写入和读取方共享同一个密钥；
- 实现简单，速度快；
- 适合 learn-claude-code 教学版、单机 OpenClaw 文件缓存。

缺点：谁能验证，谁也能伪造。

### 3.2 Ed25519：适合多服务 / 只读验证方

- 写入方持有 private key；
- 读取方只拿 public key；
- 子 Agent、只读 worker、审计器可以验证但不能伪造。

缺点：密钥管理和轮换稍复杂。

经验法则：

- 单进程教学：HMAC；
- 多 worker / 多 Agent / 审计链路：Ed25519；
- 高风险缓存：签名 + keyId + rotation plan。

## 4. learn-claude-code：Python 教学版 HMAC

```python
# learn-claude-code/cache_integrity.py
from __future__ import annotations

from dataclasses import dataclass, asdict
from typing import Any, Literal
import base64
import hashlib
import hmac
import json

Purpose = Literal["answer", "planning", "external_side_effect", "security_decision"]


def canonical_json(value: Any) -> bytes:
    return json.dumps(
        value,
        sort_keys=True,
        ensure_ascii=False,
        separators=(",", ":"),
    ).encode("utf-8")


def sha256_hex(value: Any) -> str:
    return "sha256:" + hashlib.sha256(canonical_json(value)).hexdigest()


@dataclass(frozen=True)
class CacheEnvelopeToSign:
    key: str
    tenant_id: str
    actor_id: str
    purpose: Purpose
    risk: Literal["low", "medium", "high"]
    schema_version: int
    source_version: str
    observed_at: str
    expires_at: str
    value_hash: str


@dataclass
class SignedCacheEntry:
    envelope: CacheEnvelopeToSign
    value: dict[str, Any]
    key_id: str
    alg: str
    signature: str


class CacheIntegritySigner:
    def __init__(self, keys: dict[str, bytes], active_key_id: str):
        self.keys = keys
        self.active_key_id = active_key_id

    def sign(self, envelope: CacheEnvelopeToSign) -> tuple[str, str]:
        payload = canonical_json(asdict(envelope))
        key = self.keys[self.active_key_id]
        sig = hmac.new(key, payload, hashlib.sha256).digest()
        return self.active_key_id, base64.urlsafe_b64encode(sig).decode()

    def verify(self, entry: SignedCacheEntry) -> bool:
        key = self.keys.get(entry.key_id)
        if not key:
            return False

        # 先确认 valueHash 真的对应 value
        if entry.envelope.value_hash != sha256_hex(entry.value):
            return False

        payload = canonical_json(asdict(entry.envelope))
        expected = hmac.new(key, payload, hashlib.sha256).digest()
        actual = base64.urlsafe_b64decode(entry.signature.encode())
        return hmac.compare_digest(expected, actual)
```

写入路径：

```python
value = {"total_usd": 21.06, "date": "2026-05-07"}

envelope = CacheEnvelopeToSign(
    key="agent-cache:v7:billing:metadraw:daily-cost:2026-05-07",
    tenant_id="metadraw",
    actor_id="cron:aws-billing",
    purpose="answer",
    risk="medium",
    schema_version=3,
    source_version="aws-cost-explorer:2026-05-08",
    observed_at="2026-05-09T08:30:00Z",
    expires_at="2026-05-09T09:00:00Z",
    value_hash=sha256_hex(value),
)

key_id, signature = signer.sign(envelope)
entry = SignedCacheEntry(envelope, value, key_id, "hmac-sha256", signature)
```

读取路径必须先 verify，再交给上一课的 `CacheValidator`：

```python
if not signer.verify(entry):
    audit.write("cache.integrity_failed", {"key": entry.envelope.key, "keyId": entry.key_id})
    cache.evict(entry.envelope.key)
    raise RuntimeError("cache entry failed integrity verification")

validation = validator.validate(entry.envelope, requested_purpose="answer")
```

顺序很重要：**先完整性验证，再业务校验。**

## 5. pi-mono：CacheIntegrityMiddleware

生产版建议把完整性验证做成 Cache Middleware，让业务代码默认绕不开。

```ts
// pi-mono/cache/CacheIntegrityMiddleware.ts
import { createHmac, timingSafeEqual } from "node:crypto";

type Purpose = "answer" | "planning" | "external_side_effect" | "security_decision";

type SignedEnvelope = {
  key: string;
  tenantId: string;
  actorId: string;
  purpose: Purpose;
  risk: "low" | "medium" | "high";
  schemaVersion: number;
  sourceVersion: string;
  observedAt: string;
  expiresAt: string;
  valueHash: string;
};

type SignedCacheEntry<T> = {
  envelope: SignedEnvelope;
  value: T;
  integrity: {
    alg: "hmac-sha256";
    keyId: string;
    signature: string;
  };
};

type Keyring = {
  activeKeyId(): string;
  getHmacKey(keyId: string): Buffer | null;
};

function canonicalJson(value: unknown): string {
  // 生产里建议用稳定 JSON 序列化库，避免 key order / undefined 差异
  return JSON.stringify(sortObject(value));
}

function sortObject(value: any): any {
  if (Array.isArray(value)) return value.map(sortObject);
  if (value && typeof value === "object") {
    return Object.fromEntries(
      Object.keys(value).sort().map((key) => [key, sortObject(value[key])]),
    );
  }
  return value;
}

function hmacSha256(key: Buffer, payload: string): string {
  return createHmac("sha256", key).update(payload).digest("base64url");
}

export class CacheIntegrityMiddleware {
  constructor(
    private keyring: Keyring,
    private hashValue: (value: unknown) => string,
    private audit: (event: string, payload: Record<string, unknown>) => void,
  ) {}

  sign<T>(entry: Omit<SignedCacheEntry<T>, "integrity">): SignedCacheEntry<T> {
    const keyId = this.keyring.activeKeyId();
    const key = this.keyring.getHmacKey(keyId);
    if (!key) throw new Error(`active integrity key missing: ${keyId}`);

    const valueHash = this.hashValue(entry.value);
    const envelope = { ...entry.envelope, valueHash };
    const signature = hmacSha256(key, canonicalJson(envelope));

    return {
      envelope,
      value: entry.value,
      integrity: { alg: "hmac-sha256", keyId, signature },
    };
  }

  verify<T>(entry: SignedCacheEntry<T>): boolean {
    const key = this.keyring.getHmacKey(entry.integrity.keyId);
    if (!key) {
      this.audit("cache.integrity.key_missing", { keyId: entry.integrity.keyId });
      return false;
    }

    if (entry.envelope.valueHash !== this.hashValue(entry.value)) {
      this.audit("cache.integrity.value_hash_mismatch", { key: entry.envelope.key });
      return false;
    }

    const expected = hmacSha256(key, canonicalJson(entry.envelope));
    const actual = entry.integrity.signature;

    const ok = timingSafeEqual(Buffer.from(expected), Buffer.from(actual));
    if (!ok) {
      this.audit("cache.integrity.signature_mismatch", {
        key: entry.envelope.key,
        keyId: entry.integrity.keyId,
        tenantId: entry.envelope.tenantId,
      });
    }
    return ok;
  }
}
```

在 cache read path 里串起来：

```ts
const entry = await cache.get<SignedCacheEntry<BillingSummary>>(key);
if (!entry) return null;

if (!integrity.verify(entry)) {
  await cache.evict(key);
  return await origin.fetchFreshBillingSummary(ctx);
}

return await validation.read(entry, ctx);
```

注意：`verify` 失败不要 fallback 到“直接使用 value”。要么 evict + live fetch，要么 block。

## 6. OpenClaw 文件缓存实战

OpenClaw 这类 Always-on Agent 经常用文件保存状态：

```text
.openclaw/cache/
  billing-daily-cost.json
  heartbeat-state.json
  tool-result-cache.jsonl
```

文件缓存尤其容易被脚本、手工编辑、旧版本 worker 改坏。建议采用 sidecar 签名或内嵌签名。

### 6.1 内嵌签名格式

```json
{
  "envelope": {
    "key": "agent-course:lesson-index:v1",
    "tenantId": "owner:67431246",
    "actorId": "cron:agent-course",
    "purpose": "planning",
    "risk": "low",
    "schemaVersion": 1,
    "sourceVersion": "git:9cfa7f6",
    "observedAt": "2026-05-09T08:30:00Z",
    "expiresAt": "2026-05-09T11:30:00Z",
    "valueHash": "sha256:..."
  },
  "value": {
    "lastLesson": 276,
    "nextCandidate": "cache-integrity-signing-tamper-detection"
  },
  "integrity": {
    "alg": "hmac-sha256",
    "keyId": "cache-integrity-2026-05",
    "signature": "..."
  }
}
```

### 6.2 审计事件

验证失败必须留审计，而不是静默 miss：

```jsonl
{"event":"cache.integrity_failed","key":"agent-course:lesson-index:v1","reason":"signature_mismatch","action":"evict_and_refetch","runId":"cron:3eba6ee3"}
```

这能区分：

- 正常过期：可忽略；
- schema 过旧：需要迁移；
- 完整性失败：可能是 bug、手工误改或安全事件。

## 7. 生产注意事项

### 7.1 Canonicalization 必须稳定

签名最容易踩坑的是 JSON 序列化不稳定：

- key 顺序不同；
- `undefined` / `null` 处理不同；
- 时间格式不同；
- 浮点数格式不同；
- Unicode 转义不同。

生产里用明确的 canonical JSON 库，或者把签名字段限制成简单字符串/数字。

### 7.2 keyId 必须进入 envelope 外层

签名需要 `keyId` 找密钥，但签名 payload 通常不包含 signature 本身。推荐：

```text
integrity.keyId 用来选 key
canonical(envelope) 用来验签
```

轮换时读路径保留旧 key，只允许新写入使用 active key。

### 7.3 验签失败的策略按风险分级

- low risk answer：evict + refetch；
- planning：evict + refetch + audit；
- external_side_effect：block，必须 live check；
- security_decision：block + alert。

不要在高风险路径里“验签失败但继续用”。

### 7.4 签名不是加密

签名保证“没被改”和“来自可信写入者”，不保证没人看见。敏感字段仍然要沿用第 269 课的字段级加密与分层披露。

## 8. 小结

成熟 Agent 的缓存读取链路应该变成：

```text
read cache
  -> verify integrity signature
  -> verify valueHash
  -> validate schema/invariants/freshness/purpose
  -> check lineage / policy
  -> use, revalidate, evict, or block
```

今天的重点：

1. `valueHash` 只能防意外改动，不能防“改完重算 hash”；
2. 签名要覆盖关键 metadata，不要只签 value；
3. 单服务可用 HMAC，多服务建议 Ed25519；
4. 验签失败是安全信号，要审计、驱逐或阻断；
5. 签名解决完整性，不解决保密性。

下一次你给 Agent 做缓存时，可以问自己一句：**这条缓存如果被人手工改了，系统会发现吗？**
