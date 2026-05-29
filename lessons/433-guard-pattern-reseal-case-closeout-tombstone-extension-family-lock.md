# 433. Agent 护栏复封案例关闭与候选族锁定

> **核心思想：ResealCase routed 不等于可以关闭。成熟 Agent 在关闭复封案例前，要把失败证据固化成 TombstoneExtension，把相似 candidate family 锁住，把后续 release gate 接到新的 forbidden signature 上，并留下 CloseoutReceipt。否则这次复封只是挡住了一个版本，挡不住同一类坏修复换个名字重新进发布队列。**

---

## 1. 为什么复封案例不能只停在 routed

上一课讲了 ResealReceipt 之后要创建 ResealCase：隔离 candidate version、回填 affected decisions、分类 root cause，再分流到 reject、repair、extend replay 或 incident escalation。

但 routed 只说明“下一步该去哪”，还没有把这次失败沉淀成发布系统能执行的约束。

复封案例关闭前，还要回答四个问题：

1. 这次失败有没有扩展原 tombstone 的 forbidden signature；
2. 哪些 sibling candidate 或相邻 scope 必须一起冻结；
3. 后续 repair / shadow / canary gate 要多跑哪些 regression seed；
4. 关闭后如果同类失败再次出现，应该 reopen 原 case 还是创建新 case。

这就是 **Reseal Case Closeout** 的职责：不是继续分析，而是把分析结果变成可执行的发布阻断和复发检测输入。

---

## 2. 最小数据模型

~~~text
ResealCloseoutReceipt:
  closeoutId
  caseId
  tombstoneId
  candidateVersion
  finalRootCause
  finalRouting
  tombstoneExtension:
    extensionId
    forbiddenPredicateHashes[]
    forbiddenEvidenceBoundaryDelta
    forbiddenRuntimeDelta
    addedRegressionSeedIds[]
  candidateFamilyLock:
    familyKey
    lockedVersions[]
    blockedScopes[]
    releaseGate:
      blockShadow: boolean
      blockCanary: boolean
      blockActive: boolean
    expiresAt | permanent
  recurrenceSentinel:
    signatureHash
    reopenCaseId
    matchFields[]
  closeDecision:
    close_rejected |
    close_repair_opened |
    close_replay_extended |
    close_escalated |
    hold_missing_evidence
~~~

注意这里有两个输出最重要：

- **TombstoneExtension**：让原 tombstone 从“阻止旧坏版本”升级成“阻止同类坏修复”。
- **CandidateFamilyLock**：防止 sibling candidate 复用相同 predicate、相同 evidence contract 或相同 runtime dependency 继续发布。

如果没有这两层，Agent 会进入一个很典型的坏循环：版本 A 复封，工程师或子代理复制 A 改个名字生成 A2，A2 又绕过历史失败进入 shadow/canary。

---

## 3. 关闭前的三个硬闸门

第一，证据完整性闸门。
Closeout 必须能指向 frozen forensic bundle：failed seed、bad diff decision、scope violation、dependency delta、runtime fingerprint delta。缺证据只能 hold_missing_evidence，不能 close。

第二，发布阻断闸门。
如果 root cause 是 old_failure_recurred、repair_overfit、evidence_contract_broken 或 runtime_dependency_drift，必须生成 candidate family lock，并把 lock 注入 release gate。只写日志不算关闭。

第三，复发哨兵闸门。
关闭时要生成 recurrence sentinel。后续 audit sample、shadow comparison、canary rollback 或 reseal trigger 命中同一 signature 时，系统应该优先 reopen 原 case 或升级 severity，而不是当成孤立新事故。

---

## 4. learn-claude-code：关闭复封案例的纯函数

教学版先把 ResealCase 转成 CloseoutReceipt。重点是：closeout 的返回值不能只有 status，它必须携带 tombstone extension 和 family lock。

~~~py
from dataclasses import dataclass
from typing import Literal


RootCauseClass = Literal[
    "old_failure_recurred",
    "repair_overfit",
    "predicate_too_broad",
    "predicate_too_loose",
    "evidence_contract_broken",
    "runtime_dependency_drift",
    "unknown",
]

RoutingDecision = Literal[
    "reject_candidate",
    "open_repair",
    "extend_replay",
    "escalate_incident",
    "manual_review",
]

CloseDecision = Literal[
    "close_rejected",
    "close_repair_opened",
    "close_replay_extended",
    "close_escalated",
    "hold_missing_evidence",
]


@dataclass(frozen=True)
class ResealCase:
    case_id: str
    tombstone_id: str
    pattern_id: str
    candidate_version: str
    root_cause_class: RootCauseClass
    routing_decision: RoutingDecision
    affected_decision_ids: tuple[str, ...]


@dataclass(frozen=True)
class ForensicBundle:
    predicate_hashes: tuple[str, ...]
    failed_regression_seed_ids: tuple[str, ...]
    bad_diff_decision_ids: tuple[str, ...]
    evidence_boundary_delta: int
    runtime_delta_hash: str | None
    dependency_delta_hash: str | None

    def has_minimum_evidence(self) -> bool:
        return bool(
            self.failed_regression_seed_ids
            or self.bad_diff_decision_ids
            or self.evidence_boundary_delta > 0
            or self.runtime_delta_hash
            or self.dependency_delta_hash
        )


@dataclass(frozen=True)
class TombstoneExtension:
    extension_id: str
    forbidden_predicate_hashes: tuple[str, ...]
    forbidden_evidence_boundary_delta: int
    forbidden_runtime_delta_hash: str | None
    added_regression_seed_ids: tuple[str, ...]


@dataclass(frozen=True)
class CandidateFamilyLock:
    family_key: str
    locked_versions: tuple[str, ...]
    blocked_scopes: tuple[str, ...]
    block_shadow: bool
    block_canary: bool
    block_active: bool
    permanent: bool


@dataclass(frozen=True)
class RecurrenceSentinel:
    signature_hash: str
    reopen_case_id: str
    match_fields: tuple[str, ...]


@dataclass(frozen=True)
class ResealCloseoutReceipt:
    closeout_id: str
    case_id: str
    tombstone_extension: TombstoneExtension | None
    candidate_family_lock: CandidateFamilyLock | None
    recurrence_sentinel: RecurrenceSentinel | None
    close_decision: CloseDecision


def close_decision_for(case: ResealCase) -> CloseDecision:
    if case.routing_decision == "reject_candidate":
        return "close_rejected"
    if case.routing_decision == "open_repair":
        return "close_repair_opened"
    if case.routing_decision == "extend_replay":
        return "close_replay_extended"
    if case.routing_decision == "escalate_incident":
        return "close_escalated"
    return "hold_missing_evidence"


def needs_family_lock(root_cause: RootCauseClass) -> bool:
    return root_cause in {
        "old_failure_recurred",
        "repair_overfit",
        "evidence_contract_broken",
        "runtime_dependency_drift",
    }


def close_reseal_case(
    case: ResealCase,
    evidence: ForensicBundle,
    sibling_versions: tuple[str, ...],
) -> ResealCloseoutReceipt:
    if not evidence.has_minimum_evidence():
        return ResealCloseoutReceipt(
            closeout_id=f"reseal-closeout:{case.case_id}",
            case_id=case.case_id,
            tombstone_extension=None,
            candidate_family_lock=None,
            recurrence_sentinel=None,
            close_decision="hold_missing_evidence",
        )

    extension = TombstoneExtension(
        extension_id=f"tombstone-ext:{case.case_id}",
        forbidden_predicate_hashes=evidence.predicate_hashes,
        forbidden_evidence_boundary_delta=evidence.evidence_boundary_delta,
        forbidden_runtime_delta_hash=evidence.runtime_delta_hash,
        added_regression_seed_ids=evidence.failed_regression_seed_ids,
    )

    family_lock = None
    if needs_family_lock(case.root_cause_class):
        family_lock = CandidateFamilyLock(
            family_key=f"{case.pattern_id}:{hash(evidence.predicate_hashes)}",
            locked_versions=(case.candidate_version, *sibling_versions),
            blocked_scopes=("shadow", "canary", "active"),
            block_shadow=True,
            block_canary=True,
            block_active=True,
            permanent=case.root_cause_class == "old_failure_recurred",
        )

    sentinel = RecurrenceSentinel(
        signature_hash=f"{case.tombstone_id}:{hash((evidence.predicate_hashes, evidence.runtime_delta_hash))}",
        reopen_case_id=case.case_id,
        match_fields=("predicate_hashes", "runtime_delta_hash", "failed_regression_seed_ids"),
    )

    return ResealCloseoutReceipt(
        closeout_id=f"reseal-closeout:{case.case_id}",
        case_id=case.case_id,
        tombstone_extension=extension,
        candidate_family_lock=family_lock,
        recurrence_sentinel=sentinel,
        close_decision=close_decision_for(case),
    )
~~~

这个函数的工程价值在于：任何调用方拿到 closeout receipt 后，都能明确知道应该更新 tombstone index、family lock registry 和 recurrence index。它不是一段复盘文本，而是一份可执行关闭收据。

---

## 5. pi-mono：Closeout Worker 接 release gate

生产版可以把 Closeout 做成事件驱动 worker：监听 \`guard.reseal.case.routed\`，生成 closeout receipt，再原子写入三个索引。

~~~ts
type ResealCaseRouted = {
  caseId: string;
  tombstoneId: string;
  patternId: string;
  candidateVersion: string;
  rootCause:
    | "old_failure_recurred"
    | "repair_overfit"
    | "evidence_contract_broken"
    | "runtime_dependency_drift"
    | "unknown";
  routing: "reject_candidate" | "open_repair" | "extend_replay" | "escalate_incident" | "manual_review";
};

type ResealCloseoutReceipt = {
  closeoutId: string;
  caseId: string;
  tombstoneExtensionId: string;
  familyLockId?: string;
  recurrenceSentinelId: string;
  closeDecision:
    | "close_rejected"
    | "close_repair_opened"
    | "close_replay_extended"
    | "close_escalated"
    | "hold_missing_evidence";
};

class ResealCaseCloseoutWorker {
  constructor(
    private readonly forensicStore: ForensicStore,
    private readonly tombstoneIndex: TombstoneIndex,
    private readonly familyLockRegistry: CandidateFamilyLockRegistry,
    private readonly recurrenceIndex: RecurrenceIndex,
    private readonly eventBus: EventBus,
  ) {}

  async handle(event: ResealCaseRouted): Promise<ResealCloseoutReceipt> {
    const evidence = await this.forensicStore.freezeBundle(event.caseId);

    if (!evidence.hasMinimumEvidence()) {
      const receipt = {
        closeoutId: "reseal-closeout:" + event.caseId,
        caseId: event.caseId,
        tombstoneExtensionId: "",
        recurrenceSentinelId: "",
        closeDecision: "hold_missing_evidence" as const,
      };
      await this.eventBus.publish("guard.reseal.closeout.held", receipt);
      return receipt;
    }

    const extension = await this.tombstoneIndex.extend({
      tombstoneId: event.tombstoneId,
      caseId: event.caseId,
      forbiddenPredicateHashes: evidence.predicateHashes,
      regressionSeedIds: evidence.failedRegressionSeedIds,
      evidenceBoundaryDelta: evidence.evidenceBoundaryDelta,
      runtimeDeltaHash: evidence.runtimeDeltaHash,
    });

    const familyLock = await this.maybeLockCandidateFamily(event, evidence);

    const sentinel = await this.recurrenceIndex.register({
      sourceCaseId: event.caseId,
      signature: {
        predicateHashes: evidence.predicateHashes,
        runtimeDeltaHash: evidence.runtimeDeltaHash,
        failedRegressionSeedIds: evidence.failedRegressionSeedIds,
      },
      onMatch: "reopen_or_escalate",
    });

    const receipt: ResealCloseoutReceipt = {
      closeoutId: "reseal-closeout:" + event.caseId,
      caseId: event.caseId,
      tombstoneExtensionId: extension.extensionId,
      familyLockId: familyLock?.lockId,
      recurrenceSentinelId: sentinel.sentinelId,
      closeDecision: this.closeDecisionFor(event.routing),
    };

    await this.eventBus.publish("guard.reseal.closeout.receipt", receipt);
    return receipt;
  }

  private async maybeLockCandidateFamily(event: ResealCaseRouted, evidence: FrozenForensicBundle) {
    if (!["old_failure_recurred", "repair_overfit", "evidence_contract_broken", "runtime_dependency_drift"].includes(event.rootCause)) {
      return undefined;
    }

    return this.familyLockRegistry.lock({
      familyKey: event.patternId + ":" + evidence.familySignatureHash,
      sourceCaseId: event.caseId,
      blockedVersions: [event.candidateVersion, ...evidence.siblingCandidateVersions],
      blockedStages: ["shadow", "canary", "active"],
      releaseGateMode: "deny_until_unseal",
    });
  }

  private closeDecisionFor(routing: ResealCaseRouted["routing"]): ResealCloseoutReceipt["closeDecision"] {
    if (routing === "reject_candidate") return "close_rejected";
    if (routing === "open_repair") return "close_repair_opened";
    if (routing === "extend_replay") return "close_replay_extended";
    if (routing === "escalate_incident") return "close_escalated";
    return "hold_missing_evidence";
  }
}
~~~

这里的关键不是 worker 名字，而是写入顺序：先 freeze forensic bundle，再 extend tombstone，再 lock family，再 register recurrence sentinel，最后发布 closeout receipt。顺序反了，就可能出现 receipt 已 close、release gate 还没接上阻断规则的空窗。

---

## 6. OpenClaw 实战：课程 Cron 也要有 closeout

这套模式放到 OpenClaw 课程 Cron，很像我们每 3 小时发课的收尾动作。

一次课程任务结束不能只说“发了 Telegram、提交了 Git”。更完整的 closeout 应该记录：

~~~text
LessonCloseoutReceipt:
  lessonNo: 433
  topic: guard-pattern-reseal-case-closeout-tombstone-extension-family-lock
  telegramMessageId
  commitSha
  files:
    - lessons/433-...
    - README.md
    - TOOLS.md
    - memory/2026-05-29.md
  duplicateTopicCheck:
    checkedSources:
      - TOOLS.md 已讲内容
      - lessons/
      - README.md
    result: no_duplicate
  recurrenceSentinel:
    avoidTopics:
      - 已讲过的 reseal forensics
      - 已讲过的 tombstone unseal
      - 已讲过的 recurrence detection
    nextAllowedDirection:
      - reseal closeout
      - family lock
      - release gate wiring
~~~

这不是形式主义。长期自动课程如果没有 closeout receipt，很容易发生三类问题：

- README 有了，TOOLS 没更新；
- Telegram 发了，Git 没推成功；
- 新题目只是旧题目换说法。

所以课程 Cron 的“任务关闭”也应该带证据：messageId、commit、文件 diff、去重结果和下一次选题边界。

---

## 7. 常见坑

不要把 ResealCase 的 routing 当 closeout。
Routing 是“去哪修”，closeout 是“哪些阻断和复发哨兵已经生效”。

不要只锁 candidateVersion。
很多坏修复不是版本号的问题，而是 predicate family、evidence contract 或 runtime dependency 的问题。只锁一个版本，sibling candidate 会继续绕过。

不要把 family lock 设成永久默认。
old_failure_recurred 可以永久锁；repair_overfit 更适合 deny_until_unseal；runtime_dependency_drift 可能在依赖锁刷新和 replay 通过后解锁。

不要跳过 recurrence sentinel。
复封案例关闭后，如果同类失败再次出现，系统应该自动识别为复发或相邻风险，而不是让值班 Agent 从零开始调查。

---

## 8. 小结

ResealCase Closeout 解决的是“复封之后如何真正留下工程约束”。

最小闭环是：

1. freeze forensic bundle；
2. extend tombstone；
3. lock candidate family；
4. register recurrence sentinel；
5. emit closeout receipt；
6. release gate 开始执行新的阻断规则。

成熟 Agent 不只是会发现坏候选、复封坏候选，还会在关闭案例前把失败变成后续发布系统绕不过去的规则。
