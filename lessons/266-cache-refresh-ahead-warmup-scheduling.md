# 266. Agent 缓存刷新与预热调度（Cache Refresh-Ahead & Warmup Scheduling）

> 好缓存不只是“读到了就存起来”，还要在用户真正需要之前，把高价值数据提前刷新好。Refresh-Ahead 解决“快过期才刷新”，Warmup 解决“冷启动第一口慢”。

前面我们讲了缓存键、准入、淘汰、多级缓存、迁移、墓碑。今天补一个和体验强相关的主题：**缓存什么时候主动刷新、什么时候预热、怎么避免刷新任务把源站打爆**。

## 1. 为什么 Agent 需要主动刷新？

普通 Web 缓存常见策略是 lazy loading：请求来了，miss 了，再回源。

Agent 系统里这不够：

- Heartbeat/cron 一醒来要查邮箱、日历、账单，第一轮 miss 会让响应很慢。
- 多 Sub-agent 同时启动，可能一起打同一个 profile/config API。
- RAG 索引、工具 schema、用户偏好、repo 状态这类数据变化不频繁，但冷启动非常影响首轮体验。
- 高风险决策前仍要 live check，但低风险 planning/context 可以用提前刷新的缓存。

所以生产 Agent 缓存通常分两类主动任务：

1. **Refresh-Ahead**：缓存还没过期，但快过期了，后台异步刷新。
2. **Warmup**：系统启动、定时任务前、用户高概率回来前，提前加载关键上下文。

## 2. Refresh-Ahead 的判断公式

不要所有 key 都主动刷新。建议给缓存条目加元数据：

```json
{
  "key": "tenant:t1:user:u1:calendar:24h",
  "expiresAt": 1778205600000,
  "lastAccessAt": 1778202000000,
  "hitCount": 42,
  "refreshCost": 0.02,
  "risk": "low",
  "class": "planning_context"
}
```

一个简单可用的判断：

```text
shouldRefreshAhead =
  now > expiresAt - refreshWindow
  && hitCount >= minHotHits
  && risk not in [security_decision, external_side_effect]
  && refreshCost <= budget
```

经验值：

- 用户偏好 / 工具 schema：过期前 20-30% TTL 刷新。
- 日历 / 邮件摘要：任务前 1-5 分钟预热，展示前再轻量复核。
- 权限 / 支付 / 账号状态：不要只靠 refresh-ahead，执行前必须 live check。

## 3. learn-claude-code：Python 教学版

```python
# learn-claude-code/cache_refresh_ahead.py
from __future__ import annotations

from dataclasses import dataclass
from time import time
from typing import Any, Callable


@dataclass
class CacheEntry:
    value: Any
    expires_at: float
    ttl_seconds: int
    hit_count: int = 0
    refreshing: bool = False


class RefreshAheadCache:
    def __init__(self, fetcher: Callable[[str], Any]):
        self.store: dict[str, CacheEntry] = {}
        self.fetcher = fetcher

    def get(self, key: str) -> Any | None:
        entry = self.store.get(key)
        now = time()
        if not entry or entry.expires_at <= now:
            return self.refresh_sync(key)

        entry.hit_count += 1
        if self.should_refresh_ahead(entry, now):
            self.refresh_background(key, entry)

        return entry.value

    def should_refresh_ahead(self, entry: CacheEntry, now: float) -> bool:
        age_left = entry.expires_at - now
        refresh_window = entry.ttl_seconds * 0.25
        return age_left <= refresh_window and entry.hit_count >= 3 and not entry.refreshing

    def refresh_sync(self, key: str) -> Any:
        value = self.fetcher(key)
        self.store[key] = CacheEntry(value=value, expires_at=time() + 300, ttl_seconds=300)
        return value

    def refresh_background(self, key: str, entry: CacheEntry) -> None:
        # 教学版用同步函数表示；生产里应提交到 asyncio task / queue。
        entry.refreshing = True
        try:
            value = self.fetcher(key)
            self.store[key] = CacheEntry(
                value=value,
                expires_at=time() + entry.ttl_seconds,
                ttl_seconds=entry.ttl_seconds,
                hit_count=entry.hit_count,
            )
        finally:
            entry.refreshing = False
```

教学版先抓核心：**命中时顺手判断是否快过期，后台刷新，但当前请求仍返回旧值**。这就是 stale-while-refresh 的体验优化。

## 4. pi-mono：TypeScript 生产版刷新调度器

生产版不要在请求线程里直接刷新，应交给带并发限制和预算控制的 scheduler。

```ts
// pi-mono/packages/agent-runtime/cache/refresh-scheduler.ts
export type CacheRisk = "answer" | "planning_context" | "security_decision" | "external_side_effect";

export interface RefreshableEntry<T> {
  key: string;
  value: T;
  expiresAt: number;
  ttlMs: number;
  hitCount: number;
  lastAccessAt: number;
  risk: CacheRisk;
  refreshCost: number;
}

export interface CacheStore<T> {
  get(key: string): Promise<RefreshableEntry<T> | null>;
  set(entry: RefreshableEntry<T>): Promise<void>;
  tryLock(key: string, ttlMs: number): Promise<boolean>;
  unlock(key: string): Promise<void>;
}

export class RefreshAheadScheduler<T> {
  private inFlight = 0;

  constructor(
    private readonly store: CacheStore<T>,
    private readonly fetchFresh: (key: string) => Promise<T>,
    private readonly opts = { maxConcurrent: 4, minHotHits: 3, maxRefreshCost: 0.05 },
  ) {}

  async maybeRefresh(entry: RefreshableEntry<T>): Promise<void> {
    if (!this.shouldRefresh(entry)) return;
    if (this.inFlight >= this.opts.maxConcurrent) return;

    const locked = await this.store.tryLock(`refresh:${entry.key}`, 60_000);
    if (!locked) return;

    this.inFlight += 1;
    queueMicrotask(async () => {
      const started = Date.now();
      try {
        const fresh = await this.fetchFresh(entry.key);
        await this.store.set({
          ...entry,
          value: fresh,
          expiresAt: Date.now() + entry.ttlMs,
          lastAccessAt: Date.now(),
        });
        metrics.histogram("cache.refresh_ahead.ms", Date.now() - started, { risk: entry.risk });
      } catch (error) {
        metrics.increment("cache.refresh_ahead.failed", { key: entry.key });
      } finally {
        this.inFlight -= 1;
        await this.store.unlock(`refresh:${entry.key}`);
      }
    });
  }

  private shouldRefresh(entry: RefreshableEntry<T>): boolean {
    if (entry.risk === "security_decision" || entry.risk === "external_side_effect") return false;
    if (entry.hitCount < this.opts.minHotHits) return false;
    if (entry.refreshCost > this.opts.maxRefreshCost) return false;

    const refreshWindow = entry.ttlMs * 0.25;
    return Date.now() >= entry.expiresAt - refreshWindow;
  }
}
```

这里有四个生产关键点：

1. **风险分级**：安全决策和外部副作用不靠后台刷新兜底。
2. **热度门槛**：冷门 key 不刷新，避免浪费。
3. **分布式锁**：多个 worker 不重复刷新同一个 key。
4. **并发预算**：刷新是后台任务，也不能无限开火。

## 5. OpenClaw 实战：cron 前预热上下文

OpenClaw 的定时任务很适合做 warmup：课程 cron、账单 cron、heartbeat 都有固定运行窗口。

可以把预热状态写成文件：

```json
// memory/cache/warmup/agent-course.json
{
  "job": "agent-course",
  "preparedAt": "2026-05-08T09:25:00+11:00",
  "keys": [
    "repo:agent-course:readme",
    "repo:agent-course:latest-lessons",
    "memory:tools:agent-course-topics"
  ],
  "expiresAt": "2026-05-08T09:40:00+11:00"
}
```

执行课前 5 分钟可以预热：

```bash
cd /Users/bot001/.openclaw/workspace/agent-course

git fetch origin main
python3 scripts/build_lesson_index.py > /tmp/agent-course-index.json
```

真正上课时：

- 先读 warmup index，快速知道上一课编号和主题。
- 再 `git pull --rebase --autostash origin main` 做权威同步。
- 写课、发群、commit/push。

这就是 **warmup 不替代 live check，而是缩短 live check 前的准备时间**。

## 6. 刷新调度的三条红线

### 红线 1：不要刷新高风险决策缓存

权限、余额、支付状态、审批状态，只能用于展示或 planning，执行前必须回源复核。

### 红线 2：不要让 refresh-ahead 变成源站 DDoS

必须有：

- 全局 refresh 并发上限。
- 每个源站的 QPS 上限。
- 错误率升高时熔断。
- Retry-After 尊重。

### 红线 3：刷新写入要尊重 tombstone 和 version

上一课讲过：删除墓碑版本比旧 refresh 结果新时，旧结果不能写回。

```ts
if (current.kind === "tombstone" && current.version >= refreshVersion) {
  return { skipped: true, reason: "tombstone_is_newer" };
}
```

否则后台刷新任务会把已经删除的敏感数据复活。

## 7. 一句话总结

Refresh-Ahead 让热数据不过期，Warmup 让冷启动不拖后腿；但主动刷新必须受风险、热度、预算、版本四个维度约束。成熟 Agent 的缓存系统，不是等用户踩到 miss 才行动，而是在安全边界内提前准备好答案。
