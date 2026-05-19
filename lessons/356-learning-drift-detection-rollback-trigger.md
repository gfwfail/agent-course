# 356. Agent 学习漂移检测与回滚触发（Learning Drift Detection & Rollback Trigger）

上一课讲了 Learning Canary Release：学习规则不能从复盘直接全局 active，而要经过 shadow / canary / active，并绑定 scope、traffic、rollback target 和 SLO。

今天继续讲上线后的问题：**学习规则 active 以后也不是永久正确；环境、工具、权限、用户意图和业务流程都会变，学习规则会漂移。**

成熟 Agent 不能只问“这条学习当时发布时是不是安全”，还要持续问：

- 它现在还被正确检索吗？
- 它影响的决策和当初预期一致吗？
- 它依赖的证据、工具、策略是否过期？
- 它是否开始误伤正常请求？
- 它是否已经不再阻止真实坏路径？

一句话：**学习发布不是终点，漂移检测才是长期安全线。**

## 1. 什么是学习漂移

学习漂移不是“规则写错了”，而是规则原本没错，但现实变了。

比如我们有一条学习：

~~~json
{
  "learningId": "learn.git-push.require-pr-state-check",
  "rule": "git push 前必须检查 gh pr list，避免往已 merge 分支继续 push",
  "scope": {
    "repos": ["gfwfail/agent-course"],
    "operations": ["git_push"]
  }
}
~~~

它发布时很合理。但几个月后可能出现这些漂移：

- 仓库改成 trunk-based flow，不再用 PR 分支；
- GitHub CLI 输出格式变了，解析器误判；
- 某些 cron 只提交课程内容，老板明确要求直接 push main；
- 权限模型变了，检查 PR 状态不再能证明 push 安全；
- 新的事故类型出现，这条学习挡不住真正风险。

所以学习规则要带监控，不是写进 MEMORY / Runbook / Policy 就结束。

## 2. Learning Drift Signal

漂移检测要靠事件流，而不是靠感觉。每次学习规则影响决策，都记录最小信号：

~~~json
{
  "event": "learning.influence",
  "learningId": "learn.git-push.require-pr-state-check",
  "version": 3,
  "runId": "run-356",
  "decision": "require_evidence",
  "expectedDecision": "require_evidence",
  "outcome": "good_block",
  "sourceEvidenceFresh": true,
  "toolContractVersion": "gh@2.72.0",
  "policyVersion": "policy.git.18",
  "observedAt": "2026-05-19T08:30:00Z"
}
~~~

建议最少记录六类信号：

- **retrieval drift**：该想起时没想起，或不该想起时被检索出来。
- **decision drift**：影响后的 decision 跟发布时预期不一致。
- **outcome drift**：阻断越来越多 good run，或者阻断不了 bad run。
- **dependency drift**：依赖的工具、schema、policy、runbook 版本变化。
- **evidence drift**：发布时的证据过期、撤销、失效或不再覆盖当前 scope。
- **scope drift**：业务范围扩大，但学习规则仍用旧 scope 判断。

## 3. learn-claude-code：最小漂移检测器

教学版可以做成一个纯函数：输入 release spec、影响事件、依赖版本，输出 keep / re_evaluate / shadow / rollback。

~~~python
from dataclasses import dataclass
from enum import Enum

class DriftAction(str, Enum):
    KEEP = "keep"
    RE_EVALUATE = "re_evaluate"
    DEMOTE_TO_SHADOW = "demote_to_shadow"
    ROLLBACK = "rollback"

@dataclass
class LearningRelease:
    learning_id: str
    version: int
    stage: str
    expected_tool_versions: dict[str, str]
    expected_policy_version: str
    max_false_positive_rate: float
    max_blocked_good_runs: int
    max_dependency_drift: int

@dataclass
class InfluenceEvent:
    learning_id: str
    influenced: bool
    expected_decision: str
    actual_decision: str
    outcome: str
    source_evidence_fresh: bool
    tool_versions: dict[str, str]
    policy_version: str

@dataclass
class DriftReport:
    action: DriftAction
    reasons: list[str]

def detect_learning_drift(
    release: LearningRelease,
    events: list[InfluenceEvent],
    min_samples: int = 20,
) -> DriftReport:
    relevant = [
        event for event in events
        if event.learning_id == release.learning_id and event.influenced
    ]

    if len(relevant) < min_samples:
        return DriftReport(DriftAction.KEEP, ["not enough samples"])

    reasons: list[str] = []
    blocked_good = sum(1 for event in relevant if event.outcome == "blocked_good")
    false_positive_rate = blocked_good / len(relevant)

    decision_drift = sum(
        1 for event in relevant
        if event.expected_decision != event.actual_decision
    )

    stale_evidence = sum(
        1 for event in relevant
        if not event.source_evidence_fresh
    )

    dependency_drift = 0
    for event in relevant:
        if event.policy_version != release.expected_policy_version:
            dependency_drift += 1
            continue

        for tool, expected in release.expected_tool_versions.items():
            if event.tool_versions.get(tool) != expected:
                dependency_drift += 1
                break

    if blocked_good > release.max_blocked_good_runs:
        reasons.append(f"blocked good runs exceeded: {blocked_good}")

    if false_positive_rate > release.max_false_positive_rate:
        reasons.append(f"false positive rate too high: {false_positive_rate:.2%}")

    if dependency_drift > release.max_dependency_drift:
        reasons.append(f"dependency drift exceeded: {dependency_drift}")

    if stale_evidence:
        reasons.append(f"stale evidence seen: {stale_evidence}")

    if decision_drift:
        reasons.append(f"decision drift seen: {decision_drift}")

    if blocked_good > release.max_blocked_good_runs:
        return DriftReport(DriftAction.ROLLBACK, reasons)

    if false_positive_rate > release.max_false_positive_rate:
        return DriftReport(DriftAction.DEMOTE_TO_SHADOW, reasons)

    if dependency_drift > release.max_dependency_drift or stale_evidence:
        return DriftReport(DriftAction.RE_EVALUATE, reasons)

    if decision_drift:
        return DriftReport(DriftAction.RE_EVALUATE, reasons)

    return DriftReport(DriftAction.KEEP, ["within drift budget"])
~~~

关键点不是算法复杂，而是动作要分级：

- **KEEP**：继续 active。
- **RE_EVALUATE**：暂停晋级，重新跑反事实评估和回归包。
- **DEMOTE_TO_SHADOW**：真实决策不再受影响，只记录差异。
- **ROLLBACK**：回到 rollbackTarget，发出审计事件。

## 4. pi-mono：LearningDriftMonitor

在生产版 pi-mono 里，不要把漂移检测塞进 prompt。它应该是事件流上的后台 job / middleware。

概念接口可以这样设计：

~~~ts
type LearningDriftAction =
  | "keep"
  | "re_evaluate"
  | "demote_to_shadow"
  | "rollback";

type LearningInfluenceEvent = {
  type: "learning.influence";
  learningId: string;
  version: number;
  runId: string;
  influenced: boolean;
  expectedDecision: string;
  actualDecision: string;
  outcome: "prevented_bad" | "allowed_good" | "blocked_good" | "missed_bad";
  sourceEvidenceFresh: boolean;
  toolVersions: Record<string, string>;
  policyVersion: string;
  observedAt: string;
};

type LearningDriftReport = {
  learningId: string;
  version: number;
  action: LearningDriftAction;
  reasons: string[];
  sampleSize: number;
  evidenceRefs: string[];
};

class LearningDriftMonitor {
  constructor(
    private readonly releases: LearningReleaseStore,
    private readonly events: EventStream,
    private readonly evidence: EvidenceStore,
  ) {}

  async evaluate(learningId: string): Promise<LearningDriftReport> {
    const release = await this.releases.getActive(learningId);
    const window = await this.events.query<LearningInfluenceEvent>({
      type: "learning.influence",
      learningId,
      since: release.activatedAt,
      limit: 500,
    });

    const report = computeLearningDrift(release, window);

    const evidenceRef = await this.evidence.append({
      type: "learning.drift_report",
      learningId,
      releaseVersion: release.version,
      action: report.action,
      reasons: report.reasons,
      sampleSize: window.length,
      sourceEventIds: window.map((event) => event.runId),
    });

    if (report.action === "rollback") {
      await this.releases.rollback({
        learningId,
        to: release.rollbackTarget,
        reason: report.reasons.join("; "),
        evidenceRef,
      });
    }

    if (report.action === "demote_to_shadow") {
      await this.releases.demote({
        learningId,
        stage: "shadow",
        reason: report.reasons.join("; "),
        evidenceRef,
      });
    }

    return {
      ...report,
      evidenceRefs: [evidenceRef],
    };
  }
}
~~~

这里有两个工程要点：

- 漂移检测从 **EventStream** 读真实影响事件，不靠 LLM 自我感觉。
- 回滚 / 降级要写 **EvidenceStore**，否则以后没人知道为什么规则突然失效。

## 5. OpenClaw 实战：课程 Cron 的漂移监控

OpenClaw 的课程 cron 很适合做这个例子。

这个任务每 3 小时都会：

1. 检查已讲内容，避免重复。
2. 写 lesson 文件。
3. 更新 README 和 TOOLS。
4. 发 Telegram。
5. git commit / push。
6. 写 memory 证据。

可以把“避免重复课程”的学习规则做成 active rule：

~~~json
{
  "learningId": "learn.course.avoid-duplicate-topic",
  "stage": "active",
  "rule": "选题前必须检查 TOOLS.md 已讲内容和 README 最新目录",
  "rollbackTarget": {
    "stage": "shadow",
    "version": 4
  },
  "driftBudget": {
    "maxDuplicateTopic": 0,
    "maxBlockedGoodTopics": 2,
    "maxStaleCatalogRuns": 1
  }
}
~~~

每次 cron 完成后追加影响事件：

~~~json
{
  "type": "learning.influence",
  "learningId": "learn.course.avoid-duplicate-topic",
  "runId": "cron-2026-05-19T08:30:00Z",
  "influenced": true,
  "expectedDecision": "require_catalog_check",
  "actualDecision": "require_catalog_check",
  "outcome": "allowed_good",
  "sourceEvidenceFresh": true,
  "toolVersions": {
    "rg": "14.x",
    "git": "2.x"
  },
  "policyVersion": "course-cron-v6"
}
~~~

漂移触发例子：

- README 和 TOOLS 目录不同步超过 1 次：re_evaluate。
- 发现 Telegram 发了重复主题：rollback。
- catalog 检查误阻断 2 个合理新主题：demote_to_shadow。
- TOOLS 已讲内容格式变化，解析器无法覆盖：re_evaluate。

这样课程 Agent 不是只会“记住不要重复”，而是能证明这条记忆仍然有效。

## 6. 常见坑

**坑 1：只看命中率，不看结果**

一条学习被频繁检索，不代表它有用。要看 prevented_bad、blocked_good、missed_bad。

**坑 2：漂移只报警，不改变状态**

如果检测到高误伤但规则仍 active，报警只是噪音。漂移报告必须能触发 shadow / rollback / re-eval。

**坑 3：把 dependency drift 当成失败**

工具版本变化不一定说明规则错了，但说明旧评估证据不够了。正确动作通常是 re_evaluate，不是立刻删除。

**坑 4：没有 rollback target**

active 学习规则没有 rollback target，就不应该 active。否则漂移时只能靠人临场修。

## 7. 一句话总结

学习规则上线后，现实会继续变化。成熟 Agent 要把每条 active learning 当成长期运行的生产组件：持续收集影响事件，检测 outcome / dependency / evidence / scope 漂移，并在超出预算时自动重评估、降级或回滚。

记住：**能上线的是学习能力；能持续证明没漂移的，才是可靠能力。**
