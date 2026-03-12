# Prompt Chaining：提示链，让 Agent 完成复杂任务

> 第 35 课 - 2026-03-13

## 什么是 Prompt Chaining？

**Prompt Chaining** 是将复杂任务分解为多个步骤，每个步骤的输出作为下一个步骤的输入，形成一条"链"。

想象一下写一篇文章的过程：
1. 先做大纲
2. 根据大纲写初稿
3. 审查初稿找问题
4. 修改润色
5. 最终检查

这就是 Prompt Chaining 的核心思想：**分而治之**。

## 为什么需要 Prompt Chaining？

### 单次调用的局限

```
用户: 帮我分析这个代码库，找出所有安全漏洞，修复它们，写测试，更新文档
```

一次性让 LLM 处理这么多事情，问题很多：
- **Context 爆炸**：信息太多，关键点被淹没
- **注意力分散**：LLM 可能漏掉重要细节
- **错误累积**：前面的错误会影响后面
- **难以调试**：出错了不知道哪里有问题

### Prompt Chaining 的优势

```
Step 1: 扫描代码库结构 → 输出文件列表
Step 2: 分析每个文件的安全风险 → 输出漏洞报告
Step 3: 对每个漏洞生成修复方案 → 输出补丁
Step 4: 为修复生成测试 → 输出测试代码
Step 5: 更新相关文档 → 完成
```

每一步都专注一件事，结果更可靠。

## Prompt Chaining 的核心模式

### 模式一：Sequential Chain（顺序链）

最简单的模式，步骤线性执行：

```typescript
// pi-mono 风格的顺序链
async function sequentialChain(input: string): Promise<string> {
  // Step 1: 理解需求
  const understanding = await llm.complete({
    messages: [{
      role: 'user',
      content: `分析这个需求，提取关键点：\n${input}`
    }]
  });
  
  // Step 2: 生成方案
  const plan = await llm.complete({
    messages: [{
      role: 'user',
      content: `基于这些关键点，设计实现方案：\n${understanding}`
    }]
  });
  
  // Step 3: 生成代码
  const code = await llm.complete({
    messages: [{
      role: 'user',
      content: `根据方案写代码：\n${plan}`
    }]
  });
  
  return code;
}
```

### 模式二：Branching Chain（分支链）

根据中间结果决定下一步：

```typescript
// OpenClaw 中的分支逻辑
async function branchingChain(task: string) {
  // Step 1: 分类任务
  const classification = await llm.complete({
    messages: [{
      role: 'user',
      content: `将任务分类为：code/research/writing\n任务：${task}`
    }]
  });
  
  // Step 2: 根据分类走不同分支
  switch (classification.trim()) {
    case 'code':
      return await handleCodeTask(task);
    case 'research':
      return await handleResearchTask(task);
    case 'writing':
      return await handleWritingTask(task);
    default:
      return await handleGenericTask(task);
  }
}
```

### 模式三：Loop Chain（循环链）

迭代直到满足条件：

```typescript
// learn-claude-code 风格的循环链
async function loopChain(draft: string, maxIterations = 3) {
  let current = draft;
  
  for (let i = 0; i < maxIterations; i++) {
    // 评估当前结果
    const evaluation = await llm.complete({
      messages: [{
        role: 'user',
        content: `评估这段代码，列出问题（如果没问题返回 LGTM）：\n${current}`
      }]
    });
    
    // 满足条件就退出
    if (evaluation.includes('LGTM')) {
      return current;
    }
    
    // 根据反馈改进
    current = await llm.complete({
      messages: [{
        role: 'user',
        content: `根据反馈修改代码：\n反馈：${evaluation}\n代码：${current}`
      }]
    });
  }
  
  return current;
}
```

### 模式四：MapReduce Chain（映射归约链）

并行处理多个输入，然后合并结果：

```typescript
// 分析多个文件的安全漏洞
async function mapReduceChain(files: string[]) {
  // Map: 并行分析每个文件
  const analyses = await Promise.all(
    files.map(file => 
      llm.complete({
        messages: [{
          role: 'user',
          content: `分析这个文件的安全漏洞：\n${file}`
        }]
      })
    )
  );
  
  // Reduce: 汇总所有分析结果
  const summary = await llm.complete({
    messages: [{
      role: 'user',
      content: `汇总这些安全分析，按严重程度排序：\n${analyses.join('\n---\n')}`
    }]
  });
  
  return summary;
}
```

## 实战：OpenClaw 中的 Prompt Chaining

### 案例一：Skill 执行链

当 Agent 执行一个 Skill 时：

```typescript
// 1. 先判断是否需要这个 Skill
const needsSkill = await evaluateSkillMatch(userRequest, skillDescription);

// 2. 如果需要，加载 Skill 内容
if (needsSkill) {
  const skillContent = await readSkillFile(skillPath);
  
  // 3. 执行 Skill 的指令
  const result = await executeWithSkillContext(userRequest, skillContent);
  
  return result;
}
```

### 案例二：代码审查链

```typescript
async function codeReviewChain(pr: PullRequest) {
  // Step 1: 获取 diff
  const diff = await getDiff(pr);
  
  // Step 2: 分析改动意图
  const intent = await llm.complete({
    messages: [{
      role: 'user',
      content: `这个 PR 在做什么？\n${diff}`
    }]
  });
  
  // Step 3: 检查潜在问题
  const issues = await llm.complete({
    messages: [{
      role: 'user',
      content: `检查这些改动可能的问题：
意图：${intent}
改动：${diff}`
    }]
  });
  
  // Step 4: 生成建议
  const suggestions = await llm.complete({
    messages: [{
      role: 'user',
      content: `基于这些问题，给出具体的改进建议：
问题：${issues}`
    }]
  });
  
  // Step 5: 格式化评论
  return formatAsGitHubComment(intent, issues, suggestions);
}
```

## Prompt Chaining 的关键技巧

### 1. 明确的交接格式

每个步骤输出的格式要清晰，方便下一步解析：

```typescript
// Bad: 自由格式，难解析
const result = await llm.complete({
  content: "分析这段代码"
});

// Good: 结构化输出
const result = await llm.complete({
  content: `分析这段代码，用以下格式输出：
## 功能
(代码做什么)

## 问题
- 问题1
- 问题2

## 建议
- 建议1
- 建议2`
});
```

### 2. 保留关键上下文

链条越长，前面的信息越容易丢失：

```typescript
// 保留关键信息传递给后续步骤
interface ChainContext {
  originalRequest: string;  // 始终保留原始请求
  keyDecisions: string[];   // 记录关键决策
  currentStep: number;
  artifacts: Map<string, string>;  // 中间产物
}

async function chainWithContext(ctx: ChainContext) {
  const result = await llm.complete({
    messages: [{
      role: 'system',
      content: `原始需求：${ctx.originalRequest}
已做决策：${ctx.keyDecisions.join('\n')}`
    }, {
      role: 'user',
      content: `执行第 ${ctx.currentStep} 步...`
    }]
  });
  
  return result;
}
```

### 3. 错误处理与回退

链条中任何一步都可能失败：

```typescript
async function resilientChain(input: string) {
  const steps = [analyzeStep, planStep, executeStep, verifyStep];
  let result = input;
  let lastSuccessful = -1;
  
  for (let i = 0; i < steps.length; i++) {
    try {
      result = await steps[i](result);
      lastSuccessful = i;
    } catch (error) {
      console.error(`Step ${i} failed:`, error);
      
      // 尝试恢复
      if (i > 0) {
        console.log(`Retrying from step ${lastSuccessful + 1}`);
        result = await retryWithMoreContext(steps[i], result, error);
      } else {
        throw error;
      }
    }
  }
  
  return result;
}
```

### 4. 成本控制

长链条会消耗很多 token：

```typescript
// 根据任务复杂度选择链条深度
function selectChainDepth(task: string): number {
  const complexity = estimateComplexity(task);
  
  if (complexity < 0.3) return 1;  // 简单任务，一步完成
  if (complexity < 0.6) return 2;  // 中等任务，两步
  return 3;                         // 复杂任务，完整链条
}

// 对中间步骤用更便宜的模型
async function costEfficientChain(input: string) {
  // 分类用小模型
  const category = await llm.complete({
    model: 'gpt-4o-mini',
    messages: [{ role: 'user', content: `分类：${input}` }]
  });
  
  // 核心生成用大模型
  const result = await llm.complete({
    model: 'claude-opus',
    messages: [{ role: 'user', content: `生成：${input}` }]
  });
  
  return result;
}
```

## 与 Agent Loop 的关系

Prompt Chaining 和 Agent Loop 是互补的：

| Prompt Chaining | Agent Loop |
|-----------------|------------|
| 预定义的步骤序列 | 动态决定下一步 |
| 适合结构化任务 | 适合探索性任务 |
| 可预测的执行路径 | 灵活的执行路径 |
| 更容易测试 | 更难预测 |

实际中经常结合使用：

```typescript
// Agent Loop 中使用 Prompt Chaining
async function agentLoop(task: string) {
  while (true) {
    const action = await decideAction(task);
    
    if (action.type === 'complex_operation') {
      // 对复杂操作使用 Prompt Chain
      const result = await complexOperationChain(action.params);
      context.add(result);
    } else if (action.type === 'tool_call') {
      const result = await executeTool(action.tool, action.params);
      context.add(result);
    } else if (action.type === 'done') {
      return action.result;
    }
  }
}
```

## 总结

**Prompt Chaining 核心要点：**

1. **分解复杂任务** - 每步只做一件事
2. **选择合适的链模式** - Sequential / Branching / Loop / MapReduce
3. **结构化交接** - 清晰的输入输出格式
4. **保留关键上下文** - 防止信息丢失
5. **优雅处理错误** - 失败时能恢复或回退
6. **控制成本** - 简单步骤用小模型

**什么时候用 Prompt Chaining：**
- 任务有明确的阶段划分
- 需要多次验证和改进
- 处理大量类似的输入
- 需要可解释、可调试的执行过程

下一课我们将讨论 **Tool Composition（工具组合）**，看看如何将多个工具组合成更强大的能力。

---

练习题：
1. 设计一个代码重构的 Prompt Chain，包含理解、重构、测试三个步骤
2. 实现一个带 Loop 的文档生成链，直到用户满意为止
3. 用 MapReduce 模式实现多文件的代码审查
