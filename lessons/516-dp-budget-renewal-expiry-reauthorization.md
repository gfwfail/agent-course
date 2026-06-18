# 516 - Agent 差分隐私预算续期、过期与再授权闸门

> 上一讲讲了 DP release worker：统计值要发布时，先消耗 epsilon、采样噪声、保存 seed commitment 和 release receipt。但生产里还有一个更容易被忽略的闭环：privacy budget 不是无限资源，耗尽后不能靠“明天自动清零”糊弄过去，也不能让旧 noisy release 永久复用到失真。

今天讲 **DP Budget Renewal, Expiry & Re-authorization Gate**：Agent 要把 `PrivacyBudgetLedger`、`DifferentialPrivacyReleaseReceipt`、`BudgetExpiryReview`、`RenewalRequest`、`ReauthorizationDecision` 和 `DpBudgetRenewalReceipt` 串起来，决定继续复用 noisy result、刷新预算、强制粗化维度、暂停发布，还是进入人工审批。

常见坑：

- epsilon 用完后 fallback 成精确统计，等于把隐私保护关了；
- 每天零点自动重置预算，但不看昨天是否被同一 query 打穿；
- noisy release 永久缓存，业务以为是最新统计，其实是旧快照；
- 预算续期只看 consumer，不看 purpose / counter / dimension fingerprint；
- owner 批准了更高预算，却没有记录原因、范围、TTL 和回滚条件；
- LLM 看到 “budget exhausted” 后自己改口说“大约等于 raw count”。

成熟 Agent 要把 DP 预算当成一种**会过期、会续租、会被撤销、需要再授权的能力令牌**。

---

## 核心模型

```
DifferentialPrivacyReleaseReceipt
        ↓
PrivacyBudgetLedger
        ↓
BudgetExpiryReview
        ↓
RenewalRequest
        ↓
ReauthorizationDecision
        ↓
DpBudgetRenewalReceipt
        ↓
ReleaseCacheInvalidationProbe
```

关键判断：

- noisy release 是否还能复用，还是已经超过 staleness TTL；
- epsilon ledger 是正常耗尽、异常高频消耗，还是疑似差分攻击；
- 续期是否只允许相同 purpose / dimensions / consumer；
- 是否需要降低精度：更粗维度、更长时间窗、更高 k-anonymity；
- owner 是否批准了临时预算，批准的 TTL、上限和用途是什么；
- 旧 dashboard cache、导出文件和 LLM context 是否都绑定了新 receipt。

---

## learn-claude-code：预算续期判定器

教学版先写一个纯函数：输入预算账本、最近发布收据和续期请求，输出下一步动作。

```python
from dataclasses import dataclass
from typing import Literal

Decision = Literal[
    "reuse_existing_noisy_release",
    "renew_budget_same_scope",
    "renew_budget_with_coarser_dimensions",
    "deny_budget_exhausted",
    "manual_reauthorization_required",
    "open_suspicious_query_case",
]

@dataclass
class PrivacyBudgetLedger:
    consumer_id: str
    counter_id: str
    purpose: str
    epsilon_remaining: float
    epsilon_daily_limit: float
    spent_last_24h: float
    renewal_count_7d: int
    suspicious_differencing_pattern: bool

@dataclass
class DpReleaseReceipt:
    counter_id: str
    consumer_id: str
    purpose: str
    dimensions_fingerprint: str
    noisy_value_age_minutes: int
    epsilon_cost: float

@dataclass
class RenewalRequest:
    requested_epsilon: float
    purpose: str
    dimensions_fingerprint: str
    public_release: bool
    owner_approved: bool
    owner_approval_ref: str | None
    max_staleness_minutes: int

@dataclass
class RenewalDecision:
    decision: Decision
    epsilon_granted: float
    require_cache_invalidation: bool
    reason: str

def decide_dp_budget_renewal(
    ledger: PrivacyBudgetLedger,
    last_release: DpReleaseReceipt | None,
    request: RenewalRequest,
) -> RenewalDecision:
    if ledger.suspicious_differencing_pattern:
        return RenewalDecision(
            "open_suspicious_query_case",
            0.0,
            True,
            "query pattern looks like differencing attack",
        )

    if last_release:
        same_scope = (
            last_release.consumer_id == ledger.consumer_id
            and last_release.counter_id == ledger.counter_id
            and last_release.purpose == request.purpose
            and last_release.dimensions_fingerprint == request.dimensions_fingerprint
        )
        fresh_enough = last_release.noisy_value_age_minutes <= request.max_staleness_minutes
        if same_scope and fresh_enough:
            return RenewalDecision(
                "reuse_existing_noisy_release",
                0.0,
                False,
                "existing noisy release is still fresh and purpose-bound",
            )

    if request.public_release and not request.owner_approved:
        return RenewalDecision(
            "manual_reauthorization_required",
            0.0,
            False,
            "public release needs owner approval before budget renewal",
        )

    if ledger.renewal_count_7d >= 3:
        return RenewalDecision(
            "renew_budget_with_coarser_dimensions",
            min(request.requested_epsilon, 0.05),
            True,
            "frequent renewals require lower precision",
        )

    projected_spend = ledger.spent_last_24h + request.requested_epsilon
    if projected_spend > ledger.epsilon_daily_limit:
        return RenewalDecision(
            "deny_budget_exhausted",
            0.0,
            False,
            "daily epsilon limit would be exceeded",
        )

    return RenewalDecision(
        "renew_budget_same_scope",
        request.requested_epsilon,
        True,
        "budget can renew under same purpose and dimensions",
    )
```

测试重点：预算耗尽不能偷发精确值；频繁续期不能无限维持同样精度。

```python
def test_frequent_renewals_force_coarser_dimensions():
    ledger = PrivacyBudgetLedger(
        consumer_id="dashboard",
        counter_id="counter_erasure_monthly",
        purpose="ops_report",
        epsilon_remaining=0.0,
        epsilon_daily_limit=1.0,
        spent_last_24h=0.4,
        renewal_count_7d=3,
        suspicious_differencing_pattern=False,
    )
    request = RenewalRequest(
        requested_epsilon=0.2,
        purpose="ops_report",
        dimensions_fingerprint="region_day_v2",
        public_release=False,
        owner_approved=True,
        owner_approval_ref="approval_123",
        max_staleness_minutes=60,
    )

    decision = decide_dp_budget_renewal(ledger, None, request)
    assert decision.decision == "renew_budget_with_coarser_dimensions"
    assert decision.epsilon_granted == 0.05
    assert decision.require_cache_invalidation is True
```

这类函数的价值是把隐私策略从“管理员感觉可以”变成可测的状态机。

---

## pi-mono：DpBudgetRenewalWorker

生产版建议把续期放在 DP release worker 前面。release worker 只负责“这次发布如何加噪”，renewal worker 负责“是否还有资格发布”。

```ts
type RenewalAction =
  | "reuse_existing_noisy_release"
  | "renew_budget_same_scope"
  | "renew_budget_with_coarser_dimensions"
  | "deny_budget_exhausted"
  | "manual_reauthorization_required"
  | "open_suspicious_query_case";

type DpBudgetRenewalJob = {
  consumerId: string;
  counterId: string;
  purpose: string;
  dimensionsFingerprint: string;
  requestedEpsilon: number;
  publicRelease: boolean;
  ownerApprovalRef?: string;
  idempotencyKey: string;
};

type DpBudgetRenewalReceipt = {
  consumerId: string;
  counterId: string;
  purpose: string;
  action: RenewalAction;
  epsilonGranted: number;
  dimensionsFingerprint: string;
  ownerApprovalRef?: string;
  cacheInvalidated: boolean;
  idempotencyKey: string;
  createdAt: string;
};

class DpBudgetRenewalWorker {
  constructor(
    private readonly ledger: PrivacyBudgetStore,
    private readonly releases: DpReleaseReceiptStore,
    private readonly approvals: ApprovalStore,
    private readonly cache: ReleaseCache,
    private readonly cases: PrivacyCaseStore,
    private readonly receipts: DpRenewalReceiptStore,
  ) {}

  async handle(job: DpBudgetRenewalJob): Promise<DpBudgetRenewalReceipt> {
    const existing = await this.receipts.findByIdempotencyKey(job.idempotencyKey);
    if (existing) return existing;

    return this.ledger.withConsumerLease(job.consumerId, async () => {
      const ledger = await this.ledger.get(job.consumerId, job.counterId, job.purpose);
      const lastRelease = await this.releases.findLatestSameCounter(job.consumerId, job.counterId);
      const ownerApproved = job.ownerApprovalRef
        ? await this.approvals.isValid(job.ownerApprovalRef, {
            consumerId: job.consumerId,
            counterId: job.counterId,
            purpose: job.purpose,
          })
        : false;

      const decision = decideDpBudgetRenewal(ledger, lastRelease, {
        requestedEpsilon: job.requestedEpsilon,
        purpose: job.purpose,
        dimensionsFingerprint: job.dimensionsFingerprint,
        publicRelease: job.publicRelease,
        ownerApproved,
        ownerApprovalRef: job.ownerApprovalRef,
        maxStalenessMinutes: 60,
      });

      if (decision.decision === "open_suspicious_query_case") {
        await this.cases.openDifferencingCase(job, ledger);
      }

      if (decision.epsilonGranted > 0) {
        await this.ledger.grantTemporaryBudget({
          consumerId: job.consumerId,
          counterId: job.counterId,
          purpose: job.purpose,
          epsilon: decision.epsilonGranted,
          ownerApprovalRef: job.ownerApprovalRef,
          ttlMinutes: 120,
        });
      }

      let cacheInvalidated = false;
      if (decision.requireCacheInvalidation) {
        await this.cache.invalidateByScope({
          consumerId: job.consumerId,
          counterId: job.counterId,
          purpose: job.purpose,
        });
        cacheInvalidated = true;
      }

      const receipt: DpBudgetRenewalReceipt = {
        consumerId: job.consumerId,
        counterId: job.counterId,
        purpose: job.purpose,
        action: decision.decision,
        epsilonGranted: decision.epsilonGranted,
        dimensionsFingerprint: job.dimensionsFingerprint,
        ownerApprovalRef: job.ownerApprovalRef,
        cacheInvalidated,
        idempotencyKey: job.idempotencyKey,
        createdAt: new Date().toISOString(),
      };

      await this.receipts.save(receipt);
      return receipt;
    });
  }
}
```

几个工程细节：

- `withConsumerLease` 必须包住读取、grant、cache invalidation 和 receipt，避免并发续期超发预算；
- owner approval 要绑定 `consumerId/counterId/purpose`，不能拿 A 报表的批准去发布 B 报表；
- `renew_budget_with_coarser_dimensions` 后，下游 release job 要使用新的粗粒度 fingerprint；
- `reuse_existing_noisy_release` 不扣预算，但必须检查 staleness；
- `deny_budget_exhausted` 是正常业务结果，不是异常，不应该让 LLM 自动绕路；
- suspicious case 打开后，最好冻结同 consumer 的相邻维度查询，防止继续差分。

---

## OpenClaw：课程 Cron 里的类比

课程 Cron 每 3 小时发一讲，也可以类比成一种“发布预算”：

- 一次 Telegram 消息是外部可见发布；
- README / TOOLS 更新是索引发布；
- Git commit 是可审计收据；
- 如果某次发送失败重试，不能重复发同一课，也不能只改 README 不发群。

DP 预算续期对应到课程自动化就是：

```json
{
  "course": "agent-course",
  "lesson": 516,
  "releaseScope": "telegram+lesson+readme+tools+git",
  "reuseExistingReceipt": false,
  "renewalReason": "scheduled 3h course slot",
  "idempotencyKey": "agent-course-516-20260619-0330",
  "closeout": "messageId + commitSha + README entry + TOOLS topic"
}
```

如果发现同一个 lesson id 已经发过，就应该复用 receipt 或停止，而不是换个标题再发一遍；如果 Telegram 发出去了但 Git push 失败，就要补偿 Git，而不是重新发群消息。这和 DP 预算续期一样：**先识别 scope，再决定复用、续期、降级或补偿**。

---

## 实战清单

实现 DP 预算续期时，至少检查这些点：

- [ ] ledger 维度至少包含 `consumerId/counterId/purpose`；
- [ ] noisy release cache 有 staleness TTL，不能永久复用；
- [ ] budget exhausted 只能拒绝、复用 fresh noisy result 或进入审批，不能发布 raw；
- [ ] 续期审批绑定 scope、TTL、epsilon 上限和 owner ref；
- [ ] 高频续期自动降精度：粗维度、长窗口、更低 epsilon；
- [ ] suspicious differencing pattern 自动开 case 并冻结相邻查询；
- [ ] grant budget、invalidate cache、save receipt 在同一 lease/事务边界内；
- [ ] LLM 只拿到 action 和 noisy/caveat，不拿 raw count 和内部 ledger 明细。

---

## 一句话总结

> DP 预算不是每天自动回血的计数器，而是带 scope、TTL、审批、降级和收据的隐私能力令牌；预算耗尽后的动作，决定了你的差分隐私是真保护，还是只保护到第一次麻烦为止。
