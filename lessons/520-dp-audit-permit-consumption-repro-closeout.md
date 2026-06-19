# 520 - Agent 差分隐私审计许可消耗与复现关闭闸门

> 上一讲我们把 DP seed 放进 escrow，并用 `OneTimeAuditPermit` 授权隔离 runner 复现 noisy result。但“发了许可”不等于“审计安全结束”。成熟 Agent 还要证明许可只被消费一次、复现过程没有导出 raw/seed、结果已经写入关闭收据，临时材料也被销毁。

上一讲讲的是 **Release Reproducibility & Seed Escrow Gate**。今天继续往下走一步：**DP Audit Permit Consumption & Reproducibility Closeout Gate**。

一句话：**一次性审计许可不是通行证，而是可原子消费、可追踪、可关闭、可销毁的短期能力。**

---

## 1. 为什么 permit 不能只靠 expiresAt？

很多系统会这样处理审计许可：

```json
{
  "permitId": "permit_123",
  "releaseId": "dp_rel_42",
  "caseId": "case_9",
  "expiresAt": "2026-06-19T06:00:00Z",
  "status": "active"
}
```

然后审计 runner 检查没过期就开始复现。这会留下几个洞：

- 两个 runner 并发启动，可能同时解封 seed；
- runner 失败后 status 没变，许可被重复使用；
- 复现日志如果误写 seed/noise/raw，就把隐私钥匙泄露到日志系统；
- 只记录 `matched=true`，却没有销毁 seed material 的证据；
- 复现不匹配时没有冻结发布记录，后续读者还在引用可疑结果。

所以 permit 消耗要变成一条证据链：

```text
OneTimeAuditPermit
        ↓
PermitConsumptionReceipt
        ↓
IsolatedReproductionRun
        ↓
ReproductionOutputReview
        ↓
SeedMaterialDestructionReceipt
        ↓
ReproducibilityCloseoutReceipt
```

关键不是“能复现”，而是“只能复现一次，并且复现结束后可证明没有残留”。

---

## 2. 闸门要检查什么？

DP Audit Permit Consumption Gate 要回答七个问题：

- permit 是否用 CAS/事务原子消费，防止并发双花；
- permit 的 releaseId/caseId/purpose 是否和 release receipt 一致；
- runner 是否在隔离环境执行，禁止网络导出和持久化 raw；
- 输出是否只包含 `matched/mismatch`、hash、receipt，不包含 seed/noise/raw；
- mismatch 是否触发 release freeze 或 manual review；
- seed material 是否有销毁收据和残留探针；
- closeout 是否绑定 permit consumption receipt，防止绕过许可直接关闭。

工程上可以把它记成三段：

```text
consume once -> reproduce isolated -> close with destruction proof
```

---

## 3. learn-claude-code：许可关闭闸门纯函数

教学版先写纯函数，只判断一次复现审计能不能关闭。

```python
from dataclasses import dataclass
from typing import Literal

Decision = Literal[
    "close_reproduction_matched",
    "freeze_release_mismatch",
    "reject_unconsumed_permit",
    "reject_scope_mismatch",
    "purge_leaked_sensitive_output",
    "rerun_destruction_probe",
    "manual_review_runner_violation",
]

@dataclass(frozen=True)
class OneTimeAuditPermit:
    permit_id: str
    release_id: str
    case_id: str
    purpose: str
    consumed: bool

@dataclass(frozen=True)
class PermitConsumptionReceipt:
    permit_id: str
    release_id: str
    case_id: str
    consumed_once: bool
    fencing_token: int

@dataclass(frozen=True)
class ReproductionRun:
    release_id: str
    case_id: str
    isolated_runner: bool
    output_contains_sensitive_material: bool
    matched_published_value: bool

@dataclass(frozen=True)
class DestructionProbe:
    seed_material_destroyed: bool
    temp_workspace_clean: bool
    cache_clean: bool

@dataclass(frozen=True)
class CloseoutDecision:
    decision: Decision
    reason: str
    freeze_release: bool = False

def decide_repro_closeout(
    permit: OneTimeAuditPermit,
    consumption: PermitConsumptionReceipt,
    run: ReproductionRun,
    destruction: DestructionProbe,
) -> CloseoutDecision:
    if not permit.consumed or not consumption.consumed_once:
        return CloseoutDecision(
            "reject_unconsumed_permit",
            "closeout must be backed by one atomic permit consumption",
        )

    same_scope = (
        permit.permit_id == consumption.permit_id
        and permit.release_id == consumption.release_id == run.release_id
        and permit.case_id == consumption.case_id == run.case_id
    )
    if not same_scope:
        return CloseoutDecision(
            "reject_scope_mismatch",
            "permit, consumption receipt, and reproduction run scope differ",
        )

    if not run.isolated_runner:
        return CloseoutDecision(
            "manual_review_runner_violation",
            "reproduction did not run inside the approved isolated runner",
        )

    if run.output_contains_sensitive_material:
        return CloseoutDecision(
            "purge_leaked_sensitive_output",
            "audit output contains seed, noise, raw value, or equivalent material",
            freeze_release=True,
        )

    if not (
        destruction.seed_material_destroyed
        and destruction.temp_workspace_clean
        and destruction.cache_clean
    ):
        return CloseoutDecision(
            "rerun_destruction_probe",
            "temporary seed material or workspace residue still needs proof of deletion",
        )

    if not run.matched_published_value:
        return CloseoutDecision(
            "freeze_release_mismatch",
            "reproduced noisy value does not match the published release receipt",
            freeze_release=True,
        )

    return CloseoutDecision(
        "close_reproduction_matched",
        "permit was consumed once, result matched, and temporary material was destroyed",
    )
```

最小测试：

```python
def test_mismatch_freezes_release():
    decision = decide_repro_closeout(
        OneTimeAuditPermit("p1", "rel1", "case1", "ops_report", True),
        PermitConsumptionReceipt("p1", "rel1", "case1", True, 7),
        ReproductionRun("rel1", "case1", True, False, False),
        DestructionProbe(True, True, True),
    )
    assert decision.decision == "freeze_release_mismatch"
    assert decision.freeze_release is True

def test_clean_match_closes():
    decision = decide_repro_closeout(
        OneTimeAuditPermit("p2", "rel2", "case2", "ops_report", True),
        PermitConsumptionReceipt("p2", "rel2", "case2", True, 8),
        ReproductionRun("rel2", "case2", True, False, True),
        DestructionProbe(True, True, True),
    )
    assert decision.decision == "close_reproduction_matched"
```

这个纯函数的价值是：不让 LLM 用一句“审计通过”跳过关键证据。permit、复现、销毁三张收据必须对齐。

---

## 4. pi-mono：AuditPermitConsumptionWorker

生产版要把 permit 消费做成原子操作，最好带 fencing token。后续 worker 只能拿最新 fencing token 写 closeout，防止旧 runner 复活乱写。

```ts
type AuditPermit = {
  permitId: string;
  releaseId: string;
  caseId: string;
  purpose: string;
  expiresAt: string;
  consumedAt?: string;
  fencingToken: number;
};

type ConsumptionReceipt = {
  permitId: string;
  releaseId: string;
  caseId: string;
  consumedAt: string;
  fencingToken: number;
};

type ReproCloseout = {
  permitId: string;
  releaseId: string;
  caseId: string;
  matched: boolean;
  seedDestroyed: boolean;
  outputHash: string;
  closedAt: string;
};

class AuditPermitConsumptionWorker {
  constructor(
    private readonly permits: AuditPermitStore,
    private readonly escrow: SeedEscrowStore,
    private readonly runner: IsolatedDpReproRunner,
    private readonly closeouts: ReproCloseoutStore,
    private readonly releases: DpReleaseStore,
    private readonly outbox: EventOutbox,
  ) {}

  async consumeAndClose(input: {
    permitId: string;
    expectedReleaseId: string;
  }): Promise<ReproCloseout> {
    const receipt = await this.permits.consumeOnce(input.permitId, {
      expectedReleaseId: input.expectedReleaseId,
    });

    const seedLease = await this.escrow.openOneTimeLease({
      releaseId: receipt.releaseId,
      caseId: receipt.caseId,
      fencingToken: receipt.fencingToken,
    });

    try {
      const release = await this.releases.getReceipt(receipt.releaseId);
      const result = await this.runner.reproduce({
        release,
        seedLease,
        denyNetwork: true,
        denyPersistentDisk: true,
      });

      if (result.containsSensitiveMaterial) {
        await this.releases.freeze(receipt.releaseId, "sensitive_audit_output");
        throw new Error("sensitive_audit_output");
      }

      const destruction = await seedLease.destroyAndProbe();
      if (!destruction.destroyed || !destruction.workspaceClean) {
        throw new Error("seed_material_destruction_unverified");
      }

      if (!result.matchedPublishedValue) {
        await this.releases.freeze(receipt.releaseId, "dp_reproduction_mismatch");
      }

      const closeout = await this.closeouts.insertOnce({
        permitId: receipt.permitId,
        releaseId: receipt.releaseId,
        caseId: receipt.caseId,
        matched: result.matchedPublishedValue,
        seedDestroyed: destruction.destroyed,
        outputHash: result.outputHash,
        closedAt: new Date().toISOString(),
      });

      await this.outbox.publish("dp.repro_closeout_written", closeout);
      return closeout;
    } finally {
      await seedLease.bestEffortDestroy();
    }
  }
}
```

注意两个细节：

- `consumeOnce` 必须在数据库里用唯一约束或 CAS 更新，不能先读后写；
- `finally` 里的 `bestEffortDestroy` 只是兜底，真正关闭仍要依赖销毁探针收据。

---

## 5. OpenClaw 实战类比

OpenClaw 的课程 cron 其实也有类似结构：

```text
cron trigger
  -> 写 lesson 文件
  -> 更新 README / TOOLS
  -> git commit / push
  -> Telegram 发送
  -> 最终汇报 commit + message delivery
```

如果只说“发过了”，不够。更好的关闭收据应该包含：

- lesson 文件路径和 hash；
- README 是否包含新目录项；
- TOOLS 是否追加已讲主题；
- git commit sha 和 remote head；
- Telegram message id 或发送结果；
- 失败时是否重试、撤回、补偿或标记 dead-letter。

DP 审计也是一样：不要靠“runner 跑完了”作为完成条件，要靠 permit consumption、reproduction output、destruction probe、closeout receipt 四件事闭环。

---

## 6. 设计 Checklist

落地这个闸门时，可以直接检查：

- permit 是否有 `permitId/releaseId/caseId/purpose/expiresAt/fencingToken`；
- permit 消费是否原子、幂等、不可重复；
- runner 是否禁止网络导出、持久化磁盘和明文日志；
- 审计输出是否通过敏感材料扫描；
- mismatch 是否冻结 release，而不是静默记录；
- seed material 是否有销毁探针；
- closeout 是否绑定 consumption receipt 和 output hash；
- 所有事件是否走 outbox，避免“审计完成但通知丢了”。

---

## 7. 今日要点

DP 审计许可的重点不是“给审计员一个短期 token”，而是把短期 token 变成一次性、可围栏、可关闭的能力链。

记住这个公式：

```text
DP Repro Audit = consumeOnce(permit) + isolatedReproduce(seed) + destroyAndProbe() + closeoutReceipt
```

成熟 Agent 不只会证明 DP 发布结果可复现，还会证明复现这件事本身没有变成新的隐私泄露源。
