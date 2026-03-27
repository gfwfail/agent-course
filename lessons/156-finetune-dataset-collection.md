# Agent 微调数据集自动收集（Automatic Fine-tuning Dataset Collection）

> 你花几十美元跑 Claude，实际上每次对话都是一份训练数据。这一课讲如何从生产流量中自动收集高质量样本，用来微调更便宜的小模型，长期省钱。

## 核心思路

```
生产 Agent (Claude) → 拦截高质量对话 → JSONL 数据集 → 微调小模型 (GPT-4o-mini / Llama)
```

大模型做"老师"，生产流量做"课本"，微调出专用小模型处理重复性任务。
成本可以降低 10x～100x。

---

## 什么样的对话值得收集？

不是所有对话都值得收进训练集，要过滤：

| 质量信号 | 说明 |
|---------|------|
| ✅ 用户没有追问/纠正 | 说明 Agent 第一次就答对了 |
| ✅ 工具调用成功 | 没有 tool error，没有重试 |
| ✅ 对话轮数 ≤ 3 | 简洁完成，不是反复拉扯 |
| ✅ Token 数合理 | 不是超长的边缘 case |
| ❌ 用户说"不对"/"重新来" | 质量差，跳过 |
| ❌ 有 tool error | 跳过 |
| ❌ 超时或重试 | 跳过 |

---

## 实现：收集中间件

```typescript
// collect-middleware.ts
import fs from 'fs';
import path from 'path';

interface ConversationSample {
  messages: { role: string; content: string }[];
  tools_used: string[];
  quality_score: number;
  collected_at: string;
  model: string;
}

interface TurnMetrics {
  hasToolError: boolean;
  hasUserCorrection: boolean;
  turnCount: number;
  totalTokens: number;
  toolsUsed: string[];
}

// 质量评分：0-1，>0.7 才收集
function scoreConversation(metrics: TurnMetrics): number {
  let score = 1.0;

  if (metrics.hasToolError) score -= 0.5;
  if (metrics.hasUserCorrection) score -= 0.4;
  if (metrics.turnCount > 5) score -= 0.2;
  if (metrics.totalTokens > 8000) score -= 0.1;

  return Math.max(0, score);
}

// 检测用户是否在纠正 Agent
function detectUserCorrection(content: string): boolean {
  const correctionPatterns = [
    /不对|错了|重新|你理解错了|不是这个意思/,
    /wrong|incorrect|no,? that's not|try again/i,
  ];
  return correctionPatterns.some(p => p.test(content));
}

export class DatasetCollector {
  private outputPath: string;
  private minQualityScore: number;

  constructor(outputPath: string, minQualityScore = 0.7) {
    this.outputPath = outputPath;
    this.minQualityScore = minQualityScore;
    // 确保输出目录存在
    fs.mkdirSync(path.dirname(outputPath), { recursive: true });
  }

  async collectIfWorthy(
    messages: { role: string; content: string }[],
    model: string,
    toolResults: { tool: string; error?: string }[]
  ): Promise<void> {
    const metrics: TurnMetrics = {
      hasToolError: toolResults.some(r => !!r.error),
      hasUserCorrection: messages
        .filter(m => m.role === 'user')
        .some(m => detectUserCorrection(m.content)),
      turnCount: messages.filter(m => m.role === 'user').length,
      totalTokens: messages.reduce((sum, m) => sum + m.content.length / 4, 0),
      toolsUsed: [...new Set(toolResults.map(r => r.tool))],
    };

    const score = scoreConversation(metrics);

    if (score < this.minQualityScore) {
      console.log(`[Collector] skip, score=${score.toFixed(2)}`);
      return;
    }

    const sample: ConversationSample = {
      messages,
      tools_used: metrics.toolsUsed,
      quality_score: score,
      collected_at: new Date().toISOString(),
      model,
    };

    // 追加写入 JSONL（每行一个样本）
    fs.appendFileSync(this.outputPath, JSON.stringify(sample) + '\n', 'utf8');
    console.log(`[Collector] saved sample, score=${score.toFixed(2)}`);
  }
}
```

---

## 与 Agent Loop 集成

在 pi-mono 的 `runAgent` 结束时挂钩：

```typescript
// agent-loop.ts (简化版)
import { DatasetCollector } from './collect-middleware';

const collector = new DatasetCollector(
  './datasets/production.jsonl',
  0.7
);

async function runAgent(userMessage: string) {
  const messages: Message[] = [];
  const toolResults: ToolResult[] = [];

  // ... 正常 Agent Loop ...

  // ✅ Loop 结束后收集
  await collector.collectIfWorthy(
    messages,
    'claude-sonnet-4-6',
    toolResults
  );
}
```

---

## 转换为微调格式

不同平台要求不同格式，写个转换器：

```typescript
// convert-dataset.ts
import fs from 'fs';

interface RawSample {
  messages: { role: string; content: string }[];
  quality_score: number;
}

// OpenAI fine-tuning 格式
function toOpenAIFormat(sample: RawSample) {
  return {
    messages: sample.messages.map(m => ({
      role: m.role === 'assistant' ? 'assistant' : m.role,
      content: m.content,
    })),
  };
}

// Anthropic fine-tuning 格式（messages API）
function toAnthropicFormat(sample: RawSample) {
  const system = sample.messages.find(m => m.role === 'system')?.content;
  const turns = sample.messages.filter(m => m.role !== 'system');
  return { system, messages: turns };
}

function convertDataset(
  inputPath: string,
  outputPath: string,
  format: 'openai' | 'anthropic',
  minScore = 0.8  // 转换时再提高阈值
) {
  const lines = fs.readFileSync(inputPath, 'utf8').trim().split('\n');
  const converted: string[] = [];

  for (const line of lines) {
    const sample: RawSample = JSON.parse(line);
    if (sample.quality_score < minScore) continue;

    const out = format === 'openai'
      ? toOpenAIFormat(sample)
      : toAnthropicFormat(sample);

    converted.push(JSON.stringify(out));
  }

  fs.writeFileSync(outputPath, converted.join('\n'), 'utf8');
  console.log(`Converted ${converted.length}/${lines.length} samples`);
}

// 使用
convertDataset(
  './datasets/production.jsonl',
  './datasets/openai-finetune.jsonl',
  'openai',
  0.8
);
```

---

## OpenClaw Cron：每日自动导出数据集

```typescript
// cron job: 每天凌晨 2 点导出前一天数据
{
  schedule: { kind: "cron", expr: "0 2 * * *", tz: "Australia/Sydney" },
  payload: {
    kind: "agentTurn",
    message: `
      读取 ./datasets/production.jsonl，
      过滤出昨天（${new Date().toISOString().slice(0,10)}）的样本，
      质量分 >= 0.8 的转换成 OpenAI 格式，
      上传到 R2 bucket: clawegg-email/finetune/{date}.jsonl，
      输出统计：总样本数、高质量样本数、覆盖的工具列表。
    `
  },
  sessionTarget: "isolated"
}
```

---

## 数据飞轮效果

```
第 1 周：收集 500 个高质量样本
第 2 周：微调 gpt-4o-mini，处理 80% 简单查询
第 3 周：Claude 只处理复杂任务，成本降低 70%
第 4 周：继续收集，定期重新微调...
```

微调小模型不需要很多数据：
- 单一任务（如订单查询）：**100-500 样本**就够
- 多任务通用：**1000-5000 样本**

---

## 隐私与合规

```typescript
// 收集前脱敏
import { maskPII } from './pii-masker'; // 参考第 123 课

async function collectIfWorthy(messages, ...) {
  // 脱敏后再写入磁盘
  const sanitized = messages.map(m => ({
    ...m,
    content: maskPII(m.content)
  }));
  // ... 写入 sanitized 数据
}
```

- 用户数据上 Dataset 前必须脱敏（参考 Lesson 123 PII 脱敏）
- 在隐私政策中告知用户"对话可能用于改进服务"
- 给用户提供 opt-out 选项

---

## 与已讲内容的关系

| 课程 | 关联 |
|------|------|
| Lesson 123 - PII 脱敏 | 数据收集前必须脱敏 |
| Lesson 13 - Tool Results Processing | 工具结果是质量判断的依据 |
| Lesson 8 - Observability | 用 span 数据辅助质量评分 |
| Lesson 151 - Context-Aware Rate Limiting | 控制数据收集的写入速率 |

---

## 总结

**一句话记住：** 生产流量 = 免费训练数据，大模型做老师，小模型省钱跑。

**核心步骤：**
1. Agent Loop 末尾加收集钩子
2. 质量评分过滤（有没有报错、用户有没有纠正）
3. 追加写 JSONL，保留质量分字段
4. 定期转换格式，上传，触发微调
5. 小模型上线 → 大模型只处理复杂任务 → 成本大幅下降

> 越大的系统，这个飞轮转得越快。
