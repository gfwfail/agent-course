# 328 - Agent 证据访问审计与异常检测：Evidence Access Review & Anomaly Detection

> 证据访问授权解决“这次能不能看”，访问审计与异常检测解决“看得是否正常、是否该复盘”。

上一讲我们把 Evidence 读取做成了按 actor / tenant / purpose / scope 授权的安全动作。但成熟系统不能只在读取瞬间做 allow/deny：

- 某个 Agent 是否突然大量读取敏感证据？
- 某个 purpose 是否被滥用成“万能理由”？
- 是否有人在事故后反复读取同一租户的 raw evidence？
- allow 的访问后来是否证明是误授权？

这就需要 **Evidence Access Review & Anomaly Detection**：每次证据读取都留下最小披露审计事件，再用规则和统计模型找异常，最后进入人工复核或自动降级。

---

## 核心思路

```
Evidence Read Request
        ↓
Access Policy 决策（deny / summary_only / raw）
        ↓
写 AccessAuditEvent（不写敏感原文）
        ↓
Anomaly Detector 扫描窗口
        ↓
review / throttle / degrade / revoke capability
```

关键点：

1. **审计日志不存 payload**：只存 evidenceId hash、sensitivity、purpose、decision、actor、tenant、字段级披露摘要。
2. **allow 也必须记录**：异常往往发生在“合法但不寻常”的访问里。
3. **检测不等于直接封禁**：低风险进入 review，高风险才自动降级或撤权。
4. **purpose 要可统计**：`debug_incident`、`answer_user`、`external_side_effect` 不能混在一起。

---

## learn-claude-code：文件式访问审计 + 异常扫描

教学版可以先用 JSONL 做 append-only 审计日志。

```python
# learn-claude-code/evidence_access_audit.py
from __future__ import annotations

import hashlib
import json
import time
from collections import Counter, defaultdict
from dataclasses import asdict, dataclass
from pathlib import Path
from typing import Literal

Decision = Literal["deny", "summary_only", "raw"]
Sensitivity = Literal["public", "internal", "sensitive", "secret"]


def sha256_short(value: str) -> str:
    return hashlib.sha256(value.encode("utf-8")).hexdigest()[:16]


@dataclass(frozen=True)
class AccessAuditEvent:
    ts: float
    run_id: str
    actor: str
    tenant: str
    purpose: str
    evidence_hash: str
    sensitivity: Sensitivity
    decision: Decision
    field_count: int
    reason_codes: list[str]


class EvidenceAccessAuditLog:
    def __init__(self, path: str = "audit/evidence_access.jsonl"):
        self.path = Path(path)
        self.path.parent.mkdir(parents=True, exist_ok=True)

    def append(self, event: AccessAuditEvent) -> None:
        with self.path.open("a", encoding="utf-8") as f:
            f.write(json.dumps(asdict(event), ensure_ascii=False, sort_keys=True) + "\n")

    def recent(self, since_ts: float) -> list[AccessAuditEvent]:
        if not self.path.exists():
            return []
        events: list[AccessAuditEvent] = []
        for line in self.path.read_text(encoding="utf-8").splitlines():
            raw = json.loads(line)
            if raw["ts"] >= since_ts:
                events.append(AccessAuditEvent(**raw))
        return events


def record_evidence_access(
    *,
    audit: EvidenceAccessAuditLog,
    run_id: str,
    actor: str,
    tenant: str,
    purpose: str,
    evidence_id: str,
    sensitivity: Sensitivity,
    decision: Decision,
    exposed_fields: list[str],
    reason_codes: list[str],
) -> None:
    # 注意：不要把 evidence_id 明文、字段值、证据 payload 写进审计日志
    audit.append(AccessAuditEvent(
        ts=time.time(),
        run_id=run_id,
        actor=actor,
        tenant=tenant,
        purpose=purpose,
        evidence_hash=sha256_short(evidence_id),
        sensitivity=sensitivity,
        decision=decision,
        field_count=len(exposed_fields),
        reason_codes=reason_codes,
    ))
```

异常扫描器先从简单规则开始，比一上来引入复杂 ML 更可靠。

```python
# learn-claude-code/evidence_anomaly_detector.py
from collections import Counter, defaultdict
import time

SENSITIVE = {"sensitive", "secret"}


def detect_access_anomalies(events, *, now=None):
    now = now or time.time()
    findings = []

    by_actor = defaultdict(list)
    by_actor_tenant = defaultdict(list)
    purpose_counter = Counter()

    for event in events:
        by_actor[event.actor].append(event)
        by_actor_tenant[(event.actor, event.tenant)].append(event)
        purpose_counter[(event.actor, event.purpose)] += 1

    for actor, actor_events in by_actor.items():
        raw_sensitive = [
            e for e in actor_events
            if e.decision == "raw" and e.sensitivity in SENSITIVE
        ]
        if len(raw_sensitive) >= 5:
            findings.append({
                "severity": "high",
                "actor": actor,
                "kind": "too_many_raw_sensitive_reads",
                "count": len(raw_sensitive),
                "action": "degrade_to_summary_only_and_review",
            })

    for (actor, tenant), scoped_events in by_actor_tenant.items():
        unique_evidence = {e.evidence_hash for e in scoped_events}
        if len(unique_evidence) >= 20:
            findings.append({
                "severity": "medium",
                "actor": actor,
                "tenant": tenant,
                "kind": "wide_evidence_sweep",
                "count": len(unique_evidence),
                "action": "open_access_review",
            })

    for (actor, purpose), count in purpose_counter.items():
        if purpose == "debug_incident" and count >= 30:
            findings.append({
                "severity": "medium",
                "actor": actor,
                "purpose": purpose,
                "kind": "purpose_overuse",
                "count": count,
                "action": "require_incident_id_for_future_reads",
            })

    return findings
```

最小可用门槛：

- 每 15 分钟扫最近 1 小时；
- 每天生成 access review 摘要；
- high severity 自动把 raw 降级为 summary_only，直到复核完成。

---

## pi-mono：EvidenceAccessReviewMiddleware

生产版要把审计写在工具层，而不是靠业务代码自觉调用。

```typescript
// pi-mono/src/middleware/EvidenceAccessReviewMiddleware.ts
type EvidenceDecision = "deny" | "summary_only" | "raw";
type Sensitivity = "public" | "internal" | "sensitive" | "secret";

interface EvidenceReadContext {
  runId: string;
  actor: string;
  tenant: string;
  purpose: string;
  toolName: string;
}

interface EvidenceReadResult {
  evidenceId: string;
  sensitivity: Sensitivity;
  decision: EvidenceDecision;
  exposedFields: string[];
  reasonCodes: string[];
  payload?: unknown;
}

interface AuditSink {
  append(event: Record<string, unknown>): Promise<void>;
}

interface AnomalyState {
  isRawAccessDegraded(actor: string, tenant: string): Promise<boolean>;
}

function hashId(value: string): string {
  // 真实项目用 crypto.createHash("sha256")；这里突出接口形态
  return `sha256:${value.slice(0, 6)}...`;
}

export class EvidenceAccessReviewMiddleware {
  constructor(
    private readonly audit: AuditSink,
    private readonly anomalyState: AnomalyState,
  ) {}

  async afterEvidenceRead(ctx: EvidenceReadContext, result: EvidenceReadResult) {
    const degraded = await this.anomalyState.isRawAccessDegraded(ctx.actor, ctx.tenant);

    let finalResult = result;
    if (degraded && result.decision === "raw" && result.sensitivity !== "public") {
      finalResult = {
        ...result,
        decision: "summary_only",
        exposedFields: result.exposedFields.filter((f) => f.startsWith("summary.")),
        payload: summarizeEvidence(result.payload),
        reasonCodes: [...result.reasonCodes, "ANOMALY_DEGRADED_RAW_ACCESS"],
      };
    }

    await this.audit.append({
      ts: new Date().toISOString(),
      runId: ctx.runId,
      actor: ctx.actor,
      tenant: ctx.tenant,
      purpose: ctx.purpose,
      toolName: ctx.toolName,
      evidenceHash: hashId(result.evidenceId),
      sensitivity: result.sensitivity,
      decision: finalResult.decision,
      fieldCount: finalResult.exposedFields.length,
      reasonCodes: finalResult.reasonCodes,
      // 禁止写 payload / 原始字段值
    });

    return finalResult;
  }
}

function summarizeEvidence(payload: unknown) {
  return {
    summary: "raw evidence temporarily redacted pending access review",
    payloadAvailable: false,
  };
}
```

对应的后台任务可以按滑动窗口聚合：

```typescript
// pi-mono/src/jobs/EvidenceAccessAnomalyJob.ts
export async function runEvidenceAccessAnomalyJob({ auditStore, anomalyStore, notifier }) {
  const events = await auditStore.queryRecent({ windowMinutes: 60 });
  const findings = detectAnomalies(events);

  for (const finding of findings) {
    await anomalyStore.upsertFinding(finding.dedupeKey, finding);

    if (finding.severity === "high") {
      await anomalyStore.degradeRawAccess({
        actor: finding.actor,
        tenant: finding.tenant,
        ttlMinutes: 60,
        reason: finding.kind,
      });
      await notifier.notifySecurityReview(finding);
    }
  }
}
```

---

## OpenClaw 实战：Cron 证据访问复核

OpenClaw 里最容易落地的是定时任务：

1. 读取 `audit/evidence_access.jsonl`；
2. 扫最近 1h / 24h；
3. 生成 `reports/evidence-access-review-YYYY-MM-DD.md`；
4. 只有 high severity 才主动发 Telegram；
5. 无异常保持安静。

示例报告结构：

```markdown
# Evidence Access Review - 2026-05-16

## Summary
- window: 24h
- total_reads: 184
- raw_sensitive_reads: 3
- denied_reads: 12
- findings: 1 medium, 0 high

## Findings
- medium: actor=agent-cron kind=purpose_overuse purpose=debug_incident count=31
  action=require_incident_id_for_future_reads

## Top Purpose
- answer_user: 92
- debug_incident: 31
- external_side_effect: 8

## Notes
- no raw secret payload was logged
- all audit rows contain only metadata/hash
```

---

## 常见坑

### 1. 把审计日志变成另一个泄密点

错误做法：

```json
{"evidenceId":"user-123-bank-statement", "payload":"..."}
```

正确做法：

```json
{"evidenceHash":"8d91...", "sensitivity":"sensitive", "decision":"summary_only"}
```

审计日志的目标是证明“谁在何时因为什么目的访问了哪类证据”，不是复制一份证据。

### 2. 只记录 deny，不记录 allow

真正危险的访问通常是 allow：权限没错，但频率、范围、目的不正常。

### 3. 异常检测直接封死生产

更稳的策略：

- medium：open review / require stronger purpose
- high：raw → summary_only / TTL 降级 / 通知人工
- critical：revoke capability / freeze side effects

---

## 落地检查清单

- [ ] 每次 evidence read 都写 AccessAuditEvent
- [ ] 审计日志不包含 payload、明文 secret、敏感字段值
- [ ] allow / deny / summary_only / raw 全部记录
- [ ] 按 actor、tenant、purpose、sensitivity 做聚合
- [ ] 有 high severity 自动降级策略
- [ ] access review 结果能进入后续 policy / runbook 改进

---

## 一句话总结

Evidence Access Authorization 管住“这次能不能看”；Evidence Access Review 管住“长期看得是否正常”。成熟 Agent 不只会控制证据访问，还会发现合法访问里的异常模式，并把它变成可复核、可降级、可改进的安全闭环。
