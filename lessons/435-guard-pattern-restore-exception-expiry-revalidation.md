# 435. Agent 护栏恢复例外的过期、再验证与自动收回

> **核心思想：RestoreException 不是永久白名单。成熟 Agent 要把每次 family lock 解锁后的恢复资格做成有 TTL、有使用预算、有再验证探针、有自动收回路径的租约；到期、超预算、依赖漂移或复发信号命中时，release gate 必须自动撤销例外并回到 locked 状态。**

---

## 1. 为什么 restore exception 不能长期躺在 release gate 里

上一课讲了 CandidateFamilyLock 解除不能删除锁，而是在锁上追加 restore exception：某个 candidate version 在某个 tenant / riskSurface / actionClass / stage 内临时恢复资格。

这里最容易犯的错是：exception 一加进去就忘了。

这会带来三个问题：

1. 当初允许恢复的 replay proof 会过期；
2. runtime、tool schema、policy、prompt pack 可能已经漂移；
3. 一个很小的 shadow/canary 例外，可能被后续 release gate 当成长期白名单。

所以 RestoreException 必须像 lease 一样管理：

- expiresAt 到期自动失效；
- maxDecisions / maxBadDiff 超预算自动收回；
- 关键依赖漂移时必须重新 replay；
- 复发信号命中时直接 revoke 并恢复 family lock 阻断。

---

## 2. 最小数据模型

~~~text
RestoreExceptionLease:
  exceptionId
  familyLockId
  candidateVersion
  stage: shadow | canary | active
  allowedScope:
    tenants[]
    riskSurfaces[]
    actionClasses[]
  budget:
    maxDecisions
    maxBadDiff
    maxEvidenceExpansion
    expiresAt
  dependencyFingerprint:
    policyHash
    toolSchemaHash
    promptPackHash
    evidenceContractHash
  revalidation:
    nextProbeAt
    requiredRegressionSeedIds[]
    requiredShadowSample
  status:
    active | expired | pending_revalidation | revoked | renewed

RestoreExceptionObservation:
  exceptionId
  decisionsUsed
  badDiffCount
  evidenceExpansionCount
  recurrenceSignalCount
  dependencyDrift
  now

RestoreExceptionDecision:
  decision:
    keep_active |
    renew_with_smaller_scope |
    require_revalidation |
    expire_exception |
    revoke_exception |
    manual_review
  reasonCodes[]
  nextProbeAt?
  replacementScope?
~~~

这里的关键是：release gate 读到 exception 时不能只问“存在吗”，而要问“现在仍然有效吗”。

---

## 3. 自动收回的五个触发器

第一，时间到期。
expiresAt <= now 时直接 expire_exception。过期不是事故，只是租约自然结束。

第二，预算耗尽。
decisionsUsed >= maxDecisions 或 badDiffCount > maxBadDiff 时不能继续放行。前者进入 revalidation，后者直接 revoke。

第三，证据边界扩大。
恢复例外本来就是从 tombstone / family lock 里临时开小口。如果 observed evidence 比批准时读得更多，要默认收回，而不是补一条注释。

第四，依赖漂移。
policyHash、toolSchemaHash、promptPackHash、evidenceContractHash 任一关键项变化，都要求重新跑 locked regression seeds。否则旧 proof 不能证明新运行时仍然安全。

第五，复发信号。
只要命中 locked regression seed、recurrence sentinel 或 forbidden signature，就不应该等到窗口结束，立刻 revoke exception，并把 observation 写回 Reseal / Reopen 流程。

---

## 4. learn-claude-code：恢复例外租约判定纯函数

教学版可以先把 RestoreException 当成纯数据结构，写一个不依赖 IO 的判定器。它的职责不是发布新版本，而是保护 release gate 不会继续相信过期例外。

~~~py
from dataclasses import dataclass
from typing import Literal


Stage = Literal["shadow", "canary", "active"]
Decision = Literal[
    "keep_active",
    "renew_with_smaller_scope",
    "require_revalidation",
    "expire_exception",
    "revoke_exception",
    "manual_review",
]


@dataclass(frozen=True)
class RestoreBudget:
    max_decisions: int
    max_bad_diff: int
    max_evidence_expansion: int
    expires_at: int


@dataclass(frozen=True)
class DependencyFingerprint:
    policy_hash: str
    tool_schema_hash: str
    prompt_pack_hash: str
    evidence_contract_hash: str


@dataclass(frozen=True)
class RestoreExceptionLease:
    exception_id: str
    family_lock_id: str
    candidate_version: str
    stage: Stage
    allowed_risk_surfaces: tuple[str, ...]
    budget: RestoreBudget
    dependency_fingerprint: DependencyFingerprint
    required_regression_seed_ids: tuple[str, ...]
    next_probe_at: int


@dataclass(frozen=True)
class RestoreObservation:
    now: int
    decisions_used: int
    bad_diff_count: int
    evidence_expansion_count: int
    recurrence_signal_count: int
    current_dependency_fingerprint: DependencyFingerprint
    covered_regression_seed_ids: tuple[str, ...]
    failed_regression_seed_ids: tuple[str, ...]


@dataclass(frozen=True)
class RestoreExceptionDecision:
    exception_id: str
    decision: Decision
    reason_codes: tuple[str, ...]
    next_probe_at: int | None = None


def dependencies_changed(
    lease: RestoreExceptionLease,
    obs: RestoreObservation,
) -> bool:
    return lease.dependency_fingerprint != obs.current_dependency_fingerprint


def locked_seeds_covered(
    lease: RestoreExceptionLease,
    obs: RestoreObservation,
) -> bool:
    return set(lease.required_regression_seed_ids).issubset(
        set(obs.covered_regression_seed_ids)
    )


def decide_restore_exception(
    lease: RestoreExceptionLease,
    obs: RestoreObservation,
) -> RestoreExceptionDecision:
    reasons: list[str] = []

    if obs.now >= lease.budget.expires_at:
        return RestoreExceptionDecision(
            lease.exception_id,
            "expire_exception",
            ("lease_expired",),
        )

    if obs.recurrence_signal_count > 0:
        return RestoreExceptionDecision(
            lease.exception_id,
            "revoke_exception",
            ("recurrence_signal_hit",),
        )

    if obs.failed_regression_seed_ids:
        return RestoreExceptionDecision(
            lease.exception_id,
            "revoke_exception",
            ("locked_regression_seed_failed",),
        )

    if obs.bad_diff_count > lease.budget.max_bad_diff:
        return RestoreExceptionDecision(
            lease.exception_id,
            "revoke_exception",
            ("bad_diff_budget_exceeded",),
        )

    if obs.evidence_expansion_count > lease.budget.max_evidence_expansion:
        return RestoreExceptionDecision(
            lease.exception_id,
            "revoke_exception",
            ("evidence_expansion_budget_exceeded",),
        )

    if dependencies_changed(lease, obs):
        reasons.append("dependency_fingerprint_changed")

    if obs.decisions_used >= lease.budget.max_decisions:
        reasons.append("decision_budget_exhausted")

    if obs.now >= lease.next_probe_at:
        reasons.append("scheduled_probe_due")

    if reasons:
        if not locked_seeds_covered(lease, obs):
            reasons.append("locked_seed_replay_missing")
        return RestoreExceptionDecision(
            lease.exception_id,
            "require_revalidation",
            tuple(reasons),
            next_probe_at=obs.now,
        )

    return RestoreExceptionDecision(
        lease.exception_id,
        "keep_active",
        ("within_budget",),
        next_probe_at=lease.next_probe_at,
    )
~~~

这里故意把 expire_exception 和 revoke_exception 分开：

- expire 是正常到期，通常不需要 incident；
- revoke 是安全信号失败，需要写 RevokeReceipt，并可能重开 ResealCase。

---

## 5. pi-mono：用 Agent.subscribe 做旁路观察 worker

生产版可以把 restore exception 的观察做成 event worker。pi-mono 的 Agent.subscribe 模式很适合做这种旁路账本：主决策路径继续跑，观察 worker 订阅 guard decision / tool result / message_end 事件，更新 exception usage。

~~~ts
type GuardDecisionEvent = {
  type: "guard_decision";
  decisionId: string;
  candidateVersion: string;
  familyLockId?: string;
  restoreExceptionId?: string;
  riskSurface: string;
  diffClass?: "same" | "bad_diff" | "evidence_expanded" | "unknown";
  recurrenceSignal?: boolean;
  dependencyFingerprint: string;
};

type RestoreExceptionLedger = {
  incrementUsage(exceptionId: string, event: GuardDecisionEvent): Promise<void>;
  snapshot(exceptionId: string): Promise<RestoreObservation>;
  writeDecision(decision: RestoreExceptionDecision): Promise<void>;
  revoke(exceptionId: string, reasonCodes: string[]): Promise<void>;
  expire(exceptionId: string): Promise<void>;
};

function subscribeRestoreExceptionWorker(
  agent: { subscribe(fn: (event: unknown) => void): () => void },
  leases: { get(id: string): Promise<RestoreExceptionLease | null> },
  ledger: RestoreExceptionLedger,
) {
  return agent.subscribe(async (event) => {
    if (!isGuardDecisionEvent(event)) return;
    if (!event.restoreExceptionId) return;

    await ledger.incrementUsage(event.restoreExceptionId, event);

    const lease = await leases.get(event.restoreExceptionId);
    if (!lease) return;

    const observation = await ledger.snapshot(event.restoreExceptionId);
    const decision = decideRestoreException(lease, observation);
    await ledger.writeDecision(decision);

    if (decision.decision === "revoke_exception") {
      await ledger.revoke(decision.exception_id, [...decision.reason_codes]);
      return;
    }

    if (decision.decision === "expire_exception") {
      await ledger.expire(decision.exception_id);
    }
  });
}

function isGuardDecisionEvent(event: unknown): event is GuardDecisionEvent {
  return (
    typeof event === "object" &&
    event !== null &&
    (event as { type?: string }).type === "guard_decision"
  );
}
~~~

这个 worker 的重点不是“发现问题发日志”，而是把 observation 写进 ledger，让 release gate 下一次判断时能读到最新状态。

---

## 6. Release Gate：只接受仍然有效的 exception

真正挡事故的是 gate。它不能只查 restore_exceptions where status = active，还要在放行前执行租约判定。

~~~ts
async function canUseRestoreException(input: {
  lease: RestoreExceptionLease;
  observation: RestoreObservation;
  requestedRiskSurface: string;
}) {
  if (!input.lease.allowed_risk_surfaces.includes(input.requestedRiskSurface)) {
    return {
      allow: false,
      reason: "risk_surface_outside_restore_scope",
    };
  }

  const decision = decideRestoreException(input.lease, input.observation);

  if (decision.decision === "keep_active") {
    return { allow: true, reason: "restore_exception_valid" };
  }

  if (decision.decision === "require_revalidation") {
    return { allow: false, reason: "restore_exception_needs_revalidation" };
  }

  return { allow: false, reason: decision.reason_codes.join(",") };
}
~~~

这也是为什么 exception 要保留 familyLockId：当它不可用时，系统应该自然落回原来的 family lock 阻断，而不是进入“没有规则覆盖”的空白区。

---

## 7. OpenClaw 课程 Cron 里的对应实践

这套逻辑和我们发课 cron 很像：

1. 课程选题相当于 candidate；
2. TOOLS.md 已讲列表相当于 tombstone / family lock；
3. README、lesson 文件、Telegram messageId、Git commit 是 release receipt；
4. 如果某次发课只临时允许某个相近主题继续讲，就必须有到期条件和再验证条件。

一个实用的发课守卫可以这样做：

~~~text
RestoreException:
  familyLock: "guard-pattern-family-lock"
  allowedTopic: "restore exception expiry"
  maxLessons: 1
  expiresAt: "2026-05-29T21:30:00+11:00"
  revalidate:
    - check README latest lesson number
    - check TOOLS.md taught list
    - check git remote contains previous commit
  revokeIf:
    - topic already exists
    - README and lessons disagree
    - Telegram sent but git push failed
~~~

这不是形式主义。长期自动化任务最怕的不是一次失败，而是一次“临时绕过”变成永久路径。

---

## 8. 落地检查清单

- RestoreException 必须有 expiresAt 和预算，不允许永久 active；
- release gate 必须在每次放行前重新判断 exception 状态；
- dependency fingerprint 变化后必须要求重新 replay；
- recurrence signal / bad diff / evidence expansion 超预算时直接 revoke；
- expire 和 revoke 要写不同 receipt；
- revoke 后默认回到 family lock 阻断，并路由到 Reseal / Reopen；
- revalidation 通过后不要原地续命太久，优先缩小 scope 或重新走 staged restore。

一句话总结：**恢复例外不是白名单，是一张会过期、会复查、会被自动收回的临时通行证。**
