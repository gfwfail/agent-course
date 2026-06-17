# 509 - Agent 派生证据复用许可与边界闸门

> ThawedEvidenceCloseout 证明原始冷证据已经按许可使用并销毁，但它不自动授权派生产物继续被别的任务复用。成熟 Agent 要把 derived evidence 的二次使用也做成许可、边界、审计和过期闭环。

上一讲我们讲了 Thawed Evidence Usage Closeout：冷证据解冻后，必须证明读了什么、派生了什么、清了什么、销毁了什么，以及残留探测是否通过。

今天继续补一个经常被忽略的洞：**raw evidence 销毁了，不代表 derived artifact 就可以随便用**。

常见事故是：

- incident review 生成的 diff summary 被拿去做 prompt few-shot；
- legal hold 允许保存的派生报告，被调试任务读取了敏感字段；
- replay fixture 只该用于某个 regression seed，却被合并进全局测试集；
- embedding cache 说是 redacted，实际还能反推原始 user context；
- 派生产物没有 TTL，几年后没人知道它来自哪次 thaw、哪条 permit、哪组字段。

所以需要一个 **Derived Evidence Reuse Permit Gate**：派生产物如果要离开原本的 thaw closeout 边界，必须重新申请用途绑定的复用许可。

---

## 核心模型

```
ThawedEvidenceCloseoutReceipt
        ↓
DerivedEvidenceInventory
        ↓
ReuseRequest
        ↓
DerivedReuseBoundaryReview
        ↓
DerivedEvidenceReusePermit
        ↓
ReuseExecutionReceipt
        ↓
ReuseExpiryReview
```

关键判断：

- closeout 是否已经通过，原始证据是否销毁；
- derived artifact 是否有 parent evidence hash、redaction profile、retention class；
- 新用途是否被原 thaw permit 的 purpose lineage 允许；
- artifact 是否包含 raw prompt、tool args、PII、unredacted context 等不可复用字段；
- 复用范围是否被绑定到具体 consumer、task、dataset、TTL；
- 复用后是否写入 ReuseExecutionReceipt，方便撤销和过期清理。

一句话：**派生产物不是“脱敏后就自由”，而是要证明它在新场景里仍然不越权。**

---

## learn-claude-code：复用许可判定器

教学版继续用纯函数。它不关心文件在哪，只判断这批派生产物能不能被复用。

```python
from dataclasses import dataclass
from typing import Literal

Decision = Literal[
    "issue_reuse_permit",
    "require_stronger_redaction",
    "deny_reuse",
    "manual_review",
]

Purpose = Literal[
    "incident_review",
    "regression_replay",
    "audit_report",
    "prompt_training",
    "eval_dataset",
]

@dataclass
class ThawedCloseoutReceipt:
    closeout_passed: bool
    raw_evidence_destroyed: bool
    residual_probe_clean: bool
    source_purpose: Purpose

@dataclass
class DerivedArtifact:
    registered: bool
    parent_hash_present: bool
    redaction_profile: Literal["none", "basic", "strict", "irreversible"]
    retention_class: Literal["ephemeral", "case_bound", "reusable", "legal_hold"]
    contains_raw_prompt: bool
    contains_tool_args: bool
    contains_pii: bool
    reversible_to_raw: bool

@dataclass
class ReuseRequest:
    requested_purpose: Purpose
    consumer_id: str
    ttl_days: int
    writes_to_shared_dataset: bool
    owner_approved: bool

def decide_derived_reuse(
    closeout: ThawedCloseoutReceipt,
    artifact: DerivedArtifact,
    request: ReuseRequest,
) -> Decision:
    if not closeout.closeout_passed:
        return "deny_reuse"

    if not closeout.raw_evidence_destroyed or not closeout.residual_probe_clean:
        return "manual_review"

    if not artifact.registered or not artifact.parent_hash_present:
        return "manual_review"

    if artifact.contains_raw_prompt or artifact.contains_tool_args:
        return "deny_reuse"

    if artifact.contains_pii or artifact.reversible_to_raw:
        return "require_stronger_redaction"

    if artifact.redaction_profile in {"none", "basic"}:
        return "require_stronger_redaction"

    if artifact.retention_class == "ephemeral":
        return "deny_reuse"

    if request.requested_purpose == "prompt_training":
        return "deny_reuse"

    if request.writes_to_shared_dataset:
        if artifact.retention_class != "reusable":
            return "deny_reuse"
        if artifact.redaction_profile != "irreversible":
            return "require_stronger_redaction"

    if request.ttl_days > 30 and not request.owner_approved:
        return "manual_review"

    if closeout.source_purpose == "legal_hold" and request.requested_purpose != "audit_report":
        return "deny_reuse"

    return "issue_reuse_permit"
```

最重要的测试是防止“为了 eval 顺手复用 incident 数据”。

```python
def test_blocks_incident_summary_as_prompt_training_data():
    closeout = ThawedCloseoutReceipt(
        closeout_passed=True,
        raw_evidence_destroyed=True,
        residual_probe_clean=True,
        source_purpose="incident_review",
    )
    artifact = DerivedArtifact(
        registered=True,
        parent_hash_present=True,
        redaction_profile="irreversible",
        retention_class="reusable",
        contains_raw_prompt=False,
        contains_tool_args=False,
        contains_pii=False,
        reversible_to_raw=False,
    )
    request = ReuseRequest(
        requested_purpose="prompt_training",
        consumer_id="prompt-optimizer",
        ttl_days=7,
        writes_to_shared_dataset=True,
        owner_approved=True,
    )

    assert decide_derived_reuse(closeout, artifact, request) == "deny_reuse"
```

这个测试看起来保守，但生产里很有价值：incident 数据即使被摘要和脱敏，也可能带有真实用户行为、内部策略、工具输出和失败路径。它可以用于回归验证，不应该默认进入 prompt training。

---

## pi-mono：Derived Reuse Worker

生产版可以把复用许可放在证据服务或 dataset ingestion 前面。任何任务要把 derived artifact 写进共享索引、eval dataset、报告仓库，都要先拿 permit。

```typescript
type DerivedReuseDecision =
  | "issue_reuse_permit"
  | "require_stronger_redaction"
  | "deny_reuse"
  | "manual_review";

interface DerivedReuseJob {
  closeoutReceiptId: string;
  artifactId: string;
  requestedPurpose: "incident_review" | "regression_replay" | "audit_report" | "prompt_training" | "eval_dataset";
  consumerId: string;
  ttlDays: number;
  writesToSharedDataset: boolean;
  ownerApproved: boolean;
}

class DerivedEvidenceReuseWorker {
  constructor(
    private readonly closeouts: ThawedCloseoutStore,
    private readonly artifacts: DerivedEvidenceStore,
    private readonly redactor: EvidenceRedactionService,
    private readonly permits: ReusePermitStore,
    private readonly receipts: ReceiptStore,
    private readonly outbox: Outbox,
  ) {}

  async run(job: DerivedReuseJob) {
    const closeout = await this.closeouts.get(job.closeoutReceiptId);
    const artifact = await this.artifacts.get(job.artifactId);

    const decision = decideDerivedReuse(closeout, artifact, {
      requestedPurpose: job.requestedPurpose,
      consumerId: job.consumerId,
      ttlDays: job.ttlDays,
      writesToSharedDataset: job.writesToSharedDataset,
      ownerApproved: job.ownerApproved,
    });

    return this.receipts.transaction(async (tx) => {
      if (decision === "require_stronger_redaction") {
        const redactionJob = await this.redactor.enqueue(tx, {
          artifactId: job.artifactId,
          targetProfile: job.writesToSharedDataset ? "irreversible" : "strict",
          reason: "derived_reuse_boundary",
        });

        await this.outbox.publish(tx, {
          type: "evidence.redaction_required",
          artifactId: job.artifactId,
          redactionJobId: redactionJob.id,
        });
      }

      if (decision === "issue_reuse_permit") {
        await this.permits.create(tx, {
          artifactId: job.artifactId,
          closeoutReceiptId: job.closeoutReceiptId,
          purpose: job.requestedPurpose,
          consumerId: job.consumerId,
          expiresAt: new Date(Date.now() + job.ttlDays * 24 * 60 * 60 * 1000),
          maxReads: job.writesToSharedDataset ? 1 : 50,
        });
      }

      await tx.saveDerivedReuseReceipt({
        artifactId: job.artifactId,
        closeoutReceiptId: job.closeoutReceiptId,
        decision,
        requestedPurpose: job.requestedPurpose,
        consumerId: job.consumerId,
        artifactHash: artifact.hash,
        redactionProfile: artifact.redactionProfile,
      });

      return decision;
    });
  }
}
```

这里有三个实现要点：

- dataset ingestion 不能直接读 `DerivedEvidenceStore`，必须要求 `DerivedEvidenceReusePermit`；
- permit 要绑定 consumer、purpose、TTL、maxReads，不能变成永久 token；
- stronger redaction 要生成新 artifact hash，旧 hash 不能被悄悄替换。

---

## OpenClaw 课程 Cron 的类比

这条课程流水线也有 derived artifact：

- lesson markdown 是本次 cron 的派生产物；
- README 目录是 lesson 的索引派生；
- TOOLS.md 已讲列表是长期记忆派生；
- Telegram 文案是面向群组的交付派生。

如果以后要把这些课程整理成公开合集，不能直接把工作区里的所有上下文一起复制出去。正确做法是：

- 只复用 lesson 和 README 中适合公开的内容；
- 不复用 cron payload、私有 workspace notes、token、账号信息；
- 给“公开合集”单独写 reuse permit，标明用途、范围和过期策略；
- 每次发布后保留 commit hash 和发布收据。

这就是 Derived Evidence Reuse 的直觉：**从内部任务产生的内容，可以复用，但不能忘记它来自哪里、原用途是什么、边界是什么。**

---

## 实战检查清单

设计派生证据复用时，至少问 8 个问题：

1. closeout receipt 是否证明 raw evidence 已销毁且 residual probe 通过？
2. derived artifact 是否注册了 parent hash 和 source permit？
3. redaction profile 是 basic、strict，还是 irreversible？
4. artifact 是否还能反推 raw prompt、tool args、PII 或 user context？
5. 新用途是否和原用途兼容？
6. 是否写入共享 dataset、prompt training、长期索引？
7. reuse permit 是否绑定 consumer、TTL、maxReads 和 purpose？
8. 复用后是否有 ReuseExecutionReceipt，未来能撤销、过期和审计？

---

## 小结

Thawed Evidence Usage Closeout 解决的是“原始证据有没有按许可使用并销毁”。

Derived Evidence Reuse Permit 解决的是“原始证据销毁后，派生产物能不能进入新的用途和更大的传播范围”。

成熟 Agent 不把“已脱敏”当万能通行证。任何派生产物只要要离开原任务边界，就要重新证明用途、范围、红线、TTL 和审计收据都成立。
