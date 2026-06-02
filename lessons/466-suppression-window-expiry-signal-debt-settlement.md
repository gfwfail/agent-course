# 466. Agent 抑制窗口到期后的信号债务结算（Suppression Window Expiry & Signal Debt Settlement）

上一课讲了 **Post-Stable Close Reopen Suppression & Evidence Retention**：StableCloseReceipt 后，重复旧信号不要反复 reopen，要用 ReopenSuppressionWindow、SuppressionReceipt 和 EvidenceRetentionPlan 同时做到去重与可审计。

但抑制窗口也不是一个“忽略按钮”。窗口期内被 suppress 的信号，到了 suppressionUntil 前后必须结算：

- 这些信号是否真的都是旧信号？
- 有没有同一 dedupeKey 重复次数异常升高？
- 有没有因为 payloadHash 没变，但 source、severity、频率发生变化而值得复查？
- 保留计划是否已经把必要证据压缩、归档、清理完？

所以今天补上 **Suppression Window Expiry & Signal Debt Settlement**：

~~~text
ReopenSuppressionWindow
  -> SuppressedSignalLedger
  -> ExpiryReview
  -> SignalDebtSettlement
  -> RetentionCloseReceipt | ReopenReplayCase | ExtendSuppressionWindow
~~~

一句话：**抑制窗口不是把信号扔掉，而是把它们暂时记成 signal debt；到期时要把这些债务逐项结清，证明该归档、该延长、该重开，还是该人工复核。**

---

## 1. 为什么 suppress 后还要结算

SuppressionReceipt 的意思是：

~~~text
这一次晚到信号在当前窗口内被判定为重复旧信号，所以不立即重开。
~~~

它不等于：

~~~text
这类信号以后都不重要。
~~~

生产里常见的坑：

- 同一个 old messageId 每 5 分钟重复上报，单次看都能 suppress，但 24 小时重复 200 次说明 probe 或索引器有 bug；
- payloadHash 一样，但 source 从 telegram_probe 变成 user_report，说明用户可见层也看到了残留影响；
- severity 一直是 warning，所以被压住，窗口到期前突然出现 critical，却被旧 dedupeKey 误伤；
- retention plan 写了 purge_after_24h，但 suppression ledger 还没做 expiry review，原始证据先被清了；
- close 后没人汇总 suppressed 信号，后续审计只知道“没重开”，不知道到底压了多少、为什么安全。

所以成熟 Agent 需要一个很朴素的规则：

~~~text
每张 SuppressionReceipt 都要进入 SuppressedSignalLedger。
每个窗口到期都要跑 ExpiryReview。
每次 review 都要写 SignalDebtSettlement。
~~~

这很像支付系统里的对账：重复 webhook 不重复扣款，但月底仍要对账重复通知量、失败通知、来源变化和审计保留。

---

## 2. 最小数据模型

窗口期内的每个抑制信号都进 ledger：

~~~json
{
  "kind": "SuppressedSignalLedgerEntry",
  "signalId": "sig_466_001",
  "stableCloseReceiptId": "stable_465",
  "suppressionWindowId": "supwin_465",
  "dedupeKey": "late_effect:telegram:old-run:13124",
  "payloadHash": "sha256:old_payload",
  "source": "telegram_probe",
  "severity": "warning",
  "observedAt": "2026-06-02T16:40:00+11:00",
  "suppressionReceiptId": "suppress_001"
}
~~~

到期 review 时生成窗口摘要：

~~~json
{
  "kind": "ExpiryReview",
  "suppressionWindowId": "supwin_465",
  "reviewedAt": "2026-06-02T18:30:00+11:00",
  "entryCount": 17,
  "uniqueDedupeKeys": 2,
  "uniquePayloadHashes": 1,
  "sources": ["telegram_probe", "indexer_replay"],
  "maxSeverity": "warning",
  "evidenceReadyForRetentionClose": true
}
~~~

最后输出结算：

~~~json
{
  "kind": "SignalDebtSettlement",
  "suppressionWindowId": "supwin_465",
  "decision": "close_retention",
  "reason": "all_suppressed_signals_match_stable_close_scope",
  "closedSuppressionReceiptIds": ["suppress_001", "suppress_002"],
  "retentionCloseReceiptId": "retention_close_466"
}
~~~

如果出现异常，不要悄悄继续 suppress：

~~~json
{
  "kind": "SignalDebtSettlement",
  "suppressionWindowId": "supwin_465",
  "decision": "reopen_replay",
  "reason": "source_changed_to_user_report",
  "reopenReplayCaseId": "reopen_466"
}
~~~

核心点：**suppression 是临时状态，settlement 才是关闭状态。**

---

## 3. learn-claude-code：教学版债务结算纯函数

在 learn-claude-code 里，可以先把它做成纯函数，不碰数据库，不碰网络，只判断窗口到期时该怎么处理。

~~~python
# learn_claude_code/runtime/suppression_debt_settlement.py
from dataclasses import dataclass
from datetime import datetime
from typing import Literal

Severity = Literal["info", "warning", "critical"]
Decision = Literal[
    "close_retention",
    "extend_suppression",
    "reopen_replay",
    "manual_review",
]


@dataclass(frozen=True)
class SuppressionWindow:
    window_id: str
    stable_close_receipt_id: str
    expires_at: datetime
    allowed_dedupe_keys: set[str]
    allowed_payload_hashes: set[str]
    max_duplicate_count: int


@dataclass(frozen=True)
class SuppressedSignal:
    signal_id: str
    observed_at: datetime
    dedupe_key: str
    payload_hash: str
    source: str
    severity: Severity


@dataclass(frozen=True)
class Settlement:
    decision: Decision
    reason: str
    entry_count: int
    receipt_id: str


def settle_suppressed_signals(
    window: SuppressionWindow,
    signals: list[SuppressedSignal],
    now: datetime,
) -> Settlement:
    if now < window.expires_at:
        return Settlement(
            "extend_suppression",
            "window_not_expired",
            len(signals),
            f"settlement:{window.window_id}",
        )

    if not signals:
        return Settlement(
            "close_retention",
            "no_suppressed_signals",
            0,
            f"retention-close:{window.window_id}",
        )

    for signal in signals:
        if signal.severity == "critical":
            return Settlement("reopen_replay", "critical_signal_suppressed", len(signals), signal.signal_id)
        if signal.dedupe_key not in window.allowed_dedupe_keys:
            return Settlement("reopen_replay", "new_dedupe_key_in_ledger", len(signals), signal.signal_id)
        if signal.payload_hash not in window.allowed_payload_hashes:
            return Settlement("reopen_replay", "payload_hash_changed_in_ledger", len(signals), signal.signal_id)
        if signal.source == "user_report":
            return Settlement("manual_review", "user_visible_signal_was_suppressed", len(signals), signal.signal_id)

    counts: dict[str, int] = {}
    for signal in signals:
        counts[signal.dedupe_key] = counts.get(signal.dedupe_key, 0) + 1

    if max(counts.values()) > window.max_duplicate_count:
        return Settlement(
            "manual_review",
            "duplicate_volume_exceeded_budget",
            len(signals),
            f"settlement:{window.window_id}",
        )

    return Settlement(
        "close_retention",
        "all_suppressed_signals_match_stable_close_scope",
        len(signals),
        f"retention-close:{window.window_id}",
    )
~~~

最小测试：

~~~python
from datetime import datetime, timedelta


def test_closes_when_all_signals_match_scope():
    now = datetime(2026, 6, 2, 18, 30)
    window = SuppressionWindow(
        "supwin_1",
        "stable_1",
        now - timedelta(minutes=1),
        {"late_effect:telegram:13124"},
        {"sha256:old"},
        max_duplicate_count=10,
    )

    result = settle_suppressed_signals(
        window,
        [
            SuppressedSignal(
                "sig_1",
                now - timedelta(minutes=30),
                "late_effect:telegram:13124",
                "sha256:old",
                "telegram_probe",
                "warning",
            )
        ],
        now,
    )

    assert result.decision == "close_retention"


def test_reopens_when_payload_changed_signal_was_suppressed():
    now = datetime(2026, 6, 2, 18, 30)
    window = SuppressionWindow(
        "supwin_1",
        "stable_1",
        now - timedelta(minutes=1),
        {"late_effect:telegram:13124"},
        {"sha256:old"},
        max_duplicate_count=10,
    )

    result = settle_suppressed_signals(
        window,
        [
            SuppressedSignal(
                "sig_2",
                now - timedelta(minutes=5),
                "late_effect:telegram:13124",
                "sha256:new",
                "telegram_probe",
                "warning",
            )
        ],
        now,
    )

    assert result.decision == "reopen_replay"
    assert result.reason == "payload_hash_changed_in_ledger"
~~~

教学版要强调：这里不是为了复杂，而是为了避免“每次 suppress 都合理，但整体看已经不合理”的局部正确。

---

## 4. pi-mono：用事件流做到期结算 Worker

pi-mono 里已经有 EventStream 思路：事件被 push，消费者异步迭代，done/error 决定最终结果。这个模式很适合 suppression ledger：

~~~ts
type SuppressionEvent =
  | { type: "suppressed_signal"; windowId: string; signalId: string; dedupeKey: string; payloadHash: string; source: string; severity: "info" | "warning" | "critical" }
  | { type: "window_expired"; windowId: string; expiredAt: string }
  | { type: "settlement_done"; windowId: string; settlementId: string };
~~~

生产实现不要让 UI 或 agent loop 直接扫全表；应该由 worker 认领到期窗口：

~~~ts
type SettlementDecision =
  | "close_retention"
  | "extend_suppression"
  | "reopen_replay"
  | "manual_review";

async function settleSuppressionWindow(windowId: string, now: Date) {
  const lease = await db.suppressionWindowLease.claim(windowId, {
    owner: "suppression-debt-worker",
    ttlMs: 60_000,
  });

  if (!lease.acquired) return;

  const window = await db.reopenSuppressionWindow.get(windowId);
  const entries = await db.suppressedSignalLedger.listByWindow(windowId);

  const settlement = decideSignalDebtSettlement(window, entries, now);

  await db.transaction(async (tx) => {
    await tx.signalDebtSettlement.insert({
      windowId,
      decision: settlement.decision,
      reason: settlement.reason,
      entryCount: entries.length,
      evidenceHash: hashEntries(entries),
      fenceToken: lease.fenceToken,
    });

    if (settlement.decision === "close_retention") {
      await tx.retentionCloseReceipt.insert({
        windowId,
        stableCloseReceiptId: window.stableCloseReceiptId,
        closedAt: now.toISOString(),
      });
    }

    if (settlement.decision === "reopen_replay") {
      await tx.reopenReplayCase.insert({
        windowId,
        reason: settlement.reason,
        source: "suppression_debt_settlement",
      });
    }
  });
}
~~~

几个实现细节：

- lease 必须带 fenceToken，避免旧 worker 超时后又写回；
- settlement 和 retention close / reopen case 要在同一个事务里写；
- ledger entry 不要在 review 前删除，只能先 compact，再由 RetentionCloseReceipt 决定 raw 证据清理；
- 如果 event stream 迟到，窗口已 close，也要写 ignored_after_settlement receipt，而不是静默丢弃。

这和 pi-mono 里的 AgentSession 事件处理类似：message_end、agent_end、auto_compaction 都不是孤立回调，而是要按顺序进入持久化、retry、compaction 等后续流程。

---

## 5. OpenClaw 实战：Cron 课程本身就是例子

这套课程 cron 每 3 小时会做外部副作用：

~~~text
写 lesson 文件
更新 README
更新 TOOLS
发 Telegram
git commit
git push
写 memory
~~~

如果某次 run 取消或重试，前几课已经建立了 replay close、stable close、suppression window。第 466 课对应的现实检查就是：

~~~text
1. 查询 suppression window 内被压住的重复信号：
   - Telegram messageId 是否重复上报？
   - git commit hash 是否被重复 probe？
   - README / TOOLS 是否被旧索引重复扫描？

2. 到期前做 settlement：
   - 全部匹配 stable close scope -> 写 RetentionCloseReceipt
   - messageId 不变但内容 hash 变了 -> reopen replay
   - source 从 probe 变成 user report -> manual_review
   - 重复量超过预算 -> manual_review 或 extend_suppression

3. 结算后再清理：
   - raw probe log 可以按 retention plan 降密或清理
   - stable close / settlement / retention receipt 保留
   - README / TOOLS / commit / Telegram messageId 的对账摘要保留
~~~

注意：OpenClaw 的 cron 不能只看“这次 final reply 成功了”。它还要能回答：

~~~text
这次有没有发群？
发的是哪条 messageId？
commit 是否已经在远端 main？
TOOLS 已讲内容是否包含本课？
被 suppress 的旧信号是否到期结清？
~~~

最后一个问题就是本课的新增点。

---

## 6. 实战清单

设计 Suppression Window Expiry 时，最少要有这些字段：

~~~text
SuppressionWindow:
  windowId
  stableCloseReceiptId
  expiresAt
  allowedDedupeKeys
  allowedPayloadHashes
  maxDuplicateCount

SuppressedSignalLedgerEntry:
  signalId
  windowId
  dedupeKey
  payloadHash
  source
  severity
  observedAt
  suppressionReceiptId

SignalDebtSettlement:
  settlementId
  windowId
  decision
  reason
  entryCount
  evidenceHash
  nextActionRef
~~~

决策表可以先从四类开始：

~~~text
close_retention:
  全部信号仍匹配 stable close scope，重复量在预算内，证据已可压缩归档。

extend_suppression:
  窗口未到期，或重复仍在收敛但需要更多时间观察。

reopen_replay:
  出现新 dedupeKey、新 payloadHash、critical severity，或外部现实和 close receipt 冲突。

manual_review:
  用户可见来源、重复量异常、证据缺口、窗口/ledger 不一致。
~~~

不要让 suppress 变成黑洞。每一次 suppress 都要能在到期时被点名结算。

---

## 7. 一句话总结

**成熟 Agent 的重开抑制不是“别再提醒我”，而是把重复信号临时记账；窗口到期后用 SignalDebtSettlement 证明这些信号已经归档、延长、重开或交给人工，债务清零后才算真正稳定关闭。**
