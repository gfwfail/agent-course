# 162. Agent 工具发现与能力协商（Tool Discovery & Capability Negotiation）

> **核心思想**：Agent 不应该硬编码工具列表——工具应该自我声明能力，Agent 在运行时动态发现并协商使用条件。MCP 就是这一思想的标准化落地。

---

## 为什么需要工具发现？

传统做法：把所有工具定义写死在 system prompt 里。

问题：
- 工具增多 → Token 暴涨
- 工具更新 → 需要重部署 Agent
- 不同环境能力不同（dev 没有生产 DB 权限）→ 工具列表需要手动维护
- Agent 不知道某个工具"当前是否可用"

**工具发现**解决的是：Agent 在运行时询问"我现在有哪些工具？它们能做什么？"

---

## 核心概念

### 能力声明（Capability Advertisement）

工具不仅声明 schema，还声明：
- `version`：工具版本，用于兼容性检查
- `availability`：当前是否可用（取决于配置/权限/服务状态）
- `constraints`：限制条件（如只读、仅特定环境、速率限制）
- `alternatives`：降级替代方案

### 能力协商（Capability Negotiation）

Agent 与工具注册中心之间的对话：
1. Agent 声明自己的意图（"我需要写数据库"）
2. 注册中心返回符合条件的工具列表（带版本、约束）
3. Agent 选择最合适的工具（考虑成本、速度、权限）
4. 工具执行前再做一次 capability check（防止运行时失效）

---

## TypeScript 实现

```typescript
// tool-registry.ts - 工具能力注册中心

interface ToolCapability {
  name: string;
  version: string;
  description: string;
  inputSchema: Record<string, unknown>;
  
  // 能力声明
  tags: string[];           // ["database", "write", "sql"]
  availability: () => Promise<AvailabilityResult>;
  constraints?: ToolConstraints;
  alternatives?: string[];  // 降级备选工具名
}

interface AvailabilityResult {
  available: boolean;
  reason?: string;          // 不可用原因
  degraded?: boolean;       // 降级模式（可用但受限）
  expiresAt?: number;       // 可用性有效期（ms timestamp）
}

interface ToolConstraints {
  rateLimit?: { maxPerMinute: number };
  environments?: string[];  // ["production", "staging"]
  requiredPermissions?: string[];
  readOnly?: boolean;
}

class ToolRegistry {
  private tools = new Map<string, ToolCapability>();
  private availabilityCache = new Map<string, { result: AvailabilityResult; cachedAt: number }>();
  private readonly CACHE_TTL_MS = 30_000; // 30 秒缓存

  register(tool: ToolCapability) {
    this.tools.set(tool.name, tool);
    console.log(`[Registry] Registered: ${tool.name} v${tool.version}`);
  }

  // 按意图发现工具（语义匹配 tags）
  async discover(intent: DiscoveryIntent): Promise<DiscoveryResult[]> {
    const results: DiscoveryResult[] = [];
    
    for (const [name, tool] of this.tools) {
      // Tag 匹配
      const matchScore = this.scoreMatch(tool, intent);
      if (matchScore === 0) continue;

      // 检查可用性（带缓存）
      const availability = await this.checkAvailability(name, tool);
      
      // 检查约束
      const constraintResult = this.checkConstraints(tool, intent.context);
      
      results.push({
        tool: {
          name,
          version: tool.version,
          description: tool.description,
          inputSchema: tool.inputSchema,
          constraints: tool.constraints,
        },
        availability,
        constraintResult,
        matchScore,
        alternatives: tool.alternatives ?? [],
      });
    }

    // 按匹配分 + 可用性排序
    return results
      .sort((a, b) => {
        if (a.availability.available !== b.availability.available)
          return a.availability.available ? -1 : 1;
        return b.matchScore - a.matchScore;
      });
  }

  private scoreMatch(tool: ToolCapability, intent: DiscoveryIntent): number {
    let score = 0;
    for (const tag of intent.requiredTags ?? []) {
      if (tool.tags.includes(tag)) score += 2;
      else return 0; // required tag 缺失 → 不匹配
    }
    for (const tag of intent.preferredTags ?? []) {
      if (tool.tags.includes(tag)) score += 1;
    }
    return score;
  }

  private async checkAvailability(name: string, tool: ToolCapability): Promise<AvailabilityResult> {
    const cached = this.availabilityCache.get(name);
    const now = Date.now();
    
    if (cached && now - cached.cachedAt < this.CACHE_TTL_MS) {
      return cached.result;
    }

    const result = await tool.availability();
    this.availabilityCache.set(name, { result, cachedAt: now });
    return result;
  }

  private checkConstraints(tool: ToolCapability, context: RequestContext): ConstraintResult {
    const c = tool.constraints;
    if (!c) return { ok: true };

    if (c.environments && !c.environments.includes(context.environment)) {
      return { ok: false, reason: `Tool only available in: ${c.environments.join(', ')}` };
    }
    if (c.requiredPermissions) {
      const missing = c.requiredPermissions.filter(p => !context.permissions.includes(p));
      if (missing.length > 0) {
        return { ok: false, reason: `Missing permissions: ${missing.join(', ')}` };
      }
    }
    return { ok: true };
  }
  
  // 生成注入 LLM 的工具列表（只含可用工具）
  async getAvailableToolsForLLM(context: RequestContext): Promise<LLMToolDefinition[]> {
    const results = await this.discover({
      requiredTags: [],
      context,
    });
    
    return results
      .filter(r => r.availability.available && r.constraintResult.ok)
      .map(r => ({
        name: r.tool.name,
        description: r.availability.degraded
          ? `${r.tool.description} [DEGRADED MODE: limited functionality]`
          : r.tool.description,
        input_schema: r.tool.inputSchema,
      }));
  }
}

interface DiscoveryIntent {
  requiredTags?: string[];
  preferredTags?: string[];
  context: RequestContext;
}

interface RequestContext {
  environment: string;
  permissions: string[];
  userId?: string;
}

interface DiscoveryResult {
  tool: Omit<ToolCapability, 'availability'>;
  availability: AvailabilityResult;
  constraintResult: ConstraintResult;
  matchScore: number;
  alternatives: string[];
}

interface ConstraintResult {
  ok: boolean;
  reason?: string;
}

interface LLMToolDefinition {
  name: string;
  description: string;
  input_schema: Record<string, unknown>;
}
```

---

## 工具注册示例

```typescript
// tools/database-write.ts

const registry = new ToolRegistry();

registry.register({
  name: "database_write",
  version: "2.1.0",
  description: "Write records to the production database",
  tags: ["database", "write", "sql", "persistent"],
  inputSchema: {
    type: "object",
    properties: {
      table: { type: "string" },
      data: { type: "object" },
    },
    required: ["table", "data"],
  },
  constraints: {
    environments: ["production", "staging"],
    requiredPermissions: ["db:write"],
    rateLimit: { maxPerMinute: 100 },
  },
  // 运行时检查可用性
  availability: async () => {
    try {
      await db.ping();
      const lag = await db.getReplicationLag();
      if (lag > 5000) {
        return { available: true, degraded: true, reason: "High replication lag" };
      }
      return { available: true };
    } catch (e) {
      return { available: false, reason: `DB unreachable: ${e.message}` };
    }
  },
  alternatives: ["database_write_queue"], // 降级到队列写入
});

registry.register({
  name: "database_write_queue",
  version: "1.0.0",
  description: "Queue a write operation for eventual consistency (fallback mode)",
  tags: ["database", "write", "queue", "eventual-consistency"],
  inputSchema: { /* ... */ },
  availability: async () => ({ available: true }), // 队列永远可用
});
```

---

## 在 Agent Loop 中集成

```typescript
// agent-loop.ts

class Agent {
  constructor(
    private registry: ToolRegistry,
    private llm: LLMClient,
  ) {}

  async run(userMessage: string, context: RequestContext) {
    // 1. 动态发现可用工具（而不是硬编码）
    const availableTools = await this.registry.getAvailableToolsForLLM(context);
    
    console.log(`[Agent] Discovered ${availableTools.length} available tools`);

    // 2. 把动态工具列表注入 LLM
    const response = await this.llm.complete({
      messages: [{ role: "user", content: userMessage }],
      tools: availableTools, // 动态！
    });

    // 3. 工具调用时再次验证可用性（防止发现到执行的时间窗口内失效）
    if (response.tool_calls) {
      for (const call of response.tool_calls) {
        const discoveryResult = await this.registry.discover({
          requiredTags: [],
          context,
        }).then(results => results.find(r => r.tool.name === call.name));

        if (!discoveryResult?.availability.available) {
          // 自动寻找替代品
          const alternatives = discoveryResult?.alternatives ?? [];
          if (alternatives.length > 0) {
            console.warn(`[Agent] ${call.name} unavailable, trying alternatives: ${alternatives}`);
            call.name = alternatives[0]; // 透明降级
          } else {
            return { error: `Tool ${call.name} is currently unavailable` };
          }
        }

        // 执行工具...
      }
    }
  }
}
```

---

## OpenClaw Skills 即工具发现的落地实现

OpenClaw 的 Skills 系统就是一个本地工具发现机制：

```
skills/
  weather/SKILL.md        ← 工具自描述（description = capability advertisement）
  github/SKILL.md         ← tags: ["git", "pr", "ci"]
  mysterybox/SKILL.md     ← constraints: { environments: ["production"] }
```

每次 Agent 收到请求时：
1. 扫描 `available_skills` 列表（discovery）
2. 根据用户意图匹配最合适的 SKILL（capability negotiation）
3. 读取 SKILL.md 获取完整工具规格（hydration）
4. 按 SKILL.md 指示执行（execution）

这就是完整的 **发现 → 协商 → 加载 → 执行** 生命周期。

---

## MCP 协议：标准化工具发现

[MCP (Model Context Protocol)](https://modelcontextprotocol.io) 把工具发现标准化为协议：

```json
// MCP 服务端 → 客户端能力协商
{
  "method": "tools/list",
  "result": {
    "tools": [
      {
        "name": "read_file",
        "description": "Read file contents",
        "inputSchema": { ... },
        // MCP 扩展：能力声明
        "annotations": {
          "readOnly": true,
          "requiredEnvironments": ["any"]
        }
      }
    ]
  }
}
```

本地 Skills（文件系统发现）→ MCP（网络协议发现）→ 分布式工具市场（未来）

---

## 最佳实践

| 场景 | 策略 |
|------|------|
| 工具不可用 | 检查 `alternatives`，透明降级 |
| 可用性检查开销大 | 缓存 30s，异步刷新 |
| LLM Token 预算紧张 | 按 intent 过滤，只注入相关工具 |
| 多环境部署 | 约束条件写在工具定义里，不要 if/else |
| 版本升级 | 用 `version` 字段 + semver，同时注册新旧版 |

---

## 关键原则

> **工具是服务，不是代码。** 就像微服务有健康检查端点，工具也应该声明自己的可用性和能力边界。Agent 不应该假设工具永远存在——应该在运行时发现、协商、优雅降级。

这正是 MCP 想要解决的问题，也是 OpenClaw Skills 的设计哲学。

---

*下一讲预告：Agent 响应式事件流与 RxJS 模式（Reactive Event Streams）*
