# 218. Agent 中断处理与任务改道（Interrupt Handling & Task Pivoting）

> 关键词：Interrupt Handling、Cooperative Cancellation、Task Pivot、Checkpoint、用户抢占

今天讲一个生产 Agent 很常见、但很多教学 Demo 没覆盖的问题：**用户在 Agent 跑到一半时突然说“停一下”“改成另一个方案”“先做更急的”怎么办？**

如果 Agent 只会从 prompt 开始一路跑到结束，它就像一台没有刹车的自动驾驶车：平时很顺，遇到路况变化就危险。

成熟 Agent 需要支持三类中断：

1. **Cancel**：用户说“停”“别发了”“取消任务”
2. **Pause**：用户说“先等等，我确认一下”
3. **Pivot**：用户说“方向改了，别查 A 了，查 B”

核心原则：**Agent 不是不能被打断，而是必须能安全地被打断。**

---

## 1. 反模式：长任务一跑到底

很多简化版 Agent Loop 是这样的：

```python
async def run_agent(task: str):
    plan = await llm_plan(task)

    for step in plan.steps:
        result = await call_tool(step.tool, step.args)
        await append_context(result)

    return await llm_final_answer()
```

问题在于：

- 用户说“停”时，当前循环完全不知道
- 慢工具执行时无法取消
- 已经产生的副作用不知道是否需要补偿
- 用户改目标后，旧上下文继续污染新任务
- Cron / Sub-agent 长任务里尤其容易“跑偏还停不下来”

正确做法不是让 LLM 自己“更听话”，而是在 Agent Runtime 里加入**中断信号通道**。

---

## 2. 中断模型：每个任务都要有 Interrupt State

可以给每个任务维护一个独立状态：

```ts
type InterruptKind = "cancel" | "pause" | "pivot";

type InterruptSignal = {
  kind: InterruptKind;
  reason: string;
  newGoal?: string;
  requestedBy: string;
  createdAt: number;
};

type TaskRun = {
  id: string;
  goal: string;
  status: "running" | "paused" | "cancelled" | "completed";
  checkpointId?: string;
  interrupt?: InterruptSignal;
};
```

关键点：**中断不是一条普通聊天消息，而是对正在运行任务的控制信号。**

用户发来的消息需要先经过一个轻量分类器：

```ts
function classifyUserMessage(text: string): InterruptSignal | null {
  const t = text.toLowerCase();

  if (["停", "取消", "别做了", "stop", "cancel"].some(x => t.includes(x))) {
    return { kind: "cancel", reason: text, requestedBy: "user", createdAt: Date.now() };
  }

  if (["等等", "暂停", "先别", "pause"].some(x => t.includes(x))) {
    return { kind: "pause", reason: text, requestedBy: "user", createdAt: Date.now() };
  }

  if (["改成", "换成", "先做", "不用", "instead"].some(x => t.includes(x))) {
    return {
      kind: "pivot",
      reason: text,
      newGoal: text,
      requestedBy: "user",
      createdAt: Date.now(),
    };
  }

  return null;
}
```

生产版可以用小模型分类，但规则层必须先挡住明确的“停”。

---

## 3. learn-claude-code 教学版：协作式取消

Python 教学版可以用 `asyncio.Event` 做最小实现。

```python
import asyncio
from dataclasses import dataclass

@dataclass
class Interrupt:
    kind: str
    reason: str
    new_goal: str | None = None

class RunController:
    def __init__(self):
        self._interrupt: Interrupt | None = None
        self._event = asyncio.Event()

    def request_interrupt(self, interrupt: Interrupt):
        self._interrupt = interrupt
        self._event.set()

    def check(self):
        if self._interrupt:
            raise InterruptedError(self._interrupt)

    async def wait_or_timeout(self, seconds: float):
        try:
            await asyncio.wait_for(self._event.wait(), timeout=seconds)
        except asyncio.TimeoutError:
            return None
        return self._interrupt

async def agent_loop(goal: str, controller: RunController):
    steps = await plan(goal)

    for step in steps:
        controller.check()  # 每一步开始前检查

        # 慢工具不要一次 await 到死，拆成可观察轮询
        job = await start_tool_job(step)
        while not await job.done():
            if interrupt := await controller.wait_or_timeout(0.5):
                await job.cancel()
                raise InterruptedError(interrupt)

        result = await job.result()
        await save_checkpoint(goal, step, result)

    return await final_answer()
```

这个版本的重点不是代码复杂，而是两个习惯：

- 每个 step 前检查中断
- 慢工具要支持轮询/取消，不要黑盒阻塞

---

## 4. pi-mono 生产版：TaskRun + AbortController

TypeScript 生产系统更适合用 `AbortController` 贯穿 LLM、工具、Sub-agent。

```ts
export class TaskRunController {
  private abortController = new AbortController();
  private interrupt?: InterruptSignal;

  get signal() {
    return this.abortController.signal;
  }

  request(signal: InterruptSignal) {
    this.interrupt = signal;

    if (signal.kind === "cancel" || signal.kind === "pivot") {
      this.abortController.abort(signal.reason);
    }
  }

  throwIfInterrupted() {
    if (!this.interrupt) return;

    throw new AgentInterruptedError({
      kind: this.interrupt.kind,
      reason: this.interrupt.reason,
      newGoal: this.interrupt.newGoal,
    });
  }
}
```

工具执行器必须接收 `AbortSignal`：

```ts
export type ToolCallContext = {
  runId: string;
  userId: string;
  signal: AbortSignal;
};

export async function dispatchTool(
  tool: ToolDefinition,
  args: unknown,
  ctx: ToolCallContext,
) {
  ctx.signal.throwIfAborted();

  return tool.execute(args, {
    signal: ctx.signal,
    onProgress: async progress => {
      ctx.signal.throwIfAborted();
      await publishProgress(ctx.runId, progress);
    },
  });
}
```

HTTP 工具也要把 signal 传下去：

```ts
async function fetchJson(url: string, ctx: ToolCallContext) {
  const res = await fetch(url, { signal: ctx.signal });
  return res.json();
}
```

一句话：**AbortSignal 要像 traceId 一样贯穿整条调用链。**

---

## 5. Pivot 不是 Cancel 后重开那么简单

用户说“改成查昨天的数据”时，不能粗暴保留旧上下文，也不能完全丢掉所有进度。

正确做法是三步：

1. 保存当前 checkpoint
2. 判断哪些中间结果还能复用
3. 用新目标 rebase 计划

```ts
type Checkpoint = {
  runId: string;
  goal: string;
  completedSteps: Array<{
    stepId: string;
    tool: string;
    args: unknown;
    resultRef: string;
    reusable: boolean;
  }>;
};

async function pivotRun(
  oldRun: TaskRun,
  checkpoint: Checkpoint,
  newGoal: string,
) {
  const reusable = checkpoint.completedSteps.filter(s => s.reusable);

  const rebasePrompt = {
    oldGoal: oldRun.goal,
    newGoal,
    reusableFacts: reusable.map(s => s.resultRef),
    instruction: "基于新目标重新规划。只复用仍然相关的事实，不要沿用旧目标的结论。",
  };

  return await planner.replan(rebasePrompt);
}
```

例子：

- 旧目标：查最近 7 天订单异常
- 新目标：只查昨天订单异常
- 可复用：数据库连接、字段说明、异常分类规则
- 不可复用：7 天聚合结果、旧结论、旧告警文案

Pivot 的本质是：**保留事实，丢弃结论。**

---

## 6. OpenClaw 实战：长任务要可打断、可接管

OpenClaw 里常见长任务包括：

- `exec` 后台命令
- `sessions_spawn` 子 Agent
- browser 多步骤自动化
- cron 驱动的课程、巡检、报表

设计上要遵守三个实践：

### 实践一：后台任务必须有句柄

启动长任务时保存 `sessionId` / `runId` / `processId`：

```ts
const proc = await tools.exec({
  command: "npm test",
  yieldMs: 1000,
  background: true,
});

await taskStore.save({
  runId,
  kind: "exec",
  handle: proc.sessionId,
  status: "running",
});
```

没有句柄，就没法取消、轮询、恢复。

### 实践二：每次外部副作用前检查用户是否打断

```ts
await controller.throwIfInterrupted();

// 发送消息、提交 PR、部署、删除文件前都要检查
await message.send({ target: groupId, message: content });
```

尤其是“发送到群里”“push 代码”“部署生产”这种副作用，必须在执行前最后检查一次。

### 实践三：打断后要给用户一个清楚的收尾

不要只说“失败了”。应该说：

```text
已停止。完成到第 3/7 步：数据已拉取、报告未发送、没有产生外部副作用。
如果要继续，可以说“从第 3 步继续”或“按新目标重做”。
```

中断处理的用户体验，核心是让人知道：**停在哪里、影响了什么、下一步能怎么走。**

---

## 7. 安全边界：不是所有操作都能取消

有些操作已经发出去了，就不能真正撤销：

- 邮件已发送
- PR 已创建
- 付款已提交
- 文件已删除
- 部署已开始

所以工具元数据需要声明可取消性：

```ts
type ToolSideEffect = "none" | "local" | "external" | "irreversible";

type ToolDefinition = {
  name: string;
  sideEffect: ToolSideEffect;
  cancellable: boolean;
  compensation?: string;
};
```

中断发生时按工具类型处理：

| 类型 | 处理方式 |
|---|---|
| none | 直接取消 |
| local | 尽量回滚或清理临时文件 |
| external | 停止后续动作，记录已发生副作用 |
| irreversible | 不承诺撤销，只能补偿或人工介入 |

这和 Saga Pattern 很像：取消不是魔法，取消是一套明确的补偿协议。

---

## 8. 常见坑

1. **只取消 LLM，不取消工具**：LLM 停了，shell/browser/API 还在跑。
2. **只取消父任务，不取消子 Agent**：父会话停了，sub-agent 继续发消息。
3. **Pivot 时复用旧结论**：新目标被旧答案污染。
4. **没有 checkpoint**：停了以后只能从头来。
5. **把“等等”当普通聊天**：用户以为停了，Agent 还在执行危险动作。
6. **取消后不汇报副作用**：用户不知道是否已经发送、提交、部署。

---

## 9. 生产 Checklist

- [ ] 每个长任务都有 `runId` 和可恢复 checkpoint
- [ ] 用户消息先经过 interrupt classifier
- [ ] 明确的“停/取消/别做了”优先级高于普通任务
- [ ] LLM、工具、Sub-agent 都接收同一个 cancellation signal
- [ ] 慢工具支持轮询、取消或超时
- [ ] 外部副作用前最后检查一次 interrupt state
- [ ] Pivot 时只复用事实，不复用旧结论
- [ ] 取消后汇报完成进度和已发生副作用
- [ ] 不可逆操作声明 `irreversible` 和补偿方式

---

## 10. 总结

Agent 的强大不在于“一口气做完所有事”，而在于能像靠谱员工一样：

- 老板说停，立刻停
- 老板说改，知道保留哪些进度、丢掉哪些结论
- 已经做过什么，交代清楚
- 不能撤销的事，不假装能撤销

一句话记住：

> **Interrupt Handling = 给 Agent 装刹车、离合和方向盘。**

没有中断处理的 Agent，只适合 Demo；有中断处理的 Agent，才适合生产。
