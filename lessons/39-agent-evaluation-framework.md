# 39 - Agent 评估框架：Evals，让 Agent 质量可度量

> "如果你不能度量它，你就无法改进它。" — 每个做 Agent 的人迟早都会深刻理解这句话。

## 为什么需要 Evals？

Agent 开发最大的痛点之一：**你改了提示词/模型/工具逻辑，怎么知道整体变好了还是变差了？**

传统软件有单元测试，Agent 有 **Evals（评估框架）**。

Evals 解决三个核心问题：
1. **回归检测** — 新版本有没有破坏已有能力？
2. **A/B 比较** — 换模型/改提示词真的有提升吗？
3. **持续监控** — 生产环境中 Agent 的表现在漂移吗？

---

## 核心概念

### Eval 的三要素

```
Dataset × Task → Score
```

- **Dataset**：测试用例集合（输入 + 期望结果）
- **Task**：Agent 执行的能力（工具调用、对话、推理）
- **Score**：量化的评估指标

### 评估方式分类

| 方式 | 适用场景 | 优缺点 |
|------|---------|--------|
| **规则匹配** | 结构化输出、工具参数 | 精确但死板 |
| **模型打分（LLM-as-Judge）** | 开放式回答质量 | 灵活但有偏差 |
| **人工标注** | 金标准建立 | 准确但慢 |
| **指标统计** | 延迟/成本/成功率 | 客观但不评质量 |

---

## 实战：用 Python 构建轻量 Eval 框架

参考 `learn-claude-code` 的测试模式，构建一个可复用的评估系统：

```python
# evals/framework.py
import asyncio
import json
from dataclasses import dataclass, field
from typing import Any, Callable, Optional
from datetime import datetime

@dataclass
class EvalCase:
    """单个测试用例"""
    id: str
    input: dict
    expected: Any
    tags: list[str] = field(default_factory=list)
    metadata: dict = field(default_factory=dict)

@dataclass  
class EvalResult:
    """单次评估结果"""
    case_id: str
    score: float          # 0.0 ~ 1.0
    passed: bool
    actual: Any
    expected: Any
    latency_ms: float
    cost_tokens: int = 0
    reason: str = ""

class EvalSuite:
    """评估套件"""
    
    def __init__(self, name: str, scorer: Callable):
        self.name = name
        self.scorer = scorer  # (actual, expected) -> (score, reason)
        self.cases: list[EvalCase] = []
    
    def add_case(self, case: EvalCase):
        self.cases.append(case)
        return self
    
    async def run(self, agent_fn: Callable, threshold: float = 0.7) -> dict:
        results = []
        
        for case in self.cases:
            start = asyncio.get_event_loop().time()
            
            try:
                actual = await agent_fn(case.input)
                latency_ms = (asyncio.get_event_loop().time() - start) * 1000
                score, reason = self.scorer(actual, case.expected)
            except Exception as e:
                latency_ms = (asyncio.get_event_loop().time() - start) * 1000
                actual = None
                score, reason = 0.0, f"Exception: {e}"
            
            results.append(EvalResult(
                case_id=case.id,
                score=score,
                passed=score >= threshold,
                actual=actual,
                expected=case.expected,
                latency_ms=latency_ms,
                reason=reason,
            ))
        
        return self._summarize(results, threshold)
    
    def _summarize(self, results: list[EvalResult], threshold: float) -> dict:
        passed = [r for r in results if r.passed]
        avg_score = sum(r.score for r in results) / len(results) if results else 0
        avg_latency = sum(r.latency_ms for r in results) / len(results) if results else 0
        
        return {
            "suite": self.name,
            "timestamp": datetime.utcnow().isoformat(),
            "total": len(results),
            "passed": len(passed),
            "pass_rate": len(passed) / len(results) if results else 0,
            "avg_score": avg_score,
            "avg_latency_ms": avg_latency,
            "threshold": threshold,
            "results": [vars(r) for r in results],
        }
```

---

## LLM-as-Judge 模式

当输出是自然语言时，用另一个 LLM 来打分：

```python
# evals/judges.py
import anthropic

client = anthropic.Anthropic()

async def llm_judge(
    actual: str, 
    expected: str, 
    criteria: str = "准确性和完整性"
) -> tuple[float, str]:
    """用 Claude 评估输出质量"""
    
    prompt = f"""你是一个严格的 AI 评估员。
    
任务：评估 AI 回答的质量

评估标准：{criteria}

期望答案：
{expected}

实际答案：
{actual}

请给出：
1. 分数（0.0 到 1.0，保留两位小数）
2. 简短理由（一句话）

以 JSON 格式返回：{{"score": 0.85, "reason": "..."}}"""

    response = client.messages.create(
        model="claude-3-5-haiku-20241022",  # 用便宜的模型打分，省钱
        max_tokens=200,
        messages=[{"role": "user", "content": prompt}]
    )
    
    result = json.loads(response.content[0].text)
    return result["score"], result["reason"]


# 更细粒度的多维度评估
async def multi_criteria_judge(actual: str, expected: str) -> tuple[float, str]:
    """多维度打分：准确性 × 相关性 × 完整性"""
    
    prompt = f"""评估 AI 回答，分别对以下维度打分（0-10）：
- accuracy（准确性）：事实是否正确
- relevance（相关性）：是否回答了问题
- completeness（完整性）：是否足够完整

期望: {expected}
实际: {actual}

返回 JSON: {{"accuracy": 8, "relevance": 9, "completeness": 7, "summary": "..."}}"""

    response = client.messages.create(
        model="claude-3-5-haiku-20241022",
        max_tokens=300,
        messages=[{"role": "user", "content": prompt}]
    )
    
    data = json.loads(response.content[0].text)
    # 加权平均
    score = (data["accuracy"] * 0.4 + data["relevance"] * 0.4 + data["completeness"] * 0.2) / 10
    return score, data["summary"]
```

---

## 工具调用评估

对于有工具调用的 Agent，需要专门评估工具使用的正确性：

```python
# evals/tool_eval.py

def tool_call_scorer(actual_calls: list[dict], expected_calls: list[dict]) -> tuple[float, str]:
    """评估工具调用序列的准确性"""
    
    if not expected_calls:
        # 期望不调用工具
        if not actual_calls:
            return 1.0, "正确：未调用工具"
        return 0.0, f"错误：不应调用工具，但调用了 {len(actual_calls)} 次"
    
    scores = []
    
    for i, expected in enumerate(expected_calls):
        if i >= len(actual_calls):
            scores.append(0.0)
            continue
            
        actual = actual_calls[i]
        
        # 工具名是否正确（权重 40%）
        name_match = 1.0 if actual.get("name") == expected.get("name") else 0.0
        
        # 参数是否正确（权重 60%）
        param_score = _score_params(
            actual.get("input", {}), 
            expected.get("input", {})
        )
        
        scores.append(name_match * 0.4 + param_score * 0.6)
    
    avg = sum(scores) / len(expected_calls)
    return avg, f"工具调用准确率: {avg:.0%}"


def _score_params(actual: dict, expected: dict) -> float:
    """参数匹配得分"""
    if not expected:
        return 1.0
    
    correct = 0
    for key, exp_val in expected.items():
        act_val = actual.get(key)
        if act_val == exp_val:
            correct += 1
        elif isinstance(exp_val, str) and isinstance(act_val, str):
            # 字符串模糊匹配
            if exp_val.lower() in act_val.lower() or act_val.lower() in exp_val.lower():
                correct += 0.8
    
    return correct / len(expected)
```

---

## OpenClaw 中的实际 Eval 配置

在 OpenClaw/pi 体系中，可以把 Eval 跑在隔离的 subagent 里：

```typescript
// pi-mono 风格：eval runner
import { runEval } from "./evals/runner";
import { toolCallScorer, llmJudge } from "./evals/scorers";

const AGENT_EVAL_SUITE = {
  name: "weather-agent-v2",
  cases: [
    {
      id: "basic-weather",
      input: { message: "北京今天天气怎么样？" },
      expected: {
        tool_calls: [{ name: "get_weather", input: { location: "北京" } }]
      },
      tags: ["tool-use", "basic"]
    },
    {
      id: "multi-step",  
      input: { message: "帮我比较上海和深圳明天的温差" },
      expected: {
        tool_calls: [
          { name: "get_weather", input: { location: "上海", days: 1 } },
          { name: "get_weather", input: { location: "深圳", days: 1 } },
        ]
      },
      tags: ["tool-use", "parallel"]
    },
    {
      id: "no-tool-needed",
      input: { message: "天气查询的 API 是哪家的？" },
      expected: { tool_calls: [] },
      tags: ["no-tool"]
    }
  ]
};

// 在 CI/CD 中运行
async function runCIEvals() {
  const result = await runEval(AGENT_EVAL_SUITE, {
    agentFn: weatherAgent,
    scorer: toolCallScorer,
    threshold: 0.85,  // 85% 通过率才算合格
  });
  
  console.log(`Pass rate: ${result.pass_rate * 100}%`);
  
  if (result.pass_rate < 0.85) {
    process.exit(1);  // CI 失败
  }
}
```

---

## Eval 驱动开发（Evals-Driven Development）

类似 TDD，但面向 AI：

```
1. 写 Eval Cases（描述期望行为）
        ↓
2. 跑基线（当前分数：60%）
        ↓  
3. 改 Prompt / 工具逻辑
        ↓
4. 再跑 Evals（新分数：75%）
        ↓
5. 分析失败案例 → 继续改进
        ↓
6. 达标（>85%）→ 上线
```

**关键原则：**
- Eval Cases 是你的"合同"，改 Agent 不能破坏它们
- 每次迭代都要记录分数变化
- 失败案例比通过案例更有价值

---

## 生产监控：在线 Evals

不只是离线测试，生产中也要持续评估：

```python
# 采样线上流量做评估
import random

class ProductionEvalSampler:
    def __init__(self, sample_rate: float = 0.05):  # 采样 5%
        self.sample_rate = sample_rate
        self.buffer = []
    
    async def maybe_eval(self, input: dict, output: dict):
        if random.random() > self.sample_rate:
            return
            
        # 异步评估，不影响主链路
        asyncio.create_task(self._eval_async(input, output))
    
    async def _eval_async(self, input: dict, output: dict):
        score, reason = await llm_judge(
            actual=output.get("response", ""),
            expected="",  # 生产中没有 ground truth
            criteria="回答是否有帮助、准确、安全"
        )
        
        # 上报到监控系统
        await metrics.record({
            "event": "prod_eval",
            "score": score,
            "input_preview": input.get("message", "")[:100],
            "timestamp": time.time(),
        })
        
        # 低分自动告警
        if score < 0.5:
            await alert.send(f"⚠️ 低质量输出检测: score={score}, reason={reason}")
```

---

## 实用 Eval 指标清单

| 指标 | 计算方式 | 目标值 |
|------|---------|--------|
| **任务完成率** | 成功完成任务数 / 总任务数 | > 90% |
| **工具调用准确率** | 正确调用工具 / 应调用工具 | > 85% |
| **平均延迟** | 首 token 到最终响应时间 | < 3s |
| **幻觉率** | 出现事实错误的比例 | < 5% |
| **拒绝率** | 不必要拒绝的比例 | < 2% |
| **Token 效率** | 输出质量 / token 消耗 | 持续优化 |

---

## 总结

Evals 不是一次性工作，是持续的工程实践：

1. **早建 Evals** — 在第一个 v1 上线前就要有测试集
2. **用 LLM 打分** — 人工标注太慢，用 Claude Haiku 打分性价比极高
3. **关注失败案例** — 失败比通过更有价值
4. **Evals 驱动迭代** — 每次改动必须跑 Evals，有数据才有说服力
5. **生产采样** — 线上监控是离线 Evals 的补充，不能替代

> 没有 Evals 的 Agent 优化，就像蒙着眼睛调音响。🎛️

---

*下节预告：Agent 版本管理与灰度发布 — 如何安全地将 Agent 变更推上生产*
