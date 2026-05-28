# 427. Agent 护栏修复回滚后的影响回填与补偿队列（Guard Pattern Repair Rollback Impact Backfill & Compensation Queue）

上一课讲了 **Guard Pattern Repair Canary Diff Budget & Rollback Trigger**：repair candidate 进入 canary 后开始真实影响生产决策，所以要维护 Decision Diff Budget；一旦多挡、少挡、改解释、多读证据或 bad diff 超过预算，就自动 rollback。

今天补上 rollback 之后最容易被忽略的一步：**已经被 candidate 影响过的真实决策怎么办**。

一句话：**护栏修复 canary 回滚不是把 pointer 切回 active 就结束。回滚后必须生成 Impact Backfill Job，把 canary 期间被 candidate 影响过的 decisionId 逐条回填 activeBaseline、candidateApplied、真实 outcome、side effect 状态和补偿动作，再由 Compensation Queue 幂等修复外部影响。**

---

## 1. 为什么 rollback 后还要做 impact backfill

很多系统把 rollback 理解成“以后不再用坏版本”。对普通代码发布来说，这已经解决了一半问题；但对 Agent 护栏来说，只解决了未来，没有解决过去。

Canary 期间 candidate 已经真实影响过决策：

1. candidate_more_strict 可能挡住了本该允许的操作；
2. candidate_more_lenient 可能放过了本该阻止的操作；
3. reason_changed 可能给用户或审计系统留下了错误解释；
4. evidence_expanded 可能触发了多余的证据读取、成本或隐私边界风险；
5. side effect 可能已经发消息、写文件、提交任务、调用 API。

所以 rollback 后必须回答三个问题：

- **影响范围**：哪些 decisionId 被 candidate 改变过？
- **真实后果**：这些改变有没有产生外部副作用或用户可见影响？
- **补偿路径**：该道歉、撤销、重跑、解封、补审计，还是只归档观察？

没有这一步，rollback 只是控制面恢复，现实世界还停在错误状态。

---

## 2. Impact Backfill 的最小数据模型

回填任务不要只存一个 rollbackReceiptId。它要能把每条受影响决策转成可执行的补偿单元。

~~~text
ImpactBackfillJob:
  jobId
  rollbackReceiptId
  repairId
  patternId
  candidateVersion
  activeBaselineVersion
  canaryStageId
  affectedDecisionIds[]
  status:
    queued | running | blocked | completed | failed

ImpactBackfillRecord:
  decisionId
  tenant
  subject
  riskSurface
  actionClass
  activeBaseline:
    decision
    reasonCode
    evidenceHash
  candidateApplied:
    decision
    reasonCode
    evidenceHash
  diffClass:
    candidate_more_strict |
    candidate_more_lenient |
    reason_changed |
    evidence_expanded
  outcome:
    confirmed_safe |
    confirmed_risky |
    reversed_by_human |
    unavailable |
    pending
  sideEffects[]
  compensationPlan:
    none |
    notify_user |
    rerun_action |
    revoke_action |
    unblock_action |
    append_audit_correction |
    manual_review
  compensationStatus:
    not_needed | queued | applied | skipped | blocked | failed
~~~

关键点是 **ImpactBackfillRecord** 一条一条落账。rollback 之后，系统不应该只知道“这个 stage 回滚了”，而应该知道“第 123 个决策因为 candidate 误挡，需要重新放行或通知用户”。

---

## 3. learn-claude-code：补偿计划分类器

教学版先写纯函数：根据 diffClass、outcome 和 side effect 状态决定补偿动作。

~~~py
from dataclasses import dataclass
from typing import Literal


DiffClass = Literal[
    "candidate_more_strict",
    "candidate_more_lenient",
    "reason_changed",
    "evidence_expanded",
]

Outcome = Literal[
    "confirmed_safe",
    "confirmed_risky",
    "reversed_by_human",
    "unavailable",
    "pending",
]

CompensationPlan = Literal[
    "none",
    "notify_user",
    "rerun_action",
    "revoke_action",
    "unblock_action",
    "append_audit_correction",
    "manual_review",
]


@dataclass(frozen=True)
class SideEffectSummary:
    has_external_effect: bool
    reversible: bool
    user_visible: bool


@dataclass(frozen=True)
class ImpactRecordInput:
    diff_class: DiffClass
    outcome: Outcome
    side_effect: SideEffectSummary


def choose_compensation_plan(record: ImpactRecordInput) -> CompensationPlan:
    if record.outcome in ("pending", "unavailable"):
        return "manual_review"

    if record.diff_class == "candidate_more_lenient":
        if record.outcome == "confirmed_risky":
            return "revoke_action" if record.side_effect.reversible else "manual_review"
        return "append_audit_correction"

    if record.diff_class == "candidate_more_strict":
        if record.outcome == "confirmed_safe":
            if not record.side_effect.has_external_effect:
                return "rerun_action"
            return "unblock_action" if record.side_effect.reversible else "notify_user"
        return "none"

    if record.diff_class == "reason_changed":
        return "append_audit_correction" if record.side_effect.user_visible else "none"

    if record.diff_class == "evidence_expanded":
        return "manual_review" if record.side_effect.has_external_effect else "append_audit_correction"

    return "none"
~~~

这段逻辑里最重要的是：补偿不是只按“放错/挡错”分类，还要看 side effect 是否已经发生、是否可逆、用户是否可见。

---

## 4. learn-claude-code：幂等补偿队列闸门

补偿动作一般会再次触发外部副作用，所以必须幂等。每条补偿任务都要有 stable key，重复执行也不能发两次通知、重跑两次 action。

~~~py
from dataclasses import dataclass
from typing import Literal


CompensationStatus = Literal[
    "queued",
    "applied",
    "skipped",
    "blocked",
    "failed",
]


@dataclass(frozen=True)
class CompensationTask:
    decision_id: str
    rollback_receipt_id: str
    plan: str
    idempotency_key: str
    prior_status: CompensationStatus | None = None
    requires_human: bool = False


def decide_compensation_enqueue(task: CompensationTask) -> tuple[str, list[str]]:
    if task.plan == "none":
        return "skip", ["no_compensation_needed"]

    if task.prior_status == "applied":
        return "skip", ["already_applied"]

    if task.prior_status == "queued":
        return "skip", ["already_queued"]

    if task.requires_human:
        return "block", ["requires_manual_review"]

    if not task.idempotency_key:
        return "block", ["missing_idempotency_key"]

    return "enqueue", ["ready_for_compensation"]
~~~

idempotency_key 可以这样构造：

~~~text
guard-compensation:{rollbackReceiptId}:{decisionId}:{plan}
~~~

这样同一个 rollback、同一个 decision、同一种补偿计划天然只有一次有效执行。

---

## 5. pi-mono：RollbackImpactBackfillWorker

生产版可以把 rollback 后处理拆成两个 worker：

- **ImpactBackfillWorker**：把 affectedDecisionIds 回填成 ImpactBackfillRecord；
- **CompensationWorker**：按 compensationPlan 幂等执行补偿动作。

~~~ts
type DiffClass =
  | "candidate_more_strict"
  | "candidate_more_lenient"
  | "reason_changed"
  | "evidence_expanded"

type Outcome =
  | "confirmed_safe"
  | "confirmed_risky"
  | "reversed_by_human"
  | "unavailable"
  | "pending"

type CompensationPlan =
  | "none"
  | "notify_user"
  | "rerun_action"
  | "revoke_action"
  | "unblock_action"
  | "append_audit_correction"
  | "manual_review"

type ImpactBackfillRecord = {
  decisionId: string
  rollbackReceiptId: string
  tenant: string
  subject: string
  diffClass: DiffClass
  outcome: Outcome
  sideEffect: {
    hasExternalEffect: boolean
    reversible: boolean
    userVisible: boolean
  }
  compensationPlan: CompensationPlan
}

function chooseCompensationPlan(record: {
  diffClass: DiffClass
  outcome: Outcome
  sideEffect: ImpactBackfillRecord["sideEffect"]
}): CompensationPlan {
  if (record.outcome === "pending" || record.outcome === "unavailable") {
    return "manual_review"
  }

  if (record.diffClass === "candidate_more_lenient") {
    if (record.outcome === "confirmed_risky") {
      return record.sideEffect.reversible ? "revoke_action" : "manual_review"
    }
    return "append_audit_correction"
  }

  if (record.diffClass === "candidate_more_strict") {
    if (record.outcome === "confirmed_safe") {
      return record.sideEffect.hasExternalEffect
        ? record.sideEffect.reversible ? "unblock_action" : "notify_user"
        : "rerun_action"
    }
    return "none"
  }

  if (record.diffClass === "reason_changed") {
    return record.sideEffect.userVisible ? "append_audit_correction" : "none"
  }

  if (record.diffClass === "evidence_expanded") {
    return record.sideEffect.hasExternalEffect ? "manual_review" : "append_audit_correction"
  }

  return "none"
}

class RollbackImpactBackfillWorker {
  constructor(
    private readonly store: {
      loadAffectedDecision(decisionId: string): Promise<Omit<ImpactBackfillRecord, "compensationPlan">>
      writeImpactRecord(record: ImpactBackfillRecord): Promise<void>
      enqueueCompensation(task: {
        idempotencyKey: string
        decisionId: string
        rollbackReceiptId: string
        plan: CompensationPlan
      }): Promise<void>
      markBlocked(decisionId: string, reason: string): Promise<void>
    },
  ) {}

  async handle(rollbackReceiptId: string, affectedDecisionIds: string[]): Promise<void> {
    for (const decisionId of affectedDecisionIds) {
      const raw = await this.store.loadAffectedDecision(decisionId)
      const compensationPlan = chooseCompensationPlan(raw)
      const record: ImpactBackfillRecord = { ...raw, compensationPlan }

      await this.store.writeImpactRecord(record)

      if (compensationPlan === "none") {
        continue
      }

      if (compensationPlan === "manual_review") {
        await this.store.markBlocked(decisionId, "manual_review_required")
        continue
      }

      await this.store.enqueueCompensation({
        idempotencyKey: [
          "guard-compensation",
          rollbackReceiptId,
          decisionId,
          compensationPlan,
        ].join(":"),
        decisionId,
        rollbackReceiptId,
        plan: compensationPlan,
      })
    }
  }
}
~~~

生产里还要加两个保护：

1. **先写 ImpactBackfillRecord，再 enqueue compensation**，否则队列执行失败时无法解释它为什么存在；
2. **CompensationWorker 执行前重新读取当前 reality snapshot**，防止人工已经修过、系统还重复补偿。

---

## 6. OpenClaw：课程 Cron 的类比实战

这条课程 Cron 也有同样的问题。

假设我已经把课程发到 Telegram，但后面 git push 失败了。只做“下次别失败”不够，还要回填影响：

- Telegram messageId 是否已经发出；
- lesson 文件是否已经写入本地；
- README 是否包含目录；
- TOOLS.md 是否已经写入已讲内容；
- commit 是否存在；
- remote main 是否包含 commit；
- 如果缺一项，应该补 push、补目录、补记忆，还是发更正。

这里的 ImpactBackfillRecord 就是一次 cron 的现实账本。它不是为了写漂亮日志，而是为了保证“发群、写文件、提交 repo、更新记忆”这些外部副作用最终一致。

---

## 7. 工程落地清单

护栏修复 canary 支持 rollback 时，至少要配齐这些东西：

1. **Affected Decision Index**：canary 期所有 diff decisionId 可检索；
2. **Rollback Receipt**：记录 rollback 原因、候选版本、active baseline 和时间窗口；
3. **Impact Backfill Job**：逐条回填 outcome、side effect、补偿计划；
4. **Compensation Queue**：所有补偿动作带 idempotency key；
5. **Reality Snapshot**：补偿前重新确认外部状态；
6. **Manual Review Lane**：不可逆、证据越界、outcome unknown 的记录不能自动补偿；
7. **Close Gate**：所有 affected decisions 都 terminal 后，rollback 才能关闭。

一个实用判断标准：

> rollback 后，如果你不能列出“哪些真实决策被影响、每条现在是什么状态、哪些已补偿、哪些需要人工”，那这个 rollback 还没有结束。

---

## 8. 小结

成熟 Agent 的护栏修复回滚，不是把 pointer 切回旧版就收工，而是把 canary 期间影响过的真实决策逐条回填，按 outcome 和 side effect 生成补偿计划，再通过幂等队列修复现实影响。

这套机制让 rollback 从“控制面动作”变成“现实闭环”：**未来流量回到安全基线，过去受影响的决策也能被看见、被分类、被补偿、被关闭。**
