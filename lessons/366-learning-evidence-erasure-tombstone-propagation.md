# 366. Agent 学习证据删除传播与 Tombstone 闸门（Learning Evidence Erasure & Tombstone Propagation）

上一课讲了 Learning Evidence Access Lease：raw evidence 默认不可见，只能用短 TTL 租约读取，并且读取全程审计。

今天继续补一个生产级细节：**当 evidence 被删除、过期、租户要求清除，或者 subject 发起删除请求时，已经从这份 evidence 学出来的规则怎么办？**

如果只删 raw evidence，不处理下游学习规则，就会出现危险状态：原始证据不可用，学习规则还在 prompt、policy 或 retrieval index 里继续影响决策，审计时却无法证明来源。

一句话：**evidence 删除不是文件删除，而是一次影响传播。Tombstone 要沿着 evidence -> learning rule -> prompt pack -> policy cache -> regression pack 全链路扩散。**

## 1. 为什么不能只删除 raw evidence

学习系统里 evidence 不是孤立文件，它会派生出很多下游资产：Learning Candidate、Learning Rule、Prompt Pack、Policy Guardrail、Regression Case 和 Audit Proof。

所以 evidence 被删除时，系统至少要问三个问题：哪些规则依赖它？这些规则还能不能继续影响决策？如果不能，应该 quarantine、rederive、retire，还是保留 proof-only 摘要？

这就是 Tombstone Propagation。

## 2. Tombstone 是可审计的删除证明

删除敏感 evidence 时，不要只留下空洞。要写一个 Tombstone：

    {
      "tombstoneId": "ts_01HY...",
      "evidenceId": "ev_2026_05_20_12328",
      "subject": "tenant:rskins",
      "reason": "subject_erasure_request",
      "deletedAt": "2026-05-20T14:30:00Z",
      "rawDestroyed": true,
      "allowedResiduals": ["proof_hash", "audit_event", "tombstone"],
      "propagation": {
        "rulesQuarantined": ["lr_42"],
        "promptPacksInvalidated": ["prompt_pack_18"],
        "regressionCasesRetired": ["case_77"]
      }
    }

Tombstone 的作用不是保存 raw evidence，而是保存最小删除证明：防止同一个 evidence id 被重新误用，让审计知道删除发生过，让依赖扫描器能继续发现“某条学习规则引用了已删除证据”。

## 3. 学习规则要声明 evidence dependencies

如果 Learning Rule 只写自然语言，删除传播时就没法定位影响面。生产版至少要记录 evidenceRefs：

    {
      "id": "lr_42",
      "status": "active",
      "rule": "Agent 课程发布前必须检查 TOOLS.md 已讲内容，避免重复主题",
      "evidenceRefs": [
        {
          "evidenceId": "ev_2026_05_20_12328",
          "role": "source",
          "requiredFor": "activation",
          "residualAllowedAfterErasure": false
        }
      ],
      "derivedProof": {
        "proofHash": "sha256:...",
        "claimIds": ["claim_check_topics_before_publish"]
      }
    }

关键字段是 requiredFor：activation 表示证据没了规则不能继续 active；audit_only 表示只影响审计；regression 表示删除后要退役或重建 case；explanation 表示不能再展示原始来源。

## 4. learn-claude-code：教学版 Tombstone Propagator

learn-claude-code 的教学版可以继续用文件和纯函数做清楚。核心是：输入 evidence tombstone 和 learning rules，输出每条规则的新状态与传播事件。

    from dataclasses import dataclass, field
    from typing import Literal
    import time

    RuleStatus = Literal["active", "shadow", "quarantined", "retired"]

    @dataclass
    class EvidenceRef:
        evidence_id: str
        required_for_activation: bool
        residual_allowed_after_erasure: bool = False

    @dataclass
    class LearningRule:
        rule_id: str
        status: RuleStatus
        text: str
        evidence_refs: list[EvidenceRef]
        quarantine_reason: str | None = None

    @dataclass
    class EvidenceTombstone:
        tombstone_id: str
        evidence_id: str
        subject: str
        reason: Literal["expired", "subject_erasure_request", "legal_delete", "secret_leak"]
        deleted_at: float
        allowed_residuals: list[str] = field(default_factory=lambda: ["proof_hash", "audit_event", "tombstone"])

    def propagate_tombstone(tombstone: EvidenceTombstone, rules: list[LearningRule]):
        events: list[dict] = []
        for rule in rules:
            refs = [ref for ref in rule.evidence_refs if ref.evidence_id == tombstone.evidence_id]
            if not refs:
                continue
            hard_dependency = any(
                ref.required_for_activation and not ref.residual_allowed_after_erasure
                for ref in refs
            )
            if hard_dependency and rule.status in ("active", "shadow"):
                old_status = rule.status
                rule.status = "quarantined"
                rule.quarantine_reason = f"evidence_tombstoned:{tombstone.evidence_id}"
                events.append({
                    "type": "learning.rule.quarantined",
                    "ruleId": rule.rule_id,
                    "from": old_status,
                    "to": rule.status,
                    "tombstoneId": tombstone.tombstone_id,
                    "reason": tombstone.reason,
                })
        events.append({
            "type": "evidence.tombstone.propagated",
            "evidenceId": tombstone.evidence_id,
            "tombstoneId": tombstone.tombstone_id,
            "ts": time.time(),
        })
        return rules, events

这个教学版表达了生产核心：每条规则显式声明依赖哪些 evidence；evidence 删除后扫描所有依赖；hard dependency 命中就 quarantine；不是直接删除规则，而是保留状态转移和审计事件。

## 5. pi-mono：TombstonePropagationMiddleware

在 pi-mono 这种生产实现里，Tombstone Propagation 应该放在学习规则存储层和 prompt pack 生成层之间。

流程是：EvidenceRetentionWorker 产生 tombstone；LearningRuleStore 查找引用该 evidence 的规则；TombstonePropagationMiddleware 更新规则状态；PromptPackBuilder 跳过 quarantined / retired 规则；Policy cache 和 retrieval index 收到 invalidation event。

    type RuleStatus = "active" | "shadow" | "quarantined" | "retired";

    interface EvidenceRef {
      evidenceId: string;
      requiredFor: "activation" | "audit_only" | "regression" | "explanation";
      residualAllowedAfterErasure: boolean;
    }

    interface LearningRule {
      id: string;
      status: RuleStatus;
      evidenceRefs: EvidenceRef[];
      quarantineReason?: string;
      updatedAt: number;
    }

    async function propagateEvidenceTombstone(tombstone, ruleStore, events) {
      const rules = await ruleStore.findByEvidenceId(tombstone.evidenceId);
      for (const rule of rules) {
        const refs = rule.evidenceRefs.filter((ref) => ref.evidenceId === tombstone.evidenceId);
        const blocksActivation = refs.some(
          (ref) => ref.requiredFor === "activation" && !ref.residualAllowedAfterErasure,
        );
        if (blocksActivation && (rule.status === "active" || rule.status === "shadow")) {
          const previousStatus = rule.status;
          rule.status = "quarantined";
          rule.quarantineReason = "evidence_tombstoned:" + tombstone.evidenceId;
          rule.updatedAt = Date.now();
          await ruleStore.save(rule);
          await events.emit("learning.rule.quarantined", {
            ruleId: rule.id, previousStatus, tombstoneId: tombstone.tombstoneId,
            evidenceId: tombstone.evidenceId, reason: tombstone.reason,
          });
        }
      }
      await events.emit("learning.prompt_pack.invalidate_requested", {
        evidenceId: tombstone.evidenceId, tombstoneId: tombstone.tombstoneId,
      });
      await events.emit("learning.retrieval_index.invalidate_requested", {
        evidenceId: tombstone.evidenceId, tombstoneId: tombstone.tombstoneId,
      });
    }

最重要的是：PromptPackBuilder 必须只读取 active 且 evidence 状态可用的规则。不能指望异步清理永远及时，运行时也要二次校验。

    function canInjectRule(rule: LearningRule, tombstonedEvidenceIds: Set<string>) {
      if (rule.status !== "active") return false;
      return !rule.evidenceRefs.some((ref) => {
        if (!tombstonedEvidenceIds.has(ref.evidenceId)) return false;
        return ref.requiredFor === "activation" && !ref.residualAllowedAfterErasure;
      });
    }

这样即使异步传播慢了一分钟，已删除证据派生的规则也不会继续进入 prompt。

## 6. OpenClaw 实战：memory / TOOLS / Git 也要跟着传播

OpenClaw 这类 Always-on Agent 很适合展示这个问题。比如课程 Cron 每次会产生四类证据：Telegram messageId、Git commit hash、memory/YYYY-MM-DD.md 运行事实、TOOLS.md 已讲内容。

如果某条课程内容需要删除或修正，不能只删 lesson 文件。应该有一个 propagation checklist：

    1. 写 tombstone：说明删除哪个 lesson / message / commit evidence，原因是什么
    2. 查依赖：README、TOOLS 已讲内容、memory 日志、后续课程引用
    3. 下游处理：README 删除或替换链接，TOOLS 标记 retired / superseded，memory 保留最小 tombstone，后续课程更新引用
    4. 验证：rg 旧 evidenceId / topic / messageId，git diff --check，提交传播结果

这就是文件系统版本的 Tombstone Propagation。它比数据库系统朴素，但原则完全一样：**删除不是让历史消失，而是让系统停止依赖不该继续依赖的东西。**

## 7. 常见坑

第一，删除 raw 后不扫依赖。这会造成“数据删了，行为还在”的隐私问题。

第二，Tombstone 里塞太多原始内容。Tombstone 只保留最小证明，不应该变成 raw evidence 的复制品。

第三，直接 hard delete learning rule。这会破坏审计链。更好的做法是 quarantined 或 retired，并记录原因和 tombstoneId。

第四，只做异步清理，不做运行时注入校验。异步任务一定会延迟或失败，prompt 注入前必须检查 evidence dependency 是否仍然有效。

第五，不处理 regression case。如果回归样本包含被删除 subject 的 raw 输入，就要退役、脱敏重建，或者改成 synthetic fixture。

## 8. 生产落地清单

- Evidence Store 支持 tombstone，不复用 evidence id；
- Learning Rule 显式记录 evidenceRefs 和 requiredFor；
- Evidence 删除时触发 propagation event；
- active / shadow 规则遇到 hard dependency tombstone 自动 quarantine；
- Prompt Pack / retrieval index / policy cache 收到 invalidation；
- prompt 注入前二次校验 tombstoned evidence；
- Regression Case 支持 retire 或 synthetic rebuild；
- Tombstone 只保留 proof hash、audit event 和最小 metadata；
- 每次 propagation 都写 append-only audit log。

## 9. 关键 takeaway

成熟 Agent 的学习系统，不只是会从 evidence 学规则，还要会在 evidence 被删除后停止依赖它。

**Tombstone Propagation 把“删除数据”升级成“删除影响”：raw 清掉，派生规则隔离，prompt 不再注入，索引失效，审计仍能证明发生过什么。**

这一步做完，学习系统才真正具备隐私删除、证据过期、合规审计和长期安全运行的基本能力。
