# 506 - Agent 清理后的冷证据保留与审计抽样闸门

> rollback alias 删掉以后，旧路径才算真正离开运行面；但它不能从证据面消失。成熟 Agent 要把清理后的证据转入冷保留，并用低频抽样持续证明历史仍可解释、可审计、可恢复。

上一讲讲了 Post-Retirement Rollback Alias Expiry：旧 primary 退役后，rollback alias 到期前要检查隐藏依赖、迟到 callback、补偿任务和证据 hash，确认可以安全删除。

今天继续往后走一步：**alias 已经删了，如何确保未来审计、复盘、合规和回归重放还能找到证据**。

很多系统会在这里出问题：

- 清理运行配置时顺手删掉旧证据索引
- evidence bundle 存在，但缺少可验证 hash 和 schema version
- 审计查询只能依赖旧 alias，alias 删除后历史解释断链
- 冷归档多年后格式漂移，恢复演练才发现读不回来
- 抽样只检查文件存在，不检查能否解释真实 decision

所以清理后还需要一个 **Cold Evidence Retention Gate**：把热路径证据从运行系统中摘掉，但把审计能力用更便宜、更稳定的方式保留下来。

---

## 核心模型

```
FinalRetirementReceipt
        ↓
ColdEvidenceRetentionPlan
        ↓
EvidenceIndexFreezeReceipt
        ↓
RetentionPolicyLease
        ↓
AuditSamplingProbe
        ↓
EvidenceRetrievalReceipt
        ↓
RetentionHealthReview
        ↓
ColdEvidenceRetentionCloseout
```

关键判断：

- final cleanup 是否已经完成并带有 receipt
- evidence index 是否冻结，hash 是否可复算
- 保留策略是否覆盖审计、复盘、回归、合规四类用途
- 冷证据是否能按 decisionId / cutoverId / receiptId 找回
- 抽样是否真的重建了历史解释，而不只是 `file exists`
- 是否存在 schema drift、redaction drift 或 legal hold 冲突

---

## learn-claude-code：冷证据保留判定器

先把清理后的保留决策写成纯函数。它不负责上传文件，只负责回答：现在能不能把热证据转冷、要不要补索引、要不要阻断关闭。

```python
from dataclasses import dataclass
from typing import Literal

Decision = Literal[
    "activate_cold_retention",
    "rebuild_evidence_index",
    "extend_hot_retention",
    "block_retention_closeout",
    "manual_review",
]

@dataclass
class FinalRetirementReceipt:
    cleanup_receipt_verified: bool
    alias_removed: bool
    post_cleanup_probe_ok: bool
    final_evidence_hash: str | None

@dataclass
class RetentionReadiness:
    index_frozen: bool
    hash_recomputable: bool
    schema_version_pinned: bool
    decision_lookup_ok: bool
    receipt_lookup_ok: bool
    legal_hold_conflict: bool
    redaction_profile_pinned: bool

@dataclass
class AuditSample:
    sampled_decisions: int
    retrieval_failures: int
    explanation_failures: int
    schema_drift_found: bool
    redaction_drift_found: bool

def decide_cold_retention(
    receipt: FinalRetirementReceipt,
    readiness: RetentionReadiness,
    sample: AuditSample,
) -> Decision:
    if not receipt.cleanup_receipt_verified or not receipt.alias_removed:
        return "block_retention_closeout"

    if not receipt.post_cleanup_probe_ok or not receipt.final_evidence_hash:
        return "manual_review"

    if readiness.legal_hold_conflict:
        return "extend_hot_retention"

    index_ready = (
        readiness.index_frozen
        and readiness.hash_recomputable
        and readiness.schema_version_pinned
        and readiness.decision_lookup_ok
        and readiness.receipt_lookup_ok
        and readiness.redaction_profile_pinned
    )

    if not index_ready:
        return "rebuild_evidence_index"

    if sample.retrieval_failures > 0 or sample.explanation_failures > 0:
        return "block_retention_closeout"

    if sample.schema_drift_found or sample.redaction_drift_found:
        return "rebuild_evidence_index"

    if sample.sampled_decisions < 10:
        return "manual_review"

    return "activate_cold_retention"
```

这里最容易被忽略的是 `explanation_failures`。证据文件能下载不够，系统还要能回答：

- 当时为什么切流？
- 哪个 receipt 允许删除 alias？
- 旧 decision 用的是哪个 runtime fingerprint？
- 如果今天重放，应该用哪个 schema 和 redaction profile？

测试要覆盖“文件还在但解释失败”的情况：

```python
def test_blocks_when_cold_evidence_cannot_explain_decision():
    receipt = FinalRetirementReceipt(
        cleanup_receipt_verified=True,
        alias_removed=True,
        post_cleanup_probe_ok=True,
        final_evidence_hash="sha256:abc",
    )
    readiness = RetentionReadiness(
        index_frozen=True,
        hash_recomputable=True,
        schema_version_pinned=True,
        decision_lookup_ok=True,
        receipt_lookup_ok=True,
        legal_hold_conflict=False,
        redaction_profile_pinned=True,
    )
    sample = AuditSample(
        sampled_decisions=25,
        retrieval_failures=0,
        explanation_failures=1,
        schema_drift_found=False,
        redaction_drift_found=False,
    )

    assert decide_cold_retention(receipt, readiness, sample) == "block_retention_closeout"
```

---

## pi-mono：Cold Evidence Retention Worker

生产里建议把冷保留做成独立 worker，不要塞进 cleanup worker。cleanup 负责删除运行 alias；retention 负责证明删除后还能审计。

```typescript
type ColdRetentionDecision =
  | "activate_cold_retention"
  | "rebuild_evidence_index"
  | "extend_hot_retention"
  | "block_retention_closeout"
  | "manual_review";

interface ColdRetentionJob {
  cutoverId: string;
  finalRetirementReceiptId: string;
  retiredPath: string;
  retentionClass: "audit" | "compliance" | "regression" | "incident_review";
}

class ColdEvidenceRetentionWorker {
  constructor(
    private readonly receipts: ReceiptStore,
    private readonly evidence: EvidenceArchive,
    private readonly index: EvidenceIndex,
    private readonly sampler: AuditSampler,
    private readonly audit: AuditLog,
  ) {}

  async run(job: ColdRetentionJob) {
    const finalReceipt = await this.receipts.getFinalRetirementReceipt(
      job.finalRetirementReceiptId,
    );

    const readiness = await this.index.reviewReadiness({
      cutoverId: job.cutoverId,
      retiredPath: job.retiredPath,
      retentionClass: job.retentionClass,
    });

    const sample = await this.sampler.probeColdEvidence({
      cutoverId: job.cutoverId,
      minDecisions: 25,
      requireExplanation: true,
    });

    const decision = decideColdRetention(finalReceipt, readiness, sample);

    await this.audit.append({
      type: "cold_evidence_retention_decision",
      cutoverId: job.cutoverId,
      retiredPath: job.retiredPath,
      retentionClass: job.retentionClass,
      decision,
      sampleSize: sample.sampledDecisions,
      retrievalFailures: sample.retrievalFailures,
      explanationFailures: sample.explanationFailures,
    });

    if (decision === "rebuild_evidence_index") {
      await this.index.rebuild({
        cutoverId: job.cutoverId,
        expectedHash: finalReceipt.finalEvidenceHash,
      });
      return;
    }

    if (decision === "extend_hot_retention") {
      await this.evidence.extendHotRetention({
        cutoverId: job.cutoverId,
        reason: "legal_hold_or_retention_conflict",
      });
      return;
    }

    if (decision !== "activate_cold_retention") {
      await this.receipts.writeRetentionBlocker({
        cutoverId: job.cutoverId,
        finalRetirementReceiptId: job.finalRetirementReceiptId,
        decision,
      });
      return;
    }

    await this.evidence.activateColdRetention({
      cutoverId: job.cutoverId,
      finalRetirementReceiptId: job.finalRetirementReceiptId,
      retentionClass: job.retentionClass,
      immutableHash: finalReceipt.finalEvidenceHash,
    });

    await this.receipts.writeColdEvidenceRetentionCloseout({
      cutoverId: job.cutoverId,
      finalRetirementReceiptId: job.finalRetirementReceiptId,
      sampleReceiptIds: sample.receiptIds,
      immutableHash: finalReceipt.finalEvidenceHash,
    });
  }
}
```

注意两个边界：

1. `activateColdRetention` 必须是幂等操作，重复执行只能返回同一个 closeout receipt。
2. `rebuild` 只能重建索引，不能改写原始 evidence bundle；原始证据 hash 是历史锚点。

---

## OpenClaw：课程 Cron 的类比

这门课自己的发布流程也有类似模式：

1. `git commit && git push` 是运行面交付。
2. README 目录和 TOOLS 已讲清单是索引。
3. Telegram messageId 是外部投递收据。
4. lesson 文件内容是可审计证据。

如果某天我们删除旧分支、清理临时文件，只要 lesson、README、TOOLS 和 Telegram 投递记录还能对上，就能解释“第 506 课讲了什么、何时发布、发布到了哪里、仓库里是哪次提交”。

一个简单的 cron 自检可以这样做：

```bash
lesson="lessons/506-post-cleanup-cold-evidence-retention-audit-sampling.md"

test -f "$lesson"
rg "506-post-cleanup-cold-evidence-retention-audit-sampling" README.md
rg "清理后的冷证据保留" ../TOOLS.md
git log -1 --oneline
```

这不是形式主义。Agent 自动化越久，越需要能回答“历史为什么是这样”的问题。

---

## 设计要点

| 问题 | 错误做法 | 推荐做法 |
| --- | --- | --- |
| alias 已删除 | 认为事故生命周期结束 | 进入冷证据保留阶段 |
| 证据存在性 | 只检查文件存在 | 抽样重建历史解释 |
| 索引漂移 | 直接覆盖旧索引 | 重建派生索引，原始 bundle hash 不变 |
| 合规冲突 | 清理后再补救 | legal hold 冲突时延长热保留 |
| 抽样成本 | 全量频繁扫描 | 分层低频抽样，失败再扩大范围 |

---

## 一句话总结

最终清理让旧路径离开运行面；冷证据保留让旧历史继续可解释。成熟 Agent 删除的是依赖，不是证据；降低的是运行成本，不是审计能力。
