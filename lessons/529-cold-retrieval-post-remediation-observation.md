# 529. Agent 冷取回修复后的观察窗口与 SLA 再基线闸门

> RetrievalRemediationCloseoutReceipt 只能说明这次修复动作完成了，不能说明冷路径已经重新稳定。成熟 Agent 要在修复后进入观察窗口，持续采样真实读取路径，再决定关闭、延长观察、回滚修复或重新定 SLA 基线。

上一讲讲了 **Cold Retrieval SLA Breach Remediation & Hot Fallback Guard**：冷取回 SLA 违约后，可以修冷索引、修 archive bundle、提供脱敏降级摘要，但不能偷偷复活 hot raw。

今天继续往后走：**修复完成以后，什么时候才算真的恢复？**

常见事故是：

- 索引重建成功，但真实审计查询的 P95 还是抖；
- synthetic drill 通过，但长尾用户的 lookup key 仍然 missing；
- 关闭 remediation 后，降级摘要 TTL 没清理，变成新的热路径；
- 为了达标，团队把 SLA 阈值调宽，却没有留下 rebaseline 证据；
- 修复版本只在一个 worker 生效，其他 runtime 仍然读旧索引。

所以要加一层 **Post-Remediation Observation & SLA Rebaseline Gate**：修复 closeout 后，不直接宣布稳定，而是进入短观察窗口，用真实读路径样本、降级路径清理、runtime 指纹和新旧 SLA 对比来决定能不能关闭。

---

## 1. 核心链路

```
RetrievalRemediationCloseoutReceipt
        ↓
PostRemediationObservationLease
        ↓
ReadPathSampleWindow
        ↓
DegradedFallbackExpiryProbe
        ↓
SlaRebaselineReview
        ↓
ColdRetrievalStabilityReceipt
```

每一环回答一个问题：

- remediation closeout：修复动作是否真的完成；
- observation lease：观察窗口多长、采样哪些 cold path、谁负责；
- read path sample：真实请求是否恢复，不只看 synthetic drill；
- fallback expiry probe：临时脱敏摘要是否按 TTL 退出；
- SLA rebaseline：阈值变化是合理新基线，还是掩盖退化；
- stability receipt：关闭时要证明冷路径稳定、热路径仍缺席。

一句话：**修复不是终点，稳定才是终点；再基线不是改阈值，而是带证据解释为什么新阈值仍然满足审计读取责任。**

---

## 2. learn-claude-code：观察闸门纯函数

教学版先写一个纯函数。它不查 Redis、不跑 worker，只根据收据和观察样本决定下一步。

```python
from dataclasses import dataclass
from typing import Literal

Decision = Literal[
    "close_stable",
    "continue_observation",
    "extend_observation",
    "rollback_repair",
    "reopen_remediation",
    "approve_sla_rebaseline",
    "manual_review",
]

@dataclass
class RemediationCloseout:
    verified: bool
    repair_type: Literal["index", "bundle", "lease_policy", "degraded_summary"]
    synthetic_retrieval_passed: bool
    hot_fallback_used: bool
    runtime_fingerprint: str

@dataclass
class ObservationWindow:
    elapsed_minutes: int
    required_minutes: int
    sample_count: int
    min_samples: int
    p95_ms: int
    error_rate: float
    missing_lookup_keys: int
    lease_denials: int
    runtime_fingerprint_mismatch: bool
    degraded_fallback_active: bool

@dataclass
class SlaReview:
    rebaseline_requested: bool
    old_p95_ms: int
    proposed_p95_ms: int
    reason: str
    owner_approved: bool
    hot_absence_verified: bool

def decide_post_remediation_observation(
    closeout: RemediationCloseout,
    window: ObservationWindow,
    sla: SlaReview,
) -> Decision:
    if not closeout.verified or closeout.hot_fallback_used:
        return "manual_review"

    if not closeout.synthetic_retrieval_passed:
        return "reopen_remediation"

    if window.runtime_fingerprint_mismatch:
        return "rollback_repair"

    if window.degraded_fallback_active and window.elapsed_minutes >= window.required_minutes:
        return "reopen_remediation"

    if window.missing_lookup_keys > 0 or window.lease_denials > 0:
        return "reopen_remediation"

    if window.error_rate > 0.01:
        return "extend_observation"

    if window.elapsed_minutes < window.required_minutes or window.sample_count < window.min_samples:
        return "continue_observation"

    if sla.rebaseline_requested:
        if not sla.owner_approved or not sla.hot_absence_verified:
            return "manual_review"
        # 新 P95 不能无限放宽。这里用 2x 做教学阈值，生产中应按业务 SLA 配置。
        if sla.proposed_p95_ms > sla.old_p95_ms * 2:
            return "manual_review"
        return "approve_sla_rebaseline"

    if window.p95_ms > sla.old_p95_ms:
        return "extend_observation"

    return "close_stable"
```

这个函数的重点不是阈值，而是顺序：

1. 先确认没有 hot fallback；
2. 再确认 synthetic drill 通过；
3. 再看真实读路径样本；
4. 最后才允许讨论 SLA rebaseline。

最危险的测试是“修复看起来好了，但降级摘要还活着”：

```python
def test_reopens_when_degraded_fallback_survives_observation_window():
    closeout = RemediationCloseout(
        verified=True,
        repair_type="index",
        synthetic_retrieval_passed=True,
        hot_fallback_used=False,
        runtime_fingerprint="idx-v12",
    )
    window = ObservationWindow(
        elapsed_minutes=60,
        required_minutes=30,
        sample_count=400,
        min_samples=100,
        p95_ms=900,
        error_rate=0.0,
        missing_lookup_keys=0,
        lease_denials=0,
        runtime_fingerprint_mismatch=False,
        degraded_fallback_active=True,
    )
    sla = SlaReview(
        rebaseline_requested=False,
        old_p95_ms=1500,
        proposed_p95_ms=1500,
        reason="",
        owner_approved=False,
        hot_absence_verified=True,
    )

    assert decide_post_remediation_observation(closeout, window, sla) == "reopen_remediation"
```

降级路径是临时止血，不是新架构。观察窗口结束时它还活着，就说明 repair 没有真正接回正常冷路径。

---

## 3. pi-mono：Post-Remediation Observer

生产版建议把观察逻辑做成独立 worker，而不是塞进修复 worker 里。

```typescript
type ObservationDecision =
  | "close_stable"
  | "continue_observation"
  | "extend_observation"
  | "rollback_repair"
  | "reopen_remediation"
  | "approve_sla_rebaseline"
  | "manual_review";

interface PostRemediationObservationJob {
  coldPathId: string;
  remediationCloseoutId: string;
  observationLeaseId: string;
}

class ColdRetrievalPostRemediationObserver {
  constructor(
    private readonly receipts: ReceiptStore,
    private readonly metrics: ColdRetrievalMetrics,
    private readonly fallback: DegradedFallbackStore,
    private readonly runtime: RuntimeFingerprintStore,
    private readonly rollback: ColdRepairRollbackService,
    private readonly audit: AuditLog,
  ) {}

  async run(job: PostRemediationObservationJob) {
    const closeout = await this.receipts.getRemediationCloseout(
      job.remediationCloseoutId,
    );
    const lease = await this.receipts.getObservationLease(
      job.observationLeaseId,
    );

    const [sampleWindow, fallbackState, runtimeState, slaReview] =
      await Promise.all([
        this.metrics.collectReadPathWindow({
          coldPathId: job.coldPathId,
          from: lease.startedAt,
          to: new Date().toISOString(),
        }),
        this.fallback.getState(job.coldPathId),
        this.runtime.compare({
          expected: closeout.runtimeFingerprint,
          scope: lease.runtimeScope,
        }),
        this.receipts.getPendingSlaRebaselineReview(job.coldPathId),
      ]);

    const decision = decidePostRemediationObservation(
      closeout,
      {
        ...sampleWindow,
        degradedFallbackActive: fallbackState.active,
        runtimeFingerprintMismatch: runtimeState.hasMismatch,
        elapsedMinutes: lease.elapsedMinutes(),
        requiredMinutes: lease.requiredMinutes,
        minSamples: lease.minSamples,
      },
      slaReview,
    );

    await this.audit.append({
      type: "cold_retrieval_post_remediation_observation_decision",
      coldPathId: job.coldPathId,
      remediationCloseoutId: job.remediationCloseoutId,
      observationLeaseId: job.observationLeaseId,
      decision,
      sampleWindow,
      fallbackActive: fallbackState.active,
      runtimeMismatches: runtimeState.mismatches,
    });

    if (decision === "continue_observation") {
      await this.receipts.touchObservationLease(job.observationLeaseId);
      return;
    }

    if (decision === "extend_observation") {
      await this.receipts.extendObservationLease({
        observationLeaseId: job.observationLeaseId,
        reason: "read_path_not_stable_enough",
        extraMinutes: 30,
      });
      return;
    }

    if (decision === "rollback_repair") {
      await this.rollback.rollbackToLastKnownGood({
        coldPathId: job.coldPathId,
        reason: "runtime_fingerprint_mismatch_after_repair",
      });
      await this.receipts.openRemediationCase({
        coldPathId: job.coldPathId,
        reason: "post_remediation_runtime_mismatch",
      });
      return;
    }

    if (decision === "reopen_remediation") {
      await this.receipts.openRemediationCase({
        coldPathId: job.coldPathId,
        reason: "post_remediation_observation_failed",
      });
      return;
    }

    if (decision === "approve_sla_rebaseline") {
      await this.receipts.writeSlaRebaselineReceipt({
        coldPathId: job.coldPathId,
        sourceCloseoutId: job.remediationCloseoutId,
        approvedAt: new Date().toISOString(),
      });
    }

    if (decision === "manual_review") {
      await this.receipts.openIncident({
        coldPathId: job.coldPathId,
        reason: "post_remediation_observation_requires_review",
      });
      return;
    }

    await this.receipts.writeColdRetrievalStabilityReceipt({
      coldPathId: job.coldPathId,
      remediationCloseoutId: job.remediationCloseoutId,
      observationLeaseId: job.observationLeaseId,
      closedAt: new Date().toISOString(),
      hotAbsenceVerified: true,
      degradedFallbackExpired: !fallbackState.active,
      runtimeFingerprint: closeout.runtimeFingerprint,
    });
  }
}
```

这里有两个工程要点：

- `Promise.all` 只并发读取观测数据，不并发写 closeout。写状态要走单点事务，避免两个 worker 同时关闭和重开；
- `approve_sla_rebaseline` 只是写 rebaseline receipt，后面仍然要写 stability receipt，把“阈值变更”和“路径稳定”绑定在同一条证据链上。

---

## 4. SLA 再基线不是改配置

冷路径从热存储迁到冷归档后，延迟通常会升高。合理的 rebaseline 是允许的，但必须有证据：

```typescript
interface SlaRebaselineReview {
  coldPathId: string;
  oldSla: {
    p95Ms: number;
    errorRateMax: number;
    missingKeyMax: number;
  };
  proposedSla: {
    p95Ms: number;
    errorRateMax: number;
    missingKeyMax: number;
  };
  evidence: {
    sampleCount: number;
    observationWindowMinutes: number;
    consumerAckComplete: boolean;
    hotAbsenceVerified: boolean;
    degradedFallbackExpired: boolean;
  };
  ownerApprovalRef: string;
}

function validateSlaRebaseline(review: SlaRebaselineReview): string[] {
  const problems: string[] = [];

  if (review.evidence.sampleCount < 100) {
    problems.push("not_enough_real_read_path_samples");
  }
  if (!review.evidence.consumerAckComplete) {
    problems.push("consumer_ack_incomplete");
  }
  if (!review.evidence.hotAbsenceVerified) {
    problems.push("hot_absence_not_verified");
  }
  if (!review.evidence.degradedFallbackExpired) {
    problems.push("degraded_fallback_still_active");
  }
  if (review.proposedSla.missingKeyMax > 0) {
    problems.push("missing_lookup_key_cannot_be_normalized");
  }
  if (review.proposedSla.p95Ms > review.oldSla.p95Ms * 2) {
    problems.push("p95_rebaseline_too_large_for_auto_approval");
  }

  return problems;
}
```

注意 `missingKeyMax`。延迟可以重新定基线，但 **missing lookup key 不能被正常化**。找不到证据不是慢，是审计责任断链。

---

## 5. OpenClaw 实战：课程 Cron 的类比

这个课程自动化也有 post-remediation observation。

比如某次 cron 发现：

- lesson 文件已写入；
- README 目录项漏了；
- Telegram 已经发出；
- git commit 成功。

修复 README 后，不能只说“补了目录项”。还要观察：

1. README link 能不能打开 lesson；
2. git status 是否干净；
3. remote 是否包含新 commit；
4. TOOLS.md 已讲清单是否同步；
5. 下一次选题是否不会重复。

对应到冷证据治理：

- lesson 文件 = cold bundle；
- README = cold index；
- Telegram message = downstream consumer；
- git commit = immutable version point；
- TOOLS.md = topic ledger；
- cron 下一轮不重复 = recurrence watch。

如果只修 README，不验证下一轮不会重复，那只是“局部修复”。如果修完后还要从草稿里临时复制摘要给群里，那就是 degraded fallback 没退出。

---

## 6. 常见坑

**坑 1：修复 closeout 等于稳定 closeout。**

修复 closeout 只证明动作完成，稳定 closeout 要证明真实读路径连续稳定。

**坑 2：只跑 synthetic drill，不看真实样本。**

synthetic drill 覆盖标准路径，真实样本覆盖历史边角、长尾 consumer 和旧 lookup key。

**坑 3：把 SLA rebaseline 当成调大超时。**

rebaseline 必须绑定 owner approval、样本窗口、consumer ack 和 hot absence。否则就是把退化写进配置。

**坑 4：降级摘要没有退出证明。**

降级摘要存在一天，就多一天热路径风险。观察关闭前必须证明它已过期或被清理。

**坑 5：runtime 指纹不一致还继续观察。**

不同 worker 跑不同索引版本时，样本指标没有意义。先 rollback 或 converge runtime，再谈稳定。

---

## 7. 检查清单

- [ ] remediation closeout 后创建 observation lease；
- [ ] 观察窗口同时采集 P95、error rate、missing key、lease denial；
- [ ] 真实 read path sample 数量达到最小样本；
- [ ] synthetic retrieval drill 和真实样本都通过；
- [ ] degraded summary fallback 已按 TTL 退出；
- [ ] runtime fingerprint 全部收敛；
- [ ] SLA rebaseline 有 owner approval 和证据包；
- [ ] missing lookup key 不允许被 rebaseline 正常化；
- [ ] stability receipt 绑定 remediation closeout、observation lease 和 SLA review；
- [ ] close 后仍保留短期 recurrence watch。

---

## 总结

冷路径修复后的最大风险，不是修不好，而是“看起来好了就关单”。

成熟 Agent 会把 remediation closeout、观察租约、真实读取样本、降级路径退出、SLA 再基线和稳定收据串起来。这样系统不只是恢复可用，还能证明：冷证据仍然可找、可验、可解释、可限权读取，而且没有把已经退出运行面的热证据悄悄带回来。
