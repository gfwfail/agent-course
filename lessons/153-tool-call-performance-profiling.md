# 153 - Agent 工具调用性能剖析（Tool Call Performance Profiling）

> "你的 Agent 慢，但你不知道哪里慢。" —— 每个写完 Agent 又发现延迟高的开发者

## 为什么需要性能剖析？

Agent 的一次执行可能涉及：
- 5 次工具调用（其中 1 次慢了 3 秒）
- 3 次 LLM 推理（其中 1 次耗时占总时间 70%）
- 2 次重试（完全被忽略）

**没有剖析，你只能猜。有了剖析，你能精准优化。**

性能剖析的三大核心问题：
1. **哪个工具最慢？**（P99 延迟）
2. **哪个工具调用最频繁？**（调用次数 × 成本）
3. **工具调用的串并行结构是否合理？**（瀑布图分析）

---

## 核心数据结构

```typescript
// 单次工具调用的性能记录
interface ToolCallProfile {
  toolName: string;
  callId: string;           // 唯一标识，方便关联
  traceId: string;          // 所属 Agent 执行链路
  startTime: number;        // ms timestamp
  endTime: number;
  durationMs: number;
  status: 'success' | 'error' | 'timeout';
  inputBytes: number;       // 参数大小
  outputBytes: number;      // 结果大小
  retryCount: number;
  metadata?: Record<string, unknown>; // 扩展字段（如 SQL、API endpoint）
}

// 一次 Agent 完整运行的剖析报告
interface AgentRunProfile {
  sessionId: string;
  totalDurationMs: number;
  llmDurationMs: number;      // 纯 LLM 推理时间
  toolDurationMs: number;     // 纯工具执行时间（排除并发重叠）
  toolCalls: ToolCallProfile[];
  criticalPath: string[];     // 决定总时长的工具链（关键路径）
  hotspots: ToolHotspot[];
}

interface ToolHotspot {
  toolName: string;
  callCount: number;
  totalDurationMs: number;
  avgDurationMs: number;
  p50Ms: number;
  p95Ms: number;
  p99Ms: number;
  errorRate: number;
  estimatedCost: number;      // USD
}
```

---

## 核心实现：Profiling 中间件

```typescript
class ToolProfiler {
  private store: Map<string, ToolCallProfile[]> = new Map(); // traceId -> profiles

  // 包装任意工具，自动注入剖析逻辑
  wrap<T extends ToolFn>(toolName: string, fn: T): T {
    return (async (params: unknown, ctx: ToolContext) => {
      const callId = crypto.randomUUID();
      const traceId = ctx.traceId ?? 'default';
      const startTime = Date.now();
      const inputBytes = JSON.stringify(params).length;

      let status: ToolCallProfile['status'] = 'success';
      let result: unknown;

      try {
        result = await fn(params, ctx);
      } catch (err) {
        status = err instanceof TimeoutError ? 'timeout' : 'error';
        throw err;
      } finally {
        const endTime = Date.now();
        const profile: ToolCallProfile = {
          toolName,
          callId,
          traceId,
          startTime,
          endTime,
          durationMs: endTime - startTime,
          status,
          inputBytes,
          outputBytes: result ? JSON.stringify(result).length : 0,
          retryCount: ctx.retryCount ?? 0,
        };
        this.record(traceId, profile);
      }

      return result;
    }) as T;
  }

  private record(traceId: string, profile: ToolCallProfile) {
    if (!this.store.has(traceId)) this.store.set(traceId, []);
    this.store.get(traceId)!.push(profile);
  }

  // 生成单次运行的热点报告
  getHotspots(traceId: string): ToolHotspot[] {
    const calls = this.store.get(traceId) ?? [];
    const grouped = new Map<string, ToolCallProfile[]>();

    for (const c of calls) {
      if (!grouped.has(c.toolName)) grouped.set(c.toolName, []);
      grouped.get(c.toolName)!.push(c);
    }

    return [...grouped.entries()].map(([toolName, profiles]) => {
      const durations = profiles.map(p => p.durationMs).sort((a, b) => a - b);
      const n = durations.length;
      return {
        toolName,
        callCount: n,
        totalDurationMs: durations.reduce((s, d) => s + d, 0),
        avgDurationMs: Math.round(durations.reduce((s, d) => s + d, 0) / n),
        p50Ms: durations[Math.floor(n * 0.5)],
        p95Ms: durations[Math.floor(n * 0.95)],
        p99Ms: durations[Math.floor(n * 0.99)] ?? durations[n - 1],
        errorRate: profiles.filter(p => p.status !== 'success').length / n,
        estimatedCost: this.estimateCost(toolName, profiles),
      };
    }).sort((a, b) => b.totalDurationMs - a.totalDurationMs); // 按总耗时排序
  }

  private estimateCost(toolName: string, profiles: ToolCallProfile[]): number {
    // 示例：DB 查询按调用次数，外部 API 按流量
    const costPerCall: Record<string, number> = {
      'web_search': 0.001,
      'db_query': 0.0001,
      'llm_call': 0.002,
    };
    return (costPerCall[toolName] ?? 0) * profiles.length;
  }
}
```

---

## 关键路径分析（Critical Path）

串行调用决定最终延迟，并行调用不叠加。**找到关键路径才能有效优化。**

```typescript
// 基于时间线的关键路径算法
function findCriticalPath(calls: ToolCallProfile[]): string[] {
  if (calls.length === 0) return [];

  // 按结束时间排序，逆向追踪
  const sorted = [...calls].sort((a, b) => b.endTime - a.endTime);
  const criticalPath: string[] = [];

  let cursor = sorted[0].endTime; // 从最晚结束的工具往前追

  for (const call of sorted) {
    // 如果这个工具的结束时间紧接 cursor，它在关键路径上
    if (Math.abs(call.endTime - cursor) < 10) { // 10ms 容忍
      criticalPath.unshift(call.toolName);
      cursor = call.startTime;
    }
  }

  return criticalPath;
}

// 示例输出：
// criticalPath = ['db_query', 'llm_call', 'send_email']
// 说明：这三个工具串行执行，优化任何一个都能降低总延迟
```

---

## 瀑布图：可视化工具调用时序

```typescript
function renderWaterfall(calls: ToolCallProfile[]): string {
  if (calls.length === 0) return '(no tool calls)';

  const origin = Math.min(...calls.map(c => c.startTime));
  const end    = Math.max(...calls.map(c => c.endTime));
  const totalMs = end - origin;
  const WIDTH = 60; // 字符宽度

  const lines = calls
    .sort((a, b) => a.startTime - b.startTime)
    .map(c => {
      const startPct = (c.startTime - origin) / totalMs;
      const widthPct  = c.durationMs / totalMs;
      const pad    = Math.round(startPct * WIDTH);
      const bar    = Math.max(1, Math.round(widthPct * WIDTH));
      const icon   = c.status === 'success' ? '█' : '░';
      return `${c.toolName.padEnd(20)} |${' '.repeat(pad)}${icon.repeat(bar)}| ${c.durationMs}ms`;
    });

  const header = `${'Tool'.padEnd(20)} |${'-'.repeat(WIDTH)}| Duration`;
  const ruler  = `${''.padEnd(20)} 0ms${' '.repeat(WIDTH - 6)}${totalMs}ms`;
  return [header, ...lines, ruler].join('\n');
}

/* 示例输出：
Tool                 |------------------------------------------------------------| Duration
web_search           |████████████████                                            | 820ms
db_query             |                ████████                                    | 410ms  ← 并行
llm_call             |                        ████████████████████████            | 1240ms
send_notification    |                                                ██          | 95ms

0ms                                                                          2565ms
*/
```

---

## 聚合分析：跨会话统计

单次剖析说明不了问题，**跨会话聚合才能发现系统性瓶颈**。

```typescript
class AggregateProfileStore {
  // Redis ZADD：按工具名 + 时间桶存储
  async record(profile: ToolCallProfile) {
    const bucket = Math.floor(Date.now() / 60_000); // 1分钟桶
    const key = `prof:${profile.toolName}:${bucket}`;
    await redis.rpush(key, JSON.stringify({
      d: profile.durationMs,
      s: profile.status,
    }));
    await redis.expire(key, 86400); // 保留 24h
  }

  // 查询过去 N 分钟某工具的 P95
  async getP95(toolName: string, minutes = 60): Promise<number> {
    const now = Math.floor(Date.now() / 60_000);
    const keys = Array.from({ length: minutes }, (_, i) =>
      `prof:${toolName}:${now - i}`
    );

    const all = (await Promise.all(keys.map(k => redis.lrange(k, 0, -1))))
      .flat()
      .map(s => JSON.parse(s).d as number)
      .sort((a, b) => a - b);

    if (all.length === 0) return 0;
    return all[Math.floor(all.length * 0.95)];
  }

  // 找出过去 1h 中最慢的 5 个工具
  async topSlowTools(limit = 5): Promise<Array<{ tool: string; p95Ms: number }>> {
    const tools = await redis.smembers('prof:tools'); // 维护工具名称集合
    const results = await Promise.all(
      tools.map(async tool => ({ tool, p95Ms: await this.getP95(tool) }))
    );
    return results.sort((a, b) => b.p95Ms - a.p95Ms).slice(0, limit);
  }
}
```

---

## OpenClaw 集成：Cron 定期报告

```typescript
// 每小时自动生成性能报告推送到 Telegram
// cron job payload:
// { kind: "agentTurn", message: "生成过去1小时的工具性能报告，找出 P95 > 500ms 的工具并给出优化建议" }

// 在 agent 代码中，收到这条指令后：
async function generatePerfReport() {
  const store = new AggregateProfileStore();
  const slowTools = await store.topSlowTools(10);

  const report = slowTools
    .filter(t => t.p95Ms > 500)
    .map(t => `⚠️ ${t.tool}: P95=${t.p95Ms}ms`)
    .join('\n');

  return report || '✅ 所有工具 P95 < 500ms，性能健康';
}
```

---

## 实战：在 pi-mono 中接入 Profiler

```typescript
// pi-mono/src/tools/registry.ts
import { ToolProfiler } from './profiler';

const profiler = new ToolProfiler();

export function registerTool(name: string, fn: ToolFn) {
  // 注册时自动包装
  const wrapped = profiler.wrap(name, fn);
  toolRegistry.set(name, wrapped);
}

// 在 Agent Loop 结束时打印报告
export async function runAgentLoop(task: string, ctx: AgentContext) {
  ctx.traceId = crypto.randomUUID();

  const result = await executeLoop(task, ctx);

  // 打印瀑布图和热点
  const calls = profiler.getCalls(ctx.traceId);
  console.log('\n=== Tool Call Waterfall ===');
  console.log(renderWaterfall(calls));
  console.log('\n=== Hotspots ===');
  for (const h of profiler.getHotspots(ctx.traceId)) {
    console.log(`${h.toolName}: avg=${h.avgDurationMs}ms p99=${h.p99Ms}ms count=${h.callCount}`);
  }

  return result;
}
```

---

## 优化决策矩阵

根据剖析数据，选择对应优化策略：

| 症状 | 诊断 | 优化手段 |
|------|------|----------|
| 单工具 P99 高 | 工具本身慢 | 加缓存 / 超时截断 / 换实现 |
| 多工具串行执行 | 未并发 | 重构为 `Promise.all` |
| 同一工具重复调用 | 缺去重 | 接入 Deduplication（第96课） |
| LLM 时间占比 > 80% | 推理太多轮 | 减少工具调用轮次 / Prompt 压缩 |
| 工具频繁 retry | 不稳定 | 接入 Circuit Breaker（第47课）|
| 总时间长但单工具不慢 | 关键路径串行 | DAG 并行重构（第83课） |

---

## 本课小结

```
性能剖析三板斧：
  1. 中间件 wrap → 自动采集每次工具调用的时序数据
  2. 瀑布图 → 可视化串并行结构，找到关键路径
  3. 聚合 P95/P99 → 跨会话统计，发现系统性瓶颈

黄金法则：
  - 先剖析，再优化，别猜
  - 优化关键路径上的工具，效果最大
  - P99 比 avg 更重要（用户感受的是最慢的那次）
  - 并发 > 缓存 > 算法，按顺序优化
```

**下节预告**：Agent 渐进式任务分解与动态子目标生成（Progressive Task Decomposition & Dynamic Subgoal Generation）

---

*本课对应 learn-claude-code 中的 `profiling/` 目录、pi-mono 的 `ToolRegistry.wrap()` 设计。*
