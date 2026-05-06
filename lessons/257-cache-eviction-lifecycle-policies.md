# 257. Agent 缓存淘汰与生命周期策略（Cache Eviction & Lifecycle Policies）

前几课我们把缓存的安全、可观测性、一致性都讲了一遍。今天补一个容易被忽略、但线上迟早会遇到的问题：**缓存不能只会写入，还要知道什么时候该主动消失**。

一句话：

> Agent 缓存要给不同类型的数据定义生命周期：TTL 只是底线，真正生产可用的是 size limit、priority、risk class、access pattern 和 explicit purge 的组合。

很多 Agent 缓存事故不是因为没缓存，而是因为：

- 低价值结果占满空间，高价值上下文被挤掉；
- 敏感数据 TTL 太长，跨任务残留；
- 错误结果被缓存太久，后续 run 一直复用；
- 没有 purge API，用户说“忘掉”但缓存还记得。

---

## 1. 为什么 Agent 缓存比普通后端更需要 lifecycle

普通后端缓存通常围绕接口：

```text
GET /products -> cache 5 minutes
```

Agent 缓存更复杂，因为同一个工具结果可能被用于：

1. 当前回答；
2. 后续任务；
3. 子 Agent 交接；
4. 记忆候选；
5. 审计复盘。

所以缓存条目不应该只有 `key/value/ttl`，至少要有：

```json
{
  "key": "tool:gmail:search:...",
  "class": "tool_result",
  "risk": "sensitive",
  "priority": 80,
  "size_bytes": 48122,
  "created_at": 1778116200,
  "last_accessed_at": 1778116500,
  "expires_at": 1778119800,
  "tags": ["actor:67431246", "tool:gmail", "pii"]
}
```

这让系统能做出更聪明的判断：

- 空间不足时，先删低 priority、低命中、可重建的数据；
- 敏感数据到期必须硬删除；
- 错误/降级结果短 TTL；
- 用户要求删除时按 tag 精准 purge。

---

## 2. 四类常见缓存生命周期

### A. Ephemeral：只服务当前 run

适合：工具中间结果、一次性推理草稿、搜索结果片段。

建议：

```text
TTL: 5-30 minutes
Eviction: run 结束后可清理
Purge: 按 runId 删除
```

### B. Session：服务当前会话

适合：会话状态、用户刚确认过的偏好、当前任务 checkpoint。

建议：

```text
TTL: hours-days
Eviction: LRU + priority
Purge: 用户 stop / reset / logout 时删除
```

### C. Shared：可跨 run 复用

适合：公开文档、工具 schema、依赖清单、非敏感构建结果。

建议：

```text
TTL: days-weeks
Eviction: LFU/LRU 混合
Purge: dependency tag invalidation
```

### D. Sensitive：带权限或隐私边界

适合：邮件、订单、客户资料、私有 repo、账单数据。

建议：

```text
TTL: short
Eviction: risk-first，过期硬删
Purge: actor/tenant/scope tag 强制支持
```

生产规则：**Sensitive cache 不允许 silent extension**。命中不会自动续命，必须重新授权或重新查询。

---

## 3. learn-claude-code：Python 教学版

下面是一个最小可跑的生命周期缓存：写入时带 policy；读取会更新访问时间；空间超限时按 risk/priority/last_accessed 淘汰。

```python
# learn_claude_code/lifecycle_cache.py
from __future__ import annotations

import time
from dataclasses import dataclass, field
from enum import Enum
from typing import Any

class Risk(str, Enum):
    PUBLIC = "public"
    INTERNAL = "internal"
    SENSITIVE = "sensitive"

@dataclass(frozen=True)
class CachePolicy:
    ttl_seconds: int
    priority: int = 50          # 0-100, 越高越重要
    risk: Risk = Risk.INTERNAL
    tags: tuple[str, ...] = ()
    max_stale_seconds: int = 0  # sensitive 通常必须是 0

@dataclass
class CacheEntry:
    key: str
    value: Any
    policy: CachePolicy
    size_bytes: int
    created_at: float
    last_accessed_at: float
    expires_at: float

    def expired(self, now: float) -> bool:
        return now >= self.expires_at

class LifecycleCache:
    def __init__(self, max_bytes: int):
        self.max_bytes = max_bytes
        self.entries: dict[str, CacheEntry] = {}
        self.used_bytes = 0

    def get(self, key: str) -> Any | None:
        now = time.time()
        entry = self.entries.get(key)
        if not entry:
            return None

        if entry.expired(now):
            self.delete(key)
            return None

        entry.last_accessed_at = now
        return entry.value

    def set(self, key: str, value: Any, size_bytes: int, policy: CachePolicy) -> None:
        now = time.time()
        if policy.risk == Risk.SENSITIVE and policy.max_stale_seconds > 0:
            raise ValueError("sensitive cache cannot be served stale")

        if key in self.entries:
            self.delete(key)

        self.entries[key] = CacheEntry(
            key=key,
            value=value,
            policy=policy,
            size_bytes=size_bytes,
            created_at=now,
            last_accessed_at=now,
            expires_at=now + policy.ttl_seconds,
        )
        self.used_bytes += size_bytes
        self.evict_if_needed()

    def purge_by_tag(self, tag: str) -> int:
        keys = [k for k, e in self.entries.items() if tag in e.policy.tags]
        for key in keys:
            self.delete(key)
        return len(keys)

    def delete(self, key: str) -> None:
        entry = self.entries.pop(key, None)
        if entry:
            self.used_bytes -= entry.size_bytes

    def evict_if_needed(self) -> None:
        now = time.time()

        # 1. 过期条目先删
        for key in list(self.entries):
            if self.entries[key].expired(now):
                self.delete(key)

        if self.used_bytes <= self.max_bytes:
            return

        def eviction_score(entry: CacheEntry) -> tuple[int, int, float]:
            # 越小越先淘汰：public/internal 优先删，sensitive 过期已经删；低 priority；更久没访问
            risk_order = {
                Risk.PUBLIC: 0,
                Risk.INTERNAL: 1,
                Risk.SENSITIVE: 2,
            }[entry.policy.risk]
            return (risk_order, entry.policy.priority, entry.last_accessed_at)

        victims = sorted(self.entries.values(), key=eviction_score)
        for entry in victims:
            if self.used_bytes <= self.max_bytes:
                break
            self.delete(entry.key)
```

关键点：

- `ttl_seconds` 管时间；
- `priority` 管价值；
- `risk` 管安全策略；
- `tags` 支持用户/租户/依赖级 purge；
- `evict_if_needed()` 不只是 LRU，而是带策略的淘汰。

---

## 4. pi-mono：生产版中间件写法

在 pi-mono 里建议把 lifecycle 放进 cache middleware，而不是散落在每个工具调用里。

```ts
// packages/agent-runtime/src/cache/lifecycle-policy.ts
export type CacheRisk = 'public' | 'internal' | 'sensitive';
export type CacheClass = 'tool_result' | 'session_state' | 'schema' | 'memory_candidate';

export interface CacheLifecyclePolicy {
  class: CacheClass;
  ttlMs: number;
  priority: number; // 0-100
  risk: CacheRisk;
  tags: string[];
  allowStale: boolean;
  purgeOnRunEnd?: boolean;
}

export function policyForToolResult(input: {
  toolName: string;
  actorId: string;
  tenantId?: string;
  containsPii?: boolean;
  retryable?: boolean;
}): CacheLifecyclePolicy {
  const baseTags = [
    `actor:${input.actorId}`,
    `tool:${input.toolName}`,
    input.tenantId ? `tenant:${input.tenantId}` : 'tenant:none',
  ];

  if (input.containsPii) {
    return {
      class: 'tool_result',
      ttlMs: 10 * 60_000,
      priority: 70,
      risk: 'sensitive',
      tags: [...baseTags, 'pii'],
      allowStale: false,
      purgeOnRunEnd: false,
    };
  }

  if (input.retryable === false) {
    return {
      class: 'tool_result',
      ttlMs: 60_000,
      priority: 20,
      risk: 'internal',
      tags: [...baseTags, 'negative-cache'],
      allowStale: false,
      purgeOnRunEnd: true,
    };
  }

  return {
    class: 'tool_result',
    ttlMs: 30 * 60_000,
    priority: 50,
    risk: 'internal',
    tags: baseTags,
    allowStale: true,
  };
}
```

Dispatcher 调用时统一写 envelope：

```ts
// packages/agent-runtime/src/cache/lifecycle-cache.ts
interface CacheEnvelope<T> {
  value: T;
  policy: CacheLifecyclePolicy;
  createdAt: number;
  lastAccessedAt: number;
  expiresAt: number;
  sizeBytes: number;
}

export class LifecycleCache {
  constructor(
    private store: Map<string, CacheEnvelope<unknown>>,
    private maxBytes: number,
  ) {}

  async get<T>(key: string): Promise<T | undefined> {
    const now = Date.now();
    const entry = this.store.get(key) as CacheEnvelope<T> | undefined;
    if (!entry) return undefined;

    const expired = now >= entry.expiresAt;
    if (expired && !entry.policy.allowStale) {
      this.store.delete(key);
      return undefined;
    }

    if (entry.policy.risk === 'sensitive' && expired) {
      this.store.delete(key);
      return undefined;
    }

    entry.lastAccessedAt = now;
    return entry.value;
  }

  async set<T>(key: string, value: T, policy: CacheLifecyclePolicy): Promise<void> {
    if (policy.risk === 'sensitive' && policy.allowStale) {
      throw new Error('sensitive cache cannot allow stale reads');
    }

    const sizeBytes = Buffer.byteLength(JSON.stringify(value), 'utf8');
    this.store.set(key, {
      value,
      policy,
      createdAt: Date.now(),
      lastAccessedAt: Date.now(),
      expiresAt: Date.now() + policy.ttlMs,
      sizeBytes,
    });

    await this.evict();
  }

  async purgeByTag(tag: string): Promise<number> {
    let count = 0;
    for (const [key, entry] of this.store.entries()) {
      if (entry.policy.tags.includes(tag)) {
        this.store.delete(key);
        count++;
      }
    }
    return count;
  }

  private async evict(): Promise<void> {
    const entries = [...this.store.entries()];
    const used = entries.reduce((sum, [, e]) => sum + e.sizeBytes, 0);
    if (used <= this.maxBytes) return;

    const riskWeight = { public: 0, internal: 1, sensitive: 2 } as const;
    const victims = entries.sort(([, a], [, b]) => {
      return (
        riskWeight[a.policy.risk] - riskWeight[b.policy.risk] ||
        a.policy.priority - b.policy.priority ||
        a.lastAccessedAt - b.lastAccessedAt
      );
    });

    let current = used;
    for (const [key, entry] of victims) {
      if (current <= this.maxBytes) break;
      this.store.delete(key);
      current -= entry.sizeBytes;
    }
  }
}
```

注意：生产版还可以把 eviction metrics 发到 observability：

```text
cache_evictions_total{class,risk,reason}
cache_bytes{class,risk}
cache_purge_total{tag_type}
```

没有这些指标，缓存空间问题只会在“突然变慢/突然 OOM”时才被发现。

---

## 5. OpenClaw 实战：给课程 cron 做 lifecycle

以这个课程 cron 为例，可以把缓存分成几类：

```text
lessons index / README summary      -> shared, TTL 24h, tag repo:agent-course
TOOLS.md 已讲内容摘要               -> session/internal, TTL 3h, tag file:TOOLS
Telegram message receipt            -> sensitive-ish audit, TTL 7d, tag channel:-5115329245
git status / gh pr list live result -> ephemeral, TTL 2m, purgeOnRunEnd
```

一个简单的文件式策略：

```json
{
  "key": "agent-course:readme-summary",
  "class": "shared",
  "risk": "internal",
  "priority": 60,
  "ttlSeconds": 86400,
  "tags": ["repo:agent-course", "file:README.md"]
}
```

每次写课程文件后，要主动 purge：

```bash
# 伪代码：课程仓库发生变化，清掉依赖它的缓存
openclaw-cache purge --tag repo:agent-course
openclaw-cache purge --tag file:README.md
```

如果没有专门 cache CLI，也可以在 Agent 自己的状态目录里做文件级清理：

```python
from pathlib import Path
import json

cache_dir = Path.home() / ".openclaw" / "workspace" / "memory" / "cache"

for path in cache_dir.glob("*.json"):
    envelope = json.loads(path.read_text())
    if "repo:agent-course" in envelope.get("tags", []):
        path.unlink()
```

核心不是具体命令，而是这条原则：**写操作完成后，主动清掉依赖旧状态的缓存**。

---

## 6. 常见坑

### 坑 1：只靠 TTL，不做空间淘汰

TTL 只能保证“过一段时间删”，不能保证“空间不爆”。生产缓存必须有 max size。

### 坑 2：LRU 一把梭

LRU 会把最近访问但低价值的数据留下来，把很重要但暂时没访问的数据删掉。Agent 场景最好加 `priority`。

### 坑 3：错误结果缓存太久

Negative cache 很有用，但 TTL 要短。比如 404/权限失败/网络降级结果，不应该缓存几个小时。

### 坑 4：用户要求删除，只删 memory 不删 cache

“忘掉这个”必须同时处理：

- 长期记忆；
- 会话状态；
- 工具结果缓存；
- 向量索引；
- outbox/inbox 中可删除的非审计字段。

---

## 7. 一套可落地的默认值

可以从这张表开始：

| 类型 | TTL | Priority | Risk | Stale | Purge |
|---|---:|---:|---|---|---|
| 工具 schema | 7d | 80 | public/internal | yes | schema version |
| Git status / PR 状态 | 2m | 40 | internal | no | run end |
| 公开文档片段 | 24h | 60 | public | yes | URL/tag |
| 邮件/订单/客户资料 | 10m | 70 | sensitive | no | actor/tenant |
| 错误结果 negative cache | 30-60s | 20 | internal | no | run end |
| 记忆候选 | 1h | 50 | sensitive/internal | no | actor/session |

---

## 8. 总结

Agent 缓存生命周期可以用五个字段管住：

```text
class + risk + ttl + priority + tags
```

- `class` 决定它是什么；
- `risk` 决定安全边界；
- `ttl` 决定最长能活多久；
- `priority` 决定空间紧张时谁先走；
- `tags` 决定谁能精准清除它。

缓存的目标不是“永远命中”，而是：

> 该快的时候快，该忘的时候真忘，该删的时候删得准。

这就是 Agent 缓存从 demo 走向生产必须补上的生命周期管理。
