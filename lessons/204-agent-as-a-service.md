# Lesson 204: Agent 作为 REST API 服务（Agent-as-a-Service）

> 把 Agent 包装成标准 HTTP 服务，让任何应用都能调用 AI 能力

---

## 为什么需要 Agent-as-a-Service？

你写了一个超厉害的 Agent，现在问题来了：

- 前端同事想调用它，但他不懂 Python/TypeScript Agent SDK
- 移动 App 需要调用，但 Agent 只能在服务器端跑
- 多个业务系统都想用，但不可能每个都内嵌一份 Agent 代码

**解决方案**：把 Agent 包装成一个标准的 REST API 服务。

```
Client App ──HTTP──▶ Agent Gateway ──▶ Agent Loop ──▶ LLM + Tools
                         │
                    身份验证、限流
                    请求队列、日志
```

---

## 两种交互模式

Agent 的执行可能很慢（几秒到几分钟），所以要支持两种模式：

### 模式一：同步模式（适合快速任务，< 30s）

```
POST /agent/run
{ "input": "查一下最近的订单状态" }

──等待──

200 OK
{ "output": "最近3笔订单都已发货...", "duration_ms": 2300 }
```

### 模式二：异步模式（适合长任务）

```
POST /agent/jobs
{ "input": "分析本月所有用户的行为数据" }

202 Accepted
{ "job_id": "job_abc123", "status": "pending" }

──轮询──

GET /agent/jobs/job_abc123
{ "job_id": "job_abc123", "status": "completed", "output": "..." }
```

---

## TypeScript 实现（基于 Express）

### 核心架构

```typescript
// agent-service/src/server.ts
import express from 'express'
import { randomUUID } from 'crypto'
import { runAgent } from './agent'
import { JobStore } from './job-store'
import { authMiddleware, rateLimitMiddleware } from './middleware'

const app = express()
app.use(express.json())

// 全局中间件
app.use(authMiddleware)
app.use(rateLimitMiddleware)

const jobs = new JobStore()

// ─── 同步接口 ─────────────────────────────────────────
app.post('/agent/run', async (req, res) => {
  const { input, session_id, context } = req.body
  
  if (!input) return res.status(400).json({ error: 'input is required' })
  
  // 超时保护：30s 后自动返回 timeout
  const controller = new AbortController()
  const timeout = setTimeout(() => controller.abort(), 30_000)
  
  try {
    const start = Date.now()
    const output = await runAgent(input, {
      sessionId: session_id ?? randomUUID(),
      context,
      signal: controller.signal,
    })
    
    res.json({
      output,
      duration_ms: Date.now() - start,
    })
  } catch (err: any) {
    if (err.name === 'AbortError') {
      res.status(408).json({ error: 'Request timeout. Use /agent/jobs for long-running tasks.' })
    } else {
      res.status(500).json({ error: err.message })
    }
  } finally {
    clearTimeout(timeout)
  }
})

// ─── 异步接口 ─────────────────────────────────────────
app.post('/agent/jobs', async (req, res) => {
  const { input, session_id, webhook_url } = req.body
  if (!input) return res.status(400).json({ error: 'input is required' })
  
  const jobId = randomUUID()
  const sessionId = session_id ?? randomUUID()
  
  // 立即返回 job_id，后台运行
  jobs.create(jobId, { input, sessionId, status: 'pending' })
  
  res.status(202).json({ job_id: jobId, status: 'pending' })
  
  // 后台执行（不 await）
  runAgentJob(jobId, input, sessionId, webhook_url)
})

app.get('/agent/jobs/:jobId', (req, res) => {
  const job = jobs.get(req.params.jobId)
  if (!job) return res.status(404).json({ error: 'Job not found' })
  res.json(job)
})

// 列出 job 历史（支持分页）
app.get('/agent/jobs', (req, res) => {
  const { limit = 20, cursor } = req.query
  res.json(jobs.list({ limit: Number(limit), cursor: cursor as string }))
})

async function runAgentJob(
  jobId: string,
  input: string,
  sessionId: string,
  webhookUrl?: string
) {
  jobs.update(jobId, { status: 'running', started_at: new Date().toISOString() })
  
  try {
    const output = await runAgent(input, { sessionId })
    jobs.update(jobId, {
      status: 'completed',
      output,
      completed_at: new Date().toISOString(),
    })
    
    // 发 Webhook 通知
    if (webhookUrl) await notifyWebhook(webhookUrl, { job_id: jobId, status: 'completed', output })
    
  } catch (err: any) {
    jobs.update(jobId, {
      status: 'failed',
      error: err.message,
      completed_at: new Date().toISOString(),
    })
    if (webhookUrl) await notifyWebhook(webhookUrl, { job_id: jobId, status: 'failed', error: err.message })
  }
}

app.listen(3000, () => console.log('Agent service running on :3000'))
```

### 身份验证中间件

```typescript
// middleware/auth.ts
import { Request, Response, NextFunction } from 'express'

// API Key 验证（生产环境可换成 JWT）
export function authMiddleware(req: Request, res: Response, next: NextFunction) {
  // 健康检查不需要认证
  if (req.path === '/health') return next()
  
  const apiKey = req.headers['x-api-key'] as string
  if (!apiKey) return res.status(401).json({ error: 'Missing X-Api-Key header' })
  
  const validKey = process.env.AGENT_API_KEY
  if (apiKey !== validKey) return res.status(403).json({ error: 'Invalid API key' })
  
  // 把 API key 对应的用户信息挂到 req 上（便于限流、审计）
  ;(req as any).clientId = deriveClientId(apiKey)
  next()
}

function deriveClientId(apiKey: string): string {
  // 实际项目里从数据库查 clientId
  return apiKey.slice(0, 8)
}
```

### 限流中间件

```typescript
// middleware/rate-limit.ts
import { Request, Response, NextFunction } from 'express'

interface RateLimitState {
  count: number
  resetAt: number
}

const limits = new Map<string, RateLimitState>()
const WINDOW_MS = 60_000  // 1 分钟
const MAX_REQUESTS = 20   // 每分钟最多 20 次

export function rateLimitMiddleware(req: Request, res: Response, next: NextFunction) {
  const clientId = (req as any).clientId ?? req.ip
  const now = Date.now()
  
  let state = limits.get(clientId)
  if (!state || state.resetAt < now) {
    state = { count: 0, resetAt: now + WINDOW_MS }
    limits.set(clientId, state)
  }
  
  state.count++
  
  // 设置 Rate Limit 响应头（让客户端知道余量）
  res.setHeader('X-RateLimit-Limit', MAX_REQUESTS)
  res.setHeader('X-RateLimit-Remaining', Math.max(0, MAX_REQUESTS - state.count))
  res.setHeader('X-RateLimit-Reset', Math.ceil(state.resetAt / 1000))
  
  if (state.count > MAX_REQUESTS) {
    res.setHeader('Retry-After', Math.ceil((state.resetAt - now) / 1000))
    return res.status(429).json({ error: 'Rate limit exceeded', retry_after: Math.ceil((state.resetAt - now) / 1000) })
  }
  
  next()
}
```

### Job Store（带 TTL 清理）

```typescript
// job-store.ts
interface Job {
  job_id: string
  status: 'pending' | 'running' | 'completed' | 'failed'
  input: string
  output?: string
  error?: string
  created_at: string
  started_at?: string
  completed_at?: string
}

export class JobStore {
  private jobs = new Map<string, Job>()
  private readonly TTL_MS = 24 * 60 * 60 * 1000  // 24小时后清理
  
  create(jobId: string, data: Partial<Job>): Job {
    const job: Job = {
      job_id: jobId,
      status: 'pending',
      input: '',
      created_at: new Date().toISOString(),
      ...data,
    }
    this.jobs.set(jobId, job)
    // TTL 到期后自动删除
    setTimeout(() => this.jobs.delete(jobId), this.TTL_MS)
    return job
  }
  
  get(jobId: string): Job | undefined {
    return this.jobs.get(jobId)
  }
  
  update(jobId: string, patch: Partial<Job>): void {
    const job = this.jobs.get(jobId)
    if (job) Object.assign(job, patch)
  }
  
  list({ limit, cursor }: { limit: number; cursor?: string }): { jobs: Job[]; next_cursor?: string } {
    const all = [...this.jobs.values()].sort(
      (a, b) => new Date(b.created_at).getTime() - new Date(a.created_at).getTime()
    )
    const startIdx = cursor ? all.findIndex(j => j.job_id === cursor) + 1 : 0
    const slice = all.slice(startIdx, startIdx + limit)
    return {
      jobs: slice,
      next_cursor: slice.length === limit ? slice[slice.length - 1].job_id : undefined,
    }
  }
}
```

---

## OpenClaw 实战：把 OpenClaw Agent 暴露为 API

OpenClaw 内置了 `sessions_spawn` 机制，完全可以包装成 API：

```typescript
// openclaw-agent-service.ts
// 用 OpenClaw 的 subagent 运行机制

app.post('/agent/jobs', async (req, res) => {
  const { input } = req.body
  const jobId = randomUUID()
  
  res.status(202).json({ job_id: jobId })
  
  // 通过 OpenClaw sessions_spawn 在隔离 session 中运行
  // 等价于在 OpenClaw 里触发一个 isolated agentTurn job
  // 结果通过 webhook 回调
  const result = await fetch('http://localhost:YOUR_PORT/api/sessions/spawn', {
    method: 'POST',
    body: JSON.stringify({
      task: input,
      mode: 'run',
      runtime: 'subagent',
      delivery: {
        mode: 'webhook',
        to: `https://your-service.com/webhooks/agent-result?job_id=${jobId}`
      }
    })
  })
})
```

---

## Webhook 回调设计

```typescript
// 客户端无需轮询，任务完成后主动推送
async function notifyWebhook(url: string, payload: object) {
  try {
    const body = JSON.stringify(payload)
    const signature = computeHmac(body, process.env.WEBHOOK_SECRET!)
    
    await fetch(url, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'X-Agent-Signature': signature,  // 让客户端验签，防伪造
        'X-Agent-Timestamp': Date.now().toString(),
      },
      body,
    })
  } catch (err) {
    console.error('Webhook delivery failed:', err)
    // 可以加重试队列
  }
}

function computeHmac(body: string, secret: string): string {
  const { createHmac } = require('crypto')
  return createHmac('sha256', secret).update(body).digest('hex')
}
```

客户端验签：
```typescript
// 客户端收到 Webhook 后验证签名
function verifyWebhookSignature(body: string, signature: string, secret: string): boolean {
  const expected = createHmac('sha256', secret).update(body).digest('hex')
  return timingSafeEqual(Buffer.from(signature), Buffer.from(expected))
}
```

---

## 与其他模式的区别

| 模式 | 适用场景 | 调用方 |
|------|---------|--------|
| **Agent-as-a-Service（本课）** | 任何 HTTP 客户端调用 Agent | 前端、移动端、其他服务 |
| **MCP Server（Lesson 192）** | LLM 调用你的工具 | Claude / GPT 等 LLM |
| **Webhook-Driven（Lesson 89）** | 外部事件触发 Agent | 第三方平台（GitHub、Stripe 等）|
| **Subagents（Lesson 33）** | Agent 内部任务分发 | 父 Agent |

---

## 生产部署检查清单

```
✅ API 身份验证（API Key / JWT）
✅ 限流（Rate Limiting）防滥用
✅ 同步超时保护（< 30s 用同步，超时提示用异步）
✅ 异步 Job 状态轮询接口
✅ Webhook 回调 + HMAC 签名验证
✅ Job TTL 自动清理（防内存泄漏）
✅ 健康检查接口 GET /health
✅ OpenAPI 文档（Swagger）
✅ 结构化日志（含 job_id / client_id）
✅ 错误响应格式统一 { error: string }
```

---

## 关键要点

1. **快任务用同步，慢任务用异步** — 30s 是经验分界线，超过就要返回 job_id 让客户端轮询
2. **Webhook 比轮询友好** — 给客户端提供 `webhook_url` 参数，任务完成主动推送，减少无效请求
3. **签名验证是必须的** — Webhook 回调必须带 HMAC 签名，防止第三方伪造回调
4. **与 MCP Server 不同** — MCP 是 LLM 调用工具的协议；AaaS 是人类开发者调用 Agent 的 HTTP 接口
5. **Job Store 加 TTL** — 不要让 Job 记录永远堆积，24 小时后清理是合理的默认值

---

*下一课预告：Agent 对话树剪枝与上下文复用（Conversation Tree Pruning & Context Reuse）*
