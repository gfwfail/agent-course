# 448. Agent 退役护栏归档腐化检测与重封存收据（Retired Guard Archive Corruption Detection & Reseal Receipt）

> 关键词：archive corruption、schema drift、evidence hash、reseal receipt、cold archive integrity

上一课讲长期护栏退役后不能直接遗忘，要把 RetirementReceipt、最后证据、回归种子和复发演练计划保存成 RetirementArchive。

今天继续补一个生产里很容易漏掉的点：**冷归档还在，不代表它仍然可审计。**

半年后你可能会遇到这些情况：

- evidence schema 升级，旧证据 JSON 还能读，但字段语义变了；
- hash 算法、redaction 策略、密钥轮换导致旧 hash 无法复验；
- 存储 bucket lifecycle 把附件转到冷层，取回延迟和权限变了；
- PII 政策更新，旧 raw evidence 不能继续保存；
- replay harness 升级，旧 regression seed 结果变成 unknown。

这叫 **Archive Corruption**。它不一定是文件损坏，更常见的是证据语义、权限、保留策略或回放环境腐化。

成熟 Agent 不能只问“文件还在吗”，而要问：

> 这份 retired guard archive 今天还能不能证明当初为什么退役、旧风险是什么、复发时应该怎么重开？

---

## 1. 腐化不是丢文件

冷归档腐化至少有五类：

1. **Byte corruption**：对象缺失、hash 不一致、附件无法读取；
2. **Schema corruption**：字段还在，但版本迁移后含义不再清楚；
3. **Policy corruption**：当前隐私/保留/访问策略不允许按原方式读取；
4. **Replay corruption**：回归种子无法在当前 runtime 解释或重放；
5. **Lineage corruption**：archive 无法追到 RetirementReceipt、ReplacementGuard、Tombstone 或 ReopenCase。

很多团队只做第一类 hash check，于是事故复发时才发现：

- 能下载旧 JSON，但不知道字段代表哪个 policy version；
- 能打开旧日志，但已经不能合法展示 raw payload；
- seed 还在，但 expected decision 没绑定模型、工具 schema、evidence boundary；
- archive 里写了 replacementGuardId，但新系统已经找不到这条 guard。

所以 cold archive 需要一个定期的 **Corruption Sentinel**。

---

## 2. 状态机：corrupted 后不要原地修

建议给 RetirementArchive 增加完整状态：

~~~text
sealed
  -> corruption_scan_due
  -> scan_clean
  -> corruption_detected
  -> reseal_required
  -> resealed
  -> disposal_review
  -> reopen_case
~~~

关键边界：

- sealed 表示 archive 当时封存成功，不表示永久可信；
- scan_clean 要写 CorruptionScanReceipt，记录当前 schema / policy / hash / replay harness；
- corruption_detected 不能直接覆盖旧 archive；
- reseal_required 要生成 ResealPlan，说明是迁移、补索引、降密、延保留，还是重开；
- resealed 必须保留旧 hash 和新 hash 的映射，形成链式证据；
- disposal_review 只在 retention 到期或政策禁止继续保存时触发；
- 如果腐化影响风险判断，就进入 reopen_case，而不是只修归档。

一句话：归档可以迁移，但证据链不能断。

---

## 3. learn-claude-code：腐化扫描纯函数

教学版先写一个纯函数，输入封存档案和当前观测，输出下一步动作。

~~~python
# learn_claude_code/guard_patterns/archive_corruption.py
from dataclasses import dataclass
from typing import Literal

Action = Literal[
    "scan_clean",
    "reseal_required",
    "repair_lineage",
    "disposal_review",
    "open_reopen_case",
    "manual_review",
]


@dataclass(frozen=True)
class SealedArchive:
    archive_id: str
    guard_id: str
    sealed_hash: str
    schema_version: int
    policy_version: str
    replay_harness_version: str
    retained_until_epoch: int
    receipt_ref: str | None
    replacement_guard_ref: str | None
    seed_count: int


@dataclass(frozen=True)
class ArchiveObservation:
    now_epoch: int
    bytes_hash_ok: bool
    readable: bool
    schema_migratable: bool
    current_schema_version: int
    policy_allows_retention: bool
    policy_allows_replay: bool
    receipt_resolves: bool
    replacement_guard_resolves: bool
    replayable_seed_count: int
    unknown_seed_results: int
    recurrence_hits: int


@dataclass(frozen=True)
class CorruptionDecision:
    action: Action
    reason: str
    followups: tuple[str, ...] = ()


def decide_archive_corruption(
    archive: SealedArchive,
    observation: ArchiveObservation,
) -> CorruptionDecision:
    if observation.now_epoch > archive.retained_until_epoch:
        return CorruptionDecision(
            "disposal_review",
            "retention_expired",
            ("prove_no_legal_hold", "write_disposal_or_extension_receipt"),
        )

    if not observation.policy_allows_retention:
        return CorruptionDecision(
            "disposal_review",
            "current_policy_forbids_retention",
            ("declassify_or_destroy_with_receipt",),
        )

    if not observation.readable or not observation.bytes_hash_ok:
        return CorruptionDecision(
            "reseal_required",
            "byte_integrity_failed",
            ("restore_from_replica", "write_reseal_receipt"),
        )

    if observation.current_schema_version > archive.schema_version:
        if not observation.schema_migratable:
            return CorruptionDecision(
                "manual_review",
                "schema_drift_not_migratable",
                ("owner_schema_mapping_review",),
            )
        return CorruptionDecision(
            "reseal_required",
            "schema_version_drift",
            ("migrate_archive_view", "preserve_old_hash_mapping"),
        )

    if not observation.receipt_resolves or not observation.replacement_guard_resolves:
        return CorruptionDecision(
            "repair_lineage",
            "lineage_ref_broken",
            ("reindex_receipt_refs", "verify_replacement_guard_ref"),
        )

    replay_coverage = observation.replayable_seed_count / max(archive.seed_count, 1)
    if not observation.policy_allows_replay or replay_coverage < 0.9:
        return CorruptionDecision(
            "reseal_required",
            "replay_contract_corrupted",
            ("refresh_seed_metadata", "write_replay_harness_mapping"),
        )

    if observation.unknown_seed_results > 0:
        return CorruptionDecision(
            "manual_review",
            "seed_results_unknown_after_replay",
            ("classify_unknown_results",),
        )

    if observation.recurrence_hits > 0:
        return CorruptionDecision(
            "open_reopen_case",
            "retired_risk_recurred_during_corruption_scan",
            ("create_reopen_case_from_archive",),
        )

    return CorruptionDecision("scan_clean", "archive_integrity_policy_and_replay_clean")
~~~

这个函数的重点是：腐化扫描不是“能不能打开文件”，而是同时校验 byte、schema、policy、lineage、replay 五条证据链。

---

## 4. pi-mono：ArchiveCorruptionSentinel

pi-mono 的 session manager 已经把 session JSONL 设计成带版本和迁移逻辑的事件流：SessionHeader 有 version，migrations.ts 会读第一行 header，再把旧 session 文件搬到正确位置。

这类思路可以直接迁移到 guard archive：

~~~ts
type SealedGuardArchive = {
  archiveId: string;
  guardId: string;
  sealedHash: string;
  schemaVersion: number;
  policyVersion: string;
  replayHarnessVersion: string;
  receiptRef: string;
  replacementGuardRef?: string;
  seedRefs: string[];
  retainedUntil: number;
};

type ResealReceipt = {
  receiptId: string;
  archiveId: string;
  oldHash: string;
  newHash: string;
  fromSchemaVersion: number;
  toSchemaVersion: number;
  reason: string;
  migrationProofRef: string;
  createdAt: number;
};

async function scanArchiveCorruption(
  archive: SealedGuardArchive,
  now: number,
): Promise<
  | { action: "scan_clean"; reason: string }
  | { action: "reseal_required"; reason: string; receipt: ResealReceipt }
  | { action: "repair_lineage"; reason: string; repairKey: string }
  | { action: "open_reopen_case"; reason: string; caseKey: string }
> {
  const currentPolicy = await loadEvidencePolicy();
  const currentSchema = await loadArchiveSchemaVersion();
  const bytes = await readColdArchiveBytes(archive.archiveId);
  const hashOk = await verifyHash(bytes, archive.sealedHash);

  if (!hashOk) {
    return {
      action: "reseal_required",
      reason: "sealed_hash_mismatch",
      receipt: await buildResealReceipt(archive, "sealed_hash_mismatch", now),
    };
  }

  if (!currentPolicy.canRetain(archive)) {
    return {
      action: "reseal_required",
      reason: "policy_requires_declassification",
      receipt: await buildResealReceipt(archive, "declassify_for_current_policy", now),
    };
  }

  if (archive.schemaVersion < currentSchema.version) {
    const migrated = await migrateArchiveView(bytes, archive.schemaVersion, currentSchema.version);
    return {
      action: "reseal_required",
      reason: "schema_migration_required",
      receipt: await writeResealedArchive(archive, migrated, now),
    };
  }

  const lineageOk = await refsResolve([
    archive.receiptRef,
    archive.replacementGuardRef,
    ...archive.seedRefs,
  ]);
  if (!lineageOk) {
    return {
      action: "repair_lineage",
      reason: "archive_lineage_ref_broken",
      repairKey: `archive-lineage:${archive.archiveId}:${archive.sealedHash}`,
    };
  }

  const replay = await shadowReplaySeeds(archive.seedRefs, {
    harnessVersion: archive.replayHarnessVersion,
  });
  if (replay.recurrenceHits.length > 0) {
    return {
      action: "open_reopen_case",
      reason: "retired_guard_recurred",
      caseKey: `reopen:${archive.guardId}:${replay.runHash}`,
    };
  }

  return { action: "scan_clean", reason: "archive_verified" };
}
~~~

注意 reseal 不是简单 overwrite。正确做法是：

- oldHash 永久保留；
- newHash 写入新 sealed archive；
- migrationProofRef 记录迁移脚本版本、输入输出字段映射、脱敏策略；
- outbox 发出 ArchiveResealed 事件；
- 后续读取默认读最新 archive，但审计能沿 receipt 链追到旧版本。

这和 session migration 的原则一致：移动或迁移文件可以，但要从 header / version / path 证明迁移后的对象仍是同一条历史链。

---

## 5. OpenClaw：课程 Cron 的现实类比

这个课程 cron 本身就是一个小型证据链：

- lesson 文件证明内容；
- README 目录证明可发现；
- TOOLS 已讲内容证明去重；
- Telegram messageId 证明外部发布；
- git commit sha 证明远端版本；
- memory 日志证明执行过程。

如果半年后只剩 lesson 文件，但 Telegram messageId 丢了、README 没收录、TOOLS 主题不一致，就会出现“内容还在，但证据链腐化”。

所以课程 cron 也可以做 Corruption Sentinel：

~~~text
for each recent lesson:
  verify lesson file exists
  verify README contains lesson link
  verify TOOLS contains topic summary
  verify memory contains messageId and commit sha
  verify remote main contains commit
  if any ref missing:
    create repair_lineage task
~~~

这不是为了形式主义，而是为了让自动课程长期可维护：以后查“这个主题讲没讲过”时，系统能靠证据回答，而不是靠模型记忆猜。

---

## 6. 实战 checklist

做 retired guard archive 时，至少加这几个字段：

~~~json
{
  "archiveId": "retired_guard_course_2026_05",
  "sealedHash": "sha256:old...",
  "schemaVersion": 3,
  "policyVersion": "evidence-policy-2026-05",
  "replayHarnessVersion": "guard-replay-12",
  "receiptRef": "retirement_receipt_447",
  "replacementGuardRef": "long_term_guard_v9",
  "seedRefs": ["seed_001", "seed_002"],
  "retainedUntil": 1800000000,
  "nextCorruptionScanAt": 1780000000
}
~~~

每次扫描输出不可变收据：

~~~json
{
  "type": "CorruptionScanReceipt",
  "archiveId": "retired_guard_course_2026_05",
  "result": "reseal_required",
  "checked": ["bytes", "schema", "policy", "lineage", "replay"],
  "oldHash": "sha256:old...",
  "newHash": "sha256:new...",
  "reason": "schema_version_drift",
  "createdAt": 1780000001
}
~~~

---

## 7. 常见坑

**坑 1：只校验对象 hash。**
hash 正确只能证明 bytes 没变，不能证明当前系统还能解释它。

**坑 2：迁移后覆盖旧 archive。**
这会断掉审计链。要写 ResealReceipt，把 oldHash -> newHash 连起来。

**坑 3：策略不允许读取时直接删除。**
先进入 disposal_review，判断是否能降密、延长 legal hold、保留摘要，最后再写销毁或延展收据。

**坑 4：replay unknown 当通过。**
unknown 是证据缺口，不是 pass。高风险 retired guard 出现 unknown，至少 manual_review。

**坑 5：lineage ref 断了不管。**
归档的价值在于能追溯。receiptRef、replacementGuardRef、seedRefs 断链时，要修索引或重封存。

---

## 8. 总结

RetirementArchive 解决的是“退役后不要遗忘”。

Archive Corruption Sentinel 解决的是“保存下来的证据今天是否仍然可信”。

成熟 Agent 的冷归档不是一堆旧 JSON，而是一条会定期自检、必要时重封存、能证明迁移过程没有改写历史的证据链。退役不是删掉护栏，重封存也不是改写历史；每一次迁移、降密、修链、销毁，都要留下 receipt。
