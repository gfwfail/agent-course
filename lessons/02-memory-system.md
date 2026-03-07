# 02 - Memory System：让 Agent 拥有长期记忆

## 为什么需要 Memory？

Agent 每次启动都是"失忆"的——它只能看到当前 session 的对话历史。但用户期望：

- "你记得我上周说过喜欢简洁风格吗？"
- "帮我继续上次的项目"
- "我之前让你记住的那个 API key"

没有 Memory System，Agent 就像一个每天都要重新介绍自己的同事。

## 记忆的三个层次

```
┌─────────────────────────────────────────────────────────┐
│                    Working Memory                        │
│         当前 session 的对话历史 (context window)          │
├─────────────────────────────────────────────────────────┤
│                    Short-term Memory                     │
│           最近几天的日志 (memory/YYYY-MM-DD.md)           │
├─────────────────────────────────────────────────────────┤
│                    Long-term Memory                      │
│              提炼的重要信息 (MEMORY.md)                   │
└─────────────────────────────────────────────────────────┘
```

## 实现方案 1: 文件系统

OpenClaw 的方案——简单直接：

```markdown
# workspace/MEMORY.md

## 用户偏好
- 喜欢简洁回复，不要废话
- 时区：Australia/Sydney
- 编程语言：主用 TypeScript/Rust

## 重要决定
- 2024-03-01: 决定用 fly.io 而不是 K8s
- 2024-03-05: API 风格统一用 REST

## 项目上下文
- mysterybox: Laravel 项目，皮肤盲盒平台
- clawclaw: OpenClaw 托管平台
```

```markdown
# workspace/memory/2024-03-08.md

## 今天干了什么
- 09:00 帮老板查了 AWS 账单
- 14:00 修了 mysterybox 的 bug
- 18:00 部署了新版本

## 学到的教训
- Cluster Autoscaler 需要 pending pods 才会扩容
```

### 关键代码（OpenClaw 风格）

```typescript
// 每次 session 开始，注入记忆
async function injectMemory(context: AgentContext) {
  const memoryPath = path.join(workspace, 'MEMORY.md');
  const todayPath = path.join(workspace, 'memory', `${today()}.md`);
  const yesterdayPath = path.join(workspace, 'memory', `${yesterday()}.md`);
  
  // 构建 system prompt
  let systemPrompt = basePrompt;
  
  if (await exists(memoryPath)) {
    systemPrompt += `\n## Long-term Memory\n${await read(memoryPath)}`;
  }
  
  // 最近两天的日志
  for (const path of [yesterdayPath, todayPath]) {
    if (await exists(path)) {
      systemPrompt += `\n## Recent: ${basename(path)}\n${await read(path)}`;
    }
  }
  
  return systemPrompt;
}
```

## 实现方案 2: 向量数据库

适合海量记忆的场景：

```typescript
// 存储记忆
async function remember(text: string, metadata: object) {
  const embedding = await embed(text);  // OpenAI/Voyage embeddings
  await vectorDB.upsert({
    id: uuid(),
    vector: embedding,
    metadata: { text, timestamp: Date.now(), ...metadata }
  });
}

// 回忆相关记忆
async function recall(query: string, topK = 5) {
  const embedding = await embed(query);
  const results = await vectorDB.query({
    vector: embedding,
    topK,
    includeMetadata: true
  });
  return results.map(r => r.metadata.text);
}
```

### 实际使用

```typescript
// 在 Agent 处理消息前，召回相关记忆
async function beforeMessage(message: string) {
  const memories = await recall(message);
  
  // 注入到 context
  return {
    role: 'system',
    content: `Relevant memories:\n${memories.join('\n')}`
  };
}
```

## 实现方案 3: 混合模式（推荐）

OpenClaw 的 `memory_search` 工具就是这种：

```typescript
// memory_search: 语义搜索 MEMORY.md + memory/*.md
interface MemorySearchTool {
  name: 'memory_search';
  parameters: {
    query: string;
    maxResults?: number;
    minScore?: number;
  };
}

// Agent 被训练成：回答关于过去的问题前，先搜索记忆
// "你还记得上周我们讨论的部署方案吗？"
// → Agent 调用 memory_search("部署方案 上周")
// → 返回相关片段
// → Agent 基于片段回答
```

## 记忆的写入时机

### 自动写入

```typescript
// 每次对话结束后
async function afterConversation(messages: Message[]) {
  const summary = await llm.summarize(messages);
  const todayPath = `memory/${today()}.md`;
  
  await appendFile(todayPath, `\n## ${time()}\n${summary}\n`);
}
```

### 显式写入

```typescript
// 用户说"记住这个"时
const REMEMBER_TRIGGERS = ['记住', 'remember', '记下来', '别忘了'];

if (REMEMBER_TRIGGERS.some(t => message.includes(t))) {
  // 让 Agent 决定写到哪里
  await agent.run(`用户让你记住: "${message}"
请决定：
1. 这是临时信息 → 写到 memory/${today()}.md
2. 这是长期偏好 → 更新 MEMORY.md
3. 这是项目相关 → 更新项目文档`);
}
```

## 记忆的整理（Memory Consolidation）

人类睡觉时会整理记忆，Agent 也需要：

```typescript
// 定期任务（比如每周一次）
async function consolidateMemory() {
  // 1. 读取最近 7 天的日志
  const recentLogs = await readRecentDays(7);
  
  // 2. 让 LLM 提炼重要信息
  const insights = await llm.extract(`
从这些日志中提炼：
1. 用户偏好变化
2. 重要决定
3. 项目进展
4. 学到的教训

日志内容：
${recentLogs}
  `);
  
  // 3. 更新 MEMORY.md
  await updateMemoryFile(insights);
  
  // 4. 可选：归档旧日志
  await archiveOldLogs(30); // 超过 30 天的压缩存档
}
```

## learn-claude-code 中的实现

```python
# claude_code/memory.py

class MemoryManager:
    def __init__(self, workspace: Path):
        self.workspace = workspace
        self.memory_file = workspace / "MEMORY.md"
        self.daily_dir = workspace / "memory"
    
    def get_context(self) -> str:
        """获取注入到 system prompt 的记忆"""
        parts = []
        
        # 长期记忆
        if self.memory_file.exists():
            parts.append(f"## Long-term Memory\n{self.memory_file.read_text()}")
        
        # 最近两天
        for offset in [1, 0]:  # yesterday, today
            date = (datetime.now() - timedelta(days=offset)).strftime("%Y-%m-%d")
            daily = self.daily_dir / f"{date}.md"
            if daily.exists():
                parts.append(f"## {date}\n{daily.read_text()}")
        
        return "\n\n".join(parts)
    
    def log_today(self, content: str):
        """写入今天的日志"""
        today = datetime.now().strftime("%Y-%m-%d")
        daily = self.daily_dir / f"{today}.md"
        
        self.daily_dir.mkdir(exist_ok=True)
        
        with daily.open("a") as f:
            f.write(f"\n## {datetime.now().strftime('%H:%M')}\n{content}\n")
```

## 安全考虑

记忆系统的安全问题：

```typescript
// 1. 敏感信息标记
interface Memory {
  content: string;
  sensitivity: 'public' | 'private' | 'secret';
  scope: 'main_session' | 'all_sessions';
}

// 2. 在共享场景下过滤
function getMemoryForContext(context: AgentContext) {
  if (context.isGroupChat) {
    // 群聊不暴露私人记忆
    return memories.filter(m => m.sensitivity === 'public');
  }
  return memories;
}

// 3. OpenClaw 的做法：MEMORY.md 只在 main session 加载
// 见 AGENTS.md:
// - **ONLY load in main session** (direct chats with your human)
// - **DO NOT load in shared contexts** (Discord, group chats)
```

## 实战技巧

### 1. 控制记忆大小

```typescript
const MAX_MEMORY_TOKENS = 4000;

function trimMemory(memory: string): string {
  const tokens = countTokens(memory);
  if (tokens <= MAX_MEMORY_TOKENS) return memory;
  
  // 策略：保留开头（偏好）+ 结尾（最新）
  const lines = memory.split('\n');
  const header = lines.slice(0, 20).join('\n');  // 重要配置
  const recent = lines.slice(-50).join('\n');     // 最近内容
  
  return `${header}\n\n[...truncated...]\n\n${recent}`;
}
```

### 2. 记忆冲突处理

```typescript
// 新信息和旧记忆冲突时
async function resolveConflict(oldMemory: string, newInfo: string) {
  return await llm.run(`
旧记忆: ${oldMemory}
新信息: ${newInfo}

用户偏好/决定可能会变化。请：
1. 如果新信息更新了旧记忆，返回更新后的版本
2. 如果是补充信息，合并两者
3. 如果是不同主题，两者都保留
  `);
}
```

### 3. 记忆索引

```typescript
// 为大量记忆建立索引
interface MemoryIndex {
  topics: Map<string, string[]>;  // topic -> memory IDs
  timeline: { date: string; ids: string[] }[];
  entities: Map<string, string[]>;  // person/project -> memory IDs
}

// 查询时先过滤
function queryMemory(query: string, index: MemoryIndex) {
  const topics = extractTopics(query);
  const relevantIds = topics.flatMap(t => index.topics.get(t) || []);
  return relevantIds.map(id => memories.get(id));
}
```

## 总结

| 方案 | 适用场景 | 优点 | 缺点 |
|------|----------|------|------|
| 文件系统 | 个人 Agent | 简单、可读、易编辑 | 搜索效率低 |
| 向量数据库 | 海量记忆 | 语义搜索强 | 复杂、成本高 |
| 混合模式 | 生产环境 | 平衡 | 需要维护两套 |

核心原则：
1. **层次分明**：Working → Short-term → Long-term
2. **按需召回**：不是所有记忆都要每次加载
3. **定期整理**：从日志提炼到长期记忆
4. **安全隔离**：敏感记忆不暴露给外部

下节课：**Error Recovery（错误恢复）**——Agent 出错时如何优雅处理。
