# 443. Agent 护栏族锁释放后的残余约束监控与复发哨兵

> **核心思想：FamilyLockReleaseReceipt 证明族锁已经按证据释放，但不代表系统可以忘掉这段事故。成熟 Agent 要把 residual constraints、tombstone 和 recurrence sentinel 转成带期限的 PostReleaseMonitoringLease，持续观察复发、范围漂移和证据过期，必要时自动重开锁。**

---

## 1. 为什么锁释放后还要继续监控

上一课讲了 RestoreFinalizationReceipt 和 FamilyLockReleaseGate：active 阶段稳定以后，只有当回归种子、发布债务、临时许可、rollback chain 都定版，才能 full release 或 scoped release family lock。

但 release receipt 不是“事故记忆删除证明”。它只说明当前证据允许把族锁从阻断状态移开。

真正上线后还有三类风险：

1. residual constraint 仍然存在，只是被接受在限定范围内运行；
2. tombstone 和 recurrence sentinel 仍然可能在新流量里命中；
3. 依赖、工具 schema、证据源、策略版本可能在释放后漂移。

所以族锁释放后，要创建 PostReleaseMonitoringLease。它不是普通告警，而是一张有期限、有 scope、有触发动作的监控租约：

- 观察 released scope 内的真实决策；
- 按 recurrence sentinel 匹配历史失败特征；
- 检查 residual constraints 是否被突破；
- 在 lease 到期时决定 archive、extend 或 reopen lock。

一句话：族锁释放是解除阻断，不是解除记忆。

---

## 2. 最小数据模型

~~~text
PostReleaseMonitoringLease:
  leaseId
  familyLockId
  releaseReceiptId
  releasedScope
  residualConstraints[]
  tombstoneRefs[]
  recurrenceSentinelRefs[]
  dependencyFingerprint
  watchSignals:
    recurrence_match
    scope_violation
    residual_constraint_breach
    dependency_drift
    unexplained_decision_diff
  startsAt
  expiresAt
  onBreach:
    reopen_family_lock
    freeze_release_lane
    open_reseal_case
    escalate_owner
  status:
    observing | breached | expired | archived

PostReleaseSignal:
  signalId
  leaseId
  signalKind
  decisionId
  evidenceRefs[]
  severity
  observedAt

PostReleaseLeaseDecision:
  decisionId
  leaseId
  decision:
    archive_clean
    extend_monitoring
    reopen_family_lock
    open_reseal_case
    escalate_owner
  reason
  emittedAt
~~~

这里最容易漏的是 dependencyFingerprint。很多复发不是旧 predicate 又错了，而是证据源 schema、工具返回字段、策略版本或 runtime group 变了。release 后如果依赖漂移，不能等用户结果变坏才发现。

另一个重点是 onBreach 必须预先声明。监控命中以后，系统不能临场让 LLM 自由发挥；高风险 breach 要自动 freeze release lane，必要时重开 family lock。

---

## 3. learn-claude-code：租约判定纯函数

教学版先写成纯函数。输入 release 后的信号、租约约束和当前依赖指纹，输出下一步动作。

~~~py
from dataclasses import dataclass
from typing import Literal


SignalKind = Literal[
    "recurrence_match",
    "scope_violation",
    "residual_constraint_breach",
    "dependency_drift",
    "unexplained_decision_diff",
]

Decision = Literal[
    "archive_clean",
    "extend_monitoring",
    "reopen_family_lock",
    "open_reseal_case",
    "escalate_owner",
]


@dataclass(frozen=True)
class PostReleaseSignal:
    kind: SignalKind
    severity: int
    evidence_refs: list[str]


@dataclass(frozen=True)
class MonitoringLease:
    expires_at_epoch: int
    dependency_fingerprint: str
    expected_dependency_fingerprint: str
    residual_constraints: list[str]
    min_clean_decisions: int
    observed_clean_decisions: int
    signals: list[PostReleaseSignal]


@dataclass(frozen=True)
class LeaseDecision:
    decision: Decision
    reason: str


def decide_post_release_lease(
    lease: MonitoringLease,
    now_epoch: int,
) -> LeaseDecision:
    if lease.dependency_fingerprint != lease.expected_dependency_fingerprint:
        return LeaseDecision("extend_monitoring", "dependency_fingerprint_changed")

    for signal in lease.signals:
        if signal.kind == "recurrence_match" and signal.severity >= 8:
            return LeaseDecision("reopen_family_lock", "high_severity_recurrence")

        if signal.kind == "scope_violation":
            return LeaseDecision("reopen_family_lock", "released_scope_violated")

        if signal.kind == "residual_constraint_breach":
            return LeaseDecision("open_reseal_case", "residual_constraint_breached")

        if signal.kind == "unexplained_decision_diff" and signal.severity >= 7:
            return LeaseDecision("escalate_owner", "decision_diff_needs_review")

    if now_epoch < lease.expires_at_epoch:
        return LeaseDecision("extend_monitoring", "lease_window_still_open")

    if lease.observed_clean_decisions < lease.min_clean_decisions:
        return LeaseDecision("extend_monitoring", "not_enough_clean_traffic")

    if lease.residual_constraints:
        return LeaseDecision("extend_monitoring", "residual_constraints_still_active")

    return LeaseDecision("archive_clean", "lease_completed_without_breach")
~~~

这段逻辑的重点是顺序：高危复发和 scope violation 优先于“窗口还没结束”。因为一旦确认 release 后又命中历史失败指纹，继续等窗口结束就是在放大事故。

---

## 4. pi-mono：事件驱动的 release 后监控 worker

生产版可以在 FamilyLockReleased 事件后创建 lease，然后让事件流里的 decision/outcome/backstop 信号持续喂给 worker。

~~~ts
type SignalKind =
  | "recurrence_match"
  | "scope_violation"
  | "residual_constraint_breach"
  | "dependency_drift"
  | "unexplained_decision_diff";

type PostReleaseMonitoringLease = {
  leaseId: string;
  familyLockId: string;
  releaseReceiptId: string;
  releasedScopeHash: string;
  residualConstraints: string[];
  tombstoneRefs: string[];
  recurrenceSentinelRefs: string[];
  dependencyFingerprint: string;
  expiresAt: string;
  status: "observing" | "breached" | "expired" | "archived";
};

type PostReleaseSignal = {
  signalId: string;
  leaseId: string;
  signalKind: SignalKind;
  decisionId: string;
  evidenceRefs: string[];
  severity: number;
  observedAt: string;
};

type LeaseAction =
  | "keep_observing"
  | "archive_clean"
  | "extend_monitoring"
  | "reopen_family_lock"
  | "open_reseal_case"
  | "escalate_owner";

class PostReleaseLeaseStore {
  async appendSignal(signal: PostReleaseSignal): Promise<void> {
    throw new Error("insert immutable signal");
  }

  async decideWithFence(args: {
    leaseId: string;
    expectedStatus: "observing";
    action: LeaseAction;
    reason: string;
    evidenceRefs: string[];
  }): Promise<void> {
    // 真实实现应在事务里：
    // 1. select lease for update
    // 2. 校验 status 仍是 observing
    // 3. 写 PostReleaseLeaseDecision
    // 4. 对 reopen / reseal / escalate 写 outbox
    throw new Error("implement with transaction and outbox");
  }
}

async function handlePostReleaseSignal(params: {
  lease: PostReleaseMonitoringLease;
  signal: PostReleaseSignal;
  store: PostReleaseLeaseStore;
}) {
  await params.store.appendSignal(params.signal);

  let action: LeaseAction = "keep_observing";
  let reason = "signal_recorded";

  if (params.signal.signalKind === "recurrence_match" && params.signal.severity >= 8) {
    action = "reopen_family_lock";
    reason = "high_severity_recurrence";
  }

  if (params.signal.signalKind === "scope_violation") {
    action = "reopen_family_lock";
    reason = "released_scope_violated";
  }

  if (params.signal.signalKind === "residual_constraint_breach") {
    action = "open_reseal_case";
    reason = "residual_constraint_breached";
  }

  if (action === "keep_observing") {
    return;
  }

  await params.store.decideWithFence({
    leaseId: params.lease.leaseId,
    expectedStatus: "observing",
    action,
    reason,
    evidenceRefs: params.signal.evidenceRefs,
  });
}
~~~

这里的关键是 decideWithFence。lease breach 可能由多个信号同时触发，必须用 status fence 防止同时创建多个 reopen case 或多个 reseal case。事件可以重复到达，但决策只能幂等落地一次。

---

## 5. OpenClaw 课程 Cron 实战

这门课的发布流程也适合做 PostReleaseMonitoringLease：

1. Telegram 消息发出后，用 messageId 当 releaseReceiptRef；
2. git push 成功后，用 commit sha 当 dependency fingerprint；
3. README、TOOLS、memory 都更新后，进入 observing；
4. 下一轮课程启动时先检查上一轮 lesson、README、TOOLS 是否一致；
5. 如果发现 Telegram 已发但 repo 没推，自动补齐并记录 breach；
6. 如果发现题目重复，等价于 recurrence sentinel 命中，必须换题或重开修正。

这能避免一个真实问题：外部副作用已经发生，但本地 repo、长期记忆和课程索引没有对齐。release 后监控不是为了多写日志，而是为了下一轮不会在错误状态上继续扩散。

---

## 6. 工程落地要点

- FamilyLockReleaseReceipt 之后立即创建 PostReleaseMonitoringLease；
- residual constraints 要变成可检测信号，不要只写在文档里；
- tombstone 和 recurrence sentinel 要在 release 后继续匹配真实流量；
- dependency fingerprint 漂移时先 extend monitoring，别急着 archive；
- breach 动作要预声明：reopen lock、freeze lane、open reseal case、escalate owner；
- lease 决策要用事务和 fence，保证多信号并发时只落地一次；
- clean archive 需要同时满足时间窗口、最小干净样本数和残余约束关闭。

---

## 7. 一句话总结

> 族锁释放不是把事故从系统里抹掉，而是把阻断切换成有期限的监控租约；成熟 Agent 会在 release 后继续用 residual constraints、tombstone 和 recurrence sentinel 证明恢复没有复发。
