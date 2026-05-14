# 320. Agent 证据新鲜度与过期闸门（Evidence Freshness & Staleness Gate）

上一课讲了 Evidence Gap Detection：证据缺了要能发现、能补就只读补证、不能补就降级。

今天继续补最后一块：**证据存在，不代表还能用。**

比如：

- 10 分钟前的 `git status clean`，不能证明现在还能安全 push
- 1 小时前的审批，可能已经超过有效期
- 昨天的健康检查，不能作为今天部署后的 post-check
- 旧的权限快照，不能证明当前 token 仍有权限

这就是 Evidence Freshness & Staleness Gate：**每条证据都要带 freshness policy，副作用前必须验证它没有过期。**

## 1. 为什么“有证据”还不够？

很多 Agent 风险不是完全没证据，而是用了过期证据：

> “我刚刚检查过。”  
> 但这个“刚刚”到底是 5 秒前、5 分钟前，还是上一次 session？

生产系统里，证据至少有三类时间约束：

1. **maxAge**：证据最多能老多久
2. **validUntil**：证据明确到什么时候失效
3. **refreshBeforeSideEffect**：某些副作用前必须重新取证

成熟 Agent 不应该只问：

- 我有没有证据？

还要问：

- 这条证据对当前动作还新鲜吗？
- 这个动作风险等级是否要求重新取证？
- 如果过期，能不能用只读方式刷新？

## 2. learn-claude-code：最小 Freshness Gate

先定义 EvidenceEnvelope，把时间策略写进证据本身。

```python
# learn-claude-code: evidence_freshness.py
from dataclasses import dataclass
from datetime import datetime, timedelta, timezone
from typing import Literal

Risk = Literal["low", "medium", "high", "critical"]

@dataclass
class FreshnessPolicy:
    max_age_seconds: int
    refresh_before_side_effect: bool = False
    min_risk_requiring_refresh: Risk = "high"

@dataclass
class EvidenceEnvelope:
    id: str
    kind: str
    created_at: datetime
    payload_hash: str
    policy: FreshnessPolicy
    valid_until: datetime | None = None

@dataclass
class FreshnessDecision:
    action: Literal["allow", "refresh", "block"]
    reason: str
    evidence_id: str
```

判断新鲜度不要写散在业务代码里，要集中成一个 gate。

```python
RISK_ORDER = {"low": 1, "medium": 2, "high": 3, "critical": 4}


def check_freshness(
    evidence: EvidenceEnvelope,
    *,
    now: datetime,
    side_effect: bool,
    risk: Risk,
) -> FreshnessDecision:
    if evidence.valid_until and now > evidence.valid_until:
        return FreshnessDecision("refresh", "evidence valid_until expired", evidence.id)

    age = (now - evidence.created_at).total_seconds()
    if age > evidence.policy.max_age_seconds:
        return FreshnessDecision("refresh", f"evidence too old: {age:.0f}s", evidence.id)

    must_refresh = (
        side_effect
        and evidence.policy.refresh_before_side_effect
        and RISK_ORDER[risk] >= RISK_ORDER[evidence.policy.min_risk_requiring_refresh]
    )
    if must_refresh:
        return FreshnessDecision("refresh", "side effect requires fresh read", evidence.id)

    return FreshnessDecision("allow", "fresh enough", evidence.id)
```

关键点：**refresh 不等于重新执行动作，只能重新读取事实。**

例如：

- `git status` 可以刷新
- `gh pr view` 可以刷新
- 查询部署健康可以刷新
- 但“重新发送消息”“重新部署”“重新扣款”不能作为刷新

## 3. 不同证据的默认 freshness 策略

可以先从一张简单策略表开始。

```python
DEFAULT_POLICIES = {
    "git_status_clean": FreshnessPolicy(
        max_age_seconds=30,
        refresh_before_side_effect=True,
        min_risk_requiring_refresh="medium",
    ),
    "approval_receipt": FreshnessPolicy(
        max_age_seconds=15 * 60,
        refresh_before_side_effect=False,
        min_risk_requiring_refresh="high",
    ),
    "policy_decision": FreshnessPolicy(
        max_age_seconds=5 * 60,
        refresh_before_side_effect=True,
        min_risk_requiring_refresh="high",
    ),
    "post_deploy_health": FreshnessPolicy(
        max_age_seconds=60,
        refresh_before_side_effect=False,
        min_risk_requiring_refresh="high",
    ),
    "delivery_receipt": FreshnessPolicy(
        max_age_seconds=24 * 60 * 60,
        refresh_before_side_effect=False,
        min_risk_requiring_refresh="critical",
    ),
}
```

经验规则：

- **环境状态**短 TTL：git status、CI、deploy health、权限快照
- **人类授权**中 TTL：approval、waiver、exception
- **历史回执**长 TTL：message id、commit hash、artifact hash
- **安全决策**副作用前最好刷新：policy decision、blast radius budget、kill switch

## 4. pi-mono：EvidenceFreshnessMiddleware

生产里适合挂在 Tool Gateway 前面：工具调用前先检查 proof 里的证据是否还可用。

```ts
// pi-mono: EvidenceFreshnessMiddleware.ts
type FreshnessAction = 'allow' | 'refresh' | 'block'

type EvidenceEnvelope = {
  id: string
  kind: string
  createdAt: string
  validUntil?: string
  payloadHash: string
  policy: {
    maxAgeSeconds: number
    refreshBeforeSideEffect: boolean
    minRiskRequiringRefresh: 'low' | 'medium' | 'high' | 'critical'
  }
}

type RefreshResult =
  | { ok: true; evidence: EvidenceEnvelope }
  | { ok: false; reason: string }

export class EvidenceFreshnessMiddleware {
  constructor(
    private evidenceStore: EvidenceStore,
    private refreshers: EvidenceRefresherRegistry,
    private audit: AuditLog,
  ) {}

  async beforeToolCall(ctx: ToolCallContext) {
    const required = await this.evidenceStore.loadRequiredProof(ctx.proofId)
    const stale: EvidenceEnvelope[] = []

    for (const evidence of required.items) {
      const decision = checkFreshness(evidence, {
        now: new Date(),
        sideEffect: ctx.tool.sideEffect,
        risk: ctx.risk.level,
      })

      if (decision.action === 'refresh') stale.push(evidence)
      if (decision.action === 'block') {
        throw new Error(`Blocked by stale evidence: ${decision.reason}`)
      }
    }

    for (const evidence of stale) {
      const refresher = this.refreshers.get(evidence.kind)
      if (!refresher) {
        throw new Error(`No readonly refresher for stale evidence: ${evidence.kind}`)
      }

      const refreshed: RefreshResult = await refresher.refresh({
        oldEvidenceId: evidence.id,
        readonly: true,
        runId: ctx.runId,
      })

      if (!refreshed.ok) {
        throw new Error(`Failed to refresh evidence ${evidence.id}: ${refreshed.reason}`)
      }

      await this.evidenceStore.linkReplacement({
        oldEvidenceId: evidence.id,
        newEvidenceId: refreshed.evidence.id,
        reason: 'freshness_refresh_before_tool_call',
      })
    }

    await this.audit.record({
      runId: ctx.runId,
      tool: ctx.tool.name,
      event: 'evidence_freshness_checked',
      staleCount: stale.length,
    })
  }
}
```

注意这里的边界：

- Middleware 可以刷新证据
- 但刷新器必须声明 `readonly: true`
- 如果没有只读刷新器，就阻断高风险副作用
- 刷新后的新证据要和旧证据建立 replacement link，方便审计

## 5. OpenClaw 实战：git push 前的 freshness gate

以课程 cron 为例，`git push` 前不要只依赖几分钟前的信息。

一个可靠 gate 应该要求：

```json
{
  "action": "git_push",
  "requiredFreshEvidence": [
    { "kind": "gh_pr_list", "maxAgeSeconds": 120 },
    { "kind": "git_pull_rebase", "maxAgeSeconds": 300 },
    { "kind": "git_status_clean_or_expected", "maxAgeSeconds": 30 },
    { "kind": "diff_check", "maxAgeSeconds": 300 }
  ]
}
```

执行前流程：

1. 检查 `gh pr list --state all --limit 5` 是否新鲜
2. 检查 `git pull --rebase --autostash origin main` 是否刚跑过
3. 检查 `git status --short` 是否符合预期
4. 检查 `git diff --check` 是否无空白错误
5. 任何证据过期，就重新只读/低风险刷新
6. 刷新失败则不 push

这能避免一种常见事故：

> Agent 先检查了 repo 状态，中间又改了文件或远端更新了，但最后继续用旧状态直接 push。

## 6. Freshness Gate 和 Evidence Gap Gate 的区别

两者经常一起出现，但解决的问题不同：

- **Evidence Gap Gate**：证据缺不缺？链路断没断？
- **Evidence Freshness Gate**：证据还新不新？现在还能不能用？

组合起来就是：

```text
required evidence exists
        ↓
chain is complete
        ↓
evidence is fresh enough
        ↓
side effect allowed
```

缺任何一层，都不应该贸然执行高风险动作。

## 7. 常见坑

### 坑 1：把缓存当证据

缓存可以帮助决策，但缓存命中本身不一定是新鲜证据。缓存证据也要带 `createdAt`、`source`、`maxAge`。

### 坑 2：刷新证据时偷偷产生副作用

比如为了确认消息是否发送成功，又发了一条测试消息。这不是刷新，是新副作用。

### 坑 3：所有证据统一 TTL

审批、git status、部署健康、message receipt 的时间语义完全不同。统一 TTL 会导致该刷的不刷、不该刷的频繁刷。

### 坑 4：刷新后覆盖旧证据

不要覆盖。旧证据是历史事实，新证据是当前事实。用 replacement link 连接它们。

## 8. 一句话总结

**证据不是永久通行证，而是有保质期的事实快照。**

成熟 Agent 在执行副作用前，不只要证明“我检查过”，还要证明：

> 我刚刚检查过，而且这条证据对当前风险等级仍然有效。
