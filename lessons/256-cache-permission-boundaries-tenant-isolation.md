# 256. Agent 缓存权限边界与租户隔离（Cache Permission Boundaries & Tenant Isolation）

前几课讲了缓存失效、击穿、预热、可观测性和一致性。今天补一个更容易出事故的问题：**缓存里的数据不是谁都能读**。

一句话：

> Agent 缓存 key 里必须带权限边界，cache hit 之后也要重新校验 scope；否则缓存会把“上一个用户能看的东西”错误地交给“下一个用户”。

这课特别适合多租户 Agent、客服 Agent、后台运营 Agent、企业知识库 Agent。

---

## 1. 事故长什么样

很多人第一次写缓存会这样：

```text
cache key = "order:123"
```

问题是：

```text
Tenant A 的用户查 order:123 -> 缓存命中 order:123
Tenant B 的用户也查 order:123 -> 也命中同一个 key
```

如果订单 ID 在不同租户内是局部递增，或者工具结果里混了用户权限过滤后的字段，就会直接越权。

更隐蔽的是 Agent 场景：

1. Agent A 以 admin scope 查询客户资料，缓存了完整字段；
2. Agent B 以 support scope 查询同一个客户；
3. 缓存层只看 `customer:42`，直接返回 admin 版本；
4. LLM 把敏感字段写进回复、日志或长期记忆。

所以缓存安全不是“外面做过 auth 就行”，缓存层自己也要懂边界。

---

## 2. 三条生产规则

### 规则 1：Cache Key 必须包含租户和权限指纹

不要只用资源 ID：

```text
bad:  customer:42
```

至少包含：

```text
good: tenant:{tenant_id}:actor:{actor_id}:scope:{scope_hash}:customer:42
```

如果数据是全租户共享但字段受权限影响，可以不用 `actor_id`，但必须带 `scope_hash`。

### 规则 2：Cache Value 必须记录生产它时的权限上下文

缓存值不要只存 payload：

```json
{
  "payload": { "id": 42, "email": "..." },
  "tenant_id": "t_1",
  "scope_hash": "a13f",
  "produced_by": "support.read",
  "created_at": 1778088600
}
```

命中后先检查 metadata，再把 payload 交给 Agent。

### 规则 3：敏感工具结果默认不跨 actor 复用

这些结果不要做全局共享缓存：

- 用户资料、订单、付款、工单；
- 内部日志、审计结果；
- GitHub private repo issue/PR；
- Gmail/Drive/Slack 搜索结果；
- 任何带 PII 或商业机密的数据。

可以缓存，但只能在明确边界内缓存。

---

## 3. learn-claude-code：Python 教学版

下面是一个最小实现：cache key 由 resource + security context 共同决定，命中后还会二次校验。

```python
# learn_claude_code/secure_cache.py
from __future__ import annotations

import hashlib
import json
import time
from dataclasses import dataclass, asdict
from typing import Any, Callable

@dataclass(frozen=True)
class SecurityContext:
    tenant_id: str
    actor_id: str
    scopes: tuple[str, ...]

    def scope_hash(self) -> str:
        raw = ",".join(sorted(self.scopes))
        return hashlib.sha256(raw.encode()).hexdigest()[:12]

@dataclass
class CacheEnvelope:
    payload: Any
    tenant_id: str
    actor_id: str | None
    scope_hash: str
    created_at: float

class SecureCache:
    def __init__(self) -> None:
        self.store: dict[str, CacheEnvelope] = {}

    def key(self, ctx: SecurityContext, resource: str, *, share_across_actor: bool = False) -> str:
        actor = "shared" if share_across_actor else ctx.actor_id
        return f"tenant:{ctx.tenant_id}:actor:{actor}:scope:{ctx.scope_hash()}:{resource}"

    def get_or_load(
        self,
        ctx: SecurityContext,
        resource: str,
        loader: Callable[[], Any],
        *,
        share_across_actor: bool = False,
        ttl_seconds: int = 300,
    ) -> Any:
        key = self.key(ctx, resource, share_across_actor=share_across_actor)
        env = self.store.get(key)

        if env and self._allowed(ctx, env, share_across_actor) and time.time() - env.created_at <= ttl_seconds:
            return env.payload

        payload = loader()
        self.store[key] = CacheEnvelope(
            payload=payload,
            tenant_id=ctx.tenant_id,
            actor_id=None if share_across_actor else ctx.actor_id,
            scope_hash=ctx.scope_hash(),
            created_at=time.time(),
        )
        return payload

    def _allowed(self, ctx: SecurityContext, env: CacheEnvelope, share_across_actor: bool) -> bool:
        if env.tenant_id != ctx.tenant_id:
            return False
        if env.scope_hash != ctx.scope_hash():
            return False
        if not share_across_actor and env.actor_id != ctx.actor_id:
            return False
        return True
```

使用方式：

```python
ctx = SecurityContext(
    tenant_id="tenant_a",
    actor_id="user_9",
    scopes=("orders:read",),
)

order = cache.get_or_load(
    ctx,
    "order:123",
    loader=lambda: order_api.get_order("123"),
    ttl_seconds=60,
)
```

注意：`share_across_actor=True` 只能用于“同租户同权限看到的数据完全一致”的场景，例如公开产品目录，不适合订单和用户资料。

---

## 4. pi-mono：TypeScript 生产中间件

生产系统里建议把它做成工具中间件，所有 tool call 自动经过边界检查。

```ts
// pi-mono/packages/agent/src/cache/SecureToolCache.ts
import crypto from "node:crypto";

type SecurityContext = {
  tenantId: string;
  actorId: string;
  scopes: string[];
};

type CacheEnvelope<T> = {
  payload: T;
  tenantId: string;
  actorId: string | null;
  scopeHash: string;
  createdAt: number;
};

type ToolCachePolicy = {
  ttlMs: number;
  shareAcrossActor?: boolean;
  cacheable?: boolean;
};

const scopeHash = (scopes: string[]) =>
  crypto.createHash("sha256").update([...scopes].sort().join(",")).digest("hex").slice(0, 16);

export class SecureToolCache {
  constructor(private readonly kv: Map<string, CacheEnvelope<unknown>>) {}

  async run<T>(args: {
    ctx: SecurityContext;
    toolName: string;
    input: unknown;
    policy: ToolCachePolicy;
    execute: () => Promise<T>;
  }): Promise<T> {
    const { ctx, toolName, input, policy, execute } = args;

    if (policy.cacheable === false) return execute();

    const hash = scopeHash(ctx.scopes);
    const actorPart = policy.shareAcrossActor ? "shared" : ctx.actorId;
    const inputHash = crypto.createHash("sha256").update(JSON.stringify(input)).digest("hex").slice(0, 16);
    const key = `tenant:${ctx.tenantId}:actor:${actorPart}:scope:${hash}:tool:${toolName}:input:${inputHash}`;

    const cached = this.kv.get(key) as CacheEnvelope<T> | undefined;
    if (cached && this.allowed(ctx, cached, policy) && Date.now() - cached.createdAt < policy.ttlMs) {
      return cached.payload;
    }

    const payload = await execute();
    this.kv.set(key, {
      payload,
      tenantId: ctx.tenantId,
      actorId: policy.shareAcrossActor ? null : ctx.actorId,
      scopeHash: hash,
      createdAt: Date.now(),
    });
    return payload;
  }

  private allowed<T>(ctx: SecurityContext, env: CacheEnvelope<T>, policy: ToolCachePolicy) {
    if (env.tenantId !== ctx.tenantId) return false;
    if (env.scopeHash !== scopeHash(ctx.scopes)) return false;
    if (!policy.shareAcrossActor && env.actorId !== ctx.actorId) return false;
    return true;
  }
}
```

工具注册时直接标策略：

```ts
registry.register("orders.search", searchOrders, {
  cache: {
    cacheable: true,
    ttlMs: 60_000,
    shareAcrossActor: false,
  },
});

registry.register("plans.publicCatalog", listPlans, {
  cache: {
    cacheable: true,
    ttlMs: 10 * 60_000,
    shareAcrossActor: true,
  },
});
```

重点：**是否共享缓存由工具作者声明，不让 LLM 临时决定。**

---

## 5. OpenClaw 实战：文件缓存也要带边界

OpenClaw 里很多长期运行任务会把中间状态写到 workspace，比如：

```text
memory/heartbeat-state.json
.cache/github-prs.json
.cache/gmail-unread.json
```

如果这些文件会被不同用户、不同群、不同账号复用，就要把边界写进路径：

```text
.cache/telegram/-5115329245/github-prs.json
.cache/user/67431246/gmail-unread.json
.cache/tenant/metadraw/billing-costs.json
```

高风险建议：

1. 文件名带 `channel_id` / `user_id` / `tenant_id`；
2. 文件内容带 metadata；
3. 读取后校验 metadata，不匹配就当 cache miss；
4. 写文件用原子写：先写 `.tmp`，再 rename；
5. 敏感缓存设置短 TTL，并避免进入 LLM 长期记忆。

示例：

```python
# openclaw_cache_file.py
import json
import os
import time
from pathlib import Path

def write_scoped_cache(path: Path, *, tenant_id: str, actor_id: str, payload: dict):
    path.parent.mkdir(parents=True, exist_ok=True)
    tmp = path.with_suffix(path.suffix + ".tmp")
    data = {
        "meta": {
            "tenant_id": tenant_id,
            "actor_id": actor_id,
            "created_at": time.time(),
        },
        "payload": payload,
    }
    tmp.write_text(json.dumps(data, ensure_ascii=False, indent=2))
    os.replace(tmp, path)

def read_scoped_cache(path: Path, *, tenant_id: str, actor_id: str, max_age: int):
    if not path.exists():
        return None
    data = json.loads(path.read_text())
    meta = data.get("meta", {})
    if meta.get("tenant_id") != tenant_id or meta.get("actor_id") != actor_id:
        return None
    if time.time() - meta.get("created_at", 0) > max_age:
        return None
    return data.get("payload")
```

---

## 6. 安全缓存决策表

| 数据类型 | 是否可缓存 | 是否可跨 actor 共享 | 建议 TTL |
|---|---:|---:|---:|
| 公开文档/产品目录 | 可以 | 可以 | 10-60 分钟 |
| 租户配置 | 可以 | 同租户同 scope | 1-10 分钟 |
| 用户订单/工单 | 可以 | 不建议 | 30-120 秒 |
| 私有搜索结果 | 可以 | 不可以 | 30-300 秒 |
| 支付/部署/删除前置检查 | 不建议 | 不可以 | 直接 live check |
| Secret / token 原文 | 不缓存 | 不可以 | 0 |

---

## 7. 常见坑

### 坑 1：只在写 key 时带 tenant，读 value 时不校验

Key 写错、迁移脚本 bug、手工复制缓存文件，都可能造成串租户。命中后必须校验 envelope。

### 坑 2：scope 变化后继续用旧缓存

用户权限被撤销后，`actor_id` 没变，但 `scopes` 变了。key 和 envelope 都必须包含 `scope_hash`。

### 坑 3：把 admin 结果缓存成共享结果

工具默认应该 `shareAcrossActor=false`，只有明确证明安全时才打开共享。

### 坑 4：把缓存结果写进长期记忆

缓存可以过期，但 MEMORY.md / 向量库里的内容可能不会。敏感缓存结果进入长期记忆前要重新做 PII masking 和权限检查。

---

## 8. 总结

Agent 缓存的第一目标不是快，而是：**不能把不该看的数据给错人。**

生产上记住这四件事：

1. key 带 `tenant_id + actor_id + scope_hash`；
2. value 带 envelope，cache hit 后二次校验；
3. 敏感工具默认不跨 actor 共享；
4. 高风险动作前不要信缓存，重新 live check。

缓存做对了，Agent 会更快；权限边界做对了，Agent 才敢上线。
