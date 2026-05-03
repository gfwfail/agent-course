# 231. Agent 任务优先级继承与防优先级反转（Priority Inheritance & Inversion Avoidance）

> 核心观点：Agent 有了优先级队列还不够。  
> **高优先级任务依赖低优先级子任务时，优先级必须沿依赖链继承，否则老板的紧急任务会被后台摘要卡住。**

---

## 1. 问题：优先级队列也会翻车

很多 Agent 系统会把任务分成：

```text
critical > high > normal > low
```

看起来很合理，但生产里会遇到一个经典问题：**优先级反转**。

例子：

```text
T1: low    每日整理日志，占着 DB export 锁
T2: normal 批量生成课程，排队等模型
T3: high   老板问“生产为什么挂了”，需要同一个 DB export
```

如果系统只看任务创建时的优先级，T3 会被 T1 卡住。更糟糕的是，T1 可能还会被更多 normal 任务抢占，导致 high 任务一直等。

这就是 Priority Inversion：

```text
高优先级任务 → 等低优先级任务释放资源
低优先级任务 → 又被中优先级任务打断
最终：高优先级任务最慢
```

Agent 系统里更常见，因为任务经常拆成 Sub-agent、工具调用、后台 job、cron、队列任务。

---

## 2. 解决思路：优先级继承

规则很简单：

> 当高优先级任务等待某个低优先级任务/资源时，低优先级任务临时继承高优先级，直到释放依赖。

也就是：

```text
high task waits for low task
→ low task effectivePriority = high
→ 调度器先让 low task 跑完释放资源
→ high task 继续执行
→ low task 恢复原优先级
```

关键是区分两个字段：

```ts
type Task = {
  id: string;
  basePriority: Priority;       // 创建时的原始优先级
  effectivePriority: Priority;  // 调度时使用的动态优先级
  waitingFor?: string[];        // 正在等待哪些 task/resource
};
```

`basePriority` 是身份，`effectivePriority` 是当前调度权重。

---

## 3. learn-claude-code：Python 教学版

下面是一个最小可运行的优先级继承调度器。重点不是队列实现，而是 `inherit_priority()` 这一步。

```python
# priority_inheritance.py
from dataclasses import dataclass, field
from enum import IntEnum
from heapq import heappush, heappop


class Priority(IntEnum):
    LOW = 1
    NORMAL = 2
    HIGH = 3
    CRITICAL = 4


@dataclass
class Task:
    id: str
    base_priority: Priority
    effective_priority: Priority | None = None
    waiting_for: list[str] = field(default_factory=list)
    done: bool = False

    def __post_init__(self):
        if self.effective_priority is None:
            self.effective_priority = self.base_priority


class Scheduler:
    def __init__(self):
        self.tasks: dict[str, Task] = {}

    def add(self, task: Task):
        self.tasks[task.id] = task

    def inherit_priority(self):
        # 每轮先重置，避免 boost 永久污染
        for t in self.tasks.values():
            t.effective_priority = t.base_priority

        changed = True
        while changed:
            changed = False
            for waiter in self.tasks.values():
                if waiter.done:
                    continue
                for blocked_id in waiter.waiting_for:
                    blocked = self.tasks.get(blocked_id)
                    if not blocked or blocked.done:
                        continue

                    if blocked.effective_priority < waiter.effective_priority:
                        blocked.effective_priority = waiter.effective_priority
                        changed = True

    def next_task(self) -> Task | None:
        self.inherit_priority()
        heap = []
        for t in self.tasks.values():
            if t.done:
                continue
            # 还在等待别人的任务不能直接运行
            if any(not self.tasks[x].done for x in t.waiting_for if x in self.tasks):
                continue
            heappush(heap, (-int(t.effective_priority), t.id, t))
        return heappop(heap)[2] if heap else None


s = Scheduler()
s.add(Task(id="daily-log-export", base_priority=Priority.LOW))
s.add(Task(id="incident-debug", base_priority=Priority.HIGH, waiting_for=["daily-log-export"]))
s.add(Task(id="normal-summary", base_priority=Priority.NORMAL))

print(s.next_task().id)
# daily-log-export
# 它本来是 LOW，但被 incident-debug 继承成 HIGH，所以先跑完释放依赖。
```

没有继承时，下一个会是 `normal-summary`；有继承时，先跑 `daily-log-export`，高优先级事故排查反而更快。

---

## 4. pi-mono：生产版任务调度设计

生产里建议把优先级继承放在 TaskStore / QueueMiddleware 层，而不是让 LLM 自己决定。

```ts
type Priority = 'low' | 'normal' | 'high' | 'critical';

const rank: Record<Priority, number> = {
  low: 1,
  normal: 2,
  high: 3,
  critical: 4,
};

type TaskRecord = {
  id: string;
  basePriority: Priority;
  effectivePriority: Priority;
  status: 'queued' | 'running' | 'waiting' | 'done' | 'failed';
  waitingForTaskIds: string[];
  waitingForResourceKeys: string[];
  boostedBy: string[];
};

export function recomputeEffectivePriorities(tasks: TaskRecord[]): TaskRecord[] {
  const byId = new Map(tasks.map(t => [t.id, { ...t, effectivePriority: t.basePriority, boostedBy: [] }]));

  let changed = true;
  while (changed) {
    changed = false;

    for (const waiter of byId.values()) {
      if (waiter.status === 'done' || waiter.status === 'failed') continue;

      for (const depId of waiter.waitingForTaskIds) {
        const dep = byId.get(depId);
        if (!dep || dep.status === 'done' || dep.status === 'failed') continue;

        if (rank[dep.effectivePriority] < rank[waiter.effectivePriority]) {
          dep.effectivePriority = waiter.effectivePriority;
          dep.boostedBy = [...new Set([...dep.boostedBy, waiter.id])];
          changed = true;
        }
      }
    }
  }

  return [...byId.values()];
}
```

调度器取任务时只看 `effectivePriority`：

```ts
export function pickNextRunnable(tasks: TaskRecord[]): TaskRecord | undefined {
  const boosted = recomputeEffectivePriorities(tasks);

  return boosted
    .filter(t => t.status === 'queued')
    .filter(t => t.waitingForTaskIds.length === 0)
    .sort((a, b) => rank[b.effectivePriority] - rank[a.effectivePriority])[0];
}
```

生产版还要加三条保护：

1. **Boost TTL**：继承优先级不能无限持续，避免低优先级任务永久霸占高优先级。
2. **Boost reason**：记录 `boostedBy`，方便解释“为什么这个低优先级任务突然先跑”。
3. **Cycle detection**：任务依赖成环时直接进 DLQ 或人工处理。

---

## 5. OpenClaw 实战：Cron / Sub-agent / Heartbeat 怎么用

OpenClaw 这类 Always-on Agent 最容易出现三类优先级反转：

### 场景 A：Heartbeat 后台维护卡住人工请求

```text
heartbeat memory maintenance 正在跑 git/status/read 大量文件
老板突然要求排查线上事故
```

做法：

- 用户直接请求 = `high`
- cron 课程 = `normal`
- memory 整理 = `low`
- 如果 high 请求依赖 low 任务持有的工作区锁，low 临时 boost 到 high，快速释放锁

### 场景 B：Sub-agent fan-out 中某个低优先级子任务成关键路径

```text
主任务 high
├─ 子任务 A normal：查日志
├─ 子任务 B low：拉取历史配置
└─ 最终答案依赖 B
```

如果 B 是最终答案关键路径，它就不该继续按 low 跑。父任务 high 等它时，B 应继承 high。

### 场景 C：外部副作用锁

比如发送 Telegram、git push、部署、写 MEMORY.md，都应该有资源锁：

```text
resource:telegram:-5115329245
resource:repo:agent-course
resource:memory:MEMORY.md
```

当 high 任务等待这些锁时，锁持有者应被 boost，优先完成并释放。

---

## 6. 一张决策表

| 情况 | 是否继承优先级 | 原因 |
|---|---:|---|
| high 等 low 释放资源锁 | 是 | 典型优先级反转 |
| high 只是排在 low 后面，没有依赖 | 否 | 普通队列排序即可 |
| low 是 high 的子任务关键路径 | 是 | 父任务等待它完成 |
| low 是可选增强任务 | 否 | 可跳过，不该 boost |
| 依赖链成环 | 否，进 DLQ | boost 解决不了死锁 |
| 外部 API 限流中 | 部分继承 | 可提高队列顺序，但不能突破限流 |

---

## 7. 常见坑

### 坑 1：只给任务设优先级，不给资源设锁

如果不知道任务在等什么，就无法判断是否需要继承。至少要记录：

```text
waitingForTaskIds
waitingForResourceKeys
heldResourceKeys
```

### 坑 2：Boost 后忘记恢复

`effectivePriority` 应该每轮从 `basePriority` 重新计算，不要永久写死。

### 坑 3：把所有子任务都 boost

只有“父任务正在等待的关键路径子任务”才继承优先级。可选任务、预取任务、后台增强任务不该 boost。

### 坑 4：优先级绕过安全策略

Priority 只影响“先后顺序”，不能绕过：

- 审批
- 权限
- 速率限制
- 成本配额
- 副作用两阶段提交

`critical` 也不能免审批。紧急不等于无权限。

---

## 8. 生产 Checklist

- [ ] 任务同时有 `basePriority` 和 `effectivePriority`
- [ ] 记录 `waitingForTaskIds / waitingForResourceKeys`
- [ ] 调度前重新计算继承链
- [ ] Boost 有 TTL 和 reason
- [ ] 依赖图做 cycle detection
- [ ] 可选子任务不继承父优先级
- [ ] 优先级不绕过安全策略
- [ ] 日志能解释：任务为什么被提前执行

---

## 9. 总结

普通优先级队列解决的是：

```text
谁更重要，谁先跑
```

优先级继承解决的是：

```text
谁挡住了重要任务，谁先把路让开
```

Agent 系统越自动化，后台任务、cron、sub-agent、工具锁就越多。没有 Priority Inheritance，高优先级请求会被低优先级维护任务悄悄拖死。

一句话：**调度不是只看任务重要性，还要看依赖链上的阻塞关系。**
