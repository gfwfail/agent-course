# 110 - Agent Shadow Testing & Canary Validation
# 影子测试与金丝雀验证：新版 Agent 安全上线策略

> 新版 Agent 写好了，怎么知道它在生产环境不会崩？答：先当影子，再当金丝雀。

---

## 为什么需要这个？

Agent 和传统软件不同，同样的输入可能产生不同输出（LLM 非确定性）。  
你没法靠单元测试覆盖所有场景，但你又不敢直接切流量——万一新版乱调工具，损失谁来负责？

**影子测试（Shadow Testing）**：新版 Agent 并行接收真实流量，但输出不返回用户，只做对比记录。  
**金丝雀验证（Canary Validation）**：逐步放量（1% → 10% → 50% → 100%），每步监控指标，不达标自动回滚。

---

## 核心架构

```
用户请求
    │
    ▼
┌─────────────────┐
│  Traffic Router │  ← 流量路由器
└────────┬────────┘
         │
    ┌────┴────┐
    │         │
    ▼         ▼ (异步复制)
 老版 Agent  新版 Agent (Shadow)
    │         │
    │         └──→ 影子输出存 DB / 不返回用户
    │
    └──→ 真实响应返回用户
         │
         ▼
    对比分析 / 指标监控
```

---

## Python 实现（learn-claude-code 风格）

### 1. 影子测试框架

```python
# shadow_tester.py
import asyncio
import time
import json
from dataclasses import dataclass, asdict
from typing import Any, Callable, Awaitable
import anthropic

@dataclass
class ShadowResult:
    request_id: str
    input: dict
    primary_output: str
    shadow_output: str
    primary_latency_ms: float
    shadow_latency_ms: float
    tools_called_primary: list[str]
    tools_called_shadow: list[str]
    diverged: bool  # 输出差异是否超过阈值
    timestamp: float

class ShadowTester:
    """
    并行运行主版本和影子版本 Agent，对比输出差异
    影子版本的输出不返回用户
    """
    
    def __init__(self, primary_agent, shadow_agent, result_store):
        self.primary = primary_agent
        self.shadow = shadow_agent
        self.store = result_store
        self.divergence_threshold = 0.3  # 30% 差异触发告警
    
    async def handle_request(self, request_id: str, user_message: str, context: dict) -> str:
        """处理用户请求，同时在影子中运行新版 Agent"""
        
        # 并行运行两个版本
        primary_task = asyncio.create_task(
            self._run_agent(self.primary, user_message, context)
        )
        shadow_task = asyncio.create_task(
            self._run_agent(self.shadow, user_message, context)
        )
        
        # 等待两者都完成（影子超时不影响主版本响应）
        primary_start = time.time()
        primary_result = await primary_task
        primary_latency = (time.time() - primary_start) * 1000
        
        # 影子结果在后台收集，不阻塞用户响应
        asyncio.create_task(
            self._collect_shadow_result(
                request_id, user_message,
                primary_result, shadow_task,
                primary_latency
            )
        )
        
        # 立即返回主版本结果给用户
        return primary_result["output"]
    
    async def _run_agent(self, agent, message: str, context: dict) -> dict:
        """运行单个 Agent，收集工具调用记录"""
        tools_called = []
        start = time.time()
        
        try:
            output = await agent.run(message, context, tool_callback=lambda t: tools_called.append(t))
            return {
                "output": output,
                "tools": tools_called,
                "latency_ms": (time.time() - start) * 1000,
                "error": None
            }
        except Exception as e:
            return {
                "output": "",
                "tools": tools_called,
                "latency_ms": (time.time() - start) * 1000,
                "error": str(e)
            }
    
    async def _collect_shadow_result(
        self, request_id, user_message,
        primary_result, shadow_task, primary_latency
    ):
        try:
            # 影子最多等 30 秒，超时直接丢弃
            shadow_result = await asyncio.wait_for(shadow_task, timeout=30)
        except asyncio.TimeoutError:
            shadow_result = {"output": "", "tools": [], "latency_ms": 30000, "error": "timeout"}
        
        # 计算差异度
        diverged = self._calculate_divergence(
            primary_result["output"],
            shadow_result["output"]
        )
        
        record = ShadowResult(
            request_id=request_id,
            input={"message": user_message},
            primary_output=primary_result["output"],
            shadow_output=shadow_result["output"],
            primary_latency_ms=primary_latency,
            shadow_latency_ms=shadow_result["latency_ms"],
            tools_called_primary=primary_result["tools"],
            tools_called_shadow=shadow_result["tools"],
            diverged=diverged,
            timestamp=time.time()
        )
        
        await self.store.save(record)
        
        # 高差异率告警
        if diverged:
            await self._alert(record)
    
    def _calculate_divergence(self, primary: str, shadow: str) -> bool:
        """简单的差异检测：工具调用序列不同 or 语义差异大"""
        if not primary and not shadow:
            return False
        if not primary or not shadow:
            return True
        
        # 长度差异超过 50% 认为差异显著
        len_ratio = abs(len(primary) - len(shadow)) / max(len(primary), len(shadow))
        return len_ratio > self.divergence_threshold
    
    async def _alert(self, result: ShadowResult):
        print(f"⚠️ Shadow divergence detected: {result.request_id}")
        print(f"   Primary tools: {result.tools_called_primary}")
        print(f"   Shadow tools:  {result.tools_called_shadow}")
```

### 2. 金丝雀路由器

```python
# canary_router.py
import random
import asyncio
from enum import Enum

class CanaryStage(Enum):
    SHADOW   = 0    # 影子模式（0% 真实流量）
    CANARY_1 = 1    # 1% 流量
    CANARY_10 = 10  # 10% 流量
    CANARY_50 = 50  # 50% 流量  
    FULL     = 100  # 全量

@dataclass
class CanaryMetrics:
    total_requests: int = 0
    new_version_requests: int = 0
    new_version_errors: int = 0
    new_version_p99_latency_ms: float = 0
    user_satisfaction_score: float = 1.0  # 0-1

class CanaryRouter:
    """
    金丝雀发布路由器
    自动监控指标，不达标则回滚
    """
    
    def __init__(self, primary_agent, new_agent, metrics_store):
        self.primary = primary_agent
        self.new = new_agent
        self.metrics = metrics_store
        self.stage = CanaryStage.SHADOW
        self.shadow_tester = ShadowTester(primary_agent, new_agent, metrics_store)
        
        # 晋级阈值
        self.thresholds = {
            "max_error_rate": 0.02,       # 错误率 < 2%
            "max_latency_p99_ms": 5000,   # P99 < 5s
            "min_satisfaction": 0.85,     # 满意度 > 85%
            "min_samples": 100,           # 至少 100 个样本才评估
        }
    
    async def route(self, request_id: str, message: str, context: dict) -> str:
        """根据当前金丝雀阶段路由请求"""
        
        if self.stage == CanaryStage.SHADOW:
            # 影子模式：全部走主版本，新版并行但不用
            return await self.shadow_tester.handle_request(request_id, message, context)
        
        elif self.stage == CanaryStage.FULL:
            # 全量：直接走新版
            return await self._run_new(request_id, message, context)
        
        else:
            # 金丝雀：按比例分流
            canary_pct = self.stage.value
            if random.random() * 100 < canary_pct:
                return await self._run_new(request_id, message, context)
            else:
                return await self._run_primary(message, context)
    
    async def _run_new(self, request_id: str, message: str, context: dict) -> str:
        start = time.time()
        try:
            result = await self.new.run(message, context)
            latency = (time.time() - start) * 1000
            await self.metrics.record_new_version(request_id, success=True, latency_ms=latency)
            return result
        except Exception as e:
            await self.metrics.record_new_version(request_id, success=False, latency_ms=0)
            # 回退到主版本
            return await self._run_primary(message, context)
    
    async def _run_primary(self, message: str, context: dict) -> str:
        return await self.primary.run(message, context)
    
    async def try_promote(self) -> bool:
        """尝试晋级到下一个阶段"""
        metrics = await self.metrics.get_current_metrics()
        
        if metrics.new_version_requests < self.thresholds["min_samples"]:
            print(f"⏳ 样本不足 ({metrics.new_version_requests}/{self.thresholds['min_samples']})")
            return False
        
        error_rate = metrics.new_version_errors / max(metrics.new_version_requests, 1)
        
        if error_rate > self.thresholds["max_error_rate"]:
            print(f"❌ 错误率过高: {error_rate:.1%} > {self.thresholds['max_error_rate']:.1%}")
            await self.rollback()
            return False
        
        if metrics.new_version_p99_latency_ms > self.thresholds["max_latency_p99_ms"]:
            print(f"❌ 延迟过高: {metrics.new_version_p99_latency_ms}ms")
            await self.rollback()
            return False
        
        # 晋级！
        stages = list(CanaryStage)
        current_idx = stages.index(self.stage)
        if current_idx < len(stages) - 1:
            self.stage = stages[current_idx + 1]
            print(f"✅ 晋级到: {self.stage.name} ({self.stage.value}%流量)")
            return True
        
        return False
    
    async def rollback(self):
        """回滚到影子模式"""
        self.stage = CanaryStage.SHADOW
        print("🔄 回滚到影子模式！检查 metrics 后手动晋级")
```

---

## TypeScript 实现（pi-mono 风格）

```typescript
// shadow-canary.ts

interface AgentResult {
  output: string;
  toolsCalled: string[];
  latencyMs: number;
  error?: string;
}

interface ShadowRecord {
  requestId: string;
  primary: AgentResult;
  shadow: AgentResult;
  diverged: boolean;
  timestamp: number;
}

class ShadowCanarySystem {
  private stage: number = 0; // 0=shadow, 1, 10, 50, 100
  private readonly stages = [0, 1, 10, 50, 100];
  private metrics = { total: 0, errors: 0, latencies: [] as number[] };

  constructor(
    private primaryAgent: Agent,
    private newAgent: Agent,
    private store: ResultStore
  ) {}

  async handle(requestId: string, message: string): Promise<string> {
    this.metrics.total++;

    if (this.stage === 0) {
      // Shadow mode
      const [primary, shadow] = await Promise.allSettled([
        this.runAgent(this.primaryAgent, message),
        this.runAgent(this.newAgent, message),
      ]);

      const primaryResult = primary.status === 'fulfilled' 
        ? primary.value 
        : { output: '', toolsCalled: [], latencyMs: 0, error: String(primary.reason) };
      
      const shadowResult = shadow.status === 'fulfilled'
        ? shadow.value
        : { output: '', toolsCalled: [], latencyMs: 0, error: String(shadow.reason) };

      // 异步存储对比结果
      this.store.save({
        requestId,
        primary: primaryResult,
        shadow: shadowResult,
        diverged: this.isDiverged(primaryResult.output, shadowResult.output),
        timestamp: Date.now(),
      }).catch(console.error);

      return primaryResult.output;
    }

    // Canary mode
    const useNew = Math.random() * 100 < this.stage;
    if (useNew) {
      try {
        const result = await this.runAgent(this.newAgent, message);
        this.metrics.latencies.push(result.latencyMs);
        return result.output;
      } catch {
        this.metrics.errors++;
        return this.runAgent(this.primaryAgent, message).then(r => r.output);
      }
    }

    return this.runAgent(this.primaryAgent, message).then(r => r.output);
  }

  private async runAgent(agent: Agent, message: string): Promise<AgentResult> {
    const toolsCalled: string[] = [];
    const start = Date.now();
    try {
      const output = await agent.run(message, {
        onToolCall: (name: string) => toolsCalled.push(name),
      });
      return { output, toolsCalled, latencyMs: Date.now() - start };
    } catch (e) {
      return { output: '', toolsCalled, latencyMs: Date.now() - start, error: String(e) };
    }
  }

  private isDiverged(a: string, b: string): boolean {
    if (!a && !b) return false;
    if (!a || !b) return true;
    const longer = Math.max(a.length, b.length);
    return Math.abs(a.length - b.length) / longer > 0.3;
  }

  async promote(): Promise<boolean> {
    const errorRate = this.metrics.errors / Math.max(this.metrics.total, 1);
    const p99 = this.getP99(this.metrics.latencies);

    if (errorRate > 0.02 || p99 > 5000) {
      console.log(`🔄 Rollback! error=${(errorRate * 100).toFixed(1)}% p99=${p99}ms`);
      this.stage = 0;
      return false;
    }

    const idx = this.stages.indexOf(this.stage);
    if (idx < this.stages.length - 1) {
      this.stage = this.stages[idx + 1];
      console.log(`✅ Promoted to ${this.stage}% canary`);
      return true;
    }
    return false;
  }

  private getP99(latencies: number[]): number {
    if (!latencies.length) return 0;
    const sorted = [...latencies].sort((a, b) => a - b);
    return sorted[Math.floor(sorted.length * 0.99)];
  }
}
```

---

## OpenClaw 实战：Cron 自动晋级

```yaml
# 每 30 分钟检查金丝雀指标，自动晋级或回滚
schedule:
  kind: "every"
  everyMs: 1800000

payload:
  kind: "agentTurn"
  message: |
    检查金丝雀 Agent 指标：
    1. 从 metrics store 获取过去 30 分钟数据
    2. 计算错误率、P99 延迟、用户满意度
    3. 如达标且样本 >= 100，调用 canary_router.promote()
    4. 如不达标，调用 canary_router.rollback()
    5. 发送状态报告到监控频道
```

```python
# OpenClaw cron job 里调用的实际检查逻辑
async def canary_health_check():
    router = get_canary_router()  # 从全局状态获取
    metrics = await router.metrics.get_last_30min()
    
    report = {
        "stage": router.stage.name,
        "error_rate": f"{metrics.error_rate:.2%}",
        "p99_ms": metrics.p99_latency_ms,
        "samples": metrics.total_requests,
    }
    
    promoted = await router.try_promote()
    
    msg = f"""
🐤 金丝雀状态报告
阶段: {report['stage']} → {"晋级✅" if promoted else "维持/回滚🔄"}
错误率: {report['error_rate']}
P99延迟: {report['p99_ms']}ms
样本量: {report['samples']}
    """
    
    await notify_channel(msg)
```

---

## 关键决策点

| 场景 | 策略 |
|------|------|
| 大改动（换模型/重构工具）| 先跑 1 周影子，积累 10k 样本再放量 |
| 小改动（调 prompt）| 直接 1% 金丝雀，24h 后晋级 |
| 紧急修复 | 跳过影子，直接 10% 金丝雀，快速晋级 |
| 回滚后 | 必须分析 shadow records 找根因再重来 |

---

## 总结

```
影子测试 → 零风险观察新版行为
金丝雀发布 → 最小化爆炸半径
自动晋级/回滚 → 无需人工盯着
对比分析 → 持续改进新版 Agent
```

**核心原则**：新版 Agent 先当影子学徒，用数据证明自己，再逐步接管流量。  
不要赌，要用数据说话。

---

*第 110 课 | Agent 开发课程 | 2026-03-22*
