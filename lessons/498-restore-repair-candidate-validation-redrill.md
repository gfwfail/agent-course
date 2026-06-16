# 498. Agent 恢复演练修复后的候选验证与再演练晋级闸门（Restore Repair Candidate Validation & Re-drill Promotion Gate）

上一课讲的是：稳定基线归档的 restore drill 失败后，先判断是 `archive_corrupt`、`restore_path_broken`、`baseline_invalid` 还是 `sandbox_violation`，再决定修归档、修恢复路径、重开基线案件或隔离演练环境。

这一课继续往后走：**修复计划执行完，不等于恢复能力已经回来了**。成熟 Agent 要把修复结果当成一个 `RestoreRepairCandidate`，先验证，再隔离重演练，最后才允许刷新归档或重新打开常态恢复通道。

## 1. 为什么不能“修完就过”

restore drill 的特殊点是：它验证的是未来事故里的最后一道保险。如果修复动作本身没有被验证，系统会出现一种很危险的错觉：

- 归档文件看起来补齐了，但 hash lineage 断了
- restore script 能跑通一次，但依赖了本机残留环境
- schema migration 修了当前样本，但旧 snapshot 仍然恢复失败
- sandbox violation 被绕过了，不是被修复了
- OpenClaw cron 写了 lesson、README、TOOLS，但没有证明 Git/Telegram/恢复路径三者一致

所以这里需要一个新闸门：

```text
RestoreRepairCloseoutReceipt
        |
        v
RestoreRepairCandidate
        |
        v
CandidateValidationReport
        |
        v
IsolatedReDrillRun
        |
        v
ReDrillPromotionReceipt
```

目标不是证明“代码改了”，而是证明：**修复候选可以在隔离环境里稳定恢复同一份归档，并且没有引入新的副作用。**

## 2. 最小数据模型

```python
from dataclasses import dataclass
from typing import Literal

FailureKind = Literal[
    "archive_corrupt",
    "restore_path_broken",
    "baseline_invalid",
    "sandbox_violation",
]

Decision = Literal[
    "promote_repaired_archive",
    "rerun_repair",
    "refresh_candidate",
    "quarantine_candidate",
    "reopen_baseline_case",
    "manual_review",
]


@dataclass(frozen=True)
class RestoreRepairCandidate:
    candidate_id: str
    failure_kind: FailureKind
    archive_hash_before: str
    archive_hash_after: str
    restore_path_version: str
    migration_version: str
    sandbox_profile: str
    repair_actions: list[str]


@dataclass(frozen=True)
class CandidateValidationReport:
    candidate_id: str
    lineage_preserved: bool
    patch_targets_original_failure: bool
    schema_compatible: bool
    forbidden_side_effects: list[str]
    missing_artifacts: list[str]
    deterministic_replay_rate: float
    runtime_fingerprint_match: bool


@dataclass(frozen=True)
class IsolatedReDrillRun:
    candidate_id: str
    attempts: int
    successful_restores: int
    smoke_replay_passed: bool
    migration_replay_passed: bool
    external_writes_detected: bool
    max_duration_ms: int
```

注意这里把 `forbidden_side_effects` 和 `external_writes_detected` 分开：

- `forbidden_side_effects` 是验证报告发现的设计风险，比如 restore 脚本会写生产 bucket
- `external_writes_detected` 是隔离重演练时真实捕获到的副作用

一个是静态/计划层，一个是运行层。生产系统两个都要看。

## 3. learn-claude-code：纯函数闸门

教学版可以先写一个很小的判定函数，把工程判断固定下来：

```python
def decide_re_drill_promotion(
    candidate: RestoreRepairCandidate,
    validation: CandidateValidationReport,
    redrill: IsolatedReDrillRun,
) -> Decision:
    if validation.candidate_id != candidate.candidate_id:
        return "manual_review"

    if redrill.candidate_id != candidate.candidate_id:
        return "manual_review"

    if candidate.failure_kind == "baseline_invalid":
        return "reopen_baseline_case"

    if validation.forbidden_side_effects or redrill.external_writes_detected:
        return "quarantine_candidate"

    if validation.missing_artifacts:
        return "refresh_candidate"

    if not validation.lineage_preserved:
        return "refresh_candidate"

    if not validation.patch_targets_original_failure:
        return "rerun_repair"

    if not validation.schema_compatible:
        return "rerun_repair"

    if not validation.runtime_fingerprint_match:
        return "rerun_repair"

    if validation.deterministic_replay_rate < 0.99:
        return "rerun_repair"

    restore_rate = redrill.successful_restores / max(redrill.attempts, 1)

    if restore_rate < 1.0:
        return "rerun_repair"

    if not redrill.smoke_replay_passed:
        return "rerun_repair"

    if not redrill.migration_replay_passed:
        return "rerun_repair"

    if redrill.max_duration_ms > 120_000:
        return "manual_review"

    return "promote_repaired_archive"
```

这个函数的价值不在复杂，而在顺序：

1. 身份对齐：candidate、validation、redrill 必须指向同一个候选
2. 基线无效：不是修归档能解决，直接重开 baseline case
3. 副作用：一票否决，先隔离
4. 缺 artifact / 血缘断裂：刷新候选
5. 修复没命中原故障：继续修
6. 隔离重演练全绿：才允许晋级

## 4. pi-mono：生产版 Worker

生产里不要让 “repair job succeeded” 直接更新 active archive。更稳的做法是让一个 Worker 消费 `RestoreRepairCloseoutReceipt`，生成候选、验证、重演练，再用事务写晋级收据。

```ts
type ReDrillDecision =
  | "promote_repaired_archive"
  | "rerun_repair"
  | "refresh_candidate"
  | "quarantine_candidate"
  | "reopen_baseline_case"
  | "manual_review";

interface RestoreRepairCandidate {
  candidateId: string;
  sourceDrillId: string;
  failureKind:
    | "archive_corrupt"
    | "restore_path_broken"
    | "baseline_invalid"
    | "sandbox_violation";
  archiveHashBefore: string;
  archiveHashAfter: string;
  restorePathVersion: string;
  migrationVersion: string;
  sandboxProfile: string;
}

interface ReDrillPromotionReceipt {
  receiptId: string;
  candidateId: string;
  decision: ReDrillDecision;
  archiveHashAfter: string;
  validationReportId: string;
  redrillRunId: string;
  createdAt: string;
  reasons: string[];
}

class RestoreRepairReDrillWorker {
  constructor(
    private readonly candidates: RestoreRepairCandidateStore,
    private readonly validator: RestoreCandidateValidator,
    private readonly redrillRunner: IsolatedRestoreRunner,
    private readonly archivePointer: ArchivePointerService,
    private readonly receipts: ReceiptStore,
  ) {}

  async handle(closeoutReceiptId: string): Promise<ReDrillPromotionReceipt> {
    const candidate = await this.candidates.createFromCloseout(closeoutReceiptId);
    const validation = await this.validator.validate(candidate.candidateId);
    const redrill = await this.redrillRunner.run({
      candidateId: candidate.candidateId,
      sandboxProfile: "deny-external-writes",
      attempts: 3,
    });

    const { decision, reasons } = decideReDrill(candidate, validation, redrill);

    return this.receipts.transaction(async (tx) => {
      const receipt = await tx.reDrillPromotionReceipts.create({
        candidateId: candidate.candidateId,
        decision,
        archiveHashAfter: candidate.archiveHashAfter,
        validationReportId: validation.reportId,
        redrillRunId: redrill.runId,
        createdAt: new Date().toISOString(),
        reasons,
      });

      if (decision === "promote_repaired_archive") {
        await this.archivePointer.compareAndSwap({
          tx,
          expectedHash: candidate.archiveHashBefore,
          nextHash: candidate.archiveHashAfter,
          receiptId: receipt.receiptId,
        });
      }

      if (decision === "quarantine_candidate") {
        await tx.quarantine.create({
          candidateId: candidate.candidateId,
          reason: reasons.join("; "),
          sourceReceiptId: receipt.receiptId,
        });
      }

      return receipt;
    });
  }
}
```

这里有三个关键点：

- `compareAndSwap`：归档指针必须基于已知旧 hash 切换，避免并发修复互相覆盖
- `deny-external-writes`：恢复演练默认不能写外部系统，除非专门白名单
- `receiptId`：任何指针变化都要能追到验证报告和重演练 run

## 5. OpenClaw 课程 Cron 的实战映射

这个课程 cron 本身就是一个小型恢复闭环：

```text
lesson file created
README updated
TOOLS updated
Telegram sent
git commit pushed
```

如果某一步失败，不能只补跑最后一步。比如 Telegram 发了，但 Git push 失败：

- 本地 lesson、README、TOOLS 是 candidate state
- Telegram message 是外部副作用
- Git commit 是归档点
- push 是远端收敛证明

更稳的做法：

```yaml
restore_repair_candidate:
  candidate_id: lesson-498
  artifacts:
    - lessons/498-restore-repair-candidate-validation-redrill.md
    - README.md
    - ../TOOLS.md
  validation:
    readme_contains_lesson: true
    tools_contains_topic: true
    git_status_clean_after_commit: true
  redrill:
    telegram_preview_generated: true
    no_duplicate_lesson_number: true
    remote_branch_contains_commit: true
  promotion:
    decision: promote_repaired_archive
```

同样的思想可以放到 OpenClaw 的任何自动化里：不是“脚本跑完”，而是“脚本产物、外部副作用、远端状态、审计记录全部对上”。

## 6. 常见坑

### 坑 1：只验证修复后的最新归档

恢复能力要验证旧归档、当前归档和 migration path。只测最新 snapshot，等于默认事故永远发生在最新版本。

### 坑 2：隔离环境太像生产

restore drill 要故意断掉本机缓存、临时文件、环境变量和生产写权限。否则只是证明这台机器上能跑。

### 坑 3：把 sandbox violation 当成权限问题修

如果演练发现写了不该写的系统，不要简单加权限。应该先问：restore 为什么需要写那里？

### 坑 4：没有 CAS

两个 repair candidate 同时完成时，如果没有 `expectedHash`，后完成的可能覆盖先完成的归档指针。

### 坑 5：没有重演练次数

恢复演练至少要多次运行，确认不是靠时间、缓存、随机顺序刚好成功。

## 7. 最小落地清单

- [ ] 每次 repair 生成 `RestoreRepairCandidate`
- [ ] 验证 candidate 是否命中原始 failure kind
- [ ] 校验 archive lineage 和 runtime fingerprint
- [ ] 隔离环境重演练至少 3 次
- [ ] 默认禁止外部写入
- [ ] restore smoke replay 和 migration replay 都要通过
- [ ] active archive 指针用 CAS 切换
- [ ] 晋级、隔离、重跑都写 receipt
- [ ] 回滚点和旧 archive hash 保留到 sentinel 退出

## 8. 总结

恢复演练失败后的修复，不是把脚本改绿就结束。

真正可靠的路径是：

```text
Repair closeout
  -> candidate
  -> validation
  -> isolated re-drill
  -> CAS promotion
  -> receipt
```

一句话：**修复恢复系统，也要像发布生产系统一样，有候选、有验证、有隔离重放、有收据、有回滚点。**
