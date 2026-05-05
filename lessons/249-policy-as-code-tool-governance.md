# 249. Agent Policy-as-Code 与工具治理（Policy-as-Code & Tool Governance）

> 一句话：不要把“哪些工具能用、什么参数危险、什么时候要审批”写死在 prompt 里；把它们做成可测试、可版本化、可审计的 Policy-as-Code。

前几课我们讲了密钥边界、权限委托、吊销、环境指纹。今天再往上抽一层：**工具治理**。

生产 Agent 最危险的不是它不会调用工具，而是：

- 工具太多，LLM 看见了不该看的能力；
- 同一个工具在私聊、群聊、cron、sub-agent 里风险不同；
- 参数看起来正常，但组合起来就是危险操作；
- 政策散落在 prompt、if 判断、README 和人工习惯里，没人知道最终规则是什么。

Policy-as-Code 的目标是把这些规则收束成一个工程化入口：

```text
intent + actor + channel + tool + args + environment
        -> policy decision: allow / deny / require_approval / dry_run
```

---

## 1. 核心模型：Policy Decision 不只是 true/false

不要只返回 boolean。一个成熟的策略引擎至少返回：

```json
{
  "effect": "require_approval",
  "risk": "high",
  "reasons": ["git_push targets protected branch", "external side effect"],
  "requiredApprover": "owner",
  "constraints": {
    "maxRetries": 0,
    "mustRevalidateBeforeExecute": true
  }
}
```

推荐四种 effect：

| effect | 含义 | 例子 |
|---|---|---|
| `allow` | 可直接执行 | 读文件、查状态、跑只读 API |
| `deny` | 明确禁止 | 群聊读取私密记忆、未授权部署生产 |
| `require_approval` | 需要人工确认 | push、发邮件、删除资源、付款 |
| `dry_run` | 只能演练，不真实执行 | 第一次跑运维 runbook、未知环境变更 |

重点：**Policy 运行在工具执行层，不依赖 LLM 自觉遵守。**

---

## 2. learn-claude-code：Python 教学版

教学版可以先用纯函数表达策略，方便单元测试：

```python
from dataclasses import dataclass
from typing import Literal

Effect = Literal["allow", "deny", "require_approval", "dry_run"]
Risk = Literal["low", "medium", "high", "critical"]

@dataclass
class ToolRequest:
    actor: str
    channel: str       # private / group / cron / subagent
    tool: str
    args: dict
    scopes: set[str]

@dataclass
class PolicyDecision:
    effect: Effect
    risk: Risk
    reasons: list[str]

class PolicyEngine:
    def decide(self, req: ToolRequest) -> PolicyDecision:
        if req.channel == "group" and req.tool == "read_memory":
            return PolicyDecision("deny", "critical", ["group cannot read private memory"])

        if req.tool == "git_push":
            branch = req.args.get("branch", "")
            if branch in {"main", "master"}:
                return PolicyDecision("require_approval", "high", ["push to protected branch"])
            return PolicyDecision("require_approval", "medium", ["external repository write"])

        if req.tool.startswith("delete_"):
            return PolicyDecision("require_approval", "high", ["destructive operation"])

        if req.tool.startswith("read_"):
            return PolicyDecision("allow", "low", ["read-only tool"])

        return PolicyDecision("dry_run", "medium", ["unknown tool defaults to dry-run"])
```

然后在 Tool Dispatcher 里统一拦截：

```python
class ToolDispatcher:
    def __init__(self, policy: PolicyEngine):
        self.policy = policy

    def call(self, req: ToolRequest):
        decision = self.policy.decide(req)

        if decision.effect == "deny":
            raise PermissionError("DENIED: " + "; ".join(decision.reasons))

        if decision.effect == "require_approval":
            return {"status": "waiting_approval", "decision": decision}

        if decision.effect == "dry_run":
            return self.dry_run(req)

        return self.execute(req.tool, req.args)
```

这样 LLM 即使说“直接 push main”，Dispatcher 也不会直接执行。

---

## 3. pi-mono：TypeScript 生产版 Policy Middleware

生产系统里建议把 policy 做成中间件，输出结构化 decision，后续审批、审计、执行前复核都复用它：

```ts
type Effect = "allow" | "deny" | "require_approval" | "dry_run";
type Risk = "low" | "medium" | "high" | "critical";

type ToolContext = {
  runId: string;
  actor: { id: string; role: "owner" | "agent" | "guest" };
  channel: "private" | "group" | "cron" | "subagent";
  toolName: string;
  args: Record<string, unknown>;
  scopes: string[];
};

type PolicyDecision = {
  effect: Effect;
  risk: Risk;
  reasons: string[];
  constraints?: {
    requireFreshObservation?: boolean;
    bindApprovalToPlanHash?: boolean;
    maxRetries?: number;
  };
};

export interface PolicyRule {
  name: string;
  match(ctx: ToolContext): boolean;
  decide(ctx: ToolContext): PolicyDecision;
}

export class PolicyEngine {
  constructor(private rules: PolicyRule[]) {}

  decide(ctx: ToolContext): PolicyDecision {
    for (const rule of this.rules) {
      if (rule.match(ctx)) return rule.decide(ctx);
    }
    return {
      effect: "dry_run",
      risk: "medium",
      reasons: ["no matching policy rule"],
    };
  }
}
```

几条典型规则：

```ts
const rules: PolicyRule[] = [
  {
    name: "group-no-private-memory",
    match: (ctx) => ctx.channel === "group" && ctx.toolName === "read_memory",
    decide: () => ({
      effect: "deny",
      risk: "critical",
      reasons: ["group channel cannot access private memory"],
    }),
  },
  {
    name: "git-push-needs-approval",
    match: (ctx) => ctx.toolName === "git_push",
    decide: (ctx) => ({
      effect: "require_approval",
      risk: ctx.args.branch === "main" ? "high" : "medium",
      reasons: ["git push is an external write"],
      constraints: {
        requireFreshObservation: true,
        bindApprovalToPlanHash: true,
        maxRetries: 0,
      },
    }),
  },
];
```

中间件执行：

```ts
export function policyMiddleware(policy: PolicyEngine) {
  return async function beforeTool(ctx: ToolContext) {
    const decision = policy.decide(ctx);

    await auditLog.write({
      runId: ctx.runId,
      toolName: ctx.toolName,
      effect: decision.effect,
      risk: decision.risk,
      reasons: decision.reasons,
    });

    if (decision.effect === "deny") {
      throw new Error(`POLICY_DENIED:${decision.reasons.join("|")}`);
    }

    if (decision.effect === "require_approval") {
      throw new ApprovalRequiredError(decision);
    }

    if (decision.effect === "dry_run") {
      ctx.args.__dryRun = true;
    }
  };
}
```

这就把“安全规则”从 prompt 变成了系统边界。

---

## 4. OpenClaw 实战：把渠道能力、审批、记忆边界放进一套策略

OpenClaw 这种 Always-on Agent 特别适合 Policy-as-Code，因为同一个 Agent 会出现在：

- 私聊：老板权限高，可以执行更多操作；
- 群聊：不能泄露私密记忆，发言要克制；
- cron：可自动执行，但外部副作用要可审计；
- sub-agent：默认权限更小，只能拿父任务委托的能力。

可以把策略入口想成：

```text
before every tool call:
  1. 读取 channel capabilities
  2. 读取 actor / session / delegation scopes
  3. 计算 tool risk
  4. 返回 allow / deny / approval / dry-run
  5. 写 audit log
```

一个实用规则表：

```yaml
rules:
  - name: private-memory-only-in-main-private-chat
    when:
      tool: read_memory
      channel_not: private
    effect: deny
    risk: critical

  - name: external-message-send
    when:
      tool: message.send
    effect: require_approval
    except:
      - cron_lesson_group_whitelisted
    risk: medium

  - name: destructive-shell-command
    when:
      tool: exec
      args_match: "rm -rf|drop database|kubectl delete"
    effect: require_approval
    risk: critical

  - name: unknown-tool-default
    when:
      tool: "*"
    effect: dry_run
    risk: medium
```

注意最后一条：**未知默认 dry-run**，不要未知默认 allow。

---

## 5. 常见坑

1. **只在 prompt 写规则**  
   Prompt 是提示，不是边界。真正边界必须在工具层。

2. **Policy 只返回 allow/deny**  
   生产里大量场景不是非黑即白，而是“可执行但要审批/复核/降级”。

3. **规则不可测试**  
   Policy-as-Code 的价值之一就是能写测试：群聊读 MEMORY 必须 deny，push main 必须 approval。

4. **没有审计 decision**  
   事故复盘时不仅要知道工具有没有执行，还要知道为什么当时允许/拒绝。

5. **默认允许未知工具**  
   新工具上线时最容易漏规则。默认 dry-run 或 deny 更安全。

---

## 6. 最小落地清单

如果你在做自己的 Agent 框架，先实现这 5 件事：

- `PolicyDecision { effect, risk, reasons, constraints }`
- `PolicyEngine.decide(ctx)` 纯函数，可单测
- Tool Dispatcher 执行前强制调用 policy
- `require_approval` 和 `dry_run` 作为一等结果处理
- 每次 decision 写入审计日志

成熟 Agent 的安全感不是“模型应该不会乱来”，而是：

```text
模型可以提议任何事，但工具层只执行 Policy 允许的事。
```

这就是 Policy-as-Code 的核心。
