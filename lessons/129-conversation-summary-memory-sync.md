# Lesson 129: Agent 对话摘要与长期记忆同步
# Conversation Summary & Long-Term Memory Sync

## 为什么需要这个模式？

Agent 的 context window 是有限的。随着对话越来越长：
- 早期的重要决策被挤出窗口
- 跨会话的用户偏好丢失
- 每次新会话都要重新"认识"用户

解决方案：**主动提取对话精华，异步写入长期记忆**。

---

## 核心架构：三层记忆模型

```
┌─────────────────────────────────────────┐
│  工作记忆（Working Memory）              │
│  当前 context window 内的对话           │
│  容量: ~200K tokens，会话结束即丢失      │
└─────────────────┬───────────────────────┘
                  │ 摘要提取
                  ▼
┌─────────────────────────────────────────┐
│  情景记忆（Episodic Memory）             │
│  memory/YYYY-MM-DD.md 日记文件          │
│  按时间索引，保留原始事件               │
└─────────────────┬───────────────────────┘
                  │ 蒸馏精华
                  ▼
┌─────────────────────────────────────────┐
│  语义记忆（Semantic Memory）             │
│  MEMORY.md 长期记忆文件                 │
│  提炼后的洞察、偏好、重要事实           │
└─────────────────────────────────────────┘
```

这正是 **OpenClaw** 的 AGENTS.md 中描述的实际架构！

---

## 实现：对话摘要提取器

```typescript
// memory/conversation-summarizer.ts
import Anthropic from "@anthropic-ai/sdk";
import fs from "fs/promises";
import path from "path";

interface ConversationMessage {
  role: "user" | "assistant";
  content: string;
  timestamp: number;
}

interface MemorySummary {
  decisions: string[];     // 重要决策
  preferences: string[];   // 用户偏好
  facts: string[];         // 关键事实
  todos: string[];         // 待办事项
  lessons: string[];       // 经验教训
}

class ConversationSummarizer {
  private client: Anthropic;
  private workspaceDir: string;

  constructor(workspaceDir: string) {
    this.client = new Anthropic();
    this.workspaceDir = workspaceDir;
  }

  // 第一步：提取对话摘要（情景记忆）
  async extractEpisodicMemory(
    messages: ConversationMessage[],
    date: string
  ): Promise<string> {
    // 构建对话文本
    const dialogText = messages
      .map(m => `[${new Date(m.timestamp).toTimeString().slice(0, 5)}] ${m.role}: ${m.content}`)
      .join("\n");

    const response = await this.client.messages.create({
      model: "claude-opus-4-5",
      max_tokens: 1024,
      messages: [{
        role: "user",
        content: `请将以下对话总结为简洁的日记条目（中文）。
保留：重要决策、完成的任务、遇到的问题、用户的明确偏好。
去掉：闲聊、重复内容、技术细节的完整代码。

对话内容：
${dialogText}

输出格式：
- 要点1
- 要点2
...`
      }]
    });

    const summary = response.content[0].type === "text"
      ? response.content[0].text
      : "";

    // 写入日记文件
    const diaryPath = path.join(this.workspaceDir, "memory", `${date}.md`);
    const existingContent = await fs.readFile(diaryPath, "utf-8").catch(() => "");
    const timestamp = new Date().toLocaleTimeString("zh-CN");
    const newEntry = `\n## ${timestamp}\n${summary}\n`;

    await fs.writeFile(diaryPath, existingContent + newEntry, "utf-8");
    return summary;
  }

  // 第二步：蒸馏长期记忆（语义记忆）
  async syncToLongTermMemory(recentDays: number = 7): Promise<void> {
    // 读取近期日记
    const today = new Date();
    const recentEntries: string[] = [];

    for (let i = 0; i < recentDays; i++) {
      const date = new Date(today);
      date.setDate(date.getDate() - i);
      const dateStr = date.toISOString().slice(0, 10);
      const diaryPath = path.join(this.workspaceDir, "memory", `${dateStr}.md`);

      const content = await fs.readFile(diaryPath, "utf-8").catch(() => null);
      if (content) recentEntries.push(`=== ${dateStr} ===\n${content}`);
    }

    if (recentEntries.length === 0) return;

    // 读取现有长期记忆
    const memoryPath = path.join(this.workspaceDir, "MEMORY.md");
    const existingMemory = await fs.readFile(memoryPath, "utf-8").catch(() => "");

    // 用 LLM 合并更新长期记忆
    const response = await this.client.messages.create({
      model: "claude-opus-4-5",
      max_tokens: 2048,
      system: `你是一个记忆管理系统。你的任务是：
1. 分析近期日记条目，提取值得长期记忆的洞察
2. 与现有长期记忆合并（去重、更新过时信息）
3. 输出更新后的完整 MEMORY.md 内容
保持简洁，只记录真正重要的事项。`,
      messages: [{
        role: "user",
        content: `现有长期记忆：
${existingMemory || "（空）"}

近期日记（${recentDays}天）：
${recentEntries.join("\n\n")}

请输出更新后的 MEMORY.md 内容（保持 Markdown 格式）：`
      }]
    });

    const updatedMemory = response.content[0].type === "text"
      ? response.content[0].text
      : existingMemory;

    await fs.writeFile(memoryPath, updatedMemory, "utf-8");
    console.log("✅ 长期记忆已同步更新");
  }
}
```

---

## OpenClaw 实战：心跳触发记忆同步

在 OpenClaw 的 `HEARTBEAT.md` 中配置定期同步：

```markdown
## 记忆维护（每周一次）
- [ ] 读取 memory/YYYY-MM-DD.md（近7天）
- [ ] 提炼重要事件写入 MEMORY.md
- [ ] 删除已过期/不再相关的条目
```

或用 Cron 在每天深夜自动触发：

```typescript
// OpenClaw cron job（每天凌晨2点）
{
  name: "memory-sync",
  schedule: { kind: "cron", expr: "0 2 * * *", tz: "Australia/Sydney" },
  payload: {
    kind: "agentTurn",
    message: "请执行记忆同步：读取近7天日记，更新 MEMORY.md 长期记忆，去掉过时内容。",
    timeoutSeconds: 120
  },
  sessionTarget: "isolated"
}
```

---

## learn-claude-code 中的记忆同步模式

`learn-claude-code` 的 `memory.ts` 实现了类似的分层写入：

```typescript
// 情景记忆：记录原始事件（append-only）
async function appendEpisodicMemory(event: MemoryEvent): Promise<void> {
  const date = new Date().toISOString().slice(0, 10);
  const episodicPath = `memory/${date}.md`;
  const line = `- [${event.type}] ${event.content}\n`;
  await fs.appendFile(episodicPath, line);
}

// 语义记忆：提炼后更新（read-modify-write）
async function updateSemanticMemory(insight: string): Promise<void> {
  const memory = await readMemoryFile();
  const updated = await distillWithLLM(memory, insight);
  await writeMemoryFile(updated);
}

// 程序记忆：操作模式（写入 AGENTS.md / TOOLS.md）
async function updateProceduralMemory(lesson: string): Promise<void> {
  await appendToFile("AGENTS.md", `\n## 经验教训\n- ${lesson}`);
}
```

---

## 关键设计决策

### 1. 何时触发摘要？

| 触发时机 | 适用场景 |
|---------|---------|
| 会话结束时 | 短对话，实时性要求高 |
| 消息数阈值（每20条） | 长对话，避免遗忘 |
| 定时 Cron（每天） | 批量处理，节省成本 |
| 重要事件触发 | 决策、错误等关键节点 |

### 2. 摘要粒度控制

```typescript
// 细粒度：每次对话
await summarizer.extractEpisodicMemory(messages, today);

// 粗粒度：批量蒸馏（省钱）
if (daysSinceLastSync >= 7) {
  await summarizer.syncToLongTermMemory(7);
}
```

### 3. 防止记忆污染

```typescript
// 过滤低质量内容
const SKIP_PATTERNS = [
  /^(好的|明白|收到|OK)/,  // 简单确认
  /^HEARTBEAT_OK$/,          // 心跳响应
];

function shouldRecord(message: string): boolean {
  return !SKIP_PATTERNS.some(p => p.test(message.trim()));
}
```

---

## 效果对比

```
❌ 无记忆同步：
用户："上次我说我不喜欢啰嗦的回答来着"
Agent："您好！请问有什么可以帮助您的呢？"（重新认识）

✅ 有记忆同步：
Agent 加载 MEMORY.md → "老板偏好简洁回答"
Agent："收到，直接说结果就好。"（立即适配）
```

---

## 总结

| 记忆类型 | 文件 | 写入时机 | 保留期 |
|---------|------|---------|-------|
| 工作记忆 | context window | 实时 | 会话内 |
| 情景记忆 | memory/YYYY-MM-DD.md | 每次对话后 | 30天 |
| 语义记忆 | MEMORY.md | 每周蒸馏 | 永久 |
| 程序记忆 | AGENTS.md/TOOLS.md | 学到新知识时 | 永久 |

**核心思想**：不要让 Agent 依赖 context window 来"记住"事情。主动写文件，主动蒸馏，让记忆成为资产而非负担。
