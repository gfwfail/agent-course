# 226. Agent 数据新鲜度与过期管理（Data Freshness & Staleness Management）

> 关键词：freshness、staleness、TTL、evidence age、cache invalidation、real-time context

上一课讲了 **配置漂移检测与自动修复**：Agent 要持续判断现实世界是否还符合 desired state。

今天讲一个更容易被忽略、但生产里非常致命的问题：**数据新鲜度与过期管理**。

很多 Agent 出错不是因为模型不会推理，而是因为它拿着旧数据做了一个看似合理的决定：

```txt
9:00 查到库存还有 5 个
9:08 用户下单前 Agent 仍引用 9:00 的库存
9:09 真实库存已经变成 0
结果：Agent 给出了错误承诺
```

一句话：**Agent 的上下文不是事实本身，只是某个时间点的观察。观察必须带时间戳、TTL 和过期策略。**

---

## 1. 为什么 Agent 需要 Freshness 管理？

传统软件里，缓存过期通常是工程层问题；但 Agent 系统里，缓存会进入 LLM 上下文，变成“推理依据”。

这会产生几个新问题：

1. LLM 不天然知道工具结果多久前拿的；
2. 多轮对话中旧结果会一直留在上下文；
3. Sub-agent 可能把旧观察当成新结论传给主 Agent；
4. Heartbeat / Cron 周期任务很容易复用上一轮状态；
5. 用户问“现在/最新/刚刚”时，旧缓存必须强制失效；
6. 一些数据允许旧一点，一些数据必须秒级新鲜。

典型例子：

| 数据类型 | 可接受新鲜度 |
|---|---:|
| 天气预报 | 30-60 分钟 |
| 日历 upcoming events | 5-15 分钟 |
| 股票/币价 | 秒级到 1 分钟 |
| 支付状态 | 必须实时或 webhook 确认 |
| GitHub PR 状态 | 1-5 分钟 |
| 文档内容 | 可以按版本号缓存 |
| 用户偏好 | 可以长期缓存，但要可更新 |

所以生产 Agent 需要一个统一模型：**每条观察都要有 freshness metadata**。

---

## 2. 核心模型：Observation 不是裸数据

不要把工具结果直接塞进 messages：

```json
{
  "status": "paid"
}
```

应该包装成带新鲜度的观察：

```ts
type Observation<T> = {
  data: T;
  source: string;
  observedAt: string;
  expiresAt?: string;
  maxAgeMs: number;
  version?: string;
  stalePolicy: "allow" | "warn" | "refresh" | "block";
};
```

字段含义：

- `observedAt`：什么时候观察到的；
- `expiresAt/maxAgeMs`：多久后不可信；
- `version`：如果外部系统有版本号，优先用版本判断；
- `stalePolicy`：过期后怎么处理：
  - `allow`：允许使用旧数据；
  - `warn`：可用但必须提示“数据可能过期”；
  - `refresh`：回答前自动重新拉取；
  - `block`：过期就禁止继续执行。

关键思想：**让 freshness 从“隐含直觉”变成“显式契约”。**

---

## 3. learn-claude-code：Python 教学版

先写一个最小 FreshnessGuard。

```python
# learn-claude-code/freshness.py
from dataclasses import dataclass
from datetime import datetime, timedelta, timezone
from typing import Any, Literal

StalePolicy = Literal["allow", "warn", "refresh", "block"]


@dataclass
class Observation:
    data: Any
    source: str
    observed_at: datetime
    max_age_seconds: int
    stale_policy: StalePolicy = "refresh"
    version: str | None = None

    @property
    def age_seconds(self) -> float:
        return (datetime.now(timezone.utc) - self.observed_at).total_seconds()

    @property
    def is_stale(self) -> bool:
        return self.age_seconds > self.max_age_seconds


@dataclass
class FreshnessDecision:
    action: Literal["use", "use_with_warning", "refresh", "block"]
    reason: str
    age_seconds: float


class FreshnessGuard:
    def decide(self, obs: Observation, intent: str) -> FreshnessDecision:
        # 用户明确要“最新/现在/实时”，收紧策略
        real_time_intent = any(word in intent for word in ["now", "latest", "current", "实时", "现在", "最新"])

        if real_time_intent and obs.is_stale:
            return FreshnessDecision(
                action="refresh",
                reason="user requested current data and observation is stale",
                age_seconds=obs.age_seconds,
            )

        if not obs.is_stale:
            return FreshnessDecision("use", "observation is fresh", obs.age_seconds)

        if obs.stale_policy == "allow":
            return FreshnessDecision("use", "stale data allowed by policy", obs.age_seconds)

        if obs.stale_policy == "warn":
            return FreshnessDecision("use_with_warning", "data is stale but still usable with warning", obs.age_seconds)

        if obs.stale_policy == "refresh":
            return FreshnessDecision("refresh", "data exceeded max age", obs.age_seconds)

        return FreshnessDecision("block", "stale data cannot be used safely", obs.age_seconds)
```

Agent Loop 使用方式：

```python
# learn-claude-code/agent_loop.py
async def answer_with_observation(intent: str, observation: Observation, tools):
    guard = FreshnessGuard()
    decision = guard.decide(observation, intent)

    if decision.action == "refresh":
        fresh_data = await tools.call(observation.source, {"refresh": True})
        observation = Observation(
            data=fresh_data,
            source=observation.source,
            observed_at=datetime.now(timezone.utc),
            max_age_seconds=observation.max_age_seconds,
            stale_policy=observation.stale_policy,
        )

    elif decision.action == "block":
        return {
            "status": "blocked",
            "message": "这条数据已经过期，且该场景不允许使用旧数据。请重新查询后再执行。",
        }

    warning = None
    if decision.action == "use_with_warning":
        warning = f"注意：数据约 {int(decision.age_seconds)} 秒前获取，可能不是最新。"

    return {
        "status": "ok",
        "warning": warning,
        "data": observation.data,
        "freshness": {
            "source": observation.source,
            "observedAt": observation.observed_at.isoformat(),
            "ageSeconds": int(observation.age_seconds),
        },
    }
```

教学版重点不是复杂，而是建立一个习惯：**任何会影响决策的数据，都不要裸奔进入上下文。**

---

## 4. pi-mono：生产版 FreshnessPolicy

生产系统里，freshness 应该是工具元数据的一部分，而不是每个业务函数手写判断。

```ts
// pi-mono/tooling/freshness.ts
export type StalePolicy = "allow" | "warn" | "refresh" | "block";

export type FreshnessPolicy = {
  maxAgeMs: number;
  stalePolicy: StalePolicy;
  tightenOnRealtimeIntent?: boolean;
  versionField?: string;
};

export type ToolResult<T> = {
  ok: boolean;
  data?: T;
  error?: string;
  meta: {
    source: string;
    observedAt: string;
    freshness?: FreshnessPolicy;
    cacheKey?: string;
    version?: string;
  };
};
```

工具注册时声明新鲜度：

```ts
// pi-mono/tools/github.ts
registry.register({
  name: "github.getPullRequest",
  description: "Get current GitHub pull request status",
  freshness: {
    maxAgeMs: 60_000,
    stalePolicy: "refresh",
    tightenOnRealtimeIntent: true,
    versionField: "updated_at",
  },
  async execute({ owner, repo, number }) {
    const pr = await github.pulls.get({ owner, repo, pull_number: number });

    return {
      ok: true,
      data: pr.data,
      meta: {
        source: "github.getPullRequest",
        observedAt: new Date().toISOString(),
        version: pr.data.updated_at,
      },
    };
  },
});
```

然后在工具中间件统一处理：

```ts
// pi-mono/middleware/freshness-middleware.ts
export function freshnessMiddleware(cache: ToolCache): ToolMiddleware {
  return async (ctx, next) => {
    const policy = ctx.tool.freshness;
    if (!policy) return next(ctx);

    const cached = await cache.get(ctx.cacheKey);
    if (!cached) return next(ctx);

    const ageMs = Date.now() - Date.parse(cached.meta.observedAt);
    const realtime = /最新|现在|实时|current|latest|now/i.test(ctx.userIntent ?? "");
    const maxAgeMs = realtime && policy.tightenOnRealtimeIntent
      ? Math.min(policy.maxAgeMs, 10_000)
      : policy.maxAgeMs;

    if (ageMs <= maxAgeMs) {
      return {
        ...cached,
        meta: {
          ...cached.meta,
          cache: "hit",
          ageMs,
          fresh: true,
        },
      };
    }

    if (policy.stalePolicy === "allow") {
      return { ...cached, meta: { ...cached.meta, cache: "stale-hit", ageMs } };
    }

    if (policy.stalePolicy === "warn") {
      return {
        ...cached,
        meta: {
          ...cached.meta,
          cache: "stale-hit",
          ageMs,
          warning: `Data is ${ageMs}ms old and may be stale`,
        },
      };
    }

    if (policy.stalePolicy === "block") {
      return {
        ok: false,
        error: "STALE_DATA_BLOCKED",
        meta: {
          source: ctx.tool.name,
          observedAt: new Date().toISOString(),
          ageMs,
          suggestion: "Refresh the source before making this decision",
        },
      };
    }

    // refresh: 继续执行真实工具
    const fresh = await next(ctx);
    if (fresh.ok) await cache.set(ctx.cacheKey, fresh, policy.maxAgeMs);
    return fresh;
  };
}
```

这样 LLM 不需要自己猜“缓存还能不能用”，工具层会给它一个明确、结构化的结果。

---

## 5. OpenClaw 实战：Heartbeat / Cron 尤其需要 freshness

OpenClaw 里最容易出现旧数据的地方是主动任务：Heartbeat 和 Cron。

比如一个 Heartbeat 检查日历：

```json
{
  "calendar": {
    "lastCheckedAt": "2026-05-03T08:00:00+11:00",
    "nextEvents": ["10:00 meeting"],
    "maxAgeMinutes": 15
  }
}
```

如果 9:30 还拿这份状态判断“老板 10 点有没有会”，风险就很高。

更稳的写法是把状态拆成：

```json
{
  "calendar": {
    "observation": {
      "observedAt": "2026-05-03T09:25:00+11:00",
      "expiresAt": "2026-05-03T09:40:00+11:00",
      "source": "gog.calendar.list",
      "data": ["10:00 meeting"]
    },
    "policy": {
      "maxAgeMinutes": 15,
      "stalePolicy": "refresh"
    }
  }
}
```

OpenClaw 的经验法则：

- 邮件 unread：5-10 分钟内算新；
- Calendar upcoming：15 分钟内算新；
- 天气：30-60 分钟内算新；
- GitHub PR/CI：1-5 分钟内算新；
- 账单/成本：日报级别即可；
- 支付/订单状态：必须实时查或等 webhook。

如果用户问“最新 PR 状态”“现在账单有没有异常”“刚刚有没有邮件”，就不要复用上一轮结果，直接 refresh。

---

## 6. 常见坑

### 坑 1：只缓存 data，不缓存 observedAt

没有时间戳的缓存，本质上不可审计。

### 坑 2：把 TTL 当成全局配置

不同数据源新鲜度要求完全不同。天气和支付状态不能用同一个 TTL。

### 坑 3：LLM 负责判断缓存是否过期

LLM 可以理解策略，但不应该承担时间计算和安全决策。过期判断应该是 deterministic code。

### 坑 4：只在读取时检查，不在执行副作用前检查

执行外部副作用前要二次检查 freshness：

```txt
plan generated at 09:00
approval granted at 09:20
execute at 09:21
```

这时计划依据可能已经过期。高风险操作执行前必须 refresh critical observations。

### 坑 5：Sub-agent 汇报时丢失 freshness

子 Agent 不应该只回：

```txt
PR 已经 merge
```

而应该回：

```txt
PR #32 状态为 merged，观察时间 09:28，来源 GitHub API。
```

---

## 7. 设计 Checklist

给 Agent 加 Freshness 管理时，检查 8 件事：

1. 工具结果是否都有 `observedAt`？
2. 每类数据是否有独立 `maxAge`？
3. 过期策略是 `allow/warn/refresh/block` 哪一种？
4. 用户说“最新/现在/实时”时是否强制 refresh？
5. 副作用执行前是否检查 critical observations？
6. Sub-agent 交接是否保留观察时间和来源？
7. 回答用户时是否提示旧数据风险？
8. 日志里是否能追溯“基于哪一刻的数据做了决定”？

---

## 8. 一句话总结

**Agent 不应该把上下文里的旧观察当成永恒事实。**

生产级 Agent 要给每条关键数据加 `observedAt + maxAge + stalePolicy`：能用就用，过期就刷新，危险就阻断。

数据新鲜度管理做好了，Agent 才能从“会说”升级为“可信地行动”。
