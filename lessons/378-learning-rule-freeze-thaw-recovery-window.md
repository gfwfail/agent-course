# 378. Agent 学习规则冻结后的解冻闸门与恢复窗口（Learning Rule Freeze Thaw & Recovery Window）

上一课讲了 Emergency Freeze / Kill Switch：事故现场还没定位坏规则时，先用 Freeze Manifest 把可疑学习规则从生产影响路径里拿掉，让它只能 shadow、dry-run 或直接 block。

今天继续补下一层：**冻结只是踩刹车，不能把系统永远停在路边。**

学习规则被冻结以后，团队很容易犯两个错误：

- 一直不解冻，导致系统长期失去学习能力；
- 根因刚修完就全量解冻，结果同类误伤立刻复发。

一句话：**Thaw Gate 负责判断什么时候可以解冻，Recovery Window 负责限制解冻后的观察期和影响面。**

## 1. 为什么不能直接删 Freeze Manifest

很多系统会把冻结做成一个临时开关：

    learning.freeze = true

事故缓解后再改回：

    learning.freeze = false

这在 Agent 学习系统里不够。因为 Freeze Manifest 影响过 prompt pack、policy cache、retrieval index、queue task 和 tool dispatch。解冻不是把布尔值改回去，而是要证明四件事：

1. 根因已经定位或至少被隔离；
2. 修复证据足够支撑恢复部分影响面；
3. 队列里的旧任务已经重新评估；
4. 解冻后有明确观察窗口、失败预算和自动回冻条件。

所以生产系统需要一个独立的 Thaw Request：

    {
      "thawId": "thaw_lfz_agent_course_topic_dedup_20260522",
      "freezeId": "lfz_agent_course_topic_dedup_20260522",
      "requestedBy": "recovery_orchestrator",
      "requestedScope": {
        "channels": ["cron"],
        "ruleFamilies": ["learning.agent_course.topic_dedup"],
        "riskTiers": ["medium"]
      },
      "targetMode": "canary_apply",
      "evidenceRefs": [
        "root_cause_case_91",
        "regression_replay_pack_42",
        "post_fix_probe_17"
      ],
      "recoveryWindow": {
        "durationMinutes": 120,
        "trafficShare": 0.1,
        "maxFalsePositiveRate": 0.01,
        "autoRefreezeOn": ["policy_drift", "false_positive_spike"]
      }
    }

注意 targetMode 不是直接 active，而是 canary_apply。解冻本身也要分阶段。

## 2. Thaw Gate 的输入

一个实用的解冻闸门至少检查这些输入：

- Freeze Manifest：冻结范围、冻结模式、过期时间、原始证据。
- Root Cause Case：根因分类、影响窗口、是否还有 unknown。
- Repair Evidence：代码修复、配置修复、规则修复或作用域收窄证据。
- Regression Replay：冻结事故对应的回归包是否通过。
- Queue Revalidation：冻结窗口内排队任务是否重新跑过 policy/evidence/freeze 检查。
- Recovery Budget：解冻后的流量、失败率、观察时间和自动回冻条件。

输出也不要只有 allow/deny。更稳的是：

    deny
    hold_for_more_evidence
    shadow_only
    canary_apply
    full_thaw

这样系统可以在证据不足时先 shadow，而不是卡死或冒险全开。

## 3. learn-claude-code：最小解冻判定器

教学版可以先写纯函数：输入 thaw request 和证据摘要，输出解冻阶段。

    from dataclasses import dataclass
    from typing import Literal

    ThawDecision = Literal[
        "deny",
        "hold_for_more_evidence",
        "shadow_only",
        "canary_apply",
        "full_thaw",
    ]

    @dataclass(frozen=True)
    class ThawEvidence:
        root_cause_known: bool
        repair_applied: bool
        regression_passed: bool
        queue_revalidated: bool
        unknown_rate: float
        false_positive_rate: float
        clean_minutes: int

    @dataclass(frozen=True)
    class RecoveryBudget:
        min_clean_minutes: int
        max_unknown_rate: float
        max_false_positive_rate: float
        allow_full_thaw_after_minutes: int

    def decide_thaw(evidence: ThawEvidence, budget: RecoveryBudget) -> ThawDecision:
        if not evidence.repair_applied:
            return "deny"

        if not evidence.root_cause_known:
            return "shadow_only"

        if not evidence.regression_passed or not evidence.queue_revalidated:
            return "hold_for_more_evidence"

        if evidence.unknown_rate > budget.max_unknown_rate:
            return "hold_for_more_evidence"

        if evidence.false_positive_rate > budget.max_false_positive_rate:
            return "shadow_only"

        if evidence.clean_minutes < budget.min_clean_minutes:
            return "canary_apply"

        if evidence.clean_minutes >= budget.allow_full_thaw_after_minutes:
            return "full_thaw"

        return "canary_apply"

这个函数的重点不是复杂，而是把解冻条件显式化：

- 没修复，拒绝；
- 根因未知，只能 shadow；
- 回归或队列未验证，继续 hold；
- 指标刚刚变好，只能 canary；
- 观察窗口足够干净，才允许 full thaw。

## 4. pi-mono：Thaw Orchestrator 不直接改运行态

生产版不要让某个 handler 直接删除 freeze。更稳的是用 Thaw Orchestrator 生成新的状态事件：

    type ThawDecision =
      | "deny"
      | "hold_for_more_evidence"
      | "shadow_only"
      | "canary_apply"
      | "full_thaw";

    interface ThawRequest {
      thawId: string;
      freezeId: string;
      requestedScope: LearningRuleScope;
      targetMode: "shadow_only" | "canary_apply" | "full_thaw";
      recoveryWindow: {
        durationMinutes: number;
        trafficShare: number;
        maxFalsePositiveRate: number;
        autoRefreezeOn: string[];
      };
      evidenceRefs: string[];
    }

    export class LearningThawOrchestrator {
      constructor(
        private readonly freezeStore: FreezeStore,
        private readonly evidenceStore: EvidenceStore,
        private readonly replayGate: RegressionReplayGate,
        private readonly queueGate: QueueRevalidationGate,
        private readonly eventBus: EventBus,
      ) {}

      async evaluate(request: ThawRequest): Promise<ThawDecision> {
        const freeze = await this.freezeStore.getActive(request.freezeId);
        if (!freeze) return "deny";

        const evidence = await this.evidenceStore.summarize(request.evidenceRefs);
        const replay = await this.replayGate.check(request.requestedScope);
        const queue = await this.queueGate.check(request.freezeId);

        const decision = decideThaw({
          rootCauseKnown: evidence.rootCauseKnown,
          repairApplied: evidence.repairApplied,
          regressionPassed: replay.passed,
          queueRevalidated: queue.passed,
          unknownRate: evidence.unknownRate,
          falsePositiveRate: evidence.falsePositiveRate,
          cleanMinutes: evidence.cleanMinutes,
        }, request.recoveryWindow);

        await this.eventBus.publish({
          type: "learning.thaw.decided",
          thawId: request.thawId,
          freezeId: request.freezeId,
          decision,
          evidenceRefs: request.evidenceRefs,
        });

        return decision;
      }
    }

关键点：Thaw Orchestrator 只产出 decision 和 event。真正改变 prompt pack、policy cache、retrieval index 的动作，应该由后续 Activation Worker 幂等执行，并生成 Thaw Receipt。

## 5. Recovery Window：解冻后的观察期

解冻通过不代表事故结束。它只是进入 Recovery Window。

Recovery Window 要约束三件事：

    scope
      只恢复 freeze scope 的一个子集，不能比冻结范围更宽。

    traffic
      先 5% 或 10% canary，按稳定窗口逐步提升。

    budget
      false positive、unknown、policy drift、queue retry failure 超预算就自动 refreeze。

一个最小的恢复窗口事件：

    {
      "type": "learning.recovery_window.started",
      "freezeId": "lfz_agent_course_topic_dedup_20260522",
      "thawId": "thaw_lfz_agent_course_topic_dedup_20260522",
      "scope": {
        "channels": ["cron"],
        "ruleFamilies": ["learning.agent_course.topic_dedup"]
      },
      "trafficShare": 0.1,
      "startedAt": "2026-05-22T05:30:00Z",
      "endsAt": "2026-05-22T07:30:00Z",
      "autoRefreezeOn": ["false_positive_spike", "policy_drift"]
    }

这能防止“修好了，全部放开”这种过于乐观的恢复方式。

## 6. OpenClaw 实战：课程 Cron 的解冻

还是拿课程 Cron 举例。假设上一课的 topic_dedup 规则被冻结成 shadow_only。修复后不要直接让它重新 enforce，而是这样走：

1. 生成 Thaw Request，绑定 freezeId、修复 commit、回归 replay 和本次课程 dry-run 证据。
2. 对 lessons/README/TOOLS 的去重逻辑先跑 shadow，对比 primary topic 和 shadow topic。
3. 只让 1 个 cron run 使用 canary_apply，其余继续 shadow。
4. 如果本次发课成功、没有误判重复、Git push 和 Telegram message 都有证据，再扩大恢复范围。

伪流程：

    freeze = lfz_agent_course_topic_dedup_20260522
    thaw = create_thaw_request(freeze, evidence=[commit, replay, dry_run])
    decision = thaw_gate.evaluate(thaw)

    if decision == "canary_apply":
        run_course_cron(learning_dedup_mode="apply", traffic_share=0.1)
    elif decision == "shadow_only":
        run_course_cron(learning_dedup_mode="shadow")
    else:
        keep_freeze()

对 OpenClaw 这种 always-on agent 来说，解冻必须保守：发错一条消息、推错一个 repo、误写一条 memory，都是外部可见副作用。Recovery Window 能把恢复过程变成可观察、可回滚的生产变更。

## 7. 一个容易忽略的边界：解冻不能扩大权限

Thaw Request 的 requestedScope 必须是 Freeze Manifest scope 的子集。比如 freeze 只覆盖：

    channels = ["cron"]
    ruleFamilies = ["learning.agent_course.topic_dedup"]

那么 thaw 不能顺手恢复：

    channels = ["cron", "telegram_dm"]
    ruleFamilies = ["learning.agent_course.topic_dedup", "learning.git_push_gate"]

这应该直接拒绝。解冻是恢复被冻结的影响面，不是发布新能力。

## 8. 这节课的核心

Emergency Freeze 让系统先停损，Thaw Gate 让系统有证据地恢复，Recovery Window 让恢复过程可观察、可回滚。

成熟 Agent 的学习系统，不只是知道什么时候踩刹车，还要知道什么时候、以多大范围、带着哪些证据慢慢松开刹车。
