# Agent 双系统决策架构（Dual-Process Decision Architecture）

> 快系统 = 直觉反射，慢系统 = 深度推理。Agent 应该像人类大脑一样，根据任务难度自动切换两种模式。

---

## 核心思想

认知科学家 Daniel Kahneman 在《思考，快与慢》中描述了人类的两种思维系统：

| | System 1（快系统） | System 2（慢系统） |
|---|---|---|
| 特征 | 直觉、自动、低耗能 | 推理、刻意、高耗能 |
| 速度 | 毫秒级 | 秒级甚至更长 |
| 适合 | 模式匹配、简单分类 | 复杂推理、多步规划 |
| 代价 | 低 token / 低延迟 | 高 token / 高延迟 |

Agent 设计中的问题：大多数 Agent **总是用慢系统**（每次都全力思考），或**总是用快系统**（根本不思考）。

正确做法：**根据任务复杂度自动路由**。

---

## 为什么重要

- 回答"现在几点"用 extended thinking = 浪费钱
- 处理"设计一个分布式锁方案"不加思考 = 输出垃圾
- **错误的系统匹配 = 成本爆炸或质量崩塌**

---

## 架构设计

```
用户输入
  ↓
[复杂度分类器] ← 轻量 Haiku 快速打分
  ↓
  ├── SIMPLE → System 1（Haiku/Sonnet，无 thinking）
  ├── MODERATE → System 1.5（Sonnet，低 thinking budget）
  └── COMPLEX → System 2（Sonnet/Opus，高 thinking budget）
```

---

## TypeScript 实现

```typescript
// dual-process-agent.ts

type Complexity = 'SIMPLE' | 'MODERATE' | 'COMPLEX';

interface ProcessConfig {
  model: string;
  thinkingBudget: number; // 0 = 禁用 thinking
  maxTokens: number;
  label: string;
}

// 两套系统的配置
const SYSTEM_CONFIGS: Record<Complexity, ProcessConfig> = {
  SIMPLE: {
    model: 'claude-haiku-4-5',
    thinkingBudget: 0,
    maxTokens: 1024,
    label: 'System1-Reflex',
  },
  MODERATE: {
    model: 'claude-sonnet-4-5',
    thinkingBudget: 2000,  // 低 budget
    maxTokens: 4096,
    label: 'System1.5-Balanced',
  },
  COMPLEX: {
    model: 'claude-sonnet-4-5',
    thinkingBudget: 10000, // 高 budget（extended thinking）
    maxTokens: 16000,
    label: 'System2-Deliberate',
  },
};

// ── 复杂度分类器 ──────────────────────────────
async function classifyComplexity(
  input: string,
  context: string[],
  anthropic: Anthropic
): Promise<{ complexity: Complexity; reason: string }> {
  
  // 快速规则判断，绕过 LLM
  const simplePatterns = [
    /^(你好|hi|hello|谢谢|thanks)/i,
    /^(现在几点|今天几号|天气怎么样)/,
    /^(帮我|给我)(查|看|找).{1,20}$/,
  ];
  
  if (simplePatterns.some(p => p.test(input.trim()))) {
    return { complexity: 'SIMPLE', reason: 'matched simple pattern' };
  }

  // 复杂关键词快速检测
  const complexKeywords = ['设计', '架构', '优化方案', '分析', '对比', '权衡', 
                            'design', 'architect', 'tradeoff', 'analyze'];
  const hasComplexKeyword = complexKeywords.some(k => input.includes(k));
  
  // 长输入或多轮历史 → 倾向复杂
  const isLongInput = input.length > 300;
  const hasDeepContext = context.length > 6;
  
  if (hasComplexKeyword || (isLongInput && hasDeepContext)) {
    return { complexity: 'COMPLEX', reason: 'complex keywords or deep context' };
  }

  // 中间情况：用 Haiku 快速分类（成本极低）
  const response = await anthropic.messages.create({
    model: 'claude-haiku-4-5',
    max_tokens: 100,
    messages: [{
      role: 'user',
      content: `Rate the complexity of this task: "${input.slice(0, 200)}"
Output JSON only: {"complexity": "SIMPLE|MODERATE|COMPLEX", "reason": "one sentence"}`
    }]
  });

  try {
    const text = response.content[0].type === 'text' ? response.content[0].text : '';
    const json = JSON.parse(text.match(/\{.*\}/s)?.[0] || '{}');
    return {
      complexity: json.complexity || 'MODERATE',
      reason: json.reason || 'llm classified',
    };
  } catch {
    return { complexity: 'MODERATE', reason: 'classification failed, using default' };
  }
}

// ── 双系统 Agent 主函数 ────────────────────────
async function dualProcessAgent(
  userMessage: string,
  conversationHistory: Array<{ role: string; content: string }>,
  anthropic: Anthropic
): Promise<{
  response: string;
  processUsed: string;
  thinkingUsed: boolean;
  tokensUsed: number;
}> {
  const startTime = Date.now();
  
  // Step 1: 分类复杂度
  const { complexity, reason } = await classifyComplexity(
    userMessage,
    conversationHistory.map(m => m.content),
    anthropic
  );
  
  const config = SYSTEM_CONFIGS[complexity];
  console.log(`[DualProcess] Complexity=${complexity} (${reason}) → ${config.label}`);

  // Step 2: 按复杂度配置调用 LLM
  const requestParams: any = {
    model: config.model,
    max_tokens: config.maxTokens,
    messages: [
      ...conversationHistory,
      { role: 'user', content: userMessage }
    ],
  };

  // 仅在需要时启用 extended thinking
  if (config.thinkingBudget > 0) {
    requestParams.thinking = {
      type: 'enabled',
      budget_tokens: config.thinkingBudget,
    };
  }

  const response = await anthropic.messages.create(requestParams);

  // 提取文字响应（跳过 thinking block）
  const textContent = response.content
    .filter(block => block.type === 'text')
    .map(block => (block as any).text)
    .join('');

  const hasThinking = response.content.some(block => block.type === 'thinking');
  const latency = Date.now() - startTime;

  console.log(`[DualProcess] Done in ${latency}ms, thinking=${hasThinking}, tokens=${response.usage.output_tokens}`);

  return {
    response: textContent,
    processUsed: config.label,
    thinkingUsed: hasThinking,
    tokensUsed: response.usage.input_tokens + response.usage.output_tokens,
  };
}
```

---

## Python 版本

```python
# dual_process_agent.py
import anthropic
import re
import json
from dataclasses import dataclass
from enum import Enum

class Complexity(Enum):
    SIMPLE = "SIMPLE"
    MODERATE = "MODERATE"
    COMPLEX = "COMPLEX"

@dataclass
class ProcessConfig:
    model: str
    thinking_budget: int  # 0 = disabled
    max_tokens: int
    label: str

CONFIGS = {
    Complexity.SIMPLE:   ProcessConfig("claude-haiku-4-5",  0,     1024,  "System1-Reflex"),
    Complexity.MODERATE: ProcessConfig("claude-sonnet-4-5", 2000,  4096,  "System1.5-Balanced"),
    Complexity.COMPLEX:  ProcessConfig("claude-sonnet-4-5", 10000, 16000, "System2-Deliberate"),
}

def classify_complexity(user_input: str, client: anthropic.Anthropic) -> Complexity:
    """用规则+Haiku 快速分类"""
    simple_patterns = [r"^(你好|hi|hello|谢谢)", r"(几点|今天|天气)"]
    if any(re.search(p, user_input, re.I) for p in simple_patterns):
        return Complexity.SIMPLE
    
    complex_kw = ["设计", "架构", "分析", "优化方案", "tradeoff"]
    if any(kw in user_input for kw in complex_kw) or len(user_input) > 300:
        return Complexity.COMPLEX
    
    resp = client.messages.create(
        model="claude-haiku-4-5",
        max_tokens=80,
        messages=[{
            "role": "user",
            "content": f'Classify: "{user_input[:200]}"\nJSON: {{"complexity":"SIMPLE|MODERATE|COMPLEX"}}'
        }]
    )
    try:
        text = resp.content[0].text
        data = json.loads(re.search(r"\{.*\}", text, re.S).group())
        return Complexity(data["complexity"])
    except Exception:
        return Complexity.MODERATE

def dual_process_agent(user_message: str, client: anthropic.Anthropic) -> dict:
    complexity = classify_complexity(user_message, client)
    cfg = CONFIGS[complexity]
    print(f"[DualProcess] {complexity.value} → {cfg.label}")
    
    params = dict(
        model=cfg.model,
        max_tokens=cfg.max_tokens,
        messages=[{"role": "user", "content": user_message}],
    )
    if cfg.thinking_budget > 0:
        params["thinking"] = {"type": "enabled", "budget_tokens": cfg.thinking_budget}
    
    response = client.messages.create(**params)
    text = " ".join(b.text for b in response.content if b.type == "text")
    has_thinking = any(b.type == "thinking" for b in response.content)
    
    return {
        "response": text,
        "process": cfg.label,
        "thinking_used": has_thinking,
        "tokens": response.usage.input_tokens + response.usage.output_tokens,
    }
```

---

## OpenClaw 集成：自适应 thinking 模式

OpenClaw 本身就内置了 dual-process 的概念：

```
/reasoning         → 切换 System 2（开启 extended thinking）
/reasoning stream  → System 2 实时展示思考过程
/status            → 查看当前 Reasoning 状态
```

在 SOUL.md 中可以声明策略：

```markdown
## 思考策略

- 日常问答 → 快速响应（System 1）
- 架构设计 / 复杂分析 → 开启 /reasoning（System 2）
- 危险操作（删除/部署）→ 必须慢系统，先想清楚再执行
```

**pi-mono 实现参考（ModelRouter + thinking）：**

```typescript
// pi-mono/src/model-router.ts 风格
class ModelRouter {
  route(task: Task): ModelConfig {
    // 复杂度打分 → 自动决定是否启用 thinking
    const score = this.scoreComplexity(task);
    
    return {
      model: score > 0.7 ? 'claude-sonnet-4-5' : 'claude-haiku-4-5',
      thinking: score > 0.7 
        ? { type: 'enabled', budget_tokens: Math.floor(score * 10000) }
        : undefined,
    };
  }
  
  private scoreComplexity(task: Task): number {
    let score = 0;
    if (task.requiresMultiStep) score += 0.3;
    if (task.hasAmbiguity) score += 0.2;
    if (task.isIrreversible) score += 0.3;  // 不可逆操作必须慢系统
    if (task.contextDepth > 5) score += 0.2;
    return Math.min(score, 1.0);
  }
}
```

---

## 成本对比

| 场景 | 总是 System 2 | 双系统路由 | 节省 |
|------|-------------|----------|------|
| 80% 简单问答 + 20% 复杂任务 | $10 / 1000次 | $3.2 / 1000次 | **68%** |
| 延迟（简单问答） | 3-8s | 0.5-1s | **85%** |
| 延迟（复杂任务） | 3-8s | 8-20s | 质量提升 |

> 复杂任务反而更慢、更贵 = 故意的，因为值得

---

## 关键设计原则

1. **分类器要轻**：用规则优先，LLM 兜底，分类器本身不能比任务还贵
2. **不可逆操作强制 System 2**：宁可多花 $0.01，不要操作失误
3. **慢不等于好**：System 2 用错地方会让用户等待很长时间却没必要
4. **暴露给用户**：告诉用户"这个问题我需要认真想一下"比悄悄等待更好

---

## 实际运行示例

```
用户: 你好

→ [System1-Reflex] matched simple pattern
→ 响应时间: 0.4s | tokens: 85
→ "你好！有什么可以帮你的？"

────────────────────────────────────

用户: 帮我设计一个支持多租户的 Agent 工具权限架构

→ [System2-Deliberate] complex keywords detected
→ extended thinking: budget=10000 tokens
→ 响应时间: 18.2s | tokens: 4821
→ [深度架构方案，包含权限矩阵、数据隔离策略...]
```

---

## 总结

| 概念 | 实现 |
|------|------|
| System 1 | Haiku + 无 thinking，模式匹配快速响应 |
| System 2 | Sonnet + extended thinking，深度推理 |
| 路由器 | 规则 → Haiku 分类 → 复杂度打分 |
| 强制 System 2 | 不可逆操作、高风险决策 |
| OpenClaw 对应 | `/reasoning` 手动切换，`adaptive` 自动切换 |

**核心洞见**：好的 Agent 不是"总是最努力思考"，而是"知道什么时候该努力思考"。

---

*下一课预告：Agent 工具调用熔断与自动恢复（Circuit Breaker with Auto-Recovery）*
