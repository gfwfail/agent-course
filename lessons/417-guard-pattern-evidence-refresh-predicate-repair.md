# 417. Agent 护栏模式证据刷新与谓词修复（Guard Pattern Evidence Refresh & Predicate Repair）

上一课讲了 **Guard Pattern Effectiveness Telemetry & Retirement Gate**：Pattern KB 入库以后，要靠 PatternMatch / PatternOutcome 证明它是否仍然有效，没用就收窄、观察或退役。

今天补上退役前最容易漏掉的一步：**pattern 变差时，不要马上删除；先判断是不是证据过期、谓词漂移或依赖迁移导致的可修复退化。**

一句话：**Guard Pattern KB 发现 stale evidence / predicate drift 后，要先进入受控刷新和谓词修复流程：刷新证据、重放历史命中、比较新旧 predicate，再决定 promote_repaired、keep_observe、tighten_scope、retire_pattern 或 manual_review；成熟 Agent 不把旧经验一删了之，而是能证明它该修、该降级，还是该退场。**

---

## 1. 为什么不能直接 retire

Pattern lifecycle worker 发现一个 pattern 误伤变多，常见原因不一定是“经验错了”。更常见的是：证据引用还指向旧 schema；工具输出字段改名；事故 closeout 后补了新的 regression case，但 evidenceRefs 没同步；risk surface 拆分后旧 scope 太宽；recommendedAction 仍然有效，但触发条件需要从字符串匹配升级为结构化谓词。

如果直接 retire，会丢掉真正有价值的历史经验；如果继续 active，又会把旧经验带进新决策。所以要加一层 **Pattern Repair Gate**：先修证据和谓词，修完用 replay 证明没有变坏，再决定是否回到 active。

---

## 2. Pattern Repair 的输入和输出

最小输入不是一段复盘文字，而是一份结构化 repair case。

~~~text
PatternRepairCase:
  patternId
  patternVersion
  trigger: stale_evidence | predicate_drift | schema_migration | scope_too_broad | outcome_regression
  currentLifecycleState: active | observe | demoted | retired_candidate
  staleEvidenceRefs
  changedDependencies:
    toolSchemas
    guardPackVersions
    replayCaseTags
    riskSurfaces
  recentFalseMatches
  recentHelpfulMatches
  requiredReplayWindow

PatternRepairDecision:
  promote_repaired     修复后可回 active
  keep_observe         修复不够确定，只观察不阻断
  tighten_scope        pattern 有用，但必须缩小 risk surface
  retire_pattern       风险面消失或 replay 证明无价值
  manual_review        证据不足或影响高风险副作用
~~~

关键点：**repair decision 不是编辑文件后的感觉，而是 evidence refresh + predicate replay 的结果。**

---

## 3. learn-claude-code：纯函数刷新闸门

教学版先把刷新判断写成纯函数。它只决定能不能自动修，不做外部副作用。

~~~py
from dataclasses import dataclass
from typing import Literal

RepairAction = Literal[
    "refresh_evidence",
    "repair_predicate",
    "tighten_scope",
    "keep_observe",
    "retire_pattern",
    "manual_review",
]

@dataclass(frozen=True)
class PatternRepairSignal:
    stale_evidence_count: int
    false_match_count: int
    helpful_match_count: int
    harmful_action_count: int
    schema_changed: bool
    risk_surface_removed: bool
    risk_surface_split: bool
    predicate_dep_changed: bool
    has_replay_fixtures: bool
    affects_external_side_effects: bool

def decide_repair_action(signal: PatternRepairSignal) -> tuple[RepairAction, list[str]]:
    if signal.risk_surface_removed:
        return "retire_pattern", ["risk_surface_removed"]
    if signal.affects_external_side_effects and not signal.has_replay_fixtures:
        return "manual_review", ["side_effect_pattern_without_replay_fixtures"]
    if signal.harmful_action_count > 0:
        return "keep_observe", ["harmful_action_seen_demote_before_repair"]
    if signal.stale_evidence_count >= 3 and not signal.predicate_dep_changed:
        return "refresh_evidence", ["multiple_stale_evidence_refs"]
    if signal.schema_changed or signal.predicate_dep_changed:
        if signal.has_replay_fixtures:
            return "repair_predicate", ["predicate_dependency_changed_with_replay"]
        return "manual_review", ["predicate_dependency_changed_without_replay"]
    if signal.risk_surface_split:
        return "tighten_scope", ["risk_surface_split"]
    if signal.false_match_count > signal.helpful_match_count * 2:
        return "tighten_scope", ["false_matches_dominate_helpful_matches"]
    return "keep_observe", ["insufficient_signal_for_active_repair"]
~~~

这个函数有两个生产里很重要的约束：高风险外部副作用没有 replay fixture，不能自动修；出现 harmful action 后，先降到 observe，再修 predicate，不能边阻断线上边试修。

---

## 4. learn-claude-code：比较新旧 Predicate

修 predicate 不是“让它少误匹配”这么简单。要同时检查三类样本：过去 true positive 仍然命中；过去 false positive 不再命中；新增边界样本不会误触发高风险动作。

~~~py
from dataclasses import dataclass
from typing import Callable, Literal

@dataclass(frozen=True)
class ReplaySample:
    sample_id: str
    expected: bool
    risk: Literal["low", "medium", "high"]
    payload: dict

@dataclass(frozen=True)
class PredicateReplayReport:
    passed: bool
    kept_true_positives: int
    removed_false_positives: int
    new_false_positives: list[str]
    missed_true_positives: list[str]
    requires_manual_review: bool

def replay_predicate_change(
    old_predicate: Callable[[dict], bool],
    new_predicate: Callable[[dict], bool],
    samples: list[ReplaySample],
) -> PredicateReplayReport:
    kept_true_positives = 0
    removed_false_positives = 0
    new_false_positives: list[str] = []
    missed_true_positives: list[str] = []

    for sample in samples:
        old_match = old_predicate(sample.payload)
        new_match = new_predicate(sample.payload)
        if sample.expected and old_match and new_match:
            kept_true_positives += 1
        if not sample.expected and old_match and not new_match:
            removed_false_positives += 1
        if not sample.expected and new_match:
            new_false_positives.append(sample.sample_id)
        if sample.expected and not new_match:
            missed_true_positives.append(sample.sample_id)

    high_risk_new_false_positive = any(
        s.sample_id in new_false_positives and s.risk == "high" for s in samples
    )
    passed = not new_false_positives and not missed_true_positives and kept_true_positives > 0
    return PredicateReplayReport(
        passed=passed,
        kept_true_positives=kept_true_positives,
        removed_false_positives=removed_false_positives,
        new_false_positives=new_false_positives,
        missed_true_positives=missed_true_positives,
        requires_manual_review=high_risk_new_false_positive,
    )
~~~

这里的标准比普通测试更严格：**新 predicate 不能制造新的 false positive，尤其不能在 high risk 样本上误触发。**

---

## 5. pi-mono：PatternRepairCase 事件

生产版不要让 lifecycle worker 直接改 pattern。它应该创建 repair case，由 repair worker 消费。

~~~ts
type PatternRepairTrigger =
  | "stale_evidence"
  | "predicate_drift"
  | "schema_migration"
  | "scope_too_broad"
  | "outcome_regression";

type PatternRepairCaseOpened = {
  type: "guard_pattern.repair_case_opened";
  caseId: string;
  patternId: string;
  patternVersion: number;
  trigger: PatternRepairTrigger;
  currentLifecycleState: "active" | "observe" | "demoted" | "retired_candidate";
  staleEvidenceRefs: string[];
  changedDependencies: {
    toolSchemas: string[];
    guardPackVersions: string[];
    replayCaseTags: string[];
    riskSurfaces: string[];
  };
  recentFalseMatchEventIds: string[];
  recentHelpfulMatchEventIds: string[];
  requiredReplayWindow: { startedAt: Date; endedAt: Date };
  openedAt: Date;
};

type PatternRepairReceipt = {
  type: "guard_pattern.repair_receipt_recorded";
  receiptId: string;
  caseId: string;
  patternId: string;
  oldVersion: number;
  newVersion?: number;
  decision:
    | "promote_repaired"
    | "keep_observe"
    | "tighten_scope"
    | "retire_pattern"
    | "manual_review";
  refreshedEvidenceRefs: string[];
  replayReportRef?: string;
  reasons: string[];
  recordedAt: Date;
};
~~~

注意 `oldVersion` 和 `newVersion`：pattern 修复要走版本化，不要原地覆盖。否则后面做 outcome 对比时，会分不清到底是哪版 predicate 造成误伤。

---

## 6. pi-mono：PatternRepairWorker

Repair worker 的职责是把“退化信号”转成“可验证的新版本或退役决定”。

~~~ts
type PatternRepairResult = {
  decision: PatternRepairReceipt["decision"];
  reasons: string[];
  refreshedEvidenceRefs: string[];
  replayReportRef?: string;
  newVersion?: number;
};

class PatternRepairWorker {
  constructor(
    private readonly patterns: PatternStore,
    private readonly evidence: EvidenceStore,
    private readonly replay: PatternReplayService,
    private readonly eventBus: EventBus,
  ) {}

  async handleRepairCase(event: PatternRepairCaseOpened) {
    const pattern = await this.patterns.getVersion(event.patternId, event.patternVersion);
    const refreshed = await this.evidence.refreshRefs({
      refs: pattern.evidenceRefs,
      changedDependencies: event.changedDependencies,
    });

    if (refreshed.blockedRefs.length > 0) {
      await this.recordReceipt(event, {
        decision: "manual_review",
        reasons: ["evidence_refresh_blocked"],
        refreshedEvidenceRefs: refreshed.validRefs,
      });
      return;
    }

    const candidate = await this.patterns.buildRepairCandidate({
      pattern,
      refreshedEvidenceRefs: refreshed.validRefs,
      changedDependencies: event.changedDependencies,
    });

    const replayReport = await this.replay.comparePredicateVersions({
      oldPredicate: pattern.predicate,
      newPredicate: candidate.predicate,
      window: event.requiredReplayWindow,
      includeFalseMatches: event.recentFalseMatchEventIds,
      includeHelpfulMatches: event.recentHelpfulMatchEventIds,
    });

    if (replayReport.highRiskNewFalsePositiveCount > 0) {
      await this.recordReceipt(event, {
        decision: "manual_review",
        reasons: ["high_risk_new_false_positive"],
        refreshedEvidenceRefs: refreshed.validRefs,
        replayReportRef: replayReport.reportRef,
      });
      return;
    }

    if (replayReport.passed) {
      const newVersion = await this.patterns.publishRepairedVersion(candidate);
      await this.recordReceipt(event, {
        decision: "promote_repaired",
        reasons: ["predicate_replay_passed"],
        refreshedEvidenceRefs: refreshed.validRefs,
        replayReportRef: replayReport.reportRef,
        newVersion: newVersion.version,
      });
      return;
    }

    await this.recordReceipt(event, {
      decision: replayReport.keptHelpfulMatches ? "keep_observe" : "retire_pattern",
      reasons: replayReport.reasons,
      refreshedEvidenceRefs: refreshed.validRefs,
      replayReportRef: replayReport.reportRef,
    });
  }

  private async recordReceipt(repairCase: PatternRepairCaseOpened, result: PatternRepairResult) {
    await this.eventBus.publish({
      type: "guard_pattern.repair_receipt_recorded",
      receiptId: crypto.randomUUID(),
      caseId: repairCase.caseId,
      patternId: repairCase.patternId,
      oldVersion: repairCase.patternVersion,
      ...result,
      recordedAt: new Date(),
    });
  }
}
~~~

这个 worker 有一个很硬的边界：**证据刷新失败时，不允许靠猜测修 predicate。** 证据链不完整，就进入 manual_review。

---

## 7. OpenClaw 课程 Cron 的实战映射

这个课程 cron 其实也有 Pattern Repair 的影子。每 3 小时发布课程前，流程会检查 `TOOLS.md` 已讲内容、`README.md` 目录、远端 `main` 是否包含本次 commit、Telegram messageId 是否记录到 memory；如果中途失败，要用 memory 里的证据恢复，而不是凭印象重发。

把它抽象成 Pattern KB：

~~~text
pattern: avoid_duplicate_agent_course_topic
predicate:
  topic not in TOOLS.md taught list
  topic not in README lesson summaries
  latest lesson number + 1
evidenceRefs:
  TOOLS.md line with taught topic
  README.md new entry
  git commit sha
  Telegram messageId
repairTrigger:
  stale_evidence if README/TOOLS diverge
  predicate_drift if topic naming changes
  scope_too_broad if similar but not duplicate topics get blocked
~~~

如果哪天 README 已更新但 TOOLS.md 没更新，正确动作不是“以后不查 TOOLS.md”，而是打开 PatternRepairCase，刷新 README / TOOLS / memory 三份证据，用最近课程记录 replay 去重 predicate，修复目录或已讲列表，写 repair receipt，再继续下一课。

---

## 8. 常见坑

1. 只刷新 evidence，不重放 predicate：证据引用更新了，不代表匹配逻辑仍然正确。
2. 原地改 pattern：Pattern 必须 versioned，旧版本解释旧 outcome，新版本解释新 outcome。
3. 用 false positive 样本修到过窄：修复回放必须同时包含 helpful matches。
4. 高风险副作用自动修：会影响发消息、push、部署、付款、删改数据的 pattern，没有 replay fixture 就不要自动修。
5. 退役没有归档原因：retire 也要 receipt，后续才能知道旧 pattern 为什么退出热路径。

---

## 9. 落地清单

给你的 Agent 系统加 Pattern Repair，可以按这个顺序做：

1. 给每个 pattern 加 `patternId + version + evidenceRefs + predicateDeps`；
2. lifecycle worker 只产出 repair case，不直接改 pattern；
3. evidence refresh 要区分 validRefs / blockedRefs / migratedRefs；
4. predicate repair 必须 replay true positive、false positive、high risk boundary samples；
5. 修复通过后发布新 version，并保留 old version；
6. 所有结果写 PatternRepairReceipt；
7. 外部副作用相关 pattern 没有 replay fixture 时必须 manual_review。

---

## 10. 记住

Pattern KB 的维护不是“旧了就删”，而是三段式：

~~~text
发现退化 -> 刷新证据与修复谓词 -> 用 replay 决定晋级、观察、收窄或退役
~~~

成熟 Agent 的知识库，不是记住更多事故，而是能持续修复自己的记忆索引：证据新鲜、谓词可回放、版本可追责、退役有收据。
