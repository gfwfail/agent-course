# 337. Agent 外部发布更正与撤回通知（Release Correction & Retraction Notices）

上一课讲了 **Release Verification**：Agent 发出去的 Telegram 消息、GitHub 评论、Email、状态页更新，不能只在发送当时验证一次，还要持续复核外部副本有没有漂移。

今天继续讲漂移后的收口动作：发现已经发布的内容有错、过期、权限不对、漏脱敏时，Agent 不能只在内部账本里标记 `bad`，还要对外做可追踪的更正或撤回通知。

核心思想：

> 外部发布一旦影响了别人，就形成了承诺。Agent 发现发布内容不再可信时，必须生成 Correction / Retraction Notice，绑定原始 releaseId 和外部 messageId，说明旧内容哪里不可信、当前状态是什么、后续该相信哪份内容，并把通知本身重新写入 Release Ledger。

---

## 1. 为什么不能只 delete？

很多系统把“发错消息”的处理简化成删除。但对 Agent 来说，delete 只解决“旧内容还在不在”，不解决“已经看到的人怎么办”。

典型场景：

- 发到群里的部署结论后来被验证为错误，直接删除会让看到旧消息的人继续按旧结论行动。
- GitHub 评论里引用的证据过期了，删掉评论会破坏审计链。
- Email 无法真正撤回，只能补发更正通知。
- Status Page 已公开展示过错误影响范围，必须留下更正记录。
- 原消息含敏感片段时，既要尽快删除，也要保留内部审计证据说明删了什么、为什么删。

所以外部修复通常不是一个动作，而是一个策略决策：

```text
DriftEvent / RevocationEvent
  -> Correction Planner
  -> Notice Artifact
  -> Channel Policy
  -> edit/delete/send_notice/manual_review
  -> Release Ledger append
  -> Verification Schedule
```

成熟 Agent 要区分三种动作：

- **Correction**：旧内容部分错误，用新内容纠正。
- **Retraction**：旧内容整体不可信，明确撤回。
- **Removal**：旧内容违规或敏感，尽快删除，但删除后仍可能需要补发安全的 notice。

---

## 2. 数据结构：CorrectionNotice

更正通知必须是结构化对象，不应该让 LLM 临场写一段自由文本就直接发。

```json
{
  "noticeId": "notice_01H...",
  "noticeType": "correction",
  "sourceReleaseId": "rel_01H...",
  "sourceExternalRef": {
    "provider": "telegram",
    "target": "-5115329245",
    "messageId": "12167"
  },
  "reason": "content_changed",
  "audience": "group_members",
  "replacementArtifactId": "artifact_01H...",
  "claims": [
    {
      "oldClaim": "CI is green and release is safe to deploy.",
      "status": "replaced",
      "newClaim": "CI is failing on TypeScript export validation; do not deploy yet."
    }
  ],
  "requiredActions": ["ignore_previous_message", "use_replacement_artifact"],
  "createdAt": "2026-05-17T09:30:00.000Z"
}
```

关键字段：

- `sourceReleaseId`：绑定被更正的发布记录。
- `sourceExternalRef`：绑定外部副本，方便用户和审计者定位。
- `reason`：来自 drift / revocation / incident，不允许凭空编。
- `claims`：按 claim 粒度说明旧内容哪部分错了。
- `replacementArtifactId`：新的可信内容，不直接引用 raw evidence。
- `requiredActions`：告诉接收者应该忽略旧消息、等待复核、还是使用新版本。

更正通知本身也是一次外发，所以它要走同样的 egress policy、降密、发布账本和发布后复核。

---

## 3. learn-claude-code：最小 Correction Planner

教学版先做一个纯函数：输入 drift 和旧 release，输出建议动作与通知草稿。重点是“可解释、可审计”，不是让模型自由发挥。

```python
# learn-claude-code/release_correction_planner.py
from dataclasses import dataclass, asdict
from datetime import datetime, timezone
from typing import Literal
import hashlib
import json

NoticeType = Literal["correction", "retraction", "removal_notice"]
Action = Literal["edit_original", "delete_original", "send_notice", "manual_review"]
DriftKind = Literal[
    "content_changed",
    "visibility_changed",
    "source_evidence_revoked",
    "secret_pattern_detected",
    "link_broken",
]

@dataclass
class ExternalRef:
    provider: str
    target: str
    message_id: str

@dataclass
class ReleaseRecord:
    release_id: str
    external_ref: ExternalRef
    audience: str
    expected_summary: str
    artifact_id: str

@dataclass
class DriftEvent:
    drift_id: str
    release_id: str
    kind: DriftKind
    severity: str
    observed_summary: str | None = None

@dataclass
class CorrectionNotice:
    notice_id: str
    notice_type: NoticeType
    source_release_id: str
    source_external_ref: ExternalRef
    reason: str
    audience: str
    body: str
    actions: list[Action]
    created_at: str

def stable_notice_id(release_id: str, drift_id: str) -> str:
    raw = f"{release_id}:{drift_id}"
    return "notice_" + hashlib.sha256(raw.encode("utf-8")).hexdigest()[:16]

def plan_notice(release: ReleaseRecord, drift: DriftEvent) -> CorrectionNotice:
    if drift.kind == "secret_pattern_detected":
        notice_type: NoticeType = "removal_notice"
        actions: list[Action] = ["delete_original", "send_notice", "manual_review"]
        body = (
            "更正：上一条发布内容已撤回，因为自动复核发现可能包含不应公开的信息。"
            "请不要继续转发或依赖旧消息；新的脱敏说明会在复核后发布。"
        )
    elif drift.kind == "source_evidence_revoked":
        notice_type = "retraction"
        actions = ["send_notice", "manual_review"]
        body = (
            "撤回：上一条发布内容依赖的证据已经失效，当前结论不再可信。"
            "请忽略旧消息，等待带新证据的更新。"
        )
    elif drift.kind == "content_changed":
        notice_type = "correction"
        actions = ["edit_original", "send_notice"]
        body = (
            "更正：上一条发布内容与发布账本记录不一致，已进入复核。"
            f"原摘要：{release.expected_summary}。"
            f"当前观察：{drift.observed_summary or '无法确认'}。"
            "在复核完成前，请不要把旧消息作为事实来源。"
        )
    elif drift.kind == "visibility_changed":
        notice_type = "retraction"
        actions = ["send_notice", "manual_review"]
        body = "撤回：上一条发布内容的可见范围发生变化，已暂停信任并进入人工复核。"
    else:
        notice_type = "correction"
        actions = ["send_notice"]
        body = "更正：上一条发布内容引用的外部链接已失效，请以最新发布记录为准。"

    return CorrectionNotice(
        notice_id=stable_notice_id(release.release_id, drift.drift_id),
        notice_type=notice_type,
        source_release_id=release.release_id,
        source_external_ref=release.external_ref,
        reason=drift.kind,
        audience=release.audience,
        body=body,
        actions=actions,
        created_at=datetime.now(timezone.utc).isoformat(),
    )

if __name__ == "__main__":
    release = ReleaseRecord(
        release_id="rel_336",
        external_ref=ExternalRef("telegram", "-5115329245", "12167"),
        audience="group_members",
        expected_summary="Release verification is healthy.",
        artifact_id="artifact_336",
    )
    drift = DriftEvent(
        drift_id="drift_abc",
        release_id="rel_336",
        kind="content_changed",
        severity="high",
        observed_summary="Message body no longer matches approved artifact.",
    )
    notice = plan_notice(release, drift)
    print(json.dumps(asdict(notice), ensure_ascii=False, indent=2))
```

这里的重点：

1. `notice_id` 是幂等的，同一个 drift 不会重复刷屏。
2. `body` 只引用摘要，不把 raw evidence 重新外发。
3. 高风险 drift 默认 `manual_review`，不让 Agent 自动覆盖复杂事故。

---

## 4. pi-mono：CorrectionNoticeMiddleware

生产版可以放在 Release Verification 之后。它不是消息发送器，而是“外部修复计划器”：把 drift 转成受控副作用。

```ts
// pi-mono/packages/agent-security/src/correction-notice.ts
export type NoticeType = 'correction' | 'retraction' | 'removal_notice'
export type CorrectionAction =
  | 'edit_original'
  | 'delete_original'
  | 'send_notice'
  | 'manual_review'

export interface ExternalRef {
  provider: 'telegram' | 'github' | 'email' | 'statuspage'
  target: string
  messageId: string
  url?: string
}

export interface ReleaseRecord {
  releaseId: string
  artifactId: string
  externalRef: ExternalRef
  audience: 'owner' | 'team' | 'group_members' | 'public'
  expectedContentHash: string
  recallPolicy: {
    allowEdit: boolean
    allowDelete: boolean
    requireNoticeOnDelete: boolean
  }
}

export interface DriftEvent {
  driftId: string
  releaseId: string
  kind:
    | 'content_changed'
    | 'visibility_changed'
    | 'source_evidence_revoked'
    | 'secret_pattern_detected'
    | 'link_broken'
  severity: 'low' | 'medium' | 'high' | 'critical'
}

export interface CorrectionNotice {
  noticeId: string
  noticeType: NoticeType
  sourceReleaseId: string
  sourceExternalRef: ExternalRef
  reason: DriftEvent['kind']
  audience: ReleaseRecord['audience']
  body: string
  actions: CorrectionAction[]
  artifactRefs: string[]
}

export interface ChannelPolicy {
  canSendNotice(audience: ReleaseRecord['audience']): boolean
  canEdit(ref: ExternalRef): boolean
  canDelete(ref: ExternalRef): boolean
}

export function planCorrectionNotice(
  release: ReleaseRecord,
  drift: DriftEvent,
  policy: ChannelPolicy,
): CorrectionNotice {
  const actions: CorrectionAction[] = []
  let noticeType: NoticeType = 'correction'

  if (drift.kind === 'secret_pattern_detected') {
    noticeType = 'removal_notice'
    if (release.recallPolicy.allowDelete && policy.canDelete(release.externalRef)) {
      actions.push('delete_original')
    } else {
      actions.push('manual_review')
    }
    actions.push('send_notice')
  } else if (
    drift.kind === 'source_evidence_revoked' ||
    drift.kind === 'visibility_changed'
  ) {
    noticeType = 'retraction'
    actions.push('send_notice', 'manual_review')
  } else if (
    drift.kind === 'content_changed' &&
    release.recallPolicy.allowEdit &&
    policy.canEdit(release.externalRef)
  ) {
    actions.push('edit_original', 'send_notice')
  } else {
    actions.push('send_notice')
  }

  if (!policy.canSendNotice(release.audience)) {
    return {
      noticeId: `notice_${release.releaseId}_${drift.driftId}`,
      noticeType,
      sourceReleaseId: release.releaseId,
      sourceExternalRef: release.externalRef,
      reason: drift.kind,
      audience: release.audience,
      body: 'Notice blocked by channel policy; manual review required.',
      actions: ['manual_review'],
      artifactRefs: [release.artifactId],
    }
  }

  return {
    noticeId: `notice_${release.releaseId}_${drift.driftId}`,
    noticeType,
    sourceReleaseId: release.releaseId,
    sourceExternalRef: release.externalRef,
    reason: drift.kind,
    audience: release.audience,
    body: renderNotice(noticeType, drift.kind),
    actions,
    artifactRefs: [release.artifactId],
  }
}

function renderNotice(type: NoticeType, reason: DriftEvent['kind']): string {
  if (type === 'removal_notice') {
    return '更正：上一条消息已撤回，因为复核发现它可能包含不应公开的信息。请不要继续依赖或转发旧消息。'
  }
  if (type === 'retraction') {
    return `撤回：上一条消息因为 ${reason} 已不再可信。请以之后带新证据的更新为准。`
  }
  return `更正：上一条消息因为 ${reason} 已进入修正流程。请以本更正后的内容为准。`
}
```

这个 middleware 的边界很清楚：

- 它决定 **应该修复什么**；
- Channel adapter 决定 **能不能 edit/delete/send**；
- Egress policy 决定 **notice 内容能不能外发**；
- Release Ledger 记录 **修复动作是否真的发生**。

---

## 5. OpenClaw 实战：Telegram 更正闭环

用课程 cron 举例，发布课程后应该记录：

```json
{
  "releaseId": "lesson_337",
  "artifactId": "lesson_md_337",
  "externalRef": {
    "provider": "telegram",
    "target": "-5115329245",
    "messageId": "12201"
  },
  "expectedContentHash": "sha256:...",
  "recallPolicy": {
    "allowEdit": true,
    "allowDelete": true,
    "requireNoticeOnDelete": true
  },
  "verifyAfter": "2026-05-18T09:30:00.000Z"
}
```

如果后续复核发现消息内容被改了：

```text
DriftEvent(content_changed, high)
  -> planCorrectionNotice()
  -> edit 原消息：开头标记“已更正”
  -> send_notice：发一条短更正说明，引用原 messageId
  -> ledger append: correction_notice_sent
  -> schedule verification in 15m
```

Telegram 上推荐的更正文案要短：

```text
更正：上一条 Agent 课程消息已进入修正，原因是发布后复核发现内容与批准稿不一致。
请以这条更正后的版本为准；旧消息不要再作为引用来源。
Ref: releaseId=lesson_337, messageId=12201
```

注意：对群聊，不要把内部 hash、raw evidence、审计备注全贴出去。群里只需要知道“旧消息是否可信”和“现在该信哪条”。

---

## 6. 工程规则

建议默认策略：

```yaml
releaseCorrection:
  idempotencyKey: sourceReleaseId + driftId
  secretPatternDetected:
    actions: [delete_original, send_notice, manual_review]
    noticeType: removal_notice
  sourceEvidenceRevoked:
    actions: [send_notice, manual_review]
    noticeType: retraction
  contentChanged:
    actions: [edit_original, send_notice]
    noticeType: correction
  linkBroken:
    actions: [send_notice]
    noticeType: correction
  publicAudience:
    requireManualReviewFor: [visibility_changed, secret_pattern_detected]
  notice:
    maxLength: 800
    includeRawEvidence: false
    includeSourceReleaseId: true
    includeExternalRef: true
```

几个坑：

1. **更正通知重复发送**
   必须用 `sourceReleaseId + driftId` 做幂等键，否则复核 job 每 15 分钟刷一次群。

2. **更正内容再次泄漏**
   Notice 也要走降密流水线，不能为了说明泄漏而复述泄漏内容。

3. **删除破坏审计链**
   外部消息可以删，内部 Release Ledger 不能删；内部只追加 `deleted` / `corrected` / `retracted` 事件。

4. **只改原消息不发通知**
   如果旧消息已经被人看到，只 silent edit 可能不够。高风险更正应该额外发 notice。

5. **无法撤回的渠道假装成功**
   Email、第三方转发、截图都无法可靠删除。此时动作应降级为 retraction notice + audit record。

一句话总结：

> Release Verification 发现外部副本不可信；Correction / Retraction Notice 负责把这个事实告诉受影响的人，并把修复动作重新纳入证据链。

成熟 Agent 不只是“发现自己发错了”，还要能体面、及时、可审计地告诉外部世界：上一条不该再信，新的可信边界在哪里。
