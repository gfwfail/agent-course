# 220. Agent 死信队列与失败任务恢复（Dead Letter Queue & Failed Task Recovery）

> 关键词：DLQ、retry exhausted、failed task recovery、poison message、人工复核、幂等重放

上一课讲了 **Task Lease + Heartbeat**：解决长任务执行中 worker 掉线、僵尸任务、任务接管的问题。

但还有一个更现实的问题：

> 任务没有掉线，而是反复失败；重试很多次以后，系统应该怎么办？

很多 Agent 系统一开始只做简单重试：

```txt
失败 → retry → 失败 → retry → 失败 → 标记 failed
```

这在 Demo 里够用，生产里不够。

因为失败任务里往往藏着真正重要的信息：

- 工具 Schema 改了，老参数不兼容；
- 用户给的数据格式异常；
- 外部 API 持续 500；
- 某类 prompt 总是导致错误工具调用；
- Agent 生成了一个“毒任务”，每次执行都会污染下游。

如果只是 `status = failed`，这些问题会沉进数据库没人看。

正确做法是：失败超过阈值后，把任务送进 **死信队列 DLQ（Dead Letter Queue）**。

一句话：**Retry 负责自动恢复，DLQ 负责可观测、可复盘、可人工介入的失败闭环。**

---

## 1. 核心思想

Agent 任务失败不要只存：

```txt
status = failed
error = "Tool call failed"
```

而要记录成可恢复的失败档案：

```txt
status = dead_letter
attempts = 5
lastError = {
  type: "SCHEMA_VALIDATION_ERROR",
  message: "missing required field: projectId",
  recoverable: false,
  toolName: "deploy_app",
  stepId: "deploy"
}
failureHistory = [...]
checkpoint = {...}
replayableInput = {...}
deadLetterReason = "retry_exhausted"
createdAt = "2026-05-02T06:30:00+11:00"
```

DLQ 至少要回答 5 个问题：

1. **为什么失败？** 错误类型、工具名、步骤、原始错误。
2. **失败了几次？** attempts、首次失败时间、最后失败时间。
3. **还能不能恢复？** recoverable / needs_human / permanent。
4. **从哪里恢复？** checkpoint、lastSuccessfulStep。
5. **如何重放？** replayableInput、idempotencyKey、lease/fencing 信息。

没有这些信息，`failed` 只是一个墓碑；有了这些信息，DLQ 就是故障分析入口。

---

## 2. 失败分类：不是所有失败都该重试

Agent 失败大致分四类：

| 类型 | 例子 | 策略 |
|---|---|---|
| transient | 网络抖动、HTTP 502、超时 | 指数退避重试 |
| rate_limited | 429、配额暂时耗尽 | 按 Retry-After 延迟 |
| correctable | 参数缺失、格式错误、工具返回可修复 hint | 让 LLM 修正后重试 |
| poison | Schema 永久不兼容、权限不足、用户输入非法 | 进 DLQ，等待人工/规则修复 |

关键点：

> DLQ 不是“所有错误的垃圾桶”，而是“自动恢复已经无效，需要保留证据并升级处理的任务区”。

---

## 3. learn-claude-code：最小 Python 教学版

教学版可以用 SQLite 做一个非常小的失败任务队列。

```python
# learn-claude-code/dead_letter_queue.py
import json
import sqlite3
import time
from dataclasses import dataclass, asdict

MAX_ATTEMPTS = 5


@dataclass
class AgentError:
    type: str
    message: str
    recoverable: bool
    tool_name: str | None = None
    step_id: str | None = None
    suggestion: str | None = None


def now() -> int:
    return int(time.time())


def record_failure(conn: sqlite3.Connection, task_id: str, error: AgentError):
    task = conn.execute(
        "SELECT attempts, failure_history FROM tasks WHERE id = ?",
        (task_id,),
    ).fetchone()

    attempts = task[0] + 1
    history = json.loads(task[1] or "[]")
    history.append({"at": now(), "error": asdict(error)})

    should_dead_letter = attempts >= MAX_ATTEMPTS or not error.recoverable

    if should_dead_letter:
        conn.execute(
            """
            UPDATE tasks
            SET status = 'dead_letter',
                attempts = ?,
                last_error = ?,
                failure_history = ?,
                dead_letter_reason = ?
            WHERE id = ?
            """,
            (
                attempts,
                json.dumps(asdict(error), ensure_ascii=False),
                json.dumps(history, ensure_ascii=False),
                "retry_exhausted" if attempts >= MAX_ATTEMPTS else "non_recoverable",
                task_id,
            ),
        )
        return "dead_letter"

    conn.execute(
        """
        UPDATE tasks
        SET status = 'pending',
            attempts = ?,
            next_run_at = ?,
            last_error = ?,
            failure_history = ?
        WHERE id = ?
        """,
        (
            attempts,
            now() + backoff_seconds(attempts),
            json.dumps(asdict(error), ensure_ascii=False),
            json.dumps(history, ensure_ascii=False),
            task_id,
        ),
    )
    return "retry_scheduled"


def backoff_seconds(attempts: int) -> int:
    # 2, 4, 8, 16... capped at 5 minutes
    return min(2 ** attempts, 300)
```

执行任务时不要直接吞异常：

```python
def classify_error(exc: Exception) -> AgentError:
    msg = str(exc)

    if "429" in msg:
        return AgentError(
            type="RATE_LIMITED",
            message=msg,
            recoverable=True,
            suggestion="retry after provider cooldown",
        )

    if "missing required field" in msg:
        return AgentError(
            type="SCHEMA_VALIDATION_ERROR",
            message=msg,
            recoverable=False,
            suggestion="inspect tool schema and regenerate tool arguments",
        )

    return AgentError(
        type="UNKNOWN",
        message=msg,
        recoverable=True,
        suggestion="retry with exponential backoff",
    )


async def run_task(conn, task):
    try:
        result = await execute_agent_steps(task)
        mark_completed(conn, task["id"], result)
    except Exception as exc:
        error = classify_error(exc)
        outcome = record_failure(conn, task["id"], error)
        if outcome == "dead_letter":
            notify_operator(task["id"], error)
```

这个版本很小，但已经具备生产系统最重要的结构：

- 失败历史可追溯；
- 可恢复错误继续排队；
- 不可恢复或超过次数进入 DLQ；
- 进入 DLQ 时通知人工；
- 后续可以从 checkpoint 重放。

---

## 4. pi-mono：生产版 FailedTaskManager

生产里建议把失败处理做成 TaskRuntime 的基础设施，而不是散落在每个工具里。

```ts
// pi-mono/packages/agent-runtime/src/failed-task-manager.ts
export type FailureKind =
  | "transient"
  | "rate_limited"
  | "correctable"
  | "poison";

export interface AgentFailure {
  kind: FailureKind;
  type: string;
  message: string;
  recoverable: boolean;
  toolName?: string;
  stepId?: string;
  suggestion?: string;
  raw?: unknown;
}

export interface DeadLetterEntry {
  taskId: string;
  runId: string;
  userId?: string;
  attempts: number;
  reason: "retry_exhausted" | "non_recoverable" | "manual_quarantine";
  lastFailure: AgentFailure;
  failureHistory: Array<{ at: string; failure: AgentFailure }>;
  checkpoint?: unknown;
  replayableInput: unknown;
  idempotencyKey?: string;
  createdAt: string;
}

export interface TaskStore {
  scheduleRetry(input: {
    taskId: string;
    attempts: number;
    nextRunAt: Date;
    failure: AgentFailure;
  }): Promise<void>;

  moveToDeadLetter(entry: DeadLetterEntry): Promise<void>;

  loadCheckpoint(taskId: string): Promise<unknown>;
}

export class FailedTaskManager {
  constructor(
    private store: TaskStore,
    private maxAttempts = 5,
  ) {}

  async handleFailure(input: {
    taskId: string;
    runId: string;
    attempts: number;
    failure: AgentFailure;
    replayableInput: unknown;
    idempotencyKey?: string;
  }): Promise<"retry" | "dead_letter"> {
    const attempts = input.attempts + 1;

    const shouldDLQ =
      attempts >= this.maxAttempts ||
      input.failure.kind === "poison" ||
      input.failure.recoverable === false;

    if (shouldDLQ) {
      const checkpoint = await this.store.loadCheckpoint(input.taskId);

      await this.store.moveToDeadLetter({
        taskId: input.taskId,
        runId: input.runId,
        attempts,
        reason: attempts >= this.maxAttempts ? "retry_exhausted" : "non_recoverable",
        lastFailure: input.failure,
        failureHistory: [{ at: new Date().toISOString(), failure: input.failure }],
        checkpoint,
        replayableInput: input.replayableInput,
        idempotencyKey: input.idempotencyKey,
        createdAt: new Date().toISOString(),
      });

      return "dead_letter";
    }

    await this.store.scheduleRetry({
      taskId: input.taskId,
      attempts,
      nextRunAt: new Date(Date.now() + this.backoffMs(attempts, input.failure)),
      failure: input.failure,
    });

    return "retry";
  }

  private backoffMs(attempts: number, failure: AgentFailure): number {
    if (failure.kind === "rate_limited") return Math.min(60_000 * attempts, 15 * 60_000);
    return Math.min(1_000 * 2 ** attempts, 5 * 60_000);
  }
}
```

Agent Loop 中集成：

```ts
try {
  await agentRuntime.run(task);
  await taskStore.markCompleted(task.id);
} catch (error) {
  const failure = classifyAgentFailure(error);

  const outcome = await failedTaskManager.handleFailure({
    taskId: task.id,
    runId: task.runId,
    attempts: task.attempts,
    failure,
    replayableInput: task.input,
    idempotencyKey: task.idempotencyKey,
  });

  if (outcome === "dead_letter") {
    await alerts.send({
      level: "warning",
      title: "Agent task moved to DLQ",
      body: `${task.id}: ${failure.type} - ${failure.message}`,
    });
  }
}
```

这里的重点不是代码多复杂，而是边界清楚：

- `AgentRuntime` 负责执行；
- `classifyAgentFailure` 负责错误归类；
- `FailedTaskManager` 负责重试 / DLQ 决策；
- `TaskStore` 负责持久化；
- `alerts` 负责通知。

这就是生产代码和 Demo 代码最大的区别：**失败路径也是一等公民。**

---

## 5. OpenClaw：cron / sub-agent 失败恢复实战

OpenClaw 里有很多天然适合 DLQ 的场景：

- cron 定时课程发布；
- 长任务 sub-agent 修 bug；
- GitHub issue 自动处理；
- 邮件/日历/社交通知巡检；
- 外部 API 写操作（发消息、push、部署）。

一个实用的 OpenClaw 文件式 DLQ 可以长这样：

```txt
memory/dlq/
  2026-05-02-agent-course-0630.json
  2026-05-02-github-issue-184.json
```

记录内容：

```json
{
  "task": "agent-course-cron",
  "runId": "3eba6ee3-2d11-4afa-aa7f-a47b63226982",
  "status": "dead_letter",
  "reason": "push_failed",
  "attempts": 3,
  "lastSuccessfulStep": "lesson_written",
  "nextHumanAction": "check gh auth and rerun git push",
  "checkpoint": {
    "lessonFile": "lessons/220-dead-letter-queue-failed-task-recovery.md",
    "messageDraftSaved": true,
    "readmeUpdated": true
  },
  "error": {
    "type": "GIT_PUSH_FAILED",
    "message": "remote rejected main",
    "recoverable": true
  },
  "createdAt": "2026-05-02T06:30:00+11:00"
}
```

这样下一次 heartbeat 或人工接管时，Agent 不需要重新猜：

- 已经写了 lesson，不要重复生成；
- README 已更新，不要重复插入目录；
- 群消息可能还没发，要检查 messageId；
- git push 失败，优先处理认证/冲突；
- 从 checkpoint 继续，而不是从头再做。

如果配合上一课的 lease，就能形成完整可靠任务系统：

```txt
lease 保证同一时间只有一个 worker 做
checkpoint 保证失败后能继续
retry 处理短暂问题
DLQ 处理自动恢复不了的问题
idempotencyKey 保证重放不产生重复副作用
```

---

## 6. DLQ 后台管理界面应该显示什么？

不要只做一个“失败任务列表”。真正有用的 DLQ 页面应该有：

```txt
Task ID       Type          Attempts  Last Error              Action
agent-0630    cron_course   5         GIT_PUSH_FAILED         Retry / Edit / Ignore
issue-184     gh_fix        3         TEST_FAILED             Open Logs / Spawn Debug Agent
email-991     inbox_triage  1         PERMISSION_DENIED       Request Approval
```

每条任务至少提供 4 个动作：

1. **Retry**：原输入原 checkpoint 重放。
2. **Retry with patch**：人工修正参数后重放。
3. **Ignore / Archive**：确认不需要处理。
4. **Spawn debug agent**：把失败上下文交给子 Agent 分析。

对于 Agent 平台，第四个动作很强：

> DLQ 不只是给人看的，也可以成为“故障自愈 Agent”的输入源。

比如每天定时扫描 DLQ：

```ts
const entries = await dlq.list({ status: "open", limit: 20 });

for (const entry of entries) {
  if (entry.lastFailure.type === "SCHEMA_VALIDATION_ERROR") {
    await spawnSubAgent({
      task: "Analyze this failed task and propose a schema migration",
      context: entry,
    });
  }
}
```

这就是从“失败记录”升级为“自动修复入口”。

---

## 7. 常见坑

### 坑 1：DLQ 里没有 replayable input

只存错误，不存原始输入，后面无法重放。

正确做法：任务创建时就保存 `replayableInput`，并确保里面不包含会过期的临时引用。

---

### 坑 2：重放没有幂等键

DLQ 任务重试时可能重复发消息、重复扣款、重复部署。

正确做法：所有外部副作用都带 `idempotencyKey`，重放时复用原 key 或显式生成 replay key。

---

### 坑 3：所有错误都无限重试

无限重试会把毒任务变成系统负载攻击。

正确做法：设置 maxAttempts，并对 poison / non_recoverable 错误直接进 DLQ。

---

### 坑 4：DLQ 没有负责人

进了 DLQ 但没人看，等于换个表继续烂。

正确做法：DLQ 要接告警、Dashboard、定期巡检，超过 SLA 未处理要升级。

---

## 8. 和之前课程的关系

- 和 **错误消息工程化**：DLQ 依赖结构化错误，不然无法分类。
- 和 **幂等性与重试**：DLQ 是重试耗尽后的下一站。
- 和 **任务租约与心跳续约**：lease 解决执行权，DLQ 解决失败归宿。
- 和 **工具调用回放**：DLQ 提供 replayable input，回放系统负责复现。
- 和 **Agent 自愈机制**：DLQ 是自愈系统最重要的数据源之一。

---

## 9. 设计 Checklist

做 Agent 任务系统时，问自己 8 个问题：

- [ ] 每个任务是否记录 attempts？
- [ ] 是否区分 transient / rate_limited / correctable / poison？
- [ ] 是否有 maxAttempts？
- [ ] 失败历史是否 append-only 保存？
- [ ] 是否保存 checkpoint 和 replayableInput？
- [ ] 外部副作用是否有 idempotencyKey？
- [ ] DLQ 是否有告警和负责人？
- [ ] 是否支持人工修正后重放？

如果这些问题答不上来，说明你的 Agent 只设计了“成功路径”，还没设计“失败路径”。

---

## 总结

Agent 系统一定会失败，关键不是让失败消失，而是让失败变得：

- 可见；
- 可分类；
- 可重放；
- 可恢复；
- 可人工介入；
- 可用于系统改进。

**Retry 是自动恢复的第一道防线，DLQ 是失败闭环的最后保险。**

没有 DLQ 的 Agent 平台，失败会悄悄堆积成事故；有 DLQ 的 Agent 平台，失败会变成下一轮优化的数据。