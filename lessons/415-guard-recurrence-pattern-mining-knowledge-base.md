# 415. Agent 护栏复发处置后的模式挖掘与知识库（Guard Recurrence Pattern Mining & Knowledge Base）

上一课讲了 **Guard Finding Recurrence Dedupe & Escalation Budget**：复发信号命中后，不要每条都开 ticket，也不要把重复失败静音，而是按 dedupe window 合并证据，并用 escalation budget 判断何时升级。

今天继续往后一层：**复发 case 处理完以后，不要只关闭它；要把它沉淀成下次能自动识别、自动路由、自动补 replay 的 Failure Pattern。**

一句话：**Guard recurrence closeout 应该产出 Recurrence Pattern，进入 Pattern Knowledge Base；后续新 audit finding、replay fail、runtime drift 或 canary signal 先匹配 pattern，再决定 auto_route、extend_replay、tighten_guard、require_human 或 retire_pattern。**

---

## 1. 为什么 closeout 之后还要做 pattern mining

很多团队处理复发事件的流程是：

~~~text
detect recurrence -> reopen case -> patch -> replay -> close
~~~

这只能解决这一次。

成熟 Agent 系统更需要的是：

~~~text
detect recurrence -> reopen case -> patch -> replay -> close
                 -> extract pattern -> update knowledge base
                 -> next similar failure routes faster
~~~

因为复发问题往往不是完全相同的 bug，而是同一类失效模式的不同表现：

1. guard 证据过期，但不同工具返回字段不一样；
2. prompt policy 没覆盖某个副作用组合；
3. replay case 有覆盖，但缺少相邻 tenant 或 runtime group；
4. canary 没炸，active 才炸，因为采样窗口太短；
5. 修复过一次 false_allow，下一次变成 evidence_overread。

如果没有 pattern knowledge base，每次都像第一次遇到一样排查，Agent 的修复能力不会积累。

---

## 2. Recurrence Pattern 的最小字段

一个可用的 pattern 不需要很复杂，但必须能被机器匹配、路由和退役。

~~~text
RecurrencePattern:
  patternId              稳定 ID
  sourceCaseIds          来自哪些 reopen case / incident
  riskSurface            影响的风险面
  failureMode            false_allow / false_block / evidence_overread / policy_stale ...
  signaturePredicates    可匹配的结构化条件
  recommendedAction      route / extend_replay / tighten_guard / human_review
  requiredReplayTags     以后必须补跑的 replay tags
  confidence             这个 pattern 的可信度
  expiresAt              过期时间，防止旧经验永久误导
  evidenceRefs           支撑证据
~~~

关键点：**pattern 不是复盘文字，而是能参与未来决策的结构化知识。**

---

## 3. learn-claude-code：纯函数 Pattern 提取器

教学版先把 closeout bundle 转成 pattern candidate，再过质量闸门。

~~~py
from dataclasses import dataclass
from typing import Literal


FailureMode = Literal[
    "false_allow",
    "false_block",
    "evidence_overread",
    "policy_stale",
    "outcome_mismatch",
]

Action = Literal[
    "auto_route_repair",
    "extend_replay_matrix",
    "tighten_guard_scope",
    "require_human_review",
    "do_not_promote",
]


@dataclass(frozen=True)
class ReopenCloseout:
    case_id: str
    risk_surface: str
    failure_mode: FailureMode
    root_cause: str
    repeated_count: int
    affected_runtime_groups: tuple[str, ...]
    missing_replay_tags: tuple[str, ...]
    fixed_by_guard_ids: tuple[str, ...]
    evidence_refs: tuple[str, ...]
    false_positive_count: int


@dataclass(frozen=True)
class PatternCandidate:
    pattern_id: str
    source_case_ids: tuple[str, ...]
    risk_surface: str
    failure_mode: FailureMode
    predicates: tuple[str, ...]
    recommended_action: Action
    required_replay_tags: tuple[str, ...]
    confidence: float
    expires_in_days: int
    evidence_refs: tuple[str, ...]


def stable_pattern_id(closeout: ReopenCloseout) -> str:
    normalized_root = closeout.root_cause.lower().replace(" ", "-")[:48]
    return f"pattern:{closeout.risk_surface}:{closeout.failure_mode}:{normalized_root}"


def recommend_action(closeout: ReopenCloseout) -> Action:
    if closeout.false_positive_count > 2:
        return "require_human_review"
    if closeout.missing_replay_tags:
        return "extend_replay_matrix"
    if closeout.failure_mode in ("false_allow", "evidence_overread"):
        return "tighten_guard_scope"
    if closeout.repeated_count >= 3:
        return "auto_route_repair"
    return "do_not_promote"


def confidence(closeout: ReopenCloseout) -> float:
    score = 0.35
    score += min(closeout.repeated_count, 5) * 0.08
    score += min(len(closeout.evidence_refs), 5) * 0.05
    score += min(len(closeout.affected_runtime_groups), 3) * 0.04
    score -= closeout.false_positive_count * 0.12
    return max(0.0, min(score, 0.95))


def extract_pattern(closeout: ReopenCloseout) -> PatternCandidate:
    predicates = (
        f"risk_surface == {closeout.risk_surface}",
        f"failure_mode == {closeout.failure_mode}",
        f"root_cause contains {closeout.root_cause}",
    )

    return PatternCandidate(
        pattern_id=stable_pattern_id(closeout),
        source_case_ids=(closeout.case_id,),
        risk_surface=closeout.risk_surface,
        failure_mode=closeout.failure_mode,
        predicates=predicates,
        recommended_action=recommend_action(closeout),
        required_replay_tags=closeout.missing_replay_tags,
        confidence=confidence(closeout),
        expires_in_days=90,
        evidence_refs=closeout.evidence_refs,
    )


def quality_gate(candidate: PatternCandidate) -> tuple[bool, str]:
    if candidate.confidence < 0.55:
        return False, "confidence too low; keep as case note only"
    if not candidate.evidence_refs:
        return False, "pattern needs evidence refs"
    if len(candidate.predicates) < 3:
        return False, "pattern predicates are too vague"
    if candidate.recommended_action == "do_not_promote":
        return False, "no actionable future behavior"
    return True, "promote to pattern knowledge base"
~~~

这里的质量闸门很重要：**不要把每次复盘都写成规则。低置信度经验只适合留在 case note，不能进入自动决策链。**

---

## 4. pi-mono：PatternKnowledgeBase Worker

生产版建议把 pattern mining 放在 closeout 之后的异步 worker，不要阻塞修复主流程。

~~~ts
type GuardRecurrenceClosedEvent = {
  type: "guard_recurrence.closed";
  caseId: string;
  riskSurface: string;
  failureMode:
    | "false_allow"
    | "false_block"
    | "evidence_overread"
    | "policy_stale"
    | "outcome_mismatch";
  rootCause: string;
  repeatedCount: number;
  affectedRuntimeGroups: string[];
  missingReplayTags: string[];
  fixedByGuardIds: string[];
  evidenceRefs: string[];
  falsePositiveCount: number;
};

type PatternPromotedEvent = {
  type: "guard_pattern.promoted";
  patternId: string;
  sourceCaseId: string;
  recommendedAction: string;
  requiredReplayTags: string[];
  confidence: number;
};

class PatternKnowledgeBase {
  constructor(
    private readonly store: PatternStore,
    private readonly eventBus: EventBus,
  ) {}

  async onRecurrenceClosed(event: GuardRecurrenceClosedEvent) {
    const candidate = extractPatternCandidate(event);
    const gate = evaluatePatternQuality(candidate);

    if (!gate.promote) {
      await this.store.appendCaseNote(event.caseId, {
        kind: "pattern_candidate_rejected",
        reason: gate.reason,
        candidate,
      });
      return;
    }

    const existing = await this.store.findByPatternId(candidate.patternId);
    const pattern = existing
      ? mergePatternEvidence(existing, candidate)
      : candidate;

    await this.store.upsert(pattern);

    await this.eventBus.publish<PatternPromotedEvent>({
      type: "guard_pattern.promoted",
      patternId: pattern.patternId,
      sourceCaseId: event.caseId,
      recommendedAction: pattern.recommendedAction,
      requiredReplayTags: pattern.requiredReplayTags,
      confidence: pattern.confidence,
    });
  }
}
~~~

注意两个细节：

1. **findByPatternId + mergePatternEvidence**：同类 case 要增强同一个 pattern，而不是创建一堆相似规则。
2. **appendCaseNote**：被拒绝的 candidate 也要留痕，方便以后人工复盘为什么没晋级。

---

## 5. 运行时怎么使用 Pattern Knowledge Base

Pattern KB 不是文档库，它应该挂在四个入口上：

~~~text
Audit finding triage:
  新 finding 先匹配 pattern，命中后自动补 owner、SLO、repair path。

Replay selection:
  改到某个 risk surface 时，把 pattern.requiredReplayTags 加进 replay set。

Guard pack release:
  发布前检查 active patterns，命中高置信 pattern 但缺 replay 证据时 block promotion。

Human escalation:
  给 owner 展示“这像哪类旧问题、上次怎么修、哪些证据可信”。
~~~

最实用的匹配策略是两层：

~~~text
第一层：结构化 predicate 精确匹配，便宜、稳定、可解释；
第二层：语义相似度只做候选召回，最终仍要通过 predicate / evidence gate。
~~~

不要让向量相似度直接决定 block 或 allow。它适合找线索，不适合做安全决策。

---

## 6. OpenClaw 课程 Cron 实战映射

这套模式放到本课程 cron，也很好理解：

~~~text
Recurring issue:
  课程选题重复、README 目录漏更、TOOLS.md 已讲内容漏写、Telegram 发了但 git 没推。

Pattern:
  riskSurface = course_publish_side_effect
  failureMode = outcome_mismatch
  predicates = [
    "lesson file exists",
    "README lacks same lesson id",
    "TOOLS lacks same topic",
    "remote main does not contain commit"
  ]

Recommended action:
  before final response, run diff/check/status/ls-remote gate
~~~

这就是把一次 cron 事故变成以后每次发布前的自动检查。

在 Always-on Agent 里，长期可靠不是靠“我下次小心点”，而是靠：

~~~text
事故 -> case -> replay -> pattern -> gate -> telemetry -> retirement
~~~

---

## 7. 常见坑

**坑 1：把 pattern 写成散文**

散文只能给人看，不能给系统执行。至少要有 predicates、action、confidence、expiry。

**坑 2：pattern 永不过期**

旧策略、旧工具、旧 prompt 下的经验可能会误导新系统。每条 pattern 都要有 expiresAt 或 effectiveness telemetry。

**坑 3：命中 pattern 就自动修**

高置信 pattern 可以自动路由，但不一定能自动修改。涉及权限、外部副作用、跨模块写入时仍然要过 release gate 或 human review。

**坑 4：只记录成功修复的 pattern**

失败修复也很有价值。它可以告诉系统：这类问题不要走某条 repair path，直接升级人工或扩 replay。

---

## 8. 小结

今天的核心：

~~~text
Guard recurrence closeout 不应该只关闭 case；
它应该抽取 Recurrence Pattern；
Pattern 进入 Knowledge Base；
未来 finding、replay、release、escalation 都先查 pattern；
pattern 必须有质量闸门、证据、置信度和退役机制。
~~~

成熟 Agent 的学习不是“记住一段复盘”，而是把复发事件沉淀成下次能自动识别、自动补证据、自动少走弯路的工程资产。
