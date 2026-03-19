# 92 - Agent 流式响应聚合（Streaming Response Aggregation）

> **核心思想**：多个 Sub-agent 同时产出流式结果时，如何把这些碎片聚合成一条流，实时推给用户——不等最慢的那个，不丢中间结果，出错了局部降级。

---

## 为什么需要流式聚合？

你拆了一个任务给 3 个 Sub-agent 并行执行，每个都在流式输出——

**没有聚合器时：**
```
Sub-agent A: ████████████████ (12s 后一次性返回)
Sub-agent B: ████████ (8s 后一次性返回)
Sub-agent C: ████████████ (10s 后一次性返回)
用户等待：12s 后看到全部结果
```

**有聚合器时：**
```
t=0.5s: [B] "正在分析..."
t=1.2s: [A] "找到 3 个候选方案"
t=2.1s: [C] "数据库查询中..."
t=3.0s: [B] "初步结论：..."
...用户实时看到进展，总体感知延迟降低 80%
```

---

## 核心模式

### 1. Fan-In 聚合器

```typescript
// pi-mono 风格：异步生成器 Fan-In
async function* fanInStreams<T>(
  streams: AsyncIterable<T>[],
  options: {
    timeout?: number;       // 单流超时（ms）
    errorStrategy?: 'fail-fast' | 'skip' | 'partial';
  } = {}
): AsyncGenerator<{ source: number; chunk: T } | { source: number; error: Error }> {
  const { timeout = 30_000, errorStrategy = 'partial' } = options;

  // 把每条流转成 Promise 队列，推入共享 channel
  const channel: Array<{ source: number; chunk: T } | { source: number; done: true } | { source: number; error: Error }> = [];
  let resolve: (() => void) | null = null;
  const wait = () => new Promise<void>(r => { resolve = r; });
  const notify = () => { resolve?.(); resolve = null; };

  let active = streams.length;

  // 每条流独立消费，结果推入 channel
  streams.forEach((stream, i) => {
    (async () => {
      try {
        const timer = timeout ? setTimeout(() => {
          channel.push({ source: i, error: new Error(`Stream ${i} timeout`) });
          notify();
        }, timeout) : null;

        for await (const chunk of stream) {
          channel.push({ source: i, chunk });
          notify();
        }

        if (timer) clearTimeout(timer);
        channel.push({ source: i, done: true });
      } catch (err) {
        channel.push({ source: i, error: err as Error });
      } finally {
        notify();
      }
    })();
  });

  // 主循环：从 channel 拉取并 yield
  while (active > 0) {
    while (channel.length === 0) await wait();

    const item = channel.shift()!;

    if ('done' in item) {
      active--;
    } else if ('error' in item) {
      active--;
      if (errorStrategy === 'fail-fast') throw item.error;
      if (errorStrategy === 'skip') continue;
      yield item; // partial: 把错误也传出去，让调用方决定
    } else {
      yield item;
    }
  }
}
```

### 2. 带优先级的聚合器

```typescript
// 某些 sub-agent 的结果更重要，应该优先展示
type Priority = 'high' | 'normal' | 'low';

interface PriorityChunk<T> {
  source: string;
  priority: Priority;
  chunk: T;
  timestamp: number;
}

class PriorityStreamAggregator<T> {
  private queues = {
    high: [] as PriorityChunk<T>[],
    normal: [] as PriorityChunk<T>[],
    low: [] as PriorityChunk<T>[],
  };
  private waiters: Array<() => void> = [];

  push(item: PriorityChunk<T>) {
    this.queues[item.priority].push(item);
    this.waiters.shift()?.(); // 唤醒一个等待者
  }

  async *drain(): AsyncGenerator<PriorityChunk<T>> {
    while (true) {
      // 先取 high，再 normal，最后 low
      const item =
        this.queues.high.shift() ??
        this.queues.normal.shift() ??
        this.queues.low.shift();

      if (item) {
        yield item;
      } else {
        // 等待新数据
        await new Promise<void>(r => this.waiters.push(r));
      }
    }
  }
}
```

### 3. 实时推送给用户

```typescript
// OpenClaw / pi-mono 中接入聚合器
async function runParallelAgentsWithStreaming(
  tasks: AgentTask[],
  onChunk: (source: string, text: string) => void,
  onDone: (results: Map<string, string>) => void
) {
  const results = new Map<string, string>();
  const accumulated = new Map<string, string>();

  // 启动所有 sub-agent，每个返回一个 AsyncIterable<string>
  const streams = tasks.map(task => ({
    id: task.id,
    stream: runSubAgent(task),
  }));

  const allStreams = streams.map(s => s.stream);

  for await (const { source, chunk } of fanInStreams(allStreams, {
    timeout: 30_000,
    errorStrategy: 'partial',
  })) {
    const taskId = streams[source].id;

    if ('error' in chunk) {
      onChunk(taskId, `[错误: ${(chunk as any).error.message}]`);
      results.set(taskId, '[failed]');
      continue;
    }

    const text = (chunk as any).chunk as string;
    const prev = accumulated.get(taskId) ?? '';
    accumulated.set(taskId, prev + text);

    // 实时推送给用户
    onChunk(taskId, text);
  }

  // 所有流结束，汇总结果
  for (const [id, text] of accumulated) {
    results.set(id, text);
  }

  onDone(results);
}
```

---

## OpenClaw 的实际实现

OpenClaw 的 `sessions_spawn` + `streamTo: "parent"` 就是这个模式的生产实现：

```typescript
// OpenClaw 内部：子 session 的流式输出汇入父 session
const subagent = await sessions_spawn({
  task: '分析竞品数据',
  streamTo: 'parent',  // 关键：流式推回父会话
  runtime: 'subagent',
  mode: 'run',
});

// 父 session 收到 subagent 的流式 token，直接转发给用户
// 不需要等 subagent 完全结束
```

多个 sub-agent 同时 `streamTo: "parent"` 时，OpenClaw 在父 session 层面做 fan-in，按到达顺序交错输出。

---

## 流取消传播

```typescript
// 用户中途取消 → 需要向下传播到所有 sub-stream
class CancellableStreamAggregator<T> {
  private abortController = new AbortController();

  get signal() { return this.abortController.signal; }

  cancel(reason?: string) {
    this.abortController.abort(reason ?? 'user cancelled');
  }

  async *aggregate(
    streams: AsyncIterable<T>[],
  ): AsyncGenerator<T> {
    const { signal } = this;

    for await (const item of fanInStreams(streams)) {
      if (signal.aborted) {
        // 清理：通知所有子流停止
        break;
      }
      yield (item as any).chunk;
    }
  }
}

// 使用
const agg = new CancellableStreamAggregator<string>();

// 用户按了停止按钮
onUserCancel(() => agg.cancel('user stopped'));

for await (const chunk of agg.aggregate(subStreams)) {
  sendToUI(chunk);
}
```

---

## 聚合策略对比

| 策略 | 适用场景 | 优点 | 缺点 |
|------|---------|------|------|
| **Interleave（交错）** | 多 agent 并行探索 | 用户实时看到进展 | 输出可能跳跃 |
| **Sequential（顺序）** | 有依赖的 pipeline | 结果连贯 | 等最慢的 |
| **First-wins** | 竞速模式（hedge request）| 延迟最低 | 浪费其他结果 |
| **Merge（合并）** | 同质内容（摘要集合）| 信息完整 | 需后处理去重 |
| **Priority（优先级）** | 重要度不同的子任务 | 关键结果先到 | 需要手工标优先级 |

---

## 错误处理三策略

```typescript
// fail-fast：任一子流出错，立刻终止（适合强依赖场景）
// skip：跳过出错的子流，继续（适合 best-effort 场景）
// partial：把错误也 yield 出去，让调用方决定（最灵活）

const results = fanInStreams(streams, { errorStrategy: 'partial' });

for await (const item of results) {
  if ('error' in item) {
    // 记录错误，继续处理其他流
    console.error(`Sub-agent ${item.source} failed:`, item.error);
    notifyUser(`部分结果不可用（来源 ${item.source}）`);
  } else {
    // 正常 chunk
    process(item.chunk);
  }
}
```

---

## 与已学知识的联系

| 已学模块 | 本课扩展点 |
|---------|-----------|
| 16 - Streaming 流式输出 | 单流 → 多流 Fan-In |
| 26 - Concurrent Tool Execution | 工具级并发 → 流级聚合 |
| 72 - Multi-Agent Negotiation | 协商结果 → 流式回传 |
| 60 - Backpressure & Flow Control | 聚合器内部的背压处理 |

---

## 实战建议

1. **不要等所有流结束**：有结果就推，用户体验 >> 完美排版
2. **设超时保护**：单个 sub-agent 卡死不能拖累整体
3. **保留 source 标签**：方便调试和用户理解哪个 agent 说的什么
4. **考虑前端渲染**：交错输出可能需要前端分 slot 渲染，避免视觉跳动
5. **优先级 + 超时组合**：high-priority stream 超时降级到 normal，不能让重要的等不重要的

---

## 小结

流式聚合是多 Agent 系统的"神经末梢"——它决定了用户感知到的响应速度。实现要点：

- **Fan-In** 把多路异步流合并到单一迭代器
- **Priority Queue** 让重要结果先到
- **取消传播** 保证资源及时释放
- **Partial Error** 局部失败不影响整体

下次你拆了一个任务给 5 个 Sub-agent，别傻等全部完成再返回——用聚合器，让用户边等边看，体验翻倍。🚀
