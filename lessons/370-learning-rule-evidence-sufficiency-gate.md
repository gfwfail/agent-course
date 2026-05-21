# 370. Agent 学习规则证据充分性闸门（Learning Rule Evidence Sufficiency Gate）

上一课讲了 Revalidation Scheduler：学习规则 confidence 降级后，系统要主动安排 readonly replay、metadata probe、dry-run 和 sandbox probe，重新收集证据。

今天补上再验证后最关键的一步：**探针跑完不代表规则可以重新生效；必须先判断证据是否充分。**

很多 Agent 学习系统会犯一个隐蔽错误：只要某个 probe 成功，就把规则从 shadow 拉回 enforce。问题是，单个 probe 只能证明一个角度：

- metadata probe 只能证明依赖还存在；
- readonly replay 只能证明旧案例里表现不错；
- dry-run 只能证明当前状态不会立刻失败；
- sandbox probe 只能证明隔离环境可行。

这些都不是“生产环境可重新强制执行”的充分证据。成熟做法是加一个 **Evidence Sufficiency Gate**：把证据按风险等级、来源独立性、新鲜度、覆盖率和反例数量评分，只有满足门槛的学习规则才能 promote。

一句话：**再验证负责收集证据，证据充分性闸门负责判断这些证据够不够让规则重新影响真实动作。**

## 1. 为什么不能只看一个成功信号

假设有一条学习规则：

    ruleId: lr_check_pr_before_push
    decision: git push 前必须检查 PR 是否已经 merge
    actionTier: shadow_only
    confidence: 0.61

Revalidation 跑出了这些结果：

    metadata_probe: pass
    readonly_replay: pass 8/10
    dry_run_tool: skipped
    dependency_snapshot: gh cli version changed

这时能不能 promote 到 enforce？

不能急。因为证据有明显缺口：

- GitHub CLI 版本变了，输出格式可能变；
- readonly replay 有 2 个失败或 unknown case；
- 没有当前分支 dry-run 证据；
- 没有证明规则不会误伤正常 push；
- 没有记录证据是否来自独立来源。

所以 gate 应该输出：

    decision: keep_shadow
    missingEvidence:
      - current_state_dry_run
      - false_positive_analysis
      - dependency_compatibility_snapshot

这比“probe 成功，所以恢复 enforce”稳得多。

## 2. 证据充分性矩阵

建议把 evidence sufficiency 做成显式矩阵，而不是写在 prompt 里靠模型感觉判断：

    {
      "riskTier": "external_side_effect",
      "required": {
        "minIndependentSources": 2,
        "minReplayCases": 20,
        "maxUnknownRate": 0.05,
        "maxFalsePositiveRate": 0.02,
        "requiresFreshDependencySnapshot": true,
        "requiresDryRun": true
      }
    }

按风险分层：

    preference            -> 低门槛，允许少量 unknown
    planning_hint         -> 中门槛，不直接阻断动作
    external_side_effect  -> 高门槛，要 dry-run + replay + dependency snapshot
    security_decision     -> 最高门槛，要独立证据、低误伤、人工或策略授权

重点不是追求“证据越多越好”，而是每个风险等级都知道最低证据组合是什么。

## 3. Gate 的输入输出

输入应该是结构化 evidence bundle：

    {
      "ruleId": "lr_check_pr_before_push",
      "version": 4,
      "targetTier": "enforce",
      "riskTier": "external_side_effect",
      "evidence": [
        {"kind": "readonly_replay", "cases": 42, "pass": 40, "unknown": 1, "falsePositive": 1},
        {"kind": "metadata_probe", "fresh": true, "dependencyHash": "sha256:abc"},
        {"kind": "dry_run_tool", "fresh": true, "result": "pass"}
      ]
    }

输出必须可执行、可审计：

    {
      "decision": "promote",
      "targetTier": "enforce",
      "confidenceCap": 0.91,
      "evidenceRefs": ["ev_replay_42", "ev_dep_abc", "ev_dryrun_01"],
      "missingEvidence": [],
      "reason": "meets external_side_effect sufficiency policy"
    }

如果不够，就不要让后续系统猜：

    {
      "decision": "hold",
      "targetTier": "shadow_only",
      "missingEvidence": ["dry_run_tool", "false_positive_budget"],
      "reason": "not enough fresh evidence for enforce"
    }

这类输出可以直接接到 promotion gate、policy cache 和 audit log。

## 4. learn-claude-code：教学版充分性闸门

learn-claude-code 的核心 agent loop 很简单：LLM 产生 tool_use，系统执行工具，再把 tool_result 放回消息。这个项目适合用纯函数把机制讲清楚，不需要一上来做复杂中间件。

可以新增一个教学版 gate：

    from dataclasses import dataclass
    from typing import Literal

    RiskTier = Literal["preference", "planning_hint", "external_side_effect", "security_decision"]
    Decision = Literal["promote", "hold", "demote", "manual_review"]

    @dataclass
    class ReplayEvidence:
        cases: int
        passed: int
        unknown: int
        false_positive: int

    @dataclass
    class EvidenceBundle:
        rule_id: str
        risk_tier: RiskTier
        has_fresh_dependency_snapshot: bool
        has_dry_run: bool
        independent_sources: int
        replay: ReplayEvidence

    @dataclass
    class SufficiencyDecision:
        decision: Decision
        target_tier: str
        missing_evidence: list[str]
        confidence_cap: float

    def evaluate_sufficiency(bundle: EvidenceBundle) -> SufficiencyDecision:
        missing: list[str] = []
        replay = bundle.replay

        unknown_rate = replay.unknown / max(replay.cases, 1)
        false_positive_rate = replay.false_positive / max(replay.cases, 1)

        if bundle.risk_tier in ("external_side_effect", "security_decision"):
            if not bundle.has_fresh_dependency_snapshot:
                missing.append("fresh_dependency_snapshot")
            if not bundle.has_dry_run:
                missing.append("dry_run_evidence")
            if bundle.independent_sources < 2:
                missing.append("second_independent_source")
            if replay.cases < 20:
                missing.append("minimum_replay_cases")
            if unknown_rate > 0.05:
                missing.append("unknown_rate_below_5_percent")
            if false_positive_rate > 0.02:
                missing.append("false_positive_rate_below_2_percent")

        if bundle.risk_tier == "security_decision" and missing:
            return SufficiencyDecision("manual_review", "shadow_only", missing, 0.70)

        if missing:
            return SufficiencyDecision("hold", "shadow_only", missing, 0.80)

        return SufficiencyDecision("promote", "enforce", [], 0.93)

这里有两个细节：

- confidenceCap 是上限，不是最终 confidence；最终还要交给 confidence calibration。
- missingEvidence 是机器可读字段，后续 scheduler 可以自动补 probe。

## 5. pi-mono：把 gate 放在事件流之后

pi-mono 里已经能通过 session.subscribe 观察工具执行事件，也有 sandbox executor 的 timeout / abort / docker 隔离。Evidence Sufficiency Gate 适合放在 revalidation worker 和 promotion gate 之间。

伪代码可以这样接：

    type RiskTier = "preference" | "planning_hint" | "external_side_effect" | "security_decision";
    type SufficiencyOutcome = "promote" | "hold" | "demote" | "manual_review";

    interface EvidenceBundle {
      ruleId: string;
      version: number;
      riskTier: RiskTier;
      evidenceRefs: string[];
      replayCases: number;
      unknownRate: number;
      falsePositiveRate: number;
      independentSources: number;
      hasFreshDependencySnapshot: boolean;
      hasDryRunEvidence: boolean;
    }

    async function evaluateLearningEvidence(ctx, bundle: EvidenceBundle) {
      const missing: string[] = [];

      if (bundle.riskTier === "external_side_effect" || bundle.riskTier === "security_decision") {
        if (!bundle.hasFreshDependencySnapshot) missing.push("fresh_dependency_snapshot");
        if (!bundle.hasDryRunEvidence) missing.push("dry_run_evidence");
        if (bundle.independentSources < 2) missing.push("independent_source");
        if (bundle.replayCases < 20) missing.push("minimum_replay_cases");
        if (bundle.unknownRate > 0.05) missing.push("unknown_rate_budget");
        if (bundle.falsePositiveRate > 0.02) missing.push("false_positive_budget");
      }

      const outcome: SufficiencyOutcome =
        missing.length === 0
          ? "promote"
          : bundle.riskTier === "security_decision"
            ? "manual_review"
            : "hold";

      await ctx.eventStream.publish({
        type: "learning.evidence_sufficiency_decided",
        ruleId: bundle.ruleId,
        version: bundle.version,
        outcome,
        missingEvidence: missing,
        evidenceRefs: bundle.evidenceRefs,
      });

      return { outcome, missingEvidence: missing };
    }

它和 tool middleware 的关系：

- tool middleware 负责执行前权限、timeout、sandbox、kill switch；
- revalidation worker 负责跑 probe；
- evidence sufficiency gate 负责判断 probe 结果够不够；
- promotion gate 才负责真正改变 rule tier。

不要把这四层揉成一个“聪明函数”，否则出事故时很难定位到底是证据不够、策略不允许，还是工具执行失败。

## 6. OpenClaw 实战：课程 Cron 的发课证据

这个课程 Cron 每 3 小时自动发一课，本身就需要一个小型 sufficiency gate。因为它会做三个外部动作：

1. 发 Telegram 群消息；
2. 写课件和 README；
3. git commit / push。

所以“主题没重复”不能只靠一句记忆。至少要有这些证据：

    evidence:
      - TOOLS.md 已讲内容 rg 未命中候选主题
      - README.md 最新目录未包含候选 slug
      - lessons/*.md 未存在候选文件
      - git status 确认没有未处理冲突
      - Telegram send 返回 messageId
      - git ls-remote 验证远端 main 包含新 commit

可以写成 JSONL：

    {"type":"learning.evidence_sufficiency_decided","ruleId":"agent-course:no-repeat","outcome":"promote","missingEvidence":[]}
    {"type":"course.publish.proof","lesson":370,"telegramMessageId":12345,"commit":"abc1234","remoteVerified":true}

这样以后如果 README、TOOLS.md、lessons 之间不一致，系统不会只凭一个文件继续发课，而是先把规则降级到 hold，补 probe 后再执行。

## 7. 常见坑

第一，把 evidence count 当充分性。10 条同源旧日志，不如 2 条独立新证据。

第二，只看成功率，不看误伤率。学习规则最危险的不是没拦住坏事，而是开始拦正常动作。

第三，证据充分性和置信度混在一起。充分性回答“能不能进入某个 tier”，置信度回答“进入后相信多少”。

第四，missing evidence 不结构化。只写“证据不够”没法自动补 probe；必须写清楚缺什么。

## 8. 小结

学习规则从 shadow 回到 enforce，应该走这条链路：

    revalidation plan
      -> probes collect evidence
      -> evidence sufficiency gate
      -> confidence calibration
      -> promotion gate
      -> audit event

成熟 Agent 不只是会重新验证旧经验，还要知道“证据够不够让旧经验重新掌权”。Evidence Sufficiency Gate 把这个判断从模型直觉变成可审计、可测试、可补证的工程闸门。
