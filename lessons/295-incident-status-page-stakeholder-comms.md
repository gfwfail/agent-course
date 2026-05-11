# 295. Agent 事故状态页与利益相关方沟通（Status Page & Stakeholder Comms）

上一课我们做了 Incident Command Center：事故能被分级、去重、升级、冻结、恢复。

今天补上生产系统里很容易被忽略的一环：**事故期间，Agent 应该怎么对外沟通。**

一句话：

> 事故处理不是只把系统修好，还要让相关人知道“发生了什么、影响什么、现在谁在处理、下一次什么时候更新”。

---

## 1. 为什么 Agent 需要状态页/沟通层

很多 Agent 事故最伤人的不是故障本身，而是沟通混乱：

- 同一个事故，群里刷了 10 条重复告警
- 用户只看到“失败了”，不知道是否会重试
- 老板问进度时，Agent 临时翻日志，回答前后不一致
- 事故恢复了，但没人确认副作用有没有补偿
- 复盘时找不到当时对外说过什么

所以 Command Center 后面要接一个 Communication Layer：

```text
incident opened/updated/resolved
  -> build audience-specific update
  -> rate limit / dedupe
  -> publish to status page / group / DM / issue
  -> store communication record as evidence
```

核心不是“多发消息”，而是**少而准地发可信更新**。

---

## 2. 最小数据模型：IncidentUpdate

```json
{
  "incidentId": "inc_20260511_001",
  "version": 3,
  "severity": "P1",
  "status": "mitigating",
  "audience": "internal",
  "summary": "Telegram delivery is degraded; retries are queued.",
  "impact": "Course posts and heartbeat notifications may be delayed.",
  "currentAction": "Delivery worker is retrying with backoff; duplicate sends are blocked.",
  "nextUpdateAt": "2026-05-11T15:45:00Z",
  "links": ["state/incidents/inc_20260511_001.json"],
  "publishedAt": "2026-05-11T15:30:00Z"
}
```

字段重点：

- `audience`：老板、内部群、用户、状态页，说法不一样
- `summary`：一句话讲清楚当前状态
- `impact`：影响范围，不要只说技术错误
- `currentAction`：正在做什么，避免“没人管”的感觉
- `nextUpdateAt`：承诺下一次更新时间
- `version`：更新有序，避免旧消息覆盖新消息

---

## 3. learn-claude-code：Python 教学版

先做一个文件式沟通发布器：负责节流、生成文案、记录证据。

```python
# learn_claude_code/incident_comms.py
from __future__ import annotations

import json
import time
from dataclasses import dataclass, asdict
from pathlib import Path
from typing import Literal

Audience = Literal["owner", "internal", "public"]


@dataclass
class IncidentUpdate:
    incident_id: str
    version: int
    severity: str
    status: str
    audience: Audience
    summary: str
    impact: str
    current_action: str
    next_update_at: float | None
    published_at: float


class IncidentComms:
    def __init__(self, path: str = "state/incident-updates.jsonl"):
        self.path = Path(path)
        self.path.parent.mkdir(parents=True, exist_ok=True)
        self.path.touch(exist_ok=True)

    def _history(self) -> list[dict]:
        return [json.loads(line) for line in self.path.read_text().splitlines() if line.strip()]

    def should_publish(self, incident_id: str, audience: Audience, min_interval_sec: int) -> bool:
        now = time.time()
        for item in reversed(self._history()):
            if item["incident_id"] == incident_id and item["audience"] == audience:
                return now - item["published_at"] >= min_interval_sec
        return True

    def build_update(self, incident: dict, audience: Audience) -> IncidentUpdate:
        now = time.time()
        history = [x for x in self._history() if x["incident_id"] == incident["incident_id"]]
        version = len(history) + 1

        impact = incident.get("impact", "Some automated actions may be delayed.")
        if audience == "public":
            # public 文案不泄露内部路径、token、栈信息
            current_action = "We are mitigating the issue and will post another update soon."
        else:
            current_action = incident.get("current_action", "Commander is collecting evidence and applying controls.")

        return IncidentUpdate(
            incident_id=incident["incident_id"],
            version=version,
            severity=incident["severity"],
            status=incident["status"],
            audience=audience,
            summary=incident["title"],
            impact=impact,
            current_action=current_action,
            next_update_at=now + 15 * 60 if incident["status"] != "resolved" else None,
            published_at=now,
        )

    def render_markdown(self, update: IncidentUpdate) -> str:
        next_line = (
            f"Next update: {time.strftime('%H:%M UTC', time.gmtime(update.next_update_at))}"
            if update.next_update_at else
            "Next update: none, incident resolved"
        )
        return "\n".join([
            f"[{update.severity}] {update.summary}",
            f"Status: {update.status}",
            f"Impact: {update.impact}",
            f"Action: {update.current_action}",
            next_line,
            f"Incident: {update.incident_id} v{update.version}",
        ])

    def publish(self, update: IncidentUpdate) -> str:
        # 教学版只写 JSONL；真实系统在这里接 Telegram/Slack/status page
        with self.path.open("a") as f:
            f.write(json.dumps(asdict(update), ensure_ascii=False) + "\n")
        return self.render_markdown(update)
```

使用方式：

```python
incident = {
    "incident_id": "inc_20260511_001",
    "severity": "P1",
    "status": "mitigating",
    "title": "Telegram delivery is degraded",
    "impact": "Agent course posts may be delayed, but messages are queued.",
    "current_action": "Retry worker is backing off; duplicate-send guard is enabled.",
}

comms = IncidentComms()

if comms.should_publish(incident["incident_id"], "internal", min_interval_sec=900):
    update = comms.build_update(incident, "internal")
    print(comms.publish(update))
```

这个版本有三个关键点：

- 每个 audience 独立节流
- public 文案自动脱敏
- 所有对外更新都落 JSONL，复盘可追踪

---

## 4. pi-mono：生产版中间件

生产里，沟通不应该散落在业务代码里，而是 Incident Middleware 的后置动作。

```ts
// pi-mono/src/incident/IncidentCommsMiddleware.ts
export type Audience = 'owner' | 'internal' | 'public'

export interface Incident {
  id: string
  severity: 'P0' | 'P1' | 'P2' | 'P3'
  status: 'open' | 'mitigating' | 'resolved'
  title: string
  impact?: string
  currentAction?: string
  updatedAt: string
}

export interface IncidentUpdate {
  incidentId: string
  version: number
  audience: Audience
  severity: Incident['severity']
  status: Incident['status']
  summary: string
  impact: string
  currentAction: string
  nextUpdateAt?: string
}

export interface CommsSink {
  publish(update: IncidentUpdate, text: string): Promise<{ messageId: string }>
}

export class IncidentCommsMiddleware {
  constructor(
    private readonly sinks: Record<Audience, CommsSink>,
    private readonly store: {
      lastPublished(incidentId: string, audience: Audience): Promise<Date | null>
      countUpdates(incidentId: string): Promise<number>
      savePublished(update: IncidentUpdate, messageId: string): Promise<void>
    },
    private readonly minIntervalMs = 15 * 60 * 1000,
  ) {}

  async onIncidentChanged(incident: Incident): Promise<void> {
    const audiences = this.routeAudiences(incident)

    for (const audience of audiences) {
      if (!(await this.canPublish(incident.id, audience, incident.status))) continue

      const update = await this.buildUpdate(incident, audience)
      const text = this.render(update)
      const result = await this.sinks[audience].publish(update, text)

      await this.store.savePublished(update, result.messageId)
    }
  }

  private routeAudiences(incident: Incident): Audience[] {
    if (incident.severity === 'P0') return ['owner', 'internal', 'public']
    if (incident.severity === 'P1') return ['owner', 'internal']
    return ['internal']
  }

  private async canPublish(
    incidentId: string,
    audience: Audience,
    status: Incident['status'],
  ): Promise<boolean> {
    if (status === 'resolved') return true

    const last = await this.store.lastPublished(incidentId, audience)
    if (!last) return true

    return Date.now() - last.getTime() >= this.minIntervalMs
  }

  private async buildUpdate(incident: Incident, audience: Audience): Promise<IncidentUpdate> {
    const version = (await this.store.countUpdates(incident.id)) + 1

    return {
      incidentId: incident.id,
      version,
      audience,
      severity: incident.severity,
      status: incident.status,
      summary: incident.title,
      impact: incident.impact ?? 'Some automated workflows may be delayed.',
      currentAction: audience === 'public'
        ? 'We are working on mitigation and will update when there is material progress.'
        : incident.currentAction ?? 'Commander is collecting evidence and applying controls.',
      nextUpdateAt: incident.status === 'resolved'
        ? undefined
        : new Date(Date.now() + this.minIntervalMs).toISOString(),
    }
  }

  private render(update: IncidentUpdate): string {
    const next = update.nextUpdateAt
      ? `Next update: ${update.nextUpdateAt}`
      : 'Next update: none, incident resolved'

    return [
      `[${update.severity}] ${update.summary}`,
      `Status: ${update.status}`,
      `Impact: ${update.impact}`,
      `Action: ${update.currentAction}`,
      next,
      `Incident: ${update.incidentId} v${update.version}`,
    ].join('\n')
  }
}
```

这里有几个生产细节：

- `routeAudiences()` 控制谁该被打扰
- `canPublish()` 防止事故期间刷屏
- `buildUpdate()` 对 public 自动脱敏
- `savePublished()` 把 messageId 作为审计证据

---

## 5. OpenClaw：把它接到 Cron / Heartbeat

OpenClaw 里最常见的是两种沟通：

1. **直接通知老板**：P0/P1，需要人类决策
2. **群内状态更新**：课程、部署、任务队列等共享工作流

一个实用的发送前闸门：

```ts
// openclaw-style pseudo code
async function maybeNotifyIncident(ctx, incident) {
  const comms = await ctx.incidentComms.buildUpdate({
    incident,
    audience: incident.severity === 'P0' ? 'owner' : 'internal',
  })

  const gate = await ctx.gates.check({
    name: 'incident-comms-before-send',
    checks: [
      { name: 'has-impact', pass: Boolean(comms.impact) },
      { name: 'has-next-update-or-resolved', pass: Boolean(comms.nextUpdateAt || incident.status === 'resolved') },
      { name: 'no-secret-leak', pass: !/token|password|secret|AKIA/i.test(comms.text) },
      { name: 'not-duplicate', pass: await ctx.comms.notRecentlySent(comms.dedupeKey) },
    ],
  })

  if (!gate.pass) {
    await ctx.evidence.write('incident-comms-blocked', { incidentId: incident.id, gate })
    return
  }

  const sent = await ctx.message.send({
    target: incident.channelId,
    message: comms.text,
  })

  await ctx.evidence.write('incident-comms-sent', {
    incidentId: incident.id,
    audience: comms.audience,
    messageId: sent.messageId,
  })
}
```

注意：事故沟通本身也是副作用，发送前必须过闸门：

- 是否有影响范围
- 是否承诺下一次更新时间
- 是否泄露密钥/内部路径
- 是否重复发送
- 是否写入 evidence

---

## 6. 状态页不是博客，是状态机

不要把状态页做成随手编辑的 Markdown。

更稳的做法是固定状态机：

```text
investigating -> identified -> monitoring -> resolved
```

每次更新只能从合法状态转移：

```ts
const allowedTransitions = {
  investigating: ['identified', 'monitoring', 'resolved'],
  identified: ['monitoring', 'resolved'],
  monitoring: ['resolved'],
  resolved: [],
} as const

function assertTransition(from: string, to: string) {
  if (!allowedTransitions[from]?.includes(to as never)) {
    throw new Error(`invalid incident status transition: ${from} -> ${to}`)
  }
}
```

这样能避免事故状态乱跳：

- 刚打开就 “resolved” 但没有验证
- monitoring 又退回 investigating 却没有新 incident
- resolved 后继续追加“还在修”的消息

---

## 7. 实战建议

### 建议一：不同 audience 不同信息密度

- 老板：影响、风险、是否需要决策
- 内部群：证据、当前动作、owner、下一步
- 公开状态页：影响范围、缓解状态、下一次更新时间

### 建议二：只在“状态变化”或“承诺时间到”时更新

不要每次日志变化都发消息。好的更新触发条件：

- severity 改变
- impact 扩大/缩小
- control mode 改变
- mitigation 开始/完成
- 到了 `nextUpdateAt`
- incident resolved

### 建议三：resolved 必须带 proof

比如：

```json
{
  "status": "resolved",
  "proof": {
    "delivery_error_rate_15m": 0,
    "queued_messages": 0,
    "duplicate_send_count": 0,
    "verified_at": "2026-05-11T16:02:00Z"
  }
}
```

没有 proof，只能叫 `monitoring`，不能叫 `resolved`。

---

## 8. 小结

今天的重点：

- Incident Command Center 负责处理事故
- Communication Layer 负责稳定沟通
- `audience + rate limit + dedupe + evidence` 是事故沟通四件套
- public 文案必须脱敏，owner/internal 可以给更多技术细节
- 状态页应该是状态机，不是随手写公告
- resolved 必须有 proof

成熟 Agent 的事故沟通标准不是“喊得快”，而是：

> 该通知的人及时知道，该沉默的时候不刷屏，每一次对外更新都有证据可追。🫡
