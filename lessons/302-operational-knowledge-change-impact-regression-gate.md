# 302. Agent 运维知识变更影响分析与回归门控（Knowledge Change Impact Analysis & Regression Gate）

上一课讲了：检索到的运维知识如果互相冲突，不能直接塞进上下文，要先做冲突仲裁和可信度排序。

今天继续往前推一步：**知识本身发生变更时，不能只 merge 文档。**

因为 Agent 的运维知识不是普通说明书，它会直接影响：

- 是否允许执行副作用工具；
- 是否要求人工确认；
- Runbook 选择哪条路径；
- 事故升级是否触发 freeze；
- 上下文注入时给模型看哪些规则。

一句话：

> 运维知识库的每一次变更，都应该像代码变更一样做影响分析和回归测试。

---

## 1. 问题：改一条知识，可能改变很多 Agent 行为

假设有人把一条知识从：

```json
{
  "id": "kb-deploy-prod-guard",
  "claimType": "required_check",
  "statement": "生产发布前必须检查 error_budget_remaining > 20%。"
}
```

改成：

```json
{
  "id": "kb-deploy-prod-guard",
  "claimType": "recommend",
  "statement": "生产发布前建议检查 error_budget_remaining。"
}
```

看起来只是文字变温和了，但实际含义是：

- 原来会阻断低错误预算发布；
- 现在可能只 warning；
- Cron 自动发布可能从 `manual_review` 变成 `allow`；
- 事故窗口里的变更可能绕过人工审批。

这就是 Knowledge Change 的危险点：**文档 diff 很小，行为 diff 很大。**

所以知识变更前要回答三个问题：

1. 哪些 Runbook / Tool Policy / Middleware 会读到这条知识？
2. 哪些历史事故回归 case 可能受影响？
3. 变更前后，关键 intent 的决策有没有从更安全变成更危险？

---

## 2. 设计：Knowledge Change Impact Gate

可以把知识变更做成一条门控流水线：

```text
Knowledge Diff
     ↓
Normalize changed claims
     ↓
Find dependent runbooks / policies / regression cases
     ↓
Replay impacted intents before vs after
     ↓
Detect decision downgrade
     ↓
allow / require_review / block
```

核心不是检查 Markdown 格式，而是检查 **行为变化**。

一个实用的决策等级：

```text
allow < warn < require_confirmation < manual_review < block
```

如果变更让决策从 `block` 降到 `warn`，这就是安全降级，必须人工 review，甚至直接 block。

---

## 3. learn-claude-code：最小影响分析器

教学版用 Python 写一个最小 gate：输入变更前后的 claims，再 replay 一组历史 intent。

```python
# learn-claude-code/knowledge_change_gate.py
from dataclasses import dataclass
from typing import Literal

Decision = Literal["allow", "warn", "require_confirmation", "manual_review", "block"]
ClaimType = Literal["forbid", "required_check", "recommend", "fact", "exception"]

DECISION_RANK: dict[Decision, int] = {
    "allow": 0,
    "warn": 1,
    "require_confirmation": 2,
    "manual_review": 3,
    "block": 4,
}

@dataclass(frozen=True)
class Claim:
    id: str
    service: str
    env: str
    operation_type: str
    claim_type: ClaimType
    statement: str
    status: str = "active"

@dataclass(frozen=True)
class IntentCase:
    id: str
    service: str
    env: str
    operation_type: str
    side_effect: bool
    min_expected_decision: Decision

@dataclass
class ReplayResult:
    case_id: str
    before: Decision
    after: Decision
    downgraded: bool
    reason: str


def decide(claims: list[Claim], case: IntentCase) -> Decision:
    matched = [
        c for c in claims
        if c.status == "active"
        and c.service == case.service
        and c.env == case.env
        and c.operation_type == case.operation_type
    ]

    if any(c.claim_type == "forbid" for c in matched) and case.side_effect:
        return "block"

    if any(c.claim_type == "required_check" for c in matched) and case.side_effect:
        return "manual_review"

    if any(c.claim_type == "recommend" for c in matched):
        return "warn"

    return "allow"


def replay_change(before: list[Claim], after: list[Claim], cases: list[IntentCase]) -> list[ReplayResult]:
    results: list[ReplayResult] = []

    for case in cases:
        old_decision = decide(before, case)
        new_decision = decide(after, case)

        downgraded = DECISION_RANK[new_decision] < DECISION_RANK[old_decision]
        below_minimum = DECISION_RANK[new_decision] < DECISION_RANK[case.min_expected_decision]

        results.append(ReplayResult(
            case_id=case.id,
            before=old_decision,
            after=new_decision,
            downgraded=downgraded or below_minimum,
            reason=(
                "decision safety downgraded"
                if downgraded else
                "below regression minimum"
                if below_minimum else
                "ok"
            ),
        ))

    return results


def gate(results: list[ReplayResult]) -> Decision:
    if any(r.reason == "below regression minimum" for r in results):
        return "block"
    if any(r.downgraded for r in results):
        return "manual_review"
    return "allow"
```

使用示例：

```python
before = [
    Claim(
        id="kb-prod-deploy-budget",
        service="agent-course",
        env="prod",
        operation_type="git_push",
        claim_type="required_check",
        statement="push 前必须检查 gh pr list 和 git diff --check",
    )
]

after = [
    Claim(
        id="kb-prod-deploy-budget",
        service="agent-course",
        env="prod",
        operation_type="git_push",
        claim_type="recommend",
        statement="push 前建议检查 gh pr list 和 git diff --check",
    )
]

cases = [
    IntentCase(
        id="cron-course-push-main",
        service="agent-course",
        env="prod",
        operation_type="git_push",
        side_effect=True,
        min_expected_decision="manual_review",
    )
]

results = replay_change(before, after, cases)
print(results[0])
print(gate(results))  # manual_review 或 block
```

这段代码的重点：

- 不看“文字有没有变好看”；
- 只看“决策有没有变危险”；
- 用历史 case 固定最低安全行为；
- 知识库变更也要走 regression gate。

---

## 4. pi-mono：生产版 KnowledgeChangeMiddleware

在 pi-mono 里，这个能力适合放在知识库写入路径，而不是执行路径临时补救。

```ts
// pi-mono/packages/agent-runtime/src/ops/KnowledgeChangeMiddleware.ts
export type Decision = 'allow' | 'warn' | 'require_confirmation' | 'manual_review' | 'block';

const rank: Record<Decision, number> = {
  allow: 0,
  warn: 1,
  require_confirmation: 2,
  manual_review: 3,
  block: 4,
};

export interface KnowledgeClaim {
  id: string;
  service: string;
  env: string;
  operationType: string;
  claimType: 'forbid' | 'required_check' | 'recommend' | 'fact' | 'exception';
  statement: string;
  status: 'draft' | 'active' | 'stale' | 'archived';
}

export interface RegressionCase {
  id: string;
  service: string;
  env: string;
  operationType: string;
  sideEffect: boolean;
  minExpectedDecision: Decision;
}

export interface KnowledgeChangeReport {
  changedClaimIds: string[];
  impactedCaseIds: string[];
  decision: Decision;
  downgrades: Array<{
    caseId: string;
    before: Decision;
    after: Decision;
    reason: string;
  }>;
}

export class KnowledgeChangeMiddleware {
  constructor(
    private readonly resolver: {
      decide(claims: KnowledgeClaim[], regressionCase: RegressionCase): Promise<Decision>;
    },
    private readonly caseStore: {
      findImpacted(claims: KnowledgeClaim[]): Promise<RegressionCase[]>;
    },
    private readonly audit: {
      append(event: unknown): Promise<void>;
    },
  ) {}

  async evaluateChange(input: {
    actor: string;
    before: KnowledgeClaim[];
    after: KnowledgeClaim[];
  }): Promise<KnowledgeChangeReport> {
    const changed = this.changedClaims(input.before, input.after);
    const impactedCases = await this.caseStore.findImpacted(changed);

    const downgrades: KnowledgeChangeReport['downgrades'] = [];

    for (const c of impactedCases) {
      const beforeDecision = await this.resolver.decide(input.before, c);
      const afterDecision = await this.resolver.decide(input.after, c);

      if (rank[afterDecision] < rank[beforeDecision]) {
        downgrades.push({
          caseId: c.id,
          before: beforeDecision,
          after: afterDecision,
          reason: 'decision downgraded by knowledge change',
        });
      }

      if (rank[afterDecision] < rank[c.minExpectedDecision]) {
        downgrades.push({
          caseId: c.id,
          before: beforeDecision,
          after: afterDecision,
          reason: 'after decision below regression minimum',
        });
      }
    }

    const decision: Decision = downgrades.some(d => d.reason.includes('below'))
      ? 'block'
      : downgrades.length > 0
        ? 'manual_review'
        : 'allow';

    const report: KnowledgeChangeReport = {
      changedClaimIds: changed.map(c => c.id),
      impactedCaseIds: impactedCases.map(c => c.id),
      decision,
      downgrades,
    };

    await this.audit.append({
      type: 'knowledge_change_evaluated',
      actor: input.actor,
      report,
      at: new Date().toISOString(),
    });

    return report;
  }

  private changedClaims(before: KnowledgeClaim[], after: KnowledgeClaim[]): KnowledgeClaim[] {
    const oldById = new Map(before.map(c => [c.id, JSON.stringify(c)]));
    return after.filter(c => oldById.get(c.id) !== JSON.stringify(c));
  }
}
```

生产版建议加四个细节：

1. `findImpacted()` 不只按 service/env 找，还要看 `runbook.dependencies.knowledgeEntries`。
2. 对 `forbid -> recommend`、`required_check -> fact` 这类降级直接高亮。
3. 报告必须写审计日志，方便事故复盘追踪“哪次知识变更导致行为改变”。
4. 高风险服务的知识变更默认 require PR review，不能让 Agent 自己 silent merge。

---

## 5. OpenClaw 实战：课程 Cron 的知识门控

OpenClaw 这种长期运行 Agent 最容易积累自己的操作知识，比如：

- push 前必须 `gh pr list --state all --limit 5`；
- 必须先 `git pull --rebase --autostash origin main`；
- 发送 Telegram 前要避免重复课程；
- 更新 TOOLS.md 后要提交 repo；
- 外部消息发送失败要记录 evidence。

这些规则一旦写入 MEMORY / TOOLS / SKILL，就会影响后续自动化。

所以可以把知识变更加一个本地 gate：

```bash
python tools/knowledge_change_gate.py \
  --before .ops-kb/before.json \
  --after .ops-kb/after.json \
  --cases .ops-kb/regression-cases.json \
  --report artifacts/knowledge-change-report.json
```

报告里至少包含：

```json
{
  "decision": "manual_review",
  "changedClaimIds": ["kb-agent-course-push-check"],
  "impactedCaseIds": ["cron-agent-course-main-push"],
  "downgrades": [
    {
      "caseId": "cron-agent-course-main-push",
      "before": "manual_review",
      "after": "warn",
      "reason": "decision downgraded by knowledge change"
    }
  ]
}
```

如果 `decision != allow`，Agent 不应该继续自动推送知识库变更，而是汇报：

> 这次知识变更会降低生产 push 前检查强度，需要人工确认。

这就是把“记忆”从随手笔记升级成可治理的运行系统。

---

## 6. 什么时候必须做 Knowledge Change Gate？

不是所有文档都要重流程。建议这些情况强制 gate：

- 改了 `forbid` / `required_check` / `exception` 类型 claim；
- 改了 prod / payment / deploy / rollback / auth 相关知识；
- 改了工具权限、审批条件、Kill Switch、错误预算规则；
- 删除或 archive 事故复盘知识；
- 修改了 Runbook 依赖的知识条目；
- 让某个动作从 manual 变成 automatic。

低风险知识，比如解释性 FAQ、只读查询说明，可以只做格式检查和审计。

---

## 7. 小结

今天的核心：

- 运维知识变更不是普通文档变更，它会改变 Agent 行为；
- 要用 impacted regression cases replay 变更前后的决策；
- 重点检查安全降级：`block/manual_review -> warn/allow`；
- learn-claude-code 用最小 Python gate 教清楚行为 diff；
- pi-mono 用 KnowledgeChangeMiddleware 放进知识写入路径；
- OpenClaw 长期记忆 / TOOLS / Runbook 也应该逐步被这种 gate 管起来。

成熟 Agent 不只是会学习新知识，还要知道：**这次学习会不会让自己变得更危险。**
