# 109 - Agent 批处理与微批优化（Batch Processing & Mini-Batch Optimization）

> **核心思想**：把多个独立请求聚合成批次，用一次 API 调用处理，成本降 50%、吞吐提升数倍。

---

## 为什么需要批处理？

Agent 系统在处理大量任务时面临两个矛盾：

| 策略 | 延迟 | 成本 | 吞吐 |
|------|------|------|------|
| 逐个请求 | 最低 | 最高 | 最低 |
| 全量批次 | 最高 | 最低 | 最高 |
| **微批（Mini-Batch）** | **中等** | **低** | **高** |

微批是生产环境的黄金方案：**在可接受延迟内聚合请求，平衡成本与响应速度**。

---

## 一、Anthropic Batch API（省 50% 成本）

Anthropic 提供异步批处理 API，**价格直接打五折**，适合非实时任务：

```python
# learn-claude-code 风格 - Python 实现
import anthropic
import asyncio
from dataclasses import dataclass
from typing import Any

@dataclass
class BatchRequest:
    custom_id: str
    prompt: str
    metadata: dict = None

class AnthropicBatchProcessor:
    def __init__(self):
        self.client = anthropic.Anthropic()
    
    def submit_batch(self, requests: list[BatchRequest]) -> str:
        """提交批次，返回 batch_id"""
        batch_items = [
            {
                "custom_id": req.custom_id,
                "params": {
                    "model": "claude-opus-4-5",
                    "max_tokens": 1024,
                    "messages": [{"role": "user", "content": req.prompt}]
                }
            }
            for req in requests
        ]
        
        batch = self.client.messages.batches.create(requests=batch_items)
        print(f"批次已提交: {batch.id}，共 {len(requests)} 条请求")
        return batch.id
    
    async def wait_for_results(self, batch_id: str, poll_interval: int = 30) -> dict:
        """轮询直到批次完成，返回 custom_id -> result 映射"""
        while True:
            batch = self.client.messages.batches.retrieve(batch_id)
            
            if batch.processing_status == "ended":
                results = {}
                for result in self.client.messages.batches.results(batch_id):
                    if result.result.type == "succeeded":
                        results[result.custom_id] = result.result.message.content[0].text
                    else:
                        results[result.custom_id] = None  # 失败的请求
                return results
            
            print(f"批次处理中... 已完成 {batch.request_counts.succeeded}/{batch.request_counts.processing}")
            await asyncio.sleep(poll_interval)

# 使用示例
async def main():
    processor = AnthropicBatchProcessor()
    
    # 构造 100 个评估任务
    requests = [
        BatchRequest(
            custom_id=f"eval-{i}",
            prompt=f"评估以下代码质量（1-10分）：\n```\n{code_snippet}\n```",
        )
        for i, code_snippet in enumerate(code_list)
    ]
    
    batch_id = processor.submit_batch(requests)
    results = await processor.wait_for_results(batch_id)
    
    for custom_id, score in results.items():
        print(f"{custom_id}: {score}")
```

**适用场景**：
- 批量内容审核
- 大规模数据分析
- 离线评估任务
- 定时报告生成

---

## 二、微批聚合器（实时场景）

对于需要快速响应但又想降低成本的场景，**微批聚合器**是答案：

```typescript
// pi-mono / OpenClaw 风格 - TypeScript 实现

interface PendingRequest<T> {
  id: string;
  payload: T;
  resolve: (result: any) => void;
  reject: (error: Error) => void;
  timestamp: number;
}

interface MiniBatchConfig {
  maxBatchSize: number;    // 最大批次大小
  maxWaitMs: number;       // 最大等待时间（ms）
  processBatch: (items: any[]) => Promise<any[]>;  // 批处理函数
}

class MiniBatchAggregator<TInput, TOutput> {
  private queue: PendingRequest<TInput>[] = [];
  private flushTimer: NodeJS.Timeout | null = null;
  private config: MiniBatchConfig;

  constructor(config: MiniBatchConfig) {
    this.config = config;
  }

  /**
   * 添加单个请求，返回 Promise
   * 调用方感知不到批处理的存在
   */
  async add(payload: TInput): Promise<TOutput> {
    return new Promise((resolve, reject) => {
      const request: PendingRequest<TInput> = {
        id: crypto.randomUUID(),
        payload,
        resolve,
        reject,
        timestamp: Date.now(),
      };

      this.queue.push(request);

      // 触发条件 1: 队列满了 → 立即 flush
      if (this.queue.length >= this.config.maxBatchSize) {
        this.flush();
        return;
      }

      // 触发条件 2: 设置计时器 → 超时 flush
      if (!this.flushTimer) {
        this.flushTimer = setTimeout(() => this.flush(), this.config.maxWaitMs);
      }
    });
  }

  private async flush(): Promise<void> {
    if (this.queue.length === 0) return;

    // 清空计时器
    if (this.flushTimer) {
      clearTimeout(this.flushTimer);
      this.flushTimer = null;
    }

    // 取出当前队列（原子操作，防止并发问题）
    const batch = this.queue.splice(0, this.config.maxBatchSize);
    
    console.log(`[MiniBatch] 处理批次 size=${batch.length}`);

    try {
      const results = await this.config.processBatch(batch.map(r => r.payload));
      
      // 按顺序分发结果
      batch.forEach((request, index) => {
        request.resolve(results[index]);
      });
    } catch (error) {
      // 批次失败 → 所有请求都 reject
      batch.forEach(request => request.reject(error as Error));
    }
  }
}

// ====== 在 Agent 中使用 ======

// 创建聚合器：最多等 100ms 或凑够 20 个请求
const embeddingBatcher = new MiniBatchAggregator<string, number[]>({
  maxBatchSize: 20,
  maxWaitMs: 100,
  
  processBatch: async (texts: string[]) => {
    // 一次 API 调用处理所有文本
    const response = await openai.embeddings.create({
      model: "text-embedding-3-small",
      input: texts,  // OpenAI 支持批量
    });
    return response.data.map(item => item.embedding);
  },
});

// 调用侧无感知 - 看起来像单个请求
async function getEmbedding(text: string): Promise<number[]> {
  return embeddingBatcher.add(text);
}

// 并发调用 → 自动聚合成批次
const embeddings = await Promise.all([
  getEmbedding("用户问题 A"),
  getEmbedding("用户问题 B"),
  getEmbedding("用户问题 C"),
  // ...更多并发请求会被自动聚合
]);
```

---

## 三、OpenClaw 中的批处理模式

在 OpenClaw 这类 Always-on Agent 平台中，批处理主要用于**工具调用合并**：

```typescript
// 模拟 OpenClaw 内部的工具批处理逻辑

class ToolBatchExecutor {
  private pendingTools: Map<string, {
    toolName: string;
    params: any;
    priority: number;
  }> = new Map();

  // 将可并行的工具调用分组执行
  async executeBatch(toolCalls: ToolCall[]): Promise<ToolResult[]> {
    // 1. 依赖分析 - 找出可并行的工具
    const { parallel, sequential } = this.analyzeDependencies(toolCalls);

    const results: ToolResult[] = [];

    // 2. 并行执行独立工具
    if (parallel.length > 0) {
      const parallelResults = await Promise.allSettled(
        parallel.map(call => this.executeTool(call))
      );
      results.push(...parallelResults.map(r => 
        r.status === 'fulfilled' ? r.value : { error: r.reason }
      ));
    }

    // 3. 顺序执行有依赖的工具
    for (const call of sequential) {
      const result = await this.executeTool(call);
      results.push(result);
    }

    return results;
  }

  private analyzeDependencies(calls: ToolCall[]) {
    // 无副作用的只读工具 → 并行
    const readOnlyTools = new Set(['web_search', 'read_file', 'memory_search']);
    
    const parallel = calls.filter(c => readOnlyTools.has(c.name));
    const sequential = calls.filter(c => !readOnlyTools.has(c.name));
    
    return { parallel, sequential };
  }
}
```

---

## 四、批处理队列 + 优先级调度

生产环境中，不同请求有不同优先级：

```typescript
// 带优先级的批处理队列

enum Priority {
  HIGH = 0,   // 实时用户请求
  NORMAL = 1, // 后台任务
  LOW = 2,    // 离线分析
}

class PriorityBatchQueue {
  private queues: Map<Priority, Array<PendingRequest<any>>> = new Map([
    [Priority.HIGH, []],
    [Priority.NORMAL, []],
    [Priority.LOW, []],
  ]);

  private readonly BATCH_LIMITS = {
    [Priority.HIGH]: { maxSize: 5, maxWait: 20 },    // 快速响应
    [Priority.NORMAL]: { maxSize: 20, maxWait: 200 }, // 均衡
    [Priority.LOW]: { maxSize: 100, maxWait: 5000 },  // 高吞吐
  };

  async add(payload: any, priority: Priority = Priority.NORMAL): Promise<any> {
    const queue = this.queues.get(priority)!;
    const limits = this.BATCH_LIMITS[priority];

    return new Promise((resolve, reject) => {
      queue.push({ payload, resolve, reject, id: crypto.randomUUID(), timestamp: Date.now() });
      
      if (queue.length >= limits.maxSize) {
        this.flushQueue(priority);
      }
    });
  }

  private async flushQueue(priority: Priority): Promise<void> {
    const queue = this.queues.get(priority)!;
    if (queue.length === 0) return;

    const batch = queue.splice(0, this.BATCH_LIMITS[priority].maxSize);
    
    // 高优先级请求插队处理
    console.log(`[Queue] 处理 ${Priority[priority]} 批次，size=${batch.length}`);
    await this.processBatch(batch, priority);
  }
}
```

---

## 五、实战：内容审核批处理 Agent

```python
# 完整示例：使用 Anthropic Batch API 的内容审核系统

import asyncio
import json
from dataclasses import dataclass, field
from typing import Literal
import anthropic

@dataclass 
class ModerationTask:
    content_id: str
    text: str
    content_type: Literal["comment", "post", "profile"]

class ContentModerationAgent:
    """使用批处理的内容审核 Agent"""
    
    def __init__(self):
        self.client = anthropic.Anthropic()
        self.pending: list[ModerationTask] = []
        self.flush_threshold = 50  # 积攒 50 条再批量处理
    
    def queue(self, task: ModerationTask):
        self.pending.append(task)
    
    async def process_all(self) -> dict[str, dict]:
        """批量处理所有待审核内容"""
        if not self.pending:
            return {}
        
        # 构造 Batch API 请求
        batch_requests = [
            {
                "custom_id": task.content_id,
                "params": {
                    "model": "claude-haiku-4-5",  # 用便宜的模型
                    "max_tokens": 256,
                    "messages": [{
                        "role": "user",
                        "content": f"""审核以下{task.content_type}内容，以JSON返回：
{{
  "is_safe": true/false,
  "category": "safe/spam/hate/nsfw/other",
  "confidence": 0-1,
  "reason": "简短说明"
}}

内容：{task.text[:500]}"""
                    }]
                }
            }
            for task in self.pending
        ]
        
        # 提交批次（比逐条便宜 50%）
        batch = self.client.messages.batches.create(requests=batch_requests)
        print(f"✅ 提交审核批次 {batch.id}，共 {len(batch_requests)} 条")
        
        # 等待完成
        while True:
            batch = self.client.messages.batches.retrieve(batch.id)
            if batch.processing_status == "ended":
                break
            await asyncio.sleep(10)
        
        # 解析结果
        results = {}
        for result in self.client.messages.batches.results(batch.id):
            if result.result.type == "succeeded":
                try:
                    text = result.result.message.content[0].text
                    # 提取 JSON（模型可能输出额外文字）
                    start = text.find('{')
                    end = text.rfind('}') + 1
                    results[result.custom_id] = json.loads(text[start:end])
                except:
                    results[result.custom_id] = {"is_safe": True, "category": "parse_error"}
        
        self.pending.clear()
        return results

# 使用
async def run_moderation():
    agent = ContentModerationAgent()
    
    # 积累任务
    for item in user_generated_content:
        agent.queue(ModerationTask(
            content_id=item["id"],
            text=item["text"],
            content_type="comment"
        ))
    
    # 一次批量处理，节省 50% 成本
    decisions = await agent.process_all()
    
    # 处理违规内容
    for content_id, decision in decisions.items():
        if not decision["is_safe"]:
            await flag_content(content_id, decision["category"])
```

---

## 六、关键指标与调优

| 参数 | 建议值 | 说明 |
|------|--------|------|
| `maxWaitMs` | 50-200ms | 实时场景；离线可设 5s+ |
| `maxBatchSize` | 10-50 | 太大增加延迟，太小失去优势 |
| 批次失败策略 | 拆分重试 | 二分法定位失败请求 |
| 队列溢出 | 背压 | 超过容量时拒绝新请求并返回 503 |

**监控指标**：
- `batch_fill_rate`：批次填充率（越高越省钱）
- `p50/p99_batch_wait`：等待时间分布
- `batch_failure_rate`：批次失败率

---

## 总结

```
单次请求 → 微批聚合 → Anthropic Batch API
   ↓              ↓                ↓
最低延迟     平衡方案        最低成本(50% off)
实时对话   后台队列任务      离线分析/审核
```

**选型原则**：
- 用户等待 → **单次请求**
- 后台处理、可接受秒级延迟 → **微批聚合器**
- 离线任务、不在乎时间 → **Anthropic Batch API**

配合第 96 课的**请求去重合并**和第 97 课的**自适应超时**，可以构建出生产级高效 Agent 基础设施。
