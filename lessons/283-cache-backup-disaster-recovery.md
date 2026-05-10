# 283. Agent 缓存备份与灾难恢复（Cache Backup & Disaster Recovery）

> 缓存不是主数据库，但在 Agent 系统里，缓存经常承载工具结果、摘要、证据、向量索引、权限投影和运行时优化状态。缓存丢了不一定丢数据，但可能让 Agent 变慢、变贵、忘记已验证事实，甚至在恢复过程中把旧数据复活。

上一课讲了缓存损坏隔离。今天继续往生产化走一步：**缓存备份与灾难恢复**。

核心思想：

> 缓存可以重建，但重建必须可控；备份可以恢复，但恢复必须验证、分级、可回滚。

---

## 1. 为什么 Agent 缓存也需要 DR？

传统工程里很多人会说：缓存坏了就删掉重建。

对简单 KV 缓存没问题，但 Agent 的缓存常常包含：

- RAG 检索结果和摘要索引
- 昂贵工具调用结果，例如网页抓取、账单查询、日志聚合
- 已校验过的 Evidence / Citation 来源
- 会话级工作状态和中间推理产物
- 多租户隔离后的安全投影
- 减少 LLM token 的压缩上下文

如果这些全丢：

1. **成本暴涨**：大量重新回源、重新 embedding、重新 summarization。
2. **延迟飙升**：冷缓存导致每个 turn 都像首次运行。
3. **源站被打穿**：所有 worker 同时 miss，形成灾难级 stampede。
4. **恢复污染**：从旧备份恢复了已删除、已吊销、已 tombstone 的数据。
5. **审计断链**：缓存命中曾参与决策，但恢复后无法证明来源。

所以成熟 Agent 的缓存 DR 不是“把目录 tar 一下”，而是设计一条恢复流水线。

---

## 2. 四类缓存，四种恢复策略

不要对所有缓存一刀切。先按 **可重建性 + 风险** 分类：

| 类别 | 示例 | 恢复策略 |
|---|---|---|
| Ephemeral 临时缓存 | L1 内存、短 TTL 搜索结果 | 不备份，丢了就丢 |
| Rebuildable 可重建缓存 | embedding index、网页摘要、API 聚合 | 备份元数据 + 后台限速重建 |
| Expensive 昂贵缓存 | 大批量账单、日志分析、长文档解析 | 定期快照 + 增量日志 |
| Sensitive/Risk 高风险缓存 | 权限投影、PII 脱敏结果、审计证据 | 加密备份 + 恢复校验 + tombstone 优先 |

关键规则：

- **临时缓存不恢复**，避免把短期状态错误复活。
- **可重建缓存优先重建而不是盲目恢复旧值**。
- **昂贵缓存恢复后必须校验 hash/signature/sourceVersion**。
- **敏感缓存恢复时 tombstone / revocation / retention policy 永远优先于备份内容**。

---

## 3. learn-claude-code：文件式 Cache Snapshot 教学版

教学版先用本地文件实现一个最小 DR 模型：

```python
# learn_claude_code/cache_dr.py
from __future__ import annotations

import hashlib
import json
import os
import shutil
import time
from dataclasses import dataclass, asdict
from pathlib import Path
from typing import Any, Literal

CacheClass = Literal["ephemeral", "rebuildable", "expensive", "sensitive"]


@dataclass
class CacheEntry:
    key: str
    value: Any
    cache_class: CacheClass
    source_version: str
    expires_at: float
    tombstone_version: int = 0

    def content_hash(self) -> str:
        raw = json.dumps(self.value, ensure_ascii=False, sort_keys=True)
        return hashlib.sha256(raw.encode()).hexdigest()


@dataclass
class SnapshotManifest:
    snapshot_id: str
    created_at: float
    entries: list[dict]


def write_entry(cache_dir: Path, entry: CacheEntry) -> None:
    cache_dir.mkdir(parents=True, exist_ok=True)
    payload = asdict(entry) | {"content_hash": entry.content_hash()}
    (cache_dir / f"{safe_key(entry.key)}.json").write_text(
        json.dumps(payload, ensure_ascii=False, indent=2)
    )


def safe_key(key: str) -> str:
    return hashlib.sha256(key.encode()).hexdigest()


def create_snapshot(cache_dir: Path, backup_dir: Path) -> SnapshotManifest:
    snapshot_id = time.strftime("snapshot-%Y%m%d-%H%M%S")
    target = backup_dir / snapshot_id
    target.mkdir(parents=True, exist_ok=True)

    entries: list[dict] = []
    for file in cache_dir.glob("*.json"):
        data = json.loads(file.read_text())

        # ephemeral 不备份，避免恢复短期噪音
        if data["cache_class"] == "ephemeral":
            continue

        shutil.copy2(file, target / file.name)
        entries.append({
            "file": file.name,
            "key": data["key"],
            "cache_class": data["cache_class"],
            "content_hash": data["content_hash"],
            "source_version": data["source_version"],
            "expires_at": data["expires_at"],
            "tombstone_version": data.get("tombstone_version", 0),
        })

    manifest = SnapshotManifest(snapshot_id=snapshot_id, created_at=time.time(), entries=entries)
    (target / "manifest.json").write_text(json.dumps(asdict(manifest), ensure_ascii=False, indent=2))
    return manifest


def restore_snapshot(snapshot_dir: Path, cache_dir: Path, active_tombstones: dict[str, int]) -> dict:
    manifest = json.loads((snapshot_dir / "manifest.json").read_text())
    cache_dir.mkdir(parents=True, exist_ok=True)

    restored, skipped, quarantined = 0, 0, 0

    for item in manifest["entries"]:
        source_file = snapshot_dir / item["file"]
        data = json.loads(source_file.read_text())

        # 1. Hash 校验：备份文件被改过就隔离
        raw = json.dumps(data["value"], ensure_ascii=False, sort_keys=True)
        if hashlib.sha256(raw.encode()).hexdigest() != data["content_hash"]:
            quarantine(snapshot_dir, source_file, reason="hash_mismatch")
            quarantined += 1
            continue

        # 2. Tombstone 优先：当前系统说它已删除，就不能从备份复活
        active_version = active_tombstones.get(data["key"], 0)
        if active_version > data.get("tombstone_version", 0):
            skipped += 1
            continue

        # 3. 过期检查：过期的 rebuildable 不恢复，只进入后台重建队列
        if data["expires_at"] < time.time() and data["cache_class"] == "rebuildable":
            enqueue_rebuild(data["key"], data["source_version"])
            skipped += 1
            continue

        shutil.copy2(source_file, cache_dir / source_file.name)
        restored += 1

    return {"restored": restored, "skipped": skipped, "quarantined": quarantined}


def quarantine(snapshot_dir: Path, file: Path, reason: str) -> None:
    q = snapshot_dir / "quarantine"
    q.mkdir(exist_ok=True)
    os.rename(file, q / f"{reason}-{file.name}")


def enqueue_rebuild(key: str, source_version: str) -> None:
    # 教学版：真实系统可以写 SQLite/Redis 队列
    print(f"REBUILD {key=} {source_version=}")
```

这段代码有三个关键点：

1. snapshot 只备份非 ephemeral 缓存。
2. restore 前先做 hash 校验。
3. tombstone 版本比备份新时，拒绝恢复，防止“已删除数据复活”。

---

## 4. pi-mono：生产版 CacheRecoveryManager

生产系统建议把 DR 做成中间层，而不是散落在脚本里。

```ts
// pi-mono/packages/agent-runtime/src/cache/cache-recovery.ts
import { createHash } from "crypto";

export type CacheClass = "ephemeral" | "rebuildable" | "expensive" | "sensitive";

export interface CacheEnvelope<T = unknown> {
  key: string;
  value: T;
  cacheClass: CacheClass;
  tenantId: string;
  sourceVersion: string;
  expiresAt: number;
  contentHash: string;
  signature?: string;
  tombstoneVersion: number;
  encrypted: boolean;
}

export interface BackupStore {
  listSnapshots(): Promise<string[]>;
  readManifest(snapshotId: string): Promise<BackupManifest>;
  readEntry(snapshotId: string, key: string): Promise<CacheEnvelope>;
}

export interface BackupManifest {
  snapshotId: string;
  createdAt: number;
  entries: Array<{
    key: string;
    cacheClass: CacheClass;
    tenantId: string;
    contentHash: string;
    sourceVersion: string;
  }>;
}

export interface RecoveryPolicy {
  shouldRestore(entry: CacheEnvelope, ctx: RecoveryContext): Promise<RecoveryDecision>;
}

export interface RecoveryContext {
  tenantId: string;
  activeTombstoneVersion: number;
  now: number;
  purpose: "answer" | "planning" | "external_side_effect" | "security_decision";
}

export type RecoveryDecision =
  | { action: "restore" }
  | { action: "skip"; reason: string }
  | { action: "rebuild"; reason: string }
  | { action: "quarantine"; reason: string };

export class CacheRecoveryManager {
  constructor(
    private readonly backup: BackupStore,
    private readonly cache: { put(entry: CacheEnvelope): Promise<void> },
    private readonly policy: RecoveryPolicy,
    private readonly rebuildQueue: { enqueue(key: string, sourceVersion: string): Promise<void> },
    private readonly audit: { record(event: unknown): Promise<void> },
  ) {}

  async restoreSnapshot(snapshotId: string, ctx: RecoveryContext) {
    const manifest = await this.backup.readManifest(snapshotId);
    const result = { restored: 0, skipped: 0, rebuild: 0, quarantined: 0 };

    for (const item of manifest.entries) {
      const entry = await this.backup.readEntry(snapshotId, item.key);

      const integrity = this.verifyIntegrity(entry);
      if (!integrity.ok) {
        await this.audit.record({ type: "cache.restore.quarantine", key: item.key, reason: integrity.reason });
        result.quarantined++;
        continue;
      }

      const decision = await this.policy.shouldRestore(entry, ctx);

      if (decision.action === "restore") {
        await this.cache.put(entry);
        result.restored++;
      } else if (decision.action === "rebuild") {
        await this.rebuildQueue.enqueue(entry.key, entry.sourceVersion);
        result.rebuild++;
      } else if (decision.action === "quarantine") {
        await this.audit.record({ type: "cache.restore.quarantine", key: entry.key, reason: decision.reason });
        result.quarantined++;
      } else {
        result.skipped++;
      }

      await this.audit.record({
        type: "cache.restore.decision",
        snapshotId,
        key: entry.key,
        cacheClass: entry.cacheClass,
        decision: decision.action,
        reason: "reason" in decision ? decision.reason : undefined,
      });
    }

    return result;
  }

  private verifyIntegrity(entry: CacheEnvelope): { ok: true } | { ok: false; reason: string } {
    const raw = JSON.stringify(entry.value);
    const hash = createHash("sha256").update(raw).digest("hex");
    if (hash !== entry.contentHash) {
      return { ok: false, reason: "content_hash_mismatch" };
    }
    return { ok: true };
  }
}
```

策略层单独抽出来：

```ts
// pi-mono/packages/agent-runtime/src/cache/recovery-policy.ts
export class DefaultRecoveryPolicy implements RecoveryPolicy {
  async shouldRestore(entry: CacheEnvelope, ctx: RecoveryContext): Promise<RecoveryDecision> {
    if (entry.cacheClass === "ephemeral") {
      return { action: "skip", reason: "ephemeral_not_restored" };
    }

    if (ctx.activeTombstoneVersion > entry.tombstoneVersion) {
      return { action: "skip", reason: "newer_tombstone_exists" };
    }

    if (entry.tenantId !== ctx.tenantId) {
      return { action: "quarantine", reason: "tenant_mismatch" };
    }

    if (ctx.now > entry.expiresAt && entry.cacheClass === "rebuildable") {
      return { action: "rebuild", reason: "expired_rebuildable" };
    }

    if (entry.cacheClass === "sensitive" && ctx.purpose !== "security_decision") {
      return { action: "skip", reason: "sensitive_requires_explicit_purpose" };
    }

    return { action: "restore" };
  }
}
```

这里的设计重点是：**恢复不是文件复制，而是 Policy Decision**。

---

## 5. OpenClaw 实战：Cron 驱动缓存快照与恢复演练

OpenClaw 这种 Always-on Agent，可以用 cron 做两件事：

1. **定期快照**：把高价值缓存、manifest、审计日志打包。
2. **定期恢复演练**：在隔离目录恢复一次，验证 hash、schema、tombstone、权限边界。

一个文件式结构可以这样设计：

```text
.openclaw/cache/
  entries/
    <cacheKeyHash>.json
  tombstones.jsonl
  audit.jsonl
  snapshots/
    snapshot-2026-05-10T02-30-00Z/
      manifest.json
      entries/
      audit-tail.jsonl
      restore-report.json
```

快照 Manifest 不存敏感明文，只记录可验证元数据：

```json
{
  "snapshotId": "snapshot-2026-05-10T02-30-00Z",
  "createdAt": "2026-05-10T02:30:00Z",
  "gitSha": "dbee1d0",
  "policyVersion": "cache-dr-v1",
  "entries": [
    {
      "keyHash": "sha256:9a4...",
      "cacheClass": "expensive",
      "tenantId": "tenant_123",
      "contentHash": "sha256:bb1...",
      "sourceVersion": "billing-api:2026-05-09",
      "encrypted": true,
      "expiresAt": "2026-05-11T00:00:00Z",
      "tombstoneVersion": 17
    }
  ]
}
```

恢复演练报告应该写清楚：

```json
{
  "snapshotId": "snapshot-2026-05-10T02-30-00Z",
  "checkedAt": "2026-05-10T03:00:00Z",
  "entries": 1420,
  "restorable": 1188,
  "rebuildQueued": 201,
  "skippedByTombstone": 29,
  "quarantined": 2,
  "maxRestoreLagMs": 840,
  "result": "warning"
}
```

这样一旦真实事故发生，Agent 不需要临场猜：

- 哪个 snapshot 可用？
- 有多少条能恢复？
- 哪些需要重建？
- 有没有敏感/跨租户风险？
- 恢复大概要多久？

答案都在最近一次恢复演练里。

---

## 6. 灾难恢复 Runbook

建议生产系统准备一个固定 Runbook：

1. **Freeze writes**：暂停缓存写入或切到只读模式。
2. **Collect evidence**：记录事故时间、影响范围、当前 tombstone/revocation 状态。
3. **Select snapshot**：选择事故前最近且演练通过的快照。
4. **Restore to staging**：先恢复到隔离缓存命名空间，不直接覆盖生产。
5. **Validate**：hash、signature、schema、tenant、retention、tombstone 全部校验。
6. **Warm critical keys**：先恢复/重建高优先级 key，低价值 key 延后。
7. **Progressive cutover**：按 tenant / keyClass / 百分比逐步切流量。
8. **Monitor**：观察 hit ratio、origin QPS、restore error、security mismatch。
9. **Postmortem**：归档恢复报告，更新备份策略。

一句话：

> 真正的 DR 不是“我有备份”，而是“我定期证明这份备份能安全恢复”。

---

## 7. 常见坑

- **只备份 value，不备份 manifest**：恢复后无法验证来源、版本、租户、风险等级。
- **恢复时忽略 tombstone**：最危险，会把已删除或已撤权数据复活。
- **直接覆盖生产缓存**：一旦备份污染，会把问题扩大。
- **没有限速重建**：恢复后所有 miss 同时回源，源站第二次事故。
- **敏感缓存明文备份**：备份系统本身变成数据泄露入口。
- **从不演练**：等事故发生才发现备份格式早就不兼容。

---

## 8. Checklist

设计 Agent 缓存 DR 时，至少检查：

- [ ] 缓存按 cacheClass 分类了吗？
- [ ] ephemeral 缓存明确不恢复了吗？
- [ ] manifest 记录 contentHash/sourceVersion/tenant/tombstoneVersion 了吗？
- [ ] 恢复前做 hash/signature/schema 校验了吗？
- [ ] tombstone / retention / revocation 是否优先于备份？
- [ ] sensitive 缓存是否加密且按 purpose 恢复？
- [ ] rebuildable 缓存是否进入限速重建队列？
- [ ] 是否定期做隔离恢复演练？
- [ ] 是否有 restore-report 和审计日志？
- [ ] 是否支持渐进切流和快速回滚？

---

## 总结

缓存 DR 的目标不是让缓存永不丢，而是：

1. 丢了能知道影响范围。
2. 恢复时不复活不该复活的数据。
3. 重建时不打穿源站。
4. 恢复后能证明每条缓存为什么可信。

成熟 Agent 的缓存系统，既要快，也要可恢复、可验证、可审计。

> 缓存可以是加速器，但恢复流程必须是安全系统。没有演练过的备份，只是心理安慰。
