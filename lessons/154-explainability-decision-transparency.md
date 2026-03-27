# 154 - Agent 可解释性与决策透明化（Explainability & Decision Transparency）

> "我不知道 AI 为什么这么做，所以我不敢用它。" —— 每个准备把 Agent 推向生产的产品经理

## 为什么需要可解释性？

当 Agent 执行了一个关键操作，用户或运营团队问：

> "它为什么这么判断？"  
> "数据从哪来的？"  
> "这个决定我能信任吗？"

如果回答不上来，信任就断了。

Agent 可解释性不是"让 AI 解释自己"，而是**在 Agent 架构层面主动设计决策追踪、意图标注、输出归因**，让每一步都有据可查。

---

## 三个维度

```
┌────────────────────────────────────────┐
│         可解释性三维度                   │
│                                        │
│  1. 决策透明  → 为什么选这个工具/模型？  │
│  2. 数据溯源  → 这个答案从哪里来的？     │
│  3. 置信标注  → AI 有多确定？           │
└────────────────────────────────────────┘
```

---

## 实现模式：决策追踪器

### TypeScript 实现

```typescript
// explainability/decision-tracker.ts

export interface DecisionRecord {
  id: string;
  timestamp: number;
  sessionId: string;
  
  // 意图层
  userIntent: string;           // LLM 解析的用户意图
  intentConfidence: number;     // 0-1
  
  // 工具选择层
  toolCandidates: ToolCandidate[];
  selectedTool: string;
  selectionReason: string;      // LLM 给出的选择理由
  
  // 执行层
  toolInput: Record<string, unknown>;
  toolOutput: unknown;
  executionMs: number;
  
  // 输出层
  responseSnippet: string;      // 最终回复摘要
  sourcesUsed: string[];        // 引用了哪些数据源
}

export interface ToolCandidate {
  name: string;
  score: number;        // 相关性分数
  reason: string;       // 为什么考虑这个工具
  rejected: boolean;    // 是否被排除
  rejectionReason?: string;
}

export class DecisionTracker {
  private records: Map<string, DecisionRecord[]> = new Map();
  
  startDecision(sessionId: string, userIntent: string): string {
    const id = `dec_${Date.now()}_${Math.random().toString(36).slice(2, 8)}`;
    const record: DecisionRecord = {
      id,
      timestamp: Date.now(),
      sessionId,
      userIntent,
      intentConfidence: 0,
      toolCandidates: [],
      selectedTool: '',
      selectionReason: '',
      toolInput: {},
      toolOutput: null,
      executionMs: 0,
      responseSnippet: '',
      sourcesUsed: [],
    };
    
    if (!this.records.has(sessionId)) {
      this.records.set(sessionId, []);
    }
    this.records.get(sessionId)!.push(record);
    return id;
  }
  
  recordToolSelection(
    sessionId: string,
    decisionId: string,
    candidates: ToolCandidate[],
    selected: string,
    reason: string
  ) {
    const record = this.findRecord(sessionId, decisionId);
    if (record) {
      record.toolCandidates = candidates;
      record.selectedTool = selected;
      record.selectionReason = reason;
    }
  }
  
  recordExecution(
    sessionId: string,
    decisionId: string,
    input: Record<string, unknown>,
    output: unknown,
    ms: number
  ) {
    const record = this.findRecord(sessionId, decisionId);
    if (record) {
      record.toolInput = input;
      record.toolOutput = output;
      record.executionMs = ms;
    }
  }
  
  recordResponse(
    sessionId: string,
    decisionId: string,
    snippet: string,
    sources: string[]
  ) {
    const record = this.findRecord(sessionId, decisionId);
    if (record) {
      record.responseSnippet = snippet;
      record.sourcesUsed = sources;
    }
  }
  
  // 生成可读的解释
  explain(sessionId: string, decisionId: string): string {
    const record = this.findRecord(sessionId, decisionId);
    if (!record) return '找不到该决策记录';
    
    const lines: string[] = [
      `📋 决策解释 [${record.id}]`,
      ``,
      `**用户意图：** ${record.userIntent}`,
      ``,
      `**工具选择：**`,
      `  ✅ 选择了：${record.selectedTool}`,
      `  理由：${record.selectionReason}`,
    ];
    
    const rejected = record.toolCandidates.filter(c => c.rejected);
    if (rejected.length > 0) {
      lines.push(`  ❌ 排除了：${rejected.map(c => `${c.name}（${c.rejectionReason}）`).join('、')}`);
    }
    
    lines.push(``);
    lines.push(`**数据来源：** ${record.sourcesUsed.join('、') || '无外部来源'}`);
    lines.push(`**执行耗时：** ${record.executionMs}ms`);
    
    return lines.join('\n');
  }
  
  getHistory(sessionId: string): DecisionRecord[] {
    return this.records.get(sessionId) ?? [];
  }
  
  private findRecord(sessionId: string, id: string) {
    return this.records.get(sessionId)?.find(r => r.id === id);
  }
}
```

---

## 与 Agent Loop 集成

```typescript
// agent/explainable-agent.ts
import Anthropic from '@anthropic-ai/sdk';

const client = new Anthropic();
const tracker = new DecisionTracker();

async function runExplainableAgent(
  sessionId: string,
  userMessage: string,
  tools: Anthropic.Tool[]
) {
  const decisionId = tracker.startDecision(sessionId, userMessage);
  
  // 在 system prompt 中要求 LLM 标注决策理由
  const systemPrompt = `你是一个高度透明的 AI 助理。
每次调用工具时，必须先输出一段 <reasoning> 标签，解释：
1. 为什么选择这个工具（而不是其他工具）
2. 这个工具的输出将如何用于最终回答

格式示例：
<reasoning>
选择 search_web 而非 read_file，因为用户问的是实时股价，需要最新数据。
</reasoning>`;
  
  const response = await client.messages.create({
    model: 'claude-opus-4-5',
    max_tokens: 4096,
    system: systemPrompt,
    messages: [{ role: 'user', content: userMessage }],
    tools,
  });
  
  // 解析决策理由
  for (const block of response.content) {
    if (block.type === 'text' && block.text.includes('<reasoning>')) {
      const reasoningMatch = block.text.match(/<reasoning>([\s\S]*?)<\/reasoning>/);
      if (reasoningMatch) {
        const reason = reasoningMatch[1].trim();
        tracker.recordToolSelection(sessionId, decisionId, [], '', reason);
      }
    }
    
    if (block.type === 'tool_use') {
      const start = Date.now();
      // 执行工具...
      const output = await executeToolMock(block.name, block.input);
      tracker.recordExecution(sessionId, decisionId, block.input as Record<string, unknown>, output, Date.now() - start);
      
      // 记录工具选择细节
      tracker.recordToolSelection(
        sessionId, decisionId,
        buildCandidates(tools, block.name),
        block.name,
        `LLM 判断 ${block.name} 最适合处理该请求`
      );
    }
  }
  
  return { decisionId, explanation: tracker.explain(sessionId, decisionId) };
}

// 构建候选工具列表（含被排除的）
function buildCandidates(tools: Anthropic.Tool[], selected: string): ToolCandidate[] {
  return tools.map(t => ({
    name: t.name,
    score: t.name === selected ? 1.0 : 0.3,
    reason: t.description ?? '',
    rejected: t.name !== selected,
    rejectionReason: t.name !== selected ? '当前任务不需要此工具' : undefined,
  }));
}

async function executeToolMock(name: string, input: unknown) {
  // 实际项目中这里分发真实工具调用
  return { result: `${name} executed`, input };
}
```

---

## 置信度标注：让 Agent 说"我不确定"

```typescript
// explainability/confidence-annotator.ts

export interface AnnotatedResponse {
  content: string;
  confidence: 'high' | 'medium' | 'low';
  confidenceScore: number;   // 0-1
  caveat?: string;           // 低置信度时的提示
  sources: string[];
}

async function generateWithConfidence(
  question: string
): Promise<AnnotatedResponse> {
  const prompt = `回答以下问题，并在末尾用 JSON 格式给出评估：

问题：${question}

在回答末尾附上：
\`\`\`json
{
  "confidence": "high|medium|low",
  "confidenceScore": 0.0-1.0,
  "caveat": "如果置信度低，说明不确定的原因",
  "sources": ["数据来源1", "数据来源2"]
}
\`\`\``;

  const response = await client.messages.create({
    model: 'claude-opus-4-5',
    max_tokens: 2048,
    messages: [{ role: 'user', content: prompt }],
  });
  
  const text = response.content[0].type === 'text' ? response.content[0].text : '';
  
  // 提取 JSON 元数据
  const jsonMatch = text.match(/```json\n([\s\S]*?)\n```/);
  let meta = { confidence: 'medium' as const, confidenceScore: 0.5, sources: [] as string[] };
  
  if (jsonMatch) {
    try {
      meta = { ...meta, ...JSON.parse(jsonMatch[1]) };
    } catch {}
  }
  
  const content = text.replace(/```json[\s\S]*?```/, '').trim();
  
  return { content, ...meta };
}

// 使用示例
async function demo() {
  const result = await generateWithConfidence('2024年最新的 GPT-5 价格是多少？');
  
  console.log('回答：', result.content);
  console.log(`置信度：${result.confidence}（${(result.confidenceScore * 100).toFixed(0)}%）`);
  
  if (result.confidence === 'low') {
    console.log('⚠️ 注意：', result.caveat);
  }
  
  if (result.sources.length > 0) {
    console.log('来源：', result.sources.join(', '));
  }
}
```

---

## OpenClaw 实战：透明化 Heartbeat 决策

```typescript
// OpenClaw heartbeat 中，每次决策都记录为可审查的事件

// HEARTBEAT.md 中记录决策摘要：
// - 为什么发送了告警（而不是静默）
// - 使用了哪些数据源（邮件/日历/监控）
// - 置信度评估

// 实现方式：在 heartbeat 响应中附加 <decision> 块
const heartbeatSystem = `
你是一个 OpenClaw Heartbeat Agent。
每次检查后，用 <decision> 块记录你的决策逻辑：

<decision>
{
  "checked": ["email", "calendar"],
  "action": "send_alert|silent|HEARTBEAT_OK",
  "reason": "发现1封未读紧急邮件，触发告警",
  "confidence": "high",
  "skipped": ["weather - 非工作日不检查"]
}
</decision>
`;
```

---

## pi-mono 中的可解释性钩子

```typescript
// pi-mono/src/agent/explainable-middleware.ts

import { ToolMiddleware } from './types';

// 工具调用中间件：记录每次决策的完整上下文快照
export const explainabilityMiddleware: ToolMiddleware = async (
  toolName,
  params,
  next
) => {
  const start = Date.now();
  const decisionContext = {
    tool: toolName,
    params,
    timestamp: new Date().toISOString(),
    // 捕获调用栈中的 reasoning（从 AsyncLocalStorage 传播）
    reasoning: AsyncLocalStorage.getStore()?.currentReasoning ?? 'N/A',
  };
  
  try {
    const result = await next(toolName, params);
    
    // 成功：记录完整决策
    await auditLog.append({
      ...decisionContext,
      success: true,
      durationMs: Date.now() - start,
      outputSummary: summarize(result),
    });
    
    return result;
  } catch (err) {
    // 失败：记录失败原因，方便回溯
    await auditLog.append({
      ...decisionContext,
      success: false,
      durationMs: Date.now() - start,
      error: String(err),
    });
    throw err;
  }
};

function summarize(output: unknown): string {
  const str = JSON.stringify(output);
  return str.length > 200 ? str.slice(0, 200) + '...' : str;
}
```

---

## 用户侧透明化："/explain" 指令

```typescript
// 给用户提供随时可查的解释接口

async function handleExplainCommand(sessionId: string): Promise<string> {
  const history = tracker.getHistory(sessionId);
  
  if (history.length === 0) {
    return '本次会话暂无决策记录。';
  }
  
  const last = history[history.length - 1];
  return tracker.explain(sessionId, last.id);
}

// 注册为 Agent 工具
const explainTool: Anthropic.Tool = {
  name: 'explain_last_decision',
  description: '解释 Agent 上一次决策的完整逻辑，包括工具选择原因、数据来源和置信度',
  input_schema: {
    type: 'object',
    properties: {},
  },
};
```

---

## 设计原则

| 原则 | 做法 |
|------|------|
| **主动标注** | 不是事后解释，而是执行时同步记录 |
| **分层粒度** | 用户看摘要，运维看详细，审计看原始日志 |
| **不影响性能** | 记录是异步写入，不阻塞主流程 |
| **可查可导出** | 支持 `/explain` 命令和批量导出 |
| **承认不确定** | 置信度低时主动告知用户 |

---

## 避坑清单

- ❌ 不要让 LLM "事后编造"解释 —— 记录要在执行时生成
- ❌ 不要记录敏感参数（密码、token）—— 先脱敏再记录
- ❌ 不要每次都让用户看完整日志 —— 分层展示
- ✅ 置信度低时主动提示，不要假装百分百确定
- ✅ 工具被拒绝的原因也要记录，帮助调试

---

## 小结

可解释性不是"让 AI 更聪明"，而是**让人更信任 AI**：

1. **决策追踪器** —— 记录每次工具选择的完整上下文
2. **置信度标注** —— 让 Agent 主动说"我不确定"
3. **分层展示** —— 用户看结论，工程师看过程，审计看原始
4. **"/explain" 指令** —— 随时可查，随时可审

> 透明的 Agent 不是弱的 Agent，是可以被信任的 Agent。
