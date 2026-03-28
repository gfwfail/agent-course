# 161 - Agent 自适应并发控制

**Adaptive Concurrency Control**

---

## 为什么需要自适应并发控制？

并发工具执行（Lesson 12）让 Agent 同时跑多个工具，但"跑多少个"是个大问题：

- **太少**：明明可以并行，却串行等待，浪费时间
- **太多**：下游服务被打垮，错误率飙升，反而更慢
- **固定值**：白天低峰期浪费，晚上高峰期崩溃

真正生产级的 Agent 需要像 **TCP 拥塞控制** 一样，根据实时反馈动态调整并发度。

---

## 核心算法：AIMD（加法增加 / 乘法减少）

TCP 的成功秘诀：
- 成功 → 并发 +1（慢慢探索上限）
- 失败/超时 → 并发 ×0.5（快速收缩保稳定）

```
ConcurrencyLimit
      │
  20  │         ████
  15  │       ██    ██
  10  │     ██        ████
   5  │   ██              ██
   1  │ ██                  ████
      └──────────────────────────── 时间
         ↑成功慢增   ↑失败快降
```

---

## TypeScript 实现

```typescript
interface ConcurrencyStats {
  successCount: number;
  errorCount: number;
  timeoutCount: number;
  avgLatencyMs: number;
  currentLimit: number;
}

class AdaptiveConcurrencyController {
  private currentLimit: number;
  private inFlight: number = 0;
  private stats: ConcurrencyStats;
  
  // AIMD 参数
  private readonly MIN_LIMIT = 1;
  private readonly MAX_LIMIT = 50;
  private readonly INCREASE_STEP = 1;        // 成功时 +1
  private readonly DECREASE_FACTOR = 0.5;    // 失败时 ×0.5
  private readonly ERROR_THRESHOLD = 0.1;    // 错误率超 10% 触发降速
  private readonly LATENCY_THRESHOLD = 2000; // 平均延迟超 2s 触发降速
  
  // 滑动窗口（最近 100 次调用）
  private readonly WINDOW_SIZE = 100;
  private window: Array<{ success: boolean; latencyMs: number }> = [];

  constructor(initialLimit = 10) {
    this.currentLimit = initialLimit;
    this.stats = {
      successCount: 0,
      errorCount: 0,
      timeoutCount: 0,
      avgLatencyMs: 0,
      currentLimit: initialLimit,
    };
  }

  // 获取执行许可（信号量）
  async acquire(): Promise<() => void> {
    // 等待有空位
    while (this.inFlight >= this.currentLimit) {
      await new Promise(resolve => setTimeout(resolve, 10));
    }
    this.inFlight++;
    
    const startTime = Date.now();
    let released = false;
    
    // 返回 release 函数
    return (success: boolean = true) => {
      if (released) return;
      released = true;
      this.inFlight--;
      
      const latencyMs = Date.now() - startTime;
      this.recordOutcome(success, latencyMs);
      this.adjust();
    };
  }

  private recordOutcome(success: boolean, latencyMs: number): void {
    this.window.push({ success, latencyMs });
    if (this.window.length > this.WINDOW_SIZE) {
      this.window.shift();
    }
    
    if (success) this.stats.successCount++;
    else this.stats.errorCount++;
    
    // 更新平均延迟（指数移动平均）
    this.stats.avgLatencyMs = this.stats.avgLatencyMs * 0.9 + latencyMs * 0.1;
  }

  private adjust(): void {
    if (this.window.length < 10) return; // 样本不足，不调整
    
    const recentErrors = this.window.filter(r => !r.success).length;
    const errorRate = recentErrors / this.window.length;
    const avgLatency = this.window.reduce((s, r) => s + r.latencyMs, 0) / this.window.length;
    
    if (errorRate > this.ERROR_THRESHOLD || avgLatency > this.LATENCY_THRESHOLD) {
      // 降速：乘法减少
      const newLimit = Math.max(
        this.MIN_LIMIT,
        Math.floor(this.currentLimit * this.DECREASE_FACTOR)
      );
      if (newLimit < this.currentLimit) {
        console.log(`[ConcurrencyCtrl] ⬇ ${this.currentLimit} → ${newLimit} | errorRate=${(errorRate*100).toFixed(1)}% latency=${avgLatency.toFixed(0)}ms`);
        this.currentLimit = newLimit;
        // 清窗口，避免连续降速
        this.window = [];
      }
    } else {
      // 加速：加法增加（每 WINDOW_SIZE 次成功调用才加一次）
      if (this.window.length >= this.WINDOW_SIZE) {
        const newLimit = Math.min(this.MAX_LIMIT, this.currentLimit + this.INCREASE_STEP);
        if (newLimit > this.currentLimit) {
          console.log(`[ConcurrencyCtrl] ⬆ ${this.currentLimit} → ${newLimit}`);
          this.currentLimit = newLimit;
          this.window = [];
        }
      }
    }
    
    this.stats.currentLimit = this.currentLimit;
  }

  getStats(): ConcurrencyStats {
    return { ...this.stats };
  }
}
```

---

## 集成到工具执行器

```typescript
// 全局控制器（或按工具类型分别创建）
const globalController = new AdaptiveConcurrencyController(10);

// 按工具类型隔离（推荐：不同服务的容量不同）
const controllers = new Map<string, AdaptiveConcurrencyController>([
  ['web_search', new AdaptiveConcurrencyController(5)],
  ['database', new AdaptiveConcurrencyController(20)],
  ['llm_call',  new AdaptiveConcurrencyController(3)],
  ['file_io',   new AdaptiveConcurrencyController(15)],
]);

async function executeToolWithConcurrencyControl(
  toolName: string,
  toolFn: () => Promise<unknown>,
  timeoutMs = 5000
): Promise<unknown> {
  // 获取对应工具类型的控制器
  const controller = controllers.get(toolName) ?? globalController;
  const release = await controller.acquire();
  
  let success = true;
  try {
    // 带超时执行
    const result = await Promise.race([
      toolFn(),
      new Promise((_, reject) => 
        setTimeout(() => reject(new Error('timeout')), timeoutMs)
      ),
    ]);
    return result;
  } catch (err) {
    success = false;
    throw err;
  } finally {
    release(success); // 通知控制器结果
  }
}
```

---

## 与 OpenClaw Tool Middleware 集成

```typescript
// OpenClaw 风格的工具中间件
function adaptiveConcurrencyMiddleware(
  controller: AdaptiveConcurrencyController
) {
  return async (
    toolName: string,
    params: unknown,
    next: (params: unknown) => Promise<unknown>
  ): Promise<unknown> => {
    const release = await controller.acquire();
    let success = true;
    
    try {
      return await next(params);
    } catch (err) {
      success = false;
      throw err;
    } finally {
      release(success);
    }
  };
}

// 注册中间件
const registry = new ToolRegistry();
registry.use(adaptiveConcurrencyMiddleware(
  new AdaptiveConcurrencyController(10)
));
```

---

## 进阶：多维度自适应

```typescript
class MultiDimensionalController {
  private limitByPriority = new Map<string, number>([
    ['critical', 20],   // 关键工具保留更多并发
    ['normal',   10],
    ['background', 3],
  ]);

  // 高优先级任务可以"抢占"背景任务的配额
  async acquireWithPriority(priority: 'critical' | 'normal' | 'background') {
    const limit = this.limitByPriority.get(priority)!;
    // ...（同上，按优先级分别管理）
  }
  
  // 系统负载感知：CPU/内存高时自动降速
  async applySystemLoadFactor(): Promise<void> {
    const { loadavg } = await import('os');
    const load1m = loadavg()[0];
    const cpuCount = require('os').cpus().length;
    const loadFactor = Math.max(0.2, 1 - (load1m / cpuCount));
    
    for (const [key, baseLimit] of this.limitByPriority) {
      this.limitByPriority.set(key, Math.max(1, Math.floor(baseLimit * loadFactor)));
    }
  }
}
```

---

## Python 版本（asyncio）

```python
import asyncio
import time
from dataclasses import dataclass, field
from collections import deque

@dataclass
class AdaptiveSemaphore:
    initial_limit: int = 10
    min_limit: int = 1
    max_limit: int = 50
    error_threshold: float = 0.1
    latency_threshold_ms: float = 2000.0
    
    _limit: int = field(init=False)
    _in_flight: int = field(init=False, default=0)
    _window: deque = field(init=False)
    _lock: asyncio.Lock = field(init=False)
    
    def __post_init__(self):
        self._limit = self.initial_limit
        self._window = deque(maxlen=100)
        self._lock = asyncio.Lock()
    
    async def __aenter__(self):
        while self._in_flight >= self._limit:
            await asyncio.sleep(0.01)
        self._in_flight += 1
        self._start = time.monotonic()
        return self
    
    async def __aexit__(self, exc_type, *_):
        latency_ms = (time.monotonic() - self._start) * 1000
        success = exc_type is None
        self._in_flight -= 1
        
        self._window.append({'success': success, 'latency': latency_ms})
        await self._adjust()
    
    async def _adjust(self):
        if len(self._window) < 10:
            return
        
        async with self._lock:
            errors = sum(1 for r in self._window if not r['success'])
            error_rate = errors / len(self._window)
            avg_latency = sum(r['latency'] for r in self._window) / len(self._window)
            
            if error_rate > self.error_threshold or avg_latency > self.latency_threshold_ms:
                new_limit = max(self.min_limit, int(self._limit * 0.5))
                if new_limit < self._limit:
                    print(f"[Semaphore] ⬇ {self._limit} → {new_limit}")
                    self._limit = new_limit
                    self._window.clear()
            elif len(self._window) >= 100:
                new_limit = min(self.max_limit, self._limit + 1)
                if new_limit > self._limit:
                    print(f"[Semaphore] ⬆ {self._limit} → {new_limit}")
                    self._limit = new_limit
                    self._window.clear()


# 使用示例
sem = AdaptiveSemaphore(initial_limit=5)

async def call_tool(tool_name: str, payload: dict):
    async with sem:
        return await actual_tool_call(tool_name, payload)

# 并发执行多个工具
results = await asyncio.gather(
    *[call_tool("search", {"q": q}) for q in queries],
    return_exceptions=True
)
```

---

## 观测与调试

```typescript
// 每 30 秒打印一次并发控制状态
setInterval(() => {
  for (const [name, ctrl] of controllers) {
    const stats = ctrl.getStats();
    console.log(JSON.stringify({
      tool: name,
      limit: stats.currentLimit,
      inFlight: /* ... */,
      errorRate: (stats.errorCount / (stats.successCount + stats.errorCount) * 100).toFixed(1) + '%',
      avgLatency: stats.avgLatencyMs.toFixed(0) + 'ms',
    }));
  }
}, 30_000);
```

**Grafana 看板关键指标：**
- `agent_tool_concurrency_limit{tool="web_search"}` — 当前并发限制
- `agent_tool_in_flight{tool="web_search"}` — 实时飞行中请求数
- `agent_tool_error_rate{tool="web_search"}` — 错误率
- `agent_tool_latency_p95{tool="web_search"}` — P95 延迟

---

## 与现有 Patterns 的关系

| Pattern | 解决什么 |
|---------|---------|
| Concurrent Tool Execution (L12) | 如何启用并发 |
| Rate Limiting & Backoff (L15) | 外部 API 限速处理 |
| Backpressure & Flow Control (L80) | 队列满了怎么办 |
| **Adaptive Concurrency Control (本课)** | **并发上限本身应该是多少** |

---

## 小结

自适应并发控制是 Agent 弹性架构的最后一块拼图：

1. **AIMD 算法** — 成功慢增，失败快降，平衡吞吐与稳定性
2. **按工具类型隔离** — 不同服务有不同容量上限
3. **多维度感知** — 错误率 + 延迟 + 系统负载三合一
4. **可观测** — 并发限制变化要记录、要告警
5. **优先级感知** — 关键任务不被背景任务拖累

> 💡 把这个控制器加进去之后，Agent 在下游服务故障时会自动"慢下来"保护自己，而不是一直重试把对方打死。这就是生产级 Agent 和玩具 Agent 的区别。
