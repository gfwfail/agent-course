# Agent API 设计最佳实践（Agent API Design Best Practices）

> 你的 Agent 跑得再好，没有好的 API 层，前端开发者也会骂娘。这一课讲如何为 Agent 设计真正好用的 HTTP API。

## 核心挑战

Agent 和普通后端服务有本质区别：

| 维度 | 普通 API | Agent API |
|------|----------|-----------|
| 响应时间 | 几十毫秒 | 几秒到几分钟 |
| 响应模式 | 请求-响应 | 流式/异步/轮询 |
| 状态 | 无状态为主 | 有对话状态 |
| 错误 | 确定性错误 | 非确定性（LLM 幻觉） |
| 幂等性 | 可设计 | 复杂（LLM 输出不可预期） |

所以 Agent API 需要专门的设计模式。

---

## 1. 三种响应模式的选择

### 模式 A：流式 SSE（推荐用于对话）

```typescript
// server/routes/agent.ts (pi-mono 风格)
import { Router } from 'express';

const router = Router();

// POST /api/agent/chat
// 返回 SSE 流
router.post('/chat', async (req, res) => {
  const { message, sessionId } = req.body;

  // 设置 SSE 头
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');
  res.setHeader('X-Accel-Buffering', 'no'); // Nginx 不要缓冲

  const sendEvent = (event: string, data: unknown) => {
    res.write(`event: ${event}\n`);
    res.write(`data: ${JSON.stringify(data)}\n\n`);
  };

  // 客户端断开时清理
  let cancelled = false;
  req.on('close', () => { cancelled = true; });

  try {
    const agent = new Agent(sessionId);

    // 发送思考状态
    sendEvent('thinking', { status: 'started' });

    // 流式获取 Agent 输出
    for await (const chunk of agent.stream(message)) {
      if (cancelled) break;

      switch (chunk.type) {
        case 'thinking':
          sendEvent('thinking', { text: chunk.text });
          break;
        case 'tool_call':
          sendEvent('tool_call', {
            tool: chunk.tool,
            input: chunk.input,
            id: chunk.id
          });
          break;
        case 'tool_result':
          sendEvent('tool_result', {
            id: chunk.id,
            output: chunk.output,
            error: chunk.error
          });
          break;
        case 'text':
          sendEvent('text', { delta: chunk.delta });
          break;
        case 'done':
          sendEvent('done', {
            usage: chunk.usage,
            duration: chunk.duration
          });
          break;
      }
    }
  } catch (err) {
    sendEvent('error', { message: err.message, code: err.code });
  } finally {
    res.end();
  }
});
```

**前端消费 SSE：**

```typescript
// client/hooks/useAgent.ts
export function useAgentChat(sessionId: string) {
  const [messages, setMessages] = useState<Message[]>([]);
  const [thinking, setThinking] = useState(false);

  const send = async (userMessage: string) => {
    const eventSource = new EventSource(
      `/api/agent/chat`,
      // SSE 不支持 POST body，要用 fetch + ReadableStream
    );

    // 更好的方式：用 fetch + ReadableStream
    const response = await fetch('/api/agent/chat', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ message: userMessage, sessionId }),
    });

    const reader = response.body!.getReader();
    const decoder = new TextDecoder();
    let buffer = '';

    while (true) {
      const { done, value } = await reader.read();
      if (done) break;

      buffer += decoder.decode(value, { stream: true });
      const lines = buffer.split('\n');
      buffer = lines.pop() || '';

      for (const line of lines) {
        if (line.startsWith('data: ')) {
          const data = JSON.parse(line.slice(6));
          // 处理各种事件类型...
          handleEvent(data);
        }
      }
    }
  };

  return { messages, thinking, send };
}
```

---

### 模式 B：异步任务（推荐用于长任务）

```typescript
// 提交任务，立即返回 jobId
// POST /api/agent/tasks
router.post('/tasks', async (req, res) => {
  const { prompt, type } = req.body;

  const job = await taskQueue.enqueue({
    prompt,
    type,
    userId: req.user.id,
    createdAt: Date.now(),
  });

  // 202 Accepted，不是 200 OK
  res.status(202).json({
    jobId: job.id,
    status: 'queued',
    estimatedDuration: estimateDuration(type),
    statusUrl: `/api/agent/tasks/${job.id}`,
    // 可选：提供 webhook 通知
    webhookHint: '可在请求体中传 webhookUrl，任务完成后回调',
  });
});

// 查询任务状态
// GET /api/agent/tasks/:jobId
router.get('/tasks/:jobId', async (req, res) => {
  const job = await taskQueue.get(req.params.jobId);

  if (!job) {
    return res.status(404).json({ error: 'Task not found' });
  }

  // 权限检查
  if (job.userId !== req.user.id) {
    return res.status(403).json({ error: 'Forbidden' });
  }

  res.json({
    jobId: job.id,
    status: job.status, // queued | running | completed | failed
    progress: job.progress, // 0-100
    result: job.status === 'completed' ? job.result : undefined,
    error: job.status === 'failed' ? job.error : undefined,
    createdAt: job.createdAt,
    completedAt: job.completedAt,
    // 客户端轮询建议
    pollAfterMs: job.status === 'running' ? 2000 : undefined,
  });
});
```

---

### 模式 C：Webhook 回调（推荐用于后端集成）

```typescript
// POST /api/agent/tasks (带 webhook)
router.post('/tasks', async (req, res) => {
  const { prompt, webhookUrl, webhookSecret } = req.body;

  // 验证 webhook URL 合法性
  if (webhookUrl && !isValidWebhookUrl(webhookUrl)) {
    return res.status(400).json({ error: 'Invalid webhook URL' });
  }

  const job = await taskQueue.enqueue({
    prompt,
    webhookUrl,
    webhookSecret, // 用于签名验证
  });

  res.status(202).json({ jobId: job.id });
});

// 任务完成后触发 webhook
async function deliverWebhook(job: Job, result: AgentResult) {
  if (!job.webhookUrl) return;

  const payload = JSON.stringify({
    jobId: job.id,
    status: 'completed',
    result,
    timestamp: Date.now(),
  });

  // 用 HMAC 签名 (与 Stripe/GitHub 一样)
  const signature = createHmac('sha256', job.webhookSecret)
    .update(payload)
    .digest('hex');

  // 带重试的 webhook 投递
  for (let attempt = 0; attempt < 3; attempt++) {
    try {
      const resp = await fetch(job.webhookUrl, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'X-Agent-Signature': `sha256=${signature}`,
          'X-Agent-Delivery': job.id,
        },
        body: payload,
        signal: AbortSignal.timeout(10_000),
      });

      if (resp.ok) return; // 成功
      
      // 4xx 不重试（客户端错误）
      if (resp.status >= 400 && resp.status < 500) break;
    } catch {
      // 网络错误，等待后重试
      await sleep(2 ** attempt * 1000);
    }
  }
}
```

---

## 2. 会话管理 API

Agent 对话是有状态的，API 需要明确管理会话生命周期：

```typescript
// 创建会话
// POST /api/sessions
router.post('/sessions', async (req, res) => {
  const { persona, tools, config } = req.body;

  const session = await sessionStore.create({
    userId: req.user.id,
    persona: persona || 'default',
    tools: tools || getAllowedTools(req.user.tier),
    config: {
      model: config?.model || 'claude-sonnet-4-5',
      maxTokens: config?.maxTokens || 8192,
      temperature: config?.temperature || 0.7,
    },
    expiresAt: Date.now() + 24 * 60 * 60 * 1000, // 24h
  });

  res.status(201).json({
    sessionId: session.id,
    expiresAt: session.expiresAt,
    tools: session.tools.map(t => t.name),
  });
});

// 获取会话历史
// GET /api/sessions/:id/messages
router.get('/sessions/:id/messages', async (req, res) => {
  const { limit = 20, before } = req.query;

  const messages = await messageStore.list({
    sessionId: req.params.id,
    limit: Number(limit),
    before: before as string,
  });

  res.json({
    messages,
    hasMore: messages.length === Number(limit),
    nextCursor: messages.at(-1)?.id,
  });
});

// 删除会话（软删除）
// DELETE /api/sessions/:id
router.delete('/sessions/:id', async (req, res) => {
  await sessionStore.softDelete(req.params.id, req.user.id);
  res.status(204).end();
});
```

---

## 3. 错误响应标准化

Agent API 的错误要让前端和 LLM 都能理解：

```typescript
// errors/AgentError.ts
export class AgentApiError extends Error {
  constructor(
    public readonly code: string,         // 机器可读
    public readonly message: string,      // 人类可读
    public readonly httpStatus: number,
    public readonly details?: unknown,    // 调试信息
    public readonly retryAfterMs?: number // 限流时告知重试时间
  ) {
    super(message);
  }
}

// 错误码规范
export const ErrorCodes = {
  // 4xx - 客户端错误
  SESSION_NOT_FOUND: 'session_not_found',        // 404
  SESSION_EXPIRED: 'session_expired',              // 410
  RATE_LIMITED: 'rate_limited',                    // 429
  CONTEXT_TOO_LONG: 'context_too_long',           // 400
  INVALID_TOOL: 'invalid_tool',                    // 400
  QUOTA_EXCEEDED: 'quota_exceeded',               // 402

  // 5xx - 服务端错误
  LLM_UNAVAILABLE: 'llm_unavailable',             // 503
  TOOL_EXECUTION_FAILED: 'tool_execution_failed',  // 500
  AGENT_TIMEOUT: 'agent_timeout',                  // 504
} as const;

// 统一错误处理中间件
function errorHandler(err: Error, req: Request, res: Response) {
  if (err instanceof AgentApiError) {
    const body: Record<string, unknown> = {
      error: {
        code: err.code,
        message: err.message,
      },
    };

    if (err.details && process.env.NODE_ENV !== 'production') {
      body.error.details = err.details; // 只在开发环境暴露细节
    }

    if (err.retryAfterMs) {
      res.setHeader('Retry-After', Math.ceil(err.retryAfterMs / 1000));
      body.error.retryAfterMs = err.retryAfterMs;
    }

    return res.status(err.httpStatus).json(body);
  }

  // 未知错误
  console.error('Unexpected error:', err);
  res.status(500).json({
    error: {
      code: 'internal_error',
      message: 'An unexpected error occurred',
    },
  });
}
```

---

## 4. API 版本管理

Agent 能力迭代快，必须从一开始做版本管理：

```typescript
// 方式一：URL 前缀（推荐，简单直接）
app.use('/api/v1', v1Router);
app.use('/api/v2', v2Router);

// 方式二：请求头版本（对前端更友好）
function versionMiddleware(req: Request, res: Response, next: NextFunction) {
  const version = req.headers['x-api-version'] || 'v1';
  req.apiVersion = version;

  // 返回实际使用的版本
  res.setHeader('X-API-Version', version);
  next();
}

// 版本路由
router.post('/chat', versionMiddleware, async (req, res) => {
  if (req.apiVersion === 'v2') {
    return handleChatV2(req, res); // 支持多模态
  }
  return handleChatV1(req, res); // 纯文本
});

// 废弃通知
function deprecationMiddleware(endpoint: string, sunsetDate: string) {
  return (req: Request, res: Response, next: NextFunction) => {
    res.setHeader('Deprecation', 'true');
    res.setHeader('Sunset', sunsetDate);
    res.setHeader('Link', `</api/v2${req.path}>; rel="successor-version"`);
    next();
  };
}

// 在 v1 路由上加废弃头
v1Router.use('/chat', deprecationMiddleware('/chat', 'Sat, 31 Dec 2025 00:00:00 GMT'));
```

---

## 5. OpenClaw 实战：Agent API 完整配置

OpenClaw 本身就是一个 Agent 服务，它的 API 设计可以作为参考：

```typescript
// OpenClaw 风格的 Agent API 设计要点
// 参考 openclaw/packages/core/src/server/routes/

// 1. 工具调用进度实时推送
router.post('/run', async (req, res) => {
  res.sse(); // OpenClaw 内置 SSE 助手

  const runner = new AgentRunner({
    onToolStart: (tool, input) => {
      res.sendEvent('tool_start', { tool, input });
    },
    onToolEnd: (tool, result) => {
      res.sendEvent('tool_end', { tool, result });
    },
    onText: (delta) => {
      res.sendEvent('text', { delta });
    },
  });

  await runner.run(req.body.message);
  res.sendEvent('done', {});
  res.end();
});

// 2. 用 sessionKey 关联前后多个请求
// (OpenClaw 的 sessions_send 就是这个模式)
router.post('/sessions/:key/messages', async (req, res) => {
  const session = await getOrCreateSession(req.params.key);
  const result = await session.send(req.body.message);
  res.json(result);
});
```

---

## 6. API 设计的关键原则

```
✅ DO:
- 用 202 Accepted 而不是 200 OK 返回异步任务
- 在响应头里带 Retry-After（限流）和 Sunset（废弃）
- 暴露 jobId / sessionId，让客户端可以轮询/重连
- 对所有工具调用的执行过程用 SSE 实时推送（用户体验关键！）
- 错误码用 snake_case 字符串，不要用数字枚举
- 生产环境不暴露 stack trace，开发环境完整暴露

❌ DON'T:
- 不要让客户端等 30 秒才收到响应（先 202 再轮询）
- 不要在同步接口里跑 Agent（超时必死）
- 不要在错误里暴露内部实现（工具名、DB 结构）
- 不要在不同 endpoint 用不同的时间戳格式（统一 ISO 8601）
- 不要让 v1 和 v2 用同一个 session store 格式（数据迁移地狱）
```

---

## 总结

| 场景 | 推荐模式 |
|------|----------|
| 对话 UI | SSE 流式 |
| 批量任务 | 异步 + 轮询 |
| 后端集成 | 异步 + Webhook |
| 实时协作 | WebSocket（课程 152）|

Agent API 设计的核心是：**承认 Agent 是慢的、不确定的，在 API 层把这种异步性和不确定性封装好，不要泄漏给调用方。**

好的 API = 调用方不需要关心 Agent 内部用了几个工具、跑了多少 token。

---

**参考实现：**
- pi-mono: `packages/pi/src/server/` - 完整的 Agent HTTP 服务实现
- OpenClaw: `packages/core/src/server/routes/` - SSE + 会话管理参考
- learn-claude-code: `examples/api-server/` - 教学用 Express 实现
