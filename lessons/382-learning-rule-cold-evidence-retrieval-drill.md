# 382. Agent 学习规则冷证据取回演练与完整性巡检（Cold Evidence Retrieval Drill & Integrity Audit）

上一课讲了 Post-Closeout Evidence Compaction：恢复关闭后，把热路径里的 raw evidence 压缩成 cold summary、canonical hash、regression refs 和 retrieval lease policy。

但冷存储有一个常见陷阱：**写进去的时候都觉得没问题，真正要审计时才发现取不出来、验不过、schema 读不了。**

这不是小问题。学习规则事故恢复后的冷证据，通常要在几个月后回答这些问题：

- 当时为什么关闭 recovery case？
- counterfactual 监控到底通过了哪些预算？
- regression pack 引用的是哪个版本？
- raw bundle hash 是否还能验证？
- 如果要 break-glass 取回 raw evidence，租约边界是否还有效？
- 旧 schema 是否还能被当前系统诚实理解？

所以今天讲：Cold Evidence Retrieval Drill & Integrity Audit。

一句话：**冷证据不是放进冷库就完事；要定期抽样取回、验 hash、验 schema、验 lease、验 replay，证明几年后系统仍然能在正确权限下复盘当年的学习规则恢复。**

## 1. 为什么冷证据需要演练

很多团队的证据冷存储流程长这样：

    closeout -> compaction -> upload cold bundle -> done

看起来完整，实际缺了最关键的一步：定期证明它还能用。

冷证据会因为很多原因慢慢失效：

- 对象存储 key 迁移，index 没更新；
- 加密 key 轮换后，旧 bundle 无法解密；
- schema registry 删除了旧版本；
- canonicalization 规则变化，hash 复算不一致；
- regression refs 指向的 fixture 被重写；
- retrieval lease policy 和当前权限系统脱节；
- summary 还在，但 raw bundle 已过保留期却没有 tombstone；
- audit log 只记录“压缩成功”，没有记录“取回成功”。

这类问题平时不冒烟，事故复盘时才爆。

所以 mature Agent 的标准不是“我压缩过证据”，而是“我最近演练过取回，并且有报告证明它仍然可验证”。

## 2. DrillSpec：把取回演练结构化

不要让 cron 随机挑几个文件读一下。取回演练要有 DrillSpec，声明抽样范围、允许动作、验证项目和失败处理。

一个最小 DrillSpec：

    {
      "drillId": "cold_evidence_drill_20260523",
      "purpose": "post_closeout_audit_readiness",
      "sample": {
        "ruleFamily": "learning_rule_recovery",
        "closedBeforeDays": 7,
        "maxItems": 20,
        "riskWeighted": true
      },
      "checks": [
        "manifest_exists",
        "cold_summary_hash",
        "raw_bundle_hash_or_tombstone",
        "schema_upgrade_path",
        "regression_refs_resolve",
        "retrieval_lease_policy_enforced",
        "audit_event_written"
      ],
      "allowedRawRead": false,
      "failureAction": "open_repair_ticket"
    }

注意 allowedRawRead 默认是 false。大多数演练只需要验证 rawBundleHash、metadata、tombstone 和 lease policy，不需要真的把敏感 raw payload 放回 prompt。

只有 break-glass 演练才会拿短 TTL lease 去取 raw，而且要独立审计。

## 3. learn-claude-code：教学版完整性检查器

教学版可以先做成纯函数：输入 manifest、summary、索引和 registry，输出 drill result。

    from dataclasses import dataclass
    from hashlib import sha256
    import json

    @dataclass(frozen=True)
    class ColdSummary:
        bundle_id: str
        rule_id: str
        schema_version: str
        raw_bundle_hash: str
        regression_refs: list[str]
        summary_hash: str

    @dataclass(frozen=True)
    class CompactionManifest:
        manifest_id: str
        bundle_id: str
        summary_hash: str
        raw_bundle_hash: str
        schema_version: str
        retrieval_lease_policy_id: str
        tombstone_id: str | None = None

    def canonical_hash(value) -> str:
        encoded = json.dumps(
            value,
            sort_keys=True,
            separators=(",", ":"),
            ensure_ascii=False,
        ).encode("utf-8")
        return "sha256:" + sha256(encoded).hexdigest()

    def verify_cold_summary(summary: ColdSummary, manifest: CompactionManifest) -> list[str]:
        failures: list[str] = []

        calculated = canonical_hash({
            "bundleId": summary.bundle_id,
            "ruleId": summary.rule_id,
            "schemaVersion": summary.schema_version,
            "rawBundleHash": summary.raw_bundle_hash,
            "regressionRefs": summary.regression_refs,
        })

        if calculated != summary.summary_hash:
            failures.append("summary_hash_mismatch")

        if summary.summary_hash != manifest.summary_hash:
            failures.append("manifest_summary_hash_mismatch")

        if summary.raw_bundle_hash != manifest.raw_bundle_hash:
            failures.append("raw_bundle_hash_mismatch")

        if not manifest.retrieval_lease_policy_id:
            failures.append("missing_retrieval_lease_policy")

        return failures

这个函数不需要知道对象存储细节。它只回答一个问题：summary 和 manifest 是否还能互相证明。

## 4. DrillRunner：抽样、验证、出报告

取回演练通常不是全量跑。全量成本高，也容易产生不必要的敏感读取。更合理的是 risk-weighted sampling：

- 高风险规则每周抽；
- 最近刚 closeout 的规则优先抽；
- 曾经发生过 hash/schema 问题的 family 加权；
- legal hold / tombstoned / raw expired 都要覆盖；
- 每次演练至少覆盖一种旧 schema。

教学版 runner：

    def run_cold_evidence_drill(manifests, load_summary, registry, lease_policy_store):
        results = []

        for manifest in manifests:
            summary = load_summary(manifest.bundle_id)
            failures = verify_cold_summary(summary, manifest)

            if not registry.can_read(manifest.schema_version):
                failures.append("schema_version_unreadable")

            if not lease_policy_store.exists(manifest.retrieval_lease_policy_id):
                failures.append("lease_policy_missing")

            if not summary.regression_refs:
                failures.append("missing_regression_refs")

            results.append({
                "manifestId": manifest.manifest_id,
                "bundleId": manifest.bundle_id,
                "status": "pass" if not failures else "fail",
                "failures": failures,
            })

        return {
            "passed": sum(1 for item in results if item["status"] == "pass"),
            "failed": sum(1 for item in results if item["status"] == "fail"),
            "items": results,
        }

这里重点是输出可机器处理的 result，而不是只在日志里写一句 checked ok。

## 5. pi-mono：事件驱动的冷证据巡检 Worker

生产版可以把它放进后台任务系统，不要塞进主 Agent Loop：

    eventBus.subscribe("learning.cold_evidence.compacted", async (event) => {
      await drillScheduler.schedule({
        jobId: "drill:" + event.manifestId,
        runAfter: addDays(event.createdAt, 7),
        manifestId: event.manifestId,
        reason: "first_post_compaction_drill",
      });
    });

定期 worker：

    async function runColdEvidenceDrill(job) {
      const manifest = await manifestStore.load(job.manifestId);
      const summary = await coldSummaryStore.load(manifest.summaryRef);

      const result = await coldEvidenceAuditor.verify({
        manifest,
        summary,
        schemaRegistry,
        regressionPackStore,
        leasePolicyStore,
        rawReadMode: "metadata_only",
      });

      await auditLog.append({
        type: "learning.cold_evidence.drill_result",
        drillId: job.jobId,
        manifestId: manifest.manifestId,
        status: result.status,
        failures: result.failures,
        checkedAt: new Date().toISOString(),
      });

      if (result.status !== "pass") {
        await repairQueue.enqueue({
          type: "cold_evidence_integrity_repair",
          manifestId: manifest.manifestId,
          failures: result.failures,
          severity: result.severity,
        });
      }

      return result;
    }

这里不要自动修所有东西。比如 schema unreadable 可以补 registry adapter；但 raw hash mismatch 可能是证据损坏或篡改，必须进入更严格的 incident 流程。

## 6. 取回演练不等于泄露演练

冷证据取回有三种层级：

    metadata_only
      只读 manifest、index、summary hash、retention/tombstone metadata。

    proof_only
      读取 summary、receipt、regression refs，复算 canonical hash。

    break_glass_raw
      通过短 TTL lease 读取 raw bundle，必须带 purpose、approval、maxReads 和 no-prompt/no-external 约束。

默认演练应该是 metadata_only 或 proof_only。

break_glass_raw 只用于两种情况：

- 定期小样本验证冷存储真的能解密；
- 事故审计必须复核 raw。

而且 raw 取回以后要明确禁止：

- 自动写入长期记忆；
- 自动进入 prompt；
- 自动发到外部渠道；
- 被子 Agent 继承；
- 超过 TTL 后继续缓存。

## 7. OpenClaw 实战：课程 Cron 的冷证据演练

拿这个课程 cron 举例。每 3 小时一课，closeout 证据包括：

- lesson markdown 文件；
- README 目录行；
- TOOLS 已讲内容；
- Telegram messageId；
- git commit hash；
- remote main verification；
- memory/YYYY-MM-DD.md 完成记录。

上一课已经把这些压缩成 cold summary。本课的演练可以这样做：

    {
      "manifestId": "compact_agent_course_381",
      "checks": {
        "lessonFileExists": true,
        "readmeEntryExists": true,
        "toolsTopicExists": true,
        "telegramMessageIdRecorded": true,
        "gitCommitReachableOnOriginMain": true,
        "memorySummaryExists": true
      },
      "result": "pass"
    }

如果某个检查失败：

- README 行没了：补目录并记录 repair；
- commit 不在 origin/main：阻断后续课程 push，先查 repo 状态；
- TOOLS 去重列表丢了：恢复已讲内容，避免重复选题；
- messageId 缺失：不能假装发过，标记为 incomplete evidence；
- lesson 文件 hash 变了：确认是修订还是误改。

这就是 OpenClaw always-on Agent 的现实收益：不是“讲完课就算了”，而是下一次 cron 能证明上一批课的证据还在、能查、能验。

## 8. Repair Ticket：失败后怎么收口

Drill fail 以后不要只报警。要生成结构化 Repair Ticket：

    {
      "ticketId": "repair_cold_evidence_20260523_001",
      "manifestId": "compact_agent_course_381",
      "failure": "regression_ref_missing",
      "severity": "medium",
      "repairPlan": [
        "locate regression fixture by closeoutReceiptId",
        "restore cold index ref",
        "rerun proof_only drill",
        "append repair receipt"
      ],
      "allowedActions": [
        "metadata_read",
        "index_update",
        "receipt_append"
      ],
      "forbiddenActions": [
        "rewrite_original_manifest",
        "delete_tombstone",
        "raw_read_without_lease"
      ]
    }

核心原则：**修复是追加，不是篡改历史。**

原 manifest 错了也不要原地改掉。追加 Repair Receipt，记录新 index、新 adapter、新 tombstone 或新 summary hash，并保留原始失败证据。

## 9. 最小检查清单

做冷证据巡检时，至少问 10 个问题：

- [ ] manifest 是否存在且 hash 可复算？
- [ ] cold summary 是否存在且 hash 匹配？
- [ ] rawBundleHash 是否还在，或是否有合法 tombstone？
- [ ] schemaVersion 是否有 upgrade/read adapter？
- [ ] regression refs 是否还能 resolve？
- [ ] closeout receipt 是否还能关联 rule version？
- [ ] retrieval lease policy 是否还存在？
- [ ] 默认演练是否避免 raw payload 进入 prompt？
- [ ] drill result 是否写入 append-only audit log？
- [ ] 失败时是否生成 Repair Ticket，而不是静默跳过？

## 10. 这一课的关键

第 381 课解决的是：closeout 后怎么把证据从热路径压到冷存储。

第 382 课解决的是：压进去以后，怎么证明它还活着、没坏、能取、能验、权限边界还在。

成熟 Agent 的证据系统不是一个仓库，而是一套持续证明机制：

    compaction
      把证据降温。

    retrieval drill
      证明证据还能取回。

    integrity audit
      证明证据没有坏。

    repair receipt
      证明坏了以后怎么修。

记住一句话：

**没有演练过的冷证据，只是一个未来审计事故。**
