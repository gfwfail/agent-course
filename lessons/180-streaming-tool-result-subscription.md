# 180 - Agent 实时流式工具结果订阅（Real-time Streaming Tool Result Subscription）

> "工具调用不一定要等结果出来再继续——让结果自己流过来。"

---

## 为什么需要流式工具结果？

传统工具调用是同步 request-response：

```
Agent → call_tool(args) → [等待...] → result → Agent 继续
```

对于这些场景，这种模式会成为瓶颈：

| 场景 | 问题 |
|------|------|
| 长时间运行的任务（爬虫、编译） | 等待期间 context 空转 |
| 实时数据（股价、日志尾随） | 批量返回导致数据延迟 |
| 大文件逐行处理 | 必须全量加载才能返回 |
| 多工具并发聚合 | 最慢的工具拖慢全部 |

解决方案：**工具结果改用流式订阅**，Agent Loop 接受 `AsyncGenerator` 而不是 `Promise<string>`。

---

## 核心概念

```
Agent
  │
  ├── call_tool("tail_logs", {lines: 100})
  │         │
  │         ▼  AsyncGenerator<ToolChunk>
  │    ┌─────────────┐
  │    │  chunk 1    │──→ Agent 处理 / 实时转发
  │    │  chunk 2    │──→ ...
  │    │  chunk 3    │──→ ...
  │    │  [done]     │
  │    └─────────────┘
  │
  └── 继续下一个工具或生成回复（不必等所有 chunk）
```

---

## TypeScript 实现

### 1. 流式工具定义

```typescript
// tool-types.ts
export type ToolChunk =
  | { type: 'data';  content: string }
  | { type: 'progress'; percent: number; message: string }
  | { type: 'error'; code: string; message: string }
  | { type: 'done';  summary: string };

export interface StreamingTool {
  name: string;
  description: string;
  schema: Record<string, unknown>;
  /** 流式工具返回 AsyncGenerator */
  execute(args: Record<string, unknown>): AsyncGenerator<ToolChunk>;
}

export interface RegularTool {
  name: string;
  description: string;
  schema: Record<string, unknown>;
  execute(args: Record<string, unknown>): Promise<string>;
}

export type AnyTool = StreamingTool | RegularTool;

export function isStreamingTool(t: AnyTool): t is StreamingTool {
  // 用 execute 返回的对象是否有 Symbol.asyncIterator 来检测
  return t.execute.constructor.name === 'AsyncGeneratorFunction'
    || t.execute.toString().includes('yield');
}
```

### 2. 三个实用流式工具

```typescript
// streaming-tools.ts
import { StreamingTool, ToolChunk } from './tool-types';
import { createReadStream } from 'fs';
import { createInterface } from 'readline';

/** 工具1：实时尾随日志文件 */
export const tailLogsTool: StreamingTool = {
  name: 'tail_logs',
  description: '实时流式读取日志文件最后 N 行，适合大文件',
  schema: {
    file: { type: 'string', description: '日志文件路径' },
    lines: { type: 'number', description: '读取行数', default: 50 },
  },
  async *execute({ file, lines = 50 }: { file: string; lines: number }) {
    const rl = createInterface({
      input: createReadStream(file as string),
      crlfDelay: Infinity,
    });

    let count = 0;
    for await (const line of rl) {
      yield { type: 'data' as const, content: line + '\n' };
      count++;
      if (count % 10 === 0) {
        yield { type: 'progress' as const, percent: -1, message: `已读取 ${count} 行` };
      }
    }
    yield { type: 'done' as const, summary: `共读取 ${count} 行` };
  },
};

/** 工具2：逐步执行 shell 命令并流式返回输出 */
export const execStreamTool: StreamingTool = {
  name: 'exec_stream',
  description: '执行命令并实时流式返回 stdout/stderr',
  schema: {
    command: { type: 'string', description: '要执行的 shell 命令' },
  },
  async *execute({ command }: { command: string }) {
    const { spawn } = await import('child_process');
    const child = spawn('sh', ['-c', command as string]);

    // 用 AsyncGenerator 包装事件流
    const chunks: ToolChunk[] = [];
    let resolve: (() => void) | null = null;
    let done = false;

    const push = (chunk: ToolChunk) => {
      chunks.push(chunk);
      resolve?.();
      resolve = null;
    };

    child.stdout.on('data', (data) => push({ type: 'data', content: data.toString() }));
    child.stderr.on('data', (data) => push({ type: 'data', content: `[stderr] ${data}` }));
    child.on('close', (code) => {
      done = true;
      push({ type: 'done', summary: `exit code: ${code}` });
      resolve?.();
    });

    while (!done || chunks.length > 0) {
      if (chunks.length === 0) {
        await new Promise<void>((r) => { resolve = r; });
      }
      while (chunks.length > 0) {
        yield chunks.shift()!;
      }
    }
  },
};

/** 工具3：轮询 API 直到条件满足 */
export const pollUntilTool: StreamingTool = {
  name: 'poll_until',
  description: '轮询 URL 直到响应包含指定内容，流式报告进度',
  schema: {
    url: { type: 'string' },
    contains: { type: 'string', description: '等待出现的关键词' },
    intervalMs: { type: 'number', default: 2000 },
    timeoutMs: { type: 'number', default: 60000 },
  },
  async *execute({ url, contains, intervalMs = 2000, timeoutMs = 60000 }:
    { url: string; contains: string; intervalMs: number; timeoutMs: number }) {
    const start = Date.now();
    let attempt = 0;

    while (Date.now() - start < timeoutMs) {
      attempt++;
      const elapsed = Date.now() - start;
      const percent = Math.min(100, Math.round((elapsed / timeoutMs) * 100));

      try {
        const res = await fetch(url as string);
        const text = await res.text();
        if (text.includes(contains as string)) {
          yield { type: 'done', summary: `✅ 第 ${attempt} 次轮询命中：找到 "${contains}"` };
          return;
        }
        yield { type: 'progress', percent, message: `尝试 #${attempt}，未命中，继续等待...` };
      } catch (err) {
        yield { type: 'error', code: 'FETCH_ERROR', message: String(err) };
      }

      await new Promise((r) => setTimeout(r, intervalMs as number));
    }

    yield { type: 'error', code: 'TIMEOUT', message: `${timeoutMs}ms 超时，条件未满足` };
  },
};
```

### 3. 支持流式工具的 Agent Loop

```typescript
// streaming-agent-loop.ts
import Anthropic from '@anthropic-ai/sdk';
import { AnyTool, ToolChunk, isStreamingTool } from './tool-types';
import { tailLogsTool, execStreamTool, pollUntilTool } from './streaming-tools';

const client = new Anthropic();

/** 把流式工具的 chunks 聚合成最终结果字符串 */
async function executeStreamingTool(
  tool: ReturnType<typeof tailLogsTool.execute extends (...args: any) => infer R ? () => R : never>,
  onChunk: (chunk: ToolChunk) => void,
): Promise<string> {
  const parts: string[] = [];
  for await (const chunk of tool) {
    onChunk(chunk);  // 实时回调（可以转发给用户）
    if (chunk.type === 'data')     parts.push(chunk.content);
    if (chunk.type === 'done')     parts.push(`\n[完成] ${chunk.summary}`);
    if (chunk.type === 'progress') parts.push(`\n[进度 ${chunk.percent}%] ${chunk.message}`);
    if (chunk.type === 'error')    parts.push(`\n[错误 ${chunk.code}] ${chunk.message}`);
  }
  return parts.join('');
}

const tools: AnyTool[] = [tailLogsTool, execStreamTool, pollUntilTool];

async function runAgent(userMessage: string) {
  const messages: Anthropic.MessageParam[] = [
    { role: 'user', content: userMessage },
  ];

  // 构建 Anthropic tool schema
  const toolSchemas = tools.map((t) => ({
    name: t.name,
    description: t.description,
    input_schema: { type: 'object' as const, properties: t.schema },
  }));

  while (true) {
    const response = await client.messages.create({
      model: 'claude-opus-4-5',
      max_tokens: 4096,
      tools: toolSchemas,
      messages,
    });

    // 收集 LLM 回复
    messages.push({ role: 'assistant', content: response.content });

    if (response.stop_reason === 'end_turn') {
      const text = response.content
        .filter((b) => b.type === 'text')
        .map((b) => (b as Anthropic.TextBlock).text)
        .join('');
      console.log('\n🤖 Agent:', text);
      break;
    }

    if (response.stop_reason === 'tool_use') {
      const toolResults: Anthropic.ToolResultBlockParam[] = [];

      for (const block of response.content) {
        if (block.type !== 'tool_use') continue;

        const tool = tools.find((t) => t.name === block.name);
        if (!tool) {
          toolResults.push({ type: 'tool_result', tool_use_id: block.id, content: 'Tool not found' });
          continue;
        }

        let result: string;
        if (isStreamingTool(tool)) {
          // 流式执行：实时打印进度
          const generator = tool.execute(block.input as Record<string, unknown>);
          result = await executeStreamingTool(
            generator as any,
            (chunk) => {
              if (chunk.type === 'progress') {
                process.stdout.write(`\r⏳ ${chunk.message}`);
              } else if (chunk.type === 'data') {
                process.stdout.write(chunk.content);
              }
            },
          );
        } else {
          // 普通工具：直接 await
          result = await (tool as any).execute(block.input);
        }

        toolResults.push({ type: 'tool_result', tool_use_id: block.id, content: result });
      }

      messages.push({ role: 'user', content: toolResults });
    }
  }
}

// 使用示例
runAgent('帮我执行 ls -la /tmp 并分析输出');
```

---

## 在 OpenClaw 中的对应实现

OpenClaw 的 `exec` 工具本质上就是流式工具的生产级实现：

```typescript
// OpenClaw exec 工具支持 yieldMs 参数
// 内部用 backgrounding 实现"不等所有输出就先返回"
{
  action: "exec",
  command: "npm run build",
  yieldMs: 5000,   // 5秒后后台化，继续其他工作
  background: false
}
```

`yieldMs` 是流式工具的实用近似：**不阻塞 context，先看前几秒输出，后续用 process(poll) 追取剩余**。

---

## 流式 vs 轮询：选哪个？

| 场景 | 推荐方式 |
|------|---------|
| 命令执行输出 | AsyncGenerator（push 模型） |
| 外部 API 任务 | 轮询 + 指数退避（pull 模型） |
| 实时数据订阅 | WebSocket + Generator |
| 大文件逐行 | ReadStream + Generator |
| 短任务（<2s） | 普通 await 即可，别过度设计 |

---

## 关键设计原则

1. **chunk 类型统一** — `data / progress / error / done` 四种，LLM 消费时知道怎么解读
2. **聚合后再注入** — Generator 跑完才把结果写入 `tool_result`，LLM 看到的仍是完整字符串
3. **实时转发给用户** — `onChunk` 回调实现用户侧流式体验，跟 LLM 侧解耦
4. **超时必须有** — Generator 内部加 `timeoutMs` 保障，别让 Agent Loop 永远等下去
5. **AbortSignal 传播** — 用户取消时能中断 Generator（对接 Lesson 30 工具超时取消）

---

## 总结

```
传统工具：call → [黑盒等待] → result
流式工具：call → chunk₁ → chunk₂ → ... → done
```

流式工具结果订阅让 Agent 从"等结果"变成"看结果流入"，适合所有耗时超过 2 秒或需要实时反馈的工具。配合 OpenClaw 的 `exec(yieldMs)` 模式，是目前最务实的长任务处理方案。

---

*下一课预告：Agent 工具调用热路径优化（Hot Path Optimization）— 识别高频工具路径并针对性加速*
