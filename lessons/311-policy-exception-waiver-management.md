# 第 311 课：Agent 策略例外与临时豁免管理（Policy Exception & Waiver Management）

前几课我们讲了策略影子模式、差异预算、决策解释、fixture replay、覆盖矩阵和变异测试。到这里，策略系统已经比较“硬”了：该拦的能拦，该测的能测。

但生产里还有一个更现实的问题：

> 策略不能只有 allow / deny。有些事平时必须拦，但在明确授权、短时间窗口、限定范围内，可以临时放行。

比如：

- 周末 freeze 期间禁止部署，但 P0 修复需要临时放行一次；
- 默认禁止直接 push main，但某个自动课程 repo 明确允许 cron 直接推；
- 生产 shell 操作默认 require approval，但一次事故处理中允许 30 分钟内执行特定 runbook；
- 群聊里禁止写长期记忆，但 owner 明确说“记住这条”时允许一次。

这就是今天的主题：**Policy Exception / Waiver Management**。不要把例外写进 prompt，也不要把策略规则直接改松；要把例外做成可审计、可过期、可撤销、可精确匹配的授权对象。

## 1. 核心思想

策略例外不是“绕过策略”，而是策略输入的一部分。

一个合格的 waiver 至少要包含：

```text
who      谁批准的
what     允许哪个动作/工具/目标
scope    限定在哪个 repo/env/channel/user 上
why      原因和关联 incident / request
when     生效时间与过期时间
limits   次数、风险等级、最大影响范围
proof    审批消息、ticket、run id 等证据
```

策略引擎的决策顺序应该是：

```text
1. 先计算基础策略：allow / deny / require_approval / dry_run
2. 如果基础策略允许，直接通过
3. 如果基础策略阻断，再查找是否有匹配 waiver
4. waiver 匹配、未过期、未超次数、未撤销 → 降级为 allow_with_waiver
5. 否则维持原始阻断
```

重点：**waiver 只能缩小范围地放行，不能扩大成通用权限。**

## 2. learn-claude-code：最小 Python 教学版

先定义策略输入、决策和 waiver：

```python
from dataclasses import dataclass, field
from datetime import datetime, timezone
from typing import Literal

Action = Literal["allow", "deny", "require_approval", "dry_run", "allow_with_waiver"]

@dataclass
class PolicyInput:
    actor: str
    channel: str
    tool: str
    target: str
    risk: str
    run_id: str

@dataclass
class PolicyDecision:
    action: Action
    reason_codes: list[str]
    waiver_id: str | None = None

@dataclass
class Waiver:
    id: str
    approved_by: str
    actor: str
    tool: str
    target: str
    max_risk: str
    reason: str
    evidence_ref: str
    expires_at: datetime
    max_uses: int = 1
    used_by_runs: set[str] = field(default_factory=set)
    revoked: bool = False
```

基础策略保持严格：

```python
def base_policy(i: PolicyInput) -> PolicyDecision:
    if i.tool == "git_push" and i.target == "main":
        return PolicyDecision("require_approval", ["MAIN_PUSH_REQUIRES_APPROVAL"])

    if i.tool == "shell" and i.risk in ["high", "critical"]:
        return PolicyDecision("dry_run", ["HIGH_RISK_SHELL_DRY_RUN"])

    if i.channel == "group" and i.tool == "memory_write":
        return PolicyDecision("deny", ["GROUP_MEMORY_WRITE_DENIED"])

    return PolicyDecision("allow", ["POLICY_OK"])
```

再写 waiver 匹配器：

```python
RISK_ORDER = {"low": 1, "medium": 2, "high": 3, "critical": 4}

def waiver_matches(w: Waiver, i: PolicyInput, now: datetime) -> bool:
    if w.revoked:
        return False
    if now >= w.expires_at:
        return False
    if len(w.used_by_runs) >= w.max_uses and i.run_id not in w.used_by_runs:
        return False
    if w.actor != i.actor:
        return False
    if w.tool != i.tool:
        return False
    if w.target != i.target:
        return False
    if RISK_ORDER[i.risk] > RISK_ORDER[w.max_risk]:
        return False
    return True
```

最终策略把基础决策和 waiver 合并：

```python
def decide(i: PolicyInput, waivers: list[Waiver], now: datetime) -> PolicyDecision:
    base = base_policy(i)
    if base.action == "allow":
        return base

    for w in waivers:
        if waiver_matches(w, i, now):
            w.used_by_runs.add(i.run_id)
            return PolicyDecision(
                action="allow_with_waiver",
                reason_codes=base.reason_codes + ["WAIVER_APPLIED"],
                waiver_id=w.id,
            )

    return base
```

测试重点不是“能不能放行”，而是证明它只放行这一格：

```python
now = datetime.now(timezone.utc)
waiver = Waiver(
    id="wv_123",
    approved_by="owner",
    actor="course-cron",
    tool="git_push",
    target="main",
    max_risk="high",
    reason="Agent course repo allows scheduled direct push",
    evidence_ref="telegram:67431246/msg/abc",
    expires_at=now.replace(hour=23, minute=59),
)

ok = decide(
    PolicyInput("course-cron", "private", "git_push", "main", "high", "run_1"),
    [waiver],
    now,
)
assert ok.action == "allow_with_waiver"
assert ok.waiver_id == "wv_123"

# 换 actor 不行
blocked = decide(
    PolicyInput("unknown-agent", "private", "git_push", "main", "high", "run_2"),
    [waiver],
    now,
)
assert blocked.action == "require_approval"

# 换风险等级不行
critical = decide(
    PolicyInput("course-cron", "private", "git_push", "main", "critical", "run_3"),
    [waiver],
    now,
)
assert critical.action == "require_approval"
```

## 3. pi-mono：生产版 Waiver Store + Policy Middleware

生产里不要把 waiver 写死在代码里，应该放到一个可查询、可审计的 store。

```ts
type PolicyAction = 'allow' | 'deny' | 'require_approval' | 'dry_run' | 'allow_with_waiver'

type Waiver = {
  id: string
  approvedBy: string
  actor: string
  tool: string
  target: string
  maxRisk: 'low' | 'medium' | 'high' | 'critical'
  reason: string
  evidenceRef: string
  expiresAt: string
  maxUses: number
  usedByRuns: string[]
  revokedAt?: string
}

type PolicyInput = {
  actor: string
  tool: string
  target: string
  risk: 'low' | 'medium' | 'high' | 'critical'
  runId: string
}
```

WaiverStore 接口：

```ts
interface WaiverStore {
  findCandidates(input: PolicyInput): Promise<Waiver[]>
  markUsed(waiverId: string, runId: string): Promise<void>
  appendAudit(event: {
    type: 'waiver_applied' | 'waiver_rejected'
    waiverId: string
    runId: string
    reason: string
  }): Promise<void>
}
```

Policy middleware：

```ts
const riskRank = { low: 1, medium: 2, high: 3, critical: 4 }

function matches(w: Waiver, input: PolicyInput, now = new Date()): boolean {
  if (w.revokedAt) return false
  if (new Date(w.expiresAt).getTime() <= now.getTime()) return false
  if (w.actor !== input.actor) return false
  if (w.tool !== input.tool) return false
  if (w.target !== input.target) return false
  if (riskRank[input.risk] > riskRank[w.maxRisk]) return false
  if (!w.usedByRuns.includes(input.runId) && w.usedByRuns.length >= w.maxUses) return false
  return true
}

export async function applyWaiverGate(input: PolicyInput, baseDecision: PolicyDecision, store: WaiverStore) {
  if (baseDecision.action === 'allow') return baseDecision

  const candidates = await store.findCandidates(input)
  const waiver = candidates.find(w => matches(w, input))

  if (!waiver) return baseDecision

  await store.markUsed(waiver.id, input.runId)
  await store.appendAudit({
    type: 'waiver_applied',
    waiverId: waiver.id,
    runId: input.runId,
    reason: waiver.reason,
  })

  return {
    action: 'allow_with_waiver' as const,
    reasonCodes: [...baseDecision.reasonCodes, 'WAIVER_APPLIED'],
    constraints: [`waiver=${waiver.id}`, `expiresAt=${waiver.expiresAt}`],
    evidenceRefs: [waiver.evidenceRef],
  }
}
```

这个中间件最好放在 `PolicyEngine` 和 `ToolDispatcher` 之间：

```text
LLM intent
  → Tool args canonicalization
  → Base Policy
  → Waiver Gate
  → Audit Log
  → Tool Dispatcher
```

这样 LLM 不需要知道 waiver 细节，它只看到最终 decision 和 constraints；真正的授权逻辑在工具层。

## 4. OpenClaw 实战：把“例外”写成文件式授权

OpenClaw 里可以用一个简单 JSONL 文件做 waiver store：

```json
{"id":"wv_course_push_2026_05","actor":"agent-course-cron","tool":"git_push","target":"gfwfail/agent-course:main","maxRisk":"medium","approvedBy":"owner","reason":"Scheduled course publishing","evidenceRef":"cron:3eba6ee3-2d11-4afa-aa7f-a47b63226982","expiresAt":"2026-06-01T00:00:00Z","maxUses":200,"usedByRuns":[]}
```

执行高风险动作前，不要只问“用户以前说过可以吗”，而是做三步：

1. 重新采样当前动作：`actor/tool/target/risk/runId`
2. 查 waiver 是否精确匹配、未过期、未撤销、未超次数
3. 把 `waiverId/evidenceRef` 写进最终完成记录

这和我们这个课程 cron 很像：正常记忆里有“不能直接 push main”的长期规则，但当前任务明确要求“git commit && git push 更新 repo”。这类场景就应该被建模为**受限例外**，而不是把全局规则永久改松。

## 5. 常见坑

### 坑 1：永久例外

`expiresAt = never` 基本等于新权限。除非是产品级策略变更，否则 waiver 必须过期。

### 坑 2：范围太宽

坏例子：

```json
{"tool":"shell","target":"*","maxRisk":"critical"}
```

好例子：

```json
{"tool":"shell","target":"prod-web-1:systemctl restart app","maxRisk":"high"}
```

### 坑 3：只记录审批，不记录使用

审批只是开始，真正危险的是使用。每次 waiver applied 都要记录 `runId / tool / argsHash / result / verification`。

### 坑 4：waiver 可以覆盖 deny

不是所有 deny 都能被 waiver 覆盖。比如数据越权、密钥泄露、未知 actor，这类应该是 hard deny：

```text
hard_deny > waiver > require_approval > dry_run > allow
```

## 6. 小结

Policy Waiver 的本质是：

> 把“这次特殊情况可以”从一句聊天记录，升级成可审计、可过期、可撤销、可精确匹配的授权对象。

成熟 Agent 的策略系统不是永远不通融，而是：

- 默认严格；
- 例外明确；
- 范围最小；
- 自动过期；
- 全程留证；
- 可随时撤销。

一句话：**策略例外不是后门，是带锁、带监控、带过期时间的一次性通行证。**
