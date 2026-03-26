# 144. Agent 思维链验证与逻辑一致性检查
# Chain-of-Thought Verification & Logical Consistency Check

## 为什么需要这个？

Agent 开启 Extended Thinking 后，推理过程可能出现：
- **自我矛盾**：前半段说"A 为真"，后半段的结论却依赖"A 为假"
- **幻觉漂移**：推理链越来越偏，每步看似合理，整体却错误
- **隐含假设**：推理过程暗含未验证的前提，导致结论不可信

CoT 验证就是在 Agent 得出结论**之前**，用一个独立"裁判"检查推理链的逻辑一致性。

---

## 核心架构

```
用户输入
   ↓
[主 LLM] → 推理链 (chain-of-thought)
   ↓
[验证器 LLM] → 检测矛盾/漏洞
   ↓
   ├── 通过 → 执行最终工具调用 / 返回答案
   └── 失败 → 打回主 LLM 重新推理（附错误说明）
```

---

## 实现代码（TypeScript，pi-mono 风格）

```typescript
// cot-verifier.ts

import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();

interface ReasoningStep {
  content: string;
  type: "thinking" | "conclusion" | "tool_call";
}

interface VerificationResult {
  valid: boolean;
  issues: string[];
  confidence: number; // 0-1
}

// 从 Extended Thinking 响应中提取推理链
function extractReasoningChain(response: Anthropic.Message): ReasoningStep[] {
  const steps: ReasoningStep[] = [];

  for (const block of response.content) {
    if (block.type === "thinking") {
      steps.push({ content: block.thinking, type: "thinking" });
    } else if (block.type === "text") {
      steps.push({ content: block.text, type: "conclusion" });
    } else if (block.type === "tool_use") {
      steps.push({
        content: JSON.stringify({ tool: block.name, input: block.input }),
        type: "tool_call",
      });
    }
  }

  return steps;
}

// 验证器：独立 LLM 裁判
async function verifyReasoningChain(
  userQuery: string,
  steps: ReasoningStep[]
): Promise<VerificationResult> {
  const chainText = steps
    .map((s, i) => `[步骤${i + 1}/${s.type}]\n${s.content}`)
    .join("\n\n---\n\n");

  const verifierPrompt = `你是一个逻辑一致性裁判。请检查以下推理链是否存在问题。

用户问题：${userQuery}

推理链：
${chainText}

请检查以下问题（用 JSON 格式回答）：
1. 是否存在自我矛盾（前后陈述相互否定）？
2. 是否存在未经验证的隐含假设？
3. 结论是否能从推理步骤中合理推导？
4. 是否有逻辑跳跃（缺少中间步骤）？

回答格式：
{
  "valid": true/false,
  "issues": ["问题1", "问题2"],
  "confidence": 0.0-1.0,
  "summary": "一句话总结"
}`;

  const response = await client.messages.create({
    model: "claude-opus-4-5",
    max_tokens: 1024,
    messages: [{ role: "user", content: verifierPrompt }],
  });

  try {
    const text =
      response.content[0].type === "text" ? response.content[0].text : "{}";
    const jsonMatch = text.match(/\{[\s\S]*\}/);
    return jsonMatch ? JSON.parse(jsonMatch[0]) : { valid: true, issues: [], confidence: 0.5 };
  } catch {
    return { valid: true, issues: [], confidence: 0.5 };
  }
}

// 主 Agent 函数：带 CoT 验证的推理循环
async function agentWithCoTVerification(
  userQuery: string,
  tools: Anthropic.Tool[],
  maxRetries = 2
): Promise<string> {
  let attempt = 0;
  let previousIssues: string[] = [];

  while (attempt <= maxRetries) {
    // 构建系统提示，若有历史错误则附上
    const systemPrompt =
      attempt === 0
        ? "你是一个严谨的助手，请仔细推理后再给出答案。"
        : `你是一个严谨的助手。你上一次的推理存在以下问题，请修正后重新推理：\n${previousIssues.map((i) => `- ${i}`).join("\n")}`;

    // 调用主 LLM（开启 Extended Thinking）
    const response = await client.messages.create({
      model: "claude-opus-4-5",
      max_tokens: 16000,
      thinking: { type: "enabled", budget_tokens: 10000 },
      system: systemPrompt,
      messages: [{ role: "user", content: userQuery }],
      tools,
    });

    // 提取推理链
    const steps = extractReasoningChain(response);

    // 验证推理链
    console.log(`[CoT Verifier] 第 ${attempt + 1} 次推理，开始验证...`);
    const verification = await verifyReasoningChain(userQuery, steps);

    console.log(
      `[CoT Verifier] 验证结果: valid=${verification.valid}, confidence=${verification.confidence}`
    );

    if (verification.valid || verification.confidence >= 0.85) {
      // 推理通过，提取最终答案
      const finalText = response.content.find((b) => b.type === "text");
      return finalText?.type === "text"
        ? finalText.text
        : "推理完成，但无法提取文本答案。";
    }

    // 推理失败，记录问题，重试
    previousIssues = verification.issues;
    console.warn(`[CoT Verifier] 发现 ${verification.issues.length} 个问题，重试...`);
    attempt++;
  }

  return "达到最大重试次数，返回最后一次推理结果（可能存在逻辑问题）。";
}

// 示例使用
async function main() {
  const result = await agentWithCoTVerification(
    "如果所有鸟都会飞，企鹅是鸟，那么企鹅能飞吗？请分析这个论证的问题。",
    [] // 无工具调用，纯推理验证
  );
  console.log("最终答案:", result);
}

main().catch(console.error);
```

---

## 多路径一致性检查（更强版本）

生成 N 条独立推理链，检查它们是否收敛到同一结论：

```typescript
// multi-path-consistency.ts

async function multiPathConsistencyCheck(
  query: string,
  numPaths = 3
): Promise<{ answer: string; consistent: boolean; agreement: number }> {
  // 并行生成多条推理链（用不同 temperature 模拟多样性）
  const paths = await Promise.all(
    Array.from({ length: numPaths }, (_, i) =>
      client.messages.create({
        model: "claude-sonnet-4-5",
        max_tokens: 2000,
        // 用不同的 system 提示引导不同推理角度
        system: `你是从第${i + 1}个角度分析问题的推理者。`,
        messages: [{ role: "user", content: query }],
      })
    )
  );

  const answers = paths.map((p) =>
    p.content[0].type === "text" ? p.content[0].text : ""
  );

  // 让裁判 LLM 判断这些答案是否一致
  const consistencyCheck = await client.messages.create({
    model: "claude-haiku-4-5", // 用便宜的模型做聚合
    max_tokens: 512,
    messages: [
      {
        role: "user",
        content: `以下是对同一问题的 ${numPaths} 个独立回答，请判断它们的核心结论是否一致（JSON格式）：

${answers.map((a, i) => `回答${i + 1}: ${a.slice(0, 500)}`).join("\n\n")}

{"consistent": true/false, "agreement": 0.0-1.0, "synthesis": "综合结论"}`,
      },
    ],
  });

  const text =
    consistencyCheck.content[0].type === "text"
      ? consistencyCheck.content[0].text
      : "{}";
  const jsonMatch = text.match(/\{[\s\S]*\}/);
  const result = jsonMatch ? JSON.parse(jsonMatch[0]) : { consistent: false, agreement: 0, synthesis: "" };

  return {
    answer: result.synthesis || answers[0],
    consistent: result.consistent,
    agreement: result.agreement,
  };
}
```

---

## 在 OpenClaw 中集成（工具中间件版）

```typescript
// cot-verification-middleware.ts
// 作为 OpenClaw 工具中间件：高风险工具调用前先验证推理

import type { ToolMiddleware } from "openclaw";

const HIGH_RISK_TOOLS = new Set([
  "bash",
  "file_write",
  "database_query",
  "send_email",
  "deploy",
]);

export const cotVerificationMiddleware: ToolMiddleware = {
  name: "cot-verification",

  async before(toolCall, context) {
    // 只对高风险工具启用验证
    if (!HIGH_RISK_TOOLS.has(toolCall.name)) return;

    const recentThinking = context.session.getLastThinkingBlock();
    if (!recentThinking) return;

    const result = await verifyReasoningChain(context.userQuery ?? "", [
      { content: recentThinking, type: "thinking" },
      {
        content: JSON.stringify(toolCall.input),
        type: "tool_call",
      },
    ]);

    if (!result.valid && result.confidence < 0.7) {
      // 阻断工具调用，返回错误让 LLM 重新推理
      throw new Error(
        `[CoT Verification Failed] 推理存在以下问题：\n${result.issues.join("\n")}\n请重新分析后再执行操作。`
      );
    }

    console.log(
      `[CoT Verifier] ${toolCall.name} 推理验证通过 (confidence: ${result.confidence})`
    );
  },
};
```

---

## 关键指标与调优

| 场景 | 推荐策略 | 验证模型 |
|------|---------|---------|
| 高风险操作（写文件/发邮件） | 单路验证 + 阻断 | claude-opus |
| 复杂推理问题 | 多路径一致性检查 | claude-sonnet |
| 批量低风险任务 | 采样抽检（20%） | claude-haiku |
| 实时对话 | 仅检查矛盾，不检查完整性 | claude-haiku |

**验证阈值建议：**
- `confidence >= 0.85`：自动通过
- `0.7 <= confidence < 0.85`：记录警告，通过但标记
- `confidence < 0.7`：阻断并要求重新推理

---

## 与已学知识的联系

| 已学模式 | 联系点 |
|---------|--------|
| **Agent Reflection**（自我反思） | CoT 验证是实时的、结构化的反思 |
| **Output Validation & Self-Correction** | 验证对象不同：此篇验证*推理过程*，彼篇验证*输出结果* |
| **Multi-Model Fallback Chain** | 验证失败可触发切换更强模型重新推理 |
| **Tool Call Dry-Run** | 都是"执行前检查"，可组合使用 |
| **Structured Thinking (CoT/ToT/GoT)** | 本篇是对 CoT 结果的质量保障层 |

---

## 一句话总结

> **CoT 验证 = 给 Agent 的推理过程加一层独立审计。**
> 主 LLM 负责推理，验证器 LLM 负责找 bug，两者互不依赖，共同保障决策质量。

---

*下一课预告：Agent 工具调用图谱可视化（Tool Call Graph Visualization）— 让 Agent 的执行轨迹一目了然*
