# 327. Agent 证据访问授权与租户隔离（Evidence Access Authorization & Tenant Isolation）

上一课讲了 Evidence Schema 演进：证据格式会变，但审计器必须能诚实地理解历史证据。

今天再往生产环境推进一步：**谁可以读这些证据？**

很多 Agent 系统一开始把 Evidence 当内部日志：只要知道 `evidenceId` 就能读完整 payload。单机 demo 没问题，但到了多用户、多租户、多 Agent 团队时，这会直接变成数据泄露：

- A 用户的子 Agent 拿到 B 用户的 `evidenceId`，读到了工具结果
- 审计页为了展示方便，把 raw evidence 全量返回给前端
- 一个低权限 Agent 用“我要排查问题”为理由读取了安全决策证据
- 跨组织任务复用证据时，没有检查 tenant/project/channel 边界
- 证据索引做了脱敏，但详情接口绕过了索引权限

一句话：**Evidence 不只是可验证资产，也是敏感资产。能证明，不代表谁都能看。**

## 1. 核心原则：Evidence 读取也要走 Policy

证据访问建议至少绑定四个维度：

1. `tenantId`：属于哪个租户/组织
2. `subject`：谁在读，用户、Agent、审计服务还是后台任务
3. `purpose`：为什么读，回答问题、审计、调试、发布证明、安全决策
4. `scope`：只能读哪些 run / project / channel / evidence kind

一个 EvidenceEnvelope 可以这样设计：

```json
{
  "evidenceId": "ev_tool_123",
  "tenantId": "tenant_acme",
  "projectId": "agent-course",
  "runId": "run_2026_05_15_1430",
  "kind": "tool_result",
  "classification": "internal",
  "payloadRef": "s3://evidence/raw/ev_tool_123.json",
  "payloadHash": "sha256:...",
  "acl": {
    "owners": ["agent:main"],
    "readRoles": ["auditor", "run_owner"],
    "allowedPurposes": ["audit_query", "release_verification"]
  }
}
```

注意：这里把 `payloadRef` 和 metadata 分开。索引可以被广泛查询，raw payload 必须二次授权。

## 2. learn-claude-code：最小 Evidence Access Gate

教学版先不用复杂 RBAC，一个纯 Python gate 就能表达关键规则。

```python
# learn-claude-code: evidence_access_gate.py
from dataclasses import dataclass
from typing import Literal

Purpose = Literal["answer", "debug", "audit_query", "release_verification", "security_decision"]


@dataclass(frozen=True)
class Subject:
    subject_id: str
    tenant_id: str
    roles: set[str]
    project_ids: set[str]


@dataclass(frozen=True)
class EvidenceMeta:
    evidence_id: str
    tenant_id: str
    project_id: str
    run_id: str
    kind: str
    classification: Literal["public", "internal", "confidential", "secret"]
    read_roles: set[str]
    allowed_purposes: set[Purpose]


@dataclass(frozen=True)
class AccessDecision:
    allow: bool
    reason: str
    redaction: Literal["none", "summary_only", "deny_payload"] = "none"


class EvidenceAccessGate:
    def decide(self, subject: Subject, ev: EvidenceMeta, purpose: Purpose) -> AccessDecision:
        if subject.tenant_id != ev.tenant_id:
            return AccessDecision(False, "tenant_mismatch", "deny_payload")

        if ev.project_id not in subject.project_ids:
            return AccessDecision(False, "project_out_of_scope", "deny_payload")

        if purpose not in ev.allowed_purposes:
            return AccessDecision(False, "purpose_not_allowed", "deny_payload")

        if not (subject.roles & ev.read_roles):
            return AccessDecision(False, "missing_required_role", "deny_payload")

        if ev.classification in {"confidential", "secret"} and "auditor" not in subject.roles:
            return AccessDecision(True, "metadata_only_for_sensitive_evidence", "summary_only")

        return AccessDecision(True, "allowed", "none")


gate = EvidenceAccessGate()
subject = Subject(
    subject_id="agent:course-cron",
    tenant_id="tenant_gfwfail",
    roles={"run_owner"},
    project_ids={"agent-course"},
)

ev = EvidenceMeta(
    evidence_id="ev_git_push_327",
    tenant_id="tenant_gfwfail",
    project_id="agent-course",
    run_id="run_327",
    kind="tool_result",
    classification="internal",
    read_roles={"run_owner", "auditor"},
    allowed_purposes={"release_verification", "audit_query"},
)

print(gate.decide(subject, ev, "release_verification"))
```

这个 gate 的价值是：**拒绝发生在读取 raw payload 之前。**

如果只在 UI 层隐藏字段，底层 API 仍然能返回完整证据，那不叫权限控制，只叫前端美化。

## 3. 访问结果不只是 allow / deny，还要决定披露级别

Evidence 访问常见三档：

| 决策 | 返回内容 | 适用场景 |
|---|---|---|
| `deny_payload` | 只返回拒绝原因 | 跨租户、越权、purpose 不匹配 |
| `summary_only` | metadata + hash + 摘要 | 调试、低权限排障、对外证明 |
| `none` | raw payload | 审计员、run owner、高权限验证器 |

这和“证据最小披露”不同：最小披露解决**返回多少**，访问授权先解决**能不能返回**。

## 4. pi-mono：EvidenceAccessMiddleware

生产版建议把访问闸门放在 EvidenceStore 的读取入口，而不是每个业务 handler 手写。

```ts
// pi-mono: EvidenceAccessMiddleware.ts
type Purpose = 'answer' | 'debug' | 'audit_query' | 'release_verification' | 'security_decision'
type Redaction = 'none' | 'summary_only' | 'deny_payload'

type Subject = {
  id: string
  tenantId: string
  roles: string[]
  projectIds: string[]
}

type EvidenceMeta = {
  evidenceId: string
  tenantId: string
  projectId: string
  runId: string
  kind: string
  classification: 'public' | 'internal' | 'confidential' | 'secret'
  readRoles: string[]
  allowedPurposes: Purpose[]
  payloadHash: string
}

type AccessDecision = {
  allow: boolean
  reason: string
  redaction: Redaction
}

class EvidenceAccessPolicy {
  decide(subject: Subject, ev: EvidenceMeta, purpose: Purpose): AccessDecision {
    if (subject.tenantId !== ev.tenantId) {
      return { allow: false, reason: 'tenant_mismatch', redaction: 'deny_payload' }
    }

    if (!subject.projectIds.includes(ev.projectId)) {
      return { allow: false, reason: 'project_out_of_scope', redaction: 'deny_payload' }
    }

    if (!ev.allowedPurposes.includes(purpose)) {
      return { allow: false, reason: 'purpose_not_allowed', redaction: 'deny_payload' }
    }

    if (!ev.readRoles.some(role => subject.roles.includes(role))) {
      return { allow: false, reason: 'missing_required_role', redaction: 'deny_payload' }
    }

    if ((ev.classification === 'confidential' || ev.classification === 'secret') && !subject.roles.includes('auditor')) {
      return { allow: true, reason: 'sensitive_metadata_only', redaction: 'summary_only' }
    }

    return { allow: true, reason: 'allowed', redaction: 'none' }
  }
}

class EvidenceStore {
  constructor(
    private readonly index: EvidenceIndex,
    private readonly rawStore: RawEvidenceStore,
    private readonly policy: EvidenceAccessPolicy,
    private readonly audit: EvidenceAuditLog,
  ) {}

  async getEvidence(input: { evidenceId: string; subject: Subject; purpose: Purpose }) {
    const meta = await this.index.getMeta(input.evidenceId)
    const decision = this.policy.decide(input.subject, meta, input.purpose)

    await this.audit.record({
      evidenceId: input.evidenceId,
      subjectId: input.subject.id,
      purpose: input.purpose,
      decision,
      observedAt: new Date().toISOString(),
    })

    if (!decision.allow) {
      throw new Error(`evidence_access_denied:${decision.reason}`)
    }

    if (decision.redaction === 'summary_only') {
      return {
        evidenceId: meta.evidenceId,
        kind: meta.kind,
        runId: meta.runId,
        payloadHash: meta.payloadHash,
        redaction: 'summary_only',
      }
    }

    return {
      ...meta,
      payload: await this.rawStore.read(meta.evidenceId),
      redaction: 'none',
    }
  }
}
```

关键点：

- `index.getMeta()` 只拿授权判断所需 metadata
- `rawStore.read()` 只能在 policy allow 后发生
- 每次读证据也写审计日志
- purpose 是调用方必须声明的参数，不能默认成 `debug`

## 5. OpenClaw 实战：课程 Cron 的证据访问闸门

以这个课程 Cron 为例，执行 `git push` 前后会产生很多证据：diff、commit、push result、Telegram message id、TOOLS 更新记录。

安全做法：

1. `release_verification` 可以读取 commit/push/message metadata
2. 群消息只展示教学内容，不贴 raw token、完整环境变量、私有路径
3. 最终完成记录写入 memory/TOOLS 时，只写必要摘要和 commit id
4. 如果另一个 Agent 要复查，只给同 tenant/project/run 的 evidence
5. 如果是跨项目 summary，只返回 `payloadHash`、状态和摘要，不返回 raw payload

一个简单的完成证明可以长这样：

```json
{
  "proof": "course_release_327",
  "purpose": "release_verification",
  "tenantId": "tenant_gfwfail",
  "projectId": "agent-course",
  "evidence": [
    { "id": "ev_lesson_file", "kind": "artifact", "redaction": "summary_only" },
    { "id": "ev_git_commit", "kind": "tool_result", "redaction": "none" },
    { "id": "ev_telegram_send", "kind": "delivery_receipt", "redaction": "summary_only" }
  ]
}
```

这样审计者能确认“课程确实发布了”，但不会因为看证明而拿到不该看的敏感内容。

## 6. 常见坑

- **只保护写入，不保护读取**：Evidence 泄露多数发生在查询/调试接口
- **把 evidenceId 当权限**：ID 不是 capability，猜不到也不等于能读
- **跨租户 join**：搜索、聚合、导出接口最容易绕过 tenant filter
- **purpose 默认值太宽**：默认 `debug` 或 `admin` 会让所有调用都像超管
- **审计日志没记录拒绝**：被拒绝的读取尝试也很重要，可能是攻击探测

## 7. 小结

成熟 Agent 的 Evidence 系统要同时满足两件事：

1. **可验证**：证据链能证明动作真实发生
2. **可控披露**：只有正确的人、为正确目的、在正确范围内读取正确粒度的证据

证据访问授权不是附加功能，而是 Evidence Store 的入口契约。

下一次设计证据系统时，可以问自己一句：

> 如果一个低权限子 Agent 拿到了 evidenceId，它能读到什么？

如果答案是“完整 payload”，那这个系统还没到生产级。
