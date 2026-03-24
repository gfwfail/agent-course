# 131 - Agent 多跳推理与工具链编排（Multi-Hop Reasoning & Tool Chain Orchestration）

> 复杂问题需要多轮工具调用，每一步的结果驱动下一步的决策——这就是多跳推理。

---

## 🎯 为什么需要多跳推理？

用户问："上周平台利润最高的盲盒，它的开箱概率分布是什么？"

这一句话需要 **3 跳**：
1. **跳 1**：查数据库 → 找出上周利润最高的盲盒 ID
2. **跳 2**：用 ID 查该盲盒的物品列表
3. **跳 3**：计算概率分布，生成报告

单跳工具调用无法完成，必须链式编排。

---

## 🔗 多跳推理的三种模式

### 模式 1：线性链（Sequential Chain）

```
Tool A → result_A → Tool B(result_A) → result_B → Tool C(result_B)
```

最简单，每步依赖上一步输出。

### 模式 2：扇出-聚合（Fan-Out + Reduce）

```
Tool A → [id1, id2, id3]
              ↓
   Tool B(id1) | Tool B(id2) | Tool B(id3)  ← 并发
              ↓
         Reduce/Merge
```

先拿到 ID 列表，再并发查详情。

### 模式 3：条件分支（Conditional Branching）

```
Tool A → result
   if result.type == "box":  → Tool B_box
   if result.type == "item": → Tool B_item
```

根据中间结果动态选择下一个工具。

---

## 💻 代码实现：线性链

### learn-claude-code 风格（Python）

```python
import anthropic
import json

client = anthropic.Anthropic()

tools = [
    {
        "name": "query_top_profitable_box",
        "description": "查询指定时间段内利润最高的盲盒，返回 box_id 和利润",
        "input_schema": {
            "type": "object",
            "properties": {
                "days": {"type": "integer", "description": "查询最近N天"}
            },
            "required": ["days"]
        }
    },
    {
        "name": "get_box_probability_distribution",
        "description": "根据 box_id 获取盲盒的物品概率分布",
        "input_schema": {
            "type": "object",
            "properties": {
                "box_id": {"type": "integer", "description": "盲盒ID"}
            },
            "required": ["box_id"]
        }
    },
    {
        "name": "generate_distribution_report",
        "description": "生成概率分布的可读报告",
        "input_schema": {
            "type": "object",
            "properties": {
                "box_name": {"type": "string"},
                "distribution": {"type": "array", "items": {"type": "object"}}
            },
            "required": ["box_name", "distribution"]
        }
    }
]

# 模拟工具执行
def execute_tool(name: str, inputs: dict) -> str:
    if name == "query_top_profitable_box":
        # 实际中：SELECT box_id, SUM(price)-SUM(item_price) profit FROM ...
        return json.dumps({"box_id": 42, "box_name": "AK精英箱", "profit": 15820.5})
    
    elif name == "get_box_probability_distribution":
        box_id = inputs["box_id"]
        # 实际中：SELECT item_name, probability FROM box_items WHERE box_id = ?
        return json.dumps({
            "box_id": box_id,
            "items": [
                {"name": "AK-47 | 红线", "rarity": "受限", "probability": 0.1598},
                {"name": "M4A4 | 热带风暴", "rarity": "受限", "probability": 0.1598},
                {"name": "AWP | 幻彩三型", "rarity": "隐秘", "probability": 0.0064},
                {"name": "★ 刺刀", "rarity": "罕见特殊", "probability": 0.0026},
            ]
        })
    
    elif name == "generate_distribution_report":
        dist = inputs["distribution"]
        lines = [f"- {i['name']} ({i['rarity']}): {i['probability']*100:.2f}%" 
                 for i in dist]
        return f"📊 {inputs['box_name']} 概率分布：\n" + "\n".join(lines)
    
    return "error: unknown tool"


def multi_hop_agent(question: str) -> str:
    """多跳推理 Agent Loop"""
    messages = [{"role": "user", "content": question}]
    hop_count = 0
    
    while True:
        response = client.messages.create(
            model="claude-opus-4-5",
            max_tokens=2048,
            tools=tools,
            messages=messages
        )
        
        # 收集这一跳的所有工具调用
        tool_calls = [b for b in response.content if b.type == "tool_use"]
        
        if not tool_calls:
            # 没有工具调用 → LLM 已生成最终答案
            final_text = next(b.text for b in response.content if b.type == "text")
            print(f"\n✅ 完成，共 {hop_count} 跳")
            return final_text
        
        hop_count += len(tool_calls)
        print(f"\n🔗 第 {hop_count} 跳：调用 {[t.name for t in tool_calls]}")
        
        # 执行工具，收集结果
        tool_results = []
        for tc in tool_calls:
            result = execute_tool(tc.name, tc.input)
            print(f"   ↳ {tc.name}({tc.input}) → {result[:80]}...")
            tool_results.append({
                "type": "tool_result",
                "tool_use_id": tc.id,
                "content": result
            })
        
        # 把 assistant 回复 + tool_results 追加到消息历史
        messages.append({"role": "assistant", "content": response.content})
        messages.append({"role": "user", "content": tool_results})


# 运行
result = multi_hop_agent("上周平台利润最高的盲盒，它的开箱概率分布是什么？")
print(result)
```

---

## 💻 扇出并发：pi-mono 风格（TypeScript）

```typescript
import Anthropic from "@anthropic-ai/sdk";

// 关键：并发执行同一跳的多个工具调用
async function executeToolsConcurrently(
  toolCalls: Anthropic.ToolUseBlock[]
): Promise<Anthropic.ToolResultBlockParam[]> {
  // 所有工具调用并发执行，不串行等待
  const results = await Promise.all(
    toolCalls.map(async (tc) => {
      const result = await dispatchTool(tc.name, tc.input);
      return {
        type: "tool_result" as const,
        tool_use_id: tc.id,
        content: JSON.stringify(result),
      };
    })
  );
  return results;
}

async function multiHopAgent(question: string): Promise<string> {
  const client = new Anthropic();
  const messages: Anthropic.MessageParam[] = [
    { role: "user", content: question },
  ];

  let totalHops = 0;
  const MAX_HOPS = 10; // 防止无限循环

  while (totalHops < MAX_HOPS) {
    const response = await client.messages.create({
      model: "claude-opus-4-5",
      max_tokens: 4096,
      tools: TOOLS_DEFINITION,
      messages,
    });

    const toolCalls = response.content.filter(
      (b): b is Anthropic.ToolUseBlock => b.type === "tool_use"
    );

    if (toolCalls.length === 0 || response.stop_reason === "end_turn") {
      const text = response.content
        .filter((b): b is Anthropic.TextBlock => b.type === "text")
        .map((b) => b.text)
        .join("");
      console.log(`✅ 多跳完成，共 ${totalHops} 跳`);
      return text;
    }

    // 并发执行本跳所有工具调用
    totalHops += toolCalls.length;
    console.log(`🔗 跳 +${toolCalls.length}（并发）: ${toolCalls.map(t => t.name).join(", ")}`);
    
    const toolResults = await executeToolsConcurrently(toolCalls);

    messages.push({ role: "assistant", content: response.content });
    messages.push({ role: "user", content: toolResults });
  }

  throw new Error(`超过最大跳数限制 ${MAX_HOPS}`);
}
```

---

## 🎛️ OpenClaw 实战：技能中的多跳

OpenClaw 的 Skill 体系天然支持多跳——工具调用本身就是多跳的基础设施。

关键实践：

```
mysterybox skill 的典型多跳：
1. grafana_query("SELECT box_id, profit ...") → top_box_id
2. grafana_query("SELECT items WHERE box_id = ?", top_box_id) → item_list  
3. format_report(item_list) → markdown 表格
```

OpenClaw 主会话的每次工具调用序列就是一次多跳推理！

---

## ⚠️ 多跳的关键坑

### 坑 1：中间结果太大，撑爆 Context Window

```python
# ❌ 错误：把整个数据库结果丢进 messages
tool_result = {"rows": [...10000行数据...]}

# ✅ 正确：在工具层截断/摘要，只返回 LLM 需要的信息
tool_result = {
    "top_item": rows[0],
    "total_count": len(rows),
    "summary": "找到 10000 条记录，top 3 已列出"
}
```

### 坑 2：没有跳数上限，LLM 绕圈

```python
MAX_HOPS = 10  # 必须有！
if hop_count > MAX_HOPS:
    return "已达最大推理步数，请简化问题"
```

### 坑 3：工具依赖关系设计不清晰

设计工具时，每个工具的 `description` 要明确说明：
- **输入来自哪里**（前一跳的哪个字段）
- **输出给谁用**（后续哪个工具需要这个结果）

---

## 📊 跳数 vs 性能

| 跳数 | 典型延迟 | 建议 |
|------|---------|------|
| 1-2 跳 | 1-3s | 正常，无需优化 |
| 3-5 跳 | 3-10s | 考虑并发化同级工具调用 |
| 6+ 跳 | 10s+ | 重新设计工具粒度，或拆为子 Agent |

---

## 🧠 总结

多跳推理是 Agent 处理复杂问题的核心能力：

1. **线性链**：简单依赖，顺序执行
2. **扇出并发**：并行提速，`Promise.all` 是关键
3. **条件分支**：动态路由，让 LLM 决定下一跳
4. **防护**：跳数上限 + 中间结果压缩 = 稳定生产

> 💡 记住：工具的粒度决定跳数。工具太粗 → 信息爆炸；工具太细 → 跳数爆炸。找到平衡点。
