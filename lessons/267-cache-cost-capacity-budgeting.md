# 267. Agent 缓存成本归因与容量预算（Cache Cost Attribution & Capacity Budgeting）

> 缓存不是“免费加速器”。它会占内存、占磁盘、占 Redis、占刷新带宽，还会制造一致性风险。成熟 Agent 要知道：每一个缓存 key 到底省了多少钱，又花了多少资源。

前面我们讲了缓存准入、命名、生命周期、墓碑、预热和刷新。今天补上最后一块很工程化的能力：**缓存成本归因与容量预算**。

一句话：不要只看 hit ratio，要看 **net value**。

```text
net_value = saved_origin_cost + saved_latency_value - storage_cost - refresh_cost - stale_risk_cost
```

命中率高但重建很便宜、value 很大、长期没人用的缓存，可能是亏钱的；命中率一般但每次 miss 都要调用昂贵 LLM/API 的 key，反而值得重点保护。

## 1. 为什么 Agent 缓存要算账？

Agent 系统里的缓存成本比普通 Web 更复杂：

- LLM 结果缓存：省 token 很明显，但 value 可能很大，存储成本高。
- 工具结果缓存：省 API 调用，但 stale 可能影响决策。
- RAG 检索缓存：省 embedding/search 延迟，但受索引版本影响。
- 用户画像/权限缓存：命中很有价值，但安全边界更严格。
- Cron/heartbeat 预热缓存：提升体验，但后台刷新可能把源站打爆。

所以缓存层要按 `key_class` 做归因：

```json
{
  "keyClass": "llm:summary",
  "hits": 1280,
  "misses": 74,
  "bytes": 84211320,
  "savedCostUsd": 12.84,
  "storageCostUsd": 0.09,
  "refreshCostUsd": 0.37,
  "p95RebuildMs": 4200,
  "staleServeCount": 31
}
```

这个账本让你回答三个问题：

1. 哪些缓存真的省钱？
2. 哪些缓存占空间但没价值？
3. 容量紧张时应该先淘汰谁？

## 2. Cache Budget Ledger：缓存账本模型

建议每次 cache get/set/refresh 都记一笔轻量事件：

```text
cache_event = {
  key_class,
  event: hit | miss | set | refresh | evict | stale_hit,
  bytes,
  origin_cost_usd,
  latency_saved_ms,
  rebuild_ms,
  risk,
  tenant,
  ts
}
```

聚合后得到每类缓存的预算状态：

```text
value_score =
  saved_cost_usd * 100
  + saved_latency_ms * latency_weight
  - storage_bytes * storage_weight
  - refresh_cost_usd * 100
  - stale_hits * risk_weight
```

经验规则：

- `value_score > 0`：值得保留，甚至预热。
- `value_score ≈ 0`：只保留短 TTL，不主动刷新。
- `value_score < 0`：进入淘汰候选，降低 TTL 或禁写。

这比“LRU 一刀切”更适合 Agent，因为 Agent 的 key 价值差异极大。

## 3. learn-claude-code：Python 教学版

```python
# learn-claude-code/cache_budget_ledger.py
from __future__ import annotations

from dataclasses import dataclass, field
from time import time


@dataclass
class CacheStats:
    hits: int = 0
    misses: int = 0
    bytes: int = 0
    saved_cost_usd: float = 0.0
    refresh_cost_usd: float = 0.0
    stale_hits: int = 0
    rebuild_ms_total: float = 0.0
    rebuild_count: int = 0

    def hit_ratio(self) -> float:
        total = self.hits + self.misses
        return self.hits / total if total else 0.0

    def avg_rebuild_ms(self) -> float:
        return self.rebuild_ms_total / self.rebuild_count if self.rebuild_count else 0.0

    def net_value(self) -> float:
        storage_cost = self.bytes / 1_000_000_000 * 0.20  # 示例：$0.20/GB-month
        stale_risk_cost = self.stale_hits * 0.01
        return self.saved_cost_usd - self.refresh_cost_usd - storage_cost - stale_risk_cost


@dataclass
class CacheBudgetLedger:
    stats: dict[str, CacheStats] = field(default_factory=dict)

    def _bucket(self, key_class: str) -> CacheStats:
        self.stats.setdefault(key_class, CacheStats())
        return self.stats[key_class]

    def record_hit(self, key_class: str, origin_cost_usd: float) -> None:
        bucket = self._bucket(key_class)
        bucket.hits += 1
        bucket.saved_cost_usd += origin_cost_usd

    def record_miss(self, key_class: str, rebuild_ms: float) -> None:
        bucket = self._bucket(key_class)
        bucket.misses += 1
        bucket.rebuild_count += 1
        bucket.rebuild_ms_total += rebuild_ms

    def record_set(self, key_class: str, size_bytes: int) -> None:
        self._bucket(key_class).bytes += size_bytes

    def record_refresh(self, key_class: str, cost_usd: float) -> None:
        self._bucket(key_class).refresh_cost_usd += cost_usd

    def eviction_candidates(self) -> list[tuple[str, float]]:
        scored = [(key_class, s.net_value()) for key_class, s in self.stats.items()]
        return sorted(scored, key=lambda item: item[1])


ledger = CacheBudgetLedger()
ledger.record_hit("llm:summary", origin_cost_usd=0.004)
ledger.record_set("llm:summary", size_bytes=120_000)
print(ledger.eviction_candidates())
```

教学版重点不是精确计费，而是建立习惯：**每次命中都记录省了什么，每次写入都记录花了什么**。

## 4. pi-mono：TypeScript 生产版预算中间件

生产系统建议把预算做成 cache middleware，不污染业务工具。

```ts
// pi-mono/packages/agent-runtime/cache/cache-budget-middleware.ts
export type CacheEvent = "hit" | "miss" | "set" | "refresh" | "evict" | "stale_hit";

export interface CacheBudgetEvent {
  keyClass: string;
  event: CacheEvent;
  bytes?: number;
  originCostUsd?: number;
  refreshCostUsd?: number;
  rebuildMs?: number;
  risk?: "low" | "medium" | "high";
  tenantId?: string;
  ts: number;
}

export interface MetricsSink {
  increment(name: string, tags: Record<string, string>): void;
  histogram(name: string, value: number, tags: Record<string, string>): void;
  gauge(name: string, value: number, tags: Record<string, string>): void;
}

export class CacheBudgetMiddleware {
  constructor(private readonly metrics: MetricsSink) {}

  record(event: CacheBudgetEvent): void {
    const tags = {
      key_class: event.keyClass,
      event: event.event,
      risk: event.risk ?? "unknown",
    };

    this.metrics.increment("cache.events_total", tags);

    if (event.bytes !== undefined) {
      this.metrics.histogram("cache.bytes", event.bytes, tags);
    }

    if (event.originCostUsd !== undefined && event.event === "hit") {
      this.metrics.histogram("cache.saved_cost_usd", event.originCostUsd, tags);
    }

    if (event.refreshCostUsd !== undefined) {
      this.metrics.histogram("cache.refresh_cost_usd", event.refreshCostUsd, tags);
    }

    if (event.rebuildMs !== undefined) {
      this.metrics.histogram("cache.rebuild_ms", event.rebuildMs, tags);
    }
  }

  shouldAdmit(input: {
    keyClass: string;
    estimatedSizeBytes: number;
    estimatedOriginCostUsd: number;
    risk: "low" | "medium" | "high";
    monthlyBudgetBytesLeft: number;
  }): boolean {
    if (input.risk === "high") return false;
    if (input.estimatedSizeBytes > input.monthlyBudgetBytesLeft) return false;

    const valueDensity = input.estimatedOriginCostUsd / Math.max(input.estimatedSizeBytes, 1);
    return valueDensity >= 0.00000001;
  }
}
```

这个中间件提供两个能力：

1. **观测**：持续上报每类 key 的成本、大小、重建耗时。
2. **准入**：空间不够时，不是随机拒绝，而是拒绝“价值密度太低”的缓存。

生产里可以再加 Redis/ClickHouse 聚合，按小时生成报表：

```sql
SELECT
  key_class,
  sumIf(origin_cost_usd, event = 'hit') AS saved_usd,
  sumIf(refresh_cost_usd, event = 'refresh') AS refresh_usd,
  sumIf(bytes, event = 'set') AS written_bytes,
  countIf(event = 'stale_hit') AS stale_hits
FROM cache_events
WHERE ts > now() - interval 24 hour
GROUP BY key_class
ORDER BY saved_usd - refresh_usd DESC;
```

## 5. OpenClaw 实战：文件式缓存预算报表

OpenClaw 的课程 cron、heartbeat、账单巡检都适合用“轻量文件账本”：

```json
// memory/cache-budget/2026-05-08.json
{
  "agent-course": {
    "repo_readme": {
      "hits": 4,
      "misses": 1,
      "bytes": 52000,
      "savedCostUsd": 0.003,
      "refreshCostUsd": 0.001,
      "lastUsedAt": "2026-05-08T12:29:00+11:00"
    },
    "tools_topics": {
      "hits": 4,
      "misses": 0,
      "bytes": 48000,
      "savedCostUsd": 0.008,
      "refreshCostUsd": 0.000,
      "lastUsedAt": "2026-05-08T12:29:00+11:00"
    }
  }
}
```

Heartbeat 可以每天做一次小报表：

```python
# openclaw_cache_budget_report.py
import json
from pathlib import Path

path = Path("memory/cache-budget/2026-05-08.json")
data = json.loads(path.read_text())

for job, classes in data.items():
    print(f"## {job}")
    for key_class, s in classes.items():
        storage_cost = s["bytes"] / 1_000_000_000 * 0.20
        net = s["savedCostUsd"] - s["refreshCostUsd"] - storage_cost
        hit_ratio = s["hits"] / max(s["hits"] + s["misses"], 1)
        print(f"- {key_class}: hit={hit_ratio:.0%}, net=${net:.4f}")
```

这不需要上复杂平台，但能让 Agent 在长期运行中知道：哪些文件缓存值得保留，哪些应该压缩、降 TTL 或删除。

## 6. 容量策略：从“缓存满了”到“预算驱动”

建议给每类缓存配置预算：

```yaml
cacheBudgets:
  llm.summary:
    maxBytes: 2GB
    minNetValueUsdPerGb: 5
    allowRefreshAhead: true
  tool.search_results:
    maxBytes: 500MB
    minNetValueUsdPerGb: 1
    allowRefreshAhead: false
  security.permission:
    maxBytes: 50MB
    minNetValueUsdPerGb: 999
    allowRefreshAhead: false
    forceLiveCheckBeforeSideEffect: true
```

当容量紧张时，淘汰顺序建议：

1. 负 net value 的 key class。
2. stale risk 高但 saved cost 低的缓存。
3. 长期没命中的大 value。
4. 可重建且重建成本低的缓存。
5. 最后才动昂贵 LLM/RAG 结果缓存。

## 7. 常见坑

- **只看命中率**：90% hit ratio 不代表值钱，可能只是命中了便宜数据。
- **只看 token 省钱**：忽略 Redis 内存、刷新 API、后台任务成本。
- **不按租户归因**：一个大客户可能吃掉所有缓存预算。
- **把高风险缓存也算正收益**：安全/支付/权限类缓存的 stale risk 应该重罚。
- **没有容量上限**：缓存最终会变成另一个无限增长的数据库。

## 8. 小结

今天的核心：**缓存优化要从 hit ratio 升级到 net value**。

落地三步：

1. 每个 cache event 记录 key_class、bytes、saved_cost、refresh_cost、risk。
2. 按 key_class/tenant 聚合，算 net value 和 value density。
3. 用预算驱动准入、刷新和淘汰。

成熟 Agent 的缓存系统，不只是“快”，还要知道自己为什么快、快得值不值、什么时候该把缓存删掉。
