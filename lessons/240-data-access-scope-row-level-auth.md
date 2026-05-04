# 240. Agent 数据访问作用域与行级授权（Data Access Scope & Row-Level Authorization）

> 核心观点：Agent 不是“超级管理员”。每次工具调用都必须带着明确的 `actor / tenant / scope / purpose`，数据层再做行级过滤。权限不能只写在 prompt 里，必须落到查询和工具执行层。

很多 Agent 系统一开始会这样设计：

```text
LLM 判断用户想查订单 → 调用 search_orders(query="最近订单") → 工具返回数据库结果
```

这在 Demo 里很好用，但生产环境危险：

- 用户 A 可能通过模糊查询看到用户 B 的订单；
- 子 Agent 继承了过大的权限；
- 工具返回“太多数据”，再靠 LLM 自觉不泄漏；
- 多租户系统里忘记加 `tenant_id` 条件，一次事故就是全库泄漏。

正确做法是：**LLM 只表达意图，授权上下文由系统注入，数据层强制执行行级权限。**

---

## 1. 权限上下文：每次工具调用都带 Scope

一个最小可用的访问上下文：

```ts
type AccessScope = {
  actorId: string;        // 谁在操作：user_123 / agent_abc
  tenantId: string;       // 属于哪个租户/组织
  roles: string[];        // owner / admin / support / readonly
  scopes: string[];       // orders:read / orders:write / billing:read
  purpose: string;        // support_debug / user_self_service / cron_report
  runId: string;          // 本次 Agent run，用于审计
};
```

注意：这些字段**不应该由 LLM 生成**。它们来自登录态、会话元数据、OpenClaw channel/user 映射、后台任务配置或审批结果。

LLM 可以说：“我要查订单”。系统决定它最多只能查什么。

---

## 2. learn-claude-code：Python 教学版

先写一个简单的 `AccessScope` 和 guard：

```python
from dataclasses import dataclass
from typing import Sequence

@dataclass(frozen=True)
class AccessScope:
    actor_id: str
    tenant_id: str
    roles: Sequence[str]
    scopes: Sequence[str]
    purpose: str
    run_id: str

class PermissionDenied(Exception):
    pass

def require(scope: AccessScope, needed: str):
    if needed not in scope.scopes:
        raise PermissionDenied(f"missing scope: {needed}")
```

工具执行时，不允许调用者传 `tenant_id` 覆盖系统值：

```python
def search_orders(db, access: AccessScope, status: str | None = None, limit: int = 20):
    require(access, "orders:read")

    limit = min(limit, 50)  # 防止 LLM 一次捞太多

    sql = """
      SELECT id, status, total, created_at
      FROM orders
      WHERE tenant_id = ?
    """
    params = [access.tenant_id]

    if status:
        sql += " AND status = ?"
        params.append(status)

    sql += " ORDER BY created_at DESC LIMIT ?"
    params.append(limit)

    return db.query(sql, params)
```

关键点：

- `tenant_id` 来自 `access.tenant_id`，不是 LLM 参数；
- `orders:read` 在工具入口检查；
- `limit` 有硬上限；
- 查询结果只返回必要字段，不返回用户隐私字段。

这就是行级授权最朴素的形态。

---

## 3. pi-mono：TypeScript 生产版中间件

生产系统里，不应该每个工具都手写权限检查。可以做成工具中间件：

```ts
type ToolCall = {
  name: string;
  args: Record<string, unknown>;
};

type ToolDef = {
  name: string;
  requiredScopes?: string[];
  dataDomain?: "orders" | "billing" | "users";
  execute(args: any, ctx: ToolContext): Promise<any>;
};

type ToolContext = {
  access: AccessScope;
  audit(event: AuditEvent): Promise<void>;
};

function withAccessControl(tool: ToolDef): ToolDef {
  return {
    ...tool,
    async execute(args, ctx) {
      for (const needed of tool.requiredScopes ?? []) {
        if (!ctx.access.scopes.includes(needed)) {
          await ctx.audit({
            type: "permission_denied",
            runId: ctx.access.runId,
            actorId: ctx.access.actorId,
            tool: tool.name,
            needed,
          });
          throw new Error(`Permission denied: missing ${needed}`);
        }
      }

      // 禁止 LLM 传入系统级隔离字段
      const sanitizedArgs = { ...args };
      delete sanitizedArgs.tenantId;
      delete sanitizedArgs.actorId;

      return tool.execute(sanitizedArgs, ctx);
    },
  };
}
```

然后工具只声明自己需要什么：

```ts
export const listOrdersTool = withAccessControl({
  name: "list_orders",
  requiredScopes: ["orders:read"],
  dataDomain: "orders",
  async execute(args: { status?: string; limit?: number }, ctx) {
    return orderRepo.list({
      tenantId: ctx.access.tenantId,       // 强制从 ctx 注入
      status: args.status,
      limit: Math.min(args.limit ?? 20, 50),
    });
  },
});
```

这比在 system prompt 里写“不要查别人的订单”可靠得多。

Prompt 是提醒，权限中间件才是边界。

---

## 4. Row-Level Authorization：别只在工具层防

真正稳的设计是三层都防：

```text
Agent Loop
  ↓ 注入 AccessScope
Tool Middleware
  ↓ 检查 requiredScopes，删除危险参数
Repository / SQL Layer
  ↓ 强制 tenant_id / owner_id 条件
Database Policy（可选）
  ↓ PostgreSQL RLS / MySQL view / scoped user
```

PostgreSQL 甚至可以用 RLS：

```sql
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_orders ON orders
USING (tenant_id = current_setting('app.tenant_id'));
```

请求开始时设置：

```sql
SELECT set_config('app.tenant_id', 'tenant_123', true);
```

这样即使某个 Repository 忘记写 `WHERE tenant_id = ...`，数据库也会兜底。

---

## 5. OpenClaw 实战：群聊、私聊、Cron 权限不同

OpenClaw 这种 Always-on Agent 特别需要访问作用域：

```text
私聊老板：
  actorId = telegram:67431246
  scopes = ["repo:write", "message:send", "billing:read"]

群聊普通成员：
  actorId = telegram:xxx
  scopes = ["chat:reply", "public:search"]

Cron 课程任务：
  actorId = cron:agent-course
  scopes = ["course:write", "telegram:send:rust-group", "git:push:agent-course"]
```

这能避免两个常见坑：

1. 群聊里有人诱导 Agent 泄露老板私人记忆；
2. Cron 子任务拿到主会话过大的权限。

所以子 Agent 派发时不要只传 prompt，还要传最小 Scope：

```ts
await spawnSubAgent({
  task: "检查网站并截图",
  access: {
    actorId: "cron:site-watch",
    tenantId: "owner-bot001",
    scopes: ["browser:read", "github:issue:create"],
    purpose: "site_monitoring",
  },
});
```

---

## 6. 设计 Checklist

做生产 Agent 的数据访问时，至少检查这 8 项：

- [ ] LLM 不能自己生成 `tenantId / actorId / role`；
- [ ] 工具声明 `requiredScopes`；
- [ ] 工具中间件统一检查权限；
- [ ] Repository 强制按 `tenantId / ownerId` 过滤；
- [ ] 返回字段最小化，不把全量用户资料丢给 LLM；
- [ ] limit/pageSize 有硬上限；
- [ ] permission denied 写审计日志；
- [ ] 子 Agent 权限只能缩小，不能扩大。

---

## 7. 一句话总结

Agent 数据安全不能靠“模型乖不乖”，要靠工程边界：

```text
AccessScope 注入身份 → Tool Middleware 检查权限 → Repository 强制行级过滤 → Audit 记录拒绝与访问
```

让 LLM 负责理解意图，让系统负责决定它能看什么。生产 Agent 的安全感，来自每一层都不信任上一层。
