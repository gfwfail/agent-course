# 79 - Agent 输出格式化与多平台渲染

> **核心问题：** 同一个 Agent，在 Telegram / Discord / Web / CLI 上输出的格式完全不同——如何优雅地适配多平台？

---

## 问题背景

Agent 说了句话：

```
这是结果：| 名称 | 价格 |
          | ... | ... |
```

- Telegram：Markdown 表格？**不支持**，原样显示乱码
- Discord：支持，但需要代码块包裹
- Web：HTML 表格最好看
- CLI：终端彩色输出

同一份内容，不同平台渲染方式差异巨大。如果 Agent 直接硬编码格式，就会在某些平台输出乱码或体验极差。

---

## 核心思路：输出与格式分离

```
Agent 生成「语义内容」  →  Formatter 转换为「平台格式」  →  Channel 发送
```

关键原则：
1. Agent 输出结构化数据（或带标记的中间格式）
2. Formatter 层负责平台适配
3. Channel 层负责最终发送

---

## 实现方案一：平台感知 Formatter

### Python（learn-claude-code 风格）

```python
from dataclasses import dataclass
from enum import Enum
from typing import Optional
import re

class Platform(Enum):
    TELEGRAM = "telegram"
    DISCORD = "discord"
    SLACK = "slack"
    WEB = "web"
    CLI = "cli"

@dataclass
class AgentResponse:
    text: str
    code_blocks: list[dict] = None   # [{lang, code}]
    table: dict = None               # {headers, rows}
    links: list[dict] = None         # [{text, url}]

class OutputFormatter:
    def format(self, response: AgentResponse, platform: Platform) -> str:
        match platform:
            case Platform.TELEGRAM:
                return self._format_telegram(response)
            case Platform.DISCORD:
                return self._format_discord(response)
            case Platform.CLI:
                return self._format_cli(response)
            case _:
                return self._format_plain(response)
    
    def _format_telegram(self, r: AgentResponse) -> str:
        # Telegram: 支持 MarkdownV2，不支持表格，链接用 [text](url)
        text = self._escape_telegram(r.text)
        
        if r.table:
            # 表格转成 bullet list
            lines = [f"**{h}**" for h in r.table["headers"]]
            header_line = " | ".join(lines)
            rows = []
            for row in r.table["rows"]:
                pairs = [f"{r.table['headers'][i]}: {v}" 
                         for i, v in enumerate(row)]
                rows.append("• " + ", ".join(pairs))
            text += "\n" + "\n".join(rows)
        
        if r.code_blocks:
            for block in r.code_blocks:
                text += f"\n```{block['lang']}\n{block['code']}\n```"
        
        return text
    
    def _format_discord(self, r: AgentResponse) -> str:
        # Discord: 支持 Markdown 表格（在代码块外），用 ``` 包代码
        text = r.text
        
        if r.table:
            headers = " | ".join(r.table["headers"])
            separator = " | ".join(["---"] * len(r.table["headers"]))
            rows = [" | ".join(str(v) for v in row) 
                    for row in r.table["rows"]]
            table_str = "\n".join([headers, separator] + rows)
            text += "\n" + table_str
        
        if r.code_blocks:
            for block in r.code_blocks:
                text += f"\n```{block['lang']}\n{block['code']}\n```"
        
        return text
    
    def _format_cli(self, r: AgentResponse) -> str:
        # CLI: ANSI 颜色 + ASCII 表格
        BOLD = "\033[1m"
        RESET = "\033[0m"
        CYAN = "\033[36m"
        
        text = r.text
        
        if r.table:
            col_widths = [len(h) for h in r.table["headers"]]
            for row in r.table["rows"]:
                for i, v in enumerate(row):
                    col_widths[i] = max(col_widths[i], len(str(v)))
            
            def fmt_row(values, bold=False):
                cells = [str(v).ljust(col_widths[i]) 
                         for i, v in enumerate(values)]
                prefix = BOLD if bold else ""
                return prefix + " | ".join(cells) + RESET
            
            separator = "-+-".join("-" * w for w in col_widths)
            lines = [fmt_row(r.table["headers"], bold=True), separator]
            lines += [fmt_row(row) for row in r.table["rows"]]
            text += "\n" + "\n".join(lines)
        
        return text
    
    def _escape_telegram(self, text: str) -> str:
        # MarkdownV2 需要转义特殊字符
        special = r'_*[]()~`>#+-=|{}.!'
        return re.sub(f'([{re.escape(special)}])', r'\\\1', text)
    
    def _format_plain(self, r: AgentResponse) -> str:
        return r.text

# 使用示例
formatter = OutputFormatter()

response = AgentResponse(
    text="用户统计数据：",
    table={
        "headers": ["用户", "开箱次数", "消费金额"],
        "rows": [
            ["Alice", 42, "$1,200"],
            ["Bob", 18, "$540"],
        ]
    }
)

print(formatter.format(response, Platform.TELEGRAM))
print("---")
print(formatter.format(response, Platform.DISCORD))
```

---

## 实现方案二：平台感知中间件（OpenClaw 架构）

OpenClaw 的实际做法：**在系统提示中注入平台信息**，让 LLM 直接生成适合当前平台的格式。

```typescript
// OpenClaw 的 channel context 注入
// 参考 OpenClaw 的 session 配置

interface ChannelContext {
  platform: 'telegram' | 'discord' | 'slack' | 'web';
  capabilities: {
    markdown: boolean;
    tables: boolean;
    codeBlocks: boolean;
    htmlLinks: boolean;
    maxMessageLength: number;
  };
}

const CHANNEL_CAPABILITIES: Record<string, ChannelContext['capabilities']> = {
  telegram: {
    markdown: true,      // MarkdownV2
    tables: false,       // ❌ 不支持
    codeBlocks: true,    // ✅ 支持
    htmlLinks: false,    // 用 [text](url) 格式
    maxMessageLength: 4096,
  },
  discord: {
    markdown: true,
    tables: true,        // ✅ 支持
    codeBlocks: true,
    htmlLinks: false,
    maxMessageLength: 2000,
  },
  web: {
    markdown: true,
    tables: true,
    codeBlocks: true,
    htmlLinks: true,
    maxMessageLength: Infinity,
  },
};

function buildPlatformPrompt(channel: string): string {
  const caps = CHANNEL_CAPABILITIES[channel] ?? CHANNEL_CAPABILITIES.web;
  
  const rules: string[] = [];
  if (!caps.tables) {
    rules.push("不要使用 Markdown 表格，改用 bullet list 或缩进格式");
  }
  if (caps.maxMessageLength < 2000) {
    rules.push(`回复不超过 ${caps.maxMessageLength} 字符`);
  }
  
  return rules.length > 0
    ? `\n\n## 输出格式规则（当前平台：${channel}）\n${rules.map(r => `- ${r}`).join('\n')}`
    : '';
}

// 注入到 system prompt
const systemPrompt = basePrompt + buildPlatformPrompt(session.channel);
```

---

## 实现方案三：结构化输出 + 后处理渲染

Agent 返回结构化 JSON，由渲染层处理。

```python
import anthropic
import json

client = anthropic.Anthropic()

RENDER_SYSTEM = """
你是一个数据分析 Agent。
始终以 JSON 格式返回结果：
{
  "summary": "一句话摘要",
  "details": [{"key": "...", "value": "..."}],
  "code_example": {"lang": "python", "code": "..."}  // 可选
}
"""

def agent_with_structured_output(query: str) -> dict:
    response = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=1024,
        system=RENDER_SYSTEM,
        messages=[{"role": "user", "content": query}]
    )
    
    try:
        return json.loads(response.content[0].text)
    except json.JSONDecodeError:
        # fallback: 提取 JSON 块
        import re
        match = re.search(r'\{.*\}', response.content[0].text, re.DOTALL)
        if match:
            return json.loads(match.group())
        return {"summary": response.content[0].text, "details": []}

class TelegramRenderer:
    def render(self, data: dict) -> str:
        lines = [f"📊 *{data['summary']}*\n"]
        
        for item in data.get('details', []):
            lines.append(f"• *{item['key']}*: {item['value']}")
        
        if code := data.get('code_example'):
            lines.append(f"\n```{code['lang']}\n{code['code']}\n```")
        
        return "\n".join(lines)

class DiscordRenderer:
    def render(self, data: dict) -> str:
        lines = [f"**{data['summary']}**\n"]
        
        if data.get('details'):
            # Discord 支持表格
            lines.append("| 项目 | 值 |")
            lines.append("|------|-----|")
            for item in data['details']:
                lines.append(f"| {item['key']} | {item['value']} |")
        
        return "\n".join(lines)

# 统一入口
RENDERERS = {
    'telegram': TelegramRenderer(),
    'discord': DiscordRenderer(),
}

def handle_request(query: str, platform: str) -> str:
    data = agent_with_structured_output(query)
    renderer = RENDERERS.get(platform, TelegramRenderer())
    return renderer.render(data)
```

---

## OpenClaw 实战：AGENTS.md 中的平台格式规则

看看 OpenClaw 的 AGENTS.md 是如何处理这个问题的：

```markdown
## Tools

**📝 Platform Formatting:**
- **Discord/WhatsApp:** No markdown tables! Use bullet lists instead
- **Discord links:** Wrap multiple links in `<>` to suppress embeds
- **WhatsApp:** No headers — use **bold** or CAPS for emphasis
```

这是最简单粗暴但有效的方案：**直接在 Agent 的系统指令中写死平台规则**。

配合 Runtime 注入的 `channel=telegram` 信息，LLM 就能在生成时自动适配。

---

## 消息分割：长内容处理

Telegram 单条消息上限 4096 字符，Discord 上限 2000 字符。需要自动切割：

```python
def split_message(text: str, max_len: int = 4096) -> list[str]:
    """按段落切割长消息，保持 Markdown 代码块完整性"""
    if len(text) <= max_len:
        return [text]
    
    chunks = []
    current = ""
    in_code_block = False
    
    for line in text.split('\n'):
        # 追踪代码块状态
        if line.startswith('```'):
            in_code_block = not in_code_block
        
        # 如果加上这行会超长
        if len(current) + len(line) + 1 > max_len:
            if in_code_block:
                # 先关闭代码块
                current += "\n```"
                chunks.append(current)
                current = "```\n" + line  # 新块继续开代码块
            else:
                chunks.append(current)
                current = line
        else:
            current = current + '\n' + line if current else line
    
    if current:
        chunks.append(current)
    
    return chunks

# 发送多条消息
async def send_long_message(bot, chat_id: int, text: str):
    for chunk in split_message(text, max_len=4000):
        await bot.send_message(chat_id, chunk, parse_mode='Markdown')
```

---

## pi-mono 的实现思路

pi-mono 中 Agent 输出统一走 `MessageBus`，渲染层订阅事件：

```typescript
// 伪代码，展示架构思路
interface AgentOutputEvent {
  type: 'text' | 'table' | 'code' | 'image';
  content: unknown;
  metadata: { platform: string; sessionId: string };
}

class OutputPipeline {
  private formatters = new Map<string, Formatter>();
  
  register(platform: string, formatter: Formatter) {
    this.formatters.set(platform, formatter);
  }
  
  async process(event: AgentOutputEvent): Promise<string> {
    const formatter = this.formatters.get(event.metadata.platform)
      ?? this.formatters.get('default')!;
    
    return formatter.format(event);
  }
}

// 注册各平台 formatter
const pipeline = new OutputPipeline();
pipeline.register('telegram', new TelegramFormatter());
pipeline.register('discord', new DiscordFormatter());
pipeline.register('default', new PlainTextFormatter());
```

---

## 总结：三种策略对比

| 策略 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| **LLM 直接感知平台** | 最灵活，LLM 自适应 | 不稳定，可能忘记规则 | 对话式 Agent |
| **结构化输出 + 渲染层** | 渲染完全可控 | 多一次 JSON 解析 | 数据密集型 Agent |
| **后处理转换** | 无需改 LLM 输出 | 转换规则复杂易出 bug | 已有输出，需适配多端 |

**最佳实践**：
1. 在 System Prompt 中注入平台能力约束
2. 对结构化数据（表格、列表）用结构化输出
3. 长消息自动切割，保持 Markdown 代码块完整
4. 维护一份平台能力注册表，避免硬编码散落各处

---

*下一课预告：Agent 输出审核与内容过滤（Output Auditing & Content Filtering）——在 Agent 回复发出之前自动拦截违禁内容*
