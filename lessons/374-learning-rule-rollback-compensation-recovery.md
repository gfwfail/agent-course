# 374. Agent 学习规则回滚补偿与恢复编排（Learning Rule Rollback Compensation & Recovery Orchestration）

上一课讲了 Runtime Drift Detector 和 Auto-Rollback：当学习规则在真实运行路径里发生版本、hash、cache 或索引漂移时，要阻断高风险副作用，并回滚到 Last Known Good Activation。

今天补回滚之后最容易漏的一层：**回滚不是把版本号改回去就完事了，还要补偿已经发生的影响，并编排恢复流程。**

真实生产系统里，学习规则漂移可能已经影响了一部分请求：

- 有些消息已经按错误 prompt pack 发出去了；
- 有些工具调用已经被错误策略 allow 或 deny；
- 有些用户偏好被旧规则污染到了长期记忆；
- 有些 regression pack 已经引用了漂移期间的坏证据；
- 有些队列任务还拿着旧 rule hash 在重试。

一句话：**Auto-Rollback 负责让系统停止继续坏，Compensation & Recovery 负责处理已经坏过的痕迹，并证明系统可以重新前进。**

## 1. 回滚后要先生成 Recovery Case

不要让 rollback worker 静默完成。每次回滚都应该创建一个结构化 Recovery Case：

    {
      "recoveryId": "rec_lr_check_pr_before_push_v7_20260522",
      "driftId": "drift_lr_check_pr_before_push_v7_20260522",
      "rollbackReceiptId": "rb_act_lr_check_pr_before_push_v7_to_v6",
      "ruleId": "lr_check_pr_before_push",
      "badActivationId": "act_lr_check_pr_before_push_v7",
      "recoveredActivationId": "act_lr_check_pr_before_push_v6",
      "affectedWindow": {
        "from": "2026-05-22T02:10:00Z",
        "to": "2026-05-22T02:31:40Z"
      },
      "status": "compensating",
      "requiredChecks": [
        "impact_scan",
        "side_effect_compensation",
        "cache_flush",
        "queue_reconciliation",
        "post_recovery_probe"
      ]
    }

这个 case 是后续所有补偿动作的主键：扫描影响、撤销副作用、刷新缓存、重放队列、追加回归样例，全部都挂到同一个 recoveryId 上。

## 2. 补偿不是所有动作都能 undo

Agent 系统里的副作用大致分三类：

- reversible：能撤销，例如错误写入的临时 cache、错误生成的候选 memory、错误排队的 job。
- compensatable：不能直接撤销，但能补偿，例如发错通知后追加更正消息、错误 deny 后重新放行工单。
- irreversible：不能撤销也不能自动补偿，例如已经执行的外部支付、已发送的公开消息、已删除的远端资源。

所以 Recovery Orchestrator 不能只有 rollback()，而要按 side effect ledger 逐条分类：

    {
      "effectId": "eff_tool_git_push_guard_42",
      "ruleId": "lr_check_pr_before_push",
      "activationId": "act_lr_check_pr_before_push_v7",
      "effectType": "tool_policy_decision",
      "operation": "deny",
      "target": "git.push",
      "reversibility": "compensatable",
      "compensation": {
        "type": "replay_decision_under_lkg",
        "requiresHumanReview": false
      }
    }

重点：**补偿计划必须来自事先记录的 side effect ledger，不能靠事后猜。**

## 3. learn-claude-code：最小补偿编排器

教学版可以先实现一个纯函数：输入 recovery case、side effects 和 last known good rule，输出补偿计划。

    from dataclasses import dataclass, asdict
    from datetime import datetime, timezone
    import json

    @dataclass(frozen=True)
    class SideEffect:
        effect_id: str
        activation_id: str
        effect_type: str
        target: str
        reversibility: str
        risk: str

    @dataclass(frozen=True)
    class CompensationAction:
        action_id: str
        effect_id: str
        action_type: str
        status: str
        requires_human_review: bool
        reason: str

    def plan_compensation(recovery_id: str, bad_activation_id: str, effects: list[SideEffect]) -> list[CompensationAction]:
        actions: list[CompensationAction] = []

        for effect in effects:
            if effect.activation_id != bad_activation_id:
                continue

            if effect.reversibility == "reversible":
                action_type = "undo_effect"
                review = effect.risk == "high"
                reason = "effect can be safely undone"
            elif effect.reversibility == "compensatable":
                action_type = "apply_compensating_action"
                review = effect.risk in {"high", "critical"}
                reason = "effect cannot be undone directly; compensate with a follow-up action"
            else:
                action_type = "manual_review"
                review = True
                reason = "irreversible effect requires human decision"

            actions.append(CompensationAction(
                action_id=f"{recovery_id}_{effect.effect_id}",
                effect_id=effect.effect_id,
                action_type=action_type,
                status="pending",
                requires_human_review=review,
                reason=reason,
            ))

        return actions

    def to_json(actions: list[CompensationAction]) -> str:
        return json.dumps({
            "createdAt": datetime.now(timezone.utc).isoformat(),
            "actions": [asdict(action) for action in actions],
        }, ensure_ascii=False, sort_keys=True)

这段代码故意不执行补偿，只生成计划。教学版先把“该做什么”算清楚，再由执行器逐条幂等 apply。

## 4. pi-mono：Recovery Orchestrator 放在事件流外层

pi-mono 里可以把恢复编排做成事件订阅器：rollback receipt 一生成，就创建 recovery case，然后扫描 drift window 内的 side effect ledger。

伪代码：

    interface RecoveryCase {
      recoveryId: string;
      driftId: string;
      rollbackReceiptId: string;
      badActivationId: string;
      recoveredActivationId: string;
      status: "impact_scanning" | "compensating" | "verifying" | "closed" | "manual_review";
      affectedWindow: { from: string; to: string };
    }

    interface CompensationAction {
      actionId: string;
      effectId: string;
      actionType: "undo_effect" | "apply_compensating_action" | "manual_review";
      idempotencyKey: string;
      requiresHumanReview: boolean;
    }

    export class LearningRuleRecoveryOrchestrator {
      constructor(private readonly store: RecoveryStore) {}

      async onRollbackReceipt(receipt: RollbackReceipt) {
        const recovery = await this.store.createRecoveryCase({
          driftId: receipt.driftId,
          rollbackReceiptId: receipt.rollbackReceiptId,
          badActivationId: receipt.fromActivationId,
          recoveredActivationId: receipt.toActivationId,
          affectedWindow: receipt.affectedWindow,
        });

        const effects = await this.store.findSideEffects({
          activationId: receipt.fromActivationId,
          from: receipt.affectedWindow.from,
          to: receipt.affectedWindow.to,
        });

        const actions = planCompensation(recovery, effects);

        for (const action of actions) {
          await this.store.enqueueCompensation({
            ...action,
            idempotencyKey: `recovery:${recovery.recoveryId}:effect:${action.effectId}`,
          });
        }
      }
    }

注意这里的 idempotencyKey：恢复任务经常会重试，不能因为 worker 重启就重复发更正消息、重复撤销 cache、重复 reopen 工单。

## 5. OpenClaw 实战：课程 cron 也需要恢复清单

拿我们这个课程 cron 举例。它每 3 小时会：

1. 选题，检查 TOOLS.md 已讲内容；
2. 写 lesson 文件；
3. 更新 README.md；
4. 发 Telegram；
5. git commit/push；
6. 回写 TOOLS.md 和 memory。

如果某条“学习规则”漂移，比如“已讲内容去重”加载了旧 TOOLS.md，导致选题重复，回滚只能阻止下一次继续重复，但已经发生的影响包括：

- Telegram 群里已经发了重复课程；
- lesson 文件已经写入；
- README 已经追加目录；
- git commit 已经推送；
- TOOLS.md 可能又追加了一条错误记录。

一个恢复清单应该像这样：

    {
      "recoveryId": "rec_agent_course_duplicate_topic_20260522",
      "actions": [
        {
          "target": "telegram_message",
          "action": "send_correction_or_followup",
          "reversibility": "compensatable"
        },
        {
          "target": "lesson_file",
          "action": "append_superseded_note_or_new_lesson",
          "reversibility": "compensatable"
        },
        {
          "target": "README.md",
          "action": "fix_catalog_entry",
          "reversibility": "reversible"
        },
        {
          "target": "TOOLS.md",
          "action": "remove_or_mark_bad_topic",
          "reversibility": "reversible"
        },
        {
          "target": "git_commit",
          "action": "forward_fix_commit",
          "reversibility": "compensatable"
        }
      ]
    }

这里不要直接 git reset public main。更稳的是 forward fix：追加一个修正提交，保留证据链。

## 6. 关闭 Recovery Case 的条件

Recovery Case 不能补偿动作跑完就关。至少要过四个检查：

- impact_scan_complete：漂移窗口内的 side effects 都被扫描过；
- compensation_applied：可撤销/可补偿项都有成功收据或人工复核记录；
- stale_work_drained：旧 activationId 的队列任务、cache、prompt pack 已清干净；
- post_recovery_probe_passed：运行时重新探针确认所有入口都在 Last Known Good 或新修复版本。

关闭收据可以长这样：

    {
      "recoveryId": "rec_lr_check_pr_before_push_v7_20260522",
      "status": "closed",
      "closedAt": "2026-05-22T03:26:00Z",
      "proof": {
        "compensationActions": 12,
        "manualReviewItems": 1,
        "staleJobsDrained": 7,
        "postRecoveryProbeHash": "sha256:probe_9ac"
      }
    }

## 7. 工程落地清单

如果你要在自己的 Agent 平台加这一层，最小闭环是：

1. 所有高风险工具调用都写 side effect ledger，记录 activationId/ruleVersion/reversibility。
2. rollback worker 生成 Rollback Receipt，不能静默回滚。
3. Recovery Orchestrator 根据 rollback receipt 创建 Recovery Case。
4. Impact Scanner 扫漂移窗口内所有 side effects、队列任务、cache key、memory candidate。
5. Compensation Worker 按 idempotencyKey 执行 undo/compensate/manual_review。
6. Post-Recovery Probe 重新证明运行时一致。
7. 把 Recovery Case 里的失败样本回灌到 regression pack，避免同类漂移再次漏检。

## 8. 今日要点

成熟 Agent 的回滚不是“坏了退回去”，而是一条完整恢复链：

> Drift detected -> rollback to Last Known Good -> create Recovery Case -> scan impact -> compensate side effects -> drain stale work -> probe runtime -> close with proof -> feed regression.

没有补偿和恢复编排的回滚，只是停止出血；有 Recovery Case 和 Compensation Ledger 的回滚，才是真正把生产系统带回可信状态。
