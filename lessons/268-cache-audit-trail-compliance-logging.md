# 268. Agent 缓存审计日志与合规留痕（Cache Audit Trail & Compliance Logging）

> 缓存系统一旦参与 Agent 决策，就不只是性能组件，而是事实供应链的一部分。成熟 Agent 不只要知道“命中了缓存”，还要能回答：谁在什么时候、基于什么权限、读取了哪份缓存，并把它用于什么决策。

前面我们讲了缓存的命名、权限、生命周期、迁移、刷新、成本预算。今天补上缓存体系里最容易被忽略，但生产事故复盘时最关键的一环：**缓存审计日志与合规留痕**。

一句话：**cache hit 也要可追责**。

## 1. 为什么缓存也需要审计？

很多团队只审计数据库写操作，却不审计缓存读取。对普通 Web 来说这也许还能接受；对 Agent 来说风险更高：

- Agent 可能用缓存内容生成对外回答。
- Agent 可能基于缓存内容执行部署、发消息、发邮件等副作用。
- 缓存可能跨 session、跨 actor、跨 tenant 复用。
- 缓存内容可能来自网页、群聊、LLM 摘要、RAG 检索等不同可信度来源。
- 出事故时，单看最终回答无法知道“错误事实从哪里来”。

所以缓存层至少要记录四类问题：

```text
who    = actor / tenant / session / run_id
what   = cache_key / key_class / content_hash / source_version
why    = purpose / decision_id / tool_call_id
result = hit | miss | stale_hit | denied | bypass | evicted
```

这不是为了多写日志，而是为了在事故后能还原事实链路。

## 2. Audit Event：缓存审计事件结构

建议每次缓存读写都生成一条结构化事件：

```json
{
  "event": "cache.hit",
  "runId": "run_20260508_1530",
  "actorId": "telegram:67431246",
  "tenantId": "personal",
  "sessionId": "cron:agent-course",
  "toolCallId": "tool_018",
  "purpose": "answer",
  "keyClass": "lesson:index",
  "cacheKeyHash": "sha256:9a7c...",
  "contentHash": "sha256:41bc...",
  "sourceVersion": "git:df49208",
  "trust": "verified",
  "stalenessMs": 12042,
  "decision": "allow",
  "ts": "2026-05-08T05:30:12.000Z"
}
```

注意两个细节：

1. `cacheKeyHash` 通常用 hash，不直接把完整 key 写入日志，避免泄露 PII。
2. `contentHash` 记录值的摘要，不记录原文，既能追踪版本，又能降低敏感数据暴露。

## 3. learn-claude-code：Python 教学版

```python
# learn-claude-code/cache_audit.py
from __future__ import annotations

import hashlib
import json
from dataclasses import dataclass, asdict
from datetime import datetime, timezone
from pathlib import Path
from typing import Literal


def sha256_text(value: str) -> str:
    return "sha256:" + hashlib.sha256(value.encode("utf-8")).hexdigest()


@dataclass(frozen=True)
class CacheAuditEvent:
    event: Literal["cache.hit", "cache.miss", "cache.stale_hit", "cache.denied", "cache.set"]
    run_id: str
    actor_id: str
    tenant_id: str
    purpose: Literal["answer", "planning", "external_side_effect", "security_decision"]
    key_class: str
    cache_key_hash: str
    content_hash: str | None
    decision: Literal["allow", "deny", "bypass"]
    reason: str
    ts: str


class CacheAuditLog:
    def __init__(self, path: str = "memory/cache-audit.jsonl") -> None:
        self.path = Path(path)
        self.path.parent.mkdir(parents=True, exist_ok=True)

    def write(self, event: CacheAuditEvent) -> None:
        with self.path.open("a", encoding="utf-8") as f:
            f.write(json.dumps(asdict(event), ensure_ascii=False) + "\n")


class AuditedCache:
    def __init__(self, audit: CacheAuditLog) -> None:
        self.store: dict[str, str] = {}
        self.audit = audit

    def get(
        self,
        key: str,
        *,
        run_id: str,
        actor_id: str,
        tenant_id: str,
        purpose: Literal["answer", "planning", "external_side_effect", "security_decision"],
    ) -> str | None:
        value = self.store.get(key)
        event_name = "cache.hit" if value is not None else "cache.miss"

        self.audit.write(CacheAuditEvent(
            event=event_name,
            run_id=run_id,
            actor_id=actor_id,
            tenant_id=tenant_id,
            purpose=purpose,
            key_class=key.split(":", 1)[0],
            cache_key_hash=sha256_text(key),
            content_hash=sha256_text(value) if value is not None else None,
            decision="allow",
            reason="read-through cache lookup",
            ts=datetime.now(timezone.utc).isoformat(),
        ))
        return value

    def set(self, key: str, value: str, *, run_id: str, actor_id: str, tenant_id: str) -> None:
        self.store[key] = value
        self.audit.write(CacheAuditEvent(
            event="cache.set",
            run_id=run_id,
            actor_id=actor_id,
            tenant_id=tenant_id,
            purpose="planning",
            key_class=key.split(":", 1)[0],
            cache_key_hash=sha256_text(key),
            content_hash=sha256_text(value),
            decision="allow",
            reason="cache write after verified source read",
            ts=datetime.now(timezone.utc).isoformat(),
        ))
```

教学版关键点：审计逻辑在 cache wrapper 里，不依赖 LLM 自觉写日志。只要 Agent 读缓存，事件就自动落盘。

## 4. pi-mono：TypeScript 生产版审计中间件

生产系统建议把审计做成 cache middleware，并把事件写到 append-only sink：Kafka、ClickHouse、S3/R2 JSONL、Postgres audit table 都可以。

```ts
// pi-mono/packages/agent-runtime/cache/cache-audit-middleware.ts
import { createHash } from "node:crypto";

export type CachePurpose =
  | "answer"
  | "planning"
  | "external_side_effect"
  | "security_decision";

export interface CacheAuditContext {
  runId: string;
  sessionId: string;
  actorId: string;
  tenantId: string;
  toolCallId?: string;
  purpose: CachePurpose;
}

export interface CacheAuditEvent {
  type: "cache.hit" | "cache.miss" | "cache.stale_hit" | "cache.denied" | "cache.set";
  runId: string;
  sessionId: string;
  actorId: string;
  tenantId: string;
  toolCallId?: string;
  purpose: CachePurpose;
  keyClass: string;
  cacheKeyHash: string;
  contentHash?: string;
  sourceVersion?: string;
  trust?: "trusted" | "verified" | "untrusted" | "derived";
  stalenessMs?: number;
  decision: "allow" | "deny" | "bypass";
  reason: string;
  ts: string;
}

export interface AuditSink {
  append(event: CacheAuditEvent): Promise<void>;
}

function hash(value: string): string {
  return "sha256:" + createHash("sha256").update(value).digest("hex");
}

function keyClassOf(key: string): string {
  return key.split(":", 1)[0] ?? "unknown";
}

export class CacheAuditMiddleware {
  constructor(private readonly sink: AuditSink) {}

  async recordRead(input: {
    ctx: CacheAuditContext;
    key: string;
    value?: string;
    hit: boolean;
    stale?: boolean;
    sourceVersion?: string;
    trust?: CacheAuditEvent["trust"];
    stalenessMs?: number;
  }): Promise<void> {
    const type = input.hit
      ? input.stale
        ? "cache.stale_hit"
        : "cache.hit"
      : "cache.miss";

    await this.sink.append({
      type,
      runId: input.ctx.runId,
      sessionId: input.ctx.sessionId,
      actorId: input.ctx.actorId,
      tenantId: input.ctx.tenantId,
      toolCallId: input.ctx.toolCallId,
      purpose: input.ctx.purpose,
      keyClass: keyClassOf(input.key),
      cacheKeyHash: hash(input.key),
      contentHash: input.value ? hash(input.value) : undefined,
      sourceVersion: input.sourceVersion,
      trust: input.trust,
      stalenessMs: input.stalenessMs,
      decision: "allow",
      reason: input.stale ? "served stale under policy" : "cache read",
      ts: new Date().toISOString(),
    });
  }

  async recordDenied(input: {
    ctx: CacheAuditContext;
    key: string;
    reason: string;
  }): Promise<void> {
    await this.sink.append({
      type: "cache.denied",
      runId: input.ctx.runId,
      sessionId: input.ctx.sessionId,
      actorId: input.ctx.actorId,
      tenantId: input.ctx.tenantId,
      toolCallId: input.ctx.toolCallId,
      purpose: input.ctx.purpose,
      keyClass: keyClassOf(input.key),
      cacheKeyHash: hash(input.key),
      decision: "deny",
      reason: input.reason,
      ts: new Date().toISOString(),
    });
  }
}
```

这层 middleware 要放在 `SecureCache` 和 `CachePolicy` 之后，原因是审计事件必须记录最终策略决策：是允许、拒绝、旁路，还是 stale 兜底。

## 5. OpenClaw：文件式审计最小落地

OpenClaw 里很多长期任务天然适合文件式审计：cron、heartbeat、课程发布、GitHub 自动化、运维检查。

可以约定：

```text
memory/audit/cache/YYYY-MM-DD.jsonl
```

每条 JSONL 只写脱敏信息：

```json
{"type":"cache.hit","runId":"cron:3eba6ee3","purpose":"planning","keyClass":"agent-course:index","cacheKeyHash":"sha256:...","contentHash":"sha256:...","ts":"2026-05-08T05:30:00Z"}
```

最终汇报或事故复盘时，只需要串起来：

```text
inbox command -> tool calls -> cache audit -> evidence -> outbox delivery receipt
```

这就能回答：这次 Agent 为什么发了这条消息？它读了哪些缓存？缓存是否过期？来源是否可信？

## 6. 审计日志的 5 条规则

1. **Append-only**：审计日志只追加，不原地修改。
2. **Hash first**：key/value 默认写 hash，不写原文。
3. **Purpose required**：每次读取必须带 purpose，否则拒绝读取高风险缓存。
4. **Denied 也记录**：被策略拒绝的读取比成功命中更有排查价值。
5. **Retention 分级**：普通性能缓存留 7~30 天，安全/副作用决策缓存留更久。

## 7. 常见反模式

- 只记录 miss，不记录 hit：事故里最重要的往往是错误 hit。
- 日志写完整 key：可能把用户 ID、邮箱、查询语句泄露到日志系统。
- 审计依赖 LLM 总结：模型忘了写，审计就断了。
- 没有 purpose：同一份缓存用于回答和用于转账决策，风险完全不同。
- 审计日志可被普通任务覆盖：出了事故无法相信日志本身。

## 8. 课堂小结

缓存审计不是“合规团队的形式主义”，而是 Agent 可追责架构的一部分。

- Cache key 解决“这是谁”。
- Cache policy 解决“能不能用”。
- Cache budget 解决“值不值得”。
- Cache audit trail 解决“用过以后能不能解释清楚”。

成熟 Agent 的缓存层，不只是让系统更快，而是让事实来源、权限边界和最终决策都能被复盘。

一句话收尾：**没有审计的 cache hit，只是一次无法证明来源的幸运猜测。**
