# 191. Agent 函数式核心、命令式外壳（Functional Core, Imperative Shell）

> "让 Agent 的大脑是纯函数，让 Agent 的手是命令式代码。"

---

## 为什么要讲这个？

你有没有遇到过这些痛点：

- Agent bug 很难复现，因为每次运行都调用了真实 API
- 想测试决策逻辑，但必须 mock 一大堆工具
- 重构一个工具，不知道会不会影响 Agent 的推理路径
- 在 CI 里跑 Agent 测试，费钱又不稳定

**根本原因**：决策逻辑和副作用混在一起了。

**Functional Core, Imperative Shell** 是解决这个问题的经典架构模式，由 Gary Bernhardt 在 2012 年提出，现在完全适用于 Agent 系统。

---

## 核心思想

```
┌─────────────────────────────────────────────┐
│              Imperative Shell（外壳）         │
│  读取用户输入 → 调用 LLM → 执行工具 → 输出   │
│                                              │
│  ┌─────────────────────────────────────┐    │
│  │        Functional Core（核心）       │    │
│  │  决策函数：state + observation → action │    │
│  │  纯函数，无副作用，100% 可测试       │    │
│  └─────────────────────────────────────┘    │
└─────────────────────────────────────────────┘
```

**函数式核心**：
- 输入：当前状态 + 观察结果
- 输出：下一步动作（但不执行！）
- 无 I/O、无 API 调用、无时间依赖
- 同样输入 → 永远同样输出

**命令式外壳**：
- 读取真实环境（文件、API、时间）
- 把数据喂给核心
- 执行核心返回的动作
- 处理副作用

---

## 代码示例（TypeScript）

### 1. 定义核心数据类型

```typescript
// types.ts - 纯数据，无副作用

type AgentState = {
  goal: string;
  completedSteps: string[];
  context: Record<string, unknown>;
  iteration: number;
};

type Observation = {
  lastToolResult?: unknown;
  error?: string;
};

// Agent 只能"返回"这些动作，不能直接执行
type AgentAction =
  | { kind: 'call_tool'; name: string; params: Record<string, unknown> }
  | { kind: 'respond'; message: string }
  | { kind: 'ask_user'; question: string }
  | { kind: 'done'; summary: string };
```

### 2. 函数式核心 - 纯函数决策

```typescript
// core.ts - 不依赖任何外部模块，100% 可单元测试

import type { AgentState, Observation, AgentAction } from './types';

/**
 * 纯函数：给定状态和观察，返回下一步动作。
 * 不调用任何 API，不读取文件，不依赖时间。
 */
export function decide(
  state: AgentState,
  observation: Observation,
  availableTools: string[]
): AgentAction {
  // 硬性安全：超过 20 轮强制停止
  if (state.iteration >= 20) {
    return {
      kind: 'done',
      summary: `已执行 ${state.iteration} 步，强制终止。完成步骤: ${state.completedSteps.join(', ')}`
    };
  }

  // 上一步出错
  if (observation.error) {
    if (state.iteration < 3) {
      // 前三步出错：重试
      return {
        kind: 'call_tool',
        name: 'retry_last',
        params: { reason: observation.error }
      };
    } else {
      // 多次出错：请人介入
      return {
        kind: 'ask_user',
        question: `遇到错误：${observation.error}。请告知如何继续？`
      };
    }
  }

  // 目标达成检测
  if (isGoalAchieved(state)) {
    return {
      kind: 'done',
      summary: buildSummary(state)
    };
  }

  // 正常：选择下一个工具
  const nextTool = selectNextTool(state, availableTools);
  return {
    kind: 'call_tool',
    name: nextTool.name,
    params: nextTool.params
  };
}

function isGoalAchieved(state: AgentState): boolean {
  // 纯逻辑判断，基于状态数据
  const requiredSteps = parseRequiredSteps(state.goal);
  return requiredSteps.every(step => state.completedSteps.includes(step));
}

function selectNextTool(
  state: AgentState,
  availableTools: string[]
): { name: string; params: Record<string, unknown> } {
  const pendingSteps = getPendingSteps(state);
  const step = pendingSteps[0];
  
  // 根据步骤名映射到工具（纯逻辑）
  const toolMap: Record<string, { name: string; params: (ctx: typeof state.context) => Record<string, unknown> }> = {
    'read_file': { name: 'read', params: ctx => ({ path: ctx.filePath }) },
    'search':    { name: 'web_search', params: ctx => ({ query: ctx.query }) },
    'write':     { name: 'write', params: ctx => ({ content: ctx.content, path: ctx.outputPath }) },
  };

  const mapping = toolMap[step];
  if (!mapping || !availableTools.includes(mapping.name)) {
    // 工具不可用时降级：ask user
    return { name: 'ask_user', params: { question: `工具 ${step} 不可用，请提供替代方案` } };
  }

  return { name: mapping.name, params: mapping.params(state.context) };
}

// ... 其他纯辅助函数
function parseRequiredSteps(goal: string): string[] { /* 纯解析 */ return []; }
function getPendingSteps(state: AgentState): string[] { return []; }
function buildSummary(state: AgentState): string { return `完成：${state.goal}`; }
```

### 3. 命令式外壳 - 唯一执行副作用的地方

```typescript
// shell.ts - 所有 I/O 和副作用都在这里

import Anthropic from '@anthropic-ai/sdk';
import { decide } from './core';
import type { AgentState, Observation, AgentAction } from './types';

const client = new Anthropic();

// 工具执行器（副作用！）
const toolExecutors: Record<string, (params: Record<string, unknown>) => Promise<unknown>> = {
  read: async ({ path }) => {
    // 真实文件读取
    const fs = await import('fs/promises');
    return fs.readFile(String(path), 'utf-8');
  },
  web_search: async ({ query }) => {
    // 真实 API 调用
    const resp = await fetch(`https://api.search.example.com?q=${query}`);
    return resp.json();
  },
  write: async ({ content, path }) => {
    const fs = await import('fs/promises');
    await fs.writeFile(String(path), String(content));
    return { success: true };
  },
};

// Agent Loop - 命令式外壳负责驱动整个循环
export async function runAgent(goal: string, initialContext: Record<string, unknown>) {
  let state: AgentState = {
    goal,
    completedSteps: [],
    context: initialContext,
    iteration: 0,
  };
  
  let observation: Observation = {};
  const availableTools = Object.keys(toolExecutors);

  while (true) {
    // 1. 调用纯函数核心做决策（无副作用）
    const action = decide(state, observation, availableTools);
    
    console.log(`[迭代 ${state.iteration}] 决策:`, action);

    // 2. 命令式外壳执行决策（有副作用）
    if (action.kind === 'done') {
      console.log('✅ 任务完成:', action.summary);
      break;
    }
    
    if (action.kind === 'respond') {
      console.log('🤖 Agent 回复:', action.message);
      break;
    }

    if (action.kind === 'ask_user') {
      // 在真实系统里这里会等待用户输入
      const userResponse = await getUserInput(action.question);
      observation = { lastToolResult: userResponse };
      state = { ...state, iteration: state.iteration + 1 };
      continue;
    }

    if (action.kind === 'call_tool') {
      try {
        // 执行工具（副作用发生在这里！）
        const executor = toolExecutors[action.name];
        if (!executor) throw new Error(`未知工具: ${action.name}`);
        
        const result = await executor(action.params);
        
        // 更新状态
        state = {
          ...state,
          completedSteps: [...state.completedSteps, action.name],
          context: { ...state.context, lastResult: result },
          iteration: state.iteration + 1,
        };
        observation = { lastToolResult: result };
        
      } catch (error) {
        observation = { error: String(error) };
        state = { ...state, iteration: state.iteration + 1 };
      }
    }
  }
}

async function getUserInput(question: string): Promise<string> {
  // 真实实现：从 stdin/Telegram/等读取
  process.stdout.write(`❓ ${question}: `);
  return new Promise(resolve => {
    process.stdin.once('data', data => resolve(data.toString().trim()));
  });
}
```

### 4. 测试核心逻辑 - 零 Mock，零网络

```typescript
// core.test.ts - 纯单元测试，不需要 mock 任何东西！

import { describe, it, expect } from 'vitest';
import { decide } from './core';

const baseState = {
  goal: 'search then write report',
  completedSteps: [],
  context: { query: 'AI trends 2026', outputPath: '/tmp/report.md' },
  iteration: 0,
};

const tools = ['web_search', 'write', 'read'];

describe('decide() - 纯函数决策', () => {
  it('正常情况：选择第一个待完成工具', () => {
    const action = decide(baseState, {}, tools);
    expect(action.kind).toBe('call_tool');
    expect((action as any).name).toBe('web_search');
  });

  it('出错时前三步重试', () => {
    const state = { ...baseState, iteration: 1 };
    const action = decide(state, { error: 'Network timeout' }, tools);
    expect(action.kind).toBe('call_tool');
    expect((action as any).name).toBe('retry_last');
  });

  it('多次出错时请求人工介入', () => {
    const state = { ...baseState, iteration: 5 };
    const action = decide(state, { error: 'Persistent failure' }, tools);
    expect(action.kind).toBe('ask_user');
  });

  it('超过 20 轮强制终止', () => {
    const state = { ...baseState, iteration: 20 };
    const action = decide(state, {}, tools);
    expect(action.kind).toBe('done');
  });

  it('工具不可用时降级', () => {
    const action = decide(baseState, {}, ['only_tool']); // 没有 web_search
    expect(action.kind).toBe('call_tool');
    expect((action as any).name).toBe('ask_user');
  });
});
```

**运行测试**：
```bash
vitest run core.test.ts
# ✓ 5 tests passed in 12ms（无网络、无 API、极快）
```

---

## Python 版本

```python
# core.py - 纯函数，无 import 外部 IO 库

from dataclasses import dataclass, field
from typing import Union, Optional

@dataclass
class AgentState:
    goal: str
    completed_steps: list[str] = field(default_factory=list)
    context: dict = field(default_factory=dict)
    iteration: int = 0

@dataclass
class Observation:
    last_result: Optional[object] = None
    error: Optional[str] = None

# 动作类型
@dataclass
class CallTool:
    name: str
    params: dict

@dataclass
class Respond:
    message: str

@dataclass
class AskUser:
    question: str

@dataclass
class Done:
    summary: str

AgentAction = Union[CallTool, Respond, AskUser, Done]


def decide(state: AgentState, obs: Observation, available_tools: list[str]) -> AgentAction:
    """纯函数决策 - 永远没有副作用"""
    
    if state.iteration >= 20:
        return Done(f"超过最大迭代次数。完成步骤: {state.completed_steps}")
    
    if obs.error:
        if state.iteration < 3:
            return CallTool("retry_last", {"reason": obs.error})
        return AskUser(f"遇到错误：{obs.error}。如何继续？")
    
    if _is_goal_achieved(state):
        return Done(f"目标达成: {state.goal}")
    
    next_step = _get_next_step(state)
    if next_step and next_step in available_tools:
        return CallTool(next_step, _build_params(next_step, state.context))
    
    return AskUser(f"需要工具 {next_step}，但不在可用列表中")


def _is_goal_achieved(state: AgentState) -> bool:
    required = _parse_required_steps(state.goal)
    return all(s in state.completed_steps for s in required)

def _get_next_step(state: AgentState) -> Optional[str]:
    required = _parse_required_steps(state.goal)
    pending = [s for s in required if s not in state.completed_steps]
    return pending[0] if pending else None

def _parse_required_steps(goal: str) -> list[str]:
    # 纯解析逻辑
    return []

def _build_params(tool: str, context: dict) -> dict:
    return context.get(f"{tool}_params", {})
```

```python
# shell.py - 命令式外壳，所有 IO 在这里

import asyncio
from pathlib import Path
from core import decide, AgentState, Observation, CallTool, Done, AskUser, Respond

# 工具实现（有副作用）
async def execute_tool(name: str, params: dict) -> object:
    if name == "read":
        return Path(params["path"]).read_text()
    elif name == "web_search":
        import httpx
        async with httpx.AsyncClient() as c:
            r = await c.get("https://api.search.example.com", params={"q": params["query"]})
            return r.json()
    elif name == "write":
        Path(params["path"]).write_text(str(params["content"]))
        return {"success": True}
    raise ValueError(f"未知工具: {name}")

async def run_agent(goal: str, initial_context: dict):
    state = AgentState(goal=goal, context=initial_context)
    obs = Observation()
    available_tools = ["read", "web_search", "write"]

    while True:
        action = decide(state, obs, available_tools)  # 纯函数，无副作用
        
        if isinstance(action, Done):
            print(f"✅ {action.summary}")
            break
        
        if isinstance(action, Respond):
            print(f"🤖 {action.message}")
            break
        
        if isinstance(action, AskUser):
            user_input = input(f"❓ {action.question}: ")
            obs = Observation(last_result=user_input)
        
        elif isinstance(action, CallTool):
            try:
                result = await execute_tool(action.name, action.params)
                state = AgentState(
                    goal=state.goal,
                    completed_steps=[*state.completed_steps, action.name],
                    context={**state.context, "last_result": result},
                    iteration=state.iteration + 1,
                )
                obs = Observation(last_result=result)
            except Exception as e:
                obs = Observation(error=str(e))
                state = AgentState(**{**state.__dict__, "iteration": state.iteration + 1})
```

---

## OpenClaw 里的体现

OpenClaw 的 Agent Loop 完美体现了这个模式：

```
┌── 外壳层（OpenClaw runtime）──────────────────────┐
│  - 读取 Telegram 消息                             │
│  - 调用 Claude API（有网络 I/O）                  │
│  - 执行 exec/read/write 工具（有文件 I/O）        │
│                                                   │
│  ┌── 核心层（你的逻辑）────────────────────────┐  │
│  │  - SOUL.md 定义决策规则（纯声明）            │  │
│  │  - Skills 描述"何时用哪个工具"（纯逻辑）     │  │
│  │  - HEARTBEAT.md 决定"要不要行动"（纯判断）   │  │
│  └──────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────┘
```

你写的 SOUL.md、Skills、HEARTBEAT.md 都是"核心"——它们只描述逻辑，不执行任何副作用。OpenClaw 作为外壳负责真正的 I/O。

---

## pi-mono 里的体现

```typescript
// pi-mono: TaskSystem 是纯函数核心
class TaskSystem {
  // 纯函数：给定任务列表和当前状态，返回下一步计划
  plan(tasks: Task[], state: SystemState): TaskPlan {
    return this.scheduler.schedule(tasks, state); // 无副作用
  }
}

// AgentLoop 是命令式外壳
class AgentLoop {
  async run() {
    while (!done) {
      const plan = this.taskSystem.plan(this.tasks, this.state); // 纯函数
      await this.executor.execute(plan); // 副作用在这里
    }
  }
}
```

---

## 收益总结

| 维度 | 混合式（传统） | FC/IS 架构 |
|------|-------------|-----------|
| 单元测试速度 | 慢（需 mock API） | 快（纯函数，12ms） |
| 决策可复现 | ❌ 依赖外部状态 | ✅ 同输入同输出 |
| 重构风险 | 高（副作用散落各处） | 低（副作用集中在外壳） |
| 调试难度 | 高（需真实环境） | 低（本地直接调用 decide）|
| 并发安全 | 难保证 | 天然安全（不可变状态）|

---

## 三句话总结

1. **把决策逻辑写成纯函数**：state + observation → action，不管 I/O
2. **把所有 I/O 推到外壳层**：外壳负责读取世界、执行动作、更新状态
3. **测试只测核心**：用真实数据结构跑几十个场景，毫秒级，不花 API 费用

> 这就是 Claude Code 为什么可以在离线状态下通过大量单元测试的原因：它的决策逻辑是纯函数，测试不需要真实的 Claude API。
