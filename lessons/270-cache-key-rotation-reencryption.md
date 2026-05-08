# 270. Agent 缓存密钥轮换与重加密（Cache Key Rotation & Re-encryption）

> 字段加密只是第一步。真正上线后，密钥会泄露、员工会离职、KMS policy 会调整、算法会升级。成熟 Agent 的缓存加密必须支持“不断服务的同时换钥匙”：新写入用新 key，旧数据渐进重加密，读路径能兼容多版本，出问题还能回滚。

上一课讲了缓存敏感字段分层：public/internal/sensitive/secret。今天继续往生产走一步：**缓存密钥轮换与重加密**。

一句话：**加密缓存不要只问“能不能解密”，还要问“以后怎么安全换钥匙”**。

## 1. 为什么 Agent 缓存需要 key rotation？

Agent 系统里的缓存通常活得比一次对话更久：

- L2/L3 缓存可能跨 session、跨 worker 共享；
- 备份、审计、迁移任务会长期保留旧 envelope；
- 敏感字段加密后仍然要支持检索、刷新和回源；
- 某个 key 泄露后，不能靠“清空全部缓存”解决所有问题。

如果缓存 envelope 里没有 `keyId`、`algorithm`、`version`，后面就会变成灾难：

```json
{
  "encrypted_sensitive": {
    "email": "gAAAAAB..."
  }
}
```

这份数据三个月后没人知道该用哪把 key 解，也不知道是否已经完成轮换。

## 2. 正确 envelope：每个密文字段带 key 元数据

建议敏感字段不要只存字符串，而是存一个 `EncryptedField`：

```json
{
  "schemaVersion": 2,
  "keyClass": "user_profile",
  "public": { "name": "Alice" },
  "encryptedSensitive": {
    "email": {
      "alg": "AES-256-GCM",
      "keyId": "cache-user-v3",
      "ciphertext": "base64...",
      "aad": "tenant:acme:key:user_profile"
    }
  }
}
```

关键字段：

- `keyId`：解密用哪把 key；
- `alg`：算法版本，方便未来迁移；
- `aad`：Additional Authenticated Data，把 tenant/keyClass/cacheKey 绑定进认证数据，防止密文被搬到别的租户还能解；
- `schemaVersion`：envelope 结构版本，支持迁移。

## 3. 双阶段轮换：读兼容，写前进

密钥轮换不要一刀切。推荐三步：

1. **Prepare**：KMS/Secret Manager 发布新 key，旧 key 仍可 decrypt；
2. **Write-forward**：所有新写入使用新 `keyId`，读路径同时支持新旧 key；
3. **Re-encrypt**：后台扫描旧 envelope，读旧 key 解密，再用新 key 写回；完成后旧 key 进入 decrypt-only，再最终 revoke。

注意：旧 key 在重加密完成前不要立刻删除，否则缓存会出现大面积不可读。

## 4. learn-claude-code：Python 教学版

```python
# learn-claude-code/key_rotation_cache.py
from __future__ import annotations

from dataclasses import dataclass
from typing import Any
import base64
import json


@dataclass(frozen=True)
class EncryptedField:
    alg: str
    key_id: str
    ciphertext: str
    aad: str


class Keyring:
    def __init__(self, active_key_id: str, keys: dict[str, bytes]) -> None:
        self.active_key_id = active_key_id
        self.keys = keys

    def encrypt(self, value: Any, *, aad: str) -> EncryptedField:
        # 教学版用 base64 模拟；生产请用 AES-GCM/KMS envelope encryption
        raw = json.dumps(value, ensure_ascii=False).encode()
        token = base64.b64encode(self.keys[self.active_key_id] + b":" + raw).decode()
        return EncryptedField(
            alg="demo-base64", key_id=self.active_key_id, ciphertext=token, aad=aad
        )

    def decrypt(self, field: EncryptedField, *, expected_aad: str) -> Any:
        if field.aad != expected_aad:
            raise ValueError("AAD mismatch: ciphertext moved across boundary")
        key = self.keys[field.key_id]
        raw = base64.b64decode(field.ciphertext.encode())
        prefix, payload = raw.split(b":", 1)
        if prefix != key:
            raise ValueError("wrong key")
        return json.loads(payload.decode())


def needs_rotation(field: EncryptedField, active_key_id: str) -> bool:
    return field.key_id != active_key_id


def reencrypt_field(field: EncryptedField, *, keyring: Keyring, aad: str) -> EncryptedField:
    value = keyring.decrypt(field, expected_aad=aad)
    return keyring.encrypt(value, aad=aad)
```

真实系统里 `Keyring` 不应该把 key material 暴露给 LLM；它属于工具执行层。LLM 只能看到 `keyId`、`status`、`rotationProgress` 这类元数据。

## 5. pi-mono：TypeScript 中间件形态

```ts
// pi-mono/cache/KeyRotationMiddleware.ts
type EncryptedField = {
  alg: "AES-256-GCM";
  keyId: string;
  ciphertext: string;
  aad: string;
};

type CacheEnvelope = {
  schemaVersion: number;
  keyClass: string;
  encryptedSensitive: Record<string, EncryptedField>;
};

type CryptoProvider = {
  activeKeyId(keyClass: string): Promise<string>;
  decrypt(field: EncryptedField, expectedAad: string): Promise<unknown>;
  encrypt(value: unknown, keyClass: string, aad: string): Promise<EncryptedField>;
};

export class KeyRotationMiddleware {
  constructor(private crypto: CryptoProvider) {}

  async onCacheRead(key: string, envelope: CacheEnvelope) {
    const activeKeyId = await this.crypto.activeKeyId(envelope.keyClass);
    const rotated: string[] = [];

    for (const [name, field] of Object.entries(envelope.encryptedSensitive)) {
      if (field.keyId !== activeKeyId) {
        rotated.push(name); // 记录：需要后台重加密，不在读路径同步写大对象
      }
    }

    return {
      envelope,
      rotationHint: rotated.length
        ? { action: "schedule_reencrypt", fields: rotated, cacheKey: key }
        : null,
    };
  }

  async reencryptEnvelope(key: string, envelope: CacheEnvelope) {
    const next: CacheEnvelope = { ...envelope, encryptedSensitive: { ...envelope.encryptedSensitive } };

    for (const [name, field] of Object.entries(envelope.encryptedSensitive)) {
      const aad = `${envelope.keyClass}:${key}:${name}`;
      const value = await this.crypto.decrypt(field, aad);
      next.encryptedSensitive[name] = await this.crypto.encrypt(value, envelope.keyClass, aad);
    }

    return next;
  }
}
```

生产建议：读路径只返回 `rotationHint`，由后台 worker 异步重加密，避免一次用户请求突然承担大量 CPU/KMS 成本。

## 6. OpenClaw 实战：文件缓存怎么做

OpenClaw 这种长期运行 Agent，很适合用一个小型 rotation manifest：

```json
{
  "cache:user_profile": {
    "activeKeyId": "cache-user-v3",
    "decryptKeyIds": ["cache-user-v2", "cache-user-v3"],
    "revokedKeyIds": ["cache-user-v1"],
    "rotationStartedAt": "2026-05-08T11:30:00Z"
  }
}
```

执行策略：

- 写入缓存前查 manifest：只允许 active key 加密；
- 读取缓存时：`keyId` 必须在 `decryptKeyIds`；
- 若读到 revoked key：禁止解密，直接回源或标记 blocked；
- 心跳/cron 扫描旧 envelope：限速重加密，记录进度；
- 完成率 100% 后再把旧 key 从 decrypt-only 移到 revoked。

可以配一个 JSONL 审计日志：

```jsonl
{"event":"cache_key_rotation_started","keyClass":"user_profile","from":"v2","to":"v3"}
{"event":"cache_reencrypted","cacheKeyHash":"abc123","fields":["email"],"to":"v3"}
{"event":"cache_key_revoked","keyClass":"user_profile","keyId":"v2"}
```

## 7. 常见坑

- **只换新写入，不处理旧缓存**：半年后旧 key 仍然不能删；
- **读路径同步重加密所有字段**：用户请求变慢，还容易打爆 KMS；
- **没有 AAD**：密文可能被复制到别的 tenant/keyClass 后误解密；
- **没有进度指标**：不知道什么时候能安全 revoke 旧 key；
- **把 key value 放进 prompt**：这是红线，LLM 只需要 key metadata。

## 8. 小结

今天的模式可以压成四句话：

1. 每个密文字段都要带 `keyId/alg/aad`；
2. 轮换分 prepare、write-forward、reencrypt、revoke；
3. 读路径兼容旧 key，但不要同步做重活；
4. key material 永远留在工具层，LLM 只看脱敏元数据。

缓存加密解决“别人拿到缓存看不懂”，密钥轮换解决“旧风险不会永久跟着系统走”。成熟 Agent 不只是会保护数据，还要能持续更新保护方式。