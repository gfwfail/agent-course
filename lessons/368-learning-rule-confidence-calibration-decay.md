# 368. Agent 学习规则置信度校准与时间衰减（Learning Rule Confidence Calibration & Decay）

上一课讲了 Synthetic Substitute：raw evidence 被删除以后，可以用允许保留的抽象 claim 重新派生一条 shadow 规则。

今天补一个生产里很容易漏掉的问题：**学习规则不是二元状态，不应该只有 active / retired。**

一条规则刚上线、长期没命中、证据变旧、证据被替换成 synthetic evidence、工具 schema 变了、最近 outcome 变差，这些情况都不一定要求立刻删除规则，但都应该降低它的置信度。置信度低到某个阈值后，规则要从 active 降到 advisory、shadow、manual_review，甚至重新评估。

一句话：**Learning Rule 要像模型预测一样校准 confidence，也要像缓存一样有 decay。**

## 1. 为什么 active 不等于永远可信

很多 Agent 系统会把学习规则写成这样：

    {
      "ruleId": "lr_check_pr_before_push",
      "status": "active",
      "text": "git push 前检查 PR 是否已 merge"
    }

这在小系统里能跑，但在长期运行的 Agent 里会出问题。

因为规则的可信度会随时间变化：

1. 当初触发这条规则的事故已经很久没有复发；
2. 依赖的 GitHub / CI / policy 行为变了；
3. 原始 evidence 被 tombstone，只剩 synthetic evidence；
4. 最近规则命中时经常造成 false positive；
5. 规则 scope 太宽，阻断了本来安全的操作。

这些都不是“规则立刻无效”，而是“规则的 confidence 下降”。成熟系统要把下降过程显式建模，而不是等到误伤后才回滚。

## 2. 最小模型：confidence + decay + action tier

建议给每条学习规则维护三个值：

    {
      "ruleId": "lr_check_pr_before_push",
      "version": 4,
      "status": "active",
      "confidence": 0.86,
      "lastCalibratedAt": "2026-05-21T06:30:00+10:00",
      "evidenceFreshnessDays": 12,
      "recentInfluence": {
        "preventedBadRuns": 4,
        "falsePositiveBlocks": 1,
        "unknownOutcomes": 3
      },
      "actionTier": "enforce"
    }

confidence 不是凭感觉填的，它至少来自四类信号：

- evidence quality：证据类型、是否 synthetic、是否过期、是否可重放；
- replay score：反事实评估里 preventedBadAllows / falsePositives 的比例；
- production outcome：真实命中后的 preventedBadRuns、blockedGoodRuns、unknownOutcomes；
- dependency freshness：工具 schema、policy、prompt pack、scope 是否发生漂移。

然后把 confidence 映射成 action tier：

    confidence >= 0.85 -> enforce
    confidence >= 0.65 -> warn_or_confirm
    confidence >= 0.45 -> shadow_only
    otherwise          -> re_evaluate_or_retire

这比单纯 active / retired 更细。它允许 Agent 在不确定时降级，而不是继续强硬执行旧经验。

## 3. 时间衰减不是简单过期

decay 的重点不是“过了 30 天就删”，而是让旧规则逐渐失去强制力。

一个实用公式：

    adjusted_confidence =
      base_confidence
      * evidence_multiplier
      * dependency_multiplier
      * time_decay
      * outcome_multiplier

其中：

- raw verified evidence：evidence_multiplier = 1.0；
- synthetic evidence：evidence_multiplier = 0.85；
- summary only：evidence_multiplier = 0.7；
- dependency drift detected：dependency_multiplier = 0.6；
- 30 天没有命中：time_decay 可以从 1.0 降到 0.8；
- 最近 false positive 偏高：outcome_multiplier 降到 0.5 或更低。

注意，时间衰减应该按 riskSurface 区分。安全规则、权限规则、资金规则可以衰减慢一点；UI 偏好、临时流程、项目状态类学习应该衰减快一点。

## 4. learn-claude-code：教学版校准器

learn-claude-code 的教学版可以用一个纯函数表达规则校准。它不需要数据库，只要输入规则、证据和近期 outcome，就能输出 action tier。

    from dataclasses import dataclass
    from math import exp
    from typing import Literal

    EvidenceType = Literal["raw_verified", "synthetic", "summary_only"]
    ActionTier = Literal["enforce", "warn_or_confirm", "shadow_only", "re_evaluate_or_retire"]

    @dataclass
    class LearningRule:
        rule_id: str
        base_confidence: float
        risk_surface: Literal["security", "side_effect", "preference"]
        days_since_evidence: int
        evidence_type: EvidenceType
        dependency_drift: bool

    @dataclass
    class InfluenceStats:
        prevented_bad_runs: int
        false_positive_blocks: int
        unknown_outcomes: int

    def evidence_multiplier(kind: EvidenceType) -> float:
        return {
            "raw_verified": 1.0,
            "synthetic": 0.85,
            "summary_only": 0.7,
        }[kind]

    def time_decay(rule: LearningRule) -> float:
        half_life = {
            "security": 180,
            "side_effect": 90,
            "preference": 30,
        }[rule.risk_surface]
        return exp(-rule.days_since_evidence / half_life)

    def outcome_multiplier(stats: InfluenceStats) -> float:
        total = stats.prevented_bad_runs + stats.false_positive_blocks + stats.unknown_outcomes
        if total == 0:
            return 0.9
        good = stats.prevented_bad_runs / total
        bad = stats.false_positive_blocks / total
        return max(0.2, min(1.2, 0.8 + good - bad * 1.5))

    def calibrate(rule: LearningRule, stats: InfluenceStats) -> tuple[float, ActionTier]:
        dependency = 0.6 if rule.dependency_drift else 1.0
        confidence = (
            rule.base_confidence
            * evidence_multiplier(rule.evidence_type)
            * dependency
            * time_decay(rule)
            * outcome_multiplier(stats)
        )

        if confidence >= 0.85:
            tier = "enforce"
        elif confidence >= 0.65:
            tier = "warn_or_confirm"
        elif confidence >= 0.45:
            tier = "shadow_only"
        else:
            tier = "re_evaluate_or_retire"
        return round(confidence, 3), tier

这里的重点不是公式多完美，而是把“这条学习还靠不靠谱”变成可测试、可解释的函数。

## 5. pi-mono：LearningConfidenceMiddleware

pi-mono 的 Agent loop 已经围绕 EventStream 和 tool execution 组织，适合把规则置信度校准做成执行前中间件。

流程可以是：

1. 检索候选 Learning Rules；
2. 读取最近 influence events；
3. 对每条规则计算 calibrated confidence；
4. 根据 action tier 决定 enforce / warn / shadow / skip；
5. 把 calibration event 写回事件流，供后续监控和回放。

    type ActionTier = "enforce" | "warn_or_confirm" | "shadow_only" | "re_evaluate_or_retire";

    interface LearningRule {
      id: string;
      version: number;
      baseConfidence: number;
      evidenceType: "raw_verified" | "synthetic" | "summary_only";
      daysSinceEvidence: number;
      dependencyDrift: boolean;
      riskSurface: "security" | "side_effect" | "preference";
    }

    interface CalibratedRule extends LearningRule {
      calibratedConfidence: number;
      actionTier: ActionTier;
    }

    async function calibrateLearningRules(ctx, rules: LearningRule[]): Promise<CalibratedRule[]> {
      const output: CalibratedRule[] = [];

      for (const rule of rules) {
        const stats = await ctx.learningInfluenceStore.recentStats(rule.id, rule.version);
        const calibrated = calculateConfidence(rule, stats);

        await ctx.eventStream.publish({
          type: "learning.confidence_calibrated",
          ruleId: rule.id,
          version: rule.version,
          confidence: calibrated.confidence,
          actionTier: calibrated.actionTier,
          inputs: {
            evidenceType: rule.evidenceType,
            dependencyDrift: rule.dependencyDrift,
            daysSinceEvidence: rule.daysSinceEvidence,
            stats,
          },
        });

        if (calibrated.actionTier !== "re_evaluate_or_retire") {
          output.push({ ...rule, calibratedConfidence: calibrated.confidence, actionTier: calibrated.actionTier });
        }
      }

      return output;
    }

执行工具前，只有 enforce tier 可以直接阻断或强制改写行为；warn_or_confirm 只加提示或要求确认；shadow_only 只记录“如果启用会怎样”，不能影响真实副作用。

## 6. OpenClaw 实战：课程 Cron 的去重规则也要衰减

OpenClaw 课程 Cron 里，TOOLS.md 的“已讲内容”就是一个很朴素的 Learning Rule Store：它告诉我不要重复讲过的主题。

但这里也有置信度问题：

- 最近 20 课的主题，重复风险高，应该强 enforce；
- 一年前讲过但方向已经演进的主题，可以 warn，不一定 block；
- 如果旧条目只有标题没有摘要，confidence 要低一点；
- 如果 README / lesson 文件和 TOOLS.md 不一致，要触发 dependency drift；
- 如果某课被重写或 tombstone，后续引用要降级到 synthetic / summary。

文件系统版可以这样落地：

    memory/learning-rule-confidence.jsonl
    {"ruleId":"topic:no-repeat","version":1,"confidence":0.94,"tier":"enforce","evidence":"README+TOOLS match"}
    {"ruleId":"topic:old-summary-only","version":1,"confidence":0.58,"tier":"shadow_only","evidence":"TOOLS only, no lesson file"}

这能避免两个极端：一是完全无视历史导致重复；二是把很久以前、上下文已经变了的条目永久当成硬禁令。

## 7. 常见坑

第一，把 confidence 当人工标签。置信度应该由证据、回放、outcome 和漂移信号计算出来，人工只能调整权重，不能随手填分。

第二，只按时间衰减。时间只是一个维度，最近 false positive、dependency drift、synthetic evidence 都应该影响置信度。

第三，低 confidence 直接删除。更稳的做法是先 shadow / re-evaluate，把影响路径和回放证据补齐。

第四，不记录校准事件。没有 learning.confidence_calibrated 事件，就无法解释为什么某条规则今天从 enforce 变成 warn。

第五，所有规则同一半衰期。安全、资金、副作用、偏好类学习的 decay 速度必须不同。

## 8. 生产落地清单

- Learning Rule 增加 baseConfidence、calibratedConfidence、lastCalibratedAt；
- 证据类型区分 raw_verified / synthetic / summary_only；
- influence event 记录 preventedBadRuns、falsePositiveBlocks、unknownOutcomes；
- dependency drift 会降低 confidence 或触发 replay；
- confidence 映射到 enforce / warn_or_confirm / shadow_only / re_evaluate_or_retire；
- 不同 riskSurface 使用不同 decay 半衰期；
- 每次校准写 learning.confidence_calibrated 审计事件；
- 低置信度规则先降级或重评估，不直接静默删除；
- 外部副作用工具只允许 enforce tier 的规则强制阻断。

核心思想：**学习规则不是写进系统就永久正确。成熟 Agent 要持续问自己：这条经验今天还有多可信？可信到什么程度，才配影响真实行动？**
