# 296. Agent 事故整改行动追踪（Post-Incident Action Tracker & CAPA）

上一课我们讲了事故状态页：事故发生时，Agent 要少而准地通知不同利益相关方。

但事故结束后还有一个更容易被忽略的问题：**复盘写完了，整改项谁跟？什么时候到期？证据在哪里？会不会下次又犯？**

今天讲一个生产级 Agent 必备能力：Post-Incident Action Tracker，也可以叫 CAPA（Corrective and Preventive Actions，纠正与预防措施）。

一句话：

> 事故复盘不是终点；每个 root cause 都要落成可追踪、可验收、可升级的行动项。

---

## 1. 为什么需要整改行动追踪

很多团队事故复盘看起来很认真，但一个月后问题又出现：

- action item 写在文档里，没人负责
- owner 离职/休假后任务没人接
- due date 过了没人提醒
- “已修复”没有证据，只有一句话
- 相同 root cause 重复出现，但没人把旧事故关联起来

Agent 如果只会生成复盘报告，还不算成熟。成熟的 Agent 应该能把事故转成一个持续跟踪的闭环：

```text
Incident resolved
  -> extract root causes
  -> create corrective/preventive actions
  -> assign owner + due date + verification gate
  -> remind/escalate when stale
  -> close only with evidence
  -> feed regression tests / runbooks / policy
```

重点不是“多建几个 TODO”，而是让事故教训真的进入系统。

---

## 2. 最小数据模型：IncidentAction

```json
{
  "actionId": "act_inc_20260511_001_01",
  "incidentId": "inc_20260511_001",
  "kind": "preventive",
  "severity": "P1",
  "owner": "platform-oncall",
  "title": "Add idempotency key to Telegram course delivery",
  "rootCause": "Retry path could resend the same lesson without checking prior messageId.",
  "dueAt": "2026-05-18T00:00:00Z",
  "status": "open",
  "verification": {
    "type": "test",
    "command": "pytest tests/test_course_delivery_idempotency.py",
    "requiredEvidence": ["test_output", "commit_sha"]
  },
  "evidence": [],
  "createdAt": "2026-05-11T17:30:00Z",
  "updatedAt": "2026-05-11T17:30:00Z"
}
```

字段设计要注意：

- `kind`：`corrective` 修当前问题，`preventive` 防止复发
- `owner`：必须是可通知、可升级的主体，不要写“团队”这种虚词
- `dueAt`：没有到期日的整改项等于愿望
- `verification`：关闭标准要前置定义
- `evidence`：关闭必须有证据，如 PR、测试输出、配置快照、监控截图

---

## 3. learn-claude-code：Python 教学版

先做一个文件式 Action Tracker：创建整改项、检测逾期、带证据关闭。

```python
# learn_claude_code/incident_action_tracker.py
from __future__ import annotations

import json
import time
from dataclasses import dataclass, asdict, field
from pathlib import Path
from typing import Literal

ActionKind = Literal["corrective", "preventive"]
ActionStatus = Literal["open", "in_progress", "blocked", "closed"]


@dataclass
class Verification:
    type: str
    command: str | None = None
    required_evidence: list[str] = field(default_factory=list)


@dataclass
class IncidentAction:
    action_id: str
    incident_id: str
    kind: ActionKind
    severity: str
    owner: str
    title: str
    root_cause: str
    due_at: float
    status: ActionStatus
    verification: Verification
    evidence: list[dict]
    created_at: float
    updated_at: float


class ActionTracker:
    def __init__(self, path: str = "state/incident-actions.json"):
        self.path = Path(path)
        self.path.parent.mkdir(parents=True, exist_ok=True)
        if not self.path.exists():
            self.path.write_text("[]")

    def _load(self) -> list[dict]:
        return json.loads(self.path.read_text())

    def _save(self, rows: list[dict]) -> None:
        tmp = self.path.with_suffix(".tmp")
        tmp.write_text(json.dumps(rows, ensure_ascii=False, indent=2))
        tmp.replace(self.path)

    def create_from_review(self, review: dict) -> list[IncidentAction]:
        now = time.time()
        existing = self._load()
        created: list[IncidentAction] = []

        for idx, item in enumerate(review.get("action_items", []), start=1):
            action = IncidentAction(
                action_id=f"act_{review['incident_id']}_{idx:02d}",
                incident_id=review["incident_id"],
                kind=item.get("kind", "preventive"),
                severity=review.get("severity", "P2"),
                owner=item["owner"],
                title=item["title"],
                root_cause=item["root_cause"],
                due_at=item["due_at"],
                status="open",
                verification=Verification(**item["verification"]),
                evidence=[],
                created_at=now,
                updated_at=now,
            )
            existing.append(asdict(action))
            created.append(action)

        self._save(existing)
        return created

    def stale_actions(self, now: float | None = None) -> list[dict]:
        now = now or time.time()
        return [
            a for a in self._load()
            if a["status"] != "closed" and a["due_at"] < now
        ]

    def close_with_evidence(self, action_id: str, evidence: list[dict]) -> dict:
        rows = self._load()
        for action in rows:
            if action["action_id"] != action_id:
                continue

            required = set(action["verification"].get("required_evidence", []))
            provided = {e["type"] for e in evidence}
            missing = required - provided
            if missing:
                raise ValueError(f"missing evidence: {sorted(missing)}")

            action["evidence"].extend(evidence)
            action["status"] = "closed"
            action["updated_at"] = time.time()
            self._save(rows)
            return action

        raise KeyError(action_id)
```

这里有两个关键点：

1. **创建和关闭都写文件**：行动项不是 prompt 里的临时文本，而是系统状态
2. **关闭时校验证据**：没有 `test_output` / `commit_sha` 就不能 close

---

## 4. pi-mono：中间件化整改闭环

生产里不要让业务代码到处手写整改逻辑，可以做一个 `IncidentActionMiddleware`，监听事故复盘事件。

```ts
// pi-mono/src/ops/incident-action-middleware.ts
type ActionStatus = "open" | "in_progress" | "blocked" | "closed";

type IncidentAction = {
  actionId: string;
  incidentId: string;
  kind: "corrective" | "preventive";
  severity: "P0" | "P1" | "P2" | "P3";
  owner: string;
  title: string;
  rootCause: string;
  dueAt: string;
  status: ActionStatus;
  verification: {
    type: "test" | "monitor" | "runbook" | "policy";
    command?: string;
    requiredEvidence: string[];
  };
  evidence: Array<{ type: string; uri: string; observedAt: string }>;
};

export class IncidentActionMiddleware {
  constructor(
    private readonly store: IncidentActionStore,
    private readonly notifier: Notifier,
    private readonly policy: ActionPolicy,
  ) {}

  async onIncidentReview(review: IncidentReview) {
    const actions = review.actionItems.map((item, index): IncidentAction => ({
      actionId: `act_${review.incidentId}_${String(index + 1).padStart(2, "0")}`,
      incidentId: review.incidentId,
      kind: item.kind,
      severity: review.severity,
      owner: item.owner,
      title: item.title,
      rootCause: item.rootCause,
      dueAt: item.dueAt,
      status: "open",
      verification: item.verification,
      evidence: [],
    }));

    await this.store.upsertMany(actions);

    for (const action of actions) {
      await this.notifier.send(action.owner, this.renderAssignment(action));
    }
  }

  async close(actionId: string, evidence: IncidentAction["evidence"]) {
    const action = await this.store.get(actionId);
    const decision = this.policy.canClose(action, evidence);

    if (!decision.allow) {
      throw new Error(`cannot close action: ${decision.reason}`);
    }

    await this.store.update(actionId, {
      status: "closed",
      evidence,
    });
  }

  async sweepStale(now = new Date()) {
    const stale = await this.store.findOpenBefore(now.toISOString());

    for (const action of stale) {
      const escalation = this.policy.escalationTarget(action);
      await this.notifier.send(escalation, this.renderEscalation(action));
    }
  }

  private renderAssignment(action: IncidentAction) {
    return [
      `New incident action: ${action.title}`,
      `Incident: ${action.incidentId}`,
      `Due: ${action.dueAt}`,
      `Close requires: ${action.verification.requiredEvidence.join(", ")}`,
    ].join("\n");
  }

  private renderEscalation(action: IncidentAction) {
    return [
      `Overdue incident action: ${action.title}`,
      `Owner: ${action.owner}`,
      `Severity: ${action.severity}`,
      `Root cause: ${action.rootCause}`,
    ].join("\n");
  }
}
```

`ActionPolicy` 可以很简单：

```ts
export class ActionPolicy {
  canClose(action: IncidentAction, evidence: IncidentAction["evidence"]) {
    const required = new Set(action.verification.requiredEvidence);
    const provided = new Set(evidence.map((e) => e.type));

    for (const item of required) {
      if (!provided.has(item)) {
        return { allow: false, reason: `missing ${item}` };
      }
    }

    if (action.severity === "P0" && !provided.has("reviewer_approval")) {
      return { allow: false, reason: "P0 action requires reviewer approval" };
    }

    return { allow: true };
  }

  escalationTarget(action: IncidentAction) {
    if (action.severity === "P0" || action.severity === "P1") {
      return "incident-commander";
    }
    return action.owner;
  }
}
```

这样整改闭环就变成系统行为，而不是“复盘文档里的一段建议”。

---

## 5. OpenClaw 实战：Cron/Heartbeat 扫描整改项

OpenClaw 里可以把整改项放在：

```text
memory/incident-actions.json
```

每次 heartbeat 或每天 cron 做三件事：

1. 扫描 `status != closed && dueAt < now` 的行动项
2. 对 P0/P1 逾期项通知负责人或老板
3. 对已提交证据的行动项运行最小验证，例如测试、PR 状态、监控查询

示例文件：

```json
[
  {
    "actionId": "act_inc_course_delivery_01",
    "incidentId": "inc_course_delivery_duplicate",
    "severity": "P1",
    "owner": "agent-course-maintainer",
    "title": "Add duplicate-send guard before Telegram publish",
    "dueAt": "2026-05-18T00:00:00Z",
    "status": "open",
    "verification": {
      "type": "test",
      "requiredEvidence": ["commit_sha", "test_output", "message_id_check"]
    },
    "evidence": []
  }
]
```

Cron 的伪代码：

```text
read memory/incident-actions.json
for action in open actions:
  if dueAt passed:
    notify owner / escalate
  if evidence exists:
    run verification gate
    close only if gate passed
write updated file
```

注意：如果是外部通知，不要每 3 分钟刷屏。要记录 `lastNotifiedAt`，按严重级别节流：

- P0/P1：每 2-4 小时提醒一次
- P2：每天一次
- P3：周报汇总即可

---

## 6. 常见坑

### 坑 1：整改项标题太虚

坏例子：

```text
Improve reliability
```

好例子：

```text
Add idempotency key check before Telegram lesson publish
```

Agent 不能跟踪“提高可靠性”，只能跟踪具体动作。

### 坑 2：owner 不明确

坏例子：

```json
{"owner": "team"}
```

好例子：

```json
{"owner": "platform-oncall"}
```

owner 必须能被通知、能被升级。

### 坑 3：关闭不需要证据

如果整改项可以被一句“done”关闭，它迟早会变成形式主义。

关闭至少要有一种证据：

- PR / commit sha
- 测试输出
- 监控截图或查询结果
- runbook 版本号
- policy 变更记录
- reviewer approval

---

## 7. 设计原则

把事故整改做成系统时，记住四句话：

1. **Root cause 必须产生 action**
2. **Action 必须有 owner 和 due date**
3. **Close 必须有 evidence**
4. **Overdue 必须升级，不要静默过期**

这套机制很朴素，但非常有用。

---

## 8. 小结

今天这一课的重点：

- Incident Review 之后要接 Action Tracker
- corrective 修当前问题，preventive 防未来复发
- 每个行动项都要 owner、due date、verification、evidence
- learn-claude-code 用 JSON 文件实现最小闭环
- pi-mono 用中间件把创建、提醒、升级、关闭统一治理
- OpenClaw 可以用 Cron/Heartbeat 定期扫描逾期整改项

一句话收尾：

> 复盘让事故变成经验；整改追踪让经验真的改变系统。
