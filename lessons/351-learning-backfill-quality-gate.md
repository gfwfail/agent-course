# 351. Agent 学习回灌质量闸门（Learning Backfill Quality Gate）

上一课讲了 Repair Closeout：自动修复完成后，要把 SLO 证据、回归用例、runbook patch 和 memory refs 一起归档。

今天补上一个很容易被忽略的安全点：**不是所有“学到的东西”都应该立刻写进长期记忆。学习回灌必须先经过质量闸门，确认它可执行、可验证、不过期、不和旧知识冲突，才允许进入 MEMORY / Runbook / Policy / Regression Pack。**

成熟链路应该是：

~~~text
Repair Closeout
  -> Learning Candidate
  -> Quality Gate
  -> Conflict Check
  -> Regression Link
  -> Write to Memory / Runbook / Policy
  -> Audit Event
~~~

核心原则：**Agent 的学习不是“把复盘总结塞进记忆”，而是把可验证的行为改进安全地发布到未来执行路径。**

## 1. 为什么学习回灌也需要闸门

很多 Agent 系统会在任务结束时自动总结经验：

~~~text
这次失败是因为没先检查部署状态。
下次部署后应该查看日志。
遇到 502 可以重启服务。
~~~

这些话听起来都对，但直接写进长期记忆会有风险：

- “重启服务”可能只适用于 canary，不适用于 production。
- “查看日志”太模糊，未来 Agent 不知道看哪个日志、用哪个工具。
- 一次事故的临时 workaround 可能被误当成通用规则。
- 新规则可能和现有 runbook 冲突。
- 没有绑定 regression case，未来无法证明这条学习真的防复发。

所以学习回灌要从 free-form note 变成结构化候选项：

~~~json
{
  "id": "learn.repair.deploy-502.check-health-before-restart",
  "sourceTicket": "repair-2026-05-19-001",
  "claim": "遇到部署后 502 时，先检查 health probe 和最近 deploy event，再决定是否重启服务",
  "scope": {
    "services": ["web-api"],
    "environments": ["staging", "production"],
    "operationTypes": ["incident_repair", "deploy_verify"]
  },
  "actionableRule": "before restart_service, require health_probe + deploy_event evidence",
  "evidenceRefs": ["slo-window-123", "probe-456", "commit-abc"],
  "regressionCases": ["incident.deploy_502_no_blind_restart"],
  "expiresAt": "2026-08-19T00:00:00Z",
  "risk": "high"
}
~~~

这条候选学习能回答四个问题：

- 从哪次真实事件来？
- 以后在哪些场景生效？
- 它具体改变什么行为？
- 有什么证据和测试证明它应该生效？

## 2. learn-claude-code：最小学习质量闸门

教学版先写成纯函数：输入 learning candidate 和当前知识库索引，输出 accept / revise / reject。

~~~python
from dataclasses import dataclass, field
from enum import Enum
from typing import Literal

Risk = Literal["low", "medium", "high"]

class LearningDecision(str, Enum):
    ACCEPT = "accept"
    REVISE = "revise"
    REJECT = "reject"

@dataclass
class LearningCandidate:
    id: str
    source_ticket: str
    claim: str
    scope: dict
    actionable_rule: str
    evidence_refs: list[str]
    regression_cases: list[str]
    risk: Risk
    expires_at: str | None = None
    tags: list[str] = field(default_factory=list)

@dataclass
class ExistingKnowledge:
    id: str
    scope: dict
    claim: str
    superseded: bool = False

@dataclass
class GateResult:
    decision: LearningDecision
    reasons: list[str]
    required_edits: list[str] = field(default_factory=list)

def overlaps_scope(a: dict, b: dict) -> bool:
    for key in ["services", "environments", "operationTypes"]:
        left = set(a.get(key, []))
        right = set(b.get(key, []))
        if left and right and left.isdisjoint(right):
            return False
    return True

def decide_learning_backfill(
    candidate: LearningCandidate,
    existing: list[ExistingKnowledge],
) -> GateResult:
    reasons: list[str] = []
    edits: list[str] = []

    if len(candidate.claim.strip()) < 20:
        edits.append("claim must explain the lesson, not just name the incident")

    if not candidate.scope.get("services"):
        edits.append("scope.services is required")

    if not candidate.scope.get("operationTypes"):
        edits.append("scope.operationTypes is required")

    if not candidate.actionable_rule:
        edits.append("actionable_rule is required")

    if candidate.risk in {"medium", "high"} and not candidate.evidence_refs:
        reasons.append("medium/high risk learning requires evidence refs")

    if candidate.risk == "high" and not candidate.regression_cases:
        reasons.append("high risk learning requires a regression case")

    if candidate.risk == "high" and not candidate.expires_at:
        edits.append("high risk learning needs review/expiry date")

    conflicts = [
        item.id for item in existing
        if not item.superseded and overlaps_scope(candidate.scope, item.scope)
        and item.claim != candidate.claim
    ]

    if conflicts:
        reasons.append("possible conflict with existing knowledge: " + ", ".join(conflicts))

    if reasons:
        return GateResult(LearningDecision.REJECT, reasons, edits)

    if edits:
        return GateResult(LearningDecision.REVISE, ["candidate is incomplete"], edits)

    return GateResult(LearningDecision.ACCEPT, ["learning is scoped, actionable, evidenced, and test-linked"])
~~~

这个 gate 有三个重点：

- scope 防止一条经验被错误泛化到所有服务。
- evidence_refs 防止“感觉上学到了”。
- regression_cases 把经验接进未来发布前的自动验证。

## 3. pi-mono：把学习候选接到中间件

生产系统里，学习回灌通常发生在 run 完成后。可以把它放在 Agent event stream 的后处理阶段，但写入长期存储前必须先过 gate。

~~~ts
type LearningRisk = 'low' | 'medium' | 'high';

type LearningCandidate = {
  id: string;
  sourceRunId: string;
  sourceTicketId?: string;
  claim: string;
  scope: {
    services?: string[];
    environments?: string[];
    operationTypes?: string[];
  };
  actionableRule: string;
  evidenceRefs: string[];
  regressionCases: string[];
  risk: LearningRisk;
  expiresAt?: string;
};

type LearningGateDecision =
  | { status: 'accepted'; candidate: LearningCandidate; audit: string[] }
  | { status: 'revise'; missing: string[] }
  | { status: 'rejected'; reasons: string[] };

function validateLearningCandidate(candidate: LearningCandidate): LearningGateDecision {
  const missing: string[] = [];
  const reasons: string[] = [];

  if (!candidate.scope.services?.length) {
    missing.push('scope.services');
  }

  if (!candidate.scope.operationTypes?.length) {
    missing.push('scope.operationTypes');
  }

  if (!candidate.actionableRule.trim()) {
    missing.push('actionableRule');
  }

  if (candidate.risk !== 'low' && candidate.evidenceRefs.length === 0) {
    reasons.push('risk requires evidenceRefs');
  }

  if (candidate.risk === 'high' && candidate.regressionCases.length === 0) {
    reasons.push('high risk learning requires regressionCases');
  }

  if (candidate.risk === 'high' && !candidate.expiresAt) {
    missing.push('expiresAt');
  }

  if (reasons.length > 0) {
    return { status: 'rejected', reasons };
  }

  if (missing.length > 0) {
    return { status: 'revise', missing };
  }

  return {
    status: 'accepted',
    candidate,
    audit: [
      'learning.scoped',
      'learning.actionable',
      'learning.evidenced',
      'learning.regression_linked',
    ],
  };
}

async function persistLearningIfAccepted(
  candidate: LearningCandidate,
  store: { write: (candidate: LearningCandidate) => Promise<void> },
  audit: { append: (event: unknown) => Promise<void> },
) {
  const decision = validateLearningCandidate(candidate);

  await audit.append({
    type: 'learning_backfill.gate_decision',
    candidateId: candidate.id,
    sourceRunId: candidate.sourceRunId,
    decision: decision.status,
    at: new Date().toISOString(),
  });

  if (decision.status !== 'accepted') {
    return decision;
  }

  await store.write(decision.candidate);
  await audit.append({
    type: 'learning_backfill.persisted',
    candidateId: candidate.id,
    evidenceRefs: candidate.evidenceRefs,
    regressionCases: candidate.regressionCases,
  });

  return decision;
}
~~~

这里不要让 LLM 直接写 MEMORY。LLM 可以提出 candidate，但持久化必须由 deterministic gate 决定。

## 4. OpenClaw：课程 cron 的学习回灌例子

这套课程 cron 每 3 小时运行一次，本身就需要学习回灌质量闸门。

一次成功发布后，可以产生候选学习：

~~~json
{
  "id": "learn.course.push.preflight-gh-account",
  "sourceRunId": "cron:3eba6ee3-2d11-4afa-aa7f-a47b63226982",
  "claim": "课程发布前必须确认 GitHub CLI 已切换到 gfwfail，否则 push 可能失败或推到错误身份",
  "scope": {
    "services": ["agent-course"],
    "operationTypes": ["git_push", "course_publish"]
  },
  "actionableRule": "before git push, run gh auth switch --user gfwfail and verify gh auth status",
  "evidenceRefs": ["git-status-clean", "remote-main-contains-commit"],
  "regressionCases": ["course_publish.requires_gfwfail_auth"],
  "risk": "medium",
  "expiresAt": "2026-08-19T00:00:00Z"
}
~~~

它可以写进 TOOLS / MEMORY / runbook，因为它：

- 有明确服务：agent-course。
- 有明确操作：git_push、course_publish。
- 有明确动作：push 前确认 gfwfail。
- 有证据：远端 commit 校验。
- 有回归用例：后续自动发布必须检查身份。

反过来，下面这种就不应该直接写：

~~~text
以后 push 前小心一点。
~~~

它没有 scope、没有可执行动作、没有证据、没有验证方式。最多进入 revise，不进入长期记忆。

## 5. 实战检查清单

给 Agent 加学习回灌时，可以用这组条件做 Done Criteria：

- 每条学习都有 sourceRunId / sourceTicketId。
- 每条学习都有 scope：service、environment、operationType 至少两项。
- 每条学习必须能转成一条 actionable rule。
- 中高风险学习必须绑定 evidence refs。
- 高风险学习必须绑定 regression case 和 review/expiry date。
- 写入前必须检查和现有 runbook / policy / memory 是否冲突。
- 写入后必须有 audit event，能追踪是谁、何时、因为什么事件学到这条规则。

一句话总结：**Agent 真正可靠的学习，不是记住更多文字，而是把经验变成有范围、有证据、有测试、有过期时间的未来行为约束。**
