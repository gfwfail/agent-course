# 146 - Agent 混沌工程与弹性测试（Chaos Engineering & Resilience Testing）

> 你的 Agent 能扛住真实世界的混乱吗？

---

## 为什么需要混沌工程？

你做了熔断器、重试、超时、健康检查……但你怎么知道它们真的有效？

**混沌工程**的核心思想：**主动向系统注入故障，在生产前找到弱点**。

Netflix 的 Chaos Monkey、AWS 的 GameDay 都是这个思路。Agent 系统同样需要——工具超时、LLM 限速、Sub-agent 崩溃、网络抖动，这些不是"如果"而是"什么时候"。

---

## 故障注入的四个维度

```
┌─────────────────────────────────────────────────────┐
│              Agent Chaos Taxonomy                   │
├─────────────────────┬───────────────────────────────┤
│  工具层故障          │  LLM 层故障                   │
│  - 超时              │  - 限速 (429)                 │
│  - 随机错误          │  - 响应截断                   │
│  - 返回坏数据        │  - 幻觉工具调用               │
│  - 永久不响应        │  - 模型切换                   │
├─────────────────────┼───────────────────────────────┤
│  Sub-agent 层故障   │  状态层故障                   │
│  - 崩溃             │  - Redis 断线                 │
│  - 返回错误结果     │  - 内存溢出                   │
│  - 响应超慢         │  - 检查点损坏                 │
└─────────────────────┴───────────────────────────────┘
```

---

## TypeScript 实现：ChaosToolRegistry

```typescript
// chaos/ChaosToolRegistry.ts
interface ChaosConfig {
  toolName: string;
  fault: FaultSpec;
  probability: number;   // 0.0 ~ 1.0，触发概率
  enabled: boolean;
}

type FaultSpec =
  | { kind: 'timeout'; delayMs: number }
  | { kind: 'error'; message: string; code?: string }
  | { kind: 'corrupt'; transform: (result: unknown) => unknown }
  | { kind: 'slow'; delayMs: number }  // 慢但不失败
  | { kind: 'flaky'; errorRate: number };  // 随机错误

class ChaosToolRegistry {
  private configs: Map<string, ChaosConfig> = new Map();
  private metrics: Map<string, ChaosMetrics> = new Map();

  register(config: ChaosConfig) {
    this.configs.set(config.toolName, config);
    this.metrics.set(config.toolName, { injected: 0, passed: 0 });
  }

  // 包装原始工具，注入混沌
  wrap<T extends Record<string, unknown>>(
    toolName: string,
    handler: (params: T) => Promise<unknown>
  ): (params: T) => Promise<unknown> {
    return async (params: T) => {
      const config = this.configs.get(toolName);
      const metrics = this.metrics.get(toolName)!;

      // 不在混沌配置里，或未启用，直接执行
      if (!config || !config.enabled) {
        return handler(params);
      }

      // 概率触发
      if (Math.random() > config.probability) {
        metrics.passed++;
        return handler(params);
      }

      metrics.injected++;
      console.log(`🔥 [Chaos] Injecting ${config.fault.kind} into ${toolName}`);

      return this.applyFault(config.fault, handler, params);
    };
  }

  private async applyFault(
    fault: FaultSpec,
    handler: (params: unknown) => Promise<unknown>,
    params: unknown
  ): Promise<unknown> {
    switch (fault.kind) {
      case 'timeout':
        await new Promise(r => setTimeout(r, fault.delayMs));
        throw new Error(`Tool timeout after ${fault.delayMs}ms`);

      case 'error':
        throw Object.assign(new Error(fault.message), { code: fault.code });

      case 'slow':
        await new Promise(r => setTimeout(r, fault.delayMs));
        return handler(params);  // 慢但成功

      case 'corrupt':
        const result = await handler(params);
        return fault.transform(result);  // 返回脏数据

      case 'flaky':
        if (Math.random() < fault.errorRate) {
          throw new Error('Flaky tool error');
        }
        return handler(params);
    }
  }

  getReport(): ChaosReport {
    const report: ChaosReport = { tools: {} };
    for (const [name, m] of this.metrics) {
      const total = m.injected + m.passed;
      report.tools[name] = {
        injected: m.injected,
        passed: m.passed,
        injectionRate: total > 0 ? m.injected / total : 0,
      };
    }
    return report;
  }
}
```

---

## 弹性测试套件

```typescript
// chaos/ResilienceTestSuite.ts
import { expect } from 'vitest';

interface ResilienceScenario {
  name: string;
  setup: (chaos: ChaosToolRegistry) => void;
  run: (agent: Agent) => Promise<AgentResult>;
  assert: (result: AgentResult) => void;
}

const RESILIENCE_SCENARIOS: ResilienceScenario[] = [
  {
    name: '工具超时 - Agent 应该重试并成功',
    setup: (chaos) => {
      // 前两次超时，第三次成功
      let callCount = 0;
      chaos.register({
        toolName: 'web_search',
        fault: { kind: 'flaky', errorRate: 0.66 },
        probability: 1.0,
        enabled: true,
      });
    },
    run: (agent) => agent.run('搜索 TypeScript 最佳实践'),
    assert: (result) => {
      expect(result.success).toBe(true);
      expect(result.retryCount).toBeGreaterThan(0);
    },
  },

  {
    name: 'LLM 限速 - Agent 应该等待后继续',
    setup: (chaos) => {
      chaos.register({
        toolName: '__llm_call',  // 特殊钩子
        fault: { kind: 'error', message: 'Rate limited', code: '429' },
        probability: 0.5,
        enabled: true,
      });
    },
    run: (agent) => agent.run('分析这段代码'),
    assert: (result) => {
      expect(result.success).toBe(true);
      expect(result.totalDurationMs).toBeLessThan(30_000);
    },
  },

  {
    name: '工具返回脏数据 - Agent 应该检测并降级',
    setup: (chaos) => {
      chaos.register({
        toolName: 'database_query',
        fault: {
          kind: 'corrupt',
          transform: () => ({ rows: null, error: 'CORRUPTED' }),
        },
        probability: 1.0,
        enabled: true,
      });
    },
    run: (agent) => agent.run('查询用户总数'),
    assert: (result) => {
      // Agent 应该识别到数据异常，回答用户"数据暂时不可用"
      expect(result.output).toMatch(/不可用|异常|稍后再试/);
      expect(result.success).toBe(true);  // 软失败，不崩溃
    },
  },

  {
    name: '所有工具宕机 - Agent 应该优雅降级',
    setup: (chaos) => {
      for (const tool of ['web_search', 'code_exec', 'database_query']) {
        chaos.register({
          toolName: tool,
          fault: { kind: 'error', message: 'Service unavailable' },
          probability: 1.0,
          enabled: true,
        });
      }
    },
    run: (agent) => agent.run('帮我写一个排序算法'),
    assert: (result) => {
      // 没有工具，Agent 仍然能用自身知识回答
      expect(result.output.length).toBeGreaterThan(100);
      expect(result.crashed).toBe(false);
    },
  },
];

async function runResilienceSuite(agent: Agent): Promise<SuiteReport> {
  const chaos = new ChaosToolRegistry();
  const results: ScenarioResult[] = [];

  for (const scenario of RESILIENCE_SCENARIOS) {
    console.log(`\n🧪 Running: ${scenario.name}`);
    chaos.clear();
    scenario.setup(chaos);
    agent.setChaosRegistry(chaos);

    try {
      const result = await scenario.run(agent);
      scenario.assert(result);
      results.push({ name: scenario.name, passed: true });
      console.log(`  ✅ PASSED`);
    } catch (err) {
      results.push({ name: scenario.name, passed: false, error: String(err) });
      console.log(`  ❌ FAILED: ${err}`);
    }
  }

  return {
    total: results.length,
    passed: results.filter(r => r.passed).length,
    failed: results.filter(r => !r.passed).length,
    details: results,
    chaosReport: chaos.getReport(),
  };
}
```

---

## Python 版本（learn-claude-code 风格）

```python
# chaos_engineering.py
import asyncio
import random
from dataclasses import dataclass, field
from typing import Any, Callable, Awaitable
from functools import wraps

@dataclass
class FaultConfig:
    fault_type: str  # 'timeout' | 'error' | 'slow' | 'corrupt' | 'flaky'
    probability: float = 1.0
    delay_ms: int = 0
    error_message: str = "Chaos fault injected"
    error_rate: float = 0.5
    transform: Callable = None
    enabled: bool = True

class ChaosMiddleware:
    """工具中间件：透明注入故障，不修改工具本身"""
    
    def __init__(self):
        self.faults: dict[str, FaultConfig] = {}
        self.stats: dict[str, dict] = {}
    
    def register_fault(self, tool_name: str, config: FaultConfig):
        self.faults[tool_name] = config
        self.stats[tool_name] = {"injected": 0, "passed": 0}
    
    def wrap(self, tool_name: str, func: Callable) -> Callable:
        @wraps(func)
        async def wrapper(*args, **kwargs):
            config = self.faults.get(tool_name)
            stats = self.stats.get(tool_name, {"injected": 0, "passed": 0})
            
            if not config or not config.enabled:
                return await func(*args, **kwargs)
            
            if random.random() > config.probability:
                stats["passed"] += 1
                return await func(*args, **kwargs)
            
            stats["injected"] += 1
            print(f"🔥 [Chaos] Injecting {config.fault_type} into {tool_name}")
            
            match config.fault_type:
                case "timeout":
                    await asyncio.sleep(config.delay_ms / 1000)
                    raise TimeoutError(f"Tool {tool_name} timed out")
                
                case "slow":
                    await asyncio.sleep(config.delay_ms / 1000)
                    return await func(*args, **kwargs)
                
                case "error":
                    raise RuntimeError(config.error_message)
                
                case "flaky":
                    if random.random() < config.error_rate:
                        raise RuntimeError(f"Flaky: {tool_name} failed randomly")
                    return await func(*args, **kwargs)
                
                case "corrupt":
                    result = await func(*args, **kwargs)
                    return config.transform(result) if config.transform else None
        
        return wrapper


# 在 Agent Loop 中集成
class ResilientAgentLoop:
    def __init__(self, tools: dict, chaos: ChaosMiddleware = None):
        self.tools = tools
        self.chaos = chaos
        
        # 用混沌中间件包装所有工具
        if chaos:
            self.tools = {
                name: chaos.wrap(name, fn)
                for name, fn in tools.items()
            }
    
    async def run(self, task: str) -> dict:
        """Agent Loop with resilience built-in"""
        retry_count = 0
        max_retries = 3
        
        while retry_count <= max_retries:
            try:
                result = await self._execute(task)
                return {"success": True, "output": result, "retries": retry_count}
            except TimeoutError as e:
                retry_count += 1
                backoff = 2 ** retry_count
                print(f"⏱️ Timeout, retry {retry_count}/{max_retries} in {backoff}s")
                await asyncio.sleep(backoff)
            except RuntimeError as e:
                # 不可重试的错误，降级处理
                return {"success": False, "output": "服务暂时不可用，请稍后重试", "error": str(e)}
        
        return {"success": False, "output": "达到最大重试次数", "retries": retry_count}


# 测试示例
async def test_resilience():
    chaos = ChaosMiddleware()
    
    # 模拟 web_search 50% 概率超时
    chaos.register_fault("web_search", FaultConfig(
        fault_type="flaky",
        error_rate=0.5,
        probability=1.0
    ))
    
    # 模拟 database 返回脏数据
    chaos.register_fault("database", FaultConfig(
        fault_type="corrupt",
        transform=lambda r: {"rows": None, "corrupted": True},
        probability=0.8
    ))
    
    agent = ResilientAgentLoop(
        tools={"web_search": real_web_search, "database": real_db_query},
        chaos=chaos
    )
    
    result = await agent.run("查询最新的 TypeScript 数据")
    print(f"Result: {result}")
    print(f"Chaos stats: {chaos.stats}")
```

---

## 与 OpenClaw 集成：Cron 驱动的定期混沌测试

```typescript
// OpenClaw cron job：每天凌晨 3 点运行弹性测试
const cronJob = {
  schedule: { kind: 'cron', expr: '0 3 * * *', tz: 'Australia/Sydney' },
  payload: {
    kind: 'agentTurn',
    message: `
      运行 Agent 弹性测试套件：
      1. 加载 ChaosToolRegistry
      2. 注入随机故障（超时/错误/脏数据）
      3. 验证 Agent 弹性机制是否生效
      4. 生成测试报告
      5. 如果通过率 < 80%，立即告警到 Telegram
    `,
  },
  delivery: { mode: 'announce' },
  sessionTarget: 'isolated',
};

// 在 OpenClaw 工具策略管道中启用混沌（仅测试环境）
const toolPolicy = {
  middleware: [
    process.env.CHAOS_ENABLED === 'true' 
      ? new ChaosToolRegistry(loadChaosConfig())
      : null,
  ].filter(Boolean),
};
```

---

## 最佳实践：混沌工程的三条铁律

```
┌─────────────────────────────────────────────────────────┐
│  铁律 1：先在测试环境"爆炸"，别在生产首次发现           │
│                                                         │
│  铁律 2：每次修复弱点后，必须加对应的混沌测试用例        │
│         （防止回归）                                     │
│                                                         │
│  铁律 3：混沌测试的目标是"Agent 不崩溃"                 │
│         不是"Agent 永远成功"                            │
│         优雅降级 > 硬崩溃                               │
└─────────────────────────────────────────────────────────┘
```

### 推荐测试矩阵

| 故障类型 | 期望行为 | 验证指标 |
|---------|---------|---------|
| 工具超时 50% | 重试后成功 | `retryCount > 0 && success` |
| 所有工具宕机 | 降级到纯文本回答 | `crashed === false` |
| LLM 429 | 等待后继续 | `totalTime < maxTime` |
| 数据库返回 null | 软失败，提示用户 | `output contains 暂时不可用` |
| Sub-agent 崩溃 | 主 Agent 继续工作 | `mainAgent.alive === true` |

---

## 总结

混沌工程是 Agent 生产就绪的最后一道关卡：

- **ChaosToolRegistry**：透明注入故障，工具本身无需修改
- **弹性测试套件**：标准化场景，CI 中自动运行
- **Cron 驱动**：每天定期"折磨"自己的 Agent，提前发现问题
- **三条铁律**：先爆炸、防回归、优雅降级

> 一个没经历过混沌测试的 Agent，就像没有做过压测的服务器——你永远不知道它什么时候会倒。

---

*下一课预告：Agent 工具调用图谱与执行热点分析（Tool Call Graph & Hotspot Analysis）*
