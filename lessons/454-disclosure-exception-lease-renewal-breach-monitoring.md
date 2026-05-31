# 454. Agent 披露例外租约续期与违约监控（Disclosure Exception Lease Renewal & Breach Monitoring）

> 关键词：exception lease、renewal gate、breach monitor、legal hold expiry、lease receipt

上一课讲了披露撤回遇到 legal hold 或保留争议时，不能把 manual_review 当黑洞，要生成 DisclosureDisputeCase、HoldClaimPacket、ArbitrationDecisionReceipt 和 ExceptionLease。

今天继续讲一个更容易被忽略的坑：**ExceptionLease 不是永久白名单。**

很多系统做到这里就停了：

~~~text
recipient cannot delete because legal hold
agent grants exception
case closed
~~~

这很危险。legal hold 会过期，接收方义务会变，派生物可能继续扩散，水印扫描可能发现外泄。例外租约必须像生产锁一样被续期、监控、收回和对账。

正确链路应该是：

~~~text
ExceptionLease
  -> LeaseHeartbeat
  -> RenewalGate
  -> BreachMonitor
  -> RenewalReceipt / ExpiryErasureOrder / BreachRevocationReceipt
~~~

一句话：**允许暂时不删，不代表允许永远不管。**

---

## 1. ExceptionLease 应该管什么

一个合格的披露例外租约至少要绑定这些字段：

- leaseId：租约唯一 ID；
- sourceDisputeCaseId：来自哪一次争议仲裁；
- exportId / recipientId：哪份披露、哪个接收方；
- allowedFields：允许临时保留的字段范围；
- allowedArtifacts：允许保留的派生产物类型；
- expiresAt：到期时间；
- renewalWindowStartsAt：提前多久进入续期审查；
- heartbeatEverySeconds：接收方多久要提交一次保留状态；
- breachSignals：哪些信号会触发立即撤销；
- onExpire：到期动作，通常是 order_erasure；
- onBreach：违约动作，通常是 revoke_and_open_violation。

注意两个边界：

- 续期必须重新验证 hold claim，不能复制旧理由；
- 违约监控要独立于接收方自报，必须接水印扫描、访问日志、下载日志、派生物清单等旁路信号。

---

## 2. learn-claude-code：租约续期与违约判定纯函数

教学版先写成纯函数：输入租约、最近 heartbeat、旁路观测和新的 legal hold claim，输出下一步动作。

~~~python
# learn_claude_code/guard_patterns/disclosure_exception_lease.py
from dataclasses import dataclass
from typing import Literal

LeaseAction = Literal[
    "keep_active",
    "enter_renewal_review",
    "renew_lease",
    "order_erasure_on_expiry",
    "revoke_and_open_violation",
    "request_missing_heartbeat",
]

BreachKind = Literal[
    "none",
    "field_scope_exceeded",
    "unauthorized_download",
    "watermark_leak",
    "heartbeat_missing",
    "derived_artifact_unapproved",
]


@dataclass(frozen=True)
class ExceptionLease:
    lease_id: str
    export_id: str
    recipient_id: str
    allowed_fields: tuple[str, ...]
    allowed_artifacts: tuple[str, ...]
    expires_at_epoch: int
    renewal_window_starts_at_epoch: int
    heartbeat_due_at_epoch: int
    max_extensions: int
    extensions_used: int


@dataclass(frozen=True)
class LeaseHeartbeat:
    lease_id: str
    submitted_at_epoch: int
    retained_fields: tuple[str, ...]
    retained_artifacts: tuple[str, ...]
    signed: bool
    evidence_hash: str | None


@dataclass(frozen=True)
class LeaseObservation:
    downloads_after_lease: int
    watermark_seen_elsewhere: bool
    observed_fields: tuple[str, ...]
    observed_artifacts: tuple[str, ...]


@dataclass(frozen=True)
class RenewalClaim:
    signed: bool
    legal_basis: str | None
    evidence_hash: str | None
    requested_expires_at_epoch: int | None


@dataclass(frozen=True)
class LeaseDecision:
    action: LeaseAction
    reason: str
    breach_kind: BreachKind = "none"
    next_expires_at_epoch: int | None = None
    required_actions: tuple[str, ...] = ()


def evaluate_exception_lease(
    lease: ExceptionLease,
    heartbeat: LeaseHeartbeat | None,
    observation: LeaseObservation,
    renewal_claim: RenewalClaim | None,
    now_epoch: int,
    max_renewal_seconds: int = 30 * 24 * 60 * 60,
) -> LeaseDecision:
    if observation.watermark_seen_elsewhere:
        return LeaseDecision(
            "revoke_and_open_violation",
            "watermark_detected_outside_authorized_recipient",
            "watermark_leak",
            required_actions=("disable_export_access", "freeze_related_exports", "open_violation_case"),
        )

    if observation.downloads_after_lease > 0:
        return LeaseDecision(
            "revoke_and_open_violation",
            "recipient_downloaded_after_exception_was_granted",
            "unauthorized_download",
            required_actions=("disable_export_access", "request_access_log_attestation"),
        )

    fields_over_scope = tuple(
        field for field in observation.observed_fields
        if field not in lease.allowed_fields
    )
    if fields_over_scope:
        return LeaseDecision(
            "revoke_and_open_violation",
            "retained_fields_exceed_exception_scope",
            "field_scope_exceeded",
            required_actions=("minimize_retained_packet", "open_violation_case"),
        )

    artifacts_over_scope = tuple(
        artifact for artifact in observation.observed_artifacts
        if artifact not in lease.allowed_artifacts
    )
    if artifacts_over_scope:
        return LeaseDecision(
            "revoke_and_open_violation",
            "derived_artifacts_exceed_exception_scope",
            "derived_artifact_unapproved",
            required_actions=("delete_unapproved_artifacts", "open_violation_case"),
        )

    if heartbeat is None or heartbeat.submitted_at_epoch > lease.heartbeat_due_at_epoch:
        if now_epoch > lease.heartbeat_due_at_epoch:
            return LeaseDecision(
                "request_missing_heartbeat",
                "lease_heartbeat_missing_or_late",
                "heartbeat_missing",
                required_actions=("notify_recipient", "start_grace_timer"),
            )

    if heartbeat and (not heartbeat.signed or not heartbeat.evidence_hash):
        return LeaseDecision(
            "request_missing_heartbeat",
            "lease_heartbeat_not_verifiable",
            "heartbeat_missing",
            required_actions=("request_signed_heartbeat_with_evidence_hash",),
        )

    if now_epoch >= lease.expires_at_epoch:
        return LeaseDecision(
            "order_erasure_on_expiry",
            "exception_lease_expired",
            required_actions=("send_erasure_order", "require_downstream_attestation"),
        )

    if now_epoch < lease.renewal_window_starts_at_epoch:
        return LeaseDecision("keep_active", "lease_inside_valid_window")

    if renewal_claim is None:
        return LeaseDecision(
            "enter_renewal_review",
            "renewal_window_open_but_claim_missing",
            required_actions=("request_renewal_claim",),
        )

    if lease.extensions_used >= lease.max_extensions:
        return LeaseDecision(
            "order_erasure_on_expiry",
            "maximum_extensions_reached",
            required_actions=("prepare_expiry_erasure_order",),
        )

    if (
        not renewal_claim.signed
        or not renewal_claim.legal_basis
        or not renewal_claim.evidence_hash
        or renewal_claim.requested_expires_at_epoch is None
    ):
        return LeaseDecision(
            "enter_renewal_review",
            "renewal_claim_not_verifiable",
            required_actions=("request_signed_renewal_claim",),
        )

    next_expires_at = min(
        renewal_claim.requested_expires_at_epoch,
        now_epoch + max_renewal_seconds,
    )
    return LeaseDecision(
        "renew_lease",
        "verifiable_renewal_claim_with_clean_observation",
        next_expires_at_epoch=next_expires_at,
        required_actions=("write_renewal_receipt", "schedule_next_breach_probe"),
    )
~~~

这个函数把几件事分清楚了：

- watermark leak、下载、范围超出是违约，不是续期问题；
- heartbeat 缺失先催补，但要进入 grace timer；
- 到期默认删除，不默认续期；
- 续期需要新的、可验证的 legal hold claim；
- 每次续期都要写 RenewalReceipt，不能悄悄改 expiresAt。

---

## 3. pi-mono：Lease Monitor Worker

生产版建议把租约监控做成定时 worker，不要绑在用户请求路径上。它定期扫描即将到期、心跳逾期或有旁路风险信号的租约。

~~~ts
// packages/agent-governance/src/disclosure/exception-lease-monitor.ts
type LeaseMonitorOutcome =
  | "kept_active"
  | "renewal_requested"
  | "renewed"
  | "erasure_ordered"
  | "revoked_for_breach";

interface LeaseMonitorJob {
  leaseId: string;
  probeId: string;
}

class DisclosureExceptionLeaseMonitor {
  constructor(
    private readonly repo: DisclosureRepository,
    private readonly policy: DisclosurePolicyEngine,
    private readonly outbox: Outbox,
    private readonly clock: Clock,
  ) {}

  async run(job: LeaseMonitorJob): Promise<{ outcome: LeaseMonitorOutcome }> {
    return this.repo.transaction(async (tx) => {
      const lease = await tx.exceptionLeases.getForUpdate(job.leaseId);
      if (!lease || lease.status !== "active") {
        return { outcome: "kept_active" };
      }

      const heartbeat = await tx.leaseHeartbeats.latest(lease.leaseId);
      const observation = await tx.leaseObservations.snapshot(lease.leaseId);
      const renewalClaim = await tx.renewalClaims.latestOpen(lease.leaseId);

      const decision = this.policy.evaluateExceptionLease({
        lease,
        heartbeat,
        observation,
        renewalClaim,
        now: this.clock.now(),
      });

      await tx.leaseDecisionReceipts.insert({
        leaseId: lease.leaseId,
        probeId: job.probeId,
        action: decision.action,
        reason: decision.reason,
        breachKind: decision.breachKind,
        evidenceRefs: observation.evidenceRefs,
        decidedAt: this.clock.now(),
      });

      if (decision.action === "renew_lease") {
        await tx.exceptionLeases.extend({
          leaseId: lease.leaseId,
          nextExpiresAt: decision.nextExpiresAt,
          incrementExtensionsUsed: true,
        });

        await this.outbox.publish(tx, {
          type: "disclosure.exceptionLease.renewed",
          leaseId: lease.leaseId,
          exportId: lease.exportId,
          recipientId: lease.recipientId,
        });

        return { outcome: "renewed" };
      }

      if (decision.action === "order_erasure_on_expiry") {
        await tx.exceptionLeases.markExpiring(lease.leaseId);

        await this.outbox.publish(tx, {
          type: "disclosure.erasureOrder.requested",
          leaseId: lease.leaseId,
          exportId: lease.exportId,
          recipientId: lease.recipientId,
          reason: decision.reason,
        });

        return { outcome: "erasure_ordered" };
      }

      if (decision.action === "revoke_and_open_violation") {
        await tx.exceptionLeases.revoke({
          leaseId: lease.leaseId,
          reason: decision.reason,
          breachKind: decision.breachKind,
        });

        await tx.violationCases.openFromLeaseBreach({
          leaseId: lease.leaseId,
          exportId: lease.exportId,
          recipientId: lease.recipientId,
          breachKind: decision.breachKind,
          evidenceRefs: observation.evidenceRefs,
        });

        await this.outbox.publish(tx, {
          type: "disclosure.exceptionLease.revokedForBreach",
          leaseId: lease.leaseId,
          breachKind: decision.breachKind,
        });

        return { outcome: "revoked_for_breach" };
      }

      if (decision.action === "enter_renewal_review" || decision.action === "request_missing_heartbeat") {
        await this.outbox.publish(tx, {
          type: "disclosure.exceptionLease.actionRequired",
          leaseId: lease.leaseId,
          recipientId: lease.recipientId,
          reason: decision.reason,
          requiredActions: decision.requiredActions,
        });

        return { outcome: "renewal_requested" };
      }

      return { outcome: "kept_active" };
    });
  }
}
~~~

这里有三个生产要点：

- getForUpdate 防止两个 worker 同时续同一张租约；
- 所有决策先写 LeaseDecisionReceipt，再改状态；
- 违约撤销和 violation case 必须在同一个事务里落账，避免租约已经撤销但案件没开。

---

## 4. OpenClaw：cron 的租约对账实战

OpenClaw 里这个模式很适合用 cron 跑：

~~~text
every 6 hours:
  scan active ExceptionLease
  check:
    - expiresAt 是否进入 renewal window
    - heartbeat 是否按时到
    - ExportManifest 字段范围是否被突破
    - watermark scan 是否命中外部位置
    - recipient download/access log 是否有撤回后访问
  write:
    - LeaseDecisionReceipt
    - RenewalReceipt
    - ExpiryErasureOrder
    - BreachRevocationReceipt
~~~

关键是 cron 不要只发提醒。它要能推进状态：

- 没到期、没违约：写 keep_active receipt；
- 进入续期窗口：发 renewal_claim_requested；
- claim 合格：续期并写 RenewalReceipt；
- claim 不合格：到期进入删除命令；
- 发现违约：立刻 revoke，并开 ViolationCase。

在真实系统里，OpenClaw 的课程 cron 本身也在用类似思想：每次讲课后不是只发消息，而是更新 lesson 文件、README、TOOLS、memory、Git commit 和 Telegram messageId。这样下次 cron 能从文件状态继续，不靠“我记得”。

---

## 5. 常见坑

**坑 1：把 expiresAt 当展示字段。**
到期时间必须参与状态机。过期后默认进入删除，不是 UI 上变红。

**坑 2：续期只改数据库字段。**
续期必须有 RenewalClaim、决策原因和 RenewalReceipt，否则审计时无法证明为什么延长。

**坑 3：只信接收方 heartbeat。**
heartbeat 是自报，必须和水印、访问日志、派生物扫描交叉验证。

**坑 4：违约后只发邮件。**
违约要撤销租约、冻结相关 export、开 violation case，并阻止新的 disclosure export 复用同一接收方义务。

**坑 5：无限续期。**
每张租约都要有 maxExtensions 或 sunset review，否则 legal hold 会变成永久例外。

---

## 6. 小结

ExceptionLease 的价值不是“给不能删除找个理由”，而是把现实里的临时保留变成可审计的工程状态。

今天的模式可以记成四句话：

- 例外必须有范围；
- 范围必须被监控；
- 到期必须重新证明；
- 违约必须自动收回。

成熟 Agent 不怕现实世界有例外，怕的是例外没有租约、没有心跳、没有续期闸门、没有违约收回。
