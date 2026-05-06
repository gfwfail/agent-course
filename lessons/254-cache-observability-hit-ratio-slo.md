# 254. Agent 缓存可观测性与命中率 SLO（Cache Observability & Hit Ratio SLO）

前面几课我们讲了缓存失效、缓存击穿、预热、热点治理。今天补上最后一块：**怎么知道缓存真的有用，而不是“看起来用了缓存”**。

很多 Agent 系统加缓存后还是慢，原因通常不是缓存算法错，而是没有观测：不知道命中率、不知道 rebuild 多慢、不知道哪些 key 热、不知道 stale 数据被用了多少次。

## 核心思想

缓存不是一个 `get/set` 工具，而是一套有 SLO 的运行时系统：

- **hit ratio**：命中率是否达标
- **rebuild latency**：重建一次缓存有多慢
- **stale served**：给用户返回过期数据的次数
- **inflight rebuilds**：正在重建的 key 数量
- **hot key list**：最贵、最热、最容易击穿的 key
- **error fallback**：源站失败时是否安全降级

一个实用目标可以这样定：

> 高价值只读工具缓存命中率 ≥ 80%，P95 rebuild < 2s，stale fallback 有记录，连续 3 次 rebuild 失败触发告警。

## learn-claude-code：Python 教学版

教学版先做简单的指标记录器。重点不是 Prometheus，而是把每次缓存决策结构化记录下来。

```python
# learn_claude_code/cache_metrics.py
from dataclasses import dataclass, field
from time import perf_counter
from collections import defaultdict

@dataclass
class CacheStats:
    hits: int = 0
    misses: int = 0
    stale: int = 0
    rebuild_errors: int = 0
    rebuild_ms: list[float] = field(default_factory=list)

    @property
    def hit_ratio(self) -> float:
        total = self.hits + self.misses
        return self.hits / total if total else 0.0

class CacheMetrics:
    def __init__(self):
        self.by_key_class: dict[str, CacheStats] = defaultdict(CacheStats)

    def hit(self, key_class: str):
        self.by_key_class[key_class].hits += 1

    def miss(self, key_class: str):
        self.by_key_class[key_class].misses += 1

    def served_stale(self, key_class: str):
        self.by_key_class[key_class].stale += 1

    def rebuild(self, key_class: str, fn):
        started = perf_counter()
        try:
            return fn()
        except Exception:
            self.by_key_class[key_class].rebuild_errors += 1
            raise
        finally:
            elapsed = (perf_counter() - started) * 1000
            self.by_key_class[key_class].rebuild_ms.append(elapsed)

    def report(self) -> list[dict]:
        rows = []
        for key_class, stat in self.by_key_class.items():
            rows.append({
                "key_class": key_class,
                "hit_ratio": round(stat.hit_ratio, 3),
                "hits": stat.hits,
                "misses": stat.misses,
                "stale": stat.stale,
                "rebuild_errors": stat.rebuild_errors,
                "avg_rebuild_ms": round(sum(stat.rebuild_ms) / len(stat.rebuild_ms), 1)
                    if stat.rebuild_ms else 0,
            })
        return rows
```

接到 Agent 工具层：

```python
metrics = CacheMetrics()

async def cached_tool_call(cache, key, key_class, loader):
    item = await cache.get(key)
    if item and not item.expired:
        metrics.hit(key_class)
        return item.value

    if item and item.soft_expired:
        metrics.served_stale(key_class)
        # 后台重建，当前请求先返回旧值
        schedule_background_rebuild(key, loader)
        return item.value

    metrics.miss(key_class)
    return metrics.rebuild(key_class, lambda: loader())
```

这就是最小闭环：每次缓存决策都变成可统计事件。

## pi-mono：TypeScript 生产中间件

生产版不要让业务工具到处手写指标。用中间件包住 CacheClient。

```ts
// pi-mono/packages/agent-runtime/src/cache/ObservedCache.ts
type CacheEvent =
  | { type: 'hit'; keyClass: string; key: string }
  | { type: 'miss'; keyClass: string; key: string }
  | { type: 'stale'; keyClass: string; key: string }
  | { type: 'rebuild'; keyClass: string; key: string; ms: number; ok: boolean };

interface MetricsSink {
  emit(event: CacheEvent): void;
}

export class ObservedCache<T> {
  constructor(
    private readonly cache: CacheClient<T>,
    private readonly metrics: MetricsSink,
  ) {}

  async getOrLoad(args: {
    key: string;
    keyClass: string;
    ttlMs: number;
    loader: () => Promise<T>;
  }): Promise<T> {
    const cached = await this.cache.get(args.key);

    if (cached?.state === 'fresh') {
      this.metrics.emit({ type: 'hit', keyClass: args.keyClass, key: args.key });
      return cached.value;
    }

    if (cached?.state === 'stale') {
      this.metrics.emit({ type: 'stale', keyClass: args.keyClass, key: args.key });
      void this.rebuild(args);
      return cached.value;
    }

    this.metrics.emit({ type: 'miss', keyClass: args.keyClass, key: args.key });
    return this.rebuild(args);
  }

  private async rebuild(args: {
    key: string;
    keyClass: string;
    ttlMs: number;
    loader: () => Promise<T>;
  }): Promise<T> {
    const started = Date.now();
    try {
      const value = await args.loader();
      await this.cache.set(args.key, value, { ttlMs: args.ttlMs });
      this.metrics.emit({
        type: 'rebuild',
        keyClass: args.keyClass,
        key: args.key,
        ms: Date.now() - started,
        ok: true,
      });
      return value;
    } catch (error) {
      this.metrics.emit({
        type: 'rebuild',
        keyClass: args.keyClass,
        key: args.key,
        ms: Date.now() - started,
        ok: false,
      });
      throw error;
    }
  }
}
```

业务工具只声明 `keyClass`：

```ts
const countries = await observedCache.getOrLoad({
  key: `countries:${locale}`,
  keyClass: 'catalog.countries',
  ttlMs: 10 * 60_000,
  loader: () => catalogApi.listCountries(locale),
});
```

这样后面看报表时，不会被具体 key 淹没，而是按业务类别判断：

- `catalog.countries` 命中率低：可能 TTL 太短或 locale key 过细
- `github.pr_status` rebuild 慢：可能 API 限流或缺少 batch
- `email.unread_digest` stale 太高：可能后台刷新失败

## OpenClaw：Heartbeat/Cron 自动巡检

OpenClaw 适合做缓存健康巡检：定时读指标，发现异常才打扰人。

```ts
// openclaw-cron/cache-health-check.ts
const SLO = {
  minHitRatio: 0.8,
  maxP95RebuildMs: 2_000,
  maxErrorRate: 0.02,
};

export async function checkCacheHealth(metrics: CacheMetricsStore) {
  const rows = await metrics.queryLastHour();
  const problems = rows.filter((row) =>
    row.hitRatio < SLO.minHitRatio ||
    row.p95RebuildMs > SLO.maxP95RebuildMs ||
    row.errorRate > SLO.maxErrorRate
  );

  if (problems.length === 0) {
    return { notify: false, message: 'HEARTBEAT_OK' };
  }

  return {
    notify: true,
    message: problems.map((p) =>
      `缓存异常 ${p.keyClass}: hit=${pct(p.hitRatio)}, p95=${p.p95RebuildMs}ms, err=${pct(p.errorRate)}`
    ).join('\n'),
  };
}
```

实战里可以把它接到：

- Heartbeat：轻量巡检，只在异常时通知
- Cron：每小时生成缓存 SLO 摘要
- Grafana：按 `key_class` 展示命中率/重建耗时/错误率
- GitHub PR：改缓存策略前后对比指标

## 常见坑

### 1. 只看全局命中率

全局命中率 90% 可能没意义，因为一个超热但不重要的 key 把数字拉高了。要按 `key_class` 看。

### 2. 没记录 stale

stale-while-revalidate 很好用，但必须知道用了多少过期数据。否则你以为系统很稳，其实只是一直在吃旧数据。

### 3. 把 key 原文打进日志

缓存 key 里可能有邮箱、订单号、用户 ID。指标里建议记录：

- `key_class`
- `key_hash`
- `tenant`
- `tool_name`

不要直接记录完整 key。

### 4. SLO 没有动作

SLO 不是贴墙上的数字。低于阈值后要触发动作：

- 自动延长 soft TTL
- 降低 rebuild 并发
- 加入预热队列
- 告警人工检查
- 对高风险工具强制 live check

## 小结

缓存观测的关键不是“有没有 Prometheus”，而是 Agent 每次缓存决策都有证据：

1. 这次是 hit / miss / stale？
2. 重建花了多久？
3. 失败时有没有安全 fallback？
4. 哪类 key 正在拖慢系统？
5. 指标低于 SLO 后有没有自动动作？

没有观测的缓存只是玄学优化；有 SLO 的缓存才是生产系统。
