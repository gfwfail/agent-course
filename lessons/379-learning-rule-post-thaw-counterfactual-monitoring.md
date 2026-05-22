# 379. Agent 学习规则解冻后的反事实监控与影子回放（Post-Thaw Counterfactual Monitoring & Shadow Replay）

上一课讲了 Thaw Gate 和 Recovery Window：学习规则被冻结以后，不能一把删掉 Freeze Manifest，而要带着根因、修复、回归、队列重验和恢复预算，分阶段从 shadow 到 canary 再到 full thaw。

今天继续补恢复后的下一层：**解冻成功不等于事故彻底结束，系统还要继续问一句：如果旧规则还在，会不会又错？如果新规则接管，有没有漏拦？**

这就是 Post-Thaw Counterfactual Monitoring。

一句话：**主路径执行新规则，影子路径持续重放旧规则、冻结规则和候选规则，用反事实差异证明恢复真的稳定。**

## 1. 为什么解冻后还要反事实监控

很多团队在恢复窗口结束后会做两件事：

1. 把规则切回 active；
2. 关闭事故单。

这不够。因为学习规则事故通常不是单点错误，而是上下文、证据、prompt pack、policy cache、retrieval index、队列任务共同作用后的行为偏差。

解冻后至少有三类风险：

- 旧坏规则在某些长尾输入上仍然会给出危险建议；
- 新修复规则过度收紧，把正常任务误判成风险；
- 冻结期间积压或重新验证过的任务，在真实流量下出现新组合。

所以恢复后要保留一段反事实监控：

    primary = thawed_rule_v8
    shadow_old = frozen_rule_v7
    shadow_candidate = repaired_rule_v8_shadow_config

真实动作只由 primary 决定；shadow 路径只计算“如果我当时用另一套规则，会做什么”，并记录差异。

## 2. 反事实监控看什么

不要只记录 shadow 命中率。真正有用的是差异分类：

    same_decision
      primary 和 shadow 都 allow / warn / block，说明行为一致。

    old_would_fail
      旧规则会产生已知坏决策，primary 避免了它。这是修复有效的正证据。

    primary_too_loose
      primary allow，但 shadow block，且后续结果显示 shadow 更合理。这可能是漏拦。

    primary_too_strict
      primary block / warn，但 shadow allow，且后续结果正常。这可能是误伤。

    unknown_diff
      两边不同，但缺少 outcome 或证据不足，不能立刻下结论。

这类监控比“错误率下降了”更细，因为它能回答恢复期最关键的问题：**新规则到底修掉了什么，又新引入了什么。**

一个最小事件长这样：

    {
      "type": "learning.counterfactual_diff.recorded",
      "ruleId": "learning.agent_course.topic_dedup",
      "primaryVersion": "v8",
      "shadowVersions": ["v7", "v8-shadow"],
      "inputHash": "sha256:8b7c...",
      "primaryDecision": "allow",
      "shadowDecision": "block",
      "diffType": "primary_too_loose",
      "outcomeRef": "course_run_379_post_check",
      "recoveryWindowId": "rw_thaw_lfz_agent_course_topic_dedup_20260522"
    }

注意 inputHash 和 outcomeRef 就够了，不要把用户原文或敏感证据直接塞进长期监控事件。

## 3. learn-claude-code：最小反事实分类器

教学版可以先写一个纯函数：输入 primary / shadow / outcome，输出 diff type。

    from dataclasses import dataclass
    from typing import Literal

    Decision = Literal["allow", "warn", "block"]
    Outcome = Literal["success", "false_positive", "missed_risk", "unknown"]
    DiffType = Literal[
        "same_decision",
        "old_would_fail",
        "primary_too_loose",
        "primary_too_strict",
        "unknown_diff",
    ]

    RANK = {"allow": 0, "warn": 1, "block": 2}

    @dataclass(frozen=True)
    class CounterfactualInput:
        primary_decision: Decision
        shadow_decision: Decision
        shadow_is_known_bad_version: bool
        outcome: Outcome

    def classify_counterfactual_diff(case: CounterfactualInput) -> DiffType:
        if case.primary_decision == case.shadow_decision:
            return "same_decision"

        if case.shadow_is_known_bad_version and case.outcome == "success":
            return "old_would_fail"

        primary_rank = RANK[case.primary_decision]
        shadow_rank = RANK[case.shadow_decision]

        if primary_rank < shadow_rank and case.outcome == "missed_risk":
            return "primary_too_loose"

        if primary_rank > shadow_rank and case.outcome == "false_positive":
            return "primary_too_strict"

        return "unknown_diff"

这个函数故意很小，但它把一个重要原则写死了：**没有 outcome 的 diff 不能直接当事故，只能进入 unknown_diff。**

否则 shadow 和 primary 一不同就告警，团队很快会被噪音淹没。

## 4. 用任务系统保存反事实回放状态

learn-claude-code 里的 `s07_task_system.py` 已经展示了一个很实用的模式：任务状态写到 `.tasks/task_*.json`，这样上下文压缩或进程重启后还在。

反事实监控也应该这么做，不要只靠当前 turn 的内存。

可以把每个恢复窗口建成一个持久任务：

    {
      "id": 42,
      "subject": "post-thaw counterfactual monitoring",
      "status": "in_progress",
      "ruleId": "learning.agent_course.topic_dedup",
      "primaryVersion": "v8",
      "shadowVersions": ["v7"],
      "budgets": {
        "maxPrimaryTooLoose": 0,
        "maxPrimaryTooStrictRate": 0.02,
        "maxUnknownDiffRate": 0.1
      }
    }

这和教学代码里的 TaskManager 思路一致：外部化状态，任务可恢复，依赖关系可审计。

## 5. pi-mono：用 Agent.subscribe 旁路记录 shadow diff

pi-mono 的 `packages/agent/src/agent.ts` 里，Agent 暴露了 `subscribe(fn)`，内部每个 AgentEvent 都会推给 listener。这个结构很适合做旁路监控：不改变主 loop，只监听事件、生成影子评估任务和 diff 事件。

生产版可以这么挂：

    agent.subscribe(async (event) => {
      if (event.type !== "turn_end") return;

      const primary = extractLearningDecision(event);
      if (!primary) return;

      const shadow = await shadowRuleEngine.evaluate({
        ruleId: primary.ruleId,
        versions: ["v7", "v8-shadow"],
        inputRef: primary.inputRef,
        evidenceRefs: primary.evidenceRefs,
      });

      const diff = classifyCounterfactualDiff({
        primaryDecision: primary.decision,
        shadowDecision: shadow.decision,
        shadowIsKnownBadVersion: shadow.version === "v7",
        outcome: await outcomeStore.lookup(primary.runId),
      });

      await eventBus.publish({
        type: "learning.counterfactual_diff.recorded",
        runId: primary.runId,
        ruleId: primary.ruleId,
        primaryVersion: primary.version,
        shadowVersion: shadow.version,
        diffType: diff,
      });
    });

重点是：subscribe 做的是旁路观察，不应该让 shadow 结果直接影响当前真实动作。否则“监控”会变成另一套未发布策略，恢复期反而更危险。

## 6. 预算闸门：什么时候继续观察，什么时候回冻

反事实监控最终要接一个预算闸门。否则数据只进不出，没有运维意义。

一个恢复后的预算可以这样定义：

    {
      "windowMinutes": 180,
      "minSamples": 20,
      "maxPrimaryTooLoose": 0,
      "maxPrimaryTooStrictRate": 0.02,
      "maxUnknownDiffRate": 0.1,
      "autoRefreezeOn": ["primary_too_loose", "unknown_diff_spike"]
    }

判定逻辑：

    def evaluate_post_thaw_budget(metrics):
        if metrics.samples < metrics.min_samples:
            return "continue_observing"

        if metrics.primary_too_loose > metrics.max_primary_too_loose:
            return "auto_refreeze"

        if metrics.primary_too_strict_rate > metrics.max_primary_too_strict_rate:
            return "manual_review"

        if metrics.unknown_diff_rate > metrics.max_unknown_diff_rate:
            return "extend_shadow_replay"

        return "close_recovery_window"

这比单纯按时间关闭恢复窗口更稳：时间只是观察长度，样本和差异预算才是恢复质量。

## 7. OpenClaw 实战：课程 Cron 怎么用

拿本课程 Cron 举例。假设 `topic_dedup` 学习规则刚从冻结状态恢复：

1. primary 使用修复后的去重规则，决定今天讲什么；
2. shadow_old 用冻结前的旧规则也跑一遍，但不影响发课；
3. shadow_candidate 用更严格的新规则再跑一遍；
4. 课程发布后，把 README、TOOLS、Telegram messageId、git commit 作为 outcome 证据；
5. 如果 primary 选题成功且 shadow_old 会误判，就记录 old_would_fail；
6. 如果 primary 选题和历史重复，立刻 auto_refreeze 并阻断下一次 cron 的副作用。

伪流程：

    primary_topic = choose_topic(rule_version="v8")
    shadow_old = choose_topic(rule_version="v7", mode="shadow")
    shadow_strict = choose_topic(rule_version="v8-strict", mode="shadow")

    publish_lesson(primary_topic)
    outcome = verify_lesson_published(readme=True, tools=True, telegram=True, git=True)

    record_counterfactual(primary_topic, shadow_old, outcome)
    record_counterfactual(primary_topic, shadow_strict, outcome)

    decision = post_thaw_budget_gate(metrics_for("topic_dedup"))
    if decision == "auto_refreeze":
        create_freeze_manifest(rule_id="learning.agent_course.topic_dedup")

这个流程的好处是，恢复不是凭感觉结束，而是由真实发布结果、历史去重结果和影子差异共同证明。

## 8. 实战踩坑

第一，shadow replay 必须只读。

它可以计算“如果当时会怎么决策”，但不能发消息、写 memory、提交代码或触发外部 API。

第二，diff 事件要最小披露。

记录 inputHash、ruleVersion、decision、outcomeRef 就够了；raw prompt、用户隐私、secret 不要进入长期监控指标。

第三，不要让 known_bad shadow 永远跟跑。

旧规则只需要在恢复窗口内证明“它确实会失败”。窗口关闭后归档到 regression pack 即可，避免长期消耗成本。

第四，unknown_diff 不要自动升级成事故。

unknown 的意思是证据不足。先补 outcome、扩样本、延长 shadow replay；只有和 missed_risk 或 false_positive 绑定后，才进入回冻或人工复核。

## 9. 小结

今天这课的核心：

- 解冻后还要做反事实监控，不要恢复完就关事故单；
- primary 负责真实动作，shadow 负责只读回放旧规则和候选规则；
- diff 要结合 outcome 分类，不能看到差异就告警；
- 用预算闸门决定 close、extend、manual_review 或 auto_refreeze；
- OpenClaw 课程 Cron 可以用 README/TOOLS/Telegram/Git 证据作为 outcome，验证去重规则恢复是否真的稳定。

成熟 Agent 的恢复，不是“开关已经打开”，而是能持续证明：旧错误不会回来，新修复没有制造更隐蔽的新错误。
