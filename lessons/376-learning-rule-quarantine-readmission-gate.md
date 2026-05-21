# 376. Agent 学习规则隔离后的重新接纳闸门（Learning Rule Quarantine & Re-admission Gate）

上一课讲了 Post-Recovery Regression Hardening：Recovery Case 关闭后，要把失败抽成 Regression Candidate，进入回归包，并在下一次学习规则、prompt pack、policy cache 或 retrieval index 发布前 replay，防止同类错误复发。

今天继续补一个经常被忽略的环节：**被 quarantine 或 rollback 的学习规则，不能因为修了一行配置、补了一个 fixture，就直接回到 active。它需要重新接纳流程。**

学习规则一旦进过隔离区，就说明它曾经污染过决策路径，或者它依赖的证据、schema、权限边界、运行时激活状态出现过问题。重新启用它，本质上不是“解除封印”，而是一次带历史风险的再发布。

一句话：**Quarantine 负责把可疑规则撤出生产，Re-admission Gate 负责证明它什么时候、以多大范围、带哪些限制重新回来。**

## 1. 为什么不能直接从 quarantined 变 active

很多系统会把学习规则状态设计得很简单：

    active -> quarantined -> active

这在生产里很危险。因为 quarantined 的原因可能完全不同：

- source evidence 被 tombstone，规则需要重新派生；
- schema migration 补了 synthetic 字段，关键决策不能直接依赖；
- canary false positive 超预算，规则误拦截了好任务；
- activation hash 漂移，prompt pack / policy cache / retrieval index 不一致；
- tenant 或 subject 绑定错误，个人偏好污染了公共上下文；
- rollback 后虽然补偿完成，但影响窗口里还有未验证的下游痕迹。

这些问题的修复证据不同，重新接纳路径也不同。简单改回 active，相当于跳过了事故后的学习。

更稳的状态机应该是：

    active
      -> quarantined
      -> repaired
      -> shadow_readmission
      -> canary_readmission
      -> active

其中 repaired 只代表“有候选修复”，不代表“可以影响生产”。

## 2. Re-admission Request 要绑定隔离原因和修复证据

每次请求重新接纳，都应该生成一个 Re-admission Request：

    {
      "requestId": "readmit_lr_check_pr_before_push_v8_20260522",
      "ruleId": "lr_check_pr_before_push",
      "candidateVersion": "v8",
      "previousBadVersion": "v7",
      "lastKnownGoodVersion": "v6",
      "quarantineReason": "runtime_activation_hash_drift",
      "repairEvidence": [
        "fix_policy_cache_invalidation_4f2",
        "reg_agent_course_duplicate_topic",
        "post_recovery_probe_9ac"
      ],
      "requestedScope": {
        "tenants": ["internal"],
        "channels": ["cron"],
        "tools": ["git.push", "message.send"],
        "trafficPercent": 5
      }
    }

这里有两个关键点：

第一，request 绑定 candidateVersion，而不是原地复活旧版本。被隔离的版本保持历史状态，新版本带新的 hash、依赖锁和修复证据。

第二，requestedScope 是请求方想要的范围，不是 gate 最终允许的范围。Gate 可以缩小范围、降级到 shadow、要求人工复核，或者直接拒绝。

## 3. learn-claude-code：最小重新接纳判定器

教学版可以先写一个纯函数：输入 quarantine 记录、修复证据、回归结果和请求范围，输出重新接纳决策。

    from dataclasses import dataclass
    from typing import Literal

    Decision = Literal["reject", "shadow_only", "canary", "active"]

    @dataclass(frozen=True)
    class ReadmissionRequest:
        rule_id: str
        candidate_version: str
        quarantine_reason: str
        requested_traffic_percent: int
        requested_risk: str
        repair_evidence_count: int

    @dataclass(frozen=True)
    class ReplaySummary:
        total_cases: int
        failed_cases: int
        false_positive_rate: float
        deterministic: bool

    @dataclass(frozen=True)
    class ReadmissionDecision:
        decision: Decision
        allowed_traffic_percent: int
        reason: str
        required_next_checks: list[str]

    def decide_readmission(req: ReadmissionRequest, replay: ReplaySummary) -> ReadmissionDecision:
        if req.repair_evidence_count == 0:
            return ReadmissionDecision("reject", 0, "missing_repair_evidence", [])

        if not replay.deterministic:
            return ReadmissionDecision("reject", 0, "replay_not_deterministic", ["fix_flaky_regression_cases"])

        if replay.total_cases < 20:
            return ReadmissionDecision("shadow_only", 0, "insufficient_replay_coverage", ["collect_more_cases"])

        if replay.failed_cases > 0:
            return ReadmissionDecision("reject", 0, "regression_replay_failed", ["repair_rule_or_fixture"])

        if replay.false_positive_rate > 0.01:
            return ReadmissionDecision("shadow_only", 0, "false_positive_budget_exceeded", ["observe_shadow_diff"])

        if req.requested_risk in {"high", "critical"}:
            return ReadmissionDecision("canary", min(req.requested_traffic_percent, 5), "high_risk_requires_canary", ["observe_canary_slo"])

        return ReadmissionDecision("canary", min(req.requested_traffic_percent, 25), "readmission_allowed_with_cap", ["observe_canary_slo"])

这段代码没有任何模型调用，故意保持可测试。生产系统里，LLM 可以解释原因，但不能替代 gate 判定。

## 4. pi-mono：Re-admission Gate 放在规则发布流水线前

pi-mono 里可以把重新接纳闸门放在 LearningReleaseMiddleware 之前。只有 gate 给出 shadow_only / canary / active，发布控制器才允许创建 release spec。

伪代码：

    type ReadmissionAction = "reject" | "shadow_only" | "canary" | "active";

    interface ReadmissionDecision {
      action: ReadmissionAction;
      ruleId: string;
      candidateVersion: string;
      allowedScope: {
        tenants: string[];
        channels: string[];
        tools: string[];
        trafficPercent: number;
      };
      reasonCodes: string[];
      evidenceRefs: string[];
    }

    export class LearningRuleReadmissionGate {
      constructor(
        private readonly quarantineStore: QuarantineStore,
        private readonly regression: RegressionReplayEngine,
        private readonly dependency: DependencyLockVerifier,
      ) {}

      async check(request: ReadmissionRequest): Promise<ReadmissionDecision> {
        const quarantine = await this.quarantineStore.findLatest(request.ruleId);
        if (!quarantine) {
          return reject(request, "rule_was_not_quarantined");
        }

        if (request.candidateVersion === quarantine.badVersion) {
          return reject(request, "cannot_readmit_same_bad_version");
        }

        const dependencyOk = await this.dependency.verify(request.candidateManifest);
        if (!dependencyOk.ok) {
          return reject(request, "dependency_lock_mismatch", dependencyOk.evidenceRefs);
        }

        const replay = await this.regression.runForRule({
          ruleId: request.ruleId,
          candidateVersion: request.candidateVersion,
          includeRecoveryFixtures: true,
        });

        if (replay.failures.length > 0) {
          return reject(request, "recovery_regression_failed", replay.evidenceRefs);
        }

        const scope = shrinkScope(request.requestedScope, quarantine.risk);

        return {
          action: quarantine.risk === "critical" ? "shadow_only" : "canary",
          ruleId: request.ruleId,
          candidateVersion: request.candidateVersion,
          allowedScope: scope,
          reasonCodes: ["quarantined_rule_requires_gradual_readmission"],
          evidenceRefs: [...replay.evidenceRefs, ...dependencyOk.evidenceRefs],
        };
      }
    }

核心是两条：

1. 旧坏版本不能复活，只能发布候选新版本。
2. 重新接纳默认缩小范围，先 shadow 或 canary，用真实运行证据再晋级。

## 5. OpenClaw 实战：课程 cron 的规则重新接纳

拿课程 cron 举例。假设“选题去重规则”因为读取旧 TOOLS.md 导致重复课程，被 quarantine 了。

修复后不要直接让它重新决定发布。更稳的重新接纳计划是：

    {
      "ruleId": "lr_agent_course_topic_dedup",
      "candidateVersion": "v4",
      "readmissionPlan": [
        {
          "stage": "shadow",
          "traffic": 0,
          "behavior": "只计算候选是否重复，不影响真实选题",
          "exitCriteria": "连续 3 次课程 cron 与主规则结论一致"
        },
        {
          "stage": "canary",
          "traffic": 10,
          "behavior": "只拦截明显出现在 README.md 和 TOOLS.md 的重复主题",
          "exitCriteria": "无 false positive，且 regression fixture 全通过"
        },
        {
          "stage": "active",
          "traffic": 100,
          "behavior": "恢复为正式选题前置闸门",
          "exitCriteria": "写入 activation receipt 和 runtime consistency proof"
        }
      ]
    }

这就是把“我修好了”变成“我先旁路观察，再小范围接管，最后带收据生效”。

## 6. 重新接纳要限制权限扩大

Re-admission Gate 有一个默认原则：**重新接纳不能扩大原规则的权限边界。**

如果旧规则只影响 cron 选题，新规则不能顺手影响 Telegram 消息编辑、GitHub PR review 或生产部署。如果旧规则只属于某个 tenant，新规则不能跨 tenant 注入 prompt pack。

可以用 scope diff 做硬闸门：

    {
      "previousAllowedScope": {
        "channels": ["cron"],
        "tools": ["git.status", "git.push"],
        "tenants": ["agent-course"]
      },
      "requestedScope": {
        "channels": ["cron", "telegram"],
        "tools": ["git.status", "git.push", "message.delete"],
        "tenants": ["agent-course", "mysterybox"]
      },
      "decision": "reject",
      "reason": "readmission_scope_expansion_requires_new_rule_review"
    }

重新接纳是恢复可信，不是趁机扩权。需要扩权时，应该走新规则创建和完整发布流程。

## 7. 什么时候才能从 canary 进 active

canary 通过不等于立刻 active。至少要满足：

- cleanWindow：连续一段时间没有 drift、rollback、false positive；
- influenceEvents：真实影响事件数量达到最小样本；
- regressionReplay：恢复后固化的 fixture 全部通过；
- activationReceipt：prompt pack、policy cache、retrieval index 都看到同一 candidateVersion；
- ownerReview：高风险规则有人确认 reasonCodes 和证据；
- rollbackReady：仍然保留 lastKnownGoodVersion 和一键 rollback path。

晋级记录可以写成 Promotion Decision Record：

    {
      "decisionId": "promote_readmit_lr_agent_course_topic_dedup_v4",
      "fromStage": "canary_readmission",
      "toStage": "active",
      "cleanWindowHours": 24,
      "influenceEvents": 42,
      "regressionFailures": 0,
      "rollbackTarget": "lr_agent_course_topic_dedup_v3",
      "decision": "promote"
    }

这样以后如果 v4 再出问题，系统能知道它是基于哪些证据重新回来的。

## 8. 工程落地清单

给 Agent 平台补 Re-admission Gate，最小闭环是：

1. quarantined 规则不能原地 active，只能创建 candidate version。
2. Re-admission Request 绑定 quarantineReason、repairEvidence、requestedScope。
3. Gate 检查 dependency lock、regression pack、scope diff 和 replay determinism。
4. 默认从 shadow 或 canary 重新接纳，限制 traffic 和作用域。
5. canary 晋级 active 需要 clean window、influence sample、activation receipt 和 rollback target。
6. 所有 readmission / promotion / rejection 写入不可变决策记录。
7. 如果请求扩大权限，拒绝 readmission，改走新规则完整发布流程。

## 9. 今日要点

成熟 Agent 的学习规则生命周期不是：

> quarantine -> 修一下 -> active。

而是：

> quarantine -> candidate repair -> readmission request -> dependency/replay/scope gate -> shadow -> canary -> active receipt。

被隔离过的规则重新回来时，系统要更谨慎，不是更健忘。重新接纳闸门让 Agent 能安全地恢复有价值的经验，同时不把旧事故悄悄带回生产。
