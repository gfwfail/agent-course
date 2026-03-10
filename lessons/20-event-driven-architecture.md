# 第 20 课：Event-Driven Architecture 事件驱动架构

> 让 Agent 能够响应世界的变化，而不只是被动等待用户输入。

## 为什么需要事件驱动？

传统的 Agent 是 **请求-响应** 模式：用户问一句，Agent 答一句。但现实世界是动态的：

- 📧 新邮件到了
- 📅 日历提醒
- 🔔 GitHub PR 有新评论
- 💬 群里有人 @你
- ⏰ 定时任务触发
- 📱 设备状态变化

**事件驱动架构 (EDA)** 让 Agent 能够主动感知和响应这些变化。

## 核心概念

### 1. Event（事件）

事件是系统中发生的「事情」的记录：

```typescript
// pi-mono 风格的事件定义
interface AgentEvent {
  id: string;
  type: string;        // 事件类型
  source: string;      // 来源
  timestamp: number;
  payload: unknown;    // 事件数据
  metadata?: {
    priority?: 'low' | 'normal' | 'high' | 'urgent';
    ttl?: number;      // 过期时间
    idempotencyKey?: string;
  };
}

// 具体事件示例
interface MessageEvent extends AgentEvent {
  type: 'message.received';
  payload: {
    channelId: string;
    senderId: string;
    content: string;
    attachments?: Attachment[];
  };
}

interface ScheduleEvent extends AgentEvent {
  type: 'schedule.triggered';
  payload: {
    jobId: string;
    scheduledTime: number;
    context?: string;
  };
}
```

### 2. Event Source（事件源）

事件从哪里来？

```typescript
// OpenClaw 支持的事件源
type EventSource = 
  | 'telegram'      // Telegram 消息
  | 'discord'       // Discord 消息
  | 'cron'          // 定时任务
  | 'heartbeat'     // 心跳检查
  | 'webhook'       // 外部 Webhook
  | 'node'          // 设备节点
  | 'system';       // 系统事件

// 事件源接口
interface IEventSource {
  readonly name: string;
  
  // 订阅事件
  subscribe(handler: EventHandler): Unsubscribe;
  
  // 可选：发送确认
  ack?(eventId: string): Promise<void>;
  
  // 可选：发送失败
  nack?(eventId: string, error: Error): Promise<void>;
}
```

### 3. Event Handler（事件处理器）

处理事件的逻辑：

```typescript
type EventHandler = (event: AgentEvent) => Promise<EventResult>;

interface EventResult {
  handled: boolean;
  response?: string;
  actions?: Action[];
  error?: Error;
}
```

## 实战：OpenClaw 的事件系统

### 事件流程

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Telegram   │────▶│   Gateway   │────▶│   Session   │
│  Discord    │     │  (Router)   │     │  (Agent)    │
│  Webhook    │     └─────────────┘     └─────────────┘
│  Cron       │            │                   │
│  Heartbeat  │            ▼                   ▼
└─────────────┘     ┌─────────────┐     ┌─────────────┐
                    │  EventBus   │────▶│   Tools     │
                    │             │     │   Actions   │
                    └─────────────┘     └─────────────┘
```

### Gateway 作为事件路由器

```typescript
// OpenClaw Gateway 核心逻辑（简化）
class Gateway {
  private sessions: Map<string, Session> = new Map();
  private eventBus: EventBus;
  
  async handleEvent(event: AgentEvent): Promise<void> {
    // 1. 确定目标 session
    const sessionKey = this.resolveSession(event);
    
    // 2. 获取或创建 session
    let session = this.sessions.get(sessionKey);
    if (!session) {
      session = await this.createSession(sessionKey);
    }
    
    // 3. 根据事件类型路由
    switch (event.type) {
      case 'message.received':
        await session.handleMessage(event.payload);
        break;
        
      case 'schedule.triggered':
        await session.handleCron(event.payload);
        break;
        
      case 'heartbeat.poll':
        await session.handleHeartbeat();
        break;
        
      default:
        await session.handleGenericEvent(event);
    }
  }
  
  private resolveSession(event: AgentEvent): string {
    // Telegram 私聊 -> 用户专属 session
    // Telegram 群组 -> 群组共享 session  
    // Cron -> 主 session 或隔离 session
    // ...
  }
}
```

### Cron 事件源

```typescript
// OpenClaw 的 Cron 实现
interface CronJob {
  id: string;
  schedule: CronSchedule;
  payload: CronPayload;
  sessionTarget: 'main' | 'isolated';
  enabled: boolean;
}

type CronSchedule = 
  | { kind: 'at'; atMs: number }           // 一次性
  | { kind: 'every'; everyMs: number }     // 周期性
  | { kind: 'cron'; expr: string };        // Cron 表达式

type CronPayload =
  | { kind: 'systemEvent'; text: string }   // 注入到 main session
  | { kind: 'agentTurn'; message: string }; // 独立运行

// 触发逻辑
class CronScheduler {
  async tick(): Promise<void> {
    const now = Date.now();
    const dueJobs = this.jobs.filter(job => this.isDue(job, now));
    
    for (const job of dueJobs) {
      const event: ScheduleEvent = {
        id: randomUUID(),
        type: 'schedule.triggered',
        source: 'cron',
        timestamp: now,
        payload: {
          jobId: job.id,
          scheduledTime: now,
          ...job.payload
        }
      };
      
      await this.gateway.handleEvent(event);
    }
  }
}
```

### Heartbeat 事件

```typescript
// 心跳：定期检查是否有事情要做
class HeartbeatSource implements IEventSource {
  private intervalMs: number = 30 * 60 * 1000; // 30 分钟
  
  start(): void {
    setInterval(async () => {
      const event: AgentEvent = {
        id: randomUUID(),
        type: 'heartbeat.poll',
        source: 'heartbeat',
        timestamp: Date.now(),
        payload: {
          prompt: 'Read HEARTBEAT.md if it exists...'
        }
      };
      
      await this.handler(event);
    }, this.intervalMs);
  }
}

// Agent 处理心跳
async handleHeartbeat(): Promise<string> {
  // 检查 HEARTBEAT.md
  const heartbeatMd = await this.readFile('HEARTBEAT.md');
  
  if (!heartbeatMd || heartbeatMd.trim() === '') {
    return 'HEARTBEAT_OK';  // 没事干
  }
  
  // 有任务要处理
  return await this.runTasks(heartbeatMd);
}
```

## 事件过滤与路由

### 基于条件的过滤

```typescript
interface EventFilter {
  type?: string | string[];
  source?: string | string[];
  priority?: Priority[];
  custom?: (event: AgentEvent) => boolean;
}

class EventRouter {
  private routes: Map<EventFilter, EventHandler> = new Map();
  
  // 注册路由
  on(filter: EventFilter, handler: EventHandler): void {
    this.routes.set(filter, handler);
  }
  
  // 路由事件
  async route(event: AgentEvent): Promise<void> {
    for (const [filter, handler] of this.routes) {
      if (this.matches(event, filter)) {
        await handler(event);
        return;  // 或继续传播
      }
    }
  }
  
  private matches(event: AgentEvent, filter: EventFilter): boolean {
    if (filter.type) {
      const types = Array.isArray(filter.type) ? filter.type : [filter.type];
      if (!types.includes(event.type)) return false;
    }
    
    if (filter.source) {
      const sources = Array.isArray(filter.source) ? filter.source : [filter.source];
      if (!sources.includes(event.source)) return false;
    }
    
    if (filter.custom && !filter.custom(event)) {
      return false;
    }
    
    return true;
  }
}

// 使用
router.on(
  { type: 'message.received', source: 'telegram' },
  handleTelegramMessage
);

router.on(
  { type: ['schedule.triggered', 'heartbeat.poll'] },
  handleBackgroundTask
);

router.on(
  { custom: (e) => e.metadata?.priority === 'urgent' },
  handleUrgentEvent
);
```

### 优先级队列

```typescript
class PriorityEventQueue {
  private queues: Map<Priority, AgentEvent[]> = new Map([
    ['urgent', []],
    ['high', []],
    ['normal', []],
    ['low', []]
  ]);
  
  enqueue(event: AgentEvent): void {
    const priority = event.metadata?.priority ?? 'normal';
    this.queues.get(priority)!.push(event);
  }
  
  dequeue(): AgentEvent | undefined {
    // 按优先级顺序取
    for (const priority of ['urgent', 'high', 'normal', 'low']) {
      const queue = this.queues.get(priority as Priority)!;
      if (queue.length > 0) {
        return queue.shift();
      }
    }
    return undefined;
  }
}
```

## 事件持久化与重放

### 为什么要持久化？

1. **可靠性**：进程崩溃后不丢事件
2. **审计**：追溯发生了什么
3. **调试**：重放问题场景
4. **多实例**：事件共享

### Event Store 实现

```typescript
interface EventStore {
  // 追加事件
  append(event: AgentEvent): Promise<void>;
  
  // 读取事件
  read(options: ReadOptions): AsyncIterable<AgentEvent>;
  
  // 标记已处理
  markProcessed(eventId: string): Promise<void>;
}

interface ReadOptions {
  fromId?: string;
  fromTimestamp?: number;
  types?: string[];
  limit?: number;
  unprocessedOnly?: boolean;
}

// SQLite 实现
class SqliteEventStore implements EventStore {
  async append(event: AgentEvent): Promise<void> {
    await this.db.run(`
      INSERT INTO events (id, type, source, timestamp, payload, processed)
      VALUES (?, ?, ?, ?, ?, 0)
    `, [event.id, event.type, event.source, event.timestamp, JSON.stringify(event.payload)]);
  }
  
  async *read(options: ReadOptions): AsyncIterable<AgentEvent> {
    let sql = 'SELECT * FROM events WHERE 1=1';
    const params: any[] = [];
    
    if (options.fromTimestamp) {
      sql += ' AND timestamp >= ?';
      params.push(options.fromTimestamp);
    }
    
    if (options.unprocessedOnly) {
      sql += ' AND processed = 0';
    }
    
    sql += ' ORDER BY timestamp ASC';
    
    if (options.limit) {
      sql += ' LIMIT ?';
      params.push(options.limit);
    }
    
    const rows = await this.db.all(sql, params);
    for (const row of rows) {
      yield {
        id: row.id,
        type: row.type,
        source: row.source,
        timestamp: row.timestamp,
        payload: JSON.parse(row.payload)
      };
    }
  }
}
```

### 启动时恢复

```typescript
class Gateway {
  async start(): Promise<void> {
    // 恢复未处理的事件
    const unprocessed = this.eventStore.read({ unprocessedOnly: true });
    
    for await (const event of unprocessed) {
      try {
        await this.handleEvent(event);
        await this.eventStore.markProcessed(event.id);
      } catch (error) {
        console.error(`Failed to process event ${event.id}:`, error);
        // 可以重试或进入死信队列
      }
    }
    
    // 启动实时事件监听
    this.startEventSources();
  }
}
```

## 防重复处理（幂等性）

同一事件可能被处理多次（重试、重启等），需要保证幂等：

```typescript
class IdempotentEventHandler {
  private processed: Set<string> = new Set();
  private store: EventStore;
  
  async handle(event: AgentEvent): Promise<void> {
    // 使用 idempotencyKey 或 event.id
    const key = event.metadata?.idempotencyKey ?? event.id;
    
    // 检查是否已处理
    if (this.processed.has(key)) {
      console.log(`Event ${key} already processed, skipping`);
      return;
    }
    
    // 数据库级别去重
    const exists = await this.store.exists(key);
    if (exists) {
      this.processed.add(key);
      return;
    }
    
    // 处理事件
    await this.doHandle(event);
    
    // 标记已处理
    await this.store.markProcessed(key);
    this.processed.add(key);
  }
}
```

## 实战模式

### 1. 事件聚合

多个相关事件合并处理：

```typescript
class EventAggregator {
  private buffer: Map<string, AgentEvent[]> = new Map();
  private timeoutMs: number = 5000;
  
  async aggregate(event: AgentEvent, groupKey: string): Promise<AgentEvent[] | null> {
    if (!this.buffer.has(groupKey)) {
      this.buffer.set(groupKey, []);
      
      // 设置超时刷新
      setTimeout(() => this.flush(groupKey), this.timeoutMs);
    }
    
    this.buffer.get(groupKey)!.push(event);
    return null;  // 暂不处理
  }
  
  private async flush(groupKey: string): Promise<void> {
    const events = this.buffer.get(groupKey);
    if (events && events.length > 0) {
      this.buffer.delete(groupKey);
      await this.handleBatch(events);
    }
  }
}

// 使用场景：群消息聚合
// 5 秒内的多条消息合并成一次 LLM 调用
```

### 2. 死信队列 (DLQ)

处理失败的事件：

```typescript
class DeadLetterQueue {
  async send(event: AgentEvent, error: Error, attempts: number): Promise<void> {
    await this.store.append({
      ...event,
      type: 'dlq.entry',
      metadata: {
        ...event.metadata,
        originalType: event.type,
        error: error.message,
        attempts,
        dlqTime: Date.now()
      }
    });
  }
  
  // 重试 DLQ 中的事件
  async retry(): Promise<void> {
    const dlqEvents = this.store.read({ types: ['dlq.entry'] });
    for await (const event of dlqEvents) {
      // 恢复原始类型重新处理
      event.type = event.metadata.originalType;
      await this.gateway.handleEvent(event);
    }
  }
}
```

### 3. 事件驱动的工作流

```typescript
// 复杂任务分解为事件链
class WorkflowEngine {
  async executeWorkflow(workflow: Workflow): Promise<void> {
    // 发起第一个事件
    await this.emit({
      type: 'workflow.step',
      payload: {
        workflowId: workflow.id,
        stepIndex: 0,
        stepData: workflow.steps[0]
      }
    });
  }
  
  // 每个步骤完成后触发下一步
  async handleStepComplete(event: StepCompleteEvent): Promise<void> {
    const { workflowId, stepIndex, result } = event.payload;
    const workflow = await this.loadWorkflow(workflowId);
    
    if (stepIndex + 1 < workflow.steps.length) {
      // 还有下一步
      await this.emit({
        type: 'workflow.step',
        payload: {
          workflowId,
          stepIndex: stepIndex + 1,
          stepData: workflow.steps[stepIndex + 1],
          previousResult: result
        }
      });
    } else {
      // 工作流完成
      await this.emit({
        type: 'workflow.complete',
        payload: { workflowId, finalResult: result }
      });
    }
  }
}
```

## 对比其他架构

| 特性 | 请求-响应 | 事件驱动 |
|------|----------|---------|
| 耦合度 | 高 | 低 |
| 实时性 | 同步等待 | 异步推送 |
| 可扩展性 | 较难 | 容易 |
| 追溯能力 | 无 | 有完整事件流 |
| 复杂度 | 简单 | 需要额外基础设施 |
| 调试难度 | 容易 | 需要好的可观测性 |

## 总结

事件驱动架构让 Agent 从「被动应答」变成「主动感知」：

1. **定义清晰的事件类型** — 消息、定时、心跳、Webhook...
2. **统一的事件路由** — Gateway 负责分发
3. **持久化 + 幂等** — 保证可靠性
4. **优先级队列** — 重要事件优先处理
5. **死信队列** — 失败事件不丢失

这是构建「Always-on Agent」的核心架构模式。

---

下节预告：**Graceful Degradation（优雅降级）** — 当外部服务挂了，Agent 如何保持可用？
