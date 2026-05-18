# 353. Agent 学习冲突检测与优先级仲裁（Learning Conflict Resolution & Precedence）

上一课讲了 Learning Effectiveness：学习写入系统后，还要持续证明它真的影响了正确决策，否则就降权或退役。

今天讲写入前更容易被忽略的一关：**新学习不能直接覆盖旧记忆；它必须先和现有 Memory、Runbook、Policy 做冲突检测，再按优先级仲裁。**

成熟链路应该是：

~~~text
Incident / Repair Closeout
  -> Learning Candidate
  -> Quality Gate
  -> Conflict Detection
  -> Precedence Resolution
  -> Persist / Supersede / Shadow / Reject / Manual Review
  -> Evidence Bundle
~~~

核心原则：**Agent 的长期记忆不是 append-only 笔记本，而是会影响未来动作的行为配置。只要会改变行为，就必须有冲突仲裁。**

## 1. 为什么学习冲突很危险

最常见的坏味道是“新经验看起来正确，但范围过宽”。

比如一次事故复盘得到：

~~~json
{
  "claim": "遇到部署后 502 时先不要重启服务",
  "actionableRule": "before restart_service, require health_probe + deploy_event evidence"
}
~~~

这条经验本身没错。但系统里可能已经有旧规则：

~~~json
{
  "claim": "P0 支付链路不可用超过 3 分钟时允许值班 Agent 重启服务",
  "actionableRule": "allow restart_service when incident.severity == P0 and payment_down_minutes >= 3"
}
~~~

如果新学习被直接写进 MEMORY.md 或 Policy，就可能把 P0 紧急恢复也挡住。

学习冲突通常有四类：

- **action conflict**：一条说 allow，一条说 block / require_approval。
- **scope overlap**：两条规则都命中同一个 service/env/operationType。
- **freshness conflict**：旧规则过期但仍在生效，或新规则没有验证窗口。
- **authority conflict**：事故复盘、人工审批、代码策略、运行指标的可信度不同。

所以学习候选项必须从自然语言升级成结构化 claim，才能比较。

## 2. 学习项要先拆成可比较的 claim

不要只保存一段 Markdown：

~~~md
以后部署后 502 不要着急重启，先看健康检查。
~~~

应该保存成可仲裁的数据：

~~~json
{
  "id": "learn.deploy-502.require-health-before-restart",
  "source": {
    "type": "incident_review",
    "evidenceRefs": ["incident:inc-502-2026-05-19", "probe:health-7241"],
    "createdAt": "2026-05-19T23:30:00Z"
  },
  "scope": {
    "services": ["web-api"],
    "envs": ["production"],
    "operationTypes": ["incident_repair", "deploy_verify"]
  },
  "condition": "after_deploy == true and symptom == 502",
  "action": {
    "tool": "restart_service",
    "decision": "require_evidence",
    "requiredEvidence": ["health_probe", "deploy_event"]
  },
  "supersedes": [],
  "reviewAfter": "2026-06-19T00:00:00Z"
}
~~~

最关键的字段是 scope、condition、tool、decision。没有这些字段，系统只能做语义相似度，做不了可靠仲裁。

## 3. learn-claude-code：最小冲突仲裁器

教学版可以用纯函数实现：输入一个 candidate 和现有 rules，输出 persist / supersede / shadow / reject / manual_review。

~~~python
from dataclasses import dataclass, field
from enum import Enum
from datetime import datetime, timezone

class Decision(str, Enum):
    ALLOW = "allow"
    REQUIRE_EVIDENCE = "require_evidence"
    REQUIRE_APPROVAL = "require_approval"
    BLOCK = "block"

class Resolution(str, Enum):
    PERSIST = "persist"
    SUPERSEDE = "supersede"
    SHADOW = "shadow"
    REJECT = "reject"
    MANUAL_REVIEW = "manual_review"

DECISION_STRICTNESS = {
    Decision.ALLOW: 0,
    Decision.REQUIRE_EVIDENCE: 1,
    Decision.REQUIRE_APPROVAL: 2,
    Decision.BLOCK: 3,
}

SOURCE_AUTHORITY = {
    "runtime_metric": 4,
    "human_approval": 3,
    "policy": 3,
    "incident_review": 2,
    "agent_summary": 1,
}

@dataclass
class Scope:
    services: set[str]
    envs: set[str]
    operation_types: set[str]

@dataclass
class LearningRule:
    id: str
    source_type: str
    scope: Scope
    tool: str
    condition: str
    decision: Decision
    evidence_refs: list[str]
    created_at: datetime
    review_after: datetime | None = None
    supersedes: list[str] = field(default_factory=list)

@dataclass
class Conflict:
    existing_id: str
    reason: str
    strictness_delta: int
    authority_delta: int

@dataclass
class ResolutionDecision:
    action: Resolution
    reasons: list[str]
    conflicts: list[Conflict]

def overlaps(left: set[str], right: set[str]) -> bool:
    return "*" in left or "*" in right or bool(left & right)

def scope_overlaps(left: Scope, right: Scope) -> bool:
    return (
        overlaps(left.services, right.services)
        and overlaps(left.envs, right.envs)
        and overlaps(left.operation_types, right.operation_types)
    )

def find_conflicts(candidate: LearningRule, existing_rules: list[LearningRule]) -> list[Conflict]:
    conflicts: list[Conflict] = []
    for rule in existing_rules:
        if candidate.tool != rule.tool:
            continue
        if not scope_overlaps(candidate.scope, rule.scope):
            continue

        strictness_delta = (
            DECISION_STRICTNESS[candidate.decision]
            - DECISION_STRICTNESS[rule.decision]
        )
        authority_delta = (
            SOURCE_AUTHORITY.get(candidate.source_type, 0)
            - SOURCE_AUTHORITY.get(rule.source_type, 0)
        )

        if strictness_delta != 0:
            conflicts.append(
                Conflict(
                    existing_id=rule.id,
                    reason=f"same tool/scope but decision changes {rule.decision}->{candidate.decision}",
                    strictness_delta=strictness_delta,
                    authority_delta=authority_delta,
                )
            )
    return conflicts

def resolve_learning_candidate(
    candidate: LearningRule,
    existing_rules: list[LearningRule],
) -> ResolutionDecision:
    if not candidate.evidence_refs:
        return ResolutionDecision(
            Resolution.REJECT,
            ["candidate has no evidence refs"],
            [],
        )

    conflicts = find_conflicts(candidate, existing_rules)
    if not conflicts:
        return ResolutionDecision(
            Resolution.PERSIST,
            ["no overlapping decision conflicts"],
            [],
        )

    weakens_existing = any(conflict.strictness_delta < 0 for conflict in conflicts)
    lower_authority = any(conflict.authority_delta < 0 for conflict in conflicts)

    if weakens_existing and lower_authority:
        return ResolutionDecision(
            Resolution.MANUAL_REVIEW,
            ["candidate weakens a higher-authority existing rule"],
            conflicts,
        )

    if weakens_existing:
        return ResolutionDecision(
            Resolution.SHADOW,
            ["candidate relaxes an existing guardrail; run in shadow first"],
            conflicts,
        )

    if all(conflict.authority_delta >= 0 for conflict in conflicts):
        return ResolutionDecision(
            Resolution.SUPERSEDE,
            ["candidate is stricter or equal-authority for overlapping rules"],
            conflicts,
        )

    return ResolutionDecision(
        Resolution.MANUAL_REVIEW,
        ["conflict requires owner review"],
        conflicts,
    )

now = datetime(2026, 5, 19, tzinfo=timezone.utc)

existing = [
    LearningRule(
        id="policy.p0-payment-restart-allowed",
        source_type="policy",
        scope=Scope({"payment-api"}, {"production"}, {"incident_repair"}),
        tool="restart_service",
        condition="severity == P0 and payment_down_minutes >= 3",
        decision=Decision.ALLOW,
        evidence_refs=["policy:incident-response-v4"],
        created_at=now,
    )
]

candidate = LearningRule(
    id="learn.deploy-502.require-health-before-restart",
    source_type="incident_review",
    scope=Scope({"*"}, {"production"}, {"incident_repair"}),
    tool="restart_service",
    condition="after_deploy == true and symptom == 502",
    decision=Decision.REQUIRE_EVIDENCE,
    evidence_refs=["incident:inc-502-2026-05-19"],
    created_at=now,
)

print(resolve_learning_candidate(candidate, existing))
~~~

这个最小版本已经能抓住关键：

- 同 tool、同 scope 才需要仲裁。
- 更严格的学习通常可以 supersede 或进入 candidate。
- 放宽 guardrail 的学习必须 shadow 或人工复核。
- 低权威来源不能覆盖高权威策略。

## 4. pi-mono：把 Memory 加载升级成可解释的 Precedence Pack

pi-mono 的 packages/mom/src/agent.ts 里，getMemory 会读取 workspace-level MEMORY.md 和 channel-specific MEMORY.md，然后注入 system prompt。

现在的简化路径是：

~~~ts
function getMemory(channelDir: string): string {
  const parts: string[] = [];

  const workspaceMemoryPath = join(channelDir, "..", "MEMORY.md");
  if (existsSync(workspaceMemoryPath)) {
    const content = readFileSync(workspaceMemoryPath, "utf-8").trim();
    if (content) {
      parts.push("### Global Workspace Memory\n" + content);
    }
  }

  const channelMemoryPath = join(channelDir, "MEMORY.md");
  if (existsSync(channelMemoryPath)) {
    const content = readFileSync(channelMemoryPath, "utf-8").trim();
    if (content) {
      parts.push("### Channel-Specific Memory\n" + content);
    }
  }

  return parts.join("\n\n") || "(no working memory yet)";
}
~~~

这适合 MVP，但生产版应该多一层 precedence pack：

~~~ts
type MemorySource = "policy" | "workspace" | "channel" | "incident" | "session";

type MemoryClaim = {
  id: string;
  source: MemorySource;
  authority: number;
  scope: {
    channelId?: string;
    services?: string[];
    envs?: string[];
    operationTypes?: string[];
  };
  text: string;
  decisionImpact?: {
    tool: string;
    decision: "allow" | "require_evidence" | "require_approval" | "block";
  };
  supersedes?: string[];
  evidenceRefs: string[];
};

type PrecedencePack = {
  active: MemoryClaim[];
  shadow: MemoryClaim[];
  suppressed: Array<{
    claim: MemoryClaim;
    suppressedBy: string;
    reason: string;
  }>;
};

function buildPrecedencePack(claims: MemoryClaim[]): PrecedencePack {
  const active: MemoryClaim[] = [];
  const shadow: MemoryClaim[] = [];
  const suppressed: PrecedencePack["suppressed"] = [];

  const sorted = [...claims].sort((a, b) => b.authority - a.authority);
  const activeByTool = new Map<string, MemoryClaim>();

  for (const claim of sorted) {
    const impact = claim.decisionImpact;
    if (!impact) {
      active.push(claim);
      continue;
    }

    const key = impact.tool + ":" + (claim.scope.channelId ?? "*");
    const existing = activeByTool.get(key);

    if (!existing) {
      activeByTool.set(key, claim);
      active.push(claim);
      continue;
    }

    if (claim.supersedes?.includes(existing.id)) {
      suppressed.push({
        claim: existing,
        suppressedBy: claim.id,
        reason: "explicitly superseded",
      });
      activeByTool.set(key, claim);
      active.push(claim);
      continue;
    }

    shadow.push(claim);
    suppressed.push({
      claim,
      suppressedBy: existing.id,
      reason: "lower precedence overlapping decision impact",
    });
  }

  return { active, shadow, suppressed };
}
~~~

注入 prompt 时，只把 active claims 放进主上下文；shadow/suppressed 写进审计事件或调试区，不默认影响决策。

这样 Mom / pi-mono 的记忆系统就从“把两个 MEMORY.md 拼起来”升级成：

- workspace 规则和 channel 规则可以明确优先级；
- 旧事故经验可以被 supersede，但不会静默消失；
- 低权威 session 学习不会覆盖 policy；
- 未来每次回答都能解释“为什么用了这条记忆，为什么没用那条”。

## 5. OpenClaw 课程 Cron：先查已讲内容也是冲突仲裁

这个课程 Cron 本身就是一个小型学习仲裁场景。

每 3 小时要做三件事：

1. 读取 TOOLS.md 的“已讲内容”。
2. 选择一个没有重复的新主题。
3. 写 lesson、更新 README、发 Telegram、再把新主题写回 TOOLS.md。

如果不做冲突检测，就会重复讲同一个知识点；如果 TOOLS.md 和 README 不一致，就会产生目录和长期记忆漂移。

可以把这个流程结构化：

~~~json
{
  "candidateTopic": "Agent 学习冲突检测与优先级仲裁",
  "conflictChecks": [
    {
      "source": "TOOLS.md",
      "query": "学习冲突 / conflict / precedence",
      "result": "no exact taught topic"
    },
    {
      "source": "README.md",
      "query": "353 lesson slot",
      "result": "next lesson number available"
    }
  ],
  "decision": "persist",
  "evidenceRefs": [
    "agent-course/README.md",
    "workspace/TOOLS.md",
    "telegram:Rust学习小组"
  ]
}
~~~

这和生产 Agent 的学习系统是同一件事：**新知识进入长期系统前，先查旧知识、比较作用范围、记录为什么可以写入。**

## 6. 实战落地清单

给学习系统加冲突仲裁时，至少实现这 6 个点：

- Learning Candidate 必须有 scope / condition / decisionImpact / evidenceRefs。
- 每次写入前检索同 service/env/operationType/tool 的旧规则。
- 放宽 guardrail 的学习默认 shadow，不直接 active。
- 高权威来源可以覆盖低权威来源，但必须记录 supersedes。
- 被覆盖的旧知识不要删除，标记 suppressed/archived 并保留证据。
- 每次仲裁输出 ResolutionDecision，进入审计日志和回归包。

一句话总结：**会学习的 Agent 不只是会写记忆，而是知道新记忆和旧规则打架时该听谁的。**
