# 235. Agent 工具结果置信度与降级决策（Tool Result Confidence & Degradation Decisions）

> 核心思想：工具调用成功不等于结果可信。生产 Agent 不应该只看 `success: true`，还要看结果的完整性、来源质量、新鲜度和冲突情况，给每个工具结果打 `confidence`，再决定：直接回答、换工具复查、降级输出，还是向用户确认。

很多 Agent 的幻觉不是 LLM 凭空编的，而是“工具给了半截答案，Agent 当成 100% 事实”：

- 搜索 API 只返回了 3 条结果，但 Agent 说“全网都没有”；
- 数据库查询超时后只查到第一页，Agent 当成完整报表；
- 浏览器截图加载失败，Agent 仍然判断 UI 正常；
- 两个工具结果互相冲突，Agent 没有标记不确定性。

所以工具结果要从“布尔成功”升级为“可信度信号”。

---

## 1. 结果信封：success 之外再加 confidence

最小结构：

```ts
type ToolConfidence = 'high' | 'medium' | 'low' | 'unknown'

type ToolResult<T> = {
  success: boolean
  data?: T
  error?: string
  confidence: ToolConfidence
  completeness: 'complete' | 'partial' | 'sampled' | 'unknown'
  observedAt: string
  sources: Array<{
    name: string
    quality: 'primary' | 'secondary' | 'user_provided' | 'unverified'
    url?: string
  }>
  caveats: string[]
  suggestedAction?: 'answer' | 'retry' | 'fallback' | 'ask_user' | 'abort'
}
```

关键点：

- `success=true` 只代表工具执行成功；
- `confidence=high` 才代表 Agent 可以放心用；
- `completeness=partial/sampled` 必须进入最终回答或触发补查；
- `caveats` 是给 LLM 的，不是给人类日志看的，要写得可行动。

---

## 2. learn-claude-code：Python 教学版

在教学版里可以先写一个简单评分器，把工具结果归一化：

```python
# learn_claude_code/tool_confidence.py
from dataclasses import dataclass, field
from datetime import datetime, timezone
from typing import Any, Literal

Confidence = Literal['high', 'medium', 'low', 'unknown']
Completeness = Literal['complete', 'partial', 'sampled', 'unknown']
Action = Literal['answer', 'retry', 'fallback', 'ask_user', 'abort']

@dataclass
class ConfidenceResult:
    success: bool
    data: Any = None
    error: str | None = None
    confidence: Confidence = 'unknown'
    completeness: Completeness = 'unknown'
    observed_at: str = field(default_factory=lambda: datetime.now(timezone.utc).isoformat())
    caveats: list[str] = field(default_factory=list)
    suggested_action: Action = 'ask_user'


def score_search_result(items: list[dict], timed_out: bool, total_estimated: int | None) -> ConfidenceResult:
    if timed_out and not items:
        return ConfidenceResult(
            success=False,
            error='Search timed out before returning usable results',
            confidence='low',
            completeness='unknown',
            caveats=['搜索超时，不能据此判断没有结果'],
            suggested_action='retry',
        )

    if timed_out or (total_estimated and len(items) < min(total_estimated, 10)):
        return ConfidenceResult(
            success=True,
            data=items,
            confidence='medium',
            completeness='partial',
            caveats=['结果可能不完整；回答时必须说明这是部分结果'],
            suggested_action='fallback',
        )

    if len(items) >= 5:
        return ConfidenceResult(
            success=True,
            data=items,
            confidence='high',
            completeness='complete',
            suggested_action='answer',
        )

    return ConfidenceResult(
        success=True,
        data=items,
        confidence='low',
        completeness='sampled',
        caveats=['样本太少，不足以支持强结论'],
        suggested_action='ask_user',
    )
```

Agent Loop 收到工具结果后，不是直接塞给 LLM，而是先做决策：

```python
def route_after_tool(result: ConfidenceResult) -> str:
    if result.suggested_action == 'retry':
        return 'retry_same_tool_with_backoff'
    if result.suggested_action == 'fallback':
        return 'call_secondary_tool'
    if result.suggested_action == 'ask_user':
        return 'ask_one_clarifying_question'
    if result.suggested_action == 'abort':
        return 'stop_and_report_blocker'
    return 'compose_answer'
```

这一步的价值是：LLM 不需要每次重新猜“这个结果能不能信”，工具层已经把风险信号说清楚。

---

## 3. pi-mono：生产版中间件

生产系统里建议把它做成 Tool Middleware：所有工具返回值统一过 `ConfidenceMiddleware`。

```ts
// pi-mono/tooling/confidence.ts
export type ConfidenceLevel = 'high' | 'medium' | 'low' | 'unknown'

export interface NormalizedToolResult<T = unknown> {
  ok: boolean
  data?: T
  error?: string
  meta: {
    confidence: ConfidenceLevel
    completeness: 'complete' | 'partial' | 'sampled' | 'unknown'
    observedAt: string
    caveats: string[]
    suggestedAction: 'answer' | 'retry' | 'fallback' | 'ask_user' | 'abort'
  }
}

export interface ToolContext {
  toolName: string
  elapsedMs: number
  timeoutMs: number
  sourceQuality?: 'primary' | 'secondary' | 'unverified'
}

export function attachConfidence<T>(raw: any, ctx: ToolContext): NormalizedToolResult<T> {
  const caveats: string[] = []
  let confidence: ConfidenceLevel = 'high'
  let completeness: NormalizedToolResult['meta']['completeness'] = 'complete'
  let suggestedAction: NormalizedToolResult['meta']['suggestedAction'] = 'answer'

  if (!raw.ok) {
    return {
      ok: false,
      error: raw.error ?? 'Tool failed without detailed error',
      meta: {
        confidence: 'low',
        completeness: 'unknown',
        observedAt: new Date().toISOString(),
        caveats: ['工具失败，不能把缺失结果解释为否定结论'],
        suggestedAction: raw.retryable ? 'retry' : 'fallback',
      },
    }
  }

  if (ctx.elapsedMs > ctx.timeoutMs * 0.8) {
    confidence = 'medium'
    caveats.push('工具接近超时，结果可能来自降级路径')
  }

  if (raw.partial === true || raw.nextCursor) {
    confidence = confidence === 'high' ? 'medium' : confidence
    completeness = 'partial'
    suggestedAction = 'fallback'
    caveats.push('结果存在分页/截断，不能当作全集')
  }

  if (ctx.sourceQuality === 'unverified') {
    confidence = 'low'
    suggestedAction = 'ask_user'
    caveats.push('来源未验证，只能作为线索，不能作为最终事实')
  }

  return {
    ok: true,
    data: raw.data,
    meta: {
      confidence,
      completeness,
      observedAt: new Date().toISOString(),
      caveats,
      suggestedAction,
    },
  }
}
```

然后在 Agent Loop 做统一路由：

```ts
export async function handleToolResult(result: NormalizedToolResult, agent: AgentRuntime) {
  switch (result.meta.suggestedAction) {
    case 'retry':
      return agent.retryLastTool({ reason: result.meta.caveats.join('; ') })
    case 'fallback':
      return agent.callFallbackTool({ previousResult: result })
    case 'ask_user':
      return agent.askUser('我拿到的结果可信度不够，需要您确认一个关键信息。')
    case 'abort':
      return agent.finishBlocked(result.error ?? '结果不可信，已停止。')
    case 'answer':
      return agent.composeAnswer({ evidence: result })
  }
}
```

生产建议：

- 对每类工具配置默认评分规则：DB、Search、Browser、LLM Judge、Payment API 不一样；
- 所有 `low confidence` 的最终回答必须带限定词；
- 高风险副作用前只接受 `high + complete + primary source`；
- 低可信结果进入观测日志，方便后续调参。

---

## 4. OpenClaw 实战：工具证据不要偷换成事实

OpenClaw 里最常见的场景：`exec`、`web_fetch`、`canvas snapshot`、`message poll` 都是观察，不是永恒事实。

一个安全写法：

```text
观察：git status 当前为空
observedAt: 2026-05-04T02:37:00Z
confidence: high
completeness: complete
scope: /Users/bot001/.openclaw/workspace/agent-course

结论：当前仓库没有未提交改动
限制：只对这个路径、这个时间点成立；push 前仍需重新检查
```

再比如网页检查：

```text
观察：截图中首页 hero 区域加载正常
confidence: medium
completeness: sampled
caveats:
- 只检查了桌面视口
- 没检查登录后页面
- 没覆盖移动端和支付流程
suggestedAction: fallback -> 再跑 mobile viewport / login flow
```

这能避免 Agent 把“我看了一眼桌面首页”说成“网站完全正常”。

---

## 5. 降级策略矩阵

| confidence | completeness | 任务类型 | 推荐动作 |
|---|---|---|---|
| high | complete | 普通问答 | 直接回答 |
| high | partial | 报表/搜索 | 说明范围，必要时继续分页 |
| medium | complete | 低风险任务 | 回答但带 caveat |
| medium | partial | 代码/诊断 | 换第二工具复查 |
| low | any | 外部写操作 | 禁止执行，问用户或重查 |
| unknown | unknown | 任何任务 | 不要假装知道，先补证据 |

一句话规则：

```text
read-only 可以带 caveat；write/external/destructive 必须 high confidence。
```

---

## 6. 常见反模式

### 反模式 1：空结果 = 不存在

```text
搜索结果为空
错误结论：没有相关资料
正确结论：本次搜索没有返回结果，可能是关键词/权限/网络/分页问题
```

### 反模式 2：截图成功 = 功能正常

截图只能证明“某个视口某一刻长这样”，不能证明登录、支付、移动端、错误状态都正常。

### 反模式 3：工具成功 = 可以执行副作用

副作用前要检查：

- 观察是否新鲜；
- 来源是否主数据源；
- 是否完整；
- 是否和其他证据冲突；
- 用户是否授权。

---

## 7. 小结

Agent 的可靠性不是来自“永远相信工具”，而是来自“知道工具结果什么时候不够可靠”。

落地三步：

1. 工具返回值统一加 `confidence/completeness/caveats/suggestedAction`；
2. Agent Loop 根据可信度自动 retry/fallback/ask/abort；
3. 最终回答和外部副作用都受可信度门控。

成熟 Agent 的口头禅不是“我查到了”，而是：“我查到的证据可信度是多少，足够支撑什么动作。”
