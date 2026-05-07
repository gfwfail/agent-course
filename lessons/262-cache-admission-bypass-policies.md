# 262. Agent 缓存准入与旁路策略（Cache Admission & Bypass Policies）

前面几课我们把缓存生命周期讲了一圈：失效、击穿、预热、可观测、一致性、权限、淘汰、多级缓存、回源预算。今天补一个更前置的问题：**不是所有东西都应该进缓存**。

一句话：

> Cache Admission 决定“这个结果值不值得写入缓存”；Bypass Policy 决定“这次请求该不该绕过缓存”。成熟 Agent 的缓存不是看到结果就 set，而是先判断价值、风险、新鲜度、权限和复用概率。

很多缓存事故不是因为没缓存，而是因为缓存了不该缓存的东西：

- 一次性错误结果被缓存，后续所有回答都错；
- 用户 A 的个性化结果被错误复用给用户 B；
- 高风险副作用前用了缓存，错过真实状态变化；
- 低复用的大结果塞满缓存，把真正热点挤出去；
- LLM 生成的中间推理被缓存，后续 run 继承了不完整上下文。

所以缓存系统需要两个门：

```text
read path:  request -> BypassPolicy -> cache/source
write path: source result -> AdmissionPolicy -> cache or drop
```

---

## 1. 什么时候应该 Bypass Cache

Bypass 不是“缓存坏了才不用”，而是正常策略的一部分。

常见旁路条件：

1. **用户明确要求实时**：例如“现在”“最新”“刚刚”。
2. **外部副作用前复核**：发消息、删除资源、部署、扣款前必须 live check。
3. **权限/租户范围不稳定**：actor、scope、tenant 刚变化时，先绕过共享缓存。
4. **高风险安全判断**：审批、密钥、权限、账单异常不能依赖 stale。
5. **缓存污染嫌疑**：来源 trust 低、validation 失败、命中异常激增。
6. **调试模式**：开发者需要观察真实工具返回。

可以把请求用途分成几类：

```text
answer                 -> 可用 fresh cache，低风险可 stale
planning               -> 可用 cache，但执行前要复核
external_side_effect   -> 默认 bypass，必须 live check
security_decision      -> 默认 bypass，失败就 block
bulk_background        -> 优先 cache/stale，避免打源站
```

关键点：**Bypass 也要有预算**。如果所有请求都绕过缓存，上一课的 Miss Budget 会立刻被打爆。

---

## 2. 什么时候应该 Admission Cache

Admission 是写入前的门禁。一个结果可以被工具成功返回，但不代表值得缓存。

建议至少检查这些字段：

- `reusable`：是否会被多个 run/用户复用；
- `scope`：是否绑定 actor/tenant/scope；
- `risk`：缓存错误是否会导致外部副作用或泄露；
- `size`：是否过大，需要压缩或 rawPointer；
- `freshness`：是否很快过期；
- `confidence`：结果是否完整可信；
- `errorType`：错误是否适合 negative cache；
- `costToRecompute`：重新获取是否昂贵；
- `frequency`：是否有重复访问迹象。

一个简单规则：

```text
高复用 + 高回源成本 + 低风险 + 可验证 -> cache
低复用 + 大体积 + 个性化 + 高风险 -> don't cache
暂时性 5xx/timeout -> 通常不 cache 或极短 negative cache
确定性 404/not_found -> 可短 TTL negative cache
```

很多系统会用 TinyLFU 思路：**不是第一次看到就缓存，而是访问频率达到门槛才 admission**。Agent 场景里可以简化成“candidate counter”。

---

## 3. learn-claude-code：Python 教学版

下面是一个最小可用的准入 + 旁路策略。重点不是复杂算法，而是把“能不能读缓存、能不能写缓存”变成可测试的纯函数。

```python
# learn_claude_code/cache_admission.py
from __future__ import annotations

import time
from dataclasses import dataclass
from enum import Enum
from typing import Any

class Purpose(str, Enum):
    ANSWER = "answer"
    PLANNING = "planning"
    EXTERNAL_SIDE_EFFECT = "external_side_effect"
    SECURITY_DECISION = "security_decision"
    BULK_BACKGROUND = "bulk_background"

@dataclass(frozen=True)
class CacheRequest:
    key: str
    actor: str
    tenant: str
    scope_hash: str
    purpose: Purpose
    user_asked_realtime: bool = False
    debug_live: bool = False
    risk: str = "low"  # low | medium | high

@dataclass(frozen=True)
class ToolResult:
    value: Any
    source: str
    confidence: float
    reusable: bool
    personalized: bool
    size_bytes: int
    ttl_seconds: int
    error_type: str | None = None  # not_found | rate_limited | timeout | server_error

@dataclass(frozen=True)
class PolicyDecision:
    allow: bool
    reason: str
    ttl_seconds: int | None = None
    cache_scope: str | None = None

class BypassPolicy:
    def decide(self, req: CacheRequest) -> PolicyDecision:
        if req.debug_live:
            return PolicyDecision(False, "debug_live")
        if req.user_asked_realtime:
            return PolicyDecision(False, "user_requested_realtime")
        if req.purpose in {Purpose.EXTERNAL_SIDE_EFFECT, Purpose.SECURITY_DECISION}:
            return PolicyDecision(False, f"{req.purpose.value}_requires_live_check")
        if req.risk == "high" and req.purpose != Purpose.BULK_BACKGROUND:
            return PolicyDecision(False, "high_risk_requires_fresh_source")
        return PolicyDecision(True, "cache_read_allowed")

class AdmissionPolicy:
    def __init__(self) -> None:
        self.frequency: dict[str, int] = {}

    def decide(self, req: CacheRequest, result: ToolResult) -> PolicyDecision:
        self.frequency[req.key] = self.frequency.get(req.key, 0) + 1

        if result.ttl_seconds <= 0:
            return PolicyDecision(False, "zero_ttl")
        if result.confidence < 0.75:
            return PolicyDecision(False, "low_confidence")
        if req.purpose in {Purpose.EXTERNAL_SIDE_EFFECT, Purpose.SECURITY_DECISION}:
            return PolicyDecision(False, "high_risk_purpose_not_cacheable")
        if result.error_type in {"timeout", "server_error", "rate_limited"}:
            return PolicyDecision(False, "transient_error_not_cacheable")
        if result.error_type == "not_found":
            return PolicyDecision(True, "negative_cache_not_found", ttl_seconds=min(result.ttl_seconds, 60))
        if result.personalized:
            scope = f"tenant:{req.tenant}:actor:{req.actor}:scope:{req.scope_hash}"
        else:
            scope = f"tenant:{req.tenant}:shared"
        if result.size_bytes > 128_000 and self.frequency[req.key] < 3:
            return PolicyDecision(False, "large_low_frequency_result")
        if not result.reusable and self.frequency[req.key] < 2:
            return PolicyDecision(False, "low_reuse_candidate")
        return PolicyDecision(True, "admitted", ttl_seconds=result.ttl_seconds, cache_scope=scope)

class PolicyCache:
    def __init__(self) -> None:
        self.store: dict[str, tuple[Any, float, str]] = {}
        self.bypass = BypassPolicy()
        self.admission = AdmissionPolicy()

    def scoped_key(self, req: CacheRequest, scope: str) -> str:
        return f"{scope}:{req.key}"

    def get(self, req: CacheRequest) -> Any | None:
        read = self.bypass.decide(req)
        if not read.allow:
            return None

        # 真实系统会从 envelope 读取 scope；教学版尝试 shared + actor 两种。
        prefixes = [
            f"tenant:{req.tenant}:actor:{req.actor}:scope:{req.scope_hash}",
            f"tenant:{req.tenant}:shared",
        ]
        now = time.time()
        for prefix in prefixes:
            item = self.store.get(self.scoped_key(req, prefix))
            if item is None:
                continue
            value, expires_at, _reason = item
            if expires_at > now:
                return value
        return None

    def maybe_set(self, req: CacheRequest, result: ToolResult) -> PolicyDecision:
        decision = self.admission.decide(req, result)
        if not decision.allow:
            return decision
        assert decision.cache_scope is not None
        assert decision.ttl_seconds is not None
        key = self.scoped_key(req, decision.cache_scope)
        self.store[key] = (result.value, time.time() + decision.ttl_seconds, decision.reason)
        return decision
```

使用方式：

```python
cache = PolicyCache()

req = CacheRequest(
    key="github:repo:gfwfail/agent-course:readme",
    actor="bot001",
    tenant="owner",
    scope_hash="repo-read",
    purpose=Purpose.ANSWER,
)

cached = cache.get(req)
if cached is None:
    result = ToolResult(
        value={"latestLesson": 262},
        source="github",
        confidence=0.95,
        reusable=True,
        personalized=False,
        size_bytes=512,
        ttl_seconds=300,
    )
    decision = cache.maybe_set(req, result)
    print(decision.reason)
```

这个例子虽然小，但已经有生产意识：

- read path 先经过 BypassPolicy；
- write path 先经过 AdmissionPolicy；
- personalized 结果只能进 actor-scoped cache；
- transient error 不缓存；
- `not_found` 可以短 TTL negative cache；
- 大结果必须有访问频率证据才缓存。

---

## 4. pi-mono：中间件生产版

在 pi-mono 这种 TypeScript 生产架构里，不建议业务代码到处写 `if (cacheable)`。更好的做法是把准入和旁路放进 Tool/Cache Middleware。

```ts
// pi-mono/cache/cache-policy.ts
export type CachePurpose =
  | 'answer'
  | 'planning'
  | 'external_side_effect'
  | 'security_decision'
  | 'bulk_background';

export interface CacheContext {
  key: string;
  actorId: string;
  tenantId: string;
  scopeHash: string;
  purpose: CachePurpose;
  risk: 'low' | 'medium' | 'high';
  realtimeRequested?: boolean;
  debugLive?: boolean;
}

export interface ResultEnvelope<T> {
  value: T;
  confidence: number;
  reusable: boolean;
  personalized: boolean;
  sizeBytes: number;
  ttlMs: number;
  errorType?: 'not_found' | 'timeout' | 'server_error' | 'rate_limited';
  tags: string[];
  observedAt: string;
}

export type CacheDecision =
  | { action: 'read'; reason: string }
  | { action: 'bypass'; reason: string }
  | { action: 'admit'; reason: string; ttlMs: number; scope: string }
  | { action: 'drop'; reason: string };

export class CachePolicy {
  private frequency = new Map<string, number>();

  decideRead(ctx: CacheContext): CacheDecision {
    if (ctx.debugLive) return { action: 'bypass', reason: 'debug_live' };
    if (ctx.realtimeRequested) return { action: 'bypass', reason: 'realtime_requested' };
    if (ctx.purpose === 'external_side_effect') {
      return { action: 'bypass', reason: 'side_effect_requires_live' };
    }
    if (ctx.purpose === 'security_decision') {
      return { action: 'bypass', reason: 'security_requires_live' };
    }
    if (ctx.risk === 'high') return { action: 'bypass', reason: 'high_risk' };
    return { action: 'read', reason: 'cache_read_allowed' };
  }

  decideWrite<T>(ctx: CacheContext, result: ResultEnvelope<T>): CacheDecision {
    this.frequency.set(ctx.key, (this.frequency.get(ctx.key) ?? 0) + 1);
    const seen = this.frequency.get(ctx.key)!;

    if (result.ttlMs <= 0) return { action: 'drop', reason: 'zero_ttl' };
    if (result.confidence < 0.75) return { action: 'drop', reason: 'low_confidence' };
    if (ctx.purpose === 'external_side_effect' || ctx.purpose === 'security_decision') {
      return { action: 'drop', reason: 'unsafe_purpose' };
    }
    if (['timeout', 'server_error', 'rate_limited'].includes(result.errorType ?? '')) {
      return { action: 'drop', reason: 'transient_error' };
    }
    if (result.errorType === 'not_found') {
      return {
        action: 'admit',
        reason: 'negative_cache_not_found',
        ttlMs: Math.min(result.ttlMs, 60_000),
        scope: this.scope(ctx, result),
      };
    }
    if (result.sizeBytes > 256_000 && seen < 3) {
      return { action: 'drop', reason: 'large_low_frequency' };
    }
    if (!result.reusable && seen < 2) {
      return { action: 'drop', reason: 'not_enough_reuse_signal' };
    }
    return { action: 'admit', reason: 'admitted', ttlMs: result.ttlMs, scope: this.scope(ctx, result) };
  }

  private scope<T>(ctx: CacheContext, result: ResultEnvelope<T>): string {
    if (result.personalized) {
      return `tenant:${ctx.tenantId}:actor:${ctx.actorId}:scope:${ctx.scopeHash}`;
    }
    return `tenant:${ctx.tenantId}:shared`;
  }
}
```

中间件接入：

```ts
// pi-mono/cache/cache-middleware.ts
export function createCacheMiddleware(policy: CachePolicy, cache: CacheStore): ToolMiddleware {
  return async (ctx, next) => {
    const cacheCtx = toCacheContext(ctx);
    const readDecision = policy.decideRead(cacheCtx);

    ctx.audit.push({ type: 'cache.read_decision', ...readDecision, key: cacheCtx.key });

    if (readDecision.action === 'read') {
      const hit = await cache.get(cacheCtx);
      if (hit) {
        return { ...hit.value, meta: { cache: 'hit', observedAt: hit.observedAt } };
      }
    }

    const result = await next(ctx);
    const envelope = toResultEnvelope(result);
    const writeDecision = policy.decideWrite(cacheCtx, envelope);

    ctx.audit.push({ type: 'cache.write_decision', ...writeDecision, key: cacheCtx.key });

    if (writeDecision.action === 'admit') {
      await cache.set({
        key: cacheCtx.key,
        scope: writeDecision.scope,
        value: envelope.value,
        ttlMs: writeDecision.ttlMs,
        tags: envelope.tags,
        observedAt: envelope.observedAt,
      });
    }

    return result;
  };
}
```

生产版有两个重点：

1. 每次 read/write decision 都进 audit，方便解释“为什么没命中/为什么没缓存”；
2. 业务工具只返回 envelope，不直接决定缓存策略，避免策略散落全项目。

---

## 5. OpenClaw 实战：课程 Cron 的缓存策略

拿我们这个课程 cron 举例，每 3 小时要做几件事：

- 读取 `TOOLS.md` 的已讲列表；
- 读取 `agent-course/README.md` 和 lessons；
- 生成新课；
- 发 Telegram；
- git commit/push；
- 更新记忆。

哪些能缓存？哪些必须旁路？

```text
读取历史课程目录      -> 可 cache，短 TTL；写新课前重新 read 本地文件
检查已讲主题          -> 可 cache summary，但最终选题前读 TOOLS.md live
Telegram 发消息        -> 绝不 cache，Outbox/receipt 而不是 cache
git status / pull      -> 绝不 cache，push 前 live check
GitHub PR 状态         -> push 前 bypass cache，必须 live
课程生成草稿          -> 可暂存 draft，但不能当事实缓存
```

这个策略背后的原则：

- **信息检索**可以缓存；
- **外部副作用**不能靠缓存；
- **最终验收前**必须 live check；
- **草稿缓存**只能作为候选，不能当作已发布事实。

一个 OpenClaw 风格的 checklist：

```text
[read] README / lessons summary: cache allowed
[read] TOOLS 已讲内容: cache allowed for planning, live read before update
[write] lesson file: no cache, direct file write
[send] Telegram: no cache, record messageId
[git] status/pull/diff/push: no cache, live command
[final] update memory: write evidence, not cached conclusion
```

这能避免一个典型 bug：Agent 根据旧 README 以为第 261 课是最新，于是又创建第 262；但另一个 cron 已经提交了第 262。正确做法是：**计划时可用缓存，落盘前必须 live revalidate 文件系统和 git 状态**。

---

## 6. 常见坑

### 坑 1：把“工具成功”当成“可缓存”

HTTP 200 可能只是返回了部分数据；LLM summary 也可能带 caveat。Admission 应该看 confidence/completeness，而不是只看 status code。

### 坑 2：错误结果缓存太久

`not_found` 可以 negative cache，但 timeout/429/5xx 通常不能长期缓存。否则一次源站抖动会变成持续错误。

### 坑 3：忽略个性化范围

只要结果里混入 actor、tenant、permission、locale、channel capability，就不能进全局共享缓存。

### 坑 4：所有实时请求都绕过缓存

Bypass 要接 Miss Budget。否则用户连续问“最新呢？现在呢？”会把源站打穿。

### 坑 5：缓存 LLM 中间结论

可以缓存工具证据、文档片段、结构化摘要；慎重缓存“模型判断”。模型判断应该带 evidence hash，证据变了就失效。

---

## 7. 落地建议

如果你已经有缓存系统，下一步不要急着上复杂算法，先加三张表/三个日志字段：

```text
cache_read_decisions(key, purpose, action, reason, actor, tenant, observed_at)
cache_write_decisions(key, action, reason, ttl, scope, size_bytes, confidence)
cache_candidates(key, seen_count, last_seen_at, admitted_at)
```

然后问三个问题：

1. 哪些 key 经常被 drop？是不是 TTL/size/复用策略不合理？
2. 哪些 key 经常 bypass？是不是业务其实要求实时？
3. 哪些 admitted key 从来没有 hit？是不是污染缓存空间？

先把“为什么缓存/为什么不缓存”讲清楚，再优化命中率。

---

## 8. 小结

今天这课的核心：

- Bypass Policy 控制读路径：这次能不能用缓存；
- Admission Policy 控制写路径：这个结果值不值得缓存；
- 个性化/高风险/外部副作用默认更保守；
- transient error 不应长期缓存，确定性 not_found 可短 TTL negative cache；
- 大结果和低复用结果要有准入门槛；
- 所有 decision 都要审计，方便调试和治理。

一句话收尾：

> 缓存不是越多越好。成熟 Agent 的缓存系统，真正厉害的地方是知道什么时候不缓存、什么时候不用缓存。
