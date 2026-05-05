# 250. Agent 意图账本与行动可追责（Intent Ledger & Action Accountability）

> 核心思想：不要只记录 Agent **做了什么工具调用**，还要记录它当时认为自己在完成什么**用户意图**。否则出了问题只能看到 `git push`、`message.send`、`deploy`，却解释不清“为什么要这么做”。

很多系统已经有 tool audit log，但 audit log 只回答：

- 谁调用了哪个工具？
- 参数是什么？
- 成功还是失败？

生产 Agent 更需要回答：

- 这个动作对应哪个用户请求？
- Agent 是怎么把请求拆成这些动作的？
- 哪些动作是必要的，哪些是它自己扩展出来的？
- 最终结果是否真的满足了原始意图？

这就是 **Intent Ledger（意图账本）**。

---

## 1. 为什么 Tool Audit Log 不够

假设日志里有一条：

```json
{
  "tool": "git.push",
  "args": { "remote": "origin", "branch": "main" },
  "status": "success"
}
```

这条日志只能证明“推了代码”。但事故复盘时真正要问的是：

- 用户是否真的要求 push？
- push 前是否完成测试？
- push 的内容是否对应用户要的功能？
- 有没有越权做了额外修改？

所以需要把 **用户意图 → 计划 → 工具动作 → 结果证据** 串起来。

---

## 2. Intent Ledger 数据结构

一个最小可用的账本条目：

```ts
type IntentLedgerEntry = {
  intentId: string;
  runId: string;
  actor: {
    userId: string;
    channel: "telegram" | "web" | "cron" | "api";
    authority: "owner" | "member" | "guest" | "system";
  };
  userRequest: string;
  normalizedIntent: string;
  scope: string[];
  risk: "low" | "medium" | "high" | "critical";
  plannedActions: Array<{
    actionId: string;
    tool: string;
    reason: string;
    expectedOutcome: string;
  }>;
  evidence: Array<{
    source: string;
    observedAt: string;
    summary: string;
  }>;
  status: "planned" | "executing" | "completed" | "failed" | "cancelled";
  outcome?: string;
};
```

重点字段：

- `normalizedIntent`：把自然语言请求翻译成稳定、可审计的任务表达。
- `scope`：这次任务允许触碰的边界，例如 `repo:agent-course`、`telegram:rust-group`。
- `plannedActions.reason`：每个动作必须解释它服务于哪个意图。
- `evidence`：用于证明动作前提成立，例如当前 git 分支、PR 状态、测试结果。

---

## 3. learn-claude-code：Python 教学版

教学版可以用 JSONL 做追加写账本：

```python
# intent_ledger.py
import json
import time
import uuid
from pathlib import Path
from typing import Literal, TypedDict

class PlannedAction(TypedDict):
    actionId: str
    tool: str
    reason: str
    expectedOutcome: str

class IntentEntry(TypedDict):
    intentId: str
    runId: str
    userRequest: str
    normalizedIntent: str
    scope: list[str]
    risk: Literal["low", "medium", "high", "critical"]
    plannedActions: list[PlannedAction]
    status: str
    createdAt: float

class IntentLedger:
    def __init__(self, path: str = "memory/intent-ledger.jsonl"):
        self.path = Path(path)
        self.path.parent.mkdir(parents=True, exist_ok=True)

    def create(
        self,
        *,
        run_id: str,
        user_request: str,
        normalized_intent: str,
        scope: list[str],
        risk: Literal["low", "medium", "high", "critical"],
        planned_actions: list[PlannedAction],
    ) -> IntentEntry:
        entry: IntentEntry = {
            "intentId": str(uuid.uuid4()),
            "runId": run_id,
            "userRequest": user_request,
            "normalizedIntent": normalized_intent,
            "scope": scope,
            "risk": risk,
            "plannedActions": planned_actions,
            "status": "planned",
            "createdAt": time.time(),
        }
        self.append(entry)
        return entry

    def append(self, entry: dict):
        with self.path.open("a", encoding="utf-8") as f:
            f.write(json.dumps(entry, ensure_ascii=False) + "\n")

    def mark(self, intent_id: str, status: str, outcome: str | None = None):
        self.append({
            "type": "status_update",
            "intentId": intent_id,
            "status": status,
            "outcome": outcome,
            "updatedAt": time.time(),
        })
```

Agent Loop 里这样用：

```python
ledger = IntentLedger()

intent = ledger.create(
    run_id=run_id,
    user_request="给 Rust 学习小组发一节 Agent 课并提交 repo",
    normalized_intent="publish_agent_course_lesson",
    scope=["repo:agent-course", "telegram:-5115329245"],
    risk="medium",
    planned_actions=[
        {
            "actionId": "write_lesson",
            "tool": "file.write",
            "reason": "保存课程正文，满足持久化要求",
            "expectedOutcome": "lessons/250-*.md exists",
        },
        {
            "actionId": "send_group",
            "tool": "telegram.sendMessage",
            "reason": "向学习小组交付课程内容",
            "expectedOutcome": "provider message id returned",
        },
        {
            "actionId": "git_push",
            "tool": "git.push",
            "reason": "同步课程仓库，满足 repo 更新要求",
            "expectedOutcome": "remote contains commit sha",
        },
    ],
)

try:
    # execute actions...
    ledger.mark(intent["intentId"], "completed", "lesson published and pushed")
except Exception as exc:
    ledger.mark(intent["intentId"], "failed", str(exc))
    raise
```

这比普通日志多了一层：每个动作都必须能回到原始 intent。

---

## 4. pi-mono：TypeScript 生产中间件

生产版建议把它做成 Agent middleware：

```ts
// intent-ledger-middleware.ts
import { z } from "zod";

const PlannedActionSchema = z.object({
  actionId: z.string(),
  tool: z.string(),
  reason: z.string().min(8),
  expectedOutcome: z.string().min(8),
});

const IntentPlanSchema = z.object({
  normalizedIntent: z.string(),
  scope: z.array(z.string()).min(1),
  risk: z.enum(["low", "medium", "high", "critical"]),
  plannedActions: z.array(PlannedActionSchema).min(1),
});

type IntentLedgerStore = {
  create(entry: unknown): Promise<{ intentId: string }>;
  bindToolCall(input: {
    intentId: string;
    actionId: string;
    toolCallId: string;
    toolName: string;
  }): Promise<void>;
  complete(intentId: string, outcome: string): Promise<void>;
  fail(intentId: string, error: string): Promise<void>;
};

export function intentLedgerMiddleware(store: IntentLedgerStore) {
  return async (ctx: AgentContext, next: () => Promise<AgentResult>) => {
    const plan = IntentPlanSchema.parse(await ctx.planner.makePlan(ctx.messages));

    const { intentId } = await store.create({
      runId: ctx.runId,
      actor: ctx.actor,
      userRequest: ctx.latestUserMessage,
      ...plan,
      status: "planned",
      createdAt: new Date().toISOString(),
    });

    ctx.intentId = intentId;
    ctx.allowedPlannedActions = new Map(
      plan.plannedActions.map((action) => [action.tool, action])
    );

    try {
      const result = await next();
      await store.complete(intentId, result.summary ?? "completed");
      return result;
    } catch (error) {
      await store.fail(intentId, error instanceof Error ? error.message : String(error));
      throw error;
    }
  };
}
```

工具分发层再强制绑定：

```ts
export async function dispatchTool(ctx: AgentContext, call: ToolCall) {
  const planned = ctx.allowedPlannedActions?.get(call.name);

  if (!planned) {
    throw new Error(
      `Tool ${call.name} is not linked to any planned intent action. ` +
      `Create or revise the intent plan before executing.`
    );
  }

  await ctx.intentLedger.bindToolCall({
    intentId: ctx.intentId,
    actionId: planned.actionId,
    toolCallId: call.id,
    toolName: call.name,
  });

  return ctx.toolRegistry.execute(call.name, call.args);
}
```

这样做有一个很实际的收益：

> Agent 临时“手痒”想多调一个工具时，分发层会问：这个工具调用服务于哪个已登记动作？答不上来就不能执行。

---

## 5. OpenClaw 实战：Cron 课程任务怎么记账

OpenClaw 里很多任务来自 Cron、Heartbeat、Telegram，不一定是人工实时对话。Intent Ledger 对这些任务尤其重要。

例如本课程 Cron 的账本可以这样写：

```json
{
  "intentId": "course-2026-05-06-0930",
  "source": "cron:agent-course-every-3h",
  "normalizedIntent": "publish_agent_course_lesson",
  "scope": [
    "workspace:/agent-course",
    "telegram:-5115329245",
    "github:gfwfail/agent-course"
  ],
  "risk": "medium",
  "plannedActions": [
    "check_duplicate_topics",
    "write_lesson_file",
    "update_readme",
    "send_telegram_lesson",
    "update_tools_memory",
    "commit_and_push"
  ],
  "completionGates": [
    "lesson_file_exists",
    "readme_contains_lesson",
    "tools_contains_topic",
    "telegram_message_id_recorded",
    "git_remote_contains_commit"
  ]
}
```

注意：这不是替代 audit log，而是 audit log 的上游索引。

- Audit log：工具调用事实。
- Intent ledger：这些事实为什么存在。
- Completion gates：这些事实是否满足任务完成标准。

三者组合起来，Agent 才真正可追责。

---

## 6. 和已有模式的区别

| 模式 | 解决什么问题 | Intent Ledger 补充什么 |
|---|---|---|
| Tool Audit Log | 记录工具调用事实 | 记录调用背后的用户意图 |
| Decision Provenance | 追踪结论从哪些证据来 | 追踪行动从哪个任务目标来 |
| Effect Journal | 让副作用可恢复 | 解释副作用为什么被允许执行 |
| Completion Gates | 判断任务是否完成 | 给 gates 绑定原始意图和计划动作 |
| Policy-as-Code | 判断动作是否合规 | 给 policy 提供 intent/scope/risk 上下文 |

一句话：

> Audit Log 让你知道 Agent 做过什么；Intent Ledger 让你知道它为什么认为自己应该这么做。

---

## 7. 生产落地 Checklist

1. **每个 run 必须有 intentId**：没有 intent，不允许执行中高风险工具。
2. **每个工具调用必须绑定 actionId**：工具调用不能悬空。
3. **scope 必须可机器校验**：不要只写“处理项目”，要写 `repo:x/y`、`channel:id`。
4. **计划变更必须追加记录**：不要覆盖旧计划，要 append `plan_revised` 事件。
5. **外部副作用前检查 intent 状态**：cancelled/failed 的 intent 不能继续执行。
6. **最终回复引用 intent outcome**：完成不是一句“好了”，要有 evidence。
7. **保留低成本查询入口**：支持按用户、时间、repo、channel、tool 反查。

---

## 8. 常见坑

### 坑 1：只在任务结束后总结

任务结束后再让 LLM 总结“我为什么这么做”，很容易美化历史。

正确做法：执行前写 `plannedActions`，执行中追加事件，执行后只写 outcome。

### 坑 2：intent 太泛

`help_user`、`fix_bug` 这种 intent 没有审计价值。

更好的 intent：

```text
publish_agent_course_lesson_250_to_rust_group
fix_airalo_topup_without_exposing_raw_provider_payload
```

越具体，越容易判断是否越界。

### 坑 3：账本不可查询

JSONL 很适合教学，但生产要能查：

- 这个用户昨天触发了哪些高风险动作？
- 这个 repo 最近哪些 intent 导致 push？
- 哪些工具调用没有绑定 intent？

所以 pi-mono 生产版建议落 PostgreSQL / SQLite / Redis Stream。

---

## 总结

成熟 Agent 的可追责链路应该是：

```text
User Request
  -> Normalized Intent
  -> Scoped Plan
  -> Planned Action
  -> Tool Call
  -> Evidence
  -> Outcome
  -> Completion Gate
```

如果只记录工具调用，Agent 像一个“会留下脚印的黑盒”。

如果记录 Intent Ledger，Agent 就变成一个可以复盘、可以审计、可以持续改进的工程系统。

下一步可以把 Intent Ledger 接到 Policy-as-Code：高风险工具不仅看 `tool + args`，还要看 `intent + scope + actor + evidence`，这才是真正生产级的 Agent 治理。
