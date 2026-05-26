# 413. Agent 护栏审计发现的复发检测与回归重放（Guard Finding Recurrence Detection & Regression Replay）

上一课讲了 **Guard Finding Reliability Metrics & Repair SLO**：不要只看 ticket 关闭数，要把 false allow、false block、积压、超时、修复质量和复发风险变成可度量的运营面板。

今天继续往后一层：**指标发现风险以后，系统要能判断“这是不是同一个问题又回来了”。**

一句话：**Guard finding 关闭后不能只靠 SLO 面板观察趋势；要把每个 close_verified 的 finding 固化成 Recurrence Signature 和 Regression Replay Case，后续审计样本命中相同风险指纹时自动判定 repeat/new/adjacent/regression，并决定 reopen、extend replay、升级 owner 或创建新 finding。**

---

## 1. 为什么复发检测不能只看标题相似

很多团队会这样判断复发：

~~~text
finding title 相似 -> 可能复发
guardId 一样 -> 可能复发
同一个 owner -> 可能复发
~~~

这太粗。

Agent 护栏问题的复发通常藏在行为路径里：

1. 同一个 guardId，但输入证据不同，实际是新问题；
2. 不同 guardId，但最终错误 actionClass 一样，实际是旧问题换了入口；
3. 修复后 false allow 消失了，但 adjacent risk surface 出现 false block；
4. canary 当时通过，active 后因为 runtime group 差异又复现；
5. prompt、tool schema、model provider 变化后，旧 replay case 没覆盖新路径。

所以复发检测要看 **风险指纹**，不是看一句描述。

---

## 2. Recurrence Signature：关闭时就要生成

复发检测不是发现新 finding 时才临时算。正确时机是在 close gate 通过时生成。

一个最小签名可以包含：

~~~text
signatureId
sourceFindingId
guardId
findingKind
riskSurface
actionClass
evidenceShapeHash
decisionPathHash
expectedGuardDecision
blockedToolNames
runtimeGroup
modelFamily
repairCommit
replayCaseIds
expiresAt
~~~

注意：不要把 raw evidence 放进 signature。用 shape/hash/claims 就够了。

原因很简单：signature 会被长期保留、跨团队读取、进入指标系统。它应该能证明“问题形状”，但不要泄漏用户输入、密钥、订单详情或私密内容。

---

## 3. 四种命中结果

新审计样本进来后，不要只输出 yes/no。至少分四类。

~~~text
repeat:
  同一个风险指纹再次出现，旧修复没有真正挡住。

adjacent:
  相邻风险面出现相似错误，旧修复 scope 太窄。

regression:
  旧 replay case 曾经通过，现在失败了，说明新变更打破了修复。

new:
  没有足够相似签名，创建新 finding。
~~~

这四类对应的动作不一样：

~~~text
repeat -> reopen 原 finding，升级 owner，缩短 SLO
adjacent -> 创建 linked finding，要求扩展 replay matrix
regression -> block release 或 rollback guard pack
new -> 正常 triage
~~~

---

## 4. learn-claude-code：纯函数复发判定器

教学版先用纯函数，把“新审计样本”和“历史签名”比对。

~~~py
from dataclasses import dataclass
from typing import Literal


FindingKind = Literal["false_allow", "false_block", "explainability_gap", "evidence_overread", "policy_stale", "outcome_mismatch"]
MatchKind = Literal["repeat", "adjacent", "regression", "new"]


@dataclass(frozen=True)
class RecurrenceSignature:
    signature_id: str
    source_finding_id: str
    guard_id: str
    kind: FindingKind
    risk_surface: str
    action_class: str
    evidence_shape_hash: str
    decision_path_hash: str
    expected_guard_decision: str
    repair_commit: str
    replay_case_ids: tuple[str, ...]


@dataclass(frozen=True)
class AuditSample:
    sample_id: str
    guard_id: str
    kind: FindingKind
    risk_surface: str
    action_class: str
    evidence_shape_hash: str
    decision_path_hash: str
    observed_decision: str
    failing_replay_case_id: str | None = None


@dataclass(frozen=True)
class RecurrenceMatch:
    match_kind: MatchKind
    signature_id: str | None
    source_finding_id: str | None
    confidence: float
    next_action: str
    reason: str


def detect_recurrence(sample: AuditSample, signatures: list[RecurrenceSignature]) -> RecurrenceMatch:
    for sig in signatures:
        if sample.failing_replay_case_id and sample.failing_replay_case_id in sig.replay_case_ids:
            return RecurrenceMatch(
                match_kind="regression",
                signature_id=sig.signature_id,
                source_finding_id=sig.source_finding_id,
                confidence=0.99,
                next_action="block_release_and_reopen_regression",
                reason="previous replay case failed again",
            )

        same_fingerprint = (
            sample.guard_id == sig.guard_id
            and sample.kind == sig.kind
            and sample.risk_surface == sig.risk_surface
            and sample.action_class == sig.action_class
            and sample.evidence_shape_hash == sig.evidence_shape_hash
            and sample.decision_path_hash == sig.decision_path_hash
        )
        if same_fingerprint:
            return RecurrenceMatch(
                match_kind="repeat",
                signature_id=sig.signature_id,
                source_finding_id=sig.source_finding_id,
                confidence=0.95,
                next_action="reopen_source_finding_and_escalate_owner",
                reason="same guard/risk/action/evidence/decision fingerprint",
            )

        adjacent = (
            sample.kind == sig.kind
            and sample.action_class == sig.action_class
            and sample.evidence_shape_hash == sig.evidence_shape_hash
            and sample.risk_surface != sig.risk_surface
        )
        if adjacent:
            return RecurrenceMatch(
                match_kind="adjacent",
                signature_id=sig.signature_id,
                source_finding_id=sig.source_finding_id,
                confidence=0.72,
                next_action="create_linked_finding_and_expand_replay_matrix",
                reason="same action/evidence shape appeared on adjacent risk surface",
            )

    return RecurrenceMatch(
        match_kind="new",
        signature_id=None,
        source_finding_id=None,
        confidence=0.0,
        next_action="create_new_finding",
        reason="no matching recurrence signature",
    )
~~~

这个函数故意保守：只有 replay case 失败或完整指纹命中才判定强复发；相邻风险面只给 adjacent，不直接当 repeat。

---

## 5. pi-mono：用事件流投影 Recurrence Index

在 pi-mono 这种事件驱动架构里，复发检测不要塞进单个工具调用。更合理的是做一个 projector：

~~~ts
type FindingClosedEvent = {
  type: "guard_finding.closed_verified";
  findingId: string;
  guardId: string;
  kind: "false_allow" | "false_block" | "explainability_gap" | "evidence_overread" | "policy_stale" | "outcome_mismatch";
  riskSurface: string;
  actionClass: string;
  evidenceShapeHash: string;
  decisionPathHash: string;
  expectedGuardDecision: "allow" | "warn" | "dry_run" | "block";
  repairCommit: string;
  replayCaseIds: string[];
};

type AuditSampleEvent = {
  type: "guard_audit.sample_failed";
  sampleId: string;
  guardId: string;
  kind: FindingClosedEvent["kind"];
  riskSurface: string;
  actionClass: string;
  evidenceShapeHash: string;
  decisionPathHash: string;
  observedDecision: string;
  failingReplayCaseId?: string;
};

type RecurrenceDetectedEvent = {
  type: "guard_finding.recurrence_detected";
  sampleId: string;
  matchKind: "repeat" | "adjacent" | "regression" | "new";
  sourceFindingId?: string;
  signatureId?: string;
  confidence: number;
  nextAction: "reopen" | "linked_finding" | "block_release" | "new_finding";
};
~~~

然后流程是：

~~~text
closed_verified -> build RecurrenceSignature -> update RecurrenceIndex
sample_failed -> query RecurrenceIndex -> emit recurrence_detected
recurrence_detected -> reopen / linked finding / block release / new triage
~~~

这跟 pi-mono extension 里的 guard/hook 思路一致：工具可以继续专注执行，复发判断作为事件旁路，统一留下证据。

---

## 6. OpenClaw 课程 Cron 的真实例子

拿我们这个课程 cron 做例子。

每 3 小时它会做这些副作用：

~~~text
写 lessons/413-xxx.md
更新 README.md
更新 TOOLS.md 已讲内容
发送 Telegram 群消息
git commit + push
写 memory/YYYY-MM-DD.md
~~~

假设之前出现过一个 finding：

~~~text
kind: false_allow
riskSurface: course_publish
actionClass: telegram_send
问题：README 和 TOOLS 已更新，但 Telegram 发送失败后仍然 git push，导致外部课程目录与群消息不一致
~~~

关闭时应该生成 recurrence signature：

~~~json
{
  "signatureId": "sig_course_publish_pending_telegram",
  "sourceFindingId": "finding_telegram_outbox_042",
  "guardId": "external_effect_outcome_barrier",
  "riskSurface": "course_publish",
  "actionClass": "git_push_after_telegram_send",
  "evidenceShapeHash": "lesson+readme+tools+telegram_message_id+commit_sha",
  "expectedGuardDecision": "block_until_telegram_terminal_matched",
  "replayCaseIds": ["replay_course_telegram_pending_before_push"]
}
~~~

以后如果 audit sample 发现：

~~~text
messageId 缺失
commit 已推送
memory 写了“已发布”
~~~

这不是新问题，是 repeat 或 regression。系统应该重开原 finding，阻断下游发布，并要求补 Telegram 对账或更正 memory，而不是开一个普通低优先级 bug。

---

## 7. 实战落地清单

把复发检测落地时，至少做这几件事：

1. close gate 通过时生成 RecurrenceSignature；
2. 每个 signature 只存安全摘要、hash、shape，不存 raw evidence；
3. replay case id 必须写进 signature；
4. audit sample 先查 RecurrenceIndex，再创建 finding；
5. repeat 自动 reopen 原 finding，并提升 severity 或缩短 SLO；
6. adjacent 自动要求扩展 replay matrix；
7. regression 自动阻断 release 或回滚 guard pack；
8. 所有判断写 RecurrenceDetectedEvent，方便事后复盘。

---

## 8. 常见坑

**坑一：只按 guardId 匹配。**

同一个 guardId 可能覆盖多个风险面，按 guardId 匹配会把新问题误判成复发。

**坑二：signature 放 raw prompt。**

复发索引会长期存在，raw prompt 容易造成隐私和凭证泄漏。用 claim/hash/shape。

**坑三：repeat 只创建新 ticket。**

repeat 的意义是“旧修复不够”，应该 reopen 原 finding 并升级验证要求。

**坑四：adjacent 不扩 replay。**

相邻风险面命中说明原 replay matrix 太窄。只修当前 case，下次还会从另一个入口回来。

---

## 9. 记住这一句

**成熟 Agent 的护栏修复，不是 close 以后祈祷别再出事；而是把每次关闭变成可匹配的复发指纹，把每次新失败先过复发检测，再决定重开、扩展回归、阻断发布还是创建新问题。**
