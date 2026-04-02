# 199 - Agent 反模式与常见陷阱（Anti-Patterns & Common Pitfalls）

> 知道怎么做对很重要，但知道怎么避免做错更重要。这节课是一张避坑地图。

---

## 为什么要讲反模式？

经过 198 节课我们学了各种正确姿势。但真实项目里，开发者踩的坑几乎都是固定的那几类。本节把最常见的 **10 大 Agent 反模式** 系统化梳理，每个都配有：

- 🚩 **问题症状**：你会观察到什么
- 💥 **根本原因**：为什么会这样
- ✅ **正确做法**：参考哪节课

---

## 反模式 1：上帝工具（God Tool）

### 🚩 症状
一个工具做所有事：

```typescript
// ❌ 反模式：一个工具管天下
const godTool = {
  name: "do_everything",
  description: "可以查数据库、发邮件、写文件、调API...",
  parameters: {
    action: { type: "string", enum: ["query_db", "send_email", "write_file", "call_api"] },
    // 20个可选参数...
  }
}
```

### 💥 根本原因
开发者怕 LLM 选错工具，把所有功能塞进一个。结果适得其反：
- LLM 不知道该传哪些参数
- 工具内部逻辑复杂，出错难定位
- 无法针对单个功能做权限控制

### ✅ 正确做法
一个工具一个职责。参考 **#98 Plugin Architecture** 和 **#120 Tool Least Privilege**。

```typescript
// ✅ 正确：职责单一
const queryDbTool = { name: "query_database", ... }
const sendEmailTool = { name: "send_email", ... }
const writeFileTool = { name: "write_file", ... }
```

---

## 反模式 2：Context 垃圾填埋场（Context Landfill）

### 🚩 症状
```typescript
// ❌ 每次请求都把所有历史塞进去
const messages = [
  ...allConversationHistory,  // 500 轮对话
  ...allToolResults,          // 每个工具的完整输出
  ...allDocuments,            // 上传的所有文档
  userMessage
]
```

表现：
- 成本暴涨，响应变慢
- LLM "忘记"早期的重要指令
- 超出 context window 报错

### 💥 根本原因
"放进去肯定比不放强"——这个直觉是错的。Context 太长反而导致注意力稀释（attention dilution），关键信息被淹没。

### ✅ 正确做法
- **#33 Sliding Window + Dynamic Summarization**：滑动窗口
- **#86 Context Externalization**：把历史存到外部，按需检索
- **#130 Lazy Context Loading**：延迟加载，用到才拿
- **#66 Context Distillation**：蒸馏精华，丢弃噪声

```typescript
// ✅ 只保留最近 N 轮 + 关键摘要
const relevantHistory = await contextManager.getRelevant({
  recentTurns: 10,
  semanticQuery: userMessage,
  maxTokens: 2000
})
```

---

## 反模式 3：工具爆炸（Tool Explosion）

### 🚩 症状
注册了 50+ 个工具，全部传给 LLM。

```typescript
// ❌ 把工具仓库倾泻给 LLM
const tools = await toolRegistry.getAll() // 返回 67 个工具
const response = await llm.call({ tools, messages })
```

### 💥 根本原因
研究表明，工具数量超过 20 个，LLM 的工具选择准确率明显下降。工具列表本身也消耗大量 token。

### ✅ 正确做法
- **#99 NL2SQL & NL2API**：按意图动态选工具
- **#162 Tool Discovery & Capability Negotiation**：运行时发现
- **#169 User Profiling & Personalization**：按用户角色过滤

```typescript
// ✅ 按意图筛选相关工具（最多 10 个）
const relevantTools = await toolSelector.select({
  query: userMessage,
  maxTools: 10,
  userRole: session.userRole
})
```

---

## 反模式 4：同步阻塞（Sync Blocking）

### 🚩 症状
```typescript
// ❌ 在 async 代码里用同步阻塞操作
function executeTools(calls: ToolCall[]) {
  const results = []
  for (const call of calls) {
    // 每个工具串行等待，总时间 = 所有工具时间之和
    const result = callToolSync(call)  
    results.push(result)
  }
  return results
}
```

### 💥 根本原因
Agent 里有大量 I/O 操作（API 调用、数据库查询、文件读写）。串行执行是最大的性能杀手。

### ✅ 正确做法
- **#29 Concurrent Tool Execution**：并发执行独立工具
- **#132 Structured Concurrency**：结构化并发管理生命周期
- **#116 Async Tool Execution & Polling**：慢工具异步化

```typescript
// ✅ 并发执行，总时间 = 最慢工具的时间
const results = await Promise.all(
  toolCalls.map(call => executeTool(call))
)
```

---

## 反模式 5：无界重试（Unbounded Retry）

### 🚩 症状
```typescript
// ❌ 无限重试，没有退出条件
async function callTool(tool, params) {
  while (true) {
    try {
      return await tool.execute(params)
    } catch (e) {
      console.log("失败了，再试...")
      await sleep(1000)
    }
  }
}
```

### 💥 根本原因
工具失败有两种：**可重试**（网络抖动、限流）和**不可重试**（参数错误、权限不足）。不区分就无限重试，轻则浪费资源，重则打崩下游服务。

### ✅ 正确做法
- **#30 Rate Limiting & Backoff**：指数退避
- **#145 Intelligent Error Classification & Retry**：错误分类
- **#147 Timeout Budget Inheritance**：超时预算继承

```typescript
// ✅ 分类重试：可重试 vs 永久失败
const RETRYABLE_ERRORS = ['RATE_LIMIT', 'NETWORK_ERROR', 'TIMEOUT']
const MAX_RETRIES = 3

async function callWithRetry(tool, params) {
  for (let attempt = 0; attempt <= MAX_RETRIES; attempt++) {
    try {
      return await tool.execute(params)
    } catch (e) {
      if (!RETRYABLE_ERRORS.includes(e.code) || attempt === MAX_RETRIES) throw e
      await sleep(Math.min(1000 * 2 ** attempt, 30000)) // 指数退避，上限 30s
    }
  }
}
```

---

## 反模式 6：提示词意大利面（Prompt Spaghetti）

### 🚩 症状
```typescript
// ❌ 动态拼接，没有结构，难以维护
const systemPrompt = `你是一个助手。` +
  (isAdmin ? `你有管理员权限。` : ``) +
  (hasMemory ? `以下是历史：${memory}` : ``) +
  (tools.length > 0 ? `可用工具：${tools.map(t => t.name).join(',')}` : ``) +
  userPrefs.join('') +
  `今天是${new Date()}。`
```

### 💥 根本原因
随着功能增加，字符串拼接的 prompt 变得不可测试、不可版本控制、不可复用。

### ✅ 正确做法
- **#88 Prompt Template Engine**：模板引擎
- **#82 Dynamic System Prompt**：分层注入
- **#84 Prompt Caching**：静态部分缓存

```typescript
// ✅ 分层结构化组装
const systemPrompt = promptEngine.render('agent-base', {
  fragments: [
    'role-definition',
    hasMemory && fragment('memory-context', { memory }),
    isAdmin && 'admin-permissions',
    'tool-instructions',
    fragment('temporal-context', { now: new Date() })
  ].filter(Boolean),
  tokenBudget: 4000
})
```

---

## 反模式 7：静默吞异常（Silent Exception Swallowing）

### 🚩 症状
```typescript
// ❌ 工具失败了，但 LLM 什么都不知道
async function executeTool(call: ToolCall) {
  try {
    return await tools[call.name].execute(call.params)
  } catch (e) {
    console.error(e)
    return null  // 🚩 返回 null，LLM 以为成功了
  }
}
```

### 💥 根本原因
开发者怕工具报错影响用户体验，把错误吃掉。但 LLM 不知道工具失败，会继续基于"空结果"做出错误决策。

### ✅ 正确做法
- **#103 Tool Call Hallucination Detection**：结构化错误反馈
- **#102 Tool Result Transformation Pipeline**：错误包装
- **#150 Functional Error Handling (Result Pattern)**：显式错误类型

```typescript
// ✅ 错误也是 tool_result，LLM 可以感知和恢复
async function executeTool(call: ToolCall): Promise<ToolResult> {
  try {
    const data = await tools[call.name].execute(call.params)
    return { tool_use_id: call.id, type: "tool_result", content: JSON.stringify(data) }
  } catch (e) {
    return {
      tool_use_id: call.id,
      type: "tool_result",
      is_error: true,
      content: `Error: ${e.message}. Please try with different parameters or use an alternative approach.`
    }
  }
}
```

---

## 反模式 8：无状态假设（Stateless Assumption）

### 🚩 症状
```typescript
// ❌ 假设每次请求都是全新的，没有持久化状态
app.post('/agent', async (req, res) => {
  const response = await agent.run(req.body.message)
  res.json({ response })
  // 对话结束，状态消失...
})
```

用户下次说"继续做刚才那个任务"，Agent 完全不记得。

### 💥 根本原因
把 Agent 当做无状态 API 来设计。但真实的 Agent 任务往往跨越多轮对话、多次重启。

### ✅ 正确做法
- **#86 Context Externalization**：外部化状态
- **#32 State Persistence**：持久化检查点
- **#108 Health Check & Auto-Restart**：重启后恢复

```typescript
// ✅ 会话状态持久化
const session = await sessionStore.getOrCreate(req.sessionId)
const response = await agent.run({
  message: req.body.message,
  history: session.history,
  checkpoint: session.lastCheckpoint
})
await sessionStore.save(req.sessionId, { 
  history: session.history,
  lastCheckpoint: response.checkpoint
})
```

---

## 反模式 9：过度智能（Over-Engineering with LLM）

### 🚩 症状
```typescript
// ❌ 用 LLM 做一切决策，包括本不需要的
async function shouldRetry(error: Error): Promise<boolean> {
  const response = await llm.call({
    messages: [{ role: "user", content: `这个错误应该重试吗？${error.message}` }]
  })
  return response.includes("是")
}
```

### 💥 根本原因
"LLM 什么都会"——这个想法导致用 LLM 做规则性决策，浪费金钱和时间。

### ✅ 正确做法
- **#196 Dual-Process Decision Architecture**：快慢系统分工
- **#121 Confidence-Aware Routing**：置信度路由
- 规则性决策用代码，模糊性判断用 LLM

```typescript
// ✅ 规则性决策不用 LLM
function shouldRetry(error: Error): boolean {
  return ['RATE_LIMIT', 'NETWORK_ERROR'].includes(error.code)
}

// LLM 只用于真正需要理解语义的地方
async function shouldEscalateToHuman(context: AgentContext): Promise<boolean> {
  // 这里确实需要理解上下文，适合用 LLM
  const risk = await riskAssessor.evaluate(context)
  return risk.score > 0.8
}
```

---

## 反模式 10：裸奔部署（Naked Deployment）

### 🚩 症状
```bash
# ❌ 直接部署，没有任何可观测性
node agent.js

# 出问题了？
# - 不知道哪个工具调用失败了
# - 不知道用了多少 token
# - 不知道响应时间分布
# - 不知道错误率
```

### 💥 根本原因
Agent 是"黑盒"，出问题靠猜。

### ✅ 正确做法
- **#88 OpenTelemetry 分布式追踪**
- **#107 Structured Logging & Trace Correlation**
- **#123 Real-time Monitoring & Alerting**
- **#197 Tool Call Chain Visualization**

```typescript
// ✅ 每次工具调用自动追踪
const tracedTools = tools.map(tool => 
  withTracing(tool, { 
    metrics: ['latency', 'errors', 'tokenUsage'],
    alertOn: { errorRate: 0.05, p99Latency: 5000 }
  })
)
```

---

## 快速检查清单

在生产环境部署 Agent 之前，对照检查：

```
□ 工具职责单一（无上帝工具）
□ Context 有上限控制（不超 X tokens）
□ 工具数量动态筛选（< 20 个）
□ I/O 操作全部 async（无同步阻塞）
□ 重试有上限 + 错误分类（非无限重试）
□ Prompt 有模板管理（非字符串拼接）
□ 工具失败有结构化错误（不吃异常）
□ 会话状态持久化（不假设无状态）
□ 规则性决策不用 LLM（不过度依赖）
□ 部署有监控 + 追踪（不裸奔）
```

---

## OpenClaw 实战：反模式检测中间件

把以上规则编码为运行时检测：

```typescript
// anti-pattern-detector.ts
export function createAntiPatternDetector() {
  return {
    // 检测1：工具数量过多
    checkToolCount(tools: Tool[]) {
      if (tools.length > 20) {
        console.warn(`⚠️ 工具过多 (${tools.length})，建议按意图过滤到 <20 个`)
      }
    },

    // 检测2：Context token 过多
    checkContextSize(messages: Message[], limit = 50000) {
      const tokens = estimateTokens(messages)
      if (tokens > limit) {
        console.warn(`⚠️ Context 过大 (${tokens} tokens)，超过上限 ${limit}`)
      }
    },

    // 检测3：工具返回 null（可能吃了异常）
    checkToolResult(result: ToolResult) {
      if (result.content === null || result.content === undefined) {
        console.warn(`⚠️ 工具 ${result.tool_use_id} 返回 null，检查是否吃了异常`)
      }
    },

    // 检测4：重试次数异常
    checkRetryCount(toolName: string, retries: number) {
      if (retries > 3) {
        console.warn(`⚠️ 工具 ${toolName} 已重试 ${retries} 次，检查错误分类逻辑`)
      }
    }
  }
}

// 集成到 Agent Loop
const detector = createAntiPatternDetector()

async function agentLoop(userMessage: string) {
  const tools = await getTools(userMessage)
  detector.checkToolCount(tools)  // ← 检测工具数量
  
  const messages = await buildContext(userMessage)
  detector.checkContextSize(messages)  // ← 检测 context 大小

  // ... 正常 agent loop
}
```

---

## 总结

| 反模式 | 核心症状 | 一句话解法 |
|--------|----------|-----------|
| 上帝工具 | 一个工具做所有事 | 一个工具一个职责 |
| Context 垃圾填埋场 | 所有历史塞进去 | 滑动窗口 + 语义检索 |
| 工具爆炸 | 50+ 工具全给 LLM | 按意图动态筛选 |
| 同步阻塞 | 串行等待所有工具 | Promise.all 并发 |
| 无界重试 | 失败就无限重试 | 分类 + 上限 + 退避 |
| Prompt 意大利面 | 字符串拼接 prompt | 模板引擎分层组装 |
| 静默吞异常 | 工具失败返回 null | 结构化错误反馈 |
| 无状态假设 | 每次从零开始 | 会话状态持久化 |
| 过度智能 | 什么都用 LLM | 规则归规则，LLM 归 LLM |
| 裸奔部署 | 没有监控追踪 | OpenTelemetry + 告警 |

> **一个 Agent 的成熟度，往往不体现在它能做多少，而体现在它能避开多少坑。**

---

*本节课是第 199 讲，建议配合 [#145 智能错误分类](145-intelligent-error-classification-retry.md)、[#123 实时监控](123-realtime-monitoring-alerting.md)、[#191 函数式核心](191-functional-core-imperative-shell.md) 一起食用。*
