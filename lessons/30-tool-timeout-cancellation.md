# Tool Timeout & Cancellation：工具超时与取消机制

## 为什么需要超时与取消？

在 Agent 系统中，工具调用可能会卡住、变慢或者永远不返回：

```
用户: "帮我分析这个网站"
Agent: 调用 web_fetch("https://very-slow-site.com")
       ... 等了 5 分钟还没响应 ...
用户: "算了，不要了"
Agent: ??? (工具还在跑，怎么停？)
```

**没有超时机制的后果：**
- 资源被长期占用（内存、连接、进程）
- 用户体验极差（干等着）
- 成本浪费（持续计费）
- 系统不稳定（累积的僵尸任务）

## 核心概念

### 1. 超时层级

工具执行涉及多层超时：

```
┌─────────────────────────────────────────────┐
│ Agent Turn Timeout (整个轮次)               │
│  ┌─────────────────────────────────────┐    │
│  │ Tool Call Timeout (单次工具调用)    │    │
│  │  ┌─────────────────────────────────┐│    │
│  │  │ Network/IO Timeout (网络/IO)   ││    │
│  │  └─────────────────────────────────┘│    │
│  └─────────────────────────────────────┘    │
└─────────────────────────────────────────────┘
```

每层都需要合理的超时设置：

```typescript
// pi-mono 风格的超时配置
interface TimeoutConfig {
  turnTimeout: number;      // 整个 Agent 轮次: 5-10 分钟
  toolTimeout: number;      // 单个工具调用: 30-120 秒
  networkTimeout: number;   // 网络请求: 10-30 秒
}
```

### 2. 取消信号传递

当需要取消时，信号需要层层传递：

```
用户取消 → Agent Runtime → Tool Executor → 实际操作
                ↓               ↓              ↓
           设置取消标记    检查信号并退出   中断进程/请求
```

## 实现模式

### 模式 1: AbortController 模式 (推荐)

Web 标准的取消机制，Node.js 原生支持：

```typescript
// learn-claude-code / pi-mono 中的实现思路
class ToolExecutor {
  private activeControllers = new Map<string, AbortController>();
  
  async execute(
    toolName: string,
    params: unknown,
    options: { timeout?: number; signal?: AbortSignal }
  ) {
    // 创建本次调用的 AbortController
    const controller = new AbortController();
    const callId = crypto.randomUUID();
    this.activeControllers.set(callId, controller);
    
    // 如果外部传入信号，级联绑定
    if (options.signal) {
      options.signal.addEventListener('abort', () => {
        controller.abort(options.signal!.reason);
      });
    }
    
    // 设置超时
    const timeout = options.timeout ?? 30000;
    const timeoutId = setTimeout(() => {
      controller.abort(new Error(`Tool timeout after ${timeout}ms`));
    }, timeout);
    
    try {
      const tool = this.tools.get(toolName);
      return await tool.execute(params, controller.signal);
    } finally {
      clearTimeout(timeoutId);
      this.activeControllers.delete(callId);
    }
  }
  
  // 取消所有正在执行的工具
  cancelAll(reason?: string) {
    for (const [id, controller] of this.activeControllers) {
      controller.abort(reason ?? 'Cancelled by user');
    }
    this.activeControllers.clear();
  }
}
```

### 模式 2: 工具内部响应取消信号

工具需要主动检查和响应取消：

```typescript
// 一个支持取消的 web_fetch 工具
async function webFetch(
  url: string,
  signal?: AbortSignal
): Promise<string> {
  // fetch 原生支持 AbortSignal
  const response = await fetch(url, { signal });
  
  // 大文件流式读取时检查信号
  const reader = response.body!.getReader();
  const chunks: Uint8Array[] = [];
  
  while (true) {
    // 每个 chunk 前检查是否被取消
    if (signal?.aborted) {
      reader.cancel();
      throw new Error('Fetch cancelled');
    }
    
    const { done, value } = await reader.read();
    if (done) break;
    chunks.push(value);
  }
  
  return new TextDecoder().decode(Buffer.concat(chunks));
}
```

### 模式 3: 子进程超时控制

执行 shell 命令时的超时处理：

```typescript
// OpenClaw exec 工具的超时实现思路
import { spawn, ChildProcess } from 'child_process';

interface ExecOptions {
  command: string;
  timeout?: number;    // 毫秒
  signal?: AbortSignal;
}

async function execWithTimeout(options: ExecOptions): Promise<string> {
  const { command, timeout = 30000, signal } = options;
  
  return new Promise((resolve, reject) => {
    const child = spawn('sh', ['-c', command]);
    let stdout = '';
    let killed = false;
    
    // 超时处理
    const timeoutId = setTimeout(() => {
      killed = true;
      // 先尝试 SIGTERM，给进程清理机会
      child.kill('SIGTERM');
      // 5 秒后如果还没死，强制 SIGKILL
      setTimeout(() => {
        if (!child.killed) child.kill('SIGKILL');
      }, 5000);
      reject(new Error(`Command timeout after ${timeout}ms`));
    }, timeout);
    
    // 响应外部取消信号
    signal?.addEventListener('abort', () => {
      if (!killed) {
        killed = true;
        child.kill('SIGTERM');
        reject(new Error('Command cancelled'));
      }
    });
    
    child.stdout.on('data', (data) => { stdout += data; });
    child.on('close', (code) => {
      clearTimeout(timeoutId);
      if (!killed) {
        resolve(stdout);
      }
    });
  });
}
```

### 模式 4: 带重试的超时

超时后可能需要重试：

```typescript
async function executeWithRetry<T>(
  fn: (signal: AbortSignal) => Promise<T>,
  options: {
    timeout: number;
    retries: number;
    backoff: (attempt: number) => number;
  }
): Promise<T> {
  let lastError: Error | undefined;
  
  for (let attempt = 0; attempt <= options.retries; attempt++) {
    const controller = new AbortController();
    const timeoutId = setTimeout(() => {
      controller.abort(new Error('Timeout'));
    }, options.timeout);
    
    try {
      const result = await fn(controller.signal);
      clearTimeout(timeoutId);
      return result;
    } catch (error) {
      clearTimeout(timeoutId);
      lastError = error as Error;
      
      // 如果是超时，考虑重试
      if (error instanceof Error && error.message === 'Timeout') {
        if (attempt < options.retries) {
          const delay = options.backoff(attempt);
          console.log(`Attempt ${attempt + 1} timed out, retrying in ${delay}ms`);
          await sleep(delay);
          continue;
        }
      }
      // 其他错误直接抛出
      throw error;
    }
  }
  
  throw lastError;
}

// 使用
const result = await executeWithRetry(
  (signal) => webFetch('https://slow-api.com', signal),
  {
    timeout: 10000,      // 10 秒超时
    retries: 3,          // 最多重试 3 次
    backoff: (n) => Math.min(1000 * Math.pow(2, n), 10000)  // 指数退避
  }
);
```

## OpenClaw 中的实际应用

### exec 工具的 timeout 参数

```typescript
// OpenClaw 的 exec 工具 schema
{
  name: "exec",
  parameters: {
    command: { type: "string" },
    timeout: { 
      type: "number",
      description: "Timeout in seconds (optional, kills process on expiry)"
    },
    yieldMs: {
      type: "number", 
      description: "Milliseconds to wait before backgrounding"
    }
  }
}
```

**yieldMs 的妙用：** 不是超时杀死，而是超时后台化：

```
exec(command="npm run build", yieldMs=10000)

执行流程:
0s     开始执行
10s    还没完成 → 转后台继续，返回 session ID
...    (后台继续跑)
用户可以用 process(action="poll") 检查状态
```

### process 工具的会话管理

```typescript
// 轮询等待，带超时
await process({
  action: "poll",
  sessionId: "abc123",
  timeout: 60000  // 最多等 60 秒
});

// 直接杀掉
await process({
  action: "kill",
  sessionId: "abc123"
});
```

## 用户取消的处理

### 场景: 用户说"停"或"算了"

```typescript
// Agent runtime 中的取消逻辑
class AgentRuntime {
  private currentExecution: AbortController | null = null;
  
  async handleMessage(message: string) {
    // 检测取消意图
    if (this.isCancelIntent(message)) {
      if (this.currentExecution) {
        this.currentExecution.abort('User cancelled');
        this.currentExecution = null;
        return "好的，已取消当前操作。";
      }
      return "没有正在进行的操作。";
    }
    
    // 正常执行
    this.currentExecution = new AbortController();
    try {
      return await this.runAgentLoop(message, this.currentExecution.signal);
    } finally {
      this.currentExecution = null;
    }
  }
  
  private isCancelIntent(message: string): boolean {
    const cancelPatterns = [
      /^(停|stop|cancel|算了|不要了|取消)/i,
      /^(别.{0,3}了|不用了)/,
    ];
    return cancelPatterns.some(p => p.test(message.trim()));
  }
}
```

## 最佳实践

### 1. 超时值设置指南

```typescript
const TIMEOUT_PRESETS = {
  // 快速操作 (文件读写、内存计算)
  fast: 5_000,      // 5 秒
  
  // 网络请求 (API 调用、网页抓取)
  network: 30_000,  // 30 秒
  
  // 重操作 (代码构建、大文件处理)
  heavy: 120_000,   // 2 分钟
  
  // 交互式操作 (需要用户输入)
  interactive: 300_000,  // 5 分钟
  
  // 长任务 (部署、大规模数据处理)
  // 建议: 不要用超时，用 background + polling
  longRunning: Infinity
};
```

### 2. 优雅退出 vs 强制终止

```typescript
async function gracefulKill(process: ChildProcess, gracePeriod = 5000) {
  // 第一步: 温和请求退出
  process.kill('SIGTERM');
  
  // 等待优雅退出
  const exited = await Promise.race([
    new Promise(resolve => process.on('exit', () => resolve(true))),
    sleep(gracePeriod).then(() => false)
  ]);
  
  // 如果还没退出，强制杀死
  if (!exited) {
    console.warn('Process did not exit gracefully, sending SIGKILL');
    process.kill('SIGKILL');
  }
}
```

### 3. 资源清理

```typescript
class ResourceAwareExecutor {
  async executeWithCleanup<T>(
    execute: (signal: AbortSignal) => Promise<T>,
    cleanup: () => Promise<void>,
    timeout: number
  ): Promise<T> {
    const controller = new AbortController();
    const timeoutId = setTimeout(() => controller.abort(), timeout);
    
    try {
      return await execute(controller.signal);
    } catch (error) {
      // 出错或超时都要清理
      await cleanup().catch(e => {
        console.error('Cleanup failed:', e);
      });
      throw error;
    } finally {
      clearTimeout(timeoutId);
    }
  }
}

// 示例: 临时文件操作
await executor.executeWithCleanup(
  async (signal) => {
    const tmpFile = await createTempFile();
    return await processFile(tmpFile, signal);
  },
  async () => {
    await removeTempFiles();
  },
  30000
);
```

### 4. 超时信息要有用

```typescript
// ❌ 不好的错误信息
throw new Error('Timeout');

// ✅ 好的错误信息
throw new Error(
  `Tool 'web_fetch' timed out after 30s while fetching https://example.com. ` +
  `The server may be slow or unreachable. ` +
  `Consider: 1) Retry later 2) Use a different source 3) Increase timeout`
);
```

## 调试技巧

### 1. 超时追踪

```typescript
function withTimeoutTracking<T>(
  name: string,
  promise: Promise<T>,
  timeout: number
): Promise<T> {
  const start = Date.now();
  
  const timeoutPromise = new Promise<never>((_, reject) => {
    setTimeout(() => {
      const elapsed = Date.now() - start;
      reject(new Error(
        `${name} timed out after ${elapsed}ms (limit: ${timeout}ms)`
      ));
    }, timeout);
  });
  
  return Promise.race([promise, timeoutPromise]).finally(() => {
    const elapsed = Date.now() - start;
    if (elapsed > timeout * 0.8) {
      console.warn(`${name} took ${elapsed}ms, approaching timeout of ${timeout}ms`);
    }
  });
}
```

### 2. 慢操作检测

```typescript
// 在开发环境警告慢操作
const SLOW_THRESHOLD = 5000;

async function detectSlowTool<T>(
  toolName: string,
  execute: () => Promise<T>
): Promise<T> {
  const start = Date.now();
  const warnTimeout = setTimeout(() => {
    console.warn(`⚠️ ${toolName} is taking longer than ${SLOW_THRESHOLD}ms`);
  }, SLOW_THRESHOLD);
  
  try {
    return await execute();
  } finally {
    clearTimeout(warnTimeout);
    const elapsed = Date.now() - start;
    if (elapsed > SLOW_THRESHOLD) {
      console.log(`📊 ${toolName} completed in ${elapsed}ms (slow)`);
    }
  }
}
```

## 小结

| 层级 | 机制 | 用途 |
|------|------|------|
| 网络层 | fetch signal | HTTP 请求取消 |
| 进程层 | SIGTERM/SIGKILL | 子进程终止 |
| 工具层 | AbortController | 单工具超时 |
| 轮次层 | Turn timeout | 整体执行限制 |
| 用户层 | Cancel intent | 用户主动取消 |

**核心原则：**
1. 每个异步操作都应该可取消
2. 超时要合理，不能太短（失败）也不能太长（卡死）
3. 取消后必须清理资源
4. 错误信息要帮助定位问题
5. 长任务用 background + polling，别硬等

下节课我们讲 **Agent Warm-up & Cold Start**，探讨如何优化 Agent 的启动性能。
