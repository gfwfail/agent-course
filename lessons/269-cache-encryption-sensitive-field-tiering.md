# 269. Agent 缓存加密与敏感字段分层（Cache Encryption & Sensitive Field Tiering）

> 缓存不是保险柜。Agent 把工具结果、用户偏好、检索片段、网页摘要写进缓存后，如果不区分敏感级别，缓存就会变成“更快的数据泄露通道”。成熟做法不是简单地“全都加密”，而是按字段敏感度分层：能不存就不存，必须存就最小化，加密只保护该保护的部分。

前面我们讲了缓存权限、租户隔离、投毒防御、审计日志。今天补一块生产系统很关键的能力：**缓存加密与敏感字段分层**。

一句话：**缓存 value 不是一坨字符串，而是一份带安全标签的数据资产**。

## 1. 为什么“全量缓存”很危险？

Agent 缓存里经常混着这些数据：

- 用户身份、邮箱、手机号、Telegram ID；
- API 响应里的 access token、订单号、地址；
- 私聊里的偏好、记忆、业务决策；
- RAG 文档片段、网页抓取结果、LLM 摘要；
- 子 Agent 交接状态、工具执行结果。

如果直接 `cache.set(key, JSON.stringify(result))`，会有三个问题：

1. **过度存储**：只需要 title，却把整份用户资料缓存了。
2. **权限扩散**：原始敏感字段被 L2/L3 缓存、日志、备份、调试工具复制多份。
3. **误注入 LLM**：下次 cache hit 时，敏感字段被直接塞回上下文。

所以缓存写入前要先做字段分层。

## 2. 四级字段分层

建议给缓存字段打四类标签：

```text
public        可明文缓存，可注入 LLM，例如文档标题、公开价格
internal      可缓存，但默认不展示，例如内部 id、sourceVersion
sensitive     可加密缓存，只有工具层可解密，例如邮箱、地址、订单详情
secret        不进入缓存，只保存 SecretRef，例如 API key、token、密码
```

对应策略：

| 级别 | 缓存策略 | LLM 注入 | 日志 |
|---|---|---|---|
| public | 明文 | 允许 | 可记录摘要 |
| internal | 明文或 hash | 按需 | hash/摘要 |
| sensitive | 字段级加密 | 默认禁止，必要时脱敏 | hash |
| secret | 不缓存原文 | 禁止 | SecretRef |

重点：**加密不是万能许可**。如果 Agent 不需要这个字段完成任务，最好的策略是根本别存。

## 3. learn-claude-code：Python 教学版

```python
# learn-claude-code/secure_cache_value.py
from __future__ import annotations

import base64
import json
from dataclasses import dataclass, asdict
from typing import Any, Literal
from cryptography.fernet import Fernet

Sensitivity = Literal["public", "internal", "sensitive", "secret"]


@dataclass(frozen=True)
class FieldPolicy:
    name: str
    sensitivity: Sensitivity


@dataclass
class CacheEnvelope:
    schema_version: int
    key_class: str
    public: dict[str, Any]
    internal: dict[str, Any]
    encrypted_sensitive: dict[str, str]
    secret_refs: dict[str, str]


class FieldEncryptor:
    def __init__(self, key: bytes) -> None:
        self.fernet = Fernet(key)

    def encrypt_json(self, value: Any) -> str:
        raw = json.dumps(value, ensure_ascii=False).encode("utf-8")
        return self.fernet.encrypt(raw).decode("utf-8")

    def decrypt_json(self, token: str) -> Any:
        raw = self.fernet.decrypt(token.encode("utf-8"))
        return json.loads(raw.decode("utf-8"))


def build_cache_envelope(
    *,
    key_class: str,
    data: dict[str, Any],
    policies: list[FieldPolicy],
    encryptor: FieldEncryptor,
) -> CacheEnvelope:
    public: dict[str, Any] = {}
    internal: dict[str, Any] = {}
    encrypted_sensitive: dict[str, str] = {}
    secret_refs: dict[str, str] = {}
    policy_by_name = {p.name: p.sensitivity for p in policies}

    for name, value in data.items():
        level = policy_by_name.get(name, "sensitive")  # 未声明字段默认敏感

        if level == "public":
            public[name] = value
        elif level == "internal":
            internal[name] = value
        elif level == "sensitive":
            encrypted_sensitive[name] = encryptor.encrypt_json(value)
        elif level == "secret":
            # 不缓存 secret 原文，只保存引用；真实值由 Secret Manager 在工具执行层解析
            secret_refs[name] = f"secretref:{key_class}:{name}"

    return CacheEnvelope(
        schema_version=1,
        key_class=key_class,
        public=public,
        internal=internal,
        encrypted_sensitive=encrypted_sensitive,
        secret_refs=secret_refs,
    )


def to_llm_context(envelope: CacheEnvelope) -> dict[str, Any]:
    # 默认只给 LLM public + 必要 internal，不解密 sensitive
    return {
        "key_class": envelope.key_class,
        "public": envelope.public,
        "source": envelope.internal.get("source"),
    }
```

教学版关键点：

- 未声明字段默认 `sensitive`，不要默认 public。
- `secret` 不进入缓存，只存引用。
- LLM 上下文生成函数默认不解密 sensitive。

## 4. pi-mono：TypeScript 生产版中间件

生产里可以把字段分层做成 Cache Middleware，写入前自动包装 value，读取后按 purpose 决定能否解密。

```ts
// pi-mono/packages/agent-runtime/cache/sensitive-cache.ts
import { createCipheriv, createDecipheriv, randomBytes } from "node:crypto";

export type Sensitivity = "public" | "internal" | "sensitive" | "secret";
export type CachePurpose = "answer" | "planning" | "external_side_effect" | "security_decision";

export interface FieldRule {
  path: string;
  sensitivity: Sensitivity;
}

export interface SecureCacheEnvelope {
  schemaVersion: 1;
  keyClass: string;
  public: Record<string, unknown>;
  internal: Record<string, unknown>;
  encryptedSensitive: Record<string, { alg: "aes-256-gcm"; iv: string; tag: string; data: string }>;
  secretRefs: Record<string, string>;
}

export interface KeyProvider {
  getDataKey(keyClass: string, tenantId: string): Promise<Buffer>; // KMS/Vault 派生
}

function encryptJson(value: unknown, key: Buffer) {
  const iv = randomBytes(12);
  const cipher = createCipheriv("aes-256-gcm", key, iv);
  const plaintext = Buffer.from(JSON.stringify(value), "utf8");
  const data = Buffer.concat([cipher.update(plaintext), cipher.final()]);
  const tag = cipher.getAuthTag();
  return {
    alg: "aes-256-gcm" as const,
    iv: iv.toString("base64"),
    tag: tag.toString("base64"),
    data: data.toString("base64"),
  };
}

function decryptJson(payload: SecureCacheEnvelope["encryptedSensitive"][string], key: Buffer) {
  const decipher = createDecipheriv("aes-256-gcm", key, Buffer.from(payload.iv, "base64"));
  decipher.setAuthTag(Buffer.from(payload.tag, "base64"));
  const raw = Buffer.concat([
    decipher.update(Buffer.from(payload.data, "base64")),
    decipher.final(),
  ]);
  return JSON.parse(raw.toString("utf8"));
}

export class SensitiveCacheCodec {
  constructor(private readonly keyProvider: KeyProvider) {}

  async encode(input: {
    keyClass: string;
    tenantId: string;
    value: Record<string, unknown>;
    rules: FieldRule[];
  }): Promise<SecureCacheEnvelope> {
    const key = await this.keyProvider.getDataKey(input.keyClass, input.tenantId);
    const rules = new Map(input.rules.map((r) => [r.path, r.sensitivity]));

    const envelope: SecureCacheEnvelope = {
      schemaVersion: 1,
      keyClass: input.keyClass,
      public: {},
      internal: {},
      encryptedSensitive: {},
      secretRefs: {},
    };

    for (const [field, value] of Object.entries(input.value)) {
      const level = rules.get(field) ?? "sensitive";
      if (level === "public") envelope.public[field] = value;
      if (level === "internal") envelope.internal[field] = value;
      if (level === "sensitive") envelope.encryptedSensitive[field] = encryptJson(value, key);
      if (level === "secret") envelope.secretRefs[field] = `secretref:${input.keyClass}:${field}`;
    }

    return envelope;
  }

  async decodeForPurpose(input: {
    envelope: SecureCacheEnvelope;
    tenantId: string;
    purpose: CachePurpose;
  }): Promise<Record<string, unknown>> {
    // 回答和规划默认不解密敏感字段，避免误注入 LLM
    if (input.purpose === "answer" || input.purpose === "planning") {
      return { ...input.envelope.public, _source: input.envelope.internal.source };
    }

    // 外部副作用/安全决策需要敏感字段时，必须在工具层解密，并配合审计日志
    const key = await this.keyProvider.getDataKey(input.envelope.keyClass, input.tenantId);
    const sensitive = Object.fromEntries(
      Object.entries(input.envelope.encryptedSensitive).map(([name, payload]) => [
        name,
        decryptJson(payload, key),
      ]),
    );

    return { ...input.envelope.public, ...input.envelope.internal, ...sensitive };
  }
}
```

这段中间件要和前几课结合：

- 和权限边界结合：data key 按 `tenantId + keyClass` 派生。
- 和审计日志结合：每次解密 sensitive 字段都写 audit event。
- 和准入策略结合：`secret` 字段直接拒绝缓存原文。
- 和生命周期结合：sensitive 字段 TTL 更短，禁止 stale。

## 5. OpenClaw 实战：文件缓存怎么做？

OpenClaw 这种本地文件/长期记忆场景，建议最少做到三件事：

```text
memory/cache/
  public/      # 可复用公开资料、课程索引、非敏感摘要
  scoped/      # 带 actor/tenant/scope 的私有缓存
  secure/      # envelope 格式，敏感字段加密或只存 SecretRef
```

写入前先问四个问题：

1. 这个字段下次真的需要吗？不需要就丢。
2. 这个字段能不能给 LLM 看？不能就不要进入 prompt context。
3. 这个字段是否属于 secret？是就只存引用。
4. 这个字段多久后必须失效？敏感数据 TTL 要短于普通缓存。

一个实际例子：缓存邮件搜索结果时，不要缓存完整邮件原文。可以这样分层：

```json
{
  "public": {
    "subject": "Invoice for May",
    "senderDomain": "stripe.com"
  },
  "internal": {
    "messageId": "msg_123",
    "observedAt": "2026-05-08T08:30:00Z"
  },
  "encryptedSensitive": {
    "snippet": "...aes-gcm..."
  },
  "secretRefs": {}
}
```

LLM 做摘要时只看到 subject/domain；真正需要打开邮件时，再通过工具层按权限读取原文。

## 6. 设计 Checklist

- [ ] cache value 是否有 envelope，而不是裸 JSON？
- [ ] 字段是否显式标注 public/internal/sensitive/secret？
- [ ] 未声明字段是否默认 sensitive？
- [ ] secret 是否只存 SecretRef，不存原文？
- [ ] sensitive 是否字段级加密，而不是整库裸存？
- [ ] LLM context 是否默认只注入 public 字段？
- [ ] 每次 sensitive 解密是否写审计日志？
- [ ] sensitive 缓存是否禁止 stale、TTL 更短？
- [ ] data key 是否按 tenant/keyClass 隔离？

## 7. 常见坑

**坑 1：整份 JSON 加密，然后读取时全量解密塞给 LLM**  
这只是把风险从磁盘移动到了上下文。正确做法是字段级分层，默认不解密 sensitive。

**坑 2：把 API token 当 sensitive 加密缓存**  
token/password/API key 应该是 `secret`，不进入缓存。缓存里只放 SecretRef，由工具执行层临时取用。

**坑 3：日志里打印解密后的 value**  
审计日志记录 hash、字段名、用途、actor，不记录敏感原文。

**坑 4：所有租户共用一把缓存加密 key**  
一旦泄露就是全局事故。至少按 tenant 派生 data key，更高要求可按 keyClass/region 再细分。

## 8. 和前面缓存课的关系

- 权限边界解决“谁能读”；字段分层解决“读到什么”。
- 审计日志解决“谁什么时候解密过”；字段加密解决“静态存储泄露时损失多大”。
- 准入策略决定“值不值得缓存”；字段分层决定“哪些字段能缓存”。
- 生命周期策略决定“何时忘记”；敏感字段通常应该更快忘记。

## 总结

Agent 缓存的安全边界不能只靠“这个 key 应该没人猜得到”。生产级缓存要把 value 当成结构化资产管理：字段分层、SecretRef、字段级加密、按用途解密、审计留痕。

记住一句话：**缓存不是保险柜；只有最小化 + 分层 + 审计，才是 Agent 缓存安全的基本盘。**
