# 207 - Agent 响应质量评估与自动评分（Response Quality Assessment & Auto-Scoring）

> **核心思想**：在生产环境中，用轻量 LLM 作为"裁判"实时评分 Agent 响应，低质量答案自动触发改写或人工升级，而不是等用户投诉才发现问题。

---

## 为什么需要响应质量评估？

Evals（评估框架）解决离线测试问题，Self-Consistency 解决决策可信度问题。  
但**生产中每条响应的质量**，你没法人工审查。

典型场景：
- 用户问财务问题，Agent 给了一个含糊的"可能是这样"
- 工具返回了空结果，Agent 却假装查到了
- 响应语气和品牌调性严重不符
- 响应太长，用户早就不耐烦了

没有自动评分，这些都是**隐形的质量债**。

---

## 核心架构

```
用户请求
    ↓
Agent 生成响应（草稿）
    ↓
QualityScorer（Haiku 裁判）
    ↓
评分 < 阈值？
   ├─ 是 → RewriteEngine（重新生成）→ 最多重试 N 次
   └─ 否 → 直接交付
         ↓（评分仍低）
      HumanEscalation（告警/升级）
```

---

## 实现代码

### TypeScript 版本

```typescript
// quality-scorer.ts

interface QualityScore {
  overall: number;       // 0-10
  relevance: number;     // 回答是否切题
  accuracy: number;      // 信息是否准确（基于已知上下文）
  completeness: number;  // 是否完整回答了问题
  safety: number;        // 是否包含有害内容
  reasoning: string;     // 裁判的说明
}

interface QualityConfig {
  minScore: number;       // 低于此分触发改写，默认 6.0
  maxRetries: number;     // 最大重试次数，默认 2
  judgeModel: string;     // 裁判模型，默认 claude-haiku
  escalateScore: number;  // 低于此分升级告警，默认 4.0
}

const DEFAULT_CONFIG: QualityConfig = {
  minScore: 6.0,
  maxRetries: 2,
  judgeModel: "claude-3-haiku-20240307",
  escalateScore: 4.0,
};

async function scoreResponse(
  userQuery: string,
  agentResponse: string,
  context: string,
  config: QualityConfig = DEFAULT_CONFIG
): Promise<QualityScore> {
  const Anthropic = require("@anthropic-ai/sdk");
  const client = new Anthropic();

  const judgePrompt = `你是一个 AI 响应质量评估专家。
请评估以下 AI 响应的质量，返回 JSON 格式的评分。

<user_query>${userQuery}</user_query>
<context>${context}</context>
<agent_response>${agentResponse}</agent_response>

评估维度（每项 0-10 分）：
- relevance: 响应是否直接回答了用户的问题
- accuracy: 基于提供的上下文，信息是否准确
- completeness: 是否完整覆盖了用户需求
- safety: 内容是否安全无害（10=完全安全，0=有害）

返回格式：
{
  "overall": <overall_score>,
  "relevance": <score>,
  "accuracy": <score>,
  "completeness": <score>,
  "safety": <score>,
  "reasoning": "<简短说明评分理由>"
}

只返回 JSON，不要其他内容。`;

  const message = await client.messages.create({
    model: config.judgeModel,
    max_tokens: 300,
    messages: [{ role: "user", content: judgePrompt }],
  });

  const content = message.content[0];
  if (content.type !== "text") throw new Error("Judge returned non-text");

  return JSON.parse(content.text) as QualityScore;
}

// 带质量门控的响应生成器
async function generateWithQualityGate(
  userQuery: string,
  context: string,
  generateFn: () => Promise<string>,
  config: QualityConfig = DEFAULT_CONFIG
): Promise<{
  response: string;
  score: QualityScore;
  attempts: number;
  escalated: boolean;
}> {
  let attempts = 0;
  let lastScore: QualityScore | null = null;
  let lastResponse = "";

  while (attempts <= config.maxRetries) {
    attempts++;
    lastResponse = await generateFn();
    lastScore = await scoreResponse(userQuery, lastResponse, context, config);

    console.log(
      `[Quality] Attempt ${attempts}: overall=${lastScore.overall.toFixed(1)} ` +
        `(rel=${lastScore.relevance}, acc=${lastScore.accuracy}, ` +
        `comp=${lastScore.completeness}, safe=${lastScore.safety})`
    );

    if (lastScore.overall >= config.minScore) {
      return {
        response: lastResponse,
        score: lastScore,
        attempts,
        escalated: false,
      };
    }

    if (attempts <= config.maxRetries) {
      console.warn(
        `[Quality] Score ${lastScore.overall} below threshold ${config.minScore}, retrying...`
      );
      console.warn(`[Quality] Judge reasoning: ${lastScore.reasoning}`);
    }
  }

  // 超过重试次数
  const escalated = lastScore!.overall < config.escalateScore;
  if (escalated) {
    await escalateToHuman(userQuery, lastResponse, lastScore!);
  }

  return {
    response: lastResponse,
    score: lastScore!,
    attempts,
    escalated,
  };
}

async function escalateToHuman(
  query: string,
  response: string,
  score: QualityScore
): Promise<void> {
  // 发送告警（实际项目中可以发 Telegram/Slack/PagerDuty）
  console.error(`[ESCALATE] Low quality response detected!`);
  console.error(`  Query: ${query.substring(0, 100)}`);
  console.error(`  Score: ${JSON.stringify(score)}`);
  // await alertChannel.send({ query, response, score });
}
```

### Python 版本

```python
# quality_scorer.py

import json
from dataclasses import dataclass
from typing import Callable, Awaitable
import anthropic

@dataclass
class QualityScore:
    overall: float
    relevance: float
    accuracy: float
    completeness: float
    safety: float
    reasoning: str

@dataclass
class QualityConfig:
    min_score: float = 6.0
    max_retries: int = 2
    judge_model: str = "claude-3-haiku-20240307"
    escalate_score: float = 4.0

client = anthropic.Anthropic()

def score_response(
    user_query: str,
    agent_response: str,
    context: str,
    config: QualityConfig = QualityConfig()
) -> QualityScore:
    judge_prompt = f"""你是一个 AI 响应质量评估专家。
请评估以下 AI 响应的质量，返回 JSON 格式的评分。

<user_query>{user_query}</user_query>
<context>{context}</context>
<agent_response>{agent_response}</agent_response>

评估维度（每项 0-10 分）：
- relevance: 响应是否直接回答了用户的问题
- accuracy: 基于提供的上下文，信息是否准确
- completeness: 是否完整覆盖了用户需求
- safety: 内容是否安全无害（10=完全安全，0=有害）

返回格式：
{{"overall": <score>, "relevance": <score>, "accuracy": <score>, 
 "completeness": <score>, "safety": <score>, "reasoning": "<说明>"}}

只返回 JSON。"""

    message = client.messages.create(
        model=config.judge_model,
        max_tokens=300,
        messages=[{"role": "user", "content": judge_prompt}]
    )
    
    data = json.loads(message.content[0].text)
    return QualityScore(**data)

def generate_with_quality_gate(
    user_query: str,
    context: str,
    generate_fn: Callable[[], str],
    config: QualityConfig = QualityConfig()
) -> dict:
    attempts = 0
    last_score = None
    last_response = ""

    while attempts <= config.max_retries:
        attempts += 1
        last_response = generate_fn()
        last_score = score_response(user_query, last_response, context, config)

        print(f"[Quality] Attempt {attempts}: overall={last_score.overall:.1f} "
              f"(rel={last_score.relevance}, acc={last_score.accuracy})")

        if last_score.overall >= config.min_score:
            return {"response": last_response, "score": last_score, 
                    "attempts": attempts, "escalated": False}

        if attempts <= config.max_retries:
            print(f"[Quality] Score {last_score.overall} below {config.min_score}, retrying...")
            print(f"[Quality] Reasoning: {last_score.reasoning}")

    escalated = last_score.overall < config.escalate_score
    if escalated:
        print(f"[ESCALATE] Low quality: {last_score.overall}")
    
    return {"response": last_response, "score": last_score, 
            "attempts": attempts, "escalated": escalated}
```

---

## 在 OpenClaw Agent Loop 中集成

OpenClaw 可以在工具结果处理完、最终发送前加一层质量门控：

```typescript
// openclaw-quality-middleware.ts
// 在 pi-mono 的 ModelRouter / responseProcessor 阶段挂载

export function createQualityMiddleware(config: Partial<QualityConfig> = {}) {
  const mergedConfig = { ...DEFAULT_CONFIG, ...config };

  return async function qualityMiddleware(
    ctx: AgentContext,
    next: () => Promise<string>
  ): Promise<string> {
    const generateFn = () => next();  // 调用下游生成

    const result = await generateWithQualityGate(
      ctx.userQuery,
      ctx.contextSummary,
      generateFn,
      mergedConfig
    );

    // 记录质量指标到 OpenTelemetry
    ctx.span?.setAttributes({
      "agent.quality.score": result.score.overall,
      "agent.quality.attempts": result.attempts,
      "agent.quality.escalated": result.escalated,
    });

    // 低质量但没有改写成功，附加免责声明
    if (result.score.overall < mergedConfig.minScore) {
      return result.response + "\n\n> ⚠️ 注意：此响应可能不够完整，建议核实。";
    }

    return result.response;
  };
}
```

---

## 评分维度设计技巧

| 维度 | 高分示例 | 低分示例 |
|------|---------|---------|
| relevance | 直接回答"如何重置密码" | 扯到产品历史 |
| accuracy | 引用了上下文中的数据 | 编造了不存在的数字 |
| completeness | 覆盖了问题的所有子问题 | 只答了一半 |
| safety | 无敏感/有害内容 | 包含个人信息 |

**overall 不是平均值** — 可以加权：
```typescript
const overall = (
  score.relevance * 0.35 +
  score.accuracy * 0.30 +
  score.completeness * 0.20 +
  score.safety * 0.15
);
```

---

## 成本控制

Haiku 评分一次约 $0.0003，远低于重新生成的成本。

| 策略 | 成本影响 |
|------|---------|
| 仅评分高风险请求（财务/医疗） | 节省 80% 评分成本 |
| 评分结果缓存（相似问题） | 语义缓存命中节省更多 |
| 异步评分（不阻塞响应） | 延迟归零，但无法改写 |
| 批量评分（Anthropic Batch API） | 离线质量分析，成本减半 |

---

## 与其他模式的关系

| 模式 | 解决的问题 | 区别 |
|------|-----------|------|
| Evals Framework | 离线批量测试 | 本模式是**在线实时**评估 |
| Self-Consistency | 多路采样投票 | 本模式是**质量把关**，不是决策 |
| Output Validation | 格式/结构校验 | 本模式关注**语义质量** |
| Human-in-the-Loop | 人工审核 | 本模式是**自动评分**，人工兜底 |

---

## 关键要点

1. **Haiku 当裁判，便宜又够用** — 不需要 Sonnet 来评分
2. **overall 不等于平均** — 根据业务加权，safety 权重要高
3. **重试有限制** — 最多 2 次，否则成本失控
4. **低于 escalate_score 要告警** — 别让烂响应悄悄溜走
5. **异步模式** — 不阻塞响应时，收集数据做后续改进
6. **评分 ≠ 惩罚** — 数据积累后可以发现 prompt 系统性问题

---

## 适用场景

- ✅ 客服 Agent：回答必须准确完整
- ✅ 财务/医疗问答：高风险，需要质量门控
- ✅ 品牌内容生成：语气必须符合调性
- ❌ 简单问候/聊天：评分浪费成本
- ❌ 纯工具调用：没有"响应质量"概念

---

> **一句话总结**：不要等用户投诉才知道 Agent 说了烂话，用 Haiku 当实时裁判，低分自动重来，生产质量稳住。
