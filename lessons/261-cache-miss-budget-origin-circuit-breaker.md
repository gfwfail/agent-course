# 261. Agent 缓存回源预算与熔断保护（Cache Miss Budget & Origin Circuit Breaker）

上一课讲了多级缓存：L1 负责快，L2 负责共享，L3/source 负责权威。今天补一个生产里非常容易被忽略的问题：**缓存 miss 不是免费事件**。

一句话：

> Cache Miss Budget 是给回源请求设置预算、并发上限和熔断策略；当缓存大面积失效、外部 API 抖动或热点 key 未命中时，Agent 优先返回 stale/降级/排队，而不是把源站、数据库或第三方 API 打穿。

很多 Agent 系统一开始只关注 cache hit rate，忽略了 miss 的破坏力：

- prompt/schema 缓存同时过期，所有 worker 同时扫描文件；
- GitHub/邮箱/账单 API cache miss，瞬间触发几十个外部请求；
- 用户一句“查所有订单”，每个订单都 miss，DB 被打爆；
- L2 Redis 短暂不可用，所有请求直接落到 L3/source。

缓存不是万能盾牌。**真正要保护的是回源路径**。

---

## 1. 为什么 Agent 需要 Miss Budget

Agent 的回源比普通 Web 服务更危险，因为它经常是“模型驱动”的：

- LLM 可能拆出很多子查询；
- 工具调用结果会继续触发后续工具；
- cron/heartbeat/sub-agent 可以并发运行；
- 一个 cache tag 失效后，多个 run 都认为自己应该重新加载。

如果没有预算，缓存失效会变成放大器：

```text
1 个配置变更
  -> 100 个 cache key 失效
  -> 20 个 worker 同时 miss
  -> 2000 次数据库/API 回源
  -> 源站变慢
  -> 更多请求超时重试
  -> 雪崩
```

Miss Budget 的目标不是让所有请求都成功，而是在压力下做取舍：

1. 高优先级请求优先回源；
2. 低风险回答可以用 stale；
3. 低优先级批处理排队；
4. 源站异常时快速熔断，避免重试风暴；
5. 所有降级都带 caveat，不假装“实时”。

---

## 2. 设计原则：预算、并发、熔断、降级

### A. 每个 key class 都要有预算

不要所有缓存 miss 共用一个池。建议按用途分桶：

```text
tool_schema      -> 高优先级，小并发，短超时
user_profile     -> 中优先级，可 stale
external_api     -> 严格限流，失败熔断
billing_live     -> 高风险，不能 stale，但可以 ask/retry later
bulk_background  -> 低优先级，排队慢慢跑
```

预算字段可以包括：

- `maxConcurrent`：同时回源数；
- `tokensPerMinute`：每分钟 miss 次数；
- `timeoutMs`：回源超时；
- `staleMaxAgeSec`：允许 stale 的最长时间；
- `priority`：高优先级可抢占低优先级；
- `circuitBreaker`：失败率过高时暂停回源。

### B. 高风险动作不能用 stale，但也不能无限打源站

例如“删除资源前检查状态”：

- 不能直接用 10 分钟前的缓存；
- 但如果源站熔断，也不应该强行继续；
- 正确行为是阻断副作用，并告诉用户“无法确认当前状态”。

也就是：

```text
answer/report    -> 可以 stale + 标注时间
planning         -> 可以 stale，但副作用前复核
external write   -> stale 不允许；live check 失败则 abort/ask
```

### C. 熔断器保护的是源站，不是缓存

当回源连续失败、超时或 429：

- `closed`：正常；
- `open`：短时间直接拒绝回源，返回 stale/降级；
- `half_open`：放少量探测请求，成功后恢复。

重点：open 状态不是“系统坏了”，而是系统在保护自己。

---

## 3. learn-claude-code：Python 教学版

下面是一个最小 Miss Budget。它用 token bucket 控制每分钟 miss，用 semaphore 控制并发，用 circuit breaker 控制失败后的暂停。

```python
# learn_claude_code/cache_miss_budget.py
from __future__ import annotations

import asyncio
import time
from dataclasses import dataclass
from typing import Any, Awaitable, Callable

@dataclass
class CacheEntry:
    value: Any
    expires_at: float
    observed_at: float

    def fresh(self) -> bool:
        return time.time() < self.expires_at

    def age(self) -> float:
        return time.time() - self.observed_at

class CircuitBreaker:
    def __init__(self, *, failure_threshold: int, reset_after: float) -> None:
        self.failure_threshold = failure_threshold
        self.reset_after = reset_after
        self.failures = 0
        self.opened_at: float | None = None

    def allow(self) -> bool:
        if self.opened_at is None:
            return True
        if time.time() - self.opened_at > self.reset_after:
            # half-open：允许一次探测
            return True
        return False

    def record_success(self) -> None:
        self.failures = 0
        self.opened_at = None

    def record_failure(self) -> None:
        self.failures += 1
        if self.failures >= self.failure_threshold:
            self.opened_at = time.time()

class MissBudget:
    def __init__(self, *, max_concurrent: int, tokens_per_minute: int) -> None:
        self.sem = asyncio.Semaphore(max_concurrent)
        self.tokens_per_minute = tokens_per_minute
        self.tokens = tokens_per_minute
        self.refilled_at = time.time()

    def _refill(self) -> None:
        now = time.time()
        elapsed = now - self.refilled_at
        if elapsed >= 60:
            self.tokens = self.tokens_per_minute
            self.refilled_at = now

    async def acquire(self) -> bool:
        self._refill()
        if self.tokens <= 0:
            return False
        self.tokens -= 1
        await self.sem.acquire()
        return True

    def release(self) -> None:
        self.sem.release()

class BudgetedCache:
    def __init__(self) -> None:
        self.cache: dict[str, CacheEntry] = {}
        self.budgets = {
            "external_api": MissBudget(max_concurrent=3, tokens_per_minute=30),
            "tool_schema": MissBudget(max_concurrent=1, tokens_per_minute=10),
        }
        self.breakers = {
            "external_api": CircuitBreaker(failure_threshold=3, reset_after=30),
            "tool_schema": CircuitBreaker(failure_threshold=2, reset_after=10),
        }

    async def get_or_load(
        self,
        key: str,
        key_class: str,
        loader: Callable[[], Awaitable[Any]],
        *,
        ttl: float,
        allow_stale: bool,
        stale_max_age: float,
    ) -> tuple[Any, str]:
        entry = self.cache.get(key)
        if entry and entry.fresh():
            return entry.value, "fresh"

        breaker = self.breakers[key_class]
        if not breaker.allow():
            if allow_stale and entry and entry.age() <= stale_max_age:
                return entry.value, "stale:circuit_open"
            raise RuntimeError(f"origin circuit open for {key_class}")

        budget = self.budgets[key_class]
        acquired = await budget.acquire()
        if not acquired:
            if allow_stale and entry and entry.age() <= stale_max_age:
                return entry.value, "stale:miss_budget_exhausted"
            raise RuntimeError(f"miss budget exhausted for {key_class}")

        try:
            value = await loader()
            self.cache[key] = CacheEntry(value=value, expires_at=time.time() + ttl, observed_at=time.time())
            breaker.record_success()
            return value, "reloaded"
        except Exception:
            breaker.record_failure()
            if allow_stale and entry and entry.age() <= stale_max_age:
                return entry.value, "stale:loader_failed"
            raise
        finally:
            budget.release()
```

关键点不是代码多复杂，而是 `get_or_load` 不再只有 hit/miss 两种结果，而是有：

- `fresh`：正常命中；
- `reloaded`：预算允许，成功回源；
- `stale:...`：降级命中，并说明原因；
- exception：不能安全回答或不能安全执行。

---

## 4. pi-mono：TypeScript 生产版中间件

生产版建议把 Miss Budget 做成缓存中间件，而不是散落在每个工具里。

```ts
// pi-mono/cache/miss-budget.ts
export type CacheUse = "answer" | "planning" | "external_side_effect" | "security_decision";

export interface MissBudgetPolicy {
  keyClass: string;
  maxConcurrent: number;
  tokensPerMinute: number;
  timeoutMs: number;
  staleMaxAgeMs: number;
  allowStaleFor: CacheUse[];
  circuit: {
    failureThreshold: number;
    resetAfterMs: number;
  };
}

export interface CacheDecision<T> {
  value?: T;
  status: "fresh" | "reloaded" | "stale" | "blocked";
  reason?: string;
  observedAt?: string;
  caveat?: string;
}

export class OriginCircuitBreaker {
  private failures = 0;
  private openedAt: number | null = null;

  constructor(private readonly threshold: number, private readonly resetAfterMs: number) {}

  allow(): boolean {
    if (this.openedAt === null) return true;
    return Date.now() - this.openedAt > this.resetAfterMs;
  }

  success() {
    this.failures = 0;
    this.openedAt = null;
  }

  failure() {
    this.failures += 1;
    if (this.failures >= this.threshold) this.openedAt = Date.now();
  }
}

export class MissBudgetMiddleware {
  private breakers = new Map<string, OriginCircuitBreaker>();

  constructor(
    private readonly policies: Record<string, MissBudgetPolicy>,
    private readonly limiter: { acquire(key: string): Promise<() => void> },
  ) {}

  private breakerFor(policy: MissBudgetPolicy) {
    const existing = this.breakers.get(policy.keyClass);
    if (existing) return existing;
    const breaker = new OriginCircuitBreaker(policy.circuit.failureThreshold, policy.circuit.resetAfterMs);
    this.breakers.set(policy.keyClass, breaker);
    return breaker;
  }

  async getOrLoad<T>(args: {
    key: string;
    keyClass: string;
    use: CacheUse;
    read: () => Promise<{ value: T; fresh: boolean; observedAt: Date } | null>;
    load: () => Promise<T>;
    write: (value: T) => Promise<void>;
  }): Promise<CacheDecision<T>> {
    const policy = this.policies[args.keyClass];
    const cached = await args.read();

    if (cached?.fresh) {
      return { value: cached.value, status: "fresh", observedAt: cached.observedAt.toISOString() };
    }

    const allowStale = policy.allowStaleFor.includes(args.use);
    const staleOk = cached && Date.now() - cached.observedAt.getTime() <= policy.staleMaxAgeMs;
    const breaker = this.breakerFor(policy);

    if (!breaker.allow()) {
      if (allowStale && staleOk) {
        return {
          value: cached.value,
          status: "stale",
          reason: "origin_circuit_open",
          observedAt: cached.observedAt.toISOString(),
          caveat: "源站暂时熔断，结果不是实时数据",
        };
      }
      return { status: "blocked", reason: "origin_circuit_open" };
    }

    let release: (() => void) | undefined;
    try {
      release = await this.limiter.acquire(`cache-miss:${args.keyClass}`);
      const value = await withTimeout(args.load(), policy.timeoutMs);
      await args.write(value);
      breaker.success();
      return { value, status: "reloaded" };
    } catch (error) {
      breaker.failure();
      if (allowStale && staleOk) {
        return {
          value: cached.value,
          status: "stale",
          reason: "origin_reload_failed",
          observedAt: cached.observedAt.toISOString(),
          caveat: "回源失败，已使用旧数据降级",
        };
      }
      return { status: "blocked", reason: "origin_reload_failed" };
    } finally {
      release?.();
    }
  }
}

async function withTimeout<T>(promise: Promise<T>, timeoutMs: number): Promise<T> {
  return await Promise.race([
    promise,
    new Promise<T>((_, reject) => setTimeout(() => reject(new Error("origin timeout")), timeoutMs)),
  ]);
}
```

这里最重要的字段是 `use`。同一个缓存值，在不同用途下策略不同：

```ts
const profileForAnswer = await cache.getOrLoad({
  key: "tenant:t1:user:42:profile:v3",
  keyClass: "user_profile",
  use: "answer",          // 可以 stale，但要标注
  read, load, write,
});

const profileBeforeDelete = await cache.getOrLoad({
  key: "tenant:t1:user:42:profile:v3",
  keyClass: "user_profile",
  use: "external_side_effect", // stale 不允许；失败就 blocked
  read, load, write,
});
```

这就是成熟 Agent 的差异：不是“缓存命中就用”，而是“按用途判断这个数据能不能支撑下一步动作”。

---

## 5. OpenClaw 实战：课程 Cron 里的 Miss Budget

以这个 Agent 课程 cron 为例，它每 3 小时会做：

1. 读 TOOLS.md，检查已讲内容；
2. 读 lessons/README，决定下一课；
3. 写 lesson；
4. 发 Telegram；
5. git commit/push。

哪些地方适合缓存？

- `README.md` 目录解析：可以缓存；
- `TOOLS.md 已讲内容`：可以缓存，但写入前要重新读取；
- Telegram message id：不能靠缓存猜，要用 send receipt；
- git remote state：push 前必须 live check；
- GitHub PR/branch 状态：push 前 live check，不能 stale。

如果 GitHub API 或 git remote 抖动：

```text
课程内容生成可以继续写本地文件
但 git push 不能靠旧状态继续执行
应标记 [blocked]：无法确认远端状态
```

如果 Telegram send 成功但后续 git push 失败：

- Outbox/日志保存 messageId；
- 下次恢复时不要重复发群；
- 只补 git commit/push 或修正 repo 状态。

这和 Miss Budget 是一套思想：**低风险步骤可降级，高风险副作用必须确认现实。**

---

## 6. 常见坑

### 坑 1：缓存 miss 时无限 retry

错误做法：

```text
cache miss -> loader timeout -> retry 3 次 -> 每个 worker 都 retry
```

正确做法：

- retry 前先看 budget；
- 429/5xx 进入 breaker；
- 有 stale 就返回 stale；
- 没 stale 且用途高风险就 blocked。

### 坑 2：把 stale 当 fresh

降级不是作弊，必须显式告诉上游：

```json
{
  "status": "stale",
  "observedAt": "2026-05-07T08:12:00Z",
  "caveat": "源站熔断，结果不是实时数据"
}
```

否则 LLM 会把旧数据说得像刚查的一样。

### 坑 3：所有 key class 一个限流器

这样会出现“低价值批处理吃光预算，高价值安全检查没法回源”。预算必须按 key class/priority 隔离。

### 坑 4：熔断后没有探测恢复

breaker open 后要有 half-open 探测，否则系统会永远降级。探测请求也要小流量，不要一到 reset 时间全量放开。

---

## 7. 实战检查清单

给 Agent 缓存加 Miss Budget 时，检查这 8 项：

- [ ] 每个 cache key 有 `keyClass`；
- [ ] 每个 keyClass 有 `maxConcurrent/tokensPerMinute/timeout`；
- [ ] 外部 API/DB 回源有 circuit breaker；
- [ ] stale 返回必须带 `observedAt/caveat/reason`；
- [ ] `external_side_effect/security_decision` 默认不允许 stale；
- [ ] 429/5xx/timeout 会消耗失败计数并可能熔断；
- [ ] 低优先级批处理不会吃掉高优先级预算；
- [ ] metrics 记录 `miss_budget_exhausted/circuit_open/stale_served/origin_latency`。

---

## 8. 一句话总结

> 缓存命中让 Agent 快；Miss Budget 让缓存失效时 Agent 不崩。

成熟 Agent 不怕 cache miss，怕的是 miss 后无脑回源、无脑重试、无脑继续执行。把回源当成稀缺资源来治理，Agent 才能在 Redis 抖动、API 限流、批量失效时保持可控。