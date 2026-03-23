# 124 - Agent 无限循环检测与逃出机制（Infinite Loop Detection & Escape）

> Agent 卡住了……它一直在调同一个工具，或者每次重试都失败，或者在推理里兜圈子。  
> **你要么提前发现，要么让它自己逃出来，否则就是一笔永无止境的账单。**

---

## 为什么 Agent 会死循环？

| 循环类型 | 典型场景 | 后果 |
|---------|---------|------|
| **工具循环** | 反复调用同一工具（结果相同→LLM 继续调） | Token 爆炸 |
| **重试循环** | 错误处理触发重试，重试又触发错误 | 账单无底洞 |
| **推理循环** | LLM 在同一问题上兜圈（"我需要先知道X→X依赖Y→Y依赖X"） | 超时挂死 |
| **错误链循环** | 工具 A 失败→触发工具 B→B 也失败→回到 A | 无效资源消耗 |
| **子 Agent 循环** | Sub-agent 委托 Parent→Parent 委托回 Sub-agent | 递归炸弹 |

---

## 核心检测策略

### 1. 哈希去重（相同调用检测）

最简单有效：对每次工具调用计算指纹，发现重复立即拦截。

```typescript
// pi-mono 风格：工具调用指纹中间件
import { createHash } from 'crypto'

interface ToolCallRecord {
  hash: string
  count: number
  firstSeen: number
  lastSeen: number
}

class LoopDetector {
  private callHistory = new Map<string, ToolCallRecord>()
  private readonly MAX_DUPLICATE_CALLS = 3
  private readonly WINDOW_MS = 60_000 // 1 分钟滑动窗口

  // 计算工具调用指纹
  fingerprint(toolName: string, args: Record<string, unknown>): string {
    const payload = JSON.stringify({ toolName, args }, Object.keys(args).sort())
    return createHash('sha256').update(payload).digest('hex').slice(0, 16)
  }

  check(toolName: string, args: Record<string, unknown>): void {
    const hash = this.fingerprint(toolName, args)
    const now = Date.now()

    // 清理过期记录
    for (const [key, record] of this.callHistory) {
      if (now - record.lastSeen > this.WINDOW_MS) {
        this.callHistory.delete(key)
      }
    }

    const existing = this.callHistory.get(hash)
    if (existing) {
      existing.count++
      existing.lastSeen = now

      if (existing.count >= this.MAX_DUPLICATE_CALLS) {
        throw new LoopDetectedError(
          `工具循环检测：${toolName} 在 ${this.WINDOW_MS / 1000}s 内被相同参数调用了 ${existing.count} 次`,
          { toolName, args, hash, count: existing.count }
        )
      }
    } else {
      this.callHistory.set(hash, { hash, count: 1, firstSeen: now, lastSeen: now })
    }
  }

  reset(): void {
    this.callHistory.clear()
  }
}

class LoopDetectedError extends Error {
  constructor(message: string, public readonly context: Record<string, unknown>) {
    super(message)
    this.name = 'LoopDetectedError'
  }
}
```

---

### 2. 迭代计数器（硬上限）

最粗暴但最可靠的保险：无论什么原因，迭代超限就停。

```python
# learn-claude-code 风格：Python Agent Loop 计数器
from dataclasses import dataclass, field
from typing import Optional
import time

@dataclass
class LoopBudget:
    max_iterations: int = 50          # 最大迭代轮次
    max_tool_calls: int = 100         # 最大工具调用次数
    max_duration_seconds: float = 300 # 最大执行时间（5分钟）
    warn_at_percent: float = 0.8      # 达到 80% 时发出警告

    _iterations: int = field(default=0, init=False)
    _tool_calls: int = field(default=0, init=False)
    _start_time: float = field(default_factory=time.time, init=False)

    def tick_iteration(self) -> None:
        self._iterations += 1
        self._check_limits("iterations", self._iterations, self.max_iterations)

    def tick_tool_call(self) -> None:
        self._tool_calls += 1
        self._check_limits("tool_calls", self._tool_calls, self.max_tool_calls)

    def _check_limits(self, label: str, current: int, limit: int) -> None:
        # 时间检查
        elapsed = time.time() - self._start_time
        if elapsed > self.max_duration_seconds:
            raise LoopBudgetExceeded(
                f"执行时间超限：{elapsed:.1f}s > {self.max_duration_seconds}s"
            )

        # 软限制警告
        if current >= int(limit * self.warn_at_percent) and current % 5 == 0:
            print(f"⚠️  Loop budget warning: {label} = {current}/{limit} ({current/limit*100:.0f}%)")

        # 硬限制
        if current >= limit:
            raise LoopBudgetExceeded(
                f"Agent 超出预算：{label} = {current} >= {limit}",
                stats=self.stats()
            )

    def stats(self) -> dict:
        return {
            "iterations": self._iterations,
            "tool_calls": self._tool_calls,
            "elapsed_seconds": round(time.time() - self._start_time, 2),
        }

class LoopBudgetExceeded(Exception):
    def __init__(self, message: str, stats: dict = None):
        super().__init__(message)
        self.stats = stats or {}
```

---

### 3. 相似度检测（推理循环）

工具参数可能有细微差异（比如 offset 递增）但语义相同，哈希检测会漏掉。用向量相似度或简单的文本相似度来捕捉。

```typescript
// 简化版：基于 Jaccard 相似度的语义去重
function jaccardSimilarity(a: string, b: string): number {
  const setA = new Set(a.toLowerCase().split(/\s+/))
  const setB = new Set(b.toLowerCase().split(/\s+/))
  const intersection = new Set([...setA].filter(x => setB.has(x)))
  const union = new Set([...setA, ...setB])
  return intersection.size / union.size
}

class SemanticLoopDetector {
  private recentOutputs: string[] = []
  private readonly WINDOW = 5          // 检查最近 5 条 LLM 输出
  private readonly THRESHOLD = 0.85    // 85% 相似即视为重复

  checkLLMOutput(output: string): void {
    for (const prev of this.recentOutputs) {
      const sim = jaccardSimilarity(output, prev)
      if (sim >= this.THRESHOLD) {
        throw new LoopDetectedError(
          `推理循环检测：LLM 输出与之前内容相似度 ${(sim * 100).toFixed(0)}%`,
          { similarity: sim, threshold: this.THRESHOLD }
        )
      }
    }

    this.recentOutputs.push(output)
    if (this.recentOutputs.length > this.WINDOW) {
      this.recentOutputs.shift()
    }
  }
}
```

---

### 4. 调用图环检测（子 Agent 循环）

多 Agent 场景下，防止 A→B→A 的委托死循环：

```typescript
class AgentCallGraph {
  // 有向图：caller → Set<callee>
  private graph = new Map<string, Set<string>>()

  recordCall(callerId: string, calleeId: string): void {
    if (!this.graph.has(callerId)) {
      this.graph.set(callerId, new Set())
    }
    this.graph.get(callerId)!.add(calleeId)

    // DFS 检测环
    if (this.hasCycle(calleeId, callerId, new Set())) {
      throw new LoopDetectedError(
        `Sub-agent 循环检测：${callerId} → ${calleeId} 形成环路`,
        { callerId, calleeId }
      )
    }
  }

  private hasCycle(current: string, target: string, visited: Set<string>): boolean {
    if (current === target) return true
    if (visited.has(current)) return false

    visited.add(current)
    const callees = this.graph.get(current) ?? new Set()
    for (const callee of callees) {
      if (this.hasCycle(callee, target, visited)) return true
    }
    return false
  }
}
```

---

## 逃出机制：检测到循环后怎么办？

检测到死循环只是第一步，**关键是优雅退出**，而不是直接崩溃。

```typescript
// 三层逃出策略
type EscapeStrategy = 'summarize' | 'escalate' | 'abort'

interface EscapeConfig {
  onToolLoop: EscapeStrategy   // 工具循环 → 默认 summarize
  onBudgetExceeded: EscapeStrategy  // 超预算 → 默认 escalate
  onTimeout: EscapeStrategy    // 超时 → 默认 abort
}

class AgentLoopEscapeHandler {
  constructor(
    private config: EscapeConfig = {
      onToolLoop: 'summarize',
      onBudgetExceeded: 'escalate',
      onTimeout: 'abort',
    }
  ) {}

  async handle(
    error: LoopDetectedError | LoopBudgetExceeded,
    context: AgentContext
  ): Promise<AgentResult> {
    const strategy = this.selectStrategy(error)
    console.warn(`🔄 Loop escape: ${strategy}`, { error: error.message })

    switch (strategy) {
      case 'summarize':
        // 让 LLM 根据已有信息给出当前最佳答案
        return this.summarizeAndReturn(context)

      case 'escalate':
        // 转交人工，保留完整上下文
        return this.escalateToHuman(context, error)

      case 'abort':
        // 直接终止，返回错误结果
        return {
          success: false,
          error: error.message,
          partial: context.getPartialResult(),
        }
    }
  }

  private async summarizeAndReturn(context: AgentContext): Promise<AgentResult> {
    // 注入特殊系统消息，强制 LLM 基于已有信息作答
    const summaryPrompt = `
    你陷入了循环。请停止继续尝试，基于目前已收集的信息给出最佳答案。
    如果信息不足，明确说明缺少什么，让用户决定下一步。
    不要再调用任何工具。
    `
    return context.runWithInjectedMessage(summaryPrompt)
  }

  private async escalateToHuman(
    context: AgentContext,
    error: Error
  ): Promise<AgentResult> {
    await context.notify({
      channel: 'telegram',
      message: `⚠️ Agent 需要帮助\n\n原因：${error.message}\n\n任务：${context.taskDescription}`,
    })
    return { success: false, escalated: true, error: error.message }
  }

  private selectStrategy(error: Error): EscapeStrategy {
    if (error instanceof LoopDetectedError) return this.config.onToolLoop
    if (error instanceof LoopBudgetExceeded) return this.config.onBudgetExceeded
    return this.config.onTimeout
  }
}
```

---

## 完整集成：把所有检测器组合成中间件

```typescript
// OpenClaw / pi-mono 风格：LoopGuard 中间件
class LoopGuard {
  private hashDetector = new LoopDetector()
  private semanticDetector = new SemanticLoopDetector()
  private callGraph = new AgentCallGraph()
  private budget: LoopBudget
  private escapeHandler: AgentLoopEscapeHandler

  constructor(config?: Partial<LoopBudgetConfig>) {
    this.budget = new LoopBudget(config)
    this.escapeHandler = new AgentLoopEscapeHandler()
  }

  // 在每次工具调用前调用
  beforeToolCall(toolName: string, args: Record<string, unknown>): void {
    this.budget.tick_tool_call()
    this.hashDetector.check(toolName, args)
  }

  // 在每次 LLM 输出后调用
  afterLLMOutput(output: string): void {
    this.budget.tick_iteration()
    this.semanticDetector.checkLLMOutput(output)
  }

  // 在 Sub-agent 调用前调用
  beforeSubagentCall(callerId: string, calleeId: string): void {
    this.callGraph.recordCall(callerId, calleeId)
  }

  // 重置（新任务开始时）
  reset(): void {
    this.hashDetector.reset()
    // budget 和 semantic detector 重新实例化
  }
}

// 在 Agent Loop 中使用
async function runAgentWithGuard(task: string, tools: Tool[]): Promise<AgentResult> {
  const guard = new LoopGuard({ max_iterations: 30, max_tool_calls: 60 })

  try {
    return await agentLoop(task, tools, {
      beforeToolCall: (name, args) => guard.beforeToolCall(name, args),
      afterLLMOutput: (output) => guard.afterLLMOutput(output),
    })
  } catch (error) {
    if (error instanceof LoopDetectedError || error instanceof LoopBudgetExceeded) {
      return escapeHandler.handle(error, context)
    }
    throw error
  }
}
```

---

## OpenClaw 实战：Cron Agent 的循环保护

OpenClaw 的 Cron 任务天然有循环风险——如果任务没完成就下一个触发点来了：

```typescript
// OpenClaw Cron Job 配置加上循环保护
{
  "schedule": { "kind": "every", "everyMs": 10800000 }, // 每3小时
  "payload": {
    "kind": "agentTurn",
    "message": "执行定时任务...",
    "timeoutSeconds": 300  // ← 硬超时！超时直接终止，不会和下次重叠
  },
  "sessionTarget": "isolated"  // ← 隔离会话，不共享状态，天然防止跨任务循环
}
```

**关键设计原则：**
- `timeoutSeconds` < 触发间隔（300s << 10800s）
- `isolated` 会话：每次都是全新 Agent，不携带上一次的"执行惯性"
- Cron 任务本身不重试（失败就失败，等下一次触发）

---

## 检测指标监控

```typescript
// 接入第 123 课的监控体系
const loopMetrics = {
  loopsDetected: new Counter({
    name: 'agent_loops_detected_total',
    labelNames: ['type', 'escape_strategy'],
  }),
  loopEscapeLatency: new Histogram({
    name: 'agent_loop_escape_duration_ms',
    labelNames: ['strategy'],
  }),
}

// 告警规则
// loops_detected_total > 5 per minute → PagerDuty
// escape_strategy=abort 出现 → 立即告警
```

---

## 快速参考：选哪种检测？

```
工具调用重复     → 哈希去重（最常见，首选）
LLM 推理兜圈    → 语义相似度（配合哈希）
时间/次数超限   → LoopBudget（必须有，兜底保险）
Sub-agent 环路  → 调用图 DFS（多 Agent 架构必备）
Cron 任务重叠   → timeoutSeconds + isolated session
```

---

## 总结

| 策略 | 实现复杂度 | 覆盖场景 | 推荐优先级 |
|------|-----------|---------|----------|
| LoopBudget（硬上限） | ⭐ 极简 | 所有循环 | 🔴 必须有 |
| 哈希去重 | ⭐⭐ 简单 | 工具循环 | 🔴 必须有 |
| 语义相似度 | ⭐⭐⭐ 中等 | 推理循环 | 🟡 建议加 |
| 调用图检测 | ⭐⭐⭐ 中等 | Multi-agent | 🟡 多 Agent 必备 |
| 逃出 + 摘要 | ⭐⭐ 简单 | 优雅降级 | 🟢 用户体验加分 |

**一句话原则：先加硬上限保命，再加哈希去重省钱，最后加语义检测提质量。**
