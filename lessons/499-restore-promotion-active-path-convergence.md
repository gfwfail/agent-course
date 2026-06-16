# 499. Agent 修复归档晋级后的活跃恢复路径收敛闸门（Restore Promotion Active Path Convergence Gate）

上一课讲的是：restore drill 修复完成后，不能直接刷新归档；要先生成 `RestoreRepairCandidate`，经过候选验证、隔离重演练和 CAS 晋级，最后写下 `ReDrillPromotionReceipt`。

这一课继续往后走：**归档指针晋级成功，不等于生产恢复路径已经真的用上它了**。成熟 Agent 还要证明 active restore path、runner、缓存、索引和告警都已经收敛到修复后的归档，否则下一次事故可能仍然从旧路径恢复。

## 1. 为什么还需要 Active Path 收敛

`promote_repaired_archive` 只是把 repaired archive 写进权威归档指针。真实系统里还有很多派生路径：

- restore runner 可能缓存了旧 archive hash
- recovery index 可能还指向旧 snapshot
- warm standby 环境可能没有拉取新归档
- drill scheduler 可能还在跑旧 restore path version
- alert/routing 规则可能不知道这次恢复系统已经刷新
- OpenClaw cron 可能 lesson 文件、README、TOOLS 已提交，但 Telegram 消息和远端 commit 没有对账

所以第 499 课是 **Restore Promotion Active Path Convergence Gate**：把 `ReDrillPromotionReceipt` 后面的生产收敛显式建模，直到所有恢复入口都证明自己加载了新 archive hash，再允许关闭修复链路。

```text
ReDrillPromotionReceipt
  -> ActiveRestorePathInventory
  -> RuntimeConvergenceProbe
  -> RecoveryCacheInvalidationReceipt
  -> ActiveRestoreConvergenceReceipt
```

关键思想：**晋级是控制面动作，收敛是运行面事实**。两者必须分开证明。

## 2. 最小数据模型

```python
from dataclasses import dataclass
from typing import Literal

Decision = Literal[
    "close_converged",
    "wait_convergence",
    "invalidate_cache",
    "rollback_archive_pointer",
    "quarantine_restore_path",
    "manual_review",
]


@dataclass(frozen=True)
class ReDrillPromotionReceipt:
    receipt_id: str
    candidate_id: str
    decision: Literal["promote_repaired_archive"]
    archive_hash_before: str
    archive_hash_after: str
    restore_path_version: str


@dataclass(frozen=True)
class ActiveRestorePathInventory:
    receipt_id: str
    restore_runners: list[str]
    recovery_indexes: list[str]
    warm_standby_nodes: list[str]
    scheduler_versions: list[str]
    expected_archive_hash: str
    expected_restore_path_version: str


@dataclass(frozen=True)
class RuntimeConvergenceProbe:
    receipt_id: str
    runners_seen: int
    runners_total: int
    indexes_seen: int
    indexes_total: int
    standby_seen: int
    standby_total: int
    stale_archive_hits: int
    stale_restore_path_hits: int
    last_probe_age_seconds: int
    smoke_restore_passed: bool
    alert_route_updated: bool


@dataclass(frozen=True)
class RecoveryCacheInvalidationReceipt:
    receipt_id: str
    invalidated_keys: list[str]
    failed_keys: list[str]
    cache_epoch: int
    post_invalidation_stale_hits: int
```

`Inventory` 是“应该有哪些入口”，`Probe` 是“实际看到了哪些入口”，`CacheInvalidationReceipt` 是“旧路径有没有被清理”。不要只看 active pointer。

## 3. learn-claude-code：纯函数闸门

教学版先写一个纯函数，把判断顺序固定下来：

```python
def decide_active_restore_convergence(
    promotion: ReDrillPromotionReceipt,
    inventory: ActiveRestorePathInventory,
    probe: RuntimeConvergenceProbe,
    cache: RecoveryCacheInvalidationReceipt,
) -> Decision:
    if promotion.receipt_id != inventory.receipt_id:
        return "manual_review"

    if promotion.receipt_id != probe.receipt_id:
        return "manual_review"

    if promotion.receipt_id != cache.receipt_id:
        return "manual_review"

    if inventory.expected_archive_hash != promotion.archive_hash_after:
        return "manual_review"

    if inventory.expected_restore_path_version != promotion.restore_path_version:
        return "manual_review"

    if cache.failed_keys:
        return "invalidate_cache"

    if cache.post_invalidation_stale_hits > 0:
        return "invalidate_cache"

    if probe.stale_archive_hits > 0:
        return "rollback_archive_pointer"

    if probe.stale_restore_path_hits > 0:
        return "quarantine_restore_path"

    if probe.last_probe_age_seconds > 900:
        return "wait_convergence"

    if probe.runners_seen < probe.runners_total:
        return "wait_convergence"

    if probe.indexes_seen < probe.indexes_total:
        return "wait_convergence"

    if probe.standby_seen < probe.standby_total:
        return "wait_convergence"

    if not probe.smoke_restore_passed:
        return "quarantine_restore_path"

    if not probe.alert_route_updated:
        return "wait_convergence"

    return "close_converged"
```

这个顺序很重要：

1. 先确认所有输入绑定同一个 promotion receipt
2. 再确认 inventory 期望值就是晋级后的 hash/version
3. cache 清理失败先重试清理
4. 运行面看到旧 archive，说明恢复入口还可能用旧证据，优先回滚或阻断
5. restore path 版本旧，隔离路径而不是关闭案件
6. 所有入口、索引、standby 和 alert 都收敛后才 close

## 4. pi-mono：ActiveRestoreConvergenceWorker

生产版可以让 Worker 消费 `ReDrillPromotionReceipt`，先列出所有恢复入口，再探测运行面，最后写 `ActiveRestoreConvergenceReceipt`。

```ts
type ConvergenceDecision =
  | "close_converged"
  | "wait_convergence"
  | "invalidate_cache"
  | "rollback_archive_pointer"
  | "quarantine_restore_path"
  | "manual_review";

interface ActiveRestoreConvergenceReceipt {
  receiptId: string;
  promotionReceiptId: string;
  archiveHashAfter: string;
  decision: ConvergenceDecision;
  inventoryId: string;
  probeId: string;
  cacheReceiptId: string;
  createdAt: string;
  reasons: string[];
}

class ActiveRestoreConvergenceWorker {
  constructor(
    private readonly store: RestoreOpsStore,
    private readonly inventory: RestorePathInventoryService,
    private readonly probes: RuntimeRestoreProbeService,
    private readonly cache: RecoveryCacheService,
    private readonly archivePointer: ArchivePointerService,
    private readonly quarantine: RestorePathQuarantineService,
  ) {}

  async handle(promotionReceiptId: string): Promise<ActiveRestoreConvergenceReceipt> {
    const promotion = await this.store.getReDrillPromotionReceipt(promotionReceiptId);
    const inventory = await this.inventory.collect({
      receiptId: promotion.receiptId,
      expectedArchiveHash: promotion.archiveHashAfter,
      expectedRestorePathVersion: promotion.restorePathVersion,
    });

    const cacheReceipt = await this.cache.invalidateByArchiveTransition({
      fromHash: promotion.archiveHashBefore,
      toHash: promotion.archiveHashAfter,
      reasonReceiptId: promotion.receiptId,
    });

    const probe = await this.probes.run({
      receiptId: promotion.receiptId,
      expectedArchiveHash: promotion.archiveHashAfter,
      expectedRestorePathVersion: promotion.restorePathVersion,
      requireSmokeRestore: true,
    });

    const { decision, reasons } = decideActiveRestoreConvergence(
      promotion,
      inventory,
      probe,
      cacheReceipt,
    );

    return this.store.transaction(async (tx) => {
      const receipt = await tx.writeActiveRestoreConvergenceReceipt({
        promotionReceiptId,
        archiveHashAfter: promotion.archiveHashAfter,
        decision,
        inventoryId: inventory.inventoryId,
        probeId: probe.probeId,
        cacheReceiptId: cacheReceipt.receiptId,
        reasons,
      });

      if (decision === "rollback_archive_pointer") {
        await this.archivePointer.compareAndSwap({
          tx,
          expectedHash: promotion.archiveHashAfter,
          nextHash: promotion.archiveHashBefore,
          reasonReceiptId: receipt.receiptId,
        });
      }

      if (decision === "quarantine_restore_path") {
        await this.quarantine.block({
          tx,
          restorePathVersion: promotion.restorePathVersion,
          reasonReceiptId: receipt.receiptId,
          staleHits: probe.staleRestorePathHits,
        });
      }

      if (decision === "wait_convergence") {
        await tx.scheduleConvergenceProbe({
          promotionReceiptId,
          runAfterSeconds: 300,
          reason: "runtime paths not converged",
        });
      }

      return receipt;
    });
  }
}
```

生产实现里有三个细节：

- `invalidateByArchiveTransition` 要按 `fromHash -> toHash` 清理，不要全量清缓存
- `compareAndSwap` 回滚时必须确认 active 仍然是 `archiveHashAfter`
- `wait_convergence` 要写下一次 probe，而不是靠人记得再跑

## 5. OpenClaw 课程 Cron 的实战映射

这门课的自动发布也有 active path 收敛问题：

```text
lesson file
README index
TOOLS 已讲内容
Telegram group message
git commit
remote branch
```

如果 lesson 文件写好了、commit 也有了，但 Telegram 没发出去，群里的运行面没有收敛；如果 Telegram 发了但 push 失败，远端归档没有收敛；如果 README 没更新，检索入口没有收敛。

可以把本次课程发布也看成一次收敛检查：

```yaml
active_restore_convergence:
  promotion: lesson-499-created
  expected_paths:
    - lessons/499-restore-promotion-active-path-convergence.md
    - README.md
    - ../TOOLS.md
    - telegram:-5115329245
    - origin/main
  probes:
    readme_contains_lesson: true
    tools_contains_topic: true
    telegram_sent: true
    git_status_clean: true
    remote_contains_commit: true
```

Agent 自动化要从“我执行过”升级成“所有入口都能看到同一个结果”。

## 6. 常见坑

### 坑 1：只看数据库里的 active pointer

指针是控制面事实，runner 是否真正加载是运行面事实。两个都要看。

### 坑 2：忘记 warm standby

很多恢复链路平时不处理流量，只有事故时才启用。它们最容易保留旧 archive hash。

### 坑 3：缓存清理没有收据

缓存失效要记录 invalidated keys、failed keys、cache epoch 和 stale hit。否则出事后不知道旧路径从哪里来的。

### 坑 4：alert route 没更新

恢复系统修好了，但告警还指向旧 owner 或旧 runbook，下一次事故仍然没人按新路径处理。

### 坑 5：等待收敛没有上限

`wait_convergence` 需要 lease、deadline 和 max attempts。超过预算要 manual review 或 quarantine。

## 7. 最小落地清单

- [ ] `promote_repaired_archive` 后创建 active path inventory
- [ ] 列出 restore runner、recovery index、warm standby 和 scheduler
- [ ] 按 archive hash transition 精准清缓存
- [ ] runtime probe 检查每个入口加载的新 hash/version
- [ ] 做一次真实 smoke restore
- [ ] 检查 alert route/runbook 是否更新
- [ ] stale archive hit 触发 rollback 或阻断
- [ ] stale restore path hit 触发 quarantine
- [ ] `wait_convergence` 写下次 probe lease
- [ ] close 前写 `ActiveRestoreConvergenceReceipt`

## 8. 总结

恢复归档晋级不是终点。真正的闭环是：

```text
promote repaired archive
  -> inventory active paths
  -> invalidate stale caches
  -> probe runtime convergence
  -> smoke restore
  -> close with receipt
```

一句话：**控制面说“新归档已晋级”不够，运行面也要证明所有恢复入口都已经看见、加载并成功使用它。**
