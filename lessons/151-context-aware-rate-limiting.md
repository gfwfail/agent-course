# 151. Agent 上下文感知速率限制（Context-Aware Rate Limiting）

> 不是简单的 RPS 上限，而是根据工具类型、用户级别、系统负载、时间窗口动态调整每个工具的速率配额。

---

## 为什么需要「上下文感知」限速？

传统限流：`工具 X 每秒最多调用 10 次` — 简单粗暴，问题多：

| 问题 | 场景 |
|------|------|
| 高优任务被卡住 | VIP 用户触发同一限速桶 |
| 低峰期浪费配额 | 夜间 RPS=1 也被限速 |
| 工具类型忽略 | 读操作和写操作用同一配额 |
| 无法感知下游压力 | DB 已满载，Agent 仍狂发查询 |

**上下文感知限速**的核心：`limit = f(tool_type, user_tier, load, time)`

---

## 架构设计

```
┌──────────────────────────────────────────────┐
│              Tool Middleware                  │
│                                               │
│  invoke(tool, args, ctx)                      │
│      ↓                                        │
│  RateLimitEngine.acquire(tool, ctx)           │
│      ↓                                        │
│  ┌─────────────────────────────────┐          │
│  │  Context Evaluator              │          │
│  │  • user_tier (free/pro/vip)     │          │
│  │  • tool_category (read/write)   │          │
│  │  • system_load (0.0~1.0)        │          │
│  │  • time_window (peak/off-peak)  │          │
│  └────────────┬────────────────────┘          │
│               ↓                               │
│  ┌─────────────────────────────────┐          │
│  │  Dynamic Quota Calculator       │          │
│  │  base_rps × tier_factor         │          │
│  │       × load_factor             │          │
│  │       × time_factor             │          │
│  └────────────┬────────────────────┘          │
│               ↓                               │
│  ┌─────────────────────────────────┐          │
│  │  Token Bucket (per composite    │          │
│  │  key: tool+user_tier+window)    │          │
│  └─────────────────────────────────┘          │
└──────────────────────────────────────────────┘
```

---

## TypeScript 实现

### 1. 核心数据结构

```typescript
// rate-limit-engine.ts

interface RateLimitContext {
  userId: string;
  userTier: 'free' | 'pro' | 'vip';
  systemLoad: number;     // 0.0 ~ 1.0，来自监控
  currentHour: number;    // 0-23，判断峰谷
}

interface ToolRateConfig {
  category: 'read' | 'write' | 'external' | 'compute';
  baseRps: number;        // 基准 RPS（pro 级别、正常负载、平峰）
}

// 工具速率配置表
const TOOL_CONFIGS: Record<string, ToolRateConfig> = {
  'database_query':    { category: 'read',     baseRps: 20 },
  'database_write':    { category: 'write',    baseRps: 5  },
  'web_search':        { category: 'external', baseRps: 3  },
  'send_email':        { category: 'external', baseRps: 1  },
  'llm_summarize':     { category: 'compute',  baseRps: 2  },
  'file_read':         { category: 'read',     baseRps: 50 },
  'file_write':        { category: 'write',    baseRps: 10 },
};
```

### 2. 动态配额计算

```typescript
// 三个维度的调节因子
function calcDynamicRps(
  toolName: string,
  ctx: RateLimitContext
): number {
  const config = TOOL_CONFIGS[toolName] ?? { category: 'read', baseRps: 10 };

  // 因子 1: 用户等级
  const tierFactor: Record<string, number> = {
    free: 0.3,
    pro: 1.0,
    vip: 3.0,
  };

  // 因子 2: 系统负载（负载越高，限速越严）
  // load=0.0 → factor=1.5（空闲时放宽）
  // load=0.5 → factor=1.0（正常）
  // load=0.9 → factor=0.2（高压时大幅收紧）
  const loadFactor = ctx.systemLoad < 0.5
    ? 1.5 - ctx.systemLoad
    : Math.max(0.1, 1.0 - (ctx.systemLoad - 0.5) * 1.6);

  // 因子 3: 时间窗口（工作时间 9-18 为峰值）
  const isPeak = ctx.currentHour >= 9 && ctx.currentHour < 18;
  const timeFactor = isPeak ? 1.0 : 1.5; // 低峰期放宽 50%

  // 写操作额外收紧
  const categoryFactor = config.category === 'write' ? 0.5
    : config.category === 'external' ? 0.7
    : 1.0;

  const dynamicRps = config.baseRps
    * tierFactor[ctx.userTier]
    * loadFactor
    * timeFactor
    * categoryFactor;

  return Math.max(0.1, dynamicRps); // 最低 0.1 rps，永不归零
}
```

### 3. 令牌桶实现（内存版）

```typescript
interface TokenBucket {
  tokens: number;
  capacity: number;
  lastRefillMs: number;
  rps: number;
}

class RateLimitEngine {
  private buckets = new Map<string, TokenBucket>();

  // 组合键：tool + userTier，同一工具 VIP 有独立桶
  private bucketKey(toolName: string, tier: string) {
    return `${toolName}:${tier}`;
  }

  async acquire(toolName: string, ctx: RateLimitContext): Promise<void> {
    const rps = calcDynamicRps(toolName, ctx);
    const key = this.bucketKey(toolName, ctx.userTier);

    let bucket = this.buckets.get(key);
    if (!bucket) {
      bucket = { tokens: rps, capacity: rps * 2, lastRefillMs: Date.now(), rps };
      this.buckets.set(key, bucket);
    }

    // 动态更新 rps（负载/时间变化时自动生效）
    bucket.rps = rps;
    bucket.capacity = rps * 2;

    // 补充令牌
    const now = Date.now();
    const elapsed = (now - bucket.lastRefillMs) / 1000;
    bucket.tokens = Math.min(bucket.capacity, bucket.tokens + elapsed * bucket.rps);
    bucket.lastRefillMs = now;

    if (bucket.tokens >= 1) {
      bucket.tokens -= 1;
      return; // 立即放行
    }

    // 等待到令牌可用
    const waitMs = ((1 - bucket.tokens) / bucket.rps) * 1000;
    await new Promise(resolve => setTimeout(resolve, waitMs));
    bucket.tokens = 0;
  }

  // 查看当前各桶状态（调试用）
  status() {
    const result: Record<string, any> = {};
    for (const [key, b] of this.buckets) {
      result[key] = {
        tokens: b.tokens.toFixed(2),
        capacity: b.capacity.toFixed(2),
        rps: b.rps.toFixed(2),
      };
    }
    return result;
  }
}
```

### 4. 工具中间件集成

```typescript
// 一行集成到现有 ToolRegistry
const rateLimiter = new RateLimitEngine();

// 获取实时系统负载（可接 Prometheus / process.cpuUsage）
async function getSystemLoad(): Promise<number> {
  // 简单示例：用 CPU 使用率近似
  return new Promise(resolve => {
    const start = process.cpuUsage();
    setTimeout(() => {
      const { user, system } = process.cpuUsage(start);
      // 100ms 内 CPU 时间 / 总时间
      resolve(Math.min(1.0, (user + system) / 100_000 / 2));
    }, 100);
  });
}

async function rateLimitedInvoke(
  toolName: string,
  args: unknown,
  ctx: RateLimitContext
) {
  // 每次调用刷新系统负载（或后台定期刷新）
  ctx.systemLoad = await getSystemLoad();

  await rateLimiter.acquire(toolName, ctx);

  // 正常执行工具
  return await tools[toolName](args);
}
```

---

## Redis 分布式版本（多实例）

内存桶在多实例部署时各自独立，需要用 Redis：

```typescript
import Redis from 'ioredis';
const redis = new Redis();

// Lua 原子令牌桶（防竞态）
const TOKEN_BUCKET_SCRIPT = `
local key = KEYS[1]
local capacity = tonumber(ARGV[1])
local rps = tonumber(ARGV[2])
local now = tonumber(ARGV[3])

local data = redis.call('HMGET', key, 'tokens', 'last_refill')
local tokens = tonumber(data[1]) or capacity
local last_refill = tonumber(data[2]) or now

-- 补充令牌
local elapsed = (now - last_refill) / 1000
tokens = math.min(capacity, tokens + elapsed * rps)

if tokens >= 1 then
  tokens = tokens - 1
  redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
  redis.call('EXPIRE', key, 3600)
  return 1  -- 允许
else
  redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
  redis.call('EXPIRE', key, 3600)
  -- 返回需等待的毫秒数（负数）
  return math.floor(((1 - tokens) / rps) * 1000) * -1
end
`;

async function acquireDistributed(
  toolName: string,
  ctx: RateLimitContext
): Promise<void> {
  const rps = calcDynamicRps(toolName, ctx);
  const key = `rl:${toolName}:${ctx.userTier}`;

  const result = await (redis as any).eval(
    TOKEN_BUCKET_SCRIPT, 1, key,
    rps * 2,  // capacity
    rps,
    Date.now()
  ) as number;

  if (result < 0) {
    // result 是负的等待毫秒
    await new Promise(r => setTimeout(r, -result));
  }
}
```

---

## OpenClaw 实战：按用户等级保护工具

在 OpenClaw 的 skill 中，可以用这个模式保护外部 API 调用：

```typescript
// skills/mysterybox/tools.ts（示意）

const rl = new RateLimitEngine();

export async function grafanaQuery(sql: string, userCtx: RateLimitContext) {
  // Grafana API 是外部资源，严格限速
  await rl.acquire('grafana_query', userCtx);

  const resp = await fetch(`${GRAFANA_URL}/api/ds/query`, {
    method: 'POST',
    headers: { Authorization: `Bearer ${GRAFANA_KEY}` },
    body: JSON.stringify({ queries: [{ rawSql: sql }] }),
  });
  return resp.json();
}
```

效果：
- **free 用户**：`grafana_query` 基准 3 RPS × 0.3 = **0.9 RPS**
- **vip 用户**：3 RPS × 3.0 = **9 RPS**
- **高负载时（load=0.9）**：9 RPS × 0.2 = **1.8 RPS**（VIP 也自动降速）

---

## 与之前课程的区别

| 课程 | 解决的问题 |
|------|-----------|
| Rate Limiting & Backoff（早期课）| 处理**外部 API 返回 429**，是被动响应 |
| 本课（Context-Aware Rate Limiting）| Agent **主动**对自身工具调用限速，防止压垮下游 |
| Backpressure & Flow Control | 队列满时拒绝入队，偏系统级流量控制 |

---

## 关键设计总结

```
配额 = 基准 RPS
      × 用户等级因子   (free:0.3 / pro:1.0 / vip:3.0)
      × 系统负载因子   (空闲放宽 / 高压收紧)
      × 时间窗口因子   (低峰放宽 50%)
      × 工具类型因子   (写操作额外 ×0.5)

最低保障：永远 ≥ 0.1 RPS，不完全封锁

存储：内存（单实例）或 Redis Lua 脚本（多实例）
复合桶键：tool + user_tier（VIP 有独立配额桶）
```

---

## 参考代码

- `pi-mono/src/rate-limit/` — Token Bucket 基础实现
- `learn-claude-code/examples/middleware/` — 工具中间件范式
- OpenClaw `skills/mysterybox/` — Grafana API 调用场景
