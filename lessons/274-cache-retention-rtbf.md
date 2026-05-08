# 274. Agent 缓存保留期限与被遗忘权（Cache Retention & Right-to-be-Forgotten）

> 上一课我们讲了：解密后的数据也要按用途做最小披露。今天继续补缓存安全闭环里最现实的一层：**缓存什么时候必须过期、什么时候必须被彻底遗忘**。

一句话：**TTL 解决“多久后不再用”，Retention Policy 解决“哪些数据根本不能长期留”；Right-to-be-Forgotten 解决“用户要求删除后，所有缓存副本都要真的忘”。**

## 1. 为什么普通 TTL 不够？

很多 Agent 缓存一开始只是为了快：

- 工具结果缓存 24 小时；
- 用户画像缓存 7 天；
- 邮件/订单摘要缓存到文件；
- 子 Agent 中间结果顺手写进 workspace。

问题是：**TTL 是技术字段，不是合规策略**。

如果用户注销、租户合同结束、管理员执行删除请求，系统不能等缓存自然过期；如果缓存有 L1 内存、L2 Redis、L3 文件、向量索引、审计摘要，删一个 key 也不代表真的忘了。

成熟 Agent 至少要区分三种时间：

- `expiresAt`：过期后不能再用于回答；
- `retainUntil`：为了审计/纠纷可保留到什么时候；
- `deleteAfter`：到点必须物理删除或不可逆匿名化。

## 2. CacheEnvelope 里要放保留策略

不要只存 value：

```json
{
  "key": "agent:v12:tool:gmail:user:42:args:abc",
  "tenant": "acme",
  "subjectId": "user_42",
  "dataClass": "sensitive",
  "purpose": "answer",
  "createdAt": "2026-05-09T09:30:00+11:00",
  "expiresAt": "2026-05-09T10:30:00+11:00",
  "retainUntil": "2026-06-09T00:00:00+10:00",
  "deleteAfter": "2026-06-10T00:00:00+10:00",
  "erasable": true,
  "legalHold": false,
  "tombstoneVersion": 7
}
```

几个关键字段：

- `subjectId`：这条缓存属于哪个用户/主体，RTBF 删除靠它反查；
- `dataClass`：public/internal/sensitive/secret 决定默认保留期；
- `erasable`：是否允许用户删除请求立即清除；
- `legalHold`：合规/纠纷保留时禁止物理删除，但读取仍应默认 deny；
- `tombstoneVersion`：配合上一批课程的 tombstone，防旧 worker 复活已删除数据。

## 3. learn-claude-code：Python 教学版

```python
# learn-claude-code/cache_retention.py
from __future__ import annotations

from dataclasses import dataclass, replace
from datetime import datetime, timedelta, timezone
from typing import Literal

DataClass = Literal["public", "internal", "sensitive", "secret"]


@dataclass(frozen=True)
class CacheEntry:
    key: str
    subject_id: str
    data_class: DataClass
    value: dict
    expires_at: datetime
    retain_until: datetime
    delete_after: datetime
    erasable: bool = True
    legal_hold: bool = False
    tombstone_version: int = 0


class RetentionPolicy:
    def windows(self, data_class: DataClass) -> tuple[timedelta, timedelta, timedelta]:
        # expires, retain, delete-after grace
        if data_class == "public":
            return timedelta(days=7), timedelta(days=30), timedelta(days=31)
        if data_class == "internal":
            return timedelta(days=1), timedelta(days=14), timedelta(days=15)
        if data_class == "sensitive":
            return timedelta(hours=1), timedelta(days=7), timedelta(days=8)
        if data_class == "secret":
            return timedelta(minutes=10), timedelta(hours=1), timedelta(hours=2)
        raise ValueError(data_class)

    def stamp(self, entry: CacheEntry, now: datetime) -> CacheEntry:
        expire, retain, delete_after = self.windows(entry.data_class)
        return replace(
            entry,
            expires_at=now + expire,
            retain_until=now + retain,
            delete_after=now + delete_after,
        )


class RetentionSweeper:
    def __init__(self, store):
        self.store = store

    def sweep(self, now: datetime) -> dict[str, int]:
        expired = deleted = held = 0
        for entry in self.store.scan():
            if entry.expires_at <= now:
                self.store.mark_unusable(entry.key)
                expired += 1

            if entry.delete_after <= now:
                if entry.legal_hold:
                    self.store.deny_reads(entry.key, reason="legal_hold")
                    held += 1
                else:
                    self.store.delete(entry.key)
                    deleted += 1
        return {"expired": expired, "deleted": deleted, "held": held}

    def forget_subject(self, subject_id: str, reason: str, now: datetime) -> int:
        count = 0
        for entry in self.store.find_by_subject(subject_id):
            if entry.legal_hold:
                self.store.deny_reads(entry.key, reason="rtbf_legal_hold")
                continue
            if entry.erasable:
                self.store.tombstone(entry.key, subject_id, reason, now)
                self.store.delete(entry.key)
                count += 1
        return count


now = datetime.now(timezone.utc)
policy = RetentionPolicy()
entry = CacheEntry(
    key="gmail:user_42:last_unread",
    subject_id="user_42",
    data_class="sensitive",
    value={"subject": "invoice", "from": "vendor@example.com"},
    expires_at=now,
    retain_until=now,
    delete_after=now,
)
entry = policy.stamp(entry, now)
```

教学版重点不是存储实现，而是两个动作分开：

1. `sweep()`：定时清理过期/到期删除；
2. `forget_subject()`：用户删除请求触发按 `subject_id` 全局遗忘。

## 4. pi-mono：Retention 中间件

```ts
// pi-mono/cache/RetentionMiddleware.ts
type DataClass = "public" | "internal" | "sensitive" | "secret";

type RetentionWindow = {
  expiresMs: number;
  retainMs: number;
  deleteAfterMs: number;
};

type CacheEnvelope<T> = {
  key: string;
  tenant: string;
  subjectId: string;
  dataClass: DataClass;
  value: T;
  expiresAt: string;
  retainUntil: string;
  deleteAfter: string;
  erasable: boolean;
  legalHold: boolean;
  tombstoneVersion: number;
};

export class RetentionMiddleware {
  constructor(
    private windows: Record<DataClass, RetentionWindow>,
    private index: SubjectCacheIndex,
    private audit: (event: string, payload: Record<string, unknown>) => void,
  ) {}

  beforeWrite<T>(input: Omit<CacheEnvelope<T>, "expiresAt" | "retainUntil" | "deleteAfter">): CacheEnvelope<T> {
    const now = Date.now();
    const window = this.windows[input.dataClass];

    const envelope = {
      ...input,
      expiresAt: new Date(now + window.expiresMs).toISOString(),
      retainUntil: new Date(now + window.retainMs).toISOString(),
      deleteAfter: new Date(now + window.deleteAfterMs).toISOString(),
    };

    this.index.add(input.tenant, input.subjectId, input.key);
    this.audit("cache_retention_stamp", {
      tenant: input.tenant,
      subjectId: input.subjectId,
      key: input.key,
      dataClass: input.dataClass,
      expiresAt: envelope.expiresAt,
      deleteAfter: envelope.deleteAfter,
    });

    return envelope;
  }

  beforeRead<T>(envelope: CacheEnvelope<T>, now = new Date()): T | undefined {
    if (envelope.legalHold) {
      this.audit("cache_read_denied", { key: envelope.key, reason: "legal_hold" });
      return undefined;
    }

    if (new Date(envelope.expiresAt) <= now) {
      this.audit("cache_read_denied", { key: envelope.key, reason: "expired" });
      return undefined;
    }

    return envelope.value;
  }
}

export interface SubjectCacheIndex {
  add(tenant: string, subjectId: string, key: string): void;
  keysForSubject(tenant: string, subjectId: string): Promise<string[]>;
}
```

关键设计：写入缓存时同步维护 `SubjectCacheIndex`。否则真的收到“删除 user_42 所有数据”时，只能全库扫描，慢而且容易漏掉 L1/L2/L3 的副本。

## 5. Right-to-be-Forgotten 执行器

RTBF 不能只删 Redis。建议做成一个多阶段 job：

```ts
// pi-mono/cache/ForgetSubjectJob.ts
export async function forgetSubject(args: {
  tenant: string;
  subjectId: string;
  reason: string;
  requestedBy: string;
  index: SubjectCacheIndex;
  cache: { delete(key: string): Promise<void>; tombstone(key: string, version: number): Promise<void> };
  vector: { deleteBySubject(subjectId: string): Promise<number> };
  files: { deleteBySubject(subjectId: string): Promise<number> };
  audit: (event: string, payload: Record<string, unknown>) => void;
}) {
  const keys = await args.index.keysForSubject(args.tenant, args.subjectId);

  let cacheDeleted = 0;
  for (const key of keys) {
    await args.cache.tombstone(key, Date.now());
    await args.cache.delete(key);
    cacheDeleted++;
  }

  const vectorDeleted = await args.vector.deleteBySubject(args.subjectId);
  const fileDeleted = await args.files.deleteBySubject(args.subjectId);

  args.audit("subject_forget_completed", {
    tenant: args.tenant,
    subjectId: args.subjectId,
    requestedBy: args.requestedBy,
    reason: args.reason,
    cacheDeleted,
    vectorDeleted,
    fileDeleted,
  });
}
```

注意：**先写 tombstone，再 delete**。否则后台 refresh、旧 worker、异步子任务可能把刚删的数据重新写回来。

## 6. OpenClaw 实战：文件缓存也要可遗忘

OpenClaw 很常见的缓存形态是文件：

```text
.openclaw/workspace/cache/
  tool-results/gmail/user_42/...
  vector-index/user_42/...
  summaries/user_42/...
  audit/cache-retention.jsonl
  tombstones/user_42.json
```

建议每条缓存旁边配一个 `.meta.json`：

```json
{
  "tenant": "default",
  "subjectId": "user_42",
  "dataClass": "sensitive",
  "source": "gmail.unread",
  "expiresAt": "2026-05-09T10:30:00+11:00",
  "retainUntil": "2026-05-16T00:00:00+10:00",
  "deleteAfter": "2026-05-17T00:00:00+10:00",
  "erasable": true,
  "legalHold": false
}
```

Heartbeat/Cron 可以跑一个轻量 sweeper：

```bash
find .openclaw/workspace/cache -name '*.meta.json' -print0 |
  xargs -0 jq -r 'select(.deleteAfter < now | not) | .key'
```

生产里不要直接照抄这个 shell；它表达的是模式：**缓存不是只有 value 文件，还必须有可扫描、可审计、可删除的 metadata**。

## 7. 常见坑

- **只删主缓存，不删派生缓存**：摘要、embedding、report 都可能含用户信息；
- **只删磁盘，不删内存 L1**：进程内缓存要订阅 invalidation bus；
- **只删 value，不删索引**：subject index 不清理会造成幽灵引用；
- **legal hold 还能被读**：保留是为了合规，不代表业务可继续使用；
- **审计日志写敏感原文**：RTBF 审计只记 hash/count/reason，不记被删内容。

## 8. Checklist

设计缓存保留策略时问 6 个问题：

- 这条缓存属于哪个 `subjectId`？
- 它的 `dataClass` 是什么？
- 多久后不能再回答用户？
- 最晚什么时候必须删除/匿名化？
- 用户删除请求能不能立即清掉？
- 删除时会不会漏掉 L1、L2、文件、向量库、摘要和子任务中间结果？

## 9. 小结

缓存安全不是“加密 + 脱敏”就结束了。真正成熟的 Agent 缓存要能回答三句话：

1. 我为什么还保留这份数据？
2. 到什么时候必须停止使用它？
3. 用户要求删除时，我如何证明所有副本都忘了？

下一次你写 `cache.set()`，别只传 TTL。把 `subjectId / dataClass / expiresAt / retainUntil / deleteAfter / erasable` 一起写进去。**会记住是能力，会按规则忘记才是可靠。**
