# 106 - Agent 分布式锁与并发控制（Distributed Locking & Concurrency Control）

> 多个 Agent 实例同时跑，谁来保证"只有一个人能改这个东西"？

---

## 问题背景

水平扩容后，你有 N 个 Agent 实例同时运行：

```
User A → Agent Instance 1 ─┐
User B → Agent Instance 2 ─┼─→ 同一个用户账户 / 同一个文件 / 同一个 DB 行
User C → Agent Instance 3 ─┘
```

没有并发控制的后果：
- **余额被双扣**（两个 Agent 同时读到余额 100，各自扣 80，最终余额 -60）
- **文件被覆盖**（两个 Agent 同时 read-modify-write，后写的覆盖前写的）
- **任务被重复执行**（定时任务多个实例同时触发）

---

## 核心概念

### 1. 互斥锁 vs 读写锁

| 锁类型 | 场景 | 并发度 |
|--------|------|--------|
| 互斥锁（Mutex）| 读写都独占 | 最低 |
| 读写锁（RW Lock）| 多读单写 | 中等 |
| 乐观锁（Optimistic）| 冲突少时高效 | 最高 |

### 2. 分布式锁的三要素

```
1. 互斥性：同一时刻只有一个持有者
2. 安全释放：只有加锁者能释放（防止误删他人的锁）
3. 自动过期：持有者崩溃后锁能自动释放（防死锁）
```

---

## 实现方案

### 方案一：Redis SET NX PX（最常用）

```typescript
// pi-mono 风格：Redis 分布式锁
class RedisDistributedLock {
  private redis: Redis;
  private lockPrefix = 'agent:lock:';

  async acquire(
    resource: string,
    ttlMs: number = 30_000
  ): Promise<string | null> {
    const lockKey = this.lockPrefix + resource;
    // value = 随机 token，确保只有加锁者能释放
    const token = crypto.randomUUID();

    // SET key value NX PX ttl
    // NX = 不存在时才设置（原子性互斥）
    const ok = await this.redis.set(lockKey, token, 'NX', 'PX', ttlMs);
    return ok === 'OK' ? token : null;
  }

  async release(resource: string, token: string): Promise<boolean> {
    const lockKey = this.lockPrefix + resource;
    
    // Lua 脚本保证"检查 + 删除"原子性
    const luaScript = `
      if redis.call("GET", KEYS[1]) == ARGV[1] then
        return redis.call("DEL", KEYS[1])
      else
        return 0
      end
    `;
    const result = await this.redis.eval(luaScript, 1, lockKey, token);
    return result === 1;
  }

  // 带自动重试的锁获取
  async acquireWithRetry(
    resource: string,
    opts: { ttlMs?: number; retryMs?: number; maxWaitMs?: number } = {}
  ): Promise<string> {
    const { ttlMs = 30_000, retryMs = 100, maxWaitMs = 5_000 } = opts;
    const deadline = Date.now() + maxWaitMs;

    while (Date.now() < deadline) {
      const token = await this.acquire(resource, ttlMs);
      if (token) return token;
      
      // 指数退避 + 抖动，避免惊群
      const jitter = Math.random() * 50;
      await sleep(retryMs + jitter);
    }
    
    throw new Error(`Failed to acquire lock for "${resource}" within ${maxWaitMs}ms`);
  }
}
```

### 方案二：装饰器模式（让工具自动加锁）

```typescript
// 自动加锁的工具包装器
function withLock(
  lockFn: (args: any) => string,  // 从参数中提取锁 key
  opts?: { ttlMs?: number }
) {
  return function decorator(
    target: any,
    methodName: string,
    descriptor: PropertyDescriptor
  ) {
    const original = descriptor.value;
    
    descriptor.value = async function(...args: any[]) {
      const lock = (this as any).lock as RedisDistributedLock;
      const resource = lockFn(args[0]);
      
      const token = await lock.acquireWithRetry(resource, opts);
      try {
        return await original.apply(this, args);
      } finally {
        await lock.release(resource, token);
      }
    };
    return descriptor;
  };
}

// 使用示例
class WalletTool {
  @withLock((args) => `wallet:${args.userId}`, { ttlMs: 10_000 })
  async deductBalance(args: { userId: string; amount: number }) {
    // 这里保证同一用户只有一个实例在执行
    const balance = await db.getBalance(args.userId);
    if (balance < args.amount) throw new Error('Insufficient balance');
    await db.setBalance(args.userId, balance - args.amount);
    return { newBalance: balance - args.amount };
  }
}
```

### 方案三：乐观锁（高并发读多写少场景）

```typescript
// 版本号 CAS（Compare-And-Swap）
class OptimisticLockTool {
  async updateUserProfile(args: {
    userId: string;
    patch: Record<string, any>;
  }) {
    const MAX_RETRIES = 3;
    
    for (let attempt = 0; attempt < MAX_RETRIES; attempt++) {
      // 1. 读取当前版本
      const { data, version } = await db.getUserWithVersion(args.userId);
      
      // 2. 本地计算新值
      const newData = { ...data, ...args.patch };
      
      // 3. 带版本号的条件写入
      const updated = await db.updateIfVersion(
        args.userId,
        newData,
        version  // "只有版本还是 X 时才更新"
      );
      
      if (updated) {
        return { success: true, data: newData };
      }
      
      // 版本冲突，稍等重试
      console.log(`CAS conflict on attempt ${attempt + 1}, retrying...`);
      await sleep(Math.pow(2, attempt) * 10);  // 指数退避
    }
    
    throw new Error('Optimistic lock failed after max retries');
  }
}
```

---

## OpenClaw 实战：Cron 单实例锁

OpenClaw 的定时任务可能被多个实例触发，用分布式锁保证幂等：

```typescript
// OpenClaw cron handler with leader election
class AgentCronHandler {
  async handleScheduledTask(taskId: string, fn: () => Promise<void>) {
    const lockKey = `cron:${taskId}`;
    const lock = getRedisLock();
    
    // 尝试成为"领导者"执行任务
    let token: string | null = null;
    try {
      token = await lock.acquire(lockKey, 60_000);  // 60s TTL
      if (!token) {
        // 其他实例已经在执行，跳过
        console.log(`Task ${taskId}: another instance is running, skipping`);
        return;
      }
      
      console.log(`Task ${taskId}: acquired lock, executing...`);
      await fn();
      
    } finally {
      if (token) {
        await lock.release(lockKey, token);
      }
    }
  }
}

// 使用
cronHandler.handleScheduledTask('daily-report', async () => {
  await generateAndSendDailyReport();
});
```

---

## learn-claude-code 风格：Context 中的锁状态

```python
# Python 实现：Agent 工具级别的细粒度锁
import asyncio
import uuid
import time
from contextlib import asynccontextmanager

class AgentLockManager:
    def __init__(self, redis_client):
        self.redis = redis_client
        self._local_locks: dict[str, asyncio.Lock] = {}
    
    @asynccontextmanager
    async def lock(self, resource: str, ttl_seconds: int = 30):
        """用 async context manager 管理锁生命周期"""
        token = str(uuid.uuid4())
        lock_key = f"agent:lock:{resource}"
        
        # 先获取分布式锁
        acquired = await self.redis.set(
            lock_key, token, nx=True, ex=ttl_seconds
        )
        if not acquired:
            raise LockAcquisitionError(f"Resource '{resource}' is locked")
        
        try:
            yield token
        finally:
            # Lua 脚本原子释放
            script = """
            if redis.call('get', KEYS[1]) == ARGV[1] then
                return redis.call('del', KEYS[1])
            else
                return 0
            end
            """
            await self.redis.eval(script, 1, lock_key, token)
    
    @asynccontextmanager
    async def lock_with_retry(self, resource: str, max_wait: float = 5.0, **kwargs):
        """带重试的锁获取"""
        deadline = time.monotonic() + max_wait
        while True:
            try:
                async with self.lock(resource, **kwargs) as token:
                    yield token
                    return
            except LockAcquisitionError:
                if time.monotonic() >= deadline:
                    raise
                await asyncio.sleep(0.1 + (uuid.uuid4().int % 50) / 1000)  # jitter

# Tool 函数中使用
async def transfer_funds(args: dict, ctx: AgentContext):
    """转账工具 - 必须锁定双方账户"""
    from_id = args['from_user_id']
    to_id = args['to_user_id']
    amount = args['amount']
    
    # 按 ID 排序加锁，防止死锁（A→B 和 B→A 同时发生时）
    lock_order = sorted([from_id, to_id])
    
    lock_mgr = ctx.get_service('lock_manager')
    async with lock_mgr.lock(f"wallet:{lock_order[0]}"):
        async with lock_mgr.lock(f"wallet:{lock_order[1]}"):
            from_balance = await get_balance(from_id)
            if from_balance < amount:
                return {"error": "insufficient_funds"}
            
            await set_balance(from_id, from_balance - amount)
            to_balance = await get_balance(to_id)
            await set_balance(to_id, to_balance + amount)
            
            return {"success": True, "transferred": amount}
```

---

## 常见陷阱

### ❌ 锁 TTL 设置不当

```typescript
// 错误：TTL 太短，任务未完成锁就过期了
const token = await lock.acquire('task', 1000);  // 1秒
await longRunningTask();  // 如果超过1秒，锁已被释放，另一个实例可以进来！

// 正确：使用看门狗（Watchdog）自动续期
async function acquireWithWatchdog(resource: string) {
  const token = await lock.acquire(resource, 30_000);
  
  // 启动看门狗，每 10 秒续期一次
  const watchdog = setInterval(async () => {
    await lock.extend(resource, token, 30_000);
  }, 10_000);
  
  return {
    token,
    release: async () => {
      clearInterval(watchdog);
      await lock.release(resource, token);
    }
  };
}
```

### ❌ 忘记按顺序加锁导致死锁

```typescript
// 危险：两个 Agent 互相等待
// Agent A: lock('user:1') → lock('user:2')
// Agent B: lock('user:2') → lock('user:1')  ← 死锁！

// 安全：始终按相同顺序加锁
function getLockOrder(ids: string[]): string[] {
  return [...ids].sort();  // 字典序排列
}
```

### ❌ 在锁内部调用 LLM（持锁时间太长）

```typescript
// 错误：LLM 调用可能很慢，锁一直占着
async function badPattern(resource: string) {
  const token = await lock.acquire(resource, 30_000);
  const llmResult = await callLLM('analyze this...');  // ← 可能需要 10s+！
  await updateDB(llmResult);
  await lock.release(resource, token);
}

// 正确：先 LLM，后加锁写入
async function goodPattern(resource: string) {
  // 1. 锁外做耗时计算
  const llmResult = await callLLM('analyze this...');
  
  // 2. 锁内只做快速写入
  const token = await lock.acquire(resource, 5_000);  // 短 TTL 足矣
  try {
    await updateDB(llmResult);
  } finally {
    await lock.release(resource, token);
  }
}
```

---

## 选型速查

| 场景 | 推荐方案 |
|------|----------|
| 余额扣减、库存扣减 | Redis 互斥锁 + 短 TTL |
| 用户配置更新 | 乐观锁（版本号 CAS）|
| Cron 单实例执行 | Redis 互斥锁（锁 = 领导者选举）|
| 文件生成（幂等） | 幂等 key + 乐观锁 |
| 高并发读、偶尔写 | 读写锁 |
| 跨服务事务 | Saga Pattern（见第55课）|

---

## 小结

```
分布式锁三要素：互斥 + 安全释放 + 自动过期
实现：Redis SET NX PX + Lua 原子释放
进阶：乐观锁（冲突少时更高效）
避坑：
  - 锁 TTL 要大于任务时长，或用看门狗续期
  - 多资源加锁按固定顺序防死锁
  - 锁内不要做耗时操作（LLM 调用在锁外）
OpenClaw 应用：Cron 任务幂等保障 = 单实例分布式锁
```

---

**下一课预告**：Agent 链路中的数据血缘追踪（Data Lineage Tracking）——知道每个工具输出从哪来，去哪了。
