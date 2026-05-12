# 304. Agent 运维知识晋级流水线与灰度发布（Knowledge Promotion Pipeline & Staged Rollout）

> 核心观点：运维知识不要一写进 KB 就全量生效。成熟 Agent 要把知识当成“会改变行为的配置发布”：先 draft，经过验证进入 candidate，再小流量 canary，最后 promote 到 active；一旦指标异常，自动降级或回滚。

前几课我们讲了：

- Runbook 漂移检测：执行前确认依赖没过期；
- 知识生命周期：draft / active / stale / archived；
- 变更影响分析：知识变更要 replay regression；
- 版本固定与回滚：高风险 Runbook 要 pin 到已验证 revision。

今天补上中间最关键的一环：**知识如何安全上线**。

---

## 1. 为什么需要 Knowledge Promotion Pipeline？

很多 Agent 系统的事故不是“没知识”，而是：

1. 新复盘写进 MEMORY / KB 后，Agent 立刻改变行为；
2. 新规则只在一个案例里合理，但影响了其他场景；
3. 新知识和旧 Runbook 没有一起验证；
4. 出问题后不知道是哪个知识 revision 造成的。

所以知识发布要像代码发布一样走流水线：

```text
Draft → Validate → Candidate → Canary → Active
        ↓ failed       ↓ bad metric     ↓ incident
      Rejected       Rollback          Deprecate
```

重点不是流程好看，而是每一步都有**证据**：测试结果、影响范围、canary 指标、pin manifest、rollback target。

---

## 2. 最小数据模型

一条运维知识不要只有 Markdown 文本，至少要有这些字段：

```json
{
  "id": "kb.cache.tombstone.delete-race",
  "revision": 7,
  "status": "candidate",
  "risk": "high",
  "scope": {
    "services": ["cache"],
    "operationTypes": ["delete", "refresh"],
    "tenants": ["canary"]
  },
  "claim": "删除缓存后必须写 tombstone，阻止旧 refresh 复活数据",
  "checks": [
    "regression.cache.delete_race",
    "policy.no_secret_in_prompt",
    "runbook.cache-delete.verify"
  ],
  "canary": {
    "trafficPercent": 5,
    "startedAt": "2026-05-13T17:30:00Z",
    "successThreshold": 0.995,
    "maxPolicyDenyRate": 0.02
  },
  "promoteAfter": "2026-05-14T17:30:00Z",
  "rollbackTo": 6
}
```

这里有三个设计点：

- `status` 控制它能不能被 Agent 检索进执行上下文；
- `scope` 控制它影响哪些服务 / 操作 / 租户；
- `checks + canary` 控制它能不能晋级。

---

## 3. learn-claude-code：文件式 Knowledge Promoter

教学版用 JSON 文件就够了：`ops_kb/*.json` 存知识，`promotion_events.jsonl` 存审计事件。

```python
# learn-claude-code/examples/knowledge_promoter.py
from __future__ import annotations

import json
from dataclasses import dataclass
from pathlib import Path
from typing import Literal

Status = Literal["draft", "candidate", "canary", "active", "rejected", "rolled_back"]


@dataclass
class GateResult:
    ok: bool
    reason: str
    evidence: dict


class KnowledgePromoter:
    def __init__(self, kb_dir: Path, event_log: Path):
        self.kb_dir = kb_dir
        self.event_log = event_log

    def load(self, knowledge_id: str) -> dict:
        return json.loads((self.kb_dir / f"{knowledge_id}.json").read_text())

    def save(self, entry: dict) -> None:
        path = self.kb_dir / f"{entry['id']}.json"
        tmp = path.with_suffix(".tmp")
        tmp.write_text(json.dumps(entry, ensure_ascii=False, indent=2))
        tmp.replace(path)

    def append_event(self, event: dict) -> None:
        with self.event_log.open("a") as f:
            f.write(json.dumps(event, ensure_ascii=False) + "\n")

    def validate(self, entry: dict, regression_results: dict[str, bool]) -> GateResult:
        missing = [check for check in entry.get("checks", []) if check not in regression_results]
        failed = [check for check in entry.get("checks", []) if regression_results.get(check) is False]

        if missing:
            return GateResult(False, "missing_required_checks", {"missing": missing})
        if failed:
            return GateResult(False, "regression_failed", {"failed": failed})
        if entry.get("risk") == "high" and not entry.get("rollbackTo"):
            return GateResult(False, "high_risk_requires_rollback_target", {})

        return GateResult(True, "validated", {"checked": entry.get("checks", [])})

    def promote_to_candidate(self, knowledge_id: str, regression_results: dict[str, bool]) -> GateResult:
        entry = self.load(knowledge_id)
        result = self.validate(entry, regression_results)

        if not result.ok:
            entry["status"] = "rejected"
            self.save(entry)
            self.append_event({"type": "knowledge.rejected", "id": knowledge_id, **result.__dict__})
            return result

        entry["status"] = "candidate"
        self.save(entry)
        self.append_event({"type": "knowledge.candidate", "id": knowledge_id, **result.__dict__})
        return result

    def start_canary(self, knowledge_id: str, percent: int) -> GateResult:
        entry = self.load(knowledge_id)
        if entry["status"] != "candidate":
            return GateResult(False, "only_candidate_can_start_canary", {"status": entry["status"]})
        if percent <= 0 or percent > 25:
            return GateResult(False, "unsafe_canary_percent", {"percent": percent})

        entry["status"] = "canary"
        entry.setdefault("canary", {})["trafficPercent"] = percent
        self.save(entry)
        self.append_event({"type": "knowledge.canary_started", "id": knowledge_id, "percent": percent})
        return GateResult(True, "canary_started", {"percent": percent})
```

这个版本故意很朴素，但已经有三个关键护栏：

- required checks 缺失不能晋级；
- high risk 必须有 rollback target；
- canary 百分比有硬上限。

---

## 4. pi-mono：运行时只注入允许阶段的知识

生产系统里，知识检索层要尊重 promotion 状态。不要让 `draft` 被注入 prompt，也不要让 `candidate` 影响全量用户。

```ts
// pi-mono/packages/agent-runtime/src/knowledge/KnowledgePromotionMiddleware.ts
export type KnowledgeStatus =
  | 'draft'
  | 'candidate'
  | 'canary'
  | 'active'
  | 'rejected'
  | 'rolled_back';

export interface KnowledgeEntry {
  id: string;
  revision: number;
  status: KnowledgeStatus;
  risk: 'low' | 'medium' | 'high';
  scope: {
    services?: string[];
    operationTypes?: string[];
    tenants?: string[];
  };
  canary?: {
    trafficPercent: number;
    salt: string;
  };
  claim: string;
}

export interface KnowledgeRequest {
  tenantId: string;
  service: string;
  operationType: string;
  actorId: string;
}

function stableBucket(input: string): number {
  let hash = 2166136261;
  for (const ch of input) {
    hash ^= ch.charCodeAt(0);
    hash = Math.imul(hash, 16777619);
  }
  return Math.abs(hash) % 100;
}

function matchesScope(entry: KnowledgeEntry, req: KnowledgeRequest): boolean {
  const s = entry.scope;
  return (!s.services || s.services.includes(req.service))
    && (!s.operationTypes || s.operationTypes.includes(req.operationType))
    && (!s.tenants || s.tenants.includes(req.tenantId));
}

export function canInjectKnowledge(entry: KnowledgeEntry, req: KnowledgeRequest): boolean {
  if (!matchesScope(entry, req)) return false;

  if (entry.status === 'active') return true;

  if (entry.status === 'canary') {
    const percent = entry.canary?.trafficPercent ?? 0;
    const salt = entry.canary?.salt ?? `${entry.id}:${entry.revision}`;
    const bucket = stableBucket(`${salt}:${req.tenantId}:${req.actorId}`);
    return bucket < percent;
  }

  // draft/candidate/rejected/rolled_back 默认不进执行上下文
  return false;
}
```

这里的核心是：

- `candidate` 只是“通过静态验证”，不代表能影响线上；
- `canary` 用稳定哈希分桶，避免同一租户一会儿命中一会儿不命中；
- `active` 才能全量注入。

---

## 5. Canary 指标门：不是时间到了就晋级

灰度发布最容易犯的错是：过了 24 小时就自动 promote。正确做法是：**时间 + 指标 + 证据**同时满足。

```ts
// pi-mono/packages/ops/src/knowledge/PromotionGate.ts
interface CanaryMetrics {
  evaluatedRuns: number;
  successfulRuns: number;
  policyDenyRate: number;
  rollbackCount: number;
  incidentCount: number;
}

interface PromotionDecision {
  decision: 'promote' | 'hold' | 'rollback';
  reason: string;
  evidence: Record<string, unknown>;
}

export function decidePromotion(metrics: CanaryMetrics): PromotionDecision {
  if (metrics.incidentCount > 0 || metrics.rollbackCount > 0) {
    return {
      decision: 'rollback',
      reason: 'canary_caused_incident_or_rollback',
      evidence: metrics,
    };
  }

  if (metrics.evaluatedRuns < 100) {
    return {
      decision: 'hold',
      reason: 'not_enough_canary_samples',
      evidence: metrics,
    };
  }

  const successRate = metrics.successfulRuns / metrics.evaluatedRuns;
  if (successRate < 0.995 || metrics.policyDenyRate > 0.02) {
    return {
      decision: 'hold',
      reason: 'quality_gate_not_met',
      evidence: { ...metrics, successRate },
    };
  }

  return {
    decision: 'promote',
    reason: 'canary_metrics_healthy',
    evidence: { ...metrics, successRate },
  };
}
```

注意：`hold` 不是失败，它表示样本不够或信号还不稳。Agent 自动化里不要急着“完成任务”，稳定比速度重要。

---

## 6. OpenClaw 实战：文件式 promotion manifest

OpenClaw 里可以把 promotion 状态放到 workspace 文件，Cron/Heartbeat 定时评估：

```json
// ops-kb/promotion-manifest.json
{
  "updatedAt": "2026-05-13T17:30:00Z",
  "entries": {
    "kb.cache.tombstone.delete-race": {
      "revision": 7,
      "status": "canary",
      "trafficPercent": 5,
      "rollbackTo": 6,
      "requiredEvidence": [
        "regression-report.json",
        "canary-metrics.json",
        "pin-manifest.hash"
      ]
    }
  }
}
```

执行上下文注入前，OpenClaw 可以做一个小 gate：

```python
# openclaw-workspace/scripts/select_ops_knowledge.py
import json
from pathlib import Path

manifest = json.loads(Path("ops-kb/promotion-manifest.json").read_text())

def allowed(entry_id: str, tenant: str) -> bool:
    item = manifest["entries"].get(entry_id)
    if not item:
        return False
    if item["status"] == "active":
        return True
    if item["status"] != "canary":
        return False

    # 教学版：用 hash 做稳定分桶
    bucket = abs(hash(f"{entry_id}:{tenant}")) % 100
    return bucket < item.get("trafficPercent", 0)
```

这让知识发布变成可审计、可回滚、可灰度的流程，而不是“编辑一个文件，祈祷 Agent 别学坏”。

---

## 7. 常见坑

### 坑 1：把 draft 知识放进 prompt

草稿可以给人看，但不能直接影响执行。否则“没审核的想法”会变成线上行为。

### 坑 2：只验证 happy path

知识变更最危险的是副作用场景：删除、部署、发消息、改权限。Regression 必须覆盖这些路径。

### 坑 3：没有 rollback target

没有 `rollbackTo` 的高风险知识，不能上线。否则坏知识上线后只能手工猜怎么退。

### 坑 4：Canary 不稳定分桶

如果每次随机命中，用户体验和指标都会抖。必须用 tenant/actor 稳定 hash。

---

## 8. 一句话总结

**知识会改变 Agent 行为，所以知识发布就是一次生产变更。**

把 draft、candidate、canary、active、rollback 做成流水线，Agent 才能既持续学习，又不把每次学习都变成线上赌博。
