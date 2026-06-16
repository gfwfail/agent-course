# 500. Agent 活跃恢复路径漂移复核与定期再验证闸门（Active Restore Path Drift Revalidation Gate）

上一课讲的是：修复归档晋级后，要证明 restore runner、recovery index、warm standby、scheduler、cache 和 alert route 都已经加载新的 archive hash/version，最后写下 `ActiveRestoreConvergenceReceipt`。

这一课继续往后走：**收敛成功不是永久事实**。生产系统会继续部署、扩容、清缓存、切 region、升级 runner、迁移索引；任何一个动作都可能让恢复入口重新漂移回旧版本，或者让“看起来可恢复”的路径在真正事故时不可用。

所以第 500 课是 **Active Restore Path Drift Revalidation Gate**：把已经 close 的恢复路径变成可周期复核、可自动纠偏、可重新打开案件的运行资产。

## 1. 为什么收敛后还要复核

`ActiveRestoreConvergenceReceipt` 只能证明某个时间点已经收敛。它不能保证：

- 新扩容的 runner 也加载了 repaired archive
- standby 节点重启后没有拿到旧 snapshot
- scheduler 升级后 drill job 没有停跑
- recovery index compaction 没有丢掉新 hash
- cache TTL 到期后没有回源旧路径
- alert route 变更后 restore failure 还能通知到正确 owner
- OpenClaw 课程仓库 push 后，README/TOOLS/Telegram 对账不会在下一轮 cron 漂移

成熟 Agent 要把“恢复路径可用”当成长期不变量，而不是事故闭环时的一次性检查。

```text
ActiveRestoreConvergenceReceipt
  -> RestorePathDriftAuditLease
  -> PeriodicRestoreProbeRun
  -> DriftRevalidationReview
  -> RestorePathDriftCloseoutReceipt
```

关键思想：**收敛证明的是过去，复核保护的是未来**。

## 2. 最小数据模型

```python
from dataclasses import dataclass
from typing import Literal

Decision = Literal[
    "renew_audit_lease",
    "refresh_probe_only",
    "reopen_restore_case",
    "quarantine_drifted_path",
    "rollback_to_lkg_archive",
    "manual_review",
]


@dataclass(frozen=True)
class ActiveRestoreConvergenceReceipt:
    receipt_id: str
    archive_hash: str
    restore_path_version: str
    decision: Literal["close_converged"]
    closed_at_epoch: int


@dataclass(frozen=True)
class RestorePathDriftAuditLease:
    lease_id: str
    convergence_receipt_id: str
    expected_archive_hash: str
    expected_restore_path_version: str
    expires_at_epoch: int
    max_probe_age_seconds: int
    min_successful_probe_count: int


@dataclass(frozen=True)
class PeriodicRestoreProbeRun:
    lease_id: str
    archive_hash_seen: str
    restore_path_version_seen: str
    successful_probe_count: int
    failed_probe_count: int
    stale_runner_count: int
    stale_index_count: int
    stale_cache_hits: int
    standby_restore_passed: bool
    alert_route_verified: bool
    last_probe_age_seconds: int


@dataclass(frozen=True)
class DriftRevalidationReview:
    lease_id: str
    deployment_epoch_changed: bool
    dependency_fingerprint_changed: bool
    region_topology_changed: bool
    owner_ack_missing: bool
    lkg_archive_available: bool
```

这里的 `Lease` 是“我打算多久复核一次、复核什么版本”；`ProbeRun` 是“真实运行面看到了什么”；`Review` 是“这次漂移是不是由环境变更引起，以及能不能安全纠偏”。

## 3. learn-claude-code：纯函数闸门

教学版仍然先写纯函数。它不做副作用，只把输入证据压成一个明确动作。

```python
def decide_restore_path_drift_revalidation(
    convergence: ActiveRestoreConvergenceReceipt,
    lease: RestorePathDriftAuditLease,
    probe: PeriodicRestoreProbeRun,
    review: DriftRevalidationReview,
    now_epoch: int,
) -> Decision:
    if convergence.decision != "close_converged":
        return "manual_review"

    if lease.convergence_receipt_id != convergence.receipt_id:
        return "manual_review"

    if probe.lease_id != lease.lease_id:
        return "manual_review"

    if review.lease_id != lease.lease_id:
        return "manual_review"

    if lease.expected_archive_hash != convergence.archive_hash:
        return "manual_review"

    if lease.expected_restore_path_version != convergence.restore_path_version:
        return "manual_review"

    archive_drifted = probe.archive_hash_seen != lease.expected_archive_hash
    path_drifted = probe.restore_path_version_seen != lease.expected_restore_path_version

    if archive_drifted and review.lkg_archive_available:
        return "rollback_to_lkg_archive"

    if archive_drifted:
        return "reopen_restore_case"

    if path_drifted:
        return "quarantine_drifted_path"

    if probe.stale_runner_count > 0 or probe.stale_index_count > 0:
        return "quarantine_drifted_path"

    if probe.stale_cache_hits > 0:
        return "refresh_probe_only"

    if not probe.standby_restore_passed:
        return "reopen_restore_case"

    if not probe.alert_route_verified:
        return "refresh_probe_only"

    if probe.failed_probe_count > 0:
        return "refresh_probe_only"

    if probe.successful_probe_count < lease.min_successful_probe_count:
        return "refresh_probe_only"

    if probe.last_probe_age_seconds > lease.max_probe_age_seconds:
        return "refresh_probe_only"

    if review.deployment_epoch_changed or review.dependency_fingerprint_changed:
        return "refresh_probe_only"

    if review.region_topology_changed and review.owner_ack_missing:
        return "manual_review"

    if now_epoch >= lease.expires_at_epoch:
        return "renew_audit_lease"

    return "renew_audit_lease"
```

判断顺序有几个细节：

1. 先保证所有证据绑定同一个 convergence receipt 和 audit lease
2. archive hash 漂移比 restore path version 漂移更严重，因为它影响恢复内容本身
3. cache 旧命中可以先刷新和重探，不必立刻重开事故
4. standby restore 失败必须重开案件，因为真正灾难恢复依赖它
5. topology 变化但 owner 没确认，不要自动续租
6. lease 到期不是关闭，而是续约下一轮复核

## 4. pi-mono：Drift Revalidation Worker

生产版可以把这个闸门做成一个周期 worker。它消费上一课的 `ActiveRestoreConvergenceReceipt`，定期生成 probe run 和 closeout receipt。

```ts
type DriftDecision =
  | "renew_audit_lease"
  | "refresh_probe_only"
  | "reopen_restore_case"
  | "quarantine_drifted_path"
  | "rollback_to_lkg_archive"
  | "manual_review";

interface RestorePathDriftCloseoutReceipt {
  receiptId: string;
  convergenceReceiptId: string;
  leaseId: string;
  decision: DriftDecision;
  expectedArchiveHash: string;
  observedArchiveHash: string;
  expectedRestorePathVersion: string;
  observedRestorePathVersion: string;
  reasons: string[];
  nextProbeAfter?: string;
  createdAt: string;
}

class RestorePathDriftRevalidationWorker {
  constructor(
    private readonly store: RestoreOpsStore,
    private readonly probes: RestoreProbeService,
    private readonly topology: RuntimeTopologyService,
    private readonly quarantine: RestorePathQuarantineService,
    private readonly archivePointer: ArchivePointerService,
    private readonly outbox: Outbox,
  ) {}

  async handle(convergenceReceiptId: string): Promise<RestorePathDriftCloseoutReceipt> {
    const convergence = await this.store.getActiveRestoreConvergenceReceipt(
      convergenceReceiptId,
    );

    const lease = await this.store.getOrCreateRestorePathDriftAuditLease({
      convergenceReceiptId: convergence.receiptId,
      expectedArchiveHash: convergence.archiveHash,
      expectedRestorePathVersion: convergence.restorePathVersion,
      maxProbeAgeSeconds: 900,
      minSuccessfulProbeCount: 3,
    });

    const probe = await this.probes.runPeriodicProbe({
      leaseId: lease.leaseId,
      expectedArchiveHash: lease.expectedArchiveHash,
      expectedRestorePathVersion: lease.expectedRestorePathVersion,
      includeStandbyRestore: true,
      verifyAlertRoute: true,
    });

    const review = await this.topology.reviewDriftContext({
      leaseId: lease.leaseId,
      sinceReceiptId: convergence.receiptId,
    });

    const { decision, reasons } = decideRestorePathDriftRevalidation(
      convergence,
      lease,
      probe,
      review,
      Math.floor(Date.now() / 1000),
    );

    if (decision === "quarantine_drifted_path") {
      await this.quarantine.quarantine({
        leaseId: lease.leaseId,
        reason: "restore_path_drift",
        staleRunners: probe.staleRunnerIds,
      });
    }

    if (decision === "rollback_to_lkg_archive") {
      await this.archivePointer.rollbackToLkg({
        reasonReceiptId: convergence.receiptId,
        observedArchiveHash: probe.archiveHashSeen,
      });
    }

    const receipt = await this.store.insertRestorePathDriftCloseoutReceipt({
      convergenceReceiptId: convergence.receiptId,
      leaseId: lease.leaseId,
      decision,
      expectedArchiveHash: lease.expectedArchiveHash,
      observedArchiveHash: probe.archiveHashSeen,
      expectedRestorePathVersion: lease.expectedRestorePathVersion,
      observedRestorePathVersion: probe.restorePathVersionSeen,
      reasons,
      nextProbeAfter:
        decision === "renew_audit_lease" || decision === "refresh_probe_only"
          ? new Date(Date.now() + 15 * 60 * 1000).toISOString()
          : undefined,
    });

    await this.outbox.publish("restore_path_drift_revalidation.closed", {
      receiptId: receipt.receiptId,
      decision: receipt.decision,
    });

    return receipt;
  }
}
```

注意：`refresh_probe_only` 不是“忽略问题”，而是把低风险漂移转成短周期重探；`reopen_restore_case` 和 `rollback_to_lkg_archive` 才是影响恢复控制面的动作。

## 5. OpenClaw 课程 Cron 类比

这门课本身就是一个小型恢复路径：

1. 写 lesson 文件
2. 更新 README 目录
3. 更新 TOOLS 已讲列表
4. `git add . && git commit && git push`
5. 发 Telegram 群消息

上一课的 convergence 类似“这一轮都完成了”。但长期看还会漂移：

- README 可能漏了新课
- TOOLS 已讲列表可能没加
- commit push 成功但 Telegram 发送失败
- cron 下次又选了重复主题
- GitHub 账号没切到 `gfwfail`

所以可以给课程 cron 也加一个轻量 drift lease：

```python
def audit_course_delivery(
    lesson_file_exists: bool,
    readme_has_lesson: bool,
    tools_has_topic: bool,
    git_pushed: bool,
    telegram_sent: bool,
) -> str:
    if not lesson_file_exists:
        return "reopen_delivery_case"
    if not readme_has_lesson or not tools_has_topic:
        return "repair_index"
    if not git_pushed:
        return "retry_push"
    if not telegram_sent:
        return "retry_telegram"
    return "close_delivery_verified"
```

这就是工程化 Agent 的核心：不是“我刚才做了”，而是“我能反复证明它仍然成立”。

## 6. 实战落地建议

- 每个 close receipt 后面都应该问一句：这个结论会不会随时间漂移？
- 会漂移的结论，要配 `Lease`，不要只配 `status=closed`
- probe 要覆盖真实恢复入口，不要只查控制面数据库
- 低风险异常走 `refresh_probe_only`，高风险异常才重开案件或回滚
- drift closeout receipt 要写入审计链，方便后续事故复盘
- lease 续约要有退出条件，否则后台任务会变成永久噪音

## 7. 小结

第 500 课的重点：

- `ActiveRestoreConvergenceReceipt` 证明恢复路径曾经收敛
- `RestorePathDriftAuditLease` 让收敛结论进入周期复核
- `PeriodicRestoreProbeRun` 用真实 runner、index、cache、standby、alert 证明运行面没漂移
- `DriftRevalidationReview` 区分部署、依赖、拓扑变化带来的正常重探与危险漂移
- `RestorePathDriftCloseoutReceipt` 把续租、重探、隔离、重开、回滚都留下证据

成熟 Agent 的恢复系统不是事故时能跑一次，而是在事故之间也持续证明：下一次需要它时，它仍然能跑。
