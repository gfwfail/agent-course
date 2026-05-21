# 373. Agent 学习规则运行时漂移检测与自动回滚（Learning Rule Runtime Drift Detection & Auto-Rollback）

上一课讲了 Activation Receipt：Promotion Decision applied 以后，还要生成生效收据，证明 rule store、prompt pack、policy cache、retrieval index 和真实运行入口都看到了同一个 ruleVersion/hash。

今天继续补下一层：**Activation Receipt 只能证明某个时间点生效，不能证明规则后续一直没有漂移。**

真实系统里，学习规则发布成功以后仍然可能慢慢变脏：

- 某个 worker 重启后加载了旧 prompt pack；
- policy cache 被人工刷新到旧版本；
- retrieval index 延迟重建，部分 shard 查不到新规则；
- 配置热更新漏了 dependency lock；
- 多实例灰度期间，有 10% 流量还在旧 rule hash；
- 规则命中了真实决策，但 runtime evidence 没有记录 reason code。

一句话：**Activation Receipt 证明“刚刚生效了”，Runtime Drift Detector 证明“现在仍然一致”，Auto-Rollback 负责在漂移影响高风险动作前把系统退回可信状态。**

## 1. 什么叫学习规则漂移

学习规则漂移不是模型输出变了这么简单。这里的 drift 指的是：发布链路声明的 expected runtime state，和实际运行入口观察到的 observed runtime state 不一致。

一个最小的 expected state 可以长这样：

    {
      "ruleId": "lr_check_pr_before_push",
      "ruleVersion": 7,
      "tier": "enforce",
      "activationId": "act_lr_check_pr_before_push_v7",
      "artifactHashes": {
        "promptPack": "sha256:prompt_7aa",
        "policyCache": "sha256:policy_42c",
        "retrievalIndex": "sha256:index_911"
      }
    }

运行时探针看到的 observed state：

    {
      "component": "policyCache",
      "ruleId": "lr_check_pr_before_push",
      "observedVersion": 6,
      "observedHash": "sha256:policy_old",
      "trafficShare": 0.18,
      "sampledAt": "2026-05-21T11:30:00Z"
    }

这就是 drift。特别危险的是 tier 已经是 enforce，但部分运行路径仍在旧版本：系统看起来已经有护栏，实际某些请求还没被新护栏保护。

## 2. Drift Detector 的输出不能只有 true/false

漂移检测要输出结构化结果，方便后续自动处理：

    {
      "driftId": "drift_20260521_lr_check_pr_before_push_v7",
      "ruleId": "lr_check_pr_before_push",
      "expectedVersion": 7,
      "observations": [
        {
          "component": "policyCache",
          "observedVersion": 6,
          "status": "version_mismatch",
          "trafficShare": 0.18
        }
      ],
      "severity": "block_side_effects",
      "recommendedAction": "rollback_to_last_known_good",
      "evidenceRefs": ["ev_probe_policy_cache_1"],
      "createdAt": "2026-05-21T11:30:00Z"
    }

这里有两个重点：

- severity 要按影响面分级，不能所有 mismatch 都直接炸生产；
- recommendedAction 要可执行，后续 rollback worker 才能幂等处理。

一个实用的分级：

- observe_only：低风险组件轻微延迟，继续观察。
- refresh_component：单组件 cache 漂移，先刷新。
- block_side_effects：高风险 enforce 规则漂移，阻断相关副作用。
- rollback_to_last_known_good：刷新失败或影响面扩大，回滚到上一版激活收据。
- incident_escalation：回滚失败或证据链断裂，升级事故。

## 3. 自动回滚要回到 Last Known Good，不是随便回旧版

自动回滚最常见的坑是“看到 v7 漂移，就退回 v6”。这不够严谨。

正确做法是回到 Last Known Good Activation：

    {
      "activationId": "act_lr_check_pr_before_push_v6",
      "ruleVersion": 6,
      "state": "activated",
      "artifactHashes": {
        "promptPack": "sha256:prompt_6bc",
        "policyCache": "sha256:policy_5fa",
        "retrievalIndex": "sha256:index_802"
      },
      "verifiedAt": "2026-05-21T08:30:08Z"
    }

为什么不能只按 version 回滚？

- v6 的某个 artifact 可能已经被重建，hash 不再一样；
- v6 可能后来被 tombstone 或 revocation 作废；
- v6 可能只是 applied，从未拿到 activation receipt；
- rollback 本身也需要生成新的 Rollback Receipt。

所以 rollback target 必须来自已验证的 activation ledger，而不是靠版本号猜。

## 4. learn-claude-code：最小漂移检测器

教学版可以把 expected manifest 和 runtime observations 都放在 JSON 里，先用纯函数检测漂移：

    from dataclasses import dataclass, asdict
    from datetime import datetime, timezone
    import json

    @dataclass(frozen=True)
    class DriftObservation:
        component: str
        expected_version: int
        observed_version: int
        expected_hash: str
        observed_hash: str
        traffic_share: float
        status: str

    @dataclass(frozen=True)
    class DriftReport:
        drift_id: str
        rule_id: str
        observations: list[DriftObservation]
        severity: str
        recommended_action: str
        created_at: str

    def compare_component(name: str, expected: dict, observed: dict) -> DriftObservation:
        observed_version = observed.get("rule_version", -1)
        observed_hash = observed.get("hash", "missing")
        ok = observed_version == expected["rule_version"] and observed_hash == expected["hash"]

        if ok:
            status = "ok"
        elif observed_hash == "missing":
            status = "missing"
        elif observed_version != expected["rule_version"]:
            status = "version_mismatch"
        else:
            status = "hash_mismatch"

        return DriftObservation(
            component=name,
            expected_version=expected["rule_version"],
            observed_version=observed_version,
            expected_hash=expected["hash"],
            observed_hash=observed_hash,
            traffic_share=observed.get("traffic_share", 0.0),
            status=status,
        )

    def classify_drift(observations: list[DriftObservation], tier: str) -> tuple[str, str]:
        bad = [o for o in observations if o.status != "ok"]
        if not bad:
            return "observe_only", "none"

        affected_share = sum(o.traffic_share for o in bad)
        critical_component = any(o.component in {"policyCache", "promptPack"} for o in bad)

        if tier == "enforce" and critical_component and affected_share >= 0.05:
            return "block_side_effects", "rollback_to_last_known_good"
        if critical_component:
            return "refresh_component", "refresh_component"
        return "observe_only", "continue_probe"

    def detect_drift(rule_id: str, tier: str, manifest: dict, runtime: dict) -> DriftReport:
        observations = [
            compare_component(name, expected, runtime.get(name, {}))
            for name, expected in manifest.items()
        ]
        severity, action = classify_drift(observations, tier)

        return DriftReport(
            drift_id=f"drift_{rule_id}_{int(datetime.now(timezone.utc).timestamp())}",
            rule_id=rule_id,
            observations=observations,
            severity=severity,
            recommended_action=action,
            created_at=datetime.now(timezone.utc).isoformat(),
        )

    def to_json(report: DriftReport) -> str:
        payload = asdict(report)
        payload["observations"] = [asdict(o) for o in report.observations]
        return json.dumps(payload, ensure_ascii=False, sort_keys=True)

这段代码可以直接接在上一课的 Activation Receipt 后面：receipt 生成 expected manifest，drift detector 定期读取 runtime snapshot 做比对。

## 5. pi-mono：Drift Middleware + Rollback Worker

pi-mono 里更适合把它做成两个组件：

- LearningRuleDriftMiddleware：在高风险动作前抽样检查相关 rule 的 runtime state；
- LearningRuleRollbackWorker：消费 drift event，刷新组件或回滚到 last known good。

伪代码：

    type DriftSeverity =
      | "observe_only"
      | "refresh_component"
      | "block_side_effects"
      | "rollback_to_last_known_good"
      | "incident_escalation";

    interface DriftReport {
      driftId: string;
      ruleId: string;
      expectedVersion: number;
      observations: Array<{
        component: "promptPack" | "policyCache" | "retrievalIndex";
        observedVersion: number;
        observedHash: string;
        status: "ok" | "missing" | "version_mismatch" | "hash_mismatch";
        trafficShare: number;
      }>;
      severity: DriftSeverity;
      recommendedAction: string;
      createdAt: string;
    }

    export class LearningRuleDriftMiddleware {
      constructor(private readonly runtime: LearningRuleRuntime) {}

      async beforeToolCall(ctx, next) {
        const relatedRules = await ctx.learningRules.findRulesForTool(ctx.tool.name);
        const reports = await Promise.all(
          relatedRules.map(rule => this.runtime.detectDrift(rule))
        );

        for (const report of reports) {
          if (report.severity === "block_side_effects") {
            await ctx.events.publish("learning.rule.drift_detected", report);
            throw new Error(
              `Learning rule drift detected for ${report.ruleId}; side effect blocked`
            );
          }

          if (report.severity !== "observe_only") {
            await ctx.events.publish("learning.rule.drift_detected", report);
          }
        }

        return next();
      }
    }

    export async function rollbackWorker(ctx, report: DriftReport) {
      if (report.recommendedAction === "refresh_component") {
        await ctx.learningRules.refreshDriftedComponents(report);
        return;
      }

      if (report.recommendedAction !== "rollback_to_last_known_good") return;

      const target = await ctx.activationLedger.findLastKnownGood(report.ruleId);
      if (!target) {
        await ctx.incidents.open({
          severity: "P1",
          reason: "missing_last_known_good_activation",
          driftId: report.driftId,
        });
        return;
      }

      await ctx.learningRules.restoreArtifacts(target.artifactHashes);
      const receipt = await ctx.learningRules.verifyActivation(target);
      await ctx.rollbackLedger.append({
        rollbackId: ctx.ids.next("rollback"),
        driftId: report.driftId,
        targetActivationId: target.activationId,
        receiptId: receipt.activationId,
        state: receipt.state === "activated" ? "rolled_back" : "rollback_failed",
      });
    }

这里的关键是：高风险副作用前遇到 enforce 规则漂移，要先 block，再让后台 worker 处理修复或回滚。

## 6. OpenClaw 实战：课程 Cron 也需要漂移守卫

用我们这个 Agent 开发课程 cron 举例。一次完整发布依赖几类状态：

- README 目录必须包含新课程；
- lessons/373 文件必须存在；
- TOOLS.md 已讲内容必须追加；
- Telegram messageId 必须记录；
- GitHub 远端 main 必须包含 commit；
- 下一次选题必须基于最新已讲列表，不能回退到旧记忆。

如果某个运行入口漂移，比如 TOOLS.md 更新失败但 git push 成功，系统下次可能重复讲同一个主题。

可以做一个 cron drift probe：

    {
      "probe": "agent_course_lesson_state",
      "expectedLesson": 373,
      "checks": {
        "lessonFile": "ok",
        "readmeIndex": "ok",
        "toolsTopic": "missing",
        "remoteCommit": "ok",
        "telegramDelivery": "ok"
      },
      "severity": "refresh_component",
      "recommendedAction": "append_missing_tools_topic"
    }

如果缺的是 Telegram delivery 或 remote commit，就不是 refresh_component，而应该 block 下一次课程并升级人工处理。因为那说明外部副作用证据不完整。

## 7. 实战建议

把 Learning Rule Runtime Drift Detection 落地时，先记住五条：

- 每条 enforce 学习规则都要有 expected manifest。
- 每个关键 runtime 入口都要能回答 observed ruleVersion/hash。
- drift report 要带 trafficShare，不然无法判断影响面。
- rollback 目标必须来自 Last Known Good Activation Receipt。
- rollback 以后也要生成新 receipt，证明系统真的退回去了。

成熟 Agent 的学习系统不是“发布一次就结束”，而是持续证明：规则还在、版本一致、证据有效、漂移可控、坏了能退。
