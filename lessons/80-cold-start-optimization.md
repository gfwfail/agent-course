# 80 - Agent 冷启动优化（Cold-Start Optimization）

> 用户发来第一条消息，等了 3 秒才开始回复——这就是冷启动问题。

---

## 什么是冷启动？

Agent 的"冷启动"指系统从 **完全空闲** 到 **处理第一个请求** 这段时间内的额外延迟。它通常比后续请求慢得多，原因包括：

| 冷启动来源 | 典型延迟 |
|-----------|---------|
| LLM API TCP 握手 + TLS | 50-200ms |
| HTTP/2 连接建立 | 100-300ms |
| 工具模块懒加载 | 50-500ms |
| 数据库连接池初始化 | 100-1000ms |
| 技能/知识库首次加载 | 100ms-几秒 |
| Node.js / Python 冷导入 | 100-2000ms |

**累加起来，第一个请求可能比后续请求慢 1-5 秒。**

---

## 核心解法一：HTTP 连接池复用

### 问题
每次请求新建 HTTP 连接 → TCP 握手 + TLS 握手 → 浪费 200ms+

### OpenClaw 的解法（Node.js + undici）

```typescript
// pi-mono/src/providers/anthropic/client.ts
import { Agent as UndiciAgent, setGlobalDispatcher } from 'undici'

// 进程启动时初始化全局连接池
const pool = new UndiciAgent({
  connections: 10,           // 最多 10 个并发连接
  pipelining: 1,             // HTTP/1.1 不 pipeline，HTTP/2 自动复用
  keepAliveTimeout: 60_000,  // 60s 保持连接
  keepAliveMaxTimeout: 300_000,
})

setGlobalDispatcher(pool)

// 之后的 fetch() 自动复用池中连接，无需重建握手
```

### Python 版本（httpx）

```python
# learn-claude-code/src/client.py
import httpx
import asyncio

# 全局客户端，进程级别复用
_client: httpx.AsyncClient | None = None

async def get_client() -> httpx.AsyncClient:
    global _client
    if _client is None or _client.is_closed:
        _client = httpx.AsyncClient(
            http2=True,          # HTTP/2 多路复用
            limits=httpx.Limits(
                max_connections=20,
                max_keepalive_connections=10,
                keepalive_expiry=60,
            ),
            timeout=httpx.Timeout(30.0, connect=5.0),
        )
    return _client

async def call_llm(prompt: str) -> str:
    client = await get_client()
    resp = await client.post(
        "https://api.anthropic.com/v1/messages",
        headers={"x-api-key": API_KEY, ...},
        json={...},
    )
    return resp.json()["content"][0]["text"]
```

**效果：首次请求后，后续请求节省 150-300ms。**

---

## 核心解法二：工具懒加载 → 预加载

### 问题
Agent 启动时不加载工具，第一次用到才初始化 → 首次工具调用慢。

### 错误的懒加载方式

```typescript
// ❌ 懒加载：第一次调用才初始化，用户感知到延迟
class FileReadTool {
  private db: Database | null = null

  async execute(path: string) {
    if (!this.db) {
      this.db = await Database.connect()  // 首次调用时才连接，慢！
    }
    return this.db.read(path)
  }
}
```

### 正确：启动时预热

```typescript
// ✅ 预热：进程启动时就初始化
class FileReadTool {
  private db: Database

  // 工厂方法，显式初始化
  static async create(): Promise<FileReadTool> {
    const tool = new FileReadTool()
    tool.db = await Database.connect()  // 启动时完成
    return tool
  }

  async execute(path: string) {
    return this.db.read(path)  // 直接用，无等待
  }
}

// 进程入口：并行初始化所有工具
async function bootstrap() {
  const [fileTool, searchTool, browserTool] = await Promise.all([
    FileReadTool.create(),
    SearchTool.create(),
    BrowserTool.create(),
  ])
  
  return new Agent({ tools: [fileTool, searchTool, browserTool] })
}
```

---

## 核心解法三：LLM 连接预热 Ping

有些 LLM provider 的连接在闲置后会被关闭（即使连接池还在）。解法：定期发"心跳"保活。

```typescript
// OpenClaw 启动时触发一次最小化请求来预热连接
async function warmupLLMConnection() {
  try {
    await anthropic.messages.create({
      model: "claude-haiku-3-5",  // 用最便宜的模型
      max_tokens: 1,
      messages: [{ role: "user", content: "." }],
    })
    console.log("[warmup] LLM connection ready")
  } catch (e) {
    // 预热失败不影响正常流程
    console.warn("[warmup] LLM warmup failed:", e.message)
  }
}

// 进程启动后立即执行，不等待
bootstrap().then(() => {
  warmupLLMConnection()  // fire-and-forget
  startServer()
})
```

**成本：1 个 1-token 请求 ≈ 0.0001 分钱。换来首个真实请求快 200ms。**

---

## 核心解法四：技能文件缓存

OpenClaw 的 Skills 系统每次需要时读取 SKILL.md 文件。如果技能很多，文件 I/O 累加不可忽视。

```typescript
// 内存缓存技能内容，带 TTL
const skillCache = new Map<string, { content: string; loadedAt: number }>()
const SKILL_CACHE_TTL = 5 * 60 * 1000  // 5 分钟

async function loadSkill(path: string): Promise<string> {
  const cached = skillCache.get(path)
  if (cached && Date.now() - cached.loadedAt < SKILL_CACHE_TTL) {
    return cached.content
  }

  const content = await fs.readFile(path, "utf8")
  skillCache.set(path, { content, loadedAt: Date.now() })
  return content
}

// 进程启动时预加载所有已知技能
async function preloadSkills(skillPaths: string[]) {
  await Promise.all(skillPaths.map(loadSkill))
  console.log(`[warmup] ${skillPaths.length} skills loaded`)
}
```

---

## 核心解法五：首 Token 时间（TTFT）优化

冷启动不只影响连接，还影响 LLM 首 token 到达时间。

```typescript
// 技巧：流式输出 + 立刻开始处理，不等完整响应
async function* streamWithEarlyFlush(prompt: string) {
  const stream = await anthropic.messages.stream({
    model: "claude-sonnet-4-5",
    max_tokens: 2000,
    messages: [{ role: "user", content: prompt }],
  })

  // 拿到第一个 token 就立刻输出，不等全部完成
  for await (const chunk of stream) {
    if (chunk.type === "content_block_delta") {
      yield chunk.delta.text  // 立刻 yield，前端立刻渲染
    }
  }
}

// 前端感知 TTFT 而不是完整响应时间
// 用户看到第一个字的时间：通常 300-800ms，远好于等待完整响应的 3-10s
```

---

## 监控冷启动延迟

```typescript
// 记录每个请求是"冷"还是"热"
let isFirstRequest = true

async function handleRequest(input: string) {
  const isCold = isFirstRequest
  isFirstRequest = false
  
  const start = Date.now()
  const result = await agent.run(input)
  const duration = Date.now() - start
  
  // 上报指标
  metrics.record("agent.request.duration", duration, {
    cold_start: isCold,
    model: "claude-sonnet-4-5",
  })
  
  if (isCold) {
    console.log(`[cold-start] first request took ${duration}ms`)
  }
  
  return result
}
```

---

## 实战检查清单

```
冷启动优化检查表：

✅ HTTP 连接池是否在进程启动时初始化？
✅ 数据库/Redis 连接是否预创建？
✅ 高频工具是否在 bootstrap 阶段预热？
✅ 技能文件是否有内存缓存？
✅ 是否开启了流式输出（优化 TTFT）？
✅ 是否在监控区分冷/热请求延迟？
✅ 闲置连接是否有 keepalive 保活？
```

---

## 小结

| 优化手段 | 节省延迟 | 复杂度 |
|---------|---------|--------|
| HTTP 连接池 | 150-300ms | 低 |
| 工具预加载 | 100-500ms | 低 |
| LLM 预热 Ping | 100-200ms | 低 |
| 技能文件缓存 | 10-50ms | 低 |
| 流式输出（TTFT） | 体感快 2-5x | 中 |

**最低成本的优化往往效果最显著。** 先把连接池和预加载做好，大多数冷启动问题就解决了 80%。

---

> 下一课预告：**Agent Observability Dashboard** — 把 Agent 的行为数据可视化，快速定位性能瓶颈与异常行为。
