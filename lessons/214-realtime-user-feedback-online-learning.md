# 214 - Agent 实时用户反馈与在线学习（Real-time User Feedback & Online Learning）

> 好的 Agent 不只是执行任务，它在每次交互中都在悄悄变得更好。

---

## 为什么需要实时反馈？

大多数 Agent 是单向的：给输入，出输出，然后就完了。但用户的反馈信号无处不在：

- 一个 👎 反应
- 一句「不对，重新生成」
- 连续第三次问同样的问题
- 直接修改 Agent 的输出再发出去

这些信号是黄金。不收集 = 白白浪费。

**在线学习（Online Learning）** 的核心思路：不需要重训练模型，在运行时根据用户反馈实时调整 Agent 行为。

---

## 反馈信号分类

```
显式反馈（Explicit）                  隐式反馈（Implicit）
─────────────────────────            ─────────────────────────
👍 点赞                               重复发送相似问题（没满足需求）
👎 点踩                               极短回复（"好"、"嗯"）
「重新生成」                           快速追问（首条回复不够用）
直接纠正（「不是这个意思」）             用户自己编辑内容后使用
明确评分（1-5 星）                      对话中途放弃任务
```

---

## 架构设计

```
用户消息 ──► Agent Loop ──► 响应输出
                │                │
                │                ▼
                │         FeedbackCapture
                │                │
                │         ┌──────▼──────┐
                │         │ FeedbackBuf │ (内存 + Redis)
                │         └──────┬──────┘
                │                │
                ▼                ▼
         FeedbackMiddleware  FeedbackAggregator
         (注入历史偏好)        (阈值触发更新)
                │                │
                ▼                ▼
         System Prompt     PromptVariantStore
         动态调整           (A/B 结果持久化)
```

---

## TypeScript 实现

### 1. 反馈数据结构

```typescript
// feedback-types.ts
export type FeedbackSignal = 
  | { type: 'thumbs_up';   messageId: string; sessionId: string }
  | { type: 'thumbs_down'; messageId: string; sessionId: string }
  | { type: 'regenerate';  messageId: string; sessionId: string; reason?: string }
  | { type: 'correction';  messageId: string; sessionId: string; original: string; corrected: string }
  | { type: 'implicit_retry'; sessionId: string; query: string }

export interface FeedbackEntry {
  signal: FeedbackSignal;
  timestamp: number;
  promptSnapshot: string;    // 当时使用的 system prompt hash
  responseSnapshot: string;  // 对应的 response 摘要
}

export interface SessionFeedbackProfile {
  sessionId: string;
  positiveCount: number;
  negativeCount: number;
  corrections: Array<{ original: string; corrected: string }>;
  preferredStyle: 'verbose' | 'concise' | 'unknown';
  lastUpdated: number;
}
```

### 2. 反馈捕获器

```typescript
// feedback-capture.ts
import Anthropic from '@anthropic-ai/sdk';

export class FeedbackCapture {
  private buffer: Map<string, FeedbackEntry[]> = new Map();
  private readonly BUFFER_TTL = 24 * 60 * 60 * 1000; // 24h

  // 记录显式反馈
  record(signal: FeedbackSignal): void {
    const entry: FeedbackEntry = {
      signal,
      timestamp: Date.now(),
      promptSnapshot: '',   // 由调用方填入
      responseSnapshot: '', // 由调用方填入
    };

    const key = signal.sessionId;
    if (!this.buffer.has(key)) this.buffer.set(key, []);
    this.buffer.get(key)!.push(entry);

    // 窗口内超过 10 条 thumbs_down → 立即触发调整
    const recentNegative = this.buffer.get(key)!
      .filter(e => e.signal.type === 'thumbs_down')
      .filter(e => Date.now() - e.timestamp < 30 * 60 * 1000);
    
    if (recentNegative.length >= 3) {
      this.emit('quality_degradation', key);
    }
  }

  // 捕获隐式反馈：检测重复查询
  detectImplicitRetry(sessionId: string, messages: Anthropic.MessageParam[]): boolean {
    const userMessages = messages
      .filter(m => m.role === 'user')
      .slice(-6) // 最近 6 条
      .map(m => (typeof m.content === 'string' ? m.content : ''));

    if (userMessages.length < 2) return false;

    const latest = userMessages[userMessages.length - 1];
    const prev = userMessages.slice(0, -1);
    
    // 简单相似度检测：词重叠 > 60%
    return prev.some(msg => this.jaccardSimilarity(latest, msg) > 0.6);
  }

  private jaccardSimilarity(a: string, b: string): number {
    const setA = new Set(a.toLowerCase().split(/\s+/));
    const setB = new Set(b.toLowerCase().split(/\s+/));
    const intersection = new Set([...setA].filter(x => setB.has(x)));
    const union = new Set([...setA, ...setB]);
    return intersection.size / union.size;
  }

  getSessionFeedback(sessionId: string): FeedbackEntry[] {
    return (this.buffer.get(sessionId) || [])
      .filter(e => Date.now() - e.timestamp < this.BUFFER_TTL);
  }

  private emit(event: string, sessionId: string): void {
    console.warn(`[FeedbackCapture] Event: ${event} for session ${sessionId}`);
  }
}
```

### 3. 反馈驱动的 Prompt 调整

```typescript
// feedback-middleware.ts
import Anthropic from '@anthropic-ai/sdk';
import { FeedbackCapture, SessionFeedbackProfile } from './feedback-types';

export class FeedbackMiddleware {
  private client: Anthropic;
  private capture: FeedbackCapture;
  private profiles: Map<string, SessionFeedbackProfile> = new Map();

  constructor(capture: FeedbackCapture) {
    this.client = new Anthropic();
    this.capture = capture;
  }

  // 根据反馈历史构建个性化 system prompt 片段
  buildFeedbackAwarePrompt(sessionId: string, basePrompt: string): string {
    const profile = this.buildProfile(sessionId);
    if (!profile) return basePrompt;

    const adjustments: string[] = [];

    // 负面反馈 > 正面 → 说明风格需要调整
    if (profile.negativeCount > profile.positiveCount + 2) {
      adjustments.push('用户对最近几条回复不满意，请更加简洁直接，避免废话。');
    }

    // 有纠正记录 → 提取用户偏好
    if (profile.corrections.length > 0) {
      const correctionHints = profile.corrections
        .slice(-3) // 最近 3 条纠正
        .map(c => `用户将"${c.original.slice(0, 50)}"改为"${c.corrected.slice(0, 50)}"`)
        .join('\n');
      adjustments.push(`用户曾做出如下纠正，注意学习：\n${correctionHints}`);
    }

    // 偏好简洁 → 压缩输出
    if (profile.preferredStyle === 'concise') {
      adjustments.push('用户偏好简洁回答，直接给结论，不要列举背景知识。');
    }

    if (adjustments.length === 0) return basePrompt;

    return `${basePrompt}

<user_feedback_context>
${adjustments.join('\n')}
</user_feedback_context>`;
  }

  // 从反馈记录推断用户画像
  private buildProfile(sessionId: string): SessionFeedbackProfile | null {
    const entries = this.capture.getSessionFeedback(sessionId);
    if (entries.length === 0) return null;

    const positive = entries.filter(e => e.signal.type === 'thumbs_up').length;
    const negative = entries.filter(e => e.signal.type === 'thumbs_down').length;
    const corrections = entries
      .filter(e => e.signal.type === 'correction')
      .map(e => {
        const s = e.signal as Extract<typeof e.signal, { type: 'correction' }>;
        return { original: s.original, corrected: s.corrected };
      });

    // 判断偏好风格：纠正记录中更短的内容 → 偏好简洁
    const conciseCorrections = corrections.filter(
      c => c.corrected.length < c.original.length * 0.7
    );
    const preferredStyle = conciseCorrections.length > corrections.length / 2
      ? 'concise'
      : 'unknown';

    return {
      sessionId,
      positiveCount: positive,
      negativeCount: negative,
      corrections,
      preferredStyle,
      lastUpdated: Date.now(),
    };
  }

  // 完整的反馈感知 Agent 调用
  async callWithFeedback(
    sessionId: string,
    baseSystemPrompt: string,
    messages: Anthropic.MessageParam[]
  ): Promise<string> {
    // 1. 检测隐式重试
    if (this.capture.detectImplicitRetry(sessionId, messages)) {
      this.capture.record({ type: 'implicit_retry', sessionId, query: '' });
      console.log(`[FeedbackMiddleware] Implicit retry detected for ${sessionId}`);
    }

    // 2. 注入反馈感知的 system prompt
    const adjustedPrompt = this.buildFeedbackAwarePrompt(sessionId, baseSystemPrompt);

    // 3. 调用 LLM
    const response = await this.client.messages.create({
      model: 'claude-haiku-4-5',
      max_tokens: 1024,
      system: adjustedPrompt,
      messages,
    });

    return response.content[0].type === 'text' ? response.content[0].text : '';
  }
}
```

### 4. 跨会话反馈持久化（Redis）

```typescript
// feedback-store.ts
import { createClient } from 'redis';
import { SessionFeedbackProfile } from './feedback-types';

export class FeedbackStore {
  private redis = createClient({ url: process.env.REDIS_URL });
  private readonly KEY_PREFIX = 'agent:feedback:';
  private readonly TTL = 7 * 24 * 3600; // 7天

  async saveProfile(profile: SessionFeedbackProfile): Promise<void> {
    await this.redis.setEx(
      `${this.KEY_PREFIX}${profile.sessionId}`,
      this.TTL,
      JSON.stringify(profile)
    );
  }

  async loadProfile(sessionId: string): Promise<SessionFeedbackProfile | null> {
    const raw = await this.redis.get(`${this.KEY_PREFIX}${sessionId}`);
    return raw ? JSON.parse(raw) : null;
  }

  // 统计全局反馈质量（运营指标）
  async getGlobalStats(): Promise<{ avgSatisfaction: number; totalSessions: number }> {
    const keys = await this.redis.keys(`${this.KEY_PREFIX}*`);
    if (keys.length === 0) return { avgSatisfaction: 0, totalSessions: 0 };

    let totalPositive = 0;
    let totalNegative = 0;

    for (const key of keys) {
      const raw = await this.redis.get(key);
      if (!raw) continue;
      const profile: SessionFeedbackProfile = JSON.parse(raw);
      totalPositive += profile.positiveCount;
      totalNegative += profile.negativeCount;
    }

    const total = totalPositive + totalNegative;
    return {
      avgSatisfaction: total > 0 ? totalPositive / total : 0,
      totalSessions: keys.length,
    };
  }
}
```

---

## Python 实现

```python
# feedback_system.py
from dataclasses import dataclass, field
from typing import Literal, Optional
from collections import defaultdict
import time
import anthropic

@dataclass
class FeedbackEntry:
    signal_type: Literal['thumbs_up', 'thumbs_down', 'correction', 'implicit_retry']
    session_id: str
    timestamp: float = field(default_factory=time.time)
    original: Optional[str] = None
    corrected: Optional[str] = None

class FeedbackCapture:
    def __init__(self):
        self.buffer: dict[str, list[FeedbackEntry]] = defaultdict(list)
    
    def record(self, entry: FeedbackEntry) -> None:
        self.buffer[entry.session_id].append(entry)
        self._check_quality_threshold(entry.session_id)
    
    def _check_quality_threshold(self, session_id: str) -> None:
        """最近30分钟内3条thumbs_down触发告警"""
        recent = [
            e for e in self.buffer[session_id]
            if e.signal_type == 'thumbs_down'
            and time.time() - e.timestamp < 1800
        ]
        if len(recent) >= 3:
            print(f"⚠️  Quality degradation detected for {session_id}")
    
    def get_session_entries(self, session_id: str) -> list[FeedbackEntry]:
        ttl = 24 * 3600
        return [
            e for e in self.buffer[session_id]
            if time.time() - e.timestamp < ttl
        ]

class FeedbackAwareAgent:
    def __init__(self):
        self.client = anthropic.Anthropic()
        self.capture = FeedbackCapture()
    
    def build_adjusted_prompt(self, session_id: str, base_prompt: str) -> str:
        entries = self.capture.get_session_entries(session_id)
        if not entries:
            return base_prompt
        
        positive = sum(1 for e in entries if e.signal_type == 'thumbs_up')
        negative = sum(1 for e in entries if e.signal_type == 'thumbs_down')
        corrections = [e for e in entries if e.signal_type == 'correction']
        
        adjustments = []
        
        if negative > positive + 2:
            adjustments.append("用户对最近回复不满意，请更加简洁直接。")
        
        if corrections:
            hints = "\n".join(
                f'- 将"{c.original[:40]}"改为"{c.corrected[:40]}"'
                for c in corrections[-3:]
            )
            adjustments.append(f"用户纠正记录：\n{hints}")
        
        if not adjustments:
            return base_prompt
        
        feedback_ctx = "\n".join(adjustments)
        return f"{base_prompt}\n\n<user_feedback_context>\n{feedback_ctx}\n</user_feedback_context>"
    
    def respond(
        self,
        session_id: str,
        base_system: str,
        messages: list[dict]
    ) -> str:
        system = self.build_adjusted_prompt(session_id, base_system)
        
        response = self.client.messages.create(
            model="claude-haiku-4-5",
            max_tokens=1024,
            system=system,
            messages=messages
        )
        return response.content[0].text
    
    # OpenClaw 集成：处理 Telegram 反应信号
    def handle_reaction(self, session_id: str, message_id: str, emoji: str) -> None:
        """在 OpenClaw 中，用户的 👍👎 反应直接映射到反馈信号"""
        signal_type = 'thumbs_up' if emoji in ['👍', '🔥', '❤️'] else 'thumbs_down'
        self.capture.record(FeedbackEntry(
            signal_type=signal_type,
            session_id=session_id,
        ))
        print(f"[Feedback] {signal_type} recorded for session {session_id}")
```

---

## OpenClaw 实战：用 Reaction 驱动反馈

OpenClaw 天然支持 Telegram 反应（reactions），直接与反馈系统对接：

```typescript
// 在 OpenClaw 消息处理器中
// SOUL.md / AGENTS.md 配置反馈感知逻辑

// 1. 每次 Agent 响应后，记录 messageId → session 映射
const messageLog = new Map<string, string>(); // messageId → sessionId

// 2. 当用户对消息做出反应时（OpenClaw heartbeat 或 webhook）
function onReaction(messageId: string, emoji: string) {
  const sessionId = messageLog.get(messageId);
  if (!sessionId) return;
  
  feedbackCapture.record({
    type: emoji === '👍' ? 'thumbs_up' : 'thumbs_down',
    messageId,
    sessionId,
  });
}

// 3. 下次调用时自动注入调整后的 prompt
const adjustedPrompt = middleware.buildFeedbackAwarePrompt(sessionId, BASE_PROMPT);
```

**OpenClaw memory 文件 = 跨会话反馈持久化：**

```
# memory/feedback-profile.json
{
  "preferredStyle": "concise",
  "positiveCount": 12,
  "negativeCount": 2,
  "corrections": [
    { "original": "以下是详细的步骤说明...", "corrected": "直接给代码" }
  ]
}
```

> 这就是 OpenClaw `USER.md` 和 `memory/` 目录的本质：用文件作反馈持久化，跨重启不丢失。

---

## 与 pi-mono 集成

```typescript
// pi-mono/src/agent/feedback-processor.ts
import { responseProcessor } from '@pi/core';

// pi-mono 的 responseProcessor hook 天然是注入反馈收集的位置
responseProcessor.use(async (response, context) => {
  const { sessionId, messageId } = context;
  
  // 1. 存储 messageId → response 映射，等待用户反馈
  await feedbackStore.pendingFeedback.set(messageId, {
    sessionId,
    responseSnapshot: response.content.slice(0, 200),
    timestamp: Date.now(),
  });
  
  // 2. 异步检查 3 分钟内有没有收到 thumbs_down
  setTimeout(async () => {
    const feedback = await feedbackCapture.getForMessage(messageId);
    if (feedback?.type === 'thumbs_down') {
      // 触发低质量告警 or 自动重试策略
      await qualityAlert.fire(sessionId, messageId);
    }
  }, 3 * 60 * 1000);
  
  return response;
});
```

---

## 在线学习 vs 离线训练：选型对比

| 维度 | 在线学习（本节方案）| 离线微调 |
|------|------------------|---------|
| 生效时间 | 当轮/当日 | 训练完成后（天/周）|
| 成本 | 极低（prompt 调整）| 高（GPU 时间）|
| 适用场景 | 行为/风格偏好 | 知识/能力提升 |
| 风险 | Prompt 注入风险 | 训练数据污染 |
| OpenClaw 支持 | ✅ memory 文件直接实现 | ❌ 需要外部平台 |

---

## 关键设计原则

1. **反馈必须轻量**：收集不能比交互本身重，否则用户放弃
2. **隐式 > 显式**：大多数用户不点赞，但他们的行为会说话
3. **Session 内即时生效，跨 Session 持久化**：当轮用了就感受到改善
4. **纠正信号最有价值**：用户愿意改你说的，代表他对结果有期望
5. **防 Prompt 注入**：反馈内容不能无校验地拼入 system prompt

---

## 小结

```
反馈信号 → 捕获 → 分析 → 调整 Prompt → 更好的响应 → 更多正向反馈
    ↑___________________________________|
                  在线学习飞轮
```

**OpenClaw 用户的直接价值**：
- `memory/feedback-profile.json` 跨重启记住你的偏好
- Telegram 反应（👍👎）直接驱动下一轮 prompt 调整
- Heartbeat 定期蒸馏反馈 → 更新 `USER.md`

不需要重训练模型。一个 JSON 文件 + 一段 middleware，你的 Agent 就能在用的过程中越来越懂你。
