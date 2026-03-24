# 133 - Agent 工具调用预算控制（Tool Call Budget Control）

## 为什么需要工具调用预算？

Agent 在自主运行时，有一个常见的"失控"场景：

- 任务看起来简单，但 Agent 反复调用工具，迟迟无法收敛
- 每次工具失败都重试，无限消耗 API 调用次数
- 爬取数据时递归展开，调用链越来越深

这与 Token Budget（控制思考链长度）不同：
- **Token Budget** → 控制 LLM 内部思考消耗
- **Tool Call Budget** → 控制外部工具调用的次数、成本、时长

一个健壮的 Agent 需要**主动预算控制**：在运行过程中追踪资源消耗，接近上限时自动降级，而不是等崩溃后再处理。

---

## 三个维度的预算

```
┌──────────────────────────────────────────┐
│           Tool Call Budget               │
│                                          │
│  1. 调用次数  (count)    ≤ 50 次         │
│  2. 累计成本  (cost)     ≤ $0.10         │
│  3. 总耗时    (time)     ≤ 30s           │
│                                          │
│  任意一个触及阈值 → 触发降级策略          │
└──────────────────────────────────────────┘
```

**降级策略三档：**
1. **警告（Warning）**：75% → 提示 LLM "资源不多了，尽快收敛"
2. **限速（Throttle）**：90% → 只允许关键工具调用
3. **强制结束（Hard Stop）**：100% → 不再调用工具，返回当前最优结果

---

## 核心实现

### 1. BudgetTracker：预算追踪器

```typescript
// pi-mono 风格：不可变状态 + 纯函数更新
interface Budget {
  maxCalls: number;
  maxCostUsd: number;
  maxTimeMs: number;
}

interface BudgetState {
  callCount: number;
  totalCostUsd: number;
  startTimeMs: number;
}

type BudgetStatus = 'ok' | 'warning' | 'throttle' | 'exceeded';

class BudgetTracker {
  private state: BudgetState;

  constructor(private budget: Budget) {
    this.state = {
      callCount: 0,
      totalCostUsd: 0,
      startTimeMs: Date.now(),
    };
  }

  // 记录一次工具调用
  record(costUsd: number = 0): void {
    this.state.callCount++;
    this.state.totalCostUsd += costUsd;
  }

  // 获取当前状态（0~1 的使用率，取三个维度的最大值）
  getUsageRatio(): number {
    const elapsed = Date.now() - this.state.startTimeMs;
    return Math.max(
      this.state.callCount / this.budget.maxCalls,
      this.state.totalCostUsd / this.budget.maxCostUsd,
      elapsed / this.budget.maxTimeMs,
    );
  }

  getStatus(): BudgetStatus {
    const ratio = this.getUsageRatio();
    if (ratio >= 1.0) return 'exceeded';
    if (ratio >= 0.9) return 'throttle';
    if (ratio >= 0.75) return 'warning';
    return 'ok';
  }

  getSummary(): string {
    const elapsed = ((Date.now() - this.state.startTimeMs) / 1000).toFixed(1);
    return (
      `calls=${this.state.callCount}/${this.budget.maxCalls} ` +
      `cost=$${this.state.totalCostUsd.toFixed(4)}/$${this.budget.maxCostUsd} ` +
      `time=${elapsed}s/${this.budget.maxTimeMs / 1000}s`
    );
  }
}
```

---

### 2. 工具调用中间件：拦截每次调用

```typescript
// 包装 tool dispatcher，注入预算控制
function withBudgetControl(
  dispatch: ToolDispatcher,
  tracker: BudgetTracker,
  // 按工具名定义成本（$）
  toolCostMap: Record<string, number> = {},
): ToolDispatcher {
  return async (toolName: string, params: unknown) => {
    const status = tracker.getStatus();

    // 超出预算：拒绝调用
    if (status === 'exceeded') {
      return {
        error: 'BUDGET_EXCEEDED',
        message: `工具调用预算已耗尽。${tracker.getSummary()}`,
        suggestion: '请基于已有信息给出最终答案。',
      };
    }

    // 限速阶段：只允许白名单工具
    const CRITICAL_TOOLS = ['final_answer', 'report_error'];
    if (status === 'throttle' && !CRITICAL_TOOLS.includes(toolName)) {
      return {
        error: 'BUDGET_THROTTLED',
        message: `预算使用已达 90%，当前只允许调用关键工具。${tracker.getSummary()}`,
        allowedTools: CRITICAL_TOOLS,
      };
    }

    // 记录调用
    const cost = toolCostMap[toolName] ?? 0;
    tracker.record(cost);

    // 实际调用
    const result = await dispatch(toolName, params);

    // 警告阶段：在结果中附加提示
    if (status === 'warning') {
      return {
        ...result,
        _budgetWarning: `⚠️ 预算使用已达 75%，请尽快收敛。${tracker.getSummary()}`,
      };
    }

    return result;
  };
}
```

---

### 3. 在 Agent Loop 中注入预算提示

```typescript
// 根据预算状态动态修改系统提示
function buildSystemPromptWithBudget(
  basePrompt: string,
  tracker: BudgetTracker,
): string {
  const status = tracker.getStatus();

  const budgetHints: Record<string, string> = {
    ok: '',
    warning: '\n\n⚠️ 【预算提醒】工具调用资源已使用 75%，请减少不必要的工具调用，尽快完成任务。',
    throttle:
      '\n\n🚨 【预算紧张】工具调用资源已使用 90%！只能调用 final_answer 或 report_error。请立即总结当前已知信息作为最终答案。',
    exceeded:
      '\n\n❌ 【预算耗尽】禁止继续调用工具。请基于已掌握的信息直接给出最终答案，不能再调用任何工具。',
  };

  return basePrompt + budgetHints[status];
}
```

---

### 4. OpenClaw 实战：Skills 层的预算控制

OpenClaw 的 Skills 是在 Agent Loop 外层执行的，可以在 Skill 入口处初始化预算：

```typescript
// skills/my-research-skill/index.ts
import { SkillContext } from '@openclaw/sdk';

export async function run(ctx: SkillContext) {
  // 初始化预算：最多 20 次工具调用，$0.05，60 秒内完成
  const tracker = new BudgetTracker({
    maxCalls: 20,
    maxCostUsd: 0.05,
    maxTimeMs: 60_000,
  });

  // 按工具类型定义成本
  const toolCostMap = {
    web_search: 0.001,
    web_fetch: 0.002,
    exec: 0.0,
    // LLM 内部调用成本由 Token 计费，这里只计外部工具
  };

  // 注入预算中间件
  const budgetedDispatch = withBudgetControl(
    ctx.dispatch,
    tracker,
    toolCostMap,
  );

  // 运行 Agent Loop，用 budgetedDispatch 替换原始 dispatch
  await runAgentLoop({
    ...ctx,
    dispatch: budgetedDispatch,
    systemPrompt: (prev) => buildSystemPromptWithBudget(prev, tracker),
  });

  // 任务结束后记录预算使用情况
  ctx.log.info('Budget summary', { summary: tracker.getSummary() });
}
```

---

### 5. learn-claude-code 的做法：循环次数硬限制

learn-claude-code（Python 实现）用最简单的方式控制预算：

```python
# learn-claude-code/agent.py（示意）
class Agent:
    MAX_TURNS = 20  # 最多 20 轮工具调用

    async def run(self, task: str) -> str:
        messages = [{"role": "user", "content": task}]
        turn = 0

        while turn < self.MAX_TURNS:
            response = await self.llm.call(messages)

            if response.stop_reason == "end_turn":
                return response.text  # 正常结束

            # 执行工具调用
            tool_results = await self.execute_tools(response.tool_calls)
            messages.extend([
                {"role": "assistant", "content": response.content},
                {"role": "user", "content": tool_results},
            ])
            turn += 1

        # 超出轮次：注入强制结束提示
        messages.append({
            "role": "user",
            "content": (
                f"[系统] 已达到最大轮次限制（{self.MAX_TURNS}）。"
                "请立即基于现有信息给出最终答案，不要再调用工具。"
            ),
        })
        final = await self.llm.call(messages, tools=[])  # 禁用工具
        return final.text
```

关键点：**超出限制后，禁用工具（`tools=[]`）强制 LLM 给出最终答案**，而不是直接报错。

---

## 进阶：自适应预算调整

静态预算有时太死板。可以根据任务复杂度动态调整：

```typescript
// 任务开始时，让 LLM 估算所需工具调用次数
async function estimateBudget(task: string, llm: LLM): Promise<Budget> {
  const estimation = await llm.call([
    {
      role: 'user',
      content: `
分析以下任务，估算完成所需的工具调用次数和时间：
任务：${task}

请以 JSON 格式回答：
{"estimatedCalls": <number>, "estimatedTimeSeconds": <number>, "complexity": "low|medium|high"}
      `,
    },
  ]);

  const { estimatedCalls, estimatedTimeSeconds, complexity } =
    JSON.parse(estimation.text);

  // 给估算值加 50% 缓冲
  const buffer = 1.5;
  return {
    maxCalls: Math.ceil(estimatedCalls * buffer),
    maxCostUsd: complexity === 'high' ? 0.2 : complexity === 'medium' ? 0.1 : 0.05,
    maxTimeMs: Math.ceil(estimatedTimeSeconds * buffer * 1000),
  };
}
```

---

## 与其他模式的关系

| 模式 | 控制对象 | 时机 |
|------|---------|------|
| Token Budget（128课） | LLM 思考 Token | 调用 LLM 时 |
| Rate Limiting（15课） | API 请求速率 | 发出 HTTP 请求时 |
| Load Shedding（114课） | 系统级负载 | 请求进入系统时 |
| **Tool Call Budget（本课）** | **单次任务的工具调用** | **Agent Loop 运行中** |

四者互补，各管一层。Tool Call Budget 最靠近业务逻辑，是 Agent 自我约束的最后一道防线。

---

## 小结

| 要点 | 说明 |
|------|------|
| 三维预算 | 次数 + 成本 + 时间，取最严格的维度 |
| 四级状态 | ok → warning → throttle → exceeded |
| 降级而非崩溃 | 超限时提示 LLM 收敛，而不是抛异常 |
| 工具中间件 | 预算控制与业务逻辑解耦，透明注入 |
| 禁用工具强制答案 | `tools=[]` 是最优雅的强制收敛方式 |

> **核心思想**：给 Agent 设预算，不是为了限制它，而是为了让它在资源耗尽前做出最好的决策——就像一个好的工程师，在 deadline 前优先交付能跑的版本。
