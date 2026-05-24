# 395. Agent 人工升级的决策 SLA 与负责人路由（Escalation Decision SLA & Owner Routing）

上一课讲了 **Reopen Dispatch 的 Dead Letter Queue**：自动恢复修不动时，要把 ticket 变成结构化 Escalation Case，并冻结相关下游。

今天继续补闭环：**Escalation Case 创建出来以后，不能只是发一条“请人工看看”。**

一句话：**人工升级也要有 owner、SLA、ack、decision deadline 和逾期再升级规则，否则 DLQ 只是把自动系统的卡死换成人工队列的卡死。**

---

## 1. 人工升级不是“扔给人类”

很多 Agent 系统一遇到高风险错误，就写一句：

> manual review required

这句话在工程上几乎没用。因为它没有回答：

- 谁负责看？
- 多久内必须 ack？
- 多久内必须给 decision？
- 需要什么证据才能决策？
- 逾期以后升级给谁？
- 人类决策回来以后，哪个 worker 可以继续执行？

恢复系统里最危险的状态不是 failed，而是 **open but unowned**。

所以 Escalation Case 至少要变成一个可调度、可追踪、可超时处理的对象。

---

## 2. 把 Escalation Case 拆成四个时间点

建议不要只有一个 deadline，而是拆成四个阶段：

1. **createdAt**：系统发现自动恢复边界，生成升级案件；
2. **ackBy**：负责人必须确认“我接手了”；
3. **decisionDueAt**：负责人必须给出结构化决策；
4. **expiresAt**：决策过期，必须重新验证证据或重新升级。

~~~ts
type EscalationCase = {
  escalationId: string;
  dlqId: string;
  severity: "low" | "medium" | "high" | "critical";
  ownerRole: "operator" | "senior_engineer" | "incident_lead" | "security_owner";
  ownerId?: string;
  requiredDecision:
    | "retry_with_override"
    | "forward_fix"
    | "accept_residual_risk"
    | "close_as_duplicate"
    | "open_incident";
  evidenceRefs: string[];
  ackBy: string;
  decisionDueAt: string;
  expiresAt: string;
  status: "open" | "acknowledged" | "decided" | "expired" | "escalated";
};

type EscalationDecision = {
  decisionId: string;
  escalationId: string;
  decidedBy: string;
  decision: EscalationCase["requiredDecision"];
  rationale: string;
  allowedActions: string[];
  deniedActions: string[];
  evidenceRefs: string[];
  validUntil: string;
};
~~~

重点：人类不是自由文本回复“可以”。人类输出的是一个 **Decision Record**，后续 worker 只能按 allowedActions 继续。

---

## 3. learn-claude-code：先用纯函数做 owner routing

教学版可以先写一个非常小的路由函数。它不碰数据库，不发消息，只证明规则。

~~~python
from dataclasses import dataclass
from datetime import datetime, timedelta, timezone
from typing import Literal

Severity = Literal["low", "medium", "high", "critical"]
Reason = Literal[
    "max_attempts_exceeded",
    "lease_timeout_budget_exceeded",
    "unsafe_error",
    "missing_required_evidence",
    "manual_review_required",
]
OwnerRole = Literal["operator", "senior_engineer", "incident_lead", "security_owner"]

@dataclass(frozen=True)
class DeadLetterRecord:
    dlq_id: str
    reason: Reason
    severity: Severity
    evidence_refs: list[str]

@dataclass(frozen=True)
class EscalationRoute:
    owner_role: OwnerRole
    ack_by: datetime
    decision_due_at: datetime
    required_decision: str

def route_escalation(dlq: DeadLetterRecord, now: datetime) -> EscalationRoute:
    if dlq.reason == "unsafe_error":
        return EscalationRoute(
            owner_role="incident_lead" if dlq.severity == "critical" else "senior_engineer",
            ack_by=now + timedelta(minutes=10),
            decision_due_at=now + timedelta(minutes=30),
            required_decision="open_incident",
        )

    if dlq.reason == "missing_required_evidence":
        return EscalationRoute(
            owner_role="operator",
            ack_by=now + timedelta(minutes=30),
            decision_due_at=now + timedelta(hours=2),
            required_decision="forward_fix",
        )

    if dlq.reason == "manual_review_required":
        return EscalationRoute(
            owner_role="senior_engineer",
            ack_by=now + timedelta(minutes=15),
            decision_due_at=now + timedelta(hours=1),
            required_decision="retry_with_override",
        )

    return EscalationRoute(
        owner_role="operator",
        ack_by=now + timedelta(hours=1),
        decision_due_at=now + timedelta(hours=4),
        required_decision="retry_with_override",
    )
~~~

这段代码的价值是把“谁来处理、多久处理、处理什么”写死在可测试逻辑里。

---

## 4. pi-mono：升级创建和通知必须同事务出 outbox

生产版不要在事务里直接发 Telegram、Slack 或 Email。正确做法是：

1. 创建 Escalation Case；
2. 创建 Assignment；
3. 创建 Notification Outbox；
4. 提交事务；
5. outbox worker 再外发。

~~~ts
async function createEscalationFromDlq(
  db: Db,
  input: { dlqId: string; now: Date },
) {
  return db.transaction(async (tx) => {
    const dlq = await tx.deadLetterRecord.findUniqueOrThrow({
      where: { dlqId: input.dlqId },
      lock: "for update",
    });

    if (dlq.escalationId) {
      return tx.escalationCase.findUniqueOrThrow({
        where: { escalationId: dlq.escalationId },
      });
    }

    const route = routeEscalation({
      reason: dlq.reason,
      severity: dlq.severity,
      now: input.now,
    });

    const escalation = await tx.escalationCase.create({
      data: {
        dlqId: dlq.dlqId,
        caseId: dlq.caseId,
        severity: dlq.severity,
        ownerRole: route.ownerRole,
        requiredDecision: route.requiredDecision,
        evidenceRefs: dlq.evidenceRefs,
        ackBy: route.ackBy,
        decisionDueAt: route.decisionDueAt,
        expiresAt: route.expiresAt,
        status: "open",
      },
    });

    await tx.deadLetterRecord.update({
      where: { dlqId: dlq.dlqId },
      data: { escalationId: escalation.escalationId },
    });

    await tx.notificationOutbox.create({
      data: {
        kind: "escalation_assignment",
        targetRole: route.ownerRole,
        dedupeKey: `escalation:${escalation.escalationId}:assignment`,
        payload: {
          escalationId: escalation.escalationId,
          severity: escalation.severity,
          requiredDecision: escalation.requiredDecision,
          ackBy: escalation.ackBy,
          decisionDueAt: escalation.decisionDueAt,
        },
      },
    });

    return escalation;
  });
}
~~~

为什么要 outbox？因为“案件创建成功但通知失败”和“通知发出但案件回滚”都会让恢复链断掉。Outbox 至少能保证通知是可重放、可去重、可审计的。

---

## 5. 决策回来以后，不能直接恢复执行

人类 ack 只代表接手，不代表允许继续。

真正能解冻下游的是 **EscalationDecision**，而且必须过三个闸门：

- decision 没过期；
- evidenceRefs 覆盖 required evidence；
- allowedActions 覆盖即将执行的动作。

~~~ts
function authorizeAfterEscalation(input: {
  decision: EscalationDecision;
  action: string;
  now: Date;
  requiredEvidenceRefs: string[];
}): { ok: true } | { ok: false; reason: string } {
  if (new Date(input.decision.validUntil) <= input.now) {
    return { ok: false, reason: "decision_expired" };
  }

  for (const ref of input.requiredEvidenceRefs) {
    if (!input.decision.evidenceRefs.includes(ref)) {
      return { ok: false, reason: "decision_missing_required_evidence" };
    }
  }

  if (!input.decision.allowedActions.includes(input.action)) {
    return { ok: false, reason: "action_not_allowed_by_decision" };
  }

  return { ok: true };
}
~~~

这能防止一个常见事故：人类批准的是“补发课程消息”，worker 却拿这条批准去“强推 git main”或“删除远端消息”。

---

## 6. OpenClaw 里的实战落点

OpenClaw cron 发课就是一个小型外部副作用链：

- 写 lesson 文件；
- 更新 README；
- 更新 TOOLS；
- 发 Telegram；
- git commit；
- git push；
- 写 memory。

如果其中某一步进入 DLQ，比如 Telegram 发了但 git push 失败，而且多次 retry 仍失败，就不应该一直重跑整条 cron。

更稳的做法是：

1. DeadLetterRecord 记录失败点、messageId、commit hash、dirty files；
2. Escalation Case 路由给 operator，要求决策：retry push、forward-fix README、accept residual risk；
3. 通知 outbox 发给对应负责人或主会话；
4. 人类决策回来后，worker 只执行 allowedActions；
5. Closeout Receipt 记录最终 messageId、commit、远端 main hash 和 memory 更新。

这样“需要人工”就不是系统放弃，而是恢复工作流进入了一个有 owner、有 SLA、有证据、有收据的阶段。

---

## 7. 设计检查清单

做 Escalation SLA 时，至少检查这 8 个问题：

- open case 是否必有 ownerRole？
- critical 是否有更短 ackBy？
- ack 超时是否自动 reroute？
- decisionDueAt 超时是否升级 owner？
- decision 是否结构化，而不是自由文本？
- decision 是否有 validUntil？
- worker 恢复执行前是否校验 allowedActions？
- 下游 gate 是否等 Decision Record，而不是等“有人看过”？

---

## 8. 小结

DLQ 解决的是：**自动恢复什么时候该停。**

Escalation SLA 解决的是：**停下来以后谁负责、多久负责、怎么恢复。**

成熟 Agent 不把人工升级当成兜底口号，而是把它做成一条可调度、可超时、可追责、可继续执行的恢复路径。
