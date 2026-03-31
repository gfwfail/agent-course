# 184 - Agent 目标锚定与任务漂移检测（Goal Anchoring & Task Drift Detection）

> 长任务中用户经常"跑题"——帮 Agent 记住目标，检测漂移，温柔拉回正轨。

---

## 问题背景

用户启动一个长任务："帮我重构这个 Node.js 项目"。

然后：
- 第 3 轮：讨论到了数据库 schema
- 第 7 轮：顺手聊起了 TypeScript 配置
- 第 12 轮：开始问 CI/CD 流水线
- 第 18 轮：原来的重构任务……一行代码没动

**任务漂移（Task Drift）**：Agent 随着对话被带偏，逐渐偏离原始目标。

常见场景：
- 用户多步骤任务中途岔开话题
- 子任务过于深入遮蔽主线目标
- Agent 过于顺从，无法保持目标一致性

---

## 核心概念

### 目标锚（Goal Anchor）

在任务开始时提取并记录**主目标**，作为整个会话的"北极星"：

```typescript
interface GoalAnchor {
  id: string;
  originalIntent: string;        // 用户原始表达
  canonicalGoal: string;         // LLM 标准化后的目标
  keyResults: string[];          // 预期产出（KR）
  startedAt: number;
  turnsCompleted: number;
  status: 'active' | 'completed' | 'abandoned' | 'paused';
}
```

### 漂移信号（Drift Signals）

通过以下维度判断当前轮次是否漂移：

| 信号 | 计算方式 | 阈值 |
|------|----------|------|
| 语义相似度 | 当前消息 vs 主目标的 embedding 余弦距离 | < 0.4 |
| 话题跳变 | 连续 N 轮与主目标无关 | N ≥ 3 |
| 子任务深度 | 嵌套深度超过限制 | depth > 2 |
| 时间跨度 | 未推进主目标的轮次数 | > 8 turns |

---

## 实现：GoalAnchorManager

```typescript
// goal-anchor-manager.ts
import Anthropic from '@anthropic-ai/sdk';

const client = new Anthropic();

interface DriftReport {
  isDrifting: boolean;
  driftScore: number;       // 0.0 - 1.0，越高越漂移
  driftSignals: string[];
  suggestion: 'continue' | 'soft_redirect' | 'hard_redirect' | 'confirm_pivot';
}

class GoalAnchorManager {
  private anchor: GoalAnchor | null = null;
  private turnsWithoutProgress = 0;
  private subTaskDepth = 0;

  // ① 在第一轮提取并锚定目标
  async anchorGoal(userMessage: string): Promise<GoalAnchor> {
    const response = await client.messages.create({
      model: 'claude-opus-4-5',
      max_tokens: 500,
      messages: [{
        role: 'user',
        content: `分析以下用户请求，提取核心目标和预期产出：

用户消息：${userMessage}

以 JSON 格式返回：
{
  "canonicalGoal": "标准化的核心目标（一句话）",
  "keyResults": ["预期产出1", "预期产出2", "..."]
}`
      }]
    });

    const parsed = JSON.parse(
      (response.content[0] as Anthropic.TextBlock).text
    );

    this.anchor = {
      id: crypto.randomUUID(),
      originalIntent: userMessage,
      canonicalGoal: parsed.canonicalGoal,
      keyResults: parsed.keyResults,
      startedAt: Date.now(),
      turnsCompleted: 0,
      status: 'active'
    };

    return this.anchor;
  }

  // ② 每轮评估是否漂移
  async detectDrift(
    currentMessage: string,
    recentHistory: string[]
  ): Promise<DriftReport> {
    if (!this.anchor) {
      return { isDrifting: false, driftScore: 0, driftSignals: [], suggestion: 'continue' };
    }

    const response = await client.messages.create({
      model: 'claude-haiku-4-5',  // 用轻量模型做漂移检测，省成本
      max_tokens: 300,
      system: `你是一个任务跟踪分析器。判断当前对话是否偏离主目标。
主目标：${this.anchor.canonicalGoal}
预期产出：${this.anchor.keyResults.join(', ')}`,
      messages: [{
        role: 'user',
        content: `最近3轮对话：
${recentHistory.slice(-3).join('\n---\n')}

当前消息：${currentMessage}

以 JSON 返回：
{
  "isDrifting": boolean,
  "driftScore": 0.0-1.0,
  "driftSignals": ["信号1", "信号2"],
  "suggestion": "continue|soft_redirect|hard_redirect|confirm_pivot"
}`
      }]
    });

    const report: DriftReport = JSON.parse(
      (response.content[0] as Anthropic.TextBlock).text
    );

    if (report.isDrifting) {
      this.turnsWithoutProgress++;
    } else {
      this.turnsWithoutProgress = 0;
      this.anchor.turnsCompleted++;
    }

    // 超过 5 轮无进展，升级为 hard_redirect
    if (this.turnsWithoutProgress >= 5) {
      report.suggestion = 'hard_redirect';
    }

    return report;
  }

  // ③ 生成引导提示（注入到 system prompt）
  buildGoalReminderPrompt(report: DriftReport): string {
    if (!this.anchor || !report.isDrifting) return '';

    if (report.suggestion === 'soft_redirect') {
      return `\n\n[目标提醒] 当前主任务是：${this.anchor.canonicalGoal}。` +
             `回答完当前问题后，请温和地引导用户回到主线任务。`;
    }

    if (report.suggestion === 'hard_redirect') {
      return `\n\n[任务锚定] 主目标：${this.anchor.canonicalGoal}。` +
             `已偏离主线 ${this.turnsWithoutProgress} 轮。` +
             `请在回复末尾明确询问用户是否要暂停当前话题，继续主线任务。`;
    }

    if (report.suggestion === 'confirm_pivot') {
      return `\n\n[任务转向确认] 检测到用户可能想更改主目标。` +
             `请在回复末尾确认：是否要放弃"${this.anchor.canonicalGoal}"，转向新目标？`;
    }

    return '';
  }
}
```

---

## 集成到 Agent Loop

```typescript
// agent-loop-with-drift-detection.ts
class DriftAwareAgent {
  private goalManager = new GoalAnchorManager();
  private history: string[] = [];
  private isGoalAnchored = false;

  async chat(userMessage: string): Promise<string> {
    // 第一轮：锚定目标
    if (!this.isGoalAnchored && this.looksLikeATask(userMessage)) {
      const anchor = await this.goalManager.anchorGoal(userMessage);
      console.log(`🎯 目标已锚定: ${anchor.canonicalGoal}`);
      this.isGoalAnchored = true;
    }

    // 后续轮次：检测漂移
    let systemSuffix = '';
    if (this.isGoalAnchored && this.history.length > 2) {
      const report = await this.goalManager.detectDrift(
        userMessage,
        this.history
      );

      if (report.isDrifting) {
        console.log(`⚠️ 漂移检测: score=${report.driftScore.toFixed(2)}, signals=${report.driftSignals.join(', ')}`);
        systemSuffix = this.goalManager.buildGoalReminderPrompt(report);
      }
    }

    // 构建带目标提醒的系统提示
    const systemPrompt = `你是一个高效的 AI 助理，帮助用户完成任务。${systemSuffix}`;

    const response = await client.messages.create({
      model: 'claude-opus-4-5',
      max_tokens: 2048,
      system: systemPrompt,
      messages: [
        ...this.buildMessageHistory(),
        { role: 'user', content: userMessage }
      ]
    });

    const reply = (response.content[0] as Anthropic.TextBlock).text;
    this.history.push(`User: ${userMessage}\nAssistant: ${reply}`);
    return reply;
  }

  private looksLikeATask(msg: string): boolean {
    const taskKeywords = ['帮我', '帮助', '请', '需要', '想要', '实现', '创建', '构建', '修复'];
    return taskKeywords.some(k => msg.includes(k)) && msg.length > 20;
  }

  private buildMessageHistory(): Anthropic.MessageParam[] {
    // 取最近 10 轮避免 context 过长
    return this.history.slice(-10).flatMap(turn => {
      const [userPart, assistantPart] = turn.split('\nAssistant: ');
      return [
        { role: 'user' as const, content: userPart.replace('User: ', '') },
        { role: 'assistant' as const, content: assistantPart }
      ];
    });
  }
}
```

---

## OpenClaw 实战：SOUL.md + 心跳监控

OpenClaw 的主会话天然需要目标锚定。可以在 `HEARTBEAT.md` 中加入目标追踪：

```markdown
# HEARTBEAT.md

## 活跃目标追踪
- 检查 memory/active-goals.json
- 如果有超过 8 轮未推进的目标，主动提醒老板
- 已完成的目标标记并存档
```

```typescript
// memory/active-goals.json 结构
{
  "goals": [
    {
      "id": "goal-001",
      "canonicalGoal": "重构 mysterybox 支付模块",
      "startedAt": 1743382200000,
      "lastProgressAt": 1743382200000,
      "turnsStalled": 0,
      "status": "active"
    }
  ]
}
```

---

## pi-mono 中的实现参考

pi-mono 的 `TaskSystem` 有类似思路：每个 Task 有明确的 `objective` 字段，子任务完成后会向上汇报进度（progress propagation），防止被子任务"吞噬"。

```python
# Python 版本：轻量漂移检测器
import json
from anthropic import Anthropic

client = Anthropic()

class DriftDetector:
    def __init__(self, goal: str):
        self.goal = goal
        self.stall_count = 0

    def check(self, current_message: str, recent_turns: list[str]) -> dict:
        resp = client.messages.create(
            model="claude-haiku-4-5",
            max_tokens=200,
            system=f"主目标：{self.goal}",
            messages=[{
                "role": "user",
                "content": f"当前消息是否与主目标相关？\n消息：{current_message}\n"
                           f"以JSON返回：{{\"relevant\": bool, \"score\": 0.0-1.0}}"
            }]
        )
        result = json.loads(resp.content[0].text)

        if not result["relevant"]:
            self.stall_count += 1
        else:
            self.stall_count = 0

        result["should_redirect"] = self.stall_count >= 3
        return result
```

---

## 三种漂移策略对比

| 策略 | 触发条件 | Agent 行为 | 用户体验 |
|------|----------|------------|----------|
| **Soft Redirect** | driftScore 0.5-0.7 | 回答完当前问题后轻轻引导 | 无感知，自然 |
| **Hard Redirect** | 连续 5 轮无进展 | 明确指出偏离，询问是否继续 | 稍打断，但有价值 |
| **Confirm Pivot** | 检测到明确的目标转向 | 请用户确认是否更换目标 | 防止误操作 |

---

## 设计原则

1. **不要做目标警察**：轻量提醒，不强制打断用户流程
2. **漂移 ≠ 无价值**：岔出的话题可能是隐含需求，用 confirm_pivot 而非直接拒绝
3. **轻量模型做检测**：用 Haiku 检测漂移，用 Opus 执行任务，省成本
4. **检测不要太频繁**：前 3 轮不检测（刚开始），之后每轮检测
5. **状态持久化**：目标锚写入 Redis/文件，崩溃重启后恢复

---

## 小结

| 模式 | 解决的问题 |
|------|------------|
| GoalAnchor | 第一轮提取并记录主目标 |
| DriftDetector | 每轮评估是否偏离 |
| buildGoalReminderPrompt | 动态注入引导提示 |
| HEARTBEAT 追踪 | 跨会话的目标持续监控 |

**核心思想**：Agent 不应该是被动的"问答机器"，而是主动的**任务伙伴**——记住你要做什么，在你跑偏时轻轻拉你一把。

---

> 下一课预告：**Agent 工具调用图谱分析（Tool Call Graph Analysis）** — 记录并分析工具调用模式，自动发现预取/缓存/批量优化机会。
