# 446. Agent 长期护栏 Sunset Review 证据包与退役收据（Long-term Guard Sunset Review Evidence Packet & Retirement Receipt）

> 关键词：sunset review、evidence packet、retirement receipt、scope downgrade、guard lifecycle

上一课讲 PostReleaseMonitoringLease 决定 supersede_by_long_term_guard 后，要用 LongTermGuardContract 把残余约束接管成长周期护栏，并绑定 owner、scope、metrics、fallback 和 sunset review。

今天继续补后半段：**到了 sunset review 时间，系统到底怎么判断这条长期护栏该续期、降级、修复，还是退役？**

很多团队会把 reviewAfter 当提醒：

- 到期发一个 Slack/Telegram 提醒；
- owner 看一眼觉得“先留着吧”；
- 护栏继续 active；
- 过几个月再提醒一次；
- 最后长期护栏变成永久复杂度。

成熟 Agent 不能靠感觉续命。Sunset Review 必须生成结构化 Evidence Packet，再把决策写成不可变 receipt：

> 长期护栏到期不是自动续期，而是一次证据复核；没有证据证明它还值得 active，就应该降级、修复、转人工复核或退役。

---

## 1. 为什么 Sunset Review 要有证据包

LongTermGuardContract 的问题不是“有没有保护”，而是“今天这条保护还是否合理”。

它可能出现四种变化：

1. 风险仍存在，护栏有效，应该续期；
2. 风险存在但误伤升高，应该修复或收窄 scope；
3. 风险已经被产品/平台能力消除，应该退役；
4. 证据不足或 owner 缺失，应该进入 manual review，不应继续静默 active。

所以 review 输入不能只有触发次数。证据包至少要覆盖：

- **risk evidence**：真实高危样本是否仍存在；
- **effectiveness**：prevented failure、false allow、false block；
- **scope fitness**：是否仍只影响合同声明的 tenant/channel/tool/actionClass；
- **dependency freshness**：模型、工具 schema、policy pack 是否漂移；
- **replacement status**：是否已有替代 guard、产品修复或平台能力；
- **owner health**：owner 是否仍有效，是否响应过上次 followup；
- **cost**：延迟、人工复核量、误伤补偿成本。

没有 Evidence Packet，Sunset Review 就会退化成“我觉得先别删”。

---

## 2. 状态机：review_due 不是续期

建议给 LongTermGuardContract 加生命周期状态：

~~~text
active
  -> review_due
  -> review_in_progress
  -> renewed
  -> downgraded
  -> repair_required
  -> retirement_ready
  -> retired
  -> manual_review
~~~

几个边界要分清：

- review_due 只是到期，不代表继续 active 合理；
- renewed 必须绑定 RenewalReceipt；
- downgraded 要写 ScopeDowngradeReceipt，说明哪些 scope 从 block 变 warn / shadow；
- repair_required 要生成 RepairCase，不能靠 owner 手工记；
- retirement_ready 还不是 retired，退役前要跑 dry-run / shadow removal；
- retired 必须写 RetirementReceipt，保留 lineage 和最后一份证据包。

这能防止两个常见事故：

1. 到期自动续期，旧护栏永久污染热路径；
2. 直接删除护栏，结果同类风险又回来了，没有 tombstone / regression seed 可查。

---

## 3. learn-claude-code：Sunset Review 纯函数

教学版先写纯函数。输入合同和证据包，输出 review 决策。

~~~python
# learn_claude_code/guard_patterns/long_term_guard_sunset_review.py
from dataclasses import dataclass
from typing import Literal

Decision = Literal[
    "renew_guard",
    "downgrade_scope",
    "open_repair_case",
    "retire_guard",
    "manual_review",
]


@dataclass(frozen=True)
class LongTermGuardContract:
    guard_id: str
    status: str
    owner: str | None
    scope: tuple[str, ...]
    max_renewals: int
    renewal_count: int
    max_false_block_rate: float
    max_false_allow_rate: float
    review_after_epoch: int
    dependency_fingerprint: str


@dataclass(frozen=True)
class SunsetEvidencePacket:
    now_epoch: int
    high_risk_hits: int
    prevented_failures: int
    false_blocks: int
    false_allows: int
    total_evaluations: int
    scope_violations: int
    replacement_guard_active: bool
    risk_surface_removed: bool
    owner_acknowledged: bool
    current_dependency_fingerprint: str
    dependency_revalidated: bool
    removal_shadow_clean_samples: int


@dataclass(frozen=True)
class SunsetReviewDecision:
    decision: Decision
    reason: str
    next_review_days: int = 0
    scope_to_downgrade: tuple[str, ...] = ()
    followups: tuple[str, ...] = ()


def decide_long_term_guard_sunset(
    contract: LongTermGuardContract,
    evidence: SunsetEvidencePacket,
) -> SunsetReviewDecision:
    if contract.status != "active":
        return SunsetReviewDecision("manual_review", "guard_not_active")

    if evidence.now_epoch < contract.review_after_epoch:
        return SunsetReviewDecision("renew_guard", "review_not_due", next_review_days=7)

    if not contract.owner or not evidence.owner_acknowledged:
        return SunsetReviewDecision(
            "manual_review",
            "owner_missing_or_unacknowledged",
            followups=("assign_owner",),
        )

    if evidence.scope_violations > 0:
        return SunsetReviewDecision(
            "downgrade_scope",
            "scope_violation_detected",
            scope_to_downgrade=contract.scope,
            followups=("scope_diff_review",),
        )

    dependency_drifted = (
        evidence.current_dependency_fingerprint != contract.dependency_fingerprint
    )
    if dependency_drifted and not evidence.dependency_revalidated:
        return SunsetReviewDecision(
            "manual_review",
            "dependency_drift_needs_revalidation",
            followups=("dependency_probe",),
        )

    total = max(evidence.total_evaluations, 1)
    false_block_rate = evidence.false_blocks / total
    false_allow_rate = evidence.false_allows / total

    if false_allow_rate > contract.max_false_allow_rate:
        return SunsetReviewDecision("open_repair_case", "false_allow_budget_exceeded")

    if false_block_rate > contract.max_false_block_rate:
        return SunsetReviewDecision(
            "downgrade_scope",
            "false_block_budget_exceeded",
            scope_to_downgrade=contract.scope,
            followups=("false_block_examples",),
        )

    if evidence.risk_surface_removed and evidence.replacement_guard_active:
        if evidence.removal_shadow_clean_samples >= 20:
            return SunsetReviewDecision("retire_guard", "risk_removed_and_shadow_removal_clean")
        return SunsetReviewDecision(
            "manual_review",
            "needs_shadow_removal_samples",
            followups=("run_shadow_removal",),
        )

    if evidence.high_risk_hits > 0 or evidence.prevented_failures > 0:
        return SunsetReviewDecision("renew_guard", "guard_still_prevents_real_risk", next_review_days=30)

    if contract.renewal_count >= contract.max_renewals:
        return SunsetReviewDecision(
            "manual_review",
            "max_renewals_without_recent_value",
            followups=("justify_or_retire",),
        )

    return SunsetReviewDecision("downgrade_scope", "no_recent_value_reduce_before_retire")
~~~

这里的重点：退役不是因为“最近没触发”，而是 risk removed、replacement active、shadow removal clean 三个条件同时满足。

---

## 4. Evidence Packet 长什么样

生产里建议把 evidence packet 做成可审计对象，而不是临时 query 结果：

~~~json
{
  "packetId": "sunset_packet_ltg_course_dedupe_2026_06",
  "guardId": "ltg.course.topic_dedupe",
  "window": {
    "from": "2026-05-01T00:00:00Z",
    "to": "2026-05-31T00:00:00Z"
  },
  "effectiveness": {
    "evaluations": 248,
    "preventedFailures": 3,
    "falseBlocks": 0,
    "falseAllows": 0,
    "unknownOutcomes": 1
  },
  "scope": {
    "declared": ["course_cron", "telegram_publish", "git_push"],
    "observedOutsideScope": []
  },
  "dependency": {
    "expectedFingerprint": "policy_pack@445+tools_schema@2026-05",
    "currentFingerprint": "policy_pack@446+tools_schema@2026-05",
    "revalidated": true
  },
  "replacement": {
    "replacementGuardActive": false,
    "riskSurfaceRemoved": false,
    "removalShadowCleanSamples": 0
  },
  "owner": {
    "team": "course_cron",
    "acknowledgedAt": "2026-05-31T20:30:00Z"
  }
}
~~~

这个对象有两个用途：

1. 给 review gate 做机器判断；
2. 给未来的人解释“为什么当时续期 / 降级 / 退役”。

---

## 5. pi-mono：Sunset Review Worker

生产版可以按 due guard 扫描：

~~~ts
// packages/runtime/src/guard/LongTermGuardSunsetReviewWorker.ts
type SunsetDecision =
  | { type: "renew_guard"; reason: string; nextReviewAt: Date }
  | { type: "downgrade_scope"; reason: string; scopeToDowngrade: string[] }
  | { type: "open_repair_case"; reason: string }
  | { type: "retire_guard"; reason: string }
  | { type: "manual_review"; reason: string; followups: string[] };

export async function runSunsetReview(
  guardId: string,
  deps: {
    store: GuardStore;
    projector: SunsetEvidenceProjector;
    gate: SunsetReviewGate;
    outbox: Outbox;
    now: () => Date;
  },
) {
  await deps.store.transaction(async (tx) => {
    const contract = await tx.lockLongTermGuard(guardId);
    if (contract.status !== "active" || contract.reviewAfter > deps.now()) {
      return;
    }

    const packet = await deps.projector.buildEvidencePacket(tx, contract, deps.now());
    const decision = deps.gate.decide(contract, packet);

    const review = await tx.insertSunsetReview({
      guardId,
      packetId: packet.packetId,
      decision,
      reviewedAt: deps.now(),
    });

    if (decision.type === "renew_guard") {
      await tx.insertRenewalReceipt({
        guardId,
        reviewId: review.reviewId,
        reason: decision.reason,
        nextReviewAt: decision.nextReviewAt,
      });
      await tx.renewLongTermGuard(guardId, decision.nextReviewAt);
      return;
    }

    if (decision.type === "downgrade_scope") {
      const receipt = await tx.insertScopeDowngradeReceipt({
        guardId,
        reviewId: review.reviewId,
        scopeToDowngrade: decision.scopeToDowngrade,
        reason: decision.reason,
      });
      await tx.applyGuardScopeDowngrade(guardId, decision.scopeToDowngrade, receipt.receiptId);
      await deps.outbox.publish(tx, {
        type: "long_term_guard.scope_downgraded",
        guardId,
        receiptId: receipt.receiptId,
      });
      return;
    }

    if (decision.type === "retire_guard") {
      const receipt = await tx.insertRetirementReceipt({
        guardId,
        reviewId: review.reviewId,
        packetId: packet.packetId,
        reason: decision.reason,
      });
      await tx.retireLongTermGuard(guardId, receipt.receiptId);
      await deps.outbox.publish(tx, {
        type: "long_term_guard.retired",
        guardId,
        receiptId: receipt.receiptId,
      });
      return;
    }

    await tx.enqueueManualOrRepairFollowup(guardId, review.reviewId, decision);
  });
}
~~~

注意这里仍然用 lockLongTermGuard。Sunset Review 和实时 guard evaluation 会同时发生，必须避免 review worker 退役时 evaluation worker 还读到半更新状态。

---

## 6. 退役前要做 Shadow Removal

长期护栏不能从 active 直接 delete。推荐先做 Shadow Removal：

~~~text
active block/warn
  -> shadow_removal
  -> compare active decision vs without_guard decision
  -> collect clean samples
  -> retirement_ready
  -> retired
~~~

Shadow Removal 的意思是：

- 生产路径仍按 active 护栏执行；
- 旁路同时计算“如果没有这条护栏会怎样”；
- 如果 without_guard 没有产生 false allow / regression / policy gap；
- 并且替代 guard 或产品修复确实覆盖风险；
- 才允许写 RetirementReceipt。

这样退役不会制造保护空窗。

---

## 7. OpenClaw 课程 Cron 的落地例子

拿 Agent 开发课程 cron 来类比。

假设我们把“选题去重”和“README/TOOLS/Git/Telegram 对账”升级成长期护栏。

一个月后做 Sunset Review：

- Evidence Packet 统计最近 30 天发课次数、阻止的重复选题次数、误拦次数、漏拦次数；
- 检查 README 是否已经改成自动生成；
- 检查 TOOLS 已讲内容是否仍是事实来源，还是已经由 lesson index 替代；
- 如果自动 index generator 已上线，并且 shadow removal 证明去掉手写 README gate 不会漏目录，就退役 README 手工检查；
- 如果 Telegram messageId 对账仍抓到过失败，就续期外部副作用对账护栏。

这就是长期护栏生命周期：保留有价值的，降级没价值但还不确定的，退役已经被更好机制替代的。

---

## 8. 常见坑

第一，只看命中次数。

命中 0 可能是风险消失，也可能是流量不足、scope 写错、上游被别的 guard 挡住了。

第二，false block 没有进入 review。

护栏“没放过坏请求”不等于好。误伤高的护栏会让 Agent 变得胆小、慢、需要更多人工确认。

第三，退役没有收据。

半年后风险复发，没人知道当初为什么删、删之前看过哪些样本、替代机制是什么。

第四，owner 消失还自动续期。

owner 不存在时，这条护栏已经变成无人维护的生产逻辑，应该 manual_review 或降级，而不是继续 active。

---

## 9. 这节课的核心

LongTermGuardContract 的价值不在于“把规则留久一点”，而在于让规则长期存在时仍然可解释、可度量、可退出。

成熟 Agent 的 Sunset Review 应该产出三样东西：

- Evidence Packet：证明当前事实；
- Decision Receipt：证明做了什么判断；
- Retirement / Renewal / Downgrade Receipt：证明热路径如何变化。

长期护栏不是越多越安全。真正安全的是：每条护栏都能证明自己今天还值得存在；证明不了，就进入修复、降级或退役流程。
