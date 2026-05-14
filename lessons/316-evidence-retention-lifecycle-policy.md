# 316. Agent 证据生命周期与保留策略（Evidence Retention & Lifecycle Policy）

上一课讲了 **Proof Minimization**：证据包要最小披露，LLM 看到 ProofView，原始材料只做审计归档。

今天继续往后补一个生产系统一定会遇到的问题：**证据不能永远堆着，也不能随便删。**

这叫 **Evidence Retention Lifecycle**：每条证据从创建开始就带上数据等级、保留期限、删除时间、legal hold、可擦除性和销毁证明，让 Agent 既能证明自己做过正确的事，也不会把日志、用户隐私、CI 输出、审批材料永久留在系统里。

---

## 1. 为什么证据也需要生命周期？

Agent 做副作用时会留下很多 evidence：

- Git diff、测试日志、部署回执
- Telegram / Email / Webhook 投递回执
- 审批记录、操作者、审批理由
- 工具调用输入输出摘要
- 策略决策和阻断原因
- 事故处理时间线

如果没有保留策略，最后会出现两个极端：

1. **什么都不留**：出事后无法证明当时为什么允许执行
2. **什么都永久留**：敏感数据、用户隐私、secret 片段在归档里越积越多

成熟做法是：证据创建时就决定它属于哪一类、留多久、什么时候删除、是否允许被用户删除、是否被 legal hold 锁住。

```json
{
  "id": "ev_20260514_git_push_01",
  "kind": "git_push_receipt",
  "dataClass": "operational",
  "sensitivity": "internal",
  "createdAt": "2026-05-14T05:30:00Z",
  "retainUntil": "2026-08-12T05:30:00Z",
  "deleteAfter": "2026-11-10T05:30:00Z",
  "erasable": true,
  "legalHold": false,
  "artifactRef": "artifact://evidence/ev_20260514_git_push_01.json",
  "contentHash": "sha256:..."
}
```

---

## 2. 四个时间字段：不要只靠 TTL

很多系统只给 evidence 一个 TTL，这是不够的。

推荐拆成四个字段：

| 字段 | 含义 |
|---|---|
| `createdAt` | 证据生成时间 |
| `retainUntil` | 在此之前必须保留，不能删除 |
| `deleteAfter` | 到期后应删除或脱敏 |
| `reviewAfter` | 到期前复核是否仍有价值 |

核心区别：

- `retainUntil` 是**最低保留期**
- `deleteAfter` 是**最长保留期**
- `reviewAfter` 是**提前复核点**
- TTL 只是技术缓存，不是合规策略

举例：

```ts
const RETENTION_POLICY = {
  message_delivery_receipt: { retainDays: 90, deleteDays: 180 },
  approval_record:          { retainDays: 365, deleteDays: 730 },
  raw_tool_output:          { retainDays: 7,  deleteDays: 30  },
  security_decision:        { retainDays: 365, deleteDays: 1095 },
  user_private_content:     { retainDays: 0,  deleteDays: 7, erasable: true },
} as const;
```

---

## 3. learn-claude-code：Python 教学版

先做一个最小文件式 Evidence Store。

```python
# evidence_retention.py
from __future__ import annotations

from dataclasses import dataclass, asdict
from datetime import datetime, timedelta, timezone
from pathlib import Path
import hashlib
import json
import os
from typing import Literal

DataClass = Literal["operational", "security", "user_private", "audit"]
Sensitivity = Literal["public", "internal", "sensitive", "secret"]

RETENTION = {
    "message_delivery_receipt": {"retain_days": 90, "delete_days": 180, "erasable": True},
    "approval_record":          {"retain_days": 365, "delete_days": 730, "erasable": False},
    "raw_tool_output":          {"retain_days": 7, "delete_days": 30, "erasable": True},
    "security_decision":        {"retain_days": 365, "delete_days": 1095, "erasable": False},
}

@dataclass
class EvidenceEnvelope:
    id: str
    kind: str
    data_class: DataClass
    sensitivity: Sensitivity
    created_at: str
    retain_until: str
    delete_after: str
    erasable: bool
    legal_hold: bool
    artifact_ref: str
    content_hash: str

class EvidenceStore:
    def __init__(self, root: str = "./evidence"):
        self.root = Path(root)
        self.root.mkdir(parents=True, exist_ok=True)

    def write(self, *, kind: str, data_class: DataClass, sensitivity: Sensitivity, raw: dict) -> EvidenceEnvelope:
        policy = RETENTION[kind]
        now = datetime.now(timezone.utc)
        raw_bytes = json.dumps(raw, sort_keys=True, ensure_ascii=False).encode("utf-8")
        digest = hashlib.sha256(raw_bytes).hexdigest()
        ev_id = f"ev_{now.strftime('%Y%m%d%H%M%S')}_{digest[:10]}"

        raw_path = self.root / f"{ev_id}.raw.json"
        meta_path = self.root / f"{ev_id}.meta.json"

        raw_path.write_bytes(raw_bytes)

        env = EvidenceEnvelope(
            id=ev_id,
            kind=kind,
            data_class=data_class,
            sensitivity=sensitivity,
            created_at=now.isoformat(),
            retain_until=(now + timedelta(days=policy["retain_days"])).isoformat(),
            delete_after=(now + timedelta(days=policy["delete_days"])).isoformat(),
            erasable=policy["erasable"],
            legal_hold=False,
            artifact_ref=str(raw_path),
            content_hash=f"sha256:{digest}",
        )
        meta_path.write_text(json.dumps(asdict(env), indent=2, ensure_ascii=False))
        return env
```

关键点：

- raw evidence 和 metadata 分开存
- metadata 永远带 `content_hash`
- `legal_hold=True` 时 sweeper 不能删
- `erasable=False` 的安全审计记录不能被普通用户删除请求直接清掉

---

## 4. 自动清扫器：删除前先写销毁证明

删除 evidence 也要可审计。否则别人会问：你怎么证明不是偷偷删了关键证据？

```python
# evidence_sweeper.py
from datetime import datetime, timezone
from pathlib import Path
import json

class EvidenceSweeper:
    def __init__(self, root: str = "./evidence"):
        self.root = Path(root)
        self.tombstones = self.root / "tombstones.jsonl"
        self.audit = self.root / "sweeper-audit.jsonl"

    def sweep(self) -> list[str]:
        now = datetime.now(timezone.utc)
        deleted = []

        for meta_path in self.root.glob("*.meta.json"):
            meta = json.loads(meta_path.read_text())

            if meta.get("legal_hold"):
                continue

            delete_after = datetime.fromisoformat(meta["delete_after"])
            if now < delete_after:
                continue

            raw_path = Path(meta["artifact_ref"])
            tombstone = {
                "evidence_id": meta["id"],
                "deleted_at": now.isoformat(),
                "kind": meta["kind"],
                "content_hash": meta["content_hash"],
                "reason": "retention_expired",
            }

            self.tombstones.open("a").write(json.dumps(tombstone, ensure_ascii=False) + "\n")

            if raw_path.exists():
                raw_path.unlink()
            meta_path.unlink()

            self.audit.open("a").write(json.dumps({
                "event": "evidence_deleted",
                **tombstone,
            }, ensure_ascii=False) + "\n")

            deleted.append(meta["id"])

        return deleted
```

这里的 tombstone 很重要：

- 不保留原文
- 保留 evidence id、类型、hash、删除原因
- 后续审计可以证明“这条证据曾经存在，并按策略删除”

---

## 5. pi-mono：生产版中间件设计

在生产系统里，不要让每个工具自己决定留多久。应该有统一的 EvidenceRetentionMiddleware。

```ts
type EvidenceKind =
  | "message_delivery_receipt"
  | "approval_record"
  | "raw_tool_output"
  | "security_decision";

type RetentionRule = {
  retainDays: number;
  deleteDays: number;
  erasable: boolean;
  allowedSensitivity: Array<"public" | "internal" | "sensitive" | "secret">;
};

const retentionRules: Record<EvidenceKind, RetentionRule> = {
  message_delivery_receipt: {
    retainDays: 90,
    deleteDays: 180,
    erasable: true,
    allowedSensitivity: ["public", "internal", "sensitive"],
  },
  approval_record: {
    retainDays: 365,
    deleteDays: 730,
    erasable: false,
    allowedSensitivity: ["internal", "sensitive"],
  },
  raw_tool_output: {
    retainDays: 7,
    deleteDays: 30,
    erasable: true,
    allowedSensitivity: ["public", "internal", "sensitive"],
  },
  security_decision: {
    retainDays: 365,
    deleteDays: 1095,
    erasable: false,
    allowedSensitivity: ["internal", "sensitive"],
  },
};
```

中间件负责三件事：

1. 给 evidence 自动补生命周期字段
2. 拒绝不合法的 sensitivity/kind 组合
3. 写入前调用 redactor，确保 secret 不进 raw evidence

```ts
class EvidenceRetentionMiddleware {
  constructor(
    private readonly store: EvidenceStore,
    private readonly redactor: EvidenceRedactor,
  ) {}

  async record(input: {
    kind: EvidenceKind;
    sensitivity: "public" | "internal" | "sensitive" | "secret";
    actor: string;
    runId: string;
    payload: unknown;
  }) {
    const rule = retentionRules[input.kind];
    if (!rule.allowedSensitivity.includes(input.sensitivity)) {
      throw new Error(`invalid evidence sensitivity for ${input.kind}: ${input.sensitivity}`);
    }

    const now = new Date();
    const sanitized = await this.redactor.redact(input.payload, {
      mode: input.sensitivity === "secret" ? "hash_only" : "mask_secrets",
    });

    return this.store.put({
      kind: input.kind,
      actor: input.actor,
      runId: input.runId,
      payload: sanitized,
      createdAt: now.toISOString(),
      retainUntil: addDays(now, rule.retainDays).toISOString(),
      deleteAfter: addDays(now, rule.deleteDays).toISOString(),
      erasable: rule.erasable,
      legalHold: false,
    });
  }
}
```

---

## 6. 用户删除请求：RTBF 不能直接删审计事实

用户说“删除我的数据”时，不是所有 evidence 都能直接物理删除。

推荐分三类处理：

```ts
type ErasureDecision =
  | { action: "delete"; reason: string }
  | { action: "redact_subject"; reason: string }
  | { action: "deny_legal_hold"; reason: string }
  | { action: "keep_audit_minimized"; reason: string };

function decideErasure(ev: EvidenceEnvelope, subjectId: string): ErasureDecision {
  if (ev.legalHold) {
    return { action: "deny_legal_hold", reason: "evidence is under legal hold" };
  }

  if (ev.erasable && ev.subjectIds?.includes(subjectId)) {
    return { action: "delete", reason: "erasable subject data" };
  }

  if (ev.kind === "approval_record" || ev.kind === "security_decision") {
    return { action: "redact_subject", reason: "retain audit fact, remove subject payload" };
  }

  return { action: "keep_audit_minimized", reason: "no subject payload retained" };
}
```

这点非常重要：

- 用户隐私原文可以删
- 安全审计事实通常要保留最小摘要
- legal hold 期间不能自动删
- 删除动作本身也要产生 evidence

---

## 7. OpenClaw 实战：课程 Cron 的证据生命周期

以这个课程 Cron 为例，执行一次会产生这些 evidence：

```json
{
  "runId": "cron:agent-course:2026-05-14T05:30Z",
  "evidence": [
    {
      "kind": "git_status_before_commit",
      "retainDays": 30,
      "deleteDays": 90,
      "summary": "clean except lesson/readme/tools changes"
    },
    {
      "kind": "telegram_delivery_receipt",
      "retainDays": 90,
      "deleteDays": 180,
      "summary": "message sent to Rust学习小组"
    },
    {
      "kind": "git_push_receipt",
      "retainDays": 180,
      "deleteDays": 365,
      "summary": "pushed commit abc123 to gfwfail/agent-course"
    }
  ]
}
```

注意：群消息全文不一定要永久保存。长期只需要：

- lesson 文件 hash
- commit sha
- Telegram message id
- 发送成功状态
- 课程标题摘要

这就够证明任务完成了，不需要把所有上下文和内部 token 都归档。

---

## 8. 常见坑

### 坑 1：把日志系统当证据系统

日志通常是 best-effort，证据需要 hash、retention、删除证明和审计查询。

### 坑 2：只删 raw，不删派生摘要

RTBF 时要检查 raw、summary、embedding、cache、search index、export bundle。

### 坑 3：legal hold 没有优先级

一旦 evidence 被标记 legal hold，普通 retention sweeper 必须跳过。

### 坑 4：删除没有 tombstone

没有 tombstone，删除后无法证明是按策略删除，不是事故丢失。

### 坑 5：所有证据同一个 TTL

测试日志、审批记录、用户内容、投递回执风险不同，保留策略必须分层。

---

## 9. 一句话总结

**Evidence Retention Lifecycle = 证据从出生就带保留期限、删除期限、可擦除性、legal hold 和销毁证明。**

成熟 Agent 不只是“能证明自己做过什么”，还要能证明：

- 该留的证据还在
- 不该留的敏感内容已经删了
- 删除本身可审计
- legal hold 和用户删除请求不会互相踩踏

下一次你给 Agent 加审计日志时，不要只问“记录什么”，还要问：**留多久？谁能删？删了怎么证明？**
