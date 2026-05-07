# 263. Agent 缓存键设计与版本命名空间（Cache Key Design & Versioned Namespaces）

> 缓存最难的不是把值存进去，而是保证“同一个 key 永远代表同一类事实”。Key 设计错了，命中率越高，事故越稳定。

前几课我们讲了缓存失效、击穿、预热、可观测性、一致性、权限边界、生命周期、多级缓存、回源预算、准入策略。今天补上一个基础但经常被低估的问题：**缓存 key 怎么设计**。

## 1. 为什么 Agent 的缓存 key 更难？

普通后端缓存常见 key：

```text
user:123
product:sku_abc
order:list:page:1
```

但 Agent 系统的缓存还受这些因素影响：

- **actor**：谁在问？老板、群聊用户、子 Agent 权限不同。
- **tenant**：哪个租户/项目？多租户不能串数据。
- **purpose**：用途是什么？回答问题、计划执行、副作用前校验，风险不同。
- **tool schema version**：工具返回结构变了，旧缓存可能不能读。
- **prompt / policy version**：同样输入，在不同策略下可能应该得到不同结果。
- **source freshness**：数据源版本或更新时间不同，旧 key 不能假装是新事实。

所以 Agent 缓存 key 不是一个字符串拼接题，而是一个**上下文契约设计题**。

## 2. 好 key 的五个原则

### 原则一：key 必须可解释

坏例子：

```text
cache:4f2a9c1b
```

你看到它不知道属于谁、什么工具、什么版本、能不能删。

好例子：

```text
agent:v3:tenant:metadraw:actor:67431246:tool:gmail.search:schema:2:purpose:answer:hash:8f92a1
```

可读前缀 + hash 尾巴，既方便排查，也避免 key 太长。

### 原则二：安全边界必须进入 key

不要让不同 actor/tenant/scope 共用同一个缓存 key：

```text
# 错：只按 query 缓存
search:unpaid invoices

# 对：把权限边界放进去
tenant:metadraw:actor:67431246:scope:billing-readonly:search:unpaid-invoices
```

如果 key 缺少权限维度，缓存层就会绕过授权系统。

### 原则三：语义版本进入 key，而不是靠“希望兼容”

工具返回结构变了，缓存 key 必须换版本：

```text
tool:gmail.search:v1:...
tool:gmail.search:v2:...
```

不要让 v2 代码读 v1 数据后靠 try/catch 硬扛。版本进入 key，是最便宜的迁移策略。

### 原则四：高熵参数 hash，低熵维度明文

比如完整 query、复杂 filters 可以 hash：

```text
qhash:sha256(canonical_json({query, filters, sort}))
```

但 tenant、actor、tool、version 要明文，方便观测和批量失效。

### 原则五：canonicalize 后再 hash

这两个请求语义相同：

```json
{"limit": 20, "query": "foo"}
{"query": "foo", "limit": 20}
```

如果直接 JSON.stringify，可能得到不同 hash。必须先做 canonical JSON：排序字段、规范时间、统一大小写、移除默认值。

## 3. learn-claude-code：Python 教学版 Key Builder

```python
# learn-claude-code/cache_key.py
from __future__ import annotations

import hashlib
import json
from dataclasses import dataclass
from typing import Any, Mapping


def canonical_json(value: Any) -> str:
    """稳定序列化：字段排序、紧凑输出，保证语义相同 hash 相同。"""
    return json.dumps(value, sort_keys=True, separators=(",", ":"), ensure_ascii=False)


def short_hash(value: Any, length: int = 12) -> str:
    digest = hashlib.sha256(canonical_json(value).encode("utf-8")).hexdigest()
    return digest[:length]


@dataclass(frozen=True)
class CacheContext:
    tenant_id: str
    actor_id: str
    scope_hash: str
    purpose: str          # answer | planning | pre_action | eval
    policy_version: str   # policy-v3


@dataclass(frozen=True)
class ToolCacheKey:
    namespace: str        # agent-cache
    app_version: str      # v3
    tool_name: str        # gmail.search
    schema_version: str   # schema-v2
    context: CacheContext
    args: Mapping[str, Any]

    def build(self) -> str:
        args_hash = short_hash(self.args)
        parts = [
            self.namespace,
            self.app_version,
            f"tenant:{self.context.tenant_id}",
            f"actor:{self.context.actor_id}",
            f"scope:{self.context.scope_hash}",
            f"purpose:{self.context.purpose}",
            f"policy:{self.context.policy_version}",
            f"tool:{self.tool_name}",
            f"schema:{self.schema_version}",
            f"args:{args_hash}",
        ]
        return ":".join(parts)


ctx = CacheContext(
    tenant_id="metadraw",
    actor_id="67431246",
    scope_hash="billing-readonly",
    purpose="answer",
    policy_version="policy-v3",
)

key = ToolCacheKey(
    namespace="agent-cache",
    app_version="v3",
    tool_name="aws.cost_explorer.query",
    schema_version="schema-v1",
    context=ctx,
    args={"month": "2026-05", "groupBy": ["LINKED_ACCOUNT"]},
).build()

print(key)
```

重点：`args` 被 hash，权限与版本维度明文。

## 4. pi-mono：TypeScript 生产版

生产环境建议不要到处手写字符串，而是统一 `CacheKeyBuilder`。

```ts
// pi-mono/packages/agent-runtime/cache/cache-key.ts
import crypto from "node:crypto";

export type CachePurpose = "answer" | "planning" | "pre_action" | "eval";

export interface CacheKeyContext {
  tenantId: string;
  actorId: string;
  scopeHash: string;
  purpose: CachePurpose;
  policyVersion: string;
  appVersion: string;
}

export interface ToolKeyInput {
  toolName: string;
  schemaVersion: string;
  args: unknown;
  sourceVersion?: string;
}

function canonicalize(value: unknown): string {
  return JSON.stringify(sortDeep(value));
}

function sortDeep(value: any): any {
  if (Array.isArray(value)) return value.map(sortDeep);
  if (value && typeof value === "object") {
    return Object.fromEntries(
      Object.entries(value)
        .filter(([, v]) => v !== undefined)
        .sort(([a], [b]) => a.localeCompare(b))
        .map(([k, v]) => [k, sortDeep(v)]),
    );
  }
  return value;
}

function sha12(value: unknown): string {
  return crypto.createHash("sha256").update(canonicalize(value)).digest("hex").slice(0, 12);
}

export class CacheKeyBuilder {
  constructor(private readonly ctx: CacheKeyContext) {}

  tool(input: ToolKeyInput): string {
    const parts = [
      "agent-cache",
      this.ctx.appVersion,
      `tenant:${this.ctx.tenantId}`,
      `actor:${this.ctx.actorId}`,
      `scope:${this.ctx.scopeHash}`,
      `purpose:${this.ctx.purpose}`,
      `policy:${this.ctx.policyVersion}`,
      `tool:${input.toolName}`,
      `schema:${input.schemaVersion}`,
    ];

    if (input.sourceVersion) {
      parts.push(`source:${input.sourceVersion}`);
    }

    parts.push(`args:${sha12(input.args)}`);
    return parts.join(":");
  }

  tag(tagName: string, id: string): string {
    return `tag:${this.ctx.tenantId}:${tagName}:${id}`;
  }
}
```

工具中间件使用：

```ts
// pi-mono/packages/agent-runtime/middleware/cache.ts
export async function cachedToolCall(ctx: AgentContext, call: ToolCall) {
  const keyBuilder = new CacheKeyBuilder({
    tenantId: ctx.tenantId,
    actorId: ctx.actorId,
    scopeHash: ctx.scopeHash,
    purpose: ctx.purpose,
    policyVersion: ctx.policyVersion,
    appVersion: ctx.appVersion,
  });

  const cacheKey = keyBuilder.tool({
    toolName: call.tool.name,
    schemaVersion: call.tool.schemaVersion,
    sourceVersion: call.tool.sourceVersion,
    args: call.args,
  });

  const cached = await ctx.cache.get(cacheKey);
  if (cached) return cached;

  const result = await call.execute();
  await ctx.cache.set(cacheKey, result, { ttlMs: call.tool.cacheTtlMs });
  return result;
}
```

这让 key 设计成为运行时基础设施，而不是每个工具各写一套。

## 5. OpenClaw 实战：文件缓存也要命名空间

OpenClaw 里很多轻量缓存会落在 workspace 文件中，比如 heartbeat 状态、课程检查、外部 API 快照。不要这样写：

```text
memory/cache/result.json
```

建议按 namespace 分层：

```text
memory/cache/
  agent-course/
    v1/
      telegram-group--5115329245/
        lesson-index.json
  aws-billing/
    v2/
      tenant-metadraw/
        actor-67431246/
          cost-explorer-2026-05.json
```

文件路径本身就是 key。路径里应该体现：业务、版本、租户、actor、用途。这样清理、迁移、排查都简单。

课程 cron 的一个轻量模式：

```python
from pathlib import Path
import json

CACHE_ROOT = Path("memory/cache/agent-course/v1")

def lesson_index_cache(group_id: int) -> Path:
    safe_group = str(group_id).replace("-", "minus")
    return CACHE_ROOT / f"telegram-group-{safe_group}" / "lesson-index.json"

path = lesson_index_cache(-5115329245)
path.parent.mkdir(parents=True, exist_ok=True)
path.write_text(json.dumps({"latestLesson": 263}, ensure_ascii=False, indent=2))
```

## 6. Key 设计 Checklist

每加一个缓存，先问 8 个问题：

1. 这个缓存属于哪个 namespace？
2. 是否包含 app/tool/schema/policy version？
3. 是否包含 tenant/actor/scope 等安全边界？
4. args 是否 canonicalize 后再 hash？
5. 低熵维度是否明文，方便观测？
6. 是否有 tag 支持批量失效？
7. 是否能从 key 判断大概用途？
8. 旧版本 key 如何自然过期或迁移？

## 7. 常见坑

### 坑一：只 hash 不留明文维度

```text
cache:a8f9c2e1
```

排查时完全不知道它是什么，只能全量删。

### 坑二：把用户输入原文直接拼进 key

```text
search:老板给我查一下 2026 年 5 月 AWS 账单...
```

问题：key 过长、包含隐私、大小写/空格导致重复。应该 canonicalize + hash。

### 坑三：忘记 schema version

工具返回从：

```json
{"total": 123}
```

升级到：

```json
{"amount": 123, "currency": "USD"}
```

如果 key 没版本，旧缓存会让新代码读到错误结构。

### 坑四：把 purpose 混在一起

同一个查询用于普通回答可以读 stale 缓存；用于发钱、删库、部署前检查就必须 live check。`purpose` 不进 key，策略就会混乱。

## 8. 一句话总结

**缓存 key 是 Agent 对“事实身份”的定义：谁、在什么权限下、用哪个版本、为了什么目的、基于哪些参数得到这个结果。**

Key 设计清楚，缓存才是加速器；Key 设计模糊，缓存就是稳定制造事故的机器。
