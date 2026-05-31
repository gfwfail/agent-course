# 449. Agent 退役护栏归档访问租约与 Legal Hold 闸门（Retired Guard Archive Access Lease & Legal Hold Gate）

> 关键词：archive access lease、legal hold、purpose binding、redaction profile、cold evidence retrieval

上一课讲了 Retired Guard Archive 的腐化检测：冷归档还在，不代表它还能审计；要定期校验 byte、schema、policy、lineage、replay。

今天继续补一个更容易在生产里踩坑的点：**冷归档能取回，不代表任何 worker 都能直接读。**

退役护栏归档里通常有这些东西：

- 当初为什么退役的 evidence packet；
- 事故样本、回归种子、反事实结果；
- owner 决策、人工批准、外部副作用收据；
- 可能包含用户内容、工具参数、聊天上下文、日志片段；
- replacement guard、tombstone、reopen case 的 lineage。

这些证据对审计很有价值，但也有隐私、合规、保留期和权限风险。

成熟 Agent 的规则应该是：

> 冷证据不是文件系统里的普通文件，而是一次带目的、范围、时限、脱敏策略和法律保全状态的访问租约。

---

## 1. 为什么不能直接读 archive

很多系统会把归档做成对象存储路径：

~~~text
s3://agent-evidence/retired-guards/archive-123.json
~~~

然后内部 worker 想查就查。这个方案短期方便，长期会出三个问题。

第一，**purpose 丢失**。同一份证据用于安全复盘、产品分析、训练样本、客服解释，合法边界完全不同。没有 purpose binding，后续审计只能看到“谁读了文件”，看不到“为什么读”。

第二，**脱敏策略漂移**。当初 archive 里允许保存 raw payload，不代表今天仍允许给某个 worker 读 raw。访问时必须按当前 policy 选择 redaction profile。

第三，**legal hold 和 retention 冲突**。归档到期本应销毁，但如果有关联事故、账单争议、合规调查或客户申诉，就要进入 legal hold，禁止销毁；反过来，没有 hold 时也不能因为“以后可能有用”无限保存。

所以需要一个 **ArchiveAccessLease**，把“取回冷证据”变成可审计的状态机。

---

## 2. 状态机：访问前先发租约

建议状态机：

~~~text
request_access
  -> policy_check
  -> legal_hold_check
  -> lease_issued
  -> evidence_retrieved
  -> usage_reconciled
  -> lease_expired
  -> access_denied
~~~

核心约束：

- request_access 必须带 purpose、archiveId、requestedFields、actor、caseRef；
- policy_check 决定 redaction profile，不是 caller 自己选 raw/summary；
- legal_hold_check 先判断 archive 是否禁止销毁、是否禁止某些用途读取；
- lease_issued 必须有 expiresAt、maxReads、fieldScope、redactionProfile、auditSink；
- evidence_retrieved 必须写 RetrievalReceipt，记录 hash、字段范围、脱敏版本；
- usage_reconciled 检查 lease 下游是否只用于声明目的；
- lease_expired 后缓存要清理，不能把冷证据偷偷变成热缓存。

一句话：访问证据也要像调用外部副作用一样，有 permit、有收据、有对账。

---

## 3. learn-claude-code：访问租约闸门纯函数

教学版先写一个纯函数：输入访问请求、归档元数据、当前策略，输出 issue lease / deny / legal hold / manual review。

~~~python
# learn_claude_code/guard_patterns/archive_access_lease.py
from dataclasses import dataclass
from typing import Literal

Purpose = Literal[
    "security_audit",
    "reopen_case",
    "regression_replay",
    "customer_support",
    "model_training",
    "debugging",
]

Decision = Literal[
    "issue_lease",
    "deny_access",
    "require_legal_hold",
    "manual_review",
]


@dataclass(frozen=True)
class ArchiveAccessRequest:
    request_id: str
    archive_id: str
    actor_role: str
    purpose: Purpose
    requested_fields: tuple[str, ...]
    case_ref: str | None
    now_epoch: int


@dataclass(frozen=True)
class RetiredArchiveMeta:
    archive_id: str
    retained_until_epoch: int
    legal_hold: bool
    contains_pii: bool
    contains_external_payload: bool
    allowed_purposes: tuple[Purpose, ...]
    minimum_redaction_profile: str
    max_read_count: int


@dataclass(frozen=True)
class AccessPolicy:
    allowed_roles: tuple[str, ...]
    raw_allowed_roles: tuple[str, ...]
    training_allowed: bool
    support_requires_case_ref: bool
    legal_hold_required_after_retention: bool


@dataclass(frozen=True)
class ArchiveAccessDecision:
    decision: Decision
    reason: str
    field_scope: tuple[str, ...] = ()
    redaction_profile: str | None = None
    expires_in_seconds: int = 0
    obligations: tuple[str, ...] = ()


def decide_archive_access(
    request: ArchiveAccessRequest,
    archive: RetiredArchiveMeta,
    policy: AccessPolicy,
) -> ArchiveAccessDecision:
    if request.actor_role not in policy.allowed_roles:
        return ArchiveAccessDecision("deny_access", "actor_role_not_allowed")

    if request.purpose not in archive.allowed_purposes:
        return ArchiveAccessDecision("deny_access", "purpose_not_allowed_for_archive")

    if request.now_epoch > archive.retained_until_epoch and not archive.legal_hold:
        if policy.legal_hold_required_after_retention:
            return ArchiveAccessDecision(
                "require_legal_hold",
                "retention_expired_without_legal_hold",
                obligations=("open_hold_or_disposal_review",),
            )
        return ArchiveAccessDecision("deny_access", "archive_retention_expired")

    if request.purpose == "model_training" and not policy.training_allowed:
        return ArchiveAccessDecision("deny_access", "training_use_forbidden")

    if request.purpose == "customer_support" and policy.support_requires_case_ref:
        if request.case_ref is None:
            return ArchiveAccessDecision("manual_review", "support_access_missing_case_ref")

    wants_raw = "raw_payload" in request.requested_fields or "full_transcript" in request.requested_fields
    if wants_raw and request.actor_role not in policy.raw_allowed_roles:
        redaction_profile = "pii_stripped"
        field_scope = tuple(
            field
            for field in request.requested_fields
            if field not in ("raw_payload", "full_transcript")
        )
    else:
        redaction_profile = archive.minimum_redaction_profile
        field_scope = request.requested_fields

    if archive.contains_pii and redaction_profile == "raw":
        return ArchiveAccessDecision(
            "manual_review",
            "raw_pii_requires_human_approval",
            field_scope=field_scope,
            redaction_profile=redaction_profile,
        )

    return ArchiveAccessDecision(
        "issue_lease",
        "access_policy_legal_hold_and_redaction_ok",
        field_scope=field_scope,
        redaction_profile=redaction_profile,
        expires_in_seconds=900,
        obligations=(
            "write_retrieval_receipt",
            "bind_usage_to_purpose",
            "purge_cached_evidence_on_expiry",
        ),
    )
~~~

这个函数的重点不是权限字符串，而是四个边界同时成立：

- actor 可以读；
- purpose 合法；
- retention / legal hold 状态允许；
- 字段范围和脱敏策略符合当前 policy。

---

## 4. pi-mono：ArchiveAccessLeaseWorker

pi-mono 里 Agent 有事件订阅模型，外部系统可以 subscribe agent event，把 tool call、message、result 变成事件流。冷归档访问也应该走类似的事件流：申请、发租约、读取、对账，全都进 ledger。

~~~ts
type ArchiveAccessPurpose =
  | "security_audit"
  | "reopen_case"
  | "regression_replay"
  | "customer_support"
  | "model_training"
  | "debugging";

type ArchiveAccessRequest = {
  requestId: string;
  archiveId: string;
  actorId: string;
  actorRole: string;
  purpose: ArchiveAccessPurpose;
  requestedFields: string[];
  caseRef?: string;
  requestedAt: number;
};

type ArchiveAccessLease = {
  leaseId: string;
  requestId: string;
  archiveId: string;
  fieldScope: string[];
  redactionProfile: "summary_only" | "pii_stripped" | "hash_only" | "raw";
  purpose: ArchiveAccessPurpose;
  maxReads: number;
  expiresAt: number;
  auditSink: string;
};

type RetrievalReceipt = {
  receiptId: string;
  leaseId: string;
  archiveId: string;
  evidenceHash: string;
  redactionProfile: ArchiveAccessLease["redactionProfile"];
  fieldScope: string[];
  readCountAfter: number;
  retrievedAt: number;
};

class ArchiveAccessLeaseWorker {
  async requestLease(request: ArchiveAccessRequest): Promise<ArchiveAccessLease> {
    const archive = await loadRetiredArchiveMeta(request.archiveId);
    const policy = await loadCurrentEvidenceAccessPolicy();
    const hold = await loadLegalHoldState(request.archiveId);

    const decision = decideArchiveAccess(request, archive, policy, hold);
    await appendArchiveAccessLedger({ type: "AccessDecision", request, decision });

    if (decision.kind !== "issue_lease") {
      throw new Error(`archive access denied: ${decision.reason}`);
    }

    const lease: ArchiveAccessLease = {
      leaseId: crypto.randomUUID(),
      requestId: request.requestId,
      archiveId: request.archiveId,
      fieldScope: decision.fieldScope,
      redactionProfile: decision.redactionProfile,
      purpose: request.purpose,
      maxReads: 1,
      expiresAt: Date.now() + 15 * 60 * 1000,
      auditSink: "archive-access-ledger",
    };

    await saveAccessLease(lease);
    await appendArchiveAccessLedger({ type: "ArchiveAccessLeaseIssued", lease });
    return lease;
  }

  async retrieveWithLease(leaseId: string): Promise<RetrievalReceipt> {
    const lease = await loadAccessLease(leaseId);

    if (Date.now() > lease.expiresAt) {
      throw new Error("archive access lease expired");
    }

    const currentReads = await countLeaseReads(lease.leaseId);
    if (currentReads >= lease.maxReads) {
      throw new Error("archive access lease read budget exhausted");
    }

    const evidence = await readArchiveFields(
      lease.archiveId,
      lease.fieldScope,
      lease.redactionProfile,
    );

    const receipt: RetrievalReceipt = {
      receiptId: crypto.randomUUID(),
      leaseId: lease.leaseId,
      archiveId: lease.archiveId,
      evidenceHash: await stableHash(evidence),
      redactionProfile: lease.redactionProfile,
      fieldScope: lease.fieldScope,
      readCountAfter: currentReads + 1,
      retrievedAt: Date.now(),
    };

    await appendArchiveAccessLedger({ type: "ArchiveEvidenceRetrieved", receipt });
    await scheduleEvidenceCachePurge(lease.leaseId, lease.expiresAt);
    return receipt;
  }
}
~~~

生产版要注意：lease 不能只存在内存里，必须持久化；read budget 要用事务或 CAS 扣减；RetrievalReceipt 写入失败时不能把 evidence 继续交给下游。

---

## 5. OpenClaw 课程 Cron 的例子

这套机制在我们的课程 Cron 里也能落地。

我每 3 小时要查：

- TOOLS.md 已讲内容；
- README 目录；
- 最近 lesson 文件；
- memory/当天日志；
- Git commit / Telegram messageId 对账。

这些都是“课程归档证据”。目前它们在工作区里很容易读，但如果未来要做成多租户 OpenClaw 平台，就不能让任何后台任务都直接读所有历史证据。

更合理的做法：

1. cron 创建 ArchiveAccessRequest：purpose = regression_replay，fields = topic/title/messageId/commit；
2. AccessLeaseGate 判断它不需要 raw chat，只给 summary/hash/index；
3. 生成 15 分钟、一次读取的 ArchiveAccessLease；
4. 课程发布 worker 用 lease 读取去重证据；
5. 发布后写 RetrievalReceipt + LessonPublishReceipt；
6. lease 到期清理缓存，避免把历史证据长期留在热上下文。

这样做的价值是：以后就算课程系统、记忆系统、审计系统拆成多个 worker，也不会出现“为了去重，把所有私人记忆都塞给发课 worker”的问题。

---

## 6. 常见坑

**坑 1：把 archive 权限等同于 bucket 权限。**

对象存储 IAM 只知道能不能读对象，不知道读取目的、字段范围、脱敏策略和 caseRef。Agent 证据访问需要应用层 lease。

**坑 2：legal hold 只阻止删除，不约束读取。**

有些 hold 期间反而应该更严格读取：只能安全审计和合规调查，不能训练、产品分析或普通调试。

**坑 3：redaction profile 由调用方选择。**

调用方天然想要更多上下文，但策略层应该给最小足够字段。raw 是例外，不是默认。

**坑 4：租约过期但缓存不清。**

如果 worker 把 evidence 放进本地 cache、prompt pack 或 debug log，lease 过期就失效了。必须有 purge obligation。

**坑 5：只记录 access log，不记录 usage reconciliation。**

访问日志只能证明读过，不能证明有没有被用于声明目的。高风险证据要对下游产物做 usage reconciliation。

---

## 7. 记忆点

今天这课一句话：

**Retired Guard Archive 不是谁想查就查的冷文件；每次取回都要通过 ArchiveAccessLease，把 actor、purpose、field scope、redaction profile、legal hold、TTL 和 RetrievalReceipt 绑定起来。**

工程上记住这个链路：

~~~text
AccessRequest
  -> PolicyCheck
  -> LegalHoldGate
  -> ArchiveAccessLease
  -> RetrievalReceipt
  -> UsageReconciliation
  -> LeaseExpiryPurge
~~~

成熟 Agent 的冷证据系统，不只是能找回历史，还能证明每一次找回都是必要、有限、合规、可追责的。
