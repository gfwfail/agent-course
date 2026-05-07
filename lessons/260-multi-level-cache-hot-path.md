# 260. Agent 多级缓存与热路径优化（Multi-Level Cache & Hot Path Optimization）

前面几课我们讲了缓存失效、击穿、预热、可观测性、一致性、权限、生命周期、压缩和投毒防御。今天补一个更工程化的问题：**缓存不应该只有一层**。

一句话：

> Multi-Level Cache 是把缓存拆成 L1 进程内存、L2 Redis/SQLite、L3 对象存储/原始来源，用热路径优先命中最快层，冷路径再逐级回源，同时保持失效传播和安全边界。

Agent 系统里，最慢的不一定是模型，有时是重复读取 prompt 片段、工具 schema、用户偏好、文档摘要、GitHub/邮箱/API 查询。单层 Redis 缓存能解决一部分问题，但高频小对象仍然会被网络延迟拖慢；只用进程内存又会遇到多实例不一致、重启丢失、权限污染。

所以成熟 Agent 缓存常用三层：

```text
L1 memory   -> 毫秒级/微秒级，适合当前进程热数据
L2 shared   -> Redis/SQLite/Postgres，适合跨 run/跨 worker 共享
L3 source   -> 原始工具/API/文件/对象存储，适合权威回源
```

核心不是“层数越多越好”，而是：**读走热路径，写走一致性路径，失效必须能穿透所有层**。

---

## 1. 为什么 Agent 需要多级缓存

典型场景：

- system prompt、工具 schema、技能文件：每个 turn 都会用，适合 L1；
- 用户画像、会话摘要、租户配置：跨会话复用，适合 L2；
- 大网页、PDF 原文、工具原始响应：体积大、访问少，适合 L3 pointer；
- GitHub PR 状态、订单状态、账单数据：可缓存但高风险动作前必须 live check。

如果所有读取都走 L2 Redis：

- 每次 tool selection 都多一次网络 hop；
- Redis 短暂抖动会拖慢 Agent Loop；
- 热点 key 会把共享缓存打爆。

如果所有读取都走 L1 memory：

- 多 worker 之间状态不一致；
- 外部写后旧进程继续读旧值；
- 权限、租户、版本变更难以及时生效。

多级缓存的价值就在于：

1. L1 负责快；
2. L2 负责共享；
3. L3 负责权威；
4. invalidation bus 负责让它们不要各自为政。

---

## 2. 设计原则：热路径、版本号、失效广播

### A. Cache key 必须带版本

不要只用：

```text
user:123:profile
```

更推荐：

```text
tenant:t1:user:123:profile:v7:scope:readonly
```

版本可以来自：

- schemaVersion：结构升级；
- configVersion：配置变更；
- permissionVersion：权限变化；
- contentHash：文件/工具 schema 变化。

这样旧 key 不用马上删除，也不会被新逻辑误读。

### B. L1 要短 TTL + 小容量

L1 是 hot path，不是长期记忆。建议：

- TTL：5 秒到 5 分钟；
- 容量：按 key class 限制；
- 敏感数据默认不进 L1，或者只存脱敏 envelope；
- 高风险用途读取时绕过 L1。

### C. 写入和失效必须广播

写 L2 后要发 invalidation event：

```json
{
  "type": "cache.invalidate",
  "tags": ["tenant:t1", "user:123", "tool_schema"],
  "version": 8,
  "reason": "permission_changed"
}
```

每个 worker 收到后清自己的 L1。否则 Redis 已经更新，进程内还在读旧值。

---

## 3. learn-claude-code：Python 教学版

下面是一个最小多级缓存：L1 用 dict，L2 用 dict 模拟 Redis，miss 时回源 loader。重点看读取顺序和失效标签。

```python
# learn_claude_code/multi_level_cache.py
from __future__ import annotations

import time
from dataclasses import dataclass, field
from typing import Any, Callable

@dataclass
class CacheEntry:
    value: Any
    expires_at: float
    tags: set[str] = field(default_factory=set)
    version: int = 1

    def fresh(self) -> bool:
        return time.time() < self.expires_at

class MultiLevelCache:
    def __init__(self) -> None:
        self.l1: dict[str, CacheEntry] = {}
        self.l2: dict[str, CacheEntry] = {}  # 教学版：生产可换 Redis/SQLite
        self.index: dict[str, set[str]] = {}

    def _remember_tags(self, key: str, tags: set[str]) -> None:
        for tag in tags:
            self.index.setdefault(tag, set()).add(key)

    def set(self, key: str, value: Any, *, ttl: float, tags: set[str], version: int = 1) -> None:
        l2_entry = CacheEntry(value=value, expires_at=time.time() + ttl, tags=tags, version=version)
        l1_entry = CacheEntry(value=value, expires_at=time.time() + min(ttl, 30), tags=tags, version=version)
        self.l2[key] = l2_entry
        self.l1[key] = l1_entry
        self._remember_tags(key, tags)

    def get(self, key: str, loader: Callable[[], Any], *, ttl: float, tags: set[str]) -> Any:
        # 1) L1 hot path
        entry = self.l1.get(key)
        if entry and entry.fresh():
            return entry.value

        # 2) L2 shared cache
        entry = self.l2.get(key)
        if entry and entry.fresh():
            # 回填 L1，但 L1 TTL 更短
            self.l1[key] = CacheEntry(
                value=entry.value,
                expires_at=time.time() + min(ttl, 30),
                tags=entry.tags,
                version=entry.version,
            )
            return entry.value

        # 3) L3/source 回源
        value = loader()
        self.set(key, value, ttl=ttl, tags=tags)
        return value

    def invalidate_tags(self, tags: set[str]) -> None:
        keys: set[str] = set()
        for tag in tags:
            keys.update(self.index.get(tag, set()))

        for key in keys:
            self.l1.pop(key, None)
            self.l2.pop(key, None)

# 用法：工具 schema 是典型热路径
cache = MultiLevelCache()

def load_tool_schema() -> dict[str, Any]:
    print("expensive schema scan")
    return {"tools": ["read", "write", "message"]}

schema = cache.get(
    "tenant:t1:tool_schema:v3",
    load_tool_schema,
    ttl=300,
    tags={"tenant:t1", "tool_schema"},
)
```

这个版本很小，但已经具备生产模型：L1 快速命中，L2 跨 run 共享，tag invalidate 同时清两层。

---

## 4. pi-mono：TypeScript 生产版中间件

生产里建议把多级缓存做成统一接口，让 Agent Loop 和工具层只感知 `getOrLoad`。

```ts
// pi-mono/cache/multi-level-cache.ts
export interface CacheEnvelope<T> {
  key: string;
  value: T;
  expiresAt: number;
  tags: string[];
  version: number;
  trust: "trusted" | "verified" | "untrusted" | "derived";
}

export interface CacheLayer {
  get<T>(key: string): Promise<CacheEnvelope<T> | null>;
  set<T>(entry: CacheEnvelope<T>): Promise<void>;
  delete(key: string): Promise<void>;
}

export interface CacheLoader<T> {
  load(): Promise<T>;
  ttlMs: number;
  tags: string[];
  version: number;
  trust: CacheEnvelope<T>["trust"];
  allowL1?: boolean;
}

export class MultiLevelCache {
  constructor(
    private readonly l1: CacheLayer,
    private readonly l2: CacheLayer,
    private readonly now = () => Date.now(),
  ) {}

  async getOrLoad<T>(key: string, loader: CacheLoader<T>): Promise<T> {
    if (loader.allowL1 !== false) {
      const hit1 = await this.l1.get<T>(key);
      if (hit1 && hit1.expiresAt > this.now() && hit1.version === loader.version) {
        return hit1.value;
      }
    }

    const hit2 = await this.l2.get<T>(key);
    if (hit2 && hit2.expiresAt > this.now() && hit2.version === loader.version) {
      if (loader.allowL1 !== false) {
        await this.l1.set({ ...hit2, expiresAt: this.now() + Math.min(loader.ttlMs, 30_000) });
      }
      return hit2.value;
    }

    const value = await loader.load();
    const entry: CacheEnvelope<T> = {
      key,
      value,
      expiresAt: this.now() + loader.ttlMs,
      tags: loader.tags,
      version: loader.version,
      trust: loader.trust,
    };

    await this.l2.set(entry);
    if (loader.allowL1 !== false) {
      await this.l1.set({ ...entry, expiresAt: this.now() + Math.min(loader.ttlMs, 30_000) });
    }

    return value;
  }
}
```

工具中间件可以这样用：

```ts
const schema = await cache.getOrLoad("tool_schema:v12", {
  ttlMs: 5 * 60_000,
  tags: ["tool_schema", "runtime_config"],
  version: 12,
  trust: "trusted",
  allowL1: true,
  load: () => toolRegistry.buildSchema(),
});
```

高风险数据则关闭 L1：

```ts
const paymentStatus = await cache.getOrLoad(`payment:${id}:status`, {
  ttlMs: 10_000,
  tags: [`payment:${id}`],
  version: 1,
  trust: "verified",
  allowL1: false, // 避免进程内旧状态影响副作用
  load: () => paymentApi.fetchStatus(id),
});
```

关键点：多级缓存不是到处 `if memory else redis else api`，而是统一在 cache middleware 里实现，业务只声明用途、TTL、tag、version、trust。

---

## 5. OpenClaw 实战：课程 Cron 的热路径

OpenClaw 这种 always-on agent 很适合用多级缓存：

- L1：当前 session 内的技能文件、README 片段、课程目录摘要；
- L2：`workspace/.cache/agent-course/*.json` 保存最近 lesson index、已讲 topic hash；
- L3：GitHub repo、TOOLS.md、Telegram 发送结果、真实 git 状态。

一个文件式 L2 示例：

```python
# .openclaw/cache_index.py
import json, time
from pathlib import Path

CACHE = Path("/Users/bot001/.openclaw/workspace/.cache/agent-course-index.json")

def read_index(max_age_seconds: int = 300):
    if not CACHE.exists():
        return None
    data = json.loads(CACHE.read_text())
    if time.time() - data["observedAt"] > max_age_seconds:
        return None
    return data

def write_index(lessons: list[str], readme_hash: str):
    CACHE.parent.mkdir(parents=True, exist_ok=True)
    CACHE.write_text(json.dumps({
        "observedAt": time.time(),
        "readmeHash": readme_hash,
        "lessons": lessons,
        "tags": ["agent-course", "lesson-index"],
    }, ensure_ascii=False, indent=2))
```

但注意：课程发布前的副作用必须绕过缓存：

```text
发 Telegram 前：实时读取 lesson 文件
push 前：实时 git status / gh pr list / git diff --check
更新 TOOLS.md 前：实时 grep 已讲内容，避免重复
```

也就是：

- 选题、目录展示、草稿生成：可以用缓存；
- 发群、提交、推送、写长期记忆：必须 live check。

这正好呼应前面几课：缓存是性能层，不是事实豁免权。

---

## 6. 常见坑

### 坑 1：只删 L2，不删 L1

Redis 里已经删了，worker 内存还在命中旧值。解决：失效事件广播 + L1 短 TTL + 版本化 key。

### 坑 2：L1 缓存了权限结果

权限、审批、session control 这类结果变化很敏感。可以缓存“策略定义”，不要长期缓存“某次授权通过”。

### 坑 3：大对象塞进 L1

L1 应该存小而热的数据。PDF 原文、网页全文、批量订单列表应该存 pointer 或 summary，不要塞进进程内存。

### 坑 4：多级缓存没有统一指标

至少记录：

```text
l1_hit / l2_hit / miss / load_ms / refill_l1 / invalidate_count / stale_blocked
```

否则你不知道优化到底来自哪一层，也不知道 stale bug 出在哪一层。

---

## 7. 设计检查清单

做 Agent 多级缓存时，问 8 个问题：

1. 这个 key 是否包含 tenant、scope、schemaVersion？
2. L1 TTL 是否明显短于 L2？
3. 敏感/高风险数据是否禁止进入 L1？
4. 外部写入后是否有 tag invalidation？
5. 多 worker 是否能收到失效广播？
6. L2 miss 回源是否有 singleflight 防击穿？
7. L1/L2 命中率是否分别可观测？
8. 副作用执行前是否强制 live revalidate？

记住一句话：

> L1 让 Agent 快，L2 让 Agent 稳，L3 让 Agent 对；三层之间靠版本、标签和复核保持诚实。
