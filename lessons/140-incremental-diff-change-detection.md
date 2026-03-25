# 140 · Agent 增量 Diff 与变更检测（Incremental Diff & Change Detection）

> **核心思想**：Agent 不应该每次都全量扫描。只处理"变化了的部分"，才能做到低延迟、低成本、高频监控。

---

## 为什么需要变更检测？

想象一个监控 Agent，每 5 分钟检查一次 GitHub PR、数据库订单、配置文件……

**全量扫描的问题：**
```
每次都拉全量数据 → Token 爆炸 + 延迟高 + LLM 处理重复信息
```

**增量 Diff 的解法：**
```
只把"新增 / 修改 / 删除"的部分交给 LLM → 精准、快、省钱
```

---

## 核心模式：指纹 + 快照 + Diff

```
上次快照 ──┐
            ├──→ Diff Engine ──→ ChangeSet ──→ Agent 只处理变化
本次数据 ──┘
```

### ChangeSet 数据结构

```typescript
interface ChangeSet<T> {
  added:   T[];      // 新增
  removed: T[];      // 删除  
  updated: { before: T; after: T }[];  // 修改
  unchanged: number; // 未变化数量（仅统计，不传给 LLM）
}
```

---

## 实现：通用 Diff 引擎

```typescript
// lib/diff-engine.ts
import crypto from 'crypto';

export interface Snapshot<T> {
  fingerprint: string;           // 整体指纹，快速判断有无变化
  items: Map<string, { hash: string; data: T }>;
  capturedAt: number;
}

export class DiffEngine<T> {
  constructor(
    private keyFn: (item: T) => string,    // 提取唯一 key
    private hashFn: (item: T) => string,   // 计算项目哈希
  ) {}

  /** 从数组构建快照 */
  buildSnapshot(items: T[]): Snapshot<T> {
    const map = new Map<string, { hash: string; data: T }>();
    for (const item of items) {
      const key = this.keyFn(item);
      const hash = this.hashFn(item);
      map.set(key, { hash, data: item });
    }
    // 整体指纹：所有 key+hash 排序后 MD5
    const fingerprint = crypto
      .createHash('md5')
      .update([...map.entries()].map(([k, v]) => `${k}:${v.hash}`).sort().join('|'))
      .digest('hex');
    return { fingerprint, items: map, capturedAt: Date.now() };
  }

  /** 计算两个快照之间的变化 */
  diff(prev: Snapshot<T>, curr: Snapshot<T>): ChangeSet<T> {
    // 指纹相同 → 完全没变化，直接返回
    if (prev.fingerprint === curr.fingerprint) {
      return { added: [], removed: [], updated: [], unchanged: curr.items.size };
    }

    const added: T[] = [];
    const removed: T[] = [];
    const updated: { before: T; after: T }[] = [];
    let unchanged = 0;

    // 检查当前快照中的每一项
    for (const [key, { hash, data }] of curr.items) {
      const prevItem = prev.items.get(key);
      if (!prevItem) {
        added.push(data);                            // 新增
      } else if (prevItem.hash !== hash) {
        updated.push({ before: prevItem.data, after: data });  // 修改
      } else {
        unchanged++;                                  // 未变化
      }
    }

    // 检查删除的项
    for (const [key, { data }] of prev.items) {
      if (!curr.items.has(key)) {
        removed.push(data);
      }
    }

    return { added, removed, updated, unchanged };
  }
}
```

---

## 场景 1：监控 GitHub PR 状态变化

```typescript
// tools/github-pr-monitor.ts
import { DiffEngine } from '../lib/diff-engine';
import { readFileSync, writeFileSync, existsSync } from 'fs';

interface PR {
  number: number;
  title: string;
  state: 'open' | 'closed' | 'merged';
  reviewState: string;
  updatedAt: string;
}

const prDiffer = new DiffEngine<PR>(
  pr => String(pr.number),                          // key: PR 编号
  pr => `${pr.state}|${pr.reviewState}|${pr.updatedAt}`,  // hash: 关注的字段
);

const SNAPSHOT_FILE = '/tmp/pr-snapshot.json';

async function monitorPRs(repo: string): Promise<string> {
  // 拉取当前 PR 列表
  const currentPRs: PR[] = await fetchPRs(repo);
  const currentSnap = prDiffer.buildSnapshot(currentPRs);

  // 加载上次快照
  const prevSnap = existsSync(SNAPSHOT_FILE)
    ? JSON.parse(readFileSync(SNAPSHOT_FILE, 'utf-8'))
    : prDiffer.buildSnapshot([]);  // 首次运行，视全部为新增

  const changes = prDiffer.diff(prevSnap, currentSnap);

  // 保存新快照
  writeFileSync(SNAPSHOT_FILE, JSON.stringify(currentSnap));

  // 没有变化 → 不打扰 LLM
  if (changes.added.length === 0 && changes.removed.length === 0 && changes.updated.length === 0) {
    return `UNCHANGED: ${changes.unchanged} PRs, no changes since last check`;
  }

  // 只把变化部分交给 LLM 分析
  return JSON.stringify({
    summary: `${changes.added.length} new, ${changes.updated.length} updated, ${changes.removed.length} closed`,
    changes,  // LLM 只看 diff，不看全量
  });
}
```

---

## 场景 2：配置文件变更检测（文件系统）

```typescript
// lib/file-watcher.ts
import { createHash } from 'crypto';
import { readFileSync, statSync } from 'fs';

export class FileChangeDetector {
  private hashes = new Map<string, string>();

  /** 计算文件哈希 */
  private fileHash(path: string): string {
    try {
      return createHash('sha256').update(readFileSync(path)).digest('hex');
    } catch {
      return 'MISSING';
    }
  }

  /** 检查文件是否变化，返回 diff */
  check(paths: string[]): { changed: string[]; deleted: string[]; new: string[] } {
    const changed: string[] = [];
    const deleted: string[] = [];
    const newFiles: string[] = [];

    for (const path of paths) {
      const currentHash = this.fileHash(path);
      const prevHash = this.hashes.get(path);

      if (!prevHash && currentHash !== 'MISSING') {
        newFiles.push(path);                    // 首次出现
      } else if (prevHash && currentHash === 'MISSING') {
        deleted.push(path);                     // 文件被删除
      } else if (prevHash && prevHash !== currentHash) {
        changed.push(path);                     // 内容变化
      }

      if (currentHash !== 'MISSING') {
        this.hashes.set(path, currentHash);
      } else {
        this.hashes.delete(path);
      }
    }

    return { changed, deleted, new: newFiles };
  }
}

// OpenClaw Agent Tool
export const checkConfigFiles = {
  name: 'check_config_files',
  description: '检测配置文件变更，只返回有变化的文件',
  input_schema: {
    type: 'object',
    properties: {
      paths: { type: 'array', items: { type: 'string' }, description: '要监控的文件路径列表' },
    },
    required: ['paths'],
  },
  async execute({ paths }: { paths: string[] }) {
    const detector = getDetectorInstance();  // 单例，跨调用保持状态
    const result = detector.check(paths);
    
    if (result.changed.length === 0 && result.deleted.length === 0 && result.new.length === 0) {
      return { status: 'no_changes' };
    }
    return result;
  },
};
```

---

## 场景 3：数据库订单增量拉取

```typescript
// tools/order-monitor.ts
// 用时间戳 + 游标替代全量扫描

interface OrderCursor {
  lastId: number;
  lastCheckedAt: string;
}

async function fetchNewOrders(cursor: OrderCursor): Promise<{
  orders: Order[];
  nextCursor: OrderCursor;
}> {
  // 只拉取 cursor 之后的订单
  const orders = await db.query(`
    SELECT * FROM orders 
    WHERE id > ? OR (id = ? AND updated_at > ?)
    ORDER BY id ASC
    LIMIT 100
  `, [cursor.lastId, cursor.lastId, cursor.lastCheckedAt]);

  const nextCursor: OrderCursor = orders.length > 0
    ? { lastId: orders[orders.length - 1].id, lastCheckedAt: new Date().toISOString() }
    : cursor;  // 没有新数据，游标不动

  return { orders, nextCursor };
}

// 持久化游标（Redis 或文件）
async function runOrderMonitorAgent() {
  const cursor = await loadCursor() ?? { lastId: 0, lastCheckedAt: new Date().toISOString() };
  const { orders, nextCursor } = await fetchNewOrders(cursor);
  
  if (orders.length === 0) {
    console.log('No new orders');
    return;
  }
  
  // 只把增量数据给 LLM
  await agent.process(`处理以下新订单: ${JSON.stringify(orders)}`);
  await saveCursor(nextCursor);
}
```

---

## OpenClaw 实战：心跳中的变更检测

OpenClaw 的 Heartbeat 天然适合增量检测模式：

```typescript
// HEARTBEAT.md 会触发类似这样的逻辑

// heartbeat-handler.ts
const EMAIL_SNAPSHOT_KEY = 'heartbeat:email:snapshot';

async function emailHeartbeatCheck() {
  const emails = await fetchRecentEmails(50);
  
  // 构建当前快照
  const currentSnap = emailDiffer.buildSnapshot(emails);
  
  // 与上次比较
  const prevSnap = await redis.get(EMAIL_SNAPSHOT_KEY);
  
  if (prevSnap && prevSnap.fingerprint === currentSnap.fingerprint) {
    return 'HEARTBEAT_OK';  // 完全没变化，跳过 LLM
  }
  
  const changes = prevSnap
    ? emailDiffer.diff(prevSnap, currentSnap)
    : { added: emails, removed: [], updated: [], unchanged: 0 };
  
  await redis.set(EMAIL_SNAPSHOT_KEY, currentSnap);
  
  // 只处理新邮件
  if (changes.added.length > 0) {
    return `新邮件 ${changes.added.length} 封：${changes.added.map(e => e.subject).join(', ')}`;
  }
  
  return 'HEARTBEAT_OK';
}
```

---

## 关键设计原则

| 原则 | 做法 |
|------|------|
| **两层指纹** | 整体指纹快速判断，项目指纹精确定位 |
| **游标优于全量** | DB/API 分页用游标，避免 OFFSET 扫全表 |
| **快照外部化** | Redis / 文件存储，跨会话保持状态 |
| **不变化不打扰 LLM** | `UNCHANGED` 直接返回，节省 Token |
| **ChangeSet 小而精** | 只把 diff 交给 LLM，不传原始全量 |
| **首次运行兼容** | 无历史快照时，视全部为新增 |

---

## 适用场景

- 📊 **监控 Agent** - PR/Issue/告警/日志变更通知
- 🔄 **同步 Agent** - 两个系统间数据对齐
- 📁 **文件 Agent** - 代码变更触发 Review/Deploy
- 📧 **邮件/消息 Agent** - 新收件增量处理
- 🛒 **业务 Agent** - 订单/库存/价格变化监控

---

## 总结

```
全量扫描  →  O(n) Token 消耗，重复处理
增量 Diff →  O(k) Token 消耗，只处理变化 (k << n)
```

**三步走：**
1. `buildSnapshot(data)` — 构建当前快照（带指纹）
2. `diff(prev, curr)` — 计算 ChangeSet
3. 只把 ChangeSet 交给 Agent 处理，unchanged 计数即可

这是让 Agent 做到"低成本高频监控"的核心手段。OpenClaw 的心跳机制、Cron 任务都可以直接套用这个模式。
