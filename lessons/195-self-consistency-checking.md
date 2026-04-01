# 195 - Agent 自我一致性检验（Self-Consistency Checking）

> **核心思想**：同一问题多次问 LLM，答案投票取多数 → 幻觉自然消散

---

## 为什么需要自我一致性？

LLM 是概率模型，temperature > 0 时每次输出都略有不同。单次采样在高风险场景（代码、计算、事实判断）容易幻觉。

**Self-Consistency**（Wang et al., 2022）：不依赖单条推理路径，而是 **多路采样 + 投票**，让正确答案"浮出水面"。

```
问题 → [采样 N 次] → 答案1, 答案2, ..., 答案N → 投票 → 最终答案
```

效果：CoT + Self-Consistency 在数学推理任务上比单次 CoT 提升 **17%+**。

---

## 何时使用

| 场景 | 值得多采样？ |
|------|------------|
| 代码逻辑生成 | ✅ 高风险，bug 代价大 |
| 数学/数值计算 | ✅ 有唯一正确答案 |
| 事实核验 | ✅ 可以交叉验证 |
| 创意写作 | ❌ 多样性才是目标 |
| 简单问答 | ❌ 采样成本高于收益 |

---

## 核心实现

### TypeScript 版本

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();

// ──────────────────────────────────────────────
// 1. 多路采样
// ──────────────────────────────────────────────
async function sampleN(
  prompt: string,
  n: number,
  temperature: number = 0.7
): Promise<string[]> {
  const tasks = Array.from({ length: n }, () =>
    client.messages.create({
      model: "claude-haiku-4-5",        // 用便宜模型采样，节省成本
      max_tokens: 1024,
      temperature,
      messages: [{ role: "user", content: prompt }],
    })
  );

  const results = await Promise.all(tasks);
  return results.map(
    (r) => (r.content[0] as { text: string }).text.trim()
  );
}

// ──────────────────────────────────────────────
// 2. 答案提取 + 归一化
// ──────────────────────────────────────────────
function extractAnswer(text: string): string {
  // 尝试提取「最终答案：XXX」或「答案是 XXX」
  const patterns = [
    /最终答案[：:]\s*(.+)/,
    /答案[是为][：:]\s*(.+)/,
    /结论[：:]\s*(.+)/,
    /therefore[,，]?\s*(.+)/i,
    /the answer is\s*(.+)/i,
  ];

  for (const pat of patterns) {
    const m = text.match(pat);
    if (m) return m[1].trim().toLowerCase();
  }

  // fallback：取最后一行
  const lines = text.split("\n").filter((l) => l.trim());
  return lines[lines.length - 1].trim().toLowerCase();
}

// ──────────────────────────────────────────────
// 3. 投票（多数表决）
// ──────────────────────────────────────────────
function majorityVote(answers: string[]): {
  winner: string;
  confidence: number;
  distribution: Record<string, number>;
} {
  const counts: Record<string, number> = {};
  for (const a of answers) {
    counts[a] = (counts[a] ?? 0) + 1;
  }

  const sorted = Object.entries(counts).sort((a, b) => b[1] - a[1]);
  const [winner, winCount] = sorted[0];

  return {
    winner,
    confidence: winCount / answers.length,   // 0~1，越高越可信
    distribution: counts,
  };
}

// ──────────────────────────────────────────────
// 4. 完整流程
// ──────────────────────────────────────────────
async function selfConsistencyCheck(
  question: string,
  opts: {
    n?: number;           // 采样次数，默认 5
    threshold?: number;   // 置信度门槛，低于此值触发警告
    temperature?: number;
  } = {}
) {
  const { n = 5, threshold = 0.6, temperature = 0.7 } = opts;

  console.log(`🔄 采样 ${n} 次...`);
  const samples = await sampleN(question, n, temperature);

  const answers = samples.map(extractAnswer);
  const { winner, confidence, distribution } = majorityVote(answers);

  const result = {
    answer: winner,
    confidence,
    distribution,
    reliable: confidence >= threshold,
    samples,          // 保留原始推理链，可用于调试
  };

  if (!result.reliable) {
    console.warn(
      `⚠️  置信度 ${(confidence * 100).toFixed(0)}% 低于阈值 ${threshold * 100}%`,
      "\n分布:", distribution
    );
  }

  return result;
}

// ──────────────────────────────────────────────
// 使用示例
// ──────────────────────────────────────────────
async function main() {
  const question = `
用 Chain-of-Thought 推理，给出最终答案：

一个停车场有 3 排，每排 20 个车位。
目前 1/4 的车位被占用，剩余多少空车位？
最终答案：<数字>
`;

  const result = await selfConsistencyCheck(question, {
    n: 5,
    threshold: 0.6,
  });

  console.log("\n✅ 结果:");
  console.log("  最终答案:", result.answer);
  console.log("  置信度:", `${(result.confidence * 100).toFixed(0)}%`);
  console.log("  投票分布:", result.distribution);
  console.log("  可靠性:", result.reliable ? "✅ 高" : "⚠️  低");
}

main();
```

---

## Python 版本

```python
import asyncio
from anthropic import AsyncAnthropic
from collections import Counter
import re

client = AsyncAnthropic()

async def sample_n(prompt: str, n: int, temperature: float = 0.7) -> list[str]:
    tasks = [
        client.messages.create(
            model="claude-haiku-4-5",
            max_tokens=1024,
            temperature=temperature,
            messages=[{"role": "user", "content": prompt}],
        )
        for _ in range(n)
    ]
    results = await asyncio.gather(*tasks)
    return [r.content[0].text.strip() for r in results]

def extract_answer(text: str) -> str:
    patterns = [
        r"最终答案[：:]\s*(.+)",
        r"答案[是为][：:]\s*(.+)",
        r"therefore[,，]?\s*(.+)",
        r"the answer is\s*(.+)",
    ]
    for pat in patterns:
        if m := re.search(pat, text, re.IGNORECASE):
            return m.group(1).strip().lower()
    lines = [l.strip() for l in text.splitlines() if l.strip()]
    return lines[-1].lower() if lines else text.lower()

async def self_consistency_check(
    question: str,
    n: int = 5,
    threshold: float = 0.6,
    temperature: float = 0.7,
):
    samples = await sample_n(question, n, temperature)
    answers = [extract_answer(s) for s in samples]
    
    counts = Counter(answers)
    winner, win_count = counts.most_common(1)[0]
    confidence = win_count / n
    
    return {
        "answer": winner,
        "confidence": confidence,
        "distribution": dict(counts),
        "reliable": confidence >= threshold,
        "samples": samples,
    }

# 使用
async def main():
    result = await self_consistency_check(
        "Roger 有 5 个网球，他又买了 2 罐，每罐 3 个。他现在共有几个网球？最终答案：<数字>",
        n=5,
        threshold=0.6,
    )
    print(f"答案: {result['answer']}")
    print(f"置信度: {result['confidence']:.0%}")
    print(f"可靠: {result['reliable']}")

asyncio.run(main())
```

---

## 与 Agent 工具调用集成

工具调用前用自我一致性检验决策：

```typescript
// 在工具分发层：高风险工具调用前先检验
async function dispatchWithConsistency(
  toolName: string,
  params: unknown,
  agentDecisionPrompt: string,
  riskLevel: "low" | "high" = "low"
) {
  if (riskLevel === "low") {
    // 低风险：直接执行
    return executeTool(toolName, params);
  }

  // 高风险：先用自我一致性检验是否该调用此工具
  const check = await selfConsistencyCheck(
    `${agentDecisionPrompt}\n\n是否应该调用工具 ${toolName}？请回答 YES 或 NO，给出理由。最终答案：YES 或 NO`,
    { n: 3, threshold: 0.67 }  // 3 次采样，2/3 多数
  );

  if (!check.reliable) {
    throw new Error(`工具调用决策置信度不足 (${check.confidence.toFixed(0)})，需要人工确认`);
  }

  if (check.answer.includes("no")) {
    return { skipped: true, reason: "自我一致性检验否决" };
  }

  return executeTool(toolName, params);
}
```

---

## OpenClaw 实战模式

在 OpenClaw 中，用 `sessions_spawn` 并发跑多个子 Agent 投票：

```typescript
// 概念示意：并发 3 个子 Agent 给出独立判断
const votes = await Promise.all([
  sessions_spawn({ task: question, mode: "run" }),
  sessions_spawn({ task: question, mode: "run" }),
  sessions_spawn({ task: question, mode: "run" }),
]);

// 收集结果，多数为准
const answers = votes.map(v => extractAnswer(v.result));
const { winner } = majorityVote(answers);
```

> 注意：子 Agent 方式成本高，适合极高风险决策（如自动部署、资金操作）。日常推荐单 Model 多次采样。

---

## 成本控制策略

```
标准策略：
  • 默认单次采样
  • 检测到"高风险意图"时升级到 N=3~5
  • 用便宜模型（Haiku）采样，贵模型（Sonnet）仅做最终裁决

分层策略：
  低风险  → N=1（直接回答）
  中风险  → N=3（多数表决）
  高风险  → N=5 + 置信度门控 + 人工审批
```

---

## 关键指标

| 指标 | 含义 |
|------|------|
| `consistency_rate` | 多次采样答案一致率 |
| `low_confidence_rate` | 置信度 < 阈值的比例 |
| `escalation_rate` | 触发人工确认的比例 |
| `cost_per_check` | 每次一致性检验的 token 成本 |

---

## 与其他模式的关系

```
Self-Consistency
  ├── 比较 CoT（链式推理）：互补，CoT 是推理方式，SC 是答案聚合
  ├── 比较 Confidence-Aware Routing（Lesson 164）：SC 主动生成置信度
  ├── 比较 Output Validation（已讲）：SC 是生成侧校验，OV 是结果侧校验
  └── 比较 Multi-Agent Negotiation（已讲）：SC 是轻量版多Agent投票
```

---

## 总结

| 项目 | 内容 |
|------|------|
| 核心思路 | 多路采样 + 多数投票 = 幻觉过滤器 |
| 适用场景 | 代码生成、数学计算、事实判断 |
| 成本控制 | 用便宜模型采样，按风险等级决定 N |
| OpenClaw | 子 Agent 并发 = 分布式投票 |
| 关键参数 | N（采样数）、threshold（置信度门槛）|

**一句话记住**：LLM 单次答案是"一个人的判断"，Self-Consistency 是"陪审团裁决"。
