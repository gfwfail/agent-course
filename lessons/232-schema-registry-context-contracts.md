# 232. Agent Schema Registry 与上下文契约（Schema Registry & Context Contracts）

> 核心观点：Agent 系统里最容易烂掉的不是 prompt，而是**隐式数据结构**。  
> 工具返回、记忆文件、子 Agent 交接、任务状态都应该有 schema registry，否则一次字段改名就能让整条链路半夜炸掉。

---

## 1. 问题：上下文是隐形 API

传统后端里，服务之间通过 API 契约通信：

```text
POST /orders
body: { packageId, quantity }
response: { orderId, status }
```

但 Agent 系统里，经常变成这样：

```text
工具 A 返回一个 JSON
LLM 读懂后塞进 memory
子 Agent B 再从 memory 里猜字段
Cron C 三小时后继续处理
```

看起来灵活，实际很危险：

```json
// v1
{ "order_id": "123", "status": "paid" }

// v2
{ "orderId": "123", "paymentStatus": "paid" }
```

人类一眼能看懂，代码和 LLM 不一定稳定。尤其当上下文很长、字段很多、还有旧记忆混在里面时，模型会开始猜。

所以要把这些东西当成一等 API：

- Tool Result Schema
- Memory Schema
- Task State Schema
- Sub-agent Handoff Schema
- Artifact Metadata Schema

统一注册、版本化、校验、迁移。

---

## 2. Schema Registry 的最小设计

一个可用的 registry 至少需要四件事：

```ts
type SchemaEntry<T> = {
  name: string;                 // order.provisioning-result
  version: number;              // 1, 2, 3...
  parse: (input: unknown) => T;  // runtime validation
  migrate?: (old: unknown) => T; // 旧数据迁移
  describeForLLM: () => string;  // 给模型看的简短契约说明
};
```

关键点：

1. **写入时校验**：脏数据不要进上下文。
2. **读取时迁移**：旧数据醒来时自动升级。
3. **注入 LLM 时摘要化**：不要把完整 JSON Schema 塞进 prompt，只塞关键字段契约。
4. **失败要进 DLQ**：不能静默吞掉，否则 Agent 会基于半残数据继续行动。

---

## 3. learn-claude-code：Python 教学版

下面是一个极简 Schema Registry。用 `dataclass` 做结构，用 `parse_order_result()` 做运行时校验。

```python
# schema_registry.py
from dataclasses import dataclass
from typing import Any, Callable


@dataclass
class OrderProvisioningResultV2:
    order_id: str
    payment_status: str
    esim_iccid: str | None
    activation_code: str | None


@dataclass
class SchemaEntry:
    name: str
    version: int
    parse: Callable[[Any], Any]
    migrate: Callable[[Any], Any] | None
    llm_contract: str


class SchemaRegistry:
    def __init__(self):
        self.schemas: dict[str, SchemaEntry] = {}

    def register(self, entry: SchemaEntry):
        key = f"{entry.name}@v{entry.version}"
        self.schemas[key] = entry

    def parse(self, name: str, version: int, payload: Any):
        entry = self.schemas[f"{name}@v{version}"]
        return entry.parse(payload)

    def read(self, name: str, version: int, payload: Any, target_version: int):
        if version == target_version:
            return self.parse(name, version, payload)

        target = self.schemas[f"{name}@v{target_version}"]
        if not target.migrate:
            raise ValueError(f"No migration from v{version} to v{target_version}")

        return target.migrate({"version": version, "payload": payload})


def parse_order_result_v2(data: Any) -> OrderProvisioningResultV2:
    if not isinstance(data, dict):
        raise TypeError("order result must be object")

    order_id = data.get("order_id")
    payment_status = data.get("payment_status")

    if not isinstance(order_id, str) or not order_id:
        raise ValueError("order_id is required string")
    if payment_status not in ["pending", "paid", "failed", "refunded"]:
        raise ValueError("payment_status must be pending/paid/failed/refunded")

    return OrderProvisioningResultV2(
        order_id=order_id,
        payment_status=payment_status,
        esim_iccid=data.get("esim_iccid"),
        activation_code=data.get("activation_code"),
    )


def migrate_order_to_v2(old: Any) -> OrderProvisioningResultV2:
    version = old["version"]
    payload = old["payload"]

    if version == 1:
        # v1: orderId/paymentStatus camelCase
        return parse_order_result_v2({
            "order_id": payload.get("orderId"),
            "payment_status": payload.get("paymentStatus", payload.get("status")),
            "esim_iccid": payload.get("iccid"),
            "activation_code": payload.get("activationCode"),
        })

    raise ValueError(f"unsupported migration from v{version}")


registry = SchemaRegistry()
registry.register(SchemaEntry(
    name="order.provisioning_result",
    version=2,
    parse=parse_order_result_v2,
    migrate=migrate_order_to_v2,
    llm_contract=(
        "OrderProvisioningResult v2: order_id string; "
        "payment_status one of pending/paid/failed/refunded; "
        "esim_iccid optional string; activation_code optional string."
    ),
))

old_memory = {
    "schema": "order.provisioning_result",
    "version": 1,
    "payload": {"orderId": "ord_123", "paymentStatus": "paid", "iccid": "89852..."},
}

result = registry.read(
    old_memory["schema"],
    old_memory["version"],
    old_memory["payload"],
    target_version=2,
)
print(result)
```

这个例子解决的是：**旧记忆不会因为字段改名而污染新任务**。

---

## 4. pi-mono：生产版做成中间件

生产系统里建议用 Zod，把 schema registry 放到工具分发、memory store、task store 的统一边界。

```ts
import { z } from 'zod';

const OrderProvisioningResultV2 = z.object({
  order_id: z.string().min(1),
  payment_status: z.enum(['pending', 'paid', 'failed', 'refunded']),
  esim_iccid: z.string().nullable().optional(),
  activation_code: z.string().nullable().optional(),
});

type SchemaName = 'order.provisioning_result' | 'task.state' | 'handoff.summary';

type RegistryEntry<T> = {
  name: SchemaName;
  version: number;
  schema: z.ZodType<T>;
  migrate?: (input: { version: number; payload: unknown }) => T;
  llmContract: string;
};

class SchemaRegistry {
  private entries = new Map<string, RegistryEntry<any>>();

  register<T>(entry: RegistryEntry<T>) {
    this.entries.set(`${entry.name}@v${entry.version}`, entry);
  }

  parse<T>(name: SchemaName, version: number, payload: unknown): T {
    const entry = this.entries.get(`${name}@v${version}`);
    if (!entry) throw new Error(`Schema not registered: ${name}@v${version}`);
    return entry.schema.parse(payload);
  }

  migrateTo<T>(name: SchemaName, fromVersion: number, payload: unknown, targetVersion: number): T {
    if (fromVersion === targetVersion) {
      return this.parse<T>(name, targetVersion, payload);
    }

    const target = this.entries.get(`${name}@v${targetVersion}`);
    if (!target?.migrate) {
      throw new Error(`No migration for ${name}: v${fromVersion} -> v${targetVersion}`);
    }

    return target.schema.parse(target.migrate({ version: fromVersion, payload }));
  }

  contractForPrompt(name: SchemaName, version: number): string {
    const entry = this.entries.get(`${name}@v${version}`);
    if (!entry) throw new Error(`Schema not registered: ${name}@v${version}`);
    return entry.llmContract;
  }
}

export const registry = new SchemaRegistry();

registry.register({
  name: 'order.provisioning_result',
  version: 2,
  schema: OrderProvisioningResultV2,
  migrate: ({ version, payload }) => {
    if (version !== 1 || typeof payload !== 'object' || payload === null) {
      throw new Error(`Unsupported migration from v${version}`);
    }

    const p = payload as Record<string, unknown>;
    return {
      order_id: p.orderId,
      payment_status: p.paymentStatus ?? p.status,
      esim_iccid: p.iccid ?? null,
      activation_code: p.activationCode ?? null,
    };
  },
  llmContract:
    'order.provisioning_result@v2: order_id; payment_status=pending|paid|failed|refunded; optional esim_iccid; optional activation_code.',
});
```

工具返回写入 memory 前：

```ts
type VersionedPayload<T> = {
  schema: SchemaName;
  version: number;
  payload: T;
};

async function saveToolResult<T>(input: VersionedPayload<T>) {
  const parsed = registry.parse(input.schema, input.version, input.payload);

  await memoryStore.write({
    schema: input.schema,
    version: input.version,
    payload: parsed,
    observedAt: new Date().toISOString(),
  });
}
```

读取 memory 注入 LLM 前：

```ts
async function loadMemoryForPrompt(item: VersionedPayload<unknown>) {
  const latest = registry.migrateTo(
    item.schema,
    item.version,
    item.payload,
    2,
  );

  return {
    contract: registry.contractForPrompt(item.schema, 2),
    data: latest,
  };
}
```

这样 LLM 看到的不再是“随便一坨 JSON”，而是：

```text
Contract: order.provisioning_result@v2
Data: { order_id: "ord_123", payment_status: "paid", esim_iccid: "89852..." }
```

模型猜字段的概率会大幅下降。

---

## 5. OpenClaw 实战：给文件记忆加 schema 头

OpenClaw 里很多状态存在 Markdown / JSON 文件里，比如：

```text
memory/YYYY-MM-DD.md
MEMORY.md
memory/heartbeat-state.json
TaskFlow state
```

建议对机器可读文件加统一头：

```json
{
  "schema": "heartbeat.state",
  "version": 2,
  "updatedAt": "2026-05-04T03:30:00+11:00",
  "payload": {
    "lastChecks": {
      "email": 1777820000,
      "calendar": 1777810000
    },
    "noiseBudget": {
      "date": "2026-05-04",
      "sent": 2,
      "limit": 6
    }
  }
}
```

Agent 启动读取时，先做三步：

```text
1. parse JSON
2. registry.migrateTo(schema, version, payload, latestVersion)
3. validate 后再注入上下文
```

如果失败：

```text
不要让 LLM 猜。
写入 memory/dlq/schema-errors.jsonl，标记 blocked，必要时问用户。
```

这和前面讲过的 Session State Migration 不冲突：

- Migration 关注“旧状态怎么升级”
- Schema Registry 关注“所有上下文数据的契约在哪里注册、如何被统一校验”

---

## 6. 设计 Checklist

给 Agent 系统新增一种上下文数据时，问自己 7 个问题：

1. 这个数据有没有 `schema` 名字？
2. 有没有 `version`？
3. 写入前是否 runtime validate？
4. 读取旧版本时是否有 migration？
5. 失败时是否进入 DLQ，而不是让 LLM 猜？
6. 注入 prompt 时是否有简短 contract？
7. 是否有测试覆盖 v1/v2 迁移？

如果答案有 3 个以上是“没有”，这个上下文迟早会变成技术债。

---

## 7. 一句话总结

> Agent 的上下文不是草稿纸，而是一组会跨时间、跨工具、跨模型流动的 API。  
> **Schema Registry 让这些隐形 API 有名字、有版本、有校验、有迁移。**
