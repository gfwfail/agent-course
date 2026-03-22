# 112 - Agent 跨进程状态广播（Cross-Process State Broadcast with Pub/Sub）

> 多实例 Agent 如何实时感知彼此的状态变化？Redis Pub/Sub + 事件广播模式让每个实例都保持同步。

---

## 问题背景

生产环境中 Agent 往往多实例部署（水平扩容、多 Worker），但状态是分散的：

```
User A → Instance-1 (处理中...)
User B → Instance-2 (空闲)
User C → Instance-3 (空闲)

Instance-1 触发了一个"全局配置更新"
→ Instance-2 和 Instance-3 完全不知道！
```

**常见场景：**
- 管理员更新了 Tool 白名单 → 所有实例立刻生效
- 某个 Agent 完成了共享任务 → 通知等待中的其他 Agent
- 限流计数器溢出 → 广播给所有实例暂停
- 会话状态变更 → 让同一用户的不同请求感知到

---

## 解决方案：Redis Pub/Sub 广播模式

```
┌─────────────┐     publish      ┌────────────┐     subscribe    ┌─────────────┐
│  Instance-1 │ ───────────────► │  Redis     │ ───────────────► │  Instance-2 │
│  (发布者)   │                  │  Pub/Sub   │                  │  (订阅者)   │
└─────────────┘                  └────────────┘                  └─────────────┘
                                       │
                                       └──────────────────────────► Instance-3
```

---

## 核心实现

### 1. 事件类型定义

```typescript
// types/broadcast.ts

export type BroadcastEventType =
  | 'config.updated'      // 配置热更新
  | 'tool.whitelist.changed' // 工具白名单变化
  | 'ratelimit.exceeded'  // 限流触发
  | 'session.terminated'  // 会话强制终止
  | 'agent.task.completed' // 任务完成通知
  | 'agent.task.failed';  // 任务失败通知

export interface BroadcastEvent<T = unknown> {
  id: string;              // 唯一事件 ID（幂等用）
  type: BroadcastEventType;
  source: string;          // 发布实例 ID
  timestamp: number;
  payload: T;
}

export interface ConfigUpdatedPayload {
  key: string;
  oldValue: unknown;
  newValue: unknown;
  updatedBy: string;
}

export interface TaskCompletedPayload {
  taskId: string;
  result: unknown;
  agentId: string;
}
```

### 2. 广播发布器（Publisher）

```typescript
// broadcast/publisher.ts
import { createClient } from 'redis';
import { randomUUID } from 'crypto';
import type { BroadcastEvent, BroadcastEventType } from '../types/broadcast';

const CHANNEL = 'agent:broadcast';

export class BroadcastPublisher {
  private client: ReturnType<typeof createClient>;
  private instanceId: string;

  constructor(redisUrl: string) {
    this.client = createClient({ url: redisUrl });
    this.instanceId = process.env.INSTANCE_ID ?? randomUUID();
  }

  async connect() {
    await this.client.connect();
    console.log(`[Publisher] Instance ${this.instanceId} connected`);
  }

  async publish<T>(type: BroadcastEventType, payload: T): Promise<void> {
    const event: BroadcastEvent<T> = {
      id: randomUUID(),
      type,
      source: this.instanceId,
      timestamp: Date.now(),
      payload,
    };

    const message = JSON.stringify(event);
    const receivers = await this.client.publish(CHANNEL, message);
    
    console.log(`[Publisher] Broadcast "${type}" → ${receivers} instances`);
  }

  async disconnect() {
    await this.client.disconnect();
  }
}
```

### 3. 广播订阅器（Subscriber）

```typescript
// broadcast/subscriber.ts
import { createClient } from 'redis';
import type { BroadcastEvent, BroadcastEventType } from '../types/broadcast';

type EventHandler<T = unknown> = (event: BroadcastEvent<T>) => Promise<void>;

const CHANNEL = 'agent:broadcast';

export class BroadcastSubscriber {
  private client: ReturnType<typeof createClient>;
  private instanceId: string;
  private handlers = new Map<BroadcastEventType, EventHandler[]>();
  private seenIds = new Set<string>(); // 幂等去重

  constructor(redisUrl: string, instanceId: string) {
    this.client = createClient({ url: redisUrl });
    this.instanceId = instanceId;
  }

  // 注册事件处理器
  on<T>(type: BroadcastEventType, handler: EventHandler<T>): void {
    const existing = this.handlers.get(type) ?? [];
    existing.push(handler as EventHandler);
    this.handlers.set(type, existing);
  }

  async connect(): Promise<void> {
    await this.client.connect();

    await this.client.subscribe(CHANNEL, async (message) => {
      try {
        const event: BroadcastEvent = JSON.parse(message);

        // 忽略自己发的消息
        if (event.source === this.instanceId) return;

        // 幂等去重（防止重复消费）
        if (this.seenIds.has(event.id)) {
          console.log(`[Subscriber] Duplicate event ${event.id}, skip`);
          return;
        }
        this.seenIds.add(event.id);
        // 避免 Set 无限增长：保留最近 1000 个
        if (this.seenIds.size > 1000) {
          const first = this.seenIds.values().next().value;
          this.seenIds.delete(first);
        }

        const handlers = this.handlers.get(event.type as BroadcastEventType);
        if (!handlers?.length) return;

        console.log(`[Subscriber] Received "${event.type}" from ${event.source}`);
        await Promise.allSettled(handlers.map((h) => h(event)));
      } catch (err) {
        console.error('[Subscriber] Failed to process broadcast:', err);
      }
    });

    console.log(`[Subscriber] Instance ${this.instanceId} listening on ${CHANNEL}`);
  }

  async disconnect() {
    await this.client.unsubscribe(CHANNEL);
    await this.client.disconnect();
  }
}
```

---

## 4. Agent 集成：配置热更新实战

```typescript
// agent/config-aware-agent.ts
import { BroadcastSubscriber } from '../broadcast/subscriber';
import { BroadcastPublisher } from '../broadcast/publisher';
import type { ConfigUpdatedPayload } from '../types/broadcast';

interface AgentConfig {
  model: string;
  maxTokens: number;
  allowedTools: string[];
  systemPrompt: string;
}

export class ConfigAwareAgent {
  private config: AgentConfig;
  private subscriber: BroadcastSubscriber;
  private publisher: BroadcastPublisher;

  constructor(
    private instanceId: string,
    private redisUrl: string,
    initialConfig: AgentConfig
  ) {
    this.config = { ...initialConfig };
    this.subscriber = new BroadcastSubscriber(redisUrl, instanceId);
    this.publisher = new BroadcastPublisher(redisUrl);
  }

  async initialize(): Promise<void> {
    await this.subscriber.connect();
    await this.publisher.connect();

    // 订阅配置更新事件
    this.subscriber.on<ConfigUpdatedPayload>(
      'config.updated',
      async (event) => {
        const { key, newValue } = event.payload;

        // 动态更新本地配置
        if (key in this.config) {
          const old = (this.config as Record<string, unknown>)[key];
          (this.config as Record<string, unknown>)[key] = newValue;
          console.log(
            `[Agent ${this.instanceId}] Config "${key}" updated: ${old} → ${newValue}`
          );
        }
      }
    );

    // 订阅会话终止事件
    this.subscriber.on<{ sessionId: string; reason: string }>(
      'session.terminated',
      async (event) => {
        console.log(
          `[Agent ${this.instanceId}] Session ${event.payload.sessionId} terminated: ${event.payload.reason}`
        );
        // 清理本地会话缓存、取消进行中的任务...
      }
    );
  }

  // 更新配置并广播给所有实例
  async updateConfig<K extends keyof AgentConfig>(
    key: K,
    value: AgentConfig[K],
    updatedBy: string
  ): Promise<void> {
    const oldValue = this.config[key];
    this.config[key] = value; // 先更新自己

    // 广播给其他实例
    await this.publisher.publish<ConfigUpdatedPayload>('config.updated', {
      key,
      oldValue,
      newValue: value,
      updatedBy,
    });
  }

  getConfig(): AgentConfig {
    return { ...this.config };
  }
}
```

---

## 5. 任务完成通知：Agent 间协调

```typescript
// agent/collaborative-agent.ts
// 场景：任务 A 完成后，等待它结果的 Agent B 立即被唤醒

export class WaitingTaskManager {
  // taskId → resolve 函数
  private pendingTasks = new Map<string, (result: unknown) => void>();

  constructor(private subscriber: BroadcastSubscriber) {
    this.subscriber.on<{ taskId: string; result: unknown }>(
      'agent.task.completed',
      async (event) => {
        const { taskId, result } = event.payload;
        const resolve = this.pendingTasks.get(taskId);
        if (resolve) {
          console.log(`[Coordinator] Task ${taskId} resolved by remote agent`);
          resolve(result);
          this.pendingTasks.delete(taskId);
        }
      }
    );
  }

  // 等待某个任务完成（跨进程）
  waitForTask(taskId: string, timeoutMs = 30_000): Promise<unknown> {
    return new Promise((resolve, reject) => {
      this.pendingTasks.set(taskId, resolve);

      setTimeout(() => {
        if (this.pendingTasks.has(taskId)) {
          this.pendingTasks.delete(taskId);
          reject(new Error(`Task ${taskId} timed out after ${timeoutMs}ms`));
        }
      }, timeoutMs);
    });
  }
}
```

---

## 6. OpenClaw 实战：全局限流广播

类比 OpenClaw 中多 Session 共享速率限制的场景：

```typescript
// broadcast/ratelimit-broadcast.ts

export class GlobalRateLimitBroadcaster {
  constructor(
    private pub: BroadcastPublisher,
    private sub: BroadcastSubscriber,
    private localLimiter: { pause: () => void; resume: () => void }
  ) {
    // 收到广播 → 暂停本地限流器
    this.sub.on<{ resumeAfterMs: number }>(
      'ratelimit.exceeded',
      async (event) => {
        console.log(
          `[RateLimit] Remote instance hit limit. Pausing for ${event.payload.resumeAfterMs}ms`
        );
        this.localLimiter.pause();
        setTimeout(() => this.localLimiter.resume(), event.payload.resumeAfterMs);
      }
    );
  }

  // 本地触发限流时广播
  async broadcastExceeded(resumeAfterMs: number): Promise<void> {
    this.localLimiter.pause();
    await this.pub.publish('ratelimit.exceeded', { resumeAfterMs });
    setTimeout(() => this.localLimiter.resume(), resumeAfterMs);
  }
}
```

---

## 关键设计原则

| 原则 | 做法 |
|------|------|
| **忽略自发事件** | `event.source === instanceId` → skip |
| **幂等消费** | 缓存已处理的 event.id，去重 |
| **广播 ≠ 指令** | 接收方自行决定是否响应，不强制 |
| **发布即忘（Fire-and-forget）** | 不期待 ACK，容忍部分丢失 |
| **订阅者轻量** | 处理器快速返回，重活扔进异步队列 |

---

## Pub/Sub vs 其他方案对比

| 方案 | 延迟 | 持久化 | 有序性 | 适用场景 |
|------|------|--------|--------|---------|
| Redis Pub/Sub | ~1ms | ❌ | ❌ | 实时状态广播 |
| Redis Streams | ~5ms | ✅ | ✅ | 可靠事件流 |
| Kafka | ~10ms | ✅ | ✅ | 高吞吐审计日志 |
| WebSocket | ~2ms | ❌ | ❌ | 前端实时推送 |

> **结论：** 状态广播用 Redis Pub/Sub 够了；需要持久化 + 重放 → 用第 37 课的 Event Sourcing 方案。

---

## 总结

```
Pub/Sub 广播模式三要素：

1. Publisher  →  发布事件到 Channel
2. Channel    →  Redis 作为消息总线
3. Subscriber →  过滤 + 幂等 + 处理

适合：配置热更新、任务完成通知、全局限流广播
不适合：需要 ACK 确认、严格有序、事件持久化的场景
```

多实例 Agent 协调的关键：**不共享内存，只共享事件**。每个实例都是自治的，通过广播保持最终一致。
