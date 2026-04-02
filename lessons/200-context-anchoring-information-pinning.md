# Lesson 200: Agent 上下文锚点与关键信息钉

> **核心问题**：长任务中，LLM 对早期关键信息的注意力会"衰减"——用户在第 1 轮说的约束条件，到第 30 轮可能已经被模型"忘掉"了。Context Anchoring 就是解决这个问题的工程方案。

---

## 为什么需要 Context Anchoring？

LLM 并不是真的"忘记"了，token 还在 context window 里。真正的问题是：

1. **注意力稀释**：随着对话增长，早期 token 的注意力权重被均摊，模型不再"重视"它们
2. **位置偏差**：大多数模型对靠近末尾的内容更敏感（recency bias）
3. **关键约束漂移**：用户说"预算不超过 $100"，20 轮后模型推荐了 $200 的方案

```
轮次 1:  [关键约束] [...]
轮次 10: [关键约束] [......大量中间对话......]
轮次 30: [关键约束] [..........更多内容..........] ← 模型不再重视开头
```

---

## 核心设计：ContextAnchor 系统

### 1. 数据结构定义

```typescript
// TypeScript 实现
interface ContextAnchor {
  id: string;
  content: string;           // 锚点内容
  priority: 'critical' | 'high' | 'medium';
  ttl?: number;              // 生命周期（轮次），undefined = 永久
  reinjectEvery: number;     // 每隔多少轮重新注入
  category: 'constraint' | 'user-pref' | 'task-goal' | 'state';
  createdAt: number;         // 创建轮次
  lastInjected: number;      // 上次注入轮次
}

interface AnchorManager {
  anchors: Map<string, ContextAnchor>;
  currentTurn: number;
}
```

### 2. 锚点提取器（自动识别关键信息）

```typescript
async function extractAnchors(
  userMessage: string, 
  llm: LLMClient
): Promise<ContextAnchor[]> {
  const response = await llm.complete({
    model: 'claude-haiku-4-5',  // 用便宜模型做提取
    system: `从用户消息中提取需要"钉住"的关键信息。
返回 JSON 数组，每项包含:
- content: 需要钉住的内容（简洁重述）
- priority: critical/high/medium
- category: constraint/user-pref/task-goal/state
- reinjectEvery: 建议每隔几轮重新强调（3-10）

只提取真正重要的约束、偏好、目标。普通对话内容不需要钉住。`,
    messages: [{ role: 'user', content: userMessage }]
  });

  try {
    const items = JSON.parse(response.content);
    return items.map((item: any) => ({
      id: crypto.randomUUID(),
      ...item,
      createdAt: 0,
      lastInjected: 0
    }));
  } catch {
    return [];
  }
}

// 使用示例
const anchors = await extractAnchors(
  "帮我规划旅行，预算 $500，不要飞机，必须宠物友好", 
  llm
);
// 输出:
// [
//   { content: "预算上限 $500", priority: "critical", category: "constraint", reinjectEvery: 3 },
//   { content: "不使用飞机出行", priority: "critical", category: "constraint", reinjectEvery: 5 },
//   { content: "目的地/住宿必须宠物友好", priority: "high", category: "constraint", reinjectEvery: 5 }
// ]
```

### 3. 锚点注入中间件

```typescript
class ContextAnchorMiddleware {
  private manager: AnchorManager = {
    anchors: new Map(),
    currentTurn: 0
  };

  // 处理用户消息：提取新锚点
  async onUserMessage(message: string, llm: LLMClient): Promise<void> {
    this.manager.currentTurn++;
    
    // 自动提取锚点
    const newAnchors = await extractAnchors(message, llm);
    for (const anchor of newAnchors) {
      anchor.createdAt = this.manager.currentTurn;
      anchor.lastInjected = this.manager.currentTurn;
      this.manager.anchors.set(anchor.id, anchor);
    }
    
    // 清理过期锚点
    this.pruneExpiredAnchors();
  }

  // 构建需要注入的锚点 prompt
  buildAnchorInjection(): string {
    const toInject = this.getAnchorsToReinject();
    if (toInject.length === 0) return '';

    const critical = toInject.filter(a => a.priority === 'critical');
    const high = toInject.filter(a => a.priority === 'high');
    const medium = toInject.filter(a => a.priority === 'medium');

    let injection = '\n\n---\n⚠️ **关键约束提醒**（请严格遵守）：\n';
    
    if (critical.length > 0) {
      injection += critical.map(a => `🔴 ${a.content}`).join('\n') + '\n';
    }
    if (high.length > 0) {
      injection += high.map(a => `🟡 ${a.content}`).join('\n') + '\n';
    }
    if (medium.length > 0) {
      injection += medium.map(a => `⚪ ${a.content}`).join('\n') + '\n';
    }
    
    injection += '---\n';
    
    // 更新 lastInjected
    for (const anchor of toInject) {
      anchor.lastInjected = this.manager.currentTurn;
    }
    
    return injection;
  }

  // 决定哪些锚点需要重新注入
  private getAnchorsToReinject(): ContextAnchor[] {
    const current = this.manager.currentTurn;
    return Array.from(this.manager.anchors.values())
      .filter(anchor => {
        const turnsSinceInject = current - anchor.lastInjected;
        return turnsSinceInject >= anchor.reinjectEvery;
      })
      .sort((a, b) => {
        // critical > high > medium，然后按最久未注入排序
        const priorityOrder = { critical: 0, high: 1, medium: 2 };
        const pDiff = priorityOrder[a.priority] - priorityOrder[b.priority];
        if (pDiff !== 0) return pDiff;
        return (a.lastInjected - b.lastInjected); // 最久未注入的优先
      });
  }

  private pruneExpiredAnchors(): void {
    for (const [id, anchor] of this.manager.anchors) {
      if (anchor.ttl && 
          this.manager.currentTurn - anchor.createdAt > anchor.ttl) {
        this.manager.anchors.delete(id);
      }
    }
  }

  // 手动添加锚点（API）
  addAnchor(anchor: Omit<ContextAnchor, 'id' | 'createdAt' | 'lastInjected'>): string {
    const id = crypto.randomUUID();
    this.manager.anchors.set(id, {
      ...anchor,
      id,
      createdAt: this.manager.currentTurn,
      lastInjected: this.manager.currentTurn
    });
    return id;
  }

  // 删除锚点
  removeAnchor(id: string): void {
    this.manager.anchors.delete(id);
  }
}
```

### 4. 注入到 System Prompt

```typescript
class Agent {
  private anchorMiddleware = new ContextAnchorMiddleware();
  private baseSystemPrompt: string;

  constructor(systemPrompt: string, private llm: LLMClient) {
    this.baseSystemPrompt = systemPrompt;
  }

  async chat(userMessage: string): Promise<string> {
    // Step 1: 提取新锚点
    await this.anchorMiddleware.onUserMessage(userMessage, this.llm);
    
    // Step 2: 构建带锚点的 system prompt
    const anchorInjection = this.anchorMiddleware.buildAnchorInjection();
    const systemPrompt = anchorInjection 
      ? `${this.baseSystemPrompt}${anchorInjection}`
      : this.baseSystemPrompt;
    
    // Step 3: 调用 LLM
    const response = await this.llm.complete({
      system: systemPrompt,
      messages: this.history,
      model: 'claude-sonnet-4-5'
    });
    
    this.history.push(
      { role: 'user', content: userMessage },
      { role: 'assistant', content: response.content }
    );
    
    return response.content;
  }

  private history: Array<{ role: string; content: string }> = [];
}
```

---

## 策略进阶：位置感知注入

研究表明，LLM 对 **开头** 和 **结尾** 的注意力最高（"lost in the middle" 问题）。

```typescript
type InjectionPosition = 'system-top' | 'system-bottom' | 'user-prefix' | 'user-suffix';

interface AnchorInjectionStrategy {
  position: InjectionPosition;
  formatStyle: 'xml-tag' | 'markdown' | 'plain' | 'structured';
}

// 对于 Claude，XML 标签效果最好
function formatAnchorsAsXML(anchors: ContextAnchor[]): string {
  return `<critical_constraints>
${anchors.map(a => `  <constraint priority="${a.priority}">${a.content}</constraint>`).join('\n')}
</critical_constraints>`;
}

// 注入到 user 消息末尾（效果通常优于注入 system）
function injectToUserMessage(
  userMessage: string, 
  anchors: ContextAnchor[]
): string {
  if (anchors.length === 0) return userMessage;
  
  const critical = anchors.filter(a => a.priority === 'critical');
  if (critical.length === 0) return userMessage;
  
  return `${userMessage}

<reminders>
${critical.map(a => `- ${a.content}`).join('\n')}
</reminders>`;
}
```

---

## 实战：在 OpenClaw 中的应用

OpenClaw 的 SOUL.md 就是一种静态锚点系统——它把角色定义、权限规则"钉"在每次调用的 system prompt 里。

动态锚点则适用于跨 cron 任务的持久约束：

```typescript
// OpenClaw Cron 任务中维护跨轮次锚点
// 场景：用户设置了一个长期运行的监控任务

// 任务开始时（用户指令）
anchorMiddleware.addAnchor({
  content: '只监控错误率 > 5% 的服务，低于此阈值不发送告警',
  priority: 'critical',
  category: 'constraint',
  reinjectEvery: 1,  // 每轮都注入（因为这是告警过滤条件）
  ttl: undefined     // 永久有效
});

anchorMiddleware.addAnchor({
  content: '使用中文发送所有告警通知',
  priority: 'high', 
  category: 'user-pref',
  reinjectEvery: 5
});
```

---

## Python 版本

```python
from dataclasses import dataclass, field
from typing import Optional, Literal
import uuid
import json

@dataclass
class ContextAnchor:
    content: str
    priority: Literal['critical', 'high', 'medium']
    category: Literal['constraint', 'user-pref', 'task-goal', 'state']
    reinject_every: int
    id: str = field(default_factory=lambda: str(uuid.uuid4()))
    ttl: Optional[int] = None
    created_at: int = 0
    last_injected: int = 0


class ContextAnchorMiddleware:
    def __init__(self):
        self.anchors: dict[str, ContextAnchor] = {}
        self.current_turn = 0
    
    def add_anchor(self, anchor: ContextAnchor) -> str:
        anchor.created_at = self.current_turn
        anchor.last_injected = self.current_turn
        self.anchors[anchor.id] = anchor
        return anchor.id
    
    def build_system_suffix(self) -> str:
        to_inject = [
            a for a in self.anchors.values()
            if self.current_turn - a.last_injected >= a.reinject_every
        ]
        
        if not to_inject:
            return ""
        
        # 更新注入时间
        for anchor in to_inject:
            anchor.last_injected = self.current_turn
        
        critical = [a for a in to_inject if a.priority == 'critical']
        
        if not critical:
            return ""
        
        lines = ["", "<active_constraints>"]
        for a in critical:
            lines.append(f"  <constraint>{a.content}</constraint>")
        lines.append("</active_constraints>")
        
        return "\n".join(lines)
    
    def tick(self):
        """每轮调用前执行"""
        self.current_turn += 1
        # 清理过期
        to_remove = [
            id for id, a in self.anchors.items()
            if a.ttl and self.current_turn - a.created_at > a.ttl
        ]
        for id in to_remove:
            del self.anchors[id]
```

---

## 效果对比

| 场景 | 无锚点 | 有锚点 |
|------|--------|--------|
| 30 轮对话后遗忘约束 | 常见（~40% 违反） | 极少（< 5% 违反） |
| Token 开销 | 无额外开销 | +50~200 token/轮（可控） |
| 实现复杂度 | - | 低（中间件模式） |

---

## 关键调优参数

```typescript
const ANCHOR_CONFIG = {
  // 多少轮强制全量重注入（保底）
  forceReinjectEvery: 10,
  
  // critical 锚点最大数量（避免 prompt 膨胀）
  maxCriticalAnchors: 5,
  
  // 自动提取的置信度阈值（低于此值不创建锚点）
  extractionConfidenceThreshold: 0.8,
  
  // 注入格式（claude 用 xml，gpt 用 markdown）
  format: 'xml' as 'xml' | 'markdown'
};
```

---

## 总结

| 概念 | 说明 |
|------|------|
| **ContextAnchor** | 需要持续保持显著性的关键信息 |
| **reinjectEvery** | 每隔 N 轮重新注入到 system prompt |
| **自动提取** | 用 Haiku 从用户消息中识别需要"钉住"的信息 |
| **位置策略** | 注入到 user 消息末尾通常优于注入 system 中部 |
| **TTL** | 临时约束自动过期，避免锚点堆积 |

> **一句话记住**：Context Anchoring 就是给 LLM 贴便利贴——把真正重要的事情反复提醒，直到任务完成。
