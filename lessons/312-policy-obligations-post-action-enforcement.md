# 312 - Agent 策略义务与动作后置执行：allow 不是终点

> 关键词：Policy Obligations、Post-Action Enforcement、Evidence、Compensating Action、Side Effect Governance  
> 目标：让策略决策不只返回 `allow/deny`，还返回“允许之后必须做什么”，并在工具执行后自动验证、留证、通知或补偿。

前几课我们讲了策略影子模式、差异预算、可审计解释、fixture replay、覆盖矩阵、变异测试、临时 waiver。今天补上策略系统里很关键、但很多 Agent 框架会漏掉的一层：**Policy Obligations（策略义务）**。

很多系统的 policy decision 只有：

```text
allow / deny / require_approval / dry_run
```

但生产环境里，`allow` 经常不是“随便执行”，而是：

```text
允许执行，但必须：
1. 记录 evidence bundle
2. 执行后做 health check
3. 通知某个 channel
4. 在 10 分钟后复查状态
5. 失败时执行补偿动作
```

这就是 **Policy Obligation**：策略允许动作的同时，附带必须履行的后置义务。

---

## 1. 为什么需要 Obligations？

举几个 Agent 里的真实场景：

| 动作 | 单纯 allow 的问题 | obligation 应该要求 |
|---|---|---|
| `git push` | 推完就结束，没人知道影响 | 记录 commit、远端 URL、diff check 结果 |
| `message.send` | 发错群难追踪 | 保存 messageId、target、内容 hash |
| `deploy.trigger` | 部署成功不等于服务健康 | 部署后 health check + rollback plan |
| `config.patch` | 配置生效后可能漂移 | 写 audit log + reload verification |
| `waiver.allow` | 例外执行后没人收口 | 到期撤销 + 事后 review |

成熟 Agent 的策略不是只会“拦不拦”，而是会规定：**如果放行，必须带着哪些尾巴走完。**

---

## 2. 核心模型：Decision = Action + Obligations

一个更完整的策略返回值应该长这样：

```json
{
  "action": "allow",
  "risk": "medium",
  "reasonCodes": ["OWNER_CONFIRMED", "WITHIN_CHANGE_WINDOW"],
  "obligations": [
    {
      "type": "record_evidence",
      "required": true,
      "params": { "bundle": "release-evidence" }
    },
    {
      "type": "post_check",
      "required": true,
      "params": { "check": "git_remote_contains_commit" }
    },
    {
      "type": "notify",
      "required": false,
      "params": { "channel": "owner", "template": "deployment_summary" }
    }
  ]
}
```

注意这里的关键点：

- `obligations` 是策略的一部分，不是业务代码临时想起来的 TODO；
- 每个 obligation 要有 `required`，失败时决定是 block、rollback、retry 还是 warn；
- obligation 执行结果要进 audit log，否则无法证明已经履行。

---

## 3. learn-claude-code：最小 Python 教学版

下面是一个极简 obligation engine：策略返回义务，工具执行后逐条履行。

```python
from dataclasses import dataclass, field
from typing import Any, Literal
import hashlib
import json
import time

Action = Literal["allow", "deny", "require_approval", "dry_run"]
ObligationType = Literal["record_evidence", "post_check", "notify", "schedule_review"]

@dataclass
class Obligation:
    type: ObligationType
    required: bool
    params: dict[str, Any] = field(default_factory=dict)

@dataclass
class PolicyDecision:
    action: Action
    risk: str
    reason_codes: list[str]
    obligations: list[Obligation] = field(default_factory=list)

@dataclass
class ToolCall:
    name: str
    args: dict[str, Any]
    side_effect: bool
    target: str

@dataclass
class ToolResult:
    ok: bool
    output: Any
    evidence: dict[str, Any] = field(default_factory=dict)

class PolicyEngine:
    def decide(self, call: ToolCall) -> PolicyDecision:
        if call.name == "git.push":
            return PolicyDecision(
                action="allow",
                risk="medium",
                reason_codes=["REPO_COURSE_AUTOMATION", "OWNER_SCHEDULED_CRON"],
                obligations=[
                    Obligation("record_evidence", True, {"kind": "git_push"}),
                    Obligation("post_check", True, {"check": "remote_contains_commit"}),
                    Obligation("notify", False, {"template": "short_summary"}),
                ],
            )

        if call.side_effect:
            return PolicyDecision(
                action="require_approval",
                risk="high",
                reason_codes=["UNKNOWN_SIDE_EFFECT_TOOL"],
            )

        return PolicyDecision("allow", "low", ["READ_ONLY_TOOL"])

class ObligationRunner:
    def __init__(self):
        self.audit_events: list[dict[str, Any]] = []

    def run_all(self, decision: PolicyDecision, call: ToolCall, result: ToolResult) -> None:
        for ob in decision.obligations:
            ok, details = self.run_one(ob, call, result)
            self.audit_events.append({
                "ts": int(time.time()),
                "tool": call.name,
                "target": call.target,
                "obligation": ob.type,
                "required": ob.required,
                "ok": ok,
                "details": details,
            })

            if ob.required and not ok:
                raise RuntimeError(f"required obligation failed: {ob.type}")

    def run_one(self, ob: Obligation, call: ToolCall, result: ToolResult) -> tuple[bool, dict[str, Any]]:
        if ob.type == "record_evidence":
            content = json.dumps({"call": call.args, "result": result.evidence}, sort_keys=True)
            return True, {
                "kind": ob.params.get("kind"),
                "content_hash": hashlib.sha256(content.encode()).hexdigest(),
            }

        if ob.type == "post_check":
            # 教学版：真实系统里这里会查 GitHub API / health endpoint / message receipt
            check = ob.params["check"]
            return result.ok, {"check": check, "observed": "result.ok"}

        if ob.type == "notify":
            # optional obligation 失败不应直接让主动作失败，但要审计
            return True, {"template": ob.params.get("template"), "sent": False, "mode": "dry_demo"}

        if ob.type == "schedule_review":
            return True, {"scheduled": True}

        return False, {"error": "unknown obligation"}

# demo
call = ToolCall("git.push", {"branch": "main"}, side_effect=True, target="gfwfail/agent-course")
decision = PolicyEngine().decide(call)

if decision.action != "allow":
    raise SystemExit(f"blocked: {decision.action} {decision.reason_codes}")

result = ToolResult(ok=True, output="pushed", evidence={"commit": "abc123"})
runner = ObligationRunner()
runner.run_all(decision, call, result)

print(json.dumps(runner.audit_events, indent=2, ensure_ascii=False))
```

重点不是代码多复杂，而是执行顺序：

```text
policy decide
  -> tool execute
  -> obligation runner
  -> audit required obligations
  -> required obligation failed? block / rollback / escalate
```

这让 Agent 的“执行完成”从“命令返回 0”升级成“命令返回 0 + 策略义务履行完成”。

---

## 4. obligation 失败怎么处理？

不是所有 obligation 失败都一样。

建议分四类：

```text
required + safety-critical  -> block / rollback / incident
required + audit-critical   -> mark incomplete + escalate
optional + comms            -> retry / warn
scheduled                   -> enqueue durable task
```

例子：

- `post_check: health_ok` 失败：部署应该 rollback 或进入 incident；
- `record_evidence` 失败：动作可能已经做了，但不能算完成，要升级；
- `notify owner` 失败：可以重试，不一定回滚业务动作；
- `schedule_review` 失败：需要补建任务，否则 waiver 永久没人收口。

所以 obligation 需要有失败策略：

```json
{
  "type": "post_check",
  "required": true,
  "onFailure": "rollback_or_escalate",
  "params": { "check": "service_health" }
}
```

---

## 5. pi-mono：生产版 Middleware 形态

在 TypeScript 生产系统里，obligation 最适合放在 Tool Middleware 之后，统一包住所有工具调用。

```ts
type PolicyAction = 'allow' | 'deny' | 'require_approval' | 'dry_run'

type ObligationType =
  | 'record_evidence'
  | 'post_check'
  | 'notify'
  | 'schedule_review'
  | 'compensating_action'

type Obligation = {
  type: ObligationType
  required: boolean
  onFailure: 'warn' | 'retry' | 'escalate' | 'rollback_or_escalate'
  params: Record<string, unknown>
}

type PolicyDecision = {
  action: PolicyAction
  risk: 'low' | 'medium' | 'high' | 'critical'
  reasonCodes: string[]
  obligations: Obligation[]
}

type ToolCall = {
  id: string
  name: string
  args: Record<string, unknown>
  actor: string
  tenant: string
  sideEffect: boolean
}

type ToolResult = {
  ok: boolean
  data?: unknown
  error?: string
  evidence?: Record<string, unknown>
}

interface PolicyEngine {
  decide(call: ToolCall): Promise<PolicyDecision>
}

interface ToolExecutor {
  execute(call: ToolCall): Promise<ToolResult>
}

interface ObligationExecutor {
  fulfill(obligation: Obligation, call: ToolCall, result: ToolResult): Promise<ToolResult>
}

class PolicyObligationMiddleware implements ToolExecutor {
  constructor(
    private readonly policy: PolicyEngine,
    private readonly next: ToolExecutor,
    private readonly obligations: ObligationExecutor,
    private readonly audit: { write(event: unknown): Promise<void> },
  ) {}

  async execute(call: ToolCall): Promise<ToolResult> {
    const decision = await this.policy.decide(call)

    await this.audit.write({
      event: 'policy_decision',
      callId: call.id,
      tool: call.name,
      action: decision.action,
      risk: decision.risk,
      reasonCodes: decision.reasonCodes,
      obligationCount: decision.obligations.length,
    })

    if (decision.action !== 'allow') {
      return { ok: false, error: `policy_${decision.action}` }
    }

    const result = await this.next.execute(call)

    for (const obligation of decision.obligations) {
      const obResult = await this.obligations.fulfill(obligation, call, result)

      await this.audit.write({
        event: 'obligation_fulfilled',
        callId: call.id,
        tool: call.name,
        obligation: obligation.type,
        required: obligation.required,
        onFailure: obligation.onFailure,
        ok: obResult.ok,
        evidence: obResult.evidence,
      })

      if (!obResult.ok && obligation.required) {
        if (obligation.onFailure === 'rollback_or_escalate') {
          await this.audit.write({
            event: 'obligation_required_failed',
            callId: call.id,
            action: 'rollback_or_escalate',
          })
        }

        return {
          ok: false,
          error: `required_obligation_failed:${obligation.type}`,
          evidence: { originalResult: result.evidence, obligation: obligation.type },
        }
      }
    }

    return result
  }
}
```

这里有一个工程关键点：**业务工具不应该自己到处写通知、审计、post-check。**  
工具只负责做事；策略中间件负责规定“做完以后必须履行什么”。

---

## 6. Obligation Registry：不要把后置动作写死

生产系统里 obligation 类型会越来越多，建议注册表化：

```ts
class ObligationRegistry implements ObligationExecutor {
  private handlers = new Map<ObligationType, ObligationExecutor>()

  register(type: ObligationType, handler: ObligationExecutor) {
    this.handlers.set(type, handler)
  }

  async fulfill(obligation: Obligation, call: ToolCall, result: ToolResult) {
    const handler = this.handlers.get(obligation.type)
    if (!handler) {
      return { ok: false, error: `unknown_obligation:${obligation.type}` }
    }
    return handler.fulfill(obligation, call, result)
  }
}
```

这样 policy 可以不断新增 obligation，而工具执行主链路不需要改来改去。

---

## 7. OpenClaw 实战：课程 Cron 的 obligation

拿我们这个课程 cron 举例，一个“发课 + 写文件 + push”的 run，可以声明这些 obligations：

```yaml
tool: agent_course.publish
risk: medium
action: allow
obligations:
  - type: record_evidence
    required: true
    params:
      evidence:
        - lesson_file_path
        - telegram_message_id
        - git_commit_sha
        - git_remote_url

  - type: post_check
    required: true
    onFailure: escalate
    params:
      checks:
        - lesson_file_exists
        - readme_contains_lesson
        - tools_md_contains_topic
        - git_status_clean

  - type: notify
    required: false
    onFailure: retry
    params:
      target: owner
      template: cron_summary
```

这比“我感觉做完了”靠谱得多，因为完成标准变成了可执行检查。

对 OpenClaw 这种 always-on Agent 来说，obligation 特别适合放在：

- cron job 完成闸门；
- message 发送后 receipt 记录；
- git push 后远端 commit 验证；
- config/apply 后 reload 验证；
- waiver 到期后的自动 review。

---

## 8. 常见坑

### 坑 1：把 obligation 当日志

日志只是记录发生了什么；obligation 是“必须履行什么”。

如果 required obligation 失败还能静默成功，那它就不是 obligation，只是装饰性日志。

### 坑 2：只在成功时跑 obligation

有些 obligation 必须在失败时也跑，比如：

- 失败证据归档；
- 失败通知；
- partial side effect 检查；
- compensating action。

所以 obligation 可以拆成：

```text
pre_obligations
post_success_obligations
post_failure_obligations
always_obligations
```

### 坑 3：optional obligation 没有重试

optional 不代表不重要。通知类 obligation 可以不阻断主动作，但应该有 retry / dead letter queue，否则系统会慢慢失明。

### 坑 4：obligation 没有幂等 key

如果 Agent 崩溃重试，obligation 可能重复执行。每个 obligation 最好带：

```text
run_id + call_id + obligation_type + target_hash
```

这样 evidence 写入、通知、review task 创建都能幂等。

---

## 9. 一句话总结

> Policy 决定“能不能做”，Obligation 决定“做了以后必须怎么收口”。

成熟 Agent 的 `allow` 不应该是终点，而应该是一个带条件的执行契约：

```text
allow = execute + verify + evidence + notify + compensate_if_needed
```

没有 obligation 的策略，只能拦截风险；有 obligation 的策略，才能把自动化动作闭环成可审计、可恢复、可运营的生产系统。
