# 375. Agent 学习规则恢复后回归固化与防复发闸门（Post-Recovery Regression Hardening & Recurrence Gate）

上一课讲了 Learning Rule Rollback Compensation & Recovery：发现漂移后，不只是回滚到 Last Known Good，还要创建 Recovery Case，扫描影响窗口，补偿副作用，清理旧队列和缓存，最后用 probe 证明系统恢复可信。

今天继续往后补一层：**Recovery Case 关闭以后，不能只留一条“已恢复”的记录，还要把这次失败固化成回归样本，并在下一次学习规则发布前阻断同类复发。**

很多 Agent 平台会犯一个生产错误：事故恢复做得很认真，但恢复证据没有进入发布路径。结果下一次 prompt pack、policy cache、learning rule 或 retrieval index 改动时，同一类漂移又回来一次。

一句话：**Recovery 让系统回到可信状态，Regression Hardening 让同类错误以后更难再进生产。**

## 1. Recovery Case 关闭时要产出 Regression Candidate

不要等人事后写复盘才加测试。Recovery Case 关闭前，就应该从 drift、rollback、compensation 和 post-recovery probe 里自动抽取 Regression Candidate。

例子：

    {
      "candidateId": "regcand_lr_check_pr_before_push_v7_drift",
      "sourceRecoveryId": "rec_lr_check_pr_before_push_v7_20260522",
      "ruleId": "lr_check_pr_before_push",
      "failureMode": "runtime_activation_hash_drift",
      "trigger": {
        "expectedActivationId": "act_lr_check_pr_before_push_v6",
        "observedActivationId": "act_lr_check_pr_before_push_v7",
        "entrypoint": "tool_dispatch.git.push"
      },
      "expectedDecision": {
        "action": "require_pre_push_review",
        "risk": "high",
        "reasonCode": "learning_rule_lkg_required"
      },
      "badDecision": {
        "action": "allow",
        "risk": "medium",
        "reasonCode": "stale_policy_cache"
      },
      "evidenceRefs": [
        "drift_lr_check_pr_before_push_v7_20260522",
        "rb_act_lr_check_pr_before_push_v7_to_v6",
        "probe_post_recovery_9ac"
      ]
    }

这个 candidate 还不是正式测试。它只是把事故里最关键的“输入、期望、错误输出、证据来源”保存下来，等待去重、脱敏和最小化。

## 2. 回归样本要最小化，不能把事故日志整包塞进去

Regression pack 不是日志仓库。它要保留能复现决策错误的最小上下文：

- rule identity：ruleId、ruleVersion、activationId、ruleHash；
- runtime context：入口、工具、租户、权限、风险等级；
- injected context：prompt pack hash、policy cache hash、retrieval index hash；
- expected decision：正确动作和 reason code；
- forbidden regression：哪些危险变化不能再次出现；
- evidence proof：指向脱敏后的 drift / rollback / recovery receipt。

不要把完整聊天、完整工具输出、用户隐私、secret、原始邮件直接写进回归样本。保留 claim 和 hash，必要时通过 evidence lease 临时读取原始证据。

## 3. learn-claude-code：从 Recovery Case 生成回归样本

教学版可以写一个纯函数：输入 recovery case 和 bad/good decision，输出最小 regression fixture。

    from dataclasses import dataclass, asdict
    import hashlib
    import json

    @dataclass(frozen=True)
    class Decision:
        action: str
        risk: str
        reason_code: str
        activation_id: str
        policy_cache_hash: str

    @dataclass(frozen=True)
    class RegressionFixture:
        fixture_id: str
        source_recovery_id: str
        rule_id: str
        failure_mode: str
        expected: Decision
        forbidden_actions: list[str]
        evidence_refs: list[str]

    def stable_id(parts: list[str]) -> str:
        digest = hashlib.sha256("|".join(parts).encode()).hexdigest()[:12]
        return f"reg_{digest}"

    def build_fixture(
        recovery_id: str,
        rule_id: str,
        failure_mode: str,
        good: Decision,
        bad: Decision,
        evidence_refs: list[str],
    ) -> RegressionFixture:
        return RegressionFixture(
            fixture_id=stable_id([recovery_id, rule_id, failure_mode, bad.action, bad.reason_code]),
            source_recovery_id=recovery_id,
            rule_id=rule_id,
            failure_mode=failure_mode,
            expected=good,
            forbidden_actions=[bad.action] if bad.action != good.action else [],
            evidence_refs=evidence_refs,
        )

    def to_json(fixture: RegressionFixture) -> str:
        return json.dumps(asdict(fixture), ensure_ascii=False, sort_keys=True, indent=2)

这里的重点不是代码复杂度，而是边界清楚：fixture 里存的是可复现决策的最小事实，不是事故全文。

## 4. pi-mono：发布前跑 Recurrence Gate

pi-mono 里可以把防复发闸门放在 learning rule activation 之前。任何学习规则、prompt pack、policy cache 或 retrieval index 变更，都要 replay 对应 regression pack。

伪代码：

    interface RegressionFixture {
      fixtureId: string;
      sourceRecoveryId: string;
      ruleId: string;
      failureMode: string;
      expected: {
        action: string;
        risk: string;
        reasonCode: string;
        activationId: string;
        policyCacheHash: string;
      };
      forbiddenActions: string[];
      evidenceRefs: string[];
    }

    export class LearningRuleRecurrenceGate {
      constructor(
        private readonly fixtures: RegressionFixtureStore,
        private readonly replay: DecisionReplayEngine,
      ) {}

      async check(change: LearningRuleActivationChange): Promise<GateDecision> {
        const pack = await this.fixtures.findByRuleOrDependency({
          ruleId: change.ruleId,
          promptPackHash: change.promptPackHash,
          policyCacheHash: change.policyCacheHash,
          retrievalIndexHash: change.retrievalIndexHash,
        });

        const failures = [];

        for (const fixture of pack) {
          const observed = await this.replay.run(fixture, change);

          if (fixture.forbiddenActions.includes(observed.action)) {
            failures.push({ fixtureId: fixture.fixtureId, reason: "forbidden_action_recurred" });
            continue;
          }

          if (observed.action !== fixture.expected.action) {
            failures.push({ fixtureId: fixture.fixtureId, reason: "expected_action_changed" });
          }
        }

        return failures.length === 0
          ? { action: "allow", checkedFixtures: pack.length }
          : { action: "block", failures };
      }
    }

这个 Gate 的职责很单一：**曾经造成 Recovery Case 的错误，不允许在没有显式复核的情况下重新进入生产。**

## 5. OpenClaw 实战：课程 cron 的防复发样本

拿课程 cron 举例。假设某次事故是“已讲内容去重加载了旧 TOOLS.md，导致重复选题”。恢复动作可以是发 follow-up、修 README、补 lesson、forward-fix commit。

但恢复之后还要加一个 regression fixture：

    {
      "fixtureId": "reg_agent_course_duplicate_topic",
      "sourceRecoveryId": "rec_agent_course_duplicate_topic_20260522",
      "workflow": "agent_course_cron",
      "input": {
        "candidateTopic": "Agent 学习规则回滚补偿与恢复编排",
        "toolsMemoryContains": true,
        "readmeContainsLesson": true
      },
      "expectedDecision": {
        "action": "reject_topic",
        "reasonCode": "topic_already_taught"
      },
      "forbiddenDecision": {
        "action": "publish_lesson"
      }
    }

以后课程 cron 每次选题前，都可以先 replay 这类 fixture：如果候选主题已经出现在 TOOLS.md 或 README.md，却仍然进入 publish_lesson，就直接 block。

这比“记得下次小心”可靠得多，因为它把事故教训变成了执行路径里的闸门。

## 6. 回归固化也要有晋级流程

不是每个 Regression Candidate 都应该立刻进入强制 gate。建议分四级：

1. candidate：刚从 Recovery Case 抽取，等待脱敏和最小化。
2. shadow：跟跑发布流程，只记录是否会拦截。
3. enforce：正式阻断同类复发。
4. retired：规则或系统已经重构，样本不再适用，但保留归档证据。

晋级条件可以包括：

- sample_minimized：样本不含无关日志和敏感信息；
- replay_deterministic：同一输入多次 replay 结果稳定；
- false_positive_budget_ok：shadow 期误拦截率可接受；
- owner_approved：高风险 gate 有负责人确认；
- dependency_bound：样本绑定的 rule/prompt/policy/index 依赖清楚。

这样能避免另一个问题：为了防复发加了太多粗糙测试，最后把正常发布全部卡死。

## 7. 工程落地清单

如果你要给 Agent 平台补这一层，最小闭环是：

1. Recovery Case 关闭前自动生成 Regression Candidate。
2. Candidate 经过脱敏、最小化、去重，再进入 regression pack。
3. Regression fixture 记录 expected decision 和 forbidden regression。
4. Activation / prompt pack / policy cache / retrieval index 发布前跑 replay。
5. Recurrence Gate 对同类复发直接 block，对不确定样本 manual_review。
6. Gate 结果写入 Promotion Decision Record，方便审计为什么放行或阻断。
7. 定期清理 retired fixture，避免旧系统形态拖慢新发布。

## 8. 今日要点

成熟 Agent 的恢复闭环不是：

> 出事 -> 回滚 -> 补偿 -> 关闭。

而是：

> 出事 -> 回滚 -> 补偿 -> 关闭 Recovery Case -> 抽取 Regression Candidate -> 最小化 fixture -> 发布前 replay -> Recurrence Gate 防复发。

恢复证明系统现在好了；回归固化证明同类错误以后更难再回来。
