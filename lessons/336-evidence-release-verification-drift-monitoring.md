# 336. Agent 证据发布复核与漂移监控（Evidence Release Verification & Drift Monitoring）

上一课讲了 **Evidence Release Ledger**：Agent 把降密证据发到 Telegram、GitHub、Email 后，要记录 artifact 和外部 message/comment/email 的关系，后续才能召回。

今天继续补外发后的第二个问题：外部副本不是发出去就永远正确。消息可能被管理员删除、评论可能被人编辑、链接可能失效、平台权限可能变化，甚至召回 API 也可能突然不可用。

核心思想：

> Agent 发布证据后，必须定期复核外部副本是否仍然存在、内容是否和账本一致、可见范围是否符合策略、召回能力是否还有效；一旦发现漂移，立即写入 DriftEvent，并按严重程度触发 edit/delete/notice/manual_review。

---

## 1. 为什么发布后还要复核？

Release Ledger 解决“发到了哪里”。Release Verification 解决“现在还对不对”。

几个常见事故：

- Telegram 消息被管理员删了，内部账本还以为它仍在公开说明。
- GitHub 评论被协作者手动编辑，删掉了脱敏说明或加回了敏感片段。
- Email 退信后没有入账，用户实际没收到更正通知。
- 外部链接过期，审计页面点开只剩 404。
- 平台 token 权限变了，Agent 发现问题时已经不能 recall。

所以发布链路不能停在 `messageId`：

```text
DeclassifiedArtifact
  -> External Publication
  -> Release Ledger
  -> Verification Schedule
  -> External Fetch / Head Check
  -> Hash + Visibility + Capability Compare
  -> DriftEvent / Healthy Checkpoint
```

成熟 Agent 要能回答三个问题：

- 外部副本还存在吗？
- 外部副本还是我当时批准发布的那份内容吗？
- 如果现在需要召回，我还有权限和能力执行吗？

---

## 2. 数据结构：VerificationCheckpoint + DriftEvent

发布记录里可以只保留最新状态，但复核结果必须 append-only，方便审计。

```json
{
  "checkpointId": "chk_01H...",
  "releaseId": "rel_01H...",
  "externalRef": {
    "provider": "telegram",
    "chatId": "-5115329245",
    "messageId": "12142"
  },
  "expectedContentHash": "sha256:abc...",
  "observedContentHash": "sha256:abc...",
  "exists": true,
  "visibility": "group_members",
  "recallCapability": {
    "canDelete": true,
    "canEdit": true,
    "checkedAt": "2026-05-17T00:30:00.000Z"
  },
  "status": "healthy",
  "checkedAt": "2026-05-17T00:30:00.000Z"
}
```

如果发现不一致，写 `DriftEvent`：

```json
{
  "driftId": "drift_01H...",
  "releaseId": "rel_01H...",
  "kind": "content_changed",
  "severity": "high",
  "expectedContentHash": "sha256:abc...",
  "observedContentHash": "sha256:def...",
  "recommendedAction": "manual_review",
  "detectedAt": "2026-05-17T00:31:00.000Z"
}
```

常见 drift 类型：

- `missing`：外部副本不存在。
- `content_changed`：内容 hash 不一致。
- `visibility_changed`：可见范围变宽或变窄。
- `recall_capability_lost`：不能 delete/edit。
- `link_broken`：外部 URL 不可访问。
- `receipt_unconfirmed`：平台没有可靠回执或回执过期。

---

## 3. learn-claude-code：最小 Python 漂移检测器

教学版先把外部平台抽象成 `ExternalPublisher`，复核时只比较账本期望和平台观测值。

```python
# learn-claude-code/evidence_release_verifier.py
from dataclasses import dataclass, asdict
from datetime import datetime, timezone
from pathlib import Path
from typing import Literal, Protocol
import hashlib
import json

DriftKind = Literal[
    "missing",
    "content_changed",
    "visibility_changed",
    "recall_capability_lost",
    "link_broken",
]
Severity = Literal["low", "medium", "high", "critical"]
RecommendedAction = Literal["ignore", "notice", "edit", "delete", "manual_review"]

@dataclass
class ExternalRef:
    provider: str
    target: str
    message_id: str

@dataclass
class ExpectedRelease:
    release_id: str
    external_ref: ExternalRef
    expected_body: str
    expected_visibility: str
    require_delete: bool = True
    require_edit: bool = False

@dataclass
class ObservedPublication:
    exists: bool
    body: str | None
    visibility: str | None
    can_delete: bool
    can_edit: bool
    url_ok: bool = True

@dataclass
class DriftEvent:
    drift_id: str
    release_id: str
    kind: DriftKind
    severity: Severity
    recommended_action: RecommendedAction
    detected_at: str

class ExternalPublisher(Protocol):
    def fetch(self, ref: ExternalRef) -> ObservedPublication:
        ...

def content_hash(text: str | None) -> str | None:
    if text is None:
        return None
    return "sha256:" + hashlib.sha256(text.encode("utf-8")).hexdigest()

def new_drift_id(release_id: str, kind: str, observed_hash: str | None) -> str:
    raw = f"{release_id}:{kind}:{observed_hash or 'none'}"
    return "drift_" + hashlib.sha256(raw.encode()).hexdigest()[:16]

def detect_drift(expected: ExpectedRelease, observed: ObservedPublication) -> list[DriftEvent]:
    now = datetime.now(timezone.utc).isoformat()
    events: list[DriftEvent] = []

    def add(kind: DriftKind, severity: Severity, action: RecommendedAction):
        events.append(DriftEvent(
            drift_id=new_drift_id(expected.release_id, kind, content_hash(observed.body)),
            release_id=expected.release_id,
            kind=kind,
            severity=severity,
            recommended_action=action,
            detected_at=now,
        ))

    if not observed.exists:
        add("missing", "medium", "notice")
        return events

    if not observed.url_ok:
        add("link_broken", "medium", "notice")

    if content_hash(expected.expected_body) != content_hash(observed.body):
        add("content_changed", "high", "manual_review")

    if expected.expected_visibility != observed.visibility:
        severity: Severity = "critical" if observed.visibility == "public" else "medium"
        add("visibility_changed", severity, "manual_review")

    if expected.require_delete and not observed.can_delete:
        add("recall_capability_lost", "high", "manual_review")
    if expected.require_edit and not observed.can_edit:
        add("recall_capability_lost", "medium", "manual_review")

    return events

def append_jsonl(path: str, rows: list[DriftEvent]) -> None:
    p = Path(path)
    with p.open("a", encoding="utf-8") as f:
        for row in rows:
            f.write(json.dumps(asdict(row), ensure_ascii=False, sort_keys=True) + "\n")

if __name__ == "__main__":
    release = ExpectedRelease(
        release_id="rel_123",
        external_ref=ExternalRef("telegram", "-5115329245", "12142"),
        expected_body="CI failed because TypeScript has a missing export. Secrets were removed.",
        expected_visibility="group_members",
        require_delete=True,
    )
    observed = ObservedPublication(
        exists=True,
        body="CI failed because TypeScript has a missing export. token=abc",
        visibility="group_members",
        can_delete=True,
        can_edit=False,
    )
    drifts = detect_drift(release, observed)
    append_jsonl("/tmp/release-drift.jsonl", drifts)
    print(drifts)
```

这段代码故意只做“检测”，不直接修复。原因很简单：外部副本漂移通常意味着有人改过、删过或权限变了，自动覆盖可能制造第二次事故。

---

## 4. pi-mono：ReleaseVerificationMiddleware

生产版把复核做成后台 job。它从 Release Ledger 取出需要复核的发布记录，通过 provider adapter 读外部状态，再写 checkpoint 或 drift。

```ts
// pi-mono/packages/agent-security/src/release-verification.ts
export type Provider = 'telegram' | 'github' | 'email' | 'statuspage'
export type DriftKind =
  | 'missing'
  | 'content_changed'
  | 'visibility_changed'
  | 'recall_capability_lost'
  | 'link_broken'

export interface ExternalRef {
  provider: Provider
  target: string
  messageId: string
  url?: string
}

export interface ReleaseRecord {
  releaseId: string
  artifactId: string
  expectedContentHash: string
  expectedVisibility: 'private' | 'team' | 'group_members' | 'public'
  externalRef: ExternalRef
  recallPolicy: {
    requireDeleteCapability: boolean
    requireEditCapability: boolean
  }
  nextVerifyAt: string
}

export interface ObservedPublication {
  exists: boolean
  contentHash?: string
  visibility?: ReleaseRecord['expectedVisibility']
  canDelete: boolean
  canEdit: boolean
  urlOk: boolean
}

export interface PublicationAdapter {
  fetch(ref: ExternalRef): Promise<ObservedPublication>
}

export interface VerificationStore {
  dueReleases(now: Date, limit: number): Promise<ReleaseRecord[]>
  appendCheckpoint(input: {
    releaseId: string
    status: 'healthy' | 'drifted'
    observed: ObservedPublication
    checkedAt: string
  }): Promise<void>
  appendDrift(input: {
    releaseId: string
    kind: DriftKind
    severity: 'low' | 'medium' | 'high' | 'critical'
    recommendedAction: 'notice' | 'edit' | 'delete' | 'manual_review'
    observed: ObservedPublication
  }): Promise<void>
  scheduleNext(releaseId: string, nextVerifyAt: Date): Promise<void>
}

export class ReleaseVerificationJob {
  constructor(
    private readonly store: VerificationStore,
    private readonly adapters: Record<Provider, PublicationAdapter>,
  ) {}

  async run(now = new Date()) {
    const releases = await this.store.dueReleases(now, 100)

    for (const release of releases) {
      const adapter = this.adapters[release.externalRef.provider]
      const observed = await adapter.fetch(release.externalRef)
      const drifts = classifyDrifts(release, observed)

      await this.store.appendCheckpoint({
        releaseId: release.releaseId,
        status: drifts.length === 0 ? 'healthy' : 'drifted',
        observed,
        checkedAt: now.toISOString(),
      })

      for (const drift of drifts) {
        await this.store.appendDrift({
          releaseId: release.releaseId,
          ...drift,
          observed,
        })
      }

      await this.store.scheduleNext(release.releaseId, nextVerifyTime(now, drifts.length))
    }
  }
}

export function classifyDrifts(record: ReleaseRecord, observed: ObservedPublication) {
  const drifts: Array<{
    kind: DriftKind
    severity: 'low' | 'medium' | 'high' | 'critical'
    recommendedAction: 'notice' | 'edit' | 'delete' | 'manual_review'
  }> = []

  if (!observed.exists) {
    return [{ kind: 'missing', severity: 'medium', recommendedAction: 'notice' }]
  }

  if (!observed.urlOk) {
    drifts.push({ kind: 'link_broken', severity: 'medium', recommendedAction: 'notice' })
  }

  if (observed.contentHash !== record.expectedContentHash) {
    drifts.push({ kind: 'content_changed', severity: 'high', recommendedAction: 'manual_review' })
  }

  if (observed.visibility !== record.expectedVisibility) {
    drifts.push({
      kind: 'visibility_changed',
      severity: observed.visibility === 'public' ? 'critical' : 'medium',
      recommendedAction: 'manual_review',
    })
  }

  if (record.recallPolicy.requireDeleteCapability && !observed.canDelete) {
    drifts.push({ kind: 'recall_capability_lost', severity: 'high', recommendedAction: 'manual_review' })
  }

  if (record.recallPolicy.requireEditCapability && !observed.canEdit) {
    drifts.push({ kind: 'recall_capability_lost', severity: 'medium', recommendedAction: 'manual_review' })
  }

  return drifts
}

function nextVerifyTime(now: Date, driftCount: number) {
  const minutes = driftCount > 0 ? 15 : 24 * 60
  return new Date(now.getTime() + minutes * 60_000)
}
```

这里有两个关键设计：

- `fetch()` 和 `delete()/edit()` 分开。复核是只读动作，修复是副作用动作，必须重新过 policy。
- drift 后缩短复核间隔。健康记录一天查一次，有问题的记录 15 分钟后再查，直到关闭。

---

## 5. OpenClaw 实战：Telegram 课程发布的复核任务

用 OpenClaw 发课程到 Telegram 后，不只记录 `messageId`，还要记录期望 hash：

```json
{
  "releaseId": "rel_lesson_336",
  "artifactId": "decl_lesson_336",
  "channel": "telegram",
  "externalRef": {
    "provider": "telegram",
    "target": "-5115329245",
    "messageId": "12160"
  },
  "expectedContentHash": "sha256:...",
  "expectedVisibility": "group_members",
  "recallPolicy": {
    "requireDeleteCapability": true,
    "requireEditCapability": true
  },
  "nextVerifyAt": "2026-05-18T00:00:00.000Z"
}
```

复核 cron 可以做三件事：

1. 用 Telegram adapter 拉取 message 状态。
2. 计算当前内容 hash，和 `expectedContentHash` 比较。
3. 如果 `content_changed` 或 `recall_capability_lost`，发给 owner 或安全队列，不直接覆盖。

伪代码：

```ts
const due = await releaseLedger.dueForVerification(new Date())

for (const release of due) {
  const observed = await telegramAdapter.fetchMessage(release.externalRef)
  const drifts = classifyDrifts(release, observed)

  await releaseLedger.appendCheckpoint(release.releaseId, observed, drifts)

  if (drifts.some(d => d.severity === 'critical' || d.severity === 'high')) {
    await taskflow.create({
      title: `Review release drift: ${release.releaseId}`,
      evidenceRefs: [release.artifactId],
      drifts,
    })
  }
}
```

注意：如果 Telegram API 取不到历史消息，也不要假装复核成功。应该记录 `receipt_unconfirmed` 或 `provider_unverifiable`，并降低 confidence。

---

## 6. 工程边界

不要把所有漂移都当事故：

- 消息被正常删除：可能是 recall 成功，要看 ledger state。
- GitHub 评论被 maintainer 编辑：可能是协作修改，要进入 manual review。
- 公开页面 404：对外承诺可能失效，要 notice。
- 权限丢失：不代表泄漏，但代表未来无法自动召回，风险很高。

推荐默认策略：

```yaml
releaseVerification:
  defaultInterval: 24h
  driftInterval: 15m
  critical:
    visibility_changed_to_public: manual_review
    content_changed_with_secret_pattern: delete_then_review
  high:
    content_changed: manual_review
    recall_capability_lost: manual_review
  medium:
    missing: notice
    link_broken: notice
```

一句话总结：

> Release Ledger 让 Agent 知道证据发到了哪里；Release Verification 让 Agent 持续确认外部世界没有悄悄变样。

成熟 Agent 不是“我当时发对了”，而是“我还能证明现在外部副本仍然符合当时批准的内容和权限边界”。
