# 68 - Sliding Window + Dynamic Summarization：长对话的生存法则

> 对话越长，Token 越贵，LLM 越笨。滑动窗口 + 动态摘要，让 Agent 在超长会话里保持清醒。

---

## 问题：长对话的 Token 爆炸

Claude Sonnet 4 的 context window 是 200K tokens，听起来很大——但想象一下：

```
用户和 Agent 聊了 3 小时
每轮对话平均 500 tokens
100 轮 × 500 = 50,000 tokens

加上工具调用结果（每次 1000+ tokens）
50 轮工具调用 × 1000 = 50,000 tokens

总计：100,000 tokens/次请求
成本：~$0.30/次，每天几百次 = $90/天
```

更糟的是：**LLM 处理超长 context 时，对早期内容的注意力会下降**（Lost in the Middle 问题）。你的历史信息存在，但 LLM 看不见。

解法：**滑动窗口 + 动态摘要**

---

## 核心思路

```
完整历史                    滑动窗口策略
─────────────────────      ──────────────────────────────
[消息 1 - 3小时前]          ┌─────────────────────────┐
[消息 2 - 3小时前]          │  📝 会话摘要 (压缩历史)   │
[消息 3 - 2小时前]  ──→     │  "用户在调试一个支付bug,  │
[消息 4 - 2小时前]          │   已确认是 timeout 问题"  │
[消息 5 - 1小时前]          ├─────────────────────────┤
...                         │  最近 N 条消息 (原文)     │
[消息 98 - 5分钟前]         │  消息 95, 96, 97, 98...  │
[消息 99 - 2分钟前]         │  消息 99 (最新)           │
[消息 100 - 刚刚]  ──────→  │  消息 100 (当前)          │
                            └─────────────────────────┘
发送给 LLM 的 tokens：5,000（而不是 100,000）
```

---

## 实现：learn-claude-code 风格（Python）

```python
from dataclasses import dataclass, field
from typing import Optional
import anthropic
import json

@dataclass
class Message:
    role: str  # "user" | "assistant"
    content: str
    token_count: int = 0

@dataclass
class SlidingWindowMemory:
    """滑动窗口 + 动态摘要的对话记忆"""
    
    # 保留最近 N 条消息的原文
    window_size: int = 20
    
    # 触发摘要压缩的 token 阈值
    summary_threshold: int = 8000
    
    # 当前摘要
    summary: Optional[str] = None
    
    # 滑动窗口（最近的消息）
    window: list[Message] = field(default_factory=list)
    
    # 完整历史（用于审计，不发给 LLM）
    full_history: list[Message] = field(default_factory=list)
    
    def add_message(self, role: str, content: str):
        msg = Message(role=role, content=content)
        msg.token_count = len(content) // 4  # 粗略估算
        
        self.full_history.append(msg)
        self.window.append(msg)
        
        # 窗口超限，触发压缩
        if len(self.window) > self.window_size:
            self._compress_oldest()
    
    def _compress_oldest(self):
        """把最旧的消息压缩进摘要"""
        # 取出窗口中最旧的一半
        to_compress = self.window[:self.window_size // 2]
        self.window = self.window[self.window_size // 2:]
        
        # 将这些消息追加到摘要里（异步更新，详见下方）
        self._pending_compression = to_compress
    
    def build_messages_for_llm(self) -> list[dict]:
        """构建发送给 LLM 的消息列表"""
        messages = []
        
        # 如果有摘要，作为系统消息注入
        # （实际上以 user/assistant 对的形式插入，避免污染 system prompt）
        if self.summary:
            messages.append({
                "role": "user",
                "content": f"[对话历史摘要]\n{self.summary}\n[以上是之前的对话摘要，接下来是最近的对话]"
            })
            messages.append({
                "role": "assistant", 
                "content": "好的，我已了解之前的对话背景。"
            })
        
        # 追加滑动窗口内的原始消息
        for msg in self.window:
            messages.append({"role": msg.role, "content": msg.content})
        
        return messages
    
    def total_tokens(self) -> int:
        return sum(m.token_count for m in self.window) + (
            len(self.summary) // 4 if self.summary else 0
        )
```

---

## 摘要生成：异步 + 增量

关键技巧：**摘要不能阻塞主对话**，要在后台增量更新。

```python
import asyncio
from anthropic import AsyncAnthropic

client = AsyncAnthropic()

async def update_summary(
    existing_summary: Optional[str],
    new_messages: list[Message]
) -> str:
    """增量更新摘要：把新消息合并进已有摘要"""
    
    messages_text = "\n".join([
        f"{m.role.upper()}: {m.content}" 
        for m in new_messages
    ])
    
    if existing_summary:
        prompt = f"""你是一个对话摘要助手。

已有摘要：
{existing_summary}

新增对话：
{messages_text}

请将新增对话合并进已有摘要，保持简洁（200字以内）。
重点保留：用户的核心需求、已达成的结论、待处理的问题。
直接输出新摘要，不要加前缀。"""
    else:
        prompt = f"""请将以下对话总结为简洁摘要（200字以内）。
重点保留：用户的核心需求、已达成的结论、待处理的问题。

对话：
{messages_text}

直接输出摘要，不要加前缀。"""
    
    response = await client.messages.create(
        model="claude-haiku-4-5",  # 用便宜的模型做摘要！
        max_tokens=500,
        messages=[{"role": "user", "content": prompt}]
    )
    
    return response.content[0].text


class AgentWithSlidingWindow:
    def __init__(self):
        self.client = AsyncAnthropic()
        self.memory = SlidingWindowMemory()
        self._summary_task: Optional[asyncio.Task] = None
    
    async def chat(self, user_input: str) -> str:
        # 1. 加入用户消息
        self.memory.add_message("user", user_input)
        
        # 2. 如果有待压缩的消息，异步更新摘要
        if hasattr(self.memory, '_pending_compression'):
            msgs_to_compress = self.memory._pending_compression
            del self.memory._pending_compression
            
            # 后台更新摘要，不阻塞当前响应
            self._summary_task = asyncio.create_task(
                self._update_summary_background(msgs_to_compress)
            )
        
        # 3. 构建发送给 LLM 的消息
        messages = self.memory.build_messages_for_llm()
        
        # 4. 调用 LLM
        response = await self.client.messages.create(
            model="claude-sonnet-4-5",
            max_tokens=2000,
            system="你是一个智能助理。",
            messages=messages
        )
        
        reply = response.content[0].text
        
        # 5. 记录 Assistant 回复
        self.memory.add_message("assistant", reply)
        
        print(f"[窗口大小: {len(self.memory.window)} 条 | "
              f"Token 估算: {self.memory.total_tokens()}]")
        
        return reply
    
    async def _update_summary_background(self, msgs: list[Message]):
        """后台更新摘要"""
        try:
            self.memory.summary = await update_summary(
                self.memory.summary, msgs
            )
            print(f"[摘要已更新: {self.memory.summary[:50]}...]")
        except Exception as e:
            print(f"[摘要更新失败: {e}]")  # 失败不影响主流程
```

---

## OpenClaw 的实现：看看生产代码怎么做

OpenClaw 自身也面临同样问题。在 OpenClaw 中，context compact 机制使用类似策略：

```typescript
// OpenClaw 风格的 context compact 触发逻辑（简化版）
interface CompactConfig {
  triggerThreshold: number;  // 触发压缩的 token 比例（如 0.8 = 80% 满时）
  targetRatio: number;       // 压缩到多少比例（如 0.3 = 保留 30%）
  summaryModel: string;      // 用于摘要的模型
}

class ContextManager {
  private messages: Message[] = [];
  private compactSummary?: string;
  
  shouldCompact(maxTokens: number, config: CompactConfig): boolean {
    const currentTokens = this.estimateTokens();
    return currentTokens / maxTokens > config.triggerThreshold;
  }
  
  async compact(config: CompactConfig): Promise<void> {
    const keepCount = Math.floor(
      this.messages.length * config.targetRatio
    );
    
    // 保留最近的消息
    const toSummarize = this.messages.slice(0, -keepCount);
    const toKeep = this.messages.slice(-keepCount);
    
    // 生成摘要（用便宜的 haiku 模型）
    const summary = await generateSummary(toSummarize, config.summaryModel);
    
    // 摘要注入为首条消息
    this.compactSummary = summary;
    this.messages = toKeep;
    
    console.log(`[Compact] ${toSummarize.length} msgs → summary`);
  }
  
  buildContext(): Message[] {
    if (!this.compactSummary) return this.messages;
    
    return [
      {
        role: "user",
        content: `<conversation_summary>\n${this.compactSummary}\n</conversation_summary>`
      },
      {
        role: "assistant",
        content: "Understood. I have context from the summarized conversation."
      },
      ...this.messages
    ];
  }
}
```

---

## 三种压缩策略对比

```
策略           适用场景              优点              缺点
─────────────────────────────────────────────────────────────
1. 截断        简单场景              实现最简单         丢失早期上下文
   (仅保留最近N条)

2. 滑动窗口    大多数对话 Agent      平衡性好           摘要有延迟
   + 摘要      (本课内容)

3. 分层存储    长期运行的 Agent      最完整             实现复杂
   (热/温/冷)  (如 24小时助理)       不丢失任何信息     成本最高
```

---

## 分层存储：更进阶的方案

```python
class LayeredMemory:
    """三层记忆架构"""
    
    def __init__(self):
        # 热层：最近 10 条，完整原文（~2K tokens）
        self.hot: deque[Message] = deque(maxlen=10)
        
        # 温层：最近 50 条的摘要（~1K tokens）
        self.warm_summary: str = ""
        
        # 冷层：向量数据库，语义检索（按需加载）
        self.cold_store = VectorStore()
    
    async def retrieve_relevant(self, query: str) -> list[str]:
        """从冷层语义检索相关历史"""
        # 只在需要时才查冷层，不影响正常对话
        return await self.cold_store.search(query, top_k=3)
    
    def build_context(self, query: str) -> list[dict]:
        context = []
        
        # 1. 温层摘要（始终包含）
        if self.warm_summary:
            context.append({"role": "user", 
                           "content": f"[历史摘要] {self.warm_summary}"})
            context.append({"role": "assistant", "content": "了解。"})
        
        # 2. 热层原文（始终包含）
        context.extend([{"role": m.role, "content": m.content} 
                        for m in self.hot])
        
        return context
```

---

## 最佳实践

**1. 摘要用便宜模型**
```python
# ✅ 对：用 haiku 做摘要，省 10 倍成本
summary = await summarize(messages, model="claude-haiku-4-5")

# ❌ 错：用主模型做摘要
summary = await summarize(messages, model="claude-sonnet-4-5")
```

**2. 摘要异步，不阻塞**
```python
# ✅ 对：后台更新
asyncio.create_task(update_summary(...))

# ❌ 错：同步等待摘要完成再回复用户
await update_summary(...)
reply = await llm(...)
```

**3. 保留摘要 + 原文的分界线**
```python
# ✅ 明确标注上下文边界，LLM 更容易理解
content = """[对话历史摘要 - 截至 10 分钟前]
用户在排查支付超时问题，已确认是数据库连接池耗尽。

[最近对话原文]"""
```

**4. 摘要失败时优雅降级**
```python
try:
    summary = await update_summary(msgs)
except Exception:
    # 摘要失败 = 退化为普通截断，不崩溃
    logger.warning("Summary failed, falling back to truncation")
    self.window = self.window[-self.window_size:]
```

---

## 总结

| | 普通 Agent | 滑动窗口 Agent |
|---|---|---|
| 第 100 轮对话 token | ~100K | ~5K |
| 成本 | $0.30/次 | $0.015/次 |
| LLM 注意力质量 | 退化 | 保持稳定 |
| 实现复杂度 | 低 | 中 |

**核心公式：** 长对话 Agent = 滑动窗口（最近原文）+ 动态摘要（历史压缩）+ 摘要异步化（不阻塞响应）

下节课预告：**Agent 优先级抢占与任务中断恢复**（Task Preemption & Resume）

---

*本课对应代码：`learn-claude-code/examples/sliding_window_memory.py`*
