# 265. Agent 缓存删除墓碑与失效传播（Cache Tombstones & Invalidation Propagation）

> 缓存最危险的 bug 不是 miss，而是“已经删除的数据又被旧缓存复活”。在多级缓存、多实例 Agent、异步刷新并存的系统里，delete 不能只是 `del(key)`，还需要 tombstone（删除墓碑）和失效传播。

前面我们讲了缓存 key、标签失效、一致性、多级缓存、迁移灰度。今天补上一个生产里很容易踩坑的点：**删除语义怎么在缓存系统里可靠传播**。

## 1. 为什么 `cache.delete(key)` 不够？

典型事故：

1. 用户撤销了某个权限，DB 已更新。
2. L2 Redis 删除了 `user:permission` 缓存。
3. 某个 worker 的 L1 内存里还有旧值。
4. 另一个后台 refresh-ahead 任务还在跑，用旧观察重新写回 Redis。
5. Agent 再次读到旧权限，执行了本不该执行的工具。

这类问题叫 **resurrection（缓存复活）**。

删除在分布式系统里不是一个瞬间动作，而是一个需要传播、需要版本、需要时间窗口的状态。

## 2. Tombstone 的核心设计

Tombstone 不是“不存在”，而是一个明确的缓存条目：

```json
{
  "type": "tombstone",
  "key": "tenant:t1:user:u1:permissions",
  "deletedAt": "2026-05-08T06:30:00+11:00",
  "version": 42,
  "reason": "permission_revoked",
  "expiresAt": "2026-05-08T06:35:00+11:00"
}
```

它解决三个问题：

- **阻止旧写回**：refresh 任务如果版本低于 tombstone，禁止 set。
- **传播删除事实**：L1/L2/L3 都能看见“这个 key 刚被删除过”。
- **短期保护窗口**：给 Pub/Sub、异步 worker、跨进程同步一点时间，不让旧值立刻复活。

经验规则：

- 用户权限、账号状态、支付状态：tombstone TTL 5-30 分钟。
- 普通搜索结果、文档摘要：tombstone TTL 30-120 秒。
- 高风险副作用前：即使没 tombstone，也必须 live check。

## 3. learn-claude-code：Python 教学版

```python
# learn-claude-code/cache_tombstone.py
from __future__ import annotations

from dataclasses import dataclass
from time import time
from typing import Any


@dataclass
class Entry:
    value: Any
    version: int
    expires_at: float
    tombstone: bool = False
    reason: str | None = None


class TombstoneCache:
    def __init__(self):
        self.store: dict[str, Entry] = {}

    def get(self, key: str) -> Any | None:
        entry = self.store.get(key)
        if entry is None or entry.expires_at < time():
            return None
        if entry.tombstone:
            # 明确告诉上层：不是 miss，是刚删除过，不能回源旧快照直接写回。
            raise KeyError(f"cache tombstone: {key} reason={entry.reason}")
        return entry.value

    def set(self, key: str, value: Any, *, version: int, ttl_seconds: int) -> bool:
        current = self.store.get(key)
        if current and current.tombstone and current.version >= version:
            # 旧 refresh / 旧 worker 不允许复活数据。
            print("reject_stale_cache_write", {"key": key, "incoming": version, "tombstone": current.version})
            return False

        self.store[key] = Entry(
            value=value,
            version=version,
            expires_at=time() + ttl_seconds,
        )
        return True

    def delete_with_tombstone(self, key: str, *, version: int, reason: str, ttl_seconds: int = 300) -> None:
        self.store[key] = Entry(
            value=None,
            version=version,
            expires_at=time() + ttl_seconds,
            tombstone=True,
            reason=reason,
        )
```

关键点：`delete_with_tombstone` 不是真的清空，而是写入一个更高版本的删除事实。

## 4. pi-mono：TypeScript 生产版中间件

生产系统里建议把 tombstone 放进缓存中间件，让所有工具缓存都自动获得保护。

```ts
// pi-mono/packages/agent-runtime/cache/tombstone-cache.ts
export type CacheEnvelope<T> =
  | { kind: "value"; value: T; version: number; expiresAt: number }
  | { kind: "tombstone"; version: number; deletedAt: number; expiresAt: number; reason: string };

export interface CacheStore {
  get<T>(key: string): Promise<CacheEnvelope<T> | null>;
  set<T>(key: string, value: CacheEnvelope<T>): Promise<void>;
  delete(key: string): Promise<void>;
}

export interface InvalidationBus {
  publish(event: { key: string; version: number; reason: string; deletedAt: number }): Promise<void>;
  subscribe(handler: (event: { key: string; version: number; reason: string; deletedAt: number }) => void): void;
}

export class TombstoneCache {
  constructor(
    private readonly l1: Map<string, CacheEnvelope<unknown>>,
    private readonly l2: CacheStore,
    private readonly bus: InvalidationBus,
  ) {
    this.bus.subscribe((event) => {
      const current = this.l1.get(event.key);
      if (!current || current.version <= event.version) {
        this.l1.set(event.key, {
          kind: "tombstone",
          version: event.version,
          deletedAt: event.deletedAt,
          expiresAt: Date.now() + 5 * 60_000,
          reason: event.reason,
        });
      }
    });
  }

  async get<T>(key: string): Promise<T | null> {
    const l1 = this.l1.get(key) as CacheEnvelope<T> | undefined;
    if (l1 && l1.expiresAt > Date.now()) return this.unwrap(key, l1);

    const l2 = await this.l2.get<T>(key);
    if (!l2 || l2.expiresAt <= Date.now()) return null;

    this.l1.set(key, l2);
    return this.unwrap(key, l2);
  }

  async set<T>(key: string, value: T, opts: { version: number; ttlMs: number }): Promise<boolean> {
    const current = (this.l1.get(key) ?? await this.l2.get<T>(key)) as CacheEnvelope<T> | null;

    if (current?.kind === "tombstone" && current.version >= opts.version) {
      metrics.increment("cache.write_rejected_by_tombstone", { key });
      return false;
    }

    const envelope: CacheEnvelope<T> = {
      kind: "value",
      value,
      version: opts.version,
      expiresAt: Date.now() + opts.ttlMs,
    };

    this.l1.set(key, envelope);
    await this.l2.set(key, envelope);
    return true;
  }

  async deleteWithTombstone(key: string, opts: { version: number; reason: string; ttlMs: number }): Promise<void> {
    const tombstone: CacheEnvelope<never> = {
      kind: "tombstone",
      version: opts.version,
      deletedAt: Date.now(),
      expiresAt: Date.now() + opts.ttlMs,
      reason: opts.reason,
    };

    this.l1.set(key, tombstone);
    await this.l2.set(key, tombstone);
    await this.bus.publish({ key, version: opts.version, reason: opts.reason, deletedAt: tombstone.deletedAt });
  }

  private unwrap<T>(key: string, envelope: CacheEnvelope<T>): T | null {
    if (envelope.kind === "tombstone") {
      metrics.increment("cache.tombstone_hit", { key, reason: envelope.reason });
      return null;
    }
    return envelope.value;
  }
}
```

这段代码的核心不是 Map/Redis，而是三个动作：

1. 删除时写 tombstone。
2. 写入时比较 version，拒绝旧写。
3. 通过 bus 把删除事实传播给其他 worker 的 L1。

## 5. OpenClaw 实战：文件缓存也要墓碑

OpenClaw 这类 Always-on Agent 经常有文件式状态：

```text
memory/cache/tool-results/<key>.json
memory/cache/tombstones/<key>.json
```

删除高风险缓存时，不要只 `trash cache.json`，而是：

```json
{
  "schemaVersion": "cache-tombstone-v1",
  "key": "telegram:-5115329245:lesson-index",
  "version": 265,
  "reason": "lesson_catalog_updated",
  "deletedAt": "2026-05-08T06:30:00+11:00",
  "expiresAt": "2026-05-08T06:35:00+11:00"
}
```

Agent 读缓存前先查 tombstone：

```python
if tombstone_exists(key) and tombstone.version >= cached.version:
    bypass_cache_and_refresh()
```

课程 cron 的例子：更新 README 目录后，如果有“课程索引摘要缓存”，就写 tombstone，避免下一次课程生成还读到旧目录，导致重复选题。

## 6. 常见坑

### 坑一：tombstone TTL 太短

Pub/Sub 延迟、worker 卡顿、后台 refresh 还没结束，墓碑就过期了，旧值仍可能复活。高风险数据宁可 TTL 长一点。

### 坑二：没有版本号

只有 deletedAt 不够。跨机器时钟可能漂移，建议用数据库 revision、event sequence、monotonic version。

### 坑三：把 tombstone 当普通 miss

普通 miss 可以回源；tombstone hit 要谨慎：它代表“刚删除过”。如果回源也必须回权威源，并且写回版本必须更新。

### 坑四：只删 L2，不管 L1

多实例 Agent 最常见：Redis 已删，进程内 Map 还在。必须有失效事件传播，或者每个 L1 value 都带极短 TTL。

## 7. 一句话总结

`delete(key)` 只是本地动作，`tombstone(version)` 才是分布式删除语义。

成熟 Agent 的缓存系统不只会记住，还要会**可靠地忘记，并阻止旧记忆复活**。
