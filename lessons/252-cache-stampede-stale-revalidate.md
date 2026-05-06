# 252. Agent 缓存击穿与后台刷新（Cache Stampede & Stale-While-Revalidate）

> 核心思想：缓存命中能省钱，缓存失效瞬间也能把系统打爆。Agent 系统里，一个热门问题、一份热门文档、一个热门用户画像同时过期时，几十个会话会一起冲向 LLM/API/数据库，这就是缓存击穿/雪崩。解决办法不是“TTL 设大点”，而是：**请求合并 + 过期抖动 + stale-while-revalidate + 负缓存**。

上一课讲了缓存失效与依赖标签：实体变了，相关缓存必须精准失效。今天补另一半：缓存失效后，如何避免所有 Agent 一起重新计算。

一句话：缓存过期时，不要让每个请求都去重建；只让一个 worker 刷新，其他请求先复用可接受的 stale 数据或等待同一个 Promise。

---

## 1. Agent 系统为什么更容易缓存击穿

传统 Web 服务的缓存击穿通常是热门商品、热门页面。Agent 系统更麻烦，因为重建缓存的成本可能非常高：

- RAG 文档摘要：重新抓取、分块、embedding、总结；
- 用户画像：读取历史对话 + LLM 提炼；
- 工具选择缓存：加载 tool schema + 语义匹配；
- 价格/订单/库存：外部 API 有限流；
- Cron/Heartbeat：同一时间批量醒来，天然容易同时打同一批 key。

最危险的时间点不是缓存命中时，而是“很多 key 同时过期”时：

```txt
10:00:00  100 个 cron 同时醒来
10:00:01  pricing-doc-summary 过期
10:00:02  100 个 run 同时调用 LLM 重建摘要
10:00:05  LLM/API 429，缓存仍没重建成功
10:00:10  重试风暴开始
```

这时候系统不是被用户流量打爆，而是被自己的 Agent 打爆。

---

## 2. 四件套：SingleFlight + Jitter + SWR + Negative Cache

生产里建议同时上四层保护：

### 2.1 SingleFlight：同一个 key 只重建一次

当多个请求同时 miss 同一个 key，只允许第一个请求执行真实 loader，后面的请求 await 同一个 in-flight Promise。

```txt
A miss -> loader()
B miss -> wait A
C miss -> wait A
A done -> A/B/C 共用结果
```

这和“请求去重”类似，但重点是缓存重建路径。

### 2.2 TTL Jitter：过期时间加随机抖动

不要让一批缓存同一秒过期。

```ts
const ttl = baseTtlMs + randomBetween(-0.2 * baseTtlMs, 0.2 * baseTtlMs);
```

10 分钟 TTL 实际可能是 8~12 分钟，过期压力被摊平。

### 2.3 Stale-While-Revalidate：先返回旧值，后台刷新

如果缓存已经软过期但还没硬过期：

- 用户请求先拿旧值，保证低延迟；
- 后台只有一个 worker 刷新；
- 刷新失败时保留旧值并缩短下次重试间隔。

关键是区分两个时间：

```txt
softExpiresAt：超过后应该刷新，但仍可读
hardExpiresAt：超过后不准再读，必须重建或降级
```

### 2.4 Negative Cache：失败也要短暂缓存

外部 API 404/权限拒绝/资源不存在，不要每次都查。

```txt
user:404 -> negative cache 60s
api:429 -> backoff cache 30s
permission:denied -> negative cache 5m
```

负缓存要短 TTL，避免把临时错误缓存太久。

---

## 3. learn-claude-code：Python 教学版

下面是一个最小可用的 `AsyncSWRCache`。它支持：

- soft/hard TTL；
- 同 key 请求合并；
- 过期抖动；
- loader 失败时返回 stale；
- 负缓存。

```python
# swr_cache.py
import asyncio
import random
import time
from dataclasses import dataclass
from typing import Any, Awaitable, Callable

Loader = Callable[[], Awaitable[Any]]

@dataclass
class Entry:
    value: Any
    soft_expires_at: float
    hard_expires_at: float
    negative: bool = False
    error: str | None = None

class AsyncSWRCache:
    def __init__(self):
        self.entries: dict[str, Entry] = {}
        self.inflight: dict[str, asyncio.Task] = {}
        self.lock = asyncio.Lock()

    def _jitter(self, ttl: float, ratio: float = 0.2) -> float:
        return ttl + random.uniform(-ttl * ratio, ttl * ratio)

    async def get_or_load(
        self,
        key: str,
        loader: Loader,
        *,
        soft_ttl: float = 60,
        hard_ttl: float = 300,
        negative_ttl: float = 30,
    ) -> Any:
        now = time.time()
        entry = self.entries.get(key)

        # 1) 未软过期：直接返回
        if entry and entry.soft_expires_at > now:
            if entry.negative:
                raise RuntimeError(entry.error or "negative cache hit")
            return entry.value

        # 2) 软过期但未硬过期：返回 stale，并触发后台刷新
        if entry and entry.hard_expires_at > now:
            await self._ensure_refresh(key, loader, soft_ttl, hard_ttl, negative_ttl)
            if entry.negative:
                raise RuntimeError(entry.error or "negative cache hit")
            return entry.value

        # 3) 硬过期/不存在：等待同一个重建任务
        task = await self._ensure_refresh(key, loader, soft_ttl, hard_ttl, negative_ttl)
        await task
        fresh = self.entries[key]
        if fresh.negative:
            raise RuntimeError(fresh.error or "negative cache hit")
        return fresh.value

    async def _ensure_refresh(
        self,
        key: str,
        loader: Loader,
        soft_ttl: float,
        hard_ttl: float,
        negative_ttl: float,
    ) -> asyncio.Task:
        async with self.lock:
            task = self.inflight.get(key)
            if task and not task.done():
                return task

            task = asyncio.create_task(
                self._refresh(key, loader, soft_ttl, hard_ttl, negative_ttl)
            )
            self.inflight[key] = task
            task.add_done_callback(lambda _: self.inflight.pop(key, None))
            return task

    async def _refresh(
        self,
        key: str,
        loader: Loader,
        soft_ttl: float,
        hard_ttl: float,
        negative_ttl: float,
    ):
        now = time.time()
        try:
            value = await loader()
            self.entries[key] = Entry(
                value=value,
                soft_expires_at=now + self._jitter(soft_ttl),
                hard_expires_at=now + self._jitter(hard_ttl),
            )
        except Exception as exc:
            old = self.entries.get(key)
            if old and old.hard_expires_at > now:
                # 保留 stale，避免一次失败清空可用数据
                old.soft_expires_at = now + min(negative_ttl, 15)
                return

            self.entries[key] = Entry(
                value=None,
                soft_expires_at=now + negative_ttl,
                hard_expires_at=now + negative_ttl,
                negative=True,
                error=str(exc),
            )
```

用法：

```python
cache = AsyncSWRCache()

async def get_pricing_summary(doc_id: str):
    return await cache.get_or_load(
        f"pricing-summary:{doc_id}",
        loader=lambda: summarize_pricing_doc(doc_id),
        soft_ttl=600,
        hard_ttl=3600,
    )
```

这里有个关键细节：软过期返回 stale，不代表永远相信旧数据。高风险动作前仍要走上一课的执行前复核。

---

## 4. pi-mono：TypeScript 生产版中间件

生产系统里可以把它做成缓存中间件，统一包住昂贵工具：

```ts
type CacheEntry<T> = {
  value: T | null;
  softExpiresAt: number;
  hardExpiresAt: number;
  negative?: boolean;
  error?: string;
  observedAt: string;
};

type SwrPolicy<Args, Result> = {
  key: (args: Args, ctx: ToolContext) => string;
  softTtlMs: number;
  hardTtlMs: number;
  negativeTtlMs?: number;
  allowStale: (ctx: ToolContext) => boolean;
  bypass?: (ctx: ToolContext) => boolean;
};

class SwrCacheMiddleware {
  private inflight = new Map<string, Promise<unknown>>();

  constructor(private store: CacheStore) {}

  wrap<Args, Result>(
    toolName: string,
    policy: SwrPolicy<Args, Result>,
    handler: (args: Args, ctx: ToolContext) => Promise<Result>,
  ) {
    return async (args: Args, ctx: ToolContext): Promise<Result> => {
      if (policy.bypass?.(ctx)) return handler(args, ctx);

      const key = policy.key(args, ctx);
      const now = Date.now();
      const entry = await this.store.get<CacheEntry<Result>>(key);

      if (entry && entry.softExpiresAt > now) {
        if (entry.negative) throw new Error(entry.error ?? "negative cache hit");
        return entry.value as Result;
      }

      if (entry && entry.hardExpiresAt > now && policy.allowStale(ctx)) {
        void this.refreshOnce(key, policy, args, ctx, handler);
        if (entry.negative) throw new Error(entry.error ?? "negative cache hit");
        return entry.value as Result;
      }

      await this.refreshOnce(key, policy, args, ctx, handler);
      const fresh = await this.store.get<CacheEntry<Result>>(key);
      if (!fresh || fresh.negative) throw new Error(fresh?.error ?? "cache refresh failed");
      return fresh.value as Result;
    };
  }

  private async refreshOnce<Args, Result>(
    key: string,
    policy: SwrPolicy<Args, Result>,
    args: Args,
    ctx: ToolContext,
    handler: (args: Args, ctx: ToolContext) => Promise<Result>,
  ) {
    const existing = this.inflight.get(key);
    if (existing) return existing;

    const promise = this.refresh(key, policy, args, ctx, handler)
      .finally(() => this.inflight.delete(key));

    this.inflight.set(key, promise);
    return promise;
  }

  private async refresh<Args, Result>(
    key: string,
    policy: SwrPolicy<Args, Result>,
    args: Args,
    ctx: ToolContext,
    handler: (args: Args, ctx: ToolContext) => Promise<Result>,
  ) {
    const now = Date.now();
    const jitter = (ttl: number) => ttl + Math.floor((Math.random() - 0.5) * ttl * 0.4);

    try {
      const value = await handler(args, ctx);
      await this.store.set(key, {
        value,
        observedAt: new Date().toISOString(),
        softExpiresAt: now + jitter(policy.softTtlMs),
        hardExpiresAt: now + jitter(policy.hardTtlMs),
      });
    } catch (error) {
      const negativeTtl = policy.negativeTtlMs ?? 30_000;
      await this.store.set(key, {
        value: null,
        observedAt: new Date().toISOString(),
        softExpiresAt: now + negativeTtl,
        hardExpiresAt: now + negativeTtl,
        negative: true,
        error: error instanceof Error ? error.message : String(error),
      });
      throw error;
    }
  }
}
```

给昂贵工具加策略：

```ts
const cachedSummarizeDoc = swr.wrap(
  "summarizeDoc",
  {
    key: ({ docId }) => `doc-summary:${docId}`,
    softTtlMs: 10 * 60_000,
    hardTtlMs: 60 * 60_000,
    negativeTtlMs: 30_000,
    allowStale: (ctx) => ctx.risk !== "high",
    bypass: (ctx) => ctx.userIntent.includes("最新"),
  },
  summarizeDoc,
);
```

注意 `allowStale`：低风险问答可以先返回旧摘要，高风险发布/付款/权限变更不允许用 stale。

---

## 5. OpenClaw 实战：Heartbeat/Cron 最容易踩坑

OpenClaw 里最典型的场景是 Heartbeat 和 Cron：

- 多个定时任务整点醒来；
- 都要读同一份 README/TOOLS/MEMORY；
- 都要查同一批 GitHub PR、部署状态、账单；
- 如果每个 run 都重查，成本和限流都会上去。

一个轻量文件版 SWR 状态可以放在：

```txt
memory/cache/{sha256(key)}.json
memory/cache-locks/{sha256(key)}.lock
```

结构建议：

```json
{
  "key": "github:repo:gfwfail/agent-course:recent-prs",
  "value": [{ "number": 252, "state": "open" }],
  "observedAt": "2026-05-06T05:30:00Z",
  "softExpiresAt": "2026-05-06T05:35:00Z",
  "hardExpiresAt": "2026-05-06T06:00:00Z",
  "tags": ["github:gfwfail/agent-course", "prs"]
}
```

执行策略：

1. 普通心跳查状态：允许 stale 5~15 分钟；
2. 发消息、push、部署前：禁用 stale，强制 live check；
3. 读取失败：短负缓存，避免每 30 秒重复撞同一个坏 API；
4. key 加 jitter，避免整点雪崩；
5. 所有刷新写入 evidence：`observedAt/source/confidence`。

这和我们课程 cron 的规则是配套的：README/TOOLS 可以快速读本地缓存，但 `git push` 前必须 live check `gh pr list` 和 `git pull --rebase`。

---

## 6. 设计 Checklist

给 Agent 加缓存时，别只问“TTL 多久”，要问这 8 个问题：

- 这个 key 过期时，会不会有很多 run 同时重建？
- 重建是不是昂贵（LLM/API/DB/embedding）？
- 能不能用 SingleFlight 合并同 key 重建？
- TTL 有没有 jitter，避免整点同时过期？
- 是否区分 soft TTL 和 hard TTL？
- stale 数据在哪些风险等级下允许使用？
- loader 失败时是否保留旧值并做负缓存？
- 外部副作用前是否强制绕过缓存做 live check？

---

## 7. 和前几课的关系

- 第 96 课 Request Deduplication：合并同一时间的重复请求；
- 第 251 课 Cache Invalidation：实体变化时精准删缓存；
- 今天 Cache Stampede：缓存过期/失效后，避免重建风暴；
- 第 226 课 Data Freshness：判断旧观察能不能用于当前决策；
- 第 242 课 Pre-Action Revalidation：副作用前必须重新确认现实。

组合起来就是生产缓存闭环：

```txt
命中缓存 -> 检查 freshness -> 低风险可用 stale -> 后台 singleflight 刷新
实体变化 -> tag invalidate -> jitter 分散重建 -> 失败 negative cache
外部副作用 -> bypass cache -> live check -> 执行
```

---

## 总结

Agent 缓存不是为了“少查一次数据库”这么简单，它是在管理 LLM/API/工具调用的成本、延迟和风险。

成熟做法：

- SingleFlight 防止同 key 重建风暴；
- TTL Jitter 防止整点雪崩；
- Stale-While-Revalidate 兼顾低延迟和新鲜度；
- Negative Cache 防止错误重试风暴；
- 高风险副作用永远绕过 stale，执行前 live check。

一句话收尾：**缓存命中是优化，缓存失效才是系统设计的考试。**
