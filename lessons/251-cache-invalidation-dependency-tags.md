# 251. Agent 缓存失效与依赖标签（Cache Invalidation & Dependency Tags）

> 核心思想：缓存不是 `set(key, value, ttl)` 就完事。Agent 的上下文、工具结果、RAG 片段、用户画像都会变化，真正难的是：**什么时候必须让旧缓存失效**。生产系统里，错误缓存比没有缓存更危险。

前面讲过缓存策略、语义缓存、Prompt Caching 和数据新鲜度。今天补上缓存系统最容易被忽略的一块：**依赖标签失效（dependency tags）**。

一句话：每个缓存条目不只记录 key，还记录它依赖了哪些实体；实体变化时，按 tag 批量失效相关缓存。

---

## 1. 为什么 TTL 不够

很多 Agent 系统这样缓存工具结果：

```ts
cache.set(`order:${orderId}`, result, { ttl: 300 });
```

问题是：

- 订单 5 秒后已退款，但缓存还能活 295 秒；
- 用户权限刚被收回，画像缓存还在注入 system prompt；
- 文档刚更新，RAG 摘要仍引用旧片段；
- 工具 schema 变了，旧的 tool selection cache 还在推荐不存在的参数。

TTL 只能解决“最终会过期”，不能解决“变化后立刻不能再用”。

更可靠的设计是：

```txt
cache entry = value + ttl + tags + observedAt + version
```

例如：

```json
{
  "key": "agent:answer:user42:pricing-question",
  "tags": ["user:42", "doc:pricing", "tenant:acme"],
  "observedAt": "2026-05-06T02:30:00Z",
  "version": 7
}
```

当 `doc:pricing` 更新，所有带这个 tag 的缓存一起失效。

---

## 2. 三层缓存失效模型

建议把 Agent 缓存分三层管理：

### 第一层：时间失效

适合低风险、变化慢的数据：

- 工具列表缓存；
- 公共文档摘要；
- 非关键网页抓取结果。

规则：`ttl` 到期自动失效。

### 第二层：标签失效

适合有明确依赖的数据：

- 用户画像：`user:{id}`；
- 订单查询：`order:{id}`；
- RAG 文档片段：`doc:{id}`；
- 租户配置：`tenant:{id}`；
- 工具 schema：`tool:{name}:schema`。

规则：依赖实体变化时 `invalidateTag(tag)`。

### 第三层：执行前复核

适合高风险动作：

- 发消息；
- 下单/退款；
- 部署；
- 写数据库；
- 修改权限。

规则：即使缓存命中，外部副作用前也要重新查一次关键前提。

> 缓存可以加速观察，不能替代执行前确认。

---

## 3. learn-claude-code：Python 教学版

最小可用版本可以用 SQLite 存缓存和 tag 索引：

```python
# tagged_cache.py
import json
import sqlite3
import time
from dataclasses import dataclass
from typing import Any

@dataclass
class CacheHit:
    key: str
    value: Any
    tags: list[str]
    observed_at: float

class TaggedCache:
    def __init__(self, path: str = "memory/tagged-cache.sqlite"):
        self.db = sqlite3.connect(path)
        self.db.executescript("""
        create table if not exists cache_entries (
          key text primary key,
          value text not null,
          expires_at real not null,
          observed_at real not null
        );
        create table if not exists cache_tags (
          tag text not null,
          key text not null,
          primary key(tag, key)
        );
        """)

    def set(self, key: str, value: Any, *, ttl: int, tags: list[str]):
        now = time.time()
        expires_at = now + ttl
        with self.db:
            self.db.execute(
                "replace into cache_entries(key,value,expires_at,observed_at) values(?,?,?,?)",
                (key, json.dumps(value, ensure_ascii=False), expires_at, now),
            )
            self.db.execute("delete from cache_tags where key=?", (key,))
            self.db.executemany(
                "insert or ignore into cache_tags(tag,key) values(?,?)",
                [(tag, key) for tag in tags],
            )

    def get(self, key: str) -> CacheHit | None:
        row = self.db.execute(
            "select value, expires_at, observed_at from cache_entries where key=?",
            (key,),
        ).fetchone()
        if not row:
            return None
        value, expires_at, observed_at = row
        if expires_at < time.time():
            self.delete(key)
            return None
        tags = [r[0] for r in self.db.execute(
            "select tag from cache_tags where key=?", (key,)
        )]
        return CacheHit(key, json.loads(value), tags, observed_at)

    def delete(self, key: str):
        with self.db:
            self.db.execute("delete from cache_entries where key=?", (key,))
            self.db.execute("delete from cache_tags where key=?", (key,))

    def invalidate_tag(self, tag: str) -> int:
        keys = [r[0] for r in self.db.execute(
            "select key from cache_tags where tag=?", (tag,)
        )]
        for key in keys:
            self.delete(key)
        return len(keys)
```

工具调用时写入 tag：

```python
cache = TaggedCache()

async def get_user_profile(user_id: str):
    key = f"user-profile:{user_id}"
    hit = cache.get(key)
    if hit:
        return hit.value

    profile = await db.fetch_user_profile(user_id)
    cache.set(
        key,
        profile,
        ttl=3600,
        tags=[f"user:{user_id}", "profile"],
    )
    return profile

async def update_user_profile(user_id: str, patch: dict):
    await db.update_user_profile(user_id, patch)
    cache.invalidate_tag(f"user:{user_id}")
```

这样用户画像一更新，所有依赖这个用户的上下文缓存都会失效。

---

## 4. pi-mono：TypeScript 生产版中间件

生产版建议把 cache 做成 Agent Loop 中间件，而不是散落在每个工具里。

```ts
type CachePolicy = {
  ttlMs: number;
  tags: (args: unknown, result: unknown, ctx: ToolContext) => string[];
  key: (toolName: string, args: unknown, ctx: ToolContext) => string;
  bypassWhen?: (ctx: ToolContext) => boolean;
};

type CachedValue<T> = {
  value: T;
  tags: string[];
  observedAt: string;
  expiresAt: number;
};

class CacheInvalidationMiddleware {
  constructor(
    private store: CacheStore,
    private policies: Record<string, CachePolicy>,
  ) {}

  async execute<T>(
    toolName: string,
    args: unknown,
    ctx: ToolContext,
    next: () => Promise<T>,
  ): Promise<T> {
    const policy = this.policies[toolName];
    if (!policy || policy.bypassWhen?.(ctx)) return next();

    const key = policy.key(toolName, args, ctx);
    const cached = await this.store.get<CachedValue<T>>(key);

    if (cached && cached.expiresAt > Date.now()) {
      ctx.trace.addEvent("cache.hit", { key, tags: cached.tags });
      return cached.value;
    }

    ctx.trace.addEvent("cache.miss", { key });
    const result = await next();
    const tags = policy.tags(args, result, ctx);

    await this.store.set(key, {
      value: result,
      tags,
      observedAt: new Date().toISOString(),
      expiresAt: Date.now() + policy.ttlMs,
    });

    for (const tag of tags) {
      await this.store.addTagIndex(tag, key);
    }

    return result;
  }

  async invalidateTag(tag: string) {
    const keys = await this.store.keysByTag(tag);
    await Promise.all(keys.map((key) => this.store.delete(key)));
    await this.store.deleteTagIndex(tag);
    return { tag, invalidated: keys.length };
  }
}
```

工具策略示例：

```ts
const cachePolicies: Record<string, CachePolicy> = {
  search_docs: {
    ttlMs: 10 * 60_000,
    key: (_, args) => `search_docs:${hash(args)}`,
    tags: (args, result) => [
      "rag:index",
      ...result.documents.map((d: any) => `doc:${d.id}`),
    ],
  },

  get_user_context: {
    ttlMs: 30 * 60_000,
    key: (_, args: any) => `user_context:${args.userId}`,
    tags: (args: any) => [`user:${args.userId}`, `tenant:${args.tenantId}`],
    bypassWhen: (ctx) => ctx.risk === "high", // 高风险动作前不信缓存
  },
};
```

这套设计的关键点：

- 缓存 key 解决“同一个查询”；
- tags 解决“哪些变化会影响这个查询”；
- `bypassWhen` 解决“什么场景不能信缓存”。

---

## 5. 防缓存击穿：SingleFlight

标签失效后，大量 Agent 可能同时重新请求同一个昂贵工具，这叫 cache stampede。

解决办法：同一个 key 同时只允许一个请求真实执行，其它请求等待结果。

```ts
class SingleFlight {
  private inFlight = new Map<string, Promise<unknown>>();

  async run<T>(key: string, fn: () => Promise<T>): Promise<T> {
    const existing = this.inFlight.get(key);
    if (existing) return existing as Promise<T>;

    const promise = fn().finally(() => this.inFlight.delete(key));
    this.inFlight.set(key, promise);
    return promise;
  }
}
```

接入中间件：

```ts
const flight = new SingleFlight();

const result = await flight.run(key, async () => {
  const fresh = await next();
  await store.set(key, wrap(fresh));
  return fresh;
});
```

这样失效后的第一波流量不会把 DB / API 打爆。

---

## 6. OpenClaw 实战：Heartbeat / Cron 缓存状态

OpenClaw 这种 Always-on Agent 很适合用文件做轻量 tag cache：

```json
// memory/cache-state.json
{
  "entries": {
    "calendar:next24h": {
      "tags": ["user:boss", "calendar"],
      "observedAt": 1778034600,
      "expiresAt": 1778036400,
      "path": "memory/cache/calendar-next24h.json"
    }
  },
  "tagIndex": {
    "calendar": ["calendar:next24h"],
    "user:boss": ["calendar:next24h"]
  }
}
```

Heartbeat 查询日历时：

1. 先看 `expiresAt`；
2. 命中则读缓存文件；
3. 如果收到日历变更 webhook 或手动更新日程，执行 `invalidateTag("calendar")`；
4. 重大提醒（比如两小时内会议）发送前重新查一次 live calendar。

这就是“省 token、省 API 调用，但不拿旧数据做关键决策”。

---

## 7. Cache Policy Checklist

设计 Agent 缓存时，每个缓存都问 7 个问题：

1. **Key 是什么？** 同一请求如何稳定命中？
2. **TTL 多久？** 过短没收益，过长有风险。
3. **Tags 有哪些？** 哪些实体变化会让它变旧？
4. **谁负责 invalidate？** 写工具、webhook、cron，还是人工操作？
5. **高风险场景是否 bypass？** 副作用前是否必须 live check？
6. **是否防 stampede？** 失效后一堆请求会不会同时打爆下游？
7. **是否可观测？** hit/miss/invalidate/stale 都要有日志和指标。

---

## 8. 常见坑

### 坑 1：只按 query 缓存，不记录来源文档

RAG 搜索如果只缓存 `query -> answer`，文档更新后不知道哪些答案受影响。

正确做法：答案缓存带 `doc:{id}` tags。

### 坑 2：用户权限变化后不清上下文缓存

用户权限降级，但旧 `user_context` 还带高级权限信息，Agent 继续注入旧能力。

正确做法：权限变更 invalidate `user:{id}` 和 `tenant:{id}:permissions`。

### 坑 3：LLM 结果缓存没有 prompt/schema 版本

同一个问题在 prompt 改版后应该重新生成，否则旧行为继续污染线上。

正确做法：key 里加入 `promptHash` / `schemaVersion`。

```ts
const key = hash({ toolName, args, promptHash, schemaVersion });
```

---

## 总结

Agent 缓存的成熟度分四级：

1. **无缓存**：慢，但一致性简单；
2. **TTL 缓存**：快一点，但容易拿旧数据做错事；
3. **Tag-based invalidation**：实体变化时精准失效；
4. **Tag + SingleFlight + 高风险 live check**：既快又安全。

记住一句话：

> TTL 负责“时间到了自动忘”，Tag 负责“世界变了立刻忘”。成熟 Agent 不是记得越久越好，而是知道什么时候该忘。
