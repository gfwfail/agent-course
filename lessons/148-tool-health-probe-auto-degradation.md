# 148 - Agent 工具健康探针与自动降级
## Tool Health Probe & Auto-Degradation

---

## 为什么需要工具健康探针？

Agent 在生产环境中调用的工具往往是外部依赖：数据库、第三方 API、缓存服务……这些服务随时可能:

- 响应变慢（超时但未报错）
- 返回脏数据（接口在灰度升级）
- 半可用（部分请求成功，部分失败）

**被动错误处理（reactive）**：工具调用失败 → 捕获异常 → 重试  
**主动健康探针（proactive）**：后台定期探测 → 提前感知异常 → 请求到来时直接走降级路径

两者的区别：探针让你**在用户受影响之前**就知道工具坏了。

---

## 核心架构

```
┌─────────────────────────────────────────────────────┐
│                  Tool Registry                       │
│                                                      │
│  Tool A ──→ HealthProbe A ──→ Status: HEALTHY ✅     │
│  Tool B ──→ HealthProbe B ──→ Status: DEGRADED ⚠️   │
│  Tool C ──→ HealthProbe C ──→ Status: UNHEALTHY ❌   │
└──────────────────┬──────────────────────────────────┘
                   │
         ┌─────────▼─────────┐
         │  DegradationRouter │
         │                    │
         │  HEALTHY  → 正常调用│
         │  DEGRADED → 简化调用│
         │  UNHEALTHY→ 降级工具│
         └────────────────────┘
```

---

## TypeScript 实现

### 1. 健康状态定义

```typescript
// tool-health-probe.ts

export enum HealthStatus {
  HEALTHY   = 'healthy',    // 正常
  DEGRADED  = 'degraded',   // 性能下降，仍可用
  UNHEALTHY = 'unhealthy',  // 不可用
  UNKNOWN   = 'unknown',    // 未探测
}

export interface HealthCheckResult {
  status: HealthStatus;
  latencyMs: number;
  error?: string;
  checkedAt: number;
}

export interface ToolHealthProbe {
  toolName: string;
  // 探针函数：执行轻量级健康检查
  probe: () => Promise<HealthCheckResult>;
  // 降级工具名（UNHEALTHY 时使用）
  fallbackTool?: string;
  // 探针间隔（ms）
  intervalMs: number;
  // 连续失败几次才标记 UNHEALTHY
  failureThreshold: number;
  // 连续成功几次才恢复 HEALTHY
  recoveryThreshold: number;
}
```

### 2. 健康探针管理器

```typescript
export class ToolHealthManager {
  private probes = new Map<string, ToolHealthProbe>();
  private states = new Map<string, HealthCheckResult>();
  private consecutiveFailures = new Map<string, number>();
  private consecutiveSuccesses = new Map<string, number>();
  private timers = new Map<string, NodeJS.Timeout>();

  register(probe: ToolHealthProbe): void {
    this.probes.set(probe.toolName, probe);
    // 初始状态：未知
    this.states.set(probe.toolName, {
      status: HealthStatus.UNKNOWN,
      latencyMs: 0,
      checkedAt: 0,
    });
    this.startProbing(probe);
  }

  private startProbing(probe: ToolHealthProbe): void {
    const run = async () => {
      const result = await this.runProbe(probe);
      this.updateState(probe.toolName, result, probe);
    };

    // 立即探测一次，然后定期探测
    run();
    const timer = setInterval(run, probe.intervalMs);
    this.timers.set(probe.toolName, timer);
  }

  private async runProbe(probe: ToolHealthProbe): Promise<HealthCheckResult> {
    const start = Date.now();
    try {
      const result = await probe.probe();
      return result;
    } catch (err) {
      return {
        status: HealthStatus.UNHEALTHY,
        latencyMs: Date.now() - start,
        error: err instanceof Error ? err.message : String(err),
        checkedAt: Date.now(),
      };
    }
  }

  private updateState(
    toolName: string,
    result: HealthCheckResult,
    probe: ToolHealthProbe
  ): void {
    const isFailure = result.status === HealthStatus.UNHEALTHY;
    const prev = this.states.get(toolName)!;

    if (isFailure) {
      const failures = (this.consecutiveFailures.get(toolName) ?? 0) + 1;
      this.consecutiveFailures.set(toolName, failures);
      this.consecutiveSuccesses.set(toolName, 0);

      // 超过失败阈值 → UNHEALTHY
      if (failures >= probe.failureThreshold) {
        result.status = HealthStatus.UNHEALTHY;
        if (prev.status === HealthStatus.HEALTHY) {
          console.warn(`[HealthProbe] ⚠️  ${toolName} → UNHEALTHY (${failures} failures)`);
        }
      } else {
        // 未超阈值 → DEGRADED
        result.status = HealthStatus.DEGRADED;
      }
    } else {
      const successes = (this.consecutiveSuccesses.get(toolName) ?? 0) + 1;
      this.consecutiveSuccesses.set(toolName, successes);
      this.consecutiveFailures.set(toolName, 0);

      // 超过恢复阈值 → 恢复 HEALTHY
      if (successes >= probe.recoveryThreshold && prev.status !== HealthStatus.HEALTHY) {
        console.info(`[HealthProbe] ✅ ${toolName} → HEALTHY (recovered)`);
      }
    }

    this.states.set(toolName, result);
  }

  getStatus(toolName: string): HealthStatus {
    return this.states.get(toolName)?.status ?? HealthStatus.UNKNOWN;
  }

  getAll(): Record<string, HealthCheckResult> {
    return Object.fromEntries(this.states);
  }

  stop(): void {
    for (const timer of this.timers.values()) clearInterval(timer);
    this.timers.clear();
  }
}
```

### 3. 降级路由中间件

```typescript
export class DegradedToolRegistry {
  constructor(
    private tools: Map<string, (params: unknown) => Promise<unknown>>,
    private healthManager: ToolHealthManager,
    private probeConfigs: Map<string, ToolHealthProbe>
  ) {}

  async call(toolName: string, params: unknown): Promise<unknown> {
    const status = this.healthManager.getStatus(toolName);

    switch (status) {
      case HealthStatus.HEALTHY:
      case HealthStatus.UNKNOWN:
        // 正常调用
        return this.invoke(toolName, params);

      case HealthStatus.DEGRADED:
        // 仍然调用，但记录警告
        console.warn(`[DegradedRouter] ${toolName} is DEGRADED, calling with caution`);
        return this.invoke(toolName, params);

      case HealthStatus.UNHEALTHY: {
        // 查找降级工具
        const probe = this.probeConfigs.get(toolName);
        if (probe?.fallbackTool) {
          console.warn(
            `[DegradedRouter] ${toolName} UNHEALTHY → fallback: ${probe.fallbackTool}`
          );
          return this.invoke(probe.fallbackTool, params);
        }
        // 没有降级工具 → 返回结构化错误（让 LLM 感知）
        return {
          error: `Tool ${toolName} is currently unavailable. Please try again later or use an alternative approach.`,
          toolStatus: 'unhealthy',
        };
      }
    }
  }

  private async invoke(toolName: string, params: unknown): Promise<unknown> {
    const fn = this.tools.get(toolName);
    if (!fn) throw new Error(`Tool not found: ${toolName}`);
    return fn(params);
  }
}
```

### 4. 实战：为数据库和 API 工具注册探针

```typescript
// 数据库工具探针
const dbProbe: ToolHealthProbe = {
  toolName: 'query_database',
  intervalMs: 30_000,       // 每 30 秒探测
  failureThreshold: 3,      // 连续 3 次失败 → UNHEALTHY
  recoveryThreshold: 2,     // 连续 2 次成功 → 恢复
  fallbackTool: 'query_database_readonly_replica', // 主库挂了走只读副本
  probe: async () => {
    const start = Date.now();
    try {
      // 轻量探针：SELECT 1，不走业务逻辑
      await db.query('SELECT 1');
      const latencyMs = Date.now() - start;
      return {
        status: latencyMs < 200
          ? HealthStatus.HEALTHY
          : HealthStatus.DEGRADED,  // 慢但能用
        latencyMs,
        checkedAt: Date.now(),
      };
    } catch (err) {
      return {
        status: HealthStatus.UNHEALTHY,
        latencyMs: Date.now() - start,
        error: String(err),
        checkedAt: Date.now(),
      };
    }
  },
};

// 第三方 API 工具探针
const apiProbe: ToolHealthProbe = {
  toolName: 'call_payment_api',
  intervalMs: 60_000,
  failureThreshold: 2,
  recoveryThreshold: 3,
  fallbackTool: 'queue_payment_for_retry',  // 异步队列兜底
  probe: async () => {
    const start = Date.now();
    try {
      // 调用 /health 端点（不产生副作用）
      const res = await fetch('https://api.payment.com/health', {
        signal: AbortSignal.timeout(5000),
      });
      const latencyMs = Date.now() - start;
      return {
        status: res.ok
          ? (latencyMs < 1000 ? HealthStatus.HEALTHY : HealthStatus.DEGRADED)
          : HealthStatus.UNHEALTHY,
        latencyMs,
        checkedAt: Date.now(),
      };
    } catch {
      return {
        status: HealthStatus.UNHEALTHY,
        latencyMs: Date.now() - start,
        error: 'timeout or network error',
        checkedAt: Date.now(),
      };
    }
  },
};

// 注册
const healthManager = new ToolHealthManager();
healthManager.register(dbProbe);
healthManager.register(apiProbe);
```

---

## Python 版本（简化）

```python
import asyncio
import time
from dataclasses import dataclass, field
from enum import Enum
from typing import Callable, Awaitable, Optional

class HealthStatus(Enum):
    HEALTHY   = "healthy"
    DEGRADED  = "degraded"
    UNHEALTHY = "unhealthy"
    UNKNOWN   = "unknown"

@dataclass
class HealthCheckResult:
    status: HealthStatus
    latency_ms: float
    error: Optional[str] = None
    checked_at: float = field(default_factory=time.time)

@dataclass
class ToolHealthProbe:
    tool_name: str
    probe: Callable[[], Awaitable[HealthCheckResult]]
    interval_s: float = 30.0
    failure_threshold: int = 3
    recovery_threshold: int = 2
    fallback_tool: Optional[str] = None

class ToolHealthManager:
    def __init__(self):
        self._states: dict[str, HealthCheckResult] = {}
        self._failures: dict[str, int] = {}
        self._successes: dict[str, int] = {}
        self._probes: dict[str, ToolHealthProbe] = {}
        self._tasks: dict[str, asyncio.Task] = {}

    def register(self, probe: ToolHealthProbe):
        self._probes[probe.tool_name] = probe
        self._states[probe.tool_name] = HealthCheckResult(
            status=HealthStatus.UNKNOWN, latency_ms=0
        )
        task = asyncio.create_task(self._probe_loop(probe))
        self._tasks[probe.tool_name] = task

    async def _probe_loop(self, probe: ToolHealthProbe):
        while True:
            try:
                result = await probe.probe()
            except Exception as e:
                result = HealthCheckResult(
                    status=HealthStatus.UNHEALTHY,
                    latency_ms=0, error=str(e)
                )
            self._update(probe.tool_name, result, probe)
            await asyncio.sleep(probe.interval_s)

    def _update(self, name: str, result: HealthCheckResult, probe: ToolHealthProbe):
        if result.status == HealthStatus.UNHEALTHY:
            self._failures[name] = self._failures.get(name, 0) + 1
            self._successes[name] = 0
            if self._failures[name] >= probe.failure_threshold:
                result.status = HealthStatus.UNHEALTHY
            else:
                result.status = HealthStatus.DEGRADED
        else:
            self._successes[name] = self._successes.get(name, 0) + 1
            self._failures[name] = 0
        self._states[name] = result

    def get_status(self, name: str) -> HealthStatus:
        return self._states.get(name, HealthCheckResult(
            status=HealthStatus.UNKNOWN, latency_ms=0
        )).status

    async def call_with_degradation(self, tool_name: str, fn, fallback_fn, params):
        status = self.get_status(tool_name)
        if status == HealthStatus.UNHEALTHY and fallback_fn:
            print(f"[HealthProbe] {tool_name} UNHEALTHY → fallback")
            return await fallback_fn(params)
        return await fn(params)
```

---

## 与 OpenClaw Cron 集成：定时健康报告

```typescript
// 在 OpenClaw 中注册 Cron，每小时推送工具健康状态
// cron.add({
//   schedule: { kind: "every", everyMs: 3_600_000 },
//   payload: { kind: "systemEvent", text: "检查所有工具健康状态并汇报" },
//   sessionTarget: "main"
// })

// Agent 收到 systemEvent 后执行：
const report = Object.entries(healthManager.getAll())
  .map(([name, r]) => `${name}: ${r.status} (${r.latencyMs}ms)`)
  .join('\n');
console.log('[HealthReport]\n' + report);
```

---

## 关键设计决策

| 决策点 | 推荐方案 | 原因 |
|--------|----------|------|
| 探针频率 | 30-60s | 太频繁消耗资源，太慢感知延迟 |
| 失败阈值 | 3次 | 避免单次抖动误判 |
| 恢复阈值 | 2次 | 快速恢复，但防止一次成功就误判 |
| 探针实现 | SELECT 1 / /health 端点 | 轻量无副作用 |
| 降级工具 | 只读副本、异步队列、缓存 | 根据业务特性选择 |
| 工具不可用时 | 返回结构化错误而非抛异常 | 让 LLM 感知并自主决策 |

---

## 总结

```
工具健康探针 = 主动感知 + 自动降级
         ↓
不等用户报错，提前知道工具状态
         ↓
HEALTHY  → 正常调用
DEGRADED → 带警告调用（可记录指标）
UNHEALTHY → 自动切换降级工具或返回友好错误
         ↓
配合 Cron 定时报告，Agent 始终知道自己的"武器库"状态
```

工具健康探针是 Agent 生产稳定性的**最后一道主动防线**。不是等工具坏了再处理，而是提前知道、提前降级、用户无感知。
