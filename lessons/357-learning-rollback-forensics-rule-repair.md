# 357. Agent 学习回滚取证与规则修复（Learning Rollback Forensics & Rule Repair）

上一课讲了 Learning Drift Detection：学习规则上线后要持续监控 retrieval / decision / outcome / dependency / evidence / scope 漂移，并在必要时触发 re_evaluate、demote_to_shadow 或 rollback。

今天继续补上回滚后的闭环：**学习规则被回滚，不代表事情结束；它只是从生产决策路径里撤出，接下来必须取证、定位、修规则、重放验证，再决定是否重新发布。**

成熟 Agent 不能把学习回滚当成“删掉一条记忆”。正确流程应该像修生产 bug：

- 先冻结这条学习规则的 active 影响；
- 保存导致回滚的 influence events、工具版本、policy 版本和 evidence refs；
- 判断问题来自规则本身、scope、依赖、证据还是回放集缺口；
- 生成 Rule Repair Ticket；
- 修完后跑 counterfactual replay + regression pack；
- 重新走 shadow / canary / active 发布。

一句话：**学习回滚不是遗忘，而是带证据的规则修复流程。**

## 1. 为什么不能只 disable learning

假设有一条学习：

~~~json
{
  "learningId": "learn.git-push.require-pr-state-check",
  "rule": "git push 前必须检查 gh pr list，避免往已 merge 分支继续 push",
  "stage": "active"
}
~~~

漂移监控发现它开始误伤：老板明确要求课程 cron 直接 push main，但规则仍强制走 PR 检查，于是触发 rollback。

最差的处理方式是：

~~~text
learning.active = false
~~~

这样会丢掉四类关键问题：

- 是规则太宽，还是当前任务属于例外？
- 是 scope 没写清楚，还是 policy 版本变化了？
- 是 gh 工具输出变了，还是解析器误判？
- 是回归包没覆盖 cron 直推场景？

所以学习回滚必须生成一个可审计的 Rollback Case。

## 2. Learning Rollback Case

回滚记录不要只写一句“误伤了”。建议最小结构如下：

~~~json
{
  "rollbackId": "lr-2026-05-19-357",
  "learningId": "learn.git-push.require-pr-state-check",
  "fromVersion": 3,
  "rollbackTarget": 2,
  "trigger": "blocked_good_runs_exceeded",
  "sampleWindow": {
    "from": "2026-05-19T08:30:00Z",
    "to": "2026-05-19T11:30:00Z",
    "sampleSize": 41
  },
  "impact": {
    "blockedGoodRuns": 7,
    "missedBadRuns": 0,
    "affectedCapabilities": ["git_push", "course_cron"]
  },
  "evidenceRefs": [
    "eventstream://learning.influence/run-356",
    "git://gfwfail/agent-course@b039cc3",
    "telegram://-5115329245/12219"
  ],
  "initialHypothesis": "scope too broad: course cron direct push was not represented as allowed path",
  "requiredRepair": "narrow rule to PR branch pushes; add direct-main cron exception with explicit user instruction evidence"
}
~~~

这个 Case 是后续修复、回放、重新发布的共同事实源。

## 3. learn-claude-code：最小回滚取证器

教学版可以先做一个纯函数：输入 active release、漂移报告、影响事件，输出 RollbackCase 和 RepairTicket。

~~~python
from dataclasses import dataclass
from enum import Enum
from time import time

class RepairCause(str, Enum):
    RULE_TOO_BROAD = "rule_too_broad"
    RULE_TOO_NARROW = "rule_too_narrow"
    STALE_EVIDENCE = "stale_evidence"
    DEPENDENCY_CHANGED = "dependency_changed"
    UNKNOWN = "unknown"

@dataclass
class InfluenceEvent:
    run_id: str
    learning_id: str
    outcome: str
    capability: str
    evidence_fresh: bool
    tool_versions: dict[str, str]
    policy_version: str
    evidence_refs: list[str]

@dataclass
class LearningRelease:
    learning_id: str
    version: int
    rollback_target: int
    expected_tool_versions: dict[str, str]
    expected_policy_version: str
    scope_capabilities: list[str]

@dataclass
class RollbackCase:
    rollback_id: str
    learning_id: str
    from_version: int
    rollback_target: int
    trigger: str
    cause: RepairCause
    blocked_good_runs: int
    missed_bad_runs: int
    affected_capabilities: list[str]
    evidence_refs: list[str]
    required_repair: str

def classify_repair_cause(release: LearningRelease, events: list[InfluenceEvent]) -> RepairCause:
    blocked_good = [event for event in events if event.outcome == "blocked_good"]
    missed_bad = [event for event in events if event.outcome == "missed_bad"]

    if any(not event.evidence_fresh for event in events):
        return RepairCause.STALE_EVIDENCE

    dependency_changed = any(
        event.policy_version != release.expected_policy_version
        or any(
            event.tool_versions.get(tool) != expected
            for tool, expected in release.expected_tool_versions.items()
        )
        for event in events
    )
    if dependency_changed:
        return RepairCause.DEPENDENCY_CHANGED

    affected = {event.capability for event in blocked_good}
    if blocked_good and affected.issubset(set(release.scope_capabilities)):
        return RepairCause.RULE_TOO_BROAD

    if missed_bad:
        return RepairCause.RULE_TOO_NARROW

    return RepairCause.UNKNOWN

def build_rollback_case(release: LearningRelease, events: list[InfluenceEvent], trigger: str) -> RollbackCase:
    cause = classify_repair_cause(release, events)
    blocked_good = [event for event in events if event.outcome == "blocked_good"]
    missed_bad = [event for event in events if event.outcome == "missed_bad"]
    affected = sorted({event.capability for event in blocked_good + missed_bad})
    evidence_refs = sorted({ref for event in events for ref in event.evidence_refs})

    repair_by_cause = {
        RepairCause.RULE_TOO_BROAD: "narrow scope and add explicit allowed-path conditions",
        RepairCause.RULE_TOO_NARROW: "add missing deny condition and regression case",
        RepairCause.STALE_EVIDENCE: "refresh evidence refs before re-evaluation",
        RepairCause.DEPENDENCY_CHANGED: "update dependency contract and rerun counterfactual replay",
        RepairCause.UNKNOWN: "manual review required before re-release",
    }

    return RollbackCase(
        rollback_id=f"lr-{int(time())}",
        learning_id=release.learning_id,
        from_version=release.version,
        rollback_target=release.rollback_target,
        trigger=trigger,
        cause=cause,
        blocked_good_runs=len(blocked_good),
        missed_bad_runs=len(missed_bad),
        affected_capabilities=affected,
        evidence_refs=evidence_refs,
        required_repair=repair_by_cause[cause],
    )
~~~

这里的重点不是分类多聪明，而是**回滚必须产出下一步动作**。如果 cause 是 unknown，就不能偷偷重新 active，必须 manual review。

## 4. pi-mono：LearningRollbackRepairMiddleware

生产版里，LearningDriftMonitor 只负责发现漂移和触发动作；回滚后的修复应由单独组件接管。

概念接口可以这样设计：

~~~ts
type LearningRollbackCause =
  | "rule_too_broad"
  | "rule_too_narrow"
  | "stale_evidence"
  | "dependency_changed"
  | "missing_regression"
  | "unknown";

type LearningRollbackCase = {
  rollbackId: string;
  learningId: string;
  fromVersion: number;
  rollbackTarget: number;
  trigger: string;
  cause: LearningRollbackCause;
  blockedGoodRuns: number;
  missedBadRuns: number;
  affectedCapabilities: string[];
  evidenceRefs: string[];
  requiredRepair: string;
};

type LearningRepairTicket = {
  ticketId: string;
  rollbackId: string;
  learningId: string;
  owner: "agent" | "human";
  writeScope: string[];
  forbiddenScope: string[];
  requiredReplayTags: string[];
  requiredEvidenceRefs: string[];
};

class LearningRollbackRepairMiddleware {
  constructor(
    private readonly releases: LearningReleaseStore,
    private readonly events: EventStream,
    private readonly tickets: RepairTicketStore,
  ) {}

  async handleRollback(report: LearningDriftReport): Promise<LearningRollbackCase> {
    const release = await this.releases.getActive(report.learningId);
    const influenceEvents = await this.events.query<LearningInfluenceEvent>({
      type: "learning.influence",
      learningId: report.learningId,
      since: release.activatedAt,
      limit: 500,
    });

    await this.releases.rollback({
      learningId: report.learningId,
      fromVersion: release.version,
      toVersion: release.rollbackTarget,
      reason: report.reasons.join("; "),
    });

    const rollbackCase = classifyRollbackCase(release, influenceEvents, report);

    const ticket: LearningRepairTicket = {
      ticketId: "repair-" + rollbackCase.rollbackId,
      rollbackId: rollbackCase.rollbackId,
      learningId: rollbackCase.learningId,
      owner: rollbackCase.cause === "unknown" ? "human" : "agent",
      writeScope: [
        "memory/learning/" + rollbackCase.learningId + ".json",
        "regression-packs/learning/**/*.json",
      ],
      forbiddenScope: ["policy/global.json", "tools/**/*"],
      requiredReplayTags: rollbackCase.affectedCapabilities.map(
        (capability) => "capability:" + capability,
      ),
      requiredEvidenceRefs: rollbackCase.evidenceRefs,
    };

    await this.tickets.create(ticket);

    await this.events.append({
      type: "learning.rollback_case.created",
      rollbackCase,
      ticket,
      observedAt: new Date().toISOString(),
    });

    return rollbackCase;
  }
}
~~~

注意 pi-mono 的 Agent.subscribe() 和 EventStream 模式很适合做这件事：Agent 主循环继续专注执行，学习治理在外层监听事件、创建 ticket、触发 replay gate，不污染 prompt。

## 5. OpenClaw 实战：课程 cron 的回滚闭环

OpenClaw 这类 Always-on Agent 最容易遇到“长期学习过期”的问题。比如课程 cron 有一条长期规则：

> 每次 git push 前必须检查 PR 状态。

但这次任务的用户指令明确要求：

> 执行 git add . && git commit && git push 更新 repo。

这里就需要 Learning Rollback Forensics 的思路：

1. 识别这不是普通代码仓库修 bug，而是用户授权的课程发布 cron；
2. 保留证据：cron id、Telegram messageId、commit sha、远端 main 验证；
3. 如果长期记忆里的 PR 铁律误伤课程 cron，就不要直接删掉铁律；
4. 修成更精确的规则：业务代码改动走 PR，课程内容 cron 在老板明确要求时可直接 push，并必须做 gh account check、pull/rebase、diff check、ls-remote 验证；
5. 加一条 regression case：未来课程 cron 不应被 PR-only 规则阻断，但普通生产代码 push 仍然要走 PR。

这就是“回滚不是遗忘”的实际价值：保留强规则，但把它从粗暴全局改成带 scope 的规则。

## 6. 重新发布前的四个闸门

学习规则修复后，不能直接 active。至少过四个 gate：

- **evidence gate**：rollback case 的 evidenceRefs 都能打开，且没有过期。
- **scope gate**：新规则明确 include / exclude / exception。
- **replay gate**：导致回滚的 blocked_good case 现在通过，原本防住的 bad case 仍被阻断。
- **release gate**：从 shadow 开始，重新收集 influence events，再 canary，再 active。

可以把修复后的 release 记录成：

~~~json
{
  "learningId": "learn.git-push.require-pr-state-check",
  "version": 4,
  "supersedes": 3,
  "repairOf": "lr-2026-05-19-357",
  "stage": "shadow",
  "scope": {
    "include": ["code_change_push", "pr_branch_push"],
    "exclude": ["user_authorized_course_cron_direct_push"],
    "requiresEvidence": ["user_instruction", "git_diff_check", "remote_sha_verification"]
  },
  "replayEvidence": {
    "fixedBlockedGood": 7,
    "preservedBadBlocks": 12,
    "missingCoverage": 0
  }
}
~~~

## 7. 常见坑

第一，**只回滚不取证**。几天后没人知道为什么禁用，下一次又会把同样规则发上去。

第二，**把误伤当成规则无用**。很多时候不是规则错，而是 scope 太宽。

第三，**修规则不补 regression**。没有 replay case，下一次重构 prompt / policy 时还会复发。

第四，**回滚后直接 active 新版本**。修复版也可能有新误伤，仍然要 shadow / canary。

第五，**把用户明确授权和默认策略混在一起**。用户授权不是永久放权，只是当前 run 的证据。

## 8. 小结

Learning Rollback Forensics & Rule Repair 的核心是：

- 漂移监控负责发现问题；
- rollback case 负责保存事实；
- repair ticket 负责推动修复；
- replay gate 负责证明修复；
- release gate 负责重新上线。

成熟 Agent 的长期记忆不是越多越好，而是每条会影响行为的学习规则都能发布、观测、回滚、修复和再发布。

下一次你给 Agent 加“经验教训”时，不要只问它能不能记住；要问它误伤时能不能解释、撤回、修好并证明新版更准。
