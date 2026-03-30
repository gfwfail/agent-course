# 179. Agent 延迟预算分配与超时级联（Latency Budget Allocation & Timeout Cascading）

> **核心思想：** 用户等待的是「整体响应」，不是单个工具。给整条工具调用链分配一个总延迟预算，并随执行进度动态向下游级联——上游超时了，下游自动缩减配额，整体 SLO 始终可控。

---

## 1. 为什么需要延迟预算？

传统超时设置：每个工具各自配 `timeout: 5000ms`。问题：

```
工具A (4.9s) → 工具B (4.9s) → 工具C (4.9s) = 总耗时 14.7s ❌
用户体验：卡死
SLO：10s，全部击穿
```

**延迟预算**的思路：

```
总预算 10s
├── 工具A 分配 3s → 实际用了 4.9s（超 1.9s）
├── 工具B 分配 3s → 剩余预算 8.1-4.9=3.2s，但工具B 优先级低→压缩到 2s
└── 工具C 分配 4s → 剩余 3.2-2=1.2s，高优先级→1.2s 硬限
总耗时 ≤ 10s ✅（有些工具可能提前取消，但整体不超）
```

---

## 2. 核心数据结构

```typescript
interface ToolCallBudget {
  toolName: string;
  allocatedMs: number;    // 分配给本工具的时间
  priority: 'critical' | 'high' | 'low';
  canSkip: boolean;       // 超时后是否可以跳过（用缓存/降级结果）
}

interface LatencyBudget {
  totalMs: number;        // 总预算
  remainingMs: number;    // 剩余预算
  startTime: number;      // 整体开始时间
  tools: ToolCallBudget[];
}

// 三种结果：成功、超时降级、超时跳过
type BudgetedResult<T> =
  | { status: 'ok'; data: T; elapsedMs: number }
  | { status: 'timeout_degraded'; data: T; elapsedMs: number }  // 用了降级结果
  | { status: 'timeout_skipped'; elapsedMs: number };           // 直接跳过
```

---

## 3. 延迟预算管理器

```typescript
class LatencyBudgetManager {
  private startTime: number;
  private totalMs: number;

  constructor(totalMs: number) {
    this.totalMs = totalMs;
    this.startTime = Date.now();
  }

  /** 当前剩余预算 */
  get remainingMs(): number {
    const elapsed = Date.now() - this.startTime;
    return Math.max(0, this.totalMs - elapsed);
  }

  /** 已消耗百分比 */
  get consumedRatio(): number {
    return (Date.now() - this.startTime) / this.totalMs;
  }

  /** 是否已超出总预算 */
  get isExhausted(): boolean {
    return this.remainingMs <= 0;
  }

  /**
   * 为下一个工具分配超时
   * - critical：最多拿走剩余的 60%
   * - high：最多拿走剩余的 40%
   * - low：最多拿走剩余的 20%，且有硬上限
   */
  allocateFor(tool: ToolCallBudget): number {
    const remaining = this.remainingMs;
    if (remaining <= 0) return 0;

    const ratios = { critical: 0.6, high: 0.4, low: 0.2 };
    const ratio = ratios[tool.priority];

    // 不超过分配值，也不超过剩余的比例
    const byPriority = Math.floor(remaining * ratio);
    const byAllocation = tool.allocatedMs;

    return Math.min(byPriority, byAllocation, remaining);
  }

  /** 用 AbortSignal 实现超时感知 */
  abortSignalFor(timeoutMs: number): AbortSignal {
    const controller = new AbortController();
    const timer = setTimeout(() => controller.abort(), timeoutMs);
    // 总预算耗尽时也要取消
    const globalTimer = setTimeout(
      () => controller.abort(),
      this.remainingMs
    );
    controller.signal.addEventListener('abort', () => {
      clearTimeout(timer);
      clearTimeout(globalTimer);
    });
    return controller.signal;
  }
}
```

---

## 4. 超时级联执行器

```typescript
async function executWithBudget<T>(
  budget: LatencyBudgetManager,
  tool: ToolCallBudget,
  fn: (signal: AbortSignal) => Promise<T>,
  fallback?: () => T
): Promise<BudgetedResult<T>> {
  const start = Date.now();

  if (budget.isExhausted) {
    if (tool.canSkip) {
      return { status: 'timeout_skipped', elapsedMs: 0 };
    }
    // critical 工具：给一个最小时间兜底
    if (tool.priority === 'critical') {
      console.warn(`[BudgetManager] ${tool.toolName} 预算耗尽但为 critical，给 500ms 兜底`);
    }
  }

  const allocated = budget.allocateFor(tool);
  const signal = budget.abortSignalFor(allocated);

  try {
    const data = await fn(signal);
    return { status: 'ok', data, elapsedMs: Date.now() - start };
  } catch (err: any) {
    const elapsedMs = Date.now() - start;

    if (err.name === 'AbortError' || signal.aborted) {
      if (fallback) {
        const data = fallback();
        return { status: 'timeout_degraded', data, elapsedMs };
      }
      if (tool.canSkip) {
        return { status: 'timeout_skipped', elapsedMs };
      }
    }
    throw err;
  }
}
```

---

## 5. 实战：Agent 搜索 + 分析管道

```typescript
// 假设 Agent 需要：搜索 → 读取内容 → 分析 → 生成摘要
async function researchAgent(query: string): Promise<string> {
  // 总预算 12 秒（用户可接受的最大等待）
  const budget = new LatencyBudgetManager(12_000);

  // --- Step 1: 搜索（critical，必须成功）---
  const searchResult = await executWithBudget(
    budget,
    { toolName: 'web_search', allocatedMs: 3000, priority: 'critical', canSkip: false },
    (signal) => webSearch(query, { signal }),
    undefined
  );

  if (searchResult.status !== 'ok') {
    throw new Error('搜索失败，无法继续');
  }
  console.log(`[Budget] 搜索完成 ${searchResult.elapsedMs}ms，剩余 ${budget.remainingMs}ms`);

  // --- Step 2: 抓取前 3 个页面内容（low，可跳过）---
  const urls = searchResult.data.results.slice(0, 3);
  const pageContents: string[] = [];

  for (const url of urls) {
    if (budget.remainingMs < 500) break; // 预算不足直接跳过

    const fetchResult = await executWithBudget(
      budget,
      { toolName: 'web_fetch', allocatedMs: 2000, priority: 'low', canSkip: true },
      (signal) => fetchPage(url, { signal }),
      () => '' // 降级：返回空字符串
    );

    if (fetchResult.status === 'ok') {
      pageContents.push(fetchResult.data);
    } else {
      console.log(`[Budget] 跳过 ${url}，状态: ${fetchResult.status}`);
    }
  }

  console.log(`[Budget] 内容抓取完成，剩余 ${budget.remainingMs}ms`);

  // --- Step 3: LLM 分析（high）---
  const analysis = await executWithBudget(
    budget,
    { toolName: 'llm_analyze', allocatedMs: 5000, priority: 'high', canSkip: false },
    (signal) => llmAnalyze(query, pageContents, { signal }),
    () => '（内容分析超时，仅基于搜索结果回答）'
  );

  const finalContent = analysis.status === 'ok'
    ? analysis.data
    : (analysis as any).data ?? '无法完成分析';

  console.log(`[Budget] 总耗时: ${12_000 - budget.remainingMs}ms`);
  return finalContent;
}
```

---

## 6. OpenClaw 中的延迟预算

OpenClaw 工具调用没有内置级联超时，但可以用中间件注入：

```typescript
// OpenClaw 工具中间件：自动注入预算感知
function withLatencyBudget(budget: LatencyBudgetManager) {
  return async (toolName: string, params: any, next: Function) => {
    const toolMeta = TOOL_REGISTRY[toolName];
    const allocated = budget.allocateFor(toolMeta);

    if (allocated < 200) {
      if (toolMeta.canSkip) {
        return { _skipped: true, reason: 'budget_exhausted' };
      }
    }

    const timeoutPromise = new Promise((_, reject) =>
      setTimeout(() => reject(new Error('BudgetTimeout')), allocated)
    );

    try {
      return await Promise.race([next(params), timeoutPromise]);
    } catch (err: any) {
      if (err.message === 'BudgetTimeout') {
        console.warn(`[Budget] ${toolName} 超时，已用 ${12000 - budget.remainingMs}ms`);
        if (toolMeta.fallback) return toolMeta.fallback(params);
        if (toolMeta.canSkip) return null;
      }
      throw err;
    }
  };
}
```

---

## 7. pi-mono 集成方式

pi-mono 的 `Runner` 支持 `signal` 传递，可以在调用层面注入预算：

```typescript
// pi-mono: 给整个 agent turn 设置延迟预算
const budget = new LatencyBudgetManager(15_000);

const runner = new Runner({
  tools: tools.map(tool => ({
    ...tool,
    execute: async (params) => {
      const toolBudget = TOOL_BUDGETS[tool.name] ?? {
        allocatedMs: 5000,
        priority: 'high',
        canSkip: false
      };
      const timeout = budget.allocateFor(toolBudget);
      return Promise.race([
        tool.execute(params),
        sleep(timeout).then(() => ({ _timeout: true }))
      ]);
    }
  }))
});

await runner.run(userMessage);
```

---

## 8. 关键指标监控

```typescript
interface BudgetMetrics {
  totalBudgetMs: number;
  actualElapsedMs: number;
  sloBreached: boolean;         // 是否超出总预算
  toolsSkipped: number;         // 被跳过的工具数
  toolsDegraded: number;        // 走降级路径的工具数
  budgetUtilization: number;    // 预算使用率 0-1
  criticalToolsOnTime: boolean; // 关键工具是否都按时完成
}

// Prometheus 埋点
counter.inc('agent_budget_breach_total', { route: '/research' });
histogram.observe('agent_budget_utilization', metrics.budgetUtilization);
gauge.set('agent_tools_skipped', metrics.toolsSkipped);
```

---

## 9. 设计原则总结

| 原则 | 说明 |
|------|------|
| **总预算优先** | 先定 E2E SLO，再推算各工具分配 |
| **动态分配** | 上游快了，下游获得更多；上游慢了，下游压缩 |
| **优先级保障** | critical 工具不因预算不足而被跳过 |
| **优雅降级** | 超时 ≠ 失败，有 fallback 就用 fallback |
| **透明日志** | 每次工具调用记录分配/实际/剩余预算，可观测 |
| **可跳过工具** | 非关键工具加 `canSkip: true`，预算告急直接跳 |

---

## 10. 与相关模式的对比

| 模式 | 关注点 | 差异 |
|------|--------|------|
| 自适应超时（第97课） | 单工具超时自适应 | 无全局预算概念 |
| 对冲请求（第97课） | 消灭长尾延迟 | 关注单次调用，不关注链路总时间 |
| 负载卸载（第147课） | 系统过载时丢弃请求 | 入口层，不是执行链内部 |
| **延迟预算（本课）** | 整条工具链的时间分配 | 端到端 SLO 分解 + 级联感知 |

---

## TL;DR

- **总预算 = E2E SLO**，在第一个工具调用前就确定
- **级联分配**：剩余时间 × 优先级系数 = 本工具超时
- **三种结果**：成功 / 超时降级（用 fallback） / 超时跳过
- **critical 工具不可跳过**，low 工具预算不足直接 skip
- 配合 OpenTelemetry 记录每个工具的分配 vs 实际耗时，持续优化预算模型
