# 329 - Agent 证据委托访问与作用域共享：Evidence Delegation & Scoped Sharing

> 证据不是只能“本人读取”或“完全公开”。成熟 Agent 系统需要一种可撤销、可审计、可限权的证据委托机制。

前几讲我们已经把 Evidence 做到了：可授权访问、租户隔离、访问审计、异常检测。今天再往前走一步：当一个 Agent 需要把证据交给另一个 Agent / 子任务 / 审计机器人时，不能直接复制 raw payload，也不能靠口头说“你可以看”。

这就需要 **Evidence Delegation & Scoped Sharing**：把证据访问权包装成短期 Delegation Grant，只允许在指定 tenant、purpose、fields、risk、TTL 内使用，并且每次使用都能被审计、撤销、降级。

---

## 核心问题

常见危险做法：

```text
主 Agent 查到敏感 evidence
  ↓
直接把 raw evidence 放进 sub-agent prompt
  ↓
子 Agent 又写进日志 / 群聊 / 缓存
  ↓
事后无法知道谁看过、为什么看、还能不能撤销
```

正确模型应该是：

```text
Owner / Primary Agent
  ↓ issue delegation grant
DelegationGrant(scope + purpose + ttl + fields + maxReads)
  ↓ redeem grant
Delegate Agent
  ↓ access gate + audit + anomaly review
Evidence View(summary / redacted / raw)
```

关键点：

1. **委托的是能力，不是数据副本**：传 `grantId` / `capabilityRef`，不要传 raw evidence。
2. **委托必须更窄**：delegate 的权限只能是 owner 权限子集，不能扩大 tenant、purpose、fields。
3. **短 TTL + 次数限制**：一次子任务用完就过期，避免 capability 漂在系统里。
4. **可撤销**：owner 取消任务、incident freeze、policy 变化时，grant 立即失效。
5. **每次 redeem 都审计**：知道 grant 被谁、在哪个 run、为哪个目的使用。

---

## learn-claude-code：最小委托授权器

教学版用一个内存/文件式 GrantStore 就能讲清楚模型。

```python
# learn-claude-code/evidence_delegation.py
from __future__ import annotations

import secrets
import time
from dataclasses import dataclass, field
from typing import Literal

Decision = Literal["deny", "summary_only", "raw"]


@dataclass
class EvidenceDelegationGrant:
    grant_id: str
    evidence_id: str
    issuer: str
    delegate: str
    tenant: str
    purpose: str
    allowed_decision: Decision
    allowed_fields: set[str]
    expires_at: float
    max_reads: int = 1
    reads: int = 0
    revoked: bool = False
    reason: str = ""


class DelegationStore:
    def __init__(self):
        self.grants: dict[str, EvidenceDelegationGrant] = {}

    def issue(
        self,
        *,
        evidence_id: str,
        issuer: str,
        delegate: str,
        tenant: str,
        purpose: str,
        allowed_decision: Decision,
        allowed_fields: set[str],
        ttl_seconds: int,
        max_reads: int = 1,
        reason: str,
    ) -> EvidenceDelegationGrant:
        grant = EvidenceDelegationGrant(
            grant_id="egd_" + secrets.token_urlsafe(18),
            evidence_id=evidence_id,
            issuer=issuer,
            delegate=delegate,
            tenant=tenant,
            purpose=purpose,
            allowed_decision=allowed_decision,
            allowed_fields=allowed_fields,
            expires_at=time.time() + ttl_seconds,
            max_reads=max_reads,
            reason=reason,
        )
        self.grants[grant.grant_id] = grant
        return grant

    def revoke(self, grant_id: str, *, reason: str) -> None:
        grant = self.grants[grant_id]
        grant.revoked = True
        grant.reason = reason

    def redeem(
        self,
        *,
        grant_id: str,
        actor: str,
        tenant: str,
        purpose: str,
        requested_fields: set[str],
        requested_decision: Decision,
    ) -> EvidenceDelegationGrant:
        grant = self.grants[grant_id]
        now = time.time()

        if grant.revoked:
            raise PermissionError("delegation revoked")
        if now >= grant.expires_at:
            raise PermissionError("delegation expired")
        if grant.reads >= grant.max_reads:
            raise PermissionError("delegation read budget exhausted")
        if actor != grant.delegate:
            raise PermissionError("wrong delegate")
        if tenant != grant.tenant or purpose != grant.purpose:
            raise PermissionError("scope mismatch")
        if not requested_fields.issubset(grant.allowed_fields):
            raise PermissionError("field scope too wide")
        if requested_decision == "raw" and grant.allowed_decision != "raw":
            raise PermissionError("raw access not delegated")

        grant.reads += 1
        return grant
```

实际读取证据时，只让 delegate 拿到投影后的 view：

```python
def project_evidence(payload: dict, fields: set[str]) -> dict:
    return {key: payload[key] for key in fields if key in payload}


def read_with_delegation(store, evidence_db, *, grant_id, actor, tenant, purpose, fields):
    grant = store.redeem(
        grant_id=grant_id,
        actor=actor,
        tenant=tenant,
        purpose=purpose,
        requested_fields=fields,
        requested_decision="summary_only",
    )

    payload = evidence_db[grant.evidence_id]
    view = project_evidence(payload, grant.allowed_fields & fields)

    # 这里必须写 audit：grant_id、issuer、delegate、purpose、field_count、decision
    return view
```

这段代码的核心不是复杂，而是边界清楚：子 Agent 永远不能靠“拿到字符串”扩大权限。

---

## pi-mono：DelegatedEvidenceCapability

生产版建议把委托建模成 Capability，而不是普通 token。

```typescript
// pi-mono/src/evidence/DelegatedEvidenceCapability.ts
export type EvidenceDecision = "deny" | "summary_only" | "raw";

export interface DelegatedEvidenceCapability {
  id: string;
  evidenceId: string;
  issuerActor: string;
  delegateActor: string;
  tenantId: string;
  purpose: "answer_user" | "debug_incident" | "audit_review" | "external_side_effect";
  maxDecision: EvidenceDecision;
  allowedFields: string[];
  expiresAt: string;
  maxReads: number;
  reads: number;
  revokedAt?: string;
  reason: string;
  parentEvidenceHash: string;
}

export interface RedeemDelegationInput {
  capabilityId: string;
  actor: string;
  tenantId: string;
  purpose: string;
  requestedFields: string[];
  requestedDecision: EvidenceDecision;
  runId: string;
}
```

Middleware 负责四件事：

```typescript
// pi-mono/src/middleware/EvidenceDelegationMiddleware.ts
export class EvidenceDelegationMiddleware {
  constructor(
    private readonly capabilities: CapabilityStore,
    private readonly audit: EvidenceAuditLog,
    private readonly policy: EvidenceAccessPolicy,
  ) {}

  async redeem(input: RedeemDelegationInput): Promise<DelegatedEvidenceCapability> {
    const cap = await this.capabilities.get(input.capabilityId);
    const now = new Date();

    if (!cap) throw new Error("delegation_not_found");
    if (cap.revokedAt) throw new Error("delegation_revoked");
    if (new Date(cap.expiresAt) <= now) throw new Error("delegation_expired");
    if (cap.reads >= cap.maxReads) throw new Error("delegation_budget_exhausted");
    if (cap.delegateActor !== input.actor) throw new Error("wrong_delegate");
    if (cap.tenantId !== input.tenantId || cap.purpose !== input.purpose) {
      throw new Error("delegation_scope_mismatch");
    }

    const requested = new Set(input.requestedFields);
    const allowed = new Set(cap.allowedFields);
    for (const field of requested) {
      if (!allowed.has(field)) throw new Error("delegation_field_denied");
    }

    if (input.requestedDecision === "raw" && cap.maxDecision !== "raw") {
      throw new Error("delegation_raw_denied");
    }

    // 再跑一次当前策略：旧 grant 不能绕过新 policy / freeze / revocation
    await this.policy.assertCurrentAccess({
      actor: input.actor,
      tenantId: input.tenantId,
      purpose: input.purpose,
      evidenceId: cap.evidenceId,
      requestedDecision: input.requestedDecision,
    });

    await this.capabilities.incrementReads(cap.id);
    await this.audit.append({
      kind: "evidence_delegation_redeemed",
      runId: input.runId,
      capabilityId: cap.id,
      issuerActor: cap.issuerActor,
      delegateActor: input.actor,
      tenantId: input.tenantId,
      purpose: input.purpose,
      fieldCount: input.requestedFields.length,
      decision: input.requestedDecision,
      ts: now.toISOString(),
    });

    return cap;
  }
}
```

注意最后的 `policy.assertCurrentAccess`：Delegation Grant 不是免死金牌。它只证明“曾经被委托”，真正读取前还要服从当前策略、事故冻结、租户状态和证据撤销。

---

## OpenClaw 实战：给 sub-agent 的不是证据，而是 grant

在 OpenClaw 里，主 Agent 派发子任务时很容易把上下文塞太多。更安全的模式是：

```json
{
  "task": "复核这次 git push 的证据包是否完整，只输出缺口摘要",
  "context": {
    "evidenceDelegationGrant": "egd_xxx",
    "purpose": "audit_review",
    "allowedOutput": "summary_only",
    "forbidden": ["raw_payload", "secret", "personal_data"]
  }
}
```

子 Agent 只能通过受控工具读取：

```text
redeem_evidence_delegation(grant_id="egd_xxx", fields=["gate", "hash", "summary"])
```

不要这样做：

```text
把完整 evidence JSON 直接贴进 sub-agent prompt
```

因为 prompt、日志、重试、错误报告、消息转发都会变成新的扩散面。

---

## 设计检查清单

做 Evidence Delegation 时，至少检查：

- grant 是否绑定 `issuer / delegate / tenant / purpose / evidenceId`？
- delegate 权限是否严格小于等于 issuer 当前权限？
- TTL 是否短，是否有 `maxReads`？
- raw access 是否默认禁止，只允许 summary/redacted？
- policy 更新、incident freeze、evidence revocation 是否能让旧 grant 失效？
- redeem 是否写审计事件？
- 子 Agent 输出是否再次做 redaction / citation guard？

---

## 一句话总结

**Evidence Delegation 的本质：不要把敏感事实复制给别人，而是发一张短期、限权、可撤销、可审计的“查看证”。**

成熟 Agent 协作不是互相传大段上下文，而是在最小必要范围内共享能力，并且每一次使用都有边界、有记录、有撤销路径。
