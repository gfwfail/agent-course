# 319. Agent 证据缺口检测与修复（Evidence Gap Detection & Repair）

上一课我们讲了 Evidence Chain Verification：证据链要能证明 intent → plan → policy → approval → tool_result → post_check 没断链。

今天继续往前一步：**如果链断了，Agent 不应该只报错，而要能定位缺口、判断风险，并尝试安全修复。**

这就是 Evidence Gap Detection & Repair。

## 1. 为什么需要证据缺口检测？

生产里的证据链很少永远完美：

- 工具执行成功了，但忘了写 `post_check`
- 发消息有 provider receipt，但没有绑定原始 intent
- 审批记录存在，但 policy decision 没有挂到同一个 run
- 子 Agent 完成了任务，但 handoff evidence 没有回填父任务
- 缓存命中参与了决策，但没有记录 cache lineage

如果没有缺口检测，Agent 最终只能说：

> 我好像做了，但我证明不了。

成熟系统要能回答三件事：

1. **缺什么证据？**
2. **这个缺口会不会影响安全/审计/回滚？**
3. **能不能用只读方式补证？不能补时如何降级？**

## 2. Evidence Graph 的期望模板

先不要直接扫日志。更好的做法是定义一份“期望证据图”。

```python
# learn-claude-code: evidence_gap.py
from dataclasses import dataclass
from typing import Literal

NodeType = Literal[
    "intent",
    "plan",
    "policy_decision",
    "approval",
    "tool_call",
    "tool_result",
    "post_check",
    "delivery_receipt",
]

@dataclass
class RequiredEdge:
    source: NodeType
    target: NodeType
    required_when: str
    severity: Literal["warn", "block", "critical"]

EXPECTED_EDGES = [
    RequiredEdge("intent", "plan", "always", "block"),
    RequiredEdge("plan", "policy_decision", "side_effect", "critical"),
    RequiredEdge("policy_decision", "approval", "requires_approval", "critical"),
    RequiredEdge("policy_decision", "tool_call", "allowed", "block"),
    RequiredEdge("tool_call", "tool_result", "always", "block"),
    RequiredEdge("tool_result", "post_check", "high_risk", "critical"),
    RequiredEdge("tool_result", "delivery_receipt", "external_message", "block"),
]
```

核心思想：**不要靠人脑记“应该有什么”，把证据需求写成可执行契约。**

## 3. 检测缺口：从实际证据图里找 missing edge

```python
@dataclass
class EvidenceNode:
    id: str
    type: NodeType
    payload_hash: str
    parent_ids: list[str]
    meta: dict

@dataclass
class EvidenceGap:
    missing: RequiredEdge
    from_node_id: str | None
    reason: str
    repairable: bool


def condition_matches(edge: RequiredEdge, run_meta: dict) -> bool:
    if edge.required_when == "always":
        return True
    return bool(run_meta.get(edge.required_when))


def detect_gaps(nodes: list[EvidenceNode], run_meta: dict) -> list[EvidenceGap]:
    by_type: dict[str, list[EvidenceNode]] = {}
    for node in nodes:
        by_type.setdefault(node.type, []).append(node)

    gaps: list[EvidenceGap] = []

    for edge in EXPECTED_EDGES:
        if not condition_matches(edge, run_meta):
            continue

        sources = by_type.get(edge.source, [])
        targets = by_type.get(edge.target, [])

        if not sources:
            gaps.append(EvidenceGap(edge, None, f"missing source node: {edge.source}", False))
            continue

        if not targets:
            gaps.append(EvidenceGap(edge, sources[0].id, f"missing target node: {edge.target}", edge.target in REPAIRABLE_NODES))
            continue

        target_parent_ids = {pid for t in targets for pid in t.parent_ids}
        for source in sources:
            if source.id not in target_parent_ids:
                gaps.append(EvidenceGap(edge, source.id, f"{edge.target} not linked to {edge.source}", edge.target in REPAIRABLE_NODES))

    return gaps


REPAIRABLE_NODES = {"post_check", "delivery_receipt"}
```

这里有个重要细节：

- `intent`、`plan`、`policy_decision` 通常不能事后伪造
- `post_check`、`delivery_receipt` 可以通过只读查询补证

所以缺口不是都能修，必须区分：

- **repairable**：可以重新查状态补一条证据
- **non-repairable**：只能标记审计缺口，阻断高风险结论

## 4. pi-mono 生产版：EvidenceGapMiddleware

在生产系统里，它适合挂在副作用完成之后、最终回复之前。

```ts
// pi-mono: EvidenceGapMiddleware.ts
type GapSeverity = 'warn' | 'block' | 'critical'

type EvidenceGap = {
  runId: string
  missingType: string
  fromNodeId?: string
  severity: GapSeverity
  repairable: boolean
  reason: string
}

type RepairResult =
  | { ok: true; nodeId: string; evidenceHash: string }
  | { ok: false; reason: string }

export class EvidenceGapMiddleware {
  constructor(
    private evidenceStore: EvidenceStore,
    private repairRegistry: RepairRegistry,
    private incidentCenter: IncidentCenter,
  ) {}

  async beforeFinalResponse(ctx: AgentRunContext) {
    const graph = await this.evidenceStore.loadGraph(ctx.runId)
    const gaps = detectEvidenceGaps(graph, {
      sideEffect: ctx.plan.hasSideEffect,
      highRisk: ctx.risk.level === 'high' || ctx.risk.level === 'critical',
      externalMessage: ctx.sideEffects.some(x => x.kind === 'message'),
      requiresApproval: ctx.policy.action === 'require_approval',
    })

    const unrepaired: EvidenceGap[] = []

    for (const gap of gaps) {
      if (gap.repairable) {
        const repaired = await this.tryRepair(ctx, gap)
        if (!repaired.ok) unrepaired.push(gap)
      } else {
        unrepaired.push(gap)
      }
    }

    const worst = maxSeverity(unrepaired)

    if (worst === 'critical') {
      await this.incidentCenter.open({
        severity: 'P1',
        title: 'Critical evidence gap before final response',
        runId: ctx.runId,
        gaps: unrepaired,
      })
      throw new Error('Blocked: critical evidence gap detected')
    }

    if (worst === 'block') {
      ctx.finalResponseMode = 'blocked_with_audit_note'
      ctx.audit.gaps = unrepaired
    }
  }

  private async tryRepair(ctx: AgentRunContext, gap: EvidenceGap): Promise<RepairResult> {
    const repairer = this.repairRegistry.get(gap.missingType)
    if (!repairer) return { ok: false, reason: 'no repairer registered' }

    // 修复只能用 readonly 工具，不能为了补证制造新副作用
    return repairer.repair({ runId: ctx.runId, gap, readonly: true })
  }
}
```

重点规则：

> 补证只能用只读工具，不能为了修证据链再制造新的副作用。

比如：

- 可以查 Telegram message id 是否存在
- 可以查 Git commit hash 是否已推送
- 可以查部署健康状态
- 不可以重新发送消息
- 不可以重新部署一次

## 5. OpenClaw 实战：课程 Cron 的证据缺口

以本课程 Cron 为例，一个完整 run 至少要留下：

```json
{
  "runId": "agent-course-319",
  "expectedEvidence": [
    "lesson_file_written",
    "readme_updated",
    "tools_topic_updated",
    "telegram_message_id",
    "git_commit_sha",
    "git_push_verified"
  ]
}
```

如果最终发现：

- `lesson_file_written` 有
- `telegram_message_id` 有
- `git_commit_sha` 有
- 但缺 `git_push_verified`

Agent 不该直接说“完成”。它应该补一个只读验证：

```bash
git ls-remote origin main | grep <commit_sha>
```

查到了，追加一条 `git_push_verified` evidence。
查不到，就阻断最终完成声明，报告：

> 文件已提交，但没有证据证明 commit 在远端 main。需要重新 push 或人工检查。

这就是“完成不是感觉，是证据”。

## 6. 常见坑

### 坑 1：事后伪造 intent

`intent` 是用户请求的原始记录，不能事后补写成“看起来像”。

如果缺 intent，只能标记 non-repairable audit gap。

### 坑 2：把 retry 当 repair

修复证据缺口不是重做副作用。

- 查状态 = repair
- 再发一次消息 = duplicate side effect

### 坑 3：所有缺口都 blocking

不是所有缺口风险一样：

- 普通回答缺 citation：warn
- 发消息缺 receipt：block
- 高风险部署缺 approval：critical

缺口要按风险分级，否则系统会过度保守，最后没人愿意用。

## 7. 设计 Checklist

- [ ] 每类 run 是否有 expected evidence template？
- [ ] 缺口是否区分 warn/block/critical？
- [ ] 哪些证据允许只读补证？哪些绝不能事后补？
- [ ] repair 是否保证不产生新副作用？
- [ ] critical gap 是否会打开 incident 或阻断最终完成？
- [ ] 最终回复是否只在 evidence gate 通过后才说 completed？

## 8. 一句话总结

Evidence Chain Verification 证明“链没断”；Evidence Gap Detection & Repair 负责在链断时告诉你：**断在哪里、能不能安全补、补不了该不该阻断。**

成熟 Agent 不只是会留下证据，还要会发现自己少留了哪块证据，并且用最小副作用把审计链补完整。
