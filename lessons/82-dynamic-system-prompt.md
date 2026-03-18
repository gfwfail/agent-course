# 82 - Agent 动态系统提示词（Dynamic System Prompt）

> 静态 system prompt 是菜单；动态 system prompt 是大厨当场根据食材现做的私房菜。

---

## 问题：静态 system prompt 的天花板

大多数教程这么写 Agent：

```python
SYSTEM = """You are a helpful assistant.
You have access to tools: search, calculator, code_runner.
Always respond in JSON.
"""

response = llm.chat(system=SYSTEM, messages=history)
```

这有几个致命问题：

1. **工具列表写死**：用户没权限用的工具也出现在 prompt 里，LLM 可能去调用
2. **无法感知用户状态**：VIP 用户和普通用户得到同样的提示词
3. **记忆不注入**：Agent 不知道这个用户上次说了什么、有什么偏好
4. **Token 浪费**：不管当前任务是啥，所有工具描述全塞进去

**动态 system prompt** 的核心思想：**每次请求时，按当前上下文实时组装 prompt**。

---

## 分层组装架构

把 system prompt 拆成多个独立 Block，按需组合：

```
┌─────────────────────────────────────────┐
│         Dynamic System Prompt           │
├─────────────────────────────────────────┤
│ [CORE]      角色定义（必选，永远在）      │
│ [IDENTITY]  身份/Persona（基于配置）     │
│ [USER_CTX]  用户上下文（登录态才有）      │
│ [MEMORY]    相关记忆片段（检索后注入）    │
│ [TOOLS]     工具列表（按权限过滤）        │
│ [TASK]      当前任务上下文（可选）        │
│ [RULES]     运行时规则（动态生成）        │
└─────────────────────────────────────────┘
```

---

## 实现：Python（learn-claude-code 风格）

```python
from dataclasses import dataclass, field
from typing import Optional
import textwrap

@dataclass
class SystemPromptContext:
    # 用户信息
    user_id: Optional[str] = None
    user_name: Optional[str] = None
    user_tier: str = "free"          # free / pro / enterprise
    user_permissions: list[str] = field(default_factory=list)
    
    # 当前会话
    session_id: Optional[str] = None
    channel: str = "api"             # api / telegram / discord / web
    
    # 记忆（从向量数据库检索）
    relevant_memories: list[str] = field(default_factory=list)
    
    # 可用工具（过滤后的）
    available_tools: list[str] = field(default_factory=list)
    
    # 当前任务
    current_task: Optional[str] = None


class DynamicSystemPrompt:
    """动态 system prompt 组装器"""
    
    CORE = """
你是一个专业的 AI 助理，高效、准确、简洁。
今天是 {date}，当前时区：{timezone}。
    """.strip()
    
    USER_CONTEXT_TEMPLATE = """
## 当前用户
- 姓名：{name}
- 等级：{tier}
- 权限：{permissions}
    """.strip()
    
    MEMORY_TEMPLATE = """
## 关于此用户，你记得：
{memories}
    """.strip()
    
    TOOLS_TEMPLATE = """
## 可用工具
你可以调用以下工具（仅限此列表）：
{tools}
    """.strip()
    
    CHANNEL_RULES = {
        "telegram": "回复用 Telegram Markdown 格式，不用 HTML。消息不超过 4096 字符。",
        "discord":  "回复用 Discord Markdown，不要用 HTML 表格。",
        "web":      "回复可使用完整 Markdown，支持表格和代码块。",
        "api":      "回复为纯文本或 JSON，不要 Markdown 装饰。",
    }
    
    def build(self, ctx: SystemPromptContext) -> str:
        from datetime import datetime, timezone
        import pytz
        
        blocks = []
        
        # 1. 核心角色（永远存在）
        now = datetime.now(pytz.timezone("Asia/Shanghai"))
        blocks.append(self.CORE.format(
            date=now.strftime("%Y-%m-%d %H:%M"),
            timezone="Asia/Shanghai"
        ))
        
        # 2. 用户上下文（登录后才注入）
        if ctx.user_id:
            blocks.append(self.USER_CONTEXT_TEMPLATE.format(
                name=ctx.user_name or "未知",
                tier=ctx.user_tier,
                permissions=", ".join(ctx.user_permissions) or "基础权限"
            ))
        
        # 3. 相关记忆（检索到才注入）
        if ctx.relevant_memories:
            memory_text = "\n".join(f"- {m}" for m in ctx.relevant_memories[:5])  # 最多5条
            blocks.append(self.MEMORY_TEMPLATE.format(memories=memory_text))
        
        # 4. 工具列表（权限过滤后）
        if ctx.available_tools:
            tool_list = "\n".join(f"- {t}" for t in ctx.available_tools)
            blocks.append(self.TOOLS_TEMPLATE.format(tools=tool_list))
        
        # 5. 渠道规则
        channel_rule = self.CHANNEL_RULES.get(ctx.channel, self.CHANNEL_RULES["api"])
        blocks.append(f"## 输出规则\n{channel_rule}")
        
        # 6. 当前任务（可选）
        if ctx.current_task:
            blocks.append(f"## 当前任务\n{ctx.current_task}")
        
        return "\n\n".join(blocks)


# ---- 工具权限过滤器 ----

ALL_TOOLS = {
    "search":       {"tiers": ["free", "pro", "enterprise"]},
    "calculator":   {"tiers": ["free", "pro", "enterprise"]},
    "code_runner":  {"tiers": ["pro", "enterprise"]},
    "database":     {"tiers": ["enterprise"]},
    "send_email":   {"tiers": ["pro", "enterprise"], "perms": ["email:send"]},
    "admin_panel":  {"tiers": ["enterprise"], "perms": ["admin"]},
}

def get_allowed_tools(tier: str, permissions: list[str]) -> list[str]:
    allowed = []
    for tool, rules in ALL_TOOLS.items():
        if tier not in rules["tiers"]:
            continue
        required_perms = rules.get("perms", [])
        if all(p in permissions for p in required_perms):
            allowed.append(tool)
    return allowed


# ---- 使用示例 ----

async def handle_request(user_id: str, message: str, memory_store, llm):
    # 1. 加载用户信息
    user = await db.get_user(user_id)
    
    # 2. 检索相关记忆
    memories = await memory_store.search(
        query=message,
        user_id=user_id,
        top_k=5
    )
    
    # 3. 过滤工具
    tools = get_allowed_tools(user.tier, user.permissions)
    
    # 4. 组装上下文
    ctx = SystemPromptContext(
        user_id=user_id,
        user_name=user.name,
        user_tier=user.tier,
        user_permissions=user.permissions,
        channel=request.channel,
        relevant_memories=[m.text for m in memories],
        available_tools=tools,
    )
    
    # 5. 动态生成 system prompt
    builder = DynamicSystemPrompt()
    system = builder.build(ctx)
    
    # 6. 发送给 LLM
    response = await llm.chat(
        system=system,
        messages=[{"role": "user", "content": message}]
    )
    
    return response
```

---

## TypeScript 实现（pi-mono 风格）

```typescript
interface PromptContext {
  user?: {
    id: string;
    name: string;
    tier: 'free' | 'pro' | 'enterprise';
    permissions: string[];
  };
  channel: 'telegram' | 'discord' | 'web' | 'api';
  memories: string[];
  availableTools: string[];
  task?: string;
}

class DynamicSystemPrompt {
  private blocks: string[] = [];
  
  constructor(private ctx: PromptContext) {}
  
  private addCore(): this {
    const now = new Date().toLocaleString('zh-CN', { timeZone: 'Asia/Shanghai' });
    this.blocks.push(`你是一个专业 AI 助理。当前时间：${now}`);
    return this;
  }
  
  private addUserContext(): this {
    if (!this.ctx.user) return this;
    this.blocks.push(`
## 用户信息
- 用户：${this.ctx.user.name}（${this.ctx.user.tier}）
- 权限：${this.ctx.user.permissions.join(', ') || '基础'}
    `.trim());
    return this;
  }
  
  private addMemories(): this {
    if (this.ctx.memories.length === 0) return this;
    const list = this.ctx.memories.slice(0, 5).map(m => `- ${m}`).join('\n');
    this.blocks.push(`## 关于此用户你记得\n${list}`);
    return this;
  }
  
  private addTools(): this {
    if (this.ctx.availableTools.length === 0) return this;
    const list = this.ctx.availableTools.map(t => `- ${t}`).join('\n');
    this.blocks.push(`## 可用工具\n${list}`);
    return this;
  }
  
  private addChannelRules(): this {
    const rules: Record<string, string> = {
      telegram: 'Telegram Markdown 格式，消息 ≤4096 字符',
      discord:  'Discord Markdown，不用 HTML 表格',
      web:      '完整 Markdown，支持表格和代码块',
      api:      '纯文本或 JSON',
    };
    this.blocks.push(`## 输出格式\n${rules[this.ctx.channel]}`);
    return this;
  }
  
  build(): string {
    this.addCore()
        .addUserContext()
        .addMemories()
        .addTools()
        .addChannelRules();
    
    if (this.ctx.task) {
      this.blocks.push(`## 当前任务\n${this.ctx.task}`);
    }
    
    return this.blocks.join('\n\n');
  }
}

// pi-mono Agent 中的使用
class Agent {
  async run(userMessage: string, sessionCtx: SessionContext) {
    const memories = await this.memoryStore.search(userMessage, {
      userId: sessionCtx.userId,
      topK: 5,
    });
    
    const tools = this.toolRegistry.getToolsForUser(sessionCtx.user);
    
    const prompt = new DynamicSystemPrompt({
      user: sessionCtx.user,
      channel: sessionCtx.channel,
      memories: memories.map(m => m.content),
      availableTools: tools.map(t => t.name),
    }).build();
    
    return this.llm.complete({
      system: prompt,
      messages: this.history.get(sessionCtx.sessionId),
    });
  }
}
```

---

## OpenClaw 的实战做法

OpenClaw 自己就是最好的动态 system prompt 案例！每次响应前，它会自动组装：

```
SOUL.md          → 角色定义（Core）
USER.md          → 用户上下文（User Context）
TOOLS.md         → 工具笔记（Tool Context）
MEMORY.md        → 长期记忆（Memory，仅主会话）
memory/今天.md   → 短期记忆（Recent Context）
AGENTS.md        → 工作规则（Rules）
```

这些文件被注入为 `# Project Context`，在每次对话前自动加载。

**核心设计：文件即上下文块**。不同的 `.md` 文件对应不同的 prompt 块，可以独立更新，不影响其他部分。

```
# Runtime 行注入的是动态变量
Runtime: agent=main | host=xxx | model=claude-sonnet-4-6 | channel=telegram
```

这一行包含：渠道、模型、主机名、时区——全是**运行时动态注入**的，不是写死的。

---

## Token 预算控制

动态组装最大的风险：**prompt 越来越长**。需要给每个 Block 设上限：

```python
BLOCK_LIMITS = {
    "core":       300,    # tokens
    "user_ctx":   200,
    "memories":   500,    # 最多5条记忆，每条100 token
    "tools":      400,    # 工具列表
    "channel":    100,
    "task":       300,
}

TOTAL_BUDGET = 2000  # system prompt 总 token 上限

def build_with_budget(blocks: dict[str, str], budget: int) -> str:
    """按优先级填充 system prompt，不超过 token 预算"""
    priority = ["core", "user_ctx", "task", "tools", "memories", "channel"]
    result = []
    used = 0
    
    for key in priority:
        if key not in blocks:
            continue
        tokens = estimate_tokens(blocks[key])
        if used + tokens > budget:
            # 尝试截断而不是跳过
            truncated = truncate_to_tokens(blocks[key], budget - used)
            if truncated:
                result.append(truncated)
            break
        result.append(blocks[key])
        used += tokens
    
    return "\n\n".join(result)
```

---

## 常见坑

| 坑 | 症状 | 解法 |
|---|---|---|
| 权限工具没过滤 | LLM 尝试调用无权限工具 | 工具列表与实际可调用工具保持同步 |
| 记忆注入太多 | prompt 超 token 限制 | 限制记忆条数 + 摘要压缩 |
| 渠道规则缺失 | Telegram 收到 HTML 代码 | channel 参数必传，有默认值 |
| 时间不注入 | LLM 说"我不知道今天是几号" | 在 Core block 注入当前时间 |
| 静态缓存 prompt | 用户升级权限后还是旧工具列表 | 不要缓存 system prompt，每次新建 |

---

## 总结

动态 system prompt 的本质是**关注点分离**：

- **角色定义**（谁） → Core Block，稳定
- **用户状态**（给谁） → 运行时加载
- **记忆**（历史） → 向量检索注入
- **工具**（能做什么） → 权限过滤后注入
- **渠道**（怎么输出） → 按平台规则注入

每个 Block 独立维护，组合成一个上下文精准的 system prompt。

**OpenClaw 已经在用这套思路**——你的 SOUL.md、USER.md、MEMORY.md 就是这套架构的文件化实现。

---

*下一讲预告：Agent 知识库冷热分层（Hot/Warm/Cold Knowledge Tiering）——按访问频率和时效性分层存储，降低成本同时保持响应速度。*
