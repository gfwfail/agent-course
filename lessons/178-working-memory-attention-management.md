# 178. Agent 工作记忆与注意力管理（Working Memory & Attention Management）

> **核心思想：** LLM 的 context window 就是 Agent 的"意识"——往里塞什么、扔掉什么，决定了 Agent 能不能聚焦做好当前子任务。工作记忆（Working Memory）是认知科学概念在 Agent 工程中的直接落地。

---

## 1. 为什么需要"工作记忆"管理？

人类短期记忆容量约 7±2 个组块（Miller's Law）。LLM context window 虽大，但也有上限，更重要的是：**context 越长，注意力越分散，LLM 表现越差**（lost-in-the-middle 问题）。

```
┌─────────────────────────────────────────────────────┐
│                  Context Window (200K tokens)        │
│                                                     │
│  [System Prompt] [Long Conversation] [Tool Results] │
│       ↑                  ↑                 ↑        │
│    重要但静态          大量噪声           当前关键     │
│                                                     │
│  ❌ 问题：LLM 注意力被历史对话/无关工具结果稀释        │
└─────────────────────────────────────────────────────┘
```

**工作记忆管理**的目标：在每次 LLM 调用前，**精确控制注入哪些内容**，让 context 聚焦于当前子任务最相关的信息。

---

## 2. 工作记忆的四个槽位

参考认知科学模型（Baddeley's Working Memory Model），Agent 工作记忆分为：

| 槽位 | 类比 | Agent 实现 |
|------|------|-----------|
| **Phonological Loop** | 语音循环（近期对话） | 最近 N 轮消息 |
| **Visuospatial Sketchpad** | 视觉空间（当前任务状态） | 任务目标 + 进度快照 |
| **Central Executive** | 中央执行（决策控制） | System Prompt + 当前意图 |
| **Episodic Buffer** | 情节缓冲（关联记忆） | 按相关性检索的历史片段 |

```typescript
interface WorkingMemory {
  // 中央执行 — 静态，始终注入
  systemPrompt: string;
  currentIntent: string;

  // 情节缓冲 — 动态，按相关性填充
  relevantHistory: Message[];    // 语义检索得到

  // 视觉空间 — 当前任务快照
  taskState: TaskSnapshot;

  // 语音循环 — 滑动窗口
  recentMessages: Message[];     // 最近 6-10 条

  // 工具结果暂存区 — 临时，用完即丢
  scratchPad: ToolResult[];
}
```

---

## 3. 实现：WorkingMemoryManager

```typescript
// working-memory.ts
import { encodingForModel } from 'tiktoken';

interface WorkingMemoryConfig {
  maxTokens: number;           // 整个工作记忆 token 上限
  slots: {
    system: number;            // 系统提示占比
    taskState: number;         // 任务状态占比
    recentMessages: number;    // 近期对话占比
    episodic: number;          // 情节记忆占比
    scratchPad: number;        // 工具结果占比
  };
}

const DEFAULT_CONFIG: WorkingMemoryConfig = {
  maxTokens: 80_000,  // 留 120K 给 LLM 输出 + 安全余量
  slots: {
    system: 0.10,      // 8K   — 系统提示固定
    taskState: 0.05,   // 4K   — 任务进度快照
    recentMessages: 0.25, // 20K — 近期对话
    episodic: 0.20,    // 16K  — 检索出的相关历史
    scratchPad: 0.40,  // 32K  — 工具结果（最贪婪）
  },
};

export class WorkingMemoryManager {
  private enc = encodingForModel('cl100k_base');

  constructor(private config: WorkingMemoryConfig = DEFAULT_CONFIG) {}

  private countTokens(text: string): number {
    return this.enc.encode(text).length;
  }

  private truncateToTokens(text: string, maxTokens: number): string {
    const tokens = this.enc.encode(text);
    if (tokens.length <= maxTokens) return text;
    // 从末尾截断，保留开头（系统信息更重要）
    return new TextDecoder().decode(
      this.enc.decode(tokens.slice(0, maxTokens))
    );
  }

  /**
   * 组装当前子任务的工作记忆
   */
  assemble(input: {
    systemPrompt: string;
    currentIntent: string;
    taskState: TaskSnapshot;
    allMessages: Message[];
    episodicResults: Message[];   // 语义检索结果
    latestToolResults: ToolResult[];
  }): AssembledContext {
    const budgets = {
      system: Math.floor(this.config.maxTokens * this.config.slots.system),
      taskState: Math.floor(this.config.maxTokens * this.config.slots.taskState),
      recentMessages: Math.floor(this.config.maxTokens * this.config.slots.recentMessages),
      episodic: Math.floor(this.config.maxTokens * this.config.slots.episodic),
      scratchPad: Math.floor(this.config.maxTokens * this.config.slots.scratchPad),
    };

    // 1. 系统提示 — 强制截断（不应该超）
    const systemContent = this.truncateToTokens(
      `${input.systemPrompt}\n\n[当前意图] ${input.currentIntent}`,
      budgets.system
    );

    // 2. 任务状态快照
    const taskStateContent = this.truncateToTokens(
      this.serializeTaskState(input.taskState),
      budgets.taskState
    );

    // 3. 近期消息 — 从最新往前取，不超预算
    const recentMessages = this.selectRecentMessages(
      input.allMessages,
      budgets.recentMessages
    );

    // 4. 情节记忆 — 相关历史，去重
    const recentIds = new Set(recentMessages.map(m => m.id));
    const episodicMessages = this.selectEpisodicMessages(
      input.episodicResults.filter(m => !recentIds.has(m.id)),
      budgets.episodic
    );

    // 5. 工具结果暂存区 — 最新结果优先
    const scratchPadContent = this.selectToolResults(
      input.latestToolResults,
      budgets.scratchPad
    );

    const totalTokens =
      this.countTokens(systemContent) +
      this.countTokens(taskStateContent) +
      recentMessages.reduce((s, m) => s + this.countTokens(m.content), 0) +
      episodicMessages.reduce((s, m) => s + this.countTokens(m.content), 0) +
      this.countTokens(scratchPadContent);

    return {
      systemPrompt: systemContent,
      messages: [
        // 注入任务状态作为隐藏系统消息
        { role: 'system', content: `[任务状态]\n${taskStateContent}` },
        // 情节记忆（有标注，让 LLM 知道这是检索来的）
        ...episodicMessages.map(m => ({
          ...m,
          content: `[相关历史 ${m.timestamp}]\n${m.content}`,
        })),
        // 近期对话
        ...recentMessages,
        // 工具结果
        ...(scratchPadContent
          ? [{ role: 'tool_results' as const, content: scratchPadContent }]
          : []),
      ],
      stats: {
        totalTokens,
        budgetUsed: totalTokens / this.config.maxTokens,
        slots: {
          system: this.countTokens(systemContent),
          taskState: this.countTokens(taskStateContent),
          recentMessages: recentMessages.reduce((s, m) => s + this.countTokens(m.content), 0),
          episodic: episodicMessages.reduce((s, m) => s + this.countTokens(m.content), 0),
          scratchPad: this.countTokens(scratchPadContent),
        },
      },
    };
  }

  private selectRecentMessages(messages: Message[], budget: number): Message[] {
    const selected: Message[] = [];
    let used = 0;
    // 从最新往前
    for (let i = messages.length - 1; i >= 0; i--) {
      const tokens = this.countTokens(messages[i].content);
      if (used + tokens > budget) break;
      selected.unshift(messages[i]);
      used += tokens;
    }
    return selected;
  }

  private selectEpisodicMessages(results: Message[], budget: number): Message[] {
    const selected: Message[] = [];
    let used = 0;
    for (const msg of results) {
      const tokens = this.countTokens(msg.content);
      if (used + tokens > budget) break;
      selected.push(msg);
      used += tokens;
    }
    return selected;
  }

  private selectToolResults(results: ToolResult[], budget: number): string {
    const parts: string[] = [];
    let used = 0;
    for (const r of results.reverse()) {  // 最新的优先
      const text = `[${r.toolName}]\n${JSON.stringify(r.output, null, 2)}`;
      const tokens = this.countTokens(text);
      if (used + tokens > budget) {
        // 尝试截断塞进去
        const remaining = budget - used;
        if (remaining > 100) parts.unshift(this.truncateToTokens(text, remaining));
        break;
      }
      parts.unshift(text);
      used += tokens;
    }
    return parts.join('\n\n---\n\n');
  }

  private serializeTaskState(state: TaskSnapshot): string {
    return [
      `目标: ${state.goal}`,
      `进度: ${state.completedSteps.length}/${state.totalSteps} 步完成`,
      `当前步骤: ${state.currentStep}`,
      state.blockers.length > 0 ? `阻塞: ${state.blockers.join(', ')}` : '',
      state.decisions.length > 0 ? `已决策: ${state.decisions.slice(-3).join('; ')}` : '',
    ].filter(Boolean).join('\n');
  }
}
```

---

## 4. 注意力聚焦：Intent-Scoped Context

不同子任务需要不同的注意力焦点。`IntentRouter` 根据当前意图调整 slot 权重：

```typescript
// intent-attention-router.ts

type Intent =
  | 'code_generation'
  | 'data_analysis'
  | 'conversation'
  | 'debugging'
  | 'planning';

const INTENT_PROFILES: Record<Intent, WorkingMemoryConfig['slots']> = {
  // 写代码：工具结果（代码执行输出）最重要
  code_generation: {
    system: 0.08, taskState: 0.07,
    recentMessages: 0.20, episodic: 0.10, scratchPad: 0.55,
  },
  // 数据分析：历史查询结果 + 近期对话
  data_analysis: {
    system: 0.08, taskState: 0.05,
    recentMessages: 0.25, episodic: 0.15, scratchPad: 0.47,
  },
  // 闲聊：近期对话最重要，工具结果少
  conversation: {
    system: 0.12, taskState: 0.03,
    recentMessages: 0.50, episodic: 0.25, scratchPad: 0.10,
  },
  // 调试：工具执行结果（错误日志）最关键
  debugging: {
    system: 0.08, taskState: 0.07,
    recentMessages: 0.15, episodic: 0.10, scratchPad: 0.60,
  },
  // 规划：历史决策 + 任务状态最重要
  planning: {
    system: 0.10, taskState: 0.15,
    recentMessages: 0.25, episodic: 0.35, scratchPad: 0.15,
  },
};

export function getContextConfigForIntent(intent: Intent): WorkingMemoryConfig {
  return {
    maxTokens: 80_000,
    slots: INTENT_PROFILES[intent] ?? INTENT_PROFILES.conversation,
  };
}

// 使用示例
async function runAgentTurn(session: AgentSession, userMessage: string) {
  // 1. 识别意图
  const intent = await classifyIntent(userMessage);

  // 2. 获取对应的注意力配置
  const config = getContextConfigForIntent(intent);
  const wm = new WorkingMemoryManager(config);

  // 3. 检索情节记忆（语义相关历史）
  const episodic = await memoryStore.semanticSearch(userMessage, { limit: 10 });

  // 4. 组装工作记忆
  const ctx = wm.assemble({
    systemPrompt: session.systemPrompt,
    currentIntent: `${intent}: ${userMessage}`,
    taskState: session.taskState,
    allMessages: session.messages,
    episodicResults: episodic,
    latestToolResults: session.pendingToolResults,
  });

  console.log(`[WorkingMemory] 意图=${intent} token使用率=${(ctx.stats.budgetUsed * 100).toFixed(1)}%`);

  // 5. 调用 LLM
  return await llm.chat(ctx.systemPrompt, ctx.messages);
}
```

---

## 5. 工作记忆的"遗忘"策略

与其无限积累 scratchPad，不如主动遗忘：

```typescript
// forget-strategy.ts

export class ScratchPadManager {
  private results: Map<string, { result: ToolResult; accessCount: number; lastUsed: number }> = new Map();

  add(result: ToolResult) {
    this.results.set(result.callId, {
      result,
      accessCount: 0,
      lastUsed: Date.now(),
    });
  }

  /**
   * 获取当前活跃结果（遗忘过期/低价值的）
   */
  getActive(options: {
    maxAge?: number;       // 超过 N ms 自动遗忘
    maxCount?: number;     // 最多保留 N 个
    requiredIds?: string[]; // 强制保留指定结果
  } = {}): ToolResult[] {
    const now = Date.now();
    const { maxAge = 300_000, maxCount = 20, requiredIds = [] } = options;

    return [...this.results.values()]
      .filter(item => {
        if (requiredIds.includes(item.result.callId)) return true;  // 强制保留
        if (now - item.lastUsed > maxAge) return false;             // 太旧
        return true;
      })
      .sort((a, b) => {
        // 优先保留：访问频繁 + 最近使用
        const scoreA = a.accessCount * 0.3 + (now - a.lastUsed) * -0.7;
        const scoreB = b.accessCount * 0.3 + (now - b.lastUsed) * -0.7;
        return scoreB - scoreA;
      })
      .slice(0, maxCount)
      .map(item => {
        item.accessCount++;
        item.lastUsed = now;
        return item.result;
      });
  }

  /**
   * LLM 明确引用了某个工具结果 → 更新访问计数
   */
  markAccessed(callId: string) {
    const item = this.results.get(callId);
    if (item) {
      item.accessCount++;
      item.lastUsed = Date.now();
    }
  }

  /**
   * 任务完成后清空（对应人类"任务完成后清空工作台"）
   */
  clear() {
    this.results.clear();
  }
}
```

---

## 6. OpenClaw 实战：工作记忆监控

在 OpenClaw 的 Heartbeat 中监控工作记忆使用情况：

```typescript
// heartbeat-wm-monitor.ts
// 注入 HEARTBEAT.md 或作为 Cron job

import { WorkingMemoryManager } from './working-memory';

export async function workingMemoryHealthCheck(session: AgentSession) {
  const wm = new WorkingMemoryManager();
  const ctx = wm.assemble({
    systemPrompt: session.systemPrompt,
    currentIntent: 'heartbeat_check',
    taskState: session.taskState,
    allMessages: session.messages,
    episodicResults: [],
    latestToolResults: [],
  });

  const stats = ctx.stats;

  // 告警阈值
  if (stats.budgetUsed > 0.85) {
    console.warn(`⚠️ 工作记忆使用率 ${(stats.budgetUsed * 100).toFixed(0)}% — 考虑压缩历史`);

    // 触发 Context Distillation
    await triggerContextDistillation(session);
  }

  // 打印 slot 分布
  console.log(`
📊 工作记忆快照 (${stats.totalTokens.toLocaleString()} tokens)
  ├ System:   ${stats.slots.system.toLocaleString()} tokens
  ├ TaskState: ${stats.slots.taskState.toLocaleString()} tokens
  ├ Recent:   ${stats.slots.recentMessages.toLocaleString()} tokens
  ├ Episodic: ${stats.slots.episodic.toLocaleString()} tokens
  └ ScratchPad: ${stats.slots.scratchPad.toLocaleString()} tokens
  `);
}
```

---

## 7. pi-mono 中的工作记忆

pi-mono 的 `ContextManager` 实现了类似机制：

```typescript
// pi-mono/packages/agent/src/context/ContextManager.ts（概念示意）

export class ContextManager {
  // pi-mono 使用 "focus" 概念表示当前子任务的注意力范围
  private focusStack: FocusFrame[] = [];

  /**
   * 进入子任务时 push 新的 focus frame
   * 子任务只能看到自己的 frame + 父 frame 的摘要
   */
  pushFocus(subTaskGoal: string, relevantContext: string[]) {
    this.focusStack.push({
      goal: subTaskGoal,
      context: relevantContext,
      toolResults: [],
      startedAt: Date.now(),
    });
  }

  /**
   * 子任务完成后 pop，把结果摘要注入父 frame
   */
  popFocus(): FocusFrame {
    const frame = this.focusStack.pop()!;
    const parent = this.focusStack[this.focusStack.length - 1];
    if (parent) {
      // 子任务结果以摘要形式注入父任务
      parent.context.push(`[子任务完成: ${frame.goal}]\n${frame.summary}`);
    }
    return frame;
  }

  /**
   * 当前 LLM 调用只看到栈顶的 focus
   */
  getCurrentContext(): string[] {
    const current = this.focusStack[this.focusStack.length - 1];
    return current?.context ?? [];
  }
}
```

---

## 8. 与其他模式的关系

```
Context Window Management (通用)
       ↓
 ┌─────────────────────────────────┐
 │  Working Memory Management      │  ← 本课（结构化分槽管理）
 │  ├─ Intent-Scoped Attention     │  ← 按意图调权重
 │  ├─ Scratchpad Forgetting       │  ← 主动遗忘
 │  └─ Focus Stack (pi-mono)       │  ← 子任务隔离
 └─────────────────────────────────┘
       ↓
 Sliding Window + Summarization    ← 当 recentMessages 槽满时触发
 Context Distillation              ← 当总 budget 超 85% 时触发
 Episodic Memory (RAG)             ← 填充 episodic 槽的来源
```

---

## 9. 关键指标

| 指标 | 健康范围 | 告警阈值 |
|------|---------|---------|
| 工作记忆使用率 | < 70% | > 85% |
| ScratchPad 占比 | < 50% | > 70% |
| 遗忘率（过期工具结果比） | > 30% | < 10%（可能泄漏） |
| 意图识别准确率 | > 90% | < 80% |

---

## 总结

| 概念 | 一句话 |
|------|--------|
| 工作记忆 | 每次 LLM 调用前精确组装的 context 子集 |
| 四槽模型 | System / TaskState / Recent / Episodic / ScratchPad |
| 注意力聚焦 | 按意图调整 slot 权重（写代码 vs 闲聊权重完全不同） |
| 主动遗忘 | 工具结果按使用频率 + 时间衰减淘汰 |
| Focus Stack | 子任务只看自己的 frame，避免跨任务污染 |

> **记住：** context 不是越多越好，聚焦才能让 LLM 发挥最大效能。工作记忆管理就是在"信息量"和"注意力集中"之间找到最优平衡点。
