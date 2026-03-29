# 171 - Agent 混沌工程实战

> Chaos Engineering for Agents：故障注入 + 韧性验证，用"主动搞坏"找出生产隐患

---

## 为什么 Agent 需要混沌工程？

Agent 系统比普通服务更复杂：

- **工具链串联**：任何一个工具超时，整个 Agent Loop 卡住
- **LLM 不确定性**：相同输入可能走不同工具路径
- **状态有状态**：会话状态、Redis 缓存、外部 API 全都可能挂
- **Sub-agent 依赖**：子 Agent 挂了主 Agent 不一定感知到

混沌工程的核心思想：**与其等生产出问题，不如自己先把它搞坏，找到弱点提前修**。

---

## 混沌工程三步法

```
1. 定义"稳态"（Steady State）
   ↓ Agent 正常工作的可观测指标（成功率 > 99%、P95 < 2s）
2. 注入故障（Fault Injection）
   ↓ 工具延迟/错误/超时/网络分区
3. 观察 + 恢复（Observe & Restore）
   ↓ 验证熔断触发、降级路径、告警是否正常
```

---

## 核心实现：ChaosMiddleware

```typescript
// chaos/ChaosMiddleware.ts
import { ToolMiddleware, ToolCall, ToolResult } from '../types';

export interface ChaosScenario {
  toolPattern: RegExp;       // 哪些工具受影响
  kind: 'delay' | 'error' | 'timeout' | 'partial';
  probability: number;       // 0-1，触发概率
  delayMs?: number;          // kind=delay 时的延迟
  errorMsg?: string;         // kind=error 时的错误消息
  partialFn?: (result: ToolResult) => ToolResult; // 返回残缺结果
}

export class ChaosMiddleware implements ToolMiddleware {
  private scenarios: ChaosScenario[] = [];
  private enabled = false;
  private injectionLog: Array<{ ts: number; tool: string; scenario: string }> = [];

  enable(scenarios: ChaosScenario[]) {
    this.scenarios = scenarios;
    this.enabled = true;
    console.warn('[CHAOS] 混沌模式已启用，故障注入激活');
  }

  disable() {
    this.enabled = false;
    console.log('[CHAOS] 混沌模式已关闭');
  }

  getLog() { return this.injectionLog; }

  async wrap(call: ToolCall, next: () => Promise<ToolResult>): Promise<ToolResult> {
    if (!this.enabled) return next();

    for (const scenario of this.scenarios) {
      if (!scenario.toolPattern.test(call.name)) continue;
      if (Math.random() > scenario.probability) continue;

      // 记录注入日志
      this.injectionLog.push({ ts: Date.now(), tool: call.name, scenario: scenario.kind });

      switch (scenario.kind) {
        case 'delay':
          await sleep(scenario.delayMs ?? 3000);
          return next();

        case 'error':
          throw new Error(scenario.errorMsg ?? `[CHAOS] 工具 ${call.name} 模拟故障`);

        case 'timeout':
          // 永远不返回，触发超时
          await sleep(60_000);
          throw new Error('[CHAOS] 超时');

        case 'partial':
          const result = await next();
          return scenario.partialFn ? scenario.partialFn(result) : result;
      }
    }

    return next();
  }
}

function sleep(ms: number) {
  return new Promise(resolve => setTimeout(resolve, ms));
}
```

---

## 场景库：常见故障场景

```typescript
// chaos/scenarios.ts
export const CHAOS_SCENARIOS = {
  // 场景1：数据库工具随机超时（测试熔断器）
  dbSlowdown: {
    toolPattern: /^(db_query|db_write)/,
    kind: 'delay' as const,
    probability: 0.3,
    delayMs: 5000,
  },

  // 场景2：搜索工具随机报错（测试降级路径）
  searchFlaky: {
    toolPattern: /^web_search/,
    kind: 'error' as const,
    probability: 0.5,
    errorMsg: '搜索服务不可用',
  },

  // 场景3：外部 API 完全不响应（测试超时+熔断）
  apiBlackHole: {
    toolPattern: /^(http_get|http_post|fetch_url)/,
    kind: 'timeout' as const,
    probability: 1.0,
  },

  // 场景4：返回残缺数据（测试 LLM 对不完整数据的处理）
  partialResponse: {
    toolPattern: /^read_file/,
    kind: 'partial' as const,
    probability: 0.4,
    partialFn: (r) => ({ ...r, content: r.content?.slice(0, 50) + '...[TRUNCATED]' }),
  },
};
```

---

## 与熔断器联动

结合之前的 CircuitBreaker（Lesson 73），验证熔断是否被正确触发：

```typescript
// chaos/GameDay.ts — "混沌游戏日"
export async function runGameDay(
  agent: Agent,
  chaos: ChaosMiddleware,
  circuit: CircuitBreakerRegistry,
  testCases: string[],
) {
  console.log('=== 🔥 混沌游戏日开始 ===');

  // 1. 记录稳态指标
  const baseline = await measureSteadyState(agent, testCases.slice(0, 5));
  console.log('稳态基线:', baseline);

  // 2. 注入故障
  chaos.enable([
    CHAOS_SCENARIOS.dbSlowdown,
    CHAOS_SCENARIOS.searchFlaky,
  ]);

  // 3. 运行相同测试
  const underChaos = await measureSteadyState(agent, testCases);
  console.log('故障下指标:', underChaos);

  // 4. 检查熔断器状态
  const circuits = circuit.getAll();
  for (const [name, state] of circuits) {
    if (state === 'OPEN') {
      console.log(`✅ ${name} 熔断器正确触发 OPEN`);
    }
  }

  // 5. 关闭故障，观察自愈
  chaos.disable();
  await sleep(5000);
  const recovered = await measureSteadyState(agent, testCases.slice(0, 5));
  console.log('恢复后指标:', recovered);

  // 6. 对比报告
  printReport(baseline, underChaos, recovered);
}

async function measureSteadyState(agent: Agent, cases: string[]) {
  const results = await Promise.allSettled(cases.map(c => agent.run(c)));
  const success = results.filter(r => r.status === 'fulfilled').length;
  return {
    successRate: success / results.length,
    // 更多指标...
  };
}
```

---

## OpenClaw 集成：Cron 定期混沌实验

```typescript
// 在 OpenClaw Cron 中每周跑一次混沌实验（非高峰期）
{
  "name": "weekly-chaos-game-day",
  "schedule": { "kind": "cron", "expr": "0 3 * * 0", "tz": "Asia/Shanghai" },
  "payload": {
    "kind": "agentTurn",
    "message": "运行混沌游戏日：注入随机故障，验证熔断/降级/告警是否正常，输出报告",
    "timeoutSeconds": 600
  },
  "sessionTarget": "isolated"
}
```

---

## Python 版本（asyncio）

```python
# chaos_middleware.py
import asyncio, random, re
from dataclasses import dataclass, field
from typing import Callable, Awaitable, Any

@dataclass
class ChaosScenario:
    pattern: str
    kind: str  # delay | error | timeout | partial
    probability: float
    delay_ms: int = 3000
    error_msg: str = "chaos fault"

class ChaosMiddleware:
    def __init__(self):
        self._scenarios: list[ChaosScenario] = []
        self._enabled = False
        self._log: list[dict] = []

    def enable(self, scenarios: list[ChaosScenario]):
        self._scenarios = scenarios
        self._enabled = True

    def disable(self):
        self._enabled = False

    async def wrap(self, tool_name: str, call: Callable[[], Awaitable[Any]]) -> Any:
        if not self._enabled:
            return await call()

        for s in self._scenarios:
            if not re.search(s.pattern, tool_name): continue
            if random.random() > s.probability: continue

            self._log.append({"tool": tool_name, "scenario": s.kind})

            if s.kind == "delay":
                await asyncio.sleep(s.delay_ms / 1000)
                return await call()
            elif s.kind == "error":
                raise RuntimeError(s.error_msg)
            elif s.kind == "timeout":
                await asyncio.sleep(9999)

        return await call()
```

---

## 混沌实验检查清单

```
故障注入前：
□ 通知团队（非生产环境优先）
□ 记录当前稳态指标（成功率/延迟）
□ 准备回滚方案

注入中观察：
□ 熔断器是否在预期阈值触发？
□ 降级路径是否被走到？
□ 告警是否在预期时间内响应？
□ 用户/调用方是否感知到异常？

恢复后验证：
□ 熔断器是否自动恢复 CLOSED？
□ 成功率是否回到基线？
□ 有无僵尸连接/内存泄漏？
```

---

## 与已讲内容的关联

| 课程 | 混沌工程验证的能力 |
|------|------------------|
| Lesson 73 Circuit Breaker | 熔断阈值是否合理 |
| Lesson 82 Health Check | 探针能否检测混沌故障 |
| Lesson 97 Adaptive Timeout | 超时策略在故障下的表现 |
| Lesson 69 Graceful Degradation | 降级路径是否真的被走到 |
| Lesson 88 OpenTelemetry | 混沌注入的 Span 是否可追踪 |
| Lesson 101 Tool Mocking | 混沌 ≠ Mock，前者面向生产，后者面向测试 |

---

## 核心要点

1. **从稳态开始** — 没有基线就没有对比，先定义"正常"长什么样
2. **小爆炸原则** — 先在非核心路径/低峰期注入，逐步扩大范围
3. **自动化游戏日** — 人工混沌只做一次，Cron 混沌持续验证
4. **与告警挂钩** — 混沌实验应该触发真实告警，验证 On-Call 响应
5. **记录注入日志** — 事后可以关联：这个 P0 是混沌注入还是真实故障？
