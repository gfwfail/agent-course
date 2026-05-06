# 258. Agent 缓存压缩与摘要降级（Cache Compaction & Summarized Entries）

前几课我们讲了缓存失效、一致性、权限边界和生命周期。今天继续补一个生产里很实用的点：**缓存不一定只有“保留原文”和“删除”两种选择，中间还有“压缩成摘要后继续可用”**。

一句话：

> Cache Compaction 是把大体积、低频访问、但仍有参考价值的缓存条目，从 raw payload 降级成 summary/index，既省空间，又保留 Agent 后续判断所需的关键事实。

这对 Agent 特别重要，因为 Agent 缓存的内容经常很胖：网页全文、邮件线程、日志片段、PR diff、Grafana 查询结果、长工具输出。直接保留会挤爆存储；直接删除又会让后续任务丢上下文。

---

## 1. 为什么普通缓存淘汰不够用

传统 LRU/LFU 的逻辑很简单：空间不够就删最不常用的。

但 Agent 的缓存有一个特殊需求：

- 后续可能不需要原始 50KB 内容；
- 只需要知道“这份内容讲了什么、关键结论是什么、能不能复查原始来源”。

例如一次网页抓取缓存：

```json
{
  "url": "https://docs.example.com/webhooks",
  "raw": "... 80KB HTML extracted markdown ...",
  "tokens": 21000
}
```

如果直接删掉，下一次 Agent 只能重新抓；如果保留原文，占空间又占上下文。更好的做法是压缩成：

```json
{
  "url": "https://docs.example.com/webhooks",
  "summary": "文档说明 webhook 需要 HMAC-SHA256 签名，重试最多 5 次，2xx 视为成功。",
  "facts": [
    "signature header: X-Webhook-Signature",
    "retry schedule: 1m, 5m, 30m, 2h, 12h",
    "success status: any 2xx"
  ],
  "raw_pointer": "artifact://cache/raw/abc123",
  "compacted_at": 1778124600
}
```

这样 Agent 仍能先用摘要判断方向；真要精确引用，再按 `raw_pointer` 重新加载或重抓。

---

## 2. 什么时候应该 compact，而不是 evict

建议用三类信号判断：

### A. Size pressure：空间压力

```text
used_bytes / max_bytes > 0.8
```

空间超过 80% 后，先压缩大条目；超过 95% 才硬淘汰。

### B. Access pattern：访问模式

适合压缩：

- 大条目；
- 最近没访问；
- 但命中过多次，说明有长期价值；
- 内容可重建或有 source pointer。

不适合压缩：

- 二进制文件；
- 必须精确复原的审计日志；
- 密钥/隐私数据；
- 低置信度或错误结果。

### C. Risk policy：风险策略

敏感数据一般不要让 LLM 摘要后长期保存，因为摘要也可能包含隐私。规则可以简单点：

```text
public/internal: allow compaction
sensitive: expire or encrypted archive only
compliance/audit: immutable archive, no LLM summary rewrite
```

---

## 3. learn-claude-code：Python 教学版

下面是一个最小可跑的缓存压缩器。它不会真的调用 LLM，而是把 `summarize()` 留成接口，方便教学实现替换。

```python
# learn_claude_code/cache_compaction.py
from __future__ import annotations

import time
from dataclasses import dataclass, field
from typing import Callable, Literal

Risk = Literal["public", "internal", "sensitive", "audit"]
Mode = Literal["raw", "summary"]

@dataclass
class CacheEntry:
    key: str
    value: str
    risk: Risk
    size_bytes: int
    hits: int = 0
    last_accessed_at: float = field(default_factory=time.time)
    mode: Mode = "raw"
    raw_pointer: str | None = None
    facts: list[str] = field(default_factory=list)

    @property
    def compactable(self) -> bool:
        return self.mode == "raw" and self.risk in {"public", "internal"}

class CompactingCache:
    def __init__(self, max_bytes: int, summarize: Callable[[str], tuple[str, list[str]]]):
        self.max_bytes = max_bytes
        self.summarize = summarize
        self.entries: dict[str, CacheEntry] = {}
        self.used_bytes = 0

    def get(self, key: str) -> CacheEntry | None:
        entry = self.entries.get(key)
        if not entry:
            return None
        entry.hits += 1
        entry.last_accessed_at = time.time()
        return entry

    def set(self, key: str, value: str, risk: Risk = "internal") -> None:
        if key in self.entries:
            self.used_bytes -= self.entries[key].size_bytes

        size = len(value.encode("utf-8"))
        self.entries[key] = CacheEntry(key=key, value=value, risk=risk, size_bytes=size)
        self.used_bytes += size
        self.compact_if_needed()

    def compact_if_needed(self) -> None:
        if self.used_bytes <= self.max_bytes * 0.8:
            return

        candidates = [e for e in self.entries.values() if e.compactable]
        # 大、旧、有命中价值的先压缩
        candidates.sort(key=lambda e: (e.size_bytes, e.hits, -e.last_accessed_at), reverse=True)

        for entry in candidates:
            if self.used_bytes <= self.max_bytes * 0.7:
                break
            self.compact(entry)

    def compact(self, entry: CacheEntry) -> None:
        summary, facts = self.summarize(entry.value)
        old_size = entry.size_bytes

        entry.raw_pointer = f"artifact://raw-cache/{entry.key}"
        entry.value = summary
        entry.facts = facts
        entry.mode = "summary"
        entry.size_bytes = len(summary.encode("utf-8")) + sum(len(f.encode("utf-8")) for f in facts)

        self.used_bytes -= old_size - entry.size_bytes

# 教学版 summarizer：生产里替换成小模型/规则摘要/离线 worker
def cheap_summarize(text: str) -> tuple[str, list[str]]:
    lines = [line.strip() for line in text.splitlines() if line.strip()]
    summary = " ".join(lines[:3])[:500]
    facts = [line[:120] for line in lines[:5]]
    return summary, facts
```

关键点：

- `mode` 标记当前是 raw 还是 summary；
- `raw_pointer` 保留原始内容的复查路径；
- `risk` 决定是否允许压缩；
- 压缩不是最终回答，而是给后续 Agent Loop 的“低成本记忆索引”。

---

## 4. pi-mono：TypeScript 生产版中间件

生产里不要把 compaction 写进业务工具，应该放在 Cache Adapter / Middleware 层。

```ts
// pi-mono/cache/compacting-cache.ts
export type CacheRisk = "public" | "internal" | "sensitive" | "audit";
export type CacheMode = "raw" | "summary";

export interface CacheEnvelope<T = unknown> {
  key: string;
  value: T;
  mode: CacheMode;
  risk: CacheRisk;
  sizeBytes: number;
  hits: number;
  lastAccessedAt: number;
  tags: string[];
  facts?: string[];
  rawPointer?: string;
  compactedAt?: number;
}

export interface Summarizer {
  summarize(input: string): Promise<{ summary: string; facts: string[] }>;
}

export class CompactingCache {
  constructor(
    private readonly store: Map<string, CacheEnvelope>,
    private readonly summarizer: Summarizer,
    private readonly maxBytes: number,
  ) {}

  async set<T>(entry: CacheEnvelope<T>): Promise<void> {
    this.store.set(entry.key, entry);
    await this.compactIfNeeded();
  }

  async get<T>(key: string): Promise<CacheEnvelope<T> | null> {
    const entry = this.store.get(key) as CacheEnvelope<T> | undefined;
    if (!entry) return null;

    entry.hits += 1;
    entry.lastAccessedAt = Date.now();
    return entry;
  }

  private usedBytes(): number {
    return [...this.store.values()].reduce((sum, entry) => sum + entry.sizeBytes, 0);
  }

  private compactable(entry: CacheEnvelope): boolean {
    return entry.mode === "raw" && ["public", "internal"].includes(entry.risk);
  }

  private async compactIfNeeded(): Promise<void> {
    if (this.usedBytes() <= this.maxBytes * 0.8) return;

    const candidates = [...this.store.values()]
      .filter((entry) => this.compactable(entry))
      .sort((a, b) => {
        const scoreA = a.sizeBytes + a.hits * 1024 - a.lastAccessedAt / 1_000_000;
        const scoreB = b.sizeBytes + b.hits * 1024 - b.lastAccessedAt / 1_000_000;
        return scoreB - scoreA;
      });

    for (const entry of candidates) {
      if (this.usedBytes() <= this.maxBytes * 0.7) break;
      if (typeof entry.value !== "string") continue;

      const { summary, facts } = await this.summarizer.summarize(entry.value);
      const rawPointer = await this.archiveRaw(entry.key, entry.value);

      entry.value = summary;
      entry.mode = "summary";
      entry.facts = facts;
      entry.rawPointer = rawPointer;
      entry.compactedAt = Date.now();
      entry.sizeBytes = Buffer.byteLength(summary) + facts.join("\n").length;
    }
  }

  private async archiveRaw(key: string, raw: unknown): Promise<string> {
    // 生产里可以写 S3/R2/local artifact store，并配置 TTL/权限
    return `artifact://cache/raw/${encodeURIComponent(key)}`;
  }
}
```

这层逻辑的好处是：工具只负责返回结果；缓存层自己决定什么时候压缩、如何保留复查指针、如何按风险拒绝压缩。

---

## 5. OpenClaw 实战：课程 Cron 的压缩点

OpenClaw 这种 always-on Agent 特别适合用 compaction：

- `web_fetch` 抓到的长文档：raw 只保留短 TTL，summary 进入长期参考；
- `session_history` 大段历史：压缩成任务摘要 + decision log；
- 课程生成过程：把 lesson 草稿、Telegram messageId、commit sha 保留成结构化记录；
- heartbeat 检查结果：只保存变化摘要，不保存每次完整输出。

一个实用的文件式结构：

```text
memory/cache/
  raw/2026-05-07/github-docs-abc123.md
  compacted/2026-05-07/github-docs-abc123.json
```

压缩后的 JSON：

```json
{
  "key": "github-docs-abc123",
  "mode": "summary",
  "source": "web_fetch:https://docs.github.com/...",
  "summary": "该文档说明 GitHub Actions workflow_dispatch 的输入参数和权限限制。",
  "facts": [
    "workflow_dispatch supports inputs",
    "manual runs require write permission",
    "inputs are available under github.event.inputs"
  ],
  "rawPointer": "memory/cache/raw/2026-05-07/github-docs-abc123.md",
  "risk": "public",
  "createdAt": "2026-05-07T09:30:00+11:00",
  "compactedAt": "2026-05-07T09:35:00+11:00"
}
```

重点：最终注入 LLM 的是 `summary + facts + rawPointer`，不是整篇 raw。

---

## 6. 常见坑

### 坑 1：摘要当原文用

摘要只能用于方向判断，不能用于精确引用。需要引用、金额、权限、版本号时，必须回源验证。

### 坑 2：敏感内容被摘要后长期保存

很多人以为摘要不算敏感，这是错的。邮件摘要、账单摘要、客户资料摘要仍然可能是 PII。

### 坑 3：压缩过程覆盖审计证据

审计日志不能被“改写成摘要”。审计可以额外生成索引，但原始证据必须不可变保存。

### 坑 4：没有记录 compaction lineage

压缩后要知道摘要来自哪个 raw、什么时间压缩、用什么模型/版本压缩。否则未来排查时不知道摘要是否可信。

---

## 7. Checklist

设计 Agent Cache Compaction 时，至少检查：

- [ ] 每个缓存条目有 `mode: raw | summary`；
- [ ] summary 条目保留 `rawPointer` 或 source；
- [ ] sensitive/audit 数据默认不走 LLM 摘要；
- [ ] 空间阈值分两级：先 compact，后 evict；
- [ ] 最终回答遇到精确事实会回源验证；
- [ ] compaction 记录 `compactedAt/model/version`；
- [ ] summary 也受 TTL、权限和 purge tag 控制。

---

## 总结

缓存生命周期不是只有 TTL 和 LRU。对 Agent 来说，更好的策略是：

```text
hot + small        -> keep raw
cold + valuable    -> compact to summary
cold + low value   -> evict
sensitive expired  -> purge
exact answer needed -> rehydrate raw
```

成熟 Agent 的缓存系统，不是“存得越多越好”，而是能把大上下文压成可复查的知识索引：平时用摘要省 token，关键时刻能回源拿证据。