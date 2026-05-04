# 237. Agent 收件箱与命令总线（Inbox & Command Bus）

上一课讲 Outbox：Agent 对外发送消息，不能只靠一次 `send()`，要先落库、可重试、可查回执。

今天讲另一半：**Inbox**。真实生产里的 Agent 不只是“等用户发一条消息，然后立刻处理”。它会同时收到：Telegram 消息、Discord mention、Webhook、GitHub review、cron wake、支付回调、人工审批结果……如果这些入口都直接调用 Agent Loop，系统很快会变成一团毛线。

更稳的做法是：**所有入口先写 Inbox，再由 Command Bus 按规则处理。**

---

## 1. 问题：不要让入口直接驱动业务逻辑

常见坏写法：

```ts
bot.on('message', async (msg) => {
  const reply = await agent.run(msg.text)
  await bot.sendMessage(msg.chat.id, reply)
})

app.post('/webhook/github', async (req, res) => {
  await agent.run(`review event: ${JSON.stringify(req.body)}`)
  res.sendStatus(200)
})
```

这段 Demo 能跑，但生产会踩坑：

- Telegram 重试投递会导致同一条消息处理两次；
- Webhook 必须快速 ack，Agent 运行可能超过超时；
- 多渠道事件格式不同，路由逻辑散落各处；
- 系统崩溃时，不知道哪些事件已收到、哪些已处理；
- 高优先级人工指令可能被一堆低优先级 cron 堵住。

**入口应该只负责接收和确认，真正处理交给统一命令总线。**

---

## 2. 核心模型：Event → InboxItem → Command

可以分三层：

```text
Raw Event     平台原始事件：Telegram update / GitHub webhook / cron tick
InboxItem     已标准化、去重、可审计的收件箱记录
Command       Agent 真正要执行的意图：handle_user_message / run_cron / process_review
```

最小字段：

```ts
type InboxItem = {
  id: string
  source: 'telegram' | 'github' | 'cron' | 'webhook'
  sourceEventId: string       // 平台原始事件 id，用于去重
  actorId?: string            // 谁触发的
  channelId?: string          // 从哪里来
  type: string                // message.created / pr.reviewed / cron.tick
  payload: unknown
  priority: number
  status: 'pending' | 'processing' | 'done' | 'failed' | 'ignored'
  attempts: number
  receivedAt: string
  availableAt: string
  dedupeKey: string
}
```

关键是 `dedupeKey`：

```text
telegram:update:123456
telegram:chat:-5115329245:message:10336
github:gfwfail/esim:review:98765
cron:agent-course:2026-05-04T18:30
```

只要 Inbox 对 `dedupeKey` 加唯一约束，入口重试就不会重复触发业务动作。

---

## 3. learn-claude-code：SQLite 教学版 Inbox

教学版可以很小：一个 SQLite 表 + 一个 claim 函数。

```python
# learn-claude-code: inbox.py
import json, sqlite3, time, uuid

conn = sqlite3.connect('agent.db')
conn.execute('''
CREATE TABLE IF NOT EXISTS inbox (
  id TEXT PRIMARY KEY,
  dedupe_key TEXT UNIQUE NOT NULL,
  source TEXT NOT NULL,
  type TEXT NOT NULL,
  payload TEXT NOT NULL,
  priority INTEGER NOT NULL DEFAULT 50,
  status TEXT NOT NULL DEFAULT 'pending',
  attempts INTEGER NOT NULL DEFAULT 0,
  received_at REAL NOT NULL,
  available_at REAL NOT NULL
)
''')


def receive_event(*, dedupe_key: str, source: str, type: str, payload: dict, priority: int = 50):
    try:
        conn.execute(
            '''INSERT INTO inbox
               (id, dedupe_key, source, type, payload, priority, received_at, available_at)
               VALUES (?, ?, ?, ?, ?, ?, ?, ?)''',
            (str(uuid.uuid4()), dedupe_key, source, type, json.dumps(payload), priority, time.time(), time.time())
        )
        conn.commit()
        return {'accepted': True}
    except sqlite3.IntegrityError:
        # 平台重试/重复 webhook：已收到即可，不再重复处理
        return {'accepted': False, 'reason': 'duplicate'}


def claim_next():
    row = conn.execute('''
      SELECT id, type, payload
      FROM inbox
      WHERE status = 'pending' AND available_at <= ?
      ORDER BY priority DESC, received_at ASC
      LIMIT 1
    ''', (time.time(),)).fetchone()
    if not row:
        return None

    item_id, type_, payload = row
    changed = conn.execute('''
      UPDATE inbox
      SET status = 'processing', attempts = attempts + 1
      WHERE id = ? AND status = 'pending'
    ''', (item_id,)).rowcount
    conn.commit()

    if not changed:
        return None
    return {'id': item_id, 'type': type_, 'payload': json.loads(payload)}
```

处理循环：

```python
def dispatch(item):
    if item['type'] == 'telegram.message.created':
        return agent.run_user_message(item['payload'])
    if item['type'] == 'cron.agent_course.tick':
        return agent.run_course_cron(item['payload'])
    if item['type'] == 'github.pr.reviewed':
        return agent.handle_pr_review(item['payload'])
    return {'ignored': True, 'reason': 'unknown command'}

while True:
    item = claim_next()
    if not item:
        time.sleep(1)
        continue

    try:
        dispatch(item)
        conn.execute("UPDATE inbox SET status = 'done' WHERE id = ?", (item['id'],))
    except Exception as e:
        conn.execute(
            "UPDATE inbox SET status = 'pending', available_at = ? WHERE id = ?",
            (time.time() + 60, item['id'])  # 简单退避
        )
    conn.commit()
```

教学重点：**HTTP handler / Telegram listener 只写 Inbox，不直接跑 Agent。**

---

## 4. pi-mono：Command Bus 生产版

生产版可以抽象成两个组件：

```ts
interface InboxAdapter {
  receive(input: RawInboundEvent): Promise<ReceiveResult>
  claim(workerId: string): Promise<InboxItem | null>
  complete(id: string, result: unknown): Promise<void>
  retry(id: string, error: Error, nextAt: Date): Promise<void>
  deadLetter(id: string, error: Error): Promise<void>
}

interface CommandHandler<TPayload> {
  type: string
  handle(payload: TPayload, ctx: CommandContext): Promise<void>
}
```

CommandBus：

```ts
class CommandBus {
  private handlers = new Map<string, CommandHandler<any>>()

  register(handler: CommandHandler<any>) {
    this.handlers.set(handler.type, handler)
  }

  async dispatch(item: InboxItem, ctx: CommandContext) {
    const handler = this.handlers.get(item.type)
    if (!handler) {
      await ctx.inbox.complete(item.id, { ignored: true, reason: 'unknown_command' })
      return
    }

    await handler.handle(item.payload, ctx)
    await ctx.inbox.complete(item.id, { ok: true })
  }
}
```

注册命令：

```ts
bus.register({
  type: 'telegram.message.created',
  async handle(payload, ctx) {
    const decision = await ctx.permission.check({
      actorId: payload.from.id,
      channelId: payload.chat.id,
      action: 'chat.respond',
    })

    if (!decision.allowed) return

    const reply = await ctx.agent.run({
      input: payload.text,
      actorId: payload.from.id,
      channelId: payload.chat.id,
    })

    // 注意：这里不要直接 send，应该写 Outbox
    await ctx.outbox.enqueue({
      channel: 'telegram',
      target: payload.chat.id,
      text: reply,
      dedupeKey: `reply:${payload.chat.id}:${payload.message_id}`,
    })
  },
})
```

这样 Inbox + Outbox 就闭环了：

```text
Inbound platform event
  → Inbox receive / dedupe / priority
  → CommandBus dispatch
  → Agent run / tools
  → Outbox enqueue
  → Dispatcher send / receipt
```

---

## 5. OpenClaw 实战：cron 和群消息都应该是命令

这门课本身就是一个例子。

当前 cron 触发的是：

```text
[cron: Agent 开发课程 - 每3小时]
```

它本质上可以标准化成：

```json
{
  "type": "cron.agent_course.tick",
  "source": "openclaw-cron",
  "dedupeKey": "cron:agent-course:2026-05-04T18:30+11:00",
  "priority": 70,
  "payload": {
    "repo": "/Users/bot001/.openclaw/workspace/agent-course",
    "telegramGroupId": "-5115329245",
    "requirements": [
      "avoid duplicate topics",
      "write lesson file",
      "update README",
      "send Telegram",
      "git commit and push"
    ]
  }
}
```

好处：

- cron 重试不会发两节重复课；
- 如果发 Telegram 成功但 git push 失败，可从 Inbox/Outbox 状态恢复；
- 用户临时插入“停止课程”时，可以把同类 pending command 标记 ignored；
- 高优先级老板指令可以插队低优先级巡检任务。

---

## 6. 设计 Checklist

做 Inbox / Command Bus 时，至少检查 8 件事：

1. **入口是否快速 ack？** Webhook 不要等 Agent 跑完。
2. **是否有 dedupeKey？** 没有去重的 Inbox 只是日志。
3. **是否统一状态机？** pending / processing / done / failed / dead_letter。
4. **是否有 priority？** 所有事件 FIFO 会造成重要任务被堵。
5. **是否区分 actor/channel？** 权限判断必须知道是谁、在哪里触发。
6. **处理结果是否写回？** 方便复盘和恢复。
7. **失败是否分类？** transient 可重试，permission/config/schema 错误应进 DLQ 或等人工。
8. **输出是否走 Outbox？** Inbox 解决输入可靠性，Outbox 解决输出可靠性。

---

## 7. 一句话总结

**Outbox 让 Agent 的“说出去”可靠；Inbox + Command Bus 让 Agent 的“听进来”可靠。**

真正生产级的 Agent，不应该被某个平台事件直接牵着跑；它应该把所有外部输入先变成可去重、可排队、可审计、可恢复的命令。这样系统崩了可以续跑，平台重试不会重复，老板临时插队也不会被 cron 堵住。
