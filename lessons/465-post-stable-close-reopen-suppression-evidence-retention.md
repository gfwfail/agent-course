# 465. Agent 稳定关闭后的重开抑制与证据保留（Post-Stable Close Reopen Suppression & Evidence Retention）

上一课讲了 **Post-Replay Observation Window & Regression Watch**：ReplayCloseoutReceipt 通过以后，还要观察迟到副作用、状态回潮、旧任务复活和下游覆盖，最后写 StableCloseReceipt 或 ReopenReplayCase。

但 StableCloseReceipt 也不是终点。生产里还有两个细节很容易被忽略：

- 稳定关闭后，旧信号可能重复上报，把已经关闭的 retry 反复 reopen；
- 为了省空间直接清理原始证据，后面真有复发时又证明不了为什么当初可以稳定关闭。

所以今天补上 **Post-Stable Close Reopen Suppression & Evidence Retention**：

~~~text
StableCloseReceipt
  -> ReopenSuppressionWindow
  -> EvidenceRetentionPlan
  -> LateSignalTriage
  -> SuppressionReceipt | ReopenReplayCase
  -> RetentionCloseReceipt
~~~

一句话：**稳定关闭后，不要让重复旧信号制造告警风暴，也不要把关闭依据清理到不可审计；要用抑制窗口和证据保留计划把“别重复吵”和“真复发能追溯”同时做好。**

---

## 1. 为什么 stable close 后还要管

StableCloseReceipt 证明的是：

~~~text
观察窗口内没有发现足够证据要求重开。
~~~

它不证明：

~~~text
未来任何相似信号都应该被忽略。
~~~

常见事故：

- 监控系统每 5 分钟重复上报同一个 late_effect，自动化每次都开一个 ReopenReplayCase；
- orphan reaper 的旧日志被延迟索引，close 后被误判成 descendant_resurrection；
- 下游 overwrite 信号已经在观察窗口处理过，但 telemetry replay 又发了一遍；
- 为了清理上下文，把 closeout / observation / signal probe 原文全删了，后面审计只能看到一句 “close_stable”；
- 真正的新复发和旧信号长得很像，没有 fingerprint / time window / evidence hash 就分不出来。

所以 stable close 后至少要做两件事：

~~~text
1. Reopen suppression：短时间内重复旧信号只写 suppression receipt，不重复重开。
2. Evidence retention：原始证据按风险降密保留，保留 hash、摘要、关键 receipt，而不是全删或全留。
~~~

这一步很像支付系统的 webhook 去重：重复通知不能重复扣款，但也不能把所有通知都丢掉，必须留下“为什么这次被抑制”的收据。

---

## 2. 最小数据模型

稳定关闭后生成抑制窗口：

~~~json
{
  "kind": "ReopenSuppressionWindow",
  "stableCloseReceiptId": "stable_465",
  "retryRunId": "run_retry_789",
  "suppressionUntil": "2026-06-02T18:30:00+11:00",
  "dedupeKeys": [
    "late_effect:telegram:old-run:13124",
    "descendant:subagent:agent_123"
  ],
  "allowReopenWhen": [
    "new_dedupe_key",
    "severity_increased",
    "payload_hash_changed",
    "source_after_suppression_window"
  ]
}
~~~

同时写证据保留计划：

~~~json
{
  "kind": "EvidenceRetentionPlan",
  "stableCloseReceiptId": "stable_465",
  "retainUntil": "2026-07-02T15:30:00+11:00",
  "rawEvidence": "purge_after_24h",
  "summaries": "retain_30d",
  "hashes": "retain_180d",
  "receipts": [
    "ReplayCloseoutReceipt",
    "StableCloseReceipt",
    "SuppressionReceipt"
  ],
  "redactionProfile": "operational_minimum"
}
~~~

当后续信号进来时，输出不要只写 ignored：

~~~json
{
  "kind": "SuppressionReceipt",
  "signalId": "sig_999",
  "retryRunId": "run_retry_789",
  "decision": "suppress_duplicate",
  "dedupeKey": "late_effect:telegram:old-run:13124",
  "matchedStableCloseReceiptId": "stable_465",
  "evidenceHash": "sha256:signal"
}
~~~

如果它不是重复旧信号，则正常重开：

~~~json
{
  "kind": "ReopenReplayCase",
  "signalId": "sig_1000",
  "retryRunId": "run_retry_789",
  "reason": "payload_hash_changed",
  "nextAction": "reconcile"
}
~~~

关键点：**抑制不是沉默，而是写一张 suppression receipt；保留不是囤积所有原文，而是按审计价值保留最小证据。**

---

## 3. learn-claude-code：教学版晚到信号判定器

教学版可以先写纯函数：输入 stable close 后的晚到信号，判断 suppress / reopen / manual_review。

~~~python
# learn_claude_code/runtime/post_stable_reopen_suppression.py
from dataclasses import dataclass
from datetime import datetime
from typing import Literal

Decision = Literal["suppress_duplicate", "reopen_replay", "manual_review"]
Severity = Literal["info", "warning", "critical"]


@dataclass(frozen=True)
class ReopenSuppressionWindow:
    retry_run_id: str
    stable_close_receipt_id: str
    suppression_until: datetime
    dedupe_keys: set[str]
    payload_hashes: set[str]


@dataclass(frozen=True)
class LateSignal:
    signal_id: str
    retry_run_id: str
    observed_at: datetime
    dedupe_key: str
    payload_hash: str
    severity: Severity
    source: str


@dataclass(frozen=True)
class TriageResult:
    decision: Decision
    reason: str
    receipt_ref: str


def triage_late_signal(
    window: ReopenSuppressionWindow,
    signal: LateSignal,
) -> TriageResult:
    if signal.retry_run_id != window.retry_run_id:
        return TriageResult("manual_review", "retry_run_mismatch", signal.signal_id)

    within_window = signal.observed_at <= window.suppression_until
    known_key = signal.dedupe_key in window.dedupe_keys
    known_payload = signal.payload_hash in window.payload_hashes

    if within_window and known_key and known_payload and signal.severity != "critical":
        return TriageResult(
            "suppress_duplicate",
            "same_dedupe_key_and_payload_within_window",
            f"suppression:{signal.signal_id}",
        )

    if signal.severity == "critical":
        return TriageResult("reopen_replay", "critical_signal_after_stable_close", signal.signal_id)

    if not known_key:
        return TriageResult("reopen_replay", "new_dedupe_key", signal.signal_id)

    if not known_payload:
        return TriageResult("reopen_replay", "payload_hash_changed", signal.signal_id)

    return TriageResult("manual_review", "outside_suppression_window_duplicate", signal.signal_id)
~~~

最小测试：

~~~python
from datetime import datetime, timedelta


def test_suppresses_known_signal_inside_window():
    now = datetime(2026, 6, 2, 15, 30)
    window = ReopenSuppressionWindow(
        "run_retry_1",
        "stable_1",
        now + timedelta(hours=3),
        {"late_effect:telegram:13124"},
        {"sha256:old"},
    )

    result = triage_late_signal(
        window,
        LateSignal(
            "sig_1",
            "run_retry_1",
            now + timedelta(minutes=10),
            "late_effect:telegram:13124",
            "sha256:old",
            "warning",
            "telegram_probe",
        ),
    )

    assert result.decision == "suppress_duplicate"


def test_reopens_when_payload_changes():
    now = datetime(2026, 6, 2, 15, 30)
    window = ReopenSuppressionWindow(
        "run_retry_1",
        "stable_1",
        now + timedelta(hours=3),
        {"late_effect:telegram:13124"},
        {"sha256:old"},
    )

    result = triage_late_signal(
        window,
        LateSignal(
            "sig_2",
            "run_retry_1",
            now + timedelta(minutes=10),
            "late_effect:telegram:13124",
            "sha256:new",
            "warning",
            "telegram_probe",
        ),
    )

    assert result.decision == "reopen_replay"
    assert result.reason == "payload_hash_changed"
~~~

这段逻辑的重点不是复杂，而是明确：**重复旧信号被抑制，新 key、新 payload、critical severity 才能重开。**

---

## 4. pi-mono：生产版 Suppression Worker

生产版不要让每个监控 worker 自己判断“要不要重开”，而是统一走 Suppression Worker。

~~~ts
// packages/agent-runtime/src/replay/PostStableSuppressionWorker.ts
type LateSignal = {
  id: string;
  retryRunId: string;
  observedAt: string;
  dedupeKey: string;
  payloadHash: string;
  severity: "info" | "warning" | "critical";
  evidenceRef: string;
};

type SuppressionWindow = {
  retryRunId: string;
  stableCloseReceiptId: string;
  suppressionUntil: string;
  dedupeKeys: string[];
  payloadHashes: string[];
};

export async function handleLateSignal(signal: LateSignal, tx: Tx) {
  const window = await tx.suppressionWindows.findActive(signal.retryRunId);

  if (!window) {
    return tx.reopenCases.create({
      retryRunId: signal.retryRunId,
      reason: "no_active_suppression_window",
      sourceSignalId: signal.id,
      evidenceRef: signal.evidenceRef,
    });
  }

  const duplicate =
    new Date(signal.observedAt) <= new Date(window.suppressionUntil) &&
    window.dedupeKeys.includes(signal.dedupeKey) &&
    window.payloadHashes.includes(signal.payloadHash) &&
    signal.severity !== "critical";

  if (duplicate) {
    return tx.suppressionReceipts.create({
      signalId: signal.id,
      retryRunId: signal.retryRunId,
      stableCloseReceiptId: window.stableCloseReceiptId,
      decision: "suppress_duplicate",
      dedupeKey: signal.dedupeKey,
      payloadHash: signal.payloadHash,
      evidenceRef: signal.evidenceRef,
    });
  }

  return tx.reopenCases.create({
    retryRunId: signal.retryRunId,
    reason: signal.severity === "critical" ? "critical_signal" : "new_or_changed_signal",
    sourceSignalId: signal.id,
    evidenceRef: signal.evidenceRef,
  });
}
~~~

这里有三个工程细节：

- dedupeKey 要包含 source、effect type、externalRef，不能只用 runId；
- payloadHash 要用规范化 payload 算，不要用原始 JSON 字符串顺序；
- suppressionReceipts.create 和 reopenCases.create 要在同一事务里做，避免并发重复重开。

---

## 5. OpenClaw：课程 cron 的真实映射

这门课的 cron 本身就是一个小型 Agent workflow：

~~~text
send Telegram
  -> write lesson file
  -> update README
  -> update TOOLS
  -> git commit/push
  -> write memory
~~~

如果一次 cron 取消后 retry，再 closeout，再观察稳定，后面仍可能遇到：

- Telegram API 延迟返回旧 messageId；
- git push 的远端状态被另一个 cron 更新；
- memory 里重复记录同一节课；
- TOOLS.md 已讲内容被重复追加；
- 监控看到“message missing”旧结果，又想重发课程。

Post-Stable Suppression 在这里可以这样落地：

~~~json
{
  "dedupeKey": "agent-course:465:telegram:-5115329245",
  "payloadHash": "sha256:lesson-title-and-body",
  "stableCloseReceiptId": "stable:agent-course:465",
  "suppressDuplicateFor": "6h",
  "retain": {
    "messageId": "180d",
    "commitHash": "180d",
    "lessonHash": "180d",
    "rawTelegramResponse": "24h"
  }
}
~~~

这样重复 probe 不会导致重复发群，真正发现 payloadHash 不一致才打开 reopen。

---

## 6. 实战建议

落地时先做四条规则：

~~~text
1. 每个 stable close 自动创建 suppression window。
2. 每个 late signal 必须带 dedupeKey + payloadHash。
3. suppressed 也要写 receipt，不能静默丢弃。
4. raw evidence 短保留，receipt/hash/summary 长保留。
~~~

抑制窗口不是为了掩盖问题，而是为了防止“同一个问题重复制造系统动作”。证据保留也不是为了堆日志，而是为了在真正复发时能回答三个问题：

~~~text
当初为什么可以关闭？
这次是不是同一个信号？
如果不是，它到底变化在哪里？
~~~

---

## 7. 小结

今天这课的核心：

- StableCloseReceipt 之后还要有 ReopenSuppressionWindow；
- 重复旧信号写 SuppressionReceipt，不重复创建 ReopenReplayCase；
- 新 dedupeKey、新 payloadHash、critical severity 才能突破抑制；
- EvidenceRetentionPlan 要把 raw、summary、hash、receipt 分层保留；
- mature Agent 的关闭不是“永远不管”，而是“重复信号不折腾，新证据能重开，旧证据可审计”。

一句话收尾：**成熟 Agent 的 stable close，不是把案件埋掉，而是给它一个有期限的静默窗口和一份最小可审计证据包。**
