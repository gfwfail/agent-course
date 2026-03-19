# 86 - Agent 上下文外部化与跨会话共享（Context Externalization & Cross-Session Sharing）

> **核心问题**：Agent 的 context 默认存在内存里，重启即丢失，多实例无法共享。如何让 Agent 的"记忆"跨越进程边界？

---

## 为什么需要上下文外部化？

```
单实例 Agent（默认）:
  Process A → [context in memory] → 重启后 LOST ❌

外部化后:
  Process A ──┐
              ├──→ [Redis / DB] ←──→ Process B
  Process C ──┘                       同一份 context ✅
```

**三大场景**：
1. **水平扩容**：多个 Agent 实例处理不同用户，共享工具调用状态
2. **崩溃恢复**：进程重启后接续上次中断的任务
3. **跨会话记忆**：用户换设备，Agent 认识你

---

## 核心架构：Context Store 抽象层

```python
# learn-claude-code 风格：用抽象接口隔离存储细节

from abc import ABC, abstractmethod
from typing import Optional
import json, time

class ContextStore(ABC):
    """上下文存储抽象接口"""
    
    @abstractmethod
    async def save(self, session_id: str, context: dict, ttl: int = 3600) -> None:
        """保存 context，ttl 秒后过期"""
        pass
    
    @abstractmethod
    async def load(self, session_id: str) -> Optional[dict]:
        """加载 context，不存在返回 None"""
        pass
    
    @abstractmethod
    async def append_message(self, session_id: str, message: dict) -> None:
        """追加单条消息（避免全量读写）"""
        pass
    
    @abstractmethod
    async def delete(self, session_id: str) -> None:
        pass


class InMemoryContextStore(ContextStore):
    """本地开发用，单进程"""
    
    def __init__(self):
        self._store: dict[str, dict] = {}
    
    async def save(self, session_id: str, context: dict, ttl: int = 3600):
        self._store[session_id] = {
            **context,
            "_expires_at": time.time() + ttl
        }
    
    async def load(self, session_id: str) -> Optional[dict]:
        entry = self._store.get(session_id)
        if entry and entry["_expires_at"] > time.time():
            return {k: v for k, v in entry.items() if not k.startswith("_")}
        return None
    
    async def append_message(self, session_id: str, message: dict):
        ctx = await self.load(session_id) or {"messages": []}
        ctx["messages"].append(message)
        await self.save(session_id, ctx)
    
    async def delete(self, session_id: str):
        self._store.pop(session_id, None)
```

---

## Redis 实现：生产级方案

```python
import redis.asyncio as aioredis
import json, time

class RedisContextStore(ContextStore):
    """生产用 Redis 实现"""
    
    def __init__(self, url: str = "redis://localhost:6379"):
        self.redis = aioredis.from_url(url, decode_responses=True)
    
    def _key(self, session_id: str) -> str:
        return f"agent:ctx:{session_id}"
    
    def _msg_key(self, session_id: str) -> str:
        return f"agent:msgs:{session_id}"
    
    async def save(self, session_id: str, context: dict, ttl: int = 3600):
        # 元数据用 Hash 存（方便单字段更新）
        meta = {k: json.dumps(v) for k, v in context.items() 
                if k != "messages"}
        
        pipe = self.redis.pipeline()
        if meta:
            pipe.hset(self._key(session_id), mapping=meta)
            pipe.expire(self._key(session_id), ttl)
        await pipe.execute()
    
    async def load(self, session_id: str) -> Optional[dict]:
        pipe = self.redis.pipeline()
        pipe.hgetall(self._key(session_id))
        pipe.lrange(self._msg_key(session_id), 0, -1)
        meta_raw, msgs_raw = await pipe.execute()
        
        if not meta_raw and not msgs_raw:
            return None
        
        context = {k: json.loads(v) for k, v in meta_raw.items()}
        context["messages"] = [json.loads(m) for m in msgs_raw]
        return context
    
    async def append_message(self, session_id: str, message: dict):
        """高效追加：只写新消息，不全量读写"""
        pipe = self.redis.pipeline()
        pipe.rpush(self._msg_key(session_id), json.dumps(message))
        pipe.expire(self._msg_key(session_id), 3600)
        await pipe.execute()
    
    async def delete(self, session_id: str):
        await self.redis.delete(
            self._key(session_id), 
            self._msg_key(session_id)
        )
```

**关键设计**：消息列表用 Redis List（`RPUSH`），元数据用 Hash。`append_message` 是 O(1) 操作，不需要全量读写。

---

## 集成到 Agent Loop

```python
class ExternalizedAgent:
    """带外部化 context 的 Agent"""
    
    def __init__(self, store: ContextStore, session_id: str):
        self.store = store
        self.session_id = session_id
        self._context: Optional[dict] = None
    
    async def _ensure_loaded(self):
        """懒加载：首次使用才读 Redis"""
        if self._context is None:
            self._context = await self.store.load(self.session_id)
            if self._context is None:
                self._context = {
                    "messages": [],
                    "created_at": time.time(),
                    "metadata": {}
                }
    
    async def chat(self, user_message: str) -> str:
        await self._ensure_loaded()
        
        # 追加用户消息（立即持久化，防崩溃丢失）
        user_msg = {"role": "user", "content": user_message}
        self._context["messages"].append(user_msg)
        await self.store.append_message(self.session_id, user_msg)
        
        # 调用 LLM
        response = await self._call_llm(self._context["messages"])
        
        # 追加 AI 回复
        ai_msg = {"role": "assistant", "content": response}
        self._context["messages"].append(ai_msg)
        await self.store.append_message(self.session_id, ai_msg)
        
        return response
    
    async def update_metadata(self, key: str, value):
        """更新单个元数据字段"""
        await self._ensure_loaded()
        self._context["metadata"][key] = value
        # 只保存 metadata，不重写整个 context
        await self.store.save(self.session_id, {
            "metadata": self._context["metadata"]
        })
    
    async def _call_llm(self, messages: list) -> str:
        # 实际调用 Claude/GPT...
        pass
```

---

## OpenClaw 中的实际应用

OpenClaw 的 `sessions` 系统就是这个模式的工业级实现：

```
openclaw/
  src/
    sessions/
      SessionManager.ts     ← 管理所有 session
      SessionStore.ts       ← 抽象存储层
      stores/
        MemoryStore.ts      ← 开发用
        SqliteStore.ts      ← 本地持久化
      Session.ts            ← 单个 session，含 context
```

```typescript
// OpenClaw SessionStore 核心接口（简化）
interface SessionStore {
  // 保存完整 session 状态
  save(session: Session): Promise<void>;
  
  // 加载，用于恢复中断任务
  load(sessionKey: string): Promise<Session | null>;
  
  // 追加消息（高频操作，需要高效）
  appendMessage(sessionKey: string, msg: Message): Promise<void>;
  
  // 列举活跃 session（用于监控）
  list(filter?: SessionFilter): Promise<SessionSummary[]>;
}
```

**SQLite 实现细节**：
- 消息单独一张表（`messages`），按 `session_key` + `seq` 索引
- Session 元数据一张表（`sessions`）
- 追加消息只做 INSERT，不做 UPDATE，天然支持并发写

---

## 分布式场景：乐观锁防冲突

```python
class OptimisticContextStore(RedisContextStore):
    """多实例写同一 session 时用乐观锁"""
    
    async def save_with_cas(
        self, 
        session_id: str, 
        context: dict, 
        expected_version: int
    ) -> bool:
        """Check-And-Set：版本不匹配则拒绝写入"""
        
        key = self._key(session_id)
        
        async with self.redis.pipeline() as pipe:
            while True:
                try:
                    # WATCH：如果 key 被其他进程修改，EXEC 会失败
                    await pipe.watch(key)
                    
                    current = await pipe.hget(key, "version")
                    current_version = int(current or 0)
                    
                    if current_version != expected_version:
                        await pipe.unwatch()
                        return False  # 版本冲突，让调用方重试
                    
                    pipe.multi()
                    pipe.hset(key, "version", expected_version + 1)
                    pipe.hset(key, "data", json.dumps(context))
                    await pipe.execute()
                    return True
                    
                except aioredis.WatchError:
                    # 被其他进程抢先修改，重试
                    continue
```

**适用场景**：多个 Agent 实例协作同一个任务时，防止互相覆盖状态。

---

## Context 压缩：防止无限增长

```python
class CompressingContextStore(RedisContextStore):
    """超过阈值自动压缩历史消息"""
    
    MAX_MESSAGES = 100      # 保留最近 N 条
    COMPRESS_KEEP = 20      # 压缩后保留最新 M 条
    
    async def append_message(self, session_id: str, message: dict):
        await super().append_message(session_id, message)
        
        # 检查是否需要压缩
        count = await self.redis.llen(self._msg_key(session_id))
        if count > self.MAX_MESSAGES:
            await self._compress(session_id)
    
    async def _compress(self, session_id: str):
        """保留最新 N 条 + 用 LLM 生成摘要替换旧消息"""
        msg_key = self._msg_key(session_id)
        
        # 取出所有消息
        all_msgs = await self.redis.lrange(msg_key, 0, -1)
        old_msgs = [json.loads(m) for m in all_msgs[:-self.COMPRESS_KEEP]]
        
        # 生成摘要
        summary = await self._summarize(old_msgs)
        summary_msg = {
            "role": "system",
            "content": f"[历史摘要] {summary}",
            "_is_summary": True
        }
        
        # 重写消息列表
        recent_msgs = [json.loads(m) for m in all_msgs[-self.COMPRESS_KEEP:]]
        new_msgs = [summary_msg] + recent_msgs
        
        pipe = self.redis.pipeline()
        pipe.delete(msg_key)
        for msg in new_msgs:
            pipe.rpush(msg_key, json.dumps(msg))
        await pipe.execute()
    
    async def _summarize(self, messages: list) -> str:
        # 调用 Claude 生成摘要
        pass
```

---

## 实战：跨会话用户记忆

```python
# pi-mono 风格：把用户档案作为 context 的一部分外部化

class UserAwareAgent:
    """跨会话记住用户偏好"""
    
    def __init__(self, store: ContextStore, user_id: str):
        self.store = store
        self.user_id = user_id
        # 用户维度的 key（跨所有 session）
        self.profile_key = f"user:{user_id}:profile"
    
    async def get_user_profile(self) -> dict:
        profile = await self.store.load(self.profile_key)
        return profile or {
            "preferences": {},
            "interaction_count": 0,
            "last_seen": None
        }
    
    async def update_profile(self, key: str, value):
        profile = await self.get_user_profile()
        profile[key] = value
        profile["last_seen"] = time.time()
        profile["interaction_count"] += 1
        await self.store.save(self.profile_key, profile, ttl=86400 * 30)
    
    async def chat(self, session_id: str, message: str) -> str:
        # 同时加载会话 context 和用户档案
        profile = await self.get_user_profile()
        
        # 把用户偏好注入 system prompt
        system = f"""用户偏好: {profile.get('preferences', {})}
历史交互次数: {profile['interaction_count']}"""
        
        # ... 正常 Agent Loop
        
        await self.update_profile("interaction_count", 
                                   profile["interaction_count"] + 1)
```

---

## 选型建议

| 场景 | 推荐方案 |
|------|---------|
| 本地开发 / 单进程 | `InMemoryContextStore` |
| 单机持久化 | SQLite（OpenClaw 默认） |
| 多实例 / 高并发 | Redis + List for messages |
| 需要查询历史 | PostgreSQL（JSONB 列） |
| 超大 context | S3 / R2 + Redis 缓存热数据 |

---

## 核心原则

1. **消息追加 > 全量读写**：用 Redis List 或 DB append-only 表
2. **懒加载**：首次使用才从 Redis 读，减少延迟
3. **TTL 必设**：防止僵尸 session 耗尽存储
4. **压缩要异步**：不阻塞主 Agent Loop
5. **版本号**：多实例写时防止数据覆盖

**本质**：把 Agent 的"大脑"从进程内存搬到网络可达的存储，Agent 从有状态服务变成无状态服务——可以像 HTTP 服务一样水平扩容。
