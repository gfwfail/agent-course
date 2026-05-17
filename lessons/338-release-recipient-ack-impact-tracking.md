# 338. Agent 发布接收者回执与影响追踪（Release Recipient Acknowledgement & Impact Tracking）

上一课讲了 **Correction / Retraction Notice**：Agent 发现外部发布内容不再可信时，要对外发更正、撤回或删除通知。

今天继续补上更现实的一环：**通知发出去，不等于影响已经收口**。

在 Telegram 群、GitHub issue、Slack channel、Email thread 里，Agent 发了更正通知之后，还要知道：

- 哪些关键接收者需要看到？
- 哪些人已经确认？
- 哪些下游动作可能已经基于旧消息发生？
- 哪些渠道没有 read receipt，只能用替代信号？
- 超时后应该提醒、升级，还是开 incident？

核心思想：

> Release Ledger 证明“我发过”；Correction Notice 证明“我更正过”；Recipient Acknowledgement 证明“关键接收者已被覆盖”。成熟 Agent 的外部发布闭环，不止到 messageId，而是到 audience impact。

---

## 1. 为什么 messageId 不够？

很多 Agent 系统把外发成功当成终点：

```text
message.send -> { messageId: 12168 } -> done
```

这在普通通知里可以接受，但在高风险发布里不够：

- 部署指令发错了，只有 messageId，不能证明值班的人看到了撤回。
- 群里有人已经按旧结论操作，Agent 不知道影响范围。
- GitHub 评论更正了，但 reviewer 可能已经基于旧评论 approve。
- Email 无法确认已读，只能通过 reply、后续 action、人工 ack 兜底。
- 状态页更正了，但客户支持团队还在引用旧摘要。

所以发布后的闭环应该是：

```text
ReleaseRecord
  -> Notice / Update
  -> Audience Plan
  -> Ack Collection
  -> Impact Scan
  -> Escalation / Closeout
  -> Release Ledger append
```

关键点：不是每个读者都必须 ack，而是 **关键角色** 必须覆盖，比如 owner、on-call、approver、affected user、channel moderator。

---

## 2. 数据结构：AudienceAckPlan

先把“谁需要确认”变成结构化计划。

```json
{
  "ackPlanId": "ack_01H...",
  "releaseId": "rel_337",
  "noticeId": "notice_01H...",
  "externalRef": {
    "provider": "telegram",
    "target": "-5115329245",
    "messageId": "12168"
  },
  "requiredRecipients": [
    {
      "recipientId": "oncall:payments",
      "role": "on_call",
      "ackMode": "explicit",
      "deadlineAt": "2026-05-17T03:00:00.000Z"
    },
    {
      "recipientId": "owner:agent-course",
      "role": "owner",
      "ackMode": "reaction_or_reply",
      "deadlineAt": "2026-05-17T04:00:00.000Z"
    }
  ],
  "impactQuestions": [
    "是否有人已经根据旧消息执行了部署？",
    "是否有下游 ticket / PR / runbook 引用了旧结论？"
  ],
  "escalationPolicy": {
    "afterDeadline": "send_reminder",
    "afterReminder": "open_incident"
  }
}
```

几个字段很重要：

- `ackMode`：显式回复、reaction、按钮点击、GitHub review、Email reply 都是不同模式。
- `deadlineAt`：高风险撤回不能无限等。
- `impactQuestions`：Agent 要追踪的不只是“看没看到”，还有“有没有人已经行动”。
- `escalationPolicy`：无人确认时不能安静失败。

---

## 3. learn-claude-code：文件式 Ack Tracker

教学版用 JSONL 做一个最小可运行的回执追踪器：登记计划、记录 ack、扫描超时。

```python
# learn-claude-code/release_ack_tracker.py
from dataclasses import dataclass, asdict
from datetime import datetime, timezone, timedelta
from pathlib import Path
from typing import Literal
import json

AckMode = Literal["explicit", "reaction_or_reply", "implicit_action"]
AckStatus = Literal["pending", "acknowledged", "overdue"]

@dataclass
class RequiredRecipient:
    recipient_id: str
    role: str
    ack_mode: AckMode
    deadline_at: str

@dataclass
class AckPlan:
    ack_plan_id: str
    release_id: str
    notice_id: str
    external_provider: str
    external_target: str
    external_message_id: str
    required_recipients: list[RequiredRecipient]

@dataclass
class AckEvent:
    ack_plan_id: str
    recipient_id: str
    signal: str
    source_message_id: str | None
    observed_at: str

class AckTracker:
    def __init__(self, root: Path):
        self.root = root
        self.root.mkdir(parents=True, exist_ok=True)
        self.plans_path = self.root / "ack_plans.jsonl"
        self.events_path = self.root / "ack_events.jsonl"

    def append_jsonl(self, path: Path, item: dict) -> None:
        with path.open("a", encoding="utf-8") as f:
            f.write(json.dumps(item, ensure_ascii=False, sort_keys=True) + "\n")

    def create_plan(self, plan: AckPlan) -> None:
        self.append_jsonl(self.plans_path, asdict(plan))

    def record_ack(self, ack_plan_id: str, recipient_id: str, signal: str, source_message_id: str | None = None) -> None:
        self.append_jsonl(self.events_path, asdict(AckEvent(
            ack_plan_id=ack_plan_id,
            recipient_id=recipient_id,
            signal=signal,
            source_message_id=source_message_id,
            observed_at=datetime.now(timezone.utc).isoformat(),
        )))

    def load_jsonl(self, path: Path) -> list[dict]:
        if not path.exists():
            return []
        return [json.loads(line) for line in path.read_text("utf-8").splitlines() if line.strip()]

    def status(self, ack_plan_id: str, now: datetime | None = None) -> dict:
        now = now or datetime.now(timezone.utc)
        plan = [p for p in self.load_jsonl(self.plans_path) if p["ack_plan_id"] == ack_plan_id][-1]
        events = [e for e in self.load_jsonl(self.events_path) if e["ack_plan_id"] == ack_plan_id]
        acknowledged = {e["recipient_id"] for e in events}

        rows = []
        for recipient in plan["required_recipients"]:
            deadline = datetime.fromisoformat(recipient["deadline_at"])
            if recipient["recipient_id"] in acknowledged:
                state: AckStatus = "acknowledged"
            elif now > deadline:
                state = "overdue"
            else:
                state = "pending"
            rows.append({**recipient, "status": state})

        return {
            "ack_plan_id": ack_plan_id,
            "release_id": plan["release_id"],
            "notice_id": plan["notice_id"],
            "complete": all(r["status"] == "acknowledged" for r in rows),
            "recipients": rows,
        }

if __name__ == "__main__":
    tracker = AckTracker(Path(".agent-release-acks"))
    deadline = (datetime.now(timezone.utc) + timedelta(minutes=30)).isoformat()
    tracker.create_plan(AckPlan(
        ack_plan_id="ack_338",
        release_id="rel_337",
        notice_id="notice_337",
        external_provider="telegram",
        external_target="-5115329245",
        external_message_id="12168",
        required_recipients=[
            RequiredRecipient("oncall:agent-course", "on_call", "explicit", deadline),
            RequiredRecipient("owner:gfwfail", "owner", "reaction_or_reply", deadline),
        ],
    ))
    tracker.record_ack("ack_338", "owner:gfwfail", "telegram_reaction", "12169")
    print(json.dumps(tracker.status("ack_338"), ensure_ascii=False, indent=2))
```

这个版本刻意简单，但已经有三个生产必需属性：

1. **追加写**：ack event 不覆盖旧事件，方便审计。
2. **按 recipient 计算状态**：不是“有一个人回复了”就算完成。
3. **超时可扫描**：Cron 可以定期找 overdue，然后提醒或升级。

---

## 4. pi-mono：ReleaseAckMiddleware

生产版要放在发布账本和消息通道之间。它不负责生成消息正文，而是负责“高风险发布后必须有人确认”。

```ts
// pi-mono/packages/agent-security/src/release-ack.ts
export type AckMode = "explicit" | "reaction_or_reply" | "implicit_action";
export type AckState = "pending" | "acknowledged" | "overdue";

export interface ExternalRef {
  provider: "telegram" | "github" | "slack" | "email";
  target: string;
  messageId: string;
  url?: string;
}

export interface RequiredRecipient {
  recipientId: string;
  role: "owner" | "on_call" | "approver" | "affected_user" | "moderator";
  ackMode: AckMode;
  deadlineAt: string;
}

export interface AckPlan {
  ackPlanId: string;
  releaseId: string;
  noticeId?: string;
  externalRef: ExternalRef;
  requiredRecipients: RequiredRecipient[];
  impactQuestions: string[];
  escalationPolicy: {
    afterDeadline: "send_reminder" | "open_incident" | "manual_review";
    afterReminder: "open_incident" | "manual_review";
  };
}

export interface AckSignal {
  ackPlanId: string;
  recipientId: string;
  signal: "reply" | "reaction" | "button" | "review" | "followup_action";
  sourceRef: ExternalRef;
  observedAt: string;
}

export interface AckStore {
  savePlan(plan: AckPlan): Promise<void>;
  appendSignal(signal: AckSignal): Promise<void>;
  listSignals(ackPlanId: string): Promise<AckSignal[]>;
}

export class ReleaseAckMiddleware {
  constructor(private readonly store: AckStore) {}

  async createAckPlan(input: {
    releaseId: string;
    noticeId?: string;
    externalRef: ExternalRef;
    severity: "low" | "medium" | "high" | "critical";
    recipients: RequiredRecipient[];
  }): Promise<AckPlan | null> {
    if (input.severity === "low") return null;

    const plan: AckPlan = {
      ackPlanId: `ack:${input.releaseId}:${input.noticeId ?? "release"}`,
      releaseId: input.releaseId,
      noticeId: input.noticeId,
      externalRef: input.externalRef,
      requiredRecipients: input.recipients,
      impactQuestions: [
        "Did anyone act on the previous release?",
        "Does any downstream issue, PR, ticket, or runbook cite the old content?",
      ],
      escalationPolicy:
        input.severity === "critical"
          ? { afterDeadline: "open_incident", afterReminder: "manual_review" }
          : { afterDeadline: "send_reminder", afterReminder: "open_incident" },
    };

    await this.store.savePlan(plan);
    return plan;
  }

  async evaluate(plan: AckPlan, now = new Date()) {
    const signals = await this.store.listSignals(plan.ackPlanId);
    const acked = new Set(signals.map((signal) => signal.recipientId));
    const rows = plan.requiredRecipients.map((recipient) => ({
      ...recipient,
      state: acked.has(recipient.recipientId)
        ? "acknowledged"
        : now > new Date(recipient.deadlineAt)
          ? "overdue"
          : "pending",
    }));

    return {
      complete: rows.every((row) => row.state === "acknowledged"),
      rows,
      overdue: rows.filter((row) => row.state === "overdue"),
    };
  }
}
```

这里有几个设计边界：

- `low` severity 不强制 ack，避免所有通知都变成流程负担。
- `critical` 超时直接 incident，不靠 Agent 自己乐观等待。
- `implicit_action` 可以记录“对方执行了修复命令”这类信号，但要谨慎，不能把任何后续发言都当 ack。
- `ackPlanId` 必须稳定，保证重试不会创建多份计划。

---

## 5. OpenClaw 场景：Telegram / GitHub 发布闭环

以这个课程 cron 为例，普通课程发布不需要每次让群成员确认；但如果课程内容后来发现有错误，流程应该变成：

```text
1. Release Ledger 找到原 Telegram messageId
2. Correction Planner 生成更正通知
3. message.send 发到 Rust学习小组
4. Ack Planner 判断是否需要 owner / maintainer ack
5. Heartbeat / Cron 轮询 reply、reaction、后续修正 commit
6. 超时未确认则私聊 owner 或开一个待办
7. closeout 写回 release ledger
```

Telegram 群通常没有可靠的 per-user read receipt。Agent 不能假装知道“所有人已读”。它只能使用可验证信号：

- 对方回复“收到 / 已处理”
- 对方 reaction
- 对方点击 inline button
- 对方后续 commit / PR / issue 操作引用了更正内容
- owner 在私聊确认

没有信号就保持 `pending`，到期升级，而不是编一个“应该看到了”。

---

## 6. Impact Tracking：确认之外还要查影响

Ack 解决“人是否覆盖”，Impact Tracking 解决“旧内容是否已经造成下游影响”。

常见扫描点：

- GitHub：是否有 PR comment、review、issue 引用了旧 releaseId/messageId。
- Telegram/Slack：旧消息后面是否有人回复“我去执行”。
- CI/CD：旧消息之后是否出现相关 deploy/runbook/job。
- Docs/Runbook：是否有人复制了旧摘要。
- Ticket：是否有新 ticket 按旧结论分派。

最小结构：

```ts
export interface ImpactFinding {
  findingId: string;
  releaseId: string;
  source: "github" | "telegram" | "slack" | "ci" | "docs";
  ref: string;
  kind: "citation" | "action_taken" | "discussion" | "unknown";
  severity: "none" | "low" | "medium" | "high";
  summary: string;
  suggestedAction: "none" | "reply_correction" | "open_task" | "manual_review";
}
```

Impact scanner 的原则：

1. **只读优先**：先搜引用、回复、事件，不要为了确认影响制造新副作用。
2. **按 releaseId/messageId/hash 搜索**：不要只靠自然语言模糊匹配。
3. **高风险人工复核**：发现 action_taken 时，不要让 Agent 自动覆盖人的操作。
4. **写回账本**：impact finding 也是 release closeout 的证据。

---

## 7. 常见坑

### 坑 1：把发送成功当成触达成功

`messageId` 只能证明平台接受了消息，不能证明目标人看到，更不能证明理解。

解决：高风险发布建立 `AckPlan`，关键角色必须有独立信号。

### 坑 2：把群里任意回复当成 ack

有人发“？”、“这个啥”、“晚点看”都不是确认。

解决：按 recipient + ackMode 验证，必要时要求明确短语或按钮。

### 坑 3：没有 deadline

没有截止时间，pending 会永久挂着，事故看起来“还在处理中”。

解决：按 severity 设置 deadline，超时自动提醒或升级。

### 坑 4：忽略无 read receipt 的渠道差异

Email、Telegram、GitHub 的确认能力完全不同。

解决：渠道能力矩阵决定 ackMode，不能把不可观测的信息写成事实。

### 坑 5：只追 ack，不追 impact

所有人都说“收到”，但旧消息已经触发了部署或客户沟通，仍然没收口。

解决：更正/撤回后跑 impact scan，至少查最关键的下游系统。

---

## 8. 设计 Checklist

- [ ] 高风险 release / notice 自动创建 AckPlan
- [ ] AckPlan 绑定 releaseId、noticeId、externalRef
- [ ] requiredRecipients 按角色配置，不要求全员确认
- [ ] ackMode 区分 explicit / reaction / button / implicit_action
- [ ] 每个 required recipient 都有 deadline
- [ ] AckSignal 追加写、可审计、不覆盖
- [ ] 超时有 escalationPolicy
- [ ] 没有 read receipt 的渠道不伪造已读
- [ ] Correction / Retraction 后跑 Impact Scan
- [ ] ImpactFinding 写回 Release Ledger

---

## 9. 一句话总结

发布账本回答“Agent 发了什么”，更正通知回答“Agent 纠正了什么”，接收者回执与影响追踪回答“关键人是否收到、旧内容有没有继续伤害系统”。

成熟 Agent 的外部沟通不是把消息丢出去就结束，而是把触达、确认、影响和升级都做成可审计的闭环。

