# 279. Agent 缓存损坏隔离与自动恢复（Cache Corruption Quarantine & Recovery）

> 上一课我们讲了缓存透明日志：证明缓存历史没有被悄悄改写。今天补上生产系统里更现实的一步：**发现缓存坏了之后，不能只报错，要隔离、降级、恢复，并留下证据。**

一句话：**缓存损坏不是异常分支，而是一条必须设计好的恢复流程。**

## 1. 为什么要有 Quarantine？

很多缓存事故不是“缓存 miss”，而是“缓存命中了一份坏数据”：

- value hash 对不上：文件被截断、磁盘写入失败、并发写半截；
- signature 验证失败：写入路径不可信或 key rotation 配错；
- schema 不兼容：老版本缓存被新代码误读；
- lineage/sourceVersion 异常：缓存来源已被撤回或污染；
- plaintext/redaction 不符合策略：敏感字段被错误写入可见缓存。

如果读路径只是 `throw error`，用户会看到失败；如果读路径忽略错误继续用，事故会扩大。

正确做法是：

1. **立刻隔离坏 entry**：移到 quarantine 区，不再被普通 read 命中；
2. **记录原因与证据**：keyHash、valueHash、signatureKeyId、schemaVersion、sourceVersion；
3. **按用途降级**：普通回答可回源重建，高风险副作用必须 block；
4. **尝试自动恢复**：从源头重新拉取、从上一 checkpoint 重放、或走 stale fallback；
5. **输出审计事件**：让后续复盘知道“坏在哪里、怎么恢复的”。

缓存恢复的目标不是“永远不坏”，而是**坏了也不会静默污染决策**。

## 2. 状态机设计

把缓存 entry 从简单的 hit/miss 升级成状态机：

```text
healthy ──verify_failed──> quarantined
   │                         │
   │                         ├─recover_ok──> healthy
   │                         ├─recover_failed──> dead_letter
   │                         └─manual_release──> healthy
   └─expired/evicted──────> removed
```

建议每个 quarantine record 至少包含：

```json
{
  "quarantineId": "q_20260509_1430_9f2c",
  "time": "2026-05-09T14:30:00Z",
  "reason": "signature_mismatch",
  "keyHash": "sha256:9d2...",
  "entryPath": ".openclaw/cache/q/q_20260509_1430_9f2c.json",
  "valueHash": "sha256:83a...",
  "expectedHash": "sha256:f17...",
  "schemaVersion": 7,
  "sourceVersion": "github:gfwfail/agent-course@80c0a72",
  "purpose": "answer",
  "risk": "medium",
  "recovery": {
    "strategy": "refetch_origin",
    "status": "pending"
  }
}
```

注意：quarantine 里依然不要写敏感原文；能放 hash、metadata、加密副本，就不要放 plaintext。

## 3. learn-claude-code：Python 教学版

教学版用本地 JSON 文件模拟缓存隔离。核心思想：读缓存前验证，失败就原子移动到 quarantine 目录。

```python
# learn-claude-code/cache_quarantine.py
from __future__ import annotations

from dataclasses import dataclass, asdict
from pathlib import Path
from typing import Any, Literal
import hashlib
import json
import shutil
import time

CorruptionReason = Literal[
    "hash_mismatch",
    "signature_mismatch",
    "schema_invalid",
    "policy_violation",
    "lineage_revoked",
]


def canonical(value: Any) -> bytes:
    return json.dumps(value, sort_keys=True, separators=(",", ":")).encode()


def sha256_hex(data: bytes) -> str:
    return "sha256:" + hashlib.sha256(data).hexdigest()


@dataclass(frozen=True)
class QuarantineRecord:
    quarantine_id: str
    ts: int
    reason: CorruptionReason
    key_hash: str
    original_path: str
    quarantine_path: str
    value_hash: str | None
    expected_hash: str | None
    schema_version: int | None
    source_version: str | None
    purpose: str
    risk: Literal["low", "medium", "high"]
    recovery_status: Literal["pending", "recovered", "dead_letter"] = "pending"


class CacheQuarantine:
    def __init__(self, cache_dir: Path):
        self.cache_dir = cache_dir
        self.q_dir = cache_dir / "quarantine"
        self.index_path = self.q_dir / "index.jsonl"
        self.q_dir.mkdir(parents=True, exist_ok=True)

    def isolate(
        self,
        *,
        key: str,
        entry_path: Path,
        reason: CorruptionReason,
        purpose: str,
        risk: Literal["low", "medium", "high"],
        expected_hash: str | None,
    ) -> QuarantineRecord:
        raw = entry_path.read_bytes() if entry_path.exists() else b""
        qid = f"q_{int(time.time())}_{hashlib.sha1(key.encode()).hexdigest()[:8]}"
        q_path = self.q_dir / f"{qid}.json"

        # 原子隔离：普通读路径不会再看到原文件
        if entry_path.exists():
            shutil.move(str(entry_path), str(q_path))

        record = QuarantineRecord(
            quarantine_id=qid,
            ts=int(time.time()),
            reason=reason,
            key_hash=sha256_hex(key.encode()),
            original_path=str(entry_path),
            quarantine_path=str(q_path),
            value_hash=sha256_hex(raw) if raw else None,
            expected_hash=expected_hash,
            schema_version=None,
            source_version=None,
            purpose=purpose,
            risk=risk,
        )

        with self.index_path.open("a", encoding="utf-8") as f:
            f.write(json.dumps(asdict(record), ensure_ascii=False, sort_keys=True) + "\n")

        return record
```

读缓存时使用：

```python
# learn-claude-code/cache_reader.py
from pathlib import Path
import json

class SafeCacheReader:
    def __init__(self, cache_dir: Path):
        self.cache_dir = cache_dir
        self.quarantine = CacheQuarantine(cache_dir)

    def read(self, key: str, *, purpose: str, risk: str):
        path = self.cache_dir / f"{sha256_hex(key.encode())[7:]}.json"
        if not path.exists():
            return {"status": "miss"}

        raw = path.read_bytes()
        entry = json.loads(raw)
        actual = sha256_hex(canonical(entry["value"]))
        expected = entry["envelope"].get("valueHash")

        if actual != expected:
            q = self.quarantine.isolate(
                key=key,
                entry_path=path,
                reason="hash_mismatch",
                purpose=purpose,
                risk=risk,  # type: ignore[arg-type]
                expected_hash=expected,
            )
            return {
                "status": "quarantined",
                "reason": q.reason,
                "quarantineId": q.quarantine_id,
                "suggestedAction": "refetch_origin" if risk != "high" else "block_and_review",
            }

        return {"status": "hit", "value": entry["value"]}
```

教学重点：**验证失败后不要把坏 value 返回给 LLM**。返回的是结构化状态和下一步建议。

## 4. pi-mono：生产中间件版本

生产版建议做成 Cache Middleware，而不是散落在业务代码里：

```ts
// pi-mono/cache/CacheQuarantineMiddleware.ts
type CorruptionReason =
  | 'hash_mismatch'
  | 'signature_mismatch'
  | 'schema_invalid'
  | 'policy_violation'
  | 'lineage_revoked';

type CachePurpose = 'answer' | 'planning' | 'external_side_effect' | 'security_decision';

type CacheReadContext = {
  tenantId: string;
  actorId: string;
  runId: string;
  purpose: CachePurpose;
  risk: 'low' | 'medium' | 'high';
};

type QuarantineDecision = {
  action: 'refetch_origin' | 'serve_stale_with_caveat' | 'block' | 'dead_letter';
  reason: CorruptionReason;
  quarantineId: string;
};

export class CacheQuarantineMiddleware {
  constructor(
    private readonly store: CacheStore,
    private readonly quarantine: QuarantineStore,
    private readonly audit: CacheAuditLog,
    private readonly origin: OriginRefetcher,
  ) {}

  async read<T>(key: string, ctx: CacheReadContext): Promise<CacheReadResult<T>> {
    const entry = await this.store.getRaw(key);
    if (!entry) return { status: 'miss' };

    const verification = await verifyCacheEntry(entry, ctx);
    if (verification.ok) {
      return { status: 'hit', value: entry.value as T, evidence: verification.evidence };
    }

    const q = await this.quarantine.isolate({
      keyHash: hashKey(key),
      rawEntry: entry,
      reason: verification.reason,
      tenantId: ctx.tenantId,
      actorId: ctx.actorId,
      runId: ctx.runId,
      purpose: ctx.purpose,
      risk: ctx.risk,
      evidence: verification.evidence,
    });

    await this.audit.append({
      event: 'cache.quarantined',
      runId: ctx.runId,
      tenantId: ctx.tenantId,
      actorId: ctx.actorId,
      purpose: ctx.purpose,
      keyHash: hashKey(key),
      reason: verification.reason,
      quarantineId: q.id,
    });

    return this.decideRecovery<T>(key, ctx, q.id, verification.reason);
  }

  private async decideRecovery<T>(
    key: string,
    ctx: CacheReadContext,
    quarantineId: string,
    reason: CorruptionReason,
  ): Promise<CacheReadResult<T>> {
    if (ctx.purpose === 'external_side_effect' || ctx.purpose === 'security_decision') {
      return { status: 'blocked', reason, quarantineId };
    }

    const refreshed = await this.origin.tryRefetch<T>(key, { timeoutMs: 3_000 });
    if (refreshed.ok) {
      await this.store.set(key, refreshed.entry);
      return { status: 'recovered', value: refreshed.value, quarantineId };
    }

    if (ctx.purpose === 'answer') {
      return {
        status: 'degraded',
        caveat: '缓存损坏且回源失败，未使用损坏缓存；请稍后重试或提供更多来源。',
        quarantineId,
      };
    }

    return { status: 'blocked', reason, quarantineId };
  }
}
```

这段中间件背后的原则：

- answer：可以降级回答，但要说明 caveat；
- planning：可以 refetch，失败就要求更多证据；
- external_side_effect：不能用坏缓存继续执行；
- security_decision：必须 block，不能“差不多”。

## 5. OpenClaw 文件缓存实战

OpenClaw 这种 always-on Agent，经常会把状态放在 workspace 文件里。可以采用约定目录：

```text
.openclaw/cache/
  entries/                 # 正常缓存
  quarantine/              # 隔离坏 entry
    index.jsonl            # 隔离索引
    q_20260509_1430_9f2c.json.enc
  recovery/
    attempts.jsonl         # 自动恢复尝试
  transparency.jsonl       # 透明日志
```

Cron/Heartbeat 读缓存时的流程：

```ts
async function safeCached<T>(key: string, ctx: CacheReadContext, load: () => Promise<T>) {
  const cached = await cache.read<T>(key, ctx);

  if (cached.status === 'hit' || cached.status === 'recovered') return cached.value;

  if (cached.status === 'quarantined' || cached.status === 'miss') {
    const fresh = await load();
    await cache.write(key, fresh, ctx);
    return fresh;
  }

  if (cached.status === 'blocked') {
    throw new Error(`cache blocked: ${cached.reason}, q=${cached.quarantineId}`);
  }

  return load();
}
```

课程 cron 里的例子：

- README 目录缓存 hash 不一致：隔离缓存，重新扫描 `lessons/`；
- GitHub PR 状态缓存验证失败：隔离缓存，重新 `gh pr list`；
- Telegram message delivery receipt 缓存损坏：不要重复发送，先查 outbox/回执，再决定是否补发；
- TOOLS.md 已讲内容缓存异常：不信缓存，直接 grep 文件确认，避免重复授课。

## 6. 自动恢复策略

按风险从低到高：

| 场景 | 策略 | 是否可自动继续 |
|---|---|---|
| 普通回答缓存坏 | refetch origin | 可以 |
| 课程目录缓存坏 | 重新扫描文件系统 | 可以 |
| 用户偏好缓存坏 | 用 MEMORY/USER.md 重建摘要 | 可以，但标记低置信度 |
| 支付/部署状态缓存坏 | live check 权威 API | 只能在验证后继续 |
| 权限/安全决策缓存坏 | block + 人工复核 | 不可以 |

恢复记录也要写日志：

```json
{
  "event": "cache.recovery_attempt",
  "quarantineId": "q_20260509_1430_9f2c",
  "strategy": "refetch_origin",
  "status": "recovered",
  "newValueHash": "sha256:aa1...",
  "durationMs": 842
}
```

## 7. 常见坑

1. **只 delete，不 quarantine**  
   删除坏缓存会让证据消失，后面无法复盘。

2. **隔离后继续返回 value**  
   这是最危险的“假隔离”。隔离意味着普通路径绝对不能再使用它。

3. **高风险动作自动 fallback stale**  
   stale 可以用于解释性回答，不能用于转账、部署、删资源、权限判断。

4. **quarantine 无限增长**  
   隔离区也需要 retention：例如保留 30 天，或按审计策略归档。

5. **恢复成功不写新证据**  
   recovered entry 必须重新签名、写 lineage、写透明日志，否则只是把坏状态换个名字。

## 8. Checklist

- [ ] 每次 cache read 都做 hash/schema/signature/policy 验证；
- [ ] 验证失败先 quarantine，再决定恢复；
- [ ] quarantine record 不泄露敏感原文；
- [ ] answer/planning/side_effect/security_decision 使用不同降级策略；
- [ ] 自动恢复必须写 recovery_attempt 审计事件；
- [ ] 恢复后的缓存重新签名、重新写 lineage 和透明日志；
- [ ] 隔离区有 retention 和 dead-letter 处理；
- [ ] 高风险副作用前遇到缓存损坏必须 block。

## 9. 总结

缓存系统的成熟度可以分三层：

1. **能命中**：快；
2. **能验证**：可信；
3. **能隔离和恢复**：生产可用。

Agent 最怕的不是 cache miss，而是**cache hit 了一份坏事实**。

下一次你给 Agent 加缓存时，不要只写 `get/set`。至少再写三个动作：`verify`、`quarantine`、`recover`。这才是能上生产的缓存。🫡
