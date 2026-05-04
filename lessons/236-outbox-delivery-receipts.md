# 236. Agent Outbox 与投递回执：消息发送也要可恢复

> 主题：Agent Outbox & Delivery Receipts
> 核心：把“发消息/发邮件/发 webhook”从一次性副作用，升级成可追踪、可重试、可验收的投递流水线。

很多 Agent 的最后一步是：把结果发到 Telegram、Discord、Email、GitHub Issue 或 Webhook。

Demo 里通常直接：

```python
await message_tool.send(chat_id, text)
```

生产里这很危险：

- LLM 生成完了，但发送接口超时，不知道到底发没发出去
- Cron 重跑，重复发两遍
- 消息太长被平台截断，Agent 以为成功
- Telegram 成功返回了 message_id，但本地进程崩了，状态没保存
- 用户问“刚才发了吗？”，Agent 没有证据

解决方法是 **Outbox Pattern（发件箱模式）+ Delivery Receipt（投递回执）**。

---

## 1. 核心设计

不要让 Agent 直接发消息。

正确流程：

1. 先写入 Outbox：我要给谁发什么，幂等键是什么
2. Dispatcher 读取 pending 消息
3. 调用真实发送工具
4. 保存平台回执：message_id / timestamp / channel
5. verify：按平台能力确认消息存在或至少确认 API 成功
6. 失败进入 retry / dead-letter

数据结构：

```ts
type OutboxItem = {
  id: string;
  channel: 'telegram' | 'discord' | 'email' | 'webhook';
  target: string;
  payload: {
    text?: string;
    media?: string[];
    metadata?: Record<string, unknown>;
  };
  idempotencyKey: string;
  status: 'pending' | 'sending' | 'sent' | 'verified' | 'failed' | 'dead';
  attempts: number;
  nextAttemptAt: string;
  receipt?: {
    providerMessageId?: string;
    providerTimestamp?: string;
    raw?: unknown;
  };
  lastError?: string;
  createdAt: string;
  updatedAt: string;
};
```

关键字段是 `idempotencyKey`。

例如课程 cron 可以用：

```text
agent-course:236:telegram:-5115329245
```

这样就算 cron 被触发两次，也不会重复发同一课。

---

## 2. learn-claude-code：最小 Python 教学版

用 SQLite 做一个最小 Outbox：

```python
# learn-claude-code/outbox.py
import json
import sqlite3
import time
import uuid
from dataclasses import dataclass

@dataclass
class OutboxMessage:
    channel: str
    target: str
    text: str
    idempotency_key: str

class Outbox:
    def __init__(self, path='agent.db'):
        self.db = sqlite3.connect(path)
        self.db.execute('''
        create table if not exists outbox (
          id text primary key,
          channel text not null,
          target text not null,
          payload text not null,
          idempotency_key text unique not null,
          status text not null,
          attempts integer not null default 0,
          next_attempt_at integer not null,
          receipt text,
          last_error text,
          created_at integer not null,
          updated_at integer not null
        )
        ''')

    def enqueue(self, msg: OutboxMessage) -> str:
        now = int(time.time())
        outbox_id = str(uuid.uuid4())
        self.db.execute('''
          insert or ignore into outbox
          (id, channel, target, payload, idempotency_key, status, next_attempt_at, created_at, updated_at)
          values (?, ?, ?, ?, ?, 'pending', ?, ?, ?)
        ''', (
          outbox_id,
          msg.channel,
          msg.target,
          json.dumps({'text': msg.text}, ensure_ascii=False),
          msg.idempotency_key,
          now,
          now,
          now,
        ))
        self.db.commit()

        row = self.db.execute(
          'select id from outbox where idempotency_key=?',
          (msg.idempotency_key,)
        ).fetchone()
        return row[0]

    def pending(self, limit=10):
        now = int(time.time())
        return self.db.execute('''
          select id, channel, target, payload, attempts
          from outbox
          where status in ('pending', 'failed') and next_attempt_at <= ?
          order by created_at asc
          limit ?
        ''', (now, limit)).fetchall()

    def mark_sent(self, id: str, receipt: dict):
        now = int(time.time())
        self.db.execute('''
          update outbox
          set status='sent', receipt=?, updated_at=?
          where id=?
        ''', (json.dumps(receipt, ensure_ascii=False), now, id))
        self.db.commit()

    def mark_failed(self, id: str, error: str, attempts: int):
        now = int(time.time())
        backoff = min(300, 2 ** attempts)
        status = 'dead' if attempts >= 5 else 'failed'
        self.db.execute('''
          update outbox
          set status=?, attempts=?, next_attempt_at=?, last_error=?, updated_at=?
          where id=?
        ''', (status, attempts + 1, now + backoff, error, now, id))
        self.db.commit()
```

Dispatcher：

```python
# learn-claude-code/dispatch_outbox.py
import json
from outbox import Outbox

async def dispatch_once(send_tool):
    outbox = Outbox()

    for id, channel, target, payload_json, attempts in outbox.pending():
        payload = json.loads(payload_json)
        try:
            result = await send_tool(channel=channel, target=target, text=payload['text'])
            outbox.mark_sent(id, {
                'providerMessageId': result.get('message_id'),
                'providerTimestamp': result.get('timestamp'),
            })
        except Exception as e:
            outbox.mark_failed(id, str(e), attempts)
```

Agent Loop 里只 enqueue，不直接 send：

```python
outbox.enqueue(OutboxMessage(
    channel='telegram',
    target='-5115329245',
    text=lesson_text,
    idempotency_key='agent-course:236:telegram:-5115329245',
))
```

这样发送动作从“LLM 即兴操作”变成了“可靠队列消费”。

---

## 3. pi-mono：生产版中间件

在 pi-mono 这种 TypeScript 生产框架里，建议把 Outbox 做成 `ToolMiddleware`，拦截所有外部投递类工具：

```ts
// pi-mono/outbox/OutboxMiddleware.ts
import { createHash } from 'crypto';

type ToolCall = {
  name: string;
  args: Record<string, any>;
  runId: string;
};

type ToolResult = {
  ok: boolean;
  data?: any;
  error?: string;
};

function stableHash(input: unknown) {
  return createHash('sha256')
    .update(JSON.stringify(input, Object.keys(input as any).sort()))
    .digest('hex');
}

export class OutboxMiddleware {
  constructor(private outboxStore: OutboxStore) {}

  async beforeToolCall(call: ToolCall) {
    if (!['message.send', 'email.send', 'webhook.post'].includes(call.name)) {
      return { mode: 'passthrough' as const };
    }

    const idempotencyKey = call.args.idempotencyKey ?? stableHash({
      tool: call.name,
      target: call.args.target,
      payload: call.args.message ?? call.args.body,
      runId: call.runId,
    });

    const existing = await this.outboxStore.findByIdempotencyKey(idempotencyKey);
    if (existing?.status === 'sent' || existing?.status === 'verified') {
      return {
        mode: 'short_circuit' as const,
        result: {
          ok: true,
          data: {
            skipped: true,
            reason: 'already_delivered',
            receipt: existing.receipt,
          },
        },
      };
    }

    const item = await this.outboxStore.enqueue({
      tool: call.name,
      args: call.args,
      idempotencyKey,
      status: 'pending',
    });

    return {
      mode: 'short_circuit' as const,
      result: {
        ok: true,
        data: {
          queued: true,
          outboxId: item.id,
          idempotencyKey,
        },
      },
    };
  }
}
```

再由后台 worker 统一发送：

```ts
// pi-mono/outbox/OutboxDispatcher.ts
export class OutboxDispatcher {
  constructor(
    private store: OutboxStore,
    private realTools: ToolExecutor,
  ) {}

  async tick() {
    const items = await this.store.claimPending({ limit: 20, leaseMs: 30_000 });

    for (const item of items) {
      try {
        const result = await this.realTools.execute(item.tool, item.args);

        await this.store.markSent(item.id, {
          providerMessageId: result.data?.messageId,
          raw: result.data,
        });
      } catch (err) {
        await this.store.retryLater(item.id, {
          error: err instanceof Error ? err.message : String(err),
          backoffMs: exponentialBackoff(item.attempts),
        });
      }
    }
  }
}
```

生产建议：

- `claimPending` 要有 lease，防止多个 worker 同时发
- `idempotencyKey` 要唯一索引
- message payload 要存 hash，避免用户隐私长文本无限堆积
- dead-letter 要能人工 replay
- 高价值通知要 verify，低价值通知只记录 API success

---

## 4. OpenClaw 实战：课程 cron 怎么用

OpenClaw 课程 cron 本质上就是典型投递场景：

1. 生成课程文件
2. 更新 README / TOOLS
3. 提交推送 Git
4. 发 Telegram 群消息
5. 记录 messageId

这里最容易出问题的是第 4 步：消息发了，但后续记录失败；或者 cron 重试又发一次。

可以在 workspace 做一个轻量 outbox 文件：

```json
{
  "agent-course:236:telegram:-5115329245": {
    "status": "sent",
    "messageId": "10401",
    "lesson": 236,
    "sentAt": "2026-05-04T05:33:00Z"
  }
}
```

发送前先查：

```python
key = f"agent-course:{lesson_no}:telegram:{group_id}"
if outbox.get(key, {}).get('status') == 'sent':
    return outbox[key]['messageId']

result = message.send(target=group_id, message=text)
outbox[key] = {
    'status': 'sent',
    'messageId': result['messageId'],
    'lesson': lesson_no,
    'sentAt': now_iso(),
}
save(outbox)
```

如果 Telegram API 成功但本地写 outbox 失败，下一次启动仍可能重复。

更强做法：发送前先写 `sending`：

```json
{
  "agent-course:236:telegram:-5115329245": {
    "status": "sending",
    "payloadHash": "sha256:...",
    "startedAt": "2026-05-04T05:33:00Z"
  }
}
```

重启发现 `sending` 超过 10 分钟时，不要立刻重发，而是：

- 如果平台支持按最近消息查询：先 verify
- 如果不支持：升级人工或发一条带唯一编号的安全补发
- 对“可能重复造成困扰”的消息，宁可人工确认

---

## 5. 和 Effect Journal 的区别

第 229 课讲过 Effect Journal，两者很像，但关注点不同：

| 模式 | 关注点 | 典型对象 |
|---|---|---|
| Effect Journal | 所有外部副作用的事务状态 | GitHub PR、部署、付款、发消息 |
| Outbox | 投递类副作用的可靠发送 | Telegram、Email、Webhook、Push |

可以理解为：

> Outbox 是 Effect Journal 在“消息投递”领域的专门优化版本。

它更关心：

- 投递顺序
- 重试退避
- 回执保存
- 重复发送防护
- dead-letter/replay
- 平台消息长度、附件、限流

---

## 6. 常见坑

### 坑 1：成功返回不代表用户看到了

很多平台只表示“API 接收成功”，不代表用户已读。

所以状态要分层：

```text
queued -> sending -> sent -> verified -> acknowledged
```

不是所有平台都有 acknowledged，别假装有。

### 坑 2：用内容 hash 当幂等键可能误伤

两条不同业务消息内容相同，会被错误去重。

更好的 key：

```text
{workflow}:{business_id}:{channel}:{target}:{purpose}
```

例如：

```text
agent-course:236:telegram:-5115329245:lesson-post
```

### 坑 3：失败重试不区分错误类型

- 429：按 Retry-After
- 5xx/timeout：指数退避
- 400 payload too long：不要重试，先切割消息
- 403 permission denied：dead-letter + 告警

### 坑 4：Outbox 没有清理策略

Outbox 不是永久日志。

建议：

- `verified` 保留 30~90 天
- `dead` 保留到人工处理
- payload 可脱敏，只保留 hash + artifact 引用

---

## 7. Checklist

实现投递类工具时，检查这 8 点：

- [ ] 发送前先写 Outbox
- [ ] 有稳定 `idempotencyKey`
- [ ] `idempotencyKey` 有唯一索引
- [ ] Dispatcher 有 lease/claim 防并发重复发送
- [ ] 保存 provider message id / raw receipt
- [ ] 失败按错误类型 backoff
- [ ] dead-letter 可人工 replay
- [ ] 最终回复前能拿到投递证据

---

## 总结

Agent 自动化越强，越不能把“发出去了吧”当成完成。

可靠 Agent 的投递逻辑应该是：

```text
先入箱，再投递；有回执，可补发；能验收，不重复。
```

Outbox Pattern 让 Agent 的最后一步不再靠运气。尤其是 cron、通知、报表、客服回复、GitHub Issue 创建这些场景，一旦接入 Outbox，可靠性会立刻上一个台阶。
