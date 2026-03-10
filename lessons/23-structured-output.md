# 23. Structured Output：结构化输出

> 让 Agent 返回可解析的数据，而不是自由文本

## 为什么需要结构化输出？

想象这个场景：你让 Agent 分析一段代码，提取所有的函数名和参数。

**自由文本输出：**
```
这段代码包含了 3 个函数：
1. fetchUser - 接收 userId 参数，返回用户信息
2. updateProfile - 需要 userId 和 data 两个参数
3. deleteAccount - 只需要 userId
```

好看是好看，但程序怎么解析？正则？祈祷 LLM 每次格式都一样？

**结构化输出：**
```json
{
  "functions": [
    {"name": "fetchUser", "params": ["userId"], "returns": "User"},
    {"name": "updateProfile", "params": ["userId", "data"], "returns": "void"},
    {"name": "deleteAccount", "params": ["userId"], "returns": "void"}
  ]
}
```

直接 `JSON.parse()`，完事。

## 三种实现方式

### 方式一：Prompt 引导（最弱但最通用）

最简单的方式 - 在 Prompt 里告诉 LLM 返回 JSON：

```typescript
const prompt = `
分析以下代码，提取函数信息。

**输出格式（严格 JSON）：**
{
  "functions": [
    {"name": "函数名", "params": ["参数1", "参数2"]}
  ]
}

只输出 JSON，不要任何其他文字。

代码：
${code}
`;
```

**问题：**
- LLM 可能加 markdown 代码块
- 可能多说几句废话
- 格式不一定稳定

**补救措施 - 后处理解析：**

```typescript
function extractJSON(text: string): any {
  // 尝试直接解析
  try {
    return JSON.parse(text);
  } catch {}
  
  // 提取 ```json ... ``` 代码块
  const codeBlockMatch = text.match(/```(?:json)?\s*([\s\S]*?)```/);
  if (codeBlockMatch) {
    try {
      return JSON.parse(codeBlockMatch[1].trim());
    } catch {}
  }
  
  // 提取第一个 { ... } 或 [ ... ]
  const jsonMatch = text.match(/[\[{][\s\S]*[\]}]/);
  if (jsonMatch) {
    try {
      return JSON.parse(jsonMatch[0]);
    } catch {}
  }
  
  throw new Error('Failed to extract JSON from response');
}
```

### 方式二：Tool 强制结构（推荐）

把"输出"伪装成"工具调用"。定义一个 `output` 工具，让 LLM 必须通过它返回数据：

```typescript
const outputTool = {
  name: 'submit_analysis',
  description: '提交代码分析结果',
  parameters: {
    type: 'object',
    properties: {
      functions: {
        type: 'array',
        items: {
          type: 'object',
          properties: {
            name: { type: 'string' },
            params: { type: 'array', items: { type: 'string' } },
            returns: { type: 'string' }
          },
          required: ['name', 'params']
        }
      }
    },
    required: ['functions']
  }
};

// 强制使用这个工具
const response = await client.chat.completions.create({
  model: 'gpt-4',
  messages: [...],
  tools: [{ type: 'function', function: outputTool }],
  tool_choice: { type: 'function', function: { name: 'submit_analysis' } }
});
```

**优势：**
- Schema 验证保证格式
- 不会有多余文字
- 类型安全

### 方式三：Native JSON Mode（最强）

现代 LLM 原生支持结构化输出：

**OpenAI (response_format):**
```typescript
const response = await client.chat.completions.create({
  model: 'gpt-4o',
  messages: [...],
  response_format: {
    type: 'json_schema',
    json_schema: {
      name: 'code_analysis',
      schema: {
        type: 'object',
        properties: {
          functions: {
            type: 'array',
            items: {
              type: 'object',
              properties: {
                name: { type: 'string' },
                params: { type: 'array', items: { type: 'string' } }
              },
              required: ['name', 'params']
            }
          }
        },
        required: ['functions']
      }
    }
  }
});

const data = JSON.parse(response.choices[0].message.content);
```

**Anthropic (tool_use 模式):**
```typescript
const response = await anthropic.messages.create({
  model: 'claude-sonnet-4-20250514',
  messages: [...],
  tools: [{
    name: 'output',
    description: 'Output the structured result',
    input_schema: {
      type: 'object',
      properties: {
        functions: { /* ... */ }
      }
    }
  }],
  tool_choice: { type: 'tool', name: 'output' }
});
```

## 实战：Agent 中的结构化决策

Agent 经常需要做结构化决策。比如 Claude Code 判断下一步行动：

```typescript
interface AgentDecision {
  action: 'tool_call' | 'respond' | 'ask_user' | 'done';
  reasoning: string;
  toolName?: string;
  toolArgs?: Record<string, unknown>;
  response?: string;
}

const decisionTool = {
  name: 'make_decision',
  description: '决定下一步行动',
  parameters: {
    type: 'object',
    properties: {
      action: {
        type: 'string',
        enum: ['tool_call', 'respond', 'ask_user', 'done']
      },
      reasoning: { type: 'string' },
      toolName: { type: 'string' },
      toolArgs: { type: 'object' },
      response: { type: 'string' }
    },
    required: ['action', 'reasoning']
  }
};
```

这样 Agent Loop 就能可靠地解析决策：

```typescript
async function agentStep(messages: Message[]): Promise<AgentDecision> {
  const response = await llm.chat({
    messages,
    tools: [decisionTool, ...otherTools],
    tool_choice: { type: 'function', function: { name: 'make_decision' } }
  });
  
  const toolCall = response.choices[0].message.tool_calls?.[0];
  return JSON.parse(toolCall.function.arguments) as AgentDecision;
}
```

## 常见坑点

### 1. Schema 太复杂导致失败

```typescript
// ❌ 嵌套太深，LLM 容易出错
{
  level1: {
    level2: {
      level3: {
        level4: { /* ... */ }
      }
    }
  }
}

// ✅ 扁平化
{
  items: [{ type: 'a', data: {...} }, { type: 'b', data: {...} }]
}
```

### 2. 忘记处理 null/undefined

```typescript
// Schema 里允许 null
properties: {
  optionalField: {
    type: ['string', 'null']  // 明确允许 null
  }
}
```

### 3. 数组可能为空

```typescript
// 代码里处理空数组
const functions = result.functions ?? [];
if (functions.length === 0) {
  console.log('No functions found');
}
```

### 4. 不同模型的 Schema 支持差异

```typescript
// 统一适配层
function getStructuredOutput(provider: string, schema: JSONSchema) {
  if (provider === 'openai') {
    return { response_format: { type: 'json_schema', json_schema: schema } };
  } else if (provider === 'anthropic') {
    return { 
      tools: [{ name: 'output', input_schema: schema }],
      tool_choice: { type: 'tool', name: 'output' }
    };
  }
  // fallback to prompt-based
  return { systemPrompt: `Output JSON matching: ${JSON.stringify(schema)}` };
}
```

## 总结

| 方式 | 可靠性 | 通用性 | 推荐场景 |
|------|--------|--------|----------|
| Prompt 引导 | ⭐⭐ | ⭐⭐⭐⭐⭐ | 老模型、简单场景 |
| Tool 强制 | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | 大多数场景 |
| Native JSON | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | 支持的模型、关键数据 |

**黄金法则：**
- 关键数据 → Native JSON Mode 或 Tool 强制
- 非关键展示 → Prompt 引导 + 后处理
- 永远有 fallback 解析逻辑

下节预告：**Tool Chaining 工具链** - 如何让多个工具协作完成复杂任务。
