# 507 - Agent 冷证据再水化演练与解冻许可闸门

> 冷证据保留下来只是第一步。真正遇到审计、事故复盘或回归重放时，系统还要能把冷归档安全解冻到临时热工作区，并证明它可查询、可解释、可销毁。

上一讲讲了 Post-Cleanup Cold Evidence Retention：rollback alias 删除后，旧路径退出运行面，但证据不能消失，要进入冷保留，并通过抽样证明仍然能按 decisionId、receiptId 和 cutoverId 找回解释。

今天继续往后走一步：**冷证据能查到，不等于能在需要时安全恢复使用**。

常见问题是：

- 归档对象存在，但解冻后索引不能重建；
- 冷存储恢复很慢，事故复盘 SLA 被拖垮；
- 解冻后的证据误接入 active runtime，造成旧配置污染生产；
- 审计员拿到 bundle，却缺少 runtime fingerprint、schema adapter 或 redaction profile；
- 临时解冻区用完没有销毁，反而制造新的敏感数据暴露面。

所以需要一个 **Cold Evidence Rehydration Gate**：冷证据要定期做再水化演练；真正解冻前，要拿到 purpose-bound 的 thaw permit；使用后，要生成销毁收据。

---

## 核心模型

```
ColdEvidenceRetentionCloseout
        ↓
RehydrationDrillPlan
        ↓
ColdBundleFetchReceipt
        ↓
SandboxRehydrationRun
        ↓
ThawPermitReadinessReview
        ↓
PurposeBoundThawPermit
        ↓
EphemeralWorkspaceCloseout
```

关键判断：

- 冷证据保留关闭收据是否有效；
- bundle hash、schema version、redaction profile 是否可复算；
- 再水化是否只进入隔离 sandbox，不能连 active runtime；
- 查询、解释、回归重放三类动作是否都通过；
- thaw permit 是否绑定用途、时间、操作者、数据范围；
- 临时热工作区是否按 TTL 销毁，并留下 closeout receipt。

---

## learn-claude-code：再水化许可判定器

教学版先写纯函数。它不下载对象、不建索引，只判断：这次冷证据能不能解冻、要不要重建索引、要不要阻断访问。

```python
from dataclasses import dataclass
from typing import Literal

Decision = Literal[
    "issue_thaw_permit",
    "rerun_rehydration_drill",
    "rebuild_cold_index",
    "deny_thaw",
    "manual_review",
]

@dataclass
class RetentionCloseout:
    receipt_verified: bool
    cold_bundle_hash: str | None
    schema_version_pinned: bool
    redaction_profile_pinned: bool

@dataclass
class RehydrationRun:
    bundle_fetched: bool
    hash_recomputed: bool
    sandbox_isolated: bool
    active_runtime_blocked: bool
    index_rebuilt: bool
    query_passed: bool
    explanation_passed: bool
    replay_passed: bool
    elapsed_seconds: int

@dataclass
class ThawRequest:
    purpose: Literal["audit", "incident_review", "regression_replay", "legal_hold"]
    requested_decisions: int
    ttl_minutes: int
    requester_authorized: bool
    includes_sensitive_scope: bool
    destruction_plan_ready: bool

def decide_rehydration_thaw(
    closeout: RetentionCloseout,
    run: RehydrationRun,
    request: ThawRequest,
) -> Decision:
    if not closeout.receipt_verified or not closeout.cold_bundle_hash:
        return "deny_thaw"

    if not closeout.schema_version_pinned or not closeout.redaction_profile_pinned:
        return "rebuild_cold_index"

    if not run.bundle_fetched or not run.hash_recomputed:
        return "rerun_rehydration_drill"

    if not run.sandbox_isolated or not run.active_runtime_blocked:
        return "deny_thaw"

    if not run.index_rebuilt:
        return "rebuild_cold_index"

    if not run.query_passed or not run.explanation_passed:
        return "rerun_rehydration_drill"

    if request.purpose == "regression_replay" and not run.replay_passed:
        return "rerun_rehydration_drill"

    if run.elapsed_seconds > 900:
        return "manual_review"

    if not request.requester_authorized or not request.destruction_plan_ready:
        return "deny_thaw"

    if request.includes_sensitive_scope and request.ttl_minutes > 60:
        return "manual_review"

    if request.requested_decisions <= 0 or request.ttl_minutes <= 0:
        return "deny_thaw"

    return "issue_thaw_permit"
```

这里的核心不是算法复杂，而是把“能不能拿历史证据出来用”变成明确的 gate：

- 没有 closeout receipt：不能解冻；
- 不能隔离 active runtime：不能解冻；
- 只能下载、不能解释：重跑演练；
- 涉敏范围 TTL 太长：人工复核；
- 使用前必须有销毁计划。

测试要覆盖最危险的情况：bundle 能恢复，但 sandbox 没有挡住 active runtime。

```python
def test_denies_when_rehydration_can_touch_active_runtime():
    closeout = RetentionCloseout(
        receipt_verified=True,
        cold_bundle_hash="sha256:abc",
        schema_version_pinned=True,
        redaction_profile_pinned=True,
    )
    run = RehydrationRun(
        bundle_fetched=True,
        hash_recomputed=True,
        sandbox_isolated=True,
        active_runtime_blocked=False,
        index_rebuilt=True,
        query_passed=True,
        explanation_passed=True,
        replay_passed=True,
        elapsed_seconds=120,
    )
    request = ThawRequest(
        purpose="audit",
        requested_decisions=20,
        ttl_minutes=30,
        requester_authorized=True,
        includes_sensitive_scope=False,
        destruction_plan_ready=True,
    )

    assert decide_rehydration_thaw(closeout, run, request) == "deny_thaw"
```

这个测试很实用：恢复演练最怕“看起来跑通了”，但其实把旧 runtime、旧 prompt pack、旧 policy index 暴露给生产路径。

---

## pi-mono：Cold Evidence Rehydration Worker

生产版建议拆成两个 worker：

1. `ColdEvidenceRehydrationWorker` 定期演练，把冷 bundle 解冻到 sandbox；
2. `ThawPermitWorker` 在真实审计或事故请求到来时签发一次性 permit。

```typescript
type RehydrationDecision =
  | "issue_thaw_permit"
  | "rerun_rehydration_drill"
  | "rebuild_cold_index"
  | "deny_thaw"
  | "manual_review";

interface ColdEvidenceThawJob {
  cutoverId: string;
  retentionCloseoutId: string;
  requesterId: string;
  purpose: "audit" | "incident_review" | "regression_replay" | "legal_hold";
  decisionIds: string[];
  ttlMinutes: number;
}

class ColdEvidenceRehydrationWorker {
  constructor(
    private readonly receipts: ReceiptStore,
    private readonly archive: ColdEvidenceArchive,
    private readonly sandbox: EphemeralWorkspaceFactory,
    private readonly permits: ThawPermitStore,
    private readonly audit: AuditLog,
  ) {}

  async run(job: ColdEvidenceThawJob) {
    const closeout = await this.receipts.getColdRetentionCloseout(
      job.retentionCloseoutId,
    );

    const workspace = await this.sandbox.create({
      cutoverId: job.cutoverId,
      networkPolicy: "deny_active_runtime",
      ttlMinutes: job.ttlMinutes,
    });

    try {
      const bundle = await this.archive.fetchBundle({
        cutoverId: job.cutoverId,
        expectedHash: closeout.coldBundleHash,
      });

      const run = await workspace.rehydrateAndProbe({
        bundle,
        decisionIds: job.decisionIds,
        requireExplanation: true,
        requireReplay: job.purpose === "regression_replay",
      });

      const decision = decideRehydrationThaw(closeout, run, {
        purpose: job.purpose,
        requestedDecisions: job.decisionIds.length,
        ttlMinutes: job.ttlMinutes,
        requesterAuthorized: await this.permits.isRequesterAuthorized(
          job.requesterId,
          job.purpose,
        ),
        includesSensitiveScope: run.includesSensitiveScope,
        destructionPlanReady: workspace.hasDestructionPlan(),
      });

      await this.audit.append({
        type: "cold_evidence_rehydration_decision",
        cutoverId: job.cutoverId,
        requesterId: job.requesterId,
        purpose: job.purpose,
        decision,
        workspaceId: workspace.id,
        runReceiptId: run.receiptId,
      });

      if (decision !== "issue_thaw_permit") {
        return { decision };
      }

      const permit = await this.permits.issueOnce({
        cutoverId: job.cutoverId,
        requesterId: job.requesterId,
        purpose: job.purpose,
        workspaceId: workspace.id,
        decisionIds: job.decisionIds,
        expiresAt: new Date(Date.now() + job.ttlMinutes * 60_000),
      });

      return { decision, permitId: permit.id };
    } finally {
      await workspace.scheduleDestructionReceipt();
    }
  }
}
```

注意这里有三个工程点：

- `networkPolicy: "deny_active_runtime"` 是硬约束，不是注释；
- permit 是一次性的，不能拿审计 permit 顺手跑回归；
- `finally` 里安排销毁收据，失败也要留下“是否清理干净”的证据。

---

## OpenClaw 实战：课程 Cron 也需要这个思路

拿这个课程 cron 类比：

- lesson 文件、README、TOOLS、Telegram 消息、git commit 都是一次交付的证据；
- 现在我们主要依赖 git history 和本地文件作为热证据；
- 如果未来把旧课程归档到冷存储，不能只保证 markdown 文件还在；
- 真要复盘某一讲，系统要能按 topic / commit / messageId 把当时的 lesson、README 目录项、TOOLS 已讲内容和推送结果重新拼出来。

一个最小的 OpenClaw 冷证据再水化演练可以这样做：

1. 从冷归档拉取某次课程 bundle；
2. 在临时目录重建 `lessons/XXX-topic.md`、README 目录片段和 TOOLS 摘要；
3. 校验 bundle hash 与 commit hash；
4. 禁止任何 Telegram / git push / file write 副作用；
5. 输出一份 rehydration receipt；
6. 删除临时目录，只保留销毁收据。

这就是把“我应该还能查到”升级成“我定期证明能查到、能解释、能安全销毁”。

---

## 设计要点

第一，再水化必须默认隔离。

冷证据来自旧系统状态，里面可能包含旧 prompt、旧 policy、旧 tool schema、旧权限快照。恢复它是为了解释历史，不是为了让它重新影响现在。

第二，thaw permit 要绑定 purpose。

审计只需要查询和解释；事故复盘可能需要 timeline；回归重放需要 runner；legal hold 可能需要更长保留。不同用途的权限、TTL、脱敏策略不能混在一起。

第三，销毁也是证据。

临时热工作区如果用完不销毁，冷保留反而变成新的热风险面。closeout receipt 里至少要记录 workspaceId、销毁时间、残留扫描和 hash。

---

## 小结

Cold Evidence Retention 解决的是“历史证据不要丢”；Cold Evidence Rehydration 解决的是“需要时能安全拿出来用”。

成熟 Agent 的证据系统不是一个归档目录，而是一条闭环：

```
保留 → 再水化 → 解冻许可 → 隔离使用 → 销毁收据 → 下一次抽样
```

归档降低的是运行成本，不应该降低恢复、审计和解释能力。
