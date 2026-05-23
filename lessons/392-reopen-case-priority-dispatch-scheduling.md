# 392. Agent 重开案件优先级与处置调度（Reopen Case Priority & Dispatch Scheduling）

上一课讲了 **Monitoring Lease breach 后要生成 Reopen Case**：别只发告警，要分类、去重、冻结下游，并路由到 reconcile、compensation、incident 或 manual_review。

今天继续往下走：**Reopen Case 创建出来以后，不能谁先来谁先处理。**

一句话：**重开案件要按风险、时效、扩散面和可逆性调度，必要时抢占低风险后台任务。**

---

## 1. 为什么 Reopen Case 需要专门调度

很多 Agent 系统一开始会把 reopen 当普通任务塞回队列：

    case.created -> queue.push(case) -> worker.pick()

这在低流量系统里能跑，但生产里会出几个问题：

- 低风险 lease 到期检查，可能排在真实事故前面；
- 需要 10 分钟内补偿的任务，被一个长跑报告任务占住 worker；
- 多个 reopen case 指向同一个外部副作用，重复抢修；
- manual review、reconcile、compensation 混在一起，没有不同 SLA；
- 高风险 case 处理时，下游依赖还在继续执行。

重开案件不是“又来一个 task”。它是系统承认：**之前关闭的恢复链路又出现了新的证据缺口或风险信号。**

所以调度器至少要看四个维度：

| 维度 | 含义 | 例子 |
|---|---|---|
| severity | 影响有多大 | 是否涉及外部副作用、资金、权限、公开发布 |
| urgency | 多快必须处理 | lease 还有多久失效，SLA 剩多少 |
| blastRadius | 扩散面 | 影响多少 tenant/channel/downstream action |
| reversibility | 是否可逆 | 可删除消息 vs 已执行支付/部署 |

---

## 2. Reopen Dispatch Ticket

第 391 课的 Reopen Case 偏“事实建模”，本课加一个调度视图：`ReopenDispatchTicket`。

```ts
type ReopenRoute = "reconcile" | "compensation" | "incident" | "manual_review";
type DispatchClass = "immediate" | "expedited" | "normal" | "defer";

type ReopenDispatchTicket = {
  ticketId: string;
  caseId: string;
  route: ReopenRoute;
  dispatchClass: DispatchClass;
  priorityScore: number;
  deadlineAt: string;
  requiredWorker: string;
  lockKey: string;
  preemptLowerThan: number;
  blockedDownstreamRefs: string[];
  evidenceRefs: string[];
};
```

这里有两个字段很关键：

- `lockKey`：同一个 riskId / effectId / releaseId 只能有一个 worker 主修，避免重复补偿。
- `preemptLowerThan`：高风险重开可以抢占低优先级后台任务，但不能抢占正在执行的不可中断副作用。

---

## 3. learn-claude-code：纯函数优先级计算器

教学版先做成纯函数，不碰队列、不碰 worker。

```python
from dataclasses import dataclass
from datetime import datetime

@dataclass(frozen=True)
class ReopenCase:
    case_id: str
    route: str
    severity: str          # low | medium | high | critical
    breach_kind: str       # missing_evidence | external_drift | downstream_leak
    deadline_at: datetime
    affected_tenants: int
    affected_downstream: int
    reversible: bool
    evidence_refs: list[str]

SEVERITY_WEIGHT = {"low": 10, "medium": 30, "high": 70, "critical": 100}
BREACH_WEIGHT = {"missing_evidence": 10, "external_drift": 35, "downstream_leak": 45}
ROUTE_WORKER = {
    "reconcile": "reconciler",
    "compensation": "compensation-runner",
    "incident": "incident-commander",
    "manual_review": "review-queue",
}

def urgency_score(deadline_at: datetime, now: datetime) -> int:
    remaining_seconds = max(0, int((deadline_at - now).total_seconds()))
    if remaining_seconds <= 15 * 60:
        return 50
    if remaining_seconds <= 60 * 60:
        return 30
    if remaining_seconds <= 6 * 60 * 60:
        return 15
    return 0

def blast_radius_score(case: ReopenCase) -> int:
    return min(40, case.affected_tenants * 4 + case.affected_downstream * 2)

def dispatch_class(score: int) -> str:
    if score >= 140:
        return "immediate"
    if score >= 100:
        return "expedited"
    if score >= 50:
        return "normal"
    return "defer"

def build_dispatch_ticket(case: ReopenCase, now: datetime) -> dict:
    reversibility_penalty = 25 if not case.reversible else 0
    score = (
        SEVERITY_WEIGHT[case.severity]
        + BREACH_WEIGHT[case.breach_kind]
        + urgency_score(case.deadline_at, now)
        + blast_radius_score(case)
        + reversibility_penalty
    )
    return {
        "ticketId": f"dispatch:{case.case_id}",
        "caseId": case.case_id,
        "route": case.route,
        "dispatchClass": dispatch_class(score),
        "priorityScore": score,
        "deadlineAt": case.deadline_at.isoformat(),
        "requiredWorker": ROUTE_WORKER[case.route],
        "lockKey": f"reopen:{case.case_id}",
        "preemptLowerThan": score - 20,
        "evidenceRefs": case.evidence_refs,
    }
```

重点不是这个分数多复杂，而是让调度决策变成可测试、可解释、可审计的结构：为什么抢占？因为 high severity + downstream_leak + 不可逆 + deadline 接近。

---

## 4. pi-mono：生产版调度队列

生产版不要只靠内存优先级队列。Reopen Case 往往跨 turn、跨 worker、跨进程，需要持久化。

```ts
interface ReopenDispatchQueue {
  enqueue(ticket: ReopenDispatchTicket): Promise<void>;
  claim(worker: string, now: Date): Promise<ReopenDispatchTicket | null>;
  heartbeat(ticketId: string, workerId: string): Promise<void>;
  complete(ticketId: string, receipt: DispatchCompletionReceipt): Promise<void>;
  release(ticketId: string, reason: string): Promise<void>;
}

type DispatchCompletionReceipt = {
  ticketId: string;
  workerId: string;
  completedAt: string;
  outcome: "reconciled" | "compensated" | "escalated" | "manual_hold";
  evidenceRefs: string[];
  downstreamBarrierReleased: boolean;
};
```

`claim()` 的 SQL/Redis 语义要满足三点：

1. **按 priorityScore + deadline 排序**：高风险、快到期优先。
2. **按 lockKey 去重**：同一风险只能被一个 worker claim。
3. **带 lease**：worker 崩溃后 ticket 自动回队列。

示意实现：

```ts
async function claimReopenTicket(db: Db, worker: WorkerIdentity, now: Date) {
  return db.transaction(async (tx) => {
    const ticket = await tx.reopenDispatchTicket.findFirst({
      where: {
        status: "queued",
        requiredWorker: { in: worker.capabilities },
        notBefore: { lte: now },
      },
      orderBy: [
        { priorityScore: "desc" },
        { deadlineAt: "asc" },
        { createdAt: "asc" },
      ],
      lock: "for update skip locked",
    });

    if (!ticket) return null;

    const lockTaken = await tx.dispatchLock.tryInsert({
      key: ticket.lockKey,
      holder: worker.id,
      expiresAt: addMinutes(now, 10),
    });

    if (!lockTaken) return null;

    return tx.reopenDispatchTicket.update({
      where: { ticketId: ticket.ticketId },
      data: {
        status: "claimed",
        claimedBy: worker.id,
        claimedAt: now,
        leaseExpiresAt: addMinutes(now, 10),
      },
    });
  });
}
```

这里的 `skip locked` 很实用：多个 worker 同时抢任务时，不会互相阻塞，也不会重复拿同一张票。

---

## 5. 抢占不是乱杀任务

Reopen Case 调度器常见误区：一看到 critical 就取消所有正在跑的任务。

正确做法是区分任务可中断性：

```ts
type RunningTask = {
  taskId: string;
  priorityScore: number;
  interruptibility: "safe" | "checkpoint_only" | "unsafe";
  checkpointRef?: string;
  sideEffectPhase: "pre_effect" | "in_effect" | "post_effect";
};

function planPreemption(ticket: ReopenDispatchTicket, running: RunningTask[]) {
  return running
    .filter((task) => task.priorityScore < ticket.preemptLowerThan)
    .map((task) => {
      if (task.interruptibility === "safe" && task.sideEffectPhase === "pre_effect") {
        return { taskId: task.taskId, action: "cancel_now" };
      }
      if (task.interruptibility === "checkpoint_only") {
        return { taskId: task.taskId, action: "request_checkpoint_then_pause" };
      }
      return { taskId: task.taskId, action: "do_not_preempt" };
    });
}
```

三条红线：

- `in_effect` 阶段不要硬中断，容易制造半完成副作用；
- 没有 checkpoint 的长任务只能降并发或等安全点；
- 抢占动作本身也要留下 receipt，不能变成隐形副作用。

---

## 6. OpenClaw 课程 Cron 实战

拿课程 Cron 举例，上一课说过可能出现“Telegram 发了，但 README/TOOLS/Git 只有一部分完成”的 partial publish。

这类 breach 会生成 Reopen Case：

```json
{
  "caseId": "reopen:agent-course:lesson-392",
  "route": "compensation",
  "severity": "high",
  "breachKind": "downstream_leak",
  "affectedDownstream": 4,
  "deadlineAt": "2026-05-24T00:00:00Z",
  "evidenceRefs": [
    "telegram:messageId",
    "git:commit",
    "file:README.md",
    "file:TOOLS.md"
  ]
}
```

调度器应该做的是：

1. 生成 `ReopenDispatchTicket`，priority 高于普通 heartbeat 维护任务；
2. 抢占或延后低风险清理任务；
3. 加 `lockKey = agent-course:lesson-392`，避免两个 cron 同时补同一课；
4. 派给 compensation worker；
5. 补齐 README、TOOLS、memory、Git push；
6. 生成 `DispatchCompletionReceipt`，再释放下游 closeout gate。

成熟点在于：系统不是“发现异常就喊人”，而是知道 **谁应该先处理、谁能处理、怎么避免重复处理、什么时候可以恢复下游**。

---

## 7. 检查清单

设计 Reopen Case 调度时，至少检查：

- [ ] priorityScore 是否可解释，而不是魔法数字；
- [ ] deadline 是否参与排序；
- [ ] 同一 risk/effect/release 是否有 lockKey 去重；
- [ ] worker claim 是否有 lease 和 heartbeat；
- [ ] worker 崩溃后 ticket 是否能回队列；
- [ ] 抢占是否尊重 sideEffectPhase；
- [ ] 完成后是否写 DispatchCompletionReceipt；
- [ ] 下游屏障是否只在 receipt 验证后释放。

---

## 8. 结尾

Reopen Case 是恢复闭环重新打开的入口，而 Dispatch Scheduler 决定恢复资源先救哪里。

一句话收尾：**重开不是重新排队等运气，而是按风险和证据调度一条可抢占、可去重、可关闭的恢复路径。**
