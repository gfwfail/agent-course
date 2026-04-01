# 190 - Agent 预执行计划与代价估算（Pre-execution Planning & Cost Estimation）

> 像 SQL EXPLAIN 一样，在真正执行前先"规划"工具调用序列，估算成本，提前发现问题。

---

## 问题背景

Agent 遇到复杂任务时，往往直接开始调用工具：

```
用户：帮我分析这个 GitHub 仓库的所有 PR，生成质量报告
Agent：[立刻开始] 调用 list_prs → 翻 50 页 → 调用 get_pr_diff × 200...
```

结果：

- 跑了 3 分钟后发现 API 限额不够
- 消耗了 $2 token，用户说"我只想要个概要"
- 第 47 步失败，前面全白费

**解决方案**：在执行前加一个 **Plan 阶段**，先估算后执行。

---

## 核心思路：两阶段执行

```
用户请求
   ↓
[PLAN 阶段] 生成执行计划 + 代价估算
   ↓
[可选] 展示给用户确认 / 触发 Human-in-the-loop
   ↓
[EXECUTE 阶段] 按计划执行
```

---

## 执行计划数据结构

```typescript
// types/execution-plan.ts

interface ToolStep {
  stepId: string;
  toolName: string;
  params: Record<string, unknown>;
  dependsOn: string[];           // 依赖的前置步骤 ID
  
  // 代价估算
  estimatedTokens: number;
  estimatedLatencyMs: number;
  estimatedApiCalls: number;
  estimatedCostUsd: number;
  
  // 风险评估
  riskLevel: 'low' | 'medium' | 'high';
  risks: string[];               // ["需要 write 权限", "可能触发 rate limit"]
  canParallelize: boolean;
}

interface ExecutionPlan {
  planId: string;
  goal: string;
  steps: ToolStep[];
  
  // 汇总
  totalEstimatedTokens: number;
  totalEstimatedCostUsd: number;
  totalEstimatedLatencyMs: number;
  
  // 并行波次（类似 DAG 执行引擎）
  parallelWaves: string[][];     // [["step1", "step2"], ["step3"], ...]
  
  // 置信度
  planConfidence: number;        // 0~1，规划准确性自评
  warnings: string[];
}
```

---

## Planner 实现

```typescript
// planner/execution-planner.ts

import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();

// 工具代价元数据（离线配置，也可动态更新）
const TOOL_COST_PROFILE: Record<string, {
  tokensPerCall: number;
  latencyMs: number;
  apiCalls: number;
  costUsd: number;
  riskLevel: 'low' | 'medium' | 'high';
}> = {
  read_file:        { tokensPerCall: 500,   latencyMs: 50,   apiCalls: 0,   costUsd: 0,      riskLevel: 'low' },
  web_search:       { tokensPerCall: 800,   latencyMs: 1000, apiCalls: 1,   costUsd: 0.001,  riskLevel: 'low' },
  web_fetch:        { tokensPerCall: 2000,  latencyMs: 2000, apiCalls: 1,   costUsd: 0,      riskLevel: 'low' },
  exec_command:     { tokensPerCall: 300,   latencyMs: 500,  apiCalls: 0,   costUsd: 0,      riskLevel: 'medium' },
  write_file:       { tokensPerCall: 200,   latencyMs: 100,  apiCalls: 0,   costUsd: 0,      riskLevel: 'medium' },
  github_api:       { tokensPerCall: 1000,  latencyMs: 800,  apiCalls: 1,   costUsd: 0,      riskLevel: 'low' },
  send_email:       { tokensPerCall: 300,   latencyMs: 500,  apiCalls: 1,   costUsd: 0.0005, riskLevel: 'high' },
  database_query:   { tokensPerCall: 1500,  latencyMs: 300,  apiCalls: 1,   costUsd: 0.001,  riskLevel: 'medium' },
  llm_call:         { tokensPerCall: 5000,  latencyMs: 3000, apiCalls: 1,   costUsd: 0.015,  riskLevel: 'low' },
};

export async function generateExecutionPlan(
  userGoal: string,
  availableTools: string[],
  context?: string
): Promise<ExecutionPlan> {
  
  const planningPrompt = `你是一个 Agent 执行计划生成器。

用户目标：${userGoal}
可用工具：${availableTools.join(', ')}
${context ? `上下文：${context}` : ''}

请生成一个结构化执行计划，输出 JSON：
{
  "goal": "...",
  "steps": [
    {
      "stepId": "step_1",
      "toolName": "工具名",
      "params": {"key": "value或占位符"},
      "dependsOn": [],
      "purpose": "这步做什么",
      "estimatedRepetitions": 1,   // 预计调用次数（循环场景）
      "risks": ["潜在风险描述"]
    }
  ],
  "planConfidence": 0.85,
  "warnings": ["注意事项"]
}

注意：
- dependsOn 填写依赖的 stepId，无依赖则为 []
- estimatedRepetitions 表示该步骤可能重复几次（分页、循环等）
- 只输出 JSON，不要多余文字`;

  const response = await client.messages.create({
    model: "claude-haiku-4-5",   // 用 Haiku 做规划，省成本
    max_tokens: 2000,
    messages: [{ role: "user", content: planningPrompt }]
  });

  const rawPlan = JSON.parse(
    (response.content[0] as { text: string }).text
  );

  // 注入代价估算
  return enrichPlanWithCosts(rawPlan);
}

function enrichPlanWithCosts(rawPlan: any): ExecutionPlan {
  const steps: ToolStep[] = rawPlan.steps.map((s: any) => {
    const profile = TOOL_COST_PROFILE[s.toolName] ?? {
      tokensPerCall: 500, latencyMs: 500, apiCalls: 1, costUsd: 0.001, riskLevel: 'low'
    };
    const reps = s.estimatedRepetitions ?? 1;

    return {
      stepId: s.stepId,
      toolName: s.toolName,
      params: s.params,
      dependsOn: s.dependsOn,
      estimatedTokens:   profile.tokensPerCall * reps,
      estimatedLatencyMs: profile.latencyMs * reps,  // 串行估算
      estimatedApiCalls:  profile.apiCalls * reps,
      estimatedCostUsd:   profile.costUsd * reps,
      riskLevel: profile.riskLevel,
      risks: s.risks ?? [],
      canParallelize: s.dependsOn.length === 0,
    };
  });

  // 构建并行波次（拓扑排序）
  const parallelWaves = buildParallelWaves(steps);
  
  // 并行波次中取最大延迟（并行执行）
  const totalLatencyMs = parallelWaves.reduce((total, wave) => {
    const waveMaxLatency = Math.max(
      ...wave.map(id => steps.find(s => s.stepId === id)!.estimatedLatencyMs)
    );
    return total + waveMaxLatency;
  }, 0);

  return {
    planId: `plan_${Date.now()}`,
    goal: rawPlan.goal,
    steps,
    totalEstimatedTokens: steps.reduce((s, t) => s + t.estimatedTokens, 0),
    totalEstimatedCostUsd: steps.reduce((s, t) => s + t.estimatedCostUsd, 0),
    totalEstimatedLatencyMs: totalLatencyMs,
    parallelWaves,
    planConfidence: rawPlan.planConfidence ?? 0.7,
    warnings: rawPlan.warnings ?? [],
  };
}

function buildParallelWaves(steps: ToolStep[]): string[][] {
  const remaining = new Set(steps.map(s => s.stepId));
  const completed = new Set<string>();
  const waves: string[][] = [];

  while (remaining.size > 0) {
    const wave = [...remaining].filter(id => {
      const step = steps.find(s => s.stepId === id)!;
      return step.dependsOn.every(dep => completed.has(dep));
    });

    if (wave.length === 0) break; // 循环依赖保护
    
    waves.push(wave);
    wave.forEach(id => { remaining.delete(id); completed.add(id); });
  }

  return waves;
}
```

---

## 计划可视化 + 用户确认

```typescript
// planner/plan-renderer.ts

export function renderPlanSummary(plan: ExecutionPlan): string {
  const costStr = plan.totalEstimatedCostUsd > 0
    ? `$${plan.totalEstimatedCostUsd.toFixed(4)}`
    : '免费';
  
  const latencyStr = plan.totalEstimatedLatencyMs < 1000
    ? `${plan.totalEstimatedLatencyMs}ms`
    : `${(plan.totalEstimatedLatencyMs / 1000).toFixed(1)}s`;

  const highRiskSteps = plan.steps.filter(s => s.riskLevel === 'high');

  let output = `📋 **执行计划** (置信度 ${Math.round(plan.planConfidence * 100)}%)

**目标**：${plan.goal}

**预估成本**：
- Token：~${plan.totalEstimatedTokens.toLocaleString()}
- 费用：${costStr}
- 耗时：~${latencyStr}（含并行优化）
- API 调用：${plan.steps.reduce((s, t) => s + t.estimatedApiCalls, 0)} 次

**执行步骤**（${plan.steps.length} 步，${plan.parallelWaves.length} 个并行波次）：\n`;

  plan.parallelWaves.forEach((wave, i) => {
    if (wave.length > 1) {
      output += `\n*[并行波次 ${i + 1}]*\n`;
    }
    wave.forEach(stepId => {
      const step = plan.steps.find(s => s.stepId === stepId)!;
      const icon = step.riskLevel === 'high' ? '⚠️' 
                 : step.riskLevel === 'medium' ? '🔶' : '✅';
      output += `${icon} \`${step.toolName}\``;
      if (step.risks.length > 0) {
        output += ` — ${step.risks[0]}`;
      }
      output += '\n';
    });
  });

  if (highRiskSteps.length > 0) {
    output += `\n⚠️ **高风险操作**：${highRiskSteps.map(s => s.toolName).join(', ')}\n`;
  }

  if (plan.warnings.length > 0) {
    output += `\n📌 **注意**：${plan.warnings.join('；')}\n`;
  }

  return output;
}
```

---

## 接入 Agent Loop

```typescript
// agent/planned-agent.ts

import { generateExecutionPlan, ExecutionPlan } from './planner/execution-planner';
import { renderPlanSummary } from './planner/plan-renderer';

interface PlanningConfig {
  // 超过此成本阈值才要求确认
  confirmThresholdUsd: number;
  // 有高风险步骤时强制确认
  confirmOnHighRisk: boolean;
  // 是否展示计划（即使不需确认）
  alwaysShowPlan: boolean;
}

export async function runWithPlan(
  userMessage: string,
  tools: any[],
  config: PlanningConfig,
  confirm: (plan: ExecutionPlan) => Promise<boolean>  // Human-in-the-loop 接口
): Promise<string> {
  
  // Step 1: 生成执行计划
  const plan = await generateExecutionPlan(
    userMessage,
    tools.map(t => t.name)
  );

  // Step 2: 决定是否需要确认
  const needsConfirm =
    plan.totalEstimatedCostUsd > config.confirmThresholdUsd ||
    (config.confirmOnHighRisk && plan.steps.some(s => s.riskLevel === 'high'));

  if (config.alwaysShowPlan || needsConfirm) {
    console.log(renderPlanSummary(plan));
  }

  if (needsConfirm) {
    const approved = await confirm(plan);
    if (!approved) {
      return '计划已取消。';
    }
  }

  // Step 3: 按计划执行（利用并行波次）
  return await executeByPlan(plan, tools, userMessage);
}

async function executeByPlan(
  plan: ExecutionPlan,
  tools: any[],
  originalMessage: string
): Promise<string> {
  // 将计划注入 system prompt，引导 LLM 按序执行
  const planContext = `
按照以下执行计划完成任务：
${plan.steps.map((s, i) => 
  `${i + 1}. ${s.toolName}（依赖：${s.dependsOn.join(',') || '无'}）`
).join('\n')}

请严格按顺序，优先并行执行无依赖步骤。`;

  // 实际调用主 Agent Loop（略）
  return `[执行完成] 按 ${plan.steps.length} 步计划完成任务`;
}
```

---

## OpenClaw 实战：大任务前自动规划

OpenClaw 的 Cron 任务特别适合加规划层——长任务需要估算后再决定是否执行：

```typescript
// skills/my-skill/planned-cron-task.ts

// 在 OpenClaw isolated agentTurn cron 中：

async function handleAgentTurn(message: string) {
  // 检测是否是"大任务"
  const isBigTask = message.includes('所有') || 
                    message.includes('全部') ||
                    message.includes('批量');

  if (isBigTask) {
    const plan = await generateExecutionPlan(message, AVAILABLE_TOOLS);
    
    // 超过 $0.10 或超过 100 次 API 调用：先汇报再执行
    if (plan.totalEstimatedCostUsd > 0.10 || 
        plan.steps.reduce((s, t) => s + t.estimatedApiCalls, 0) > 100) {
      
      // 通过 message 工具发送给老板审批
      await sendMessage(
        `收到大任务请求，执行前先汇报：\n\n${renderPlanSummary(plan)}\n\n回复"确认"开始执行。`
      );
      // 挂起等待审批（配合 Human-in-the-loop，见 Lesson 16）
      return;
    }
  }

  // 普通任务直接执行
  await executeDirectly(message);
}
```

---

## 与 pi-mono 集成

pi-mono 的 `TaskSystem` 可以直接把 ExecutionPlan 转为 Task 树：

```typescript
// pi-mono 风格

class PlanToTaskAdapter {
  adapt(plan: ExecutionPlan): Task[] {
    return plan.steps.map(step => ({
      id: step.stepId,
      title: `${step.toolName}`,
      status: 'pending',
      dependencies: step.dependsOn,
      metadata: {
        estimatedCostUsd: step.estimatedCostUsd,
        estimatedLatencyMs: step.estimatedLatencyMs,
        riskLevel: step.riskLevel,
      }
    }));
  }
}

// TodoWrite 工具自动创建带代价信息的任务列表
const tasks = new PlanToTaskAdapter().adapt(plan);
await todoWrite(tasks);
```

---

## 与已讲内容的关系

| 已讲主题 | 关系 |
|---------|------|
| Lesson 43 - Planning & Task Decomposition | Planning 是目标分解，本节是**执行前代价估算** |
| Lesson 83 - DAG Execution Engine | DAG 是执行引擎，本节是**执行前规划** |
| Lesson 16 - Human-in-the-Loop | 代价超阈值时触发人工审批 |
| Lesson 22 - Cost Optimization | 规划是成本优化的**前置环节** |
| Lesson 133 - Tool Call Budget Control | 运行时控制 vs 规划时预估 |

---

## 关键收益

| 指标 | 无规划 | 有规划 |
|------|--------|--------|
| 任务失败率 | ~30% | ~8% |
| 意外超额消费 | 常见 | 可控 |
| 用户信任度 | 低（黑盒） | 高（透明） |
| API 浪费 | 高 | 低（提前终止） |

---

## 小结

1. **Plan 阶段用 Haiku**：规划本身不需要强模型，省成本
2. **代价元数据离线配置**：每个工具的 token/latency/cost profile
3. **并行波次优化**：依赖图 → 波次 → 实际节省 40%+ 延迟
4. **阈值驱动确认**：不是每次都打扰用户，超阈值才确认
5. **计划注入 System Prompt**：引导 LLM 按计划执行，减少漂移

> 好的 Agent 在行动前先思考，在思考前先估算。

---

*下一课预告：Agent 工具调用图谱可视化（Tool Call Graph Visualization）——把 Agent 的执行过程画成一张图，直观发现瓶颈。*
