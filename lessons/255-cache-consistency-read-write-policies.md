# 255. Agent 缓存一致性与读写策略（Cache Consistency & Read-Write Policies）

前几课我们把缓存的几个大坑补齐了：

- 第 251 课：实体变化时，用 dependency tags 精准失效；
- 第 252 课：缓存过期时，用 SingleFlight + SWR 防击穿；
- 第 253 课：热点数据要预热和治理；
- 第 254 课：缓存要有 hit ratio / rebuild_ms / stale rate 的可观测性。

今天补最后一个生产级问题：**Agent 什么时候可以相信缓存，什么时候必须绕过缓存，写完以后怎么保证自己读到的是新世界？**

这叫：**缓存一致性与读写策略（Cache Consistency & Read-Write Policies）**。

一句话：

> 缓存不是“有就用”。成熟 Agent 会按操作风险、数据类型和读写关系选择 read-through、write-through、write-around、read-your-writes、bypass-cache。

---

## 1. 为什么 Agent 特别容易被缓存一致性坑

普通 Web 服务缓存错了，用户刷新一下可能就好了。

Agent 缓存错了，问题更严重：

1. 它会基于旧数据做决策；
2. 它会把旧数据写进记忆；
3. 它会继续执行外部副作用；
4. 它还可能自信地汇报“已经完成”。

典型事故：

```text
T0  Agent 查询 GitHub PR #12：open，缓存 10 分钟
T1  人类合并了 PR #12
T2  Agent 继续用缓存结果，以为 PR 还 open
T3  Agent 往已合并分支继续 push
```

所以生产 Agent 的缓存层不能只暴露：

```python
cache.get(key)
cache.set(key, value)
```

而应该让调用方声明：

```text
这次读取的风险是什么？
是否在写操作前？
是否刚刚写过相关实体？
是否要求读到自己刚写的结果？
缓存不一致时是 retry、bypass、ask 还是 abort？
```

---

## 2. 五种常用读写策略

### 2.1 read-through：读不到就加载

最常见：

```text
get user:123
  cache hit -> return
  cache miss -> load from DB/API -> set cache -> return
```

适合：

- 低风险查询；
- 文档索引；
- UI 辅助信息；
- 非关键推荐结果。

不适合：

- 付款状态；
- PR merge 状态；
- 发送消息前的目标会话状态；
- 删除/部署/转账前的任何关键前提。

---

### 2.2 write-through：先写真实源，再同步写缓存

```text
update source of truth
update cache
return
```

适合：

- Agent 自己维护的状态；
- 本地任务表；
- RunManifest；
- memory index。

优点：读自己刚写的数据时快。

缺点：写路径更重，缓存写失败时要有补偿策略。

---

### 2.3 write-around：只写真实源，不立刻写缓存

```text
update source of truth
invalidate cache tags
next read rebuilds cache
```

适合：

- 外部系统数据；
- 可能被多人同时改的数据；
- GitHub PR、部署状态、订单状态。

Agent 常用这个，因为外部系统才是真实世界。

---

### 2.4 read-your-writes：同一个 run 内必须读到自己刚写的结果

这是 Agent 很需要的一致性：

```text
Agent 刚创建 issue #88
下一步总结时，不能说“没看到 issue”
```

实现方式：

- 写操作返回 authoritative result；
- run-local overlay 暂存刚写的实体；
- 后续读取同实体时优先 overlay；
- 同时后台 refresh 外部真实状态。

---

### 2.5 bypass-cache：高风险动作前强制绕过缓存

例如：

```text
push 前检查 PR 状态
部署前检查当前环境
发消息前检查收件人/渠道能力
扣款前检查订单状态
删除前检查资源仍属于目标 owner
```

这些场景不是“缓存 TTL 还没过就可以用”。

规则应该是：

```text
if operation.risk in [high, irreversible, external_write]:
    bypass cache and revalidate from source of truth
```

---

## 3. 设计一个 CachePolicy

我们不要让每个工具自己随便决定缓存策略，而是定义统一策略对象：

```python
from dataclasses import dataclass
from enum import Enum

class Consistency(str, Enum):
    EVENTUAL = "eventual"
    READ_YOUR_WRITES = "read_your_writes"
    STRONG_BEFORE_WRITE = "strong_before_write"

class CacheMode(str, Enum):
    READ_THROUGH = "read_through"
    WRITE_THROUGH = "write_through"
    WRITE_AROUND = "write_around"
    BYPASS = "bypass"

@dataclass(frozen=True)
class CachePolicy:
    mode: CacheMode
    consistency: Consistency
    max_age_seconds: int | None = None
    tags: tuple[str, ...] = ()
    reason: str = ""
```

关键点：

- `mode` 决定怎么读写缓存；
- `consistency` 决定是否允许旧值；
- `max_age_seconds` 是业务新鲜度要求，不只是缓存 TTL；
- `tags` 用于写后失效；
- `reason` 用于审计，方便解释为什么绕过缓存。

---

## 4. learn-claude-code：Python 教学版

下面是一个很小但完整的缓存门面。

```python
# learn_claude_code/cache_policy.py
from __future__ import annotations

import time
from dataclasses import dataclass
from enum import Enum
from typing import Any, Callable

class CacheMode(str, Enum):
    READ_THROUGH = "read_through"
    WRITE_THROUGH = "write_through"
    WRITE_AROUND = "write_around"
    BYPASS = "bypass"

class Consistency(str, Enum):
    EVENTUAL = "eventual"
    READ_YOUR_WRITES = "read_your_writes"
    STRONG_BEFORE_WRITE = "strong_before_write"

@dataclass
class CacheEntry:
    value: Any
    stored_at: float
    version: int
    tags: set[str]

@dataclass(frozen=True)
class CachePolicy:
    mode: CacheMode
    consistency: Consistency = Consistency.EVENTUAL
    max_age_seconds: int | None = None
    tags: tuple[str, ...] = ()
    reason: str = ""

class PolicyCache:
    def __init__(self) -> None:
        self.store: dict[str, CacheEntry] = {}
        self.overlay: dict[str, Any] = {}  # run-local read-your-writes
        self.version = 0

    def get(
        self,
        key: str,
        loader: Callable[[], Any],
        policy: CachePolicy,
    ) -> Any:
        if policy.consistency == Consistency.READ_YOUR_WRITES and key in self.overlay:
            return self.overlay[key]

        if policy.mode == CacheMode.BYPASS:
            return loader()

        entry = self.store.get(key)
        if entry and self._fresh_enough(entry, policy):
            return entry.value

        value = loader()
        if policy.mode == CacheMode.READ_THROUGH:
            self.set(key, value, tags=policy.tags)
        return value

    def write(
        self,
        key: str,
        writer: Callable[[], Any],
        policy: CachePolicy,
    ) -> Any:
        result = writer()  # source of truth first

        if policy.consistency == Consistency.READ_YOUR_WRITES:
            self.overlay[key] = result

        if policy.mode == CacheMode.WRITE_THROUGH:
            self.set(key, result, tags=policy.tags)
        elif policy.mode == CacheMode.WRITE_AROUND:
            self.invalidate_tags(policy.tags)

        return result

    def set(self, key: str, value: Any, tags: tuple[str, ...] = ()) -> None:
        self.version += 1
        self.store[key] = CacheEntry(
            value=value,
            stored_at=time.time(),
            version=self.version,
            tags=set(tags),
        )

    def invalidate_tags(self, tags: tuple[str, ...]) -> None:
        if not tags:
            return
        target = set(tags)
        for key, entry in list(self.store.items()):
            if entry.tags & target:
                del self.store[key]

    def _fresh_enough(self, entry: CacheEntry, policy: CachePolicy) -> bool:
        if policy.max_age_seconds is None:
            return True
        return time.time() - entry.stored_at <= policy.max_age_seconds
```

使用示例：

```python
cache = PolicyCache()

# 低风险：允许 read-through
profile = cache.get(
    "user:42:profile",
    loader=lambda: github_api.get_user("octocat"),
    policy=CachePolicy(
        mode=CacheMode.READ_THROUGH,
        max_age_seconds=3600,
        tags=("user:42",),
        reason="profile display only",
    ),
)

# 高风险：push 前强制绕过缓存
pr = cache.get(
    "repo:gfwfail/agent-course:prs:latest",
    loader=lambda: github_api.list_recent_prs(),
    policy=CachePolicy(
        mode=CacheMode.BYPASS,
        consistency=Consistency.STRONG_BEFORE_WRITE,
        reason="pre-push PR state must be live",
    ),
)

# 写后读自己：创建 issue 后本 run 内立刻可见
issue = cache.write(
    "repo:demo:issue:temp",
    writer=lambda: github_api.create_issue(title="bug", body="..."),
    policy=CachePolicy(
        mode=CacheMode.WRITE_AROUND,
        consistency=Consistency.READ_YOUR_WRITES,
        tags=("repo:demo:issues",),
        reason="external GitHub write; invalidate list cache",
    ),
)
```

教学版重点不是性能，而是把“为什么可以用缓存”显式化。

---

## 5. pi-mono：TypeScript 生产版中间件

生产里可以把策略挂到 tool metadata 上。

```ts
// pi-mono/agent/cache/types.ts
export type CacheMode =
  | 'read_through'
  | 'write_through'
  | 'write_around'
  | 'bypass';

export type Consistency =
  | 'eventual'
  | 'read_your_writes'
  | 'strong_before_write';

export interface CachePolicy {
  mode: CacheMode;
  consistency?: Consistency;
  maxAgeMs?: number;
  tags?: string[];
  reason: string;
}

export interface ToolCallContext {
  runId: string;
  toolName: string;
  risk: 'low' | 'medium' | 'high';
  phase: 'observe' | 'plan' | 'pre_effect' | 'effect' | 'finalize';
}
```

工具声明：

```ts
// pi-mono/tools/github.ts
export const listPullRequests = defineTool({
  name: 'github.pr.list',
  risk: 'medium',
  cache: {
    mode: 'read_through',
    consistency: 'eventual',
    maxAgeMs: 60_000,
    tags: ['github:prs'],
    reason: 'PR list is safe for planning, not for pre-push validation',
  },
  async execute(args, ctx) {
    return ctx.github.prList(args);
  },
});

export const gitPushPreflight = defineTool({
  name: 'git.push.preflight',
  risk: 'high',
  cache: {
    mode: 'bypass',
    consistency: 'strong_before_write',
    reason: 'push requires live branch and PR state',
  },
  async execute(args, ctx) {
    return ctx.github.prList({ state: 'all', limit: 5 });
  },
});
```

中间件强制升级策略：

```ts
// pi-mono/agent/cache/CacheConsistencyMiddleware.ts
export function enforceCachePolicy(
  declared: CachePolicy | undefined,
  ctx: ToolCallContext,
): CachePolicy {
  const fallback: CachePolicy = {
    mode: 'bypass',
    consistency: 'strong_before_write',
    reason: 'no cache policy declared for non-low-risk tool',
  };

  if (!declared) {
    return ctx.risk === 'low'
      ? { mode: 'read_through', consistency: 'eventual', maxAgeMs: 30_000, reason: 'default low-risk cache' }
      : fallback;
  }

  if (ctx.phase === 'pre_effect' || ctx.phase === 'effect' || ctx.risk === 'high') {
    return {
      ...declared,
      mode: 'bypass',
      consistency: 'strong_before_write',
      reason: `upgraded before side effect: ${declared.reason}`,
    };
  }

  return declared;
}
```

执行层：

```ts
export async function runToolWithCache<T>(params: {
  key: string;
  policy: CachePolicy;
  ctx: ToolCallContext;
  execute: () => Promise<T>;
  cache: CacheStore;
  overlay: RunOverlay;
}): Promise<T> {
  const policy = enforceCachePolicy(params.policy, params.ctx);

  if (policy.consistency === 'read_your_writes') {
    const value = params.overlay.get<T>(params.ctx.runId, params.key);
    if (value !== undefined) return value;
  }

  if (policy.mode !== 'bypass') {
    const cached = await params.cache.get<T>(params.key);
    if (cached && isFresh(cached, policy.maxAgeMs)) {
      return cached.value;
    }
  }

  const value = await params.execute();

  if (policy.mode === 'read_through' || policy.mode === 'write_through') {
    await params.cache.set(params.key, value, {
      tags: policy.tags ?? [],
      observedAt: Date.now(),
      reason: policy.reason,
    });
  }

  return value;
}
```

这样做的好处：

- LLM 不需要记住缓存规则；
- 高风险阶段自动绕过缓存；
- 工具的缓存行为可测试、可审计；
- 同一个工具在 observe 阶段可缓存，在 effect 前自动 live check。

---

## 6. OpenClaw 实战：课程 cron 的缓存一致性

拿我们这个课程 cron 举例。

它每 3 小时做：

1. 检查已讲内容；
2. 写 lesson 文件；
3. 更新 README；
4. 发 Telegram；
5. git commit && push。

哪些可以缓存？

```text
已讲内容列表：可短缓存，但写课前最好读本地最新 TOOLS.md
课程模板：可长期缓存
GitHub PR 状态：push 前必须 live check
Telegram 发送结果：不能缓存，要保存 messageId 回执
Git commit sha：写后读自己，直接用 git 返回结果
```

一个简单策略表：

| 数据 | 策略 | 原因 |
|---|---|---|
| lessons 目录 | read-through，maxAge 1 分钟 | 规划题目时可接受短暂旧数据 |
| TOOLS.md 已讲内容 | bypass 或 maxAge 0 | 避免重复讲课 |
| Telegram send result | write-around + receipt | 外部副作用，以 provider 返回为准 |
| GitHub PR 状态 | bypass | push 前必须 live check |
| README 渲染检查 | bypass | 最终文件必须读磁盘现状 |

执行前闸门可以写成：

```bash
# pre-push live check：不要用缓存
GH_FORCE_LIVE=1 gh pr list --state all --limit 5

git diff --check
git status --short
```

注意：`GH_FORCE_LIVE=1` 只是示意，真实 `gh` 本来就是 live API；重点是不要拿几分钟前的缓存结果替代它。

---

## 7. 一致性策略决策表

生产里可以从这张表开始：

| 场景 | 建议策略 |
|---|---|
| 低风险展示 | read-through + TTL |
| LLM 上下文补充 | read-through + maxAge + evidence timestamp |
| 用户明确问“现在/最新” | bypass |
| 外部副作用前 | bypass + strong_before_write |
| Agent 自己状态写入 | write-through |
| 外部系统写入 | write-around + invalidate tags |
| 写完马上总结 | read-your-writes overlay |
| 缓存与 live 结果冲突 | live wins，并记录 conflict event |

最重要的原则：

> 缓存只优化观察，不应该削弱执行前复核。

---

## 8. 常见错误

### 错误 1：只看 TTL，不看风险

```text
PR 状态缓存 60 秒，还没过期，所以 push。
```

错。

push 是外部副作用，必须 live check。

---

### 错误 2：写完外部系统后立刻信本地缓存

```text
创建订单成功 -> 本地缓存显示 pending -> Agent 说还没创建
```

正确做法：写操作返回值进入 run overlay，至少保证 read-your-writes。

---

### 错误 3：缓存 key 没有版本和 scope

```text
key = "plans:AU"
```

更好：

```text
key = "tenant:{tenantId}:provider:{provider}:plans:country:AU:v3"
```

缓存一致性经常不是 TTL 问题，而是 key scope 错了。

---

### 错误 4：缓存命中后没有 evidence timestamp

Agent 回答：

```text
PR 是 open。
```

更好的内部证据：

```json
{
  "value": { "number": 12, "state": "open" },
  "source": "cache",
  "observedAt": "2026-05-07T00:29:10+11:00",
  "maxAgeMs": 60000
}
```

高风险动作看到 `source=cache` 就应该停下重新查。

---

## 9. 本课小结

今天的核心不是“怎么写一个缓存”，而是：

1. 缓存策略要显式声明；
2. 高风险副作用前必须 bypass；
3. 外部写通常用 write-around + invalidate；
4. Agent 自己写过的结果要支持 read-your-writes；
5. 所有缓存值都要带 observedAt/source/reason，方便审计和降级。

最终记住一句：

> 会用缓存的 Agent 很快；知道什么时候不用缓存的 Agent 才可靠。
