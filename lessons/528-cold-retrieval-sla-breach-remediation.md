# 528. Agent 冷取回 SLA 违约修复与热路径回退闸门

> 冷路径激活后，不代表它会一直可用。真正成熟的 Agent 要在冷取回变慢、失败、索引漂移时修复冷路径，而不是偷偷把热证据复活。

上一讲讲了 **Cold Handoff Index Activation & Retrieval SLA Gate**：冷归档交接后，要激活冷索引、绑定读取租约、做合成取回演练，并证明热路径已经退出。

今天继续往后走：**冷路径上线后如果 SLA 违约怎么办？**

常见事故是：

- 冷索引 P95 变高，审计请求开始超时；
- bundle 还在，但索引 lookup key 漂移，按 decisionId 查不到；
- worker 为了“先恢复业务”，临时读回旧热缓存；
- 失败重试不断扩大读取范围，越查越多；
- 修复完成后没有证明热 fallback 没被使用过。

所以要加一层 **Cold Retrieval SLA Breach Remediation Gate**：SLA 违约可以触发修复、重建、降级查询或人工升级，但不能默认 resurrect hot evidence。

---

## 1. 核心模型

```
ColdPathActivationReceipt
        ↓
RetrievalSlaBreachSignal
        ↓
ColdPathRepairPlan
        ↓
RepairExecutionReceipt
        ↓
HotFallbackGuardProbe
        ↓
RetrievalRemediationCloseoutReceipt
```

关键判断：

- breach 是延迟、错误率、索引缺失、bundle 损坏，还是权限租约失败；
- 是否已有可验证的 cold activation receipt；
- 修复动作是否只碰冷索引和冷 bundle，不能复活 hot raw；
- fallback 是否只能走 redacted summary / hash stub / aggregate counter；
- 修复后是否重新跑 synthetic retrieval drill；
- 是否写出 closeout receipt，证明热路径没有被偷偷使用。

---

## 2. learn-claude-code：SLA 违约修复判定器

教学版用纯函数表达闸门。它不修索引，只决定下一步动作。

```python
from dataclasses import dataclass
from typing import Literal

Decision = Literal[
    "repair_cold_index",
    "repair_archive_bundle",
    "serve_degraded_summary",
    "deny_hot_fallback",
    "manual_review",
    "close_remediated",
]

@dataclass
class ActivationReceipt:
    verified: bool
    cold_index_active: bool
    hot_absence_verified: bool
    retrieval_lease_policy_active: bool

@dataclass
class BreachSignal:
    p95_ms: int
    error_rate: float
    missing_lookup_keys: int
    bundle_hash_mismatch: bool
    lease_denials: int
    affected_requests: int

@dataclass
class RepairState:
    repair_attempts: int
    index_rebuild_passed: bool
    bundle_repair_passed: bool
    synthetic_retrieval_passed: bool
    hot_fallback_requested: bool
    hot_fallback_used: bool
    degraded_summary_available: bool

def decide_cold_retrieval_remediation(
    activation: ActivationReceipt,
    breach: BreachSignal,
    repair: RepairState,
) -> Decision:
    if not activation.verified:
        return "manual_review"

    if not activation.hot_absence_verified:
        return "deny_hot_fallback"

    if repair.hot_fallback_used:
        return "manual_review"

    if repair.hot_fallback_requested:
        if repair.degraded_summary_available:
            return "serve_degraded_summary"
        return "deny_hot_fallback"

    if breach.bundle_hash_mismatch:
        return "repair_archive_bundle"

    if breach.missing_lookup_keys > 0:
        return "repair_cold_index"

    if breach.p95_ms > 5000 or breach.error_rate > 0.02:
        if repair.repair_attempts >= 3:
            return "manual_review"
        return "repair_cold_index"

    if breach.lease_denials > 0 and not activation.retrieval_lease_policy_active:
        return "manual_review"

    if repair.index_rebuild_passed and repair.synthetic_retrieval_passed:
        return "close_remediated"

    return "repair_cold_index"
```

这里最重要的是两个规则：

1. **hot fallback requested 不等于可以读 hot raw**。只能降级到 redacted summary、hash stub 或 aggregate counter。
2. **hot fallback used 必须 manual_review**。这说明隔离边界已经被突破，要开事故，不是继续自动修。

测试要覆盖最危险的场景：

```python
def test_denies_hot_fallback_even_when_cold_path_is_slow():
    activation = ActivationReceipt(
        verified=True,
        cold_index_active=True,
        hot_absence_verified=True,
        retrieval_lease_policy_active=True,
    )
    breach = BreachSignal(
        p95_ms=12000,
        error_rate=0.08,
        missing_lookup_keys=0,
        bundle_hash_mismatch=False,
        lease_denials=0,
        affected_requests=40,
    )
    repair = RepairState(
        repair_attempts=1,
        index_rebuild_passed=False,
        bundle_repair_passed=False,
        synthetic_retrieval_passed=False,
        hot_fallback_requested=True,
        hot_fallback_used=False,
        degraded_summary_available=False,
    )

    assert decide_cold_retrieval_remediation(
        activation,
        breach,
        repair,
    ) == "deny_hot_fallback"
```

冷路径慢，最多说明冷路径要修。它不自动证明旧热证据可以复活。

---

## 3. pi-mono：Remediation Worker

生产版建议把修复拆成三层：

- `SlaMonitor`：持续读取 P95、error rate、missing key；
- `RemediationPlanner`：把 breach 转成修复计划；
- `HotFallbackGuard`：拦截任何试图读取旧热路径的请求。

```typescript
type RemediationDecision =
  | "repair_cold_index"
  | "repair_archive_bundle"
  | "serve_degraded_summary"
  | "deny_hot_fallback"
  | "manual_review"
  | "close_remediated";

interface ColdRetrievalBreachJob {
  coldPathId: string;
  activationReceiptId: string;
  windowStartedAt: string;
  windowEndedAt: string;
}

class ColdRetrievalRemediationWorker {
  constructor(
    private readonly receipts: ReceiptStore,
    private readonly metrics: RetrievalMetricsStore,
    private readonly index: ColdIndexRepairService,
    private readonly archive: ColdArchiveRepairService,
    private readonly fallback: DegradedFallbackStore,
    private readonly audit: AuditLog,
  ) {}

  async run(job: ColdRetrievalBreachJob) {
    const activation = await this.receipts.getActivationReceipt(
      job.activationReceiptId,
    );
    const breach = await this.metrics.getBreachSignal({
      coldPathId: job.coldPathId,
      from: job.windowStartedAt,
      to: job.windowEndedAt,
    });
    const repair = await this.receipts.getRepairState(job.coldPathId);

    const decision = decideColdRetrievalRemediation(
      activation,
      breach,
      repair,
    );

    await this.audit.append({
      type: "cold_retrieval_remediation_decision",
      coldPathId: job.coldPathId,
      decision,
      breach,
      activationReceiptId: job.activationReceiptId,
    });

    if (decision === "repair_cold_index") {
      const receipt = await this.index.rebuild({
        coldPathId: job.coldPathId,
        sourceReceiptId: job.activationReceiptId,
      });
      await this.receipts.save(receipt);
      return;
    }

    if (decision === "repair_archive_bundle") {
      const receipt = await this.archive.verifyAndRepair({
        coldPathId: job.coldPathId,
        sourceReceiptId: job.activationReceiptId,
      });
      await this.receipts.save(receipt);
      return;
    }

    if (decision === "serve_degraded_summary") {
      await this.fallback.enableRedactedSummaryOnly({
        coldPathId: job.coldPathId,
        ttlMinutes: 30,
        reason: "cold_retrieval_sla_breach",
      });
      return;
    }

    if (decision === "deny_hot_fallback") {
      await this.audit.append({
        type: "hot_fallback_denied",
        coldPathId: job.coldPathId,
        reason: "hot_raw_resurrection_forbidden",
      });
      return;
    }

    if (decision === "manual_review") {
      await this.receipts.openIncident({
        coldPathId: job.coldPathId,
        reason: "cold_retrieval_remediation_requires_review",
      });
      return;
    }

    await this.receipts.closeRemediation({
      coldPathId: job.coldPathId,
      activationReceiptId: job.activationReceiptId,
      closedAt: new Date().toISOString(),
    });
  }
}
```

注意：`serve_degraded_summary` 不是热路径回退。它只能读取已经脱敏、已授权、低粒度的派生产物，并且要有 TTL。

---

## 4. HotFallbackGuard：运行时硬拦截

真正挡事故的不是 worker，而是工具分发层的 guard。

```typescript
class HotFallbackGuard {
  constructor(private readonly receipts: ReceiptStore) {}

  async assertAllowed(request: {
    coldPathId: string;
    requestedSource: "cold_index" | "redacted_summary" | "hot_raw";
    purpose: string;
  }) {
    if (request.requestedSource !== "hot_raw") {
      return;
    }

    const closeout = await this.receipts.getLatestRemediationCloseout(
      request.coldPathId,
    );

    await this.receipts.appendViolation({
      type: "hot_raw_fallback_attempt",
      coldPathId: request.coldPathId,
      purpose: request.purpose,
      previousCloseoutId: closeout?.id,
    });

    throw new Error(
      "Hot raw fallback is forbidden after cold handoff. Use cold repair or degraded summary.",
    );
  }
}
```

这层 guard 要放在所有读取工具前面，包括 debug 工具、admin repair 工具和 replay runner。否则最容易从“排查方便”绕过治理边界。

---

## 5. OpenClaw 实战：课程 Cron 的类比

这个课程自动化本身就是一个小型证据链：

- lesson 文件是主证据；
- README 是索引；
- TOOLS.md 是已讲清单；
- Telegram message 是外部发布；
- git commit 是不可变版本点。

如果 README 索引损坏，正确修复是重建 README 索引和校验 commit，不应该从旧聊天记录里随便复制一段未校验内容当作“热回退”。

对应到冷证据：

1. cron 发现 lesson 文件存在但 README 缺目录项；
2. 生成 repair plan：只修 README 索引；
3. 重新校验 lesson path、commit hash、Telegram message id；
4. 写 closeout：索引已恢复，未从未授权来源补数据；
5. 如果需要临时展示，只展示 lesson 摘要，不展示 raw debug scratch。

这就是 `serve_degraded_summary` 的含义：系统可以降级服务，但不能越权恢复已退出运行面的原始数据。

---

## 6. 常见坑

**坑 1：把冷路径慢当成热路径复活理由。**

冷路径慢是运维问题，热路径复活是治理边界问题。两者不能混在一起。

**坑 2：只监控错误率，不监控 missing lookup key。**

索引漂移时可能没有大量 500，只是查不到历史对象。missing key 是冷路径最重要的早期信号。

**坑 3：降级摘要没有 TTL。**

降级路径如果不自动过期，很快会变成新的热路径。

**坑 4：修复后不做 synthetic retrieval drill。**

重建索引成功不等于审计查询成功。必须用真实 lookup key 抽样演练。

---

## 7. 检查清单

- [ ] 冷路径有 P95、error rate、missing key、lease denial 四类指标；
- [ ] breach 会生成 repair plan，而不是直接扩大读取范围；
- [ ] hot raw fallback 在工具分发层硬拦截；
- [ ] 降级摘要只读脱敏派生产物，并带 TTL；
- [ ] 修复后重新跑 synthetic retrieval drill；
- [ ] closeout receipt 证明热路径未被使用；
- [ ] hot fallback attempt 会开 violation case。

---

## 总结

冷路径激活只是开始，SLA 违约才是系统边界的真正考试。

成熟 Agent 的冷证据治理，不是“冷路径坏了就临时读热数据”，而是把违约信号、修复计划、降级摘要、热回退拦截和关闭收据串成闭环。这样系统既能恢复可用性，也不会把已经退出运行面的敏感证据重新带回生产。

