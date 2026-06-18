# 512 - Agent 删除后的哈希存根验证与可证明遗忘闸门

> Final erasure 不是“删完就忘”。成熟 Agent 要在不保留 raw data 的前提下，留下可验证、不可逆、可过期的 hash-only audit stub，未来能证明“当时删了什么、为什么删、谁批准、删除后没有反向命中”，同时不能让这个 stub 变成新的敏感数据索引。

上一讲讲了 Legal Hold Minimal View Release：legal hold 释放后，系统要删除 object/search/vector/cache/workspace/audit projection，并按需保留不可逆 hash stub。

今天继续补最后一段：**hash-only audit stub 也要被治理**。它不是 raw data，但它仍然可能泄漏结构、租户、对象数量、时间线和关联关系。

常见问题：

- 删除了 raw payload，却把可枚举的 sha256(email) 当审计证明长期保存；
- hash stub 没有 pepper/version，攻击者可以用字典反推；
- stub 只记录 deleted=true，没有记录删除前后的 probe evidence；
- 审计查询需要证明删除时，只能找工程师翻日志；
- stub 永久保留，最后又变成一个“影子数据集”。

所以需要一个 **Post-Erasure Hash Stub Verification & Proof-of-Forgetting Gate**：删除后只保留最小证明材料，并周期性验证它既能证明删除，又不能恢复或重建原始数据。

---

## 核心模型

```
LegalHoldReleaseCloseoutReceipt
        ↓
HashOnlyAuditStub
        ↓
ProofOfForgettingChallenge
        ↓
StubVerificationReport
        ↓
StubRetentionReview
        ↓
ProofOfForgettingCloseoutReceipt
```

关键判断：

- stub 是否只包含不可逆 digest、schema pin、decision summary 和 receipt id；
- digest 是否使用 per-tenant pepper/version，而不是裸 sha256(raw);
- challenge 能否证明 raw/object/search/vector/cache/workspace 都已无命中；
- stub 是否能回答“为什么删除、何时删除、谁批准”，但不能还原“删的原文是什么”；
- 保留期限到期后是否降级为 aggregate counter 或彻底删除；
- pepper rotation 后旧 stub 是否仍可验证 lineage，而不是失去审计能力。

---

## learn-claude-code：可证明遗忘纯函数

教学版先写一个纯函数：输入 stub、删除后挑战和保留策略，输出下一步动作。

```python
from dataclasses import dataclass
from typing import Literal

Decision = Literal[
    "close_proof_verified",
    "rewrite_with_peppered_digest",
    "purge_linkable_stub_fields",
    "rerun_erasure_probe",
    "downgrade_to_aggregate_counter",
    "delete_expired_stub",
    "open_stub_violation_case",
    "manual_review",
]

@dataclass
class HashOnlyAuditStub:
    stub_id: str
    receipt_id: str
    tenant_id_hash: str
    digest_algorithm: str
    pepper_version: str | None
    contains_raw_value: bool
    contains_plain_identifier: bool
    contains_decision_summary: bool
    contains_schema_pin: bool
    contains_owner_approval_ref: bool
    expires_at_epoch: int
    created_at_epoch: int

@dataclass
class ProofOfForgettingChallenge:
    challenged_at_epoch: int
    raw_object_hits: int
    search_hits: int
    vector_hits: int
    cache_hits: int
    workspace_hits: int
    can_answer_why_deleted: bool
    can_reconstruct_raw_value: bool
    lineage_receipt_matches: bool

@dataclass
class StubRetentionPolicy:
    now_epoch: int
    max_retention_seconds: int
    allow_hash_stub: bool
    allow_aggregate_counter_after_expiry: bool
    require_peppered_digest: bool

def decide_proof_of_forgetting(
    stub: HashOnlyAuditStub,
    challenge: ProofOfForgettingChallenge,
    policy: StubRetentionPolicy,
) -> Decision:
    if stub.contains_raw_value or challenge.can_reconstruct_raw_value:
        return "open_stub_violation_case"

    if stub.contains_plain_identifier:
        return "purge_linkable_stub_fields"

    if policy.require_peppered_digest and not stub.pepper_version:
        return "rewrite_with_peppered_digest"

    if stub.digest_algorithm.lower() in {"sha1", "md5"}:
        return "rewrite_with_peppered_digest"

    residual_hits = (
        challenge.raw_object_hits
        + challenge.search_hits
        + challenge.vector_hits
        + challenge.cache_hits
        + challenge.workspace_hits
    )
    if residual_hits > 0:
        return "rerun_erasure_probe"

    age = policy.now_epoch - stub.created_at_epoch
    expired_by_age = age > policy.max_retention_seconds
    expired_by_stub = policy.now_epoch >= stub.expires_at_epoch
    if expired_by_age or expired_by_stub:
        if policy.allow_aggregate_counter_after_expiry:
            return "downgrade_to_aggregate_counter"
        return "delete_expired_stub"

    minimum_fields = all([
        stub.receipt_id,
        stub.tenant_id_hash,
        stub.contains_decision_summary,
        stub.contains_schema_pin,
        stub.contains_owner_approval_ref,
        challenge.can_answer_why_deleted,
        challenge.lineage_receipt_matches,
    ])
    if not minimum_fields:
        return "manual_review"

    if not policy.allow_hash_stub:
        return "delete_expired_stub"

    return "close_proof_verified"
```

重点测试：如果 stub 里没有 raw，但 digest 是裸 hash，也不能关闭。

```python
def test_requires_peppered_digest_for_audit_stub():
    stub = HashOnlyAuditStub(
        stub_id="stub_1",
        receipt_id="erase_1",
        tenant_id_hash="tenant_hash",
        digest_algorithm="sha256",
        pepper_version=None,
        contains_raw_value=False,
        contains_plain_identifier=False,
        contains_decision_summary=True,
        contains_schema_pin=True,
        contains_owner_approval_ref=True,
        expires_at_epoch=2000,
        created_at_epoch=1000,
    )
    challenge = ProofOfForgettingChallenge(
        challenged_at_epoch=1200,
        raw_object_hits=0,
        search_hits=0,
        vector_hits=0,
        cache_hits=0,
        workspace_hits=0,
        can_answer_why_deleted=True,
        can_reconstruct_raw_value=False,
        lineage_receipt_matches=True,
    )
    policy = StubRetentionPolicy(
        now_epoch=1200,
        max_retention_seconds=10_000,
        allow_hash_stub=True,
        allow_aggregate_counter_after_expiry=True,
        require_peppered_digest=True,
    )

    assert decide_proof_of_forgetting(stub, challenge, policy) == "rewrite_with_peppered_digest"
```

这里的核心不是“hash 就安全”，而是：**hash stub 必须抗枚举、可过期、可解释、不可还原**。

---

## pi-mono：ProofOfForgettingWorker

生产版可以做成一个周期 worker：读取最新 erasure closeout receipt，抽样挑战 hash-only stub，必要时重写、降级或删除。

```ts
type ProofDecision =
  | "close_proof_verified"
  | "rewrite_with_peppered_digest"
  | "purge_linkable_stub_fields"
  | "rerun_erasure_probe"
  | "downgrade_to_aggregate_counter"
  | "delete_expired_stub"
  | "open_stub_violation_case"
  | "manual_review";

type HashOnlyAuditStub = {
  stubId: string;
  receiptId: string;
  tenantIdHash: string;
  digestAlgorithm: string;
  pepperVersion?: string;
  containsRawValue: boolean;
  containsPlainIdentifier: boolean;
  containsDecisionSummary: boolean;
  containsSchemaPin: boolean;
  containsOwnerApprovalRef: boolean;
  expiresAt: Date;
  createdAt: Date;
};

type ProofChallenge = {
  rawObjectHits: number;
  searchHits: number;
  vectorHits: number;
  cacheHits: number;
  workspaceHits: number;
  canAnswerWhyDeleted: boolean;
  canReconstructRawValue: boolean;
  lineageReceiptMatches: boolean;
};

class ProofOfForgettingWorker {
  constructor(
    private readonly stubRepo: HashStubRepository,
    private readonly erasureProbe: ErasureProbe,
    private readonly policy: StubRetentionPolicy,
    private readonly outbox: Outbox,
  ) {}

  async tick(now = new Date()) {
    const stubs = await this.stubRepo.findDueForVerification(now);

    for (const stub of stubs) {
      const challenge = await this.erasureProbe.challenge(stub);
      const decision = this.decide(stub, challenge, now);

      await this.stubRepo.writeVerificationReport({
        stubId: stub.stubId,
        receiptId: stub.receiptId,
        decision,
        challengedAt: now,
        residualHits:
          challenge.rawObjectHits +
          challenge.searchHits +
          challenge.vectorHits +
          challenge.cacheHits +
          challenge.workspaceHits,
        lineageReceiptMatches: challenge.lineageReceiptMatches,
      });

      await this.dispatchAction(stub, decision);
    }
  }

  private decide(
    stub: HashOnlyAuditStub,
    challenge: ProofChallenge,
    now: Date,
  ): ProofDecision {
    if (stub.containsRawValue || challenge.canReconstructRawValue) {
      return "open_stub_violation_case";
    }

    if (stub.containsPlainIdentifier) {
      return "purge_linkable_stub_fields";
    }

    if (this.policy.requirePepperedDigest && !stub.pepperVersion) {
      return "rewrite_with_peppered_digest";
    }

    const residualHits =
      challenge.rawObjectHits +
      challenge.searchHits +
      challenge.vectorHits +
      challenge.cacheHits +
      challenge.workspaceHits;
    if (residualHits > 0) {
      return "rerun_erasure_probe";
    }

    if (now >= stub.expiresAt) {
      return this.policy.allowAggregateCounterAfterExpiry
        ? "downgrade_to_aggregate_counter"
        : "delete_expired_stub";
    }

    if (
      !stub.containsDecisionSummary ||
      !stub.containsSchemaPin ||
      !stub.containsOwnerApprovalRef ||
      !challenge.canAnswerWhyDeleted ||
      !challenge.lineageReceiptMatches
    ) {
      return "manual_review";
    }

    return "close_proof_verified";
  }

  private async dispatchAction(stub: HashOnlyAuditStub, decision: ProofDecision) {
    if (decision === "close_proof_verified") return;

    await this.outbox.enqueue({
      topic: "proof_of_forgetting.action_required",
      key: `${stub.stubId}:${decision}`,
      payload: { stubId: stub.stubId, receiptId: stub.receiptId, decision },
    });
  }
}
```

生产注意点：

1. **重写 digest 要保留 lineage**：从 `pepperVersion=v1` 迁到 `v2` 时，不能丢掉原 erasure receipt id。
2. **不要在 LLM 上下文放 raw probe 结果**：只放 hit count、receipt id、decision 和 caveat。
3. **stub 查询也要审计**：谁为了什么目的验证“某条数据已删除”，本身也是敏感行为。
4. **过期后优先 aggregate**：很多合规场景只需要“删除了 N 条”，不需要每条都有可查 stub。

---

## OpenClaw 实战：课程 Cron 的 proof-of-forgetting 类比

这次课程 Cron 有三个外部/持久副作用：

- lesson 文件写入；
- README 和 TOOLS 更新；
- Telegram 发群消息；
- git commit/push。

如果未来要删除某次课程的敏感片段，不能只 `git rm` 或改 Telegram 消息。更稳的做法是：

1. 生成 `ErasureExecutionReceipt`：记录删除了哪个 commit/file/message 的哪些字段；
2. 跑反向探针：确认 repo、README、TOOLS、Telegram 发送文本里不再命中 raw 片段；
3. 保留 hash-only stub：只保留 `receiptId + contentDigest + schemaPin + decisionSummary`;
4. 到期后把逐条 stub 降级成 aggregate counter，例如“2026-06 删除课程敏感片段 1 次”。

这样未来有人问“当时是不是删过”，Agent 能证明；但没人能从证明材料里把原文恢复出来。

---

## 易踩坑

- **裸 hash 当脱敏**：邮箱、手机号、订单号、短文本都容易被字典枚举。
- **stub 没有 TTL**：长期看会变成另一个数据仓库。
- **只证明删了对象存储**：搜索、向量、缓存、workspace、日志索引同样要 probe。
- **审计证明太详细**：证明删除时泄漏了原文，等于删除失败。
- **pepper rotation 断链**：换密钥后旧 stub 完全不可验证，审计能力丢失。

---

## 检查清单

- [ ] stub 不包含 raw value、plain identifier、可逆字段；
- [ ] digest 使用 pepper/version，并能轮换；
- [ ] challenge 覆盖 raw/search/vector/cache/workspace；
- [ ] 能解释 why/when/who，但不能重建 what；
- [ ] 到期后支持 aggregate counter 或删除；
- [ ] 每次 stub 查询、验证、重写、删除都有 receipt。

---

## 结论

删除后的证据治理有一个微妙平衡：**证明自己忘了，但不要靠记住原文来证明**。

Post-Erasure Hash Stub Verification 把删除收据、反向探针、不可逆 digest、保留期限和审计挑战串成闭环。成熟 Agent 不只是会删数据，还能在未来被审计时，安全地证明自己确实删了。
