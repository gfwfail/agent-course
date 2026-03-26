# 147 - Agent 超时预算继承与级联取消
## Timeout Budget Inheritance & Cascading Cancellation

---

## 为什么需要这个？

你的 Agent 调用了 Sub-agent，Sub-agent 又调了工具，工具又发了 HTTP 请求。  
父级超时了，但子级还在跑——这就是"幽灵任务"：

```
Parent (10s budget)
  └─ Sub-agent A (已跑 8s)
       └─ Tool: fetch_data (还在等响应...)
            └─ HTTP Request → 第三方 API (完全不知道父级已超时)
```

结果：父级报了 TimeoutError，但底层资源（连接、进程、锁）还在占用，甚至最终成功写入了数据库，造成**不一致状态**。

**解决方案：Deadline Propagation + Cascading Cancellation**

---

## 核心概念

### 1. Deadline vs Timeout

| 概念 | 含义 |
|------|------|
| Timeout | 相对时间，"再等 N 秒" |
| Deadline | 绝对时间，"必须在 T 时刻前完成" |

**Deadline 更适合传播**：父级把绝对截止时间传给子级，子级自己算剩余时间。不管中间有多少层，截止时间不变。

### 2. 预算继承规则

```
父级剩余时间 = Deadline - now()
子级预算 = 父级剩余时间 × 分配比例 - 安全余量
```

比如父级还剩 8s，分配给工具调用 80%，留 20% 给后处理：
- 工具预算 = 8 × 0.8 = 6.4s
- 工具自己还可以再留安全余量，比如减 200ms

### 3. 级联取消

当父级超时 → 立刻向所有子级发出取消信号 → 子级清理资源后退出  
这就是 **AbortController 链**（JS）/ **Context cancellation**（Go）/ **asyncio CancelledError**（Python）

---

## TypeScript 实现（pi-mono 风格）

```typescript
// deadline-context.ts
export interface DeadlineContext {
  deadline: number;           // Unix timestamp (ms)
  signal: AbortSignal;        // 取消信号
  remaining(): number;        // 剩余毫秒数
  child(fraction?: number, safetyMs?: number): DeadlineContext;
}

export function createDeadlineContext(
  timeoutMs: number,
  parentSignal?: AbortSignal
): DeadlineContext {
  const deadline = Date.now() + timeoutMs;
  const controller = new AbortController();

  // 父级取消 → 子级也取消（链式）
  if (parentSignal) {
    if (parentSignal.aborted) {
      controller.abort(parentSignal.reason);
    } else {
      parentSignal.addEventListener('abort', () => {
        controller.abort(parentSignal.reason);
      }, { once: true });
    }
  }

  // 自己的定时器
  const timer = setTimeout(() => {
    controller.abort(new Error(`Deadline exceeded after ${timeoutMs}ms`));
  }, timeoutMs);

  // 防止 Node.js 进程因 timer 保持运行
  if (typeof timer === 'object' && timer.unref) timer.unref();

  const ctx: DeadlineContext = {
    deadline,
    signal: controller.signal,

    remaining() {
      return Math.max(0, deadline - Date.now());
    },

    // 派生子 context，自动分配预算
    child(fraction = 0.8, safetyMs = 100): DeadlineContext {
      const childBudget = Math.max(0, ctx.remaining() * fraction - safetyMs);
      return createDeadlineContext(childBudget, controller.signal);
    }
  };

  return ctx;
}
```

---

## Agent Loop 集成

```typescript
// agent-runner.ts
import { createDeadlineContext, DeadlineContext } from './deadline-context';

interface ToolCall {
  name: string;
  params: Record<string, unknown>;
}

async function runAgentWithDeadline(
  task: string,
  totalBudgetMs: number
): Promise<string> {
  // 顶层：创建根 context
  const rootCtx = createDeadlineContext(totalBudgetMs);

  try {
    return await agentLoop(task, rootCtx);
  } catch (err: any) {
    if (err?.name === 'AbortError') {
      return `[TIMEOUT] Agent 超时，任务未完成: ${task}`;
    }
    throw err;
  }
}

async function agentLoop(task: string, ctx: DeadlineContext): Promise<string> {
  // 每次工具调用前检查
  while (ctx.remaining() > 0) {
    const toolCall = await llmDecide(task, ctx);
    if (!toolCall) break;

    // 派生子 context（80% 预算，留 200ms 安全余量）
    const toolCtx = ctx.child(0.8, 200);

    const result = await dispatchTool(toolCall, toolCtx);
    task = `${task}\n[tool:${toolCall.name}] → ${result}`;
  }

  if (ctx.remaining() <= 0) {
    throw new DOMException('Deadline exceeded', 'AbortError');
  }

  return await llmSummarize(task, ctx);
}

async function dispatchTool(
  call: ToolCall,
  ctx: DeadlineContext
): Promise<string> {
  // 工具执行时传入 signal，支持中断
  const timeout = ctx.remaining();

  return await Promise.race([
    executeTool(call, ctx.signal),
    new Promise<never>((_, reject) => {
      ctx.signal.addEventListener('abort', () => {
        reject(ctx.signal.reason);
      }, { once: true });
    })
  ]);
}

async function executeTool(
  call: ToolCall,
  signal: AbortSignal
): Promise<string> {
  // 工具实现需要支持 signal
  if (call.name === 'fetch_url') {
    const res = await fetch(call.params.url as string, { signal });
    return await res.text();
  }
  // ...其他工具
  return '';
}
```

---

## Sub-agent 场景：跨进程传播

子进程无法直接共享 `AbortController`，需要通过**消息协议**传播：

```typescript
// parent-agent.ts
async function spawnSubAgent(
  task: string,
  ctx: DeadlineContext
): Promise<string> {
  const subProcess = spawn('node', ['sub-agent.js'], {
    env: {
      ...process.env,
      AGENT_DEADLINE: ctx.deadline.toString(),  // 传递绝对截止时间
      AGENT_TASK: task,
    }
  });

  // 父级取消时，杀掉子进程
  ctx.signal.addEventListener('abort', () => {
    subProcess.kill('SIGTERM');
  }, { once: true });

  return new Promise((resolve, reject) => {
    let output = '';
    subProcess.stdout.on('data', (d) => output += d);
    subProcess.on('close', (code) => {
      if (code === 0) resolve(output);
      else reject(new Error(`Sub-agent exited with code ${code}`));
    });
  });
}

// sub-agent.ts（子进程）
const deadline = parseInt(process.env.AGENT_DEADLINE || '0');
const remaining = Math.max(0, deadline - Date.now());

// 用从环境变量读取的 deadline 创建自己的 context
const ctx = createDeadlineContext(remaining);
// 正常执行...
```

---

## Python 实现（asyncio 风格）

```python
# deadline_context.py
import asyncio
import time
from dataclasses import dataclass
from typing import Optional

@dataclass
class DeadlineContext:
    deadline: float           # Unix timestamp (秒)
    _task: Optional[asyncio.Task] = None

    @classmethod
    def create(cls, timeout_s: float) -> 'DeadlineContext':
        return cls(deadline=time.time() + timeout_s)

    def remaining(self) -> float:
        return max(0.0, self.deadline - time.time())

    def child(self, fraction: float = 0.8, safety_s: float = 0.1) -> 'DeadlineContext':
        child_budget = max(0.0, self.remaining() * fraction - safety_s)
        return DeadlineContext.create(child_budget)

    def is_expired(self) -> bool:
        return self.remaining() <= 0


async def run_with_deadline(coro, ctx: DeadlineContext):
    """在 deadline 内执行协程，超时自动取消"""
    remaining = ctx.remaining()
    if remaining <= 0:
        raise asyncio.TimeoutError("Deadline already exceeded")

    try:
        return await asyncio.wait_for(coro, timeout=remaining)
    except asyncio.TimeoutError:
        raise asyncio.TimeoutError(f"Deadline exceeded (had {remaining:.2f}s)")


# 使用示例
async def agent_loop(task: str, ctx: DeadlineContext) -> str:
    results = []

    for step in range(10):
        if ctx.is_expired():
            break

        tool_ctx = ctx.child(fraction=0.7)

        try:
            result = await run_with_deadline(
                execute_tool("search", {"q": task}),
                tool_ctx
            )
            results.append(result)
        except asyncio.TimeoutError:
            results.append("[tool timeout, skipping]")
            break

    return "\n".join(results)
```

---

## OpenClaw Cron 任务集成

OpenClaw 的 cron 任务本身就有 `timeoutSeconds`，可以利用它计算 deadline：

```typescript
// 在 cron agentTurn payload 中，task 会被限制在 timeoutSeconds 内
// 在 Agent 内部，从已知的任务开始时间推算

class OpenClawAgentRunner {
  private startTime = Date.now();
  private maxBudgetMs: number;

  constructor(timeoutSeconds: number) {
    // 留 10% 给收尾工作
    this.maxBudgetMs = timeoutSeconds * 1000 * 0.9;
  }

  createRootContext(): DeadlineContext {
    const elapsed = Date.now() - this.startTime;
    const remaining = Math.max(0, this.maxBudgetMs - elapsed);
    return createDeadlineContext(remaining);
  }
}
```

---

## 常见陷阱

### ❌ 忘记传播 signal

```typescript
// 错误：工具完全不知道取消信号
async function badTool(params: any) {
  const res = await fetch(params.url);  // 父级超时了，这里还在跑
  return res.text();
}

// 正确：每个 I/O 操作都传入 signal
async function goodTool(params: any, signal: AbortSignal) {
  const res = await fetch(params.url, { signal });
  return res.text();
}
```

### ❌ 子级预算超过父级

```typescript
// 错误：子级拿了比父级还多的时间
const childCtx = createDeadlineContext(10000);  // 写死 10s

// 正确：从父级派生
const childCtx = parentCtx.child(0.8);  // 父级剩余时间的 80%
```

### ❌ 取消后还继续写状态

```typescript
// 错误：signal 已中止，但还在写数据库
async function badHandler(signal: AbortSignal) {
  const data = await fetchData(signal);
  // 这里 signal 可能已经 aborted！
  await db.write(data);  // 写入了不完整数据

// 正确：每步前检查
async function goodHandler(signal: AbortSignal) {
  const data = await fetchData(signal);
  if (signal.aborted) throw signal.reason;
  await db.write(data);
}
```

---

## 完整调用链示例

```
runAgentWithDeadline(task, 30000ms)
  ├─ rootCtx: deadline = now + 30s
  │
  ├─ [Step 1] dispatchTool("search", rootCtx.child(0.8))
  │    └─ toolCtx: deadline = now + 23.8s
  │         └─ fetch(url, { signal: toolCtx.signal })  ← 2s 完成
  │
  ├─ [Step 2] dispatchTool("analyze", rootCtx.child(0.8))
  │    └─ toolCtx: deadline = now + 21.4s (rootCtx 剩 26.7s × 0.8 = 21.4s)
  │         └─ llm.call({ ... })  ← 5s 完成
  │
  └─ [Step 3] spawnSubAgent("summarize", rootCtx.child(0.6))
       └─ subCtx: deadline = now + 12.7s
            ├─ sub-agent 内部工具调用也受 deadline 约束
            └─ 子进程在 10s 内完成 ✅

总耗时: 17s < 30s 预算 ✅
```

---

## 关键要点

1. **用 Deadline（绝对时间）替代 Timeout（相对时间）** — 传播不失真
2. **AbortController 链** — 父取消，子自动取消
3. **child() 方法** — 子级预算自动从父级派生，不能超过父级
4. **每个 I/O 操作传 signal** — fetch/db/llm 全部支持取消
5. **取消后检查状态** — 避免写入不完整数据
6. **跨进程传 deadline 时间戳** — 子进程用环境变量接收

---

## 参考

- [AbortController MDN](https://developer.mozilla.org/en-US/docs/Web/API/AbortController)
- [Go context package](https://pkg.go.dev/context) — Deadline 传播的标准实现
- [gRPC Deadline Propagation](https://grpc.io/docs/guides/deadlines/) — 分布式系统经典设计
