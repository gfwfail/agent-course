# 451. Agent 退役护栏归档脱密导出与接收方义务收据（Retired Guard Archive Disclosure Export & Recipient Obligation Receipt）

> 关键词：declassified export、recipient obligation、disclosure lease、revocation receipt、external audit

上一课讲的是冷归档证据被内部 worker 取回后，如何用 UsageReconciliationReceipt 和 CachePurgeReceipt 证明“只按声明用途使用，并且过期后清干净”。

今天往外再走一步：**当冷归档证据要给外部审计、客户支持、法务、合作方或监管检查时，不能直接把 archive bundle 发出去，而要先生成脱密导出包，并让接收方义务也变成可审计收据。**

很多 Agent 系统内部证据链做得很漂亮，但一到外部披露就退化成：

~~~text
zip archive -> email/slack/link -> hope for the best
~~~

这不够。成熟 Agent 的冷证据对外披露应该是：

~~~text
DisclosureRequest
  -> DisclosureApproval
  -> DeclassificationPacket
  -> ExportManifest
  -> RecipientObligationReceipt
  -> RevocationOrExpiryReceipt
~~~

一句话：**内部可审计，不代表可以外发；外发的不是原始证据，而是带用途、字段、期限、接收方义务和撤回路径的证明包。**

---

## 1. 为什么要单独做披露导出

ArchiveAccessLease 解决内部读取，UsageReconciliation 解决内部使用。但外部披露有新的风险面：

- 接收方不在你的运行时权限系统里；
- 导出文件可能离开原来的审计边界；
- 外部只需要证明结论，不一定需要 raw payload；
- 法务 hold 可能禁止删除，但不等于允许扩大披露；
- 导出后如果发现脱密错误，需要可撤回、可通知、可重新签发。

典型场景：

- 客户问：“你们为什么阻止了我的自动化任务？”只需要决策摘要和规则版本，不需要其他用户样本；
- 审计问：“这个护栏退役是否合规？”只需要 sunset evidence、retirement receipt、hash chain；
- 法务问：“某次事故是否包含个人数据？”只需要字段级分类和访问日志，不一定需要原文；
- 合作方问：“我们 SDK 更新是否导致 guard drift？”只需要 dependency fingerprint diff 和 replay result。

如果直接导出 RetirementArchive，等于把内部证据、历史上下文、其他租户线索一起打包送出去。

所以要引入 **DeclassificationPacket**：它不是 archive 副本，而是从 archive 派生出来的最小披露证明。

---

## 2. 核心对象

建议把对外披露拆成五个对象：

- **DisclosureRequest**：谁请求、为什么请求、接收方是谁、需要证明什么；
- **DisclosureApproval**：由 policy/owner/legal gate 生成，限定允许导出的字段、脱密级别、有效期；
- **DeclassificationPacket**：把 raw evidence 投影成 hash、summary、redacted excerpt、proof view；
- **ExportManifest**：记录导出文件、字段、hash、签名、watermark、expiresAt、revocation channel；
- **RecipientObligationReceipt**：接收方确认用途、保存期限、禁止再分发、撤回响应方式；
- **RevocationOrExpiryReceipt**：导出到期或撤回时，证明通知、链接失效、后续访问被阻断。

状态机可以这样设计：

~~~text
requested
  -> approved
  -> declassified
  -> exported
  -> recipient_acknowledged
  -> active_disclosure
  -> expired
  -> revoked
  -> violation_opened
~~~

注意：approval 不是 export。approval 只是允许生成某种脱密视图；真正外发还要写 ExportManifest。

---

## 3. learn-claude-code：脱密导出闸门

教学版先写纯函数：根据 request、approval、archive metadata 和接收方状态，决定能不能导出、导出什么、是否需要人工复核。

~~~python
# learn_claude_code/guard_patterns/archive_disclosure_export.py
from dataclasses import dataclass
from typing import Literal

Decision = Literal[
    "export_declassified_packet",
    "require_manual_review",
    "deny_disclosure",
    "revoke_existing_export",
]

DisclosureLevel = Literal["hash_only", "summary", "redacted_excerpt", "raw"]


@dataclass(frozen=True)
class DisclosureRequest:
    request_id: str
    archive_id: str
    requester: str
    recipient_id: str
    purpose: str
    requested_fields: tuple[str, ...]
    requested_level: DisclosureLevel
    external_destination: str


@dataclass(frozen=True)
class DisclosureApproval:
    approval_id: str
    request_id: str
    allowed_fields: tuple[str, ...]
    max_level: DisclosureLevel
    expires_at_epoch: int
    legal_hold: bool
    allow_external_export: bool
    require_recipient_ack: bool


@dataclass(frozen=True)
class ArchiveDisclosureProfile:
    archive_id: str
    field_classes: dict[str, str]
    contains_cross_tenant_data: bool
    contains_secret_material: bool
    integrity_hash: str


@dataclass(frozen=True)
class RecipientState:
    recipient_id: str
    has_signed_obligations: bool
    active_exports: int
    last_violation_epoch: int | None


@dataclass(frozen=True)
class DisclosureDecision:
    decision: Decision
    reason: str
    export_fields: tuple[str, ...] = ()
    export_level: DisclosureLevel = "hash_only"
    obligations: tuple[str, ...] = ()


LEVEL_ORDER: dict[DisclosureLevel, int] = {
    "hash_only": 0,
    "summary": 1,
    "redacted_excerpt": 2,
    "raw": 3,
}


def decide_archive_disclosure(
    request: DisclosureRequest,
    approval: DisclosureApproval,
    profile: ArchiveDisclosureProfile,
    recipient: RecipientState,
    now_epoch: int,
) -> DisclosureDecision:
    if approval.request_id != request.request_id:
        return DisclosureDecision("deny_disclosure", "approval_does_not_match_request")

    if not approval.allow_external_export:
        return DisclosureDecision("deny_disclosure", "external_export_not_allowed")

    if now_epoch >= approval.expires_at_epoch:
        return DisclosureDecision("deny_disclosure", "approval_expired")

    if approval.require_recipient_ack and not recipient.has_signed_obligations:
        return DisclosureDecision(
            "require_manual_review",
            "recipient_obligation_ack_missing",
            obligations=("collect_recipient_obligation_receipt",),
        )

    if recipient.last_violation_epoch is not None:
        return DisclosureDecision(
            "require_manual_review",
            "recipient_has_prior_disclosure_violation",
        )

    requested_fields = set(request.requested_fields)
    allowed_fields = set(approval.allowed_fields)
    if requested_fields - allowed_fields:
        return DisclosureDecision("deny_disclosure", "requested_fields_exceed_approval")

    if LEVEL_ORDER[request.requested_level] > LEVEL_ORDER[approval.max_level]:
        return DisclosureDecision("deny_disclosure", "requested_level_exceeds_approval")

    if profile.contains_secret_material and request.requested_level != "hash_only":
        return DisclosureDecision(
            "require_manual_review",
            "secret_material_requires_hash_only_or_legal_review",
        )

    sensitive_fields = {
        field
        for field in request.requested_fields
        if profile.field_classes.get(field) in {"sensitive", "secret"}
    }
    if sensitive_fields and request.requested_level in {"redacted_excerpt", "raw"}:
        return DisclosureDecision(
            "require_manual_review",
            "sensitive_fields_need_extra_declassification_review",
            export_fields=tuple(sorted(requested_fields - sensitive_fields)),
            export_level="summary",
        )

    if profile.contains_cross_tenant_data and request.requested_level == "raw":
        return DisclosureDecision("deny_disclosure", "raw_cross_tenant_export_forbidden")

    return DisclosureDecision(
        "export_declassified_packet",
        "request_within_approval_and_recipient_obligations",
        export_fields=tuple(request.requested_fields),
        export_level=request.requested_level,
        obligations=(
            "write_declassification_packet",
            "write_export_manifest",
            "bind_revocation_channel",
        ),
    )
~~~

这个函数有两个重点：

- 字段范围和披露级别分开判定：能看 summary，不等于能看 raw；
- 接收方义务是导出前条件：没有 obligation receipt，就不能把外部披露当成普通下载。

---

## 4. pi-mono：DisclosureExportWorker

生产版要把导出变成事务型 worker：先生成脱密包，再签 Manifest，最后发出外部链接或附件。

~~~ts
type DeclassificationPacket = {
  packetId: string;
  archiveId: string;
  requestId: string;
  fields: string[];
  level: "hash_only" | "summary" | "redacted_excerpt" | "raw";
  proofHash: string;
  redactionProfile: string;
  createdAt: number;
};

type ExportManifest = {
  exportId: string;
  packetId: string;
  recipientId: string;
  destination: string;
  contentHash: string;
  signature: string;
  watermark: string;
  expiresAt: number;
  revocationEndpoint: string;
};

type RecipientObligationReceipt = {
  receiptId: string;
  recipientId: string;
  exportId: string;
  acceptedPurpose: string;
  retentionUntil: number;
  noRedistribution: boolean;
  revocationResponseSlaMs: number;
  acceptedAt: number;
};

class DisclosureExportWorker {
  async export(requestId: string): Promise<ExportManifest> {
    const request = await loadDisclosureRequest(requestId);
    const approval = await loadDisclosureApproval(requestId);
    const profile = await loadArchiveDisclosureProfile(request.archiveId);
    const recipient = await loadRecipientState(request.recipientId);

    const decision = decideArchiveDisclosure(
      request,
      approval,
      profile,
      recipient,
      Date.now(),
    );

    await appendDisclosureLedger({
      type: "DisclosureDecisionRecorded",
      requestId,
      decision,
    });

    if (decision.decision === "deny_disclosure") {
      throw new Error("Disclosure denied: " + decision.reason);
    }

    if (decision.decision === "require_manual_review") {
      await openDisclosureReviewCase({
        requestId,
        reason: decision.reason,
        obligations: decision.obligations,
      });
      throw new Error("Disclosure requires review: " + decision.reason);
    }

    const obligation = await requireRecipientObligationReceipt({
      recipientId: request.recipientId,
      purpose: request.purpose,
      retentionUntil: approval.expiresAt,
    });

    const packet = await buildDeclassificationPacket({
      archiveId: request.archiveId,
      requestId,
      fields: decision.exportFields,
      level: decision.exportLevel,
    });

    const manifest: ExportManifest = await writeSignedExportManifest({
      packet,
      recipientId: request.recipientId,
      destination: request.externalDestination,
      expiresAt: approval.expiresAt,
      obligationReceiptId: obligation.receiptId,
    });

    await appendDisclosureLedger({
      type: "DisclosureExportCompleted",
      requestId,
      packet,
      manifest,
      obligation,
    });

    return manifest;
  }

  async revoke(exportId: string, reason: string): Promise<void> {
    const manifest = await loadExportManifest(exportId);

    await disableExportAccess(manifest.exportId);
    await notifyRecipientRevocation({
      recipientId: manifest.recipientId,
      exportId,
      reason,
      revocationEndpoint: manifest.revocationEndpoint,
    });

    await appendDisclosureLedger({
      type: "DisclosureExportRevoked",
      exportId,
      reason,
      revokedAt: Date.now(),
    });
  }
}
~~~

生产落地时要坚持三个原则：

- ExportManifest 必须签名，接收方拿到的包也要能验证 hash；
- watermark 要绑定 recipient/exportId，方便追踪二次扩散；
- revoke 不能只是删除本地文件，还要通知接收方，并写 RevocationReceipt。

---

## 5. OpenClaw 课程 Cron 的例子

这门课的 cron 当前会把 lesson 发到 Telegram 群，并把文件推到 GitHub。它不是冷归档披露系统，但可以用来理解披露边界。

如果以后课程系统支持“导出历史事故/护栏证据给外部审计”，合理流程应该是：

1. 审计方提交 DisclosureRequest，只问“第 447-450 课涉及的退役护栏证据链是否完整”；
2. worker 读取冷归档，但 DeclassificationPacket 只包含 lesson slug、commit hash、messageId、receipt 类型，不包含 workspace raw memory；
3. ExportManifest 记录导出的 markdown/pdf/hash、接收方、有效期、撤回地址；
4. 接收方签 RecipientObligationReceipt，承诺只用于审计、不再分发、到期删除；
5. 如果后续发现某课误带了敏感上下文，系统按 exportId revoke，通知接收方，重新签发更小的 proof view；
6. 下一轮 cron 只把“已导出 hash/proof”纳入对账，不把外部接收方信息写进公开 README。

这就是从“发文件”升级成“披露证明系统”：外部看到足够证明的东西，但看不到不该看的上下文。

---

## 6. 常见坑

**坑 1：把 legal hold 当成可以随便导出。**

Legal hold 通常是禁止删除或要求保全，不是扩大披露权限。保全和外发是两套 gate。

**坑 2：只做字段脱敏，不做用途绑定。**

同一个 redacted summary 给客户解释和给模型训练，风险完全不同。ExportManifest 要绑定 purpose。

**坑 3：导出包没有签名和 watermark。**

没有签名就无法证明内容没被改；没有 watermark 就很难追踪二次扩散来源。

**坑 4：接收方义务只放在合同里，不进系统收据。**

Agent worker 只认系统状态。RecipientObligationReceipt 不存在，导出流程就应该卡住。

**坑 5：撤回只删下载链接，不通知接收方。**

外部披露已经离开系统边界，撤回必须有通知、确认、过期访问阻断和后续复核。

---

## 7. 记忆点

今天这课一句话：

**冷归档对外披露时，导出的不应该是 archive 原件，而是带字段范围、脱密级别、签名、水印、有效期、接收方义务和撤回路径的 DeclassificationPacket。**

工程上记住这条链：

~~~text
DisclosureRequest
  -> DisclosureApproval
  -> DeclassificationPacket
  -> ExportManifest
  -> RecipientObligationReceipt
  -> RevocationOrExpiryReceipt
~~~

成熟 Agent 的外部披露，不是把证据打包发出去，而是能证明“为什么可以发、发了哪些、谁接收、能用多久、怎么撤回、有没有越界”。
