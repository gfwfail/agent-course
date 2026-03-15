# 56 - Agent Circuit Breaker Pattern：熔断器模式

> 工具调用失败不可怕，怕的是 Agent 傻乎乎地一直重试，直到把自己和下游都拖垮。

---

## 什么是熔断器？

熔断器（Circuit Breaker）来自电气工程：电路过载时自动断开，保护设备。在分布式系统中，它是 Netflix Hystrix 推广的一种弹性模式。

**在 Agent 中的场景：**
- 数据库工具连续失败 5 次 → 继续调用只是浪费 token 和时间
- 外部 API 超时 → 每次等 30 秒会把上下文窗口耗尽
- MCP Server 挂掉 → Agent 应该「知道」这个工具暂时不可用，换策略

---

## 三种状态

```
         失败次数 >= 阈值
CLOSED ─────────────────────► OPEN
  ▲                              │
  │    成功                      │ 冷却时间过后
  └── HALF-OPEN ◄────────────────┘
        │
        │ 失败
        ▼
       OPEN（重新打开）
```

| 状态 | 行为 |
|------|------|
| **CLOSED** | 正常调用，记录失败次数 |
| **OPEN** | 直接拒绝，不发出调用，立即返回错误 |
| **HALF-OPEN** | 允许一次试探性调用，成功则关闭，失败则重新打开 |

---

## 代码实现：TypeScript（pi-mono 风格）

```typescript
// circuit-breaker.ts

type CircuitState = 'CLOSED' | 'OPEN' | 'HALF_OPEN';

interface CircuitBreakerOptions {
  failureThreshold: number;   // 失败多少次打开熔断器
  recoveryTimeout: number;    // 打开多久后进入半开状态（ms）
  successThreshold: number;   // 半开状态需要多少次成功才能关闭
}

class CircuitBreaker {
  private state: CircuitState = 'CLOSED';
  private failureCount = 0;
  private successCount = 0;
  private lastFailureTime = 0;
  private readonly name: string;

  constructor(
    name: string,
    private options: CircuitBreakerOptions = {
      failureThreshold: 5,
      recoveryTimeout: 60_000,
      successThreshold: 2,
    }
  ) {
    this.name = name;
  }

  async call<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === 'OPEN') {
      const elapsed = Date.now() - this.lastFailureTime;
      if (elapsed >= this.options.recoveryTimeout) {
        console.log(`[CircuitBreaker:${this.name}] OPEN → HALF_OPEN（开始试探）`);
        this.state = 'HALF_OPEN';
        this.successCount = 0;
      } else {
        // 快速失败，不等待
        throw new Error(
          `[CircuitBreaker:${this.name}] 熔断器已开启，拒绝调用。` +
          `剩余冷却时间: ${Math.ceil((this.options.recoveryTimeout - elapsed) / 1000)}s`
        );
      }
    }

    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (err) {
      this.onFailure();
      throw err;
    }
  }

  private onSuccess() {
    this.failureCount = 0;
    if (this.state === 'HALF_OPEN') {
      this.successCount++;
      if (this.successCount >= this.options.successThreshold) {
        console.log(`[CircuitBreaker:${this.name}] HALF_OPEN → CLOSED（恢复正常）`);
        this.state = 'CLOSED';
      }
    }
  }

  private onFailure() {
    this.failureCount++;
    this.lastFailureTime = Date.now();

    if (this.state === 'HALF_OPEN') {
      console.log(`[CircuitBreaker:${this.name}] HALF_OPEN → OPEN（试探失败）`);
      this.state = 'OPEN';
    } else if (this.failureCount >= this.options.failureThreshold) {
      console.log(`[CircuitBreaker:${this.name}] CLOSED → OPEN（失败次数: ${this.failureCount}）`);
      this.state = 'OPEN';
    }
  }

  getState() {
    return { state: this.state, failureCount: this.failureCount };
  }
}
```

---

## 在 Agent Tool Dispatch 中集成

```typescript
// tool-registry.ts（在 pi-mono 的 tool dispatch 层集成）

const breakers = new Map<string, CircuitBreaker>();

function getBreakerForTool(toolName: string): CircuitBreaker {
  if (!breakers.has(toolName)) {
    breakers.set(toolName, new CircuitBreaker(toolName, {
      failureThreshold: 3,     // 3 次失败就熔断
      recoveryTimeout: 30_000, // 30 秒后试探
      successThreshold: 1,
    }));
  }
  return breakers.get(toolName)!;
}

async function dispatchTool(toolName: string, params: unknown) {
  const breaker = getBreakerForTool(toolName);

  try {
    return await breaker.call(() => executeTool(toolName, params));
  } catch (err) {
    if (err.message.includes('熔断器已开启')) {
      // 返回结构化错误，让 Agent 知道应该换策略
      return {
        error: 'CIRCUIT_OPEN',
        tool: toolName,
        message: err.message,
        suggestion: '该工具暂时不可用，请尝试其他方案',
      };
    }
    throw err;
  }
}
```

---

## Python 实现（learn-claude-code 风格）

```python
# circuit_breaker.py
import time
import asyncio
from dataclasses import dataclass, field
from enum import Enum
from typing import Callable, TypeVar, Awaitable

T = TypeVar('T')

class State(Enum):
    CLOSED = "closed"
    OPEN = "open"
    HALF_OPEN = "half_open"

@dataclass
class CircuitBreaker:
    name: str
    failure_threshold: int = 5
    recovery_timeout: float = 60.0  # seconds
    success_threshold: int = 2

    _state: State = field(default=State.CLOSED, init=False)
    _failure_count: int = field(default=0, init=False)
    _success_count: int = field(default=0, init=False)
    _last_failure_time: float = field(default=0.0, init=False)

    async def call(self, fn: Callable[[], Awaitable[T]]) -> T:
        if self._state == State.OPEN:
            elapsed = time.time() - self._last_failure_time
            if elapsed >= self.recovery_timeout:
                print(f"[{self.name}] OPEN → HALF_OPEN")
                self._state = State.HALF_OPEN
                self._success_count = 0
            else:
                remaining = self.recovery_timeout - elapsed
                raise RuntimeError(
                    f"Circuit OPEN for {self.name}. "
                    f"Retry in {remaining:.0f}s"
                )

        try:
            result = await fn()
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise

    def _on_success(self):
        self._failure_count = 0
        if self._state == State.HALF_OPEN:
            self._success_count += 1
            if self._success_count >= self.success_threshold:
                print(f"[{self.name}] HALF_OPEN → CLOSED ✅")
                self._state = State.CLOSED

    def _on_failure(self):
        self._failure_count += 1
        self._last_failure_time = time.time()
        if self._state == State.HALF_OPEN:
            print(f"[{self.name}] HALF_OPEN → OPEN ❌")
            self._state = State.OPEN
        elif self._failure_count >= self.failure_threshold:
            print(f"[{self.name}] CLOSED → OPEN (failures: {self._failure_count})")
            self._state = State.OPEN


# 在 Claude Code tool handler 中使用
db_breaker = CircuitBreaker("database", failure_threshold=3, recovery_timeout=30.0)
api_breaker = CircuitBreaker("external_api", failure_threshold=5, recovery_timeout=120.0)

async def handle_tool_call(tool_name: str, params: dict) -> dict:
    breakers = {"query_db": db_breaker, "call_api": api_breaker}
    breaker = breakers.get(tool_name)

    try:
        if breaker:
            result = await breaker.call(lambda: execute_tool(tool_name, params))
        else:
            result = await execute_tool(tool_name, params)
        return {"success": True, "result": result}
    except RuntimeError as e:
        if "Circuit OPEN" in str(e):
            # 告诉 Agent 换策略
            return {
                "success": False,
                "error": "TOOL_UNAVAILABLE",
                "message": str(e),
                "alternatives": suggest_alternatives(tool_name),
            }
        raise
```

---

## OpenClaw 的实际应用

OpenClaw 自己怎么用熔断器思维？看看 cron job 的错误处理：

```typescript
// OpenClaw 风格：在 skill 执行层加熔断保护
// 如果 Grafana API 连续失败，自动降级到缓存数据

const grafanaBreaker = new CircuitBreaker('grafana', {
  failureThreshold: 3,
  recoveryTimeout: 5 * 60 * 1000, // 5分钟
  successThreshold: 1,
});

async function queryGrafana(sql: string) {
  try {
    return await grafanaBreaker.call(() => grafanaApi.query(sql));
  } catch (err) {
    if (err.message.includes('熔断器已开启')) {
      // 降级：返回缓存的最后一次成功结果
      const cached = await cache.get(`grafana:last-success`);
      if (cached) {
        return { ...cached, _stale: true, _note: 'Grafana 暂时不可用，返回缓存数据' };
      }
    }
    throw err;
  }
}
```

---

## 与 Retry/Backoff 的区别

| 对比维度 | Retry + Backoff | Circuit Breaker |
|----------|-----------------|-----------------|
| **触发方式** | 每次失败都重试 | 累计失败后停止重试 |
| **目标** | 短暂故障恢复 | 持续故障隔离 |
| **时间维度** | 单次调用粒度 | 跨多次调用状态 |
| **对下游影响** | 可能加剧雪崩 | 保护下游服务 |
| **组合使用** | ✅ 推荐配合使用 | ✅ 推荐配合使用 |

**最佳实践：** 在 Circuit Breaker 内部的单次调用上套 Retry（最多重试 2 次），外层用 Circuit Breaker 跟踪整体健康状态。

---

## Agent 特有的熔断策略

普通熔断器是通用的，Agent 有额外考虑：

```typescript
// Agent 上下文感知的熔断器
class AgentAwareCircuitBreaker extends CircuitBreaker {
  // 当 Agent 处于关键路径时，降低熔断阈值
  async callWithContext(
    fn: () => Promise<unknown>,
    context: { isCritical: boolean; remainingBudget: number }
  ) {
    const threshold = context.isCritical ? 1 : this.options.failureThreshold;
    // 临时调整阈值
    const originalThreshold = this.options.failureThreshold;
    this.options.failureThreshold = threshold;
    try {
      return await this.call(fn);
    } finally {
      this.options.failureThreshold = originalThreshold;
    }
  }
}

// 使用场景：最后一步的工具调用比初期探索更关键
await breaker.callWithContext(
  () => tools.finalDelivery(result),
  { isCritical: true, remainingBudget: tokenBudget }
);
```

---

## 关键要点

1. **熔断器是状态机** — CLOSED → OPEN → HALF_OPEN → CLOSED，每个状态有明确的转换条件
2. **快速失败** — OPEN 状态不等待，立即返回错误，节省 token 和时间
3. **可观测性** — 记录状态转换日志，方便排查「为什么工具突然不工作了」
4. **优雅降级** — 熔断后给 Agent 提供替代方案，而不是直接抛错
5. **粒度选择** — 按工具/服务粒度设置熔断器，不要一刀切

---

## 扩展阅读

- [Netflix Tech Blog: Hystrix Design](https://netflix.github.io/Hystrix/javadoc/com/netflix/hystrix/HystrixCircuitBreaker.html)
- [Martin Fowler: Circuit Breaker](https://martinfowler.com/bliki/CircuitBreaker.html)
- pi-mono: `packages/tools/src/resilience/`
- OpenClaw: `src/cron/job-executor.ts`（错误隔离逻辑）
