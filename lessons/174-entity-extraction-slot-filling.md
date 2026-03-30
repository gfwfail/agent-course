# 174 - Agent 实体提取与槽位填充

**Entity Extraction & Slot Filling**

---

## 为什么需要槽位填充？

用户说的话永远是不完整的。

```
用户: "帮我订明天去上海的机票"
```

这一句话里缺了什么？**出发地、出发时间（几点）、舱位、乘客姓名**……

**槽位填充（Slot Filling）** 就是解决这个问题的：
1. 从用户输入中**提取**已知实体（日期、城市等）
2. 识别**缺失**的必要槽位
3. 有针对性地**追问**缺失项
4. 验证填入的值是否合法
5. 所有槽位齐全后才执行工具调用

这不仅仅是"问问题"，而是一套结构化的多轮对话管理机制。

---

## 核心概念

```
┌─────────────────────────────────────────────────┐
│                   Slot Schema                    │
│  name: string (required)                        │
│  date: Date   (required)                        │
│  from: string (required)                        │
│  to:   string (required)                        │
│  class: enum  (optional, default: economy)      │
└─────────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────────┐
│              Entity Extractor                    │
│  "明天去上海" → { date: tomorrow, to: "上海" }  │
└─────────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────────┐
│              Slot Validator                      │
│  missing: [name, from]                          │
│  invalid: []                                    │
└─────────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────────┐
│              Clarification Engine               │
│  "请问从哪个城市出发？"                         │
└─────────────────────────────────────────────────┘
```

---

## TypeScript 实现

### 1. 定义槽位 Schema

```typescript
import { z } from "zod";

// 用 Zod 定义槽位结构 —— 兼做类型和运行时校验
const FlightSlots = z.object({
  passengerName: z.string().min(2).describe("乘客姓名"),
  fromCity:      z.string().describe("出发城市"),
  toCity:        z.string().describe("目的地城市"),
  departDate:    z.string().regex(/^\d{4}-\d{2}-\d{2}$/).describe("出发日期 YYYY-MM-DD"),
  departTime:    z.string().regex(/^\d{2}:\d{2}$/).optional().describe("出发时间 HH:MM，可选"),
  cabinClass:    z.enum(["economy", "business", "first"])
                  .default("economy").describe("舱位"),
});

type FlightSlotsType = z.infer<typeof FlightSlots>;

// 槽位元数据：哪些是必填，追问话术是什么
const SLOT_META: Record<keyof FlightSlotsType, { required: boolean; question: string }> = {
  passengerName: { required: true,  question: "请问乘客姓名是？" },
  fromCity:      { required: true,  question: "请问从哪个城市出发？" },
  toCity:        { required: true,  question: "目的地是哪个城市？" },
  departDate:    { required: true,  question: "请问出发日期是几号？（格式：2026-03-30）" },
  departTime:    { required: false, question: "有偏好的出发时间吗？（没有可跳过）" },
  cabinClass:    { required: false, question: "需要什么舱位？（经济/商务/头等，默认经济舱）" },
};
```

### 2. 实体提取器（LLM 驱动）

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();

async function extractEntities(
  userInput: string,
  existingSlots: Partial<FlightSlotsType>
): Promise<Partial<FlightSlotsType>> {
  const today = new Date().toISOString().split("T")[0];

  const result = await client.messages.create({
    model: "claude-opus-4-5",
    max_tokens: 500,
    tools: [{
      name: "fill_slots",
      description: "从用户输入中提取实体，填充槽位",
      input_schema: {
        type: "object",
        properties: {
          passengerName: { type: "string", description: "乘客姓名" },
          fromCity:      { type: "string", description: "出发城市（中文）" },
          toCity:        { type: "string", description: "目的地城市（中文）" },
          departDate:    { type: "string", description: `出发日期 YYYY-MM-DD，今天是 ${today}，"明天"应解析为 ${
            new Date(Date.now() + 86400000).toISOString().split("T")[0]
          }` },
          departTime:    { type: "string", description: "出发时间 HH:MM" },
          cabinClass:    { type: "string", enum: ["economy", "business", "first"] },
        },
        // 注意：不设 required，提取到什么返回什么
      },
    }],
    tool_choice: { type: "auto" },
    messages: [{
      role: "user",
      content: `从以下输入提取机票预订信息（只提取明确提到的信息，不要猜测）：\n\n"${userInput}"`,
    }],
  });

  // 解析工具调用结果
  const toolUse = result.content.find(b => b.type === "tool_use");
  if (!toolUse || toolUse.type !== "tool_use") return {};

  const extracted = toolUse.input as Partial<FlightSlotsType>;

  // 合并：已有槽位优先，新提取的补充空缺
  return {
    ...extracted,
    ...Object.fromEntries(
      Object.entries(existingSlots).filter(([_, v]) => v !== undefined)
    ),
  };
}
```

### 3. 槽位验证与缺失检测

```typescript
interface SlotValidationResult {
  filled:   Partial<FlightSlotsType>;
  missing:  Array<keyof FlightSlotsType>;
  invalid:  Array<{ slot: keyof FlightSlotsType; reason: string }>;
  isReady:  boolean;
}

function validateSlots(slots: Partial<FlightSlotsType>): SlotValidationResult {
  const missing: Array<keyof FlightSlotsType> = [];
  const invalid: Array<{ slot: keyof FlightSlotsType; reason: string }> = [];

  // 检查必填槽位
  for (const [key, meta] of Object.entries(SLOT_META)) {
    const slot = key as keyof FlightSlotsType;
    if (meta.required && !slots[slot]) {
      missing.push(slot);
    }
  }

  // 运行 Zod 验证（只验证已填的字段）
  const partialSchema = FlightSlots.partial();
  const parsed = partialSchema.safeParse(slots);
  if (!parsed.success) {
    for (const issue of parsed.error.issues) {
      const slot = issue.path[0] as keyof FlightSlotsType;
      if (slots[slot] !== undefined) { // 只报已填但格式错的
        invalid.push({ slot, reason: issue.message });
      }
    }
  }

  return {
    filled: slots,
    missing,
    invalid,
    isReady: missing.length === 0 && invalid.length === 0,
  };
}
```

### 4. 澄清问题生成器

```typescript
function buildClarificationQuestion(validation: SlotValidationResult): string | null {
  if (validation.isReady) return null;

  // 优先处理无效值（用户填了但格式/内容不对）
  if (validation.invalid.length > 0) {
    const { slot, reason } = validation.invalid[0];
    return `"${slots[slot]}" 格式有误（${reason}），${SLOT_META[slot].question}`;
  }

  // 一次只问一个必填槽位（渐进式收集，避免轰炸用户）
  const nextMissing = validation.missing[0];
  return SLOT_META[nextMissing].question;
}
```

### 5. 完整的多轮对话 Agent Loop

```typescript
interface ConversationState {
  slots:    Partial<FlightSlotsType>;
  history:  Array<{ role: "user" | "assistant"; content: string }>;
  turnCount: number;
}

async function flightBookingAgent(state: ConversationState, userInput: string): Promise<{
  response: string;
  state:    ConversationState;
  action?:  "book_flight";
  bookingData?: FlightSlotsType;
}> {
  // 1. 提取新实体，合并到现有槽位
  const updatedSlots = await extractEntities(userInput, state.slots);

  // 2. 验证
  const validation = validateSlots(updatedSlots);

  // 3. 更新状态
  const newState: ConversationState = {
    slots: updatedSlots,
    history: [
      ...state.history,
      { role: "user", content: userInput },
    ],
    turnCount: state.turnCount + 1,
  };

  // 4. 如果所有槽位齐全 → 执行预订
  if (validation.isReady) {
    const fullSlots = FlightSlots.parse(updatedSlots);
    const response = `✅ 好的！为您预订：
• 乘客：${fullSlots.passengerName}
• 路线：${fullSlots.fromCity} → ${fullSlots.toCity}
• 日期：${fullSlots.departDate}${fullSlots.departTime ? ` ${fullSlots.departTime}` : ""}
• 舱位：${{ economy: "经济舱", business: "商务舱", first: "头等舱" }[fullSlots.cabinClass]}

正在查询航班...`;

    return {
      response,
      state: { ...newState, history: [...newState.history, { role: "assistant", content: response }] },
      action: "book_flight",
      bookingData: fullSlots,
    };
  }

  // 5. 槽位不全 → 追问
  const question = buildClarificationQuestion(validation)!;
  
  // 显示已知信息的进度提示（让用户感知收集进展）
  const filledCount = Object.values(updatedSlots).filter(v => v !== undefined).length;
  const requiredCount = Object.values(SLOT_META).filter(m => m.required).length;
  const progressHint = filledCount > 0
    ? `（已收集 ${filledCount}/${requiredCount} 项必填信息）\n`
    : "";

  const response = progressHint + question;

  return {
    response,
    state: { ...newState, history: [...newState.history, { role: "assistant", content: response }] },
  };
}
```

---

## Python 版本（简化）

```python
from anthropic import Anthropic
from dataclasses import dataclass, field
from typing import Optional
import re

@dataclass
class BookingSlots:
    passenger_name: Optional[str] = None
    from_city:      Optional[str] = None
    to_city:        Optional[str] = None
    depart_date:    Optional[str] = None  # YYYY-MM-DD
    depart_time:    Optional[str] = None  # HH:MM
    cabin_class:    str = "economy"

SLOT_QUESTIONS = {
    "passenger_name": "请问乘客姓名是？",
    "from_city":      "从哪个城市出发？",
    "to_city":        "目的地是哪里？",
    "depart_date":    "出发日期是？（格式：2026-03-30）",
}

def get_missing_slots(slots: BookingSlots) -> list[str]:
    return [k for k in SLOT_QUESTIONS if getattr(slots, k) is None]

def validate_date(date_str: str) -> bool:
    return bool(re.match(r"^\d{4}-\d{2}-\d{2}$", date_str))

async def extract_and_fill(client: Anthropic, user_input: str, slots: BookingSlots) -> BookingSlots:
    """用 LLM 从用户输入提取实体，返回更新后的 slots"""
    response = await client.messages.create(
        model="claude-opus-4-5",
        max_tokens=400,
        tools=[{
            "name": "update_slots",
            "description": "更新机票预订信息",
            "input_schema": {
                "type": "object",
                "properties": {
                    "passenger_name": {"type": "string"},
                    "from_city":      {"type": "string"},
                    "to_city":        {"type": "string"},
                    "depart_date":    {"type": "string", "description": "YYYY-MM-DD"},
                },
            }
        }],
        tool_choice={"type": "auto"},
        messages=[{"role": "user", "content": f"提取预订信息：{user_input}"}],
    )
    
    for block in response.content:
        if block.type == "tool_use":
            for key, val in block.input.items():
                if val and getattr(slots, key) is None:  # 不覆盖已有值
                    setattr(slots, key, val)
    
    return slots

async def booking_agent_turn(client: Anthropic, user_input: str, slots: BookingSlots) -> tuple[str, BookingSlots, bool]:
    slots = await extract_and_fill(client, user_input, slots)
    missing = get_missing_slots(slots)
    
    # 验证日期格式
    if slots.depart_date and not validate_date(slots.depart_date):
        return f"日期格式不对，请重新输入（格式：2026-03-30）", slots, False
    
    if not missing:
        return f"✅ 预订确认：{slots.passenger_name}，{slots.from_city}→{slots.to_city}，{slots.depart_date}", slots, True
    
    return SLOT_QUESTIONS[missing[0]], slots, False
```

---

## OpenClaw 实战：Inline Buttons 加速槽位填充

纯文字追问有时太慢。对于枚举类型的槽位，直接给按钮：

```typescript
// OpenClaw message tool + inlineButtons
async function askCabinClass(chatId: string) {
  await sendMessage({
    target: chatId,
    message: "请选择舱位：",
    // OpenClaw capabilities: inlineButtons
    replyMarkup: {
      inline_keyboard: [[
        { text: "🪑 经济舱", callback_data: "cabin:economy" },
        { text: "💼 商务舱", callback_data: "cabin:business" },
        { text: "👑 头等舱", callback_data: "cabin:first" },
      ]],
    },
  });
}

// Callback 处理
function handleCallback(data: string, state: ConversationState) {
  if (data.startsWith("cabin:")) {
    const cabinClass = data.split(":")[1];
    return { ...state, slots: { ...state.slots, cabinClass } };
  }
}
```

---

## 最佳实践总结

| 原则 | 说明 |
|------|------|
| **一次只问一个** | 不要一次问三个问题，用户体验差且容易漏 |
| **先提取再追问** | 先尽力从当前输入提取，避免问用户已说过的 |
| **不覆盖已确认值** | merge 时保护已填槽位，防止 LLM 幻觉覆盖 |
| **显示进度** | 告知用户"还差几项"，减少挫败感 |
| **枚举用按钮** | cabin_class、priority 等固定选项用 inline buttons |
| **设置最大轮次** | 超过 8 轮仍未完成，考虑简化或放弃 |
| **验证后确认** | 所有槽位齐全后，回显摘要让用户确认一遍 |

---

## 与相关课程的关系

- **对话修复与澄清**（课程 ~110）：澄清是槽位填充的上游，先判断意图再收集槽位
- **意图识别与路由**（课程 ~90）：不同意图对应不同槽位 Schema
- **Tool Schema Design**（课程 ~22）：槽位 Schema 本质就是工具参数设计
- **Human-in-the-Loop**（课程 ~31）：槽位填充是最常见的 HITL 场景之一

---

## 小结

槽位填充是 Agent 从"能用"到"好用"的关键一步：

1. **定义槽位 Schema**：用 Zod/JSON Schema，区分必填和可选
2. **LLM 提取实体**：工具调用模式，只提取确定信息
3. **增量合并**：保护已有值，不覆盖
4. **渐进追问**：一次一个，优先无效→必填
5. **枚举用按钮**：减少输入摩擦
6. **回显确认**：所有槽位齐全后让用户确认

**核心思想：用结构化状态机管理对话进展，而不是让 LLM 在对话历史里自由发挥。**
