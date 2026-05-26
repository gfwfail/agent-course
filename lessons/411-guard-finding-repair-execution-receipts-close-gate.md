# 411. Agent 护栏审计发现修复执行收据与关闭闸门（Guard Finding Repair Execution Receipts & Close Gate）

上一课讲了 **Guard Finding Repair Queue Scheduling & Conflict Budget**：finding 进修复队列后，不能 FIFO 全部开修；要按 riskSurface、affectedGuardIds、writeScope、releaseGate 和 blast radius 分 lane 调度，保证并发修复仍然有清晰因果。

今天继续往后走：**repair batch 被调度、worker 也交了补丁，不代表 finding 可以关闭。**

一句话：**Guard finding repair 必须产出执行收据，并经过 close gate 证明 patch、replay、shadow/canary、outcome backfill 和临时缓解都已经对齐；否则只能 extend_verification、reopen 或 escalate，不能把 ticket 关成“感觉修好了”。**

---

## 1. 为什么修完代码还不能关 finding

上一课的调度器可能排出这样的 batch：

~~~json
{
  "batchId": "repair_batch_20260526_411",
  "lane": "message_egress",
  "findingIds": ["finding_101"],
  "affectedGuardIds": ["egress.shared_context"],
  "writeScope": ["guard-pack/message-egress.yaml", "policy/egress.ts"],
  "requiredGates": {
    "replay": true,
    "shadow": true,
    "canary": true
  }
}
~~~

worker 做完以后，很容易写一句：

~~~text
已修复 egress.shared_context，测试通过。
~~~

这不够。

对护栏 finding 来说，close 需要回答五个问题：

1. 修复补丁是不是对应这个 finding，而不是顺手改了别的东西？
2. 原事故样本和相邻风险面 replay 是否通过？
3. shadow/canary 中真实流量是否没有新误伤和漏挡？
4. 这个 finding 之前造成的 outcome 是否已经 backfill 或补偿？
5. 临时 freeze、sampling boost、evidence downgrade 是否已经解除或转成长期策略？

少一个，finding 都不应该 close。否则队列表面清空了，系统风险还在。

---

## 2. Execution Receipt：修复执行收据

修复执行收据不是日志，它是 close gate 的输入。

最小结构可以这样设计：

~~~ts
type RepairExecutionReceipt = {
  receiptId: string;
  batchId: string;
  findingIds: string[];
  repairCommit: string;
  guardPackVersionBefore: number;
  guardPackVersionAfter: number;
  changedFiles: string[];
  changedGuardIds: string[];
  workerId: string;
  startedAt: string;
  completedAt: string;
  patchSummary: string;
  mitigationState: {
    freezeLifted: boolean;
    samplingBoostRemoved: boolean;
    evidenceModeRestored: boolean;
    remainingMitigations: string[];
  };
  proofRefs: {
    replayReceiptIds: string[];
    shadowReceiptIds: string[];
    canaryReceiptIds: string[];
    outcomeBackfillIds: string[];
  };
};
~~~

几个字段很关键：

- repairCommit：把 finding 和真实代码变更绑定；
- guardPackVersionBefore/After：证明修的是哪个护栏包；
- changedGuardIds：close gate 要确认补丁确实碰到了 affectedGuardIds；
- mitigationState：避免临时措施永远留在生产里；
- proofRefs：后续审计不重新跑一遍，只查收据链。

---

## 3. Close Gate 的判断结果

Close gate 不是布尔值。实战里至少要有 5 种结果：

~~~text
close_verified
  修复证据完整，finding 可以关闭。

extend_verification
  patch 和 replay 通过，但 shadow/canary 样本不足，需要延长观察窗口。

reopen_finding
  修复后仍复现，或者出现相邻风险面回归。

escalate_owner
  需要人工 owner 决策，例如业务策略冲突、false positive 可接受性不明确。

manual_review
  证据不完整、收据缺失、commit 不匹配、mitigation 未收口。
~~~

这比简单的 pass/fail 更实用，因为它能把“还差什么”变成下一步动作。

---

## 4. Close Gate 的最小规则

一个保守的 close gate 可以从这几条规则开始：

~~~text
规则 1：findingIds 必须完全覆盖
  receipt.findingIds 必须包含待关闭 finding，不能用别的 batch 的收据冒充。

规则 2：affectedGuardIds 必须被触达
  changedGuardIds 至少覆盖 finding.affectedGuardIds，除非 closeReason 是 policy_stale 无需改 guard。

规则 3：releaseGate 只看 required proof
  finding 要 replay/shadow/canary，就必须有对应 receipt，不能用单元测试替代。

规则 4：原事故样本必须 stable pass
  replay receipt 里 originalIncidentCase 必须 stable_pass，不接受 flaky_pass。

规则 5：临时缓解必须有终态
  freeze、sampling boost、evidence downgrade 要么解除，要么升级成持久策略并有 policy receipt。

规则 6：outcome mismatch 必须 backfill
  如果 finding.kind 是 outcome_mismatch，必须有 outcomeBackfillIds。
~~~

核心是：close finding 不是“代码合了”，而是“风险闭环了”。

---

## 5. learn-claude-code：纯函数关闭闸门

教学版可以把 close gate 写成纯函数，输入 finding、receipt 和验证结果，输出 close decision。

~~~py
from dataclasses import dataclass
from typing import Literal


FindingKind = Literal[
    "false_allow",
    "false_block",
    "explainability_gap",
    "evidence_overread",
    "policy_stale",
    "outcome_mismatch",
]

CloseDecision = Literal[
    "close_verified",
    "extend_verification",
    "reopen_finding",
    "escalate_owner",
    "manual_review",
]


@dataclass(frozen=True)
class Finding:
    finding_id: str
    kind: FindingKind
    affected_guard_ids: tuple[str, ...]
    requires_replay: bool
    requires_shadow: bool
    requires_canary: bool


@dataclass(frozen=True)
class ReplayReceipt:
    receipt_id: str
    original_incident_status: str
    adjacent_risk_status: str
    flaky_cases: int


@dataclass(frozen=True)
class TrafficReceipt:
    receipt_id: str
    sample_count: int
    false_allows: int
    false_blocks: int
    min_required_samples: int


@dataclass(frozen=True)
class ExecutionReceipt:
    batch_id: str
    finding_ids: tuple[str, ...]
    changed_guard_ids: tuple[str, ...]
    replay_receipts: tuple[ReplayReceipt, ...]
    shadow_receipts: tuple[TrafficReceipt, ...]
    canary_receipts: tuple[TrafficReceipt, ...]
    outcome_backfill_ids: tuple[str, ...]
    remaining_mitigations: tuple[str, ...]


def has_required_guard_change(finding: Finding, receipt: ExecutionReceipt) -> bool:
    if finding.kind == "policy_stale":
        return True
    return set(finding.affected_guard_ids).issubset(set(receipt.changed_guard_ids))


def replay_passed(receipt: ExecutionReceipt) -> bool:
    if not receipt.replay_receipts:
        return False

    return all(
        r.original_incident_status == "stable_pass"
        and r.adjacent_risk_status == "stable_pass"
        and r.flaky_cases == 0
        for r in receipt.replay_receipts
    )


def traffic_healthy(receipts: tuple[TrafficReceipt, ...]) -> str:
    if not receipts:
        return "missing"

    total_samples = sum(r.sample_count for r in receipts)
    required = max(r.min_required_samples for r in receipts)
    false_allows = sum(r.false_allows for r in receipts)
    false_blocks = sum(r.false_blocks for r in receipts)

    if total_samples < required:
        return "needs_more_samples"
    if false_allows > 0:
        return "unsafe"
    if false_blocks > 0:
        return "regression"
    return "healthy"


def decide_close(finding: Finding, receipt: ExecutionReceipt) -> CloseDecision:
    if finding.finding_id not in receipt.finding_ids:
        return "manual_review"

    if not has_required_guard_change(finding, receipt):
        return "manual_review"

    if finding.requires_replay and not replay_passed(receipt):
        return "reopen_finding"

    if finding.requires_shadow:
        shadow = traffic_healthy(receipt.shadow_receipts)
        if shadow == "needs_more_samples":
            return "extend_verification"
        if shadow in {"unsafe", "regression", "missing"}:
            return "reopen_finding"

    if finding.requires_canary:
        canary = traffic_healthy(receipt.canary_receipts)
        if canary == "needs_more_samples":
            return "extend_verification"
        if canary in {"unsafe", "regression", "missing"}:
            return "reopen_finding"

    if finding.kind == "outcome_mismatch" and not receipt.outcome_backfill_ids:
        return "manual_review"

    if receipt.remaining_mitigations:
        return "escalate_owner"

    return "close_verified"
~~~

这段代码故意保守：缺收据不是通过，样本不足不是通过，仍有临时缓解也不是通过。

---

## 6. pi-mono：把 close gate 接到 EventBus

生产实现不要让 worker 自己 close finding。worker 只提交 execution receipt，close gate 消费事件后决定状态。

~~~ts
type GuardFindingCloseDecisionEvent = {
  type: "guard.finding.close_decision";
  findingId: string;
  batchId: string;
  decision:
    | "close_verified"
    | "extend_verification"
    | "reopen_finding"
    | "escalate_owner"
    | "manual_review";
  reasons: string[];
  requiredNextActions: string[];
  proofRefs: string[];
};

class GuardFindingCloseGate {
  constructor(
    private readonly findingStore: FindingStore,
    private readonly receiptStore: ReceiptStore,
    private readonly bus: EventBus,
  ) {}

  async evaluate(findingId: string, executionReceiptId: string) {
    const finding = await this.findingStore.get(findingId);
    const receipt = await this.receiptStore.getExecutionReceipt(executionReceiptId);

    const result = closeDecision(finding, receipt);

    const event: GuardFindingCloseDecisionEvent = {
      type: "guard.finding.close_decision",
      findingId,
      batchId: receipt.batchId,
      decision: result.decision,
      reasons: result.reasons,
      requiredNextActions: result.nextActions,
      proofRefs: result.proofRefs,
    };

    await this.bus.publish(event);

    if (result.decision === "close_verified") {
      await this.findingStore.close(findingId, {
        closedBy: "close_gate",
        closeReceiptId: executionReceiptId,
        proofRefs: result.proofRefs,
      });
    }

    return event;
  }
}
~~~

这里的边界很重要：

- worker 不直接 close；
- close gate 不重新猜测 worker 意图，只看收据；
- EventBus 里记录每次为什么 close、为什么延长、为什么 reopen；
- 后续 rollback 或审计可以从 findingId 追到 batchId、commit、replay、shadow、canary。

---

## 7. OpenClaw 场景：课程 Cron 的真实副作用

拿这门课自己的 cron 举例：

~~~text
发现问题：
  audit finding 发现课程消息缺少 secret scan proof。

修复 batch：
  修改 lesson generator，发送前生成 declassified summary + secret scan receipt。

execution receipt：
  repairCommit = abc123
  changedFiles = ["scripts/build-lesson.ts", "policy/message-egress.ts"]
  changedGuardIds = ["egress.lesson_secret_scan"]
  replayReceiptIds = ["replay_original_12728", "replay_adjacent_github_push"]
  shadowReceiptIds = ["shadow_24h_lesson_messages"]
  canaryReceiptIds = ["canary_rust_group_10_percent"]
  remainingMitigations = []
~~~

Close gate 不能只看“下一次 Telegram 发成功了”。它要检查：

~~~text
原 messageId 12728 的事故样本 replay 是否 stable_pass；
相邻场景 GitHub README 更新是否没被误伤；
Telegram 课程消息 shadow/canary 是否没有 false_allow；
临时 evidenceMode=hash_only 是否已经恢复或转成正式策略；
TOOLS.md / MEMORY.md 是否记录了新规则，避免下一轮又绕开。
~~~

这才叫 finding closure。

---

## 8. 常见坑

**坑 1：worker 自己把 finding 关了。**

这会让修复者同时当裁判。生产里要让 close gate 独立消费 execution receipt。

**坑 2：只看当前测试，不看原事故样本。**

原事故样本没 stable pass，修复就没有证明解决根因。

**坑 3：shadow 样本不足也关单。**

样本不足只能 extend_verification。不能用“没看到问题”当“问题不存在”。

**坑 4：临时缓解忘记收口。**

freeze、sampling boost、evidence downgrade 如果不解除或固化，会造成长期功能退化或成本膨胀。

**坑 5：outcome mismatch 没有 backfill。**

现实世界已经发生的不一致，不会因为代码修好自动消失。必须补 outcome backfill 或 compensation receipt。

---

## 9. 落地清单

给 Guard Finding Repair 加关闭闸门，可以按 8 步做：

1. worker 完成修复时只写 RepairExecutionReceipt，不直接 close finding；
2. receipt 绑定 findingIds、batchId、repairCommit、guardPackVersionBefore/After；
3. replay receipt 必须包含 original incident 和 adjacent risk surface；
4. shadow/canary receipt 必须记录 sample_count、false_allow、false_block 和 min_required_samples；
5. close gate 对样本不足输出 extend_verification，而不是 close；
6. remainingMitigations 不为空时，必须 escalate owner 或写 policy receipt；
7. outcome_mismatch 必须有 outcomeBackfillIds；
8. close_verified 事件写入 EventBus，并能反查 commit、收据和外部副作用。

---

## 10. 总结

这一课的核心：

1. repair batch 完成不等于 finding close；
2. Execution Receipt 把 finding、commit、guard pack version、验证收据和临时缓解状态串起来；
3. Close Gate 应该输出 close_verified / extend_verification / reopen_finding / escalate_owner / manual_review；
4. learn-claude-code 可以用纯函数 close gate 训练判断逻辑；
5. pi-mono 可以用 EventBus 把 close decision 变成可追踪事件；
6. OpenClaw 的 Telegram、Git push、memory write 都需要这种关闭闸门，因为它们有真实外部副作用。

成熟 Agent 的修复闭环，不是“补丁合了就完事”，而是每个 finding 都能证明：修了什么、验证了什么、影响补了什么、临时措施怎么收口，以及为什么现在可以安全关闭。
