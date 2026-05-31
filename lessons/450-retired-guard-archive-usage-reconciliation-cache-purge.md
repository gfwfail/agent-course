# 450. Agent 退役护栏归档用途对账与缓存清理收据（Retired Guard Archive Usage Reconciliation & Cache Purge Receipt）

> 关键词：usage reconciliation、cache purge receipt、purpose binding、derived artifact audit、lease expiry

上一课讲了 ArchiveAccessLease：冷归档能取回，不代表任何 worker 都能直接读；每次访问都要绑定 actor、purpose、field scope、redaction profile、legal hold、TTL 和 RetrievalReceipt。

今天补上它的下一段闭环：**证据读出来以后，必须证明它只被用于声明用途，并且租约过期后缓存确实被清掉。**

很多系统会做到“访问前审批”，但漏掉两个生产问题：

- worker 拿到证据后，把内容带进 prompt、debug log、临时 cache、derived report；
- 访问租约过期了，但 derived artifact 还在热路径里继续被用。

成熟 Agent 的证据访问不是“批准一次读取”，而是一条完整收据链：

~~~text
ArchiveAccessLease
  -> RetrievalReceipt
  -> DeclaredUsagePlan
  -> DerivedArtifactReceipt
  -> UsageReconciliationReceipt
  -> CachePurgeReceipt
~~~

一句话：**读得合法，不等于用得合规；租约过期，不等于缓存自动消失。**

---

## 1. 为什么访问后还要对账

ArchiveAccessLease 解决的是“谁、为什么、在什么范围内可以读”。但真实风险通常发生在读取之后。

典型例子：

- security_audit 读取事故证据，本来只允许生成摘要，却把 raw payload 写进 Slack；
- regression_replay 读取历史样本，本来只允许 hash 和标签，却把用户文本塞进训练集；
- customer_support 读取归档，本来只为一个 case 使用，却被复用到另一个客户工单；
- debugging 读取临时证据，本来 15 分钟有效，却被本地 worker cache 保存三天；
- cron 去重读取历史 lesson，本来只需要 title/hash，却把完整 memory 片段写进发布日志。

这些问题在普通 access log 里看不出来。access log 只能证明“读过”，不能证明：

- 读到的 evidence 被交给了哪些下游；
- 下游产物是否包含超范围字段；
- 产物用途是否仍匹配 lease purpose；
- 租约过期后缓存、prompt pack、debug log 是否清理；
- 清理失败时是否阻断后续任务。

所以要做 **Usage Reconciliation**：把 RetrievalReceipt 和实际下游产物对账。

---

## 2. 状态机：从取回到清理

建议状态机：

~~~text
lease_issued
  -> evidence_retrieved
  -> usage_declared
  -> artifact_emitted
  -> usage_reconciled
  -> purge_scheduled
  -> purge_verified
  -> close_verified
  -> violation_opened
~~~

关键对象：

- **DeclaredUsagePlan**：worker 声明证据会进入哪些 artifact，例如 replay report、audit summary、support answer；
- **DerivedArtifactReceipt**：每个下游产物记录 artifactId、purpose、field hashes、redaction profile、destination；
- **UsageReconciliationReceipt**：对比 lease scope 和 artifact 内容，确认没有 purpose drift、field overread、raw leak；
- **CachePurgeReceipt**：租约过期后证明 local cache、prompt cache、debug temp、artifact staging 都已清理；
- **ViolationCase**：发现越权用途、字段泄漏、缓存未清时打开可关闭案件。

这套链路的核心不是多记几行日志，而是把“证据使用”变成可验证的工程边界。

---

## 3. learn-claude-code：用途对账纯函数

教学版先写纯函数：输入访问租约、取回收据、下游产物，输出 close / purge / violation。

~~~python
# learn_claude_code/guard_patterns/archive_usage_reconciliation.py
from dataclasses import dataclass
from typing import Literal

Decision = Literal[
    "usage_ok_schedule_purge",
    "purge_required_now",
    "open_usage_violation",
    "manual_review",
]


@dataclass(frozen=True)
class ArchiveAccessLease:
    lease_id: str
    archive_id: str
    purpose: str
    field_scope: tuple[str, ...]
    redaction_profile: str
    expires_at_epoch: int
    allow_derived_artifacts: bool


@dataclass(frozen=True)
class RetrievalReceipt:
    receipt_id: str
    lease_id: str
    evidence_hash: str
    field_scope: tuple[str, ...]
    redaction_profile: str
    retrieved_at_epoch: int


@dataclass(frozen=True)
class DerivedArtifactReceipt:
    artifact_id: str
    lease_id: str
    artifact_kind: str
    purpose: str
    included_fields: tuple[str, ...]
    redaction_profile: str
    destination: str
    contains_raw_payload: bool
    emitted_at_epoch: int


@dataclass(frozen=True)
class CacheState:
    lease_id: str
    cache_keys: tuple[str, ...]
    prompt_cache_keys: tuple[str, ...]
    debug_temp_keys: tuple[str, ...]
    last_verified_epoch: int


@dataclass(frozen=True)
class UsageReconciliationDecision:
    decision: Decision
    reason: str
    violating_artifacts: tuple[str, ...] = ()
    purge_targets: tuple[str, ...] = ()
    obligations: tuple[str, ...] = ()


def reconcile_archive_usage(
    lease: ArchiveAccessLease,
    retrieval: RetrievalReceipt,
    artifacts: tuple[DerivedArtifactReceipt, ...],
    cache: CacheState,
    now_epoch: int,
) -> UsageReconciliationDecision:
    if retrieval.lease_id != lease.lease_id:
        return UsageReconciliationDecision(
            "open_usage_violation",
            "retrieval_receipt_does_not_match_lease",
        )

    if set(retrieval.field_scope) - set(lease.field_scope):
        return UsageReconciliationDecision(
            "open_usage_violation",
            "retrieval_scope_exceeds_lease_scope",
        )

    violations: list[str] = []
    for artifact in artifacts:
        if artifact.lease_id != lease.lease_id:
            continue

        if not lease.allow_derived_artifacts:
            violations.append(artifact.artifact_id)
            continue

        if artifact.purpose != lease.purpose:
            violations.append(artifact.artifact_id)
            continue

        if set(artifact.included_fields) - set(lease.field_scope):
            violations.append(artifact.artifact_id)
            continue

        if artifact.contains_raw_payload and lease.redaction_profile != "raw":
            violations.append(artifact.artifact_id)
            continue

        if artifact.redaction_profile == "raw" and lease.redaction_profile != "raw":
            violations.append(artifact.artifact_id)

    if violations:
        return UsageReconciliationDecision(
            "open_usage_violation",
            "derived_artifact_violates_lease_boundary",
            violating_artifacts=tuple(violations),
            obligations=("quarantine_artifacts", "notify_evidence_owner"),
        )

    purge_targets = cache.cache_keys + cache.prompt_cache_keys + cache.debug_temp_keys
    if now_epoch >= lease.expires_at_epoch and purge_targets:
        return UsageReconciliationDecision(
            "purge_required_now",
            "lease_expired_with_cached_evidence",
            purge_targets=purge_targets,
            obligations=("write_cache_purge_receipt",),
        )

    return UsageReconciliationDecision(
        "usage_ok_schedule_purge",
        "usage_matches_purpose_scope_and_redaction",
        purge_targets=purge_targets,
        obligations=("schedule_expiry_purge", "write_usage_reconciliation_receipt"),
    )
~~~

这个函数故意只做判定，不做删除。删除是副作用，要进入 worker，并写 CachePurgeReceipt。

---

## 4. pi-mono：UsageReconciliationWorker

pi-mono 的生产实现可以把 evidence access 当成事件流：取回证据、生成产物、对账、清缓存，每一步都写 ledger。

~~~ts
type UsageReconciliationReceipt = {
  receiptId: string;
  leaseId: string;
  retrievalReceiptId: string;
  checkedArtifacts: string[];
  decision:
    | "usage_ok_schedule_purge"
    | "purge_required_now"
    | "open_usage_violation"
    | "manual_review";
  reason: string;
  checkedAt: number;
};

type CachePurgeReceipt = {
  receiptId: string;
  leaseId: string;
  purgedCacheKeys: string[];
  failedCacheKeys: string[];
  promptCacheCleared: boolean;
  debugTempCleared: boolean;
  purgedAt: number;
};

class ArchiveUsageReconciliationWorker {
  async reconcile(leaseId: string): Promise<UsageReconciliationReceipt> {
    const lease = await loadArchiveAccessLease(leaseId);
    const retrieval = await loadRetrievalReceipt(leaseId);
    const artifacts = await listDerivedArtifactsByLease(leaseId);
    const cacheState = await inspectEvidenceCache(leaseId);

    const decision = reconcileArchiveUsage(
      lease,
      retrieval,
      artifacts,
      cacheState,
      Date.now(),
    );

    const receipt: UsageReconciliationReceipt = {
      receiptId: crypto.randomUUID(),
      leaseId,
      retrievalReceiptId: retrieval.receiptId,
      checkedArtifacts: artifacts.map((artifact) => artifact.artifactId),
      decision: decision.decision,
      reason: decision.reason,
      checkedAt: Date.now(),
    };

    await appendArchiveAccessLedger({
      type: "UsageReconciliationCompleted",
      receipt,
    });

    if (decision.decision === "open_usage_violation") {
      await openEvidenceUsageViolationCase({
        leaseId,
        artifactIds: decision.violatingArtifacts,
        reason: decision.reason,
      });
    }

    if (decision.decision === "purge_required_now") {
      await this.purgeExpiredEvidence(leaseId, decision.purgeTargets);
    }

    return receipt;
  }

  async purgeExpiredEvidence(
    leaseId: string,
    cacheKeys: string[],
  ): Promise<CachePurgeReceipt> {
    const purged: string[] = [];
    const failed: string[] = [];

    for (const cacheKey of cacheKeys) {
      const result = await purgeEvidenceCacheKey(cacheKey);
      if (result.ok) {
        purged.push(cacheKey);
      } else {
        failed.push(cacheKey);
      }
    }

    const receipt: CachePurgeReceipt = {
      receiptId: crypto.randomUUID(),
      leaseId,
      purgedCacheKeys: purged,
      failedCacheKeys: failed,
      promptCacheCleared: await clearPromptCacheForLease(leaseId),
      debugTempCleared: await clearDebugTempForLease(leaseId),
      purgedAt: Date.now(),
    };

    await appendArchiveAccessLedger({
      type: "CachePurgeCompleted",
      receipt,
    });

    if (failed.length > 0 || !receipt.promptCacheCleared || !receipt.debugTempCleared) {
      await openEvidenceUsageViolationCase({
        leaseId,
        artifactIds: [],
        reason: "expired_evidence_cache_purge_failed",
      });
    }

    return receipt;
  }
}
~~~

生产版要注意三点：

- artifact receipt 要在产物写出前落账，否则失败时会丢下游引用；
- purge 要允许重试，但必须幂等，重复清理不能报假失败；
- 发现 usage violation 后，不能只报警，要 quarantine artifact，防止继续扩散。

---

## 5. OpenClaw 课程 Cron 的例子

这门课程 cron 每 3 小时都会读历史课程和 TOOLS.md 来去重。现在它的风险很小，因为是在单人工作区里运行；但如果搬到多租户 OpenClaw，证据访问必须更严格。

合理链路可以是：

1. 选题 worker 申请 ArchiveAccessLease，只取 lesson title、slug、summary、commit hash；
2. 读取后写 RetrievalReceipt；
3. 生成本次 lesson draft 时写 DerivedArtifactReceipt，声明 purpose = lesson_deduplication；
4. 发布 Telegram 之前跑 UsageReconciliation，确认 draft 没带入 raw private memory；
5. 发布成功后写 LessonPublishReceipt，绑定 messageId、commit、lesson file；
6. 租约到期后写 CachePurgeReceipt，清掉选题临时索引、prompt scratch、debug temp；
7. 如果清理失败，阻断下一轮课程 cron，先处理 EvidenceUsageViolationCase。

这听起来比“读 README 然后发课”麻烦，但好处很实际：当课程系统变成多个 worker、多个用户、多个 workspace 时，去重证据不会变成隐私泄漏通道。

---

## 6. 常见坑

**坑 1：只对读取做审批，不追踪 derived artifact。**

证据真正泄漏的位置通常不是 archive read，而是报告、日志、prompt、通知、训练样本这些下游产物。

**坑 2：把 redaction 当成读取时的一次性处理。**

如果下游 artifact 拼接了别的字段，或者 worker 又补读了别的上下文，artifact 自己也要记录 redaction profile。

**坑 3：租约过期只改状态，不清缓存。**

状态 expired 没意义。必须证明 cache key、prompt cache、debug temp、staging artifact 都清了。

**坑 4：purge 失败只发 warning。**

冷证据残留在热缓存里是边界突破。清理失败要 open violation case，必要时冻结相关 worker。

**坑 5：对账只看字段名，不看用途。**

同样的 summary 字段，用于 security audit 和 model training 是两种不同风险。purpose drift 也应该算 violation。

---

## 7. 记忆点

今天这课一句话：

**ArchiveAccessLease 只解决“能不能读”，UsageReconciliation 和 CachePurgeReceipt 才解决“有没有按声明目的使用、过期后有没有清干净”。**

工程上记住这个链路：

~~~text
RetrievalReceipt
  -> DerivedArtifactReceipt
  -> UsageReconciliationReceipt
  -> CachePurgeReceipt
  -> ViolationCase
~~~

成熟 Agent 的冷证据治理，不是批准访问就结束，而是能证明证据从取回、使用、派生产物到缓存清理，每一步都没有越界。
