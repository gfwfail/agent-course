# 135 - Agent 自适应并发控制（Adaptive Concurrency Control）

> "不是并发越多越好，而是刚好够用才是最优。"

---

## 问题背景

Agent 在执行任务时会同时调用多个工具：查数据库、调外部 API、读文件……并发度越高，吞吐量越大。但并发太高会导致：

- 下游 API Rate Limit 被触发（429 Too Many Requests）
- 内存/连接池耗尽
- 响应延迟反而上升（线程争用）

**固定并发数**是最常见的错误做法——你在部署时猜一个数，然后永远不对：

```
MAX_CONCURRENCY = 5  # 这个 5 是怎么来的？拍脑袋的
```

**自适应并发控制**：让系统在运行时动态感知最优并发度，像 TCP 的拥塞控制一样自动调节。

---

## 核心算法：AIMD

TCP 拥塞控制用的是 **AIMD（Additive Increase, Multiplicative Decrease）**：

```
成功 → limit += 1        (慢慢加)
失败 → limit = limit / 2 (快速退)
```

这个思路直接适用于工具调用：

```typescript
class AdaptiveConcurrencyLimiter {
  private limit: number;
  private inflight: number = 0;
  private readonly min: number;
  private readonly max: number;

  constructor(initial = 4, min = 1, max = 32) {
    this.limit = initial;
    this.min = min;
    this.max = max;
  }

  // 成功：小步增加
  onSuccess() {
    this.limit = Math.min(this.max, this.limit + 1);
  }

  // 失败/超时/限速：快速减半
  onFailure() {
    this.limit = Math.max(this.min, Math.floor(this.limit / 2));
  }

  get available(): number {
    return Math.max(0, this.limit - this.inflight);
  }

  async run<T>(fn: () => Promise<T>): Promise<T> {
    // 等待槽位
    while (this.inflight >= this.limit) {
      await new Promise(r => setTimeout(r, 10));
    }

    this.inflight++;
    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (err) {
      // 429 或超时才退让，其他错误不影响并发度
      if (isRateLimitError(err) || isTimeoutError(err)) {
        this.onFailure();
      }
      throw err;
    } finally {
      this.inflight--;
    }
  }
}

function isRateLimitError(err: unknown): boolean {
  return err instanceof Error && 
    (err.message.includes('429') || err.message.includes('rate limit'));
}

function isTimeoutError(err: unknown): boolean {
  return err instanceof Error && err.message.includes('timeout');
}
```

---

## 进阶：Vegas 算法（基于延迟感知）

AIMD 依赖错误信号，但最好在报错之前就感知到压力。TCP Vegas 的思路：

> **延迟上升 = 队列积压的早期信号**

```typescript
class VegasConcurrencyLimiter {
  private limit: number = 4;
  private inflight: number = 0;
  private baseRtt: number | null = null;   // 历史最低延迟（无压力基准）
  private readonly alpha = 3; // 队列深度阈值（低于 alpha → 增加）
  private readonly beta = 6;  // 队列深度阈值（高于 beta → 减少）

  async run<T>(fn: () => Promise<T>): Promise<T> {
    while (this.inflight >= this.limit) {
      await new Promise(r => setTimeout(r, 5));
    }

    this.inflight++;
    const start = Date.now();

    try {
      const result = await fn();
      const rtt = Date.now() - start;
      this.adjust(rtt);
      return result;
    } finally {
      this.inflight--;
    }
  }

  private adjust(rtt: number) {
    // 更新基准 RTT
    if (this.baseRtt === null || rtt < this.baseRtt) {
      this.baseRtt = rtt;
    }

    // 估算当前队列深度
    // 公式来自 TCP Vegas：queue = limit * (1 - baseRtt/rtt)
    const queue = this.limit * (1 - this.baseRtt / rtt);

    if (queue < this.alpha) {
      // 队列还有余量 → 增加并发
      this.limit = Math.min(64, this.limit + 1);
    } else if (queue > this.beta) {
      // 队列积压 → 减少并发
      this.limit = Math.max(1, this.limit - 1);
    }
    // alpha <= queue <= beta → 稳定不动
  }
}
```

**Vegas 的优势**：在 429 发生之前就自动降速，更温柔。

---

## 工具类型分层控制

不同工具类型有不同的并发容忍度：

```typescript
class ToolAwareConcurrencyManager {
  // 为每类工具维护独立的限速器
  private limiters = new Map<string, AdaptiveConcurrencyLimiter>([
    ['database',    new AdaptiveConcurrencyLimiter(8, 1, 20)],  // DB 扛得住
    ['external_api',new AdaptiveConcurrencyLimiter(3, 1, 10)],  // 外部 API 保守
    ['llm',         new AdaptiveConcurrencyLimiter(2, 1, 5)],   // LLM 最贵
    ['filesystem',  new AdaptiveConcurrencyLimiter(16, 1, 64)], // 本地 IO 可以多
    ['default',     new AdaptiveConcurrencyLimiter(4, 1, 16)],
  ]);

  private classifyTool(toolName: string): string {
    if (toolName.match(/^(query|insert|update|delete)/)) return 'database';
    if (toolName.match(/^(http|fetch|api_call)/)) return 'external_api';
    if (toolName.match(/^(llm|chat|complete)/)) return 'llm';
    if (toolName.match(/^(read|write|file)/)) return 'filesystem';
    return 'default';
  }

  async call<T>(toolName: string, fn: () => Promise<T>): Promise<T> {
    const category = this.classifyTool(toolName);
    const limiter = this.limiters.get(category)!;
    return limiter.run(fn);
  }

  // 打印当前状态（供监控/调试用）
  status() {
    const out: Record<string, number> = {};
    for (const [name, limiter] of this.limiters) {
      out[name] = limiter.limit;
    }
    return out;
  }
}
```

---

## 与 Agent Loop 集成（OpenClaw / pi-mono 风格）

```typescript
// tool-executor.ts
import { ToolAwareConcurrencyManager } from './concurrency';

const concurrencyMgr = new ToolAwareConcurrencyManager();

export async function executeToolWithAdaptiveConcurrency(
  toolName: string,
  toolFn: () => Promise<unknown>
): Promise<unknown> {
  return concurrencyMgr.call(toolName, toolFn);
}

// 在 Agent Loop 中使用
async function executeTools(toolCalls: ToolCall[]) {
  // 并发执行，但受自适应限速控制
  return Promise.all(
    toolCalls.map(tc =>
      executeToolWithAdaptiveConcurrency(tc.name, () => dispatch(tc))
    )
  );
}
```

---

## 实战：OpenClaw 中的自适应并发

OpenClaw 的 heartbeat 会并发触发多个检查（邮件、日历、通知），每个都是外部 API 调用。用自适应并发可以避免在 API 限速时整体 hang 住：

```typescript
// heartbeat-runner.ts
const limiter = new AdaptiveConcurrencyLimiter(3, 1, 8);

async function runHeartbeatChecks(checks: (() => Promise<string>)[]) {
  const results = await Promise.allSettled(
    checks.map(check => limiter.run(check))
  );

  // 记录当前自适应出的最优并发度
  console.log(`[heartbeat] adaptive limit settled at ${limiter.limit}`);

  return results
    .filter(r => r.status === 'fulfilled')
    .map(r => (r as PromiseFulfilledResult<string>).value);
}
```

---

## 与固定并发的对比实验

```
场景：100 个外部 API 调用，API 限速为 10 req/s

固定并发 = 20：
  → 立刻触发 429，大量重试，实际吞吐 ≈ 6 req/s（大量时间在等待）

自适应并发（AIMD，初始 = 5）：
  → 前 10 秒：5 → 6 → 7 → 触发限速 → 3 → 4 → 5 → ...
  → 稳定后自动收敛到 ≈ 10（刚好不触发限速）
  → 实际吞吐 ≈ 9.5 req/s，比固定并发高 58%
```

---

## 关键设计原则

| 原则 | 说明 |
|------|------|
| **分类隔离** | 不同工具类型用独立限速器，互不影响 |
| **只对压力信号退让** | 429/timeout 才减少，业务错误不影响并发度 |
| **基准 RTT 保存** | Vegas 算法需要历史最低延迟作为参考 |
| **可观测** | 暴露当前 limit/inflight 给监控 |
| **启动保守** | 初始值宁可小，让系统自己爬升 |

---

## 总结

```
固定并发 → 猜    → 总是错的
自适应并发 → 学  → 越跑越准
```

AIMD 简单高效，5 分钟就能接入现有代码。Vegas 更智能，在限速触发前就开始降速。两者都比硬编码的 `MAX_CONCURRENCY = 5` 强得多。

**下节预告**：Agent 工具调用结果流式聚合与优先级合并（Streaming Tool Result Fan-In）

---

*Agent 开发课程 第 135 课 | 2026-03-25*
