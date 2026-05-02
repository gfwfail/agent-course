# 221. Agent 通知路由与噪声预算（Notification Routing & Noise Budget）

> 关键词：notification routing、noise budget、dedupe、digest、priority、quiet hours、proactive agent

上一课讲了 **死信队列 DLQ**：任务彻底失败以后，不能只丢一个 `failed` 状态，要留下可复盘、可恢复、可人工介入的失败档案。

今天讲另一个生产 Agent 很容易踩坑的问题：**什么时候该通知用户？通知到哪里？通知多少次？**

很多 Agent 一开始会这样写：

```txt
发现新事件 → 立刻发消息
工具失败 → 立刻发消息
任务完成 → 立刻发消息
心跳检查 → 立刻发消息
```

短期看很“主动”，长期看就是噪音制造机。

真正好用的 Always-on Agent 必须有一层 **通知路由与噪声预算（Notification Routing & Noise Budget）**：

> 不是所有事件都值得打扰用户；Agent 要学会判断紧急度、聚合重复事件、尊重免打扰时间，并把低优先级信息变成摘要。

一句话：**主动不是多说话，主动是只在值得打扰的时候打扰。**

---

## 1. 核心思想

通知系统不是 `sendMessage(event.text)`，而是一个决策管道：

```txt
Event
  → classify priority
  → dedupe repeated signals
  → check quiet hours
  → apply noise budget
  → choose route
  → send now / batch digest / suppress
```

推荐把通知分成 4 个级别：

| 等级 | 含义 | 行为 |
|---|---|---|
| critical | 安全、资金、生产事故、马上过期 | 立即通知，可突破免打扰 |
| high | 需要用户尽快决策 | 立即通知，但要去重 |
| normal | 有价值但不急 | 进入摘要，按批次发送 |
| low | 仅供记录 | 写日志/记忆，不主动发 |

关键点：**通知是外部副作用**，所以它和删库、发邮件一样，需要被治理。

一个成熟 Agent 的通知层至少要有 5 个能力：

1. **优先级分类**：事件是否真的需要打扰用户。
2. **去重**：同类事件短时间内只提醒一次。
3. **聚合**：低优先级事件合并成 digest。
4. **路由**：Telegram、Discord、邮件、日志、任务面板分别适合不同消息。
5. **预算**：限制单位时间主动消息数量，防止 Agent 变成骚扰源。

---

## 2. 反模式：把“主动”做成“刷屏”

常见错误是把所有事件都发给用户：

```python
async def handle_event(event):
    await telegram.send(event.message)
```

看起来简单，问题很多：

- heartbeat 每 30 分钟发一次“没变化”；
- 10 个工具连续失败，发 10 条错误；
- 用户半夜收到 normal 级别提醒；
- 同一个部署失败被 cron、monitor、sub-agent 各提醒一次；
- 真正 critical 的消息淹没在普通噪音里。

Agent 的主动性应该像优秀助理：

```txt
小事记下来；
同类合并说；
重要的及时说；
紧急的马上说。
```

---

## 3. learn-claude-code：Python 教学版

教学版先用一个内存状态 + JSON 文件做最小实现。

```python
# learn-claude-code/notification_router.py
import hashlib
import json
import time
from dataclasses import dataclass, asdict
from pathlib import Path
from typing import Literal

Priority = Literal["critical", "high", "normal", "low"]
Decision = Literal["send_now", "digest", "suppress", "log_only"]


@dataclass
class NotificationEvent:
    source: str
    title: str
    body: str
    priority: Priority
    user_id: str
    route_hint: str | None = None
    entity_id: str | None = None


@dataclass
class RouteDecision:
    decision: Decision
    route: str | None
    reason: str
    dedupe_key: str


class NotificationRouter:
    def __init__(self, state_path: str = "memory/notification-state.json"):
        self.state_path = Path(state_path)
        self.state_path.parent.mkdir(parents=True, exist_ok=True)
        self.state = self._load_state()

    def _load_state(self) -> dict:
        if not self.state_path.exists():
            return {"sent": {}, "digest": [], "hourlyCount": {}}
        return json.loads(self.state_path.read_text())

    def _save_state(self):
        self.state_path.write_text(json.dumps(self.state, ensure_ascii=False, indent=2))

    def dedupe_key(self, event: NotificationEvent) -> str:
        raw = f"{event.user_id}:{event.source}:{event.entity_id or event.title}"
        return hashlib.sha256(raw.encode()).hexdigest()[:16]

    def is_quiet_hours(self) -> bool:
        # 示例：23:00-08:00 免打扰；真实系统按用户时区配置
        hour = time.localtime().tm_hour
        return hour >= 23 or hour < 8

    def under_noise_budget(self, user_id: str, limit_per_hour: int = 3) -> bool:
        hour_key = time.strftime("%Y%m%d%H")
        key = f"{user_id}:{hour_key}"
        return self.state["hourlyCount"].get(key, 0) < limit_per_hour

    def mark_sent(self, user_id: str, dedupe_key: str):
        now = int(time.time())
        self.state["sent"][dedupe_key] = now

        hour_key = time.strftime("%Y%m%d%H")
        key = f"{user_id}:{hour_key}"
        self.state["hourlyCount"][key] = self.state["hourlyCount"].get(key, 0) + 1
        self._save_state()

    def decide(self, event: NotificationEvent) -> RouteDecision:
        key = self.dedupe_key(event)
        now = int(time.time())
        last_sent = self.state["sent"].get(key)

        # 同一实体 30 分钟内不重复提醒，critical 除外
        if event.priority != "critical" and last_sent and now - last_sent < 1800:
            return RouteDecision("suppress", None, "deduped within 30 minutes", key)

        if event.priority == "low":
            return RouteDecision("log_only", None, "low priority", key)

        if event.priority == "normal":
            return RouteDecision("digest", "daily_digest", "normal priority batched", key)

        if self.is_quiet_hours() and event.priority != "critical":
            return RouteDecision("digest", "morning_digest", "quiet hours", key)

        if not self.under_noise_budget(event.user_id) and event.priority != "critical":
            return RouteDecision("digest", "rate_limited_digest", "noise budget exceeded", key)

        route = event.route_hint or "telegram"
        return RouteDecision("send_now", route, "important and allowed", key)
```

调用时不要直接发消息，而是先问 Router：

```python
async def notify(router: NotificationRouter, event: NotificationEvent):
    decision = router.decide(event)

    if decision.decision == "send_now":
        await send_to_route(decision.route, f"{event.title}\n{event.body}")
        router.mark_sent(event.user_id, decision.dedupe_key)
        return

    if decision.decision == "digest":
        router.state["digest"].append({
            "at": int(time.time()),
            "event": asdict(event),
            "reason": decision.reason,
        })
        router._save_state()
        return

    if decision.decision == "log_only":
        print("log notification:", event.title)
        return

    # suppress: 什么都不做，但最好记录一下方便调试
    print("suppressed:", event.title, decision.reason)
```

这样 Agent 就从“有事就喊”变成了“有判断地汇报”。

---

## 4. pi-mono：生产版 NotificationPolicy

生产里建议把通知策略做成独立基础设施，放在 Agent Loop 外层。

```ts
// pi-mono/packages/agent-runtime/src/notification-policy.ts
export type NotificationPriority = "critical" | "high" | "normal" | "low";

export interface NotificationEvent {
  id: string;
  userId: string;
  source: "cron" | "tool" | "subagent" | "monitor" | "workflow";
  title: string;
  body: string;
  priority: NotificationPriority;
  entityKey?: string;
  routeHint?: "telegram" | "discord" | "email" | "audit_log";
  createdAt: Date;
}

export type NotificationAction =
  | { type: "send_now"; route: string; reason: string }
  | { type: "enqueue_digest"; digestKey: string; reason: string }
  | { type: "suppress"; reason: string }
  | { type: "audit_only"; reason: string };

export interface NotificationStateStore {
  getLastSent(key: string): Promise<Date | null>;
  markSent(key: string, at: Date): Promise<void>;
  incrementBudget(userId: string, windowKey: string): Promise<number>;
  enqueueDigest(userId: string, digestKey: string, event: NotificationEvent): Promise<void>;
}

export class NotificationPolicy {
  constructor(
    private readonly store: NotificationStateStore,
    private readonly config: {
      quietHours: { startHour: number; endHour: number };
      dedupeWindowMs: number;
      maxProactivePerHour: number;
    },
  ) {}

  async decide(event: NotificationEvent, now = new Date()): Promise<NotificationAction> {
    const key = this.dedupeKey(event);
    const lastSent = await this.store.getLastSent(key);

    if (
      event.priority !== "critical" &&
      lastSent &&
      now.getTime() - lastSent.getTime() < this.config.dedupeWindowMs
    ) {
      return { type: "suppress", reason: "duplicate_event" };
    }

    if (event.priority === "low") {
      return { type: "audit_only", reason: "low_priority" };
    }

    if (event.priority === "normal") {
      return { type: "enqueue_digest", digestKey: "daily", reason: "normal_priority" };
    }

    if (event.priority !== "critical" && this.inQuietHours(now)) {
      return { type: "enqueue_digest", digestKey: "morning", reason: "quiet_hours" };
    }

    if (event.priority !== "critical") {
      const used = await this.store.incrementBudget(event.userId, this.windowKey(now));
      if (used > this.config.maxProactivePerHour) {
        return { type: "enqueue_digest", digestKey: "hourly_overflow", reason: "noise_budget" };
      }
    }

    return {
      type: "send_now",
      route: event.routeHint ?? "telegram",
      reason: "priority_allowed",
    };
  }

  async apply(event: NotificationEvent, action: NotificationAction) {
    if (action.type === "send_now") {
      await this.store.markSent(this.dedupeKey(event), new Date());
    }

    if (action.type === "enqueue_digest") {
      await this.store.enqueueDigest(event.userId, action.digestKey, event);
    }
  }

  private dedupeKey(event: NotificationEvent): string {
    return [event.userId, event.source, event.entityKey ?? event.title].join(":");
  }

  private windowKey(now: Date): string {
    return now.toISOString().slice(0, 13); // YYYY-MM-DDTHH
  }

  private inQuietHours(now: Date): boolean {
    const hour = now.getHours();
    const { startHour, endHour } = this.config.quietHours;
    return startHour < endHour
      ? hour >= startHour && hour < endHour
      : hour >= startHour || hour < endHour;
  }
}
```

Agent 运行时只负责产生结构化事件：

```ts
await notifications.emit({
  id: crypto.randomUUID(),
  userId: run.userId,
  source: "workflow",
  title: "部署需要确认",
  body: "staging 部署通过，但生产发布需要人工审批。",
  priority: "high",
  entityKey: `deploy:${deployment.id}`,
  routeHint: "telegram",
  createdAt: new Date(),
});
```

通知服务负责决策和发送：

```ts
export class NotificationService {
  constructor(
    private readonly policy: NotificationPolicy,
    private readonly channels: Record<string, { send(text: string): Promise<void> }>,
  ) {}

  async emit(event: NotificationEvent) {
    const action = await this.policy.decide(event);
    await this.policy.apply(event, action);

    if (action.type !== "send_now") {
      return { delivered: false, action };
    }

    await this.channels[action.route].send(formatNotification(event));
    return { delivered: true, action };
  }
}
```

这层抽象的好处是：

- Agent Loop 不关心 Telegram/Discord/Email 细节；
- 通知策略可以热更新；
- 测试时可以只断言 `NotificationAction`，不真实发消息；
- 后续接入用户偏好、节假日、团队值班表都很自然。

---

## 5. OpenClaw：Heartbeat 里的通知纪律

OpenClaw 的 Heartbeat 天生适合做主动助手，但也最容易变成刷屏源。

推荐模式：Heartbeat 先产出事件，再让通知策略决定是否发：

```txt
heartbeat wake
  → 检查邮件/日历/项目/任务
  → 生成 NotificationEvent[]
  → NotificationPolicy 决策
  → critical/high 立即 message.send
  → normal 写入 digest 文件
  → low 只写 memory
```

一个简化版 OpenClaw 文件式状态可以是：

```json
// memory/notification-state.json
{
  "lastSent": {
    "calendar:meeting-123": 1777708800,
    "aws:billing-spike": 1777709000
  },
  "hourlyBudget": {
    "67431246:2026-05-02T19": 2
  },
  "digest": [
    {
      "title": "3 个低优先级 GitHub 通知",
      "priority": "normal",
      "createdAt": "2026-05-02T19:40:00+11:00"
    }
  ]
}
```

发送前做一个小判断：

```ts
function shouldInterrupt(event: NotificationEvent, state: NotificationState) {
  if (event.priority === "critical") return true;
  if (isQuietHours() && event.priority !== "critical") return false;
  if (wasRecentlySent(event, state)) return false;
  if (hourlyBudgetExceeded(event.userId, state)) return false;
  return event.priority === "high";
}
```

这也解释了为什么优秀的 Heartbeat 经常应该“安静”：

- 没有新东西，不说；
- 有普通信息，攒摘要；
- 有重要变化，才打断；
- 有紧急风险，马上打断。

**沉默不是偷懒，沉默是通知策略正常工作。**

---

## 6. Digest：低优先级信息不要消失，要合并

不要把 normal 事件直接丢掉。它们可能不值得马上发，但值得做成摘要。

```python
def build_digest(events: list[dict]) -> str:
    grouped: dict[str, list[dict]] = {}
    for e in events:
        grouped.setdefault(e["source"], []).append(e)

    lines = ["今日 Agent 摘要："]
    for source, items in grouped.items():
        lines.append(f"\n{source}：{len(items)} 件")
        for item in items[:5]:
            lines.append(f"- {item['title']}")
        if len(items) > 5:
            lines.append(f"- ...还有 {len(items) - 5} 件")

    return "\n".join(lines)
```

摘要发送策略可以很简单：

- 每天固定时间发一次；
- 用户主动问“今天有什么事”时发；
- digest 超过一定数量时发一条合并提醒；
- critical/high 永远不等 digest。

---

## 7. 生产 Checklist

做 Agent 通知层时，建议检查这 10 项：

- [ ] 每条通知都有 `priority`。
- [ ] 每条通知都有稳定 `entityKey` 用于去重。
- [ ] quiet hours 可配置，critical 可突破。
- [ ] 每用户/每渠道有主动消息预算。
- [ ] normal 级别进入 digest，不直接刷屏。
- [ ] low 级别只写日志/记忆。
- [ ] 发送结果可审计：sent/digested/suppressed/audit_only。
- [ ] 同类事件短时间内合并。
- [ ] 测试覆盖 critical/high/normal/low 四类路径。
- [ ] 用户可以调整偏好：少说点、多提醒、只提醒事故等。

---

## 8. 和已有模式的区别

| 模式 | 解决什么 |
|---|---|
| Error Handling | 工具/任务失败后怎么恢复 |
| DLQ | 重试耗尽后怎么留证据、人工介入 |
| Human-in-the-Loop | 高风险动作如何等待用户确认 |
| Notification Routing | 哪些事件值得通知、怎么通知、不刷屏 |
| Proactive Assistance | 如何主动发现用户可能需要的帮助 |

通知路由不是替代这些模式，而是它们的出口治理层。

没有通知策略的主动 Agent，很快会失去用户信任；有通知策略的主动 Agent，才像一个靠谱助理。

---

## 9. 小结

今天这课的重点：

1. 通知是外部副作用，必须被策略治理。
2. priority、dedupe、quiet hours、noise budget、digest 是五个基础件。
3. critical/high 才适合立即打断用户。
4. normal 进入摘要，low 写日志即可。
5. OpenClaw Heartbeat 的价值不是每次都说话，而是持续观察、只在该说时说。

记住一句话：

> 好 Agent 不是最吵的 Agent，而是最懂什么时候该闭嘴、什么时候必须开口的 Agent。
