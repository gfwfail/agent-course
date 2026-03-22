# 111 - Agent Schema 驱动工具自动生成（Schema-Driven Tool Auto-Generation）

> 与其手写几十个工具函数，不如让 OpenAPI / JSON Schema 自动生成它们。

---

## 为什么需要这个模式？

生产级 Agent 可能需要调用几十甚至上百个 API。如果每个工具都手写：

```typescript
// 手写方式：重复、脆弱、难维护
const getUserTool = {
  name: "get_user",
  description: "Get user by ID",
  input_schema: {
    type: "object",
    properties: {
      user_id: { type: "string", description: "User ID" }
    },
    required: ["user_id"]
  }
};
```

一旦 API 变动，工具 schema 和实现都要同步更新——维护噩梦。

**Schema 驱动的方案：** 从 OpenAPI spec 自动生成工具定义 + 执行器，API 更新时重新生成即可。

---

## 核心架构

```
OpenAPI Spec (YAML/JSON)
        ↓
  SchemaParser         ← 解析路径、参数、请求体
        ↓
  ToolGenerator        ← 生成 Anthropic 工具定义
        ↓
  ToolExecutor         ← 生成对应的 HTTP 执行器
        ↓
  ToolRegistry         ← 注册到 Agent
```

---

## 实现：OpenAPI → Agent Tools

### 1. 解析 OpenAPI Spec

```typescript
import SwaggerParser from "@apidevtools/swagger-parser";

interface GeneratedTool {
  definition: Anthropic.Tool;
  executor: (input: Record<string, unknown>) => Promise<unknown>;
}

async function generateToolsFromOpenAPI(
  specUrl: string,
  baseUrl: string,
  headers: Record<string, string> = {}
): Promise<Map<string, GeneratedTool>> {
  const api = await SwaggerParser.dereference(specUrl);
  const tools = new Map<string, GeneratedTool>();

  for (const [path, pathItem] of Object.entries(api.paths ?? {})) {
    for (const method of ["get", "post", "put", "patch", "delete"] as const) {
      const operation = pathItem[method];
      if (!operation || !operation.operationId) continue;

      const tool = buildTool(operation, path, method, baseUrl, headers);
      tools.set(operation.operationId, tool);
    }
  }

  return tools;
}

function buildTool(
  operation: OpenAPIV3.OperationObject,
  path: string,
  method: string,
  baseUrl: string,
  headers: Record<string, string>
): GeneratedTool {
  // 1. 构建 input_schema（合并 path params + query params + request body）
  const properties: Record<string, unknown> = {};
  const required: string[] = [];

  // Path 参数：/users/{user_id} → user_id 参数
  for (const param of operation.parameters ?? []) {
    const p = param as OpenAPIV3.ParameterObject;
    properties[p.name] = {
      type: (p.schema as OpenAPIV3.SchemaObject)?.type ?? "string",
      description: p.description ?? p.name,
    };
    if (p.required) required.push(p.name);
  }

  // Request body（POST/PUT/PATCH）
  if (operation.requestBody) {
    const body = operation.requestBody as OpenAPIV3.RequestBodyObject;
    const jsonSchema = body.content?.["application/json"]?.schema as OpenAPIV3.SchemaObject;
    if (jsonSchema?.properties) {
      for (const [key, schema] of Object.entries(jsonSchema.properties)) {
        properties[`body_${key}`] = {
          ...(schema as object),
          description: `Request body: ${key}`,
        };
      }
      if (jsonSchema.required) {
        required.push(...jsonSchema.required.map((k) => `body_${k}`));
      }
    }
  }

  const definition: Anthropic.Tool = {
    name: operation.operationId!,
    description: operation.summary ?? operation.description ?? operation.operationId!,
    input_schema: {
      type: "object",
      properties,
      required: required.length > 0 ? required : undefined,
    },
  };

  // 2. 构建执行器
  const executor = async (input: Record<string, unknown>) => {
    let resolvedPath = path;
    const queryParams: Record<string, string> = {};
    const bodyParams: Record<string, unknown> = {};

    for (const [key, value] of Object.entries(input)) {
      if (path.includes(`{${key}}`)) {
        // Path 参数替换
        resolvedPath = resolvedPath.replace(`{${key}}`, String(value));
      } else if (key.startsWith("body_")) {
        // Request body
        bodyParams[key.slice(5)] = value;
      } else {
        // Query 参数
        queryParams[key] = String(value);
      }
    }

    const url = new URL(baseUrl + resolvedPath);
    for (const [k, v] of Object.entries(queryParams)) {
      url.searchParams.set(k, v);
    }

    const response = await fetch(url.toString(), {
      method: method.toUpperCase(),
      headers: {
        "Content-Type": "application/json",
        ...headers,
      },
      body: ["post", "put", "patch"].includes(method)
        ? JSON.stringify(bodyParams)
        : undefined,
    });

    if (!response.ok) {
      throw new Error(`API Error ${response.status}: ${await response.text()}`);
    }

    return response.json();
  };

  return { definition, executor };
}
```

### 2. 集成到 Agent

```typescript
class SchemaBackedAgent {
  private tools: Map<string, GeneratedTool>;
  private client: Anthropic;

  async init(specUrl: string, apiBaseUrl: string, apiKey: string) {
    this.tools = await generateToolsFromOpenAPI(specUrl, apiBaseUrl, {
      Authorization: `Bearer ${apiKey}`,
    });

    console.log(`✅ 从 OpenAPI spec 生成了 ${this.tools.size} 个工具`);
    for (const [name] of this.tools) {
      console.log(`  - ${name}`);
    }
  }

  async run(userMessage: string): Promise<string> {
    const toolDefinitions = [...this.tools.values()].map((t) => t.definition);
    const messages: Anthropic.MessageParam[] = [
      { role: "user", content: userMessage },
    ];

    while (true) {
      const response = await this.client.messages.create({
        model: "claude-opus-4-5",
        max_tokens: 4096,
        tools: toolDefinitions,
        messages,
      });

      if (response.stop_reason === "end_turn") {
        return response.content
          .filter((b) => b.type === "text")
          .map((b) => (b as Anthropic.TextBlock).text)
          .join("");
      }

      // 处理工具调用
      const toolResults: Anthropic.ToolResultBlockParam[] = [];
      for (const block of response.content) {
        if (block.type !== "tool_use") continue;

        const tool = this.tools.get(block.name);
        if (!tool) {
          toolResults.push({
            type: "tool_result",
            tool_use_id: block.id,
            content: `Tool "${block.name}" not found`,
            is_error: true,
          });
          continue;
        }

        try {
          const result = await tool.executor(block.input as Record<string, unknown>);
          toolResults.push({
            type: "tool_result",
            tool_use_id: block.id,
            content: JSON.stringify(result),
          });
        } catch (err) {
          toolResults.push({
            type: "tool_result",
            tool_use_id: block.id,
            content: String(err),
            is_error: true,
          });
        }
      }

      messages.push({ role: "assistant", content: response.content });
      messages.push({ role: "user", content: toolResults });
    }
  }
}
```

---

## 真实案例：GitHub API Agent

```typescript
// 直接用 GitHub 的官方 OpenAPI spec
const agent = new SchemaBackedAgent();
await agent.init(
  "https://raw.githubusercontent.com/github/rest-api-description/main/descriptions/api.github.com/api.github.com.json",
  "https://api.github.com",
  process.env.GITHUB_TOKEN!
);

// GitHub spec 有 700+ 个端点，自动全部变成工具
// 但 Claude 上下文有限，需要筛选

const result = await agent.run(
  "列出 anthropics/anthropic-sdk-python 最近5个 PR，告诉我哪个改动最大"
);
```

### 工具过滤：别把 700 个工具全塞给 LLM

```typescript
async function selectRelevantTools(
  userQuery: string,
  allTools: Map<string, GeneratedTool>,
  maxTools = 20
): Promise<Anthropic.Tool[]> {
  // 方案1：关键词匹配（简单高效）
  const keywords = userQuery.toLowerCase().split(/\s+/);
  const scored = [...allTools.entries()].map(([name, tool]) => {
    const text = `${name} ${tool.definition.description}`.toLowerCase();
    const score = keywords.filter((kw) => text.includes(kw)).length;
    return { name, tool, score };
  });

  return scored
    .sort((a, b) => b.score - a.score)
    .slice(0, maxTools)
    .map((s) => s.tool.definition);

  // 方案2：向量相似度（精准但需要 embedding）
  // 见 Lesson 46: Semantic Tool Selection
}
```

---

## 进阶：从 JSON Schema 生成嵌套工具

有些 API 有复杂的嵌套请求体，需要递归处理：

```typescript
function flattenSchema(
  schema: OpenAPIV3.SchemaObject,
  prefix = ""
): { properties: Record<string, unknown>; required: string[] } {
  const properties: Record<string, unknown> = {};
  const required: string[] = [];

  if (!schema.properties) return { properties, required };

  for (const [key, value] of Object.entries(schema.properties)) {
    const fullKey = prefix ? `${prefix}_${key}` : key;
    const v = value as OpenAPIV3.SchemaObject;

    if (v.type === "object" && v.properties) {
      // 递归展平嵌套对象
      const nested = flattenSchema(v, fullKey);
      Object.assign(properties, nested.properties);
      if (schema.required?.includes(key)) {
        required.push(...nested.required);
      }
    } else if (v.type === "array" && (v.items as OpenAPIV3.SchemaObject)?.type === "object") {
      // 数组对象：接受 JSON 字符串
      properties[fullKey] = {
        type: "string",
        description: `${v.description ?? key} (JSON array)`,
      };
    } else {
      properties[fullKey] = {
        type: v.type ?? "string",
        description: v.description ?? key,
        enum: v.enum,
      };
      if (schema.required?.includes(key)) required.push(fullKey);
    }
  }

  return { properties, required };
}
```

---

## OpenClaw 实现：mcporter

OpenClaw 的 `mcporter` 技能本质上就是这个模式的落地：

```bash
# mcporter 从 MCP server 的 tool list 自动生成调用接口
mcporter call <server> <tool> --param key=value

# 内部原理：
# 1. 调用 MCP server 的 tools/list
# 2. 解析 inputSchema（就是 JSON Schema）
# 3. 生成调用代码 / 直接执行
```

MCP 协议 = 标准化的 Schema 驱动工具接口。每个 MCP server 都是一个"自描述工具包"。

---

## pi-mono 参考实现

pi-mono 中的 `ToolRegistry` 支持从外部 schema 注册工具：

```typescript
// pi-mono: packages/core/src/tools/registry.ts
export class ToolRegistry {
  registerFromSchema(schemas: ToolSchema[], executor: ToolExecutor) {
    for (const schema of schemas) {
      this.register({
        name: schema.name,
        description: schema.description,
        inputSchema: schema.parameters,
        // 单一执行器处理所有工具，用 name 区分
        execute: (input) => executor(schema.name, input),
      });
    }
  }
}
```

---

## 最佳实践总结

| 场景 | 建议 |
|------|------|
| API < 20 个端点 | 手写工具，更精细的控制 |
| API 20~200 个端点 | Schema 生成 + 关键词过滤 |
| API > 200 个端点 | Schema 生成 + 向量检索 + 懒加载 |
| API 频繁变动 | Schema 生成 + CI 自动重新生成 |
| 多个 API 集成 | 统一 SchemaLoader + 命名空间区分 |

### 注意事项

1. **工具名唯一性**：不同 API 的 `operationId` 可能冲突，加命名空间前缀：`github_list_repos`
2. **描述质量**：LLM 选择工具靠 description，OpenAPI 的 summary 质量参差不齐，必要时手动补充
3. **Token 控制**：700 个工具的 schema 可能超 100K tokens，必须过滤
4. **安全边界**：生成的工具应只暴露必要的端点，DELETE 类操作要额外确认

---

## 课后实践

1. 找一个有 OpenAPI spec 的公共 API（Stripe、Notion、Linear 都有），运行代码自动生成工具列表
2. 添加工具过滤逻辑，让 Agent 根据用户意图只加载相关工具
3. 思考：如何为自动生成的工具加上速率限制和重试？（参考 Lesson 34: Rate Limiting & Backoff）

---

*下一讲预告：Agent 工具调用可观测性 Dashboard — 用 Prometheus + Grafana 实时监控你的 Agent 在干什么*
