# 74. Agent 契约测试（Contract Testing for Agents）

> 用契约保障 Agent、Tool、LLM 之间的接口稳定性，让多 Agent 系统可以独立迭代而不互相破坏

---

## 为什么需要契约测试？

Agent 系统由多个组件组成：

```
用户 → Agent A → Tool X → 外部 API
              ↓
           Agent B → Tool Y → 数据库
```

当你修改 Tool X 的输出格式，Agent A 可能悄悄崩溃。  
当你升级 Agent B，Agent A 传给它的参数可能不再合法。

**单元测试**：测试每个组件内部逻辑  
**集成测试**：测试整个链路，慢且脆弱  
**契约测试**：只测试组件之间的"接口约定"，快速且精准

---

## 什么是契约？

契约（Contract）= **消费者对提供者的期望**

```
消费者（Consumer）：调用方，定义"我需要什么格式"
提供者（Provider）：被调用方，承诺"我会提供什么格式"
```

在 Agent 系统中，契约出现在：

| 交互点 | 消费者 | 提供者 |
|-------|--------|--------|
| Tool 调用 | Agent | Tool 函数 |
| 子 Agent 调用 | 父 Agent | 子 Agent |
| LLM 结构化输出 | 解析代码 | LLM |
| MCP Tool | Agent | MCP Server |
| Webhook 回调 | Agent | 外部服务 |

---

## 实战：Tool 契约测试

### 定义契约（JSON Schema）

```typescript
// contracts/tools/search.contract.ts
export const SearchToolContract = {
  // 消费者期望的输入格式
  input: {
    type: "object",
    required: ["query"],
    properties: {
      query: { type: "string", minLength: 1 },
      limit: { type: "number", minimum: 1, maximum: 100, default: 10 },
    },
  },
  
  // 消费者期望的输出格式
  output: {
    type: "object",
    required: ["results", "total"],
    properties: {
      results: {
        type: "array",
        items: {
          type: "object",
          required: ["id", "title", "url"],
          properties: {
            id: { type: "string" },
            title: { type: "string" },
            url: { type: "string", format: "uri" },
            snippet: { type: "string" }, // 可选，但如果存在则必须是字符串
          },
        },
      },
      total: { type: "number" },
    },
  },
};
```

### 消费者端：验证工具输出符合契约

```typescript
// agent/tools/search-with-contract.ts
import Ajv from "ajv";
import { SearchToolContract } from "../contracts/tools/search.contract";

const ajv = new Ajv();
const validateOutput = ajv.compile(SearchToolContract.output);

export async function searchWithContract(params: unknown) {
  // 1. 验证输入（消费者自己满足提供者期望）
  const validateInput = ajv.compile(SearchToolContract.input);
  if (!validateInput(params)) {
    throw new ContractViolationError(
      "SearchTool input",
      validateInput.errors
    );
  }

  // 2. 调用工具
  const result = await callSearchTool(params);

  // 3. 验证输出（确保提供者满足消费者期望）
  if (!validateOutput(result)) {
    throw new ContractViolationError(
      "SearchTool output",
      validateOutput.errors
    );
  }

  return result;
}

class ContractViolationError extends Error {
  constructor(contract: string, errors: unknown) {
    super(`Contract violation: ${contract}\n${JSON.stringify(errors, null, 2)}`);
    this.name = "ContractViolationError";
  }
}
```

---

## 实战：子 Agent 契约测试

多 Agent 系统中，父子 Agent 之间也需要契约。

### 定义子 Agent 契约

```typescript
// contracts/agents/data-analyst.contract.ts
export const DataAnalystContract = {
  // 父 Agent 发给子 Agent 的任务格式
  task: {
    type: "object",
    required: ["data", "analysisType"],
    properties: {
      data: {
        type: "array",
        items: { type: "object" },
        minItems: 1,
      },
      analysisType: {
        type: "string",
        enum: ["summary", "trend", "anomaly", "forecast"],
      },
      timeRange: {
        type: "object",
        properties: {
          start: { type: "string", format: "date-time" },
          end: { type: "string", format: "date-time" },
        },
      },
    },
  },
  
  // 子 Agent 返回给父 Agent 的结果格式
  result: {
    type: "object",
    required: ["findings", "confidence"],
    properties: {
      findings: {
        type: "array",
        items: {
          type: "object",
          required: ["key", "value", "importance"],
          properties: {
            key: { type: "string" },
            value: {},
            importance: { type: "number", minimum: 0, maximum: 1 },
          },
        },
      },
      confidence: { type: "number", minimum: 0, maximum: 1 },
      recommendations: { type: "array", items: { type: "string" } },
    },
  },
};
```

### 父 Agent：用契约调度子 Agent

```typescript
// orchestrator/spawn-with-contract.ts
import { DataAnalystContract } from "../contracts/agents/data-analyst.contract";

async function spawnDataAnalyst(task: unknown) {
  // 调用前验证任务格式
  const validateTask = ajv.compile(DataAnalystContract.task);
  if (!validateTask(task)) {
    throw new ContractViolationError("DataAnalyst task", validateTask.errors);
  }

  // 调用子 Agent（OpenClaw sessions_spawn）
  const result = await spawnSubAgent({
    agentId: "data-analyst",
    task: JSON.stringify(task),
    timeoutSeconds: 120,
  });

  // 验证返回结果
  const validateResult = ajv.compile(DataAnalystContract.result);
  const parsed = JSON.parse(result.output);
  if (!validateResult(parsed)) {
    throw new ContractViolationError("DataAnalyst result", validateResult.errors);
  }

  return parsed;
}
```

---

## 实战：LLM 结构化输出契约

LLM 输出是最容易"漂移"的地方——prompt 改一点，JSON 格式就变了。

```typescript
// contracts/llm/extraction.contract.ts
export const EntityExtractionContract = {
  output: {
    type: "object",
    required: ["entities"],
    properties: {
      entities: {
        type: "array",
        items: {
          type: "object",
          required: ["text", "type", "confidence"],
          properties: {
            text: { type: "string" },
            type: { 
              type: "string",
              enum: ["person", "org", "location", "date", "product"]
            },
            confidence: { type: "number", minimum: 0, maximum: 1 },
            metadata: { type: "object" }, // 可选扩展字段
          },
        },
      },
      processingTimeMs: { type: "number" },
    },
  },
};

// 在 LLM 调用层注入验证
async function extractEntitiesWithContract(text: string) {
  const response = await llm.complete({
    messages: [{
      role: "user",
      content: `Extract entities from: "${text}"\nRespond with JSON matching this schema: ${
        JSON.stringify(EntityExtractionContract.output, null, 2)
      }`,
    }],
    response_format: { type: "json_object" },
  });

  const parsed = JSON.parse(response.content);
  const validate = ajv.compile(EntityExtractionContract.output);
  
  if (!validate(parsed)) {
    // 不直接报错，而是尝试修复
    return repairLLMOutput(parsed, EntityExtractionContract.output);
  }

  return parsed;
}

// LLM 输出修复：用 LLM 修 LLM
async function repairLLMOutput(broken: unknown, schema: unknown) {
  const repaired = await llm.complete({
    messages: [{
      role: "user",
      content: `Fix this JSON to match the schema:\nJSON: ${JSON.stringify(broken)}\nSchema: ${JSON.stringify(schema)}\nReturn only valid JSON.`,
    }],
  });
  return JSON.parse(repaired.content);
}
```

---

## 契约测试 vs 其他测试

```
┌─────────────────────────────────────────────────────┐
│                    测试金字塔                          │
│                                                     │
│              ┌───────────┐                          │
│              │  E2E 测试  │  ← 少量，覆盖关键路径      │
│              └─────┬─────┘                          │
│          ┌─────────┴─────────┐                      │
│          │    契约测试         │  ← 中量，覆盖接口边界   │
│          └─────────┬─────────┘                      │
│    ┌───────────────┴───────────────┐                │
│    │           单元测试              │  ← 大量，快速执行  │
│    └───────────────────────────────┘                │
└─────────────────────────────────────────────────────┘
```

| 维度 | 单元测试 | 契约测试 | E2E 测试 |
|------|---------|---------|---------|
| 速度 | 毫秒 | 秒 | 分钟 |
| 范围 | 单函数 | 接口边界 | 完整链路 |
| 稳定性 | 高 | 高 | 低（网络/LLM不稳定） |
| 发现问题 | 内部bug | 接口不兼容 | 集成bug |

---

## OpenClaw 中的契约实践

OpenClaw 的 tool schema 本身就是一种契约！

```typescript
// openclaw/src/tools/registry.ts
// 每个 tool 的 JSON Schema 就是它与 LLM 之间的契约
const toolSchema = {
  name: "web_search",
  description: "Search the web using Brave Search API",
  input_schema: {
    type: "object",
    required: ["query"],
    properties: {
      query: { type: "string", description: "Search query" },
      count: { type: "number", minimum: 1, maximum: 10 },
    },
  },
};

// 验证 LLM 生成的 tool_use 调用是否符合契约
function validateToolCall(toolName: string, input: unknown): ValidationResult {
  const schema = toolRegistry.getSchema(toolName);
  if (!schema) {
    return { valid: false, error: `Unknown tool: ${toolName}` };
  }
  
  const validate = ajv.compile(schema.input_schema);
  if (!validate(input)) {
    return { valid: false, errors: validate.errors };
  }
  
  return { valid: true };
}
```

### 契约变更管理

```typescript
// contracts/versions.ts
// 语义化版本管理契约变更
export const ContractVersions = {
  "SearchTool": {
    current: "2.1.0",
    breaking: ["2.0.0", "1.0.0"], // 不向后兼容的版本
    history: {
      "2.1.0": "新增 snippet 可选字段",
      "2.0.0": "BREAKING: results 改为数组（原来是对象）",
      "1.0.0": "初始版本",
    },
  },
};

// 消费者声明自己依赖的版本
const CONSUMER_CONTRACT_VERSION = ">=2.0.0 <3.0.0";

// 启动时检查版本兼容性
function checkContractCompatibility() {
  const violations = [];
  for (const [tool, consumer_range] of Object.entries(CONSUMER_CONTRACT_VERSION)) {
    const provider_version = ContractVersions[tool]?.current;
    if (!semver.satisfies(provider_version, consumer_range)) {
      violations.push(`${tool}: provider=${provider_version}, consumer requires ${consumer_range}`);
    }
  }
  if (violations.length > 0) {
    throw new Error(`Contract incompatibilities:\n${violations.join("\n")}`);
  }
}
```

---

## CI/CD 中的契约测试

```yaml
# .github/workflows/contract-tests.yml
name: Contract Tests

on: [push, pull_request]

jobs:
  contract-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Run Contract Tests
        run: |
          # 1. 测试所有 Tool 的输入/输出契约
          npx vitest run tests/contracts/tools/
          
          # 2. 测试 Agent 间消息格式契约
          npx vitest run tests/contracts/agents/
          
          # 3. 快照测试 LLM 输出结构（使用 mock LLM）
          npx vitest run tests/contracts/llm/
      
      - name: Publish Contract Report
        if: failure()
        uses: actions/upload-artifact@v3
        with:
          name: contract-violations
          path: contract-report.json
```

---

## 关键原则

1. **契约由消费者定义**：消费者写"我需要什么"，提供者承诺满足
2. **只测边界，不测内部**：只验证接口格式，不关心实现细节
3. **版本化管理**：破坏性变更必须升主版本号
4. **失败快速**：启动时就检查契约兼容性，而不是运行时发现
5. **修复而不是崩溃**：LLM 输出违约时优先尝试自动修复

---

## 总结

```
契约测试的价值：
多 Agent 系统可以独立迭代 → 快速交付
接口变更立即被发现 → 不会悄悄崩溃
清晰的接口文档 → 新成员快速上手
```

> 没有契约的 Agent 系统就像没有 API 文档的微服务——  
> 最终会因为"我以为你返回的是这个格式"而崩溃。

---

*第 74 课 - Agent 开发实战系列*
