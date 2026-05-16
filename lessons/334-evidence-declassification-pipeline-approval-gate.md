# 334. Agent 证据降密流水线与审批闸门（Evidence Declassification Pipeline & Approval Gate）

上一课讲了 **Evidence Egress Control**：Agent 能不能把证据发到 Telegram、GitHub、Email、日志等外部通道。

今天继续往下拆：很多时候不是“能发 / 不能发”这么简单，而是要把 raw evidence 先降密成一个 **channel-safe artifact**，再决定是否允许外发。

核心思想：

> Agent 不应该把敏感证据直接发出去；应该先经过降密流水线，把 raw payload 转成可审计、可复核、可按通道发送的安全视图。

---

## 1. 为什么需要 Declassification Pipeline？

举几个真实场景：

- GitHub PR 评论里要解释 CI 失败原因，但日志里有 token。
- Telegram 群里要汇报部署结果，但原始 output 里有客户邮箱、订单号、内部 URL。
- Email 给客户要说明处理进度，但内部 evidence 里包含 runId、policy decision、工具调用参数。
- Incident status page 要公开事故影响，但不能暴露基础设施细节。

如果只靠一句 `redact()`，很容易出问题：

- 漏脱敏：正则没匹配到新格式 secret。
- 过度脱敏：把关键事实也删了，用户看不懂。
- 不可审计：事后不知道 raw evidence 如何变成这段话。
- 无审批边界：高敏内容自动“总结”后就发出去了。

所以成熟 Agent 需要一条明确流水线：

```text
Raw Evidence
  -> classify sensitivity
  -> extract allowed claims
  -> redact / generalize / aggregate
  -> validate no forbidden fields
  -> maybe require approval
  -> emit DeclassifiedArtifact
  -> egress policy checks channel
```

---

## 2. 数据结构：DeclassifiedArtifact

降密结果不要只是字符串，应该是一个带证明的 artifact：

```json
{
  "artifactId": "decl_01H...",
  "sourceEvidenceIds": ["ev_ci_log_123"],
  "targetChannel": "github_pr_comment",
  "audience": "repo_collaborators",
  "purpose": "explain_ci_failure",
  "classificationBefore": "secret",
  "classificationAfter": "internal",
  "transformations": ["secret_redaction", "path_generalization", "claim_extraction"],
  "claims": [
    "TypeScript build failed because src/payment.ts has a missing export",
    "The failure is reproducible in the latest CI run"
  ],
  "body": "CI failed in the TypeScript build. The actionable issue is a missing export in src/payment.ts. Secret-like values were removed.",
  "residualRisk": "low",
  "requiresApproval": false,
  "validatorResults": [
    { "name": "secret_scan", "ok": true },
    { "name": "channel_policy", "ok": true }
  ],
  "createdAt": "2026-05-16T11:30:00.000Z"
}
```

关键点：

- `sourceEvidenceIds`：能追溯来源。
- `classificationBefore/After`：证明真的降密了。
- `transformations`：说明做了哪些处理。
- `claims`：保留事实，不保留 raw payload。
- `validatorResults`：证明发出前检查过。
- `requiresApproval`：高风险情况不能自动发。

---

## 3. learn-claude-code：最小 Python 降密器

教学版可以先做一个纯函数：输入 raw evidence + channel policy，输出 artifact 或 block。

```python
# learn-claude-code/declassification.py
from dataclasses import dataclass, field
from datetime import datetime, timezone
import hashlib
import re
from typing import Literal

Sensitivity = Literal["public", "internal", "confidential", "secret"]

SECRET_PATTERNS = [
    re.compile(r"gh[pousr]_[A-Za-z0-9_]{20,}"),
    re.compile(r"AKIA[0-9A-Z]{16}"),
    re.compile(r"(?i)(api[_-]?key|token|secret|password)=([^\s]+)"),
]

@dataclass
class Evidence:
    evidence_id: str
    sensitivity: Sensitivity
    text: str
    claims: list[str] = field(default_factory=list)

@dataclass
class ChannelPolicy:
    channel: str
    max_sensitivity: Sensitivity
    allow_raw: bool
    require_approval_from: Sensitivity

@dataclass
class DeclassifiedArtifact:
    artifact_id: str
    source_evidence_ids: list[str]
    channel: str
    classification_before: Sensitivity
    classification_after: Sensitivity
    transformations: list[str]
    body: str
    residual_risk: Literal["low", "medium", "high"]
    requires_approval: bool

ORDER = ["public", "internal", "confidential", "secret"]

def rank(level: Sensitivity) -> int:
    return ORDER.index(level)

def redact_secrets(text: str) -> tuple[str, bool]:
    changed = False
    for pattern in SECRET_PATTERNS:
        text, count = pattern.subn("[REDACTED_SECRET]", text)
        changed = changed or count > 0
    return text, changed

def declassify(evidence: Evidence, policy: ChannelPolicy) -> DeclassifiedArtifact:
    transformations: list[str] = []

    if policy.allow_raw and rank(evidence.sensitivity) <= rank(policy.max_sensitivity):
        body = evidence.text
        after: Sensitivity = evidence.sensitivity
    else:
        # 默认不发 raw，优先发 claim summary
        body = "\n".join(f"- {claim}" for claim in evidence.claims) or "Evidence was reviewed; no channel-safe claims were extracted."
        transformations.append("claim_extraction")
        after = "internal" if rank(policy.max_sensitivity) >= rank("internal") else "public"

    body, changed = redact_secrets(body)
    if changed:
        transformations.append("secret_redaction")

    # 最后再扫一遍，防止降密结果仍含 secret
    _, still_has_secret = redact_secrets(body)
    residual_risk = "high" if still_has_secret else "low"

    requires_approval = (
        rank(evidence.sensitivity) >= rank(policy.require_approval_from)
        or residual_risk == "high"
    )

    digest = hashlib.sha256((evidence.evidence_id + policy.channel + body).encode()).hexdigest()[:16]
    return DeclassifiedArtifact(
        artifact_id=f"decl_{digest}",
        source_evidence_ids=[evidence.evidence_id],
        channel=policy.channel,
        classification_before=evidence.sensitivity,
        classification_after=after,
        transformations=transformations,
        body=body,
        residual_risk=residual_risk,
        requires_approval=requires_approval,
    )

if __name__ == "__main__":
    ev = Evidence(
        evidence_id="ev_ci_123",
        sensitivity="secret",
        text="CI failed. token=ghp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxx src/payment.ts missing export",
        claims=["TypeScript build failed because src/payment.ts has a missing export."],
    )
    policy = ChannelPolicy(
        channel="github_pr_comment",
        max_sensitivity="internal",
        allow_raw=False,
        require_approval_from="secret",
    )
    print(declassify(ev, policy))
```

这里故意保守：

- raw evidence 默认不发。
- 优先发 `claims`，而不是原文摘要。
- 降密后再做 secret scan。
- 原始 sensitivity 达到 `secret` 时要求人工 approval。

---

## 4. pi-mono：DeclassificationMiddleware

生产里应该把它放在 message/email/comment 等外发工具前面。

```ts
// pi-mono/packages/agent-security/src/declassification.ts
export type Sensitivity = 'public' | 'internal' | 'confidential' | 'secret'

export interface EvidenceRef {
  evidenceId: string
  sensitivity: Sensitivity
  claims?: string[]
  rawText?: string
}

export interface OutboundRequest {
  channel: 'telegram' | 'github_pr_comment' | 'email' | 'status_page'
  audience: string
  purpose: string
  body?: string
  evidenceRefs: EvidenceRef[]
}

export interface DeclassifiedArtifact {
  artifactId: string
  sourceEvidenceIds: string[]
  body: string
  classificationBefore: Sensitivity
  classificationAfter: Sensitivity
  transformations: string[]
  requiresApproval: boolean
  validatorResults: Array<{ name: string; ok: boolean; reason?: string }>
}

const level: Sensitivity[] = ['public', 'internal', 'confidential', 'secret']
const rank = (s: Sensitivity) => level.indexOf(s)

export class DeclassificationMiddleware {
  constructor(
    private readonly policies: Record<string, { maxSensitivity: Sensitivity; allowRaw: boolean; approvalFrom: Sensitivity }>,
    private readonly secretScanner: { scan(text: string): { ok: boolean; findings: string[] } },
  ) {}

  async prepare(req: OutboundRequest): Promise<DeclassifiedArtifact> {
    const policy = this.policies[req.channel]
    if (!policy) throw new Error(`No egress policy for ${req.channel}`)

    const before = req.evidenceRefs.reduce<Sensitivity>(
      (max, ref) => (rank(ref.sensitivity) > rank(max) ? ref.sensitivity : max),
      'public',
    )

    const transformations: string[] = []
    let body = req.body ?? ''

    const canUseRaw = policy.allowRaw && req.evidenceRefs.every(ref => rank(ref.sensitivity) <= rank(policy.maxSensitivity))

    if (!canUseRaw) {
      const claims = req.evidenceRefs.flatMap(ref => ref.claims ?? [])
      body = claims.length > 0
        ? claims.map(claim => `- ${claim}`).join('\n')
        : 'Evidence was reviewed, but no channel-safe claims were available.'
      transformations.push('claim_extraction')
    }

    const scan = this.secretScanner.scan(body)
    const validatorResults = [{ name: 'secret_scan', ok: scan.ok, reason: scan.findings.join(', ') }]

    const requiresApproval = rank(before) >= rank(policy.approvalFrom) || !scan.ok

    return {
      artifactId: `decl_${crypto.randomUUID()}`,
      sourceEvidenceIds: req.evidenceRefs.map(ref => ref.evidenceId),
      body,
      classificationBefore: before,
      classificationAfter: scan.ok ? policy.maxSensitivity : 'secret',
      transformations,
      requiresApproval,
      validatorResults,
    }
  }
}
```

工具调用前就可以这样接：

```ts
const artifact = await declassification.prepare(outbound)

if (artifact.requiresApproval) {
  return approvalQueue.enqueue({
    type: 'declassified_artifact_review',
    artifact,
    action: outbound.channel,
  })
}

await messageTool.send({
  channel: outbound.channel,
  text: artifact.body,
  evidence: { artifactId: artifact.artifactId },
})
```

注意：审批对象应该是 **artifact**，不是 raw evidence。审批人默认也只看降密视图，除非有更高权限。

---

## 5. OpenClaw 实战：群聊汇报前的降密闸门

以 OpenClaw 课程 cron 为例，每次要往 Telegram 群发消息前，可以做一个轻量 gate：

```ts
interface CourseSendIntent {
  target: 'telegram:-5115329245'
  lessonTitle: string
  repo: string
  evidenceRefs: Array<{
    evidenceId: string
    sensitivity: Sensitivity
    claims: string[]
  }>
}

async function sendCourseUpdate(intent: CourseSendIntent) {
  const artifact = await declassification.prepare({
    channel: 'telegram',
    audience: 'rust-study-group',
    purpose: 'agent_course_lesson',
    evidenceRefs: intent.evidenceRefs,
  })

  if (artifact.requiresApproval) {
    throw new Error(`Telegram send blocked: approval required for ${artifact.artifactId}`)
  }

  await message.send({
    target: '-5115329245',
    message: artifact.body,
  })

  await evidenceStore.append({
    type: 'egress_sent',
    artifactId: artifact.artifactId,
    target: intent.target,
    sentAt: new Date().toISOString(),
  })
}
```

这样做有三个好处：

1. 群聊只收到适合群聊的内容。
2. 课程 repo、commit、messageId 仍然可追踪。
3. 如果误发风险升高，系统会阻断，而不是靠 Agent 临场记得脱敏。

---

## 6. 常见策略：哪些情况必须人工审批？

建议默认规则：

- source evidence 是 `secret`：需要审批。
- 降密后 secret scanner 仍有发现：阻断或审批。
- 目标通道是 public/status_page：confidential 以上需要审批。
- audience 不包含原始证据 tenant：必须审批或 deny。
- artifact 由 LLM 摘要生成，且没有 claim-level source mapping：至少 medium risk。
- artifact 包含客户身份、订单号、内部 URL、部署细节：按通道规则判断。

一句话：

> 降密不是“让敏感内容变安全”的魔法；它是把可分享事实从敏感证据里剥离出来，并留下可审计证明。

---

## 7. 今天的落地作业

给自己的 Agent 外发链路加一个最小降密闸门：

- 每次 message/email/comment 前，要求传入 `evidenceRefs`。
- 默认禁止 raw evidence 外发。
- 只允许 claim summary 进入外部通道。
- 发送前跑一次 secret scan。
- 高敏来源生成 `DeclassifiedArtifact` 并进入 approval queue。
- 发送后记录 `artifactId -> messageId/commentId`。

做到这一步，Agent 就从“会脱敏”升级成“有可审计降密流程”。

---

## 8. 关键 takeaway

成熟 Agent 的外发安全不是最后一刻 `replace(secret, "***")`。

它应该是：

```text
Raw Evidence -> DeclassifiedArtifact -> Approval Gate -> Channel Egress
```

只有这样，Agent 才能在需要透明沟通时说清楚事实，又不会把不该外泄的证据一起带出去。
