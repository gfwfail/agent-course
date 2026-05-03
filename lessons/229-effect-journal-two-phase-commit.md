# 229. Agent 副作用日记与两阶段提交（Effect Journal & Two-Phase Commit）

> 核心观点：Agent 最危险的不是“想错”，而是**外部副作用做到一半，自己忘了做到哪一步**。  
> 发邮件、扣款、发消息、创建 PR、部署服务，都应该先写 Effect Journal，再执行，再验证。

---

## 1. 为什么需要 Effect Journal？

很多 Agent Demo 的工具调用是这样的：

```text
LLM 决定发消息 → message.send → 成功/失败 → 继续
```

这在本地演示没问题，但生产里会遇到：

- 网络超时：消息其实发出去了，但工具返回 timeout
- Agent 重启：不知道刚才是否已经执行
- 用户重复触发：同一个请求被执行两次
- Sub-agent 接管：新 worker 没有旧 worker 的上下文
- 外部 API 无幂等：重复调用会真的重复扣款/发信/部署

所以生产 Agent 需要把外部副作用变成一个可恢复状态机：

```text
prepare → commit → verify → done
```

这就是 Effect Journal：**副作用执行账本**。

---

## 2. Effect Journal 数据模型

一个副作用记录至少包含：

```ts
type EffectStatus =
  | 'prepared'   // 已登记，尚未执行
  | 'committing' // 正在执行外部调用
  | 'committed'  // 外部调用返回成功
  | 'verified'   // 已通过外部查询确认
  | 'failed';    // 执行失败，等待恢复/人工处理

interface EffectRecord {
  id: string;                 // 内部 effect id
  intent: string;             // 人类可读意图
  toolName: string;           // send_telegram / create_pr / deploy_app
  argsHash: string;           // 参数 hash，防重复
  idempotencyKey: string;     // 给外部系统或自己去重
  status: EffectStatus;
  externalId?: string;        // messageId / prNumber / deploymentId
  attempts: number;
  createdAt: string;
  updatedAt: string;
  error?: string;
}
```

关键点：

- `argsHash`：同一 intent + args 不重复 prepare
- `idempotencyKey`：跨重试、跨进程稳定不变
- `externalId`：一旦拿到外部对象 ID，后续只查它，不再创建新的
- `status`：恢复时根据状态决定下一步

---

## 3. learn-claude-code：最小 Python 教学版

下面是一个文件式 Effect Journal，适合放进 `learn-claude-code` 教学项目：

```python
# effect_journal.py
import hashlib
import json
import time
import uuid
from pathlib import Path
from typing import Callable, Dict, Any

DB = Path(".agent/effects.jsonl")
DB.parent.mkdir(parents=True, exist_ok=True)


def stable_hash(value: Dict[str, Any]) -> str:
    raw = json.dumps(value, sort_keys=True, ensure_ascii=False)
    return hashlib.sha256(raw.encode()).hexdigest()[:16]


def append(record: Dict[str, Any]) -> None:
    with DB.open("a", encoding="utf-8") as f:
        f.write(json.dumps(record, ensure_ascii=False) + "\n")


def latest_by_hash(args_hash: str):
    if not DB.exists():
        return None
    found = None
    for line in DB.read_text(encoding="utf-8").splitlines():
        r = json.loads(line)
        if r.get("argsHash") == args_hash:
            found = r
    return found


def run_effect(intent: str, tool_name: str, args: Dict[str, Any], call: Callable[[str], str]):
    args_hash = stable_hash({"tool": tool_name, "args": args})
    existing = latest_by_hash(args_hash)

    # 已完成：直接返回，不重复外部副作用
    if existing and existing["status"] in ["committed", "verified"]:
        return {"deduped": True, "externalId": existing.get("externalId")}

    effect_id = existing["id"] if existing else str(uuid.uuid4())
    key = existing["idempotencyKey"] if existing else f"effect:{effect_id}"

    append({
        "id": effect_id,
        "intent": intent,
        "toolName": tool_name,
        "argsHash": args_hash,
        "idempotencyKey": key,
        "status": "prepared",
        "attempts": (existing or {}).get("attempts", 0),
        "updatedAt": time.time(),
    })

    try:
        append({"id": effect_id, "argsHash": args_hash, "status": "committing", "updatedAt": time.time()})
        external_id = call(key)  # 外部 API 调用，必须带稳定 idempotency key
        append({
            "id": effect_id,
            "argsHash": args_hash,
            "status": "committed",
            "externalId": external_id,
            "updatedAt": time.time(),
        })
        return {"deduped": False, "externalId": external_id}
    except Exception as e:
        append({"id": effect_id, "argsHash": args_hash, "status": "failed", "error": str(e), "updatedAt": time.time()})
        raise
```

使用方式：

```python
result = run_effect(
    intent="给 Rust 学习小组发送第 229 课",
    tool_name="telegram.send_message",
    args={"chat_id": -5115329245, "lesson": 229},
    call=lambda key: telegram_send(chat_id=-5115329245, text=lesson_text, idempotency_key=key),
)
```

这段代码的重点不是“完美事务”，而是：

1. 执行前一定先登记
2. 重试时能识别同一副作用
3. 成功后保存外部 ID
4. 失败后有恢复入口

---

## 4. pi-mono：生产版中间件

在 `pi-mono` 这种 TypeScript Agent Runtime 里，Effect Journal 应该做成工具分发中间件，而不是让每个工具自己手写。

```ts
import { z } from 'zod';
import { createHash } from 'crypto';

const EffectRecordSchema = z.object({
  id: z.string(),
  toolName: z.string(),
  argsHash: z.string(),
  idempotencyKey: z.string(),
  status: z.enum(['prepared', 'committing', 'committed', 'verified', 'failed']),
  externalId: z.string().optional(),
  attempts: z.number(),
});

type ToolCall = {
  name: string;
  args: unknown;
  risk: 'read' | 'write' | 'destructive';
};

type EffectStore = {
  findByArgsHash(hash: string): Promise<z.infer<typeof EffectRecordSchema> | null>;
  prepare(input: {
    toolName: string;
    argsHash: string;
    idempotencyKey: string;
  }): Promise<z.infer<typeof EffectRecordSchema>>;
  markCommitting(id: string): Promise<void>;
  markCommitted(id: string, externalId?: string): Promise<void>;
  markFailed(id: string, error: unknown): Promise<void>;
};

function hashArgs(toolName: string, args: unknown) {
  return createHash('sha256')
    .update(JSON.stringify({ toolName, args }))
    .digest('hex')
    .slice(0, 24);
}

export function withEffectJournal(store: EffectStore) {
  return async function dispatch(call: ToolCall, next: (call: ToolCall & { idempotencyKey?: string }) => Promise<any>) {
    // 读操作不需要副作用账本
    if (call.risk === 'read') return next(call);

    const argsHash = hashArgs(call.name, call.args);
    const existing = await store.findByArgsHash(argsHash);

    if (existing?.status === 'committed' || existing?.status === 'verified') {
      return {
        ok: true,
        deduped: true,
        externalId: existing.externalId,
        effectId: existing.id,
      };
    }

    const effect = existing ?? await store.prepare({
      toolName: call.name,
      argsHash,
      idempotencyKey: `effect:${argsHash}`,
    });

    try {
      await store.markCommitting(effect.id);

      const result = await next({
        ...call,
        idempotencyKey: effect.idempotencyKey,
      });

      await store.markCommitted(effect.id, result?.externalId ?? result?.id);
      return { ...result, effectId: effect.id };
    } catch (error) {
      await store.markFailed(effect.id, error);
      throw error;
    }
  };
}
```

生产版还应增加：

- 数据库唯一索引：`unique(argsHash)`
- `SELECT ... FOR UPDATE` 或 Redis lock 防并发重复 prepare
- `verify(effect)`：通过外部 API 查询 message/pr/deployment 是否真实存在
- `recover()`：扫描 `committing/failed` 状态并恢复
- `expiresAt`：长期未完成的 effect 进入人工队列

---

## 5. OpenClaw 实战：发消息、提交代码、部署都要记账

OpenClaw 里的很多动作天然是外部副作用：

- `message.send`：发 Telegram/Discord/Slack 消息
- `exec git push`：推送远程仓库
- `gh pr create`：创建 PR
- `openclaw gateway restart`：重启服务
- Laravel Cloud / Forge / Fly API 写操作

实战建议：

```text
低风险 read：直接执行
可逆 write：Effect Journal + verify
不可逆 / 高风险：Effect Journal + dry-run + approval + verify
```

例如课程 cron 发群消息：

```json
{
  "intent": "send_agent_course_lesson_229",
  "toolName": "message.send",
  "argsHash": "sha256(chat_id + lesson_id + content_hash)",
  "idempotencyKey": "agent-course:lesson:229:telegram",
  "status": "committed",
  "externalId": "telegram_message_id"
}
```

如果 Agent 在 `message.send` 后崩溃，下次恢复时应该：

1. 先查 Journal 有没有 lesson 229 的 effect
2. 如果 status 是 `committed`，不要再发
3. 如果 status 是 `committing`，优先按内容/时间窗口查 Telegram 最近消息
4. 确认不存在才重试发送

---

## 6. 两阶段提交不是分布式事务，而是“可恢复承诺”

这里的 Two-Phase Commit 不等于数据库教科书里的 XA 事务。  
Agent 场景更现实：很多外部系统不支持真正事务，所以我们追求的是：

- **执行前有记录**：不会忘
- **重复请求可去重**：不会重复做
- **执行后可验证**：不会假装成功
- **失败后可恢复**：不会卡死在半路

一句话：

> exactly-once 很难，但 effect-once 可以靠 Journal + 幂等键 + verify 尽量逼近。

---

## 7. Checklist

给任何写操作工具上线前，问 8 个问题：

- [ ] 这个工具是否会影响外部世界？
- [ ] 是否有稳定 `idempotencyKey`？
- [ ] 参数是否能生成稳定 `argsHash`？
- [ ] 执行前是否先写入 `prepared`？
- [ ] timeout 后能否查询外部系统确认结果？
- [ ] 成功后是否保存 `externalId`？
- [ ] Agent 重启后是否能恢复 `committing/failed`？
- [ ] 高风险副作用是否经过 dry-run / approval？

---

## 8. 总结

Agent 生产化的分水岭，不是会不会调用工具，而是：

> 工具调用到一半出事时，它知不知道自己做过什么、没做什么、接下来该怎么恢复。

Effect Journal 让外部副作用从“赌一把”变成“可审计、可恢复、可去重”的工程流程。

没有账本的 Agent 靠运气执行；有账本的 Agent 才能放心自动化。
