# 371. Agent 学习规则晋级决策记录与人工复核队列（Learning Rule Promotion Decision Record & Review Queue）

上一课讲了 Evidence Sufficiency Gate：再验证 probe 跑完以后，要判断证据是否足够让学习规则重新影响真实动作。

今天继续往后走一步：**证据充分性闸门给出 promote / hold / demote / manual_review 以后，不能只改一个状态字段；必须生成可审计、可恢复、可人工处理的 Promotion Decision Record。**

很多 Agent 学习系统会在这里偷懒：

- gate 返回 promote，就直接把 rule tier 改成 enforce；
- gate 返回 hold，就只留一行日志；
- gate 返回 manual_review，就发一条消息给人，然后状态散在聊天记录里；
- 失败重试时，不知道上一次到底决定了什么、有没有执行成功。

这会让学习规则的发布链路变成黑盒。真正的问题不是“能不能晋级”，而是：**谁基于哪些证据做了什么晋级决定，这个决定有没有被执行，失败后怎么恢复，人类复核意见怎么回写到规则生命周期里。**

一句话：**Sufficiency Gate 负责判断证据够不够，Promotion Decision Record 负责把判断变成可审计的发布事实，Review Queue 负责接住需要人处理的边界案例。**

## 1. 为什么不能直接改 rule tier

假设有一条学习规则：

    ruleId: lr_check_pr_before_push
    currentTier: shadow_only
    requestedTier: enforce
    sufficiency: promote
    evidenceRefs:
      - ev_replay_370
      - ev_dryrun_gh_pr_list
      - ev_dependency_snapshot_gh_2_74

如果系统直接把它改成 enforce，会丢掉几个关键信息：

- promote 是哪个 gate 版本算出来的？
- 当时的 confidence cap 是多少？
- 是否有 missing evidence 但被人工 override？
- policy cache、prompt pack、retrieval index 是否都已经刷新？
- 如果刷新一半失败，下次重试应该从哪里继续？
- 以后发现误伤，如何找到这次晋级对应的证据包？

所以状态变更前要先创建不可变的 Promotion Decision Record：

    {
      "decisionId": "pdr_20260521_1530_lr_check_pr_before_push_v5",
      "ruleId": "lr_check_pr_before_push",
      "fromTier": "shadow_only",
      "toTier": "enforce",
      "outcome": "promote",
      "gateVersion": "learning-evidence-sufficiency@3",
      "evidenceRefs": ["ev_replay_370", "ev_dryrun_gh_pr_list"],
      "confidenceCap": 0.91,
      "executionState": "pending",
      "createdAt": "2026-05-21T05:30:00Z"
    }

后续真正改 rule store、policy cache、prompt pack、regression index，都围绕这个 decisionId 做幂等执行。

## 2. Decision Record 的最小字段

一个够用的 Promotion Decision Record 至少要包含这些字段：

    {
      "decisionId": "pdr_xxx",
      "ruleId": "lr_xxx",
      "ruleVersion": 5,
      "fromTier": "shadow_only",
      "requestedTier": "enforce",
      "approvedTier": "enforce",
      "outcome": "promote",
      "reasonCodes": ["sufficiency_met", "fresh_dependency_snapshot"],
      "missingEvidence": [],
      "evidenceRefs": ["ev_1", "ev_2"],
      "gateVersion": "learning-evidence-sufficiency@3",
      "policyVersion": "learning-promotion-policy@2",
      "confidenceCap": 0.91,
      "review": null,
      "executionState": "pending",
      "createdAt": "2026-05-21T05:30:00Z"
    }

这里有几个关键点：

- fromTier / requestedTier / approvedTier 要分开，人工复核可能把 requested enforce 降成 warn_or_confirm。
- outcome 是决策结果，不等于执行状态；promote 也可能执行失败。
- reasonCodes 要机器可读，方便统计“为什么规则不能晋级”。
- evidenceRefs 不存 raw payload，只存引用，避免把敏感证据复制到发布记录里。
- gateVersion / policyVersion 必须保留，否则以后 replay 不出同样结论。

## 3. Review Queue 不是聊天提醒

manual_review 不能只发一句“请人工确认”。它应该进入一个结构化队列：

    {
      "reviewId": "lr_review_123",
      "decisionId": "pdr_20260521_1530_lr_payment_rule_v2",
      "ruleId": "lr_payment_rule",
      "riskTier": "security_decision",
      "requestedTier": "enforce",
      "recommendedTier": "shadow_only",
      "reasonCodes": ["missing_independent_source", "high_false_positive_rate"],
      "reviewOptions": ["approve_warn", "request_more_evidence", "reject", "approve_enforce"],
      "expiresAt": "2026-05-28T05:30:00Z",
      "state": "open"
    }

人工复核输出也要结构化：

    {
      "reviewId": "lr_review_123",
      "reviewer": "owner:bot001",
      "decision": "request_more_evidence",
      "comment": "需要补一轮 current dry-run 和 false positive replay",
      "nextProbeKinds": ["dry_run_tool", "false_positive_replay"],
      "createdAt": "2026-05-21T05:35:00Z"
    }

这样 scheduler 可以自动接续：review 不是终点，而是把系统导向补证据、降级、拒绝或受控晋级。

## 4. learn-claude-code：文件式 Promotion Ledger

learn-claude-code 适合用文件式 ledger 讲清楚模型。一个最小实现可以这样写：

    from dataclasses import dataclass, asdict
    from datetime import datetime, timezone
    from pathlib import Path
    import json

    @dataclass(frozen=True)
    class PromotionDecisionRecord:
        decision_id: str
        rule_id: str
        rule_version: int
        from_tier: str
        requested_tier: str
        approved_tier: str
        outcome: str
        reason_codes: list[str]
        missing_evidence: list[str]
        evidence_refs: list[str]
        gate_version: str
        policy_version: str
        confidence_cap: float
        execution_state: str
        created_at: str

    def build_decision_record(rule, sufficiency) -> PromotionDecisionRecord:
        now = datetime.now(timezone.utc).isoformat()
        approved_tier = {
            "promote": sufficiency.target_tier,
            "hold": rule.current_tier,
            "demote": "shadow_only",
            "manual_review": rule.current_tier,
        }[sufficiency.decision]

        return PromotionDecisionRecord(
            decision_id=f"pdr_{rule.rule_id}_v{rule.version}_{int(datetime.now().timestamp())}",
            rule_id=rule.rule_id,
            rule_version=rule.version,
            from_tier=rule.current_tier,
            requested_tier=sufficiency.target_tier,
            approved_tier=approved_tier,
            outcome=sufficiency.decision,
            reason_codes=sufficiency.reason_codes,
            missing_evidence=sufficiency.missing_evidence,
            evidence_refs=sufficiency.evidence_refs,
            gate_version="learning-evidence-sufficiency@3",
            policy_version="learning-promotion-policy@1",
            confidence_cap=sufficiency.confidence_cap,
            execution_state="pending",
            created_at=now,
        )

    def append_promotion_ledger(path: Path, record: PromotionDecisionRecord) -> None:
        path.parent.mkdir(parents=True, exist_ok=True)
        with path.open("a", encoding="utf-8") as f:
            f.write(json.dumps(asdict(record), ensure_ascii=False, sort_keys=True) + "\n")

    def needs_review(record: PromotionDecisionRecord) -> bool:
        return record.outcome == "manual_review"

关键不是代码复杂，而是三个边界：

- record 先写入 ledger，再执行状态变更；
- ledger append-only，不原地覆盖历史决策；
- manual_review 进入队列，而不是散落在日志和聊天消息里。

## 5. pi-mono：把 Promotion 变成事件驱动状态机

pi-mono 的 Agent.subscribe / EventStream 模型很适合做这层：sufficiency gate 产出事件，promotion worker 消费事件，执行状态变更，再发 execution result。

伪代码：

    type PromotionOutcome = "promote" | "hold" | "demote" | "manual_review";
    type ExecutionState = "pending" | "applied" | "failed" | "superseded";

    interface PromotionDecisionRecord {
      decisionId: string;
      ruleId: string;
      ruleVersion: number;
      fromTier: string;
      requestedTier: string;
      approvedTier: string;
      outcome: PromotionOutcome;
      reasonCodes: string[];
      missingEvidence: string[];
      evidenceRefs: string[];
      gateVersion: string;
      policyVersion: string;
      confidenceCap: number;
      executionState: ExecutionState;
      createdAt: string;
    }

    async function createPromotionDecision(ctx, sufficiency): Promise<PromotionDecisionRecord> {
      const record: PromotionDecisionRecord = {
        decisionId: ctx.ids.next("pdr"),
        ruleId: sufficiency.ruleId,
        ruleVersion: sufficiency.ruleVersion,
        fromTier: sufficiency.currentTier,
        requestedTier: sufficiency.targetTier,
        approvedTier: sufficiency.outcome === "promote" ? sufficiency.targetTier : sufficiency.currentTier,
        outcome: sufficiency.outcome,
        reasonCodes: sufficiency.reasonCodes,
        missingEvidence: sufficiency.missingEvidence,
        evidenceRefs: sufficiency.evidenceRefs,
        gateVersion: "learning-evidence-sufficiency@3",
        policyVersion: "learning-promotion-policy@2",
        confidenceCap: sufficiency.confidenceCap,
        executionState: "pending",
        createdAt: new Date().toISOString(),
      };

      await ctx.promotionLedger.append(record);
      await ctx.eventStream.publish({
        type: "learning.promotion_decision_recorded",
        decisionId: record.decisionId,
        ruleId: record.ruleId,
        outcome: record.outcome,
        evidenceRefs: record.evidenceRefs,
      });

      if (record.outcome === "manual_review") {
        await ctx.reviewQueue.enqueue({
          decisionId: record.decisionId,
          ruleId: record.ruleId,
          reasonCodes: record.reasonCodes,
          missingEvidence: record.missingEvidence,
          state: "open",
        });
      }

      return record;
    }

真正 apply 时要按 decisionId 幂等：

    async function applyPromotionDecision(ctx, decisionId: string) {
      const record = await ctx.promotionLedger.get(decisionId);
      if (record.executionState !== "pending") return record;

      if (record.outcome !== "promote" && record.outcome !== "demote") {
        return record;
      }

      await ctx.ruleStore.updateTier(record.ruleId, record.ruleVersion, record.approvedTier);
      await ctx.policyCache.invalidateByRule(record.ruleId);
      await ctx.promptPackIndex.rebuildRule(record.ruleId);
      await ctx.promotionLedger.markApplied(decisionId);
    }

这层的工程价值很直接：系统重启、worker 重跑、消息重复投递，都不会重复晋级或漏掉 cache invalidation。

## 6. OpenClaw 实战：课程 Cron 的发布决策

这个课程 Cron 每次都要做外部动作：发 Telegram、写 lesson、更新 README、commit、push。它也可以套一个小型 Promotion Decision Record。

本轮课程的发布前证据应该类似：

    evidenceRefs:
      - rg TOOLS.md 未命中 "学习规则晋级决策记录与人工复核队列"
      - README.md 最新目录到 370
      - lessons/371 文件已生成
      - git diff --check 通过
      - Telegram send 返回 messageId
      - git ls-remote 验证远端 main 包含 commit

对应 decision record：

    {
      "decisionId": "course_371_publish",
      "outcome": "promote",
      "fromTier": "draft",
      "approvedTier": "published",
      "reasonCodes": ["topic_not_duplicate", "docs_updated", "delivery_verified"],
      "executionState": "applied"
    }

这听起来像“课程发课也要这么正式吗？”但原则是一样的：只要 Agent 会自动产生外部副作用，就应该能回答：

- 发了什么；
- 为什么可以发；
- 发到哪里；
- 对应哪个 Git commit；
- 失败后是否能重试；
- 下次如何避免重复。

## 7. 实战建议

落地时别一开始就做复杂 UI。先做三张表或三个 JSONL 文件就够：

    promotion-decisions.jsonl
    review-queue.jsonl
    promotion-executions.jsonl

流程：

1. sufficiency gate 只负责输出结构化结论；
2. promotion recorder 把结论写成 append-only decision record；
3. manual_review 写入 review queue；
4. promotion executor 按 decisionId 幂等 apply；
5. execution result 回写 applied / failed / superseded；
6. 所有外部副作用引用 decisionId。

成熟 Agent 的学习发布链路，不是“模型觉得可以就上线”，而是：**证据判断、发布决策、人工复核、执行状态、回滚线索，每一步都能追踪。**

## 8. 小结

今天这课的核心：

- Sufficiency Gate 的输出不能直接改生产状态；
- Promotion Decision Record 是学习规则晋级的审计事实；
- Review Queue 要结构化，不能只靠聊天提醒；
- apply promotion 要按 decisionId 幂等；
- OpenClaw 这类 cron 自动化任务，也应该把“为什么可以发、发了什么、如何验证”串成证据链。

下一次再讲更后一层：Promotion apply 以后，如何做 **Rule Activation Receipt 与 Prompt Pack / Policy Cache 一致性校验**，避免规则状态已经变了，但运行时上下文还没同步。
