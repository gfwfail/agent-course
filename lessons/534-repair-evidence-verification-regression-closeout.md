# 534. Agent 修复证据验证与回归关闭闸门

> RepairSlaCloseoutReceipt 只能证明 owner 已经接手并声称修复完成，不证明修复真的命中根因、回归种子通过、生产现实已经收敛。成熟 Agent 会把 RepairEvidencePacket、VerificationRun、RegressionReplay、RealityProbe 和 VerificationCloseoutReceipt 串起来，防止“已修复”变成又一张未经验证的票据。

上一讲讲了 **Owner Handoff Acknowledgement & Repair SLA Gate**：失败交给 owner 后，要确认接手、绑定修复 SLA、持续 heartbeat，逾期能升级。

今天继续往后走：**owner 说修好了以后，Agent 怎么证明可以关闭？**

这里最容易犯的错是：

- owner 贴了一个 commit 链接，但没有说明它修的是哪个 root cause；
- 单元测试过了，但原来的演练 seed 没重放；
- staging 通过了，但生产读取路径还在用旧缓存；
- 修复只覆盖 happy path，没有验证之前的 failure signature；
- close ticket 太早，后续重复失败又变成新事故。

所以要加一层 **Repair Evidence Verification & Regression Closeout Gate**：修复不是一句“fixed”，而是一组可重放、可探测、可关闭的证据。

---

## 1. 核心链路

```text
RepairSlaCloseoutReceipt
        ↓
RepairEvidencePacket
        ↓
VerificationRun
        ↓
RegressionReplay
        ↓
RealityProbe
        ↓
VerificationCloseoutReceipt
```

每一环回答一个问题：

- RepairSlaCloseoutReceipt：owner 是否已交付修复并请求验证；
- RepairEvidencePacket：修复证据是否绑定 root cause、commit、配置、迁移和验收条件；
- VerificationRun：最小验证是否跑过，失败是否可解释；
- RegressionReplay：原始失败 seed 是否重放通过；
- RealityProbe：真实读写路径是否已经加载修复后的 runtime；
- VerificationCloseoutReceipt：是否可以关闭、回滚、重新派单或进入观察窗口。

一句话：**owner 的修复证据只能打开验证门，不能直接关闭事故。**

---

## 2. 修复证据包应该长什么样？

一个可验证的 RepairEvidencePacket 至少要包含这些字段：

```text
repairEvidencePacket
  rootCausePacketId: rcp-532
  repairOwner: cold-path-oncall
  fixRefs:
    - commit: 9f2a1c7
    - migration: cold_index_backfill_20260627
    - config: retrieval.sla.indexVersion=v4
  changedRuntime:
    service: cold-evidence-retriever
    expectedGitSha: 9f2a1c7
    expectedConfigHash: cfg-44ab
  acceptanceCriteria:
    - seed-cold-17 passes
    - lookup archive:tenant-a:2026-06 under 1200ms
    - no hot evidence fallback observed
  ownerClaim: fixed_missing_lookup_key_index
```

注意这里不是“贴链接越多越好”，而是每个链接都能回答：**它修复了哪条 failure signature？它如何被验证？它是否已经进入真实 runtime？**

---

## 3. learn-claude-code：修复验证判定器

教学版先写纯函数。它不跑测试、不访问生产，只根据证据包、验证结果、回归重放和现实探针判断下一步。

```python
from dataclasses import dataclass
from typing import Literal

Decision = Literal[
    "request_repair_evidence",
    "run_verification",
    "rerun_regression_replay",
    "probe_reality",
    "return_to_owner",
    "rollback_repair",
    "close_verified",
    "enter_observation_window",
    "manual_review",
]

@dataclass
class RepairEvidencePacket:
    root_cause_packet_id: str
    fix_refs: list[str]
    expected_git_sha: str | None
    expected_config_hash: str | None
    acceptance_criteria: list[str]
    owner_claim: str | None

@dataclass
class VerificationRun:
    status: Literal["not_run", "passed", "failed"]
    failed_checks: list[str]
    evidence_refs: list[str]

@dataclass
class RegressionReplay:
    status: Literal["not_run", "passed", "failed", "flaky"]
    replayed_seed_ids: list[str]
    failed_seed_ids: list[str]

@dataclass
class RealityProbe:
    runtime_git_sha: str | None
    runtime_config_hash: str | None
    stale_cache_found: bool
    hot_fallback_observed: bool
    probe_errors: list[str]

def decide_repair_verification_gate(
    packet: RepairEvidencePacket,
    verification: VerificationRun,
    replay: RegressionReplay,
    reality: RealityProbe,
    required_seed_ids: list[str],
) -> Decision:
    if not packet.fix_refs or not packet.owner_claim:
        return "request_repair_evidence"

    if not packet.expected_git_sha or not packet.expected_config_hash:
        return "request_repair_evidence"

    if verification.status == "not_run":
        return "run_verification"

    if verification.status == "failed":
        return "return_to_owner"

    missing_seeds = set(required_seed_ids) - set(replay.replayed_seed_ids)
    if missing_seeds or replay.status == "not_run":
        return "rerun_regression_replay"

    if replay.status == "failed":
        return "return_to_owner"

    if replay.status == "flaky":
        return "manual_review"

    if reality.probe_errors:
        return "probe_reality"

    if reality.runtime_git_sha != packet.expected_git_sha:
        return "probe_reality"

    if reality.runtime_config_hash != packet.expected_config_hash:
        return "probe_reality"

    if reality.hot_fallback_observed:
        return "rollback_repair"

    if reality.stale_cache_found:
        return "enter_observation_window"

    return "close_verified"
```

对应测试：

```python
def test_does_not_close_when_runtime_has_not_loaded_fix():
    packet = RepairEvidencePacket(
        root_cause_packet_id="rcp-532",
        fix_refs=["commit:9f2a1c7"],
        expected_git_sha="9f2a1c7",
        expected_config_hash="cfg-44ab",
        acceptance_criteria=["seed-cold-17 passes"],
        owner_claim="fixed_missing_lookup_key_index",
    )
    verification = VerificationRun("passed", [], ["ci:991"])
    replay = RegressionReplay("passed", ["seed-cold-17"], [])
    reality = RealityProbe(
        runtime_git_sha="old-sha",
        runtime_config_hash="cfg-44ab",
        stale_cache_found=False,
        hot_fallback_observed=False,
        probe_errors=[],
    )

    assert decide_repair_verification_gate(
        packet,
        verification,
        replay,
        reality,
        required_seed_ids=["seed-cold-17"],
    ) == "probe_reality"
```

这个测试的重点是：**CI 过了不等于生产已经加载修复。** Agent 关闭事故前必须看真实 runtime，而不是只看代码仓库。

---

## 4. pi-mono：RepairVerificationWorker

生产版可以放在 RepairSlaCoordinator 之后。它消费 `RepairSlaCloseoutReceipt`，读取 owner 提交的 repair evidence，触发验证、重放 seed、探测 runtime，最后写 closeout。

```typescript
type VerificationDecision =
  | "request_repair_evidence"
  | "run_verification"
  | "rerun_regression_replay"
  | "probe_reality"
  | "return_to_owner"
  | "rollback_repair"
  | "close_verified"
  | "enter_observation_window"
  | "manual_review";

type RepairEvidencePacket = {
  rootCausePacketId: string;
  repairOwner: string;
  fixRefs: string[];
  expectedGitSha?: string;
  expectedConfigHash?: string;
  acceptanceCriteria: string[];
  ownerClaim?: string;
};

export class RepairVerificationWorker {
  constructor(
    private readonly evidenceStore: EvidenceStore,
    private readonly verifier: VerificationRunner,
    private readonly replay: RegressionReplayRunner,
    private readonly runtimeProbe: RuntimeProbe,
    private readonly outbox: Outbox,
  ) {}

  async handle(receipt: RepairSlaCloseoutReceipt) {
    const packet = await this.evidenceStore.getRepairEvidence(receipt.packetId);

    if (!packet?.fixRefs.length || !packet.ownerClaim) {
      await this.outbox.publish("repair.evidence.requested", {
        packetId: receipt.packetId,
        owner: receipt.owner,
        reason: "missing_fix_refs_or_owner_claim",
      });
      return;
    }

    const verification = await this.verifier.run({
      packetId: receipt.packetId,
      fixRefs: packet.fixRefs,
      criteria: packet.acceptanceCriteria,
    });

    if (verification.status !== "passed") {
      await this.outbox.publish("repair.returned_to_owner", {
        packetId: receipt.packetId,
        failedChecks: verification.failedChecks,
      });
      return;
    }

    const replay = await this.replay.runRequiredSeeds(receipt.requiredSeedIds);
    if (replay.status !== "passed") {
      await this.outbox.publish("repair.returned_to_owner", {
        packetId: receipt.packetId,
        failedSeeds: replay.failedSeedIds,
      });
      return;
    }

    const reality = await this.runtimeProbe.check({
      service: receipt.affectedService,
      expectedGitSha: packet.expectedGitSha,
      expectedConfigHash: packet.expectedConfigHash,
    });

    const decision = decideRepairVerificationGate(packet, verification, replay, reality);

    await this.outbox.publish("repair.verification.decided", {
      packetId: receipt.packetId,
      decision,
      runtimeGitSha: reality.runtimeGitSha,
      runtimeConfigHash: reality.runtimeConfigHash,
    });
  }
}
```

这里的设计原则：

- verifier 只看修复是否满足验收条件；
- replay 只看旧失败是否真的消失；
- runtimeProbe 只看真实服务是否加载新代码、新配置、无旧缓存；
- outbox 负责把 return、rollback、observation 或 closeout 发给后续 worker。

不要把这些步骤揉成一个“green check”。分开记录，后面才能解释为什么关闭、为什么退回、为什么回滚。

---

## 5. OpenClaw 课程 Cron 类比

这门课本身就是一个小型修复验证场景：

1. lesson 文件写完，只是修复证据；
2. README 目录更新，是索引验证；
3. TOOLS.md 已讲内容更新，是去重与长期状态验证；
4. git commit/push，是远端现实验证；
5. Telegram 发群，是消费者读路径验证。

如果只写了 lesson，但 README 没更新，下次同学找不到；
如果 README 更新了但 git push 失败，远端仓库没有现实效果；
如果 Telegram 发了但 TOOLS.md 没更新，下次 cron 可能重复选题。

所以这个 cron 的关闭条件不是“文件存在”，而是：

```text
lessonExists
READMEReferencesLesson
TOOLSContainsTopic
gitPushSucceeded
telegramMessageDelivered
```

这就是 Repair Verification 的日常版：每个副作用都有证据，每个证据都能被后续读取路径消费。

---

## 6. 实战建议

落地时可以从 4 个字段开始：

```text
repairEvidence
  rootCauseId
  fixRefs
  requiredRegressionSeeds
  expectedRuntimeFingerprint
```

然后给关闭加 4 个硬条件：

- 修复证据绑定 root cause；
- required seed 全部重放通过；
- runtime fingerprint 等于修复版本；
- 原失败路径的 reality probe 不再复现。

最重要的是：**close_verified 必须由验证系统写，不应该由 owner 手动点。**

owner 可以提交修复，Agent 负责验证现实。

---

## 7. 小结

Repair Evidence Verification & Regression Closeout Gate 解决的是一句话：**修复声明不是事实，验证收据才是事实。**

成熟 Agent 的关闭链路应该是：

```text
owner claims fixed
  → evidence packet
  → verification run
  → regression replay
  → reality probe
  → closeout receipt
```

这层闸门会让系统少一点“票关了但问题还在”，多一点“问题确实消失，并且消失这件事可以复查”。
