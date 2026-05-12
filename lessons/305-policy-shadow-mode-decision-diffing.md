# 305. Agent 策略影子模式与决策差异对比（Policy Shadow Mode & Decision Diffing）

上一课讲“知识晋级要灰度”。今天补上灰度发布里最实用的一块：**策略影子模式**。

很多 Agent 事故不是工具坏了，而是“新策略看起来更聪明，实际上悄悄改变了决策”。比如：

- 旧策略：`git push` 前必须先查 PR 状态；
- 新策略：如果是 cron 自动课程，可以直接 push；
- 结果：新策略省了一步，但破坏了老板的铁律。

所以成熟 Agent 不应该直接把新策略接到生产路径，而是先让它在旁边“跟跑”：

> 主策略照常决定真实动作；影子策略只计算如果由它来决策会怎么做，然后记录 diff、风险和证据。

---

## 1. 核心思想

Policy Shadow Mode 有三条规则：

1. **Primary policy 决定真实行为**：线上继续安全稳定。
2. **Shadow policy 只观察不执行**：不能触发工具、不能改状态、不能发消息。
3. **Decision diff 可审计**：每次不同都记录 input、primaryDecision、shadowDecision、reason、risk。

它解决的问题是：

- 新策略上线前不知道会影响多少真实请求；
- 靠测试样例覆盖不了长尾场景；
- 策略改动经常“看起来只是文案”，实际改变了副作用权限；
- 出问题后很难解释“新旧策略到底哪里不一样”。

---

## 2. learn-claude-code：最小 Python 影子策略执行器

教学版可以先把 policy 当成普通函数。

```python
# learn-claude-code/policy_shadow.py
from dataclasses import dataclass, asdict
from datetime import datetime, timezone
import json

@dataclass
class Decision:
    action: str              # allow / deny / require_approval / dry_run
    reason: str
    risk: str                # low / medium / high

@dataclass
class PolicyInput:
    tool: str
    side_effect: bool
    actor: str
    channel: str
    intent: str


def primary_policy(inp: PolicyInput) -> Decision:
    if inp.side_effect and inp.channel == "group":
        return Decision("require_approval", "group side-effect needs approval", "high")
    return Decision("allow", "default allow", "low")


def shadow_policy(inp: PolicyInput) -> Decision:
    if inp.tool in {"git_push", "deploy", "send_message"}:
        return Decision("require_approval", "critical tool must be approved", "high")
    if inp.side_effect:
        return Decision("dry_run", "side-effect starts in dry-run", "medium")
    return Decision("allow", "read-only", "low")


def run_with_shadow(inp: PolicyInput, log_path="policy-shadow.jsonl") -> Decision:
    primary = primary_policy(inp)
    shadow = shadow_policy(inp)

    if primary != shadow:
        event = {
            "ts": datetime.now(timezone.utc).isoformat(),
            "input": asdict(inp),
            "primary": asdict(primary),
            "shadow": asdict(shadow),
            "diff": {
                "actionChanged": primary.action != shadow.action,
                "riskChanged": primary.risk != shadow.risk,
            },
        }
        with open(log_path, "a") as f:
            f.write(json.dumps(event, ensure_ascii=False) + "\n")

    # 关键：真实返回 primary，不返回 shadow
    return primary


if __name__ == "__main__":
    decision = run_with_shadow(PolicyInput(
        tool="git_push",
        side_effect=True,
        actor="cron-agent",
        channel="main",
        intent="publish lesson",
    ))
    print(decision)
```

重点不是代码复杂，而是边界清楚：**shadow policy 永远不能改变真实执行结果**。

---

## 3. pi-mono：生产里的 PolicyShadowMiddleware

生产版建议放在 Tool Dispatch 前面，形成统一闸门。

```ts
// pi-mono/src/middleware/policy-shadow.ts
type PolicyAction = 'allow' | 'deny' | 'require_approval' | 'dry_run'

type Decision = {
  action: PolicyAction
  reason: string
  risk: 'low' | 'medium' | 'high'
  policyVersion: string
}

type ToolRequest = {
  runId: string
  toolName: string
  args: unknown
  actor: string
  channel: string
  sideEffect: boolean
}

type Policy = {
  decide(req: ToolRequest): Promise<Decision>
}

type AuditSink = {
  write(event: unknown): Promise<void>
}

export class PolicyShadowMiddleware {
  constructor(
    private primary: Policy,
    private shadow: Policy | null,
    private audit: AuditSink,
  ) {}

  async decide(req: ToolRequest): Promise<Decision> {
    const primaryDecision = await this.primary.decide(req)

    // shadow 失败不能影响主路径，但必须记录
    if (this.shadow) {
      queueMicrotask(async () => {
        try {
          const shadowDecision = await this.shadow!.decide(req)
          if (this.changed(primaryDecision, shadowDecision)) {
            await this.audit.write({
              type: 'policy_shadow_diff',
              runId: req.runId,
              toolName: req.toolName,
              sideEffect: req.sideEffect,
              primary: primaryDecision,
              shadow: shadowDecision,
              severity: this.diffSeverity(primaryDecision, shadowDecision),
              observedAt: new Date().toISOString(),
            })
          }
        } catch (err) {
          await this.audit.write({
            type: 'policy_shadow_error',
            runId: req.runId,
            toolName: req.toolName,
            error: String(err),
            observedAt: new Date().toISOString(),
          })
        }
      })
    }

    return primaryDecision
  }

  private changed(a: Decision, b: Decision): boolean {
    return a.action !== b.action || a.risk !== b.risk
  }

  private diffSeverity(a: Decision, b: Decision): 'info' | 'warning' | 'danger' {
    if (a.action === 'deny' && b.action === 'allow') return 'danger'
    if (a.action === 'require_approval' && b.action === 'allow') return 'danger'
    if (a.risk === 'high' && b.risk === 'low') return 'warning'
    return 'info'
  }
}
```

这里有个生产细节：shadow 用 `queueMicrotask` 异步跑，避免拖慢主链路；但如果你要做强一致审计，也可以同步跑，只是要设置 timeout。

---

## 4. OpenClaw 实战：课程 Cron 的影子策略

OpenClaw 里最适合拿来试的场景就是 cron：

- 主策略：按当前流程发课、写文件、提交、push；
- 影子策略：模拟更严格的发布门控；
- 只记录“如果用新门控，这次会不会被要求审批/阻断”。

一个影子事件可以长这样：

```json
{
  "type": "policy_shadow_diff",
  "runId": "cron:agent-course:2026-05-13T06:30:00+11:00",
  "operation": "git_push",
  "primary": {
    "action": "allow",
    "reason": "course cron allowlisted",
    "policyVersion": "course-policy@2026-05-01"
  },
  "shadow": {
    "action": "require_approval",
    "reason": "all pushes require live PR-state check evidence",
    "policyVersion": "course-policy@2026-05-13-shadow"
  },
  "severity": "warning",
  "evidence": [
    "gh pr list --state all --limit 5 executed",
    "git diff --check passed"
  ]
}
```

如果连续 7 天 shadow diff 都是低风险，才考虑把 shadow 晋级成 primary。反过来，如果出现 `primary=allow, shadow=deny`，不要急着切新策略，先看是不是 shadow 太保守，还是 primary 真有漏洞。

---

## 5. 常见坑

**坑 1：shadow 偷偷产生副作用**  
影子策略只能算 decision，不能调用真实工具。需要数据时，读 primary 已经收集的 evidence，不要自己再发请求。

**坑 2：只看 action，不看 reason**  
两个策略都返回 `allow`，但 reason 不同，也可能代表安全模型变了。高风险工具建议记录 reason diff。

**坑 3：diff 太多没人看**  
要按风险分级：

- `info`：只进日志；
- `warning`：每日汇总；
- `danger`：立即通知或冻结晋级。

**坑 4：没有晋级标准**  
Shadow mode 不是永久旁路。上线前写清楚：运行多久、覆盖多少请求、danger diff 为 0、warning diff 已解释，才允许 promote。

---

## 6. 一句话总结

Policy Shadow Mode 的价值是：

> 让新策略先在真实流量旁边“实习”，证明它比旧策略更安全，再让它接管方向盘。

成熟 Agent 的策略升级，不靠信心，靠影子运行、差异证据和可回滚晋级。
