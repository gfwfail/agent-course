# 228. Agent 工具参数规范化与执行前校验（Tool Argument Canonicalization & Preflight Validation）

> 关键词：argument canonicalization、preflight validation、schema guard、safe defaults、tool adapter

上一课讲了 **运行时指纹**：每次 run 都要记录模型、prompt、工具 schema、配置和 gitSha，方便未来复现。

今天讲一个更贴近日常踩坑的点：**Agent 调工具前，不能把 LLM 生成的参数原样丢给工具执行。**

LLM 很擅长“理解意图”，但不擅长保证参数永远精确：

```txt
用户：把明天下午 3 点的会议提前半小时
LLM 参数：{"time": "tomorrow 14:30"}
工具需要：{"startAt": "2026-05-04T14:30:00+10:00"}

用户：查最近 7 天订单
LLM 参数：{"days": "seven", "status": "all"}
SQL 工具需要：{"from": "2026-04-26", "to": "2026-05-03", "status": null}

用户：把文件写到 lessons/228.md
LLM 参数：{"path": "~/workspace/agent-course/lessons/228.md"}
文件工具需要：绝对路径 + workspace 边界校验
```

所以生产 Agent 最好在工具执行前加一层：

**LLM args → Canonicalizer → Validator → Risk Preflight → Tool Adapter**

一句话：**LLM 负责表达意图，工具层负责把意图变成唯一、合法、可审计、可拒绝的执行参数。**

---

## 1. 为什么只靠 JSON Schema 不够？

JSON Schema 能检查“形状”，但很多生产问题不只是类型错误：

| 问题 | JSON Schema 能发现吗？ | 例子 |
|---|---:|---|
| 相对日期 | 不够 | `tomorrow` 要结合时区和当前日期 |
| 路径越界 | 不够 | `../../.ssh/id_rsa` 类型是 string，但不安全 |
| 模糊枚举 | 不够 | `urgent`、`high priority` 都可能映射到 `high` |
| 单位混乱 | 不够 | `5m` 是 5 minutes 还是 5 meters？ |
| 默认值风险 | 不够 | `deleteDays` 省略时不能默认删全部 |
| 权限交叉 | 不够 | 参数合法，但当前用户无权操作该 resource |
| 副作用预检 | 不够 | 发消息、删文件、付款都需要二次确认或 dry-run |

所以工具执行前至少要做四件事：

1. **Normalize**：把同义词、相对时间、短路径、单位转成标准形式；
2. **Validate**：用强 schema 检查必填、类型、范围、互斥字段；
3. **Authorize**：结合 user/session/task permissions 判断是否允许；
4. **Preflight**：对高风险操作做 dry-run、diff、审批或拒绝。

---

## 2. 核心模型：Canonical Tool Call

把 LLM 产出的 raw args 和最终执行 args 分开存：

```ts
type RawToolCall = {
  tool: string;
  args: unknown;
  llmCallId: string;
};

type CanonicalToolCall<TArgs> = {
  tool: string;
  rawArgs: unknown;
  args: TArgs;
  normalized: Array<{
    field: string;
    from: unknown;
    to: unknown;
    reason: string;
  }>;
  risk: "low" | "medium" | "high" | "critical";
  requiresApproval: boolean;
  idempotencyKey?: string;
};
```

注意：`rawArgs` 不丢，方便审计；真正执行只用 `args`。

---

## 3. learn-claude-code：Python 教学版

教学版先做一个简单的 `ToolArgGuard`，专门保护文件写入工具。

```python
# learn-claude-code/tool_arg_guard.py
from dataclasses import dataclass
from pathlib import Path
from typing import Any
import hashlib
import json


@dataclass
class CanonicalCall:
    tool: str
    raw_args: dict[str, Any]
    args: dict[str, Any]
    normalized: list[dict[str, Any]]
    risk: str
    requires_approval: bool
    idempotency_key: str


class ToolArgError(Exception):
    pass


class ToolArgGuard:
    def __init__(self, workspace: str):
        self.workspace = Path(workspace).resolve()

    def canonicalize_write_file(self, raw: dict[str, Any]) -> CanonicalCall:
        normalized = []

        if "path" not in raw or "content" not in raw:
            raise ToolArgError("write_file requires path and content")

        raw_path = str(raw["path"])
        path = Path(raw_path).expanduser()

        if not path.is_absolute():
            path = self.workspace / path
            normalized.append({
                "field": "path",
                "from": raw_path,
                "to": str(path),
                "reason": "relative path resolved inside workspace",
            })

        resolved = path.resolve()

        # 关键：类型是 string 不代表安全，必须做 workspace 边界校验
        if self.workspace not in [resolved, *resolved.parents]:
            raise ToolArgError(f"path escapes workspace: {resolved}")

        content = str(raw["content"])
        overwrite = bool(raw.get("overwrite", False))

        args = {
            "path": str(resolved),
            "content": content,
            "overwrite": overwrite,
        }

        risk = "medium" if overwrite else "low"
        key_src = json.dumps(args, ensure_ascii=False, sort_keys=True)
        idempotency_key = hashlib.sha256(key_src.encode()).hexdigest()[:16]

        return CanonicalCall(
            tool="write_file",
            raw_args=raw,
            args=args,
            normalized=normalized,
            risk=risk,
            requires_approval=(risk != "low"),
            idempotency_key=idempotency_key,
        )
```

Agent Loop 中不要直接执行 raw tool call：

```python
# learn-claude-code/agent_loop.py
from tool_arg_guard import ToolArgGuard, ToolArgError


guard = ToolArgGuard(workspace="/Users/bot001/.openclaw/workspace")


async def dispatch_tool(tool_name: str, raw_args: dict):
    try:
        if tool_name == "write_file":
            call = guard.canonicalize_write_file(raw_args)
        else:
            raise ToolArgError(f"unknown tool: {tool_name}")
    except ToolArgError as e:
        return {
            "ok": False,
            "error": "INVALID_TOOL_ARGUMENTS",
            "message": str(e),
            "recoverable": True,
            "suggestion": "Regenerate tool arguments with a safe workspace path and required fields.",
        }

    if call.requires_approval:
        return {
            "ok": False,
            "error": "APPROVAL_REQUIRED",
            "canonical_args": call.args,
            "normalized": call.normalized,
            "risk": call.risk,
        }

    return await write_file(**call.args)
```

这个版本虽然简单，但已经挡住三类坑：

- LLM 少填字段；
- 相对路径漂到奇怪目录；
- `../` 越界写敏感文件。

---

## 4. pi-mono：生产版 Tool Argument Pipeline

生产版建议每个工具注册时带上自己的 canonicalizer、schema、policy：

```ts
// pi-mono/packages/agent-tools/src/types.ts
import { z } from "zod";

export type ToolRisk = "low" | "medium" | "high" | "critical";

export type CanonicalizationContext = {
  now: Date;
  timezone: string;
  workspaceRoot: string;
  userId: string;
  permissions: string[];
  channel: "telegram" | "discord" | "cli" | "web";
};

export type CanonicalizedArgs<T> = {
  args: T;
  normalized: Array<{ field: string; from: unknown; to: unknown; reason: string }>;
  risk: ToolRisk;
  requiresApproval: boolean;
  idempotencyKey?: string;
};

export type ToolDefinition<TArgs, TResult> = {
  name: string;
  description: string;
  schema: z.ZodType<TArgs>;
  canonicalize: (raw: unknown, ctx: CanonicalizationContext) => CanonicalizedArgs<TArgs>;
  execute: (args: TArgs, ctx: CanonicalizationContext) => Promise<TResult>;
};
```

例如日历工具：LLM 可以说 `tomorrow 3pm`，但执行层必须变成 ISO 时间。

```ts
// pi-mono/packages/tools-calendar/src/createEvent.ts
import { z } from "zod";
import { parseDateTime } from "../time/parseDateTime";
import { sha256 } from "../crypto";

const CreateEventArgs = z.object({
  title: z.string().min(1).max(200),
  startAt: z.string().datetime(),
  endAt: z.string().datetime(),
  attendees: z.array(z.string().email()).default([]),
  calendarId: z.string().default("primary"),
});

type CreateEventArgs = z.infer<typeof CreateEventArgs>;

export const createCalendarEventTool = {
  name: "calendar.createEvent",
  description: "Create a calendar event",
  schema: CreateEventArgs,

  canonicalize(raw: any, ctx: CanonicalizationContext) {
    const normalized = [];

    const startRaw = raw.startAt ?? raw.start ?? raw.time;
    const endRaw = raw.endAt ?? raw.end;

    const startAt = parseDateTime(startRaw, {
      now: ctx.now,
      timezone: ctx.timezone,
    });

    const endAt = endRaw
      ? parseDateTime(endRaw, { now: ctx.now, timezone: ctx.timezone })
      : new Date(startAt.getTime() + 30 * 60 * 1000);

    if (startRaw !== startAt.toISOString()) {
      normalized.push({
        field: "startAt",
        from: startRaw,
        to: startAt.toISOString(),
        reason: "natural language time parsed with user timezone",
      });
    }

    if (!endRaw) {
      normalized.push({
        field: "endAt",
        from: undefined,
        to: endAt.toISOString(),
        reason: "default duration 30 minutes",
      });
    }

    const args = CreateEventArgs.parse({
      title: raw.title,
      startAt: startAt.toISOString(),
      endAt: endAt.toISOString(),
      attendees: raw.attendees ?? [],
      calendarId: raw.calendarId ?? "primary",
    });

    const idempotencyKey = sha256(JSON.stringify({ tool: "calendar.createEvent", args }));

    return {
      args,
      normalized,
      risk: args.attendees.length > 0 ? "medium" : "low",
      requiresApproval: args.attendees.length > 0,
      idempotencyKey,
    };
  },

  async execute(args: CreateEventArgs, ctx: CanonicalizationContext) {
    return ctx.calendarApi.events.create(args);
  },
};
```

核心点：**默认值也要显式记录在 `normalized` 里**。以后用户问“为什么建了 30 分钟会议”，日志里有答案。

---

## 5. Tool Dispatcher：统一入口，不允许绕过

真正重要的是：所有工具都必须走同一个 dispatcher。

```ts
// pi-mono/packages/agent-runtime/src/toolDispatcher.ts
export class ToolDispatcher {
  constructor(
    private readonly registry: ToolRegistry,
    private readonly approval: ApprovalService,
    private readonly audit: AuditLog,
  ) {}

  async call(toolName: string, rawArgs: unknown, ctx: CanonicalizationContext) {
    const tool = this.registry.get(toolName);
    if (!tool) {
      return { ok: false, error: "UNKNOWN_TOOL", message: `Tool not found: ${toolName}` };
    }

    let canonical: CanonicalizedArgs<unknown>;
    try {
      canonical = tool.canonicalize(rawArgs, ctx);
      canonical.args = tool.schema.parse(canonical.args);
    } catch (err) {
      await this.audit.write({ toolName, rawArgs, error: String(err), stage: "canonicalize" });
      return {
        ok: false,
        error: "INVALID_TOOL_ARGUMENTS",
        message: String(err),
        recoverable: true,
      };
    }

    await this.audit.write({
      toolName,
      rawArgs,
      canonicalArgs: canonical.args,
      normalized: canonical.normalized,
      risk: canonical.risk,
      stage: "preflight",
    });

    if (canonical.requiresApproval) {
      return this.approval.request({
        toolName,
        args: canonical.args,
        normalized: canonical.normalized,
        risk: canonical.risk,
        idempotencyKey: canonical.idempotencyKey,
      });
    }

    return tool.execute(canonical.args, ctx);
  }
}
```

不要让业务代码直接 `tool.execute(rawArgs)`。一旦绕过 dispatcher，schema、权限、审计、审批都会失效。

---

## 6. OpenClaw 实战：把“不确定”变成可恢复错误

OpenClaw 的工具调用也遵循同样思路：

- `browser` 要用 snapshot refs，避免旧 ref 误点；
- `exec` 高风险命令要走 approval；
- `file_write` 要明确绝对路径和 overwrite；
- `message` 外发前要区分当前聊天、群聊、外部用户；
- `sessions_spawn` 要区分 subagent 和 ACP harness。

一个好的工具错误，不应该只说 `bad args`，而要告诉 Agent 怎么恢复：

```json
{
  "ok": false,
  "error": "INVALID_TOOL_ARGUMENTS",
  "message": "path escapes workspace: /Users/bot001/.ssh/id_rsa",
  "recoverable": true,
  "suggestion": "Use a path under /Users/bot001/.openclaw/workspace and retry."
}
```

这样 LLM 下一轮能自己修正，而不是原地打转。

---

## 7. 实战 Checklist

设计工具参数层时，至少检查这 10 项：

1. **raw args 和 canonical args 分开保存**；
2. **相对时间统一转 ISO + timezone**；
3. **路径统一 resolve + workspace 边界校验**；
4. **枚举支持同义词，但最终只保留标准值**；
5. **单位显式化：秒/毫秒、美元/分、GB/bytes**；
6. **默认值必须记录 reason**；
7. **危险默认值禁止静默填充**；
8. **权限校验使用 canonical args，不使用 raw args**；
9. **高风险工具先 preflight/dry-run/approval**；
10. **错误必须 recoverable，并给 suggestion**。

---

## 8. 总结

Agent 工具调用不是“LLM 生成 JSON → 直接执行”这么简单。

生产级链路应该是：

```txt
Raw LLM Args
  → Canonicalize
  → Schema Validate
  → Permission Check
  → Risk Preflight
  → Audit Log
  → Execute
```

**LLM 可以模糊，工具不能模糊。**

把参数规范化做好，很多“Agent 偶尔乱点、乱写、乱发”的问题会直接消失一半。剩下那一半，至少也能从审计日志里知道它为什么这么做。🫡
