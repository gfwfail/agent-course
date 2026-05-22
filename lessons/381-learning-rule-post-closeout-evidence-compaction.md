# 381. Agent 学习规则关闭后的证据压缩与冷存储（Post-Closeout Evidence Compaction & Cold Storage）

上一课讲了 Post-Thaw Closeout：反事实监控通过后，系统要生成关闭收据、归档 shadow 版本、解除临时 guardrail，并把最终版本 pin 住。

但 closeout 之后还有一个很实际的问题：**证据不能永远热存，也不能随手删。**

学习规则事故恢复会留下很多材料：

- raw tool outputs；
- counterfactual diff events；
- shadow replay snapshots；
- runtime observations；
- regression fixtures；
- Telegram / Git / README / TOOLS 的完成证据；
- closeout receipt。

如果全部长期放在热路径里，成本会上升、检索会变慢、隐私风险也会扩大。可如果直接删除，下次复盘、审计、回归测试又没证据。

所以今天讲：Post-Closeout Evidence Compaction & Cold Storage。

一句话：**closeout 之后，把恢复证据从“可执行热数据”压缩成“可验证冷证据”，保留 hash、摘要、索引和取回路径，让系统既能忘掉细节，又能证明自己曾经正确处理过。**

## 1. 为什么 closeout 后还要压缩证据

很多 Agent 系统的证据生命周期只有两档：

    active evidence
    deleted evidence

这太粗了。恢复刚结束时，证据需要被频繁读取；几周后，它更多是审计和回归材料；几个月后，可能只需要证明“当时存在过、由谁签发、摘要是什么、如有授权可以取回”。

更合理的分层是：

    hot
      当前恢复窗口、发布闸门、runtime gate 直接读取。

    warm
      近期复盘、回归、人工审查会读取，保留摘要和必要字段。

    cold
      默认不进 prompt，不参与实时决策，只保留索引、hash、receipt 和受控取回方式。

    tombstoned
      原始证据已按策略删除，只保留删除墓碑和允许保留的合成 claim。

核心目标不是“省硬盘”，而是控制证据扩散半径。

## 2. Compaction Manifest 应该长什么样

压缩不能靠一个 cron 默默搬文件。要有 manifest，说明哪些证据被压缩、保留了什么、丢弃了什么、以后怎么验证。

一个最小 Compaction Manifest：

    {
      "manifestId": "compact_topic_dedup_recovery_20260523",
      "ruleId": "learning.agent_course.topic_dedup",
      "recoveryCaseId": "recovery_20260522_001",
      "closeoutReceiptId": "closeout_topic_dedup_v8",
      "fromTier": "warm",
      "toTier": "cold",
      "createdAt": "2026-05-23T14:30:00Z",
      "retention": {
        "rawDeleteAfter": "2026-06-22T00:00:00Z",
        "summaryRetainUntil": "2027-05-23T00:00:00Z",
        "legalHold": false
      },
      "kept": [
        "receipt",
        "summary",
        "canonicalHash",
        "schemaVersion",
        "evidenceRefs",
        "regressionCaseRefs",
        "retrievalLeasePolicy"
      ],
      "dropped": [
        "rawPrompt",
        "rawToolOutput",
        "shadowIntermediateTrace"
      ],
      "hashes": {
        "rawBundleHash": "sha256:...",
        "summaryHash": "sha256:...",
        "manifestHash": "sha256:..."
      }
    }

注意这里的 dropped 不是随便删，而是“按策略不再热存”。如果 raw 还有合规保留要求，可以转到加密冷存；如果 subject 要求删除，则走 tombstone。

## 3. learn-claude-code：教学版压缩器

教学版可以先做成纯函数：输入证据 bundle，输出 cold summary 和 manifest。不要一上来就接对象存储、KMS、生命周期策略。

    from dataclasses import dataclass
    from hashlib import sha256
    import json
    from typing import Literal

    Tier = Literal["hot", "warm", "cold"]

    @dataclass(frozen=True)
    class EvidenceBundle:
        bundle_id: str
        rule_id: str
        recovery_case_id: str
        closeout_receipt_id: str
        raw_text: str
        summary: str
        regression_refs: list[str]
        evidence_refs: list[str]
        schema_version: str

    def canonical_hash(value) -> str:
        encoded = json.dumps(
            value,
            sort_keys=True,
            separators=(",", ":"),
            ensure_ascii=False,
        ).encode("utf-8")
        return "sha256:" + sha256(encoded).hexdigest()

    def compact_evidence(bundle: EvidenceBundle, to_tier: Tier = "cold") -> dict:
        raw_hash = canonical_hash({
            "bundleId": bundle.bundle_id,
            "rawText": bundle.raw_text,
            "schemaVersion": bundle.schema_version,
        })

        summary_doc = {
            "bundleId": bundle.bundle_id,
            "ruleId": bundle.rule_id,
            "recoveryCaseId": bundle.recovery_case_id,
            "closeoutReceiptId": bundle.closeout_receipt_id,
            "summary": bundle.summary,
            "regressionRefs": bundle.regression_refs,
            "evidenceRefs": bundle.evidence_refs,
            "schemaVersion": bundle.schema_version,
            "rawBundleHash": raw_hash,
        }

        manifest = {
            "manifestId": f"compact_{bundle.bundle_id}",
            "ruleId": bundle.rule_id,
            "recoveryCaseId": bundle.recovery_case_id,
            "closeoutReceiptId": bundle.closeout_receipt_id,
            "toTier": to_tier,
            "kept": ["summary", "rawBundleHash", "regressionRefs", "evidenceRefs"],
            "droppedFromHotPath": ["rawText"],
            "summaryHash": canonical_hash(summary_doc),
        }

        return {"summary": summary_doc, "manifest": manifest}

这里的关键不是代码复杂，而是边界清楚：LLM 后续默认只能看到 summary_doc；raw_text 只通过受控 lease 读取。

## 4. 和任务系统结合：压缩也要可恢复

learn-claude-code 的任务系统思路适合继续用：每次 compaction 都是一条可恢复任务。

    {
      "id": 381,
      "subject": "compact post-closeout evidence",
      "status": "in_progress",
      "ruleId": "learning.agent_course.topic_dedup",
      "requiredInputs": [
        "closeoutReceipt",
        "counterfactualSummary",
        "regressionPackRef",
        "runtimeObservation"
      ],
      "outputs": [
        "compactionManifest",
        "coldSummary",
        "tombstoneOrRawLeasePolicy"
      ]
    }

这样即使 cron 中断，也能知道压缩到哪一步，而不是留下半冷半热的数据。

## 5. pi-mono：把压缩放到 closeout 事件之后

pi-mono 生产版不要让主 Agent Loop 同步做冷存储。更合理的是事件驱动：

    eventBus.subscribe("learning.recovery_closeout.finalized", async (event) => {
      await compactionQueue.enqueue({
        jobId: "compact:" + event.closeoutReceiptId,
        ruleId: event.ruleId,
        recoveryCaseId: event.recoveryCaseId,
        closeoutReceiptId: event.closeoutReceiptId,
        evidenceRefs: event.evidenceRefs,
      });
    });

后台 worker 再做真正的压缩：

    async function runEvidenceCompaction(job) {
      const lock = await locks.acquire(job.jobId);
      if (!lock.ok) return;

      const existing = await manifestStore.findByJobId(job.jobId);
      if (existing) return existing;

      const bundle = await evidenceStore.loadBundle(job.evidenceRefs);
      const summary = await summarizer.createColdSummary(bundle, {
        redactSecrets: true,
        keepReasonCodes: true,
        keepRegressionRefs: true,
      });

      const manifest = await compactor.writeColdBundle({
        job,
        summary,
        rawBundleHash: hashCanonical(bundle),
        tier: "cold",
      });

      await hotIndex.removeRawRefs(job.evidenceRefs);
      await coldIndex.add({
        ruleId: job.ruleId,
        closeoutReceiptId: job.closeoutReceiptId,
        manifestId: manifest.manifestId,
        summaryHash: manifest.summaryHash,
      });

      return manifest;
    }

这里有两个重点：

1. worker 幂等；
2. 从 hot index 移除 raw refs，但 cold index 保留可验证入口。

## 6. 冷摘要不是普通摘要

普通摘要追求“读起来像人话”。冷摘要追求“以后能复盘”。

一个合格 cold summary 至少保留：

- ruleId / ruleVersion；
- recoveryCaseId / closeoutReceiptId；
- 事故类型和触发信号；
- 最终决策；
- 关键 reason codes；
- counterfactual metrics；
- regression refs；
- raw bundle hash；
- retention policy；
- retrieval lease policy。

示例：

    {
      "type": "learning.recovery.cold_summary",
      "ruleId": "learning.agent_course.topic_dedup",
      "finalVersion": "v8",
      "incidentClass": "duplicate_topic_selection",
      "decision": "finalized",
      "reasonCodes": [
        "counterfactual_budget_passed",
        "runtime_observation_consistent",
        "regression_pack_appended"
      ],
      "metrics": {
        "samples": 42,
        "primaryTooLoose": 0,
        "unknownDiffRate": 0.04
      },
      "regressionRefs": ["regression:topic_dedup_duplicate_20260522"],
      "rawBundleHash": "sha256:...",
      "rawAccess": {
        "mode": "lease_required",
        "maxTtlMinutes": 30,
        "allowedPurposes": ["audit", "incident_review"]
      }
    }

这类摘要可以进知识库和检索索引；raw 证据不应该默认进 prompt。

## 7. OpenClaw 实战：课程 Cron 怎么用

拿本课程 Cron 举例。每次课程完成后，我们已经有四类证据：

- lesson 文件；
- README 目录；
- TOOLS.md 已讲内容；
- Telegram messageId；
- git commit 和远端校验。

如果某个 topic_dedup 学习规则经历过恢复流程，closeout 后可以把这些证据压缩成：

    {
      "ruleId": "learning.agent_course.topic_dedup",
      "lesson": "381-learning-rule-post-closeout-evidence-compaction.md",
      "telegramMessageId": 12547,
      "gitCommit": "abc123",
      "readmeIndexed": true,
      "toolsMemoryUpdated": true,
      "coldSummary": "第 381 课成功发布，选题未重复，完成文件/群消息/Git/TOOLS 四重证据闭环。",
      "rawBundleHash": "sha256:..."
    }

以后选题去重时，只需要检索 lesson slug、标题、摘要和 hash，不需要把整篇课文、完整聊天、完整 git diff 都塞进上下文。

这就是压缩的价值：**保留行为影响需要的最小事实，降低长期上下文和隐私成本。**

## 8. 冷存储的访问闸门

冷存储不是“随便没人看”，而是“看之前要说明原因”。

建议给 raw evidence retrieval 加三道门：

1. purpose gate：只允许 audit、incident_review、legal_hold、regression_rebuild；
2. lease gate：短 TTL，限制读取次数；
3. projection gate：默认只返回 summary，需要 raw 必须更高权限。

伪代码：

    def authorize_cold_retrieval(request, manifest):
        if request.purpose not in manifest["rawAccess"]["allowedPurposes"]:
            return "deny"

        if request.ttl_minutes > manifest["rawAccess"]["maxTtlMinutes"]:
            return "deny"

        if request.view == "raw" and not request.has_break_glass:
            return "summary_only"

        return "allow_with_lease"

这样 LLM 不会因为“有历史证据”就无限制翻旧账。

## 9. 常见坑

**坑 1：把摘要当证据。**

摘要是证据视图，不是原始事实。必须保留 rawBundleHash、签名或 receipt，否则以后无法证明摘要没被改写。

**坑 2：压缩后丢掉 regression refs。**

恢复闭环最有价值的产物之一就是回归用例。冷摘要如果不指向 regression pack，下次发布前就挡不住同类事故。

**坑 3：冷存储没有 schema version。**

一年后 schema 改了，如果 cold summary 没版本，迁移和审计都会变成猜谜。

**坑 4：raw 删除没有 tombstone。**

删除不是消失。要留下 tombstone，告诉下游“这份 raw 不可再用”，避免旧索引、旧缓存把它复活。

## 10. 设计检查清单

做 post-closeout evidence compaction 时，至少检查：

- closeout receipt 是否存在？
- raw bundle 是否有 canonical hash？
- cold summary 是否保留 reason codes 和 regression refs？
- schemaVersion 是否明确？
- hot index 是否移除了 raw refs？
- cold index 是否能按 ruleId / recoveryCaseId / receiptId 找回？
- raw 访问是否需要 lease？
- 删除 raw 时是否写 tombstone？
- compaction worker 是否幂等？
- manifest 是否能独立解释“保留了什么、丢弃了什么、为什么”？

## 总结

Closeout 证明恢复结束，compaction 证明恢复证据进入了正确生命周期。

成熟 Agent 不能把所有历史都永远塞进热上下文，也不能为了省事把证据删干净。正确做法是：热路径只留最小可执行事实，冷路径保留可验证摘要、hash、索引、retention 和受控取回。

这样系统既能长期学习，又不会被自己的历史证据拖慢、拖贵、拖危险。
