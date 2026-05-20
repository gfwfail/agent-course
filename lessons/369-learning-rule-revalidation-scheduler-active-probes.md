# 369. Agent 学习规则再验证调度器与主动探针（Learning Rule Revalidation Scheduler & Active Probes）

上一课讲了 Learning Rule 的 confidence calibration：规则不是只有 active / retired，而是要根据证据、回放、真实 outcome 和时间衰减计算可信度。

今天继续往前走一步：**置信度下降以后，Agent 不能只是等它烂掉，也不能立刻删掉；应该主动安排再验证。**

很多长期运行的 Agent 都会遇到这个问题：一条学习规则两个月没命中，依赖的工具 schema 变了，证据从 raw evidence 降级成 synthetic evidence，confidence 从 0.91 掉到 0.58。此时最差的做法有两个：

1. 继续 enforce，拿旧经验强行影响生产动作；
2. 直接 retire，把可能仍然有价值的经验扔掉。

更好的做法是给学习系统加一个 **Revalidation Scheduler**：定期挑出需要复核的规则，用只读 replay、dry-run、schema check、small probe 等方式刷新证据，重新计算 confidence，再决定 promote / keep shadow / retire。

一句话：**学习规则会过期，再验证调度器负责让有价值的旧经验重新拿到证据。**

## 1. 什么情况下需要再验证

不是所有规则都要每天验证。建议用 revalidation trigger 控制成本：

    {
      "ruleId": "lr_check_pr_before_push",
      "version": 4,
      "confidence": 0.58,
      "actionTier": "shadow_only",
      "lastValidatedAt": "2026-04-21T09:30:00+10:00",
      "nextRevalidateAt": "2026-05-21T09:30:00+10:00",
      "triggers": [
        "confidence_below_warn_threshold",
        "dependency_drift_detected",
        "evidence_aged_out"
      ]
    }

常见触发条件：

- confidence 低于 enforce 阈值；
- evidence 超过 reviewAfter；
- tool schema / policy / prompt pack 版本变化；
- 最近 false positive 或 unknown outcome 增多；
- rule scope 被扩大或迁移过；
- raw evidence 被 tombstone，只剩 synthetic evidence；
- 高风险规则长期没有 replay proof。

注意，这里的再验证不是“重新相信它”，而是“重新收集证据”。结论仍然要经过 quality gate、counterfactual replay 和 confidence calibration。

## 2. Probe 必须默认只读

Revalidation 的第一原则：**不能为了验证学习规则而制造新的外部副作用。**

所以 probe 要分层：

    readonly_replay   -> 用历史 decision events 回放
    dry_run_tool      -> 调工具 dry-run / validate 模式
    metadata_probe    -> 查 schema、权限、配置、状态
    sandbox_probe     -> 在隔离环境模拟
    live_canary       -> 只有低风险且有明确授权时才允许

绝大多数学习规则只需要前四类。比如“git push 前检查 PR 是否已 merge”这条规则，再验证可以做：

- 读取最近 git push / gh pr list 的历史事件；
- 回放旧规则是否仍能阻断危险路径；
- 检查 GitHub CLI 输出 schema 是否变了；
- 在本地 dry-run 判断当前分支状态；
- 不需要真的 push 一次。

成熟 Agent 的再验证应该像测试，而不是像生产操作。

## 3. Revalidation Plan：把复核变成显式任务

建议每次再验证都生成一个 plan：

    {
      "planId": "rv_lr_check_pr_before_push_v4_20260521",
      "ruleId": "lr_check_pr_before_push",
      "version": 4,
      "reason": "confidence_below_warn_threshold",
      "allowedProbeKinds": ["readonly_replay", "metadata_probe", "dry_run_tool"],
      "blockedProbeKinds": ["live_canary"],
      "requiredEvidence": [
        "schema_snapshot",
        "counterfactual_replay_report",
        "current_policy_hash"
      ],
      "deadline": "2026-05-22T09:30:00+10:00"
    }

Plan 有三个好处：

1. 可审计：为什么验证、允许做什么、禁止做什么都写清楚；
2. 可恢复：任务中断后可以继续跑，不靠当前上下文记忆；
3. 可门控：没有 requiredEvidence，就不能重新 promote 到 enforce。

## 4. learn-claude-code：教学版调度器

learn-claude-code 的教学版可以把 revalidation scheduler 写成纯函数：输入规则列表和当前时间，输出需要执行的 plan。

    from dataclasses import dataclass
    from datetime import datetime
    from typing import Literal

    Tier = Literal["enforce", "warn_or_confirm", "shadow_only", "re_evaluate_or_retire"]
    ProbeKind = Literal["readonly_replay", "metadata_probe", "dry_run_tool", "sandbox_probe", "live_canary"]

    @dataclass
    class LearningRule:
        rule_id: str
        version: int
        confidence: float
        action_tier: Tier
        next_revalidate_at: datetime
        dependency_drift: bool
        evidence_age_days: int
        risk_surface: Literal["security", "side_effect", "preference"]

    @dataclass
    class RevalidationPlan:
        rule_id: str
        version: int
        reason: str
        allowed_probes: list[ProbeKind]
        required_evidence: list[str]

    def needs_revalidation(rule: LearningRule, now: datetime) -> str | None:
        if rule.dependency_drift:
            return "dependency_drift_detected"
        if rule.action_tier in ("shadow_only", "re_evaluate_or_retire"):
            return "low_confidence_tier"
        if rule.evidence_age_days > 90 and rule.risk_surface != "preference":
            return "evidence_aged_out"
        if now >= rule.next_revalidate_at:
            return "scheduled_review"
        return None

    def allowed_probes(rule: LearningRule) -> list[ProbeKind]:
        probes: list[ProbeKind] = ["readonly_replay", "metadata_probe"]
        if rule.risk_surface != "security":
            probes.append("dry_run_tool")
        if rule.risk_surface == "preference":
            probes.append("sandbox_probe")
        return probes

    def schedule_revalidation(rules: list[LearningRule], now: datetime) -> list[RevalidationPlan]:
        plans: list[RevalidationPlan] = []
        for rule in rules:
            reason = needs_revalidation(rule, now)
            if not reason:
                continue

            plans.append(RevalidationPlan(
                rule_id=rule.rule_id,
                version=rule.version,
                reason=reason,
                allowed_probes=allowed_probes(rule),
                required_evidence=[
                    "dependency_snapshot",
                    "counterfactual_replay_report",
                    "confidence_recalibration_event",
                ],
            ))
        return plans

这里最重要的是：scheduler 只生成计划，不直接执行外部动作。执行层还要再过 policy / permission gate。

## 5. pi-mono：LearningRevalidationWorker

pi-mono 里可以把再验证做成后台 worker。它不在用户请求的关键路径里跑，而是消费 due plans，执行只读 probe，把结果写回 EventStream。

    type ProbeKind = "readonly_replay" | "metadata_probe" | "dry_run_tool" | "sandbox_probe" | "live_canary";

    interface RevalidationPlan {
      planId: string;
      ruleId: string;
      version: number;
      reason: string;
      allowedProbeKinds: ProbeKind[];
      requiredEvidence: string[];
    }

    async function runRevalidationPlan(ctx, plan: RevalidationPlan) {
      await ctx.eventStream.publish({
        type: "learning.revalidation_started",
        planId: plan.planId,
        ruleId: plan.ruleId,
        version: plan.version,
        reason: plan.reason,
      });

      const evidence = [];

      if (plan.allowedProbeKinds.includes("metadata_probe")) {
        evidence.push(await ctx.probes.captureDependencySnapshot(plan.ruleId));
      }

      if (plan.allowedProbeKinds.includes("readonly_replay")) {
        evidence.push(await ctx.replay.runCounterfactualReplay({
          ruleId: plan.ruleId,
          version: plan.version,
          mode: "readonly",
        }));
      }

      if (plan.allowedProbeKinds.includes("dry_run_tool")) {
        evidence.push(await ctx.tools.validateCurrentState({
          ruleId: plan.ruleId,
          dryRun: true,
        }));
      }

      const recalibrated = await ctx.learningConfidence.calibrate(plan.ruleId, evidence);

      await ctx.eventStream.publish({
        type: "learning.revalidation_completed",
        planId: plan.planId,
        ruleId: plan.ruleId,
        version: plan.version,
        evidenceRefs: evidence.map((item) => item.ref),
        confidence: recalibrated.confidence,
        actionTier: recalibrated.actionTier,
      });

      return recalibrated;
    }

关键点：

- worker 只消费 plan，不自己决定越权 probe；
- 所有 probe 结果都变成 evidenceRefs；
- recalibration 结果通过事件流发布；
- promote 到 enforce 仍要过 promotion gate，不由 worker 直接改生产规则。

## 6. OpenClaw 实战：课程 Cron 的主题去重再验证

在 OpenClaw 课程 Cron 里，“避免重复已讲内容”依赖三份证据：

- TOOLS.md 的已讲内容；
- agent-course/README.md 的目录；
- lessons/*.md 的实际课件文件。

如果这三者不一致，去重规则的 confidence 就应该下降。第 369 课的 revalidation 可以这样做：

    memory/agent-course-revalidation.jsonl
    {"ruleId":"topic:no-repeat","probe":"metadata_probe","result":"README latest=368, lessons latest=368, TOOLS has confidence decay"}
    {"ruleId":"topic:no-repeat","probe":"readonly_replay","result":"candidate topic not found in README/TOOLS"}
    {"ruleId":"topic:no-repeat","confidence":0.93,"tier":"enforce"}

具体流程：

1. 先用 rg 检查 TOOLS.md 是否已讲过候选主题；
2. 再看 README.md 最新编号和 lessons 文件是否一致；
3. 如果发现 TOOLS 有主题但 lesson 缺失，标记 dependency drift；
4. 如果候选主题只和很早以前的旧课相似，降级为 warn，而不是直接 block；
5. 写入 revalidation event，作为本次发课的证据。

这比“脑子里记得没讲过”可靠，也比“只看一个文件”更抗漂移。

## 7. 常见坑

第一，把再验证做成生产动作。revalidation 默认应该只读、dry-run、sandbox，live canary 必须单独授权。

第二，只按 cron 时间扫。真正的触发源还包括 dependency drift、evidence tombstone、confidence 降级和 false positive 增多。

第三，再验证通过后直接 enforce。正确流程是 revalidation -> evidence quality gate -> confidence calibration -> promotion gate。

第四，不保存失败的 probe。失败本身也是证据，可能说明依赖漂移或规则已经不适合继续使用。

第五，所有规则同一验证频率。资金、安全、副作用规则要更严格；偏好类规则可以更便宜、更快退役。

## 8. 生产落地清单

- Learning Rule 增加 lastValidatedAt、nextRevalidateAt、revalidationPolicy；
- 定义 allowedProbeKinds，默认禁止 live_canary；
- confidence 降级、evidence 过期、dependency drift 都能创建 RevalidationPlan；
- probe 结果写 EvidenceStore，不直接改规则；
- 每次开始/完成/失败都写 learning.revalidation_* 事件；
- 重新 promote 到 enforce 前必须经过 quality gate 和 promotion gate；
- 后台 worker 有预算限制，避免低价值旧规则耗尽资源；
- 对高风险规则保留最近一次 replay proof；
- 对长期无法再验证的规则自动进入 retire / archive 流程。

核心思想：**学习不是写入一次就结束。成熟 Agent 要能发现旧经验变弱，并用安全、可审计、低副作用的方式主动重新验证它。**
