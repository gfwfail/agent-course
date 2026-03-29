# 172 - Agent GraphQL 集成模式

> GraphQL Integration Patterns：Schema 自内省 → 工具自动生成 → 类型安全查询 → 订阅实时数据

---

## 为什么 GraphQL 对 Agent 特别友好？

REST API 需要 Agent 记住几十个端点、参数格式、版本差异。GraphQL 只有 **一个端点**，Schema 就是完整的文档：

| 特性 | REST | GraphQL |
|------|------|---------|
| 端点数量 | 数十个 | 1 个 |
| 文档 | 手写 OpenAPI | Schema 自带 |
| 按需取字段 | ❌ 过多/过少 | ✅ 精确控制 |
| 实时推送 | 轮询或 SSE | Subscription |
| 类型系统 | 弱 | 强（SDL） |

Agent 可以在运行时 **Introspect** Schema，自动生成工具列表——不需要人工维护工具定义。

---

## 核心架构

```
GraphQL Endpoint
      │
      ▼
┌─────────────────┐
│ SchemaInspector │ ← Introspection Query
└────────┬────────┘
         │ types + queries + mutations
         ▼
┌─────────────────┐
│  ToolGenerator  │ ← 自动生成 Tool Schema
└────────┬────────┘
         │ tools[]
         ▼
┌─────────────────┐     ┌──────────────────┐
│  Agent Loop     │────▶│  GraphQLExecutor  │
└─────────────────┘     └──────────────────┘
         │                       │
         ▼                       ▼
   LLM 选择工具            发送 Query/Mutation
                          解析 typed 结果
```

---

## 实战代码：Schema 自内省 + 工具生成

```typescript
// graphql-agent/src/graphql-tool-factory.ts

import { GraphQLSchema, buildClientSchema, getIntrospectionQuery, IntrospectionQuery } from 'graphql';

interface GeneratedTool {
  name: string;
  description: string;
  inputSchema: {
    type: 'object';
    properties: Record<string, unknown>;
    required: string[];
  };
  _gqlOperation: string; // 底层 GQL 字符串
}

export class GraphQLToolFactory {
  private schema: GraphQLSchema | null = null;
  private endpoint: string;
  private headers: Record<string, string>;

  constructor(endpoint: string, headers: Record<string, string> = {}) {
    this.endpoint = endpoint;
    this.headers = headers;
  }

  // Step 1: 内省获取完整 Schema
  async loadSchema(): Promise<void> {
    const result = await this.gqlRequest<{ data: IntrospectionQuery }>(
      getIntrospectionQuery()
    );
    this.schema = buildClientSchema(result.data);
    console.log(`[GraphQL] Schema loaded: ${Object.keys(this.schema.getTypeMap()).length} types`);
  }

  // Step 2: 把 Query/Mutation 转换成 Agent Tool
  generateTools(operationNames?: string[]): GeneratedTool[] {
    if (!this.schema) throw new Error('Schema not loaded. Call loadSchema() first.');

    const tools: GeneratedTool[] = [];
    const queryType = this.schema.getQueryType();
    const mutationType = this.schema.getMutationType();

    for (const [typeName, typeObj] of [[queryType, 'query'], [mutationType, 'mutation']] as const) {
      if (!typeName) continue;
      const fields = typeName.getFields();
      
      for (const [fieldName, field] of Object.entries(fields)) {
        // 按需过滤
        if (operationNames && !operationNames.includes(fieldName)) continue;

        const tool = this.fieldToTool(fieldName, field, typeObj);
        tools.push(tool);
      }
    }

    return tools;
  }

  private fieldToTool(name: string, field: any, opType: string): GeneratedTool {
    const properties: Record<string, unknown> = {};
    const required: string[] = [];

    for (const arg of field.args ?? []) {
      properties[arg.name] = this.gqlTypeToJsonSchema(arg.type);
      if (arg.type.toString().endsWith('!')) {
        required.push(arg.name);
      }
    }

    // 自动生成 GQL 操作字符串
    const argDefs = field.args?.map((a: any) => `$${a.name}: ${a.type}`).join(', ') ?? '';
    const argUses = field.args?.map((a: any) => `${a.name}: $${a.name}`).join(', ') ?? '';
    const gqlOperation = `
      ${opType} ${name}${argDefs ? `(${argDefs})` : ''} {
        ${name}${argUses ? `(${argUses})` : ''} {
          ${this.buildSelectionSet(field.type)}
        }
      }
    `.trim();

    return {
      name: `gql_${name}`,
      description: field.description ?? `Execute GraphQL ${opType}: ${name}`,
      inputSchema: { type: 'object', properties, required },
      _gqlOperation: gqlOperation,
    };
  }

  private gqlTypeToJsonSchema(gqlType: any): unknown {
    const typeName = gqlType.ofType?.name ?? gqlType.name ?? '';
    const typeMap: Record<string, string> = {
      String: 'string',
      Int: 'integer',
      Float: 'number',
      Boolean: 'boolean',
      ID: 'string',
    };
    return { type: typeMap[typeName] ?? 'string' };
  }

  private buildSelectionSet(type: any, depth = 0): string {
    if (depth > 2) return ''; // 防止无限递归
    const fields = type.getFields?.();
    if (!fields) return '';
    return Object.keys(fields)
      .filter(f => !fields[f].type.getFields) // 只选标量
      .slice(0, 10) // 最多 10 个字段，控制 token
      .join('\n          ');
  }

  // 执行 GQL 请求
  async execute(tool: GeneratedTool, variables: Record<string, unknown>): Promise<unknown> {
    const result = await this.gqlRequest<{ data: unknown; errors?: unknown[] }>(
      tool._gqlOperation,
      variables
    );
    if (result.errors?.length) {
      throw new Error(`GraphQL errors: ${JSON.stringify(result.errors)}`);
    }
    return result.data;
  }

  private async gqlRequest<T>(query: string, variables?: Record<string, unknown>): Promise<T> {
    const resp = await fetch(this.endpoint, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', ...this.headers },
      body: JSON.stringify({ query, variables }),
    });
    if (!resp.ok) throw new Error(`HTTP ${resp.status}: ${await resp.text()}`);
    return resp.json() as T;
  }
}
```

---

## 集成到 Agent Loop

```typescript
// graphql-agent/src/agent.ts

import Anthropic from '@anthropic-ai/sdk';
import { GraphQLToolFactory } from './graphql-tool-factory';

async function runGraphQLAgent(userMessage: string) {
  const client = new Anthropic();

  // 1. 加载 Schema，生成工具（只暴露需要的 operation）
  const factory = new GraphQLToolFactory(
    'https://api.example.com/graphql',
    { Authorization: `Bearer ${process.env.API_TOKEN}` }
  );
  await factory.loadSchema();
  
  const generatedTools = factory.generateTools([
    'getUser', 'listOrders', 'searchProducts', 'createOrder', 'updateProfile'
  ]);

  // 2. 转换为 Anthropic tool 格式
  const tools: Anthropic.Tool[] = generatedTools.map(t => ({
    name: t.name,
    description: t.description,
    input_schema: t.inputSchema,
  }));

  const messages: Anthropic.MessageParam[] = [
    { role: 'user', content: userMessage }
  ];

  // 3. Agent Loop
  while (true) {
    const response = await client.messages.create({
      model: 'claude-opus-4-5',
      max_tokens: 4096,
      tools,
      messages,
    });

    messages.push({ role: 'assistant', content: response.content });

    if (response.stop_reason === 'end_turn') break;
    if (response.stop_reason !== 'tool_use') break;

    // 4. 执行工具调用
    const toolResults: Anthropic.ToolResultBlockParam[] = [];
    for (const block of response.content) {
      if (block.type !== 'tool_use') continue;

      const tool = generatedTools.find(t => t.name === block.name);
      if (!tool) {
        toolResults.push({ type: 'tool_result', tool_use_id: block.id, content: 'Tool not found', is_error: true });
        continue;
      }

      try {
        const result = await factory.execute(tool, block.input as Record<string, unknown>);
        toolResults.push({
          type: 'tool_result',
          tool_use_id: block.id,
          content: JSON.stringify(result),
        });
      } catch (err) {
        toolResults.push({
          type: 'tool_result',
          tool_use_id: block.id,
          content: `Error: ${err}`,
          is_error: true,
        });
      }
    }

    messages.push({ role: 'user', content: toolResults });
  }

  // 提取最终文本
  const last = messages[messages.length - 1];
  const content = Array.isArray(last.content) ? last.content : [];
  return content.find((b: any) => b.type === 'text')?.text ?? '';
}

// 使用示例
runGraphQLAgent('查询用户 123 的最近 3 笔订单，并告诉我总金额')
  .then(console.log);
```

---

## GraphQL Subscription：实时数据推送

Agent 处理长任务时，可以通过 Subscription 接收实时更新，而不是轮询：

```typescript
// graphql-agent/src/subscription-tool.ts
import { createClient } from 'graphql-ws';

interface SubscriptionHandle {
  unsubscribe: () => void;
  results: unknown[];
}

export function createSubscriptionTool(wsEndpoint: string, token: string) {
  const wsClient = createClient({
    url: wsEndpoint,
    connectionParams: { Authorization: `Bearer ${token}` },
  });

  return {
    name: 'gql_subscribe_order_updates',
    description: '订阅订单状态实时更新，返回 handle ID',
    execute: async (orderId: string, onUpdate: (data: unknown) => void): Promise<SubscriptionHandle> => {
      const results: unknown[] = [];
      
      const unsubscribe = wsClient.subscribe(
        {
          query: `subscription OnOrderUpdate($orderId: ID!) {
            orderUpdated(orderId: $orderId) {
              id status updatedAt
            }
          }`,
          variables: { orderId },
        },
        {
          next: (data) => {
            results.push(data);
            onUpdate(data); // 实时回调通知 Agent
          },
          error: (err) => console.error('[Subscription Error]', err),
          complete: () => console.log('[Subscription] Completed'),
        }
      );

      return { unsubscribe, results };
    }
  };
}

// 在 Agent 中使用
async function monitorOrder(orderId: string) {
  const subscriptionTool = createSubscriptionTool(
    'wss://api.example.com/graphql',
    process.env.API_TOKEN!
  );

  const handle = await subscriptionTool.execute(orderId, (update) => {
    console.log(`[Agent] 订单状态更新:`, update);
    // 触发 Agent 下一步动作（如发送通知、更新内部状态）
  });

  // 5 分钟后取消订阅
  setTimeout(() => handle.unsubscribe(), 5 * 60 * 1000);
}
```

---

## OpenClaw 实战：用 GraphQL Skill 查询 GitHub

OpenClaw 的 GitHub Skill 底层就是 GraphQL——GitHub API v4 是纯 GraphQL：

```typescript
// OpenClaw skill 内部示例
const GITHUB_GQL = 'https://api.github.com/graphql';

const factory = new GraphQLToolFactory(GITHUB_GQL, {
  Authorization: `bearer ${process.env.GITHUB_TOKEN}`,
});

await factory.loadSchema();

// 只暴露需要的操作，避免工具列表爆炸
const tools = factory.generateTools(['repository', 'user', 'search']);
```

**用 GQL 查询比用 REST 省 60% token**：精确选字段，不返回无用数据。

---

## 关键优化：工具列表精简

GraphQL Schema 可能有数百个 operation，直接全部暴露会：
- 超出工具数量上限（Anthropic 限制 64 个）
- 浪费 Context Token

**策略**：

```typescript
// 语义化工具过滤
function filterToolsByIntent(
  allTools: GeneratedTool[],
  userIntent: string
): GeneratedTool[] {
  const intentKeywords = userIntent.toLowerCase().split(' ');
  
  return allTools
    .map(tool => ({
      tool,
      score: intentKeywords.filter(kw => 
        tool.name.includes(kw) || tool.description.toLowerCase().includes(kw)
      ).length,
    }))
    .filter(({ score }) => score > 0)
    .sort((a, b) => b.score - a.score)
    .slice(0, 10) // 最多返回 10 个相关工具
    .map(({ tool }) => tool);
}
```

---

## 错误处理与类型安全

```typescript
// 用 zod 验证 GQL 响应
import { z } from 'zod';

const OrderSchema = z.object({
  id: z.string(),
  status: z.enum(['PENDING', 'PAID', 'SHIPPED', 'DELIVERED', 'CANCELLED']),
  total: z.number(),
  createdAt: z.string(),
});

async function getOrderSafe(factory: GraphQLToolFactory, orderId: string) {
  const raw = await factory.execute(
    { name: 'gql_getOrder', _gqlOperation: GET_ORDER_QUERY } as any,
    { id: orderId }
  );
  
  // 运行时验证，防止 Schema 漂移破坏 Agent
  const parsed = OrderSchema.safeParse((raw as any)?.getOrder);
  if (!parsed.success) {
    throw new Error(`Schema validation failed: ${parsed.error.message}`);
  }
  return parsed.data;
}
```

---

## 对比 NL2SQL / REST

| 场景 | 推荐方案 |
|------|----------|
| 内部数据库 | NL2SQL（Lesson 99） |
| 第三方 REST API | Schema-Driven Tool Auto-Gen（Lesson 159） |
| **第三方 GraphQL API** | **本课：GraphQL Tool Factory** |
| 实时数据推送 | **本课：GraphQL Subscription** |

---

## 总结

1. **一次 Introspection，永久工具定义** — Schema 即文档，Agent 自助获取
2. **精准字段选择** — 比 REST 省 60% token，只取需要的数据
3. **Subscription 赋能实时 Agent** — 不轮询，事件驱动
4. **工具语义过滤** — 按意图动态缩减工具列表，不超上限
5. **Zod 运行时验证** — 防止 Schema 漂移悄悄破坏 Agent

GraphQL 是 Agent 集成外部 API 的最佳选择之一——类型安全、自文档化、按需取数。
