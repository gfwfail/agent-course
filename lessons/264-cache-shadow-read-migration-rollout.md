# 264. Agent 缓存影子读与迁移灰度（Cache Shadow Read & Migration Rollout）

> 缓存升级最怕“一刀切”：新 key、新序列化、新策略上线后，发现命中率掉光、数据不一致、权限边界漏了。成熟做法不是直接替换，而是先影子读、比对、灰度，再切流。

前面我们已经讲了缓存 key、权限边界、一致性、准入、生命周期。今天讲一个生产落地很关键的动作：**缓存迁移怎么安全上线**。

## 1. 为什么缓存迁移不能直接切？

常见迁移场景：

- key 从 `tool:argsHash` 升级到包含 `tenant/actor/scope/purpose` 的安全 key。
- value 从裸 JSON 升级到 `envelope + meta + provenance`。
- TTL 从统一 1 小时改成按风险分级。
- L2 从 SQLite 换成 Redis，或者 Redis namespace 变更。

直接切换会有三个风险：

1. **命中率悬崖**：新 namespace 全空，所有请求同时回源。
2. **语义不一致**：旧缓存和新缓存对同一事实返回不同答案。
3. **安全回归**：新 key 少了 actor/scope，反而把数据串租户。

所以迁移要像发布代码一样：先观测，再灰度，最后切换。

## 2. 三阶段迁移模型

### 阶段一：dual write（双写）

读取仍走旧缓存，但写入时同时写旧格式和新格式。

目的：让新缓存提前变热，不要切换当天从 0 开始。

```text
read:  old_cache
write: old_cache + new_cache
```

### 阶段二：shadow read（影子读）

主路径仍返回旧缓存结果，但后台同时读新缓存，只做比对和打指标，不影响用户。

```text
result = old_cache.get(key_v1)
shadow = new_cache.get(key_v2)
compare(result, shadow)  # mismatch 只记录，不返回给用户
return result
```

### 阶段三：progressive rollout（灰度切流）

按 tenant、actor、工具、百分比逐步让主路径读新缓存。出问题时快速回滚到旧缓存。

```text
1% -> 5% -> 25% -> 50% -> 100%
```

关键指标：

- `shadow_hit_rate`：新缓存是否已经热起来。
- `mismatch_rate`：新旧结果是否一致。
- `origin_qps`：是否因为 miss 打爆源站。
- `p95_latency_ms`：新缓存路径是否更慢。
- `security_mismatch_count`：权限/tenant/scope 是否不一致，必须零容忍。

## 3. learn-claude-code：Python 教学版

```python
# learn-claude-code/cache_migration.py
from __future__ import annotations

from dataclasses import dataclass
from typing import Any, Protocol
import json


class Cache(Protocol):
    def get(self, key: str) -> Any | None: ...
    def set(self, key: str, value: Any, ttl_seconds: int) -> None: ...


@dataclass
class MigrationFlags:
    dual_write: bool = True
    shadow_read: bool = True
    rollout_percent: int = 0  # 0-100


def stable_json(value: Any) -> str:
    return json.dumps(value, sort_keys=True, ensure_ascii=False, separators=(",", ":"))


def semantically_equal(a: Any, b: Any) -> bool:
    """生产里可以忽略 observedAt、traceId 等非语义字段。"""
    if a is None or b is None:
        return a is b
    return stable_json(strip_non_semantic(a)) == stable_json(strip_non_semantic(b))


def strip_non_semantic(value: Any) -> Any:
    if isinstance(value, dict):
        return {
            k: strip_non_semantic(v)
            for k, v in value.items()
            if k not in {"observedAt", "traceId", "cacheHit"}
        }
    if isinstance(value, list):
        return [strip_non_semantic(v) for v in value]
    return value


class CacheMigrator:
    def __init__(self, old_cache: Cache, new_cache: Cache, flags: MigrationFlags):
        self.old_cache = old_cache
        self.new_cache = new_cache
        self.flags = flags

    def should_use_new(self, actor_id: str) -> bool:
        # 稳定灰度：同一个 actor 每次结果一致，避免请求抖动。
        bucket = sum(actor_id.encode("utf-8")) % 100
        return bucket < self.flags.rollout_percent

    def get(self, *, old_key: str, new_key: str, actor_id: str) -> Any | None:
        use_new = self.should_use_new(actor_id)
        primary = self.new_cache if use_new else self.old_cache
        shadow = self.old_cache if use_new else self.new_cache

        value = primary.get(new_key if use_new else old_key)

        if self.flags.shadow_read:
            shadow_value = shadow.get(old_key if use_new else new_key)
            if shadow_value is not None and value is not None:
                if not semantically_equal(value, shadow_value):
                    print("cache_migration_mismatch", {"oldKey": old_key, "newKey": new_key})

        return value

    def set(self, *, old_key: str, new_key: str, value: Any, ttl_seconds: int) -> None:
        if self.flags.dual_write:
            self.old_cache.set(old_key, value, ttl_seconds)
            self.new_cache.set(new_key, wrap_v2(value), ttl_seconds)
        else:
            self.new_cache.set(new_key, wrap_v2(value), ttl_seconds)


def wrap_v2(value: Any) -> dict[str, Any]:
    return {
        "schemaVersion": "cache-value-v2",
        "data": value,
        "meta": {"source": "migration-dual-write"},
    }
```

这个版本的重点不是缓存实现，而是迁移控制面：`dual_write`、`shadow_read`、`rollout_percent`。

## 4. pi-mono：TypeScript 生产版中间件

生产里可以把迁移做成缓存中间件，让业务工具无感。

```ts
// pi-mono/packages/agent-runtime/cache/cache-migration-middleware.ts
export interface CacheStore {
  get<T>(key: string): Promise<T | null>;
  set<T>(key: string, value: T, opts: { ttlMs: number }): Promise<void>;
}

export interface CacheMigrationPolicy {
  name: string;
  dualWrite: boolean;
  shadowRead: boolean;
  rolloutPercent: number;
  failOpenToOld: boolean;
}

export interface CacheKeys {
  oldKey: string;
  newKey: string;
  actorId: string;
  tenantId: string;
  purpose: "answer" | "planning" | "pre_action" | "eval";
}

export interface MigrationMetrics {
  increment(name: string, tags?: Record<string, string>): void;
  observe(name: string, value: number, tags?: Record<string, string>): void;
}

function stableBucket(id: string): number {
  let hash = 0;
  for (const ch of id) hash = (hash * 31 + ch.charCodeAt(0)) >>> 0;
  return hash % 100;
}

function stripNonSemantic(value: any): any {
  if (Array.isArray(value)) return value.map(stripNonSemantic);
  if (value && typeof value === "object") {
    return Object.fromEntries(
      Object.entries(value)
        .filter(([k]) => !["observedAt", "traceId", "cacheHit"].includes(k))
        .map(([k, v]) => [k, stripNonSemantic(v)]),
    );
  }
  return value;
}

function equivalent(a: unknown, b: unknown): boolean {
  return JSON.stringify(stripNonSemantic(a)) === JSON.stringify(stripNonSemantic(b));
}

export class CacheMigrationMiddleware {
  constructor(
    private readonly oldStore: CacheStore,
    private readonly newStore: CacheStore,
    private readonly policy: CacheMigrationPolicy,
    private readonly metrics: MigrationMetrics,
  ) {}

  private useNew(keys: CacheKeys): boolean {
    return stableBucket(`${keys.tenantId}:${keys.actorId}`) < this.policy.rolloutPercent;
  }

  async get<T>(keys: CacheKeys): Promise<T | null> {
    const useNew = this.useNew(keys);
    const primaryStore = useNew ? this.newStore : this.oldStore;
    const primaryKey = useNew ? keys.newKey : keys.oldKey;

    try {
      const primary = await primaryStore.get<T>(primaryKey);
      this.metrics.increment(primary ? "cache.primary_hit" : "cache.primary_miss", {
        migration: this.policy.name,
        path: useNew ? "new" : "old",
      });

      if (this.policy.shadowRead) {
        void this.shadowCompare(keys, primary, useNew);
      }

      return primary;
    } catch (error) {
      this.metrics.increment("cache.primary_error", { migration: this.policy.name });
      if (useNew && this.policy.failOpenToOld) {
        return this.oldStore.get<T>(keys.oldKey);
      }
      throw error;
    }
  }

  async set<T>(keys: CacheKeys, value: T, ttlMs: number): Promise<void> {
    if (!this.policy.dualWrite) {
      await this.newStore.set(keys.newKey, this.wrapV2(value), { ttlMs });
      return;
    }

    await Promise.all([
      this.oldStore.set(keys.oldKey, value, { ttlMs }),
      this.newStore.set(keys.newKey, this.wrapV2(value), { ttlMs }),
    ]);
  }

  private async shadowCompare<T>(keys: CacheKeys, primary: T | null, primaryIsNew: boolean) {
    const shadow = primaryIsNew
      ? await this.oldStore.get<T>(keys.oldKey)
      : await this.newStore.get<T>(keys.newKey);

    if (primary == null || shadow == null) return;

    if (!equivalent(primary, shadow)) {
      this.metrics.increment("cache.shadow_mismatch", {
        migration: this.policy.name,
        tenantId: keys.tenantId,
        purpose: keys.purpose,
      });
    }
  }

  private wrapV2<T>(value: T) {
    return {
      schemaVersion: "cache-value-v2",
      data: value,
      meta: { migratedBy: this.policy.name },
    };
  }
}
```

这里有三个生产细节：

- 灰度 bucket 用 `tenantId:actorId`，确保同一用户稳定命中新路径。
- `shadowCompare` 用 fire-and-forget，避免影子读拖慢主请求。
- 新路径异常时可 `failOpenToOld`，先保可用性，再调查迁移问题。

## 5. OpenClaw 实战：课程缓存迁移怎么做

假设 OpenClaw 的课程 cron 有一个文件缓存：

```text
memory/cache/agent-course/topic-suggestions.json
```

原本只按主题文本缓存，准备升级为：

```text
memory/cache/v2/tenant:owner/actor:67431246/tool:lesson.topic/purpose:planning/args:xxxx.json
```

推荐步骤：

1. **第 1 天**：开启 dual write。旧文件继续读，新文件同时写。
2. **第 2 天**：开启 shadow read。比较旧建议和新建议是否重复、是否漏掉已讲内容。
3. **第 3 天**：10% cron run 读取新缓存，观察 `mismatch_rate` 和 `origin_qps`。
4. **稳定后**：100% 切新缓存，旧缓存保留 7 天只读。
5. **清理**：确认无回滚需求后，把旧 namespace 移入归档或删除。

注意：如果迁移涉及权限边界，`security_mismatch_count > 0` 时不要灰度扩大，直接停。

## 6. Rollback Checklist

缓存迁移必须提前写好回滚条件：

- `mismatch_rate > 1%`：暂停扩大灰度。
- `security_mismatch_count > 0`：立即切回旧路径。
- `origin_qps` 超过基线 2 倍：关闭新路径读取，保留 dual write。
- `p95_latency_ms` 超过基线 30%：只保留 shadow read，排查新存储。
- 用户可见答案错误：回滚 rollout 到 0%，保留证据样本。

回滚不是失败，是迁移系统设计的一部分。

## 7. 小结

缓存迁移的正确姿势：

- **dual write**：先把新缓存预热。
- **shadow read**：先比较，不影响用户。
- **progressive rollout**：小流量稳定后再扩大。
- **metrics + rollback**：指标异常能自动刹车。
- **security mismatch 零容忍**：权限相关迁移不能赌。

一句话：**成熟 Agent 的缓存升级，不是“换个 key 就上线”，而是像发布一个有状态系统一样灰度迁移。**
