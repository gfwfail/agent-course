# 223. Agent 会话状态迁移与兼容升级（Session State Migration & Compatibility Upgrade）

> 关键词：state schema、migration、backward compatibility、long-running session、checkpoint upgrade

上一课讲了 **答案溯源与引用校验**：Agent 的关键结论必须能追到证据，没有证据的结论默认只是猜测。

今天讲一个生产里很容易被忽略、但一出问题就很痛的点：**Agent 的会话状态怎么升级？**

普通 Web 服务升级数据库有 migration；Agent 也一样。因为 Agent 不只存用户消息，还会存：

- 当前任务目标、计划、进度；
- 工具调用结果、checkpoint、租约；
- 用户偏好、权限快照、通知策略；
- 子 Agent 状态、未完成表单、待审批操作。

如果代码升级了，旧 session 还在跑，而状态结构变了，就会出现经典事故：

```txt
新代码期待 state.goal.title
旧状态只有 state.task
→ 读取 undefined
→ Agent 误以为没有目标
→ 开始乱问、重复执行、甚至重跑副作用
```

所以今天的核心是：**Agent State 也要有 schemaVersion，也要有 migration pipeline。**

一句话：**长会话 Agent 的升级，不是“部署新代码”这么简单，而是“带着旧记忆安全醒来”。**

---

## 1. 为什么 Agent 状态迁移比普通业务更麻烦？

数据库 migration 通常处理结构化表；Agent state 更像一个“半结构化大脑”：

```json
{
  "messages": [],
  "task": "帮我查 AWS 账单",
  "toolResults": {},
  "scratchpad": "...",
  "pendingApproval": null
}
```

问题在于：

1. **状态由 LLM 和工具共同写入**：字段可能不稳定。
2. **长任务跨版本运行**：任务开始时 v1，恢复时已经 v3。
3. **副作用不能重复**：迁移不能让已执行步骤被误判为未执行。
4. **记忆会注入 prompt**：旧字段语义错了，会污染新模型决策。
5. **多实例并发恢复**：迁移本身也要幂等、可重试。

反模式是靠 `if oldField exists` 到处兼容：

```ts
const goal = state.goal?.title ?? state.task ?? state.objective ?? '';
```

这会让兼容逻辑散落全项目，三个月后没人知道哪个字段才是真的。

正确做法：**读取状态的入口统一升级到最新版本；业务代码只面对 CurrentState。**

---

## 2. 核心设计：State Envelope + Migration Registry

把所有持久化状态外面包一层 envelope：

```ts
type StateEnvelope<T> = {
  sessionId: string;
  schemaVersion: number;
  appVersion: string;
  savedAt: string;
  checksum: string;
  state: T;
};
```

然后维护一个按版本递增的迁移表：

```txt
v1 → v2：task 字符串改成 goal 对象
v2 → v3：toolResults 数组改成按 callId 索引
v3 → v4：pendingApproval 增加 riskLevel/idempotencyKey
```

读取流程：

```txt
load envelope
  → verify checksum
  → while schemaVersion < CURRENT_VERSION: run migration
  → validate CurrentState schema
  → save upgraded state（可选 lazy write-back）
  → return CurrentState
```

关键原则：

- migration 必须 **纯函数**：输入旧 state，输出新 state，不做外部副作用；
- migration 必须 **幂等**：重复跑不会越迁越坏；
- migration 后必须 **schema validate**：失败进 DLQ，不要半残状态继续跑；
- 业务代码禁止读旧版本字段。

---

## 3. learn-claude-code：Python 教学版

教学版用 dataclass + dict migration 就够了。重点是让 Agent Loop 在恢复 session 前自动升级状态。

```python
# learn-claude-code/state_migrations.py
from dataclasses import dataclass
from typing import Any, Callable
import json

CURRENT_SCHEMA_VERSION = 4
Migration = Callable[[dict[str, Any]], dict[str, Any]]


@dataclass
class StateEnvelope:
    session_id: str
    schema_version: int
    state: dict[str, Any]


def v1_to_v2(state: dict[str, Any]) -> dict[str, Any]:
    # v1: {"task": "查账单"}
    # v2: {"goal": {"title": "查账单", "status": "active"}}
    if "goal" not in state:
        state["goal"] = {
            "title": state.pop("task", ""),
            "status": "active",
        }
    return state


def v2_to_v3(state: dict[str, Any]) -> dict[str, Any]:
    # v2: toolResults = [{callId, result}]
    # v3: toolResultsById = {callId: result}
    if "toolResultsById" not in state:
        by_id = {}
        for item in state.pop("toolResults", []):
            call_id = item.get("callId")
            if call_id:
                by_id[call_id] = item.get("result")
        state["toolResultsById"] = by_id
    return state


def v3_to_v4(state: dict[str, Any]) -> dict[str, Any]:
    approval = state.get("pendingApproval")
    if approval and "riskLevel" not in approval:
        approval["riskLevel"] = "medium"
        approval["idempotencyKey"] = f"approval:{approval.get('toolCallId', 'unknown')}"
    return state


MIGRATIONS: dict[int, Migration] = {
    1: v1_to_v2,
    2: v2_to_v3,
    3: v3_to_v4,
}


def migrate(envelope: StateEnvelope) -> StateEnvelope:
    state = dict(envelope.state)
    version = envelope.schema_version

    while version < CURRENT_SCHEMA_VERSION:
        migration = MIGRATIONS.get(version)
        if not migration:
            raise RuntimeError(f"missing migration v{version} -> v{version + 1}")
        state = migration(state)
        version += 1

    validate_current_state(state)
    return StateEnvelope(envelope.session_id, version, state)


def validate_current_state(state: dict[str, Any]) -> None:
    if not isinstance(state.get("goal"), dict):
        raise ValueError("state.goal must be object")
    if not isinstance(state.get("toolResultsById", {}), dict):
        raise ValueError("state.toolResultsById must be object")
```

Agent Loop 入口：

```python
async def resume_session(session_id: str, store):
    raw = await store.load(session_id)
    envelope = StateEnvelope(**json.loads(raw))

    upgraded = migrate(envelope)

    if upgraded.schema_version != envelope.schema_version:
        # lazy write-back：读时升级，成功校验后再写回
        await store.save(session_id, json.dumps(upgraded.__dict__))

    return AgentState(**upgraded.state)
```

教学版要记住一句话：**恢复 session 的第一步不是继续跑，而是确认这个“脑子”是当前版本能理解的脑子。**

---

## 4. pi-mono：TypeScript 生产版

生产版建议把迁移做成 `StateStore` 的透明能力，而不是散落在 Agent 业务逻辑里。

### 4.1 Zod 定义当前状态

```ts
// pi-mono/agent/state/current-state.ts
import { z } from 'zod';

export const CURRENT_STATE_VERSION = 4 as const;

export const CurrentAgentStateSchema = z.object({
  goal: z.object({
    title: z.string(),
    status: z.enum(['active', 'paused', 'done', 'failed']),
  }),
  plan: z.array(z.object({
    id: z.string(),
    text: z.string(),
    status: z.enum(['pending', 'running', 'done', 'failed']),
  })).default([]),
  toolResultsById: z.record(z.string(), z.unknown()).default({}),
  pendingApproval: z.object({
    toolCallId: z.string(),
    riskLevel: z.enum(['low', 'medium', 'high', 'critical']),
    idempotencyKey: z.string(),
  }).nullable().default(null),
});

export type CurrentAgentState = z.infer<typeof CurrentAgentStateSchema>;
```

### 4.2 Migration Registry

```ts
// pi-mono/agent/state/migrations.ts
import { CurrentAgentStateSchema, CURRENT_STATE_VERSION } from './current-state';

type UnknownState = Record<string, unknown>;
type Migration = (state: UnknownState) => UnknownState;

const migrations: Record<number, Migration> = {
  1: (state) => ({
    ...state,
    goal: state.goal ?? {
      title: typeof state.task === 'string' ? state.task : '',
      status: 'active',
    },
    task: undefined,
  }),

  2: (state) => {
    const list = Array.isArray(state.toolResults) ? state.toolResults : [];
    const toolResultsById = Object.fromEntries(
      list
        .filter((x: any) => x?.callId)
        .map((x: any) => [x.callId, x.result]),
    );

    return {
      ...state,
      toolResults: undefined,
      toolResultsById: state.toolResultsById ?? toolResultsById,
    };
  },

  3: (state) => {
    const approval: any = state.pendingApproval;
    if (!approval) return state;

    return {
      ...state,
      pendingApproval: {
        ...approval,
        riskLevel: approval.riskLevel ?? 'medium',
        idempotencyKey: approval.idempotencyKey ?? `approval:${approval.toolCallId ?? 'unknown'}`,
      },
    };
  },
};

export function migrateState(input: {
  schemaVersion: number;
  state: UnknownState;
}) {
  let version = input.schemaVersion;
  let state = structuredClone(input.state);

  while (version < CURRENT_STATE_VERSION) {
    const migration = migrations[version];
    if (!migration) {
      throw new Error(`Missing state migration ${version} -> ${version + 1}`);
    }
    state = migration(state);
    version += 1;
  }

  return {
    schemaVersion: version,
    state: CurrentAgentStateSchema.parse(state),
  };
}
```

注意这里的业务代码只拿 `CurrentAgentState`，完全不知道 v1/v2/v3 的存在。

### 4.3 StateStore 透明升级

```ts
// pi-mono/agent/state/state-store.ts
export class MigratingStateStore {
  constructor(private inner: RawStateStore) {}

  async load(sessionId: string): Promise<CurrentAgentState> {
    const envelope = await this.inner.load(sessionId);
    const upgraded = migrateState({
      schemaVersion: envelope.schemaVersion,
      state: envelope.state,
    });

    if (upgraded.schemaVersion !== envelope.schemaVersion) {
      await this.inner.save({
        ...envelope,
        schemaVersion: upgraded.schemaVersion,
        state: upgraded.state,
        migratedFrom: envelope.schemaVersion,
        migratedAt: new Date().toISOString(),
      });
    }

    return upgraded.state;
  }
}
```

生产里再加三件事：

- **migration test**：每个旧版本 fixture 都能迁到当前版本；
- **metric**：记录 `state_migration_total{from,to,status}`；
- **DLQ**：迁移失败的 session 不要继续跑，写入失败队列等待人工/自动修复。

---

## 5. OpenClaw 实战：文件记忆也要版本化

OpenClaw 里很多状态是文件形态：

- `MEMORY.md`
- `memory/YYYY-MM-DD.md`
- `memory/*.json`
- `HEARTBEAT.md`
- TaskFlow state 文件

Markdown 可以人读，但机器读的 JSON 最好加版本：

```json
{
  "schemaVersion": 2,
  "lastChecks": {
    "email": 1777720000,
    "calendar": 1777710000
  },
  "noiseBudget": {
    "dailyLimit": 5,
    "sentToday": 1
  }
}
```

Heartbeat 读取时先迁移：

```ts
const state = await loadJson('memory/heartbeat-state.json');
const current = migrateHeartbeatState(state);

if (current.schemaVersion !== state.schemaVersion) {
  await writeJsonAtomic('memory/heartbeat-state.json', current);
}
```

这能避免一种很烦的情况：你升级了 Heartbeat 逻辑，加了 `noiseBudget`，但旧文件没有这个字段，于是凌晨开始无限提醒或完全不提醒。

---

## 6. Checklist

做 Agent 状态升级时，至少检查这 8 条：

1. 每个持久化状态都有 `schemaVersion`。
2. 读取入口统一迁移到当前版本。
3. migration 是纯函数，不执行外部副作用。
4. migration 有 fixture 测试，覆盖每个历史版本。
5. 迁移后用 schema 校验，不让半残状态进入 Agent Loop。
6. 迁移失败进入 DLQ，有 sessionId、fromVersion、error、rawState 摘要。
7. 副作用步骤有 idempotencyKey，避免迁移后重复执行。
8. 新字段有安全默认值，尤其是权限、审批、通知相关字段。

---

## 7. 一句话总结

**Agent 的状态就是它的记忆。代码升级时如果不迁移记忆，新脑子读旧日记，很容易读错、漏读、重复干活。**

生产级 Agent 必须把 state migration 当成基础设施：

```txt
schemaVersion + migration registry + validation + DLQ + tests
```

做到这一步，长会话、长任务、跨版本恢复才真正可靠。否则每次部署都像给 Agent 做失忆手术。 
