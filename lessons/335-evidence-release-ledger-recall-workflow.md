# 335. Agent 证据发布账本与召回流程（Evidence Release Ledger & Recall Workflow）

上一课讲了 **Declassification Pipeline**：raw evidence 不能直接外发，要先降密成 channel-safe artifact。

今天补上外发后的闭环：证据一旦发到了 Telegram、GitHub、Email、Status Page，就不再只是内部对象，而是有了外部副本。成熟 Agent 需要记录它发到了哪里、谁能看到、后续如何撤回或更正。

核心思想：

> Agent 外发证据时，不只要拿到 messageId / commentId 回执，还要把“artifact -> external publication”的关系写进发布账本；当源证据撤销、过期、发现误脱敏或权限变化时，能自动定位外部副本并执行 recall / edit / notice。

---

## 1. 为什么 Outbox Receipt 还不够？

Outbox 解决的是“消息有没有可靠送达”。

但 Evidence Release Ledger 解决的是另一个问题：

- 这条 Telegram 消息用了哪些 evidence？
- 这条 GitHub 评论里的 summary 来自哪个 DeclassifiedArtifact？
- 后来发现 artifact 里漏脱敏了，应该删哪几条外部消息？
- 源 evidence 被撤销，已经发出去的公开解释要不要更正？
- 审计时如何证明“我们发现风险后 3 分钟内完成召回”？

没有发布账本，Agent 只能靠搜索聊天记录、猜测 PR 评论内容，很容易漏掉外部副本。

所以外发链路应该是：

```text
Raw Evidence
  -> DeclassifiedArtifact
  -> Egress Policy
  -> Send / Comment / Email
  -> External Receipt
  -> Release Ledger entry
  -> Recall monitor
```

---

## 2. 数据结构：ReleaseRecord

发布账本不是普通日志，而是可查询、可召回的索引。

```json
{
  "releaseId": "rel_01H...",
  "artifactId": "decl_ci_123",
  "sourceEvidenceIds": ["ev_ci_log_123"],
  "channel": "telegram",
  "audience": "rust_study_group",
  "externalRef": {
    "provider": "telegram",
    "chatId": "-5115329245",
    "messageId": "12142"
  },
  "classificationAfter": "internal",
  "watermarkId": "wm_01H...",
  "recallPolicy": {
    "onEvidenceRevoked": "delete_or_edit",
    "onSecretDetected": "delete_immediately",
    "onArtifactExpired": "notice_only"
  },
  "state": "published",
  "publishedAt": "2026-05-16T14:30:00.000Z",
  "lastVerifiedAt": "2026-05-16T14:31:00.000Z"
}
```

几个关键字段：

- `artifactId`：绑定降密产物，而不是只存消息文本。
- `sourceEvidenceIds`：源证据撤销时可以反查所有外部副本。
- `externalRef`：记录平台回执，支持 delete/edit/notice。
- `watermarkId`：如果外部内容泄漏，可以反查发布事件。
- `recallPolicy`：不同通道、不同风险采取不同召回动作。
- `state`：`published | recalled | corrected | notice_sent | failed_recall`。

---

## 3. learn-claude-code：最小 Python 发布账本

教学版可以用 JSONL 做 append-only ledger，再写一个反查召回计划。

```python
# learn-claude-code/evidence_release_ledger.py
from dataclasses import dataclass, asdict
from datetime import datetime, timezone
from pathlib import Path
import hashlib
import json
from typing import Literal

ReleaseState = Literal["published", "recalled", "corrected", "notice_sent", "failed_recall"]
RecallAction = Literal["delete", "edit", "notice", "manual_review", "none"]

@dataclass
class ExternalRef:
    provider: str
    target: str
    message_id: str

@dataclass
class RecallPolicy:
    on_evidence_revoked: RecallAction
    on_secret_detected: RecallAction
    on_artifact_expired: RecallAction

@dataclass
class ReleaseRecord:
    release_id: str
    artifact_id: str
    source_evidence_ids: list[str]
    channel: str
    audience: str
    external_ref: ExternalRef
    classification_after: str
    watermark_id: str | None
    recall_policy: RecallPolicy
    state: ReleaseState
    published_at: str
    last_verified_at: str | None = None

class ReleaseLedger:
    def __init__(self, path: str = "release-ledger.jsonl"):
        self.path = Path(path)

    def append(self, record: ReleaseRecord) -> None:
        with self.path.open("a", encoding="utf-8") as f:
            f.write(json.dumps(asdict(record), ensure_ascii=False, sort_keys=True) + "\n")

    def all(self) -> list[dict]:
        if not self.path.exists():
            return []
        return [json.loads(line) for line in self.path.read_text(encoding="utf-8").splitlines() if line.strip()]

    def find_by_evidence(self, evidence_id: str) -> list[dict]:
        return [r for r in self.all() if evidence_id in r["source_evidence_ids"] and r["state"] == "published"]

def new_release_id(artifact_id: str, external_ref: ExternalRef) -> str:
    raw = f"{artifact_id}:{external_ref.provider}:{external_ref.target}:{external_ref.message_id}"
    return "rel_" + hashlib.sha256(raw.encode()).hexdigest()[:16]

def plan_recall(records: list[dict], reason: Literal["evidence_revoked", "secret_detected", "artifact_expired"]) -> list[dict]:
    field = {
        "evidence_revoked": "on_evidence_revoked",
        "secret_detected": "on_secret_detected",
        "artifact_expired": "on_artifact_expired",
    }[reason]

    plan = []
    for record in records:
        action = record["recall_policy"][field]
        if action == "none":
            continue
        plan.append({
            "releaseId": record["release_id"],
            "action": action,
            "provider": record["external_ref"]["provider"],
            "target": record["external_ref"]["target"],
            "messageId": record["external_ref"]["message_id"],
            "reason": reason,
        })
    return plan

if __name__ == "__main__":
    ledger = ReleaseLedger("/tmp/release-ledger.jsonl")
    ref = ExternalRef(provider="telegram", target="-5115329245", message_id="12142")
    record = ReleaseRecord(
        release_id=new_release_id("decl_ci_123", ref),
        artifact_id="decl_ci_123",
        source_evidence_ids=["ev_ci_log_123"],
        channel="telegram",
        audience="rust_study_group",
        external_ref=ref,
        classification_after="internal",
        watermark_id="wm_abc",
        recall_policy=RecallPolicy(
            on_evidence_revoked="edit",
            on_secret_detected="delete",
            on_artifact_expired="notice",
        ),
        state="published",
        published_at=datetime.now(timezone.utc).isoformat(),
    )
    ledger.append(record)
    print(plan_recall(ledger.find_by_evidence("ev_ci_log_123"), "secret_detected"))
```

注意：这里的 `plan_recall()` 只生成计划，不直接删除消息。真正执行前还要过 Policy、审批和平台能力检查。

---

## 4. pi-mono：ReleaseLedgerMiddleware

生产里把发布账本接到所有外发工具后面：`message.send`、`github.comment`、`email.send`、`statuspage.update`。

```ts
// pi-mono/packages/agent-security/src/evidence-release-ledger.ts
export type ReleaseState = 'published' | 'recalled' | 'corrected' | 'notice_sent' | 'failed_recall'
export type RecallAction = 'delete' | 'edit' | 'notice' | 'manual_review' | 'none'

export interface ExternalRef {
  provider: 'telegram' | 'github' | 'email' | 'statuspage'
  target: string
  messageId: string
  url?: string
}

export interface RecallPolicy {
  onEvidenceRevoked: RecallAction
  onSecretDetected: RecallAction
  onArtifactExpired: RecallAction
}

export interface ReleaseRecord {
  releaseId: string
  artifactId: string
  sourceEvidenceIds: string[]
  channel: string
  audience: string
  externalRef: ExternalRef
  classificationAfter: 'public' | 'internal' | 'confidential'
  watermarkId?: string
  recallPolicy: RecallPolicy
  state: ReleaseState
  publishedAt: string
}

export interface ReleaseLedgerStore {
  append(record: ReleaseRecord): Promise<void>
  findByEvidence(evidenceId: string): Promise<ReleaseRecord[]>
  updateState(releaseId: string, state: ReleaseState, evidence?: unknown): Promise<void>
}

export async function recordPublication(args: {
  store: ReleaseLedgerStore
  artifact: {
    artifactId: string
    sourceEvidenceIds: string[]
    classificationAfter: ReleaseRecord['classificationAfter']
    watermarkId?: string
  }
  channel: string
  audience: string
  externalRef: ExternalRef
  recallPolicy: RecallPolicy
}) {
  const releaseId = await stableReleaseId(args.artifact.artifactId, args.externalRef)

  await args.store.append({
    releaseId,
    artifactId: args.artifact.artifactId,
    sourceEvidenceIds: args.artifact.sourceEvidenceIds,
    channel: args.channel,
    audience: args.audience,
    externalRef: args.externalRef,
    classificationAfter: args.artifact.classificationAfter,
    watermarkId: args.artifact.watermarkId,
    recallPolicy: args.recallPolicy,
    state: 'published',
    publishedAt: new Date().toISOString(),
  })

  return releaseId
}

async function stableReleaseId(artifactId: string, ref: ExternalRef): Promise<string> {
  const input = `${artifactId}:${ref.provider}:${ref.target}:${ref.messageId}`
  const digest = await crypto.subtle.digest('SHA-256', new TextEncoder().encode(input))
  return 'rel_' + [...new Uint8Array(digest)].map(b => b.toString(16).padStart(2, '0')).join('').slice(0, 16)
}
```

这层 middleware 的位置很重要：

```text
DeclassificationMiddleware
  -> EgressPolicyMiddleware
  -> OutboxDispatcher / message.send
  -> ReleaseLedgerMiddleware
  -> Post-send verification
```

只有外部平台返回成功回执后，才能把记录标记为 `published`。

---

## 5. OpenClaw：Telegram 外发后的召回闭环

以课程 cron 为例，发送 Telegram 群消息后至少要记录：

```json
{
  "artifactId": "decl_lesson_335",
  "sourceEvidenceIds": ["ev_lesson_md_335", "ev_git_commit_335"],
  "externalRef": {
    "provider": "telegram",
    "target": "-5115329245",
    "messageId": "12142"
  },
  "recallPolicy": {
    "onEvidenceRevoked": "edit",
    "onSecretDetected": "delete",
    "onArtifactExpired": "notice"
  }
}
```

后续如果发现 lesson 里误带了 token：

1. 根据 `sourceEvidenceId` 找到所有 `ReleaseRecord`。
2. 按 `recallPolicy.onSecretDetected` 生成召回计划。
3. 对 Telegram 执行 delete 或 edit。
4. 写入 `recalled` / `corrected` 状态和平台回执。
5. 如果平台不支持删除，发送 correction notice 并升级人工复核。

这比“赶紧去群里翻消息”可靠得多。

---

## 6. 常见坑

### 坑 1：只存 messageId，不存 source evidence

只存回执只能证明“发过”，不能证明“发的是哪份证据的哪个降密版本”。召回时必须能从 evidence 反查外部副本。

### 坑 2：把 recall 当成 best-effort

安全召回不是随缘任务。`secret_detected -> delete` 失败时应该进入 `failed_recall`，触发 incident 或人工复核。

### 坑 3：平台不支持删除就当没事

很多通道无法真正删除，比如 Email。此时 recall action 应该退化成 `notice`：发更正说明、撤销旧 artifact 的可信状态、审计记录完整保留。

### 坑 4：更正消息没有绑定旧消息

Correction notice 必须引用 `releaseId` / `messageId`，否则审计时无法证明它修正的是哪一次发布。

---

## 7. 实战 Checklist

给 Agent 加证据发布账本时，至少做到：

- [ ] 外发成功后记录 `artifactId -> externalRef`
- [ ] `sourceEvidenceIds` 可反查所有外部副本
- [ ] 每条发布记录有 recall policy
- [ ] 支持 delete / edit / notice / manual_review 四类召回动作
- [ ] 召回失败进入 incident 或人工复核
- [ ] correction notice 绑定原始 releaseId
- [ ] 审计可证明 published -> recalled/corrected 的完整时间线

---

## 8. 一句话总结

Outbox 证明消息送到了；Release Ledger 证明证据发到了哪里，以及出事后能不能找回来、改回来、说清楚。

成熟 Agent 不只是谨慎外发，还要对每一次外发后的证据副本负责。
