# 114 - Agent 负载卸载与优先级 SLA（Load Shedding & Priority SLA）

> 背压是踩刹车，负载卸载是直接扔货——过载时主动丢弃低优先级任务，确保核心 SLA 不崩。

---

## 为什么需要负载卸载？

背压（Backpressure, Lesson 69）通过减慢生产者来保护系统，但有些场景不允许减速：

- **实时用户请求**：用户已经在等，再慢就超时了
- **Cron 任务积压**：定时任务堆积，晚跑不如不跑
- **多租户竞争**：VIP 用户和普通用户同时请求，资源有限

这时候需要更激进的策略：**直接拒绝/跳过低优先级工作**。

```
没有负载卸载：                有负载卸载：
━━━━━━━━━━━━━━━━━━━━━         ━━━━━━━━━━━━━━━━━━━━━
低优先级 ▓▓▓▓▓▓▓▓▓▓           低优先级 ░░░░ (丢弃)
高优先级 ▓▓▓▓▓▓▓▓▓▓           高优先级 ▓▓▓▓▓▓▓▓▓▓
                              ✓ VIP 始终得到服务
全部超时 ❌                    低优先级快速失败 ✓
```

---

## 核心概念：负载卸载 vs 背压 vs 限流

| 机制 | 作用对象 | 行为 | 适用场景 |
|------|---------|------|---------|
| **限流** | 入口 | 超速拒绝 | 保护系统不被打爆 |
| **背压** | 生产者 | 减慢速度 | 流式处理、管道 |
| **负载卸载** | 低优先级任务 | 主动丢弃 | 过载时保 SLA |

三层联合防御：限流 → 背压 → 负载卸载，层层递进。

---

## 实现一：优先级感知的负载卸载器

```typescript
// load-shedder.ts
export enum Priority {
  CRITICAL = 0,   // 实时用户请求，绝不丢弃
  HIGH = 1,       // VIP 用户，尽力保障
  NORMAL = 2,     // 普通请求
  LOW = 3,        // 后台任务、分析等
  BACKGROUND = 4, // 可随时丢弃
}

interface Task<T> {
  id: string;
  priority: Priority;
  fn: () => Promise<T>;
  timeoutMs: number;
  enqueuedAt: number;
}

interface ShedderConfig {
  maxConcurrent: number;
  maxQueueSize: number;
  // 每个优先级在过载时允许的最大队列占比
  priorityQuotas: Record<Priority, number>;
  // CPU/内存超过这个阈值就开始卸载
  overloadThreshold: number;
}

class LoadShedder {
  private running = 0;
  private queues: Map<Priority, Task<any>[]> = new Map();
  private metrics = { shed: 0, executed: 0, timeouts: 0 };

  constructor(private config: ShedderConfig) {
    for (const p of Object.values(Priority)) {
      if (typeof p === 'number') this.queues.set(p, []);
    }
  }

  async submit<T>(task: Task<T>): Promise<T> {
    // 1. 检查系统负载
    const load = await this.getSystemLoad();
    
    // 2. 过载时按优先级决定是否卸载
    if (load > this.config.overloadThreshold) {
      if (this.shouldShed(task.priority, load)) {
        this.metrics.shed++;
        throw new LoadSheddedError(
          `Task ${task.id} shed: system load ${load.toFixed(2)}, priority ${Priority[task.priority]}`
        );
      }
    }

    // 3. 检查队列容量（按优先级分配配额）
    const queue = this.queues.get(task.priority)!;
    const maxForPriority = Math.floor(
      this.config.maxQueueSize * this.config.priorityQuotas[task.priority]
    );
    
    if (queue.length >= maxForPriority) {
      // 队列满了：低优先级直接丢，高优先级抢占一个低优先级槽位
      if (!this.tryPreempt(task)) {
        this.metrics.shed++;
        throw new LoadSheddedError(`Queue full for priority ${Priority[task.priority]}`);
      }
    }

    // 4. 入队并执行
    return this.enqueueAndRun(task);
  }

  private shouldShed(priority: Priority, load: number): boolean {
    // 负载越高，丢弃门槛越低
    // load=0.8: 只丢 BACKGROUND
    // load=0.9: 丢 BACKGROUND + LOW
    // load=0.95: 丢 BACKGROUND + LOW + NORMAL
    const thresholds: Record<Priority, number> = {
      [Priority.CRITICAL]: 2.0,  // 永不丢弃（阈值设成不可能达到）
      [Priority.HIGH]: 0.99,
      [Priority.NORMAL]: 0.95,
      [Priority.LOW]: 0.90,
      [Priority.BACKGROUND]: 0.80,
    };
    return load >= thresholds[priority];
  }

  private tryPreempt(incoming: Task<any>): boolean {
    // 找一个比 incoming 优先级更低的任务踢掉
    for (let p = Priority.BACKGROUND; p > incoming.priority; p--) {
      const lowerQueue = this.queues.get(p)!;
      if (lowerQueue.length > 0) {
        const evicted = lowerQueue.pop()!; // 踢掉最后入队的（LIFO 踢出，FIFO 执行）
        this.metrics.shed++;
        console.log(`Preempted task ${evicted.id} (${Priority[p]}) for ${incoming.id} (${Priority[incoming.priority]})`);
        return true;
      }
    }
    return false;
  }

  private async enqueueAndRun<T>(task: Task<T>): Promise<T> {
    const queue = this.queues.get(task.priority)!;
    
    return new Promise((resolve, reject) => {
      const wrappedTask = { ...task, resolve, reject };
      queue.push(wrappedTask as any);
      this.drain();
    });
  }

  private drain() {
    if (this.running >= this.config.maxConcurrent) return;

    // 按优先级从高到低取任务
    for (let p = Priority.CRITICAL; p <= Priority.BACKGROUND; p++) {
      const queue = this.queues.get(p)!;
      if (queue.length > 0) {
        const task = queue.shift() as any;
        this.execute(task);
        return;
      }
    }
  }

  private async execute(task: any) {
    this.running++;
    
    const timeoutPromise = new Promise<never>((_, reject) =>
      setTimeout(() => {
        this.metrics.timeouts++;
        reject(new Error(`Task ${task.id} timed out after ${task.timeoutMs}ms`));
      }, task.timeoutMs)
    );

    try {
      const result = await Promise.race([task.fn(), timeoutPromise]);
      this.metrics.executed++;
      task.resolve(result);
    } catch (err) {
      task.reject(err);
    } finally {
      this.running--;
      this.drain(); // 执行完继续取下一个
    }
  }

  private async getSystemLoad(): Promise<number> {
    // 实际实现可以用 os.loadavg() 或自定义指标
    // 这里用并发数作为代理指标
    const concurrencyLoad = this.running / this.config.maxConcurrent;
    const totalQueued = Array.from(this.queues.values())
      .reduce((sum, q) => sum + q.length, 0);
    const queueLoad = totalQueued / this.config.maxQueueSize;
    return Math.max(concurrencyLoad, queueLoad);
  }

  getMetrics() {
    return {
      ...this.metrics,
      running: this.running,
      queued: Array.from(this.queues.entries()).reduce((acc, [p, q]) => {
        acc[Priority[p]] = q.length;
        return acc;
      }, {} as Record<string, number>),
    };
  }
}

class LoadSheddedError extends Error {
  constructor(message: string) {
    super(message);
    this.name = 'LoadSheddedError';
  }
}
```

---

## 实现二：Agent 工具调用层的负载卸载

在工具调用中间件里加入负载卸载逻辑：

```typescript
// tool-middleware-load-shedding.ts
import { Tool, ToolResult } from './types';

// 每个工具的优先级配置
const TOOL_PRIORITIES: Record<string, Priority> = {
  // 用户直接感知的工具 → CRITICAL/HIGH
  'send_message': Priority.CRITICAL,
  'web_search': Priority.HIGH,
  'read_file': Priority.HIGH,
  
  // 后台工具 → NORMAL/LOW
  'write_file': Priority.NORMAL,
  'git_commit': Priority.LOW,
  
  // 分析/日志工具 → BACKGROUND
  'update_metrics': Priority.BACKGROUND,
  'log_event': Priority.BACKGROUND,
};

function createLoadSheddingMiddleware(shedder: LoadShedder) {
  return async (
    toolName: string,
    params: Record<string, any>,
    next: () => Promise<ToolResult>,
    context: { sessionId: string; userId: string; isVip: boolean }
  ): Promise<ToolResult> => {
    
    const basePriority = TOOL_PRIORITIES[toolName] ?? Priority.NORMAL;
    
    // VIP 用户提升一个优先级
    const priority = context.isVip 
      ? Math.max(Priority.CRITICAL, basePriority - 1) 
      : basePriority;

    try {
      return await shedder.submit({
        id: `${context.sessionId}:${toolName}:${Date.now()}`,
        priority,
        fn: next,
        timeoutMs: getToolTimeout(toolName),
        enqueuedAt: Date.now(),
      });
    } catch (err) {
      if (err instanceof LoadSheddedError) {
        // 返回标准化的卸载响应，而不是抛错
        return {
          error: 'SERVICE_OVERLOADED',
          message: '系统繁忙，请稍后重试',
          retryAfterMs: 5000,
          shed: true,
        };
      }
      throw err;
    }
  };
}

function getToolTimeout(toolName: string): number {
  const timeouts: Record<string, number> = {
    'web_search': 10000,
    'send_message': 3000,
    'read_file': 2000,
    'git_commit': 30000,
  };
  return timeouts[toolName] ?? 15000;
}
```

---

## 实现三：OpenClaw Cron 任务的负载卸载

Cron 任务是天然的 BACKGROUND 优先级候选——过载时跳过，不影响实时用户：

```typescript
// cron-load-aware.ts
interface CronJob {
  id: string;
  schedule: string;
  priority: Priority;
  fn: () => Promise<void>;
  // 允许跳过（过载时不补跑）
  skippable: boolean;
}

class LoadAwareCronScheduler {
  private shedder: LoadShedder;
  private skipStats: Map<string, number> = new Map();

  constructor(shedder: LoadShedder) {
    this.shedder = shedder;
  }

  async runJob(job: CronJob): Promise<void> {
    if (!job.skippable) {
      // 不可跳过的任务直接运行（如账单结算）
      await job.fn();
      return;
    }

    try {
      await this.shedder.submit({
        id: job.id,
        priority: job.priority,
        fn: job.fn,
        timeoutMs: 300_000, // cron 任务给更长超时
        enqueuedAt: Date.now(),
      });
    } catch (err) {
      if (err instanceof LoadSheddedError) {
        const skips = (this.skipStats.get(job.id) ?? 0) + 1;
        this.skipStats.set(job.id, skips);
        console.log(`[CronShed] Job ${job.id} skipped (${skips} total skips): ${err.message}`);
        
        // 连续跳过太多次，升级优先级发出告警
        if (skips >= 3) {
          await this.alertBacklog(job.id, skips);
        }
      } else {
        throw err;
      }
    }
  }

  private async alertBacklog(jobId: string, skips: number): Promise<void> {
    console.error(`⚠️ Job ${jobId} skipped ${skips} times! System may be critically overloaded.`);
    // 发送告警到监控系统
  }
}

// 配置示例
const cronJobs: CronJob[] = [
  {
    id: 'billing-settlement',
    schedule: '0 0 * * *',
    priority: Priority.CRITICAL,
    fn: runBillingSettlement,
    skippable: false, // 账单不能跳过
  },
  {
    id: 'metrics-aggregation',
    schedule: '*/5 * * * *',
    priority: Priority.BACKGROUND,
    fn: aggregateMetrics,
    skippable: true, // 指标聚合可以跳过
  },
  {
    id: 'agent-course-update',
    schedule: '0 */3 * * *',
    priority: Priority.LOW,
    fn: updateAgentCourse,
    skippable: true, // 课程更新可以跳过（下次补）
  },
];
```

---

## 实现四：带降级响应的完整 Agent

```typescript
// agent-with-load-shedding.ts
class ResilientAgent {
  private shedder: LoadShedder;
  
  constructor() {
    this.shedder = new LoadShedder({
      maxConcurrent: 10,
      maxQueueSize: 100,
      priorityQuotas: {
        [Priority.CRITICAL]: 0.4,    // 40% 队列给 CRITICAL
        [Priority.HIGH]: 0.3,        // 30% 给 HIGH
        [Priority.NORMAL]: 0.2,      // 20% 给 NORMAL
        [Priority.LOW]: 0.07,        // 7% 给 LOW
        [Priority.BACKGROUND]: 0.03, // 3% 给 BACKGROUND
      },
      overloadThreshold: 0.8,
    });
  }

  async handleRequest(req: AgentRequest): Promise<AgentResponse> {
    const priority = this.classifyPriority(req);
    
    try {
      return await this.shedder.submit({
        id: req.id,
        priority,
        fn: () => this.processRequest(req),
        timeoutMs: priority <= Priority.HIGH ? 30_000 : 60_000,
        enqueuedAt: Date.now(),
      });
    } catch (err) {
      if (err instanceof LoadSheddedError) {
        // 降级响应：不是崩溃，而是优雅拒绝
        return this.buildDegradedResponse(req, err);
      }
      throw err;
    }
  }

  private classifyPriority(req: AgentRequest): Priority {
    if (req.metadata?.isVip) return Priority.HIGH;
    if (req.type === 'realtime') return Priority.CRITICAL;
    if (req.type === 'batch') return Priority.LOW;
    if (req.type === 'analytics') return Priority.BACKGROUND;
    return Priority.NORMAL;
  }

  private buildDegradedResponse(req: AgentRequest, err: LoadSheddedError): AgentResponse {
    return {
      id: req.id,
      status: 'degraded',
      message: '系统当前繁忙，您的请求已被放入优先队列，请稍后重试。',
      retryAfterMs: this.calculateBackoff(req),
      // 尽量提供有用的缓存结果
      cachedResult: this.getCachedResult(req),
    };
  }

  private calculateBackoff(req: AgentRequest): number {
    // 优先级越低，退避时间越长
    const baseMs = 1000;
    const priority = this.classifyPriority(req);
    return baseMs * Math.pow(2, priority); // 1s, 2s, 4s, 8s, 16s
  }

  private getCachedResult(req: AgentRequest): any {
    // 从缓存（Semantic Cache, Lesson 54）取上次的结果
    return cache.get(req.query);
  }
}
```

---

## OpenClaw 中的实际应用

OpenClaw 本身就内置了多层负载保护：

```
用户请求
    ↓
[Rate Limiter] → 超速拒绝（每分钟 X 次）
    ↓
[Concurrency Guard] → 最多 N 个并发 session
    ↓  
[Tool Policy Pipeline] → 危险工具需确认
    ↓
[Load Shedder] → 过载时按优先级丢弃 cron/background 任务
    ↓
[Backpressure] → 减慢 sub-agent 任务分发
    ↓
[LLM API] → Anthropic 端的限流（TPM/RPM）
```

心跳（heartbeat）就是典型的 `BACKGROUND` 任务——过载时跳过，返回 `HEARTBEAT_OK` 是正确行为。

---

## 监控指标

```typescript
interface ShedderMetrics {
  // 核心指标
  shedRate: number;        // 卸载率 = shed / (shed + executed)
  shedByPriority: Record<string, number>; // 各优先级卸载数
  
  // SLA 指标  
  p50LatencyMs: number;
  p95LatencyMs: number;
  p99LatencyMs: number;
  
  // 告警阈值
  // shedRate > 0.1 (10%) → 系统过载，需扩容
  // CRITICAL 任务被 shed → 立即告警
  // 连续 shed > 5 min → PagerDuty
}
```

---

## 总结

| 场景 | 推荐优先级 | 是否 skippable |
|------|-----------|----------------|
| 实时用户对话 | CRITICAL | ❌ |
| VIP 用户请求 | HIGH | ❌ |
| 普通用户请求 | NORMAL | ❌ |
| Git 提交/推送 | LOW | ✅ |
| Cron 分析任务 | BACKGROUND | ✅ |
| 心跳检查 | BACKGROUND | ✅ |
| 账单结算 | CRITICAL | ❌ |

**核心原则：**
1. **宁可丢弃，不可全崩** — 10% 请求失败 >> 100% 请求超时
2. **优先级要提前设计** — 事后加优先级很痛苦
3. **降级要优雅** — 被丢弃的请求应该得到有意义的响应
4. **监控 shed rate** — 持续 >5% 说明需要扩容

> 背压踩刹车，负载卸载是弃货保船。两者结合，Agent 在风暴中也能稳定服务核心用户。
