# 141 - Agent 记忆衰减与主动遗忘（Memory Decay & Active Forgetting）

> **核心思想**：Agent 的记忆不应该是永久存储的数据库，而是像人类大脑一样——重要的记住，不重要的忘掉。

---

## 为什么需要"遗忘"？

大多数 Agent 实现都在拼命"记住"一切：
- RAG 向量库越塞越大
- MEMORY.md 越写越长
- 上下文窗口装满过期信息

结果：**噪音淹没信号**。Agent 被过时信息干扰，检索质量下降，Token 成本飙升。

人类大脑进化出了遗忘机制——这不是缺陷，是特性。

---

## Ebbinghaus 遗忘曲线

Hermann Ebbinghaus 1885年发现：记忆保留率随时间指数衰减：

```
R(t) = e^(-t/S)
```

- `R` = 记忆保留率（0~1）
- `t` = 距离上次强化的时间
- `S` = 记忆稳定性（重要信息衰减慢）

**关键洞察**：每次使用/强化记忆，稳定性 S 会增大（间隔重复效应）。

---

## 实现：Memory Decay Engine

```typescript
// memory/decay-engine.ts

interface MemoryItem {
  id: string;
  content: string;
  embedding?: number[];
  
  // 衰减参数
  stability: number;      // S：稳定性（越高越难忘）
  lastReviewed: number;   // 上次强化时间戳
  accessCount: number;    // 累计访问次数
  importance: number;     // 人工/自动标注的重要性 0~1
  
  createdAt: number;
  tags: string[];
}

class MemoryDecayEngine {
  private readonly BASE_STABILITY = 1.0;        // 新记忆默认稳定性（天）
  private readonly FORGET_THRESHOLD = 0.2;       // 低于此值触发遗忘
  private readonly STABILITY_MULTIPLIER = 1.8;   // 每次复习，稳定性 × 1.8

  /**
   * 计算当前记忆保留率
   * R(t) = importance × e^(-t / S)
   */
  retentionRate(item: MemoryItem): number {
    const daysSinceReview = (Date.now() - item.lastReviewed) / 86_400_000;
    const baseRetention = Math.exp(-daysSinceReview / item.stability);
    
    // 重要性系数放大保留率
    return Math.min(1.0, item.importance * baseRetention + (1 - item.importance) * baseRetention * 0.5);
  }

  /**
   * 强化记忆（使用时调用）
   * 间隔重复：间隔越长，稳定性增幅越大
   */
  reinforce(item: MemoryItem): MemoryItem {
    const daysSinceReview = (Date.now() - item.lastReviewed) / 86_400_000;
    
    // 间隔越长，强化效果越显著（间隔重复核心）
    const intervalBonus = Math.log1p(daysSinceReview);
    const newStability = item.stability * this.STABILITY_MULTIPLIER * (1 + intervalBonus * 0.1);
    
    return {
      ...item,
      stability: Math.min(newStability, 365), // 上限 365 天
      lastReviewed: Date.now(),
      accessCount: item.accessCount + 1,
    };
  }

  /**
   * 判断是否应该遗忘
   */
  shouldForget(item: MemoryItem): boolean {
    // 高重要性记忆豁免
    if (item.importance > 0.8) return false;
    
    return this.retentionRate(item) < this.FORGET_THRESHOLD;
  }

  /**
   * 新建记忆，根据内容自动评估重要性
   */
  async createMemory(
    content: string,
    tags: string[],
    importanceHint?: number
  ): Promise<MemoryItem> {
    // 自动评估重要性（可换成 LLM 打分）
    const importance = importanceHint ?? this.estimateImportance(content, tags);
    
    // 重要性高的记忆初始稳定性更高
    const stability = this.BASE_STABILITY * (1 + importance * 2);
    
    return {
      id: crypto.randomUUID(),
      content,
      stability,
      lastReviewed: Date.now(),
      accessCount: 0,
      importance,
      createdAt: Date.now(),
      tags,
    };
  }

  private estimateImportance(content: string, tags: string[]): number {
    let score = 0.3; // 基础分
    
    // 规则启发式
    if (tags.includes('user-preference')) score += 0.3;
    if (tags.includes('error-lesson')) score += 0.25;
    if (tags.includes('decision')) score += 0.2;
    if (tags.includes('casual')) score -= 0.15;
    
    // 内容长度启发（详细记录通常更重要）
    if (content.length > 200) score += 0.1;
    
    return Math.max(0, Math.min(1, score));
  }
}
```

---

## 主动遗忘：定期清理任务

```typescript
// memory/garbage-collector.ts

class MemoryGarbageCollector {
  constructor(
    private engine: MemoryDecayEngine,
    private store: MemoryStore,
    private llm: LLMClient,
  ) {}

  /**
   * 主动遗忘流程：
   * 1. 扫描所有记忆，计算保留率
   * 2. 候选遗忘 → LLM 二次确认价值
   * 3. 真正遗忘 or 提升重要性
   * 4. 生成"遗忘摘要"归档
   */
  async runGC(): Promise<GCReport> {
    const allMemories = await this.store.getAll();
    const report: GCReport = { forgotten: [], promoted: [], retained: [] };

    const candidates = allMemories.filter(m => this.engine.shouldForget(m));
    
    if (candidates.length === 0) {
      return report;
    }

    // 批量让 LLM 评估候选遗忘的记忆是否真的可以丢弃
    const assessments = await this.llmBatchAssess(candidates);

    for (const { memory, keepScore } of assessments) {
      if (keepScore < 0.3) {
        // 真正遗忘——但先写入冷存储（可回溯）
        await this.store.archive(memory.id);
        report.forgotten.push(memory.id);
      } else if (keepScore > 0.7) {
        // LLM 认为有价值 → 提升重要性防止下次被遗忘
        const updated = { ...memory, importance: Math.min(1, memory.importance + 0.2) };
        await this.store.update(updated);
        report.promoted.push(memory.id);
      } else {
        report.retained.push(memory.id);
      }
    }

    return report;
  }

  private async llmBatchAssess(
    memories: MemoryItem[]
  ): Promise<Array<{ memory: MemoryItem; keepScore: number }>> {
    // 批量评估，避免每条记忆单独调 LLM
    const prompt = `
以下是一批 Agent 记忆条目，请评估每条的长期价值（0=可以遗忘，1=必须保留）。
只返回 JSON 数组：[{"id":"...","score":0.x},...]

记忆列表：
${memories.map(m => `ID:${m.id} | ${m.content.slice(0, 100)}`).join('\n')}
    `.trim();

    const result = await this.llm.complete(prompt);
    const scores: Array<{ id: string; score: number }> = JSON.parse(result);

    return memories.map(m => ({
      memory: m,
      keepScore: scores.find(s => s.id === m.id)?.score ?? 0.5,
    }));
  }
}
```

---

## 在 OpenClaw 中集成

OpenClaw 的 `MEMORY.md` 就是 Agent 的长期记忆。我们可以用 Cron 驱动定期衰减扫描：

```typescript
// OpenClaw Cron Job: 每天凌晨 3 点运行记忆 GC
{
  schedule: { kind: "cron", expr: "0 3 * * *", tz: "Australia/Sydney" },
  payload: {
    kind: "agentTurn",
    message: `
运行记忆衰减维护：
1. 读取 MEMORY.md，识别超过 30 天未被引用的条目
2. 对低重要性、过时的条目，用【已归档】标记替代详细内容
3. 确保高重要性条目（用户偏好、重大决策）不被清理
4. 更新 MEMORY.md
    `,
  },
  sessionTarget: "isolated",
}
```

---

## 间隔重复：强化重要记忆

不只是遗忘，还要**主动强化**。每次 Agent 访问记忆时调用 `reinforce()`：

```typescript
// 在 RAG 检索后，强化被使用的记忆
async function retrieveAndReinforce(query: string): Promise<string[]> {
  const results = await vectorStore.search(query, topK: 5);
  
  // 异步强化（不阻塞主流程）
  setImmediate(async () => {
    for (const result of results) {
      const item = await memStore.get(result.id);
      const reinforced = decayEngine.reinforce(item);
      await memStore.update(reinforced);
    }
  });
  
  return results.map(r => r.content);
}
```

这样经常被用到的记忆自动变得更稳固，冷门的自然衰减——完全符合人类记忆规律。

---

## Redis 实现：用 Sorted Set 管理衰减

```python
# 用 Redis Sorted Set 存储记忆，score = 预测遗忘时间戳
# score 越小 = 越快被遗忘 = 优先处理

import redis
import math
import time

r = redis.Redis()

def add_memory(memory_id: str, stability_days: float, importance: float):
    """添加记忆，预计算遗忘时间"""
    # 遗忘时间 = 现在 + S * ln(importance/threshold)
    threshold = 0.2
    forget_at = time.time() + stability_days * 86400 * math.log(importance / threshold)
    r.zadd("memories:decay", {memory_id: forget_at})

def get_expiring_memories(limit: int = 100) -> list[str]:
    """获取即将遗忘的记忆（score < 现在）"""
    now = time.time()
    return r.zrangebyscore("memories:decay", "-inf", now, start=0, num=limit)

def reinforce_memory(memory_id: str, new_stability_days: float):
    """强化记忆，更新遗忘时间"""
    new_forget_at = time.time() + new_stability_days * 86400
    r.zadd("memories:decay", {memory_id: new_forget_at})
```

---

## 效果对比

| 指标 | 无衰减 | 有衰减 |
|------|--------|--------|
| 6 个月后记忆库大小 | 100% | ~30% |
| 检索相关性 | 下降（噪音多） | 稳定（高质量） |
| RAG P95 延迟 | 上升 | 稳定 |
| Token 消耗/次 | 持续增加 | 可控 |

---

## 关键设计原则

1. **遗忘不是删除**：先归档，保留可回溯能力
2. **重要性 × 使用频率 = 稳定性**：两者缺一不可
3. **LLM 作为最终裁判**：规则过滤 → LLM 确认，防误杀
4. **异步执行**：GC 不能阻塞主 Agent Loop
5. **透明度**：记录遗忘原因，便于审计

---

## 小结

```
新记忆创建 → 评估重要性 → 设置初始稳定性
     ↓
每次使用 → reinforce() → 稳定性 × 1.8
     ↓
Cron 定期扫描 → R(t) < 0.2 → LLM 二次评估 → 归档 or 提升
```

Agent 记忆系统的终极目标：**像专家一样记住重要的，像大脑一样忘掉无关的**。

---

*下一节预告：Agent 工具调用意图反转检测（Intent Reversal Detection）——识别用户意图在对话中途发生反转的情况。*
