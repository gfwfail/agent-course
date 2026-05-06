# 253. Agent 缓存预热与热点键治理（Cache Warming & Hot Key Governance）

> 核心思想：缓存不是等用户打到系统上才开始建。生产 Agent 里，热门 Prompt、工具 Schema、用户画像、RAG 摘要、价格表这类高频数据，应该在流量到来前预热；热点 key 要被识别、限速、分片和降级。否则前两课做了失效和防击穿，仍然会在热点集中访问时被单个 key 拖垮。

前两课讲了两件事：

- 第 251 课：缓存失效要用 dependency tags 精准删除；
- 第 252 课：缓存 miss/过期后要用 SingleFlight + SWR 防击穿。

今天补第三块：**哪些 key 应该提前热起来，哪些 key 热到危险时应该被治理**。

一句话：不要只把缓存当“被动优化”，要把它当 Agent 的一等运行时资源：有温度、有优先级、有预算、有保护。

---

## 1. 为什么 Agent 系统更需要缓存预热

传统 Web 里，缓存预热通常是首页、商品详情、配置项。Agent 系统的热点更隐蔽：

- 系统 Prompt + Tool Schema：每个 run 都要加载；
- Skills / MCP tool list：工具多时序列化成本很高；
- 用户画像 / MEMORY 摘要：私聊和 heartbeat 会反复读取；
- RAG 文档摘要：热门文档会被多个会话同时问；
- 外部 API 基础数据：国家列表、价格表、库存、汇率；
- Cron 结果：定时任务唤醒时多个 Agent 会查同一批状态。

如果这些 key 冷启动时才计算，用户看到的不是“智能 Agent”，而是“请稍等，我在加载世界”。

更糟的是：热门 key 一旦 miss，背后可能不是一次数据库查询，而是一次 LLM 总结、embedding、API 聚合，成本和延迟都很大。

---

## 2. 三层策略：Warm、Detect、Protect

生产建议把热点缓存治理拆成三层。

### 2.1 Warm：提前预热

预热来源通常有四种：

1. **启动预热**：进程起来先加载静态 system prompt、tool schema、配置；
2. **定时预热**：Cron 每隔几分钟刷新热门数据；
3. **预测预热**：某个事件发生后，提前准备下一步会用的数据；
4. **发布预热**：新版本上线前先构建 prompt cache / schema cache，避免发布后第一批用户踩冷缓存。

### 2.2 Detect：识别热点 key

缓存系统要记录每个 key 的访问次数、miss 次数、重建耗时、错误率。

一个 key 变成热点的典型信号：

```txt
requests_per_minute > 100
miss_rate > 5%
rebuild_p95_ms > 1000
inflight_waiters > 20
```

这些指标一旦超过阈值，就不能继续当普通缓存处理。

### 2.3 Protect：热点保护

热点 key 的保护手段：

- **更长 soft TTL**：允许更久 stale，减少重建频率；
- **提前刷新**：剩余 TTL 低于阈值就后台刷新；
- **分片 key**：超大结果拆成多个 shard，避免单 key 巨大；
- **只读降级**：刷新失败时返回旧值并标注 caveat；
- **重建限流**：同一类 key 同时最多 N 个 refresh；
- **优先级队列**：高价值 key 先刷新，低价值 key 延后。

---

## 3. learn-claude-code：Python 教学版

下面是一个最小的热点统计 + 预热器。重点不是 Redis 细节，而是治理模型：缓存 key 要有 profile，预热任务要按热度排序。

```python
# cache_warmer.py
import asyncio
import time
from dataclasses import dataclass, field
from typing import Awaitable, Callable, Any

Loader = Callable[[], Awaitable[Any]]

@dataclass
class KeyProfile:
    key: str
    hits: int = 0
    misses: int = 0
    total_rebuild_ms: float = 0
    last_access_at: float = 0
    last_warmed_at: float = 0
    errors: int = 0

    @property
    def requests(self) -> int:
        return self.hits + self.misses

    @property
    def miss_rate(self) -> float:
        if self.requests == 0:
            return 0
        return self.misses / self.requests

    @property
    def avg_rebuild_ms(self) -> float:
        if self.misses == 0:
            return 0
        return self.total_rebuild_ms / self.misses

    def heat_score(self) -> float:
        # 访问量越高、miss 越高、重建越贵，越值得预热
        freshness_penalty = min(time.time() - self.last_warmed_at, 3600) / 3600
        return (
            self.requests * 1.0
            + self.miss_rate * 100
            + min(self.avg_rebuild_ms / 100, 50)
            + freshness_penalty * 20
        )

@dataclass
class WarmTarget:
    key: str
    loader: Loader
    min_interval_sec: int = 300
    priority: int = 0

class CacheWarmer:
    def __init__(self, cache):
        self.cache = cache
        self.profiles: dict[str, KeyProfile] = {}
        self.targets: dict[str, WarmTarget] = {}
        self.refresh_limit = asyncio.Semaphore(3)

    def register(self, target: WarmTarget):
        self.targets[target.key] = target
        self.profiles.setdefault(target.key, KeyProfile(key=target.key))

    def record_hit(self, key: str):
        profile = self.profiles.setdefault(key, KeyProfile(key=key))
        profile.hits += 1
        profile.last_access_at = time.time()

    def record_miss(self, key: str, rebuild_ms: float, error: bool = False):
        profile = self.profiles.setdefault(key, KeyProfile(key=key))
        profile.misses += 1
        profile.total_rebuild_ms += rebuild_ms
        profile.last_access_at = time.time()
        if error:
            profile.errors += 1

    async def warm_once(self, target: WarmTarget):
        profile = self.profiles.setdefault(target.key, KeyProfile(key=target.key))
        now = time.time()
        if now - profile.last_warmed_at < target.min_interval_sec:
            return

        async with self.refresh_limit:
            started = time.time()
            try:
                value = await target.loader()
                await self.cache.set(target.key, value, ttl=900)
                profile.last_warmed_at = time.time()
                profile.total_rebuild_ms += (time.time() - started) * 1000
            except Exception:
                profile.errors += 1
                # 预热失败不应该打断主流程，只记录，交给监控/下轮重试

    async def warm_hot_keys(self, limit: int = 20):
        ranked = sorted(
            self.targets.values(),
            key=lambda t: (t.priority, self.profiles[t.key].heat_score()),
            reverse=True,
        )
        await asyncio.gather(*(self.warm_once(t) for t in ranked[:limit]))
```

用法：

```python
warmer.register(WarmTarget(
    key="tool-schema:default",
    loader=load_tool_schema,
    min_interval_sec=600,
    priority=10,
))

warmer.register(WarmTarget(
    key="pricing:countries:v1",
    loader=load_country_pricing,
    min_interval_sec=300,
    priority=8,
))

# cron / heartbeat 定期跑
await warmer.warm_hot_keys(limit=10)
```

教学版要点：

- profile 和 target 分开：不是所有访问过的 key 都值得预热；
- 预热失败只记录，不要影响用户请求；
- `refresh_limit` 必须有，否则预热本身会变成压力源；
- heat_score 不是玄学，要由访问量、miss、重建成本共同决定。

---

## 4. pi-mono：TypeScript 生产版中间件

生产里可以把热点治理做成 `CacheGovernanceMiddleware`，透明包住 cache get/set。

```ts
type CacheKeyProfile = {
  key: string;
  hits: number;
  misses: number;
  rebuildMsTotal: number;
  inflightWaiters: number;
  lastAccessAt: number;
  lastWarmAt?: number;
};

type HotKeyPolicy = {
  hotRequestThreshold: number;
  hotMissRateThreshold: number;
  extendSoftTtlMs: number;
  refreshAheadMs: number;
  maxConcurrentRefresh: number;
};

export class CacheGovernanceMiddleware {
  private profiles = new Map<string, CacheKeyProfile>();
  private refreshQueue = new Set<string>();

  constructor(
    private cache: CacheStore,
    private loaders: Map<string, () => Promise<unknown>>,
    private policy: HotKeyPolicy,
  ) {}

  async get<T>(key: string): Promise<T | null> {
    const profile = this.profile(key);
    profile.lastAccessAt = Date.now();

    const entry = await this.cache.getWithMeta<T>(key);
    if (!entry) {
      profile.misses++;
      return null;
    }

    profile.hits++;

    if (this.shouldRefreshAhead(key, entry.expiresAt)) {
      this.enqueueRefresh(key);
    }

    return entry.value;
  }

  isHot(key: string): boolean {
    const p = this.profile(key);
    const requests = p.hits + p.misses;
    const missRate = requests === 0 ? 0 : p.misses / requests;

    return (
      requests >= this.policy.hotRequestThreshold ||
      missRate >= this.policy.hotMissRateThreshold ||
      p.inflightWaiters > 20
    );
  }

  ttlFor(key: string, baseTtlMs: number): number {
    if (!this.isHot(key)) return baseTtlMs;
    // 热点 key：更长 TTL + jitter，优先保证稳定
    const extended = baseTtlMs + this.policy.extendSoftTtlMs;
    const jitter = extended * (Math.random() * 0.2 - 0.1);
    return Math.round(extended + jitter);
  }

  private shouldRefreshAhead(key: string, expiresAt: number): boolean {
    if (!this.isHot(key)) return false;
    return expiresAt - Date.now() < this.policy.refreshAheadMs;
  }

  private enqueueRefresh(key: string) {
    if (this.refreshQueue.has(key)) return;
    const loader = this.loaders.get(key);
    if (!loader) return;

    this.refreshQueue.add(key);
    queueMicrotask(async () => {
      const started = Date.now();
      try {
        const value = await loader();
        await this.cache.set(key, value, {
          ttlMs: this.ttlFor(key, 5 * 60_000),
          tags: [`cache-key:${key}`],
        });
        this.profile(key).lastWarmAt = Date.now();
      } finally {
        this.profile(key).rebuildMsTotal += Date.now() - started;
        this.refreshQueue.delete(key);
      }
    });
  }

  private profile(key: string): CacheKeyProfile {
    let p = this.profiles.get(key);
    if (!p) {
      p = {
        key,
        hits: 0,
        misses: 0,
        rebuildMsTotal: 0,
        inflightWaiters: 0,
        lastAccessAt: 0,
      };
      this.profiles.set(key, p);
    }
    return p;
  }
}
```

在 pi-mono 风格里，它应该接在这几层之间：

```txt
Agent Loop
  -> Tool Router
  -> CacheGovernanceMiddleware
  -> SWR / SingleFlight Cache
  -> Real Tool / External API / LLM
```

关键原则：

- LLM 不需要知道某个 key 是热点；
- 工具层返回的仍是正常 `ToolResult<T>`；
- 缓存治理是基础设施能力，不要散落在业务工具里。

---

## 5. OpenClaw 实战：Heartbeat/Cron 预热

OpenClaw 这种 always-on Agent 特别适合做轻量预热，因为 heartbeat 和 cron 本来就会定期醒来。

可以维护一个小文件：

```json
// memory/cache-warm-state.json
{
  "lastWarmAt": {
    "agent-course-readme": 1778044200,
    "tools-md-topic-list": 1778044200,
    "github-pr-status": 1778043900
  },
  "hotKeys": [
    "agent-course:readme",
    "agent-course:tools-topic-list",
    "openclaw:channel-capabilities"
  ]
}
```

Heartbeat 里做三件事：

1. 如果距离上次预热超过阈值，读取 README/TOOLS 的课程目录；
2. 只缓存摘要或指纹，不缓存完整大文件；
3. 预热失败静默记录，不要吵用户。

伪代码：

```python
async def heartbeat_cache_warm():
    state = read_json("memory/cache-warm-state.json")
    now = time.time()

    for key in state["hotKeys"]:
        if now - state["lastWarmAt"].get(key, 0) < 900:
            continue

        try:
            value = await build_small_summary(key)
            write_json(f"memory/cache/{key}.json", {
                "key": key,
                "value": value,
                "observedAt": now,
                "maxAgeSec": 1800,
            })
            state["lastWarmAt"][key] = now
        except Exception as exc:
            append_log("memory/cache-warm-errors.jsonl", {"key": key, "error": str(exc)})

    write_json("memory/cache-warm-state.json", state)
```

注意：OpenClaw 里不要为了预热而频繁读大文件、打外部 API。预热也是成本，必须有间隔、有上限、有噪声控制。

---

## 6. 什么时候不要预热

不是所有缓存都值得预热。下面这些不要预热：

- 强隐私数据：用户没发起请求就别主动拉；
- 一次性任务结果：热不起来，预热浪费；
- 高风险实时数据：比如余额、库存、支付状态，副作用前必须 live check；
- 成本高但访问概率低的 LLM 生成物；
- 未授权资源：没有 actor/scope 的预热很容易越权。

预热的边界：**只预热安全、共享、可过期、可降级的数据**。

---

## 7. 热点 Key 的观测指标

建议至少打这些指标：

```txt
agent_cache_hit_total{key_class}
agent_cache_miss_total{key_class}
agent_cache_rebuild_duration_ms{key_class}
agent_cache_inflight_waiters{key_class}
agent_cache_warm_total{key_class,status}
agent_cache_hot_key_count{key_class}
agent_cache_stale_served_total{key_class}
```

不要直接把完整 key 打进 metrics label，容易造成高基数灾难。用 `key_class`：

```txt
tool-schema
user-profile
rag-summary
pricing
course-index
```

完整 key 可以进结构化日志，但要脱敏。

---

## 8. 常见坑

### 坑 1：预热任务没有限流

预热本来是为了保护系统，结果 cron 一醒来同时刷 1000 个 key，把数据库和 LLM 打爆。

正确做法：预热队列必须有并发上限和预算。

### 坑 2：只按访问次数判断热点

访问 1000 次但重建只要 1ms，不一定重要；访问 20 次但每次重建要 30s，反而应该优先预热。

正确做法：heat score = 访问量 + miss率 + 重建成本 + 业务优先级。

### 坑 3：预热越权

后台预热没有用户上下文，却把某个用户的私有数据缓存成共享 key。

正确做法：key 必须包含 tenant/actor/scope，或者只预热公共数据。

### 坑 4：热点 key 永不过期

为了稳定把 TTL 拉到无限，最后数据陈旧到影响决策。

正确做法：热点 key 可以延长 soft TTL，但 hard TTL 和 high-risk live check 不能取消。

---

## 9. 和前两课的组合关系

三课放在一起，就是完整缓存治理闭环：

```txt
第 251 课：变更发生 -> dependency tags 精准失效
第 252 课：失效/过期 -> SingleFlight + SWR 防击穿
第 253 课：热点出现 -> 预热 + 热点治理 + 观测保护
```

成熟 Agent 的缓存系统不是一个 `Map<string, any>`，而是一套运行时调度系统：知道什么该忘、什么时候重建、谁值得提前准备、热点来了怎么保护。

---

## 10. 落地 Checklist

- [ ] 给缓存 key 加 `key_class`，避免 metrics 高基数；
- [ ] 记录 hit/miss/rebuild_ms/inflight_waiters；
- [ ] 为公共高频数据注册 WarmTarget；
- [ ] 预热任务设置并发上限和预算；
- [ ] 热点 key 启用 refresh-ahead；
- [ ] 热点 key 可延长 soft TTL，但保留 hard TTL；
- [ ] 私有数据 key 必须包含 actor/tenant/scope；
- [ ] 预热失败不影响主流程，只进入日志/告警；
- [ ] 副作用前仍然 live check，不信任 stale cache。

一句话总结：**缓存预热是让 Agent 提前准备，热点治理是防止 Agent 被自己最常用的知识拖垮。**
