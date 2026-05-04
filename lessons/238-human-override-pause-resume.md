# 238. Agent 人工接管与暂停恢复（Human Override & Pause/Resume Control）

前几课我们把 Agent 的入口和出口都做成了可靠系统：Inbox 负责“听进来”，Outbox 负责“说出去”。

今天讲生产里非常关键的一层：**人工接管控制面**。一个 Always-on Agent 不能只会自己跑，它还要能被人类随时：

- 暂停某个任务；
- 接管某个会话；
- 修改下一步指令；
- 恢复执行；
- 紧急停止外部副作用。

这不是“打断处理”的同义词。打断更多是对话层面的 pivot；人工接管是系统层面的 **control plane**：即使 Agent 正在跑 cron、sub-agent、部署、发消息，也必须能被高权限操作覆盖。

---

## 1. 问题：没有接管能力的 Agent 不适合生产

坏味道通常长这样：

```python
async def run_task(task):
    plan = await llm.plan(task)
    for step in plan.steps:
        result = await tools.call(step.tool, step.args)
        await memory.append(result)
    return 'done'
```

这段代码看起来很干净，但有几个生产风险：

- 用户说“停”，循环没有检查，工具还会继续执行；
- 人类已经手动修好了问题，Agent 还按旧计划继续改；
- 审批被撤销后，后台任务仍然持有旧权限；
- 多个 worker 同时跑，没人知道哪个任务被谁接管；
- 恢复时只能从头来，容易重复外部副作用。

生产 Agent 的原则是：**每个长任务都必须周期性读取 Control State，而不是只相信启动时的指令。**

---

## 2. 核心模型：TaskRun + ControlState

最小状态可以这样设计：

```ts
type TaskRun = {
  id: string
  goal: string
  status: 'running' | 'paused' | 'operator_controlled' | 'cancelled' | 'done' | 'failed'
  ownerAgent: string
  checkpoint?: unknown
  updatedAt: string
}

type ControlState = {
  taskId: string
  mode: 'auto' | 'paused' | 'manual' | 'cancelled'
  reason?: string
  operatorId?: string
  revision: number
  updatedAt: string
  resumeInstruction?: string
}
```

关键字段是 `revision`：

```text
revision = 12
```

Agent 每次执行外部副作用前都检查 revision。如果自己持有的是旧 revision，说明控制状态已经被人改过，必须停下来重新读取。

---

## 3. learn-claude-code：教学版暂停检查

教学版不用复杂框架，一个 JSON 控制文件就够了：

```python
# learn-claude-code: control_plane.py
import json
from pathlib import Path

CONTROL_DIR = Path('.agent-control')
CONTROL_DIR.mkdir(exist_ok=True)


def control_path(task_id: str) -> Path:
    return CONTROL_DIR / f'{task_id}.json'


def read_control(task_id: str) -> dict:
    path = control_path(task_id)
    if not path.exists():
        return {'mode': 'auto', 'revision': 0}
    return json.loads(path.read_text())


def set_control(task_id: str, *, mode: str, operator_id: str, reason: str = '', resume_instruction: str = ''):
    current = read_control(task_id)
    next_state = {
        'mode': mode,
        'operatorId': operator_id,
        'reason': reason,
        'resumeInstruction': resume_instruction,
        'revision': current.get('revision', 0) + 1,
    }
    control_path(task_id).write_text(json.dumps(next_state, ensure_ascii=False, indent=2))
    return next_state
```

在 Agent Loop 里加 checkpoint：

```python
class HumanOverride(Exception):
    pass


def check_control(task_id: str, seen_revision: int):
    state = read_control(task_id)

    if state['revision'] != seen_revision:
        raise HumanOverride(f"control changed: {state}")

    if state['mode'] in ('paused', 'manual', 'cancelled'):
        raise HumanOverride(f"task is {state['mode']}: {state.get('reason', '')}")

    return state


async def run_task(task_id: str, steps: list[dict]):
    control = read_control(task_id)
    seen_revision = control['revision']

    for index, step in enumerate(steps):
        # 每一步前检查
        check_control(task_id, seen_revision)

        # 外部副作用前再检查一次
        if step.get('sideEffect'):
            check_control(task_id, seen_revision)

        result = await call_tool(step['tool'], step['args'])
        save_checkpoint(task_id, {'nextStep': index + 1, 'lastResult': result})
```

这里的重点不是代码多高级，而是机制：**长任务不能一路蒙头跑到结束。**

---

## 4. pi-mono：生产版 Control Middleware

生产版建议把人工接管做成中间件，而不是散落在业务代码里。

```ts
interface ControlStore {
  get(taskId: string): Promise<ControlState>
  set(taskId: string, patch: Partial<ControlState>, actor: Actor): Promise<ControlState>
}

class ControlMiddleware {
  constructor(private store: ControlStore) {}

  async beforeStep(ctx: AgentContext, step: PlannedStep) {
    const state = await this.store.get(ctx.taskId)

    if (state.mode === 'cancelled') {
      throw new TaskCancelled(state.reason ?? 'cancelled by operator')
    }

    if (state.mode === 'paused') {
      await ctx.checkpoint.save({ pausedBefore: step.id, controlRevision: state.revision })
      throw new TaskPaused(state.reason ?? 'paused by operator')
    }

    if (state.mode === 'manual') {
      await ctx.checkpoint.save({ handoffAt: step.id, operatorId: state.operatorId })
      throw new HumanHandoffRequired(state.operatorId)
    }
  }

  async beforeToolCall(ctx: AgentContext, call: ToolCall) {
    if (!call.metadata.sideEffect) return

    // 写操作前强制二次检查，避免“刚点暂停，Agent 已经发出删除请求”
    await this.beforeStep(ctx, { id: `tool:${call.name}` } as PlannedStep)
  }
}
```

工具分发器里统一调用：

```ts
async function dispatchTool(ctx: AgentContext, call: ToolCall) {
  await controlMiddleware.beforeToolCall(ctx, call)
  await riskMiddleware.preflight(ctx, call)
  return toolRegistry.call(call.name, call.args)
}
```

这样所有工具天然支持暂停/取消，不需要每个业务 handler 重复写判断。

---

## 5. 恢复执行：不要从头跑

暂停只是第一步，恢复才是难点。

恢复时应该读取 checkpoint：

```ts
async function resumeTask(taskId: string) {
  const task = await taskStore.get(taskId)
  const control = await controlStore.get(taskId)

  if (control.mode !== 'auto') {
    throw new Error(`cannot resume while mode=${control.mode}`)
  }

  const checkpoint = await checkpointStore.load(taskId)

  return agent.run({
    goal: task.goal,
    resumeFrom: checkpoint?.nextStep ?? 0,
    previousEvidence: checkpoint?.evidence ?? [],
    operatorInstruction: control.resumeInstruction,
  })
}
```

注意：`resumeInstruction` 不是直接替换系统提示词，而是作为高优先级任务上下文进入本次 run。

比如：

```text
operatorInstruction:
  用户已确认不要删除数据库，只检查只读指标并生成报告。
```

Agent 恢复后应该基于旧证据继续，但不能复用旧计划里的危险步骤。

---

## 6. OpenClaw 实战：心跳、cron、sub-agent 都要可控

OpenClaw 这种 always-on 环境里，人工接管尤其重要：

- 用户随时可能说“停”；
- cron 可能每 3 小时自动触发；
- sub-agent 可能在后台继续跑；
- message/git/deploy 都是外部副作用。

推荐实践：

```text
memory/task-control/<task-id>.json
memory/task-checkpoints/<task-id>.json
memory/task-audit/<task-id>.jsonl
```

一次课程 cron 可以记录：

```json
{
  "taskId": "agent-course-2026-05-04T21:35",
  "mode": "auto",
  "revision": 0,
  "quietHoursPolicy": "allow_course_group",
  "lastSafePoint": "lesson_written"
}
```

执行流程：

```text
1. 收到 cron tick
2. 写 Inbox / TaskRun
3. 生成课程草稿
4. check_control
5. 写 lesson 文件和 README
6. check_control
7. 发 Telegram 前二次检查
8. git commit/push 前二次检查
9. 写 audit + memory
```

如果老板中途说“先别发群”，control state 变成：

```json
{
  "mode": "paused",
  "operatorId": "boss",
  "reason": "先审核课程内容",
  "revision": 1
}
```

Agent 下一次检查到 revision 变化，就应该停在安全点，而不是继续发送。

---

## 7. 权限：不是所有人都能接管

人工接管必须结合权限模型：

```ts
function canOverride(actor: Actor, task: TaskRun, action: ControlAction) {
  if (actor.role === 'owner') return true
  if (actor.role === 'operator' && action !== 'cancel_external_effect') return true
  if (actor.id === task.createdBy && action === 'pause') return true
  return false
}
```

群聊里尤其要小心：

- 普通成员可以请求暂停自己的任务；
- owner 可以暂停任何任务；
- 外部用户不能接管系统级 cron；
- prompt 注入内容不能写 control state。

**控制面是权限系统，不是普通聊天内容。**

---

## 8. 常见坑

1. **只在任务开始时检查一次**  
   长任务运行 10 分钟，中途用户取消也没用。

2. **只在 LLM 前检查，不在工具前检查**  
   最危险的是外部副作用前的一瞬间，必须二次检查。

3. **暂停后没有 checkpoint**  
   恢复只能重跑，容易重复发消息、重复下单、重复部署。

4. **人工恢复直接复用旧计划**  
   旧计划可能已经过期。恢复时要重新验证关键观察和风险。

5. **没有 audit log**  
   谁暂停、谁恢复、为什么接管，都应该留下证据。

---

## 9. 小结

今天的核心：

- Always-on Agent 必须有人工接管控制面；
- `ControlState` 至少包含 mode / revision / operator / reason；
- Agent Loop 和工具分发前都要检查控制状态；
- 暂停必须配 checkpoint，恢复不能从头盲跑；
- 控制面必须受权限保护，不能被普通 prompt 改写。

一句话：**成熟 Agent 的自主性，不是“谁也拦不住”，而是“该自己跑时能跑，该人类接管时立刻让位”。**
