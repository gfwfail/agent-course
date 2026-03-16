# 64 - 时间感知 Agent（Time-Aware Agents）

## 为什么 Agent 需要"感知时间"？

LLM 本身是无状态的——它不知道"现在"是什么时候。训练数据有截止日期，每次推理都是独立的。但现实中的任务 **充满了时间依赖**：

- "提醒我明天开会" → 需要知道今天是几号
- "昨天的销售数据" → 需要动态计算日期范围
- "这个任务已经超时了吗？" → 需要比较时间戳
- "现在适合发邮件吗？" → 需要知道用户时区和当前时间

时间感知是让 Agent 从"聪明的文字处理器"变成"真正有用的助理"的关键一步。

---

## 核心问题：LLM 不知道现在几点

```python
# ❌ 错误：期望 LLM 自己知道时间
response = client.messages.create(
    model="claude-3-5-sonnet",
    messages=[{"role": "user", "content": "明天的天气怎么样？"}]
)
# LLM 不知道"今天"是哪天，只能猜

# ✅ 正确：注入时间上下文
from datetime import datetime
import pytz

tz = pytz.timezone("Australia/Sydney")
now = datetime.now(tz)

system_prompt = f"""
Current time: {now.strftime("%A, %B %d, %Y — %I:%M %p")} ({tz.zone})
Current date ISO: {now.isoformat()}
"""
```

OpenClaw 就是这么做的，每次系统提示都包含当前时间：

```
Current time: Monday, March 16th, 2026 — 6:30 PM (Australia/Sydney)
```

---

## 方案一：System Prompt 时间注入

最简单也最常用的方案——把时间信息放进系统提示词。

```typescript
// OpenClaw 风格的时间注入
function buildSystemPrompt(userTimezone: string): string {
  const now = new Date();
  const formatter = new Intl.DateTimeFormat("zh-CN", {
    timeZone: userTimezone,
    year: "numeric",
    month: "long", 
    day: "numeric",
    weekday: "long",
    hour: "2-digit",
    minute: "2-digit",
    hour12: false,
  });

  return `
## 当前时间
${formatter.format(now)}（时区：${userTimezone}）
Unix 时间戳：${Math.floor(now.getTime() / 1000)}

你是一个时间感知 Agent。处理时间相关请求时，请基于以上"当前时间"进行计算，
不要使用你训练数据中的"最新"日期。
  `.trim();
}
```

**注意：** 时区非常重要！一个用户在悉尼说"今晚"和另一个用户在北京说"今晚"是完全不同的时间。

---

## 方案二：时间工具（Time Tool）

对于复杂的时间计算，给 Agent 提供专用工具比在 prompt 里塞时间更健壮：

```python
# learn-claude-code 风格的工具定义
TIME_TOOLS = [
    {
        "name": "get_current_time",
        "description": "获取当前时间，支持指定时区",
        "input_schema": {
            "type": "object",
            "properties": {
                "timezone": {
                    "type": "string",
                    "description": "IANA 时区名，如 Asia/Shanghai, Australia/Sydney",
                    "default": "UTC"
                },
                "format": {
                    "type": "string",
                    "enum": ["iso", "unix", "human", "date_only"],
                    "description": "返回格式"
                }
            }
        }
    },
    {
        "name": "calculate_time_delta",
        "description": "计算两个时间点之间的差值",
        "input_schema": {
            "type": "object",
            "properties": {
                "from_time": {"type": "string", "description": "ISO 8601 时间戳"},
                "to_time": {"type": "string", "description": "ISO 8601 时间戳，默认为现在"},
                "unit": {
                    "type": "string",
                    "enum": ["seconds", "minutes", "hours", "days"],
                    "default": "minutes"
                }
            },
            "required": ["from_time"]
        }
    },
    {
        "name": "is_business_hours",
        "description": "判断当前是否在工作时间内",
        "input_schema": {
            "type": "object",
            "properties": {
                "timezone": {"type": "string"},
                "start_hour": {"type": "number", "default": 9},
                "end_hour": {"type": "number", "default": 18}
            }
        }
    }
]

# 工具实现
def handle_time_tool(tool_name: str, params: dict):
    from datetime import datetime, timezone
    import pytz
    
    if tool_name == "get_current_time":
        tz = pytz.timezone(params.get("timezone", "UTC"))
        now = datetime.now(tz)
        fmt = params.get("format", "human")
        
        if fmt == "iso":
            return now.isoformat()
        elif fmt == "unix":
            return int(now.timestamp())
        elif fmt == "date_only":
            return now.strftime("%Y-%m-%d")
        else:  # human
            return now.strftime("%Y年%m月%d日 %H:%M:%S (%Z)")
    
    elif tool_name == "calculate_time_delta":
        from_dt = datetime.fromisoformat(params["from_time"])
        to_time = params.get("to_time")
        to_dt = datetime.fromisoformat(to_time) if to_time else datetime.now(timezone.utc)
        
        delta = to_dt - from_dt
        total_seconds = abs(delta.total_seconds())
        unit = params.get("unit", "minutes")
        
        divisors = {"seconds": 1, "minutes": 60, "hours": 3600, "days": 86400}
        return round(total_seconds / divisors[unit], 2)
    
    elif tool_name == "is_business_hours":
        tz = pytz.timezone(params.get("timezone", "UTC"))
        now = datetime.now(tz)
        start = params.get("start_hour", 9)
        end = params.get("end_hour", 18)
        is_weekday = now.weekday() < 5  # 0=Monday
        is_business = start <= now.hour < end
        return {"is_business_hours": is_weekday and is_business, "current_hour": now.hour}
```

---

## 方案三：截止时间感知（Deadline-Aware Planning）

Agent 在执行长任务时，需要知道自己还有多少时间：

```typescript
// pi-mono 风格的时间感知执行器
interface TimeConstrainedTask {
  task: string;
  deadlineMs?: number;      // 绝对截止时间
  timeoutMs?: number;       // 相对超时
  startedAt: number;
}

class DeadlineAwareAgent {
  private startTime: number;
  private deadline: number | null;

  constructor(private task: TimeConstrainedTask) {
    this.startTime = Date.now();
    this.deadline = task.deadlineMs ?? 
      (task.timeoutMs ? Date.now() + task.timeoutMs : null);
  }

  get remainingMs(): number | null {
    if (!this.deadline) return null;
    return Math.max(0, this.deadline - Date.now());
  }

  get isExpired(): boolean {
    if (!this.deadline) return false;
    return Date.now() > this.deadline;
  }

  // 根据剩余时间决定策略
  getStrategy(): "full" | "quick" | "abort" {
    const remaining = this.remainingMs;
    if (remaining === null) return "full";
    if (remaining > 30_000) return "full";   // >30s：完整策略
    if (remaining > 5_000) return "quick";   // 5-30s：快速策略
    return "abort";                           // <5s：放弃
  }

  buildContextualPrompt(): string {
    const remaining = this.remainingMs;
    const elapsed = Date.now() - this.startTime;
    
    const timeInfo = remaining !== null
      ? `⏱ 剩余时间: ${Math.floor(remaining / 1000)}秒`
      : `⏱ 已用时: ${Math.floor(elapsed / 1000)}秒`;
    
    const strategy = this.getStrategy();
    const strategyHint = {
      full: "请完整执行任务。",
      quick: "时间有限，请优先完成最重要的部分。",
      abort: "时间即将耗尽，请立即返回当前已有的结果。"
    }[strategy];
    
    return `${timeInfo}\n${strategyHint}\n\n任务：${this.task.task}`;
  }
}

// 使用示例
const agent = new DeadlineAwareAgent({
  task: "分析本月销售数据并生成报告",
  timeoutMs: 60_000  // 1分钟
});

// 在每个 agent 循环中检查
while (!agent.isExpired) {
  const strategy = agent.getStrategy();
  if (strategy === "abort") {
    console.log("时间耗尽，返回部分结果");
    break;
  }
  
  const prompt = agent.buildContextualPrompt();
  // ... 执行 LLM 调用
}
```

---

## 方案四：OpenClaw Cron + 时间感知 Heartbeat

OpenClaw 把"时间"变成了一等公民——通过 Cron 和 Heartbeat 让 Agent 主动感知时间流逝：

```json
// Cron Job 定义：每天早上 9 点执行
{
  "name": "Morning Briefing",
  "schedule": {
    "kind": "cron",
    "expr": "0 9 * * 1-5",
    "tz": "Australia/Sydney"
  },
  "payload": {
    "kind": "agentTurn",
    "message": "早安！请执行早间简报：检查邮件、日历事件、待办事项。当前是工作日早上9点。"
  },
  "sessionTarget": "isolated"
}
```

Heartbeat 文件里可以写时间敏感的检查逻辑：

```markdown
# HEARTBEAT.md

## 时间感知检查
- 如果是周一早上 → 发送本周计划
- 如果是下午 5 点后 → 不打扰，只记录
- 如果有超过 24h 未处理的消息 → 提醒用户

## 今日待办
- [ ] 检查部署状态
- [ ] 确认明天的会议
```

---

## 常见陷阱

### 1. 时区地狱（Timezone Hell）

```python
# ❌ 危险：naive datetime（没有时区信息）
from datetime import datetime
now = datetime.now()  # 是哪个时区？！

# ✅ 安全：always timezone-aware
from datetime import datetime, timezone
now = datetime.now(timezone.utc)  # 明确是 UTC

# 转换给用户显示
import pytz
user_tz = pytz.timezone("Asia/Shanghai")
user_time = now.astimezone(user_tz)
print(user_time.strftime("%Y-%m-%d %H:%M %Z"))  # 2026-03-16 15:30 CST
```

### 2. 相对时间解析

```python
# 用户说"明天"、"下周"、"三天后"
# 不要让 LLM 自己解析——用 dateparser 库

import dateparser

def parse_user_time(text: str, user_timezone: str) -> datetime | None:
    settings = {
        "TIMEZONE": user_timezone,
        "RETURN_AS_TIMEZONE_AWARE": True,
        "PREFER_DATES_FROM": "future",  # "明天"默认取未来
    }
    return dateparser.parse(text, settings=settings)

# 解析后，告诉 LLM 解析结果
parsed = parse_user_time("下周三下午三点", "Asia/Shanghai")
# → 2026-03-18 15:00:00+08:00
```

### 3. 时间上下文过期

```typescript
// ❌ 危险：session 开始时注入时间，几小时后还在用旧时间
const systemPrompt = `现在是 ${new Date().toISOString()}`; // 只算一次！

// ✅ 正确：每次调用时重新计算
function getSystemPrompt(): string {
  return `现在是 ${new Date().toISOString()}`; // 每次新鲜
}

// 或者：在 user message 里注入（更灵活）
function wrapUserMessage(userMsg: string): string {
  return `[当前时间: ${new Date().toISOString()}]\n\n${userMsg}`;
}
```

---

## 实战：构建一个时间感知调度 Agent

```python
# 完整示例：能理解"合适时机"的通知 Agent
import anthropic
from datetime import datetime
import pytz

def create_timing_aware_agent(user_timezone: str = "Australia/Sydney"):
    client = anthropic.Anthropic()
    tz = pytz.timezone(user_timezone)
    now = datetime.now(tz)
    
    hour = now.hour
    weekday = now.strftime("%A")
    
    # 根据时间生成上下文提示
    time_context = f"""
## 当前时间信息
- 时间：{now.strftime("%Y-%m-%d %H:%M %Z")}
- 星期：{weekday}
- 用户时区：{user_timezone}

## 时机判断规则
- 00:00-08:00：深夜/清晨，除非紧急否则不打扰
- 08:00-12:00：早间工作时间，适合重要通知
- 12:00-14:00：午休时间，轻量通知可以
- 14:00-18:00：下午工作时间，适合所有通知
- 18:00-22:00：晚间，适合非工作通知
- 22:00-24:00：深夜，仅紧急情况
""".strip()

    tools = [
        {
            "name": "send_notification",
            "description": "发送通知给用户",
            "input_schema": {
                "type": "object",
                "properties": {
                    "message": {"type": "string"},
                    "urgency": {
                        "type": "string",
                        "enum": ["low", "medium", "high", "critical"]
                    },
                    "reason_for_timing": {
                        "type": "string",
                        "description": "说明为什么现在是合适的发送时机"
                    }
                },
                "required": ["message", "urgency", "reason_for_timing"]
            }
        },
        {
            "name": "schedule_for_later",
            "description": "如果现在不是合适时机，安排在合适时间发送",
            "input_schema": {
                "type": "object",
                "properties": {
                    "message": {"type": "string"},
                    "scheduled_time": {
                        "type": "string",
                        "description": "ISO 8601 格式的目标发送时间"
                    },
                    "reason": {"type": "string"}
                },
                "required": ["message", "scheduled_time", "reason"]
            }
        }
    ]
    
    messages = [
        {
            "role": "user",
            "content": "服务器 CPU 使用率超过 80%，已持续 5 分钟。请决定是否立即通知用户。"
        }
    ]
    
    response = client.messages.create(
        model="claude-3-5-sonnet-20241022",
        max_tokens=1024,
        system=time_context,
        tools=tools,
        messages=messages
    )
    
    # 处理工具调用
    for block in response.content:
        if block.type == "tool_use":
            print(f"Agent 决策: {block.name}")
            print(f"参数: {block.input}")
    
    return response

# 运行
result = create_timing_aware_agent()
```

---

## 总结

时间感知 Agent 的核心原则：

| 原则 | 实现方式 |
|------|---------|
| **注入时间** | System prompt 包含当前时间、时区 |
| **工具化时间** | 提供 get_time、calc_delta 等工具 |
| **时区感知** | 永远使用 timezone-aware datetime |
| **截止时间感知** | 长任务需要 deadline 检查 |
| **时机判断** | Agent 根据时间决定行为策略 |

LLM 本身是"时间盲"的，但一个设计良好的 Agent 系统应该让时间成为一等公民——注入进去、工具化它、在规划和通知中考虑它。

---

## 参考实现

- **OpenClaw**: `/opt/homebrew/lib/node_modules/openclaw/` — cron job 时间注入、heartbeat 时间感知
- **learn-claude-code**: 工具定义和时间工具实现示例
- **pi-mono**: TypeScript 时间感知任务执行器模式
