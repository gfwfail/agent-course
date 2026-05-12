# 299. Agent 运维知识生命周期与过期清理（Operational Knowledge Lifecycle & Expiry）

上一课讲 Runbook Drift：Runbook 执行前要确认工具、策略、知识库和真实系统没有漂移。

但还有一个更隐蔽的问题：**知识库会腐烂。**

事故复盘、临时绕过、阈值调整、值班经验，一开始都很有用；但如果永不过期，Agent 迟早会被旧知识误导：

- 旧的 workaround 已经不需要了，Agent 还在绕路
- 旧告警阈值只适合小流量，现在会误报
- 旧 owner/team/channel 已经换人
- 临时冻结策略过期了，但 Agent 还当成硬规则
- 事故经验没验证过，却被当成生产事实

今天讲 Operational Knowledge Lifecycle。

一句话：

> 运维知识不是写进去就永久可信，而是要有状态、有效期、验证周期和归档路径。

---

## 1. 为什么 Agent 知识要有生命周期

人类值班会自然忘记过时经验，但 Agent 的文件和数据库不会。

所以成熟 Agent 的运行知识库至少要区分 5 种状态：

```text
draft       : 刚从事故/聊天/日志提取，还不能执行
active      : 已验证，可用于决策和 Runbook
stale       : 过了验证周期，只能作为参考，不能直接驱动副作用
superseded  : 被新知识替代，保留溯源但不再注入上下文
archived    : 历史归档，只供审计/复盘检索
```

关键原则：

- **能执行的知识必须有证据**：source / verifiedAt / verifier
- **临时知识必须有过期时间**：expiresAt / reviewAfter
- **过期知识不能静默进入 prompt**：最多作为低置信参考
- **替代关系要显式记录**：supersededBy，避免两条规则互相打架
- **清理不是删除历史**：active path 移除，audit path 保留

---

## 2. 最小数据模型：KnowledgeLifecycleEntry

```json
{
  "id": "kb_inc_20260512_message_retry",
  "title": "消息投递状态未知时先查回执再重试",
  "kind": "incident_lesson",
  "scope": "tool:message.send",
  "status": "active",
  "confidence": 0.92,
  "source": {
    "incidentId": "inc_20260512_001",
    "reviewId": "review_inc_20260512_001",
    "evidence": ["pr_42", "deploy_20260512_01", "monitor_snapshot_abc"]
  },
  "rule": {
    "trigger": "delivery_status_unknown",
    "decision": "check_receipt_before_retry",
    "risk": "side_effect_duplicate_message"
  },
  "verifiedAt": "2026-05-12T02:30:00Z",
  "reviewAfter": "2026-06-12T02:30:00Z",
  "expiresAt": "2026-08-12T02:30:00Z",
  "supersededBy": null
}
```

这比普通 Markdown 多了几个关键字段：

- `status`：是否允许进入执行路径
- `confidence`：给 Agent 决策时降权用
- `reviewAfter`：到了就要重新验证
- `expiresAt`：过了就自动从 active context 移除
- `supersededBy`：被新规则替代时可追踪

---

## 3. learn-claude-code：Python 教学版

下面是一个最小 Knowledge Lifecycle Manager：负责把知识按时间推进状态，并决定哪些可以注入 Agent 上下文。

```python
# learn_claude_code/knowledge_lifecycle.py
from __future__ import annotations

import time
from dataclasses import dataclass
from typing import Literal

KnowledgeStatus = Literal["draft", "active", "stale", "superseded", "archived"]
UsePurpose = Literal["answer", "planning", "side_effect", "audit"]


@dataclass
class KnowledgeEntry:
    id: str
    title: str
    scope: str
    status: KnowledgeStatus
    confidence: float
    verified_at: float | None
    review_after: float | None
    expires_at: float | None
    superseded_by: str | None = None


class KnowledgeLifecycleManager:
    def refresh_status(self, entry: KnowledgeEntry, now: float | None = None) -> KnowledgeEntry:
        now = now or time.time()

        if entry.status in ("archived", "superseded"):
            return entry

        if entry.superseded_by:
            entry.status = "superseded"
            return entry

        if entry.expires_at and now >= entry.expires_at:
            entry.status = "archived"
            return entry

        if entry.status == "active" and entry.review_after and now >= entry.review_after:
            entry.status = "stale"
            entry.confidence = min(entry.confidence, 0.5)

        return entry

    def can_inject(self, entry: KnowledgeEntry, purpose: UsePurpose) -> bool:
        if purpose == "audit":
            return True

        if entry.status == "active":
            return entry.confidence >= 0.7

        # stale 知识最多用于回答/规划，不能直接驱动外部副作用
        if entry.status == "stale":
            return purpose in ("answer", "planning") and entry.confidence >= 0.4

        return False

    def select_context(self, entries: list[KnowledgeEntry], purpose: UsePurpose) -> list[KnowledgeEntry]:
        refreshed = [self.refresh_status(e) for e in entries]
        allowed = [e for e in refreshed if self.can_inject(e, purpose)]
        return sorted(allowed, key=lambda e: e.confidence, reverse=True)
```

使用：

```python
now = time.time()
entries = [
    KnowledgeEntry(
        id="kb_retry_receipt",
        title="消息状态未知时先查回执",
        scope="tool:message.send",
        status="active",
        confidence=0.92,
        verified_at=now - 20 * 86400,
        review_after=now - 1,
        expires_at=now + 60 * 86400,
    )
]

manager = KnowledgeLifecycleManager()
print(manager.select_context(entries, purpose="answer"))       # 可用，但变 stale 降权
print(manager.select_context(entries, purpose="side_effect"))  # 不可用，副作用必须 fresh
```

核心点：**同一条知识在不同 purpose 下权限不同。**

回答用户时，stale 知识可以提示“可能如此”；执行生产副作用时，stale 知识必须先重新验证。

---

## 4. pi-mono：TypeScript 生产版中间件

生产环境建议把 lifecycle 检查做成 Knowledge Store 的读写中间件，不要让业务代码自己判断。

```ts
// packages/runtime/src/knowledge/KnowledgeLifecycleMiddleware.ts
import { z } from "zod";

const KnowledgeStatus = z.enum(["draft", "active", "stale", "superseded", "archived"]);
const UsePurpose = z.enum(["answer", "planning", "side_effect", "audit"]);

const KnowledgeEntry = z.object({
  id: z.string(),
  title: z.string(),
  scope: z.string(),
  status: KnowledgeStatus,
  confidence: z.number().min(0).max(1),
  verifiedAt: z.string().datetime().nullable(),
  reviewAfter: z.string().datetime().nullable(),
  expiresAt: z.string().datetime().nullable(),
  supersededBy: z.string().nullable().default(null),
  evidence: z.array(z.string()).default([]),
});

type KnowledgeEntry = z.infer<typeof KnowledgeEntry>;
type UsePurpose = z.infer<typeof UsePurpose>;

export class KnowledgeLifecyclePolicy {
  refresh(entry: KnowledgeEntry, now = new Date()): KnowledgeEntry {
    const next = { ...entry };

    if (next.status === "archived" || next.status === "superseded") {
      return next;
    }

    if (next.supersededBy) {
      return { ...next, status: "superseded" };
    }

    if (next.expiresAt && new Date(next.expiresAt) <= now) {
      return { ...next, status: "archived" };
    }

    if (next.status === "active" && next.reviewAfter && new Date(next.reviewAfter) <= now) {
      return {
        ...next,
        status: "stale",
        confidence: Math.min(next.confidence, 0.5),
      };
    }

    return next;
  }

  canUse(entry: KnowledgeEntry, purpose: UsePurpose): { ok: boolean; reason?: string } {
    if (purpose === "audit") return { ok: true };

    if (entry.status === "active" && entry.confidence >= 0.7) {
      return { ok: true };
    }

    if (entry.status === "stale" && purpose !== "side_effect" && entry.confidence >= 0.4) {
      return { ok: true, reason: "stale_reference_only" };
    }

    return { ok: false, reason: `knowledge_${entry.status}_not_allowed_for_${purpose}` };
  }
}

export class KnowledgeStore {
  constructor(
    private readonly rawStore: { listByScope(scope: string): Promise<unknown[]>; save(entry: KnowledgeEntry): Promise<void> },
    private readonly policy = new KnowledgeLifecyclePolicy(),
  ) {}

  async getContext(scope: string, purpose: UsePurpose): Promise<KnowledgeEntry[]> {
    const raw = await this.rawStore.listByScope(scope);
    const entries = raw.map((item) => KnowledgeEntry.parse(item));

    const usable: KnowledgeEntry[] = [];

    for (const entry of entries) {
      const refreshed = this.policy.refresh(entry);

      if (refreshed.status !== entry.status || refreshed.confidence !== entry.confidence) {
        await this.rawStore.save(refreshed); // 状态推进要落库，避免每次重复判断
      }

      const decision = this.policy.canUse(refreshed, purpose);
      if (decision.ok) usable.push(refreshed);
    }

    return usable.sort((a, b) => b.confidence - a.confidence);
  }
}
```

挂到 Agent Loop：

```ts
const knowledge = await knowledgeStore.getContext("tool:message.send", "side_effect");

if (knowledge.length === 0) {
  // 没有 fresh knowledge 时，不要靠旧经验执行副作用
  return {
    decision: "require_revalidation",
    reason: "no_active_operational_knowledge_for_side_effect",
  };
}
```

这让知识生命周期变成基础设施，而不是 prompt 里的软提醒。

---

## 5. OpenClaw 实战：文件式知识库 + Cron 清扫

OpenClaw 很适合用文件先做轻量落地：

```text
.openclaw/workspace/ops-kb/
  active/
    kb_retry_receipt.json
  stale/
    kb_old_threshold.json
  archived/
    kb_legacy_workaround.json
  audit.jsonl
```

一个 Cron/Heartbeat 清扫流程：

```python
# scripts/sweep_ops_kb.py
import json
import shutil
from datetime import datetime, timezone
from pathlib import Path

ROOT = Path("/Users/bot001/.openclaw/workspace/ops-kb")
AUDIT = ROOT / "audit.jsonl"


def now_iso() -> str:
    return datetime.now(timezone.utc).isoformat()


def parse_ts(value: str | None):
    if not value:
        return None
    return datetime.fromisoformat(value.replace("Z", "+00:00"))


def audit(event: dict):
    AUDIT.parent.mkdir(parents=True, exist_ok=True)
    with AUDIT.open("a") as f:
        f.write(json.dumps({"at": now_iso(), **event}, ensure_ascii=False) + "\n")


def move_entry(path: Path, target_status: str):
    entry = json.loads(path.read_text())
    old_status = entry.get("status")
    entry["status"] = target_status
    entry["updatedAt"] = now_iso()

    target_dir = ROOT / target_status
    target_dir.mkdir(parents=True, exist_ok=True)
    target = target_dir / path.name
    target.write_text(json.dumps(entry, ensure_ascii=False, indent=2) + "\n")
    path.unlink()

    audit({
        "type": "knowledge_lifecycle_transition",
        "id": entry["id"],
        "from": old_status,
        "to": target_status,
        "file": str(target),
    })


def sweep():
    now = datetime.now(timezone.utc)

    for path in (ROOT / "active").glob("*.json"):
        entry = json.loads(path.read_text())
        review_after = parse_ts(entry.get("reviewAfter"))
        expires_at = parse_ts(entry.get("expiresAt"))
        superseded_by = entry.get("supersededBy")

        if superseded_by or (expires_at and now >= expires_at):
            move_entry(path, "archived")
        elif review_after and now >= review_after:
            move_entry(path, "stale")


if __name__ == "__main__":
    sweep()
```

然后在 Agent 执行高风险 Runbook 前加一条硬规则：

```text
side_effect / prod_change / external_message：
只允许读取 ops-kb/active，不能读取 stale/archived 作为执行依据。
```

这和前面几课组合起来：

- Incident Review 产出候选知识
- Action Tracker 要求更新知识库
- Remediation Verification 验证知识有效
- Runbook Drift 检查依赖是否仍存在
- Knowledge Lifecycle 定期让旧知识降级/归档

---

## 6. 三个常见坑

### 坑 1：只设置 TTL，不设置 reviewAfter

TTL 负责“什么时候必须删除/归档”。

`reviewAfter` 负责“什么时候需要重新验证”。

很多知识不是立刻失效，而是需要降级成 stale：能参考，不能直接执行。

### 坑 2：归档时直接删除

不要直接删。

归档知识对事故复盘、审计、解释“为什么过去这么做”很重要。

正确做法是：从 active context 移除，但保留 audit path。

### 坑 3：让 LLM 自己判断知识是否过期

不要把十几条知识塞给 LLM 然后说“请注意过期时间”。

生命周期判断应该在工具/上下文注入层完成：

```text
先过滤 -> 再注入 -> LLM 只看到当前 purpose 允许看的知识
```

---

## 7. 实战检查清单

给你的 Agent 运维知识库加这几个字段：

```text
status        draft/active/stale/superseded/archived
verifiedAt    最近验证时间
reviewAfter   需要复核时间
expiresAt     强制归档时间
confidence    当前置信度
source         incident/review/pr/deploy/monitor evidence
supersededBy   被哪条新知识替代
```

给上下文注入加这条策略：

```text
answer/planning 可以引用 stale，但必须降权和标注 caveat；
side_effect/prod_change 只能使用 active 且 fresh 的知识；
archived 只允许 audit/review 检索。
```

---

## 总结

今天这课的核心：

> 运维知识不是越多越好，而是越新、越准、越可验证越好。

成熟 Agent 的知识库不是“永远记住”，而是：

1. 新知识先 draft
2. 验证后 active
3. 到期后 stale
4. 被替代后 superseded
5. 最后 archived 保留审计

这样 Agent 才不会拿去年的事故经验，去自动化今天的生产操作。
