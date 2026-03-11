# Agent Reflection：Agent 自我反思

> 第 31 课 - 让 Agent 学会评估和改进自己的输出

## 为什么需要自我反思？

一个常见问题：Agent 第一次生成的回答往往不是最优的。人类在回答复杂问题时会「想一想再说」，Agent 也应该有这个能力。

**Reflection 的核心思想：**
- 先生成初始响应
- 让 Agent 评估自己的输出
- 发现问题后自我修正
- 迭代直到满意

## 基础实现模式

### 1. 简单的 Self-Critique

```typescript
// pi-mono 风格
interface ReflectionResult {
  response: string;
  critique: string;
  isGoodEnough: boolean;
  improvedResponse?: string;
}

async function reflectAndImprove(
  agent: Agent,
  task: string,
  maxIterations: number = 3
): Promise<string> {
  let currentResponse = await agent.generate(task);
  
  for (let i = 0; i < maxIterations; i++) {
    // 让 Agent 自我评估
    const critique = await agent.generate(`
      你刚才对这个问题的回答是:
      """
      ${currentResponse}
      """
      
      请严格评估这个回答:
      1. 有没有事实错误？
      2. 有没有遗漏重要信息？
      3. 逻辑是否清晰？
      4. 是否直接回答了问题？
      
      如果回答已经足够好，输出 "APPROVED"
      否则，指出具体问题并给出改进建议。
    `);
    
    if (critique.includes('APPROVED')) {
      return currentResponse;
    }
    
    // 根据自我批评改进
    currentResponse = await agent.generate(`
      原问题: ${task}
      
      你之前的回答: ${currentResponse}
      
      你自己发现的问题: ${critique}
      
      请基于以上反馈，给出改进后的回答:
    `);
  }
  
  return currentResponse;
}
```

### 2. 结构化反思

```python
# learn-claude-code 风格
from dataclasses import dataclass
from typing import List, Optional

@dataclass
class ReflectionScore:
    accuracy: int      # 1-10
    completeness: int  # 1-10  
    clarity: int       # 1-10
    relevance: int     # 1-10
    issues: List[str]
    suggestions: List[str]

class ReflectiveAgent:
    def __init__(self, llm, threshold: int = 7):
        self.llm = llm
        self.threshold = threshold  # 最低可接受分数
    
    async def reflect(self, task: str, response: str) -> ReflectionScore:
        """让 Agent 对自己的回答打分"""
        prompt = f"""
        任务: {task}
        回答: {response}
        
        请以 JSON 格式评估这个回答:
        {{
          "accuracy": <1-10>,
          "completeness": <1-10>,
          "clarity": <1-10>,
          "relevance": <1-10>,
          "issues": ["问题1", "问题2"],
          "suggestions": ["建议1", "建议2"]
        }}
        """
        result = await self.llm.generate(prompt, response_format="json")
        return ReflectionScore(**result)
    
    async def run_with_reflection(self, task: str) -> str:
        response = await self.llm.generate(task)
        
        for _ in range(3):  # 最多反思3轮
            score = await self.reflect(task, response)
            avg_score = (score.accuracy + score.completeness + 
                        score.clarity + score.relevance) / 4
            
            if avg_score >= self.threshold:
                return response
            
            # 基于反馈改进
            improvement_prompt = f"""
            原任务: {task}
            当前回答: {response}
            
            发现的问题:
            {chr(10).join(f'- {i}' for i in score.issues)}
            
            改进建议:
            {chr(10).join(f'- {s}' for s in score.suggestions)}
            
            请改进你的回答:
            """
            response = await self.llm.generate(improvement_prompt)
        
        return response
```

## 实际应用：代码生成的 Reflection

代码生成是 Reflection 最实用的场景之一：

```typescript
// OpenClaw/pi-mono 代码反思模式
interface CodeReflection {
  code: string;
  issues: {
    type: 'bug' | 'style' | 'performance' | 'security';
    description: string;
    line?: number;
  }[];
  tests: string[];
  confidence: number;
}

async function generateCodeWithReflection(
  agent: Agent,
  requirement: string
): Promise<string> {
  // 第一步：生成代码
  let code = await agent.generate(`
    请根据以下需求生成代码:
    ${requirement}
    
    只输出代码，不要解释。
  `);
  
  // 第二步：自我 Review
  const review = await agent.generate(`
    请 review 以下代码，找出问题:
    
    \`\`\`
    ${code}
    \`\`\`
    
    检查:
    1. 是否有 bug 或边界情况未处理
    2. 是否有安全漏洞
    3. 性能是否合理
    4. 代码风格是否清晰
    
    以 JSON 格式返回 issues 列表。
  `);
  
  // 第三步：根据 Review 修复
  if (review.issues?.length > 0) {
    code = await agent.generate(`
      原代码:
      \`\`\`
      ${code}
      \`\`\`
      
      发现的问题:
      ${JSON.stringify(review.issues, null, 2)}
      
      请修复这些问题，输出完整的改进后代码:
    `);
  }
  
  // 第四步：生成测试来验证
  const tests = await agent.generate(`
    为以下代码生成单元测试:
    \`\`\`
    ${code}
    \`\`\`
    
    覆盖正常情况和边界情况。
  `);
  
  return code;
}
```

## 高级模式：Chain of Verification

更复杂的场景需要多步验证：

```typescript
// Chain of Verification (CoVe) 模式
async function chainOfVerification(
  agent: Agent,
  query: string
): Promise<string> {
  // 1. 生成初始回答
  const initial = await agent.generate(query);
  
  // 2. 生成验证问题
  const verificationQuestions = await agent.generate(`
    你刚才回答了这个问题: "${query}"
    你的回答是: "${initial}"
    
    请生成 3-5 个可以用来验证这个回答正确性的问题。
    这些问题应该能帮助检查回答中的事实是否准确。
  `);
  
  // 3. 独立回答每个验证问题
  const verifications = [];
  for (const q of verificationQuestions) {
    // 注意：不带上下文，独立回答
    const answer = await agent.generate(q, { 
      systemPrompt: '简洁回答以下问题，只基于你的知识:' 
    });
    verifications.push({ question: q, answer });
  }
  
  // 4. 基于验证结果修正
  const finalAnswer = await agent.generate(`
    原问题: ${query}
    初始回答: ${initial}
    
    验证结果:
    ${verifications.map(v => `Q: ${v.question}\nA: ${v.answer}`).join('\n\n')}
    
    基于验证结果，修正你的初始回答。
    如果验证发现初始回答有错误，请更正。
    如果初始回答正确，可以保持不变或补充细节。
  `);
  
  return finalAnswer;
}
```

## OpenClaw 中的 Reflection 实践

```typescript
// OpenClaw 工具执行后的反思
async function executeWithReflection(
  toolCall: ToolCall,
  result: ToolResult
): Promise<ReflectionAction> {
  // 工具执行后，评估结果
  const reflection = await this.reflect(`
    我刚刚执行了工具: ${toolCall.name}
    参数: ${JSON.stringify(toolCall.params)}
    结果: ${result.output}
    
    请评估:
    1. 这个结果是否符合预期？
    2. 是否需要采取后续行动？
    3. 是否有潜在风险需要注意？
    
    返回: CONTINUE（继续下一步）/ RETRY（重试）/ STOP（停止并报告）
  `);
  
  return parseReflectionAction(reflection);
}
```

## Reflection 的成本权衡

Reflection 增加 API 调用次数，需要权衡：

| 场景 | 是否使用 Reflection | 原因 |
|------|-------------------|------|
| 简单问答 | ❌ | 成本不值得 |
| 代码生成 | ✅ | bug 代价高于多几次调用 |
| 重要决策 | ✅ | 准确性优先 |
| 实时对话 | ❌ | 延迟敏感 |
| 批量任务 | ⚠️ | 可选择性抽检 |

```typescript
// 智能选择是否反思
function shouldReflect(task: Task): boolean {
  // 高风险任务必须反思
  if (task.risk === 'high') return true;
  
  // 代码相关任务建议反思
  if (task.type === 'code') return true;
  
  // 成本敏感场景跳过
  if (task.budget === 'minimal') return false;
  
  // 用户明确要求
  if (task.userRequestedReview) return true;
  
  return false;
}
```

## 关键要点

1. **Reflection 不是万能的** - 用在值得的地方
2. **设置终止条件** - 避免无限循环反思
3. **独立验证** - 验证时不要带偏见（上下文隔离）
4. **结构化评分** - 用数字而非模糊判断
5. **成本意识** - 每次反思都是一次 API 调用

## 思考题

1. 如何检测 Agent 的自我评估是否客观？
2. Reflection 和 Chain-of-Thought 有什么区别？
3. 在什么情况下，Reflection 可能会让结果变差？

---

下节预告：**Agent Orchestration Patterns - Agent 编排模式**
