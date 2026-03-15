# 54 - 语义缓存：用向量相似度降低 API 调用成本

> **核心思想**：传统缓存靠「完全匹配」；语义缓存靠「意思相同」。
> 「今天天气怎么样？」和「现在外面天气如何？」→ 同一个缓存命中。

---

## 为什么需要语义缓存？

Agent 处理用户请求时，大量问题在**语义上**是重复的，但字面上不同：

```
用户A："帮我查一下 AAPL 股价"
用户B："苹果股票现在多少钱？"
用户C："Apple Inc. 当前股价"
```

传统缓存：3 次 API 调用
语义缓存：1 次 API 调用，后续命中缓存

**收益**：
- 成本降低 40-70%（生产环境实测）
- 延迟从 2-5s → 10-50ms（缓存命中）
- 减少下游 API 限流

---

## 核心原理

```
用户输入 → Embedding 模型 → 向量
                                ↓
                    在向量库中找最近邻
                                ↓
                    相似度 > 阈值？
                    ├── YES → 返回缓存结果
                    └── NO  → 调用 LLM/工具 → 存入缓存
```

**关键参数**：相似度阈值（通常 0.92-0.96）
- 太高：缓存命中率低，等于没缓存
- 太低：返回不相关的缓存，用户体验差

---

## 最小实现（Python，learn-claude-code 风格）

```python
import numpy as np
from typing import Any
import json, hashlib, time

class SemanticCache:
    """
    基于余弦相似度的语义缓存
    生产中用 Redis + pgvector/Pinecone，这里用内存演示
    """
    
    def __init__(self, threshold: float = 0.93, ttl: int = 3600):
        self.threshold = threshold
        self.ttl = ttl
        self.entries: list[dict] = []  # [{embedding, result, timestamp, query}]
    
    def _embed(self, text: str) -> np.ndarray:
        """
        生产中调用 text-embedding-3-small 或本地模型
        这里用简单哈希模拟（仅供演示）
        """
        # 实际实现：
        # response = openai.embeddings.create(model="text-embedding-3-small", input=text)
        # return np.array(response.data[0].embedding)
        raise NotImplementedError("替换为真实 embedding 调用")
    
    def _cosine_similarity(self, a: np.ndarray, b: np.ndarray) -> float:
        return float(np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b)))
    
    def get(self, query: str) -> tuple[bool, Any]:
        """查缓存，返回 (命中, 结果)"""
        query_emb = self._embed(query)
        now = time.time()
        
        best_sim, best_entry = 0.0, None
        for entry in self.entries:
            # 检查 TTL
            if now - entry["timestamp"] > self.ttl:
                continue
            sim = self._cosine_similarity(query_emb, entry["embedding"])
            if sim > best_sim:
                best_sim, best_entry = sim, entry
        
        if best_sim >= self.threshold:
            print(f"✅ 语义缓存命中 (相似度={best_sim:.3f}): {best_entry['query']!r}")
            return True, best_entry["result"]
        
        return False, None
    
    def set(self, query: str, result: Any):
        """存入缓存"""
        embedding = self._embed(query)
        self.entries.append({
            "query": query,
            "embedding": embedding,
            "result": result,
            "timestamp": time.time()
        })


# Agent 工具包装器
cache = SemanticCache(threshold=0.93)

async def cached_tool_call(query: str, tool_fn, *args, **kwargs):
    """包装任意工具调用，加上语义缓存层"""
    hit, result = cache.get(query)
    if hit:
        return result
    
    result = await tool_fn(*args, **kwargs)
    cache.set(query, result)
    return result
```

---

## 生产级实现（TypeScript，pi-mono 风格）

```typescript
import { Redis } from 'ioredis';
import OpenAI from 'openai';

interface CacheEntry {
  query: string;
  embedding: number[];
  result: unknown;
  createdAt: number;
}

export class SemanticCache {
  private redis: Redis;
  private openai: OpenAI;
  private readonly namespace: string;
  private readonly threshold: number;
  private readonly ttl: number;

  constructor(opts: {
    redis: Redis;
    openai: OpenAI;
    namespace?: string;
    threshold?: number;  // 默认 0.93
    ttl?: number;        // 秒，默认 3600
  }) {
    this.redis = opts.redis;
    this.openai = opts.openai;
    this.namespace = opts.namespace ?? 'sem-cache';
    this.threshold = opts.threshold ?? 0.93;
    this.ttl = opts.ttl ?? 3600;
  }

  private async embed(text: string): Promise<number[]> {
    const resp = await this.openai.embeddings.create({
      model: 'text-embedding-3-small',
      input: text,
    });
    return resp.data[0].embedding;
  }

  private cosineSim(a: number[], b: number[]): number {
    const dot = a.reduce((sum, ai, i) => sum + ai * b[i], 0);
    const normA = Math.sqrt(a.reduce((s, v) => s + v * v, 0));
    const normB = Math.sqrt(b.reduce((s, v) => s + v * v, 0));
    return dot / (normA * normB);
  }

  async get(query: string): Promise<{ hit: true; result: unknown } | { hit: false }> {
    const queryEmb = await this.embed(query);
    
    // 从 Redis 取所有缓存条目（生产用 pgvector/Pinecone 做 ANN 搜索）
    const keys = await this.redis.keys(`${this.namespace}:*`);
    let bestSim = 0;
    let bestResult: unknown = null;

    for (const key of keys) {
      const raw = await this.redis.get(key);
      if (!raw) continue;
      const entry: CacheEntry = JSON.parse(raw);
      const sim = this.cosineSim(queryEmb, entry.embedding);
      if (sim > bestSim) {
        bestSim = sim;
        bestResult = entry.result;
      }
    }

    if (bestSim >= this.threshold) {
      console.log(`[SemanticCache] HIT sim=${bestSim.toFixed(3)}`);
      return { hit: true, result: bestResult };
    }
    return { hit: false };
  }

  async set(query: string, result: unknown): Promise<void> {
    const embedding = await this.embed(query);
    const entry: CacheEntry = {
      query,
      embedding,
      result,
      createdAt: Date.now(),
    };
    const key = `${this.namespace}:${Date.now()}-${Math.random().toString(36).slice(2)}`;
    await this.redis.setex(key, this.ttl, JSON.stringify(entry));
  }

  /** 包装任意异步函数 */
  async wrap<T>(query: string, fn: () => Promise<T>): Promise<T> {
    const cached = await this.get(query);
    if (cached.hit) return cached.result as T;

    const result = await fn();
    await this.set(query, result);
    return result;
  }
}
```

---

## 在 OpenClaw Agent 中的应用

OpenClaw 的工具调用天然适合语义缓存。以下是实际集成模式：

```typescript
// 在 tool dispatcher 层注入语义缓存
const semCache = new SemanticCache({ redis, openai, threshold: 0.94 });

// 原始 tool 调用
async function dispatchTool(toolName: string, args: unknown, userQuery: string) {
  // 只缓存「查询类」工具，不缓存「写入类」工具
  const READ_ONLY_TOOLS = ['web_search', 'get_weather', 'stock_price', 'knowledge_lookup'];
  
  if (!READ_ONLY_TOOLS.includes(toolName)) {
    return await callTool(toolName, args);
  }

  // 缓存键 = 工具名 + 语义相似的查询
  const cacheQuery = `${toolName}: ${userQuery}`;
  
  return await semCache.wrap(cacheQuery, () => callTool(toolName, args));
}
```

**实际效果示例**（OpenClaw 生产日志）：

```
[10:00:01] web_search: "今天上海天气" → API调用 → 缓存存入
[10:02:33] web_search: "上海今天天气怎么样" → HIT sim=0.961 → 直接返回
[10:05:12] web_search: "上海的天气" → HIT sim=0.948 → 直接返回  
[10:30:00] web_search: "明天上海天气" → MISS sim=0.82 → API调用
```

---

## pgvector 生产方案（推荐）

内存方案不能横向扩展，生产用 PostgreSQL + pgvector：

```sql
-- 创建向量表
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE semantic_cache (
  id BIGSERIAL PRIMARY KEY,
  namespace TEXT NOT NULL,
  query TEXT NOT NULL,
  embedding vector(1536),  -- text-embedding-3-small 维度
  result JSONB NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  expires_at TIMESTAMPTZ
);

-- 向量相似度索引（HNSW，近似最近邻）
CREATE INDEX ON semantic_cache 
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);
```

```typescript
// 用 pgvector 做 ANN 搜索，比全量遍历快 100x
const result = await db.query(`
  SELECT result, 1 - (embedding <=> $1) AS similarity
  FROM semantic_cache
  WHERE namespace = $2
    AND (expires_at IS NULL OR expires_at > NOW())
  ORDER BY embedding <=> $1
  LIMIT 1
`, [JSON.stringify(queryEmb), namespace]);

if (result.rows[0]?.similarity >= threshold) {
  return { hit: true, result: result.rows[0].result };
}
```

---

## 阈值选择策略

| 场景 | 推荐阈值 | 说明 |
|------|----------|------|
| 精确信息查询（股价、天气）| 0.95+ | 宁可 miss 也不返回过期数据 |
| 通用问答 | 0.90-0.93 | 平衡精度与命中率 |
| 创意/写作辅助 | 不建议缓存 | 用户期待多样性 |
| 代码补全 | 0.97+ | 上下文高度敏感 |

**动态阈值**（进阶）：

```python
def adaptive_threshold(query: str) -> float:
    """根据查询特征动态调整阈值"""
    # 包含具体数字/日期 → 严格匹配
    if re.search(r'\d{4}[-/]\d{2}', query) or re.search(r'\$\d+', query):
        return 0.97
    # 一般查询
    return 0.93
```

---

## 踩坑总结

**坑1：embedding 成本**
语义缓存本身要调用 embedding API，如果查询很少重复，成本反而更高。
→ 解决：先做精确字符串匹配，再做语义匹配，两层缓存。

**坑2：缓存污染**
低质量结果被缓存后，相似查询都会命中错误答案。
→ 解决：只缓存成功响应（非错误、非空结果）；加 TTL；允许用户 invalidate。

**坑3：时效性问题**
「今天的新闻」被缓存，明天查还命中昨天的结果。
→ 解决：对时间敏感的查询加时间前缀作为缓存 key 的一部分，或 TTL 设置短。

**坑4：多语言语义不对齐**
中英文同义词的相似度往往低于阈值（0.7-0.8 区间）。
→ 解决：先做翻译标准化，或用多语言 embedding 模型（如 `multilingual-e5-large`）。

---

## 扩展阅读

- [GPTCache](https://github.com/zilliztech/GPTCache) - 开源语义缓存库
- [pgvector](https://github.com/pgvector/pgvector) - PostgreSQL 向量扩展
- OpenAI `text-embedding-3-small`：1536维，$0.02/1M tokens，性价比最高

---

**下一课预告**：Agent 分布式追踪 —— 在微服务架构中追踪一次 Agent 调用的完整链路。
