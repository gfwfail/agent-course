# 271. Agent 缓存密钥访问授权与用途绑定（Cache Key Access & Purpose Binding）

> 上一课讲 key rotation：密钥要能换。今天补上另一个生产级问题：**谁可以用这把 key、为了什么用途用、用完有没有留下证据**。如果任何工具/worker 拿到 `keyId` 就能解密，那字段加密只是把风险从“缓存泄露”搬到了“密钥误用”。

一句话：**密钥访问必须绑定 actor、purpose、keyClass、tenant 和 cacheKey；能解密不代表应该解密。**

## 1. 为什么只靠 `keyId` 不够？

上一课的 envelope 里每个密文字段都有：

```json
{
  "alg": "AES-256-GCM",
  "keyId": "cache-user-v3",
  "ciphertext": "base64...",
  "aad": "tenant:acme:key:user_profile"
}
```

这能解决“用哪把 key 解密”和“AAD 防搬运”。但还没解决三个权限问题：

- 后台预热 worker 是否能解密用户邮箱？
- 普通回答场景是否能读取 `payment_token`？
- 子 Agent 是否能拿父 Agent 的缓存密文字段做计划？

生产里要把 decrypt 当成高风险工具调用，而不是普通函数调用。

## 2. Key Access Grant：短期、最小权限、带用途

推荐引入一个 `KeyAccessGrant`，每次解密前先拿授权：

```json
{
  "grantId": "grant_01HX...",
  "actor": "agent:main",
  "tenant": "acme",
  "keyClass": "user_profile",
  "allowedPurposes": ["answer", "reencrypt"],
  "allowedFields": ["email", "timezone"],
  "cacheKeyHash": "sha256:abc123",
  "expiresAt": "2026-05-08T15:00:00Z"
}
```

几个原则：

- **grant 短期有效**：分钟级 TTL，不长期保存；
- **用途绑定**：`answer`、`planning`、`reencrypt`、`external_side_effect` 分开；
- **字段白名单**：能解 `email` 不等于能解 `payment_token`；
- **cacheKeyHash 绑定**：不能拿 A 缓存的 grant 去解 B 缓存；
- **审计必留痕**：每次 decrypt 都记录 grantId、purpose、decision。

## 3. learn-claude-code：Python 教学版

```python
# learn-claude-code/key_access_grant.py
from __future__ import annotations

from dataclasses import dataclass
from datetime import datetime, timezone
from hashlib import sha256
from typing import Literal

Purpose = Literal["answer", "planning", "reencrypt", "external_side_effect"]


@dataclass(frozen=True)
class KeyAccessGrant:
    grant_id: str
    actor: str
    tenant: str
    key_class: str
    allowed_purposes: set[Purpose]
    allowed_fields: set[str]
    cache_key_hash: str
    expires_at: datetime


def hash_cache_key(cache_key: str) -> str:
    return "sha256:" + sha256(cache_key.encode()).hexdigest()


def authorize_decrypt(
    grant: KeyAccessGrant,
    *,
    actor: str,
    tenant: str,
    key_class: str,
    field: str,
    purpose: Purpose,
    cache_key: str,
    now: datetime | None = None,
) -> None:
    now = now or datetime.now(timezone.utc)
    if now >= grant.expires_at:
        raise PermissionError("grant expired")
    if grant.actor != actor or grant.tenant != tenant:
        raise PermissionError("actor/tenant mismatch")
    if grant.key_class != key_class:
        raise PermissionError("key class mismatch")
    if purpose not in grant.allowed_purposes:
        raise PermissionError(f"purpose denied: {purpose}")
    if field not in grant.allowed_fields:
        raise PermissionError(f"field denied: {field}")
    if grant.cache_key_hash != hash_cache_key(cache_key):
        raise PermissionError("cache key mismatch")
```

这个教学版故意不做加密细节，只强调一点：**decrypt 前必须先过授权函数**。真实系统里它可以接 KMS grant、Vault token、OPA/Cedar policy 或内部 policy engine。

## 4. pi-mono：TypeScript 中间件形态

```ts
// pi-mono/cache/KeyAccessMiddleware.ts
type Purpose = "answer" | "planning" | "reencrypt" | "external_side_effect";

type DecryptRequest = {
  actor: string;
  tenant: string;
  keyClass: string;
  field: string;
  purpose: Purpose;
  cacheKey: string;
  grantId: string;
};

type KeyGrantService = {
  authorize(req: DecryptRequest): Promise<{ allow: true } | { allow: false; reason: string }>;
};

type CryptoProvider = {
  decrypt(field: { ciphertext: string; aad: string; keyId: string }, expectedAad: string): Promise<unknown>;
};

export class KeyAccessMiddleware {
  constructor(private grants: KeyGrantService, private crypto: CryptoProvider) {}

  async decryptField(req: DecryptRequest, encrypted: { ciphertext: string; aad: string; keyId: string }) {
    const decision = await this.grants.authorize(req);

    audit("cache_decrypt_decision", {
      actor: req.actor,
      tenant: req.tenant,
      keyClass: req.keyClass,
      field: req.field,
      purpose: req.purpose,
      grantId: req.grantId,
      decision: decision.allow ? "allow" : "deny",
      reason: decision.allow ? undefined : decision.reason,
    });

    if (!decision.allow) {
      throw new Error(`decrypt denied: ${decision.reason}`);
    }

    const aad = `${req.tenant}:${req.keyClass}:${req.cacheKey}:${req.field}:${req.purpose}`;
    return this.crypto.decrypt(encrypted, aad);
  }
}

function audit(event: string, payload: Record<string, unknown>) {
  console.log(JSON.stringify({ event, ts: new Date().toISOString(), ...payload }));
}
```

注意这里的 AAD 里也放了 `purpose`。这样同一份密文如果是为 `answer` 场景写入，就不能被偷偷拿去做 `external_side_effect` 场景的解密输入。

## 5. OpenClaw 实战：哪些场景允许解密？

可以给文件缓存或工具缓存加一个小 policy：

```json
{
  "cache:user_profile": {
    "answer": ["name", "email", "timezone"],
    "planning": ["timezone"],
    "reencrypt": ["*"],
    "external_side_effect": []
  },
  "cache:billing": {
    "answer": ["invoice_total", "currency"],
    "planning": [],
    "reencrypt": ["*"],
    "external_side_effect": []
  }
}
```

落地规则：

- 普通回复：只给 `answer` grant；
- 子 Agent：默认不给 sensitive decrypt grant，除非父任务显式委托；
- cron 预热：通常只能刷新 public/internal 字段；
- key rotation worker：给 `reencrypt` grant，但不把明文注入 LLM；
- 发消息、下单、删资源等副作用：不允许直接使用缓存解密结果，必须 live check 或重新授权。

## 6. 常见坑

- **把 decrypt 当内部 helper**：调用点散落全项目，后面无法审计；
- **grant 不绑定 purpose**：回答场景拿到的权限被复用到副作用场景；
- **只按 keyClass 授权**：`user_profile` 里不同字段风险差异很大；
- **后台任务权限过大**：预热/重加密 worker 变成超级解密器；
- **明文进入日志/Prompt**：授权解密成功后，还要经过脱敏和注入策略。

## 7. 小结

今天的模式可以压成四句话：

1. decrypt 是高风险工具调用，不是普通函数；
2. 用 `KeyAccessGrant` 绑定 actor、tenant、purpose、field、cacheKey 和 TTL；
3. policy 决定“为了什么用途能看哪些字段”；
4. key rotation worker 可以重加密，但不应该把明文交给 LLM。

缓存加密保护静态数据，密钥轮换保护长期风险，而密钥访问授权保护运行时边界。成熟 Agent 的安全不是“有一把锁”，而是每次开锁都要说明：谁、为什么、开哪一扇门。