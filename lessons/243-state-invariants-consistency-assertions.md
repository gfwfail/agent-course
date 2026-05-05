# 第 243 课：Agent 状态不变量与一致性断言（State Invariants & Consistency Assertions）

> 核心思想：Agent 的状态会被 LLM、工具、子任务、重试、人工接管一起修改。不要只在“写代码时相信流程正确”，而要把系统必须永远成立的规则写成可执行的不变量（invariants），在关键节点自动断言。

---

## 1. 为什么 Agent 特别需要不变量？

普通后端服务通常是“请求进来 → 校验 → 写库 → 返回”。Agent 系统更复杂：

- LLM 可能改变计划；
- 工具调用可能部分成功；
- 子 Agent 可能并发写状态；
- Cron 任务可能被重复触发；
- 用户随时可能说“停、改目标、继续”。

这意味着状态很容易进入“看起来能跑，但已经不一致”的灰区。

典型事故：

- `task.status = done`，但 `requiredGates` 里还有测试没跑；
- `message.sent = true`，但没有 provider message id；
- PR 已 merge，却还往旧分支 push；
- 订单状态是 `paid`，但 payment receipt 缺失；
- 子 Agent 交接说完成，但 artifact 文件不存在。

不变量就是把这些“绝不能发生”的规则写死：一旦违反，立刻阻断、回滚、告警或进入人工处理。

---

## 2. 不变量 vs 普通校验

普通校验通常检查“这次输入对不对”。

不变量检查的是“系统现在整体状态是否仍然自洽”。

| 类型 | 关注点 | 例子 |
|---|---|---|
| 输入校验 | 单次请求参数 | `limit` 必须是 1-100 |
| 权限校验 | 谁能做什么 | 群聊用户不能读老板私有记忆 |
| 不变量断言 | 状态整体一致性 | `done` 任务必须有所有验收证据 |

Agent 工程里，这三层都要有；不变量是最后一道防线。

---

## 3. learn-claude-code：Python 教学版

先用最小实现表达核心模式：

```python
from dataclasses import dataclass, field
from typing import Callable, list

@dataclass
class TaskState:
    id: str
    status: str  # pending | running | done | failed
    required_gates: list[str] = field(default_factory=list)
    passed_gates: list[str] = field(default_factory=list)
    artifacts: list[str] = field(default_factory=list)
    sent_message_id: str | None = None

Invariant = Callable[[TaskState], None]

class InvariantViolation(Exception):
    pass

def done_requires_all_gates(state: TaskState):
    if state.status == "done":
        missing = set(state.required_gates) - set(state.passed_gates)
        if missing:
            raise InvariantViolation(
                f"task {state.id} cannot be done, missing gates: {sorted(missing)}"
            )

def sent_message_requires_receipt(state: TaskState):
    if "telegram_message_sent" in state.passed_gates and not state.sent_message_id:
        raise InvariantViolation(
            f"task {state.id} has send gate but no provider message id"
        )

class InvariantGuard:
    def __init__(self, invariants: list[Invariant]):
        self.invariants = invariants

    def assert_valid(self, state: TaskState, where: str):
        for invariant in self.invariants:
            try:
                invariant(state)
            except InvariantViolation as e:
                raise InvariantViolation(f"[{where}] {e}") from e

# Agent loop 关键节点调用
guard = InvariantGuard([
    done_requires_all_gates,
    sent_message_requires_receipt,
])

state = TaskState(
    id="lesson-243",
    status="done",
    required_gates=["lesson_file", "readme", "telegram_message", "git_push"],
    passed_gates=["lesson_file", "readme"],
)

guard.assert_valid(state, where="before_final_reply")
```

重点不是这段代码多复杂，而是：**状态写入后、外部副作用前、最终回复前，都要跑一次 invariant guard。**

---

## 4. pi-mono：TypeScript 生产版中间件

生产实现建议把不变量做成中间件，挂到状态存储和工具分发层。

```ts
type TaskStatus = 'pending' | 'running' | 'done' | 'failed';

type TaskState = {
  id: string;
  status: TaskStatus;
  requiredGates: string[];
  passedGates: string[];
  effects: Array<{
    kind: 'telegram' | 'github_pr' | 'deploy';
    status: 'planned' | 'committed' | 'verified';
    externalId?: string;
  }>;
};

type Invariant = {
  name: string;
  severity: 'block' | 'quarantine' | 'warn';
  check: (state: TaskState) => string | null;
};

const invariants: Invariant[] = [
  {
    name: 'done_requires_all_gates',
    severity: 'block',
    check(state) {
      if (state.status !== 'done') return null;
      const missing = state.requiredGates.filter(
        gate => !state.passedGates.includes(gate),
      );
      return missing.length ? `missing gates: ${missing.join(', ')}` : null;
    },
  },
  {
    name: 'verified_effect_requires_external_id',
    severity: 'block',
    check(state) {
      const bad = state.effects.find(
        effect => effect.status === 'verified' && !effect.externalId,
      );
      return bad ? `${bad.kind} verified without externalId` : null;
    },
  },
];

class InvariantMiddleware {
  assert(state: TaskState, phase: string) {
    const violations = invariants
      .map(rule => ({ rule, reason: rule.check(state) }))
      .filter(v => v.reason);

    const blocking = violations.filter(v => v.rule.severity === 'block');
    if (blocking.length) {
      throw new Error(
        JSON.stringify({
          type: 'InvariantViolation',
          phase,
          taskId: state.id,
          violations: blocking.map(v => ({
            name: v.rule.name,
            reason: v.reason,
          })),
        }),
      );
    }
  }
}

async function updateTaskState(
  state: TaskState,
  patch: Partial<TaskState>,
  guard: InvariantMiddleware,
) {
  const next = { ...state, ...patch };
  guard.assert(next, 'after_state_patch');
  await saveTaskState(next);
  return next;
}
```

生产上建议配合：

- structured log：记录 `phase/taskId/rule/reason`；
- dead letter queue：严重不一致的任务进入隔离区；
- replay：修复代码后用同一状态重放；
- dashboard：统计最常见 invariant violation。

---

## 5. OpenClaw 实战：课程 Cron 的不变量

以这个 Agent 开发课程 cron 为例，每次任务完成前至少应该满足：

```json
{
  "requiredInvariants": [
    "lesson_file_exists",
    "readme_contains_lesson",
    "tools_md_contains_topic",
    "telegram_message_has_id",
    "git_worktree_clean_after_push",
    "latest_commit_contains_lesson_file"
  ]
}
```

可以写一个小脚本作为验收闸门：

```bash
LESSON=243-state-invariants-consistency-assertions
TOPIC="Agent 状态不变量与一致性断言"

set -euo pipefail

test -f "lessons/${LESSON}.md"
grep -q "${LESSON}" README.md
grep -q "${TOPIC}" /Users/bot001/.openclaw/workspace/TOOLS.md
git diff --check
git status --short
```

如果 `git status --short` 还有输出，就不能说“完成”；如果 README 没目录，也不能说“完成”。

这就是不变量的价值：让 Agent 的完成不是靠自我感觉，而是靠可执行的事实。

---

## 6. 设计不变量的 5 条原则

1. **只写必须永远成立的规则**  
   不变量不是业务建议，而是破了就危险的底线。

2. **越靠近状态写入越好**  
   状态刚写坏就拦住，比最后发现容易修。

3. **外部副作用前再跑一次**  
   发消息、push、部署、扣款前必须复核关键前提。

4. **错误要给 LLM 和工程师都能读懂**  
   `InvariantViolation: missing gate git_push` 比 `assert failed` 有用 100 倍。

5. **允许 warn，但 block 要坚决**  
   文案长度超标可以 warn；付款状态不一致必须 block。

---

## 7. 常见不变量清单

Agent 项目可以从这些规则开始：

- `done` 状态必须有完成证据；
- 已验证的外部副作用必须有 `externalId`；
- 子 Agent 结果必须包含 `summary/status/evidence`；
- 高风险工具调用必须先有 approval record；
- 任务取消后不能继续执行外部写操作；
- 记忆写入必须带 source/authority；
- artifact 引用必须能被读取且 hash 匹配；
- PR push 前本地分支必须对应 open PR，不能是已 merge 分支；
- 需要 live data 的操作不能使用过期观察；
- 用户权限只能缩小，不能在子 Agent 中扩大。

---

## 8. 一句话总结

> Agent 状态不变量，就是把“这件事绝对不能错”从团队口头约定，升级成每次运行都会执行的安全带。

成熟 Agent 不只是会做事，还要能证明自己没有把系统状态做坏。
