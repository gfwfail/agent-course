# 377. Agent 学习规则紧急冻结与 Kill Switch（Learning Rule Emergency Freeze & Kill Switch）

上一课讲了 Re-admission Gate：被 quarantine 或 rollback 的学习规则不能修完就直接 active，要用 candidate version、修复证据、回归重放和 shadow/canary 逐步重新接纳。

今天补一个更偏生产事故的能力：**当你还没搞清楚是哪条学习规则在误伤生产时，系统必须能先冻结学习规则的影响面。**

这不是普通的 rollback。Rollback 通常已经知道 bad version 和 last known good。Emergency Freeze 处理的是更混乱的现场：

- 多条学习规则同时影响一个决策；
- prompt pack、policy cache、retrieval index 可能都已经吃到了规则；
- 误伤还在继续发生，但根因没查完；
- 有些动作已经进入队列，不能只改内存状态；
- 需要保留证据，后面才能做 forensics、repair、readmission。

一句话：**Kill Switch 不是让 Agent 失忆，而是让可疑学习暂时失去生产影响力。**

## 1. 为什么学习系统需要单独的 Kill Switch

普通 feature flag 可以关闭一个功能，但学习规则更麻烦。它通常不是一个明确按钮，而是散落在多个运行路径里：

- retrieval：规则被检索出来；
- prompt injection：规则被塞进 system/user context；
- policy evaluation：规则改变 allow/deny/confirm；
- tool dispatch：规则改变工具参数或执行顺序；
- sub-agent delegation：规则改变任务分配；
- memory write：规则继续影响新的学习候选。

所以学习规则的 Kill Switch 不能只做：

    learning.enabled = false

它至少要回答四个问题：

1. 冻结哪些规则：全部、某个 tenant、某类 channel、某些 risk tier、某个 rule family？
2. 冻结到什么程度：不检索、不注入、只 shadow、还是只禁止 side effect？
3. 已经排队的动作怎么办：继续、降级 dry-run、阻断、重新评估？
4. 谁在什么时候为什么冻结：必须留下 Freeze Manifest 和审计事件。

## 2. Freeze Manifest：冻结也要可审计

不要用临时环境变量表达生产冻结。更稳的是追加一份不可变 Freeze Manifest：

    {
      "freezeId": "lfz_20260522_agent_course_dedup",
      "createdAt": "2026-05-22T02:30:00Z",
      "createdBy": "runtime_drift_monitor",
      "reason": "suspected_false_positive_spike",
      "scope": {
        "tenants": ["internal"],
        "channels": ["cron"],
        "ruleFamilies": ["learning.topic_dedup", "learning.push_gate"],
        "riskTiers": ["medium", "high"]
      },
      "mode": "shadow_only",
      "queuePolicy": "revalidate_before_execute",
      "expiresAt": "2026-05-22T06:30:00Z",
      "evidenceRefs": [
        "drift_alert_8bc",
        "false_positive_sample_21",
        "operator_status_12414"
      ]
    }

几个字段很关键：

- mode 不是布尔值，而是冻结强度；
- expiresAt 防止临时冻结变成永久遗留；
- queuePolicy 处理已经排队的任务；
- evidenceRefs 让后续 rollback forensics 能追溯为什么冻结。

常见 freeze mode：

    observe_only        # 继续记录规则命中，但完全不影响决策
    shadow_only         # 计算规则输出，只写 diff，不注入生产路径
    no_side_effect      # 允许影响只读分析，禁止影响外部副作用
    block_matching      # 命中 scope 的动作直接阻断或人工复核

## 3. learn-claude-code：最小冻结判定器

教学版先写纯函数：输入规则、动作和 freeze manifest，输出运行时应该怎么降级。

    from dataclasses import dataclass
    from typing import Literal

    FreezeMode = Literal["observe_only", "shadow_only", "no_side_effect", "block_matching"]
    RuntimeAction = Literal["apply", "observe", "shadow", "dry_run", "block"]

    @dataclass(frozen=True)
    class FreezeManifest:
        freeze_id: str
        mode: FreezeMode
        tenants: set[str]
        channels: set[str]
        rule_families: set[str]
        risk_tiers: set[str]
        expires_at_epoch: int

    @dataclass(frozen=True)
    class RuleContext:
        tenant: str
        channel: str
        rule_family: str
        risk_tier: str
        now_epoch: int
        action_has_side_effect: bool

    def freeze_matches(freeze: FreezeManifest, ctx: RuleContext) -> bool:
        if ctx.now_epoch >= freeze.expires_at_epoch:
            return False
        return (
            ctx.tenant in freeze.tenants
            and ctx.channel in freeze.channels
            and ctx.rule_family in freeze.rule_families
            and ctx.risk_tier in freeze.risk_tiers
        )

    def decide_runtime_action(freeze: FreezeManifest | None, ctx: RuleContext) -> RuntimeAction:
        if freeze is None or not freeze_matches(freeze, ctx):
            return "apply"

        if freeze.mode == "observe_only":
            return "observe"

        if freeze.mode == "shadow_only":
            return "shadow"

        if freeze.mode == "no_side_effect":
            return "dry_run" if ctx.action_has_side_effect else "apply"

        return "block"

这个函数看起来简单，但它把关键边界固定住了：

- freeze 过期自动失效；
- scope 精确匹配，避免全局误伤；
- side effect 单独处理；
- LLM 不能绕过冻结决定。

## 4. pi-mono：在 Prompt Pack 和 Tool Dispatch 前都要检查

生产版不能只在一个位置检查。学习规则可能通过 prompt 注入影响模型，也可能通过 policy/tool middleware 影响执行，所以至少要两层防线。

第一层：Prompt Pack 组装前过滤。

    export class LearningFreezePromptFilter {
      constructor(private readonly freezeStore: FreezeStore) {}

      async filterRules(input: {
        tenant: string;
        channel: string;
        candidateRules: LearningRule[];
      }): Promise<{ injected: LearningRule[]; shadowed: LearningRule[] }> {
        const freezes = await this.freezeStore.activeFor(input.tenant, input.channel);
        const injected: LearningRule[] = [];
        const shadowed: LearningRule[] = [];

        for (const rule of input.candidateRules) {
          const decision = decideFreezeMode(freezes, {
            tenant: input.tenant,
            channel: input.channel,
            ruleFamily: rule.family,
            riskTier: rule.riskTier,
            sideEffect: false,
          });

          if (decision === "apply") {
            injected.push(rule);
          } else {
            shadowed.push(rule);
          }
        }

        return { injected, shadowed };
      }
    }

第二层：工具执行前再次拦截。

    export class LearningFreezeToolMiddleware implements ToolMiddleware {
      constructor(private readonly freezeStore: FreezeStore) {}

      async beforeToolCall(ctx: ToolCallContext, next: NextToolCall) {
        const freezes = await this.freezeStore.activeFor(ctx.tenant, ctx.channel);
        const decision = decideFreezeMode(freezes, {
          tenant: ctx.tenant,
          channel: ctx.channel,
          ruleFamily: ctx.learningInfluence?.ruleFamily ?? "unknown",
          riskTier: ctx.riskTier,
          sideEffect: ctx.tool.sideEffect !== "none",
        });

        await ctx.audit.write({
          type: "learning.freeze.checked",
          toolName: ctx.tool.name,
          decision,
          freezeIds: freezes.map((freeze) => freeze.freezeId),
          influencedBy: ctx.learningInfluence?.ruleIds ?? [],
        });

        if (decision === "block") {
          throw new ToolBlockedError("learning_freeze_block_matching");
        }

        if (decision === "dry_run") {
          return next({ ...ctx, dryRun: true });
        }

        if (decision === "shadow" || decision === "observe") {
          return next({ ...ctx, learningInfluence: undefined });
        }

        return next(ctx);
      }
    }

重点是：冻结检查结果也要写 audit。否则事故后你只知道“好像冻结过”，但不知道哪些动作在冻结窗口内被放行、降级或阻断。

## 5. Queue Policy：别忘了已经排队的任务

学习规则误伤时，最容易漏掉的是队列里已经生成的任务。比如某条学习规则让 Agent 决定“自动 push 课程 repo”，这个动作可能已经排队，冻结内存规则并不会取消队列里的 payload。

Freeze Manifest 里的 queuePolicy 可以这样处理：

    revalidate_before_execute
      执行前重新跑 freeze + policy + evidence freshness。

    convert_to_dry_run
      副作用动作转成 dry-run，保留计划但不真正执行。

    hold_for_review
      进入人工复核队列，保留原始 intent 和 learning influence。

    cancel_matching
      取消命中 scope 的任务，并写 cancellation evidence。

推荐默认值是 revalidate_before_execute。这样不会粗暴丢任务，也不会让旧决策绕过新冻结。

## 6. OpenClaw 实战：课程 Cron 的学习规则冻结

拿这个课程 Cron 举例。假设某天发现“已讲内容去重规则”异常，开始把新主题误判为重复，导致课程无法发布。

不要先改代码硬冲。更稳的做法：

    {
      "freezeId": "lfz_agent_course_topic_dedup_20260522",
      "scope": {
        "channels": ["cron"],
        "ruleFamilies": ["learning.agent_course.topic_dedup"],
        "riskTiers": ["medium"]
      },
      "mode": "shadow_only",
      "queuePolicy": "revalidate_before_execute",
      "expiresAt": "2026-05-22T06:30:00Z"
    }

冻结后，课程 Cron 仍然读取 README.md 和 TOOLS.md，但去重规则只写 shadow diff：

    primaryDecision:
      topic = "学习规则紧急冻结与 Kill Switch"
      publish = true

    shadowLearningDecision:
      ruleId = "lr_agent_course_topic_dedup"
      decision = "would_block"
      reason = "similar_to_learning_rule_rollback"

    freezeEffect:
      mode = "shadow_only"
      productionImpact = "ignored_shadow_decision"

这就给后续修复留下了非常干净的证据：规则本来会误拦，但 freeze 阻止了它影响生产。

## 7. 冻结后的退出条件

Emergency Freeze 不能无限期存在。退出时至少要满足：

- freeze window 内所有 side effect 都有审计；
- 命中 scope 的 queued tasks 已 revalidate 或处理完成；
- drift / false positive 样本已生成 rollback case 或 repair ticket；
- 如果要恢复规则，必须走上一课的 Re-admission Gate；
- Freeze Manifest 归档为 incident evidence，不直接删除。

解除冻结也应该是一个事件：

    {
      "type": "learning.freeze.released",
      "freezeId": "lfz_agent_course_topic_dedup_20260522",
      "releasedBy": "readmission_gate",
      "releaseReason": "candidate_v5_canary_started",
      "nextControl": "canary_readmission",
      "evidenceRefs": ["readmission_decision_77", "regression_replay_41"]
    }

注意：releaseReason 不能写“问题应该好了”。要绑定具体 gate 和 evidence。

## 8. 实战 checklist

设计学习规则 Kill Switch 时，检查这些点：

- Freeze Manifest 是追加写、可审计、带过期时间；
- 支持按 tenant/channel/rule family/risk tier 精确冻结；
- Prompt Pack 注入前会过滤或 shadow；
- Tool Dispatch 前会再次执行冻结检查；
- side effect 默认可降级 dry-run 或 block；
- 队列任务执行前必须重新验证冻结状态；
- freeze/release 都写 audit event；
- 解除冻结必须走 repair / readmission / canary 证据链。

## 9. 总结

成熟 Agent 的学习系统，不只要会学习、回滚、修复和重新接纳，还要有事故现场的刹车能力。

**Emergency Freeze & Kill Switch 的核心价值是：当你还不知道哪条学习规则坏了，先让可疑学习停止影响生产，同时保留足够证据，让后续修复不是靠猜。**

下一次设计长期记忆、学习规则或自动化策略时，别只问“怎么启用它”，也要问一句：**它误伤时，我能不能在 30 秒内冻结它的生产影响？**
