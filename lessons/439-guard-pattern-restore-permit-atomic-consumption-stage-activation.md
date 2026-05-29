# 439. Agent 护栏恢复许可的原子消费与阶段激活收据

> **核心思想：RestorePermit 不是“已经恢复”，只是允许一次具体恢复动作进入执行闸门。成熟 Agent 要在发布 worker 里原子消费 permit、抢占 lease、校验 stage/scope/fingerprint，并写 StageActivationReceipt，证明恢复动作只发生了一次、发生在正确阶段、能被观察和回滚。**

---

## 1. 为什么有 RestorePermit 还不能直接恢复

上一课讲 UnlockReviewPacket 和 UnlockDecisionReceipt：复核通过后，只生成一次性 RestorePermit。

这里最容易犯的错误是：worker 拿到 permitId 后，直接把 family lock 改成 restore 或 active。

这会制造四类生产事故：

1. 队列重试导致同一张 permit 被消费两次，shadow/canary 被重复开启；
2. permit 允许 shadow，但 worker 跳到了 canary 或 active；
3. permit 生成后依赖、runtime fingerprint 或 release lane 已漂移；
4. 激活动作成功了，但没有 receipt，后续观察、回滚、审计都找不到边界。

所以 RestorePermit 的正确位置不是“状态字段”，而是“执行许可”。它必须被一个原子事务消费，并产出 StageActivationReceipt。

可以把链路拆成五步：

- UnlockDecisionReceipt：复核结论；
- RestorePermit：一次性恢复许可；
- ActivationLease：这次执行抢到的租约和 fence token；
- StageActivationReceipt：阶段激活收据；
- ObservationWindow：激活后要观察什么、超限怎么回滚。

---

## 2. 最小数据模型

~~~text
RestorePermit:
  permitId
  familyLockId
  candidateFamilyId
  candidateVersion
  allowedStage: shadow | canary | active
  allowedScope:
    tenants[]
    riskSurfaces[]
    actionClasses[]
    maxDecisionCount
  dependencyFingerprint
  releaseLaneId
  expiresAt
  consumedAt?
  consumedByRunId?

ActivationLease:
  leaseId
  permitId
  workerRunId
  fenceToken
  expiresAt

StageActivationReceipt:
  receiptId
  permitId
  leaseId
  activatedStage
  activatedScope
  candidateVersion
  runtimeFingerprint
  releaseLaneId
  observationWindowId
  rollbackPointRef
  decision: activated | rejected | expired | drift_blocked | already_consumed
  reason
~~~

关键点：StageActivationReceipt 是恢复动作的事实边界。后面发生 bad diff、scope violation、runtime drift 或用户投诉，都要能回到这张 receipt，知道是谁、何时、按什么 scope 激活的。

---

## 3. 激活前必须检查什么

第一，检查 permit 是否仍然有效。

expiresAt、consumedAt、familyLockId、candidateVersion 都要匹配。permit 过期不能“宽限一下”，只能重新走 unlock review 或刷新复核证据。

第二，检查阶段不能升级。

permit.allowedStage = shadow 时，worker 只能激活 shadow。允许 canary 不等于允许 active。阶段升级必须由下一张 decision receipt 和下一张 permit 承载。

第三，检查 scope 不能变宽。

tenant、riskSurface、actionClass、maxDecisionCount 都必须是 permit.allowedScope 的子集。worker 可以更保守，不能更激进。

第四，检查依赖和 release lane 没漂移。

如果 dependencyFingerprint 或 releaseLaneId 变了，说明复核时看到的世界和执行时不一致。正确结果是 drift_blocked，并要求重建 review packet。

第五，创建 rollback point 和 observation window。

激活不是终点。StageActivationReceipt 里必须绑定 rollbackPointRef 和 observationWindowId，否则后续发现问题时只能人工猜怎么退。

---

## 4. learn-claude-code：纯函数激活闸门

教学版先写成纯函数。它不操作数据库，只判断某次 worker run 能不能消费 permit 并进入指定阶段。

~~~py
from dataclasses import dataclass
from typing import Literal


Stage = Literal["shadow", "canary", "active"]
Decision = Literal[
    "activated",
    "rejected",
    "expired",
    "drift_blocked",
    "already_consumed",
]


@dataclass(frozen=True)
class Scope:
    tenants: set[str]
    risk_surfaces: set[str]
    action_classes: set[str]
    max_decision_count: int


@dataclass(frozen=True)
class RestorePermit:
    permit_id: str
    family_lock_id: str
    candidate_version: str
    allowed_stage: Stage
    allowed_scope: Scope
    dependency_fingerprint: str
    release_lane_id: str
    expires_at_epoch: int
    consumed: bool


@dataclass(frozen=True)
class ActivationRequest:
    permit_id: str
    family_lock_id: str
    candidate_version: str
    requested_stage: Stage
    requested_scope: Scope
    dependency_fingerprint: str
    release_lane_id: str
    now_epoch: int


@dataclass(frozen=True)
class ActivationGateResult:
    decision: Decision
    reason: str
    activated_stage: Stage | None
    residual_risk: str


STAGE_ORDER = {"shadow": 1, "canary": 2, "active": 3}


def is_scope_subset(requested: Scope, allowed: Scope) -> bool:
    return (
        requested.tenants <= allowed.tenants
        and requested.risk_surfaces <= allowed.risk_surfaces
        and requested.action_classes <= allowed.action_classes
        and requested.max_decision_count <= allowed.max_decision_count
    )


def evaluate_activation(
    permit: RestorePermit,
    request: ActivationRequest,
) -> ActivationGateResult:
    if permit.permit_id != request.permit_id:
        return ActivationGateResult("rejected", "permit_id_mismatch", None, "high")

    if permit.consumed:
        return ActivationGateResult("already_consumed", "permit_already_used", None, "medium")

    if permit.expires_at_epoch <= request.now_epoch:
        return ActivationGateResult("expired", "permit_expired", None, "medium")

    if permit.family_lock_id != request.family_lock_id:
        return ActivationGateResult("rejected", "family_lock_mismatch", None, "high")

    if permit.candidate_version != request.candidate_version:
        return ActivationGateResult("rejected", "candidate_version_mismatch", None, "high")

    if STAGE_ORDER[request.requested_stage] > STAGE_ORDER[permit.allowed_stage]:
        return ActivationGateResult("rejected", "requested_stage_exceeds_permit", None, "high")

    if not is_scope_subset(request.requested_scope, permit.allowed_scope):
        return ActivationGateResult("rejected", "requested_scope_exceeds_permit", None, "high")

    if permit.dependency_fingerprint != request.dependency_fingerprint:
        return ActivationGateResult("drift_blocked", "dependency_fingerprint_changed", None, "high")

    if permit.release_lane_id != request.release_lane_id:
        return ActivationGateResult("drift_blocked", "release_lane_changed", None, "high")

    return ActivationGateResult(
        "activated",
        "permit_activation_allowed",
        request.requested_stage,
        "low",
    )
~~~

这段代码的重点是：permit 可以降级使用，不能升级使用。允许 canary 的 permit 可以只开 shadow；允许 shadow 的 permit 绝不能开 canary。

---

## 5. pi-mono：事务里消费 permit、写激活收据

生产系统里要把 permit consumption、lease、stage pointer、receipt 放在同一个事务里。否则 worker 崩在中间，会留下“看起来没消费，但其实已经激活”的半状态。

~~~ts
type Stage = "shadow" | "canary" | "active";

type RestorePermit = {
  permitId: string;
  familyLockId: string;
  candidateVersion: string;
  allowedStage: Stage;
  allowedScope: {
    tenants: string[];
    riskSurfaces: string[];
    actionClasses: string[];
    maxDecisionCount: number;
  };
  dependencyFingerprint: string;
  releaseLaneId: string;
  expiresAt: Date;
  consumedAt?: Date;
};

type ActivationInput = {
  permitId: string;
  requestedStage: Stage;
  requestedScope: RestorePermit["allowedScope"];
  dependencyFingerprint: string;
  releaseLaneId: string;
  workerRunId: string;
};

class RestoreActivationWorker {
  constructor(
    private db: Database,
    private runtimeProbe: RuntimeProbe,
    private clock: Clock,
  ) {}

  async activate(input: ActivationInput) {
    return this.db.transaction(async (tx) => {
      const permit = await tx.restorePermit.findForUpdate(input.permitId);
      if (!permit) throw new Error("permit_not_found");
      if (permit.consumedAt) throw new Error("permit_already_consumed");
      if (permit.expiresAt <= this.clock.now()) throw new Error("permit_expired");

      assertStageNotEscalated(input.requestedStage, permit.allowedStage);
      assertScopeSubset(input.requestedScope, permit.allowedScope);

      if (permit.dependencyFingerprint !== input.dependencyFingerprint) {
        return tx.stageActivationReceipt.create({
          decision: "drift_blocked",
          reason: "dependency_fingerprint_changed",
          permitId: permit.permitId,
          workerRunId: input.workerRunId,
        });
      }

      if (permit.releaseLaneId !== input.releaseLaneId) {
        return tx.stageActivationReceipt.create({
          decision: "drift_blocked",
          reason: "release_lane_changed",
          permitId: permit.permitId,
          workerRunId: input.workerRunId,
        });
      }

      const lease = await tx.activationLease.create({
        permitId: permit.permitId,
        workerRunId: input.workerRunId,
        fenceToken: await tx.fenceToken.next(`restore:${permit.familyLockId}`),
        expiresAt: addMinutes(this.clock.now(), 15),
      });

      const rollbackPoint = await tx.rollbackPoint.create({
        familyLockId: permit.familyLockId,
        candidateVersion: permit.candidateVersion,
        releaseLaneId: permit.releaseLaneId,
      });

      await tx.stagePointer.upsert({
        familyLockId: permit.familyLockId,
        stage: input.requestedStage,
        candidateVersion: permit.candidateVersion,
        scope: input.requestedScope,
        fenceToken: lease.fenceToken,
      });

      const runtime = await this.runtimeProbe.captureFingerprint(tx, {
        familyLockId: permit.familyLockId,
        stage: input.requestedStage,
      });

      const observationWindow = await tx.observationWindow.create({
        permitId: permit.permitId,
        stage: input.requestedStage,
        maxDecisionCount: input.requestedScope.maxDecisionCount,
        rollbackPointId: rollbackPoint.rollbackPointId,
        startsAt: this.clock.now(),
      });

      await tx.restorePermit.update({
        permitId: permit.permitId,
        consumedAt: this.clock.now(),
        consumedByRunId: input.workerRunId,
      });

      return tx.stageActivationReceipt.create({
        decision: "activated",
        reason: "permit_consumed_and_stage_pointer_updated",
        permitId: permit.permitId,
        leaseId: lease.leaseId,
        activatedStage: input.requestedStage,
        activatedScope: input.requestedScope,
        candidateVersion: permit.candidateVersion,
        runtimeFingerprint: runtime.fingerprint,
        releaseLaneId: permit.releaseLaneId,
        rollbackPointRef: rollbackPoint.rollbackPointId,
        observationWindowId: observationWindow.windowId,
      });
    });
  }
}
~~~

几个实现细节：

- findForUpdate 是必须的，避免两个 worker 同时消费同一张 permit；
- fenceToken 要写进 stage pointer，防止旧 worker 延迟写覆盖新状态；
- drift_blocked 也要写 receipt，因为它解释了为什么 permit 没被消费；
- rollback point 必须在 stage pointer 更新前创建；
- consumedAt 最好在事务最后写，和 receipt 一起提交。

---

## 6. OpenClaw 实战：课程 Cron 也有同类边界

这套模式不只适用于护栏发布。OpenClaw 课程 Cron 也有类似链路：

1. 选题去重通过，相当于生成一次 publish permit；
2. 写 lesson、README、TOOLS，相当于准备 activation scope；
3. 发送 Telegram、git commit、git push，是外部副作用；
4. memory 记录 messageId 和 commit，是 activation receipt；
5. 下次 cron 读取 TOOLS/memory，避免重复消费同一主题。

如果中间失败，比如 Telegram 发了但 git push 失败，不能假装没发生。正确做法是记录 partial activation receipt，下次重试只补 git side effect，不重新发同一课。

这就是 StageActivationReceipt 的价值：它让“到底执行到了哪一步”变成可恢复状态，而不是靠 agent 记忆。

---

## 7. 常见坑

把 permit 当状态。

permit 是执行许可，不是系统状态。状态要由 stage pointer 和 activation receipt 表达。

队列重试不幂等。

worker 重试时必须先查 permit.consumedAt 和 existing receipt。已经 activated 的 permit 应该返回同一张 receipt，而不是再执行一次。

阶段偷跑。

shadow permit 不能开 canary；canary permit 不能直接改 active。每次晋级都要有新 receipt。

只记录 success，不记录 blocked。

expired、drift_blocked、already_consumed 都要写 receipt。失败收据是未来排查和重新复核的输入。

没有 rollback point。

恢复动作没有 rollback point，就不是工程发布，只是状态突变。

---

## 8. 一句话总结

> **RestorePermit 只说明“这一次可以尝试恢复”，StageActivationReceipt 才证明“这一次恢复实际发生在正确阶段、正确范围、正确运行时，并且能观察、能回滚、不会重复执行”。**
