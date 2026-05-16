# 333. Agent 证据外发控制与通道策略（Evidence Egress Control & Channel Policy）

前面我们讲了 evidence 的访问授权、委托、限流、水印和隔离。今天补上生产里最容易漏的一环：**证据什么时候可以离开 Agent 系统**。

一句话：Agent 不是只要“能读”证据，就能把证据发到 Telegram、GitHub、Email、Slack 或日志里。所有外发动作都应该先经过 **Egress Gate（外发闸门）**，按通道能力、受众、字段敏感度和用途决定：允许发、降级摘要、要求审批，还是直接阻断。

这解决一个很现实的问题：

- Debug 时把 raw tool result 发进群；
- PR comment 里贴了内部 token / 用户数据；
- 子 Agent 把 proof bundle 原样传给低权限通道；
- 日志、消息、issue、邮件看似都是“输出”，但暴露半径完全不同；
- 一条 evidence 已允许内部 review，不代表可以公开发布。

成熟 Agent 的边界，不止在输入和决策，也在**输出离开系统的最后一厘米**。

---

## 1. 核心模型：Output 必须携带 EvidenceRef 和 Channel

不要把外发文本当成普通字符串。至少要知道：这段输出引用了哪些证据，要发到哪里，谁会看到。

```python
# learn-claude-code: egress_model.py
from dataclasses import dataclass, field
from enum import Enum

class Sensitivity(str, Enum):
    PUBLIC = "public"
    INTERNAL = "internal"
    SENSITIVE = "sensitive"
    SECRET = "secret"

@dataclass
class EvidenceRef:
    evidence_id: str
    fields: list[str]
    sensitivity: Sensitivity
    view_mode: str  # summary | redacted | raw

@dataclass
class OutboundMessage:
    channel: str          # telegram_group | github_pr | email | audit_log
    target: str           # group id / repo / recipient
    audience: str         # public | team | owner | security
    purpose: str          # teaching | incident_update | pr_review | debug
    body: str
    evidence_refs: list[EvidenceRef] = field(default_factory=list)
```

关键点：

1. **body 和 evidence_refs 分开**：不要只扫描最终文本，要知道文本从哪些 evidence 派生；
2. **channel 是策略输入**：Telegram 群、GitHub PR、私聊、审计日志的允许级别不同；
3. **view_mode 要保留**：summary 能外发，不代表 raw payload 能外发；
4. **purpose 要绑定**：同一段数据用于 incident_update 和 public_teaching，策略不同。

---

## 2. Egress Policy：按通道声明最大披露级别

教学版可以先写一个简单矩阵：每个通道允许的最高敏感度、是否允许 raw、是否必须水印。

```python
# learn-claude-code: egress_gate.py
from dataclasses import dataclass

@dataclass
class ChannelPolicy:
    max_sensitivity: Sensitivity
    allow_raw: bool
    require_watermark: bool
    require_approval_for: set[Sensitivity]

POLICIES = {
    "telegram_group": ChannelPolicy(
        max_sensitivity=Sensitivity.PUBLIC,
        allow_raw=False,
        require_watermark=True,
        require_approval_for={Sensitivity.INTERNAL, Sensitivity.SENSITIVE, Sensitivity.SECRET},
    ),
    "github_pr": ChannelPolicy(
        max_sensitivity=Sensitivity.INTERNAL,
        allow_raw=False,
        require_watermark=True,
        require_approval_for={Sensitivity.SENSITIVE, Sensitivity.SECRET},
    ),
    "audit_log": ChannelPolicy(
        max_sensitivity=Sensitivity.SENSITIVE,
        allow_raw=True,
        require_watermark=False,
        require_approval_for={Sensitivity.SECRET},
    ),
}

ORDER = [Sensitivity.PUBLIC, Sensitivity.INTERNAL, Sensitivity.SENSITIVE, Sensitivity.SECRET]

def is_above(value: Sensitivity, limit: Sensitivity) -> bool:
    return ORDER.index(value) > ORDER.index(limit)
```

然后外发前统一判断：

```python
@dataclass
class EgressDecision:
    action: str  # allow | redact | require_approval | deny
    reasons: list[str]


def check_egress(msg: OutboundMessage) -> EgressDecision:
    policy = POLICIES[msg.channel]
    reasons: list[str] = []

    for ref in msg.evidence_refs:
        if ref.view_mode == "raw" and not policy.allow_raw:
            reasons.append(f"raw_not_allowed:{ref.evidence_id}")

        if is_above(ref.sensitivity, policy.max_sensitivity):
            if ref.sensitivity in policy.require_approval_for:
                reasons.append(f"approval_required:{ref.sensitivity}:{ref.evidence_id}")
            else:
                reasons.append(f"sensitivity_too_high:{ref.evidence_id}")

    if any(r.startswith("sensitivity_too_high") for r in reasons):
        return EgressDecision("deny", reasons)
    if any(r.startswith("approval_required") for r in reasons):
        return EgressDecision("require_approval", reasons)
    if any(r.startswith("raw_not_allowed") for r in reasons):
        return EgressDecision("redact", reasons)
    return EgressDecision("allow", reasons)
```

这里的重点不是规则多复杂，而是：**外发必须有统一闸门，不能散落在每个工具调用前靠临场自觉。**

---

## 3. Redact / Declassify：能降级就别直接阻断

很多外发不是不能发，而是不能按原样发。比如群里教学可以发结论、结构、伪代码，但不能发真实 token、用户邮箱、内部 URL。

```python
# learn-claude-code: declassify.py
SECRET_MARKERS = ["api_key", "token", "password", "private_key", "authorization"]


def declassify_for_channel(msg: OutboundMessage) -> OutboundMessage:
    body = msg.body

    # 教学版：真实系统应按字段级 metadata redaction，不靠字符串猜测
    for marker in SECRET_MARKERS:
        body = body.replace(marker, f"{marker}[redacted]")

    safe_refs = []
    for ref in msg.evidence_refs:
        if ref.sensitivity in {Sensitivity.SENSITIVE, Sensitivity.SECRET}:
            safe_refs.append(EvidenceRef(
                evidence_id=ref.evidence_id,
                fields=["summary"],
                sensitivity=Sensitivity.INTERNAL,
                view_mode="summary",
            ))
        else:
            safe_refs.append(ref)

    return OutboundMessage(
        channel=msg.channel,
        target=msg.target,
        audience=msg.audience,
        purpose=msg.purpose,
        body=body,
        evidence_refs=safe_refs,
    )
```

生产版要把 declassification 当成显式流程：

```json
{
  "type": "evidence.declassified_for_egress",
  "sourceEvidenceId": "ev_raw_123",
  "derivedEvidenceId": "ev_summary_456",
  "fromSensitivity": "sensitive",
  "toSensitivity": "internal",
  "channel": "github_pr",
  "reason": "field_projection:summary_only",
  "at": "2026-05-16T08:30:00.000Z"
}
```

也就是说：降级后的摘要本身也是新证据，方便审计和复查。

---

## 4. pi-mono：做成 MessageEgressMiddleware

在 pi-mono 这种 Runtime 里，最稳妥的位置是所有外部发送工具之前：`message.send`、`github.comment`、`email.send`、`slack.post` 都走同一个 middleware。

```ts
// pi-mono: MessageEgressMiddleware.ts
type Sensitivity = 'public' | 'internal' | 'sensitive' | 'secret'
type EgressAction = 'allow' | 'redact' | 'require_approval' | 'deny'

type EvidenceRef = {
  evidenceId: string
  fields: string[]
  sensitivity: Sensitivity
  viewMode: 'summary' | 'redacted' | 'raw'
}

type OutboundMessage = {
  channel: 'telegram_group' | 'github_pr' | 'email' | 'audit_log'
  target: string
  audience: 'public' | 'team' | 'owner' | 'security'
  purpose: string
  body: string
  evidenceRefs: EvidenceRef[]
}

export class MessageEgressMiddleware {
  constructor(
    private readonly policy: EgressPolicy,
    private readonly redactor: OutputRedactor,
    private readonly audit: AuditLog,
  ) {}

  async beforeSend(msg: OutboundMessage): Promise<OutboundMessage> {
    const decision = await this.policy.check(msg)

    await this.audit.append({
      type: 'message.egress_checked',
      channel: msg.channel,
      target: msg.target,
      purpose: msg.purpose,
      evidenceRefs: msg.evidenceRefs.map(r => r.evidenceId),
      action: decision.action,
      reasons: decision.reasons,
    })

    if (decision.action === 'deny') {
      throw new Error(`egress_denied:${decision.reasons.join(',')}`)
    }

    if (decision.action === 'require_approval') {
      throw new Error(`egress_requires_approval:${decision.reasons.join(',')}`)
    }

    if (decision.action === 'redact') {
      return this.redactor.project(msg, { mode: 'channel_safe' })
    }

    return msg
  }
}
```

这里有个工程经验：**不要只在 Telegram tool 里做检查**。因为今天是 Telegram，明天可能是 GitHub comment、Email、Webhook、日志采集。Egress 是跨工具的统一能力。

---

## 5. OpenClaw 实战：发送群消息前的最后闸门

OpenClaw 的课程 Cron 很适合这个模式：我们会读本地 TOOLS、README、lesson、git 状态，然后发 Telegram 群。这里要保证：课程内容可以公开，不能把本地 secret 或内部 token 带出去。

伪流程：

```ts
// OpenClaw pseudo-flow before message.send
const draft = await composeLessonMessage()

const outbound: OutboundMessage = {
  channel: 'telegram_group',
  target: '-5115329245',
  audience: 'team',
  purpose: 'agent_course_teaching',
  body: draft.text,
  evidenceRefs: draft.evidenceRefs,
}

const safeOutbound = await messageEgress.beforeSend(outbound)

await message.send({
  target: safeOutbound.target,
  message: safeOutbound.body,
})
```

实际规则可以这样定：

- Telegram 群：只允许 public 教学内容；不允许 raw evidence；必须去掉真实 token、私有路径、用户隐私；
- GitHub PR comment：允许 internal 技术摘要；不贴生产 secret、不贴完整日志；
- 私聊老板：可发更详细摘要，但 secret 仍默认 redacted；
- audit log：可存更完整证据，但访问受 EvidenceAccessGate 控制；
- 外部公开渠道：只允许 public/declassified 内容。

这能防止 Agent 出现“内部看到了什么，就顺手发出去什么”的事故。

---

## 6. 和前几讲能力的组合关系

Evidence Egress Control 不是替代前面的能力，而是最后一道组合闸门：

- **Access Authorization**：先决定 Agent 能不能读；
- **Disclosure Budget**：限制读多少、读多细；
- **Watermarking**：允许外发时加 trace；
- **Quarantine**：可疑证据不能进入外发；
- **Egress Policy**：最终决定能不能发到这个通道。

最常见的错误是：内部 read 已经通过，就默认 output 也通过。实际上这是两件事：

> Read permission 解决“我能不能看”；Egress permission 解决“我能不能把它告诉别人”。

---

## 7. 最小实现 Checklist

如果今天要给 Agent 加外发控制，至少做 7 件事：

- [ ] OutboundMessage 结构里保留 channel / audience / purpose / evidenceRefs
- [ ] 每个外部通道声明 maxSensitivity / allowRaw / approval rules
- [ ] 所有发送工具统一走 MessageEgressMiddleware
- [ ] raw evidence 默认不能发到群聊、PR、Email
- [ ] 能降级的内容生成 declassified summary，而不是直接贴原文
- [ ] 每次 egress check 都写 audit event
- [ ] quarantined / revoked evidence 禁止外发

---

## 8. 一句话总结

成熟 Agent 的安全不是“别乱读”就够了，还要“别乱说”。**Evidence Egress Control 把每次消息、评论、邮件、日志写入都变成可审计的外发决策：按通道降级、加水印、要审批或阻断，确保内部证据不会顺手泄到外部世界。**
