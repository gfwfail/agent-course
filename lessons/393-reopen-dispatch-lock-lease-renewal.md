# 393. Agent 重开调度的去重锁与租约续期（Reopen Dispatch Lock & Lease Renewal）

上一课讲了 **Reopen Case 创建后要按风险调度**：用 priorityScore、deadline、requiredWorker 和 preemptLowerThan，让恢复任务不是随缘排队。

今天继续补一个生产里很容易踩坑的细节：**高优先级任务被 claim 之后，怎么防止多个 Agent 重复修、怎么防止 worker 崩溃后任务永远卡住。**

一句话：**Reopen Dispatch 需要 lockKey 去重、lease 租约、heartbeat 续期和 fence token 防旧 worker 写回。**

---

## 1. 只做优先级队列还不够

很多系统会写到这一步就停：

    ticket.priorityScore 高 -> worker claim -> 执行补偿

但重开任务通常不是普通后台 job，它背后可能对应外部副作用、补偿账本、下游释放闸门和人工升级。如果 claim 语义不严，会出现几个坏结果：

- 两个 worker 同时处理同一个 effectId，各发一次补偿消息；
- worker A 卡住，ticket 状态一直 claimed，永远没人接手；
- worker A 超时后 worker B 接手，但 A 又恢复了，把旧结果写回；
- heartbeat 续期没有证据，调度器不知道 worker 是活着还是只是在占坑；
- lockKey 太粗，阻塞无关任务；lockKey 太细，同一风险重复执行。

所以 Reopen Dispatch 的 claim 不是简单的“拿一条队列记录”，而是一组并发控制协议。

---

## 2. Dispatch Lock 和 Lease 的核心字段

可以把它拆成两张概念表：一张 ticket，一张 lock。

~~~ts
type ReopenDispatchTicket = {
  ticketId: string;
  caseId: string;
  route: "reconcile" | "compensation" | "incident" | "manual_review";
  priorityScore: number;
  deadlineAt: string;
  requiredWorker: string;
  lockKey: string;
  status: "queued" | "claimed" | "completed" | "released" | "expired";
  claimedBy?: string;
  leaseId?: string;
  fenceToken?: number;
  leaseExpiresAt?: string;
};

type DispatchLock = {
  lockKey: string;
  ticketId: string;
  holder: string;
  leaseId: string;
  fenceToken: number;
  expiresAt: string;
  heartbeatAt: string;
};
~~~

这里最重要的是四个字段：

| 字段 | 作用 |
|---|---|
| lockKey | 对同一个 risk/effect/release 做互斥 |
| leaseId | 标识本次 claim，不同接手轮次不同 |
| fenceToken | 单调递增，防旧 holder 写回 |
| leaseExpiresAt | worker 崩溃后自动释放 |

leaseId 用来识别“这一次租约”，fenceToken 用来识别“谁是最新主人”。两者都要带到后续 complete/release/side-effect receipt 里。

---

## 3. learn-claude-code：纯函数判断租约状态

教学版先不要碰数据库，先把状态机讲清楚。

~~~python
from dataclasses import dataclass
from datetime import datetime, timedelta
from typing import Literal

LeaseDecision = Literal["claimable", "held", "expired_takeover", "completed"]

@dataclass(frozen=True)
class DispatchTicket:
    ticket_id: str
    lock_key: str
    status: str
    claimed_by: str | None
    lease_id: str | None
    fence_token: int
    lease_expires_at: datetime | None

def lease_decision(ticket: DispatchTicket, now: datetime) -> LeaseDecision:
    if ticket.status == "completed":
        return "completed"

    if ticket.status == "queued":
        return "claimable"

    if ticket.status == "claimed" and ticket.lease_expires_at is not None:
        if ticket.lease_expires_at <= now:
            return "expired_takeover"
        return "held"

    return "held"

def renew_lease(ticket: DispatchTicket, worker_id: str, lease_id: str, now: datetime) -> DispatchTicket:
    if ticket.status != "claimed":
        raise ValueError("ticket is not claimed")
    if ticket.claimed_by != worker_id or ticket.lease_id != lease_id:
        raise ValueError("worker does not hold the current lease")

    return DispatchTicket(
        ticket_id=ticket.ticket_id,
        lock_key=ticket.lock_key,
        status=ticket.status,
        claimed_by=ticket.claimed_by,
        lease_id=ticket.lease_id,
        fence_token=ticket.fence_token,
        lease_expires_at=now + timedelta(minutes=10),
    )
~~~

这个函数看起来简单，但它把三个规则固定住：

1. completed 不能被重新 claim；
2. claimed 但没过期，只能原 holder heartbeat；
3. claimed 且过期，可以被新 worker takeover。

教学代码里要先练这种纯函数，因为它决定了生产版 SQL/Redis 实现不会凭感觉写。

---

## 4. pi-mono：claim 时生成 fence token

生产版的 claim 必须在一个事务里完成：选 ticket、抢 lock、写 lease、递增 fence token。

~~~ts
async function claimTicket(db: Db, worker: WorkerIdentity, now: Date) {
  return db.transaction(async (tx) => {
    const ticket = await tx.reopenDispatchTicket.findFirst({
      where: {
        status: { in: ["queued", "claimed"] },
        requiredWorker: { in: worker.capabilities },
        leaseExpiresAt: { lte: now },
      },
      orderBy: [
        { priorityScore: "desc" },
        { deadlineAt: "asc" },
        { createdAt: "asc" },
      ],
      lock: "for update skip locked",
    });

    if (!ticket) return null;

    const nextFenceToken = ticket.fenceToken + 1;
    const leaseId = crypto.randomUUID();
    const leaseExpiresAt = addMinutes(now, 10);

    await tx.dispatchLock.upsert({
      where: { lockKey: ticket.lockKey },
      create: {
        lockKey: ticket.lockKey,
        ticketId: ticket.ticketId,
        holder: worker.id,
        leaseId,
        fenceToken: nextFenceToken,
        expiresAt: leaseExpiresAt,
        heartbeatAt: now,
      },
      update: {
        ticketId: ticket.ticketId,
        holder: worker.id,
        leaseId,
        fenceToken: nextFenceToken,
        expiresAt: leaseExpiresAt,
        heartbeatAt: now,
      },
    });

    return tx.reopenDispatchTicket.update({
      where: { ticketId: ticket.ticketId },
      data: {
        status: "claimed",
        claimedBy: worker.id,
        leaseId,
        fenceToken: nextFenceToken,
        claimedAt: now,
        leaseExpiresAt,
      },
    });
  });
}
~~~

注意一个细节：如果 status 是 queued，leaseExpiresAt 可能是 null，真实查询条件通常要写成：

~~~ts
OR: [
  { status: "queued" },
  { status: "claimed", leaseExpiresAt: { lte: now } },
]
~~~

这里为了突出主线简化了。核心是：**新 holder 一定拿到更大的 fenceToken。**

---

## 5. complete 必须校验 leaseId + fenceToken

fence token 的价值在 complete 阶段体现。

~~~ts
async function completeTicket(
  db: Db,
  input: {
    ticketId: string;
    workerId: string;
    leaseId: string;
    fenceToken: number;
    outcome: "reconciled" | "compensated" | "escalated" | "manual_hold";
    evidenceRefs: string[];
  },
  now: Date,
) {
  return db.transaction(async (tx) => {
    const ticket = await tx.reopenDispatchTicket.findUnique({
      where: { ticketId: input.ticketId },
      lock: "for update",
    });

    if (!ticket) throw new Error("ticket not found");

    const stillCurrent =
      ticket.status === "claimed" &&
      ticket.claimedBy === input.workerId &&
      ticket.leaseId === input.leaseId &&
      ticket.fenceToken === input.fenceToken &&
      ticket.leaseExpiresAt > now;

    if (!stillCurrent) {
      await tx.dispatchCompletionReject.create({
        data: {
          ticketId: input.ticketId,
          workerId: input.workerId,
          leaseId: input.leaseId,
          fenceToken: input.fenceToken,
          reason: "stale_or_expired_lease",
          observedAt: now,
        },
      });
      return { accepted: false as const, reason: "stale_or_expired_lease" };
    }

    await tx.reopenDispatchTicket.update({
      where: { ticketId: input.ticketId },
      data: {
        status: "completed",
        completedAt: now,
        completionOutcome: input.outcome,
        completionEvidenceRefs: input.evidenceRefs,
      },
    });

    await tx.dispatchLock.delete({ where: { lockKey: ticket.lockKey } });

    return { accepted: true as const };
  });
}
~~~

这能挡住一个经典事故：

1. worker A claim ticket，fenceToken=7；
2. A 卡住，lease 过期；
3. worker B takeover，fenceToken=8；
4. A 恢复后提交 complete；
5. 系统拒绝 A，因为它的 token 已经旧了。

没有 fence token，旧 worker 很容易把新 worker 的修复结果覆盖掉。

---

## 6. heartbeat 不是“我还活着”这么简单

heartbeat 也要可审计。它至少要记录 worker 状态、当前阶段、是否进入不可中断副作用。

~~~ts
type DispatchHeartbeat = {
  ticketId: string;
  workerId: string;
  leaseId: string;
  fenceToken: number;
  phase: "planning" | "reconciling" | "external_effect" | "verifying" | "closing";
  progress: number;
  evidenceRefs: string[];
  observedAt: string;
};

async function heartbeatDispatchLease(db: Db, hb: DispatchHeartbeat, now: Date) {
  const leaseExpiresAt = addMinutes(now, 10);

  return db.reopenDispatchTicket.updateMany({
    where: {
      ticketId: hb.ticketId,
      claimedBy: hb.workerId,
      leaseId: hb.leaseId,
      fenceToken: hb.fenceToken,
      status: "claimed",
      leaseExpiresAt: { gt: now },
    },
    data: {
      leaseExpiresAt,
      heartbeatAt: now,
      heartbeatPhase: hb.phase,
      heartbeatProgress: hb.progress,
    },
  });
}
~~~

如果 heartbeat 处在 external_effect 阶段，调度器不要贸然 takeover。更稳的做法是先进入 reconcile_before_takeover：只读检查外部世界到底发生了什么，再决定是否接手。

---

## 7. OpenClaw 课程 Cron 实战

这个课程 cron 本身就是小型的 Reopen/Dispatch 场景：

1. 先看 TOOLS.md 已讲内容，避免重复主题；
2. 写 lesson 文件、README、TOOLS 三个本地状态；
3. 发 Telegram，这是外部副作用；
4. git commit/push，这是另一个外部副作用；
5. 最后写 memory，形成完成证据。

如果这个流程中间断掉，比如“Telegram 发了，但 git push 没成功”，下一轮不能重新发同一课。它应该先根据本地 lesson 文件、README、TOOLS、Telegram messageId、git remote commit 做对账，再决定：

- 本地文件存在但未 commit：继续 commit/push；
- Telegram 已发但 git 未推：不要重发 Telegram，只补 git；
- git 已推但 TOOLS 未更新：补 TOOLS 并提交修正；
- 多个 cron 同时启动：用课程号或主题作为 lockKey，只有一个实例能发布。

可以把它建模成：

~~~ts
type CourseDispatchLock = {
  lockKey: "agent-course:lesson:393";
  holder: string;
  leaseId: string;
  fenceToken: number;
  phase: "drafting" | "telegram_sent" | "git_pushed" | "memory_written";
  evidenceRefs: string[];
  expiresAt: string;
};
~~~

成熟 Agent 的后台任务不是“跑完就算”。它要能回答：谁拿了任务、租约是否仍有效、外部副作用执行到哪一步、如果我现在接手会不会重复发。

---

## 8. 实战检查清单

设计 Reopen Dispatch Lock 时，至少问八个问题：

- lockKey 是按 caseId、riskId、effectId 还是 releaseId 生成？
- 同一 lockKey 下是否允许 manual_review 和 reconcile 并行？
- lease 过期后 takeover 前，是否需要只读对账？
- heartbeat 是否记录 phase，而不只是更新时间？
- complete 是否同时校验 workerId、leaseId、fenceToken？
- 旧 worker 的 complete 被拒绝后，是否留下 reject receipt？
- 外部副作用阶段是否禁止硬抢占？
- lock 释放是否只发生在 completed、released 或 expired repair 后？

这套机制的目标不是让队列更复杂，而是让恢复任务在并发、崩溃、超时、重试下仍然只有一个可信执行者。

---

## 9. 小结

今天这课的重点：

- priority queue 只解决“先处理谁”，不解决“谁有权处理”；
- lockKey 防重复，lease 防永久卡住，heartbeat 证明 worker 仍在推进；
- fenceToken 防旧 worker 在租约过期后写回旧结果；
- complete/release 必须校验 workerId + leaseId + fenceToken；
- 接手过期任务前，先看 heartbeat phase 和外部现实，避免重复副作用。

一句话收尾：**Reopen Dispatch 的可靠性，不在于抢任务有多快，而在于任何时刻都能证明谁是当前合法 holder。**

