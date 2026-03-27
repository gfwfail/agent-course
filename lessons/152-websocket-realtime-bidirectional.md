# 152. Agent WebSocket 实时双向通信（WebSocket Real-Time Bidirectional Communication）

> HTTP 是一问一答，Agent 需要持续对话。WebSocket 让 Agent 与客户端保持**永久连接**，实时推送工具执行进度、接收外部事件触发——这是构建响应式 Agent 应用的关键基础设施。

---

## 为什么 Agent 需要 WebSocket？

传统 HTTP 模式下 Agent 的痛点：
- 工具执行需要 30 秒，客户端只能傻等
- 无法推送中间进度（"正在搜索第 3 页..."）
- 外部事件（新邮件、告警触发）无法主动通知 Agent
- 长轮询浪费带宽，SSE 只能单向

WebSocket 解决方案：
```
客户端 ←──── 双向持久连接 ────→ Agent Server
              ↑双向消息流↑
  工具执行进度推送 ←──────────── Agent
  外部触发事件  ──────────────→ Agent
  取消/暂停指令 ──────────────→ Agent
  流式思考过程  ←──────────────── Agent
```

---

## 核心架构

### 消息协议设计

```typescript
// 双向消息类型定义
type ClientMessage =
  | { type: 'run'; taskId: string; prompt: string; context?: Record<string, unknown> }
  | { type: 'cancel'; taskId: string }
  | { type: 'input'; taskId: string; value: string }   // Human-in-the-loop 输入
  | { type: 'ping' };

type ServerMessage =
  | { type: 'thinking'; taskId: string; text: string }
  | { type: 'tool_start'; taskId: string; tool: string; args: unknown }
  | { type: 'tool_result'; taskId: string; tool: string; result: unknown; durationMs: number }
  | { type: 'output'; taskId: string; text: string; done: boolean }
  | { type: 'error'; taskId: string; code: string; message: string }
  | { type: 'input_required'; taskId: string; question: string }
  | { type: 'pong'; serverTime: number };
```

### Server 端实现（Node.js + ws）

```typescript
import { WebSocketServer, WebSocket } from 'ws';
import { createServer } from 'http';

interface AgentSession {
  ws: WebSocket;
  tasks: Map<string, AbortController>;
  userId: string;
}

const sessions = new Map<string, AgentSession>();

const wss = new WebSocketServer({ server: createServer() });

wss.on('connection', (ws, req) => {
  const sessionId = crypto.randomUUID();
  const userId = authenticateRequest(req); // JWT / session token

  const session: AgentSession = {
    ws,
    tasks: new Map(),
    userId,
  };
  sessions.set(sessionId, session);

  const send = (msg: ServerMessage) => {
    if (ws.readyState === WebSocket.OPEN) {
      ws.send(JSON.stringify(msg));
    }
  };

  ws.on('message', async (raw) => {
    const msg: ClientMessage = JSON.parse(raw.toString());

    switch (msg.type) {
      case 'ping':
        send({ type: 'pong', serverTime: Date.now() });
        break;

      case 'run': {
        const abort = new AbortController();
        session.tasks.set(msg.taskId, abort);

        // 异步执行，实时推送进度
        runAgentTask({
          taskId: msg.taskId,
          prompt: msg.prompt,
          signal: abort.signal,
          onThinking: (text) => send({ type: 'thinking', taskId: msg.taskId, text }),
          onToolStart: (tool, args) => send({ type: 'tool_start', taskId: msg.taskId, tool, args }),
          onToolResult: (tool, result, durationMs) =>
            send({ type: 'tool_result', taskId: msg.taskId, tool, result, durationMs }),
          onOutput: (text, done) => send({ type: 'output', taskId: msg.taskId, text, done }),
          onInputRequired: (question) =>
            send({ type: 'input_required', taskId: msg.taskId, question }),
          onError: (code, message) => send({ type: 'error', taskId: msg.taskId, code, message }),
        }).finally(() => session.tasks.delete(msg.taskId));
        break;
      }

      case 'cancel': {
        const abort = session.tasks.get(msg.taskId);
        abort?.abort('user_cancelled');
        session.tasks.delete(msg.taskId);
        break;
      }

      case 'input': {
        // 向等待人工输入的任务注入答案
        pendingInputs.set(msg.taskId, msg.value);
        break;
      }
    }
  });

  ws.on('close', () => {
    // 取消所有未完成任务
    for (const abort of session.tasks.values()) {
      abort.abort('connection_closed');
    }
    sessions.delete(sessionId);
  });
});
```

---

## Agent Loop 与 WebSocket 集成

```typescript
// 关键：工具执行时注入进度回调
async function runAgentTask(opts: {
  taskId: string;
  prompt: string;
  signal: AbortSignal;
  onThinking: (text: string) => void;
  onToolStart: (tool: string, args: unknown) => void;
  onToolResult: (tool: string, result: unknown, ms: number) => void;
  onOutput: (text: string, done: boolean) => void;
  onInputRequired: (question: string) => Promise<string>;
  onError: (code: string, msg: string) => void;
}) {
  const messages: Message[] = [{ role: 'user', content: opts.prompt }];

  while (true) {
    if (opts.signal.aborted) {
      opts.onError('cancelled', 'Task was cancelled');
      return;
    }

    // 流式调用 LLM
    const stream = await anthropic.messages.stream({
      model: 'claude-opus-4-5',
      messages,
      tools: toolDefinitions,
    });

    // 实时推送思考过程
    for await (const chunk of stream) {
      if (chunk.type === 'content_block_delta' && chunk.delta.type === 'text_delta') {
        opts.onThinking(chunk.delta.text);
      }
    }

    const response = await stream.finalMessage();

    // 检查是否需要工具调用
    const toolUses = response.content.filter(b => b.type === 'tool_use');
    if (toolUses.length === 0) {
      // 最终输出
      const text = response.content
        .filter(b => b.type === 'text')
        .map(b => b.text)
        .join('');
      opts.onOutput(text, true);
      return;
    }

    // 并发执行工具，实时推送每个工具的状态
    const toolResults = await Promise.all(
      toolUses.map(async (toolUse) => {
        opts.onToolStart(toolUse.name, toolUse.input);
        const start = Date.now();
        try {
          // Human-in-the-loop：特殊工具需等待人工输入
          if (toolUse.name === 'ask_human') {
            const question = (toolUse.input as any).question;
            const answer = await waitForHumanInput(opts.taskId, question, opts.onInputRequired);
            const result = { answer };
            opts.onToolResult(toolUse.name, result, Date.now() - start);
            return { tool_use_id: toolUse.id, content: JSON.stringify(result) };
          }

          const result = await executeToolWithAbort(toolUse.name, toolUse.input, opts.signal);
          opts.onToolResult(toolUse.name, result, Date.now() - start);
          return { tool_use_id: toolUse.id, content: JSON.stringify(result) };
        } catch (err: any) {
          const error = { error: err.message };
          opts.onToolResult(toolUse.name, error, Date.now() - start);
          return { tool_use_id: toolUse.id, content: JSON.stringify(error) };
        }
      })
    );

    // 追加到消息历史，继续循环
    messages.push({ role: 'assistant', content: response.content });
    messages.push({ role: 'user', content: toolResults.map(r => ({ type: 'tool_result', ...r })) });
  }
}

// Human-in-the-loop：挂起等待 WebSocket 输入
const pendingInputs = new Map<string, string>();

async function waitForHumanInput(
  taskId: string,
  question: string,
  notify: (q: string) => void
): Promise<string> {
  notify(question);
  return new Promise((resolve) => {
    const interval = setInterval(() => {
      const answer = pendingInputs.get(taskId);
      if (answer !== undefined) {
        pendingInputs.delete(taskId);
        clearInterval(interval);
        resolve(answer);
      }
    }, 100);
  });
}
```

---

## 客户端实现（React 示例）

```typescript
// hooks/useAgentWebSocket.ts
import { useEffect, useRef, useState, useCallback } from 'react';

interface ToolEvent {
  tool: string;
  args: unknown;
  result?: unknown;
  durationMs?: number;
}

interface AgentState {
  status: 'idle' | 'running' | 'waiting_input' | 'done' | 'error';
  thinking: string;
  toolEvents: ToolEvent[];
  output: string;
  inputQuestion?: string;
}

export function useAgentWebSocket(url: string) {
  const ws = useRef<WebSocket | null>(null);
  const [state, setState] = useState<AgentState>({
    status: 'idle', thinking: '', toolEvents: [], output: '',
  });

  useEffect(() => {
    ws.current = new WebSocket(url);

    ws.current.onmessage = (event) => {
      const msg: ServerMessage = JSON.parse(event.data);

      setState(prev => {
        switch (msg.type) {
          case 'thinking':
            return { ...prev, thinking: prev.thinking + msg.text };

          case 'tool_start':
            return {
              ...prev,
              toolEvents: [...prev.toolEvents, { tool: msg.tool, args: msg.args }],
            };

          case 'tool_result': {
            const events = [...prev.toolEvents];
            const last = events.findIndex(e => e.tool === msg.tool && !e.result);
            if (last !== -1) events[last] = { ...events[last], result: msg.result, durationMs: msg.durationMs };
            return { ...prev, toolEvents: events };
          }

          case 'output':
            return {
              ...prev,
              output: prev.output + msg.text,
              status: msg.done ? 'done' : 'running',
            };

          case 'input_required':
            return { ...prev, status: 'waiting_input', inputQuestion: msg.question };

          case 'error':
            return { ...prev, status: 'error', output: msg.message };

          default:
            return prev;
        }
      });
    };

    // 心跳保活
    const heartbeat = setInterval(() => {
      ws.current?.send(JSON.stringify({ type: 'ping' }));
    }, 30_000);

    return () => {
      clearInterval(heartbeat);
      ws.current?.close();
    };
  }, [url]);

  const run = useCallback((prompt: string) => {
    const taskId = crypto.randomUUID();
    setState({ status: 'running', thinking: '', toolEvents: [], output: '' });
    ws.current?.send(JSON.stringify({ type: 'run', taskId, prompt }));
    return taskId;
  }, []);

  const cancel = useCallback((taskId: string) => {
    ws.current?.send(JSON.stringify({ type: 'cancel', taskId }));
    setState(prev => ({ ...prev, status: 'idle' }));
  }, []);

  const submitInput = useCallback((taskId: string, value: string) => {
    ws.current?.send(JSON.stringify({ type: 'input', taskId, value }));
    setState(prev => ({ ...prev, status: 'running', inputQuestion: undefined }));
  }, []);

  return { state, run, cancel, submitInput };
}
```

---

## SSE vs WebSocket 选型

| 场景 | 推荐方案 |
|------|---------|
| 只需服务器推送进度 | SSE（更简单，自动重连）|
| 需要客户端发送取消/输入 | **WebSocket**（双向通信）|
| 移动端弱网环境 | WebSocket（心跳+重连机制）|
| 多任务并发 | WebSocket（一个连接多路复用）|
| 纯静态部署 / CDN 边缘 | SSE（更友好）|

---

## OpenClaw 中的实践

OpenClaw Gateway 本质上就是一个 WebSocket-first 的 Agent 服务器：

```
Telegram Bot / CLI / Web UI
         ↓ (消息/指令)
   OpenClaw Gateway (WebSocket server)
         ↓
   Agent Loop (claude-sonnet-4-5)
         ↓ (工具调用实时回调)
   Tool Registry (exec / browser / nodes / cron ...)
```

`sessions_spawn` 创建的子 Agent 通过 WebSocket 向父 Session 推送进度，这正是 `streamTo: "parent"` 参数的工作原理——子 Agent 的每条输出都实时流入父会话的 WebSocket 连接。

---

## 生产注意事项

1. **重连策略**：指数退避 + jitter，最大间隔 30s
2. **消息去重**：每条消息带 `seq` 序号，断线重连后从 `lastSeq` 续传
3. **连接限制**：每用户最多 N 个并发 WebSocket（防资源耗尽）
4. **Nginx 配置**：必须设置 `proxy_read_timeout` 和 `proxy_send_timeout`，默认 60s 会踢掉长任务
5. **负载均衡**：WebSocket 需要 sticky session（或用 Redis Pub/Sub 跨节点广播）

```nginx
location /agent/ws {
    proxy_pass http://agent_backend;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_read_timeout 3600s;   # Agent 任务可能跑很久
    proxy_send_timeout 3600s;
}
```

---

## 总结

| 要点 | 核心思路 |
|------|---------|
| 双向协议 | 客户端发 run/cancel/input，服务端推 thinking/tool/output |
| 取消传播 | AbortController 贯穿整个 Agent Loop + 工具执行 |
| Human-in-the-loop | 工具挂起 → WS 推送问题 → 等待客户端 input 消息 |
| 心跳保活 | 30s ping/pong 防 NAT 超时断连 |
| 多路复用 | 一个 WS 连接承载多个并发 taskId |

**一句话**：WebSocket 把 Agent 从"黑盒请求"变成"透明实时协作"——用户看得到每一步，也控制得了每一步。
