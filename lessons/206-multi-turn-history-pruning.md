# 206 - Agent 多轮对话历史裁剪策略（Multi-Turn History Pruning）

> "Context window 是寸土寸金的地产，不是你的日记本。" —— 每个被 128K limit 卡住的人

---

## 问题：为什么需要主动裁剪？

Agent 做多轮任务时，消息历史会持续增长：

```
Turn 1: user + assistant + 3 tool calls = ~800 tokens
Turn 5: 上面 × 5 + 附加内容 = ~6000 tokens
Turn 20: 可能 30K+ tokens，接近限制
Turn 50: 💥 context overflow
```

**简单解法（不够好）：**
- 固定滑动窗口：丢掉最早的 N 条 → 可能丢关键指令
- 截断到 max_tokens：把后面砍掉 → 破坏工具调用配对
- 全部摘要压缩：每次都摘要 → 延迟高、细节丢失

**智能裁剪的目标：**
1. 保留系统指令（永不删）
2. 保留最近 K 轮（新鲜上下文）
3. 识别"重要消息"并钉住（工具结果、决策点）
4. 删除"噪声消息"（中间思考、重复确认）
5. 保持工具调用的 call/result 配对完整性

---

## 核心数据结构

```typescript
// 消息重要性评分
interface ScoredMessage {
  message: Message;
  score: number;       // 0-100，越高越重要
  pinned: boolean;     // true = 永不裁剪
  kind: 'system' | 'tool_call' | 'tool_result' | 'user' | 'assistant';
}

// 裁剪配置
interface PruningConfig {
  maxTokens: number;        // 目标 token 上限（如 100K）
  targetTokens: number;     // 裁剪后目标（如 80K，留余量）
  pinRecent: number;        // 最近 N 轮永远保留（如 4）
  minScoreToKeep: number;   // 低于此分数可被删除
}
```

---

## 实现：智能裁剪器

```typescript
// history-pruner.ts
import Anthropic from '@anthropic-ai/sdk';

type Message = Anthropic.MessageParam;

interface ScoredMessage {
  index: number;
  message: Message;
  score: number;
  pinned: boolean;
  tokenEstimate: number;
}

export class HistoryPruner {
  private config: {
    maxTokens: number;
    targetTokens: number;
    pinRecent: number;
    minScore: number;
  };

  constructor(config = {
    maxTokens: 100_000,
    targetTokens: 80_000,
    pinRecent: 4,   // 保留最近 4 条用户消息对应的轮次
    minScore: 30,
  }) {
    this.config = config;
  }

  // 粗略估算 token 数（实际用 tiktoken 更精准）
  private estimateTokens(msg: Message): number {
    const content = typeof msg.content === 'string'
      ? msg.content
      : JSON.stringify(msg.content);
    return Math.ceil(content.length / 4);
  }

  // 对消息打重要性分
  private scoreMessage(msg: Message, index: number, total: number): number {
    let score = 0;

    // 越新越重要（线性衰减）
    const recencyBonus = Math.floor((index / total) * 30); // 0-30
    score += recencyBonus;

    // 角色权重
    if (msg.role === 'user') {
      score += 25;  // 用户输入通常重要
    } else if (msg.role === 'assistant') {
      // 检查是否包含工具调用
      const hasToolUse = Array.isArray(msg.content)
        && msg.content.some(b => b.type === 'tool_use');
      score += hasToolUse ? 35 : 15;
    }

    // 工具结果：包含数据，重要
    if (Array.isArray(msg.content)) {
      const hasToolResult = msg.content.some(b => b.type === 'tool_result');
      if (hasToolResult) score += 30;
    }

    // 内容长度惩罚（超长且低信息密度的裁剪掉）
    const tokens = this.estimateTokens(msg);
    if (tokens > 2000) score -= 10; // 超长消息稍微降权

    return Math.min(100, Math.max(0, score));
  }

  // 找到工具调用的配对（call + result 必须成对删除）
  private findToolCallPairs(messages: Message[]): Map<number, number> {
    const pairs = new Map<number, number>(); // call_index -> result_index

    for (let i = 0; i < messages.length; i++) {
      const msg = messages[i];
      if (msg.role !== 'assistant' || !Array.isArray(msg.content)) continue;

      const toolUses = msg.content.filter(b => b.type === 'tool_use');
      if (toolUses.length === 0) continue;

      // 找下一条 user 消息中的 tool_result
      for (let j = i + 1; j < messages.length; j++) {
        const next = messages[j];
        if (next.role === 'user' && Array.isArray(next.content)) {
          const hasResult = next.content.some(b => b.type === 'tool_result');
          if (hasResult) {
            pairs.set(i, j);
            break;
          }
        }
      }
    }
    return pairs;
  }

  prune(messages: Message[]): Message[] {
    // 估算总 token
    let totalTokens = messages.reduce(
      (sum, m) => sum + this.estimateTokens(m), 0
    );

    // 不需要裁剪
    if (totalTokens <= this.config.maxTokens) return messages;

    console.log(`[Pruner] 总 token: ${totalTokens}，开始裁剪...`);

    const toolPairs = this.findToolCallPairs(messages);

    // 计算每条消息的分数和是否可删
    const scored: ScoredMessage[] = messages.map((msg, i) => {
      // 系统消息永远钉住
      if (msg.role === 'system') {
        return { index: i, message: msg, score: 100, pinned: true,
          tokenEstimate: this.estimateTokens(msg) };
      }

      // 最近 N 条用户消息及其对应轮次钉住
      const recentUserMessages = messages
        .map((m, idx) => ({ m, idx }))
        .filter(({ m }) => m.role === 'user')
        .slice(-this.config.pinRecent);
      const recentIndices = new Set(recentUserMessages.map(x => x.idx));
      // 也钉住这些用户消息前后的 assistant 回复
      for (const { idx } of recentUserMessages) {
        if (idx + 1 < messages.length) recentIndices.add(idx + 1);
        if (idx - 1 >= 0) recentIndices.add(idx - 1);
      }

      const pinned = recentIndices.has(i);
      const score = this.scoreMessage(msg, i, messages.length);

      return { index: i, message: msg, score, pinned,
        tokenEstimate: this.estimateTokens(msg) };
    });

    // 工具调用配对：如果 call 要删则 result 也删，反之亦然
    for (const [callIdx, resultIdx] of toolPairs) {
      const callScored = scored[callIdx];
      const resultScored = scored[resultIdx];
      // 取两者中较低分（保守策略：只要有一个要保留就都保留）
      const minScore = Math.min(callScored.score, resultScored.score);
      callScored.score = minScore;
      resultScored.score = minScore;
      // 如果任意一个被钉住，两个都钉住
      if (callScored.pinned || resultScored.pinned) {
        callScored.pinned = true;
        resultScored.pinned = true;
      }
    }

    // 按分数升序排列（低分优先删除）
    const candidates = scored
      .filter(s => !s.pinned && s.score < this.config.minScore)
      .sort((a, b) => a.score - b.score);

    const deleted = new Set<number>();

    for (const candidate of candidates) {
      if (totalTokens <= this.config.targetTokens) break;

      deleted.add(candidate.index);
      totalTokens -= candidate.tokenEstimate;

      // 如果是工具配对的一部分，同时删掉配对
      for (const [callIdx, resultIdx] of toolPairs) {
        if (candidate.index === callIdx && !scored[resultIdx].pinned) {
          deleted.add(resultIdx);
          totalTokens -= scored[resultIdx].tokenEstimate;
        } else if (candidate.index === resultIdx && !scored[callIdx].pinned) {
          deleted.add(callIdx);
          totalTokens -= scored[callIdx].tokenEstimate;
        }
      }
    }

    const result = messages.filter((_, i) => !deleted.has(i));
    console.log(`[Pruner] 删除 ${deleted.size} 条，剩余 token ≈ ${totalTokens}`);
    return result;
  }
}
```

---

## 集成到 Agent Loop

```typescript
// agent-loop-with-pruning.ts
import Anthropic from '@anthropic-ai/sdk';
import { HistoryPruner } from './history-pruner';

const client = new Anthropic();
const pruner = new HistoryPruner({
  maxTokens: 120_000,   // 触发裁剪阈值（留 8K 给输出）
  targetTokens: 90_000, // 裁剪后目标
  pinRecent: 5,         // 保留最近 5 轮
  minScore: 35,
});

async function agentLoop(
  systemPrompt: string,
  userMessage: string,
  history: Anthropic.MessageParam[] = []
) {
  history.push({ role: 'user', content: userMessage });

  while (true) {
    // 🔑 每轮调用前裁剪历史
    const prunedHistory = pruner.prune(history);

    const response = await client.messages.create({
      model: 'claude-opus-4-5',
      max_tokens: 8192,
      system: systemPrompt,
      messages: prunedHistory,
      tools: [/* your tools */],
    });

    const assistantMsg: Anthropic.MessageParam = {
      role: 'assistant',
      content: response.content,
    };
    history.push(assistantMsg);

    if (response.stop_reason === 'end_turn') {
      return { response, history };
    }

    if (response.stop_reason === 'tool_use') {
      const toolResults = await executeTools(response.content);
      history.push({
        role: 'user',
        content: toolResults,
      });
    }
  }
}

async function executeTools(
  content: Anthropic.ContentBlock[]
): Promise<Anthropic.ToolResultBlockParam[]> {
  const results: Anthropic.ToolResultBlockParam[] = [];
  for (const block of content) {
    if (block.type !== 'tool_use') continue;
    // 执行工具...
    results.push({
      type: 'tool_result',
      tool_use_id: block.id,
      content: `result for ${block.name}`,
    });
  }
  return results;
}
```

---

## OpenClaw 实战：会话历史自动裁剪

OpenClaw 的每个 session 都有消息历史。可以在 heartbeat 或长任务前主动检查并裁剪：

```typescript
// session-pruning-middleware.ts

// OpenClaw 的 session history 通过 sessions_history 工具获取
// 裁剪逻辑在发送前注入

export function createSessionPruner(maxContextTokens = 100_000) {
  const pruner = new HistoryPruner({
    maxTokens: maxContextTokens,
    targetTokens: Math.floor(maxContextTokens * 0.75),
    pinRecent: 6,
    minScore: 40,
  });

  return {
    // 在 agent turn 开始前裁剪传入的历史
    beforeTurn(messages: Anthropic.MessageParam[]): Anthropic.MessageParam[] {
      const before = messages.reduce(
        (s, m) => s + JSON.stringify(m.content).length / 4, 0
      );
      const pruned = pruner.prune(messages);
      const after = pruned.reduce(
        (s, m) => s + JSON.stringify(m.content).length / 4, 0
      );

      if (pruned.length < messages.length) {
        console.log(
          `[SessionPruner] ${messages.length}→${pruned.length} 条，` +
          `token ${Math.round(before)}→${Math.round(after)} ` +
          `(节省 ${Math.round((1 - after/before) * 100)}%)`
        );
      }
      return pruned;
    }
  };
}
```

---

## Python 版本（pi-mono 风格）

```python
# history_pruner.py
from dataclasses import dataclass, field
from typing import Any
import json

@dataclass
class PruningConfig:
    max_tokens: int = 100_000
    target_tokens: int = 80_000
    pin_recent: int = 4
    min_score: int = 30

@dataclass 
class ScoredMessage:
    index: int
    message: dict
    score: float
    pinned: bool
    token_estimate: int

def estimate_tokens(msg: dict) -> int:
    content = msg.get('content', '')
    text = content if isinstance(content, str) else json.dumps(content)
    return max(1, len(text) // 4)

def score_message(msg: dict, index: int, total: int) -> float:
    score = 0.0
    score += (index / total) * 30  # 新近度
    
    role = msg.get('role', '')
    if role == 'user':
        score += 25
    elif role == 'assistant':
        content = msg.get('content', [])
        has_tool_use = isinstance(content, list) and any(
            b.get('type') == 'tool_use' for b in content
        )
        score += 35 if has_tool_use else 15
    
    content = msg.get('content', [])
    if isinstance(content, list):
        has_tool_result = any(b.get('type') == 'tool_result' for b in content)
        if has_tool_result:
            score += 30
    
    tokens = estimate_tokens(msg)
    if tokens > 2000:
        score -= 10
    
    return min(100, max(0, score))

def prune_history(
    messages: list[dict],
    config: PruningConfig = PruningConfig()
) -> list[dict]:
    total_tokens = sum(estimate_tokens(m) for m in messages)
    
    if total_tokens <= config.max_tokens:
        return messages
    
    # 找出最近 pin_recent 轮的索引
    user_indices = [i for i, m in enumerate(messages) if m.get('role') == 'user']
    recent_user = set(user_indices[-config.pin_recent:])
    pinned_indices: set[int] = set()
    for idx in recent_user:
        pinned_indices.add(idx)
        if idx + 1 < len(messages):
            pinned_indices.add(idx + 1)
    
    scored = []
    for i, msg in enumerate(messages):
        is_system = msg.get('role') == 'system'
        pinned = is_system or i in pinned_indices
        score = score_message(msg, i, len(messages))
        scored.append(ScoredMessage(
            index=i, message=msg, score=score,
            pinned=pinned, token_estimate=estimate_tokens(msg)
        ))
    
    # 低分非钉住消息优先删除
    candidates = sorted(
        [s for s in scored if not s.pinned and s.score < config.min_score],
        key=lambda s: s.score
    )
    
    deleted: set[int] = set()
    for candidate in candidates:
        if total_tokens <= config.target_tokens:
            break
        deleted.add(candidate.index)
        total_tokens -= candidate.token_estimate
    
    result = [m.message for m in scored if m.index not in deleted]
    print(f"[Pruner] {len(messages)}→{len(result)} 条, token ≈ {total_tokens}")
    return result
```

---

## 裁剪策略对比

| 策略 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| 固定滑动窗口 | 简单、可预测 | 丢失早期重要指令 | 简单聊天 |
| 全量摘要压缩 | 压缩率高 | 延迟大、细节丢失 | 超长对话归档 |
| 重要性评分裁剪 | 保留关键信息 | 评分需要调优 | **多工具 Agent** |
| 分层存储（冷热）| 可无限扩展 | 需要外部存储 | 企业级长会话 |

---

## 关键细节：工具调用配对

这是最容易出 bug 的地方 ——

```
❌ 错误：删掉 tool_use 但保留 tool_result
messages = [
  { role: 'assistant', content: [{ type: 'tool_use', id: 'tu_1' }] },  // 被删
  { role: 'user', content: [{ type: 'tool_result', tool_use_id: 'tu_1' }] },  // 保留
]
→ API 报错：tool_result 找不到对应的 tool_use
```

```typescript
// ✅ 正确：配对原子删除
// 只要 call 或 result 其中一个被钉住，两个都保留
// 删除时必须同时删除配对
if (deleteCall) deleteResult();
if (deleteResult) deleteCall();
```

---

## 实战调参建议

```typescript
// 代码执行 Agent（工具结果很重要）
const codeAgentPruner = new HistoryPruner({
  maxTokens: 150_000,
  targetTokens: 100_000,
  pinRecent: 3,       // 最近 3 轮
  minScore: 50,       // 提高阈值，保留更多
});

// 对话 Agent（工具少，对话流更重要）
const chatAgentPruner = new HistoryPruner({
  maxTokens: 80_000,
  targetTokens: 60_000,
  pinRecent: 8,       // 保留更多最近对话
  minScore: 20,       // 降低阈值，裁剪更激进
});

// OpenClaw Heartbeat Agent（长期运行，历史积累快）
const heartbeatPruner = new HistoryPruner({
  maxTokens: 50_000,
  targetTokens: 30_000,
  pinRecent: 2,       // 只要最近 2 轮
  minScore: 60,       // 激进裁剪
});
```

---

## 总结

```
智能裁剪 = 重要性评分 + 工具配对保护 + 最近轮次锁定

核心原则：
1. 系统提示词永不删
2. 工具 call/result 必须同生共死
3. 最近 N 轮永远保留（新鲜上下文）
4. 按分数从低到高删，删到目标 token 为止
5. 宁可少删，不要破坏消息对话结构

效果：128K context → 稳定运行，不 overflow，关键信息不丢失
```

---

*下一课预告：Agent 懒执行与延迟工具绑定（Lazy Execution & Deferred Tool Binding）— 工具不一定要立刻执行，延迟到真正需要时再绑定，节省成本*
