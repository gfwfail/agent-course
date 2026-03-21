# 105 - Agent 自主学习与经验积累（Self-Learning & Experience Accumulation）

> 大多数 Agent 每次都从零开始，犯同样的错误，走同样的弯路。
> 本节教你让 Agent 在任务完成后**提取经验、向量化存储、下次自动检索复用**，真正实现"越用越好"。

---

## 问题：Agent 没有职业成长

```
第 1 次处理"用户投诉"：
  → 查了 5 个工具才找到订单
  → 忘记先检查 VIP 状态
  → 回复语气太生硬，用户不满意

第 100 次处理"用户投诉"：
  → 还是查了 5 个工具
  → 还是忘记 VIP 状态
  → 还是语气生硬

有经验的人类员工 vs 永远是新人的 Agent
```

**根本原因**：Agent 的上下文窗口清空后，经验归零。
**解决方案**：Task-level Experience Loop —— 任务完成 → 提炼经验 → 持久化 → 下次检索注入。

---

## 架构：Experience Loop

```
┌─────────────────────────────────────────────────────┐
│                   Experience Loop                    │
│                                                      │
│  Task Start                                          │
│      │                                               │
│      ▼                                               │
│  [1] Experience Retrieval  ←── Vector DB            │
│      │ 检索相关历史经验                               │
│      ▼                                               │
│  [2] Context Injection                               │
│      │ 把经验注入 system prompt                       │
│      ▼                                               │
│  [3] Task Execution                                  │
│      │ 正常执行（带经验加持）                         │
│      ▼                                               │
│  [4] Experience Extraction  ←── LLM 提炼             │
│      │ 任务完成后总结 key learnings                   │
│      ▼                                               │
│  [5] Experience Storage ────► Vector DB              │
│      │ 向量化、打标签、存储                            │
│      ▼                                               │
│  (下次任务 repeat)                                    │
└─────────────────────────────────────────────────────┘
```

---

## 实现：Python（learn-claude-code 风格）

### Step 1：经验数据结构

```python
from dataclasses import dataclass, field
from datetime import datetime
from typing import Optional
import hashlib, json

@dataclass
class Experience:
    """一条 Agent 经验"""
    id: str                          # 唯一 ID
    task_type: str                   # 任务类型标签（如 "customer_complaint"）
    context_summary: str             # 任务场景简述（用于语义检索）
    learnings: list[str]             # 核心经验列表
    outcome: str                     # 结果：success / partial / failure
    tool_sequence: list[str]         # 成功的工具调用顺序
    pitfalls: list[str]              # 踩过的坑
    created_at: datetime = field(default_factory=datetime.now)
    ttl_days: int = 90               # 经验有效期（避免过时经验误导）
    use_count: int = 0               # 被检索使用次数
    success_rate: float = 1.0        # 使用后的成功率（反馈更新）

    @property
    def is_expired(self) -> bool:
        age = (datetime.now() - self.created_at).days
        return age > self.ttl_days

    def to_prompt_text(self) -> str:
        """格式化为注入 prompt 的文本"""
        lines = [f"[历史经验] 场景：{self.context_summary}"]
        for i, learning in enumerate(self.learnings, 1):
            lines.append(f"  {i}. {learning}")
        if self.pitfalls:
            lines.append(f"  ⚠️ 注意：{'; '.join(self.pitfalls)}")
        if self.tool_sequence:
            lines.append(f"  🔧 推荐工具顺序：{' → '.join(self.tool_sequence)}")
        return "\n".join(lines)
```

### Step 2：经验提取器（用 LLM 总结）

```python
import anthropic

class ExperienceExtractor:
    """任务完成后，让 LLM 提炼经验"""
    
    def __init__(self, client: anthropic.Anthropic):
        self.client = client
    
    async def extract(
        self,
        task_description: str,
        execution_trace: list[dict],  # tool calls + results
        final_outcome: str,
        outcome_status: str  # success / partial / failure
    ) -> Experience:
        
        trace_text = self._format_trace(execution_trace)
        
        prompt = f"""你是一个经验提炼专家。分析以下 Agent 任务执行过程，提取可复用的经验教训。

任务描述：{task_description}
执行结果：{outcome_status} - {final_outcome}

执行过程：
{trace_text}

请以 JSON 格式输出：
{{
  "task_type": "任务类型标签（snake_case，如 customer_complaint）",
  "context_summary": "15字内的场景描述，用于语义检索",
  "learnings": ["经验1", "经验2", ...],  // 3-5条，具体可操作
  "pitfalls": ["坑1", "坑2"],  // 踩过的坑，可为空
  "tool_sequence": ["tool_a", "tool_b"]  // 成功路径的工具顺序，可为空
}}"""

        response = self.client.messages.create(
            model="claude-opus-4-5",
            max_tokens=1000,
            messages=[{"role": "user", "content": prompt}]
        )
        
        data = json.loads(response.content[0].text)
        exp_id = hashlib.md5(
            f"{task_description}{datetime.now().isoformat()}".encode()
        ).hexdigest()[:12]
        
        return Experience(
            id=exp_id,
            task_type=data["task_type"],
            context_summary=data["context_summary"],
            learnings=data["learnings"],
            pitfalls=data.get("pitfalls", []),
            tool_sequence=data.get("tool_sequence", []),
            outcome=outcome_status,
        )
    
    def _format_trace(self, trace: list[dict]) -> str:
        lines = []
        for step in trace:
            if step["type"] == "tool_call":
                lines.append(f"  调用 {step['tool']}({step.get('args_summary', '')})")
            elif step["type"] == "tool_result":
                result_preview = str(step.get("result", ""))[:100]
                lines.append(f"  结果：{result_preview}")
            elif step["type"] == "error":
                lines.append(f"  ❌ 错误：{step['message']}")
        return "\n".join(lines)
```

### Step 3：经验存储与检索（向量化）

```python
import sqlite3
import numpy as np

class ExperienceStore:
    """
    简化版向量存储：SQLite + numpy 余弦相似度
    生产环境可换 Chroma / Qdrant / pgvector
    """
    
    def __init__(self, db_path: str, embed_fn):
        self.db_path = db_path
        self.embed_fn = embed_fn  # text -> np.array
        self._init_db()
    
    def _init_db(self):
        conn = sqlite3.connect(self.db_path)
        conn.execute("""
            CREATE TABLE IF NOT EXISTS experiences (
                id TEXT PRIMARY KEY,
                task_type TEXT,
                context_summary TEXT,
                data_json TEXT,
                embedding BLOB,  -- 序列化的 numpy array
                created_at TEXT,
                ttl_days INTEGER,
                use_count INTEGER DEFAULT 0,
                success_rate REAL DEFAULT 1.0
            )
        """)
        conn.commit()
        conn.close()
    
    async def save(self, exp: Experience):
        """向量化并存储经验"""
        # 用 context_summary 生成 embedding（语义检索的 key）
        embedding = await self.embed_fn(exp.context_summary)
        embedding_blob = embedding.astype(np.float32).tobytes()
        
        conn = sqlite3.connect(self.db_path)
        conn.execute("""
            INSERT OR REPLACE INTO experiences 
            (id, task_type, context_summary, data_json, embedding, created_at, ttl_days)
            VALUES (?, ?, ?, ?, ?, ?, ?)
        """, (
            exp.id, exp.task_type, exp.context_summary,
            json.dumps(exp.__dict__, default=str),
            embedding_blob,
            exp.created_at.isoformat(),
            exp.ttl_days
        ))
        conn.commit()
        conn.close()
    
    async def retrieve(
        self,
        query: str,
        task_type: str = None,
        top_k: int = 3,
        min_similarity: float = 0.75
    ) -> list[Experience]:
        """语义检索最相关经验"""
        query_embedding = await self.embed_fn(query)
        
        conn = sqlite3.connect(self.db_path)
        
        # 过滤条件：未过期、任务类型匹配（可选）
        where_clauses = []
        params = []
        if task_type:
            where_clauses.append("task_type = ?")
            params.append(task_type)
        
        where_sql = f"WHERE {' AND '.join(where_clauses)}" if where_clauses else ""
        rows = conn.execute(
            f"SELECT id, data_json, embedding FROM experiences {where_sql}",
            params
        ).fetchall()
        conn.close()
        
        # 计算余弦相似度
        scored = []
        for row_id, data_json, emb_blob in rows:
            stored_emb = np.frombuffer(emb_blob, dtype=np.float32)
            similarity = self._cosine_sim(query_embedding, stored_emb)
            if similarity >= min_similarity:
                exp_data = json.loads(data_json)
                exp = Experience(**{
                    k: v for k, v in exp_data.items() 
                    if k in Experience.__dataclass_fields__
                })
                if not exp.is_expired:
                    scored.append((similarity, exp))
        
        # 按相似度降序，取 top_k
        scored.sort(key=lambda x: x[0], reverse=True)
        return [exp for _, exp in scored[:top_k]]
    
    def _cosine_sim(self, a: np.ndarray, b: np.ndarray) -> float:
        return float(np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b) + 1e-8))
    
    async def update_feedback(self, exp_id: str, success: bool):
        """任务完成后反馈更新成功率（指数移动平均）"""
        conn = sqlite3.connect(self.db_path)
        row = conn.execute(
            "SELECT use_count, success_rate FROM experiences WHERE id = ?", (exp_id,)
        ).fetchone()
        
        if row:
            use_count, success_rate = row
            alpha = 0.1  # EMA 系数
            new_rate = success_rate * (1 - alpha) + (1.0 if success else 0.0) * alpha
            conn.execute(
                "UPDATE experiences SET use_count = ?, success_rate = ? WHERE id = ?",
                (use_count + 1, new_rate, exp_id)
            )
            conn.commit()
        conn.close()
```

### Step 4：Experience-Aware Agent

```python
class ExperienceAwareAgent:
    """
    把经验循环集成进 Agent 主循环
    """
    
    def __init__(
        self,
        client: anthropic.Anthropic,
        tools: list,
        experience_store: ExperienceStore,
        extractor: ExperienceExtractor,
    ):
        self.client = client
        self.tools = tools
        self.store = experience_store
        self.extractor = extractor
        self.execution_trace = []
    
    async def run(self, task: str, task_type: str = None) -> str:
        # ① 检索相关历史经验
        experiences = await self.store.retrieve(
            query=task, 
            task_type=task_type,
            top_k=3
        )
        
        # ② 构建经验增强的 system prompt
        system = self._build_system_prompt(experiences)
        exp_ids_used = [exp.id for exp in experiences]
        
        # ③ 正常执行 Agent Loop
        messages = [{"role": "user", "content": task}]
        result = await self._agent_loop(system, messages)
        
        # ④ 执行完成后提取新经验
        outcome_status = "success"  # 实际应根据结果判断
        new_exp = await self.extractor.extract(
            task_description=task,
            execution_trace=self.execution_trace,
            final_outcome=result,
            outcome_status=outcome_status
        )
        await self.store.save(new_exp)
        
        # ⑤ 更新被使用经验的成功率反馈
        for exp_id in exp_ids_used:
            await self.store.update_feedback(exp_id, success=True)
        
        return result
    
    def _build_system_prompt(self, experiences: list[Experience]) -> str:
        base = """你是一个专业的 AI 助理，擅长高效完成任务。"""
        
        if not experiences:
            return base
        
        exp_text = "\n\n".join(exp.to_prompt_text() for exp in experiences)
        return f"""{base}

## 相关历史经验（自动检索）

以下是处理类似任务时积累的经验，供参考：

{exp_text}

请结合这些经验，避免重蹈覆辙，优先使用已验证的工具路径。"""
    
    async def _agent_loop(self, system: str, messages: list) -> str:
        """标准 Agent Loop（工具调用循环）"""
        while True:
            response = self.client.messages.create(
                model="claude-opus-4-5",
                max_tokens=4096,
                system=system,
                tools=[t.schema for t in self.tools],
                messages=messages
            )
            
            if response.stop_reason == "end_turn":
                return response.content[0].text
            
            # 处理工具调用
            tool_results = []
            for block in response.content:
                if block.type == "tool_use":
                    tool = self._get_tool(block.name)
                    result = await tool.run(**block.input)
                    
                    # 记录执行轨迹
                    self.execution_trace.append({
                        "type": "tool_call",
                        "tool": block.name,
                        "args_summary": str(block.input)[:80]
                    })
                    self.execution_trace.append({
                        "type": "tool_result",
                        "result": result
                    })
                    
                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": str(result)
                    })
            
            messages.extend([
                {"role": "assistant", "content": response.content},
                {"role": "user", "content": tool_results}
            ])
```

---

## TypeScript 版（pi-mono 风格）

```typescript
// experience-store.ts
interface Experience {
  id: string;
  taskType: string;
  contextSummary: string;
  learnings: string[];
  pitfalls: string[];
  toolSequence: string[];
  outcome: 'success' | 'partial' | 'failure';
  createdAt: Date;
  ttlDays: number;
  useCount: number;
  successRate: number;
}

class ExperienceStore {
  private db: Map<string, Experience & { embedding: number[] }> = new Map();

  async save(exp: Experience, embedFn: (text: string) => Promise<number[]>) {
    const embedding = await embedFn(exp.contextSummary);
    this.db.set(exp.id, { ...exp, embedding });
    console.log(`[Experience] Saved: "${exp.contextSummary}" (${exp.id})`);
  }

  async retrieve(
    query: string,
    embedFn: (text: string) => Promise<number[]>,
    topK = 3,
    minSimilarity = 0.75
  ): Promise<Experience[]> {
    const queryEmb = await embedFn(query);
    const now = new Date();

    const scored: [number, Experience][] = [];
    for (const [, stored] of this.db) {
      // 过滤过期
      const ageDays = (now.getTime() - new Date(stored.createdAt).getTime()) / 86400000;
      if (ageDays > stored.ttlDays) continue;

      const sim = this.cosineSim(queryEmb, stored.embedding);
      if (sim >= minSimilarity) {
        scored.push([sim, stored]);
      }
    }

    return scored
      .sort(([a], [b]) => b - a)
      .slice(0, topK)
      .map(([, exp]) => exp);
  }

  async updateFeedback(id: string, success: boolean) {
    const exp = this.db.get(id);
    if (!exp) return;
    const alpha = 0.1;
    exp.successRate = exp.successRate * (1 - alpha) + (success ? 1.0 : 0.0) * alpha;
    exp.useCount++;
  }

  private cosineSim(a: number[], b: number[]): number {
    const dot = a.reduce((sum, ai, i) => sum + ai * b[i], 0);
    const normA = Math.sqrt(a.reduce((s, ai) => s + ai * ai, 0));
    const normB = Math.sqrt(b.reduce((s, bi) => s + bi * bi, 0));
    return dot / (normA * normB + 1e-8);
  }
}

// 经验注入到 system prompt
function buildSystemWithExperiences(
  base: string,
  experiences: Experience[]
): string {
  if (experiences.length === 0) return base;

  const expText = experiences.map(exp => {
    const lines = [`[历史经验] ${exp.contextSummary}`];
    exp.learnings.forEach((l, i) => lines.push(`  ${i + 1}. ${l}`));
    if (exp.pitfalls.length) lines.push(`  ⚠️ ${exp.pitfalls.join('; ')}`);
    if (exp.toolSequence.length) lines.push(`  🔧 ${exp.toolSequence.join(' → ')}`);
    return lines.join('\n');
  }).join('\n\n');

  return `${base}\n\n## 相关历史经验\n\n${expText}`;
}
```

---

## OpenClaw 中的实际体现

OpenClaw 的 `memory/YYYY-MM-DD.md` 和 `MEMORY.md` 就是这个模式的自然语言版：

```
Task Execution
    ↓
[人工提炼] 或 [LLM 提炼] 经验
    ↓
写入 memory/2026-03-21.md（今日原始记录）
    ↓
定期蒸馏 → MEMORY.md（长期精华）
    ↓
每次启动读 MEMORY.md（经验注入）
```

AGENTS.md 中明确写道：
```
📝 Write It Down - No "Mental Notes"!
"Mental notes" don't survive session restarts. Files do.
```

这就是手工版 Experience Loop。自动版用向量 + LLM 提炼做同样的事。

---

## 经验老化策略

```python
# 不同任务类型设置不同的 TTL
TTL_CONFIG = {
    "customer_complaint": 90,    # 90天：流程类，相对稳定
    "api_debug": 30,             # 30天：技术类，版本迭代快
    "market_research": 7,        # 7天：市场类，变化极快
    "code_review": 180,          # 180天：编码规范，长期有效
}

# 按成功率动态降权（不删除，只是检索时排序靠后）
def relevance_score(similarity: float, exp: Experience) -> float:
    age_penalty = 1.0 - (age_days / exp.ttl_days) * 0.3  # 越老越低
    return similarity * exp.success_rate * age_penalty
```

---

## 对比：三种经验机制

| 机制 | 位置 | 粒度 | 适合场景 |
|------|------|------|----------|
| **Working Memory** | 当前上下文 | 实时 | 单次任务内推理 |
| **Episode Memory** | 今日日志 | 任务级 | 短期回溯、今天干了啥 |
| **Semantic Experience** | 向量 DB | 语义级 | **跨任务学习**（本节主题） |

---

## 关键设计原则

1. **经验要具体，不要泛泛** ❌ "要仔细检查" ✅ "先调用 check_vip_status 再处理退款"
2. **TTL 强制过期** 6个月前的经验可能有害而无益
3. **反馈闭环** 经验用了没用？成功率追踪，差的经验自然降权
4. **检索上限 3-5 条** 太多经验反而干扰 LLM 注意力
5. **按任务类型分桶** 检索时加 task_type 过滤，精准度大幅提升

---

## 小结

```
没有经验循环的 Agent：
  第 1 次：学习曲线 ████████░░ 80分
  第 100 次：████████░░ 80分（没有成长）

有经验循环的 Agent：
  第 1 次：████████░░ 80分
  第 10 次：█████████░ 90分
  第 50 次：██████████ 98分（越用越好）
```

**三步落地**：
1. 任务结束时调 LLM 提炼 3-5 条经验（`ExperienceExtractor`）
2. Embedding 存入 SQLite / 向量 DB（`ExperienceStore.save`）
3. 下次任务开始时语义检索注入 prompt（`ExperienceStore.retrieve`）

Agent 从此有了**职业成长**。
