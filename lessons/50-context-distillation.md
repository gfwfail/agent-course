# Lesson 50 - Context Distillation：上下文蒸馏

> "不是把旧内容删掉，而是把精华提炼出来。"

## 问题背景

Agent 运行时间越长，对话历史越来越大：

```
Turn 1:   用户问问题 A
Turn 2:   Agent 查询数据库、调用 3 个工具
Turn 3:   用户追问、Agent 再次调用工具
...
Turn 50:  Context 已经 80k tokens
Turn 51:  OOM / 超出窗口 / 费用暴涨
```

**Lesson 07（Context Window Management）** 讲的是"截断策略"——把旧消息丢掉。

**本课讲的是"蒸馏策略"**——用 LLM 把旧内容压缩成结构化知识，保留精华，丢掉噪声。

---

## 核心思路：蒸馏 vs 截断

| 策略 | 做法 | 优点 | 缺点 |
|------|------|------|------|
| 截断 | 直接删旧消息 | 简单快 | 丢失上下文 |
| 滑动窗口 | 保留最近 N 条 | 简单 | 远期信息全丢 |
| **蒸馏** | LLM 总结旧消息 | 保留要点 | 多一次 LLM 调用 |
| 分层蒸馏 | 多轮递归压缩 | 极致压缩 | 复杂度高 |

---

## 实现一：简单摘要蒸馏

最直接的做法：当对话超过阈值时，用一个小模型总结前半段。

### Python（learn-claude-code 风格）

```python
import anthropic
from typing import List

client = anthropic.Anthropic()

DISTILL_THRESHOLD = 20  # 超过 20 条消息触发蒸馏
KEEP_RECENT = 6         # 保留最近 6 条原始消息


def distill_messages(messages: List[dict]) -> str:
    """用 LLM 把一段对话蒸馏成结构化摘要"""
    
    # 构建摘要 prompt
    conversation_text = "\n".join([
        f"{m['role'].upper()}: {m['content']}"
        for m in messages
        if isinstance(m['content'], str)
    ])
    
    response = client.messages.create(
        model="claude-haiku-4-5",  # 用便宜小模型做摘要
        max_tokens=1024,
        system="""你是一个对话摘要助手。
        
请从以下对话中提炼：
1. **已完成的任务**（bullet list）
2. **关键决策**（什么问题，决定怎么做）
3. **重要数据**（数字、文件路径、API 地址等）
4. **未完成事项**（如果有）
5. **用户偏好**（用户喜欢/不喜欢什么）

输出格式：Markdown，简洁，不超过 500 字。""",
        messages=[{
            "role": "user", 
            "content": f"请摘要以下对话：\n\n{conversation_text}"
        }]
    )
    
    return response.content[0].text


def smart_compress(messages: List[dict]) -> List[dict]:
    """智能压缩消息历史"""
    
    if len(messages) <= DISTILL_THRESHOLD:
        return messages  # 还不需要压缩
    
    # 分割：旧消息 + 最近消息
    old_messages = messages[:-KEEP_RECENT]
    recent_messages = messages[-KEEP_RECENT:]
    
    # 蒸馏旧消息
    summary = distill_messages(old_messages)
    
    # 构建新的历史：摘要 + 最近原始消息
    compressed = [
        {
            "role": "user",
            "content": f"[对话历史摘要]\n{summary}\n\n[以上是之前对话的摘要，接下来是最近的对话]"
        },
        {
            "role": "assistant", 
            "content": "收到，我已了解之前的对话背景。"
        }
    ] + recent_messages
    
    print(f"[蒸馏] {len(messages)} 条 → {len(compressed)} 条 "
          f"（节省 {len(messages) - len(compressed)} 条）")
    
    return compressed


# 使用示例
class DistillingAgent:
    def __init__(self):
        self.messages = []
    
    def chat(self, user_input: str) -> str:
        self.messages.append({"role": "user", "content": user_input})
        
        # 触发蒸馏
        if len(self.messages) > DISTILL_THRESHOLD:
            self.messages = smart_compress(self.messages)
        
        response = client.messages.create(
            model="claude-sonnet-4-5",
            max_tokens=2048,
            messages=self.messages
        )
        
        reply = response.content[0].text
        self.messages.append({"role": "assistant", "content": reply})
        return reply
```

---

## 实现二：结构化知识蒸馏

更高级的做法：不是自由文本摘要，而是提取到结构化对象。

### TypeScript（pi-mono 风格）

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();

// 蒸馏出来的结构化知识
interface DistilledKnowledge {
  completedTasks: string[];       // 已完成任务
  keyFacts: Record<string, string>; // 关键事实（key-value）
  userPreferences: string[];      // 用户偏好
  pendingItems: string[];         // 待办事项
  sessionContext: string;         // 会话背景（自然语言）
  distilledAt: number;            // 蒸馏时间戳
  originalTurnCount: number;      // 原始对话轮数
}

async function distillToStructured(
  messages: Anthropic.MessageParam[]
): Promise<DistilledKnowledge> {
  
  const conversationText = messages
    .map(m => `${m.role.toUpperCase()}: ${
      typeof m.content === 'string' ? m.content : '[tool calls]'
    }`)
    .join('\n');
  
  const response = await client.messages.create({
    model: "claude-haiku-4-5",
    max_tokens: 1500,
    system: `你是对话知识提取助手，请从对话中提取结构化知识。
    
必须返回有效 JSON，格式如下：
{
  "completedTasks": ["任务1", "任务2"],
  "keyFacts": {"文件路径": "/tmp/output.json", "API地址": "https://..."},
  "userPreferences": ["喜欢简洁回答", "不要用表格"],
  "pendingItems": ["还需要检查X", "等待用户确认Y"],
  "sessionContext": "一句话描述这次对话的主要目的"
}`,
    messages: [{
      role: "user",
      content: `请提取以下对话的结构化知识：\n\n${conversationText}`
    }]
  });
  
  const text = response.content[0].type === 'text' 
    ? response.content[0].text : '{}';
  
  // 提取 JSON（可能被 markdown 包裹）
  const jsonMatch = text.match(/\{[\s\S]*\}/);
  const parsed = jsonMatch ? JSON.parse(jsonMatch[0]) : {};
  
  return {
    completedTasks: parsed.completedTasks || [],
    keyFacts: parsed.keyFacts || {},
    userPreferences: parsed.userPreferences || [],
    pendingItems: parsed.pendingItems || [],
    sessionContext: parsed.sessionContext || '',
    distilledAt: Date.now(),
    originalTurnCount: messages.length
  };
}

// 把结构化知识转回系统提示词注入
function knowledgeToSystemPrompt(knowledge: DistilledKnowledge): string {
  const lines: string[] = ['[对话历史摘要]'];
  
  if (knowledge.sessionContext) {
    lines.push(`**背景**：${knowledge.sessionContext}`);
  }
  
  if (knowledge.completedTasks.length > 0) {
    lines.push('\n**已完成**：');
    knowledge.completedTasks.forEach(t => lines.push(`- ${t}`));
  }
  
  if (Object.keys(knowledge.keyFacts).length > 0) {
    lines.push('\n**关键信息**：');
    Object.entries(knowledge.keyFacts)
      .forEach(([k, v]) => lines.push(`- ${k}: ${v}`));
  }
  
  if (knowledge.pendingItems.length > 0) {
    lines.push('\n**待处理**：');
    knowledge.pendingItems.forEach(p => lines.push(`- ${p}`));
  }
  
  if (knowledge.userPreferences.length > 0) {
    lines.push('\n**用户偏好**：');
    knowledge.userPreferences.forEach(p => lines.push(`- ${p}`));
  }
  
  return lines.join('\n');
}

// 带蒸馏的 Agent 类
class DistillingAgent {
  private messages: Anthropic.MessageParam[] = [];
  private knowledge: DistilledKnowledge | null = null;
  private readonly distillThreshold = 15;
  private readonly keepRecent = 4;
  
  async chat(userInput: string): Promise<string> {
    this.messages.push({ role: "user", content: userInput });
    
    // 触发蒸馏
    if (this.messages.length > this.distillThreshold) {
      await this.distill();
    }
    
    // 构建带知识注入的系统提示
    const systemParts = ['你是一个专业助手。'];
    if (this.knowledge) {
      systemParts.push(knowledgeToSystemPrompt(this.knowledge));
    }
    
    const response = await client.messages.create({
      model: "claude-sonnet-4-5",
      max_tokens: 2048,
      system: systemParts.join('\n\n'),
      messages: this.messages
    });
    
    const reply = response.content[0].type === 'text' 
      ? response.content[0].text : '';
    
    this.messages.push({ role: "assistant", content: reply });
    return reply;
  }
  
  private async distill(): Promise<void> {
    const toDistill = this.messages.slice(0, -this.keepRecent);
    const recent = this.messages.slice(-this.keepRecent);
    
    console.log(`[蒸馏] 压缩 ${toDistill.length} 条旧消息...`);
    
    // 合并历史知识 + 本次蒸馏
    const newKnowledge = await distillToStructured(toDistill);
    
    if (this.knowledge) {
      // 合并知识（新的覆盖旧的同名 key）
      this.knowledge = {
        ...newKnowledge,
        keyFacts: { ...this.knowledge.keyFacts, ...newKnowledge.keyFacts },
        completedTasks: [
          ...this.knowledge.completedTasks, 
          ...newKnowledge.completedTasks
        ],
        userPreferences: [
          ...new Set([
            ...this.knowledge.userPreferences,
            ...newKnowledge.userPreferences
          ])
        ]
      };
    } else {
      this.knowledge = newKnowledge;
    }
    
    // 只保留最近原始消息
    this.messages = recent;
    
    console.log(`[蒸馏完成] 已提取 ${Object.keys(this.knowledge.keyFacts).length} 个关键事实`);
  }
}
```

---

## 实现三：OpenClaw 的分层蒸馏

OpenClaw 的 Memory System 实际上就是一个多层蒸馏架构：

```
对话原文 (raw turns)
    ↓ 触发蒸馏
memory/YYYY-MM-DD.md (日级别：今天发生了什么)
    ↓ 定期整合
MEMORY.md (长期记忆：重要的持久知识)
```

```typescript
// OpenClaw 风格的分层蒸馏
interface DistillationLayer {
  name: string;
  maxAge: number;        // ms，超过这个时间就蒸馏到下一层
  maxItems: number;      // 超过这个条数就蒸馏
  prompt: string;        // 蒸馏时用的 prompt
}

const LAYERS: DistillationLayer[] = [
  {
    name: 'session',
    maxAge: 0,
    maxItems: 20,
    prompt: '总结这段对话的要点，重点是操作步骤和结果'
  },
  {
    name: 'daily',
    maxAge: 24 * 60 * 60 * 1000,  // 1 天
    maxItems: 100,
    prompt: '总结今天的重要事件和决策，提取持久有用的知识'
  },
  {
    name: 'longterm',
    maxAge: Infinity,
    maxItems: Infinity,
    prompt: '这是长期记忆，只保留真正重要的知识和偏好'
  }
];

// 实际上就是 OpenClaw 的 MEMORY.md 机制：
// - Heartbeat 时检查 memory/今天.md
// - 定期把日志提炼进 MEMORY.md
// - MEMORY.md 注入系统提示
```

---

## 触发蒸馏的时机

```python
def should_distill(messages: list, config: dict) -> tuple[bool, str]:
    """决定是否应该触发蒸馏，以及原因"""
    
    # 1. 消息条数超限
    if len(messages) > config.get('max_turns', 30):
        return True, f"消息数 {len(messages)} 超过阈值"
    
    # 2. Token 估算超限（简单估算：平均每条 200 tokens）
    estimated_tokens = sum(
        len(str(m.get('content', ''))) // 4 
        for m in messages
    )
    if estimated_tokens > config.get('max_tokens', 60000):
        return True, f"估算 token {estimated_tokens} 超过阈值"
    
    # 3. 话题切换（检测到明显的主题转换）
    if len(messages) >= 4:
        recent = messages[-1].get('content', '')
        if any(kw in str(recent).lower() for kw in ['换个话题', '另外', '顺便问一下']):
            return True, "检测到话题切换"
    
    # 4. 时间间隔（长时间暂停后继续）
    if 'timestamp' in messages[-1]:
        gap = time.time() - messages[-1]['timestamp']
        if gap > config.get('idle_threshold', 3600):  # 1小时
            return True, f"空闲时间 {gap:.0f}s 超过阈值"
    
    return False, ""
```

---

## 蒸馏质量保证

蒸馏有损耗——如何确保关键信息不丢失？

```python
def verify_distillation(
    original: list[dict], 
    distilled_summary: str,
    key_facts: list[str]  # 已知的关键信息
) -> dict:
    """验证蒸馏质量"""
    
    missing = []
    for fact in key_facts:
        if fact.lower() not in distilled_summary.lower():
            missing.append(fact)
    
    if missing:
        # 补救：强制追加遗漏的关键信息
        supplement = "\n\n[重要补充，以下信息必须保留：]\n"
        supplement += "\n".join(f"- {f}" for f in missing)
        distilled_summary += supplement
        print(f"[蒸馏补救] 追加了 {len(missing)} 个遗漏事实")
    
    return {
        "summary": distilled_summary,
        "quality_score": 1.0 - len(missing) / max(len(key_facts), 1),
        "missing_facts": missing
    }

# 使用场景：蒸馏前先标记关键信息
KEY_FACT_MARKERS = [
    r'文件路径[：:]\s*(\S+)',
    r'API[：:]\s*(https?://\S+)',
    r'密码[：:]\s*(\S+)',
    r'用户ID[：:]\s*(\d+)',
]
```

---

## 与其他模式的配合

```
用户对话
  │
  ├─ [每 N 条] → 蒸馏 → 结构化知识 → 注入系统提示
  │                                         │
  │                                    [Lesson 29 Context Injection]
  │
  ├─ [关键信息] → 向量化 → 存入向量库
  │                              │
  │                         [Lesson 34 RAG]
  │
  └─ [重要决策] → 写入 MEMORY.md
                        │
                  [OpenClaw Memory System]
```

---

## 实战建议

1. **蒸馏模型选小模型**：用 claude-haiku 或 gpt-4o-mini 做蒸馏，成本不到主模型的 1/10
2. **触发阈值要保守**：宁可早蒸馏也不要等到 OOM
3. **保留最近原始消息**：至少保留最近 4-6 条，让 LLM 有完整上下文
4. **测试蒸馏质量**：对关键业务场景写蒸馏测试（参考 Lesson 14 Agent Testing）
5. **结合 RAG 使用**：超长历史放进向量库，蒸馏只处理中等长度的对话

---

## 小结

| 场景 | 推荐策略 |
|------|---------|
| 短对话（< 20 轮） | 不需要蒸馏 |
| 中等对话（20-50 轮） | 简单摘要蒸馏 |
| 长对话（> 50 轮） | 结构化知识蒸馏 |
| 持久 Agent（跨天） | 分层蒸馏（session → daily → longterm） |
| 多用户系统 | 每用户独立蒸馏管道 |

蒸馏不是删除，是**提炼**。就像人类记忆——你不会记住每一句话，但重要的事情会以结构化方式留在大脑里。

---

*下一课预告：Agent 负载均衡与流量调度*
