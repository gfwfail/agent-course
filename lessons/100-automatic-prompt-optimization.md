# 100 - Agent 自动提示词优化（Automatic Prompt Optimization）

> **第 100 课里程碑！** 🎉
> 让 Agent 基于自身的成功/失败经验自动改进 Prompt，实现"越用越聪明"。

## 为什么需要自动优化？

手写 Prompt 的痛点：
- 凭直觉写，效果不稳定
- 任务变了，Prompt 没跟上
- 调优靠人肉，耗时且主观
- 不同模型需要不同风格

**目标**：Agent 能观察自己的执行效果，自动找出更好的 Prompt。

---

## 核心思路：把 Prompt 当成可优化的参数

```
传统做法：  固定 Prompt → LLM → 输出
自动优化：  Prompt候选集 → 执行 → 评分 → 择优 → 迭代
```

这本质是个**优化问题**：
- **搜索空间**：所有可能的 Prompt 变体
- **目标函数**：任务成功率、用户满意度、准确率等
- **优化方法**：少样本选择、指令改写、反例注入

---

## 三种自动优化策略

### 策略 1：Few-Shot 样本自动筛选（最实用）

不改 Prompt 文本，只自动选"哪些例子放进去"。

```python
# learn-claude-code 风格实现

import json
from dataclasses import dataclass
from typing import Callable
import anthropic

@dataclass
class Example:
    input: str
    output: str
    score: float = 0.0  # 历史表现评分

class FewShotOptimizer:
    """
    维护一个 example 池，根据任务表现动态选最优 few-shot 样本
    """
    
    def __init__(self, client: anthropic.Anthropic, pool_size: int = 50):
        self.client = client
        self.pool: list[Example] = []
        self.pool_size = pool_size
    
    def add_example(self, input: str, output: str, score: float):
        """成功案例入池，失败案例也要记录（负样本有用）"""
        ex = Example(input=input, output=output, score=score)
        self.pool.append(ex)
        # 池满了，淘汰低分的
        if len(self.pool) > self.pool_size:
            self.pool.sort(key=lambda x: x.score, reverse=True)
            self.pool = self.pool[:self.pool_size]
    
    def select_examples(self, query: str, k: int = 3) -> list[Example]:
        """
        为当前 query 选最相关的 k 个高分样本
        简化版：用字符串相似度；生产版：用向量嵌入
        """
        if not self.pool:
            return []
        
        # 按分数加权排序，简化相似度为字符重叠
        scored = []
        for ex in self.pool:
            overlap = len(set(query.split()) & set(ex.input.split()))
            combined = ex.score * 0.7 + overlap * 0.3
            scored.append((combined, ex))
        
        scored.sort(reverse=True)
        return [ex for _, ex in scored[:k]]
    
    def build_prompt(self, system_base: str, query: str) -> tuple[str, str]:
        """动态组装带 few-shot 的 prompt"""
        examples = self.select_examples(query)
        
        if not examples:
            return system_base, query
        
        examples_text = "\n\n".join([
            f"输入：{ex.input}\n输出：{ex.output}"
            for ex in examples
        ])
        
        enhanced_system = f"""{system_base}

## 参考样例
{examples_text}

请参照以上样例的风格和格式回答。"""
        
        return enhanced_system, query

# 使用示例
optimizer = FewShotOptimizer(client=anthropic.Anthropic())

# 记录成功的执行
optimizer.add_example(
    input="把 JSON 转成 CSV",
    output='用 Python pandas: df = pd.read_json(data); df.to_csv("out.csv")',
    score=0.95
)

optimizer.add_example(
    input="解析日期字符串",
    output='from datetime import datetime; dt = datetime.strptime(s, "%Y-%m-%d")',
    score=0.88
)

# 动态构建 prompt
system, user = optimizer.build_prompt(
    system_base="你是一个 Python 代码助手，给出简洁的代码片段。",
    query="把列表转成字典"
)
```

---

### 策略 2：指令自动改写（LLM 改进 LLM 的 Prompt）

用一个"元 LLM"（Meta-LLM）来分析失败案例，提出 Prompt 改进建议。

```python
# DSPy 核心思想的简化实现

class PromptOptimizer:
    """
    基于失败案例，让 LLM 自动改写 system prompt
    """
    
    def __init__(self, client: anthropic.Anthropic):
        self.client = client
        self.current_prompt = ""
        self.failures: list[dict] = []  # 失败案例
        self.version_history: list[dict] = []
    
    def record_failure(self, input: str, output: str, expected: str, reason: str):
        """记录一次失败执行"""
        self.failures.append({
            "input": input,
            "output": output,
            "expected": expected,
            "reason": reason
        })
    
    def optimize(self) -> str:
        """
        当积累足够失败案例后，调用元 LLM 优化 Prompt
        返回新的 system prompt
        """
        if len(self.failures) < 3:
            return self.current_prompt  # 样本不够，先别改
        
        failures_text = json.dumps(self.failures[-10:], ensure_ascii=False, indent=2)
        
        meta_prompt = f"""你是一个 Prompt 工程专家。
        
当前的 system prompt 是：
<current_prompt>
{self.current_prompt}
</current_prompt>

这个 prompt 在以下案例中失败了：
<failures>
{failures_text}
</failures>

请分析失败原因，然后改写 system prompt 来修复这些问题。
要求：
1. 保持原有功能
2. 针对失败模式加入具体指导
3. 不要过度拟合到这几个失败案例
4. 控制 prompt 长度，不要无限增长

只返回新的 system prompt，不要解释。"""

        response = self.client.messages.create(
            model="claude-opus-4-5",
            max_tokens=1000,
            messages=[{"role": "user", "content": meta_prompt}]
        )
        
        new_prompt = response.content[0].text
        
        # 记录版本历史
        self.version_history.append({
            "version": len(self.version_history) + 1,
            "prompt": new_prompt,
            "failures_fixed": len(self.failures)
        })
        
        # 清空已处理的失败案例
        self.failures = []
        self.current_prompt = new_prompt
        
        return new_prompt
```

---

### 策略 3：A/B 测试 + 自动晋级（生产最稳）

同时跑多个 Prompt 变体，用统计显著性决定哪个"赢"。

```typescript
// pi-mono 风格：TypeScript 实现

interface PromptVariant {
  id: string;
  prompt: string;
  wins: number;
  trials: number;
  active: boolean;
}

class PromptABTester {
  private variants: PromptVariant[] = [];
  private evaluator: (output: string, expected: string) => number;

  constructor(evaluator: (output: string, expected: string) => number) {
    this.evaluator = evaluator;
  }

  addVariant(id: string, prompt: string): void {
    this.variants.push({ id, prompt, wins: 0, trials: 0, active: true });
  }

  // Thompson Sampling：根据历史表现概率选择变体
  // 比纯随机更快收敛到最优
  selectVariant(): PromptVariant {
    const activeVariants = this.variants.filter((v) => v.active);
    if (activeVariants.length === 0) throw new Error("No active variants");

    // Beta 分布采样（用 wins/trials 估计成功率）
    const samples = activeVariants.map((v) => {
      const alpha = v.wins + 1; // 先验 Beta(1,1)
      const beta = v.trials - v.wins + 1;
      return { variant: v, sample: this.betaSample(alpha, beta) };
    });

    samples.sort((a, b) => b.sample - a.sample);
    return samples[0].variant;
  }

  private betaSample(alpha: number, beta: number): number {
    // 近似 Beta 分布采样
    const x = this.gammaSample(alpha);
    const y = this.gammaSample(beta);
    return x / (x + y);
  }

  private gammaSample(shape: number): number {
    // Marsaglia-Tsang 方法（简化版）
    if (shape < 1) {
      return this.gammaSample(1 + shape) * Math.pow(Math.random(), 1 / shape);
    }
    const d = shape - 1 / 3;
    const c = 1 / Math.sqrt(9 * d);
    while (true) {
      let x: number, v: number;
      do {
        x = this.normalSample();
        v = 1 + c * x;
      } while (v <= 0);
      v = v * v * v;
      const u = Math.random();
      if (u < 1 - 0.0331 * (x * x) * (x * x)) return d * v;
      if (Math.log(u) < 0.5 * x * x + d * (1 - v + Math.log(v))) return d * v;
    }
  }

  private normalSample(): number {
    let u = 0, v = 0;
    while (u === 0) u = Math.random();
    while (v === 0) v = Math.random();
    return Math.sqrt(-2.0 * Math.log(u)) * Math.cos(2.0 * Math.PI * v);
  }

  async recordResult(
    variantId: string,
    output: string,
    expected: string
  ): Promise<void> {
    const variant = this.variants.find((v) => v.id === variantId);
    if (!variant) return;

    variant.trials++;
    const score = this.evaluator(output, expected);
    if (score > 0.7) variant.wins++; // 阈值可调

    // 统计显著性检验：赢率比最差的高 20% 且样本足够 → 淘汰弱者
    await this.pruneUnderperformers();
  }

  private async pruneUnderperformers(): Promise<void> {
    const qualified = this.variants.filter((v) => v.trials >= 20);
    if (qualified.length < 2) return;

    const rates = qualified.map((v) => ({
      id: v.id,
      rate: v.wins / v.trials,
    }));

    const maxRate = Math.max(...rates.map((r) => r.rate));

    // 淘汰胜率低于最高值 20% 以上的变体
    for (const variant of this.variants) {
      const rate = variant.wins / Math.max(variant.trials, 1);
      if (variant.trials >= 20 && rate < maxRate - 0.2) {
        variant.active = false;
        console.log(`淘汰变体 ${variant.id}，胜率 ${rate.toFixed(2)}`);
      }
    }
  }

  getReport(): string {
    return this.variants
      .map(
        (v) =>
          `${v.id}: ${v.wins}/${v.trials} = ${((v.wins / Math.max(v.trials, 1)) * 100).toFixed(1)}% ${v.active ? "✅" : "❌淘汰"}`
      )
      .join("\n");
  }
}

// 使用示例
const tester = new PromptABTester((output, expected) => {
  // 简单评分：包含关键词得分
  const keywords = expected.split(" ").filter((w) => w.length > 3);
  const found = keywords.filter((k) => output.includes(k)).length;
  return found / Math.max(keywords.length, 1);
});

tester.addVariant("v1", "你是一个专业的代码助手。");
tester.addVariant("v2", "你是一个专业的代码助手。请给出简洁、可直接运行的代码。");
tester.addVariant(
  "v3",
  "你是一个专业的代码助手。优先给出代码，再简短解释。避免废话。"
);
```

---

## OpenClaw 中的实际应用

OpenClaw 的 Skills 系统天然支持 Prompt 优化迭代：

```bash
# SKILL.md 就是 Prompt，可以版本化管理
ls ~/.openclaw/skills/coding-agent/SKILL.md

# 每次改 SKILL.md 本质上就是手动 Prompt 优化
# 自动化这个流程：
# 1. 记录 skill 执行结果到 memory/*.md
# 2. 定期 cron 跑 meta-LLM 分析失败模式
# 3. 自动提 PR 更新 SKILL.md
```

```python
# OpenClaw 风格：基于 memory 文件自动优化 SKILL.md

import re
from pathlib import Path

class SkillOptimizer:
    def __init__(self, skill_path: str, memory_dir: str):
        self.skill_path = Path(skill_path)
        self.memory_dir = Path(memory_dir)
    
    def collect_feedback(self) -> list[dict]:
        """从 memory 文件收集 skill 执行反馈"""
        feedbacks = []
        for md_file in sorted(self.memory_dir.glob("*.md"))[-7:]:  # 近7天
            content = md_file.read_text()
            # 简单解析 skill 执行记录
            # 实际可以更结构化
            if "skill:coding-agent" in content and "FAILED" in content:
                feedbacks.append({
                    "date": md_file.stem,
                    "content": content,
                    "type": "failure"
                })
        return feedbacks
    
    def suggest_improvements(self, client) -> str:
        """让 LLM 分析反馈并建议改进"""
        feedbacks = self.collect_feedback()
        if not feedbacks:
            return "No failures to optimize"
        
        current_skill = self.skill_path.read_text()
        failure_context = "\n---\n".join([f["content"][:500] for f in feedbacks[:5]])
        
        response = client.messages.create(
            model="claude-opus-4-5",
            max_tokens=2000,
            messages=[{
                "role": "user",
                "content": f"""分析这个 Skill 的失败案例，提出改进建议：

当前 SKILL.md:
{current_skill[:2000]}

失败记录（近7天）:
{failure_context}

请给出：
1. 失败根因分析（1-3条）
2. 具体的 SKILL.md 改写建议
3. 改写后的关键片段（用 diff 格式）"""
            }]
        )
        
        return response.content[0].text
```

---

## 完整流水线：持续自动优化

```
┌─────────────────────────────────────────────┐
│              Agent 执行                      │
│  输入 → [Prompt v1] → LLM → 输出            │
└──────────────────┬──────────────────────────┘
                   │
              评分/标注（自动或人工）
                   │
┌──────────────────▼──────────────────────────┐
│           Failure Database                   │
│  input, output, expected, score, timestamp   │
└──────────────────┬──────────────────────────┘
                   │ 积累 N 条后触发
┌──────────────────▼──────────────────────────┐
│           Meta-LLM 分析                      │
│  分析失败模式 → 生成 Prompt 改进建议          │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│          A/B 测试验证                        │
│  新旧 Prompt 并跑 → 统计显著性 → 择优晋级    │
└──────────────────┬──────────────────────────┘
                   │
              [Prompt v2] 上线
                   │
              循环往复 ♻️
```

---

## 实战建议

### ✅ 适合自动优化的场景
- 输出有明确评分标准（代码能跑/不能跑、格式正确/错误）
- 任务量大，有足够的训练信号
- Prompt 逻辑稳定，只需微调措辞

### ❌ 不适合的场景
- 创意写作（没有"正确答案"）
- 低频任务（样本太少统计无意义）
- 核心安全约束（不能让 LLM 自己改护栏）

### 关键工程细节
```python
# 1. 评分要可靠：宁可用简单规则，别用 LLM 评 LLM（会循环偏差）
def score(output: str) -> float:
    # ✅ 好：可执行代码 = 能 import 不报错
    # ✅ 好：JSON 格式 = json.loads() 不抛异常
    # ❌ 差：用 LLM 判断"输出是否好"（偏差大）
    pass

# 2. 保留版本历史：优化可能变差，要能回滚
# 3. 设置人工审核关卡：新 Prompt 上线前 human-in-the-loop
# 4. 防止过拟合：测试集和优化集要分开
```

---

## 总结

| 策略 | 适用场景 | 复杂度 | 效果 |
|------|----------|--------|------|
| Few-Shot 自动筛选 | 有历史成功案例 | 低 | 中 |
| 指令自动改写 | 有明确失败模式 | 中 | 高 |
| A/B 测试 + 晋级 | 高流量生产系统 | 高 | 最稳 |

**核心理念**：Prompt 是代码，代码需要 CI/CD。  
自动 Prompt 优化 = 给 Prompt 加上自动化测试和持续部署。

---

*第 100 课 🎉 | Agent 开发课程 | 2026-03-21*
