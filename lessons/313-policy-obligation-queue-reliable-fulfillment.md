# 313 - Agent 策略义务队列与可靠履行：别让 post-check 丢在半路

> 关键词：Policy Obligation Queue、Reliable Fulfillment、Retry、Dead Letter、Post-Action Governance  
> 目标：把上一课的 `obligations` 从“同步执行一遍”升级成“可重试、可恢复、可审计的履行队列”。

上一课我们讲了：Policy 决定不应该只有 `allow/deny`，还应该带上执行后的义务：记录证据、post-check、通知、复查、补偿。

但生产里还有一个坑：**义务本身也会失败**。

比如：

- `message.send` 成功了，但保存 messageId 的审计存储暂时不可用；
- `deploy.trigger` 成功了，但 health check endpoint 30 秒内一直 503；
- `git.push` 成功了，但 GitHub API rate limit，没法立刻验证远端 commit；
- `schedule_review` 要创建未来任务，但调度器短暂重启。

如果 obligation 只在主流程里同步跑一次，Agent 很容易出现最糟糕的状态：**副作用已经发生，但后置义务没履行，也没人知道。**

所以今天补一层：**Policy Obligation Queue（策略义务队列）**。

---

## 1. 核心原则：副作用完成后，义务必须进入 durable queue

上一课的同步模型是：

```text
policy allow -> execute tool -> run obligations inline -> done
```

更可靠的生产模型应该是：

```text
policy allow
  -> execute tool
  -> enqueue obligation jobs durably
  -> worker retry until fulfilled / dead-letter / escalated
```

关键区别：

- 主动作成功后，先把 obligation 写入持久化队列；
- obligation worker 可以独立重试，不依赖当前 LLM turn 活着；
- 每次尝试都写 audit event；
- 超过重试预算进入 dead-letter，并按风险升级；
- required obligation 最终失败时，触发 compensation / incident / human review。

这和普通任务队列不一样：obligation queue 的语义是“策略契约必须履行”，不是普通后台任务。

---

## 2. obligation job 最少要存什么？

建议每个 job 至少有这些字段：

```json
{
  "id": "obl_01HX...",
  "runId": "run_20260514_0630",
  "toolCallId": "tool_git_push_1",
  "type": "post_check",
  "required": true,
  "risk": "medium",
  "params": { "check": "remote_contains_commit", "commit": "abc123" },
  "status": "pending",
  "attempts": 0,
  "maxAttempts": 8,
  "nextRunAt": 1778690000,
  "evidence": {
    "policyVersion": "policy-v42",
    "decisionId": "dec_abc",
    "target": "gfwfail/agent-course"
  }
}
```

几个字段特别重要：

- `required`：最终失败是否必须升级；
- `risk`：决定重试节奏和升级路径；
- `evidence`：把 obligation 跟 policy decision、工具结果、目标资源绑定起来；
- `nextRunAt`：支持指数退避，避免 tight loop；
- `maxAttempts`：防无限重试。

---

## 3. learn-claude-code：SQLite 教学版

教学版用 SQLite 模拟 durable queue。真实系统可以换成 Postgres / Redis Streams / SQS。

```python
import json
import sqlite3
import time
import uuid
from dataclasses import dataclass, asdict, field
from typing import Any, Literal

Status = Literal["pending", "running", "fulfilled", "dead_letter"]

@dataclass
class ObligationJob:
    id: str
    run_id: str
    tool_call_id: str
    type: str
    required: bool
    risk: str
    params: dict[str, Any]
    evidence: dict[str, Any]
    status: Status = "pending"
    attempts: int = 0
    max_attempts: int = 6
    next_run_at: int = field(default_factory=lambda: int(time.time()))
    last_error: str | None = None

class ObligationQueue:
    def __init__(self, path="obligations.sqlite"):
        self.db = sqlite3.connect(path)
        self.db.execute("""
        create table if not exists obligation_jobs (
          id text primary key,
          payload text not null,
          status text not null,
          next_run_at integer not null
        )
        """)
        self.db.execute("""
        create table if not exists obligation_audit (
          id text primary key,
          job_id text not null,
          ts integer not null,
          event text not null,
          details text not null
        )
        """)

    def enqueue(self, job: ObligationJob) -> None:
        self.db.execute(
            "insert into obligation_jobs values (?, ?, ?, ?)",
            (job.id, json.dumps(asdict(job), ensure_ascii=False), job.status, job.next_run_at),
        )
        self.audit(job.id, "enqueued", {"type": job.type, "required": job.required})
        self.db.commit()

    def claim_due(self, now: int) -> ObligationJob | None:
        row = self.db.execute(
            "select id, payload from obligation_jobs where status = 'pending' and next_run_at <= ? order by next_run_at limit 1",
            (now,),
        ).fetchone()
        if not row:
            return None

        job = ObligationJob(**json.loads(row[1]))
        job.status = "running"
        self._save(job)
        self.audit(job.id, "claimed", {"attempts": job.attempts})
        self.db.commit()
        return job

    def mark_fulfilled(self, job: ObligationJob, details: dict[str, Any]) -> None:
        job.status = "fulfilled"
        self._save(job)
        self.audit(job.id, "fulfilled", details)
        self.db.commit()

    def retry_or_dead_letter(self, job: ObligationJob, error: Exception) -> None:
        job.attempts += 1
        job.last_error = str(error)

        if job.attempts >= job.max_attempts:
            job.status = "dead_letter"
            self._save(job)
            self.audit(job.id, "dead_letter", {"error": str(error), "required": job.required})
        else:
            delay = min(60 * 30, 2 ** job.attempts * 10)  # capped exponential backoff
            job.status = "pending"
            job.next_run_at = int(time.time()) + delay
            self._save(job)
            self.audit(job.id, "retry_scheduled", {"error": str(error), "delay": delay})

        self.db.commit()

    def audit(self, job_id: str, event: str, details: dict[str, Any]) -> None:
        self.db.execute(
            "insert into obligation_audit values (?, ?, ?, ?, ?)",
            (str(uuid.uuid4()), job_id, int(time.time()), event, json.dumps(details, ensure_ascii=False)),
        )

    def _save(self, job: ObligationJob) -> None:
        self.db.execute(
            "update obligation_jobs set payload = ?, status = ?, next_run_at = ? where id = ?",
            (json.dumps(asdict(job), ensure_ascii=False), job.status, job.next_run_at, job.id),
        )

def fulfill(job: ObligationJob) -> dict[str, Any]:
    if job.type == "record_evidence":
        return {"recorded": True, "target": job.evidence.get("target")}

    if job.type == "post_check":
        # 教学版：真实系统里查 GitHub API、health endpoint、message receipt 等
        if not job.params.get("observed_ok", False):
            raise RuntimeError("post_check_not_ready")
        return {"check": job.params["check"], "ok": True}

    if job.type == "notify":
        return {"sent": True, "channel": job.params.get("channel")}

    raise RuntimeError(f"unknown obligation type: {job.type}")

# enqueue after tool success
queue = ObligationQueue(":memory:")
queue.enqueue(ObligationJob(
    id="obl_" + uuid.uuid4().hex,
    run_id="run_20260514_0630",
    tool_call_id="tool_git_push_1",
    type="post_check",
    required=True,
    risk="medium",
    params={"check": "remote_contains_commit", "commit": "abc123", "observed_ok": True},
    evidence={"policyVersion": "policy-v42", "target": "gfwfail/agent-course"},
))

# worker loop tick
job = queue.claim_due(int(time.time()))
if job:
    try:
        details = fulfill(job)
        queue.mark_fulfilled(job, details)
    except Exception as e:
        queue.retry_or_dead_letter(job, e)
```

这段代码故意很朴素，但已经具备生产语义：

- `enqueue` 是 durable write；
- `claim_due` 支持 worker 异步领取；
- `retry_or_dead_letter` 有退避和上限；
- `obligation_audit` 能证明每次尝试。

---

## 4. pi-mono：生产中间件形态

在 TypeScript 生产实现里，可以把 obligation queue 做成 Tool Middleware 的后置阶段：

```ts
type ObligationStatus = "pending" | "running" | "fulfilled" | "dead_letter";

type PolicyObligation = {
  type: "record_evidence" | "post_check" | "notify" | "schedule_review" | "compensating_action";
  required: boolean;
  risk: "low" | "medium" | "high" | "critical";
  params: Record<string, unknown>;
};

type ObligationJob = {
  id: string;
  runId: string;
  toolCallId: string;
  obligation: PolicyObligation;
  policyDecisionId: string;
  toolResultRef: string;
  status: ObligationStatus;
  attempts: number;
  maxAttempts: number;
  nextRunAt: Date;
};

interface ObligationStore {
  enqueueMany(jobs: ObligationJob[]): Promise<void>;
  claimDue(limit: number, now: Date): Promise<ObligationJob[]>;
  markFulfilled(id: string, evidence: unknown): Promise<void>;
  markRetry(id: string, error: Error, nextRunAt: Date): Promise<void>;
  markDeadLetter(id: string, error: Error): Promise<void>;
}

export class PolicyObligationQueueMiddleware {
  constructor(private store: ObligationStore) {}

  async afterToolExecute(ctx: ToolContext, result: ToolResult) {
    const decision = ctx.policyDecision;
    if (!decision?.obligations?.length) return result;

    const jobs = decision.obligations.map((obligation) => ({
      id: crypto.randomUUID(),
      runId: ctx.runId,
      toolCallId: ctx.toolCallId,
      obligation,
      policyDecisionId: decision.id,
      toolResultRef: result.evidenceRef,
      status: "pending" as const,
      attempts: 0,
      maxAttempts: obligation.risk === "critical" ? 12 : 6,
      nextRunAt: new Date(),
    }));

    // 注意：如果 enqueueMany 失败，不能假装完成。
    // 对 required obligation，应该让当前 turn 标记为 incomplete/escalated。
    await this.store.enqueueMany(jobs);

    return {
      ...result,
      obligationJobs: jobs.map((j) => j.id),
    };
  }
}
```

worker 则独立跑：

```ts
export class ObligationWorker {
  constructor(
    private store: ObligationStore,
    private handlers: Record<string, (job: ObligationJob) => Promise<unknown>>,
    private incidentCenter: IncidentCenter,
  ) {}

  async tick(now = new Date()) {
    const jobs = await this.store.claimDue(20, now);

    await Promise.all(jobs.map(async (job) => {
      try {
        const handler = this.handlers[job.obligation.type];
        if (!handler) throw new Error(`missing handler: ${job.obligation.type}`);

        const evidence = await handler(job);
        await this.store.markFulfilled(job.id, evidence);
      } catch (error) {
        const err = error instanceof Error ? error : new Error(String(error));

        if (job.attempts + 1 >= job.maxAttempts) {
          await this.store.markDeadLetter(job.id, err);

          if (job.obligation.required) {
            await this.incidentCenter.openOrAppend({
              severity: job.obligation.risk === "critical" ? "P1" : "P2",
              dedupeKey: `required-obligation-failed:${job.obligation.type}:${job.toolCallId}`,
              title: `Required obligation failed: ${job.obligation.type}`,
              evidence: { jobId: job.id, runId: job.runId, error: err.message },
            });
          }
          return;
        }

        await this.store.markRetry(job.id, err, backoff(job.attempts, job.obligation.risk));
      }
    }));
  }
}
```

这里有个设计重点：**LLM 不应该负责记住稍后要重试**。LLM 可以生成策略和计划，但可靠履行必须交给确定性系统。

---

## 5. OpenClaw 实战：课程 cron 的 obligation queue 怎么落地？

以这门课的 cron 为例，主动作有三个外部副作用：

1. 发 Telegram 群消息；
2. 写 lesson / README / TOOLS；
3. git commit + push。

对应 obligation 可以这样设计：

```json
[
  {
    "type": "record_evidence",
    "required": true,
    "params": {
      "files": [
        "lessons/313-policy-obligation-queue-reliable-fulfillment.md",
        "README.md",
        "../TOOLS.md"
      ]
    }
  },
  {
    "type": "post_check",
    "required": true,
    "params": {
      "check": "git_remote_contains_commit",
      "remote": "origin/main"
    }
  },
  {
    "type": "post_check",
    "required": true,
    "params": {
      "check": "telegram_message_id_recorded",
      "target": "-5115329245"
    }
  },
  {
    "type": "schedule_review",
    "required": false,
    "params": {
      "after": "24h",
      "reason": "verify lesson did not duplicate previous topic"
    }
  }
]
```

OpenClaw 当前可以先用文件式队列实现：

```text
.openclaw/workspace/agent-course/.obligations/pending/*.json
.openclaw/workspace/agent-course/.obligations/fulfilled/*.json
.openclaw/workspace/agent-course/.obligations/dead-letter/*.json
```

Heartbeat 或 cron worker 定期扫描 `pending`：

```bash
find .obligations/pending -name '*.json' -mtime +0 -print
```

生产化以后再换成 SQLite / Postgres / Redis Streams。先把语义跑通，比一上来搞复杂基础设施更重要。

---

## 6. required obligation 失败时，主任务算完成吗？

建议规则：

| 情况 | 主任务状态 |
|---|---|
| 主动作失败 | failed |
| 主动作成功，required obligation 入队失败 | incomplete / escalated |
| 主动作成功，required obligation pending | completed_with_pending_obligations |
| required obligation 最终 fulfilled | completed |
| required obligation dead-letter | incident / needs_human_review |
| optional obligation 失败 | completed_with_warning |

这能避免 Agent 说“完成了”，但其实关键后置验证还没做。

对用户回复也应该更精确：

```text
已完成主动作，验证任务已入队，等待远端状态确认。
```

而不是：

```text
搞定。
```

成熟 Agent 的完成态要比人类口头承诺更严谨。

---

## 7. 常见坑

### 坑 1：obligation 入队和主动作之间不是事务

如果工具本身是外部副作用，没法和本地 DB 做真正分布式事务。解决方法：

- 工具执行前先创建 `intent record`；
- 工具成功后立刻写 `tool result evidence`；
- obligation enqueue 失败时，把 run 标记为 `needs_reconciliation`；
- heartbeat/reconciler 根据 evidence 补建 obligation job。

### 坑 2：重试没有幂等键

`notify`、`schedule_review`、`compensating_action` 都可能重复执行。每个 handler 必须有幂等键：

```text
idempotencyKey = obligation.type + toolCallId + paramsHash
```

### 坑 3：dead-letter 只是存起来没人看

Dead-letter 不是垃圾桶，是升级入口。required obligation 进 dead-letter 必须：

- 创建 incident 或 owner notification；
- 附带 job payload 和最后错误；
- 给出补救动作；
- 关闭时必须写 evidence。

### 坑 4：让 LLM 直接处理队列

LLM 可以解释 dead-letter、建议补救，但 claim/retry/mark fulfilled 必须是确定性代码，不能靠模型“记得”。

---

## 8. 一句话总结

上一课的 Policy Obligation 解决的是：**allow 之后必须做什么**。

今天的 Obligation Queue 解决的是：**这些必须做的事，即使当前 turn 结束、API 限流、worker 重启，也不能丢。**

生产级 Agent 的副作用治理，不是“执行成功就算完”，而是：

```text
副作用可追踪 -> 义务可恢复 -> 失败可升级 -> 完成有证据
```

这才是长期自主运行的底层安全感。
