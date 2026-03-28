# 165 - Agent 跨会话知识传递与经验共享（Cross-Session Knowledge Transfer & Experience Sharing）

## 问题：Agent 失忆症

每次新会话，Agent 都是白板一张：

- 上次调试了 3 小时才搞定的 API 限流问题？不记得了
- 用户偏好的回复风格？不记得了
- 某个工具的隐藏坑（返回 null 要特殊处理）？不记得了

这不是模型能力问题，是**架构问题**。LLM 的 Context Window 是会话级的，会话结束，上下文清零。

解法不是让 LLM "更聪明"，而是**在会话之间主动传递知识**。

---

## 核心概念：Knowledge Capsule

把 Agent 在一次会话中积累的知识，打包成结构化的 **Knowledge Capsule**，在会话结束时持久化，下次会话启动时注入。

```
会话 N 结束
    ↓
Knowledge Extractor 提炼关键知识
    ↓
Knowledge Capsule 持久化到 DB/文件
    ↓
会话 N+1 启动
    ↓
Knowledge Injector 检索相关知识
    ↓
注入 System Prompt / 工具上下文
    ↓
Agent 带着"记忆"开始工作
```

---

## 三层知识分类

参考认知科学的记忆分类：

| 类型 | 内容 | 示例 | TTL |
|------|------|------|-----|
| **程序性（Procedural）** | 怎么做某事 | "调用 Stripe API 时先检查幂等键" | 长期 |
| **语义性（Semantic）** | 事实与规则 | "用户 ID 格式是 `usr_xxx`" | 长期 |
| **情节性（Episodic）** | 发生了什么 | "2024-03-15 修复了支付超时问题" | 短期 |

程序性和语义性是**高价值知识**，应长期保留；情节性更像日志，可以定期摘要压缩。

---

## TypeScript 实现

### Knowledge Capsule 数据结构

```typescript
interface KnowledgeItem {
  id: string;
  type: 'procedural' | 'semantic' | 'episodic';
  content: string;           // 自然语言描述
  tags: string[];            // 用于检索的标签
  source: string;            // 来源工具/场景
  confidence: number;        // 0-1，置信度
  usageCount: number;        // 被引用次数（用于排序）
  createdAt: number;
  expiresAt?: number;        // 情节性知识可设过期
  embedding?: number[];      // 向量，用于语义检索
}

interface KnowledgeCapsule {
  agentId: string;
  userId?: string;           // 可以是用户级或全局级
  items: KnowledgeItem[];
  version: number;
  lastUpdated: number;
}
```

### Knowledge Extractor：会话结束时提炼知识

```typescript
class KnowledgeExtractor {
  constructor(private llm: LLMClient, private store: KnowledgeStore) {}

  async extractFromSession(
    sessionId: string,
    toolCallHistory: ToolCallRecord[],
    conversationSummary: string
  ): Promise<KnowledgeItem[]> {
    
    // 1. 找到有价值的信号：错误修复、用户纠正、成功模式
    const signals = this.detectSignals(toolCallHistory);
    
    if (signals.length === 0) return [];

    // 2. 让 LLM 从这些信号中提炼知识
    const prompt = `
你是一个知识提炼专家。分析以下 Agent 会话信号，提炼出值得记住的知识。

会话摘要：
${conversationSummary}

关键信号：
${signals.map(s => `- [${s.type}] ${s.description}`).join('\n')}

请提炼知识，输出 JSON 数组，每条知识包含：
- type: "procedural" | "semantic" | "episodic"
- content: 知识内容（一句话，清晰具体）
- tags: 标签数组（用于检索）
- confidence: 置信度 0-1
- source: 来源描述

只提炼真正有用的知识，宁缺毋滥。`;

    const response = await this.llm.complete(prompt, { responseFormat: 'json' });
    const items = JSON.parse(response) as Omit<KnowledgeItem, 'id' | 'usageCount' | 'createdAt'>[];

    return items.map(item => ({
      ...item,
      id: crypto.randomUUID(),
      usageCount: 0,
      createdAt: Date.now(),
      // 情节性知识 30 天后过期
      expiresAt: item.type === 'episodic' ? Date.now() + 30 * 24 * 3600 * 1000 : undefined,
    }));
  }

  private detectSignals(history: ToolCallRecord[]): Signal[] {
    const signals: Signal[] = [];

    for (let i = 0; i < history.length; i++) {
      const record = history[i];

      // 信号1：工具调用后出错，然后被修复了
      if (record.error && history[i + 1]?.toolName === record.toolName && !history[i + 1]?.error) {
        signals.push({
          type: 'error_fix',
          description: `工具 ${record.toolName} 调用失败（${record.error}），下次调用成功，参数变化：${JSON.stringify(diff(record.params, history[i+1].params))}`,
        });
      }

      // 信号2：同一工具被调用多次但参数逐渐优化
      if (record.retryCount && record.retryCount > 2) {
        signals.push({
          type: 'retry_pattern',
          description: `工具 ${record.toolName} 需要多次重试才能成功，最终有效参数：${JSON.stringify(record.params)}`,
        });
      }

      // 信号3：响应时间异常（可能有性能知识）
      if (record.durationMs > 5000) {
        signals.push({
          type: 'slow_tool',
          description: `工具 ${record.toolName} 执行耗时 ${record.durationMs}ms，参数：${JSON.stringify(record.params)}`,
        });
      }
    }

    return signals;
  }
}
```

### Knowledge Injector：会话启动时注入知识

```typescript
class KnowledgeInjector {
  constructor(private store: KnowledgeStore, private embedder: EmbeddingClient) {}

  async buildContextBlock(
    agentId: string,
    userId: string | undefined,
    currentTask: string,
    maxTokens = 800
  ): Promise<string> {
    
    // 1. 语义检索相关知识
    const taskEmbedding = await this.embedder.embed(currentTask);
    const candidates = await this.store.semanticSearch(agentId, taskEmbedding, { limit: 20 });

    // 2. 过滤掉过期的情节性知识
    const now = Date.now();
    const valid = candidates.filter(item => !item.expiresAt || item.expiresAt > now);

    // 3. 按类型分组并排序（程序性 > 语义性 > 情节性；高置信度 + 高使用频率优先）
    const ranked = valid.sort((a, b) => {
      const typeWeight = { procedural: 3, semantic: 2, episodic: 1 };
      const scoreA = typeWeight[a.type] * 10 + a.confidence * 5 + Math.log(a.usageCount + 1);
      const scoreB = typeWeight[b.type] * 10 + b.confidence * 5 + Math.log(b.usageCount + 1);
      return scoreB - scoreA;
    });

    // 4. 按 Token 预算裁剪
    const selected: KnowledgeItem[] = [];
    let tokenCount = 0;
    for (const item of ranked) {
      const tokens = estimateTokens(item.content);
      if (tokenCount + tokens > maxTokens) break;
      selected.push(item);
      tokenCount += tokens;
    }

    if (selected.length === 0) return '';

    // 5. 格式化为 System Prompt 块
    const sections = {
      procedural: selected.filter(i => i.type === 'procedural'),
      semantic: selected.filter(i => i.type === 'semantic'),
      episodic: selected.filter(i => i.type === 'episodic'),
    };

    const lines: string[] = ['## 🧠 相关历史经验'];

    if (sections.procedural.length > 0) {
      lines.push('\n### 操作规程');
      sections.procedural.forEach(i => lines.push(`- ${i.content}`));
    }
    if (sections.semantic.length > 0) {
      lines.push('\n### 背景知识');
      sections.semantic.forEach(i => lines.push(`- ${i.content}`));
    }
    if (sections.episodic.length > 0) {
      lines.push('\n### 近期事件');
      sections.episodic.forEach(i => lines.push(`- ${i.content}`));
    }

    // 记录使用次数（正向反馈）
    await this.store.incrementUsage(selected.map(i => i.id));

    return lines.join('\n');
  }
}
```

### 集成到 Agent Loop

```typescript
class AgentWithMemory {
  private extractor: KnowledgeExtractor;
  private injector: KnowledgeInjector;
  private toolCallHistory: ToolCallRecord[] = [];

  async run(userMessage: string, agentId: string, userId?: string): Promise<string> {
    // 1. 会话启动：注入相关历史知识
    const knowledgeBlock = await this.injector.buildContextBlock(agentId, userId, userMessage);
    
    const systemPrompt = [
      this.baseSystemPrompt,
      knowledgeBlock,                     // 动态注入的历史经验
    ].filter(Boolean).join('\n\n');

    // 2. 正常 Agent Loop（记录工具调用历史）
    const result = await this.agentLoop(userMessage, systemPrompt);

    // 3. 会话结束：提炼知识并持久化
    if (this.toolCallHistory.length > 0) {
      const summary = await this.summarizeSession(userMessage, result);
      const newKnowledge = await this.extractor.extractFromSession(
        this.sessionId,
        this.toolCallHistory,
        summary
      );
      
      if (newKnowledge.length > 0) {
        await this.store.upsert(agentId, newKnowledge);
        console.log(`[Knowledge] 本次会话提炼 ${newKnowledge.length} 条新知识`);
      }
    }

    return result;
  }
}
```

---

## Python 版本

```python
import json
from dataclasses import dataclass, field
from typing import Optional
import time

@dataclass
class KnowledgeItem:
    id: str
    type: str  # procedural | semantic | episodic
    content: str
    tags: list[str]
    confidence: float
    source: str
    usage_count: int = 0
    created_at: float = field(default_factory=time.time)
    expires_at: Optional[float] = None

class KnowledgeStore:
    """简单文件系统实现，生产用 Redis + 向量DB"""
    
    def __init__(self, storage_path: str):
        self.path = storage_path
        self._cache: dict[str, list[KnowledgeItem]] = {}
    
    def save(self, agent_id: str, items: list[KnowledgeItem]):
        existing = self.load(agent_id)
        
        # 去重：相同 source 的程序性/语义性知识更新而不是追加
        merged = {item.id: item for item in existing}
        for item in items:
            # 简单去重：内容相似度（生产用向量相似度）
            duplicate = next(
                (v for v in merged.values() 
                 if v.type == item.type and v.source == item.source 
                 and self._similarity(v.content, item.content) > 0.85),
                None
            )
            if duplicate:
                # 更新置信度（加权平均）
                duplicate.confidence = (duplicate.confidence + item.confidence) / 2
                duplicate.usage_count += 1
            else:
                merged[item.id] = item
        
        self._write(agent_id, list(merged.values()))
    
    def retrieve(self, agent_id: str, query: str, limit: int = 10) -> list[KnowledgeItem]:
        items = self.load(agent_id)
        now = time.time()
        
        # 过滤过期
        valid = [i for i in items if not i.expires_at or i.expires_at > now]
        
        # 简单关键词匹配（生产用向量检索）
        scored = []
        query_words = set(query.lower().split())
        for item in valid:
            content_words = set(item.content.lower().split())
            overlap = len(query_words & content_words)
            type_bonus = {'procedural': 0.3, 'semantic': 0.2, 'episodic': 0.1}[item.type]
            score = overlap * item.confidence + type_bonus + item.usage_count * 0.05
            scored.append((score, item))
        
        scored.sort(reverse=True)
        return [item for _, item in scored[:limit]]
    
    def _similarity(self, a: str, b: str) -> float:
        """简单 Jaccard 相似度，生产用 embedding 余弦相似度"""
        a_words, b_words = set(a.lower().split()), set(b.lower().split())
        if not a_words or not b_words:
            return 0.0
        return len(a_words & b_words) / len(a_words | b_words)
```

---

## OpenClaw 的实践：MEMORY.md

OpenClaw 里的 `MEMORY.md` 就是此模式的最简落地：

```markdown
# MEMORY.md

## 操作规程（Procedural）
- 部署前必须先跑 `npm test`，有过两次因为跳过测试导致生产事故
- Cloudflare R2 上传时 Content-Type 必须显式设置，否则图片无法预览

## 背景知识（Semantic）
- 老板的 Telegram ID 是 67431246，时区 GMT+11
- mysterybox 生产数据库密码在 Bitwarden "Production" 文件夹

## 近期事件（Episodic）
- 2026-03-20：修复了 skinsmeme 的支付回调 HMAC 验签问题
- 2026-03-25：老板要求所有 Telegram 消息用中文回复
```

`AGENTS.md` 要求每次 Main Session 启动时读 `MEMORY.md`，这就是 **Knowledge Injection**。

Heartbeat 定期更新 `MEMORY.md`，这就是 **Knowledge Extraction**。

---

## 知识老化与清理

```typescript
class KnowledgeMaintainer {
  async prune(agentId: string): Promise<void> {
    const items = await this.store.load(agentId);
    const now = Date.now();
    
    const kept = items.filter(item => {
      // 删除过期情节性知识
      if (item.expiresAt && item.expiresAt < now) return false;
      
      // 删除长期未使用的低置信度知识（超过 90 天没被引用）
      const ageMs = now - item.createdAt;
      if (ageMs > 90 * 86400_000 && item.usageCount === 0 && item.confidence < 0.5) return false;
      
      return true;
    });
    
    // 情节性知识超过 50 条时，让 LLM 摘要压缩
    const episodic = kept.filter(i => i.type === 'episodic');
    if (episodic.length > 50) {
      const summary = await this.summarizeEpisodic(episodic);
      const pruned = kept.filter(i => i.type !== 'episodic');
      pruned.push(summary); // 替换为一条摘要
      await this.store.write(agentId, pruned);
    } else {
      await this.store.write(agentId, kept);
    }
  }
}
```

---

## 多 Agent 知识共享

多个 Agent 实例可以共享同一个 Knowledge Store，实现**经验传播**：

```typescript
// 全局知识库：所有 Agent 实例共享
const globalKnowledge = new KnowledgeStore('global');

// 实例级知识库：每个 Agent 自己的
const instanceKnowledge = new KnowledgeStore(`agent-${agentId}`);

// 注入时合并两层知识
const knowledge = [
  ...await globalKnowledge.retrieve(query),   // 全局经验
  ...await instanceKnowledge.retrieve(query), // 私有经验
].slice(0, 15); // 总量控制
```

这样当一个 Agent 实例发现了某个 API 的坑，修复后可以**发布到全局知识库**，所有实例立刻受益。

---

## 关键设计原则

1. **宁缺毋滥**：低质量知识比没有知识更糟糕（误导 LLM）。提炼时设置高门槛。
2. **置信度衰减**：随时间流逝，置信度应该小幅下降（世界在变化）。
3. **反向反馈**：如果某条知识导致了错误，降低其置信度，而不是直接删除。
4. **Token 预算优先**：注入时永远做 Token Budget 控制，不能因为知识多就撑爆 Context。
5. **隐私隔离**：用户级知识（偏好、个人信息）与系统级知识（工具规程、API 行为）严格分库。

---

## 总结

| 组件 | 职责 |
|------|------|
| `KnowledgeExtractor` | 会话结束后从 ToolCall 历史和对话中提炼知识 |
| `KnowledgeStore` | 持久化 + 去重 + 向量检索 |
| `KnowledgeInjector` | 会话启动时语义检索 + Token 预算控制注入 |
| `KnowledgeMaintainer` | 定期老化清理 + 情节性知识压缩摘要 |

**OpenClaw 现成实践**：`MEMORY.md` + Heartbeat 更新 + 每次主会话启动读取 = 手动版知识传递系统。

**生产进阶**：向量 DB（pgvector/Qdrant）+ 自动提炼 + 多实例共享知识库 = 真正的 Agent 长期记忆。
