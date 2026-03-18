# 81 - Agent 自适应采样与温度控制（Adaptive Sampling & Temperature Control）

> 同一个 Agent，做数学题和写诗用的是同一个 LLM——但如果你给它们设同样的 temperature，就是在拿菜刀剃须。

---

## 为什么 Temperature 很重要

LLM 的采样参数直接决定输出的"随机性"：

| 参数 | 作用 | 范围 |
|------|------|------|
| `temperature` | 控制整体随机性，越高越"发散" | 0.0 ~ 2.0 |
| `top_p` | 只从累积概率 ≥ p 的 token 中采样 | 0.0 ~ 1.0 |
| `frequency_penalty` | 降低已出现 token 的概率，减少重复 | -2.0 ~ 2.0 |
| `presence_penalty` | 鼓励出现新 topic | -2.0 ~ 2.0 |

**问题**：大多数 Agent 把这些参数写死在配置里。一套参数应对所有任务——效果必然不理想。

---

## 任务类型 × 参数矩阵

```
任务类型           temperature  top_p   frequency_penalty
─────────────────────────────────────────────────────────
代码生成/修复          0.1        0.9        0.0
数学/逻辑推理          0.0        1.0        0.0
JSON/结构化输出        0.1        0.9        0.0
工具调用规划           0.2        0.9        0.1
摘要/提取              0.3        0.9        0.0
问答/分析              0.5        0.9        0.1
自然语言对话           0.7        0.9        0.2
创意写作/头脑风暴      1.0        0.95       0.5
诗歌/角色扮演          1.2        0.95       0.8
```

---

## 实现方式一：意图分类路由

```python
# learn-claude-code 风格：基于 intent 动态选参数
from enum import Enum
from dataclasses import dataclass
from typing import Optional
import re

class TaskType(Enum):
    CODE = "code"
    MATH = "math"
    STRUCTURED = "structured"
    PLANNING = "planning"
    ANALYSIS = "analysis"
    CREATIVE = "creative"
    CHAT = "chat"

@dataclass
class SamplingParams:
    temperature: float
    top_p: float
    frequency_penalty: float = 0.0
    presence_penalty: float = 0.0
    max_tokens: Optional[int] = None

# 参数注册表
SAMPLING_PROFILES: dict[TaskType, SamplingParams] = {
    TaskType.CODE:       SamplingParams(0.1, 0.9, 0.0),
    TaskType.MATH:       SamplingParams(0.0, 1.0, 0.0),
    TaskType.STRUCTURED: SamplingParams(0.1, 0.9, 0.0),
    TaskType.PLANNING:   SamplingParams(0.2, 0.9, 0.1),
    TaskType.ANALYSIS:   SamplingParams(0.5, 0.9, 0.1),
    TaskType.CREATIVE:   SamplingParams(1.0, 0.95, 0.5),
    TaskType.CHAT:       SamplingParams(0.7, 0.9, 0.2),
}

def classify_task(prompt: str) -> TaskType:
    """简单的规则分类，生产中可用小模型替代"""
    prompt_lower = prompt.lower()
    
    code_signals = ["def ", "function", "import ", "class ", "bug", "fix", "implement", "代码", "函数"]
    math_signals = ["calculate", "compute", "equals", "solve", "计算", "求解", "等于"]
    structured_signals = ["json", "yaml", "xml", "list all", "extract", "parse", "提取", "列出所有"]
    creative_signals = ["write a poem", "story", "creative", "imagine", "写诗", "故事", "创意"]
    
    for signal in code_signals:
        if signal in prompt_lower:
            return TaskType.CODE
    for signal in math_signals:
        if signal in prompt_lower:
            return TaskType.MATH
    for signal in structured_signals:
        if signal in prompt_lower:
            return TaskType.STRUCTURED
    for signal in creative_signals:
        if signal in prompt_lower:
            return TaskType.CREATIVE
    
    return TaskType.CHAT

async def adaptive_completion(prompt: str, client, model: str = "claude-3-5-sonnet"):
    task_type = classify_task(prompt)
    params = SAMPLING_PROFILES[task_type]
    
    print(f"[sampling] task={task_type.value} temp={params.temperature} top_p={params.top_p}")
    
    response = await client.messages.create(
        model=model,
        messages=[{"role": "user", "content": prompt}],
        temperature=params.temperature,
        top_p=params.top_p,
        max_tokens=params.max_tokens or 4096,
    )
    return response
```

---

## 实现方式二：OpenClaw / pi-mono 风格——在 Agent Loop 里动态注入

```typescript
// pi-mono 风格：Tool Call 时按工具类型调整参数
interface ToolSamplingHint {
  temperature?: number;
  top_p?: number;
}

// 工具元数据里声明采样偏好
const toolRegistry: Record<string, { fn: Function; samplingHint?: ToolSamplingHint }> = {
  execute_code: {
    fn: executeCode,
    samplingHint: { temperature: 0.1 }, // 代码执行要精确
  },
  brainstorm: {
    fn: brainstorm,
    samplingHint: { temperature: 1.1 }, // 头脑风暴要发散
  },
  extract_json: {
    fn: extractJson,
    samplingHint: { temperature: 0.0 }, // JSON提取要确定性
  },
  write_email: {
    fn: writeEmail,
    samplingHint: { temperature: 0.7 }, // 邮件写作适中
  },
};

async function agentLoop(userMessage: string) {
  let baseTemperature = 0.5; // 默认值
  
  while (true) {
    const response = await llm.call({
      messages: conversationHistory,
      temperature: baseTemperature,
      tools: Object.keys(toolRegistry),
    });

    if (response.stop_reason === "tool_use") {
      for (const toolCall of response.tool_uses) {
        const tool = toolRegistry[toolCall.name];
        
        // 工具执行前：如果工具有采样偏好，下一轮 LLM 调用使用该温度
        if (tool.samplingHint?.temperature !== undefined) {
          baseTemperature = tool.samplingHint.temperature;
          console.log(`[sampling] Adjusting temperature to ${baseTemperature} for tool: ${toolCall.name}`);
        }
        
        const result = await tool.fn(toolCall.input);
        conversationHistory.push({ role: "tool", content: result });
      }
    } else {
      break; // 完成
    }
  }
}
```

---

## 实现方式三：自适应反馈循环（高阶）

根据输出质量反馈自动调整温度：

```python
class AdaptiveSampler:
    """
    对于需要确定性结果的任务（如 JSON 解析），
    如果输出无效，自动降低温度重试
    """
    
    def __init__(self, initial_temp: float = 0.5):
        self.temperature = initial_temp
        self.history: list[tuple[float, bool]] = []  # (temperature, success)
    
    async def call_with_retry(
        self,
        prompt: str,
        validator: callable,
        max_retries: int = 3,
        decay: float = 0.5,  # 每次失败降低多少温度
    ) -> str:
        temp = self.temperature
        
        for attempt in range(max_retries):
            response = await llm_call(prompt, temperature=temp)
            
            if validator(response):
                # 成功！记录并略微升温（允许更多多样性）
                self.history.append((temp, True))
                self.temperature = min(self.temperature * 1.05, 1.0)
                return response
            
            # 失败：降温重试
            print(f"[adaptive] Attempt {attempt+1} failed at temp={temp:.2f}, retrying...")
            self.history.append((temp, False))
            temp = max(temp * (1 - decay), 0.0)
        
        raise ValueError(f"Failed after {max_retries} retries")

# 使用示例
sampler = AdaptiveSampler(initial_temp=0.3)

def is_valid_json(text: str) -> bool:
    try:
        import json
        json.loads(text)
        return True
    except:
        return False

result = await sampler.call_with_retry(
    prompt="Extract user info as JSON: John Doe, 30, NYC",
    validator=is_valid_json,
)
```

---

## OpenClaw 实际应用：per-session 模型配置

OpenClaw 支持在 cron job 里指定模型参数：

```json
{
  "payload": {
    "kind": "agentTurn",
    "message": "写一首关于代码的诗",
    "model": "anthropic/claude-opus-4",
    "thinking": "adaptive"
  }
}
```

对于需要精确输出的任务（如数据分析），用较低 temperature 的模型配置；
对于创意任务，可以在 session 级别切换模型：

```bash
# 通过 session_status 工具动态切换模型
/model anthropic/claude-opus-4  # 切换到更强的模型做复杂推理
```

---

## 常见反模式 ⚠️

```python
# ❌ 反模式1：所有任务用同一个高温度
response = llm.call(prompt, temperature=0.9)  # 代码生成用 0.9 → 输出会乱

# ❌ 反模式2：结构化输出用高温度
extract_json(prompt, temperature=1.0)  # JSON 经常解析失败

# ❌ 反模式3：从不调整，写死在配置里
class Config:
    TEMPERATURE = 0.7  # 一刀切，所有 Agent 都用这个

# ✅ 正确：按任务类型路由
params = SAMPLING_PROFILES[classify_task(prompt)]
response = llm.call(prompt, **params.__dict__)
```

---

## 温度与 Thinking 模式的关系

对于支持 extended thinking（如 Claude）的模型：

```python
# 需要深度推理时：开启 thinking，temperature 强制为 1.0（API 要求）
if task_type in [TaskType.MATH, TaskType.PLANNING]:
    response = client.messages.create(
        model="claude-opus-4-5",
        thinking={"type": "enabled", "budget_tokens": 10000},
        temperature=1.0,  # thinking 模式下必须为 1.0
        messages=[...]
    )
```

**Thinking 模式下 temperature 固定为 1.0，但内部推理过程是确定性的——这是 API 设计的特殊之处。**

---

## 小结

1. **不同任务需要不同温度** — 代码/JSON 用低温，创意用高温
2. **意图分类** 是自适应采样的关键第一步，可用规则或小模型实现
3. **反馈循环** 让 Agent 能从失败中自动降温重试
4. **工具元数据** 可以携带采样偏好，在工具调用前调整参数
5. **Thinking 模式** 是特殊情况，温度固定为 1.0，但推理是确定性的

> 核心思想：把 LLM 采样参数视为"配置"，而不是"常量"。任务变了，参数也要跟着变。

---

**下一课预告：** Agent 知识蒸馏与专家模型微调——用大模型生成训练数据，蒸馏出领域专用的小模型。
