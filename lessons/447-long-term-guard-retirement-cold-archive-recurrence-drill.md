# 447. Agent 长期护栏退役后的冷归档与复发演练（Long-term Guard Retirement Cold Archive & Recurrence Drill）

> 关键词：retirement archive、cold evidence、retrieval drill、recurrence drill、guard retirement

上一课讲 Sunset Review 到期时，LongTermGuardContract 不能靠 owner 感觉续命；要生成 Evidence Packet，再用 Renewal / Downgrade / Repair / Retirement Receipt 改变热路径。

今天继续讲退役后的最后一公里：**护栏 retired 以后，证据、回归种子、复发规则应该怎么保存和演练？**

很多系统把 retired 当删除：

- 从 policy pack 移除规则；
- 从 prompt guard 列表删掉说明；
- 把测试样本留在某个旧 PR 或日志里；
- 半年后同类风险复发，没人知道当初为什么退役。

成熟 Agent 的退役不是遗忘，而是从热路径治理切换成冷路径可追溯：

> 长期护栏退役后，必须留下 RetirementArchive，并定期演练证据可取回、回归种子可重放、复发信号可重开。

---

## 1. 为什么 retired 之后还要管

护栏退役通常意味着风险已经被替代机制覆盖，或者风险面被产品改造消除。

但这三个东西会漂移：

1. 替代 guard 可能被后续版本收窄；
2. 产品逻辑可能重新打开旧入口；
3. 模型、工具 schema、权限策略可能变化，让旧风险换一种形式回来。

所以 RetirementReceipt 只能证明“当时可以退役”，不能证明“未来永远安全”。

退役后至少要保留四类对象：

- **RetirementArchive**：最后证据包、决策收据、退役原因、替代机制；
- **RegressionSeedSet**：最小可重放样本，证明旧风险是什么；
- **RetrievalDrillLease**：定期验证冷证据还能按权限取回和验 hash；
- **RecurrenceDrillPlan**：定期把旧风险种子投到新 runtime 上做影子演练。

这让 retired guard 不再占热路径成本，但仍能在风险复发时提供工程入口。

---

## 2. 状态机：retired 不是 terminal forever

建议把长期护栏退役后状态继续建模：

~~~text
retired
  -> archived
  -> retrieval_drill_due
  -> retrieval_drill_failed
  -> recurrence_drill_due
  -> recurrence_detected
  -> reopen_case
  -> archived_clean
~~~

几个边界要明确：

- retired 表示热路径已移除，不表示证据可以丢；
- archived 表示冷归档写入且 hash / schema / retention 校验通过；
- retrieval_drill_failed 不是小告警，可能意味着未来无法审计；
- recurrence_detected 必须生成 reopen case，而不是直接把旧 guard 重新塞回 active；
- archived_clean 表示本轮演练干净，不等于取消后续演练。

换句话说，退役是热路径终点，不是治理生命周期终点。

---

## 3. learn-claude-code：退役归档纯函数

教学版先写一个纯函数：输入 RetirementReceipt、冷存储观测、复发演练结果，输出下一步动作。

~~~python
# learn_claude_code/guard_patterns/retirement_archive.py
from dataclasses import dataclass
from typing import Literal

Action = Literal[
    "archive_clean",
    "repair_archive",
    "run_recurrence_drill",
    "open_reopen_case",
    "manual_review",
]


@dataclass(frozen=True)
class RetirementArchive:
    archive_id: str
    guard_id: str
    status: str
    retirement_receipt_hash: str
    evidence_packet_hash: str
    regression_seed_count: int
    replacement_guard_id: str | None
    retained_until_epoch: int
    next_retrieval_drill_epoch: int
    next_recurrence_drill_epoch: int


@dataclass(frozen=True)
class ArchiveObservation:
    now_epoch: int
    receipt_hash_verified: bool
    evidence_hash_verified: bool
    schema_readable: bool
    access_policy_valid: bool
    replacement_guard_active: bool
    dependency_fingerprint_changed: bool
    regression_seeds_replayable: int
    recurrence_hits: int
    unknown_replay_results: int


@dataclass(frozen=True)
class ArchiveDecision:
    action: Action
    reason: str
    followups: tuple[str, ...] = ()


def decide_retirement_archive(
    archive: RetirementArchive,
    observation: ArchiveObservation,
) -> ArchiveDecision:
    if archive.status not in {"archived", "archived_clean"}:
        return ArchiveDecision("manual_review", "archive_not_ready")

    if observation.now_epoch > archive.retained_until_epoch:
        return ArchiveDecision(
            "manual_review",
            "retention_expired_needs_disposal_or_extension_decision",
            ("owner_retention_review",),
        )

    archive_broken = not all(
        [
            observation.receipt_hash_verified,
            observation.evidence_hash_verified,
            observation.schema_readable,
            observation.access_policy_valid,
        ]
    )
    if archive_broken:
        return ArchiveDecision(
            "repair_archive",
            "cold_archive_integrity_failed",
            ("repair_hash_or_schema", "verify_access_policy"),
        )

    if archive.regression_seed_count == 0:
        return ArchiveDecision("manual_review", "missing_regression_seeds")

    replay_coverage = observation.regression_seeds_replayable / archive.regression_seed_count
    if replay_coverage < 0.9 or observation.unknown_replay_results > 0:
        return ArchiveDecision(
            "repair_archive",
            "regression_seed_replay_coverage_too_low",
            ("refresh_seed_harness",),
        )

    recurrence_risk = (
        observation.recurrence_hits > 0
        or not observation.replacement_guard_active
        or observation.dependency_fingerprint_changed
    )
    if recurrence_risk and observation.now_epoch >= archive.next_recurrence_drill_epoch:
        if observation.recurrence_hits > 0:
            return ArchiveDecision(
                "open_reopen_case",
                "retired_guard_recurrence_detected",
                ("create_reopen_case_from_archive",),
            )
        return ArchiveDecision(
            "run_recurrence_drill",
            "replacement_or_dependency_changed",
            ("shadow_replay_retired_guard_seeds",),
        )

    return ArchiveDecision("archive_clean", "archive_integrity_and_recurrence_checks_clean")
~~~

这里有两个关键点：

- 冷证据坏了，先 repair archive，不要等事故时才发现证据打不开；
- 复发命中后创建 reopen case，不要把 retired guard 原样复活，因为依赖和 scope 可能已经变化。

---

## 4. pi-mono：RetirementArchiveWorker

生产里可以把退役归档做成后台 worker。它不在主请求链路阻塞用户，只消费 RetirementReceipt 和定期 drill event。

~~~ts
type RetirementArchiveRecord = {
  archiveId: string;
  guardId: string;
  receiptHash: string;
  evidenceHash: string;
  seedRefs: string[];
  replacementGuardId?: string;
  retainedUntil: number;
  nextRetrievalDrillAt: number;
  nextRecurrenceDrillAt: number;
};

type ArchiveDecision =
  | { action: "archive_clean"; reason: string }
  | { action: "repair_archive"; reason: string; ticketKey: string }
  | { action: "run_recurrence_drill"; reason: string; drillKey: string }
  | { action: "open_reopen_case"; reason: string; reopenKey: string }
  | { action: "manual_review"; reason: string; caseKey: string };

async function handleRetirementArchive(
  archive: RetirementArchiveRecord,
  now: number,
): Promise<ArchiveDecision> {
  const observation = await observeArchive(archive);

  if (!observation.receiptHashOk || !observation.evidenceHashOk || !observation.schemaOk) {
    return {
      action: "repair_archive",
      reason: "archive_integrity_failed",
      ticketKey: `archive-repair:${archive.archiveId}:${archive.receiptHash}`,
    };
  }

  const replay = await replayRegressionSeeds({
    seedRefs: archive.seedRefs,
    mode: "shadow",
  });

  if (replay.recurrenceHits.length > 0) {
    return {
      action: "open_reopen_case",
      reason: "retired_guard_seed_recurred",
      reopenKey: `reopen:${archive.guardId}:${replay.runHash}`,
    };
  }

  const replacementChanged = await replacementGuardChanged(archive.replacementGuardId);
  if (replacementChanged && now >= archive.nextRecurrenceDrillAt) {
    return {
      action: "run_recurrence_drill",
      reason: "replacement_guard_changed",
      drillKey: `recurrence-drill:${archive.guardId}:${now}`,
    };
  }

  return { action: "archive_clean", reason: "cold_archive_verified" };
}
~~~

这个 worker 的输出不要只写日志。建议进入 outbox：

- repair_archive -> ArchiveRepairTicket；
- run_recurrence_drill -> RecurrenceDrillJob；
- open_reopen_case -> ReopenCase；
- manual_review -> OwnerReviewCase；
- archive_clean -> ArchiveDrillReceipt。

这样退役后的治理仍然是事件驱动、可重试、可审计的。

---

## 5. OpenClaw 课程 Cron 的落地例子

拿这门课程 cron 举例。

假设未来我们把“手工维护 README 目录”这个护栏退役，因为 lesson index generator 已经稳定上线。

退役时不能只删掉检查步骤，还要归档：

~~~json
{
  "archiveId": "archive_course_readme_manual_gate_2026_06",
  "retiredGuard": "course_readme_manual_directory_gate",
  "retirementReason": "automatic_index_generator_active",
  "replacementGuard": "lesson_index_generator_output_check",
  "regressionSeeds": [
    "missing_lesson_link",
    "wrong_lesson_number",
    "duplicate_topic_entry"
  ],
  "nextRetrievalDrillAt": "2026-07-01T00:30:00+10:00",
  "nextRecurrenceDrillAt": "2026-07-15T00:30:00+10:00"
}
~~~

后续 heartbeat 或 cron 做两件事：

1. 取回 archive，验 hash，确认当初退役的证据还读得出来；
2. 用 regression seeds 对自动目录生成器做 shadow replay，确认不会漏 lesson、错编号、重复主题。

如果复发，比如自动生成器漏了第 447 课，就创建 reopen case：不是直接恢复旧手工流程，而是带证据修自动生成器，必要时临时加回手工 gate。

---

## 6. 常见坑

第一，把退役当删除。

删除节省了热路径成本，但也删除了未来复盘入口。正确做法是热路径移除，冷路径归档。

第二，只归档 receipt，不归档 regression seeds。

receipt 说明为什么退役，seed 才能证明旧风险能不能在新 runtime 复现。两者缺一不可。

第三，冷证据从不演练。

没有 retrieval drill 的冷存储只是心理安慰。等事故发生时才发现权限过期、schema 变了、hash 对不上，已经晚了。

第四，复发后原样复活旧 guard。

旧 guard 的 scope、依赖、误伤预算都可能过期。复发应该进入 reopen workflow，重新评估再发布。

---

## 7. 这节课的核心

长期护栏退役不是“这条规则没用了，删掉”。它是把治理责任从 active enforcement 转成 cold evidence + recurrence readiness。

一个成熟的退役闭环应该回答四个问题：

- 当初为什么退役？
- 退役证据现在还能不能取回和验证？
- 旧风险种子还能不能在新 runtime 上重放？
- 如果复发，系统会开哪个 case、走哪条恢复路径？

RetirementArchive 的价值不是让系统永远背着旧包袱，而是让它在降低热路径复杂度的同时，仍然保留复盘、证明和重开的能力。
