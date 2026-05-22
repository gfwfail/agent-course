# Agent 学习规则解冻后的关闭收据与最终定版

> Post-Thaw Closeout & Finalization Receipt：反事实监控通过以后，恢复流程还不能靠一句“看起来正常”结束。要生成关闭收据，解除临时保护，归档 shadow 版本，恢复正常发布路径，并留下下一次事故能重放的证据。

上一课讲的是解冻后的反事实监控：primary 跑修复规则，shadow 跑旧规则和候选规则，用 outcome 判断新修复有没有放松或过严。

但生产系统里还有最后一步经常被漏掉：**监控通过以后，怎么正式关闭恢复窗口？**

如果没有 closeout：

- shadow 任务可能一直跑，持续烧成本；
- freeze / thaw 的临时预算可能一直影响正常策略；
- 旧规则版本没有归档，之后查事故时不知道哪版才是最终可信版本；
- 团队以为恢复结束了，但 prompt pack、policy cache、retrieval index 里可能还残留临时状态。

成熟 Agent 的恢复流程，最后必须落一张关闭收据。

## 1. 关闭不是停止监控，而是状态转换

恢复窗口关闭前，至少要确认四件事：

1. 反事实监控样本数足够；
2. 没有触发 auto_refreeze / manual_review；
3. 临时 guardrail 可以撤回或降级；
4. 最终生效版本、影子版本、回归包、证据包都有归档引用。

可以把状态机写成这样：

    frozen
      -> thaw_requested
      -> thaw_canary
      -> post_thaw_monitoring
      -> closeout_pending
      -> finalized

`finalized` 不是“没有状态”，而是一个明确状态：表示这条规则已经从恢复流程回到正常生命周期。

## 2. Closeout Receipt 应该记录什么

最小可用的关闭收据：

    {
      "receiptId": "closeout_20260522_topic_dedup_v8",
      "ruleId": "learning.agent_course.topic_dedup",
      "finalVersion": "v8",
      "closedAt": "2026-05-22T11:30:00Z",
      "recoveryCaseId": "recovery_20260521_001",
      "monitoringWindow": {
        "startedAt": "2026-05-22T08:30:00Z",
        "endedAt": "2026-05-22T11:30:00Z",
        "samples": 42
      },
      "counterfactualMetrics": {
        "oldWouldFail": 9,
        "primaryTooLoose": 0,
        "primaryTooStrictRate": 0.0,
        "unknownDiffRate": 0.04
      },
      "actions": [
        "archive_shadow_versions",
        "remove_temporary_thaw_budget",
        "restore_normal_release_policy",
        "pin_final_rule_version",
        "append_regression_pack"
      ],
      "evidenceRefs": [
        "activation_receipt:v8",
        "counterfactual_window:20260522",
        "regression_pack:topic_dedup_post_recovery",
        "git_commit:abc123"
      ]
    }

核心原则：**关闭收据不保存所有 raw 数据，但要能指向足够证据，证明为什么可以关。**

## 3. learn-claude-code：用文件式任务做关闭闸门

learn-claude-code 的 `agents/s07_task_system.py` 演示了一个很朴素但实用的模式：任务写成 `.tasks/task_*.json`，状态变化可恢复。

恢复 closeout 也可以照这个思路做成纯函数：

    def can_close_recovery_window(task):
        metrics = task["counterfactualMetrics"]
        budget = task["closeoutBudget"]

        if metrics["samples"] < budget["minSamples"]:
            return "continue_monitoring"

        if metrics["primaryTooLoose"] > budget["maxPrimaryTooLoose"]:
            return "auto_refreeze"

        if metrics["primaryTooStrictRate"] > budget["maxPrimaryTooStrictRate"]:
            return "manual_review"

        if metrics["unknownDiffRate"] > budget["maxUnknownDiffRate"]:
            return "extend_shadow_replay"

        required = [
            "activationReceipt",
            "regressionPack",
            "finalRuntimeObservation",
            "rollbackTarget",
        ]
        missing = [key for key in required if not task.get(key)]
        if missing:
            return {"decision": "hold_closeout", "missing": missing}

        return "closeout_pending"

这段代码故意不执行副作用，只做判断。真正的归档、解除临时预算、写收据应该放到下一步 handler 里。

这样设计的好处是：closeout gate 可以单元测试，也可以拿历史事故 replay。

## 4. pi-mono：用 Agent.subscribe 记录最终事件

pi-mono 的 `packages/agent/src/agent.ts` 里，`Agent.subscribe(fn)` 可以旁路接收 AgentEvent。前几课我们用它记录 activation、drift、counterfactual diff。

closeout 阶段也应该用事件，而不是悄悄改内存：

    agent.subscribe(async (event) => {
      if (event.type !== "agent_end") return;

      const pending = await recoveryStore.findCloseoutPending({
        ruleId: "learning.agent_course.topic_dedup",
      });
      if (!pending) return;

      const decision = closeoutGate({
        metrics: await counterfactualStore.metrics(pending.windowId),
        requiredEvidence: await evidenceIndex.lookup(pending.evidenceRefs),
        runtime: await runtimeObserver.observe(pending.ruleId),
      });

      await eventBus.publish({
        type: "learning.recovery_closeout.evaluated",
        ruleId: pending.ruleId,
        finalVersion: pending.finalVersion,
        decision,
        recoveryCaseId: pending.recoveryCaseId,
      });
    });

注意：这里仍然是旁路观察。真正的 finalize worker 监听 `learning.recovery_closeout.evaluated`，并且只对 `decision == "closeout_pending"` 执行最终定版。

## 5. Finalize Worker：幂等地收尾

最终定版动作必须幂等。因为 cron、worker 重试、网络抖动都会导致同一个 closeout 被执行多次。

一个生产版 worker 可以这样拆：

    async function finalizeRecoveryCloseout(command) {
      const lock = await locks.acquire(command.receiptId);
      if (!lock.ok) return;

      const existing = await receiptStore.get(command.receiptId);
      if (existing) return existing;

      await shadowStore.archive({
        ruleId: command.ruleId,
        versions: command.shadowVersions,
        reason: "post_thaw_closeout",
      });

      await thawBudgetStore.removeTemporaryBudget({
        ruleId: command.ruleId,
        recoveryCaseId: command.recoveryCaseId,
      });

      await releasePolicyStore.restoreNormalPolicy({
        ruleId: command.ruleId,
        finalVersion: command.finalVersion,
      });

      await ruleStore.pinFinalVersion({
        ruleId: command.ruleId,
        version: command.finalVersion,
      });

      await regressionPack.append(command.regressionCases);

      return receiptStore.create({
        receiptId: command.receiptId,
        ruleId: command.ruleId,
        finalVersion: command.finalVersion,
        evidenceRefs: command.evidenceRefs,
        closedAt: new Date().toISOString(),
      });
    }

这个 worker 的重点不是复杂，而是顺序清晰：

1. 先抢锁；
2. 先查收据，保证重复执行不会重复归档；
3. 再解除临时状态；
4. 最后写不可变 receipt。

## 6. OpenClaw 实战：课程 Cron 的 closeout

拿本课程 cron 举例。如果 `topic_dedup` 规则经历过 freeze -> thaw -> shadow replay，最终 closeout 不能只看 Telegram 发出去了。

应该检查：

- `lessons/380-*.md` 已存在；
- `README.md` 目录已更新；
- `TOOLS.md` 已讲内容已更新；
- Telegram 群消息有 messageId；
- git commit 已 push 到远端；
- 下一次选题不会再命中刚讲过的主题。

关闭收据可以像这样：

    {
      "ruleId": "learning.agent_course.topic_dedup",
      "finalVersion": "v8",
      "closeoutChecks": {
        "lessonFile": true,
        "readmeIndex": true,
        "toolsMemory": true,
        "telegramMessageId": 12546,
        "gitRemoteContainsCommit": true,
        "nextTopicReplayPasses": true
      },
      "decision": "finalized"
    }

这就是 OpenClaw 里最实用的工程习惯：每个外部副作用结束后，都要有文件、消息、Git、记忆四类证据互相印证。

## 7. 常见坑

**坑 1：监控窗口过了就自动关闭。**

时间到了不等于样本够了。低流量规则可能 3 小时只有 2 个样本，关闭只是在自欺欺人。

**坑 2：关闭时直接删除 shadow 数据。**

shadow 数据应该 archive，不应该 delete。下一次事故复盘时，它是判断“旧规则会不会复发”的关键证据。

**坑 3：忘了恢复正常发布策略。**

thaw 期间经常会有更严格的临时预算。如果 closeout 后还留着，系统会持续过度保守，表现成莫名其妙的 manual_review 增多。

**坑 4：收据没有 runtime observation。**

写了 receipt 但没观测运行时组件，可能只是数据库里 finalVersion 对了，prompt pack / cache / retrieval index 仍然没刷新。

## 8. 设计检查清单

做 post-thaw closeout 时，至少问自己：

- 是否有最小样本数和差异预算？
- 是否归档了 shadow 版本，而不是删除？
- 是否解除或降级了临时 thaw guardrail？
- 是否 pin 了最终规则版本？
- 是否把本次事故加入 regression pack？
- 是否有 runtime observation 证明最终版本真的在运行？
- closeout worker 是否幂等？
- receipt 是否足够让审计者复盘“为什么可以关闭”？

## 总结

恢复流程的最后一步不是“没报警，所以结束”，而是：

1. 用预算证明可以结束；
2. 用 worker 幂等地解除临时状态；
3. 用 closeout receipt 证明最终版本、证据、运行时和归档都一致；
4. 把本次事故变成下一次发布前会自动 replay 的 regression pack。

成熟 Agent 的恢复闭环，不只会发现坏规则、修好坏规则，还会带证据地宣布：这次恢复已经正式结束。
