# 91 - Agent 多模型回退链（Multi-Model Fallback Chain）

> **核心思想**：不要把鸡蛋放在一个模型里。主模型挂了、限速了、太贵了——回退链自动切换，用户无感知。

---

## 为什么需要多模型回退链？

生产环境中，单一 LLM 依赖是最大的单点故障：

| 问题 | 现象 | 后果 |
|------|------|------|
| Rate Limit (429) | API 调用频率超限 | 任务失败 |
| 服务宕机 | Anthropic/OpenAI 故障 | 全线崩溃 |
| Context 超长 | 输入超过模型上限 | 报错 |
| 成本超预算 | Token 费用爆炸 | 被迫降级 |
| 延迟太高 | 主模型响应慢 | 用户体验差 |

回退链的思路就一句话：**定义一个有序模型列表，按优先级尝试，遇到可恢复错误就切下一个。**

---

## 基础实现（Python）

```python
import anthropic
import openai
import time
from dataclasses import dataclass
from typing import Callable, Optional
from enum import Enum

class FailureReason(Enum):
    RATE_LIMIT = "rate_limit"
    CONTEXT_TOO_LONG = "context_too_long"
    SERVER_ERROR = "server_error"
    TIMEOUT = "timeout"
    COST_LIMIT = "cost_limit"

@dataclass
class ModelConfig:
    name: str           # 模型标识符
    provider: str       # "anthropic" | "openai" | "local"
    max_tokens: int     # 最大输出 token
    context_window: int # 上下文窗口大小
    cost_per_1k: float  # 每千 token 费用（美元）
    priority: int       # 优先级，数字越小越优先

# 回退链配置：从贵到便宜，从强到弱
FALLBACK_CHAIN = [
    ModelConfig("claude-opus-4-5", "anthropic", 8192, 200000, 0.075, 1),
    ModelConfig("claude-sonnet-4-5", "anthropic", 8192, 200000, 0.015, 2),
    ModelConfig("claude-haiku-3-5", "anthropic", 4096, 200000, 0.001, 3),
    ModelConfig("gpt-4o", "openai", 4096, 128000, 0.030, 4),
    ModelConfig("gpt-4o-mini", "openai", 4096, 128000, 0.0006, 5),
]

def call_with_fallback(
    messages: list[dict],
    system: str = "",
    max_cost_usd: float = 1.0,
    timeout_seconds: int = 30,
) -> tuple[str, str]:  # (response_text, model_used)
    """
    按回退链顺序尝试调用模型。
    返回 (响应文本, 实际使用的模型名)
    """
    last_error = None
    
    for model in FALLBACK_CHAIN:
        # 成本过滤：预估成本超预算就跳过
        estimated_input_tokens = sum(len(m["content"]) // 4 for m in messages)
        estimated_cost = (estimated_input_tokens / 1000) * model.cost_per_1k
        if estimated_cost > max_cost_usd:
            print(f"⚠️ {model.name} 预估成本 ${estimated_cost:.3f} 超限，跳过")
            continue
        
        # 上下文窗口过滤
        total_chars = sum(len(m["content"]) for m in messages) + len(system)
        if total_chars // 4 > model.context_window:
            print(f"⚠️ {model.name} 上下文窗口不足，跳过")
            continue
        
        try:
            print(f"🔄 尝试 {model.name}...")
            response = _call_model(model, messages, system, timeout_seconds)
            print(f"✅ {model.name} 成功")
            return response, model.name
            
        except Exception as e:
            reason = _classify_error(e)
            print(f"❌ {model.name} 失败: {reason.value} - {e}")
            last_error = e
            
            # 某些错误不值得重试（比如 context 太长），直接下一个
            # 某些错误需要等待（rate limit），加个短暂退避
            if reason == FailureReason.RATE_LIMIT:
                time.sleep(2)  # 简单退避
            elif reason == FailureReason.CONTEXT_TOO_LONG:
                pass  # 直接切下一个
    
    raise RuntimeError(f"所有模型均失败，最后错误: {last_error}")


def _call_model(model: ModelConfig, messages: list, system: str, timeout: int) -> str:
    if model.provider == "anthropic":
        client = anthropic.Anthropic()
        resp = client.messages.create(
            model=model.name,
            max_tokens=model.max_tokens,
            system=system,
            messages=messages,
        )
        return resp.content[0].text
    
    elif model.provider == "openai":
        client = openai.OpenAI()
        all_messages = []
        if system:
            all_messages.append({"role": "system", "content": system})
        all_messages.extend(messages)
        resp = client.chat.completions.create(
            model=model.name,
            messages=all_messages,
            max_tokens=model.max_tokens,
        )
        return resp.choices[0].message.content


def _classify_error(e: Exception) -> FailureReason:
    """把各种 SDK 异常统一分类"""
    error_str = str(e).lower()
    
    if "429" in error_str or "rate limit" in error_str or "rate_limit" in error_str:
        return FailureReason.RATE_LIMIT
    elif "context" in error_str or "too long" in error_str or "token" in error_str:
        return FailureReason.CONTEXT_TOO_LONG
    elif "500" in error_str or "502" in error_str or "503" in error_str:
        return FailureReason.SERVER_ERROR
    elif "timeout" in error_str:
        return FailureReason.TIMEOUT
    else:
        return FailureReason.SERVER_ERROR


# 使用示例
if __name__ == "__main__":
    response, model_used = call_with_fallback(
        messages=[{"role": "user", "content": "解释一下量子纠缠"}],
        system="你是一个物理学专家，用简单语言解释复杂概念",
        max_cost_usd=0.10,
    )
    print(f"\n使用模型: {model_used}")
    print(f"响应: {response[:200]}...")
```

---

## OpenClaw 的模型路由机制

OpenClaw 已经内置了模型路由能力，在配置里直接指定：

```yaml
# openclaw config.yaml
model:
  default: anthropic/claude-sonnet-4-6
  fallback:
    - anthropic/claude-haiku-3-5
    - openai/gpt-4o-mini
  routing:
    # 按任务类型路由到不同模型
    coding: anthropic/claude-opus-4-5
    quick_reply: anthropic/claude-haiku-3-5
    analysis: anthropic/claude-sonnet-4-6
```

在 session 里动态切换：
```
/model anthropic/claude-haiku-3-5  # 临时切换到便宜模型
/status                             # 查看当前模型
```

---

## 进阶：带健康检测的智能回退

基础回退是"失败了才切"，进阶做法是**主动探测**，提前知道哪个模型健康：

```python
import asyncio
import time
from typing import Dict

class ModelHealthMonitor:
    """后台持续监测各模型健康状态"""
    
    def __init__(self):
        self.health: Dict[str, dict] = {}
        # 初始化每个模型的健康记录
        for model in FALLBACK_CHAIN:
            self.health[model.name] = {
                "healthy": True,
                "last_check": 0,
                "failure_count": 0,
                "avg_latency_ms": 0,
                "cooldown_until": 0,
            }
    
    def record_success(self, model_name: str, latency_ms: float):
        h = self.health[model_name]
        h["healthy"] = True
        h["failure_count"] = 0
        h["cooldown_until"] = 0
        # 滑动平均延迟
        h["avg_latency_ms"] = h["avg_latency_ms"] * 0.8 + latency_ms * 0.2
    
    def record_failure(self, model_name: str, reason: FailureReason):
        h = self.health[model_name]
        h["failure_count"] += 1
        
        # 连续失败 3 次 → 冷却 60 秒
        if h["failure_count"] >= 3:
            h["healthy"] = False
            cooldown = 60 if reason == FailureReason.RATE_LIMIT else 30
            h["cooldown_until"] = time.time() + cooldown
            print(f"🚨 {model_name} 进入冷却期 {cooldown}s")
    
    def is_available(self, model_name: str) -> bool:
        h = self.health[model_name]
        if not h["healthy"]:
            # 冷却期结束后自动恢复
            if time.time() > h["cooldown_until"]:
                h["healthy"] = True
                h["failure_count"] = 0
                print(f"✅ {model_name} 冷却结束，恢复可用")
                return True
            return False
        return True
    
    def get_best_model(self, candidates: list[ModelConfig]) -> Optional[ModelConfig]:
        """从候选列表中返回最优可用模型（按优先级 + 延迟综合评分）"""
        available = [m for m in candidates if self.is_available(m.name)]
        if not available:
            return None
        # 简单策略：优先级最高的可用模型
        return min(available, key=lambda m: m.priority)


# 全局健康监控器（单例）
health_monitor = ModelHealthMonitor()


def smart_fallback_call(messages: list, system: str = "") -> tuple[str, str]:
    """利用健康监控的智能回退"""
    start = time.time()
    
    for model in FALLBACK_CHAIN:
        if not health_monitor.is_available(model.name):
            print(f"⏭️ {model.name} 在冷却期，直接跳过")
            continue
        
        try:
            t0 = time.time()
            response = _call_model(model, messages, system, timeout=30)
            latency = (time.time() - t0) * 1000
            
            health_monitor.record_success(model.name, latency)
            return response, model.name
            
        except Exception as e:
            reason = _classify_error(e)
            health_monitor.record_failure(model.name, reason)
    
    raise RuntimeError("所有模型均不可用")
```

---

## pi-mono 的实现方式

pi-mono（TypeScript）用类似策略处理模型回退：

```typescript
// packages/agent/src/model-router.ts

interface ModelSpec {
  id: string;
  provider: 'anthropic' | 'openai';
  priority: number;
  maxContextTokens: number;
}

const FALLBACK_CHAIN: ModelSpec[] = [
  { id: 'claude-sonnet-4-5', provider: 'anthropic', priority: 1, maxContextTokens: 200000 },
  { id: 'claude-haiku-3-5',  provider: 'anthropic', priority: 2, maxContextTokens: 200000 },
  { id: 'gpt-4o-mini',       provider: 'openai',    priority: 3, maxContextTokens: 128000 },
];

export async function callWithFallback(
  params: { messages: Message[]; system?: string },
  options: { maxRetries?: number } = {}
): Promise<{ content: string; modelUsed: string }> {
  const errors: Error[] = [];

  for (const model of FALLBACK_CHAIN) {
    try {
      const content = await callModel(model, params);
      return { content, modelUsed: model.id };
    } catch (err) {
      const e = err as Error;
      errors.push(e);
      
      // 429 → 等待后重试同模型（最多1次）
      if (isRateLimitError(e)) {
        await sleep(2000);
        try {
          const content = await callModel(model, params);
          return { content, modelUsed: model.id };
        } catch { /* 继续回退 */ }
      }
      
      console.warn(`[fallback] ${model.id} failed: ${e.message}, trying next...`);
    }
  }

  throw new AggregateError(errors, 'All models in fallback chain failed');
}

function isRateLimitError(e: Error): boolean {
  return e.message.includes('429') || e.message.toLowerCase().includes('rate limit');
}
```

---

## 实战技巧：根据任务类型选模型

不是所有任务都需要最强模型，按任务类型路由能大幅降低成本：

```python
from enum import Enum

class TaskType(Enum):
    QUICK_REPLY = "quick"      # 简单问答 → haiku
    ANALYSIS = "analysis"      # 深度分析 → sonnet
    CODING = "coding"          # 写代码 → opus/sonnet
    CREATIVE = "creative"      # 创意写作 → sonnet
    SUMMARIZE = "summarize"    # 摘要 → haiku

# 任务类型 → 首选模型映射
TASK_MODEL_MAP = {
    TaskType.QUICK_REPLY: ["claude-haiku-3-5", "gpt-4o-mini"],
    TaskType.ANALYSIS:    ["claude-sonnet-4-6", "claude-sonnet-4-5"],
    TaskType.CODING:      ["claude-opus-4-5", "claude-sonnet-4-6"],
    TaskType.CREATIVE:    ["claude-sonnet-4-5", "gpt-4o"],
    TaskType.SUMMARIZE:   ["claude-haiku-3-5", "gpt-4o-mini"],
}

def route_by_task(task_type: TaskType, messages: list, system: str = "") -> str:
    """先用任务偏好模型，失败了再走完整回退链"""
    preferred_models = TASK_MODEL_MAP[task_type]
    
    # 先按偏好顺序试
    for model_name in preferred_models:
        model = next((m for m in FALLBACK_CHAIN if m.name == model_name), None)
        if model:
            try:
                return _call_model(model, messages, system, 30)
            except Exception as e:
                print(f"偏好模型 {model_name} 失败: {e}")
    
    # 偏好模型全挂了，走完整回退链
    response, _ = call_with_fallback(messages, system)
    return response
```

---

## 监控与告警

回退链频繁触发 = 主模型有问题，需要监控：

```python
from dataclasses import dataclass, field
from collections import defaultdict

@dataclass
class FallbackMetrics:
    """记录回退事件，便于告警和分析"""
    total_calls: int = 0
    fallback_events: int = 0
    model_usage: dict = field(default_factory=lambda: defaultdict(int))
    failure_reasons: dict = field(default_factory=lambda: defaultdict(int))
    
    def record(self, model_used: str, had_fallback: bool, reason: str = None):
        self.total_calls += 1
        self.model_usage[model_used] += 1
        if had_fallback:
            self.fallback_events += 1
            if reason:
                self.failure_reasons[reason] += 1
    
    @property
    def fallback_rate(self) -> float:
        return self.fallback_events / max(self.total_calls, 1)
    
    def should_alert(self) -> bool:
        # 回退率超过 20% → 告警
        return self.fallback_rate > 0.2

metrics = FallbackMetrics()

# 使用时
response, model_used = call_with_fallback(messages)
metrics.record(model_used, model_used != "claude-opus-4-5")

if metrics.should_alert():
    print(f"🚨 回退率告警: {metrics.fallback_rate:.1%}")
```

---

## 关键设计原则

| 原则 | 说明 |
|------|------|
| **只回退可恢复错误** | 429/5xx → 回退；400 参数错 → 不回退直接抛 |
| **冷却期避免雪崩** | 连续失败的模型临时下线，别一直往坑里跳 |
| **成本感知路由** | 便宜任务用便宜模型，别用大炮打蚊子 |
| **透明性** | 日志记录用了哪个模型、为什么回退，便于排查 |
| **幂等请求** | 回退时重发请求，确保任务幂等（不要重复扣款） |
| **超时要设置** | 每个模型调用必须有超时，否则一个慢响应堵死链路 |

---

## 小结

多模型回退链的本质是**弹性设计**——不假设任何单一依赖永远可用。

三步走：
1. **定义回退链**：按优先级列出模型，设定成本/上下文过滤条件
2. **统一错误分类**：把各 SDK 的异常归一化，决定哪些触发回退
3. **加健康监控**：主动追踪模型可用性，失败多了就冷却，别死磕

在 OpenClaw 这类 Always-on Agent 里，回退链几乎是必选项——你不知道凌晨 3 点 Anthropic 会不会抖一下。
