# 452. Agent 退役护栏归档披露撤回与下游删除证明（Retired Guard Archive Disclosure Revocation & Downstream Erasure Proof）

> 关键词：disclosure revocation、downstream erasure、recipient attestation、revocation ledger、proof of deletion

上一课讲了冷归档证据对外披露时，不能直接发 archive 原件，而要生成 DeclassificationPacket、ExportManifest 和 RecipientObligationReceipt。

今天继续往后走一步：**外部披露不是导出后就结束。只要导出包过期、被撤回、发现脱密错误、接收方违反义务，Agent 就要能证明链接已失效、缓存已清理、接收方已确认删除、派生产物已处理。**

很多系统会把“撤回”做成这样：

~~~text
delete signed URL
send email: please delete
mark revoked = true
~~~

这只是愿望，不是证明。成熟 Agent 的披露撤回应该是一条证据链：

~~~text
RevocationRequest
  -> RevocationDecision
  -> AccessDisableReceipt
  -> RecipientNoticeReceipt
  -> DownstreamErasureAttestation
  -> RevocationReconciliationReceipt
~~~

一句话：**能外发，就必须能撤回；能撤回，就必须能证明撤回影响已经传播到下游。**

---

## 1. 为什么撤回也要做成工程对象

ExportManifest 里通常会有 expiresAt、revocation channel、watermark、recipient obligation。问题是：这些字段只有在撤回时被执行，才真的有价值。

常见风险：

- signed URL 已失效，但接收方本地副本还在；
- 导出包被转存到 ticket、Drive、Slack、邮件附件；
- 外部审计工具基于导出包生成了派生报告；
- 发现脱密错误后，只撤回最新链接，没有通知已下载的人；
- legal hold 要求保留证据，但 retention policy 又要求过期删除，两个策略冲突。

所以撤回不是一个 boolean，而是一个工作流：

- 阻断未来访问；
- 通知所有接收方；
- 收集删除/隔离证明；
- 对账下载日志、派生产物和 obligation；
- 对未响应或拒绝删除的接收方开 violation case。

---

## 2. 核心对象

建议至少建这六类对象：

- **RevocationRequest**：为什么撤回、撤回范围、是否紧急、触发源；
- **RevocationDecision**：撤回、到期关闭、人工复核、legal hold 暂停；
- **AccessDisableReceipt**：证明链接、token、bucket object policy、水印索引已经失效；
- **RecipientNoticeReceipt**：证明通知了谁、通过什么渠道、期限是什么；
- **DownstreamErasureAttestation**：接收方声明删除/隔离/保留原因，带签名和证据 hash；
- **RevocationReconciliationReceipt**：最终对账，决定 close_revoked、await_attestation、escalate_violation 或 reopen_export.

状态机可以这样写：

~~~text
active_disclosure
  -> revocation_requested
  -> access_disabled
  -> recipient_notified
  -> awaiting_attestation
  -> reconciled
  -> close_revoked
  -> violation_opened
~~~

注意：AccessDisableReceipt 只能证明“从我这里不能再取”，不能证明“对方已经删”。两个证明要分开。

---

## 3. learn-claude-code：撤回对账闸门

教学版先写纯函数：根据导出状态、访问日志、通知回执和接收方声明，决定撤回流程能否关闭。

~~~python
# learn_claude_code/guard_patterns/disclosure_revocation.py
from dataclasses import dataclass
from typing import Literal

Decision = Literal[
    "close_revoked",
    "await_recipient_attestation",
    "escalate_violation",
    "manual_legal_review",
    "reopen_export_incident",
]

AttestationState = Literal["missing", "deleted", "quarantined", "retained_under_hold", "refused"]


@dataclass(frozen=True)
class ExportManifest:
    export_id: str
    archive_id: str
    recipient_ids: tuple[str, ...]
    expires_at_epoch: int
    contains_sensitive_fields: bool
    watermark_id: str
    legal_hold: bool


@dataclass(frozen=True)
class AccessDisableReceipt:
    export_id: str
    signed_url_disabled: bool
    storage_policy_revoked: bool
    token_revoked: bool
    disabled_at_epoch: int


@dataclass(frozen=True)
class RecipientAttestation:
    recipient_id: str
    state: AttestationState
    signed_at_epoch: int | None
    evidence_hash: str | None
    derived_artifacts_count: int


@dataclass(frozen=True)
class DownloadObservation:
    recipient_id: str
    last_download_epoch: int | None
    downloads_after_disable: int
    watermark_seen_elsewhere: bool


@dataclass(frozen=True)
class RevocationDecision:
    decision: Decision
    reason: str
    required_actions: tuple[str, ...] = ()
    violating_recipients: tuple[str, ...] = ()


def reconcile_disclosure_revocation(
    manifest: ExportManifest,
    access: AccessDisableReceipt,
    attestations: tuple[RecipientAttestation, ...],
    observations: tuple[DownloadObservation, ...],
    now_epoch: int,
    attestation_deadline_epoch: int,
) -> RevocationDecision:
    if access.export_id != manifest.export_id:
        return RevocationDecision("reopen_export_incident", "access_receipt_export_mismatch")

    if not (
        access.signed_url_disabled
        and access.storage_policy_revoked
        and access.token_revoked
    ):
        return RevocationDecision(
            "await_recipient_attestation",
            "access_not_fully_disabled",
            ("disable_signed_url", "revoke_storage_policy", "revoke_export_token"),
        )

    observation_by_recipient = {item.recipient_id: item for item in observations}
    leaked = tuple(
        recipient_id
        for recipient_id, item in observation_by_recipient.items()
        if item.downloads_after_disable > 0 or item.watermark_seen_elsewhere
    )
    if leaked:
        return RevocationDecision(
            "escalate_violation",
            "post_revocation_access_or_watermark_leak_detected",
            ("open_disclosure_violation_case", "rotate_related_exports"),
            leaked,
        )

    attestation_by_recipient = {item.recipient_id: item for item in attestations}
    missing = tuple(
        recipient_id
        for recipient_id in manifest.recipient_ids
        if recipient_id not in attestation_by_recipient
        or attestation_by_recipient[recipient_id].state == "missing"
    )
    if missing and now_epoch < attestation_deadline_epoch:
        return RevocationDecision(
            "await_recipient_attestation",
            "recipient_attestation_pending",
            ("send_reminder",),
            missing,
        )

    if missing and now_epoch >= attestation_deadline_epoch:
        return RevocationDecision(
            "escalate_violation",
            "recipient_attestation_overdue",
            ("open_obligation_breach",),
            missing,
        )

    refused = tuple(
        item.recipient_id for item in attestations if item.state == "refused"
    )
    if refused:
        return RevocationDecision(
            "escalate_violation",
            "recipient_refused_erasure",
            ("open_obligation_breach",),
            refused,
        )

    retained = tuple(
        item.recipient_id
        for item in attestations
        if item.state == "retained_under_hold"
    )
    if retained and not manifest.legal_hold:
        return RevocationDecision(
            "manual_legal_review",
            "recipient_claimed_hold_but_manifest_has_no_legal_hold",
            ("verify_external_hold_basis",),
            retained,
        )

    derived_artifacts = sum(item.derived_artifacts_count for item in attestations)
    if manifest.contains_sensitive_fields and derived_artifacts > 0:
        return RevocationDecision(
            "manual_legal_review",
            "sensitive_export_has_downstream_derived_artifacts",
            ("review_derived_artifact_erasure_scope",),
        )

    return RevocationDecision(
        "close_revoked",
        "access_disabled_and_all_recipient_obligations_reconciled",
        ("write_revocation_reconciliation_receipt",),
    )
~~~

这里的关键点：

- 先证明自己这边已经禁用访问，再追接收方删除证明；
- 下载日志和水印发现优先级高于 attestation，因为现实观测比声明更硬；
- legal hold 不是万能理由，必须和 ExportManifest 的 hold 状态对齐。

---

## 4. pi-mono：Revocation Worker + Outbox

生产版不要让撤回 worker 同步等待外部接收方。它应该拆成事务提交、通知 outbox、异步对账三步。

~~~ts
// packages/agent-governance/src/disclosure/revocation-worker.ts
type RevocationOutcome =
  | "access_disabled"
  | "awaiting_attestation"
  | "close_revoked"
  | "violation_opened"
  | "manual_legal_review";

interface RevocationJob {
  exportId: string;
  requestedBy: string;
  reason: "expired" | "recipient_breach" | "declassification_error" | "manual";
  urgent: boolean;
}

interface DisclosureExport {
  id: string;
  archiveId: string;
  recipients: Array<{ id: string; noticeChannel: string }>;
  tokenId: string;
  objectKey: string;
  watermarkId: string;
  status: "active" | "revocation_requested" | "revoked" | "violation";
}

class DisclosureRevocationWorker {
  constructor(
    private readonly repo: DisclosureRepository,
    private readonly storage: ExportStorage,
    private readonly tokenStore: ExportTokenStore,
    private readonly outbox: Outbox,
  ) {}

  async run(job: RevocationJob): Promise<{ outcome: RevocationOutcome }> {
    return this.repo.transaction(async (tx) => {
      const exportRecord = await tx.exports.lockForUpdate(job.exportId);
      if (!exportRecord || exportRecord.status === "revoked") {
        return { outcome: "close_revoked" };
      }

      await this.storage.disableObjectAccess(exportRecord.objectKey);
      await this.tokenStore.revoke(exportRecord.tokenId);

      const accessReceipt = await tx.accessDisableReceipts.insert({
        exportId: exportRecord.id,
        objectKey: exportRecord.objectKey,
        tokenId: exportRecord.tokenId,
        disabledAt: new Date(),
        reason: job.reason,
      });

      for (const recipient of exportRecord.recipients) {
        await this.outbox.enqueue(tx, {
          type: "disclosure.revocation_notice",
          dedupeKey: exportRecord.id + ":" + recipient.id + ":revocation_notice",
          payload: {
            exportId: exportRecord.id,
            recipientId: recipient.id,
            noticeChannel: recipient.noticeChannel,
            accessReceiptId: accessReceipt.id,
            requiredAttestation: ["deleted", "quarantined", "retained_under_hold"],
          },
        });
      }

      await tx.exports.update(exportRecord.id, { status: "revocation_requested" });
      return { outcome: "access_disabled" };
    });
  }
}
~~~

这个 worker 的重点是：

- lockForUpdate 防止两个撤回任务同时改同一个 export；
- access disable 和 token revoke 先落地；
- 通知接收方用 outbox，避免事务里调用外部网络；
- dedupeKey 保证重试不会重复轰炸接收方。

后面再由一个 ReconciliationWorker 定期扫描 revocation_requested，读取 notice delivery、attestation、download log、水印扫描结果，调用上面的纯函数关闭或升级。

---

## 5. OpenClaw：cron 做撤回对账

在 OpenClaw 里，披露撤回对账很适合用 cron 跑，因为它不是一次 turn 能完成的同步动作。

建议建一个独立 job：

~~~text
每 30 分钟：
1. 找 status=revocation_requested 的 ExportManifest
2. 查 AccessDisableReceipt 是否完整
3. 查 RecipientNoticeReceipt 是否送达
4. 查 DownstreamErasureAttestation 是否齐全
5. 查下载日志和 watermark scan 有没有撤回后命中
6. 写 RevocationReconciliationReceipt
7. close 或 open violation case
~~~

这和我们现在课程 cron 的实践是同一种思路：一次任务不仅发 Telegram，还要写 lesson、更新 README、更新 TOOLS、git commit/push，最后在 memory 里写收据。Agent 长任务最怕“做了但没证据”，cron 的价值就是把证据链固定下来。

---

## 6. 实战检查清单

给自己的 Agent 系统加披露撤回时，至少问这 8 个问题：

- ExportManifest 有没有 expiresAt、watermarkId、revocationChannel？
- 撤回时是否同时禁用 signed URL、storage policy 和 export token？
- 接收方通知是否有 delivery receipt？
- 接收方删除/隔离/hold 是否有签名 attestation？
- 水印是否能发现导出包被转存或再分发？
- legal hold 是否和 retention / erasure policy 做了优先级仲裁？
- 未响应接收方是否会自动升级 violation case？
- close_revoked 前是否写了最终 ReconciliationReceipt？

---

## 7. 反模式

最危险的反模式有三个：

1. **把 revoke 当成 delete link**
   链接失效只是第一步，不代表外部副本消失。

2. **相信接收方口头确认**
   删除证明至少要有签名、时间、范围、派生产物说明和证据 hash。

3. **撤回后不对账下载和水印**
   如果撤回后还有访问或水印外泄，应该升级 breach，而不是继续等 attestation。

---

## 8. 一句话总结

**外部披露的闭环不是 export，而是 revoke + disable + notify + attest + reconcile。成熟 Agent 不只会把证据交出去，还能在需要时带证据地把披露收回来。**
