# 58. Event Sourcing for Agent State（事件溯源）

> 让 Agent 的每一步都可回溯、可重放、可审计

---

## 为什么 Agent 需要事件溯源？

传统 Agent 把"当前状态"存数据库：

```
tasks: [{id: 1, status: "done"}, {id: 2, status: "running"}]
```

**问题来了：**
- Task 2 为什么变成 running？什么时候变的？谁改的？
- Agent 崩溃了，能从中间继续吗？
- 想"撤销"上一步操作，怎么做？
- 生产环境出 Bug，能复现吗？

**Event Sourcing 的核心思想：**

> 不存"当前状态"，存**导致该状态的所有事件序列**。
> 状态 = 事件流的折叠（fold/reduce）结果。

```
Event 1: TaskCreated { id: 1, title: "写报告" }
Event 2: TaskStarted { id: 1, at: "09:00" }
Event 3: ToolCalled  { tool: "search", query: "AI趋势" }
Event 4: ToolResult  { tool: "search", ok: true, chars: 4200 }
Event 5: TaskDone    { id: 1, at: "09:05" }
```

当前状态 = 把这 5 个事件 reduce 出来。想知道"任何时刻的状态"，就 reduce 到那一刻。

---

## 核心概念

### Event（事件）

不可变的、带时间戳的"事实"，描述**发生了什么**：

```typescript
// pi-mono 风格
interface AgentEvent {
  id: string;           // UUID
  sessionId: string;    // 哪个 Agent 会话
  at: number;           // timestamp (ms)
  type: string;         // 事件类型
  payload: unknown;     // 事件数据
}

// 具体事件类型
type AgentEventType =
  | { type: "session.started";   payload: { model: string; systemPrompt: string } }
  | { type: "user.message";      payload: { text: string } }
  | { type: "tool.called";       payload: { name: string; args: unknown } }
  | { type: "tool.result";       payload: { name: string; ok: boolean; result: unknown; durationMs: number } }
  | { type: "assistant.message"; payload: { text: string; tokens: number } }
  | { type: "task.created";      payload: { id: string; title: string } }
  | { type: "task.completed";    payload: { id: string } }
  | { type: "session.ended";     payload: { reason: string } };
```

### Event Store（事件仓库）

只做两件事：**append（追加）** 和 **read（读取）**，从不修改、从不删除：

```typescript
class AgentEventStore {
  private events: AgentEvent[] = [];

  append(event: Omit<AgentEvent, "id" | "at">): AgentEvent {
    const e: AgentEvent = {
      id: crypto.randomUUID(),
      at: Date.now(),
      ...event,
    };
    this.events.push(e);
    this.persist(e); // 写入磁盘/DB
    return e;
  }

  read(sessionId: string, options?: { from?: number; to?: number }): AgentEvent[] {
    return this.events
      .filter(e => e.sessionId === sessionId)
      .filter(e => !options?.from || e.at >= options.from)
      .filter(e => !options?.to   || e.at <= options.to);
  }
}
```

### Projection（投影）

把事件流折叠成"视图"，用于查询：

```typescript
// 投影：某个 Session 的任务列表
interface TaskProjection {
  id: string;
  title: string;
  status: "pending" | "running" | "done";
  createdAt: number;
  completedAt?: number;
}

function projectTasks(events: AgentEvent[]): Map<string, TaskProjection> {
  const tasks = new Map<string, TaskProjection>();

  for (const e of events) {
    if (e.type === "task.created") {
      const p = e.payload as { id: string; title: string };
      tasks.set(p.id, { id: p.id, title: p.title, status: "pending", createdAt: e.at });
    } else if (e.type === "task.completed") {
      const p = e.payload as { id: string };
      const task = tasks.get(p.id);
      if (task) {
        task.status = "done";
        task.completedAt = e.at;
      }
    }
  }

  return tasks;
}
```

---

## 在 OpenClaw 中的应用

OpenClaw 已经有类似机制：每次工具调用、每次消息都是一个"事件"。
我们可以在此基础上加一层显式的 Event Store：

```typescript
// workspace/agent-course/examples/event-store.ts

import * as fs from "fs";
import * as path from "path";
import * as readline from "readline";

interface AgentEvent {
  id: string;
  sessionId: string;
  at: number;
  type: string;
  payload: unknown;
}

// JSONL（每行一个 JSON）是事件溯源的理想存储格式
// 追加高效，读取简单，人类可读
class JsonlEventStore {
  private filePath: string;

  constructor(storePath: string) {
    this.filePath = storePath;
    fs.mkdirSync(path.dirname(storePath), { recursive: true });
  }

  append(sessionId: string, type: string, payload: unknown): AgentEvent {
    const event: AgentEvent = {
      id: crypto.randomUUID(),
      sessionId,
      at: Date.now(),
      type,
      payload,
    };
    // O(1) 追加，永不重写
    fs.appendFileSync(this.filePath, JSON.stringify(event) + "\n");
    return event;
  }

  async readAll(): Promise<AgentEvent[]> {
    if (!fs.existsSync(this.filePath)) return [];

    const events: AgentEvent[] = [];
    const rl = readline.createInterface({ input: fs.createReadStream(this.filePath) });

    for await (const line of rl) {
      if (line.trim()) events.push(JSON.parse(line));
    }

    return events;
  }

  async readSession(sessionId: string): Promise<AgentEvent[]> {
    const all = await this.readAll();
    return all.filter(e => e.sessionId === sessionId);
  }
}

// 使用示例
const store = new JsonlEventStore("./agent-events.jsonl");
const sid = "session-abc-123";

// Agent 运行过程中记录事件
store.append(sid, "session.started",   { model: "claude-sonnet-4" });
store.append(sid, "user.message",      { text: "帮我分析销售数据" });
store.append(sid, "tool.called",       { name: "read_file", args: { path: "sales.csv" } });
store.append(sid, "tool.result",       { name: "read_file", ok: true, durationMs: 45 });
store.append(sid, "assistant.message", { text: "数据分析完成，月增长率 12%", tokens: 234 });
store.append(sid, "session.ended",     { reason: "completed" });

// 之后随时重建状态
const events = await store.readSession(sid);
console.log(`Session ${sid}: ${events.length} 个事件`);
```

---

## 三大杀手级特性

### 1. 时间旅行调试（Time-Travel Debugging）

```typescript
// "Agent 在 09:03 时看到了什么？"
async function replayTo(sessionId: string, timestamp: number) {
  const events = await store.readSession(sessionId);
  const snapshot = events.filter(e => e.at <= timestamp);

  // 重建该时刻的完整状态
  const tasks = projectTasks(snapshot);
  const toolCalls = snapshot.filter(e => e.type === "tool.called");
  const messages = snapshot.filter(e => e.type === "assistant.message");

  console.log(`⏪ 回到 ${new Date(timestamp).toISOString()}`);
  console.log(`任务数: ${tasks.size}, 工具调用: ${toolCalls.length}, 消息: ${messages.length}`);
  return { tasks, toolCalls, messages };
}

// 用法
await replayTo("session-abc-123", Date.parse("2026-03-15T09:03:00Z"));
```

### 2. 断点续传（Checkpoint & Resume）

```typescript
// 找到最后一个成功的 tool.result，从这里续传
async function resumeFrom(sessionId: string) {
  const events = await store.readSession(sessionId);

  // 找最后一个完整的"工具调用→结果"对
  const lastSuccessIdx = [...events].reverse().findIndex(
    e => e.type === "tool.result" && (e.payload as any).ok === true
  );

  if (lastSuccessIdx === -1) {
    console.log("从头开始");
    return null;
  }

  const resumePoint = events[events.length - 1 - lastSuccessIdx];
  console.log(`⏩ 从 ${resumePoint.at} 续传，跳过前 ${events.length - 1 - lastSuccessIdx} 个事件`);
  return resumePoint;
}
```

### 3. 审计日志（Audit Log）

```typescript
// 生成人类可读的 Agent 行为报告
async function auditReport(sessionId: string): Promise<string> {
  const events = await store.readSession(sessionId);
  const lines: string[] = [`# Agent 审计报告 - ${sessionId}\n`];

  for (const e of events) {
    const time = new Date(e.at).toISOString();
    const p = e.payload as any;

    switch (e.type) {
      case "user.message":
        lines.push(`[${time}] 👤 用户: ${p.text}`);
        break;
      case "tool.called":
        lines.push(`[${time}] 🔧 调用工具: ${p.name}(${JSON.stringify(p.args).slice(0, 80)})`);
        break;
      case "tool.result":
        lines.push(`[${time}] ${p.ok ? "✅" : "❌"} ${p.name} 返回 (${p.durationMs}ms)`);
        break;
      case "assistant.message":
        lines.push(`[${time}] 🤖 回复: ${String(p.text).slice(0, 100)}... (${p.tokens} tokens)`);
        break;
    }
  }

  return lines.join("\n");
}
```

---

## 在 learn-claude-code 中落地

Python 版（学习用）：

```python
# learn-claude-code/event_store.py

import json
import time
import uuid
from dataclasses import dataclass, asdict
from pathlib import Path
from typing import Iterator

@dataclass
class AgentEvent:
    id: str
    session_id: str
    at: float          # Unix timestamp (秒，精确到毫秒)
    type: str
    payload: dict

class EventStore:
    def __init__(self, path: str = "agent_events.jsonl"):
        self.path = Path(path)
        self.path.parent.mkdir(parents=True, exist_ok=True)

    def append(self, session_id: str, event_type: str, payload: dict) -> AgentEvent:
        event = AgentEvent(
            id=str(uuid.uuid4()),
            session_id=session_id,
            at=time.time(),
            type=event_type,
            payload=payload,
        )
        with self.path.open("a") as f:
            f.write(json.dumps(asdict(event)) + "\n")
        return event

    def read_session(self, session_id: str) -> list[AgentEvent]:
        if not self.path.exists():
            return []
        events = []
        with self.path.open() as f:
            for line in f:
                if line.strip():
                    d = json.loads(line)
                    if d["session_id"] == session_id:
                        events.append(AgentEvent(**d))
        return events

    def replay_to(self, session_id: str, timestamp: float) -> list[AgentEvent]:
        """时间旅行：重建某时刻前的状态"""
        return [e for e in self.read_session(session_id) if e.at <= timestamp]


# 与 Claude API 集成
def run_agent_with_event_sourcing(user_message: str):
    import anthropic

    store = EventStore()
    session_id = str(uuid.uuid4())
    client = anthropic.Anthropic()

    store.append(session_id, "session.started", {"model": "claude-opus-4-5"})
    store.append(session_id, "user.message", {"text": user_message})

    messages = [{"role": "user", "content": user_message}]

    while True:
        response = client.messages.create(
            model="claude-opus-4-5",
            max_tokens=4096,
            messages=messages,
        )

        # 记录 assistant 回复
        for block in response.content:
            if block.type == "text":
                store.append(session_id, "assistant.message", {
                    "text": block.text,
                    "tokens": response.usage.output_tokens,
                })

        if response.stop_reason == "end_turn":
            store.append(session_id, "session.ended", {"reason": "completed"})
            break

    return session_id
```

---

## Event Sourcing vs 传统 CRUD

| 维度 | 传统 CRUD | Event Sourcing |
|------|-----------|----------------|
| 存储 | 当前快照 | 完整历史 |
| 调试 | 难（状态已覆盖） | 易（时间旅行） |
| 审计 | 需额外实现 | 天然支持 |
| 性能 | 读快 | 写快（append-only） |
| 复杂度 | 低 | 中（需要 projection） |
| 断点续传 | 难 | 天然支持 |
| 空间占用 | 小 | 大（可压缩/归档） |

**Agent 场景下，Event Sourcing 的优势远超劣势。**
历史记录对 Agent 来说就是价值本身——它是训练数据、调试工具、审计证据。

---

## OpenClaw 的 JSONL 约定

OpenClaw 的工具调用日志本质上就是 Event Sourcing：

```bash
# 查看今天的 Agent 事件（OpenClaw 内部格式）
cat ~/.openclaw/logs/$(date +%Y-%m-%d).jsonl | jq '.type' | sort | uniq -c | sort -rn
```

你可以在此基础上扩展，把业务事件也写进来：

```typescript
// 在 OpenClaw skill 中记录业务事件
import { appendFileSync } from "fs";

const BSTORE = `${process.env.HOME}/.openclaw/workspace/events.jsonl`;

export function logEvent(type: string, payload: unknown) {
  const event = { id: crypto.randomUUID(), at: Date.now(), type, payload };
  appendFileSync(BSTORE, JSON.stringify(event) + "\n");
}

// 在工具调用前后埋点
logEvent("tool.called", { name: "send_email", to: "boss@example.com" });
const result = await sendEmail(...);
logEvent("tool.result", { name: "send_email", ok: result.ok });
```

---

## 实战建议

1. **从 JSONL 开始**，够用了。别一上来就用 Kafka/EventStore DB。
2. **事件要带足够的上下文**，"tool.called"里要存完整的 args，不然回溯时信息不足。
3. **定期归档**，超过 30 天的事件压缩成 `.jsonl.gz`，节省空间。
4. **Projection 缓存**，别每次查询都 reduce 全量事件，定期把 projection 快照存下来：

```typescript
// Snapshot 优化：每 100 个事件做一次快照
if (events.length % 100 === 0) {
  const snapshot = projectTasks(events);
  fs.writeFileSync("snapshot.json", JSON.stringify({ 
    at: Date.now(), 
    eventCount: events.length, 
    state: Object.fromEntries(snapshot) 
  }));
}
```

5. **不要把事件设计得太细**，"字符被删除" 太细了；"消息被编辑" 刚好。

---

## 小结

```
Event Sourcing = 存历史 + 现推状态
                     ↓
         调试 / 续传 / 审计 / 回溯
```

Agent 天然适合 Event Sourcing：
- Agent 的每一步工具调用都是事件
- 故障恢复需要知道"上次做到哪了"
- 生产问题需要完整的行为轨迹

**一行 `appendFileSync` 开始，然后你就有了时间机器。**

---

下一节：[59-knowledge-graph-for-agents](59-knowledge-graph-for-agents.md) — 用知识图谱给 Agent 装上结构化长期记忆
