# 241. Agent 证据冲突处理与事实仲裁（Evidence Conflict Resolution & Fact Arbitration）

> 核心观点：Agent 查到两份互相矛盾的信息时，不能“挑一个看起来顺眼的”。生产级 Agent 要把证据结构化，按来源可信度、新鲜度、权限、直接性打分；如果冲突影响外部副作用，就必须刷新、降级或请人确认。

很多 Agent 出事故，不是因为完全没查资料，而是因为**查到了互相打架的资料，却装作没有冲突**。

例如：

```text
GitHub PR 状态：open
本地记忆文件：PR 已 merge
CI 页面缓存：checks pending
用户要求：push 更新这个 PR
```

如果 Agent 只相信本地记忆，可能会继续往已合并分支 push；如果只相信旧页面缓存，也可能错过真实 CI 失败。正确做法是：先识别冲突，再按规则仲裁。

---

## 1. 把“证据”变成结构，而不是散落文本

最小证据模型：

```ts
type Evidence = {
  id: string;
  source: "user" | "tool" | "memory" | "web" | "cache";
  claim: string;
  subject: string;          // pr:66 / order:123 / deployment:abc
  value: unknown;           // open / merged / failed / amount=100
  observedAt: string;       // 什么时候观察到
  freshnessSec: number;     // 允许多久内有效
  confidence: number;       // 0..1
  directness: "primary" | "derived" | "hearsay";
};
```

重点不是字段多，而是让 Agent 能回答三个问题：

- 这条信息从哪来？
- 现在还新鲜吗？
- 它和别的信息有没有冲突？

---

## 2. learn-claude-code：Python 教学版仲裁器

先定义证据和冲突检测：

```python
from dataclasses import dataclass
from datetime import datetime, timezone

@dataclass(frozen=True)
class Evidence:
    id: str
    source: str          # user/tool/memory/cache
    subject: str         # pr:66
    field: str           # state
    value: str           # open/merged
    observed_at: datetime
    max_age_sec: int
    confidence: float
    directness: str      # primary/derived/hearsay


def is_fresh(e: Evidence, now: datetime) -> bool:
    age = (now - e.observed_at).total_seconds()
    return age <= e.max_age_sec


def find_conflicts(items: list[Evidence]) -> list[tuple[Evidence, Evidence]]:
    conflicts = []
    for i, a in enumerate(items):
        for b in items[i + 1:]:
            same_fact = a.subject == b.subject and a.field == b.field
            if same_fact and a.value != b.value:
                conflicts.append((a, b))
    return conflicts
```

再写一个简单评分规则：

```python
SOURCE_WEIGHT = {
    "tool": 1.0,      # 直接 API / DB 查询
    "user": 0.9,      # 用户明确输入
    "memory": 0.5,    # 可能过期
    "cache": 0.4,
}

DIRECTNESS_WEIGHT = {
    "primary": 1.0,
    "derived": 0.7,
    "hearsay": 0.3,
}


def score(e: Evidence, now: datetime) -> float:
    if not is_fresh(e, now):
        return 0.0
    return (
        e.confidence
        * SOURCE_WEIGHT.get(e.source, 0.2)
        * DIRECTNESS_WEIGHT.get(e.directness, 0.2)
    )


def arbitrate(items: list[Evidence], now: datetime) -> Evidence | None:
    fresh = [e for e in items if is_fresh(e, now)]
    if not fresh:
        return None
    return max(fresh, key=lambda e: score(e, now))
```

这不是要让打分变得“玄学”，而是把规则显式化：

- GitHub API 比本地记忆更可信；
- 当前查询比昨天的缓存更可信；
- 高风险操作前，过期证据直接归零。

---

## 3. pi-mono：生产版 Fact Store + Conflict Policy

生产系统可以把每个工具结果登记到 `FactStore`：

```ts
type FactKey = `${string}:${string}`; // subject:field，例如 pr:66:state

class FactStore {
  private facts = new Map<FactKey, Evidence[]>();

  add(evidence: Evidence) {
    const key = `${evidence.subject}:${evidence.field}` as FactKey;
    const list = this.facts.get(key) ?? [];
    list.push(evidence);
    this.facts.set(key, list);
  }

  get(subject: string, field: string) {
    return this.facts.get(`${subject}:${field}` as FactKey) ?? [];
  }
}
```

工具中间件自动登记证据：

```ts
function withEvidenceCapture(tool: ToolDef): ToolDef {
  return {
    ...tool,
    async execute(args, ctx) {
      const result = await tool.execute(args, ctx);

      for (const fact of tool.extractFacts?.(result, args) ?? []) {
        ctx.factStore.add({
          ...fact,
          source: "tool",
          observedAt: new Date().toISOString(),
          confidence: fact.confidence ?? 0.95,
          directness: "primary",
        });
      }

      return result;
    },
  };
}
```

以 GitHub PR 为例：

```ts
const getPullRequest = withEvidenceCapture({
  name: "github_pr_view",
  async execute(args, ctx) {
    return ctx.github.prView(args.repo, args.number);
  },
  extractFacts(result, args) {
    return [
      {
        id: `gh-pr-${args.number}-state`,
        subject: `pr:${args.number}`,
        field: "state",
        value: result.state,
        freshnessSec: 60,
      },
      {
        id: `gh-pr-${args.number}-mergeable`,
        subject: `pr:${args.number}`,
        field: "mergeable",
        value: result.mergeable,
        freshnessSec: 60,
      },
    ];
  },
});
```

这样 Agent 不只是拿到一坨 JSON，而是得到可比较、可刷新、可审计的事实。

---

## 4. 冲突策略：按风险决定动作

冲突不一定都要打断用户。关键看风险：

```ts
type ConflictAction = "use_best" | "refresh" | "ask_human" | "abort";

function decideConflictAction(input: {
  conflicts: EvidenceConflict[];
  risk: "low" | "medium" | "high" | "critical";
}): ConflictAction {
  if (input.conflicts.length === 0) return "use_best";

  if (input.risk === "critical") return "abort";
  if (input.risk === "high") return "ask_human";
  if (input.risk === "medium") return "refresh";
  return "use_best";
}
```

例子：

- 低风险：天气 API 和缓存温度差 1 度 → 用最新 primary source；
- 中风险：README 目录和 lessons 文件数量不一致 → 重新扫描文件；
- 高风险：PR 状态冲突但准备 push → 重新查 GitHub，仍冲突就问人；
- Critical：付款金额、用户权限、删除资源信息冲突 → 直接停止外部副作用。

---

## 5. OpenClaw 实战：push 前为什么要 live check

OpenClaw 里最典型的冲突场景是：

```text
MEMORY.md：PR #66 open
GitHub API：PR #66 merged
用户指令：继续 push
```

正确流程：

```text
1. 从 MEMORY.md 读到旧事实，只当 background context
2. push 前调用 gh pr list / gh pr view 做 primary source refresh
3. 如果发现 PR 已 merge：禁止继续 push 旧分支
4. checkout main → pull → 新建分支 → 新 PR
```

这就是证据仲裁的价值：记忆负责连续性，工具负责实时真相；当两者冲突时，**实时 primary source 赢**。

---

## 6. Prompt 里要明确告诉模型怎么处理冲突

可以把策略写进系统提示词，但不要只靠提示词：

```text
When evidence conflicts:
1. Prefer fresh primary-source tool results over memory/cache.
2. Before external side effects, refresh critical facts.
3. If conflict remains and risk is high, ask the operator.
4. Never hide uncertainty in the final answer.
```

然后在代码层用 `FactStore / ConflictPolicy / PreflightGuard` 强制执行。

Prompt 是行为指南，PreflightGuard 是安全带。

---

## 7. 常见坑

### 坑 1：把 memory 当数据库

Memory 是经验，不是实时状态。PR、订单、余额、部署状态，都必须在执行前刷新。

### 坑 2：只记录最终结论，不记录证据

不要只写：

```json
{"prState":"open"}
```

要写：

```json
{
  "subject": "pr:66",
  "field": "state",
  "value": "open",
  "source": "github_api",
  "observedAt": "2026-05-05T06:30:00+11:00"
}
```

没有来源和时间的事实，很快就会变成谣言。

### 坑 3：冲突时强行输出确定语气

如果证据真的冲突，要说清楚：

```text
我查到两个状态不一致：本地记录说 open，GitHub API 说 merged。
我以 GitHub API 为准，已停止往旧分支 push。
```

这比“应该没问题”安全得多。

---

## 8. 小结

生产级 Agent 的事实处理可以用一句话概括：

> 事实不是文本，是带来源、时间、置信度和风险策略的证据集合。

落地时记住四件事：

1. 工具结果进入 `FactStore`；
2. 同一 subject/field 自动检测冲突；
3. 高风险副作用前刷新 primary source；
4. 冲突无法消除时，宁可暂停，也不要假装确定。

Agent 真正靠谱，不是永远有答案，而是知道什么时候证据不够、什么时候必须重新查。