# 332. Agent 证据隔离区与安全复核（Evidence Quarantine & Safe Review）

前几讲我们已经把 Evidence 做到了可授权、可委托、可追踪、可限流披露。今天补一个生产系统里很关键的能力：**证据隔离区**。

一句话：当一条证据“可能有毒、可能错、可能泄密、可能来自不可信来源”时，不要直接删，也不要继续给 Agent 用；应该先放进 **Quarantine（隔离区）**，只允许安全复核流程读取，复核通过才释放，复核失败就撤销并传播失效。

这解决的是成熟 Agent 的一个现实问题：

- 网页抓取内容可能带 prompt injection；
- 工具返回可能包含 secret / PII；
- 子 Agent 上交的证据可能 scope 不匹配；
- policy 版本升级后，旧证据可能不再满足要求；
- 异常检测发现某证据被高频 raw 读取，需要先冻结。

如果没有 Quarantine，Agent 常见错误是：**发现异常，但证据已经进入上下文、缓存、审计报表和后续决策链，污染已经扩散。**

---

## 1. 核心模型：Evidence 不只有 valid / invalid，还要有 quarantined

不要把证据状态设计成布尔值。

```python
# learn-claude-code: evidence_quarantine.py
from dataclasses import dataclass, field
from enum import Enum
from time import time
from typing import Any

class EvidenceState(str, Enum):
    ACTIVE = "active"            # 可用于正常 proof / decision
    QUARANTINED = "quarantined"  # 只允许 safe review 读取
    RELEASED = "released"        # 复核后重新激活
    REVOKED = "revoked"          # 不可再用于下游证明

@dataclass
class EvidenceEnvelope:
    id: str
    tenant: str
    kind: str
    payload: dict[str, Any]
    hash: str
    state: EvidenceState = EvidenceState.ACTIVE
    risk_reasons: list[str] = field(default_factory=list)
    quarantined_at: float | None = None
    quarantined_by: str | None = None
    review_deadline: float | None = None
    parent_ids: list[str] = field(default_factory=list)
```

关键点：

1. **quarantined 不是 delete**：证据仍然保留，方便审计、复核、追责；
2. **quarantined 不是 active**：不能继续用于行动证明、策略放行、上下文注入；
3. **review_deadline 必须存在**：隔离区不是垃圾堆，超期要升级；
4. **risk_reasons 要结构化**：不要只写“有问题”，要写 `prompt_injection_suspected`、`secret_detected`、`scope_mismatch`。

---

## 2. Quarantine Gate：写入后立刻扫描，命中就隔离

最简单的教学版：证据进入 store 前后跑风险扫描。

```python
# learn-claude-code: quarantine_gate.py
import re
from time import time

SECRET_PATTERNS = [
    re.compile(r"sk-[A-Za-z0-9]{20,}"),
    re.compile(r"AKIA[0-9A-Z]{16}"),
    re.compile(r"BEGIN (RSA|OPENSSH) PRIVATE KEY"),
]

INJECTION_HINTS = [
    "ignore previous instructions",
    "disregard your system prompt",
    "你现在是",
    "忽略之前的指令",
]

def scan_evidence(e: EvidenceEnvelope) -> list[str]:
    text = str(e.payload).lower()
    reasons: list[str] = []

    if any(p.search(str(e.payload)) for p in SECRET_PATTERNS):
        reasons.append("secret_detected")

    if any(hint in text for hint in INJECTION_HINTS):
        reasons.append("prompt_injection_suspected")

    if e.kind == "external_web_page" and not e.payload.get("sourceUrl"):
        reasons.append("missing_source_url")

    return reasons

def quarantine_if_needed(e: EvidenceEnvelope, actor: str) -> EvidenceEnvelope:
    reasons = scan_evidence(e)
    if not reasons:
        return e

    e.state = EvidenceState.QUARANTINED
    e.risk_reasons = reasons
    e.quarantined_at = time()
    e.quarantined_by = actor
    e.review_deadline = time() + 24 * 3600
    return e
```

这里的原则是：**宁可先隔离，再复核释放；不要先扩散，事后追悔。**

---

## 3. 读取闸门：正常 Agent 不能读 quarantined payload

隔离真正有用的地方，不是 state 字段，而是读路径强制执行。

```python
class EvidenceAccessDenied(Exception):
    pass

SAFE_REVIEW_PURPOSES = {"security_review", "incident_review", "evidence_release_review"}

def read_evidence(e: EvidenceEnvelope, *, actor: str, purpose: str, raw: bool = False):
    if e.state == EvidenceState.QUARANTINED:
        if purpose not in SAFE_REVIEW_PURPOSES:
            raise EvidenceAccessDenied(
                f"evidence {e.id} is quarantined; purpose={purpose} cannot read payload"
            )

        # 复核场景也默认最小披露，raw 需要更高权限
        if not raw:
            return {
                "id": e.id,
                "state": e.state,
                "kind": e.kind,
                "hash": e.hash,
                "riskReasons": e.risk_reasons,
                "quarantinedAt": e.quarantined_at,
            }

    if e.state == EvidenceState.REVOKED:
        raise EvidenceAccessDenied(f"evidence {e.id} has been revoked")

    return e.payload
```

这跟前几讲的最小披露、访问授权、披露预算可以组合：

- normal decision：只能读 active/released；
- safe review：可读 quarantine summary；
- senior reviewer / incident commander：在审计下读 raw；
- sub-agent：默认不能读 quarantined raw。

---

## 4. Release / Revoke：复核结果必须产生新证据事件

不要直接把 `state = active` 改回去就完事。复核本身也是审计事实。

```python
@dataclass
class ReviewDecision:
    evidence_id: str
    reviewer: str
    decision: str  # release | revoke
    reason: str
    reviewed_at: float
    proof_hash: str | None = None

def apply_review(e: EvidenceEnvelope, d: ReviewDecision) -> EvidenceEnvelope:
    if e.state != EvidenceState.QUARANTINED:
        raise ValueError("only quarantined evidence can be reviewed")

    if d.decision == "release":
        # release 不等于忘记历史，risk_reasons 保留用于后续审计
        e.state = EvidenceState.RELEASED
        return e

    if d.decision == "revoke":
        e.state = EvidenceState.REVOKED
        return e

    raise ValueError(f"unknown decision: {d.decision}")
```

生产版还要追加 `EvidenceReviewEvent`：

```json
{
  "type": "evidence.reviewed",
  "evidenceId": "ev_123",
  "fromState": "quarantined",
  "toState": "revoked",
  "reviewer": "security-bot",
  "reason": "secret_detected_confirmed",
  "proofHash": "sha256:...",
  "at": "2026-05-16T05:30:00.000Z"
}
```

---

## 5. pi-mono：做成 EvidenceQuarantineMiddleware

在 pi-mono 这种 TypeScript Agent Runtime 里，建议把 quarantine 放在两条路径：

1. **evidence.write 后置扫描**：写入证据时发现风险，立刻标记 quarantined；
2. **evidence.read 前置拦截**：任何读取都先过 quarantine gate。

```ts
// pi-mono: EvidenceQuarantineMiddleware.ts
type EvidenceState = 'active' | 'quarantined' | 'released' | 'revoked'

type EvidenceEnvelope = {
  id: string
  tenant: string
  kind: string
  payloadRef: string
  hash: string
  state: EvidenceState
  riskReasons: string[]
  quarantinedAt?: string
  reviewDeadline?: string
}

type EvidenceReadRequest = {
  evidenceId: string
  actor: string
  tenant: string
  purpose: 'decision' | 'proof' | 'security_review' | 'incident_review'
  viewMode: 'summary' | 'raw'
}

export class EvidenceQuarantineMiddleware {
  constructor(
    private readonly store: EvidenceStore,
    private readonly scanner: EvidenceRiskScanner,
    private readonly audit: AuditLog,
  ) {}

  async afterWrite(evidenceId: string, actor: string) {
    const meta = await this.store.getMeta(evidenceId)
    const summary = await this.store.getSafeSummary(evidenceId)
    const reasons = await this.scanner.scan({ meta, summary })

    if (reasons.length === 0) return

    await this.store.transition(evidenceId, {
      to: 'quarantined',
      riskReasons: reasons,
      quarantinedAt: new Date().toISOString(),
      reviewDeadline: new Date(Date.now() + 24 * 3600_000).toISOString(),
    })

    await this.audit.append({
      type: 'evidence.quarantined',
      evidenceId,
      actor,
      reasons,
    })
  }

  async beforeRead(req: EvidenceReadRequest) {
    const meta = await this.store.getMeta(req.evidenceId)

    if (meta.tenant !== req.tenant) {
      throw new Error('tenant_mismatch')
    }

    if (meta.state === 'revoked') {
      throw new Error('evidence_revoked')
    }

    if (meta.state === 'quarantined') {
      const safePurpose = req.purpose === 'security_review' || req.purpose === 'incident_review'
      if (!safePurpose) throw new Error('evidence_quarantined')

      if (req.viewMode === 'raw') {
        await this.audit.append({
          type: 'quarantined_evidence.raw_read',
          evidenceId: req.evidenceId,
          actor: req.actor,
          purpose: req.purpose,
        })
      }
    }
  }
}
```

注意：`afterWrite` 最好先扫描 **safe summary**，不要为了扫描而把 raw secret 又扩散到更多组件。

---

## 6. OpenClaw 实战：课程 Cron / 子 Agent 的证据隔离

OpenClaw 里最容易落地的场景是：**外部内容进入 Agent 上下文前先证据化，再 quarantine gate**。

比如课程 Cron 需要读取网页、代码、README、历史记忆：

```ts
// OpenClaw pseudo-flow
const evidence = await evidenceStore.append({
  kind: 'external_or_workspace_context',
  tenant: 'agent-course',
  source: { tool: 'read/web_fetch', pathOrUrl },
  payloadRef: rawResultRef,
  summary: compactSummary,
})

await evidenceQuarantine.afterWrite(evidence.id, 'course-cron')

const view = await evidenceStore.read(evidence.id, {
  actor: 'course-cron',
  purpose: 'decision',
  viewMode: 'summary',
})

// 如果被 quarantine，这里会直接阻断，不会把可疑 raw 内容注入 LLM
```

实际规则可以很简单：

- 读到 API key / token / private key → quarantine；
- 外部网页包含“忽略之前指令” → quarantine；
- 子 Agent 返回的 evidence tenant 与当前任务不一致 → quarantine；
- access review 标记异常高频读取 → quarantine；
- incident freeze 命中某 repo / tenant → 新证据默认 quarantine。

这能避免“工具结果一回来就塞进 prompt”，把 prompt injection 和 secret 泄漏挡在上下文外面。

---

## 7. Quarantine 和 Revocation 的区别

很多人会把二者混成一个状态。

| 状态 | 含义 | 能否恢复 | 常见原因 |
|---|---|---:|---|
| quarantined | 暂时隔离，等待复核 | 可以 release 或 revoke | 可疑注入、疑似 secret、scope 异常 |
| revoked | 已确认失效，不可用于证明 | 通常不可恢复，只能新证据替代 | 用户撤销、证据伪造、凭证轮换、复核失败 |
| released | 曾隔离但复核通过 | 可再次 quarantine | 误报、已脱敏、source verified |

原则：

> Quarantine 是“先别用”；Revocation 是“确认不能用”。

---

## 8. 最小实现 Checklist

如果今天就要给 Agent 加证据隔离区，至少做这 6 件事：

- [ ] Evidence state 增加 `quarantined/released/revoked`
- [ ] 写入证据后跑 secret / injection / scope 扫描
- [ ] 普通 decision/proof 读取禁止 quarantined payload
- [ ] safe review purpose 只能默认读 summary
- [ ] release/revoke 必须写 ReviewDecision / AuditEvent
- [ ] revoked/quarantined 父证据要阻断下游 proof chain

---

## 9. 一句话总结

成熟 Agent 不应该假设所有证据都干净可信。**Evidence Quarantine 把“发现可疑”变成系统级安全状态：先隔离、限权复核、审计释放或撤销，再决定能不能进入后续行动链。**
