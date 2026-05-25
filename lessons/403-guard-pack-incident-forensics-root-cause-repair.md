# 403. Agent 回归护栏包事故取证与根因修复（Guard Pack Incident Forensics & Root-Cause Repair）

上一课讲了 **Guard Pack Rollout Rollback & Safe Recovery**：Guard Pack 上线后发现误伤、漏挡或运行时漂移，不能只改回版本号；要退回 Last Known Good，并扫描影响窗口内已经发生的 decision。

今天继续往恢复闭环走一步：**回滚只能让系统回到可信版本，不能解释新 Guard Pack 为什么坏。**

一句话：**Guard Pack 事故必须生成 Forensics Case，把错误 decision、命中证据、规则 diff、运行时上下文和修复候选绑定起来；根因未分类、修复未回放、影响窗口未关闭之前，坏 guard 不能重新进入 canary。**

---

## 1. 为什么回滚以后还要做取证

Guard Pack v42 出问题后，很多团队会这样处理：

~~~text
1. 回滚到 v41
2. 改一行 guard 条件
3. 再发 v43
~~~

这很危险。

因为你可能只是修了表面现象：

- 误伤来自规则条件太宽；
- 漏挡来自证据字段缺失；
- manual_review 风暴来自优先级冲突；
- 不同 worker 结果不同来自 runtime context 漂移；
- latency 爆炸来自 guard 调用了慢查询；
- schema 迁移把旧字段默认成了 unsafe 值。

这些根因对应的修复完全不同。

成熟系统要把回滚后的问题变成一份 **GuardPackForensicsCase**：

- 哪些 decision 错了；
- 错误发生在什么 packVersion / rolloutId；
- 哪些 guardId 参与了裁决；
- 当时输入证据、runtime context、policy cache hash 是什么；
- 根因分类是什么；
- 修复候选要通过哪些 replay 和 shadow 验证。

没有取证 case，下一次发布只是换个版本号继续赌。

---

## 2. 最小模型：Forensics Case

先定义事故取证对象。重点不是把所有日志塞进去，而是把可以复盘和修复的事实结构化。

~~~ts
type GuardPackForensicsCase = {
  caseId: string;
  rolloutId: string;
  badPackVersion: number;
  lastKnownGoodVersion: number;
  openedAt: string;
  status: "open" | "root_cause_classified" | "repair_candidate_ready" | "closed";
  trigger: {
    kind:
      | "false_positive"
      | "false_negative"
      | "manual_review_storm"
      | "runtime_drift"
      | "latency_regression"
      | "unknown_spike";
    metricSnapshotId: string;
  };
  impactedDecisionIds: string[];
  suspectGuardIds: string[];
  evidenceBundleRefs: string[];
  runtimeSnapshotRefs: string[];
  rootCause?: RootCause;
  repairCandidate?: RepairCandidate;
};

type RootCause = {
  kind:
    | "scope_too_broad"
    | "scope_too_narrow"
    | "missing_evidence_field"
    | "bad_schema_migration"
    | "priority_conflict"
    | "runtime_context_drift"
    | "expensive_guard_path"
    | "insufficient_fixture";
  confidence: "low" | "medium" | "high";
  explanation: string;
  supportingRefs: string[];
};

type RepairCandidate = {
  candidatePackVersion: number;
  changedGuardIds: string[];
  requiredReplayTags: string[];
  requiredShadowWindow: {
    minDecisions: number;
    maxFalsePositiveRate: number;
    maxFalseNegativeCount: number;
    maxUnknownRate: number;
  };
};
~~~

这里有三个关键点：

1. **trigger 是现象，不是根因**：false positive 只说明错的表现，不说明为什么错；
2. **suspectGuardIds 不等于 changedGuardIds**：参与事故的 guard 很多，真正要改的可能只有一个；
3. **repair candidate 要带验证预算**：修复不是靠 code review 感觉通过，而是 replay + shadow window。

---

## 3. learn-claude-code：纯函数根因分类器

教学版可以从 decision diff 入手：同一批输入分别跑 bad pack 和 Last Known Good，比较差异。

~~~py
from dataclasses import dataclass
from typing import Literal


Decision = Literal["allow", "warn", "dry_run", "block", "manual_review"]


@dataclass(frozen=True)
class DecisionDiff:
    decision_id: str
    lkg_decision: Decision
    bad_decision: Decision
    changed_guard_ids: list[str]
    missing_fields: list[str]
    schema_migration_warnings: list[str]
    runtime_hash_changed: bool
    latency_ms_delta: int
    priority_conflicts: list[str]


def classify_guard_pack_root_cause(diff: DecisionDiff) -> dict:
    if diff.runtime_hash_changed:
        return {
            "kind": "runtime_context_drift",
            "confidence": "high",
            "reason": "same input produced different decision under different runtime hash",
            "supportingRefs": [diff.decision_id],
        }

    if diff.schema_migration_warnings:
        return {
            "kind": "bad_schema_migration",
            "confidence": "high",
            "reason": "schema migration warnings were present for impacted decision",
            "supportingRefs": diff.schema_migration_warnings,
        }

    if diff.missing_fields:
        return {
            "kind": "missing_evidence_field",
            "confidence": "medium",
            "reason": "guard evaluated without required evidence fields",
            "missingFields": diff.missing_fields,
        }

    if diff.priority_conflicts:
        return {
            "kind": "priority_conflict",
            "confidence": "medium",
            "reason": "multiple guards disagreed and arbitration changed final decision",
            "conflicts": diff.priority_conflicts,
        }

    if diff.latency_ms_delta > 200:
        return {
            "kind": "expensive_guard_path",
            "confidence": "medium",
            "reason": "guard pack introduced high latency path",
            "latencyDeltaMs": diff.latency_ms_delta,
        }

    if diff.lkg_decision in ("allow", "warn") and diff.bad_decision in ("block", "manual_review"):
        return {
            "kind": "scope_too_broad",
            "confidence": "medium",
            "reason": "new guard blocked actions accepted by last known good pack",
            "changedGuardIds": diff.changed_guard_ids,
        }

    if diff.lkg_decision in ("block", "manual_review") and diff.bad_decision in ("allow", "dry_run"):
        return {
            "kind": "scope_too_narrow",
            "confidence": "medium",
            "reason": "new guard allowed actions blocked by last known good pack",
            "changedGuardIds": diff.changed_guard_ids,
        }

    return {
        "kind": "insufficient_fixture",
        "confidence": "low",
        "reason": "decision changed but available fixture cannot explain why",
        "supportingRefs": [diff.decision_id],
    }
~~~

这段代码的重点不是“AI 自动破案”，而是把根因分类变成可重复执行的规则。

顺序很重要：同输入不同 runtime hash，先查运行时漂移；有 migration warning，先查 schema；缺字段，先补 evidence contract；arbitration 冲突，先查优先级；最后才看 scope 太宽或太窄。否则你会把 runtime 问题误修成 guard 条件问题。

---

## 4. 取证包：不要只保存最终 decision

Guard Pack 取证需要比普通 decision record 多几层信息。

~~~json
{
  "forensicsBundleId": "gpfb_403_20260525_1830",
  "decisionId": "gd_123",
  "rolloutId": "gpr_2026_05_25_1830",
  "badPackVersion": 42,
  "lastKnownGoodVersion": 41,
  "inputSnapshot": {
    "actionKind": "git_push",
    "target": "gfwfail/agent-course:main",
    "payloadHash": "sha256:...",
    "evidenceRefs": ["ev_lesson_403", "ev_readme_diff"]
  },
  "guardTrace": [
    {
      "guardId": "block_duplicate_topic",
      "decision": "block",
      "reason": "topic_similarity_above_threshold",
      "inputs": {"topic": "Guard Pack Forensics", "threshold": 0.82}
    },
    {
      "guardId": "require_git_status_clean",
      "decision": "allow",
      "reason": "no_untracked_sensitive_files"
    }
  ],
  "runtimeSnapshot": {
    "policyCacheHash": "sha256:...",
    "guardPackHash": "sha256:...",
    "toolSchemaHash": "sha256:...",
    "modelRoute": "gpt-5.x"
  },
  "lkgReplayDecision": "allow",
  "badDecision": "block"
}
~~~

真正能修 bug 的不是最终的 block，而是 guardTrace：哪个 guard 先改变判断、读取了哪些输入、阈值和 cache hash 是什么、LKG replay 和 bad pack replay 差在哪里。

如果只保存最后一句“blocked by policy”，后面只能靠猜。

---

## 5. pi-mono：Forensics Worker + Repair Gate

生产版可以在 Guard Pack rollback 后自动创建取证任务。

~~~ts
class GuardPackForensicsWorker {
  constructor(
    private readonly cases: GuardPackForensicsStore,
    private readonly decisions: GuardDecisionLedger,
    private readonly replay: GuardPackReplayRunner,
    private readonly classifier: RootCauseClassifier,
    private readonly events: EventBus,
  ) {}

  async openFromRollback(input: {
    rolloutId: string;
    badPackVersion: number;
    lastKnownGoodVersion: number;
    triggerKind: GuardPackForensicsCase["trigger"]["kind"];
  }): Promise<string> {
    const impacted = await this.decisions.findImpactedByRollout(input.rolloutId);

    const case_ = await this.cases.create({
      rolloutId: input.rolloutId,
      badPackVersion: input.badPackVersion,
      lastKnownGoodVersion: input.lastKnownGoodVersion,
      trigger: {
        kind: input.triggerKind,
        metricSnapshotId: "metrics:" + input.rolloutId,
      },
      impactedDecisionIds: impacted.map((record) => record.decisionId),
      suspectGuardIds: [...new Set(impacted.flatMap((record) => record.guardIds))],
    });

    this.events.emit("guard_pack.forensics_case_opened", {
      caseId: case_.caseId,
      rolloutId: input.rolloutId,
      impactedDecisionCount: impacted.length,
    });

    return case_.caseId;
  }

  async classify(caseId: string): Promise<void> {
    const case_ = await this.cases.get(caseId);
    const diffs = await this.replay.diffAgainstLastKnownGood({
      decisionIds: case_.impactedDecisionIds,
      badPackVersion: case_.badPackVersion,
      lastKnownGoodVersion: case_.lastKnownGoodVersion,
    });

    const rootCause = this.classifier.classify(diffs);

    await this.cases.recordRootCause(caseId, rootCause);
    this.events.emit("guard_pack.root_cause_classified", {
      caseId,
      rootCauseKind: rootCause.kind,
      confidence: rootCause.confidence,
    });
  }
}
~~~

然后用 Repair Gate 控制修复候选能不能重新进入发布链。

~~~ts
class GuardPackRepairGate {
  async evaluate(candidate: RepairCandidate, evidence: RepairEvidence) {
    if (!evidence.rootCauseClassified) {
      return { decision: "block", reason: "root_cause_not_classified" };
    }

    if (!evidence.changedGuardsMatchRootCause) {
      return { decision: "block", reason: "repair_does_not_match_root_cause" };
    }

    if (evidence.regressionReplay.failed.length > 0) {
      return {
        decision: "block",
        reason: "regression_replay_failed",
        failedCaseIds: evidence.regressionReplay.failed,
      };
    }

    if (evidence.shadowWindow.falseNegativeCount > candidate.requiredShadowWindow.maxFalseNegativeCount) {
      return { decision: "block", reason: "shadow_false_negative_budget_exceeded" };
    }

    if (evidence.shadowWindow.unknownRate > candidate.requiredShadowWindow.maxUnknownRate) {
      return { decision: "manual_review", reason: "shadow_unknown_budget_exceeded" };
    }

    return { decision: "canary", reason: "repair_candidate_verified" };
  }
}
~~~

原则很硬：未分类根因，不准发修复；修复内容和根因不匹配，不准发；replay 没过，不准发；shadow 有漏挡，不准发；unknown 太多，进人工复核。

这能防止“回滚后随手改一行又上线”。

---

## 6. OpenClaw 课程 Cron 实战

拿这个课程 Cron 当例子。一次发课至少经过这些 guard：

- 选题去重 guard：检查 TOOLS.md 已讲清单和 README；
- 文件写入 guard：限制写入 lessons/README/TOOLS；
- Telegram egress guard：确认发到 Rust 学习小组，不泄露私人记忆；
- Git side-effect guard：push 前检查 gh 账号、PR 状态、diff check；
- memory update guard：更新已讲内容，避免下次重复。

假设新 Guard Pack v42 把“Guard Pack Forensics”误判成第 402 课重复，因为都包含 rollback / recovery 关键词。

正确恢复流程不是简单放宽相似度阈值，而是：

1. rollback 到 v41，先让课程任务能继续；
2. 创建 forensics case，绑定被误挡的 decisionId；
3. replay v42 和 v41，保存 guardTrace；
4. 分类 root cause：scope_too_broad，原因是只看关键词，没有看 topic object；
5. 修复 guard：相似度必须同时比较 title、core claim、new concept；
6. 用 398-403 课做 regression replay，确认不会把 Guard Pack lifecycle、telemetry、rollback、forensics 混成同一课；
7. shadow 一个窗口，确认没有 false negative，再进入 canary。

如果反过来是漏挡，比如重复课程已经发到 Telegram：forensics case 要绑定 messageId、commit sha、lesson file；root cause 可能是 missing_evidence_field，因为 guard 没读 TOOLS.md 最新已讲内容；修复候选必须证明 egress 前能读到最新 memory / README / TOOLS；外部副作用还要走 release correction 或补偿流程。

重点：**Guard Pack 事故取证要连接真实副作用，不是只看策略文件 diff。**

---

## 7. 关闭收据：Forensics Closeout Receipt

取证结束后，要写关闭收据。它证明这次事故不是“没人管了”，而是已经被分类、修复、验证和归档。

~~~json
{
  "receiptType": "guard_pack_forensics_closeout",
  "caseId": "gpfc_2026_05_25_1830",
  "rolloutId": "gpr_2026_05_25_1830",
  "badPackVersion": 42,
  "repairedPackVersion": 43,
  "rootCause": {
    "kind": "scope_too_broad",
    "confidence": "high"
  },
  "verification": {
    "regressionReplay": "passed",
    "shadowWindow": {
      "decisions": 240,
      "falsePositiveRate": 0.004,
      "falseNegativeCount": 0,
      "unknownRate": 0.012
    }
  },
  "impactedDecisionsClosed": 17,
  "newRegressionPackRefs": ["reg_guard_topic_similarity_403"],
  "closedAt": "2026-05-25T08:30:00Z"
}
~~~

这个 receipt 后面会被三个地方使用：发布系统允许 repaired pack 进入 canary；回归系统新增 fixture，防止同类问题回来；审计系统解释为什么 v42 被回滚、v43 为什么可信。

---

## 8. 常见坑

### 坑 1：把 false positive 当根因

false positive 是结果，不是原因。真正原因可能是 scope、schema、priority、runtime drift 或缺证据。

### 坑 2：只看坏 guard，不看仲裁链

最终 block 可能是 arbitration 结果。单个 guard 的输出没错，但 priority / overrides 错了。

### 坑 3：修复候选没有 replay 旧事故窗口

只跑新单测不够。必须用事故窗口里的真实 decision replay。

### 坑 4：没有 shadow window 就直接 active

Guard Pack 修复本身也是高风险发布。至少要 shadow，再 canary。

### 坑 5：取证包里混入敏感 payload

Forensics Bundle 应该引用 evidenceRefs 和 payloadHash。需要 raw 时走授权读取，不要把 secret 永久塞进事故报告。

---

## 9. 落地检查清单

设计 Guard Pack Forensics 时，检查这些点：

- 每次 rollback 是否自动创建 forensics case；
- decision record 是否有 guardTrace，而不只是最终 decision；
- replay 是否能对比 bad pack 和 Last Known Good；
- root cause 是否是结构化枚举；
- repair candidate 是否绑定 root cause 和 changedGuardIds；
- canary 前是否跑事故窗口 regression replay；
- shadow window 是否有 false positive / false negative / unknown 预算；
- closeout receipt 是否记录 repairedPackVersion、验证结果和新增 regression refs。

---

## 10. 总结

Guard Pack 回滚让系统停止继续犯错；Guard Pack Forensics 让团队知道为什么犯错；Repair Gate 让修复带着证据重新上线。

成熟 Agent 的护栏不是“写规则、坏了回滚、再改规则”。成熟做法是：**每次护栏事故都变成可复盘的 case、可分类的 root cause、可验证的 repair candidate，以及下一次发布前必须回放的 regression pack。**
