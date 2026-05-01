# 219. Agent 任务租约与心跳续约（Task Lease & Heartbeat Renewal）

> 关键词：lease、heartbeat、zombie task、worker takeover、幂等恢复、长任务可靠性

很多 Agent Demo 都假设：任务一旦开始，进程就会一直活到结束。

生产环境里这个假设不成立：

- worker 可能被 K8s 重启；
- cron 任务可能跑到一半断网；
- sub-agent 可能卡死但状态仍显示 running；
- 用户以为任务还在做，其实进程早没了；
- 新 worker 不敢接手，因为数据库里任务状态还是 `running`。

解决办法不是“祈祷进程别死”，而是给任务加一层 **租约 Lease**：

> 谁拿到 lease，谁有权执行任务；执行期间必须定期 heartbeat 续约；超过 TTL 没续约，其他 worker 可以安全接管。

这节课讲一个非常实用的模式：**Task Lease + Heartbeat Renewal**。

---

## 1. 核心思想

任务状态不要只写：

```txt
status = running
```

而是至少写成：

```txt
status = running
leaseOwner = worker-7
leaseToken = 01J...
leaseExpiresAt = 2026-05-02T03:35:00Z
lastHeartbeatAt = 2026-05-02T03:34:30Z
checkpoint = {...}
```

关键规则：

1. **Acquire**：只有当任务未被占用，或 lease 已过期时，worker 才能抢占任务。
2. **Renew**：执行中定期续约，延长 `leaseExpiresAt`。
3. **Fence Token**：每次写结果都必须带 `leaseToken`，防止旧 worker 复活后乱写。
4. **Checkpoint**：每完成一个可恢复步骤就保存进度，接管时从 checkpoint 继续。
5. **Expire & Takeover**：lease 过期不等于任务失败，只表示当前 owner 失联，可由新 owner 接手。

一句话：**lease 解决“谁有权做”，checkpoint 解决“从哪里继续”。**

---

## 2. learn-claude-code：最小 Python 教学版

教学版可以用 SQLite 模拟任务表。

```python
# learn-claude-code/task_lease.py
import sqlite3
import time
import uuid
from contextlib import contextmanager

LEASE_TTL_SECONDS = 30

@contextmanager
def tx(db_path="tasks.db"):
    conn = sqlite3.connect(db_path)
    conn.row_factory = sqlite3.Row
    try:
        conn.execute("BEGIN IMMEDIATE")
        yield conn
        conn.commit()
    except Exception:
        conn.rollback()
        raise
    finally:
        conn.close()


def now() -> int:
    return int(time.time())


def acquire_task(task_id: str, worker_id: str) -> str | None:
    """返回 lease_token；拿不到则返回 None。"""
    token = str(uuid.uuid4())
    expires = now() + LEASE_TTL_SECONDS

    with tx() as conn:
        task = conn.execute(
            "SELECT status, lease_expires_at FROM tasks WHERE id = ?",
            (task_id,),
        ).fetchone()

        if not task:
            raise ValueError(f"task not found: {task_id}")

        lease_expired = task["lease_expires_at"] is None or task["lease_expires_at"] < now()
        claimable = task["status"] in ("pending", "running") and lease_expired

        if not claimable:
            return None

        conn.execute(
            """
            UPDATE tasks
            SET status = 'running',
                lease_owner = ?,
                lease_token = ?,
                lease_expires_at = ?,
                last_heartbeat_at = ?
            WHERE id = ?
            """,
            (worker_id, token, expires, now(), task_id),
        )

    return token


def renew_lease(task_id: str, lease_token: str) -> bool:
    with tx() as conn:
        cur = conn.execute(
            """
            UPDATE tasks
            SET lease_expires_at = ?, last_heartbeat_at = ?
            WHERE id = ? AND lease_token = ? AND status = 'running'
            """,
            (now() + LEASE_TTL_SECONDS, now(), task_id, lease_token),
        )
        return cur.rowcount == 1


def save_checkpoint(task_id: str, lease_token: str, checkpoint_json: str):
    with tx() as conn:
        cur = conn.execute(
            """
            UPDATE tasks
            SET checkpoint = ?
            WHERE id = ? AND lease_token = ? AND status = 'running'
            """,
            (checkpoint_json, task_id, lease_token),
        )
        if cur.rowcount != 1:
            raise RuntimeError("lost lease; stop writing")


def complete_task(task_id: str, lease_token: str, result_json: str):
    with tx() as conn:
        cur = conn.execute(
            """
            UPDATE tasks
            SET status = 'completed', result = ?, lease_expires_at = NULL
            WHERE id = ? AND lease_token = ? AND status = 'running'
            """,
            (result_json, task_id, lease_token),
        )
        if cur.rowcount != 1:
            raise RuntimeError("cannot complete: lease no longer owned")
```

执行循环里定期续约：

```python
async def run_task(task_id: str, worker_id: str):
    token = acquire_task(task_id, worker_id)
    if not token:
        return "someone else owns it"

    last_renew = time.time()

    for step in load_steps_from_checkpoint(task_id):
        # TTL 到一半就续约，不要等快过期才续
        if time.time() - last_renew > LEASE_TTL_SECONDS / 2:
            if not renew_lease(task_id, token):
                raise RuntimeError("lost lease; abort safely")
            last_renew = time.time()

        output = await execute_step(step)
        save_checkpoint(task_id, token, to_json({"lastStep": step.id, "output": output}))

    complete_task(task_id, token, to_json({"ok": True}))
```

注意这里每次写 checkpoint / complete 都带 `lease_token`，这就是 **fencing token**。

如果旧 worker 暂停了 1 分钟后恢复，它的 token 已经过期并被新 worker 替换，旧 worker 的写入会失败，不会污染结果。

---

## 3. pi-mono：生产版 TaskLeaseManager

生产里建议把 lease 做成一个基础设施服务，而不是散落在业务代码里。

```ts
// pi-mono/packages/agent-runtime/src/task-lease.ts
export interface TaskLease {
  taskId: string;
  ownerId: string;
  token: string;
  expiresAt: Date;
}

export interface TaskStore {
  acquireLease(input: {
    taskId: string;
    ownerId: string;
    token: string;
    ttlMs: number;
  }): Promise<TaskLease | null>;

  renewLease(input: {
    taskId: string;
    token: string;
    ttlMs: number;
  }): Promise<boolean>;

  writeCheckpoint(input: {
    taskId: string;
    token: string;
    checkpoint: unknown;
  }): Promise<void>;

  complete(input: {
    taskId: string;
    token: string;
    result: unknown;
  }): Promise<void>;
}

export class TaskLeaseManager {
  constructor(
    private readonly store: TaskStore,
    private readonly opts = { ttlMs: 30_000, renewEveryMs: 10_000 },
  ) {}

  async withLease<T>(
    taskId: string,
    ownerId: string,
    fn: (lease: TaskLease) => Promise<T>,
  ): Promise<T | null> {
    const token = crypto.randomUUID();
    const lease = await this.store.acquireLease({
      taskId,
      ownerId,
      token,
      ttlMs: this.opts.ttlMs,
    });

    if (!lease) return null;

    let stopped = false;
    const renewLoop = setInterval(async () => {
      if (stopped) return;
      const ok = await this.store.renewLease({
        taskId,
        token,
        ttlMs: this.opts.ttlMs,
      });
      if (!ok) {
        stopped = true;
        // 真实项目里这里应该触发 AbortController
        console.warn(`lost lease for task ${taskId}`);
      }
    }, this.opts.renewEveryMs);

    try {
      return await fn(lease);
    } finally {
      stopped = true;
      clearInterval(renewLoop);
    }
  }
}
```

Agent Runner 使用方式：

```ts
await leaseManager.withLease(taskId, workerId, async (lease) => {
  const controller = new AbortController();

  for await (const event of agent.run({ taskId, signal: controller.signal })) {
    if (event.type === "checkpoint") {
      await store.writeCheckpoint({
        taskId,
        token: lease.token,
        checkpoint: event.state,
      });
    }

    if (event.type === "final") {
      await store.complete({
        taskId,
        token: lease.token,
        result: event.result,
      });
    }
  }
});
```

更完整的生产实现要补两点：

- `renewLease` 失败时触发 `AbortController.abort()`，立即停止工具调用；
- 所有外部副作用工具都接收 `leaseToken` 或 `runId`，保证幂等写入。

---

## 4. OpenClaw：长任务与 Cron 的落地方式

OpenClaw 里很多任务天然是长生命周期：cron、heartbeat、sub-agent、后台 exec、消息发送流水线。

可以用一个轻量状态文件模拟 lease：

```json
{
  "taskId": "agent-course-cron",
  "owner": "BOT001:pid-12345",
  "leaseToken": "01HX...",
  "leaseExpiresAt": 1777650000,
  "lastHeartbeatAt": 1777649970,
  "checkpoint": {
    "lesson": 219,
    "phase": "lesson-written"
  }
}
```

OpenClaw 实战建议：

1. **cron 入口先 acquire lease**：避免同一个定时任务因重试/补偿并发跑两份。
2. **每个阶段写 checkpoint**：比如 `lesson-written`、`message-sent`、`git-pushed`。
3. **外部副作用要有幂等键**：Telegram 发课可以记录 `messageId`，避免接管后重复发。
4. **任务完成释放 lease**：状态写 `completed`，保留结果用于审计。
5. **heartbeat 检查 stale running**：发现 `leaseExpiresAt < now` 的任务，自动标记 `recoverable` 或重新入队。

这和 OpenClaw 的工作方式很契合：

- `sessions_spawn` / sub-agent 适合执行任务；
- `memory/*.json` 或 DB 适合存 checkpoint；
- cron/heartbeat 适合发现过期 lease；
- messageId / commit hash / artifact_id 适合作为幂等副作用记录。

---

## 5. 常见坑

### 坑 1：只检查 status，不检查 lease 过期

```ts
if (task.status === "running") return;
```

这会制造永久僵尸任务。正确做法：

```ts
if (task.status === "running" && task.leaseExpiresAt > now) return;
// running 但 lease 过期，可以接管
```

### 坑 2：续约间隔太接近 TTL

TTL 30 秒，29 秒才续约，网络一抖就丢 lease。

建议：

```txt
renewEvery = TTL / 3 或 TTL / 2
```

### 坑 3：旧 worker 复活后还能写结果

必须所有写操作都带 token：

```sql
UPDATE tasks
SET result = ?
WHERE id = ? AND lease_token = ?
```

没有 token 条件就是事故入口。

### 坑 4：外部副作用没有幂等键

比如发消息、扣款、创建 PR、部署服务，都必须有 `idempotencyKey`。

```ts
await telegram.sendMessage({
  chatId,
  text,
  idempotencyKey: `lesson:${lessonNo}:telegram`,
});
```

不是所有平台原生支持幂等键，那就自己在本地记录副作用结果：

```json
{
  "sideEffects": {
    "telegramMessage": 9457,
    "gitCommit": "7f04437"
  }
}
```

---

## 6. Checklist

给长任务加 lease 时，检查这 8 项：

- [ ] 任务表有 `leaseOwner` / `leaseToken` / `leaseExpiresAt`
- [ ] acquire 是原子操作
- [ ] renew 失败会停止当前执行
- [ ] checkpoint 写入带 `leaseToken`
- [ ] complete 写入带 `leaseToken`
- [ ] lease 过期后允许新 worker 接管
- [ ] 外部副作用有幂等记录
- [ ] 有后台巡检 stale running tasks

---

## 7. 总结

Task Lease 是 Agent 长任务可靠性的地基。

没有 lease：

```txt
running = 也许在跑，也许死了，也许重复跑了
```

有 lease：

```txt
running + 未过期 lease = 明确有人负责
running + 过期 lease = 可安全接管
写入带 token = 旧 owner 无法污染结果
checkpoint = 接管后能继续，不用从头来
```

记住这句话：

> 长任务不是靠“进程一直活着”完成的，而是靠 lease、heartbeat、checkpoint 和幂等副作用完成的。

这就是 Demo Agent 和生产 Agent 的分水岭。
