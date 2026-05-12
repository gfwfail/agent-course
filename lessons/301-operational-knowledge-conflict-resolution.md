# 301. Agent 运维知识冲突仲裁与可信度排序（Operational Knowledge Conflict Resolution）

上一课讲了：生产操作前要自动检索相关 Runbook / 事故知识 / 策略片段，并注入执行上下文。

但真实系统里很快会遇到一个更麻烦的问题：**检索出来的知识互相打架**。

例如：

- 旧 Runbook 说“发布失败直接重启 queue worker”。
- 新事故复盘说“不要重启，先检查重复投递 outbox”。
- Slack 历史消息说“上次这么做没问题”。
- 用户当前指令说“直接强制推送”。

这时成熟 Agent 不能简单把所有内容塞给 LLM，让模型自己猜谁对。

一句话：

> 运维知识进入上下文前，必须先做冲突检测、可信度排序和风险仲裁。

---

## 1. 问题：知识越多，冲突越多

RAG / Knowledge Base 最容易踩的坑是：资料库越大，答案不一定越准，反而可能更混乱。

常见冲突来源：

1. **时间冲突**：旧 SOP 已被新事故复盘推翻。
2. **环境冲突**：staging 的处理方式不能套到 prod。
3. **权限冲突**：用户要求做的动作超过当前会话授权。
4. **来源冲突**：随手聊天记录 vs 正式 Runbook。
5. **目的冲突**：answer 可以引用旧资料，side_effect 必须 live check。
6. **范围冲突**：某服务专用经验被错误套到另一个服务。

所以 Ops Knowledge Retrieval 后面应该接一个 **Conflict Resolver**：

```text
Retrieve Knowledge Entries
        ↓
Normalize Claims
        ↓
Detect Conflicts
        ↓
Rank Trust
        ↓
Decide: allow / warn / require_confirmation / block
        ↓
Inject compact resolved context
```

核心原则：**不要把未仲裁的冲突直接交给最终 LLM。**

---

## 2. 知识条目要能拆成 Claim

如果知识只是 Markdown，很难判断冲突。更好的方式是把每条知识拆成结构化 claim：

```json
{
  "id": "claim-deploy-rollback-001",
  "entryId": "kb-incident-2026-05-12",
  "service": "payments-api",
  "env": "prod",
  "operationType": "rollback",
  "claimType": "required_check",
  "statement": "回滚前必须确认 outbox pending_count 小于 100，否则先暂停 dispatcher。",
  "sourceType": "incident_review",
  "sourceUrl": "incidents/2026-05-12-payments.md",
  "status": "active",
  "observedAt": "2026-05-12T04:30:00Z",
  "reviewAfter": "2026-06-12",
  "supersedes": ["claim-deploy-rollback-legacy-009"],
  "risk": "high"
}
```

最关键字段：

- `claimType`：这是禁止动作、必须检查、建议、事实，还是例外？
- `sourceType`：正式策略、事故复盘、Runbook、聊天记录、模型总结，可信度不同。
- `supersedes`：显式声明替代旧知识。
- `service/env/operationType`：避免跨范围误用。
- `risk`：风险越高，冲突越不能自动忽略。

---

## 3. learn-claude-code：最小 Python 仲裁器

教学版先不用向量库，只演示冲突检测和可信度排序。

```python
# learn-claude-code/ops_conflict_resolver.py
from dataclasses import dataclass, field
from datetime import datetime, timezone
from typing import Literal

SourceType = Literal["policy", "runbook", "incident_review", "operator_note", "chat", "model_summary"]
ClaimType = Literal["forbid", "required_check", "recommend", "fact", "exception"]
Decision = Literal["allow", "warn", "require_confirmation", "block"]

SOURCE_WEIGHT: dict[SourceType, int] = {
    "policy": 100,
    "incident_review": 90,
    "runbook": 80,
    "operator_note": 60,
    "chat": 30,
    "model_summary": 20,
}

CLAIM_WEIGHT: dict[ClaimType, int] = {
    "forbid": 100,
    "required_check": 80,
    "exception": 70,
    "recommend": 40,
    "fact": 20,
}

@dataclass
class Claim:
    id: str
    service: str
    env: str
    operation_type: str
    claim_type: ClaimType
    statement: str
    source_type: SourceType
    status: str = "active"
    observed_at: datetime = field(default_factory=lambda: datetime.now(timezone.utc))
    supersedes: list[str] = field(default_factory=list)
    risk: str = "medium"

@dataclass
class Intent:
    service: str
    env: str
    operation_type: str
    side_effect: bool
    requested_action: str

@dataclass
class Resolution:
    decision: Decision
    selected_claims: list[Claim]
    conflicts: list[tuple[Claim, Claim]]
    reason: str


def applies(claim: Claim, intent: Intent) -> bool:
    return (
        claim.status == "active"
        and claim.service == intent.service
        and claim.env == intent.env
        and claim.operation_type == intent.operation_type
    )


def trust_score(claim: Claim) -> int:
    return SOURCE_WEIGHT[claim.source_type] + CLAIM_WEIGHT[claim.claim_type]


def conflicts(a: Claim, b: Claim) -> bool:
    if b.id in a.supersedes or a.id in b.supersedes:
        return True
    if a.claim_type == "forbid" and b.claim_type in ("recommend", "exception"):
        return True
    if b.claim_type == "forbid" and a.claim_type in ("recommend", "exception"):
        return True
    return False


def resolve_claims(claims: list[Claim], intent: Intent) -> Resolution:
    candidates = [c for c in claims if applies(c, intent)]
    ranked = sorted(candidates, key=trust_score, reverse=True)

    pairs: list[tuple[Claim, Claim]] = []
    for i, a in enumerate(ranked):
        for b in ranked[i + 1:]:
            if conflicts(a, b):
                pairs.append((a, b))

    # 显式 superseded：保留高可信新 claim，丢弃被替代旧 claim
    superseded_ids = {old for c in ranked for old in c.supersedes}
    selected = [c for c in ranked if c.id not in superseded_ids]

    strongest = selected[0] if selected else None

    if strongest and strongest.claim_type == "forbid" and intent.side_effect:
        return Resolution("block", selected[:5], pairs, f"Highest-trust claim forbids this action: {strongest.id}")

    if pairs and intent.side_effect:
        return Resolution("require_confirmation", selected[:5], pairs, "Conflicting operational knowledge found before side effect")

    if pairs:
        return Resolution("warn", selected[:5], pairs, "Conflicting knowledge found, but this is read-only")

    return Resolution("allow", selected[:5], pairs, "No blocking conflict")
```

使用示例：

```python
intent = Intent(
    service="payments-api",
    env="prod",
    operation_type="rollback",
    side_effect=True,
    requested_action="rollback latest deploy",
)

resolution = resolve_claims(all_claims, intent)

if resolution.decision == "block":
    raise RuntimeError(resolution.reason)

if resolution.decision == "require_confirmation":
    print("Need human confirmation before executing")
```

这个最小版本有三个重点：

1. **先过滤适用范围**：service/env/operation_type 不匹配的不参与仲裁。
2. **高可信来源优先**：policy > incident_review > runbook > chat。
3. **副作用更严格**：只读可以 warn，写操作冲突必须确认或阻断。

---

## 4. pi-mono：生产版 Knowledge Resolver Middleware

生产里建议把它做成 Agent Loop 中间件，而不是散落在每个工具里。

```ts
// pi-mono/packages/agent/src/middleware/OpsKnowledgeResolver.ts
export type SourceType =
  | 'policy'
  | 'incident_review'
  | 'runbook'
  | 'operator_note'
  | 'chat'
  | 'model_summary';

export type ClaimType =
  | 'forbid'
  | 'required_check'
  | 'recommend'
  | 'fact'
  | 'exception';

export interface OpsClaim {
  id: string;
  service: string;
  environment: 'dev' | 'staging' | 'prod';
  operationType: string;
  claimType: ClaimType;
  statement: string;
  sourceType: SourceType;
  sourceId: string;
  status: 'active' | 'stale' | 'superseded' | 'archived';
  observedAt: string;
  reviewAfter?: string;
  supersedes?: string[];
  risk: 'low' | 'medium' | 'high' | 'critical';
}

export interface ExecutionIntent {
  service: string;
  environment: 'dev' | 'staging' | 'prod';
  operationType: string;
  sideEffect: boolean;
  userRequest: string;
}

export interface ResolutionResult {
  decision: 'allow' | 'warn' | 'require_confirmation' | 'block';
  selectedClaims: OpsClaim[];
  conflicts: Array<{ a: OpsClaim; b: OpsClaim; reason: string }>;
  contextBlock: string;
}
```

Resolver 核心逻辑：

```ts
const SOURCE_WEIGHT: Record<SourceType, number> = {
  policy: 100,
  incident_review: 90,
  runbook: 80,
  operator_note: 60,
  chat: 30,
  model_summary: 20,
};

const CLAIM_WEIGHT: Record<ClaimType, number> = {
  forbid: 100,
  required_check: 80,
  exception: 70,
  recommend: 40,
  fact: 20,
};

export class OpsKnowledgeResolver {
  constructor(private readonly store: OpsKnowledgeStore) {}

  async resolve(intent: ExecutionIntent): Promise<ResolutionResult> {
    const claims = await this.store.findClaims({
      service: intent.service,
      environment: intent.environment,
      operationType: intent.operationType,
      status: 'active',
    });

    const ranked = claims
      .filter((claim) => this.isApplicable(claim, intent))
      .sort((a, b) => this.score(b) - this.score(a));

    const superseded = new Set(ranked.flatMap((claim) => claim.supersedes ?? []));
    const selected = ranked.filter((claim) => !superseded.has(claim.id));
    const conflicts = this.detectConflicts(selected);

    const decision = this.decide(intent, selected, conflicts);

    return {
      decision,
      selectedClaims: selected.slice(0, 6),
      conflicts,
      contextBlock: this.buildContextBlock(selected.slice(0, 6), conflicts),
    };
  }

  private score(claim: OpsClaim): number {
    return SOURCE_WEIGHT[claim.sourceType] + CLAIM_WEIGHT[claim.claimType];
  }

  private isApplicable(claim: OpsClaim, intent: ExecutionIntent): boolean {
    return claim.service === intent.service
      && claim.environment === intent.environment
      && claim.operationType === intent.operationType
      && claim.status === 'active';
  }

  private detectConflicts(claims: OpsClaim[]) {
    const conflicts: Array<{ a: OpsClaim; b: OpsClaim; reason: string }> = [];

    for (let i = 0; i < claims.length; i++) {
      for (let j = i + 1; j < claims.length; j++) {
        const a = claims[i];
        const b = claims[j];

        if ((a.supersedes ?? []).includes(b.id) || (b.supersedes ?? []).includes(a.id)) {
          conflicts.push({ a, b, reason: 'supersession' });
        }

        if (a.claimType === 'forbid' && ['recommend', 'exception'].includes(b.claimType)) {
          conflicts.push({ a, b, reason: 'forbid_vs_allowance' });
        }

        if (b.claimType === 'forbid' && ['recommend', 'exception'].includes(a.claimType)) {
          conflicts.push({ a, b, reason: 'forbid_vs_allowance' });
        }
      }
    }

    return conflicts;
  }

  private decide(
    intent: ExecutionIntent,
    claims: OpsClaim[],
    conflicts: Array<{ a: OpsClaim; b: OpsClaim; reason: string }>,
  ): ResolutionResult['decision'] {
    const strongest = claims[0];

    if (strongest?.claimType === 'forbid' && intent.sideEffect) {
      return 'block';
    }

    if (conflicts.length > 0 && intent.sideEffect) {
      return 'require_confirmation';
    }

    if (conflicts.length > 0) {
      return 'warn';
    }

    return 'allow';
  }

  private buildContextBlock(claims: OpsClaim[], conflicts: ResolutionResult['conflicts']): string {
    return [
      '<resolved_operational_knowledge>',
      ...claims.map((claim) =>
        `- [${claim.sourceType}/${claim.claimType}/${claim.risk}] ${claim.statement} (${claim.id})`,
      ),
      conflicts.length ? '<conflicts_detected>true</conflicts_detected>' : '<conflicts_detected>false</conflicts_detected>',
      '</resolved_operational_knowledge>',
    ].join('\n');
  }
}
```

接入 Agent Loop：

```ts
const resolution = await opsKnowledgeResolver.resolve(intent);

if (resolution.decision === 'block') {
  return {
    type: 'blocked',
    reason: 'Operational knowledge forbids this action.',
    evidence: resolution.selectedClaims,
  };
}

if (resolution.decision === 'require_confirmation') {
  return approvalGateway.request({
    reason: 'Conflicting operational knowledge before side-effect.',
    evidence: resolution.conflicts,
    proposedAction: intent.userRequest,
  });
}

agentContext.injectSystemBlock(resolution.contextBlock);
```

生产里尤其要注意：**resolution.decision 要在工具执行前生效，而不是只作为提示词建议。**

---

## 5. OpenClaw 实战：课程 Cron 的 push 前知识仲裁

拿这个课程 Cron 举例。每次发课、改 README、更新 TOOLS、git push 前，其实都有运维知识：

- 必须切换到 `gfwfail`。
- push 前必须 `gh pr list --state all --limit 5`。
- 必须 `git pull --rebase --autostash origin main`。
- 课程不能重复已讲内容。
- 发送 Telegram 后要保留 messageId 证据。

这些知识可能来自 `TOOLS.md`、`MEMORY.md`、daily memory、repo README。执行前可以构造一个小型仲裁包：

```json
{
  "intent": {
    "service": "agent-course",
    "env": "prod",
    "operationType": "publish_lesson",
    "sideEffect": true
  },
  "selectedClaims": [
    {
      "sourceType": "memory",
      "claimType": "required_check",
      "statement": "git push 前必须 gh pr list --state all --limit 5 检查 PR 状态。",
      "risk": "high"
    },
    {
      "sourceType": "tools",
      "claimType": "required_check",
      "statement": "课程内容要避免重复 TOOLS.md 已讲列表。",
      "risk": "medium"
    }
  ],
  "decision": "allow"
}
```

如果有冲突，比如：

- 当前 cron 说“直接 push main”。
- MEMORY 说“修改任何代码必须先提 PR，不能直接 push main”。
- 课程历史 cron 又一直是直接更新课程仓库 main。

这时 Resolver 应该按 **service-specific runbook + explicit cron instruction + repo convention** 做范围判断：

- 对普通业务代码：PR 规则优先。
- 对课程自动发布仓库：当前 cron 明确要求 `git commit && git push`，且历史记录证明这是既定流程。
- 结论：允许课程仓库直接 push，但仍执行 `gh pr list` 和 `git pull --rebase` 作为安全闸门。

这就是“冲突仲裁”的价值：不是机械执行某条规则，而是把规则的适用范围、来源和风险讲清楚。

---

## 6. 决策矩阵：什么时候 allow，什么时候 block？

建议生产里用这样一张矩阵：

| 场景 | 只读回答 | 外部副作用 |
|---|---:|---:|
| 无冲突，高可信知识 | allow | allow |
| 低可信聊天记录冲突 | warn | require_confirmation |
| policy / incident_review 明确 forbid | warn + cite | block |
| 旧 Runbook 被新复盘 supersede | 使用新知识 | 使用新知识 |
| service/env 不匹配 | ignore | ignore |
| 知识过期但没有替代 | warn | require_confirmation |
| 多条高可信知识互相冲突 | warn | block / require_confirmation |

一个实用规则：

> 只读场景可以带 caveat 输出；副作用场景宁可慢一点，也不要带冲突继续执行。

---

## 7. 常见坑

### 坑 1：把“检索相关”当成“可以执行”

RAG 的相关性只说明文本相似，不说明它适用于当前操作。

必须再看：

- 状态是否 active
- 环境是否匹配
- 是否过期
- 是否被 supersede
- 来源是否可信
- 当前用途是否允许

### 坑 2：把聊天记录当正式政策

聊天记录可以作为线索，但不能直接压过 policy / incident review / runbook。

生产建议：聊天记录默认只能生成 `recommend` 或 `fact`，不能直接生成 `forbid` / `required_check`，除非人工提升。

### 坑 3：冲突只提示 LLM，不拦工具

如果冲突判断只写进 prompt，模型仍可能“看懂了但继续执行”。

正确做法：

- Resolver 产出结构化 `decision`
- Tool Dispatcher / Approval Gateway 强制执行 decision
- `block` 时工具根本不可调用

### 坑 4：没有解释为什么选这条知识

仲裁结果必须可解释：

```text
Selected claim: incident_review outranks old runbook because it supersedes claim-X and applies to prod rollback.
```

否则出了事故没人知道 Agent 为什么相信 A 不相信 B。

---

## 8. 最小落地清单

如果你现在要给 Agent 加这个能力，先做 5 件事：

1. 给知识条目加 `sourceType/status/service/env/operationType/risk` 元数据。
2. 把关键 Runbook 规则拆成 claim。
3. 定义来源可信度排序：policy > incident_review > runbook > operator_note > chat > model_summary。
4. 在副作用工具执行前跑 Conflict Resolver。
5. 把 `decision + selectedClaims + conflicts` 写进审计日志。

---

## 总结

今天这课的核心：

- 知识库不是越大越好，**可仲裁才可靠**。
- 运维知识要拆成 Claim，带来源、范围、风险和生命周期。
- 只读回答可以 warn，外部副作用必须 require_confirmation 或 block。
- 仲裁结果必须进入工具执行层，而不是只提示 LLM。
- 成熟 Agent 不只是“想起知识”，还要知道**哪条知识更可信，哪条已经不该用了**。

下一步可以继续扩展：把知识冲突结果沉淀回 KB，形成“冲突复盘 → 规则升级 → 自动防复发”的闭环。
