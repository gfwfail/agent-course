# 275. Agent 缓存来源血缘与可解释回源（Cache Lineage & Explainable Provenance）

> 上一课我们讲了：缓存不仅要会过期，还要能按主体彻底遗忘。今天补一个经常被忽略、但生产环境非常救命的能力：**缓存答案到底从哪里来，后面能不能解释、复查、精准刷新**。

一句话：**Cache value 不是孤立结果，而是由一组来源、转换步骤、策略版本和校验记录派生出来的事实包；没有血缘的缓存命中，只是一次“我也不知道为什么对”的复读。**

## 1. 为什么缓存需要 lineage？

很多 Agent 缓存长这样：

```json
{
  "key": "user:42:billing_summary",
  "value": "本月 AWS 费用上涨主要来自 Shield Advanced"
}
```

这在 demo 里够用，生产里不够：

- 用户问“你这个结论依据是什么？”答不上来；
- 源数据更新了，不知道该刷新哪些派生缓存；
- prompt / parser / policy 改版后，不知道旧缓存还能不能用；
- 合规审计时，只能看到一个 summary，看不到它由哪些工具结果和规则产生；
- 缓存投毒或错误摘要出现后，无法追踪污染范围。

成熟做法：每个缓存条目都带一份 **Lineage Manifest**：

```json
{
  "cacheKey": "agent:v12:billing:summary:tenant:metadraw:2026-05",
  "valueHash": "sha256:8a71...",
  "derivedFrom": [
    {
      "sourceType": "tool_result",
      "sourceId": "aws-cost-explorer:2026-05-01..2026-05-08",
      "contentHash": "sha256:11af...",
      "freshness": "2026-05-09T12:30:00+11:00"
    },
    {
      "sourceType": "policy",
      "sourceId": "cost-anomaly-policy:v4",
      "contentHash": "sha256:99be..."
    }
  ],
  "transforms": [
    "normalize_currency:v2",
    "group_by_linked_account:v3",
    "llm_summarize:prompt:v8"
  ],
  "confidence": 0.91,
  "explainable": true
}
```

关键点：**缓存命中时不只返回 value，还返回“为什么这个 value 可信”。**

## 2. 三类血缘必须记录

### 2.1 来源血缘：从哪些事实来

包括：

- 工具结果：API response、DB rows、文件、网页、邮件；
- 用户输入：明确指令、偏好、审批；
- 记忆/配置：MEMORY、TOOLS、policy 文件；
- 其他缓存：派生缓存依赖的上游缓存 key。

来源至少要有 `sourceId + contentHash + observedAt`。不要只记录 URL 或 key，因为同一个 URL 下一秒内容可能已经变了。

### 2.2 转换血缘：怎么加工出来

包括：

- parser 版本；
- prompt template 版本；
- policy 版本；
- redaction / projection 版本；
- aggregation 逻辑版本。

这样 prompt 改版后，可以批量标记旧 summary 为 `stale_due_to_transform_version`。

### 2.3 用途血缘：当初为什么生成

同一份事实，用在不同用途风险不同：

- `answer`：可以用 summary；
- `planning`：可以用带 caveat 的 derived fact；
- `external_side_effect`：必须回源复核；
- `security_decision`：通常禁止使用 LLM 派生缓存。

lineage 里要记录 `purpose`，否则缓存会跨用途误用。

## 3. learn-claude-code：Python 教学版

```python
# learn-claude-code/cache_lineage.py
from __future__ import annotations

import hashlib
import json
from dataclasses import dataclass, field
from datetime import datetime, timezone
from typing import Literal

SourceType = Literal["tool_result", "user_input", "memory", "policy", "cache"]
Purpose = Literal["answer", "planning", "external_side_effect", "security_decision"]


def stable_hash(value: object) -> str:
    body = json.dumps(value, sort_keys=True, ensure_ascii=False, separators=(",", ":"))
    return "sha256:" + hashlib.sha256(body.encode()).hexdigest()


@dataclass(frozen=True)
class LineageSource:
    source_type: SourceType
    source_id: str
    content_hash: str
    observed_at: str


@dataclass(frozen=True)
class LineageManifest:
    cache_key: str
    value_hash: str
    purpose: Purpose
    sources: list[LineageSource]
    transforms: list[str]
    policy_version: str
    confidence: float
    created_at: str = field(default_factory=lambda: datetime.now(timezone.utc).isoformat())

    def depends_on(self, source_id: str) -> bool:
        return any(s.source_id == source_id for s in self.sources)


class LineageCache:
    def __init__(self, store):
        self.store = store

    def set(self, key: str, value: object, *, purpose: Purpose,
            sources: list[LineageSource], transforms: list[str],
            policy_version: str, confidence: float) -> None:
        manifest = LineageManifest(
            cache_key=key,
            value_hash=stable_hash(value),
            purpose=purpose,
            sources=sources,
            transforms=transforms,
            policy_version=policy_version,
            confidence=confidence,
        )
        self.store.put(key, {"value": value, "lineage": manifest})

    def get_for_purpose(self, key: str, purpose: Purpose):
        entry = self.store.get(key)
        if not entry:
            return None

        lineage: LineageManifest = entry["lineage"]
        if lineage.value_hash != stable_hash(entry["value"]):
            raise ValueError("cache value was modified without lineage update")

        if purpose in ("external_side_effect", "security_decision"):
            # 高风险用途不能只靠派生缓存，必须拿 lineage 去回源复核
            return {"status": "needs_revalidate", "lineage": lineage}

        return {"status": "hit", "value": entry["value"], "lineage": lineage}

    def invalidate_by_source(self, source_id: str) -> int:
        invalidated = 0
        for key, entry in self.store.scan():
            if entry["lineage"].depends_on(source_id):
                self.store.mark_stale(key, reason=f"source_changed:{source_id}")
                invalidated += 1
        return invalidated
```

教学版重点：

1. 写缓存时强制带 `sources/transforms/purpose`；
2. 读缓存时验证 `valueHash`，防止 value 被改但 lineage 没更新；
3. 源数据变化时按 `sourceId` 精准失效派生缓存；
4. 高风险用途返回 `needs_revalidate`，让 Agent 回源复核。

## 4. pi-mono：Lineage Middleware

```ts
// pi-mono/cache/LineageMiddleware.ts
type Purpose = "answer" | "planning" | "external_side_effect" | "security_decision";

type LineageSource = {
  sourceType: "tool_result" | "user_input" | "memory" | "policy" | "cache";
  sourceId: string;
  contentHash: string;
  observedAt: string;
};

type CacheLineage = {
  cacheKey: string;
  valueHash: string;
  purpose: Purpose;
  sources: LineageSource[];
  transforms: string[];
  policyVersion: string;
  confidence: number;
  createdAt: string;
};

type CacheEnvelope<T> = {
  key: string;
  value: T;
  lineage: CacheLineage;
};

export class LineageMiddleware {
  constructor(
    private hash: (value: unknown) => string,
    private sourceIndex: SourceDependencyIndex,
    private audit: (event: string, payload: Record<string, unknown>) => void,
  ) {}

  beforeWrite<T>(input: {
    key: string;
    value: T;
    purpose: Purpose;
    sources: LineageSource[];
    transforms: string[];
    policyVersion: string;
    confidence: number;
  }): CacheEnvelope<T> {
    if (input.sources.length === 0) {
      throw new Error("cache write requires at least one lineage source");
    }

    const envelope: CacheEnvelope<T> = {
      key: input.key,
      value: input.value,
      lineage: {
        cacheKey: input.key,
        valueHash: this.hash(input.value),
        purpose: input.purpose,
        sources: input.sources,
        transforms: input.transforms,
        policyVersion: input.policyVersion,
        confidence: input.confidence,
        createdAt: new Date().toISOString(),
      },
    };

    for (const source of input.sources) {
      this.sourceIndex.add(source.sourceId, input.key);
    }

    this.audit("cache.lineage.write", {
      key: input.key,
      purpose: input.purpose,
      sourceCount: input.sources.length,
      transforms: input.transforms,
      confidence: input.confidence,
    });

    return envelope;
  }

  afterRead<T>(envelope: CacheEnvelope<T>, requestedPurpose: Purpose) {
    const actualHash = this.hash(envelope.value);
    if (actualHash !== envelope.lineage.valueHash) {
      this.audit("cache.lineage.hash_mismatch", { key: envelope.key });
      return { kind: "deny" as const, reason: "lineage_hash_mismatch" };
    }

    if (requestedPurpose === "external_side_effect" || requestedPurpose === "security_decision") {
      return {
        kind: "revalidate" as const,
        reason: "high_risk_purpose_requires_origin_check",
        lineage: envelope.lineage,
      };
    }

    return {
      kind: "hit" as const,
      value: envelope.value,
      lineage: envelope.lineage,
    };
  }

  invalidateSource(sourceId: string) {
    const keys = this.sourceIndex.keysDependingOn(sourceId);
    this.audit("cache.lineage.invalidate_source", { sourceId, keys });
    return keys;
  }
}
```

生产版要把 `SourceDependencyIndex` 做成可查询索引：

- `sourceId -> cacheKeys[]`：源变化时精准失效；
- `cacheKey -> sourceIds[]`：解释某个答案来源；
- `policyVersion/transformVersion -> cacheKeys[]`：策略或 prompt 改版后批量刷新。

## 5. OpenClaw 实战：给文件缓存加 manifest

OpenClaw 里很多缓存天然是文件式的，比如：

- heartbeat 状态；
- 课程目录和已讲列表；
- 工具结果摘要；
- 子 Agent 长任务 checkpoint。

建议不要只写：

```text
cache/billing-summary.md
```

而是配套写：

```text
cache/billing-summary.md
cache/billing-summary.lineage.json
```

`lineage.json` 示例：

```json
{
  "cacheKey": "billing-summary:metadraw:2026-05-09",
  "valuePath": "cache/billing-summary.md",
  "valueHash": "sha256:...",
  "purpose": "answer",
  "sources": [
    {
      "sourceType": "tool_result",
      "sourceId": "aws-cost-explorer:2026-05-07",
      "contentHash": "sha256:...",
      "observedAt": "2026-05-09T09:04:00+11:00"
    }
  ],
  "transforms": ["cost_report:v3", "summary_prompt:v8"],
  "policyVersion": "billing-alert-policy:v2",
  "confidence": 0.93
}
```

这样以后老板问“为什么你说没有严重异常？”时，Agent 可以回答：

- 数据来自哪一次 Cost Explorer 查询；
- 查询覆盖哪个日期；
- 用了哪个异常阈值策略；
- 结果是否只是 summary，是否需要实时重查。

这比单纯说“我查过了”靠谱得多。

## 6. 常见坑

### 坑 1：只记录 source URL，不记录 content hash

URL 不等于内容。必须记录当时观察到的内容 hash，否则无法证明缓存值对应哪个版本的事实。

### 坑 2：派生缓存不记录上游缓存

如果 summary 来自另一个 cached tool result，lineage 里也要记录上游 cache key。否则上游失效，下游还会继续污染回答。

### 坑 3：把 lineage 当日志，不参与运行时决策

lineage 不只是审计材料。它应该参与：

- 是否允许命中；
- 是否需要回源；
- 哪些 key 要失效；
- 答案能不能解释给用户。

### 坑 4：高风险动作直接用派生 summary

比如“删除这个没用的云资源”。summary 可以帮你定位候选项，但真正删除前必须根据 lineage 回源重新查一次。

## 7. 记住这条规则

> **缓存 value 负责快，lineage 负责可信；没有 lineage 的缓存，只能用于低风险提示，不能支撑重要决策。**

下一次你给 Agent 加缓存时，不要只问“key 怎么设计”。还要问：

1. 这个结果从哪些事实来？
2. 哪些转换参与生成？
3. 源数据变了怎么精准失效？
4. 用户追问依据时能不能解释？
5. 高风险动作前能不能回源复核？

这五个问题答得出来，缓存才真正从“性能优化”升级成“可信事实层”。
