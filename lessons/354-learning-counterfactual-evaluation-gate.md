# 354. Agent 学习反事实评估闸门（Learning Counterfactual Evaluation Gate）

上一课讲了 Learning Conflict Resolution：新学习写入前，要先和 Memory、Runbook、Policy 做冲突检测，并按 authority、strictness、supersedes 仲裁。

今天继续往前补一关：**冲突检测通过，不代表学习就应该生效；它还要先拿历史真实任务做反事实评估，证明“如果当时这条学习存在，会让决策更好，而不是更坏”。**

成熟链路应该是：

~~~text
Learning Candidate
  -> Quality Gate
  -> Conflict Detection
  -> Counterfactual Evaluation
  -> Promotion Decision
  -> Shadow / Canary / Active / Reject
  -> Evidence Bundle
~~~

核心原则：**学习规则不是文档，它会改变 Agent 未来动作；上线前必须用历史决策样本跑一次“如果当时有它会怎样”。**

## 1. 为什么需要反事实评估

很多学习项看起来正确，但一上线就变成“过度泛化”。

比如事故复盘得到一条规则：

~~~json
{
  "id": "learn.deploy-502.require-health-before-restart",
  "rule": "部署后出现 502 时，重启服务前必须先跑 health_probe"
}
~~~

这条规则在事故现场是对的。但它可能误伤历史上这些场景：

- 502 来自上游 CDN 配置错误，health_probe 对本服务没有意义。
- P0 支付链路全断，值班策略允许 3 分钟后先重启再补证据。
- staging 环境的 502 是已知压测场景，不应该升级人工。

如果只做语义冲突检测，这些问题很难暴露。反事实评估要回答四个问题：

- 历史上哪些任务会命中这条学习？
- 命中后原决策会不会改变？
- 改变方向是更安全，还是过度阻断？
- 证据是否足够支持上线，还是只能 shadow 跟跑？

所以学习发布前要有一个 counterfactual gate。

## 2. 反事实样本要长什么样

不要拿纯聊天日志直接评估。样本要保存当时的结构化决策输入和真实结果：

~~~json
{
  "caseId": "case.deploy-502-2026-05-18-0930",
  "input": {
    "service": "web-api",
    "env": "production",
    "operationType": "incident_repair",
    "symptom": "502",
    "afterDeploy": true,
    "requestedTool": "restart_service",
    "evidence": ["deploy_event"]
  },
  "originalDecision": {
    "action": "allow",
    "risk": "high",
    "reasonCodes": ["operator_requested_restart"]
  },
  "outcome": {
    "incidentReopened": true,
    "manualCorrectionNeeded": true,
    "label": "bad_allow"
  }
}
~~~

关键字段有三类：

- **decision input**：service/env/operationType/symptom/tool/evidence。
- **original decision**：当时 allow、block、require_evidence、manual_review 的结果。
- **outcome label**：后来证明当时决策是 good_allow、bad_allow、good_block、false_positive_block 等。

没有 outcome label，也可以先做弱评估，但不能直接 active，只能进入 shadow。

## 3. learn-claude-code：最小反事实评估器

教学版可以用纯函数实现：输入 learning rule 和历史 cases，输出 reject / shadow / canary / active。

~~~python
from dataclasses import dataclass, field
from enum import Enum

class Decision(str, Enum):
    ALLOW = "allow"
    REQUIRE_EVIDENCE = "require_evidence"
    MANUAL_REVIEW = "manual_review"
    BLOCK = "block"

class OutcomeLabel(str, Enum):
    GOOD_ALLOW = "good_allow"
    BAD_ALLOW = "bad_allow"
    GOOD_BLOCK = "good_block"
    FALSE_POSITIVE_BLOCK = "false_positive_block"
    UNKNOWN = "unknown"

class Promotion(str, Enum):
    REJECT = "reject"
    SHADOW = "shadow"
    CANARY = "canary"
    ACTIVE = "active"

@dataclass
class DecisionInput:
    service: str
    env: str
    operation_type: str
    symptom: str
    after_deploy: bool
    requested_tool: str
    evidence: set[str] = field(default_factory=set)

@dataclass
class HistoricalCase:
    case_id: str
    input: DecisionInput
    original_decision: Decision
    outcome: OutcomeLabel

@dataclass
class LearningRule:
    id: str
    services: set[str]
    envs: set[str]
    operation_types: set[str]
    symptom: str
    tool: str
    required_evidence: set[str]
    decision_if_missing: Decision

@dataclass
class CounterfactualResult:
    case_id: str
    original_decision: Decision
    counterfactual_decision: Decision
    outcome: OutcomeLabel
    changed: bool
    effect: str

@dataclass
class EvaluationDecision:
    promotion: Promotion
    reasons: list[str]
    results: list[CounterfactualResult]

def matches(rule: LearningRule, item: DecisionInput) -> bool:
    return (
        item.service in rule.services
        and item.env in rule.envs
        and item.operation_type in rule.operation_types
        and item.symptom == rule.symptom
        and item.requested_tool == rule.tool
    )

def apply_rule(rule: LearningRule, item: DecisionInput, current: Decision) -> Decision:
    if not matches(rule, item):
        return current
    if rule.required_evidence.issubset(item.evidence):
        return current
    return rule.decision_if_missing

def classify_effect(case: HistoricalCase, new_decision: Decision) -> str:
    changed = new_decision != case.original_decision
    if not changed:
        return "unchanged"

    if case.outcome == OutcomeLabel.BAD_ALLOW and case.original_decision == Decision.ALLOW:
        if new_decision in {Decision.REQUIRE_EVIDENCE, Decision.MANUAL_REVIEW, Decision.BLOCK}:
            return "prevented_bad_allow"

    if case.outcome == OutcomeLabel.GOOD_ALLOW and case.original_decision == Decision.ALLOW:
        if new_decision in {Decision.MANUAL_REVIEW, Decision.BLOCK}:
            return "introduced_false_positive"

    if case.outcome == OutcomeLabel.FALSE_POSITIVE_BLOCK:
        if new_decision in {Decision.MANUAL_REVIEW, Decision.BLOCK}:
            return "reinforced_false_positive"

    return "needs_review"

def evaluate_learning_rule(
    rule: LearningRule,
    cases: list[HistoricalCase],
    min_hits: int = 5,
    max_false_positive_rate: float = 0.05,
) -> EvaluationDecision:
    results: list[CounterfactualResult] = []

    for case in cases:
        if not matches(rule, case.input):
            continue

        new_decision = apply_rule(rule, case.input, case.original_decision)
        results.append(
            CounterfactualResult(
                case_id=case.case_id,
                original_decision=case.original_decision,
                counterfactual_decision=new_decision,
                outcome=case.outcome,
                changed=new_decision != case.original_decision,
                effect=classify_effect(case, new_decision),
            )
        )

    hits = len(results)
    prevented = sum(1 for result in results if result.effect == "prevented_bad_allow")
    false_positive = sum(
        1 for result in results
        if result.effect in {"introduced_false_positive", "reinforced_false_positive"}
    )
    review_needed = sum(1 for result in results if result.effect == "needs_review")
    false_positive_rate = false_positive / hits if hits else 0.0

    if hits == 0:
        return EvaluationDecision(
            Promotion.SHADOW,
            ["no matching historical cases; collect shadow evidence first"],
            results,
        )

    if false_positive_rate > max_false_positive_rate:
        return EvaluationDecision(
            Promotion.REJECT,
            [f"false positive rate too high: {false_positive_rate:.2%}"],
            results,
        )

    if hits < min_hits or review_needed > 0:
        return EvaluationDecision(
            Promotion.SHADOW,
            [f"insufficient clean evidence: hits={hits}, review_needed={review_needed}"],
            results,
        )

    if prevented >= 2 and false_positive == 0:
        return EvaluationDecision(
            Promotion.CANARY,
            [f"prevented {prevented} historical bad allows with no false positives"],
            results,
        )

    return EvaluationDecision(
        Promotion.SHADOW,
        ["low demonstrated benefit; run in shadow mode"],
        results,
    )

rule = LearningRule(
    id="learn.deploy-502.require-health-before-restart",
    services={"web-api"},
    envs={"production"},
    operation_types={"incident_repair"},
    symptom="502",
    tool="restart_service",
    required_evidence={"health_probe", "deploy_event"},
    decision_if_missing=Decision.REQUIRE_EVIDENCE,
)
~~~

这段代码故意不接 LLM。评估闸门应该尽量可重复、可解释、可单测；LLM 可以帮忙提取 case label，但 gate 本身要是确定性的。

## 4. pi-mono：把评估接到事件流和中间件

生产版不要在主 Agent Loop 里临时翻历史日志。更好的做法是把决策事件先结构化沉淀，再由 LearningCounterfactualMiddleware 离线评估。

~~~ts
type DecisionAction = "allow" | "require_evidence" | "manual_review" | "block";
type OutcomeLabel =
  | "good_allow"
  | "bad_allow"
  | "good_block"
  | "false_positive_block"
  | "unknown";

interface DecisionEvent {
  caseId: string;
  service: string;
  env: string;
  operationType: string;
  symptom?: string;
  requestedTool: string;
  evidenceRefs: string[];
  decision: {
    action: DecisionAction;
    risk: "low" | "medium" | "high" | "critical";
    reasonCodes: string[];
  };
  outcome: {
    label: OutcomeLabel;
    source: "regression_replay" | "incident_review" | "manual_label" | "runtime_metric";
  };
}

interface LearningCandidate {
  id: string;
  scope: {
    services: string[];
    envs: string[];
    operationTypes: string[];
  };
  match: {
    symptom?: string;
    requestedTool: string;
  };
  requiredEvidence: string[];
  decisionIfMissing: DecisionAction;
}

interface CounterfactualSummary {
  candidateId: string;
  matchedCases: number;
  changedDecisions: number;
  preventedBadAllows: number;
  falsePositives: number;
  unknowns: number;
  promotion: "reject" | "shadow" | "canary" | "active";
  evidenceRefs: string[];
}

class LearningCounterfactualMiddleware {
  constructor(
    private readonly decisionStore: {
      findByScope(candidate: LearningCandidate): Promise<DecisionEvent[]>;
    },
    private readonly evidenceStore: {
      write(summary: CounterfactualSummary): Promise<string>;
    },
  ) {}

  async evaluate(candidate: LearningCandidate): Promise<CounterfactualSummary> {
    const events = await this.decisionStore.findByScope(candidate);
    let changedDecisions = 0;
    let preventedBadAllows = 0;
    let falsePositives = 0;
    let unknowns = 0;

    for (const event of events) {
      const next = this.applyCandidate(candidate, event);
      if (next === event.decision.action) continue;

      changedDecisions += 1;

      if (event.outcome.label === "bad_allow" && next !== "allow") {
        preventedBadAllows += 1;
      } else if (event.outcome.label === "good_allow" && next !== "allow") {
        falsePositives += 1;
      } else if (event.outcome.label === "unknown") {
        unknowns += 1;
      }
    }

    const falsePositiveRate =
      events.length > 0 ? falsePositives / events.length : 0;

    const promotion =
      events.length === 0 || unknowns > 0
        ? "shadow"
        : falsePositiveRate > 0.05
          ? "reject"
          : preventedBadAllows >= 2 && falsePositives === 0
            ? "canary"
            : "shadow";

    const summary: CounterfactualSummary = {
      candidateId: candidate.id,
      matchedCases: events.length,
      changedDecisions,
      preventedBadAllows,
      falsePositives,
      unknowns,
      promotion,
      evidenceRefs: [],
    };

    const evidenceRef = await this.evidenceStore.write(summary);
    return { ...summary, evidenceRefs: [evidenceRef] };
  }

  private applyCandidate(
    candidate: LearningCandidate,
    event: DecisionEvent,
  ): DecisionAction {
    const inScope =
      candidate.scope.services.includes(event.service) &&
      candidate.scope.envs.includes(event.env) &&
      candidate.scope.operationTypes.includes(event.operationType) &&
      candidate.match.requestedTool === event.requestedTool &&
      (!candidate.match.symptom || candidate.match.symptom === event.symptom);

    if (!inScope) return event.decision.action;

    const hasEvidence = candidate.requiredEvidence.every((required) =>
      event.evidenceRefs.some((ref) => ref.startsWith(required)),
    );

    return hasEvidence ? event.decision.action : candidate.decisionIfMissing;
  }
}
~~~

pi-mono 这种 EventStream / middleware 风格很适合做这件事：线上决策时记录事件，学习候选产生时离线 replay，不阻塞用户当前 turn。

## 5. OpenClaw：用历史任务做学习上线前证据

OpenClaw 里可以用更朴素的文件式证据包落地：

~~~text
memory/learning-candidates/
  learn.deploy-502.require-health-before-restart.json

memory/decision-events/
  2026-05-18.jsonl
  2026-05-19.jsonl

memory/learning-evals/
  learn.deploy-502.require-health-before-restart.eval.json
~~~

每次 cron 或 heartbeat 要写长期记忆、TOOLS.md、Runbook 时，先做四步：

1. 生成 LearningCandidate，带 scope、condition、action、requiredEvidence。
2. 从 decision-events / regression packs 里筛匹配样本。
3. 跑 counterfactual evaluator，产出 matchedCases、preventedBadAllows、falsePositives、unknowns。
4. 根据结果决定 reject、shadow、canary，或者 active。

对于课程 cron，这个模式也能直接用：

- 新主题写入 TOOLS.md 前，先检索已讲内容，这是质量和冲突检查。
- 如果主题会改变后续课程方向，拿最近课程目录反事实评估：是否会导致重复、跳跃、风险过高。
- 发布 Telegram 前，把 lesson 文件、README diff、TOOLS diff、messageId、commit hash 组成 evidence bundle。

这就是 Always-on Agent 的关键习惯：不是“我觉得这条经验不错”，而是“我能证明它在历史样本上不会把我带偏”。

## 6. 实战建议

落地时先不要追求复杂模型，先做三个硬规则：

- **没有历史样本**：只能 shadow，不能 active。
- **误伤率超过预算**：reject 或 manual_review。
- **unknown 太多**：补 label 或扩大回归包，不能拿未知当安全。

一个简单的 promotion policy：

~~~json
{
  "minMatchedCasesForCanary": 5,
  "minPreventedBadAllows": 2,
  "maxFalsePositiveRate": 0.05,
  "maxUnknownRate": 0.1,
  "activeRequiresCanarySuccesses": 20
}
~~~

这套门槛比“写进记忆就生效”麻烦一点，但能避免 Agent 学到一条看似正确、实际会扩大事故范围的规则。

## 7. 本课小结

学习反事实评估闸门解决的是：**新学习在上线前，先用历史真实决策证明它会改善行为。**

记住三句话：

- 学习项会改变未来动作，所以它是生产变更。
- 冲突检测只能发现“和现有规则打架”，反事实评估才能发现“会误伤历史场景”。
- 没有 matched cases、outcome labels 和 promotion evidence 的学习，只能 shadow，不能直接 active。

成熟 Agent 不只是会从事故里学习，还要在学习生效前证明：这条经验如果早一点存在，真的会让过去做得更好。
