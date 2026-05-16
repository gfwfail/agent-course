# 331 - Agent 证据披露预算与限流视图：Evidence Disclosure Budget & Rate-Limited Views

> 授权回答“这次能不能看”，审计回答“看过什么”，披露预算回答“同一个人、同一个目的，短时间内最多能看多少”。

前面我们已经讲了 Evidence 授权、访问审计、委托共享和水印追踪。今天补一个很容易被忽略、但生产里非常实用的能力：**Evidence Disclosure Budget & Rate-Limited Views**。

很多泄漏不是一次越权造成的，而是“每次都合法，但累计起来过量”：

- 审计员一次只能看 `summary_only`，但 10 分钟扫了 500 条敏感证据；
- 子 Agent 每次拿到 3 个字段，但通过循环把完整 payload 拼回来了；
- Debug 工具每次都带合法 purpose，但同一 actor 对多个 tenant 做了宽范围扫描；
- LLM 上下文里多次注入相似证据，最后超过最小披露边界。

所以 Evidence View Pipeline 里除了 `AccessGate`，还要有一个 **DisclosureBudgetGate**：按 `actor + tenant + purpose + sensitivity + field` 统计披露量，超预算就自动降级、拒绝或要求人工复核。

---

## 核心设计

把“披露”当成一种可计量资源，而不是一次性的 allow/deny：

```text
Evidence read request
  ↓
AccessGate：当前 actor/purpose 是否有权限？
  ↓
Projection：最多允许哪些字段？raw / summary / metadata？
  ↓
DisclosureBudgetGate：最近窗口内是否已经披露过量？
  ↓
Watermark + Audit：发放视图、写入 budget debit 和 access event
```

预算维度建议至少包括：

1. **actorId**：谁在看。
2. **tenantId / subjectId**：看谁的数据。
3. **purpose**：为了什么看，比如 `incident_review`、`debug`、`training`。
4. **sensitivity**：public/internal/sensitive/secret。
5. **viewMode**：metadata、summary、redacted、raw。
6. **fieldClass**：只看订单号和看邮箱/IP/密钥，成本不一样。
7. **time window**：例如 10 分钟、1 小时、24 小时。

关键原则：

- **raw 永远最贵**：raw payload 的预算应该非常小，甚至默认 0。
- **summary 也要计费**：摘要不是无风险，尤其是多次摘要可以拼事实。
- **超预算先降级**：能用 `summary_only` 就不要直接 deny；高风险才 block。
- **预算扣减要审计**：每次披露都写 ledger，方便复盘和异常检测。
- **break-glass 单独预算**：应急访问要短 TTL、强原因、事后复核。

---

## learn-claude-code：最小披露预算闸门

教学版可以用一个内存/SQLite ledger 实现滑动窗口预算。重点不是存储，而是把“视图发放”变成一次 budget debit。

```python
# learn-claude-code/disclosure_budget.py
from __future__ import annotations

import time
from dataclasses import dataclass
from typing import Literal

ViewMode = Literal["metadata", "summary", "redacted", "raw"]
Sensitivity = Literal["public", "internal", "sensitive", "secret"]

COST = {
    "metadata": 1,
    "summary": 3,
    "redacted": 8,
    "raw": 25,
}

DEFAULT_LIMITS = {
    ("debug", "sensitive"): 30,
    ("incident_review", "sensitive"): 80,
    ("training", "sensitive"): 0,   # 训练默认不能读敏感证据
    ("debug", "secret"): 0,
}

@dataclass(frozen=True)
class DisclosureRequest:
    actor_id: str
    tenant_id: str
    purpose: str
    sensitivity: Sensitivity
    view_mode: ViewMode
    field_classes: tuple[str, ...]
    evidence_id: str

class DisclosureLedger:
    def __init__(self):
        self.events: list[dict] = []

    def used(self, req: DisclosureRequest, window_seconds: int) -> int:
        cutoff = time.time() - window_seconds
        return sum(
            e["cost"] for e in self.events
            if e["ts"] >= cutoff
            and e["actorId"] == req.actor_id
            and e["tenantId"] == req.tenant_id
            and e["purpose"] == req.purpose
            and e["sensitivity"] == req.sensitivity
        )

    def debit(self, req: DisclosureRequest, cost: int, decision: str):
        self.events.append({
            "ts": time.time(),
            "actorId": req.actor_id,
            "tenantId": req.tenant_id,
            "purpose": req.purpose,
            "sensitivity": req.sensitivity,
            "viewMode": req.view_mode,
            "fieldClasses": list(req.field_classes),
            "evidenceId": req.evidence_id,
            "cost": cost,
            "decision": decision,
        })


def disclosure_budget_gate(req: DisclosureRequest, ledger: DisclosureLedger):
    limit = DEFAULT_LIMITS.get((req.purpose, req.sensitivity), 20)
    cost = COST[req.view_mode] + len(req.field_classes)
    used = ledger.used(req, window_seconds=3600)

    if limit == 0:
        return {"action": "deny", "reason": "purpose_has_zero_budget", "used": used, "limit": limit}

    if used + cost <= limit:
        ledger.debit(req, cost, "allow")
        return {"action": "allow", "viewMode": req.view_mode, "used": used + cost, "limit": limit}

    # 超预算时先尝试降级，而不是直接失败
    if req.view_mode in ("raw", "redacted"):
        downgraded = DisclosureRequest(**{**req.__dict__, "view_mode": "summary"})
        downgraded_cost = COST["summary"] + min(len(req.field_classes), 2)
        if used + downgraded_cost <= limit:
            ledger.debit(downgraded, downgraded_cost, "downgrade_to_summary")
            return {"action": "downgrade", "viewMode": "summary", "used": used + downgraded_cost, "limit": limit}

    return {"action": "deny", "reason": "disclosure_budget_exceeded", "used": used, "limit": limit}
```

业务代码不要绕过这个 gate：

```python
def read_evidence_view(evidence_store, ledger, req: DisclosureRequest):
    # 1. AccessGate 已确认 req.actor_id 有当前 purpose 的基本权限
    decision = disclosure_budget_gate(req, ledger)
    if decision["action"] == "deny":
        raise PermissionError(decision)

    evidence = evidence_store.get(req.evidence_id)
    if decision["action"] == "downgrade":
        return evidence.to_summary()

    return evidence.project(req.view_mode, req.field_classes)
```

---

## pi-mono：DisclosureBudgetMiddleware

生产版建议把预算做成中间件，挂在 Evidence View Pipeline 里：

```typescript
// pi-mono/src/evidence/DisclosureBudget.ts
export type ViewMode = "metadata" | "summary" | "redacted" | "raw";
export type BudgetAction = "allow" | "downgrade" | "deny" | "require_approval";

export interface DisclosureBudgetRequest {
  actorId: string;
  tenantId: string;
  subjectId?: string;
  purpose: string;
  sensitivity: "public" | "internal" | "sensitive" | "secret";
  requestedViewMode: ViewMode;
  fieldClasses: string[];
  evidenceId: string;
  runId: string;
}

export interface DisclosureBudgetDecision {
  action: BudgetAction;
  effectiveViewMode?: ViewMode;
  used: number;
  limit: number;
  cost: number;
  reasonCode: string;
}
```

```typescript
// pi-mono/src/middleware/DisclosureBudgetMiddleware.ts
export class DisclosureBudgetMiddleware {
  constructor(private readonly ledger: DisclosureLedger) {}

  async decide(req: DisclosureBudgetRequest): Promise<DisclosureBudgetDecision> {
    const policy = await this.ledger.policyFor({
      actorId: req.actorId,
      tenantId: req.tenantId,
      purpose: req.purpose,
      sensitivity: req.sensitivity,
    });

    const used = await this.ledger.usedInWindow(req, policy.windowSeconds);
    const cost = this.costOf(req.requestedViewMode, req.fieldClasses);

    if (policy.limit === 0) {
      return { action: "deny", used, limit: 0, cost, reasonCode: "zero_budget" };
    }

    if (used + cost <= policy.limit) {
      await this.ledger.recordDebit(req, cost, "allow");
      return {
        action: "allow",
        effectiveViewMode: req.requestedViewMode,
        used: used + cost,
        limit: policy.limit,
        cost,
        reasonCode: "within_budget",
      };
    }

    if (req.requestedViewMode === "raw") {
      const summaryCost = this.costOf("summary", req.fieldClasses.slice(0, 2));
      if (used + summaryCost <= policy.limit) {
        await this.ledger.recordDebit(req, summaryCost, "downgrade_to_summary");
        return {
          action: "downgrade",
          effectiveViewMode: "summary",
          used: used + summaryCost,
          limit: policy.limit,
          cost: summaryCost,
          reasonCode: "raw_budget_exceeded_summary_available",
        };
      }
    }

    return {
      action: policy.breakGlassAllowed ? "require_approval" : "deny",
      used,
      limit: policy.limit,
      cost,
      reasonCode: "budget_exceeded",
    };
  }

  private costOf(mode: ViewMode, fields: string[]): number {
    const base = { metadata: 1, summary: 3, redacted: 8, raw: 25 }[mode];
    const fieldCost = fields.reduce((sum, f) => sum + (f === "secret" ? 20 : f === "pii" ? 5 : 1), 0);
    return base + fieldCost;
  }
}
```

这个中间件应该输出结构化 reason，而不是只返回布尔值。后续可以把 `reasonCode=budget_exceeded` 接到 incident、异常检测或人工审批。

---

## OpenClaw 实战：子 Agent 证据视图不要无限发

OpenClaw 里常见场景是主 Agent 委托子 Agent：

```text
主 Agent：请子 Agent 分析 20 条部署证据
子 Agent：需要读取 release evidence
风险：如果每条都给 raw log，可能把 token、邮箱、内部 URL 全塞进子 Agent 上下文
```

更稳的做法：

```json
{
  "actorId": "subagent:release-reviewer",
  "tenantId": "workspace:agent-course",
  "purpose": "release_review",
  "budget": {
    "windowSeconds": 3600,
    "limit": 60,
    "rawLimit": 0,
    "summaryLimit": 20
  },
  "allowedFields": ["commit", "diffstat", "testSummary", "deployStatus"],
  "deniedFields": ["secret", "token", "fullEnv", "rawHttpHeaders"]
}
```

Pipeline 可以这样执行：

```text
1. 子 Agent 请求 evidence view
2. 主 Agent / gateway 检查 delegation grant
3. AccessGate 决定最多 summary_only
4. DisclosureBudgetGate 扣减 summary 成本
5. Redactor 移除 token/raw headers
6. Watermark 写 traceId
7. AuditLedger 记录 view + budget debit
```

这样即使子 Agent 出 bug 循环请求，也会被预算挡住；即使内容泄漏，也能通过上一讲的 watermark 反查。

---

## 常见坑

1. **只限流请求次数，不限流字段成本**  
   读 1 次 raw secret 和读 1 次 metadata 风险完全不同，必须按 viewMode/fieldClass 计费。

2. **只在 API 层限流，内部工具绕过**  
   Agent 内部工具、子 Agent、debug 命令都要走同一个 Evidence View Pipeline。

3. **超预算直接失败，导致人绕系统**  
   先尝试降级到 summary/metadata；只有 secret/raw 或高风险 purpose 才 require approval / deny。

4. **预算没有审计**  
   预算扣减本身也是证据。没有 ledger，就无法解释为什么某次被拒绝或为什么某人看了这么多。

5. **预算不随 incident 改变**  
   进入 incident/freeze 状态后，raw budget 应立即降到 0，summary budget 降级，break-glass 需要更强审批。

---

## 一句话总结

**成熟 Agent 的 Evidence 系统，不只是“能不能看”，还要控制“看多少、看多细、多久内看、超了怎么办”。**

Disclosure Budget 把敏感证据从静态权限资产，升级成可计量、可限流、可降级、可审计的运行时资源。授权防越权，预算防“合法但过量”的数据外流。
