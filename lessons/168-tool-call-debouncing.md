# 168 - Agent 工具调用去抖动（Tool Call Debouncing）

> 消灭重复触发，让 Agent 更聪明地"等一等"

---

## 问题：Agent 狂按工具按钮

想象这个场景：用户在对话框里一边打字一边实时搜索，或者流式 LLM 输出触发多次监听回调——

```
用户输入 "查" → search("查")
用户输入 "查一" → search("查一")
用户输入 "查一下" → search("查一下")
用户输入 "查一下北京天气" → search("查一下北京天气")
```

**4 次 API 调用，只有最后一次有意义。**

这不只是浪费 token，更严重的问题是：
- 前 3 个 search 结果可能污染 LLM 上下文
- 并发请求导致乱序响应
- 第 3 个结果比第 4 个先回来 → LLM 用了过期数据

**去抖动（Debounce）= 等用户停了再说。**

---

## 核心概念：防抖 vs 节流 vs 去重

| 模式 | 行为 | 适用场景 |
|------|------|---------|
| **Debounce** | 连续触发时，只执行最后一次（等安静期） | 搜索/输入/流式回调 |
| **Throttle** | N 毫秒内最多执行一次（第一次立即执行） | 滚动事件/心跳/监控 |
| **Dedup** | 完全相同的请求合并为一个 | 幂等 API/缓存穿透 |

Debounce 和前两课（Lesson 96 去重、Lesson 151 限流）解决的是不同问题：

```
限流 = 服务端说"你调太快了，慢点"
去重 = 同时飞行的相同请求合并
去抖 = 客户端主动等，避免无意义的连续触发
```

---

## TypeScript 实现：工具调用去抖器

### 基础版：通用 Debounce

```typescript
// agent/debounce.ts

export class ToolDebouncer {
  private timers = new Map<string, NodeJS.Timeout>();
  private pending = new Map<string, {
    resolve: (value: any) => void;
    reject: (err: any) => void;
  }>();

  constructor(
    private readonly tool: (params: any) => Promise<any>,
    private readonly waitMs: number = 300
  ) {}

  /**
   * 对同一 key 的连续调用去抖动：
   * 等 waitMs 毫秒内没有新调用，才真正执行
   */
  call(key: string, params: any): Promise<any> {
    // 取消上一次等待
    const existingTimer = this.timers.get(key);
    if (existingTimer) {
      clearTimeout(existingTimer);
      // reject 上一个 pending 的 promise（已被取代）
      this.pending.get(key)?.reject(
        new DebounceCancelledError(`Superseded by newer call for key: ${key}`)
      );
    }

    return new Promise((resolve, reject) => {
      this.pending.set(key, { resolve, reject });

      const timer = setTimeout(async () => {
        this.timers.delete(key);
        this.pending.delete(key);
        try {
          const result = await this.tool(params);
          resolve(result);
        } catch (err) {
          reject(err);
        }
      }, this.waitMs);

      this.timers.set(key, timer);
    });
  }

  /** 立即取消所有待执行 */
  cancelAll() {
    for (const [key, timer] of this.timers) {
      clearTimeout(timer);
      this.pending.get(key)?.reject(new DebounceCancelledError('cancelAll'));
    }
    this.timers.clear();
    this.pending.clear();
  }
}

export class DebounceCancelledError extends Error {
  constructor(reason: string) {
    super(reason);
    this.name = 'DebounceCancelledError';
  }
}
```

### 进阶版：集成到 Agent 工具中间件

```typescript
// agent/debounce-middleware.ts

import { ToolDebouncer, DebounceCancelledError } from './debounce';

interface DebounceConfig {
  waitMs: number;
  keyFn?: (params: any) => string;  // 如何生成 debounce key
}

/**
 * 工具中间件：为指定工具添加去抖动
 */
export function withDebounce(
  toolFn: (params: any) => Promise<any>,
  config: DebounceConfig
): (params: any) => Promise<any> {
  const debouncer = new ToolDebouncer(toolFn, config.waitMs);
  const keyFn = config.keyFn ?? (() => 'default');

  return async (params: any) => {
    const key = keyFn(params);
    try {
      return await debouncer.call(key, params);
    } catch (err) {
      if (err instanceof DebounceCancelledError) {
        // 被新请求取代，静默忽略
        return null;
      }
      throw err;
    }
  };
}

// 使用示例
const debouncedSearch = withDebounce(
  async (params) => fetch(`/api/search?q=${params.query}`).then(r => r.json()),
  {
    waitMs: 400,
    keyFn: (params) => `search:${params.sessionId}`,  // 每个会话独立去抖
  }
);
```

### 实战：Agent Loop 中的流式输入去抖

```typescript
// agent/streaming-debounce.ts

import { Anthropic } from '@anthropic-ai/sdk';

/**
 * 场景：LLM 流式输出 → 触发实时工具调用
 * 问题：每 chunk 都触发一次，浪费 API
 * 方案：去抖动，等流稳定后再调
 */
export class StreamingToolDebouncer {
  private searchDebouncer: ToolDebouncer;
  private buffer = '';

  constructor(
    private readonly tools: Record<string, (p: any) => Promise<any>>
  ) {
    // 搜索类工具：400ms 去抖（用户输入停顿时间）
    this.searchDebouncer = new ToolDebouncer(
      tools.search,
      400
    );
  }

  async onStreamChunk(delta: string, sessionId: string) {
    this.buffer += delta;

    // 检测是否包含搜索意图的片段（简化示例）
    if (this.looksLikeSearchQuery(this.buffer)) {
      // 不立即调用！放进 debouncer
      try {
        const result = await this.searchDebouncer.call(
          `search:${sessionId}`,
          { query: this.buffer.trim() }
        );
        if (result) {
          console.log('[Debounced Search Result]', result);
          // 注入到 context...
        }
      } catch {
        // DebounceCancelled = 被更新的 chunk 取代，忽略
      }
    }
  }

  private looksLikeSearchQuery(text: string): boolean {
    return text.length > 5 && !text.endsWith('?');
  }
}
```

---

## Python 版：asyncio 实现

```python
# agent/debounce.py
import asyncio
from typing import Any, Callable, Coroutine, Optional
from dataclasses import dataclass

@dataclass
class PendingCall:
    future: asyncio.Future
    params: Any

class ToolDebouncer:
    def __init__(self, tool_fn: Callable, wait_s: float = 0.3):
        self.tool_fn = tool_fn
        self.wait_s = wait_s
        self._tasks: dict[str, asyncio.Task] = {}
        self._pending: dict[str, PendingCall] = {}

    async def call(self, key: str, params: Any) -> Any:
        loop = asyncio.get_event_loop()
        
        # 取消之前的等待任务
        if key in self._tasks:
            self._tasks[key].cancel()
            old_pending = self._pending.get(key)
            if old_pending and not old_pending.future.done():
                old_pending.future.cancel()

        future = loop.create_future()
        self._pending[key] = PendingCall(future=future, params=params)

        async def _delayed_exec():
            await asyncio.sleep(self.wait_s)
            try:
                result = await self.tool_fn(params)
                if not future.done():
                    future.set_result(result)
            except Exception as e:
                if not future.done():
                    future.set_exception(e)
            finally:
                self._tasks.pop(key, None)
                self._pending.pop(key, None)

        task = asyncio.create_task(_delayed_exec())
        self._tasks[key] = task

        try:
            return await future
        except asyncio.CancelledError:
            return None  # 被取代，静默返回

# 使用
async def search_tool(params):
    await asyncio.sleep(0.05)  # 模拟 API 调用
    return {"results": [f"Result for: {params['query']}"]}

debouncer = ToolDebouncer(search_tool, wait_s=0.4)

# 模拟用户快速输入
async def simulate_typing():
    for query in ["北", "北京", "北京天", "北京天气"]:
        result = asyncio.create_task(
            debouncer.call("user123", {"query": query})
        )
        await asyncio.sleep(0.1)  # 100ms 间隔，快于 400ms 去抖窗口
    
    # 等最后一次
    print(await result)  # 只有 "北京天气" 真正执行了
```

---

## OpenClaw 实战：Heartbeat 事件去抖

```typescript
// OpenClaw 中：多个事件快速触发同一个 Heartbeat 处理
// 去抖动防止在 30 秒内重复处理相同主题的 webhook

class HeartbeatDebouncer {
  private debouncer = new ToolDebouncer(
    async (params: { event: string; payload: any }) => {
      // 真正的处理逻辑
      return processHeartbeatEvent(params.event, params.payload);
    },
    500  // 500ms 安静期
  );

  async handle(event: string, payload: any) {
    // 同类型事件去抖：同一 event 类型 500ms 内只处理最后一个
    return this.debouncer.call(`event:${event}`, { event, payload });
  }
}

// 场景：GitHub webhook 短时间内推送多个 commit
// 只有最后一个 push 事件会触发 CI 流程
const hbDebouncer = new HeartbeatDebouncer();
webhook.on('push', (payload) => hbDebouncer.handle('github:push', payload));
```

---

## 何时用 Debounce，何时用 Throttle

```
用户实时搜索 (search-as-you-type)
→ Debounce ✓（等用户停止输入）

Agent 心跳检查
→ Throttle ✓（每 N 秒最多一次，第一次立即执行）

流式 LLM 触发工具
→ Debounce ✓（等流量稳定）

监控指标采集
→ Throttle ✓（固定频率，不跳过第一次）

并发同类请求
→ Dedup ✓（正在飞行的相同请求合并）

快速连续的 webhook
→ Debounce ✓（等一批都到了再处理最新的）
```

---

## 关键参数：等多久？

| 场景 | 推荐 waitMs |
|------|------------|
| 键盘输入 (搜索框) | 300–500ms |
| LLM 流式 chunk | 200–400ms |
| Webhook 事件 | 500–1000ms |
| 文件变更监听 | 100–300ms |
| 用户点击 (防双击) | 500–1000ms |

**经验法则**：waitMs = 预期"停顿时间" × 1.2

---

## 踩坑记录

### ❌ 坑 1：Debounce key 设计太宽

```typescript
// 错误：所有搜索共用一个 key → 用户 A 和用户 B 的搜索互相取消！
debouncer.call('search', params);

// 正确：按会话隔离
debouncer.call(`search:${sessionId}`, params);
```

### ❌ 坑 2：不处理 CancelledError → Promise 永远 pending

```typescript
// 错误
const result = await debouncer.call(key, params);

// 正确：总要处理取消
try {
  const result = await debouncer.call(key, params);
  if (result === null) return; // 被取代，跳过
} catch (err) {
  if (err instanceof DebounceCancelledError) return;
  throw err;
}
```

### ❌ 坑 3：忘记 cancelAll() → 进程退出时定时器泄漏

```typescript
// 进程退出时清理
process.on('SIGTERM', () => {
  globalDebouncer.cancelAll();
});
```

---

## 与其他课程的关系

```
Lesson 96  - Request Deduplication：飞行中的相同请求合并
Lesson 15  - Rate Limiting：服务端限速
Lesson 151 - Context-Aware Rate Limiting：动态限速
Lesson 168 - Tool Call Debouncing：客户端主动延迟，只执行最新触发 ← 本课
```

四课合在一起 = Agent 完整的"请求质量控制"体系。

---

## 总结

**Debounce = 聪明地等一等**

1. **为什么需要**：连续触发的工具调用大多是"过程状态"，只有最终状态有意义
2. **核心机制**：每次新调用取消上一个定时器，安静期结束后才真正执行
3. **Key 设计是关键**：按 session/user/event-type 隔离，避免跨用户干扰
4. **一定要处理取消**：Cancelled ≠ Error，是正常的"被取代"信号
5. **与 Throttle 区分**：Debounce 执行最后一次，Throttle 执行第一次

在交互式 Agent、流式输入处理、Webhook 批量到达等场景，Debounce 能把无效 API 调用减少 70-90%。
