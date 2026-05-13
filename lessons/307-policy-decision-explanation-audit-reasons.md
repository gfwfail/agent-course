# 307. Agent 策略决策解释与可审计理由（Policy Decision Explanation & Audit Reasons）

上一课讲了 **Policy Diff Budget**：shadow policy 只有通过量化预算，才能晋级成 primary policy。

但策略上线后还有一个更长期的问题：

> 当 Agent 拒绝、要求审批、降级 dry-run 时，人怎么知道它为什么这么判？

如果 `PolicyDecision` 只有 `allow/deny`，排查会非常痛苦。成熟 Agent 的策略系统必须输出 **可审计理由**：不只告诉你结果，还要告诉你触发了哪条规则、用了哪些证据、怎么补救。

---

## 1. 核心思想

Policy Explanation 不是给 LLM 写一段漂亮解释，而是给执行系统和人类审计一份结构化理由。

一个实用的策略决策至少包含：

- `action`：`allow | dry_run | require_approval | deny`；
- `risk`：本次动作风险等级；
- `reasonCodes`：稳定、可统计的机器理由，比如 `PROD_WRITE_REQUIRES_APPROVAL`；
- `evidenceRefs`：支撑判定的证据引用，比如 runbook、diff、工具 schema、用户授权；
- `constraints`：允许执行时附加的限制，比如 `maxAffectedUsers=10`；
- `remediation`：如果被拒绝，下一步怎么解锁；
- `policyVersion`：哪一版策略做出的判定。

一句话：**策略不是只负责拦截，还要负责解释自己为什么拦。**

---

## 2. learn-claude-code：最小 Python Policy Explainer

教学版可以把策略判断写成纯函数，返回结构化 `PolicyDecision`。

```python
# learn-claude-code/policy_explainer.py
from dataclasses import dataclass, field, asdict
from typing import Literal
import json

Action = Literal["allow", "dry_run", "require_approval", "deny"]
Risk = Literal["low", "medium", "high"]

@dataclass
class EvidenceRef:
    kind: str
    ref: str
    summary: str

@dataclass
class PolicyDecision:
    action: Action
    risk: Risk
    reason_codes: list[str]
    evidence_refs: list[EvidenceRef] = field(default_factory=list)
    constraints: dict[str, str | int | bool] = field(default_factory=dict)
    remediation: list[str] = field(default_factory=list)
    policy_version: str = "policy-2026-05-13"


def decide(tool_name: str, args: dict, context: dict) -> PolicyDecision:
    reasons: list[str] = []
    evidence: list[EvidenceRef] = []
    remediation: list[str] = []
    constraints: dict[str, str | int | bool] = {}

    env = args.get("env", "dev")
    is_write = context.get("toolSideEffect") == "write"
    has_approval = context.get("approval") is True

    evidence.append(EvidenceRef(
        kind="tool_schema",
        ref=f"tools/{tool_name}",
        summary=f"sideEffect={context.get('toolSideEffect', 'unknown')}",
    ))

    if env == "prod" and is_write and not has_approval:
        reasons.append("PROD_WRITE_REQUIRES_APPROVAL")
        remediation.append("获取人工审批后重试")
        remediation.append("或先用 dry_run 生成计划和影响范围")
        return PolicyDecision(
            action="require_approval",
            risk="high",
            reason_codes=reasons,
            evidence_refs=evidence,
            remediation=remediation,
        )

    if context.get("blastRadiusUsers", 0) > 100:
        reasons.append("BLAST_RADIUS_TOO_LARGE")
        remediation.append("缩小 scope，分批执行，每批最多 100 个用户")
        return PolicyDecision(
            action="deny",
            risk="high",
            reason_codes=reasons,
            evidence_refs=evidence,
            remediation=remediation,
        )

    if env == "prod" and is_write:
        reasons.append("PROD_WRITE_APPROVED_WITH_LIMITS")
        constraints["maxAffectedUsers"] = 10
        constraints["requirePostVerify"] = True
        return PolicyDecision(
            action="allow",
            risk="high",
            reason_codes=reasons,
            evidence_refs=evidence,
            constraints=constraints,
        )

    reasons.append("LOW_RISK_TOOL_ALLOWED")
    return PolicyDecision(
        action="allow",
        risk="low",
        reason_codes=reasons,
        evidence_refs=evidence,
    )


if __name__ == "__main__":
    decision = decide(
        "deploy.restart",
        {"env": "prod", "service": "api"},
        {"toolSideEffect": "write", "approval": False, "blastRadiusUsers": 5},
    )
    print(json.dumps(asdict(decision), indent=2, ensure_ascii=False))
```

这里最重要的是 `reason_codes` 要稳定。不要写 `"looks risky"` 这种自然语言，应该写可搜索、可统计、可测试的代码。

---

## 3. pi-mono：PolicyDecision 作为工具中间件契约

生产版里，Policy Middleware 不应该只返回 boolean，而应该返回完整决策，并把它写入审计日志。

```ts
// pi-mono/src/policy/policy-decision.ts
type PolicyAction = 'allow' | 'dry_run' | 'require_approval' | 'deny'
type RiskLevel = 'low' | 'medium' | 'high'

type EvidenceRef = {
  kind: 'tool_schema' | 'approval' | 'runbook' | 'diff' | 'memory' | 'runtime'
  ref: string
  summary: string
}

type PolicyDecision = {
  action: PolicyAction
  risk: RiskLevel
  reasonCodes: string[]
  evidenceRefs: EvidenceRef[]
  constraints: Record<string, unknown>
  remediation: string[]
  policyVersion: string
}

type ToolCall = {
  runId: string
  toolName: string
  args: Record<string, unknown>
  actorId: string
}

export class PolicyMiddleware {
  constructor(
    private readonly auditLog: { append(event: unknown): Promise<void> },
    private readonly policyVersion = 'policy-2026-05-13',
  ) {}

  async authorize(call: ToolCall): Promise<PolicyDecision> {
    const env = String(call.args.env ?? 'dev')
    const sideEffect = this.classifySideEffect(call.toolName)
    const approved = Boolean(call.args.approvalId)

    const evidenceRefs: EvidenceRef[] = [
      {
        kind: 'tool_schema',
        ref: `tool:${call.toolName}`,
        summary: `sideEffect=${sideEffect}`,
      },
    ]

    let decision: PolicyDecision

    if (env === 'prod' && sideEffect === 'write' && !approved) {
      decision = {
        action: 'require_approval',
        risk: 'high',
        reasonCodes: ['PROD_WRITE_REQUIRES_APPROVAL'],
        evidenceRefs,
        constraints: { dryRunAllowed: true },
        remediation: ['run dry_run first', 'attach approvalId before retry'],
        policyVersion: this.policyVersion,
      }
    } else {
      decision = {
        action: 'allow',
        risk: env === 'prod' ? 'high' : 'low',
        reasonCodes: [env === 'prod' ? 'APPROVED_PROD_WRITE' : 'LOW_RISK_ALLOWED'],
        evidenceRefs,
        constraints: env === 'prod' ? { requirePostVerify: true } : {},
        remediation: [],
        policyVersion: this.policyVersion,
      }
    }

    await this.auditLog.append({
      type: 'policy_decision',
      runId: call.runId,
      actorId: call.actorId,
      toolName: call.toolName,
      decision,
      decidedAt: new Date().toISOString(),
    })

    return decision
  }

  private classifySideEffect(toolName: string): 'read' | 'write' {
    if (toolName.startsWith('message.') || toolName.startsWith('deploy.')) return 'write'
    return 'read'
  }
}
```

这样做的好处是：

- UI 可以直接展示 `remediation`，告诉操作者怎么解锁；
- 监控可以按 `reasonCodes` 聚合，看哪条规则最常阻断；
- shadow policy 可以对比 `reasonCodes`，不只是对比 action；
- 事故复盘可以引用 `evidenceRefs`，证明当时为什么这么判。

---

## 4. OpenClaw 实战：把解释放进副作用前闸门

在 OpenClaw 这类 Always-on Agent 里，策略解释最适合放在副作用工具前：发消息、push、部署、删数据、改配置，都先生成 `PolicyDecision`。

一个课程 cron 的例子：

```json
{
  "type": "policy_decision",
  "runId": "cron-agent-course-307",
  "toolName": "git.push",
  "action": "allow",
  "risk": "medium",
  "reasonCodes": ["USER_REQUESTED_REPO_UPDATE", "DIFF_CHECK_PASSED"],
  "evidenceRefs": [
    {
      "kind": "diff",
      "ref": "git-diff-check",
      "summary": "lesson file, README and TOOLS only"
    },
    {
      "kind": "runtime",
      "ref": "gh-auth:gfwfail",
      "summary": "GitHub account switched before push"
    }
  ],
  "constraints": {
    "branch": "main",
    "postVerify": "git log --oneline -1"
  },
  "remediation": [],
  "policyVersion": "policy-2026-05-13"
}
```

注意：这份解释不是发给群友的课程内容，而是 Agent 自己的运行证据。最终对用户汇报时可以压缩成一句：

> 已完成：diff check 通过，gh 账号为 gfwfail，commit 已推送。

---

## 5. 常见坑

### 坑 1：reason 写成自由文本

自由文本适合人看，不适合统计和测试。正确做法是：

- `reasonCodes` 用稳定枚举；
- `summary/remediation` 再写自然语言。

### 坑 2：只有拒绝才解释

`allow` 也必须解释。事故复盘时最关键的问题经常是：

> 当时为什么允许它执行？

### 坑 3：解释里泄露敏感数据

`evidenceRefs.summary` 只放摘要，不放 secret、token、用户隐私。需要精查时通过受控工具按权限回源。

### 坑 4：policyVersion 缺失

没有版本号，后面策略升级后就不知道旧决策按哪套规则做的。

---

## 6. 小结

成熟 Agent 的策略系统不应该是一个黑盒 `if`：

- 决策要有 `action/risk/reasonCodes`；
- 证据要有 `evidenceRefs`；
- 放行要有 `constraints`；
- 拒绝要有 `remediation`；
- 所有决策都要带 `policyVersion` 并进入审计日志。

**可审计解释让策略从“拦路虎”变成“运行契约”。**
