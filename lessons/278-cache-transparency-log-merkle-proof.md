# 278. Agent 缓存透明日志与 Merkle 证明（Cache Transparency Log & Merkle Proof）

> 上一课我们讲了缓存完整性签名：单条缓存能证明“确实是可信写入路径签的”。今天再往前一步：**单条记录没被改，不代表历史没被删、没被回滚、没被选择性隐藏。**

一句话：**签名保护单条缓存，透明日志保护整段历史。**

## 1. 为什么还需要透明日志？

假设每条缓存都有 HMAC/Ed25519 签名，攻击者仍然可以做三件事：

1. **删除一条不利记录**：比如删除曾经命中过低置信度数据的日志；
2. **回滚到旧版本**：把缓存目录恢复到昨天的快照，签名仍然有效；
3. **选择性展示**：给审计器看 A 历史，给线上读取路径看 B 历史。

这类问题不靠单条签名解决，而靠 append-only 透明日志解决。

核心设计：

- 每次 cache write / evict / deny / integrity_failed 都追加一条 event；
- event 只追加，不原地修改；
- 每个 event 计算 hash；
- 每隔 N 条生成一个 Merkle root；
- root 写入独立位置（比如 Git commit、对象存储、审计表、外部监控）；
- 审计时可以证明：某条事件确实包含在某个 root 里，历史没有被无声改写。

## 2. 透明日志记录什么？

不要把敏感 value 原文写进透明日志。记录“可验证指纹”和关键上下文即可：

```json
{
  "seq": 10241,
  "time": "2026-05-09T11:30:00Z",
  "event": "cache.write",
  "tenantId": "metadraw",
  "actorId": "cron:agent-course",
  "purpose": "answer",
  "keyHash": "sha256:9d2...",
  "valueHash": "sha256:83a...",
  "envelopeHash": "sha256:f17...",
  "signatureKeyId": "cache-signing-2026-05",
  "sourceVersion": "github:gfwfail/agent-course@2147eec",
  "trust": "verified",
  "decision": "admit"
}
```

建议最少包含：

- `seq`：严格递增序号，防止重排；
- `event`：write / hit / stale_hit / deny / evict / integrity_failed；
- `keyHash`：缓存 key 的 hash，避免泄漏参数；
- `valueHash` / `envelopeHash`：和上一课签名内容对齐；
- `actorId` / `tenantId` / `purpose`：回答“谁为了什么用它”；
- `sourceVersion`：能回到源头复查；
- `prevEventHash`：形成 hash chain，低成本防篡改；
- `merkleRoot`：批量证明 inclusion。

## 3. Hash Chain vs Merkle Tree

两者不是二选一，最好一起用：

### Hash Chain：证明顺序没被改

```text
eventHash[n] = sha256(canonical(event[n]) + eventHash[n-1])
```

优点：实现简单，适合实时 append-only 日志。

缺点：证明某条记录存在时，可能要从头重算很多条。

### Merkle Tree：证明某条记录在批次里

```text
leaf = sha256(eventHash)
parent = sha256(left + right)
root = sha256(...)
```

优点：给某条 event 的 inclusion proof 只需要 `O(log n)` 个 sibling hash。

缺点：实现稍复杂，通常按 batch 生成 root。

工程建议：

- 实时写入：用 hash chain；
- 每 100 / 1000 条：生成 Merkle root；
- 高风险操作前：验证最近 checkpoint root 没漂移；
- 审计报告：附 inclusion proof，而不是把整份日志搬出来。

## 4. learn-claude-code：Python 教学版

```python
# learn-claude-code/cache_transparency_log.py
from __future__ import annotations

from dataclasses import dataclass, asdict
from pathlib import Path
from typing import Any, Literal
import hashlib
import json

EventType = Literal[
    "cache.write",
    "cache.hit",
    "cache.stale_hit",
    "cache.deny",
    "cache.evict",
    "cache.integrity_failed",
]


def canonical(value: Any) -> bytes:
    return json.dumps(
        value,
        sort_keys=True,
        ensure_ascii=False,
        separators=(",", ":"),
    ).encode("utf-8")


def sha256_hex(data: bytes) -> str:
    return "sha256:" + hashlib.sha256(data).hexdigest()


@dataclass(frozen=True)
class CacheLogEvent:
    seq: int
    time: str
    event: EventType
    tenant_id: str
    actor_id: str
    purpose: str
    key_hash: str
    value_hash: str | None
    envelope_hash: str | None
    source_version: str
    decision: str
    prev_event_hash: str


class TransparencyLog:
    def __init__(self, path: Path):
        self.path = path
        self.path.parent.mkdir(parents=True, exist_ok=True)
        self._last_seq = 0
        self._last_hash = "sha256:GENESIS"
        if path.exists():
            for line in path.read_text().splitlines():
                record = json.loads(line)
                self._last_seq = record["event"]["seq"]
                self._last_hash = record["eventHash"]

    def append(self, event: OmitSeqEvent) -> str:
        next_event = CacheLogEvent(
            seq=self._last_seq + 1,
            prev_event_hash=self._last_hash,
            **event,
        )
        event_hash = sha256_hex(canonical(asdict(next_event)))
        record = {"event": asdict(next_event), "eventHash": event_hash}

        with self.path.open("a", encoding="utf-8") as f:
            f.write(json.dumps(record, ensure_ascii=False, sort_keys=True) + "\n")

        self._last_seq = next_event.seq
        self._last_hash = event_hash
        return event_hash


# Python 没有内置 Omit，这里教学里用 dict 简化：
OmitSeqEvent = dict[str, Any]
```

写入缓存时追加透明日志：

```python
log = TransparencyLog(Path(".openclaw/cache/transparency.jsonl"))

log.append({
    "time": "2026-05-09T11:30:00Z",
    "event": "cache.write",
    "tenant_id": "metadraw",
    "actor_id": "cron:agent-course",
    "purpose": "answer",
    "key_hash": sha256_hex(b"agent-cache:v7:course:278"),
    "value_hash": signed_entry.envelope.value_hash,
    "envelope_hash": sha256_hex(canonical(asdict(signed_entry.envelope))),
    "source_version": "github:gfwfail/agent-course@HEAD",
    "decision": "admit",
})
```

注意：这里日志里放的是 hash，不是课程正文原文，也不是 API key。

## 5. Merkle Root：给审计报告一个短证明

```python
# learn-claude-code/merkle.py
from __future__ import annotations

import hashlib
from dataclasses import dataclass


def h(left: str, right: str = "") -> str:
    raw = (left + "|" + right).encode("utf-8")
    return "sha256:" + hashlib.sha256(raw).hexdigest()


@dataclass
class MerkleCheckpoint:
    from_seq: int
    to_seq: int
    root: str
    leaf_count: int


def merkle_root(event_hashes: list[str]) -> str:
    if not event_hashes:
        return "sha256:EMPTY"

    level = event_hashes[:]
    while len(level) > 1:
        next_level: list[str] = []
        for i in range(0, len(level), 2):
            left = level[i]
            right = level[i + 1] if i + 1 < len(level) else left
            next_level.append(h(left, right))
        level = next_level
    return level[0]
```

生成 checkpoint：

```python
records = [json.loads(line) for line in log_path.read_text().splitlines()]
batch = records[-1000:]
checkpoint = MerkleCheckpoint(
    from_seq=batch[0]["event"]["seq"],
    to_seq=batch[-1]["event"]["seq"],
    root=merkle_root([r["eventHash"] for r in batch]),
    leaf_count=len(batch),
)

Path(".openclaw/cache/checkpoints.jsonl").open("a").write(
    json.dumps(asdict(checkpoint), sort_keys=True) + "\n"
)
```

生产环境可以把 checkpoint root 再写到更难篡改的位置：

- Git commit；
- S3/R2 object lock；
- Postgres append-only audit table；
- 外部日志系统；
- 甚至定期发到只读审计频道。

## 6. pi-mono：CacheTransparencyMiddleware

生产版建议把透明日志做成缓存中间件，业务代码不直接关心 Merkle。

```ts
// pi-mono/cache/CacheTransparencyMiddleware.ts
import { createHash } from "node:crypto";

export type CacheEventType =
  | "cache.write"
  | "cache.hit"
  | "cache.stale_hit"
  | "cache.deny"
  | "cache.evict"
  | "cache.integrity_failed";

export type CacheAuditEvent = {
  seq?: number;
  time: string;
  event: CacheEventType;
  tenantId: string;
  actorId: string;
  purpose: string;
  keyHash: string;
  valueHash?: string;
  envelopeHash?: string;
  sourceVersion: string;
  decision: string;
  prevEventHash?: string;
};

function canonical(value: unknown): string {
  return JSON.stringify(value, Object.keys(value as object).sort());
}

function sha256(value: string): string {
  return "sha256:" + createHash("sha256").update(value).digest("hex");
}

export interface TransparencyStore {
  append(record: { event: Required<CacheAuditEvent>; eventHash: string }): Promise<void>;
  last(): Promise<{ seq: number; eventHash: string } | null>;
}

export class CacheTransparencyLog {
  constructor(private readonly store: TransparencyStore) {}

  async append(event: Omit<CacheAuditEvent, "seq" | "prevEventHash">): Promise<string> {
    const last = await this.store.last();
    const fullEvent: Required<CacheAuditEvent> = {
      ...event,
      valueHash: event.valueHash ?? "",
      envelopeHash: event.envelopeHash ?? "",
      seq: (last?.seq ?? 0) + 1,
      prevEventHash: last?.eventHash ?? "sha256:GENESIS",
    };

    const eventHash = sha256(canonical(fullEvent));
    await this.store.append({ event: fullEvent, eventHash });
    return eventHash;
  }
}
```

接到缓存写入中间件里：

```ts
export class CacheTransparencyMiddleware {
  constructor(private readonly log: CacheTransparencyLog) {}

  async afterWrite(ctx: {
    tenantId: string;
    actorId: string;
    purpose: string;
    key: string;
    valueHash: string;
    envelopeHash: string;
    sourceVersion: string;
  }) {
    await this.log.append({
      time: new Date().toISOString(),
      event: "cache.write",
      tenantId: ctx.tenantId,
      actorId: ctx.actorId,
      purpose: ctx.purpose,
      keyHash: sha256(ctx.key),
      valueHash: ctx.valueHash,
      envelopeHash: ctx.envelopeHash,
      sourceVersion: ctx.sourceVersion,
      decision: "admit",
    });
  }

  async afterIntegrityFailure(ctx: { tenantId: string; actorId: string; purpose: string; key: string }) {
    await this.log.append({
      time: new Date().toISOString(),
      event: "cache.integrity_failed",
      tenantId: ctx.tenantId,
      actorId: ctx.actorId,
      purpose: ctx.purpose,
      keyHash: sha256(ctx.key),
      sourceVersion: "unknown",
      decision: "evict_and_refetch",
    });
  }
}
```

关键点：透明日志要在缓存层统一写，不能靠每个业务工具自觉写。

## 7. OpenClaw 文件式落地

OpenClaw 里可以用很朴素的文件结构：

```text
.openclaw/cache/
  entries/
    agent-cache-v7-xxx.json
  audit/
    transparency-2026-05-09.jsonl
    checkpoints.jsonl
```

每次课程 cron、heartbeat、GitHub 检查都可以写一条：

```json
{"event":"cache.hit","keyHash":"sha256:...","purpose":"planning","decision":"use","seq":2781}
{"event":"cache.write","keyHash":"sha256:...","purpose":"answer","decision":"admit","seq":2782}
{"event":"cache.integrity_failed","keyHash":"sha256:...","purpose":"external_side_effect","decision":"block","seq":2783}
```

验收闸门可以这样跑：

```bash
python scripts/verify_transparency_log.py .openclaw/cache/audit/transparency-2026-05-09.jsonl
python scripts/check_merkle_checkpoint.py .openclaw/cache/audit/checkpoints.jsonl
```

如果发现 `prevEventHash` 对不上：

- answer / planning：丢弃相关缓存，回源重建；
- external_side_effect：阻断执行，要求人工确认；
- security_decision：直接 fail closed，并生成事故审计。

## 8. 常见坑

### 坑 1：日志里写原文

透明日志不是数据仓库。写 hash、metadata、sourceVersion，不写用户隐私、API key、完整工具结果。

### 坑 2：只记录 write，不记录 deny / failure

真正重要的往往是失败路径：为什么拒绝、为什么 evict、为什么完整性校验失败。

### 坑 3：checkpoint root 还存在同一台机器同一目录

如果攻击者能改日志，也可能改 checkpoint。root 至少要复制到独立存储或 Git 历史里。

### 坑 4：没有 seq

没有严格递增序号，日志可以重排；没有 `prevEventHash`，中间删一条很难发现。

## 9. 这一课的核心

- **签名**：证明单条缓存是可信写入者签的；
- **Hash Chain**：证明事件顺序没有被无声改写；
- **Merkle Root**：用短证明验证某条事件属于某个批次；
- **Checkpoint 外置**：防止日志和 root 一起被改；
- **敏感最小化**：透明日志记录指纹，不记录秘密。

成熟 Agent 的缓存，不只是“这条数据对吗”，还要能回答：**这条数据是什么时候、由谁、为了什么写入的；后来有没有被删、被回滚、被篡改。**

透明日志让缓存从性能组件升级成可审计的事实账本。
