# 84. Prompt Caching（提示词缓存）：用 cache_control 砍掉 90% 的重复 Token 费用

> **核心思想**：Agent 每次调用 LLM 都要把完整的 system prompt、工具列表、长文档重新发送一遍——但这些内容根本没变。Anthropic 的 `cache_control` 让你把这些静态内容"钉"在服务器端，后续调用直接复用，输入 Token 成本降低 90%，延迟降低 85%。

---

## 为什么需要 Prompt Caching？

一个典型的 Agent 调用：

```
System Prompt:    2000 tokens（角色定义、工具说明、规则）
工具列表:         3000 tokens（50个工具的 JSON Schema）
RAG 检索文档:     5000 tokens（相关上下文）
对话历史:         1000 tokens（最近几轮）
用户消息:           50 tokens

总计: 11050 tokens，其中 10000 tokens 是"重复的"
```

如果每秒 10 个并发请求，每小时就浪费了 3600 万个重复 Token。

**Prompt Caching 的本质**：把不变的部分打上标记，让 Anthropic 服务器缓存 KV (Key-Value) 计算结果，命中缓存时只收读取费（是写入费的 1/10）。

---

## cache_control 的工作原理

```
第一次请求（写入缓存）:
  [system: 2000 tokens] ← cache_control: {type: "ephemeral"}
  [tools: 3000 tokens]  ← cache_control: {type: "ephemeral"}
  [message: 100 tokens]
  
  服务器: 计算并缓存前 5000 tokens 的 KV 状态
  费用: 5100 tokens 按正常价 + 缓存写入费（1.25x）

第二次请求（缓存命中）:
  [system: 2000 tokens] ← 命中缓存！
  [tools: 3000 tokens]  ← 命中缓存！
  [message: 100 tokens] ← 正常处理
  
  费用: 5000 tokens 按缓存读取价（0.1x）+ 100 tokens 正常价
  节省: 约 82%
```

**关键规则**：
- 缓存生命周期：**5分钟**（每次命中后续约5分钟）
- 最小缓存块：Claude 3.5 Sonnet 需要 **1024 tokens** 以上才能缓存
- 缓存是**前缀匹配**：标记点之前的内容必须完全一致

---

## 实战：在 learn-claude-code 风格中使用

### 基础用法（Python）

```python
import anthropic

client = anthropic.Anthropic()

# 长系统提示（超过 1024 tokens 才值得缓存）
SYSTEM_PROMPT = """
你是一个专业的代码审查 Agent...
[省略 2000 tokens 的详细规则]
""" * 10  # 模拟长系统提示

# 工具列表（大型 Agent 可能有几十个工具）
TOOLS = [
    {"name": "read_file", "description": "读取文件内容", ...},
    {"name": "write_file", "description": "写入文件", ...},
    # ... 50个工具
]

def call_agent(user_message: str, conversation_history: list):
    response = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=1024,
        system=[
            {
                "type": "text",
                "text": SYSTEM_PROMPT,
                # ✅ 关键：在静态内容末尾打上缓存标记
                "cache_control": {"type": "ephemeral"}
            }
        ],
        # ✅ 工具列表也可以缓存
        tools=[
            {**tool, "cache_control": {"type": "ephemeral"}} 
            if i == len(TOOLS) - 1  # 只在最后一个工具上标记
            else tool
            for i, tool in enumerate(TOOLS)
        ],
        messages=[
            *conversation_history,  # 历史消息（动态，不缓存）
            {"role": "user", "content": user_message}  # 当前消息
        ]
    )
    
    # 查看缓存效果
    usage = response.usage
    print(f"输入 tokens: {usage.input_tokens}")
    print(f"缓存写入: {usage.cache_creation_input_tokens}")
    print(f"缓存命中: {usage.cache_read_input_tokens}")
    
    return response
```

### 带长文档的 RAG Agent（最佳缓存场景）

```python
def create_document_qa_agent(document: str, questions: list[str]):
    """
    对同一文档问多个问题时，文档只需缓存一次
    """
    
    # 构建带缓存标记的消息
    def build_messages(question: str):
        return [
            {
                "role": "user",
                "content": [
                    # 文档内容：静态，打上缓存标记
                    {
                        "type": "text",
                        "text": f"以下是需要分析的文档：\n\n{document}",
                        "cache_control": {"type": "ephemeral"}  # ← 文档缓存
                    },
                    # 问题：动态，不缓存
                    {
                        "type": "text", 
                        "text": f"\n\n请回答：{question}"
                    }
                ]
            }
        ]
    
    results = []
    total_cached = 0
    
    for i, question in enumerate(questions):
        response = client.messages.create(
            model="claude-opus-4-5",
            max_tokens=512,
            messages=build_messages(question)
        )
        
        usage = response.usage
        total_cached += usage.cache_read_input_tokens
        
        results.append({
            "question": question,
            "answer": response.content[0].text,
            "cache_hit": usage.cache_read_input_tokens > 0
        })
        
        print(f"Q{i+1}: 缓存命中 {usage.cache_read_input_tokens} tokens")
    
    print(f"\n总节省: {total_cached} tokens（约 {total_cached * 0.003 / 1000:.4f} 美元）")
    return results
```

---

## OpenClaw 中的实现

OpenClaw 在发送每个 LLM 请求时，自动对 system prompt 的静态部分和工具列表打缓存标记：

```typescript
// OpenClaw 内部简化版（conceptual）
class ClaudeProvider {
  private buildSystemBlocks(systemPrompt: string, skills: string[]): ContentBlock[] {
    const blocks: ContentBlock[] = []
    
    // 核心 system prompt → 缓存
    blocks.push({
      type: "text",
      text: systemPrompt,
      cache_control: { type: "ephemeral" }  // ← 固定内容缓存
    })
    
    // 加载的 Skill 内容 → 缓存
    for (const skillContent of skills) {
      blocks.push({
        type: "text", 
        text: skillContent,
        cache_control: { type: "ephemeral" }  // ← 技能内容缓存
      })
    }
    
    // 动态注入的上下文（用户信息、当前时间等）→ 不缓存
    // 这些每次都变，缓存无效
    
    return blocks
  }
  
  async chat(messages: Message[], context: AgentContext) {
    const systemBlocks = this.buildSystemBlocks(
      context.systemPrompt,
      context.loadedSkills
    )
    
    return await this.client.messages.create({
      model: this.model,
      system: systemBlocks,
      tools: this.buildToolsWithCache(context.availableTools),
      messages: messages
    })
  }
}
```

---

## pi-mono 中的缓存策略

```typescript
// pi-mono/src/providers/anthropic.ts 的缓存优化思路
export class AnthropicProvider {
  
  // 判断是否应该打缓存标记
  private shouldCache(content: string): boolean {
    // 只有超过 1024 tokens 才值得缓存
    // 粗略估算：1 token ≈ 4 字符
    return content.length > 4096
  }
  
  // 多轮对话中的缓存策略
  buildMessagesWithCache(
    history: Message[], 
    newMessage: string,
    maxCachePoints: number = 4  // Claude 最多支持 4 个缓存点
  ): APIMessage[] {
    const messages: APIMessage[] = []
    
    // 找出最近的几个较长的历史消息作为缓存点
    // 策略：对话历史的前 N-2 条打缓存（相对稳定）
    const stableHistory = history.slice(0, -2)  
    const recentHistory = history.slice(-2)     // 最近2条不缓存（可能被撤回/编辑）
    
    for (let i = 0; i < stableHistory.length; i++) {
      const msg = stableHistory[i]
      const isLastStable = i === stableHistory.length - 1
      
      messages.push({
        role: msg.role,
        content: isLastStable && this.shouldCache(msg.content) 
          ? [{ type: "text", text: msg.content, cache_control: { type: "ephemeral" } }]
          : msg.content
      })
    }
    
    // 最近消息正常添加
    messages.push(...recentHistory.map(m => ({ role: m.role, content: m.content })))
    messages.push({ role: "user", content: newMessage })
    
    return messages
  }
}
```

---

## 缓存标记的最佳位置

```
✅ 适合缓存（静态、长、重复使用）:
┌─────────────────────────────────────┐
│ System Prompt（角色/规则定义）      │ ← cache_control
├─────────────────────────────────────┤
│ 工具列表 JSON Schema               │ ← cache_control
├─────────────────────────────────────┤
│ RAG 检索文档（同一文档多次问答）    │ ← cache_control
├─────────────────────────────────────┤
│ 代码库（代码分析 Agent）            │ ← cache_control
└─────────────────────────────────────┘

❌ 不适合缓存（动态、变化频繁）:
┌─────────────────────────────────────┐
│ 用户当前消息                        │ 不缓存
├─────────────────────────────────────┤
│ 当前时间/日期                       │ 不缓存（每次都变）
├─────────────────────────────────────┤
│ 最近 2-3 条对话历史                 │ 不缓存（容易变）
└─────────────────────────────────────┘
```

---

## 成本计算示例

以 Claude claude-opus-4-5 为例（每百万 tokens 价格）：

| 类型 | 价格 |
|------|------|
| 输入（正常）| $15 |
| 缓存写入 | $18.75（1.25x）|
| 缓存读取 | $1.50（0.1x）|
| 输出 | $75 |

**场景**：10000 个工具 + 系统提示（5000 tokens），每次对话 5 轮

```
无缓存:
  5000 tokens × 50次调用 × $15/1M = $3.75

有缓存:
  5000 tokens × 1次写入 × $18.75/1M = $0.094  （第一次）
  5000 tokens × 49次读取 × $1.50/1M = $0.368  （后续）
  总计 = $0.462

节省: $3.29（节省 87.7%）
```

---

## 监控缓存命中率

```python
class CacheMetrics:
    def __init__(self):
        self.total_input = 0
        self.cache_writes = 0
        self.cache_reads = 0
    
    def record(self, usage: Usage):
        self.total_input += usage.input_tokens
        self.cache_writes += usage.cache_creation_input_tokens or 0
        self.cache_reads += usage.cache_read_input_tokens or 0
    
    @property
    def hit_rate(self) -> float:
        total = self.cache_writes + self.cache_reads
        return self.cache_reads / total if total > 0 else 0
    
    @property
    def savings_ratio(self) -> float:
        # 缓存读取比正常便宜 90%
        normal_cost = (self.total_input + self.cache_reads) * 1.0
        actual_cost = self.total_input * 1.0 + self.cache_writes * 1.25 + self.cache_reads * 0.1
        return 1 - (actual_cost / normal_cost) if normal_cost > 0 else 0
    
    def report(self):
        print(f"缓存命中率: {self.hit_rate:.1%}")
        print(f"预估节省: {self.savings_ratio:.1%}")
        print(f"  - 正常输入: {self.total_input:,} tokens")
        print(f"  - 缓存写入: {self.cache_writes:,} tokens")
        print(f"  - 缓存命中: {self.cache_reads:,} tokens")
```

---

## 常见陷阱

### 1. 缓存点位置错了
```python
# ❌ 错误：动态内容放在缓存标记之前
system = [
    {"type": "text", "text": f"当前时间: {datetime.now()}"},  # 每次都变！
    {"type": "text", "text": STATIC_RULES, "cache_control": {...}}  # 缓存失效
]

# ✅ 正确：静态内容在前，动态内容在后
system = [
    {"type": "text", "text": STATIC_RULES, "cache_control": {...}},  # 缓存命中
    {"type": "text", "text": f"当前时间: {datetime.now()}"},  # 动态，不缓存
]
```

### 2. 内容太短，不满足最小限制
```python
# ❌ 只有 100 tokens，无法缓存
{"type": "text", "text": "你是助手", "cache_control": {"type": "ephemeral"}}

# ✅ 确保超过 1024 tokens
# 如果 system prompt 太短，考虑合并多个静态部分
```

### 3. 以为工具缓存标记要打在每个工具上
```python
# ❌ 每个工具都打标记（浪费缓存点，Claude 最多支持4个）
tools = [
    {**tool, "cache_control": {...}} for tool in all_tools  # 错误！
]

# ✅ 只在工具列表的最后一个打标记
tools = [
    *all_tools[:-1],
    {**all_tools[-1], "cache_control": {"type": "ephemeral"}}  # 只标记最后一个
]
```

---

## 小结

| 场景 | 节省比例 | 适用性 |
|------|---------|--------|
| 长 System Prompt | 70-90% | ⭐⭐⭐⭐⭐ |
| 大型工具列表 | 60-85% | ⭐⭐⭐⭐⭐ |
| 长文档 QA | 85-95% | ⭐⭐⭐⭐⭐ |
| 代码库分析 | 80-95% | ⭐⭐⭐⭐⭐ |
| 短对话 | <20% | ⭐⭐ |

**一句话**：凡是"每次调用都发、但内容不变"的部分，都打上 `cache_control`。这是成本优化里投入产出比最高的一个技巧。

---

## 参考资源

- [Anthropic Prompt Caching 文档](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching)
- [learn-claude-code 示例](https://github.com/shareAI-lab/learn-claude-code)
- [pi-mono TypeScript 实现](https://github.com/badlogic/pi-mono)
