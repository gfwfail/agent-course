# 177 - Agent 工具执行并行安全与原子操作

> **核心问题：** 并发工具调用提升速度，但顺序错了就是灾难。怎么知道哪些能并发，哪些必须串行？出了一半怎么回滚？

---

## 为什么这是个难题

Claude 在一次 LLM 响应里可以返回多个 tool_use 块。OpenClaw / pi-mono 默认并发执行这些工具——这很好，但有个陷阱：

```
LLM 决定同时调用：
  1. read_user(id=42)        ✅ 可并发（只读）
  2. deduct_balance(42, 100) ⚠️ 写操作
  3. send_email(42, receipt) ❌ 依赖 #2 的结果，不能先跑！
```

如果 `send_email` 在 `deduct_balance` 失败后已经发出去了，用户收到收据但钱没扣——或者扣了钱但邮件发失败了。

**三个核心问题：**
1. **并行安全分析**：哪些工具可以并发？
2. **原子操作**：多步工具调用要么全成功，要么全回滚
3. **部分失败恢复**：并发中一个失败了，其他怎么办

---

## 第一部分：并行安全分析

### 工具注解 —— 声明副作用

```typescript
// pi-mono / OpenClaw 工具定义扩展
interface ToolMeta {
  name: string;
  description: string;
  // 新增：副作用声明
  sideEffects: {
    reads?: string[];    // 读取的资源（乐观并发安全）
    writes?: string[];   // 写入的资源（需要串行或锁）
    external?: boolean;  // 调用外部系统（不可回滚）
  };
  idempotent?: boolean;  // 幂等的可以安全重试
}

// 示例
const tools: ToolMeta[] = [
  {
    name: "read_user",
    sideEffects: { reads: ["users"] },
    idempotent: true,
  },
  {
    name: "deduct_balance", 
    sideEffects: { writes: ["users.balance", "transactions"] },
    idempotent: false,
  },
  {
    name: "send_email",
    sideEffects: { external: true },
    idempotent: false,  // 发两次邮件就是 bug
  },
];
```

### 依赖分析器

```typescript
class ParallelismAnalyzer {
  // 分析一批工具调用，返回可并发的"波次"
  planExecution(calls: ToolCall[]): ExecutionWave[] {
    const waves: ExecutionWave[] = [];
    const completed = new Set<string>();

    while (completed.size < calls.length) {
      const wave: ToolCall[] = [];
      
      for (const call of calls) {
        if (completed.has(call.id)) continue;
        
        // 检查依赖是否满足
        if (this.depsReady(call, completed) && 
            this.safeToParallel(call, wave)) {
          wave.push(call);
        }
      }
      
      if (wave.length === 0) {
        throw new Error("循环依赖或死锁！");
      }
      
      waves.push({ calls: wave });
      wave.forEach(c => completed.add(c.id));
    }
    
    return waves;
  }

  private safeToParallel(call: ToolCall, wave: ToolCall[]): boolean {
    const meta = getToolMeta(call.name);
    
    // 外部副作用工具：永远单独一波
    if (meta.sideEffects.external) {
      return wave.length === 0;
    }
    
    // 写同一资源的工具：不能并发
    for (const existing of wave) {
      const existingMeta = getToolMeta(existing.name);
      const conflict = meta.sideEffects.writes?.some(
        w => existingMeta.sideEffects.writes?.includes(w) ||
             existingMeta.sideEffects.reads?.includes(w)
      );
      if (conflict) return false;
    }
    
    return true;
  }
}
```

### 实际执行

```typescript
async function executeParallel(
  calls: ToolCall[],
  context: AgentContext
): Promise<ToolResult[]> {
  const analyzer = new ParallelismAnalyzer();
  const waves = analyzer.planExecution(calls);
  
  const results: Map<string, ToolResult> = new Map();
  
  for (const wave of waves) {
    console.log(`⚡ 并发执行 ${wave.calls.length} 个工具:`, 
      wave.calls.map(c => c.name));
    
    // 同一波次真正并发
    const waveResults = await Promise.allSettled(
      wave.calls.map(call => executeTool(call, context, results))
    );
    
    // 处理失败
    for (let i = 0; i < waveResults.length; i++) {
      const result = waveResults[i];
      if (result.status === 'rejected') {
        // 中止后续波次
        throw new WaveExecutionError(wave.calls[i], result.reason, results);
      }
      results.set(wave.calls[i].id, result.value);
    }
  }
  
  return Array.from(results.values());
}
```

---

## 第二部分：原子操作 —— Saga 模式进阶

> 已讲过基础 Saga（lesson 71），这里聚焦**工具级原子性**

### 问题场景

```
用户：把 $100 从 Alice 转给 Bob
Agent 工具调用：
  Step 1: deduct(alice, 100)   ✅ 成功
  Step 2: credit(bob, 100)     ❌ 失败！
  
结果：Alice 少了 $100，Bob 什么都没收到
```

### 工具原子组

```typescript
// 声明一组工具为"原子事务"
interface AtomicToolGroup {
  tools: ToolCall[];
  compensations: Map<string, ToolCall>; // 每个工具的补偿操作
}

class AtomicExecutor {
  private executed: Array<{ call: ToolCall; compensation: ToolCall }> = [];

  async execute(group: AtomicToolGroup): Promise<ToolResult[]> {
    const results: ToolResult[] = [];
    
    for (const call of group.tools) {
      try {
        const result = await executeTool(call);
        results.push(result);
        
        // 记录补偿操作（以防后面失败需要回滚）
        const compensation = group.compensations.get(call.name);
        if (compensation) {
          this.executed.push({ call, compensation });
        }
      } catch (err) {
        // 立即回滚已执行的操作
        await this.rollback();
        throw new AtomicGroupError(`原子组执行失败于 ${call.name}`, err);
      }
    }
    
    return results;
  }

  private async rollback(): Promise<void> {
    console.log(`🔄 回滚 ${this.executed.length} 个操作...`);
    
    // 反向执行补偿操作
    for (const { call, compensation } of [...this.executed].reverse()) {
      try {
        await executeTool(compensation);
        console.log(`  ✅ 已回滚: ${call.name}`);
      } catch (err) {
        // 补偿操作失败 → 需要人工干预
        console.error(`  ❌ 回滚失败: ${call.name}`, err);
        await alertHuman(`回滚失败，需要人工处理: ${call.name}`);
      }
    }
  }
}
```

### 在 Agent Loop 中使用

```typescript
// 告诉 LLM 如何声明原子操作
const systemPrompt = `
当你需要执行多个相互依赖的操作时，使用 atomic_group 工具将它们包装：

<example>
{
  "name": "atomic_group",
  "input": {
    "operations": ["deduct_balance", "credit_balance"],
    "description": "资金转账，要么全成功，要么全回滚"
  }
}
</example>
`;

// atomic_group 工具实现
tools.register({
  name: "atomic_group",
  description: "将多个工具调用包装为原子事务",
  inputSchema: {
    operations: { type: "array", items: { type: "string" } },
    description: { type: "string" }
  },
  handler: async (input, context) => {
    // 解析 LLM 接下来要调用的工具序列
    // 注入 AtomicExecutor 上下文
    context.setAtomicMode(true);
    return { status: "atomic_mode_enabled", ops: input.operations };
  }
});
```

---

## 第三部分：部分失败处理策略

并发中一个失败了，其他怎么办？三种策略：

```typescript
type FailureStrategy = 
  | "fail-fast"      // 任何失败立即取消所有（默认）
  | "best-effort"    // 继续执行，收集所有结果和错误
  | "critical-path"; // 只有关键路径失败才取消

async function executeWithStrategy(
  calls: ToolCall[],
  strategy: FailureStrategy
): Promise<BatchResult> {
  
  if (strategy === "fail-fast") {
    // 用 Promise.all — 任意失败抛出
    const results = await Promise.all(calls.map(executeTool));
    return { success: true, results };
  }
  
  if (strategy === "best-effort") {
    // 用 Promise.allSettled — 收集所有结果
    const settled = await Promise.allSettled(calls.map(executeTool));
    const results = settled.map((r, i) => ({
      tool: calls[i].name,
      success: r.status === 'fulfilled',
      value: r.status === 'fulfilled' ? r.value : null,
      error: r.status === 'rejected' ? r.reason?.message : null,
    }));
    
    const failed = results.filter(r => !r.success);
    return {
      success: failed.length === 0,
      partial: failed.length > 0,
      results,
      summary: `${results.length - failed.length}/${results.length} 成功`,
    };
  }
  
  if (strategy === "critical-path") {
    // 先执行关键路径工具，成功后再并发执行其他
    const [critical, optional] = partition(calls, c => c.meta?.critical);
    await Promise.all(critical.map(executeTool)); // 失败则抛出
    const optionalResults = await Promise.allSettled(optional.map(executeTool));
    return buildResult(critical, optionalResults);
  }
}
```

---

## OpenClaw 实战：多工具并发安全

OpenClaw 的工具分发层（tool dispatch）天然支持并发。看看如何加安全分析：

```typescript
// ~/.openclaw/workspace/skills/my-skill/index.ts

import { ToolRegistry } from "@openclaw/sdk";

const registry = new ToolRegistry();

// 注册时声明副作用
registry.register({
  name: "fetch_price",
  sideEffects: { reads: ["prices"], external: true },
  idempotent: true,
  handler: async (input) => {
    return await fetchMarketPrice(input.symbol);
  }
});

registry.register({
  name: "place_order", 
  sideEffects: { writes: ["orders", "balance"], external: true },
  idempotent: false,
  handler: async (input) => {
    // 外部副作用：永远串行！
    return await exchange.placeOrder(input);
  }
});

// 分发层自动分析并发安全性
const dispatcher = new SafeDispatcher(registry);

// LLM 同时请求这两个工具
const calls = [
  { name: "fetch_price", input: { symbol: "BTC" } },
  { name: "fetch_price", input: { symbol: "ETH" } },  // 可并发！
  { name: "place_order", input: { symbol: "BTC", qty: 1 } }, // 独立串行
];

const waves = dispatcher.plan(calls);
// Wave 1: [fetch_price BTC, fetch_price ETH]  ← 并发
// Wave 2: [place_order BTC]                    ← 串行（external）
```

---

## pi-mono 中的实现参考

```typescript
// pi-mono/src/agent/parallel-executor.ts

export class ParallelToolExecutor {
  async dispatch(toolCalls: ToolUse[]): Promise<ToolResult[]> {
    // 1. 分析依赖图
    const graph = this.buildDependencyGraph(toolCalls);
    
    // 2. 拓扑排序（已有 DAG 引擎，lesson-83）
    const waves = graph.topologicalWaves();
    
    // 3. 按波次执行
    const results = new ResultStore();
    
    for (const wave of waves) {
      const waveResults = await Promise.allSettled(
        wave.map(call => this.executeOne(call, results))
      );
      
      results.merge(wave, waveResults);
      
      if (results.hasCriticalFailure()) {
        await results.compensate(); // Saga 回滚
        break;
      }
    }
    
    return results.toArray();
  }
}
```

---

## 关键原则

| 情景 | 策略 |
|------|------|
| 纯读取操作 | ✅ 随意并发 |
| 写不同资源 | ✅ 可以并发 |
| 写同一资源 | ⚠️ 必须串行或加锁 |
| 外部副作用（邮件/支付）| ⚠️ 串行 + 幂等检查 |
| 多步依赖操作 | ⚠️ Saga 原子组 + 补偿 |
| 不可逆操作 | 🔴 人工审批再执行 |

---

## 三句话总结

1. **声明副作用**：工具注册时声明 reads/writes/external，让调度器自动分析并发安全性
2. **原子组 + 补偿**：多步操作包装为 Saga，任何步骤失败自动逆序回滚
3. **按策略处理失败**：fail-fast / best-effort / critical-path 根据业务选择，不是所有失败都要中止

> 并发是性能，原子是正确性。两者都不能妥协。 🔒⚡
