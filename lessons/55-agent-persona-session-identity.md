# 55 - Agent Persona & Session Identity：角色持久化与身份一致性

> **核心问题**：Agent 如何在每次会话中"记得自己是谁"？

---

## 问题背景

LLM 本身是无状态的。每次 API 调用，模型对自己是谁、应该怎么说话一无所知。

但一个好的 Agent 应该有**稳定的人格**：
- 说话风格一致
- 价值观和立场一致
- 对同一件事的判断方式一致

这就是 **Persona（角色）持久化**要解决的问题。

---

## 方案一：System Prompt 注入（最基础）

最简单的方案：把角色描述写进 system prompt。

```python
# learn-claude-code 风格
SYSTEM_PROMPT = """
You are Alex, a senior software engineer.
- Communicate in a direct, technical style
- Prefer code examples over abstract explanations
- If asked to do something unsafe, refuse clearly but without lecturing
"""

response = client.messages.create(
    model="claude-3-5-sonnet-20241022",
    system=SYSTEM_PROMPT,
    messages=[{"role": "user", "content": user_input}]
)
```

**问题**：硬编码，无法动态切换角色，无法跨平台复用。

---

## 方案二：SOUL.md 文件机制（OpenClaw 实现）

OpenClaw 的做法更优雅：把角色定义从代码里抽离，放进独立文件。

```
workspace/
├── SOUL.md          # 角色定义（人格、语气、规则）
├── USER.md          # 用户画像（老板是谁）
├── AGENTS.md        # 行为规范
└── IDENTITY.md      # 简短身份卡片
```

**SOUL.md 内容结构**：
```markdown
# SOUL.md - Who You Are

## 基本信息
- **名字**：性奴001
- **身份**：老板的私人 AI 助理

## 语言风格
- 专业、高效、简洁
- 不废话，直击要点

## 原则
- 老板让干啥就干啥，不要自作主张
- 遇到问题先汇报，等老板指示
```

**Agent 启动时自动加载**：
```typescript
// OpenClaw 伪代码
async function buildSystemPrompt(workspaceDir: string): Promise<string> {
  const soul = await readFile(`${workspaceDir}/SOUL.md`);
  const identity = await readFile(`${workspaceDir}/IDENTITY.md`);
  
  return `
## Your Identity
${identity}

## Your Character
${soul}

## Instructions
${AGENTS_MD}
  `.trim();
}
```

---

## 方案三：pi-mono 的 Character 系统

pi-mono 把角色定义成强类型结构：

```typescript
// pi-mono/packages/core/src/character.ts
interface Character {
  name: string;
  description: string;
  personality: string[];
  speechPatterns: string[];
  constraints: string[];
  examples: ConversationExample[];
}

const alexCharacter: Character = {
  name: "Alex",
  description: "A senior software engineer who values clarity",
  personality: [
    "Direct and concise",
    "Uses code examples freely",
    "Slightly sarcastic about bad practices"
  ],
  speechPatterns: [
    "Starts technical answers with 'Short answer: ..., Long answer: ...'",
    "Uses '→' instead of 'leads to' or 'results in'"
  ],
  constraints: [
    "Never writes code without understanding the requirement first",
    "Always mentions edge cases"
  ],
  examples: [
    {
      user: "How do I center a div?",
      assistant: "Short answer: flexbox. Long answer: depends on your layout..."
    }
  ]
};

// 编译成 system prompt
function characterToSystemPrompt(char: Character): string {
  return `
You are ${char.name}. ${char.description}

## Personality
${char.personality.map(p => `- ${p}`).join('\n')}

## Speech Patterns  
${char.speechPatterns.map(p => `- ${p}`).join('\n')}

## Constraints
${char.constraints.map(c => `- ${c}`).join('\n')}

## Example Conversations
${char.examples.map(e => `User: ${e.user}\nYou: ${e.assistant}`).join('\n\n')}
  `.trim();
}
```

**优势**：
- 强类型，IDE 有提示
- 可以程序化地组合多个角色特征
- 方便做 A/B 测试（换个 character 对比效果）

---

## 方案四：角色一致性检验（Persona Guard）

光注入还不够。LLM 在长对话中会"漂移"——开始还是那个角色，聊着聊着就失忆了。

**检测漂移**：

```python
async def check_persona_drift(
    character: Character,
    recent_messages: list[dict]
) -> float:
    """返回 0-1 的漂移分数，越高越偏离角色"""
    
    prompt = f"""
角色定义：
{character.to_prompt()}

以下是 Agent 最近的回复：
{format_messages(recent_messages)}

这些回复与角色定义的一致性如何？
返回 JSON: {{"consistency_score": 0.0-1.0, "issues": ["..."]}}
"""
    result = await llm_call(prompt, response_format="json")
    return result["consistency_score"]
```

**自动修正**（在每 N 轮对话后注入提醒）：

```typescript
class PersonaGuard {
  private turnCount = 0;
  private readonly checkInterval = 20; // 每20轮检查一次
  
  async processMessage(messages: Message[]): Promise<Message[]> {
    this.turnCount++;
    
    if (this.turnCount % this.checkInterval === 0) {
      // 注入角色提醒
      messages = [
        ...messages,
        {
          role: "system",
          content: `[PERSONA REMINDER] Remember: ${this.character.coreTraits.join(', ')}`
        }
      ];
    }
    
    return messages;
  }
}
```

---

## 方案五：跨平台身份统一（Multi-Channel Identity）

同一个 Agent 可能在 Telegram、Discord、网页同时运行。各平台的限制不同，但人格要一致。

```typescript
// OpenClaw 的 channel-aware persona
interface PlatformAdaptation {
  telegram: {
    maxLength: 4096,
    supportsMarkdown: true,
    supportsVoice: true
  };
  discord: {
    maxLength: 2000,
    supportsMarkdown: true,
    noTables: true  // Discord 不支持 markdown 表格
  };
  sms: {
    maxLength: 160,
    supportsMarkdown: false,
    style: "ultra-concise"
  };
}

function adaptPersonaToChannel(
  basePersona: Character,
  channel: keyof PlatformAdaptation
): string {
  const base = characterToSystemPrompt(basePersona);
  const adaptation = CHANNEL_RULES[channel];
  
  return `${base}

## Platform Rules (${channel})
${adaptation.rules.join('\n')}`;
}
```

**OpenClaw 的实际实现**：在 AGENTS.md 中有专门的"Platform Formatting"章节：
```markdown
## 📝 Platform Formatting
- **Discord/WhatsApp:** No markdown tables! Use bullet lists instead
- **Discord links:** Wrap multiple links in <> to suppress embeds
- **WhatsApp:** No headers — use **bold** or CAPS for emphasis
```

---

## 最佳实践

### 1. 角色分层设计

```
不变的核心 (SOUL.md)
    ↓
平台适配规则 (AGENTS.md)
    ↓
用户画像上下文 (USER.md)
    ↓
当日状态/情绪 (memory/YYYY-MM-DD.md)
```

### 2. 角色 vs 指令优先级

```typescript
const PRIORITY_ORDER = [
  "safety_rules",      // 最高：安全规则不可逾越
  "user_instructions", // 次高：用户明确指令
  "character_traits",  // 中：角色特征
  "defaults"           // 最低：兜底行为
];
```

### 3. 测试角色一致性

```python
# 角色一致性测试用例
persona_tests = [
    {
        "scenario": "用户问一个愚蠢的问题",
        "expected_behavior": "直接回答，不嘲讽，不废话",
        "forbidden": ["你真傻", "这都不知道", "首先让我们..."]
    },
    {
        "scenario": "用户让做危险操作",
        "expected_behavior": "拒绝并说明原因，不说教",
        "forbidden": ["我不允许", "这是非常危险的", "你应该"]
    }
]
```

---

## 总结

| 方案 | 适用场景 | 复杂度 |
|------|---------|--------|
| System Prompt 硬编码 | 原型/单一场景 | ⭐ |
| SOUL.md 文件加载 | 生产 Agent，需要灵活调整 | ⭐⭐ |
| 强类型 Character 对象 | 多角色、需要 A/B 测试 | ⭐⭐⭐ |
| Persona Guard（漂移检测） | 长对话、高一致性要求 | ⭐⭐⭐⭐ |
| 跨平台身份统一 | 多渠道部署 | ⭐⭐⭐ |

**OpenClaw 的做法是最务实的**：SOUL.md + IDENTITY.md + USER.md 三文件分工，在灵活性和简单性之间取得了很好的平衡。角色定义在文件里，不需要重新部署就能调整人格。

---

**下一课预告**：Agent 知识图谱 — 当 RAG 还不够时，如何用图谱做关系推理
