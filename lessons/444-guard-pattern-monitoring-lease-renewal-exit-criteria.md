# 444. Agent 护栏监控租约续期与退出条件（Guard Pattern Monitoring Lease Renewal & Exit Criteria）

> 关键词：monitoring lease、renewal gate、exit criteria、clean archive、breach budget、post-release lifecycle

上一课讲了 FamilyLockReleaseReceipt 之后不能忘记事故，要创建 PostReleaseMonitoringLease，继续观察 recurrence、scope drift、dependency fingerprint 和 residual constraints。

今天往后补一块非常容易漏的生产细节：**监控租约什么时候续期，什么时候退出？**

很多系统会犯两个相反错误：

1. 监控窗口到期就自动 archive，哪怕样本不够；
2. 监控任务永远挂着，最后变成没人看的噪音。

成熟 Agent 不能靠 cron 里一句 `if expiresAt < now then close`。PostReleaseMonitoringLease 需要一个明确的 Renewal Gate 和 Exit Criteria：

> 到期不是自动关闭，而是做一次有证据的续期 / 退出决策。

---

## 1. 为什么 lease 到期不能直接关闭

PostReleaseMonitoringLease 的目的不是“看一段时间”，而是证明解除 family lock 后系统仍然稳定。

所以 clean archive 至少要满足三类证据：

- **时间证据**：观察窗口已经覆盖最低时长；
- **样本证据**：关键 risk surface 有足够真实样本或明确豁免；
- **约束证据**：residual constraints 已关闭、降级或转成新的长期 guard；
- **依赖证据**：dependency fingerprint 没漂移，或漂移已经重新验证；
- **复发证据**：recurrence sentinel 在窗口内没有命中高风险 repeat/regression。

如果只看时间，会漏掉低流量场景。

例子：

- 监控 7 天；
- 但这 7 天高风险工具调用只有 2 次；
- 期间依赖模型还升级过；
- cron 到期直接 archive。

这不是稳定，只是没有观测到足够现实。

---

## 2. 状态机：lease 不只有 observing / closed

建议把 PostReleaseMonitoringLease 拆成这些状态：

~~~text
observing
  -> renewal_due
  -> extended
  -> clean_archive_ready
  -> archived
  -> breached
  -> reopened
  -> superseded
~~~

关键点：

- `renewal_due`：到期只是进入复核，不是关闭；
- `extended`：样本不足、依赖漂移、残余约束未闭环时续期；
- `clean_archive_ready`：证据足够，但还没原子归档；
- `archived`：写 ArchiveReceipt 后才真正退出热路径；
- `breached`：观察期发现复发或残余约束突破；
- `reopened`：已经转成 family lock reopen / reseal case；
- `superseded`：被新的 incident、guard pack 或长期监控策略接管。

这能避免两个坏味道：

1. expired lease 还在被 worker 继续写；
2. 旧 lease 和新 incident 同时处理同一批信号。

---

## 3. learn-claude-code：纯函数 Renewal Gate

教学版先写一个纯函数，输入 lease + observation summary，输出决策。

~~~python
# learn_claude_code/guard_patterns/post_release_lease_renewal.py
from dataclasses import dataclass
from typing import Literal

Decision = Literal[
    "extend_monitoring",
    "clean_archive",
    "reopen_family_lock",
    "open_reseal_case",
    "supersede_by_long_term_guard",
    "manual_review",
]


@dataclass(frozen=True)
class MonitoringLease:
    lease_id: str
    family_id: str
    status: str
    expires_at_epoch: int
    min_clean_samples: int
    required_surfaces: tuple[str, ...]
    residual_constraint_ids: tuple[str, ...]
    dependency_fingerprint: str
    extension_count: int
    max_extensions: int


@dataclass(frozen=True)
class ObservationSummary:
    now_epoch: int
    clean_samples_by_surface: dict[str, int]
    recurrence_hits: int
    max_recurrence_severity: int
    residual_breaches: int
    open_residual_constraints: int
    current_dependency_fingerprint: str
    dependency_revalidated: bool
    unknown_signal_count: int


@dataclass(frozen=True)
class RenewalDecision:
    decision: Decision
    reason: str
    extend_seconds: int = 0
    required_followup: tuple[str, ...] = ()


def decide_monitoring_lease_renewal(
    lease: MonitoringLease,
    summary: ObservationSummary,
) -> RenewalDecision:
    if lease.status != "observing":
        return RenewalDecision("manual_review", "lease_not_observing")

    if summary.recurrence_hits > 0 and summary.max_recurrence_severity >= 8:
        return RenewalDecision("reopen_family_lock", "high_severity_recurrence")

    if summary.residual_breaches > 0:
        return RenewalDecision("open_reseal_case", "residual_constraint_breach")

    if summary.current_dependency_fingerprint != lease.dependency_fingerprint:
        if not summary.dependency_revalidated:
            return RenewalDecision(
                "extend_monitoring",
                "dependency_drift_needs_revalidation",
                extend_seconds=7 * 24 * 3600,
                required_followup=("dependency_revalidation_probe",),
            )

    missing_surfaces = [
        surface
        for surface in lease.required_surfaces
        if summary.clean_samples_by_surface.get(surface, 0) < lease.min_clean_samples
    ]
    if missing_surfaces:
        if lease.extension_count >= lease.max_extensions:
            return RenewalDecision(
                "manual_review",
                "sample_gap_after_max_extensions",
                required_followup=tuple(missing_surfaces),
            )
        return RenewalDecision(
            "extend_monitoring",
            "insufficient_clean_samples",
            extend_seconds=3 * 24 * 3600,
            required_followup=tuple(missing_surfaces),
        )

    if summary.open_residual_constraints > 0:
        return RenewalDecision(
            "supersede_by_long_term_guard",
            "residual_constraints_need_long_term_owner",
            required_followup=lease.residual_constraint_ids,
        )

    if summary.unknown_signal_count > 0:
        return RenewalDecision(
            "extend_monitoring",
            "unknown_signals_need_classification",
            extend_seconds=24 * 3600,
        )

    if summary.now_epoch < lease.expires_at_epoch:
        return RenewalDecision("extend_monitoring", "not_due_yet", extend_seconds=lease.expires_at_epoch - summary.now_epoch)

    return RenewalDecision("clean_archive", "all_exit_criteria_met")
~~~

这里故意把 `now < expiresAt` 放到最后。因为严重复发、残余约束突破、依赖漂移这些信号即使没到期，也应该立即处理。

---

## 4. Exit Criteria 要写成结构化清单

不要把退出条件写成一句“观察 7 天无异常”。建议每个 lease 带一个 ExitCriteriaManifest：

~~~json
{
  "leaseId": "postrel_443",
  "familyId": "guard.family.restore.release",
  "minimumWindowHours": 168,
  "requiredCleanSamples": {
    "telegram_release": 3,
    "git_push": 3,
    "readme_index": 3,
    "tools_memory": 3
  },
  "residualConstraints": [
    {
      "id": "residual.topic_dedupe",
      "exitMode": "closed_or_promoted_to_guard",
      "owner": "course_cron"
    }
  ],
  "dependencyFingerprintPolicy": {
    "onDrift": "extend_and_revalidate",
    "probe": "repo_message_memory_consistency"
  },
  "recurrencePolicy": {
    "severityGte": 8,
    "action": "reopen_family_lock"
  },
  "maxExtensions": 3,
  "onMaxExtensionsExceeded": "manual_review"
}
~~~

这个 manifest 的价值是让 archive receipt 可复算：

- 为什么这次能退出；
- 哪些 surface 有多少 clean samples；
- 哪些 residual constraint 已关闭；
- 依赖漂移有没有重新验证；
- 超过多少次续期必须人工判断。

---

## 5. pi-mono：Renewal Worker + Archive Receipt

生产版通常是一个后台 worker 扫描 `renewal_due` 或 `expiresAt <= now` 的 lease。

TypeScript 里我会拆成三个边界：

1. projector 聚合 observation summary；
2. gate 计算 renewal decision；
3. worker 在事务里写 receipt / outbox。

~~~ts
// packages/runtime/src/guard/PostReleaseMonitoringLeaseRenewal.ts
type LeaseStatus =
  | "observing"
  | "renewal_due"
  | "extended"
  | "clean_archive_ready"
  | "archived"
  | "breached"
  | "reopened"
  | "superseded";

type RenewalAction =
  | "extend_monitoring"
  | "clean_archive"
  | "reopen_family_lock"
  | "open_reseal_case"
  | "supersede_by_long_term_guard"
  | "manual_review";

interface PostReleaseMonitoringLease {
  leaseId: string;
  familyId: string;
  status: LeaseStatus;
  fenceToken: string;
  expiresAt: string;
  extensionCount: number;
  maxExtensions: number;
  releaseReceiptRef: string;
  dependencyFingerprint: string;
}

interface ObservationSummary {
  cleanSamplesBySurface: Record<string, number>;
  recurrenceHits: number;
  maxRecurrenceSeverity: number;
  residualBreaches: number;
  openResidualConstraints: string[];
  dependencyFingerprint: string;
  dependencyRevalidated: boolean;
  unknownSignalCount: number;
}

interface RenewalReceipt {
  receiptId: string;
  leaseId: string;
  familyId: string;
  action: RenewalAction;
  reason: string;
  previousExpiresAt: string;
  nextExpiresAt?: string;
  observationHash: string;
  createdAt: string;
}

export class PostReleaseMonitoringLeaseRenewalWorker {
  constructor(
    private readonly store: PostReleaseLeaseStore,
    private readonly projector: ObservationProjector,
    private readonly gate: RenewalGate,
    private readonly outbox: Outbox,
  ) {}

  async runOnce(now: Date) {
    const leases = await this.store.listRenewalDue(now);

    for (const lease of leases) {
      const summary = await this.projector.summarize(lease.leaseId);
      const decision = this.gate.decide({ lease, summary, now });

      await this.store.transaction(async (tx) => {
        const current = await tx.getLeaseForUpdate(lease.leaseId);

        if (!current || current.fenceToken !== lease.fenceToken) {
          return;
        }

        const receipt = await tx.appendRenewalReceipt({
          leaseId: current.leaseId,
          familyId: current.familyId,
          action: decision.action,
          reason: decision.reason,
          previousExpiresAt: current.expiresAt,
          nextExpiresAt: decision.nextExpiresAt,
          observationHash: summary.hash,
        });

        if (decision.action === "extend_monitoring") {
          await tx.extendLease(current.leaseId, {
            expectedFenceToken: current.fenceToken,
            nextExpiresAt: decision.nextExpiresAt,
            extensionCount: current.extensionCount + 1,
            receiptId: receipt.receiptId,
          });
          return;
        }

        if (decision.action === "clean_archive") {
          await tx.markCleanArchiveReady(current.leaseId, receipt.receiptId);
          await this.outbox.enqueue(tx, {
            kind: "archive_post_release_monitoring_lease",
            leaseId: current.leaseId,
            receiptId: receipt.receiptId,
          });
          return;
        }

        await tx.markBreachedOrSuperseded(current.leaseId, {
          action: decision.action,
          receiptId: receipt.receiptId,
        });

        await this.outbox.enqueue(tx, {
          kind: decision.action,
          leaseId: current.leaseId,
          receiptId: receipt.receiptId,
          familyId: current.familyId,
        });
      });
    }
  }
}
~~~

这里有两个细节很重要：

- `fenceToken` 防止旧 worker 在 lease 已被续期、归档或 breach 后继续写；
- `clean_archive` 不是直接删 lease，而是先进入 `clean_archive_ready`，由 archive worker 写最终 ArchiveReceipt。

---

## 6. ArchiveReceipt 不是删除记录

监控退出时，不要把 lease 从数据库里删掉。正确做法是写 ArchiveReceipt：

~~~ts
interface ArchiveReceipt {
  archiveReceiptId: string;
  leaseId: string;
  familyId: string;
  releaseReceiptRef: string;
  renewalReceiptId: string;
  exitCriteriaManifestHash: string;
  observationSummaryHash: string;
  finalDependencyFingerprint: string;
  cleanSamplesBySurface: Record<string, number>;
  archivedAt: string;
  tombstoneRefs: string[];
  retainedRegressionRefs: string[];
}
~~~

ArchiveReceipt 的意思是：

> 这个 lease 已经退出热监控，但它留下的 tombstone、regression seed、pattern knowledge 仍然可以被未来 release gate 查询。

这能避免事故知识从系统里消失。

---

## 7. OpenClaw 课程 Cron 实战

这门课的 cron 本身就有一条 post-release monitoring lease：

- Telegram 消息是否发出；
- lesson 文件是否存在；
- README 目录是否包含新课；
- TOOLS 已讲内容是否更新；
- git commit 是否推到远端；
- memory 是否记录 messageId / commit / 主题。

第 443 课发布后，下一轮第 444 课启动时就做了隐式检查：

1. repo 是干净的；
2. README 最新目录项是 443；
3. TOOLS 已讲内容包含 443 主题；
4. 本轮题目没有重复；
5. 本轮完成后再写 444 的 release evidence。

如果发现“Telegram 发了但 git 没推”，这不是小瑕疵，而是 release reality 和 repo state 分裂。正确动作不是继续讲新课，而是先补齐旧 release 的对账证据，必要时 reopen monitoring lease。

---

## 8. 工程落地要点

- lease 到期只代表需要决策，不代表可以关闭；
- Exit Criteria 要结构化，能被代码复算；
- clean sample 要按 risk surface 分层统计，不能只看总数；
- dependency fingerprint 漂移后必须 extend + revalidate；
- maxExtensions 是防噪音阀门，超过后进入 manual_review 或长期 guard；
- ArchiveReceipt 要保留 tombstone / regression refs，不要删除事故记忆；
- renewal worker 必须用 fenceToken 和事务，避免并发续期、归档、重开互相覆盖；
- superseded 要显式记录接管者，避免旧 lease 和新 guard 同时处置信号。

---

## 9. 一句话总结

> 监控租约到期不是自动下班，而是一次证据复核；成熟 Agent 会用 Renewal Gate 和 Exit Criteria 判断是续期、归档、重开，还是把残余约束升级成长期护栏。
