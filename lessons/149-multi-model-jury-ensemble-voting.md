# 149. Agent 多模型合议制（Multi-Model Jury & Ensemble Voting）

> 多个模型投票，裁判挑最优——像陪审团一样提升 Agent 决策质量

---

## 为什么需要合议制？

单模型答案有盲区：
- 一个模型 **自信地给错答案**，你没有对比依据
- 高风险场景（代码审查、安全判断、关键决策），单点失败代价极高
- 模型间各有偏好：GPT-4o 擅长代码，Claude 擅长推理，Gemini 擅长长文档

**Multi-Model Jury**：把同一问题发给多个模型，收集答案，用仲裁策略选出最优解。

---

## 核心架构

```
用户请求
    │
    ▼
[Dispatcher] ──── 并发发给 N 个模型 ────►  Model A (GPT-4o)
    │                                      Model B (Claude Sonnet)
    │                                      Model C (Gemini Pro)
    │
    ▼
[Collector]  ──── 等待所有/超时截断 ────►  候选答案池
    │
    ▼
[Arbiter]    ──── 投票/评分/LLM裁判 ────►  最优答案
    │
    ▼
用户得到高质量响应
```

---

## TypeScript 实现（结合 pi-mono 风格）

### 核心类型

```typescript
interface JurorResult {
  model: string;
  answer: string;
  latencyMs: number;
  tokens: number;
  error?: string;
}

interface VerdictStrategy {
  name: string;
  decide(results: JurorResult[], question: string): Promise<string>;
}
```

### Dispatcher：并发发给所有模型

```typescript
import Anthropic from "@anthropic-ai/sdk";
import OpenAI from "openai";

const anthropic = new Anthropic();
const openai = new OpenAI();

async function callModel(
  model: string,
  question: string,
  timeoutMs = 15000
): Promise<JurorResult> {
  const start = Date.now();
  try {
    const controller = new AbortController();
    const timer = setTimeout(() => controller.abort(), timeoutMs);

    let answer = "";

    if (model.startsWith("claude")) {
      const res = await anthropic.messages.create(
        {
          model,
          max_tokens: 1024,
          messages: [{ role: "user", content: question }],
        },
        { signal: controller.signal }
      );
      answer = (res.content[0] as any).text;
    } else if (model.startsWith("gpt")) {
      const res = await openai.chat.completions.create(
        {
          model,
          messages: [{ role: "user", content: question }],
          max_tokens: 1024,
        },
        { signal: controller.signal }
      );
      answer = res.choices[0].message.content ?? "";
    }

    clearTimeout(timer);
    return {
      model,
      answer,
      latencyMs: Date.now() - start,
      tokens: answer.length / 4, // 粗估
    };
  } catch (err: any) {
    return {
      model,
      answer: "",
      latencyMs: Date.now() - start,
      tokens: 0,
      error: err.message,
    };
  }
}

// 并发派发给所有 jurors
async function dispatch(
  models: string[],
  question: string
): Promise<JurorResult[]> {
  const results = await Promise.allSettled(
    models.map((m) => callModel(m, question))
  );

  return results.map((r, i) =>
    r.status === "fulfilled"
      ? r.value
      : { model: models[i], answer: "", latencyMs: 0, tokens: 0, error: "rejected" }
  );
}
```

### 三种仲裁策略

#### 策略 1：LLM 裁判（最准，成本最高）

```typescript
class LLMArbitratorStrategy implements VerdictStrategy {
  name = "llm-arbitrator";

  async decide(results: JurorResult[], question: string): Promise<string> {
    const valid = results.filter((r) => !r.error && r.answer);
    if (valid.length === 0) throw new Error("All jurors failed");
    if (valid.length === 1) return valid[0].answer;

    const candidateText = valid
      .map((r, i) => `[候选 ${i + 1}] 来自 ${r.model}:\n${r.answer}`)
      .join("\n\n---\n\n");

    const prompt = `
你是一个公正的仲裁者。用户问了这个问题：
<question>${question}</question>

以下是 ${valid.length} 个模型的回答：
${candidateText}

请选出最准确、最有帮助的答案，并说明理由（50字以内）。
最后输出：VERDICT: <候选编号>

然后直接输出获胜答案的完整内容。`;

    const res = await anthropic.messages.create({
      model: "claude-opus-4-5", // 用最强模型做裁判
      max_tokens: 2048,
      messages: [{ role: "user", content: prompt }],
    });

    const raw = (res.content[0] as any).text as string;

    // 提取获胜的答案
    const match = raw.match(/VERDICT:\s*(\d+)/i);
    if (match) {
      const idx = parseInt(match[1]) - 1;
      return valid[idx]?.answer ?? valid[0].answer;
    }

    // 没有明确 VERDICT，返回裁判原文
    return raw;
  }
}
```

#### 策略 2：语义相似度投票（中等成本，适合分类/短答案）

```typescript
class SemanticVotingStrategy implements VerdictStrategy {
  name = "semantic-voting";

  // 简化版：用 LLM 评分每个答案的质量
  async decide(results: JurorResult[], question: string): Promise<string> {
    const valid = results.filter((r) => !r.error && r.answer);
    if (valid.length === 0) throw new Error("All jurors failed");
    if (valid.length === 1) return valid[0].answer;

    // 用 embedding 或简单字符串聚合找"共识"答案
    // 这里用简化版：选最长且没有明显错误的
    const scored = valid.map((r) => ({
      ...r,
      score: r.answer.length + (r.latencyMs < 5000 ? 100 : 0),
    }));

    scored.sort((a, b) => b.score - a.score);
    return scored[0].answer;
  }
}
```

#### 策略 3：快速赢家（最低成本，适合低风险场景）

```typescript
class FastestValidStrategy implements VerdictStrategy {
  name = "fastest-valid";

  async decide(results: JurorResult[]): Promise<string> {
    const valid = results
      .filter((r) => !r.error && r.answer.length > 10)
      .sort((a, b) => a.latencyMs - b.latencyMs);

    if (valid.length === 0) throw new Error("No valid answer");
    return valid[0].answer;
  }
}
```

### 合议庭主类

```typescript
class MultiModelJury {
  private models: string[];
  private strategy: VerdictStrategy;
  private cache = new Map<string, string>();

  constructor(
    models = ["claude-sonnet-4-5", "gpt-4o-mini"],
    strategy: VerdictStrategy = new LLMArbitratorStrategy()
  ) {
    this.models = models;
    this.strategy = strategy;
  }

  async consult(question: string): Promise<{
    verdict: string;
    jurorResults: JurorResult[];
    strategyUsed: string;
  }> {
    // 检查缓存
    const cacheKey = question.slice(0, 100);
    if (this.cache.has(cacheKey)) {
      return {
        verdict: this.cache.get(cacheKey)!,
        jurorResults: [],
        strategyUsed: "cache-hit",
      };
    }

    console.log(`[Jury] 派发给 ${this.models.length} 个模型...`);
    const jurorResults = await dispatch(this.models, question);

    const succeeded = jurorResults.filter((r) => !r.error);
    console.log(
      `[Jury] ${succeeded.length}/${this.models.length} 个模型成功响应`
    );

    const verdict = await this.strategy.decide(jurorResults, question);
    this.cache.set(cacheKey, verdict);

    return {
      verdict,
      jurorResults,
      strategyUsed: this.strategy.name,
    };
  }
}
```

---

## Python 实现（asyncio 版）

```python
import asyncio
import anthropic
from openai import AsyncOpenAI
from dataclasses import dataclass, field
from typing import Optional

@dataclass
class JurorResult:
    model: str
    answer: str
    latency_ms: float
    error: Optional[str] = None

async def call_claude(question: str, model: str) -> JurorResult:
    client = anthropic.AsyncAnthropic()
    start = asyncio.get_event_loop().time()
    try:
        res = await asyncio.wait_for(
            client.messages.create(
                model=model,
                max_tokens=1024,
                messages=[{"role": "user", "content": question}]
            ),
            timeout=15.0
        )
        return JurorResult(
            model=model,
            answer=res.content[0].text,
            latency_ms=(asyncio.get_event_loop().time() - start) * 1000
        )
    except Exception as e:
        return JurorResult(model=model, answer="", latency_ms=0, error=str(e))

async def call_openai(question: str, model: str) -> JurorResult:
    client = AsyncOpenAI()
    start = asyncio.get_event_loop().time()
    try:
        res = await asyncio.wait_for(
            client.chat.completions.create(
                model=model,
                messages=[{"role": "user", "content": question}],
                max_tokens=1024
            ),
            timeout=15.0
        )
        return JurorResult(
            model=model,
            answer=res.choices[0].message.content or "",
            latency_ms=(asyncio.get_event_loop().time() - start) * 1000
        )
    except Exception as e:
        return JurorResult(model=model, answer="", latency_ms=0, error=str(e))

class MultiModelJury:
    def __init__(self):
        self.jurors = [
            ("claude", "claude-sonnet-4-5"),
            ("openai", "gpt-4o-mini"),
            ("claude", "claude-haiku-4-5"),  # 便宜快速的对比
        ]

    async def consult(self, question: str) -> dict:
        tasks = []
        for provider, model in self.jurors:
            if provider == "claude":
                tasks.append(call_claude(question, model))
            else:
                tasks.append(call_openai(question, model))

        results = await asyncio.gather(*tasks, return_exceptions=False)
        valid = [r for r in results if not r.error and r.answer]
        
        if not valid:
            raise RuntimeError("All jurors failed")

        verdict = await self._arbitrate(valid, question)
        return {"verdict": verdict, "jurors": len(valid)}

    async def _arbitrate(self, results: list[JurorResult], question: str) -> str:
        if len(results) == 1:
            return results[0].answer

        client = anthropic.AsyncAnthropic()
        candidates = "\n\n".join(
            f"[{i+1}] {r.model}:\n{r.answer}"
            for i, r in enumerate(results)
        )
        
        res = await client.messages.create(
            model="claude-opus-4-5",
            max_tokens=2048,
            messages=[{
                "role": "user",
                "content": f"问题：{question}\n\n候选答案：\n{candidates}\n\n选出最优答案，输出 BEST: <编号>，然后输出该答案内容。"
            }]
        )
        
        text = res.content[0].text
        import re
        m = re.search(r"BEST:\s*(\d+)", text, re.IGNORECASE)
        if m:
            idx = int(m.group(1)) - 1
            return results[min(idx, len(results)-1)].answer
        return results[0].answer

# 使用
async def main():
    jury = MultiModelJury()
    result = await jury.consult(
        "用 Python 写一个生产级的数据库连接池，包含健康检查和自动重连"
    )
    print(f"✅ 合议结果（{result['jurors']} 位陪审员）：")
    print(result["verdict"])

asyncio.run(main())
```

---

## 与 OpenClaw Cron 集成

```typescript
// 用于定期对高优先级任务执行合议裁决
// 例如：每天早上对今日计划进行多模型评审

const jury = new MultiModelJury(
  ["claude-opus-4-5", "gpt-4o", "claude-sonnet-4-5"],
  new LLMArbitratorStrategy()
);

// 在 OpenClaw cron job 里调用
async function dailyPlanReview(planText: string) {
  const { verdict, jurorResults } = await jury.consult(
    `请评审以下今日计划，找出风险点和优化建议：\n\n${planText}`
  );

  // 汇报给老板
  console.log("📋 多模型合议评审结果：");
  jurorResults.forEach((r) => {
    const status = r.error ? "❌" : "✅";
    console.log(`  ${status} ${r.model}: ${r.latencyMs.toFixed(0)}ms`);
  });
  console.log("\n🏆 最终裁决：");
  console.log(verdict);
}
```

---

## 适用场景 vs 不适用场景

| 场景 | 推荐？ | 理由 |
|------|--------|------|
| 代码安全审查 | ✅ 强推 | 单模型可能漏掉漏洞 |
| 关键业务决策 | ✅ 强推 | 多角度交叉验证 |
| 简单闲聊 | ❌ 过杀 | 成本 3x，没必要 |
| 实时对话 | ⚠️ 慎用 | 延迟翻倍 |
| A/B 测试新模型 | ✅ 合适 | 自然对比基线 |
| 模型性能基准 | ✅ 合适 | 收集各模型表现数据 |

---

## 成本控制技巧

```typescript
// 1. 先用廉价模型快速过滤，只对复杂问题启动全合议
function shouldUseJury(question: string): boolean {
  const complexitySignals = [
    question.length > 200,
    /安全|漏洞|资金|生产|删除/.test(question),
    question.includes("？") && question.split("？").length > 2,
  ];
  return complexitySignals.filter(Boolean).length >= 2;
}

// 2. 缓存相似问题的裁决结果（见上面 cache 实现）

// 3. 动态选择陪审团规模
function getJurorCount(riskLevel: "low" | "medium" | "high"): number {
  return { low: 1, medium: 2, high: 3 }[riskLevel];
}
```

---

## 关键设计原则

1. **超时截断**：不要等最慢的模型，设定截止时间，有答案就裁决
2. **优雅降级**：一个 juror 失败不影响整体，valid results 非空即可
3. **裁判模型隔离**：仲裁者用高质量模型，jurors 可以用便宜模型
4. **结果可审计**：保存每个 juror 的原始答案，方便事后分析
5. **渐进启用**：先在高风险任务上试，验证效果后再扩展

---

## 总结

```
单模型  →  快、便宜、可能错
合议制  →  慢一点、贵一点、更可靠

用在刀刃上：关键决策、高风险操作、需要交叉验证的场景
其余场景：单模型 + 好的提示词就够了
```

Multi-Model Jury 不是银弹，但在正确的场景里，它是提升 Agent 可靠性最直接的方法。

---

*Lesson 149 / Agent 开发课程*
