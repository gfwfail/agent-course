# 203 - Agent 主动助手与意图预测（Proactive Assistance & Intent Prediction）

> 被动 Agent 等用户开口才干活；主动 Agent 在用户开口之前就已经在路上了。

---

## 核心问题

传统 Agent 是**请求-响应**模型：用户说话 → Agent 回答 → 等下一条消息。

但真正好的助理不是这样工作的。一个优秀的助理会：
- 发现日历里有重要会议 → 提前发提醒，不等你问
- 注意到你昨天在问某个 bug → 今天主动告诉你找到解法了
- 看到你经常查某类信息 → 主动整理成报告推给你

**主动助手（Proactive Agent）** 的本质是：**把用户下一个可能的需求预判出来，在他们意识到之前就准备好。**

---

## 三层主动能力模型

```
┌─────────────────────────────────────────┐
│  Layer 3: 主动推送（Proactive Push）      │
│  无需任何触发，Agent 主动发现并推送价值   │
├─────────────────────────────────────────┤
│  Layer 2: 意图预测（Intent Prediction）   │
│  用户输入时预判下一步，提前准备           │
├─────────────────────────────────────────┤
│  Layer 1: 需求感知（Need Detection）      │
│  从上下文信号检测用户潜在需求             │
└─────────────────────────────────────────┘
```

---

## Layer 1：需求感知信号

用户不会明说所有需求，但会留下信号：

```typescript
// 信号类型定义
interface NeedSignal {
  type: 
    | 'calendar_event'      // 日历事件临近
    | 'repeated_query'      // 重复查询同类问题
    | 'incomplete_action'   // 任务做了一半停了
    | 'error_pattern'       // 错误持续出现
    | 'external_event'      // 外部事件（邮件/通知）
    | 'time_based'          // 时间驱动（早上/周一）
  signal: string;
  confidence: number;       // 0-1
  suggestedAction: string;
}

// 信号检测器
class NeedDetector {
  private patterns: Map<string, number> = new Map();

  // 检测重复查询模式
  trackQuery(userId: string, query: string): NeedSignal | null {
    const key = `${userId}:${this.normalizeQuery(query)}`;
    const count = (this.patterns.get(key) || 0) + 1;
    this.patterns.set(key, count);

    // 同类问题问了 3 次 → 主动整理成文档
    if (count >= 3) {
      return {
        type: 'repeated_query',
        signal: `用户已询问"${query}"相关问题 ${count} 次`,
        confidence: 0.85,
        suggestedAction: `主动整理一份关于"${this.normalizeQuery(query)}"的参考文档`
      };
    }
    return null;
  }

  private normalizeQuery(query: string): string {
    // 简单归一化：去停用词，取关键词
    return query.toLowerCase().replace(/[？！？!,，.。]/g, '').trim();
  }
}
```

---

## Layer 2：意图预测——下一步是什么？

用户当前操作 → 预测下一个最可能的请求：

```typescript
// 意图转移矩阵（从历史数据学习）
const intentTransitions: Record<string, Array<{ next: string; prob: number }>> = {
  'check_email': [
    { next: 'reply_email', prob: 0.45 },
    { next: 'calendar_check', prob: 0.30 },
    { next: 'create_task', prob: 0.15 },
  ],
  'code_review': [
    { next: 'run_tests', prob: 0.60 },
    { next: 'check_ci', prob: 0.25 },
    { next: 'update_docs', prob: 0.10 },
  ],
  'deploy_app': [
    { next: 'check_logs', prob: 0.70 },
    { next: 'health_check', prob: 0.20 },
    { next: 'notify_team', prob: 0.08 },
  ]
};

class IntentPredictor {
  predict(currentIntent: string, threshold = 0.3): string[] {
    const transitions = intentTransitions[currentIntent] || [];
    return transitions
      .filter(t => t.prob >= threshold)
      .sort((a, b) => b.prob - a.prob)
      .map(t => t.next);
  }

  // 用户查询邮件后，预取日历数据（prob=0.30 够高）
  async prefetchForIntent(currentIntent: string, context: AgentContext) {
    const nextIntents = this.predict(currentIntent);

    // 并行预取，不阻塞当前响应
    const prefetchPromises = nextIntents.map(async (intent) => {
      const prefetcher = this.getPrefetcher(intent);
      if (prefetcher) {
        context.cache.set(`prefetch:${intent}`, await prefetcher(context));
      }
    });

    // 不 await，后台跑
    Promise.allSettled(prefetchPromises).catch(() => {});
  }

  private getPrefetcher(intent: string): ((ctx: AgentContext) => Promise<any>) | null {
    const prefetchers: Record<string, (ctx: AgentContext) => Promise<any>> = {
      'calendar_check': async (ctx) => ctx.tools.calendar.getUpcoming(48),
      'check_ci': async (ctx) => ctx.tools.github.getRecentRuns(),
      'check_logs': async (ctx) => ctx.tools.logs.getTail(100),
    };
    return prefetchers[intent] || null;
  }
}
```

---

## Layer 3：主动推送——不等用户问

这是最高级的主动能力，也是 OpenClaw Heartbeat 的核心：

```typescript
// 主动助手主循环
class ProactiveAssistant {
  private detector = new NeedDetector();
  private predictor = new IntentPredictor();

  // 在 Heartbeat 中调用
  async heartbeat(context: AgentContext): Promise<ProactiveInsight[]> {
    const insights: ProactiveInsight[] = [];

    // 1. 检查日历事件
    const calendarInsights = await this.checkCalendar(context);
    insights.push(...calendarInsights);

    // 2. 检查未完成任务
    const incompleteInsights = await this.checkIncompleteTasks(context);
    insights.push(...incompleteInsights);

    // 3. 检查外部事件（新邮件/PR comment/告警）
    const externalInsights = await this.checkExternalEvents(context);
    insights.push(...externalInsights);

    // 按 priority 排序，只推最重要的（避免打扰）
    return insights
      .sort((a, b) => b.priority - a.priority)
      .slice(0, 3); // 单次最多推 3 条
  }

  private async checkCalendar(context: AgentContext): Promise<ProactiveInsight[]> {
    const events = await context.tools.calendar.getUpcoming(2); // 2小时内
    return events.map(event => ({
      type: 'calendar_reminder',
      message: `📅 ${event.title} 将在 ${event.minutesUntil} 分钟后开始`,
      action: event.meetingUrl ? `加入会议：${event.meetingUrl}` : null,
      priority: event.minutesUntil < 15 ? 10 : 6,
    }));
  }

  private async checkIncompleteTasks(context: AgentContext): Promise<ProactiveInsight[]> {
    // 从 memory 检查昨天开了头没做完的任务
    const yesterday = await context.memory.get('incomplete_tasks');
    if (!yesterday?.length) return [];

    return yesterday.map((task: string) => ({
      type: 'incomplete_task',
      message: `🔄 昨天有个未完成的任务："${task}"，需要继续吗？`,
      action: `继续处理：${task}`,
      priority: 4,
    }));
  }

  private async checkExternalEvents(context: AgentContext): Promise<ProactiveInsight[]> {
    const insights: ProactiveInsight[] = [];

    // 检查是否有需要关注的新邮件（只看紧急/重要发件人）
    const urgentEmails = await context.tools.email.getUnread({
      fromImportantSenders: true,
      since: Date.now() - 3600_000 // 1小时内
    });

    if (urgentEmails.length > 0) {
      insights.push({
        type: 'important_email',
        message: `📧 ${urgentEmails.length} 封重要邮件待处理`,
        action: `查看邮件`,
        priority: 8,
      });
    }

    return insights;
  }
}

interface ProactiveInsight {
  type: string;
  message: string;
  action: string | null;
  priority: number; // 1-10
}
```

---

## OpenClaw 实战：HEARTBEAT.md 驱动的主动助手

OpenClaw 的 Heartbeat 机制天然就是主动助手的基础设施：

```markdown
# HEARTBEAT.md

## 主动检查清单（每次 heartbeat 按序执行）

### 优先级 HIGH（必查）
- [ ] 日历：2小时内是否有会议？
- [ ] 邮件：是否有老板/重要客户的邮件？
- [ ] 告警：生产服务是否有异常？

### 优先级 MEDIUM（轮换查，每次选 1-2 项）
- [ ] Github PR：是否有需要 review 的 PR？
- [ ] 未完成任务：memory 中有没有昨天的遗留项？
- [ ] 天气：老板今天是否需要出门？

### 发送条件
- 发现优先级 HIGH 的事项 → 立即推送
- 发现优先级 MEDIUM 的事项 → 累积 2+ 条再推
- 凌晨 23:00-07:00 → 除非 HIGH 紧急，否则静音
```

```typescript
// OpenClaw Heartbeat 处理器（伪代码对应实际行为）
async function handleHeartbeat() {
  const assistant = new ProactiveAssistant();
  const insights = await assistant.heartbeat(context);

  if (insights.length === 0) {
    return 'HEARTBEAT_OK'; // 无需打扰
  }

  // 组装推送消息
  const message = insights.map(i => {
    let text = i.message;
    if (i.action) text += `\n→ ${i.action}`;
    return text;
  }).join('\n\n');

  await sendToUser(message);
}
```

---

## Python 版：pi-mono 集成

```python
from dataclasses import dataclass
from typing import Optional
import asyncio
from datetime import datetime, timedelta

@dataclass
class NeedSignal:
    type: str
    message: str
    confidence: float
    suggested_action: Optional[str] = None
    priority: int = 5

class ProactiveAgent:
    """主动助手：不等被问，主动发现需求"""
    
    def __init__(self, tools, memory):
        self.tools = tools
        self.memory = memory
        self._query_counter: dict[str, int] = {}
    
    async def analyze_context(self) -> list[NeedSignal]:
        """扫描上下文，发现潜在需求"""
        signals = []
        
        # 并发检查多个来源
        results = await asyncio.gather(
            self._check_upcoming_deadlines(),
            self._check_stale_tasks(),
            self._check_system_health(),
            return_exceptions=True
        )
        
        for result in results:
            if isinstance(result, list):
                signals.extend(result)
        
        return sorted(signals, key=lambda s: s.priority, reverse=True)
    
    async def _check_upcoming_deadlines(self) -> list[NeedSignal]:
        signals = []
        events = await self.tools.calendar.get_events(hours=24)
        
        for event in events:
            minutes_until = (event.start_time - datetime.now()).seconds // 60
            if minutes_until <= 60:
                signals.append(NeedSignal(
                    type='deadline',
                    message=f"⏰ '{event.title}' 还有 {minutes_until} 分钟",
                    confidence=1.0,
                    suggested_action=f"准备会议材料",
                    priority=10 if minutes_until <= 15 else 7
                ))
        return signals
    
    async def _check_stale_tasks(self) -> list[NeedSignal]:
        # 从 memory 读取未完成任务
        tasks = await self.memory.get('pending_tasks', default=[])
        stale = [t for t in tasks if t.get('age_hours', 0) > 24]
        
        if stale:
            return [NeedSignal(
                type='stale_task',
                message=f"📌 有 {len(stale)} 个任务搁置超过24小时",
                confidence=0.9,
                suggested_action="查看并处理积压任务",
                priority=5
            )]
        return []
    
    async def _check_system_health(self) -> list[NeedSignal]:
        try:
            metrics = await self.tools.monitoring.get_metrics()
            if metrics.error_rate > 0.05:  # 错误率超过 5%
                return [NeedSignal(
                    type='system_alert',
                    message=f"🚨 服务错误率 {metrics.error_rate:.1%}，可能需要关注",
                    confidence=0.95,
                    suggested_action="查看错误日志",
                    priority=9
                )]
        except Exception:
            pass
        return []
    
    def track_query_pattern(self, user_id: str, query: str) -> Optional[NeedSignal]:
        """追踪查询模式，发现重复需求"""
        key = f"{user_id}:{query[:50]}"
        self._query_counter[key] = self._query_counter.get(key, 0) + 1
        
        count = self._query_counter[key]
        if count == 3:
            return NeedSignal(
                type='repeated_query',
                message=f"你已经问了 3 次类似问题，我来整理一份参考文档？",
                confidence=0.8,
                suggested_action=f"生成关于 '{query[:30]}' 的参考文档",
                priority=4
            )
        return None
```

---

## 主动推送的节奏控制

主动不等于骚扰，关键是控制推送频率：

```typescript
class PushThrottler {
  private lastPushTime = 0;
  private dailyPushCount = 0;
  private readonly MIN_INTERVAL_MS = 30 * 60 * 1000; // 最少30分钟一次
  private readonly MAX_DAILY_PUSH = 10;               // 每天最多10次

  canPush(priority: number): boolean {
    const now = Date.now();
    const hour = new Date().getHours();

    // 深夜不推（除非紧急）
    if (hour >= 23 || hour < 7) {
      return priority >= 9; // 只推最紧急的
    }

    // 检查间隔
    if (now - this.lastPushTime < this.MIN_INTERVAL_MS) {
      return priority >= 8; // 高优先级可以破间隔限制
    }

    // 检查每日上限
    if (this.dailyPushCount >= this.MAX_DAILY_PUSH) {
      return priority >= 9;
    }

    return true;
  }

  recordPush() {
    this.lastPushTime = Date.now();
    this.dailyPushCount++;
  }
}
```

---

## 意图预测的三种实现策略

| 策略 | 实现方式 | 适合场景 | 准确率 |
|------|---------|---------|--------|
| 静态规则 | `if 部署 then 检查日志` | 固定工作流 | 中 |
| 转移矩阵 | 统计历史条件概率 | 有历史数据 | 中高 |
| LLM 预测 | 让模型根据上下文预测 | 复杂场景 | 高（但贵） |

```typescript
// LLM 驱动的意图预测（用 Haiku 省钱）
async function predictNextIntent(
  recentActions: string[], 
  userContext: string
): Promise<string[]> {
  const response = await llm.complete({
    model: 'claude-haiku-4-5', // 便宜模型做预测
    messages: [{
      role: 'user',
      content: `用户最近的操作：${recentActions.join(' → ')}
用户背景：${userContext}

预测用户接下来最可能需要做的 2-3 件事，每行一个，简洁描述：`
    }]
  });
  
  return response.content.split('\n').filter(Boolean).slice(0, 3);
}
```

---

## 与 Heartbeat 的关系

```
OpenClaw Heartbeat（每30分钟）
       │
       ▼
ProactiveAssistant.heartbeat()
       │
       ├─ checkCalendar()      → 日历事件检测
       ├─ checkExternalEvents() → 邮件/通知检测
       ├─ checkIncompleteTasks() → 遗留任务检测
       └─ checkSystemHealth()  → 系统异常检测
       │
       ▼
PushThrottler.canPush() → 节奏控制
       │
       ▼
sendToUser() / HEARTBEAT_OK
```

Heartbeat 是**触发器**，ProactiveAssistant 是**分析引擎**，PushThrottler 是**质量过滤器**。三者结合才能做到"主动但不烦人"。

---

## 核心原则

1. **质量 > 数量**：一天 3 条有价值的推送，胜过 30 条无用通知
2. **优先级严格分级**：紧急 > 重要 > 有趣，宁缺毋滥
3. **尊重静默时段**：深夜/周末降噪，让人类有喘息空间
4. **预取不等于推送**：意图预测后台预取数据，但不一定打扰用户
5. **记录推送效果**：用户忽略的推送 → 降低该类型的优先级

---

## 总结

| 能力层 | 触发方式 | 实现关键 |
|--------|---------|---------|
| 需求感知 | 信号检测 | NeedDetector + 模式识别 |
| 意图预测 | 当前操作 → 预判下一步 | 转移矩阵 / LLM 预测 |
| 主动推送 | 定时 Heartbeat | ProactiveAssistant + PushThrottler |

**OpenClaw 的 Heartbeat + SOUL.md + MEMORY.md = 主动助手的完整基础设施**，不需要额外框架，已经内置了。

> 好的 Agent 不只是问啥答啥，而是知道你下一步需要什么，并且已经准备好了。
