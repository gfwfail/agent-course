# 519 - Agent 差分隐私发布可复现性与种子托管闸门

> DP 发布不是“给数字加个随机数”就结束。发布后如果被审计、申诉或复盘，团队需要证明当时的 noisy result 是按批准参数生成的；但如果把随机种子明文留下，又可能被拿来反推出原始值。成熟 Agent 要把可复现性和不可反推性拆开，用 seed escrow、commitment 和一次性审计许可连接起来。

上一讲讲了跨数据集关联边界：多份已经加噪的数据也不能随便 join。今天继续补上 DP 发布链路里很容易被忽略的一环：**Release Reproducibility & Seed Escrow Gate**。

一句话：**DP 结果要能被授权审计复现，但不能让任何普通读者拿到反推原值的钥匙。**

---

## 1. 为什么“留种子”很危险？

很多团队为了 debug，会这样保存 DP 发布记录：

```json
{
  "query": "daily_active_agents",
  "raw_value": 42,
  "epsilon": 0.5,
  "noise": -3,
  "seed": "plain-random-seed",
  "published_value": 39
}
```

这对工程排查很方便，但隐私上很危险：

- `raw_value` 不该跟发布记录长期绑定；
- `noise` 和 `seed` 一旦泄露，就能推出 raw；
- 多次重跑如果 seed 不稳定，会得到不同 noisy result，审计无法复现；
- 多次重跑如果 seed 明文稳定，又变成反推入口。

正确做法是把发布拆成三份：

- **公开发布收据**：只放 noisy result、epsilon、mechanism、seed commitment；
- **隔离 seed escrow**：加密保存 seed，只给审计 gate 一次性读取；
- **复现审计收据**：记录复现是否匹配，不暴露 raw 和 seed。

链路长这样：

```text
DifferentialPrivacyReleaseReceipt
        ↓
SeedCommitment
        ↓
EncryptedSeedEscrow
        ↓
ReproducibilityAuditRequest
        ↓
OneTimeAuditPermit
        ↓
ReproducibilityCloseoutReceipt
```

---

## 2. 闸门要检查什么？

Release Reproducibility Gate 主要回答六个问题：

- seed 是否用 KMS/密钥包加密托管，而不是明文进日志；
- seed commitment 是否能证明“这是同一个 seed”，但不能反推出 seed；
- audit request 是否有目的、owner、TTL 和 case id；
- 复现过程是否在隔离 runner 里执行，禁止导出 raw；
- noisy result 是否和发布收据一致；
- 审计结束后是否写 closeout，并销毁临时 seed material。

重点：普通业务查询只应该读 noisy result；只有审计路径可以短时解封 seed，而且结果也只能输出“匹配/不匹配”和 receipt。

---

## 3. learn-claude-code：可复现性闸门纯函数

教学版先写一个纯函数：它不解密 seed，只判断一次审计请求能不能拿到复现许可。

```python
from dataclasses import dataclass
from typing import Literal

Decision = Literal[
    "issue_one_time_audit_permit",
    "deny_missing_commitment",
    "deny_plaintext_seed",
    "deny_purpose_mismatch",
    "manual_review_expired_release",
]

@dataclass(frozen=True)
class DpReleaseReceipt:
    release_id: str
    purpose: str
    mechanism: str
    epsilon: float
    published_value: int
    seed_commitment: str | None
    seed_storage: Literal["none", "plaintext", "encrypted_escrow"]
    expired: bool

@dataclass(frozen=True)
class AuditRequest:
    case_id: str
    purpose: str
    owner_approved: bool
    ttl_minutes: int

@dataclass(frozen=True)
class ReproPolicy:
    max_ttl_minutes: int
    allowed_mechanisms: tuple[str, ...]

@dataclass(frozen=True)
class ReproDecision:
    decision: Decision
    reason: str
    permit_scope: str | None = None

def decide_reproducibility_audit(
    receipt: DpReleaseReceipt,
    request: AuditRequest,
    policy: ReproPolicy,
) -> ReproDecision:
    if receipt.seed_storage == "plaintext":
        return ReproDecision(
            "deny_plaintext_seed",
            "release stores seed material in plaintext",
        )

    if not receipt.seed_commitment:
        return ReproDecision(
            "deny_missing_commitment",
            "cannot prove seed identity without exposing it",
        )

    if receipt.purpose != request.purpose or not request.owner_approved:
        return ReproDecision(
            "deny_purpose_mismatch",
            "audit request is not approved for the release purpose",
        )

    if receipt.expired or request.ttl_minutes > policy.max_ttl_minutes:
        return ReproDecision(
            "manual_review_expired_release",
            "audit needs manual review due to expiry or excessive ttl",
        )

    if receipt.mechanism not in policy.allowed_mechanisms:
        return ReproDecision(
            "manual_review_expired_release",
            "mechanism is not covered by automated reproducibility policy",
        )

    return ReproDecision(
        "issue_one_time_audit_permit",
        "release can be reproduced in isolated audit runner",
        permit_scope=f"release:{receipt.release_id}:case:{request.case_id}",
    )
```

最小测试：

```python
def test_plaintext_seed_is_denied():
    decision = decide_reproducibility_audit(
        DpReleaseReceipt(
            release_id="rel_1",
            purpose="ops_report",
            mechanism="laplace",
            epsilon=0.5,
            published_value=39,
            seed_commitment="sha256:abc",
            seed_storage="plaintext",
            expired=False,
        ),
        AuditRequest("case_7", "ops_report", True, 30),
        ReproPolicy(60, ("laplace", "gaussian")),
    )
    assert decision.decision == "deny_plaintext_seed"

def test_valid_request_gets_one_time_permit():
    decision = decide_reproducibility_audit(
        DpReleaseReceipt(
            release_id="rel_2",
            purpose="ops_report",
            mechanism="laplace",
            epsilon=0.5,
            published_value=39,
            seed_commitment="sha256:abc",
            seed_storage="encrypted_escrow",
            expired=False,
        ),
        AuditRequest("case_8", "ops_report", True, 15),
        ReproPolicy(60, ("laplace", "gaussian")),
    )
    assert decision.decision == "issue_one_time_audit_permit"
    assert decision.permit_scope == "release:rel_2:case:case_8"
```

这就是 learn-claude-code 里最值得练的设计：LLM 不需要“相信自己会谨慎”，而是让每个危险分支都有明确 decision。

---

## 4. pi-mono：SeedEscrowWorker

生产版可以把 seed escrow 做成 DP release pipeline 的后置 worker。发布时只把 commitment 写入主收据，seed 进隔离存储。

```ts
type DpReleaseReceipt = {
  releaseId: string;
  purpose: string;
  mechanism: "laplace" | "gaussian";
  epsilon: number;
  publishedValue: number;
  seedCommitment: string;
  seedEscrowRef: string;
};

type AuditPermit = {
  permitId: string;
  releaseId: string;
  caseId: string;
  expiresAt: string;
  consumedAt?: string;
};

class SeedEscrowWorker {
  constructor(
    private readonly escrow: SeedEscrowStore,
    private readonly permits: AuditPermitStore,
    private readonly outbox: EventOutbox,
  ) {}

  async writeEscrow(input: {
    releaseId: string;
    seed: Uint8Array;
    purpose: string;
  }): Promise<{ seedCommitment: string; seedEscrowRef: string }> {
    const seedCommitment = await sha256Hex(input.seed);
    const seedEscrowRef = await this.escrow.encryptAndStore({
      releaseId: input.releaseId,
      purpose: input.purpose,
      seed: input.seed,
    });

    await this.outbox.publish("dp.seed_escrow_written", {
      releaseId: input.releaseId,
      seedCommitment,
      seedEscrowRef,
    });

    return { seedCommitment, seedEscrowRef };
  }

  async consumePermitAndReproduce(input: {
    permitId: string;
    receipt: DpReleaseReceipt;
  }): Promise<"matched" | "mismatch"> {
    const permit = await this.permits.consumeOnce(input.permitId);
    if (permit.releaseId !== input.receipt.releaseId) {
      throw new Error("permit_release_mismatch");
    }

    const seed = await this.escrow.decryptForAudit(input.receipt.seedEscrowRef);
    const commitment = await sha256Hex(seed);
    if (commitment !== input.receipt.seedCommitment) {
      throw new Error("seed_commitment_mismatch");
    }

    const reproduced = runDpMechanism({
      mechanism: input.receipt.mechanism,
      epsilon: input.receipt.epsilon,
      seed,
    });

    await zeroize(seed);

    return reproduced === input.receipt.publishedValue ? "matched" : "mismatch";
  }
}
```

几个生产细节：

- `consumeOnce` 必须是事务/CAS，防止多个 worker 同时复现；
- `decryptForAudit` 只能在隔离 runner 中调用，普通 API 没权限；
- `zeroize(seed)` 不是形式主义，至少要避免 seed 长时间留在热内存和日志；
- mismatch 不自动暴露 raw，要进入 incident/review 队列。

---

## 5. OpenClaw：Cron 发布审计怎么落地？

OpenClaw 的课程 cron 本身就是一个好类比：每次发布 lesson 后，我们会检查 lesson、README、TOOLS、Git commit、Telegram 发送是否对账。

DP 发布也一样。可以让一个 OpenClaw cron 每天抽样审计 DP release：

```text
1. 读取最近 24h 的 DifferentialPrivacyReleaseReceipt
2. 只抽样有 seedCommitment + encrypted escrow 的 release
3. 为每个样本生成 ReproducibilityAuditRequest
4. Gate 通过后发一次性 AuditPermit
5. 隔离 runner 复现 noisy result
6. 写 ReproducibilityCloseoutReceipt
7. mismatch 才通知 owner，matched 只进审计日志
```

这个 cron 不应该把 seed、raw value、noise 明文发到群里，也不应该把复现细节塞进 LLM prompt。Agent 只需要看到结构化结论：

```json
{
  "releaseId": "rel_2026_06_19_001",
  "auditCaseId": "dp_audit_8842",
  "result": "matched",
  "seedMaterialDestroyed": true,
  "permitConsumedOnce": true
}
```

---

## 6. 实战 checklist

做 DP 发布系统时，可以直接拿这 7 条做 code review：

- 发布收据里没有 raw value、noise、plaintext seed；
- 每次 release 都有 seed commitment；
- seed escrow 和业务 DB 分权存储；
- 审计许可一次性、短 TTL、绑定 case id；
- 复现 runner 禁止外部网络和日志导出 seed；
- 复现结果只输出 matched/mismatch；
- closeout receipt 记录 seed material 已销毁。

---

## 7. 关键 takeaway

DP 发布要同时满足两件看似矛盾的事：

- **可审计**：将来能证明 noisy result 确实按批准机制生成；
- **不可反推**：审计证据本身不能变成还原 raw value 的钥匙。

Seed Escrow Gate 的价值，就是把“复现能力”从普通数据访问里拆出来，变成一次性、有许可、有隔离、有销毁收据的审计流程。成熟 Agent 不只会发布隐私统计，还会证明自己发布得对，而且证明过程本身也不泄密。
