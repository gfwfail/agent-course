# 176 - Agent 工具版本控制与向后兼容（Tool Versioning & Backward Compatibility）

> 工具 API 和代码一样会演进。参数改名、字段删除、行为调整——如果没有版本策略，生产 Agent 分分钟崩掉。

---

## 为什么工具需要版本控制？

Agent 依赖工具的 **Schema**（参数名、类型、语义）来生成调用。一旦 Schema 破坏性变更：

- LLM 按旧 Schema 生成的调用直接报错
- 已缓存的提示词（Prompt Cache）失效
- Fine-tuned 模型更惨：训练数据和生产不一致

典型破坏性变更（Breaking Changes）：

| 变更类型 | 例子 |
|--------|------|
| 参数重命名 | `user_id` → `userId` |
| 参数类型变更 | `limit: string` → `limit: number` |
| 必填参数新增 | 新增必填字段 `region` |
| 返回结构变更 | 原来返回数组，现在包在 `{data:[]}` 里 |
| 工具名称变更 | `search` → `web_search` |

---

## 语义版本（SemVer）用于工具

```
tool@MAJOR.MINOR.PATCH

MAJOR: 破坏性变更（参数重命名、类型变更、移除）
MINOR: 向后兼容新增（新增可选参数）
PATCH: Bug 修复、文档更新
```

工具注册时携带版本元数据：

```typescript
// learn-claude-code / tool registry 风格
interface ToolDefinition {
  name: string;          // "search_web"
  version: string;       // "2.1.0"
  deprecated?: boolean;
  deprecatedSince?: string;
  replacedBy?: string;   // 替代工具名
  description: string;
  inputSchema: JSONSchema;
}
```

---

## 三种版本策略

### 策略 1：并行版本（Parallel Versioning）

同时运行新旧两个版本，旧版本标记 deprecated：

```typescript
// pi-mono 风格：工具注册表支持多版本
class ToolRegistry {
  private tools = new Map<string, Map<string, ToolDefinition>>();

  register(tool: ToolDefinition) {
    const key = tool.name;
    if (!this.tools.has(key)) {
      this.tools.set(key, new Map());
    }
    this.tools.get(key)!.set(tool.version, tool);
  }

  resolve(name: string, preferVersion?: string): ToolDefinition {
    const versions = this.tools.get(name);
    if (!versions) throw new Error(`Tool not found: ${name}`);

    if (preferVersion && versions.has(preferVersion)) {
      return versions.get(preferVersion)!;
    }

    // 默认返回最新非 deprecated 版本
    const active = [...versions.values()]
      .filter(t => !t.deprecated)
      .sort((a, b) => semverCompare(b.version, a.version));

    if (active.length === 0) throw new Error(`All versions deprecated: ${name}`);
    return active[0];
  }

  // 注入 LLM 的工具列表：自动加 deprecated 提示
  toAnthropicTools(agentVersion?: string): AnthropicTool[] {
    const resolved = new Map<string, ToolDefinition>();

    for (const [name] of this.tools) {
      resolved.set(name, this.resolve(name, agentVersion));
    }

    return [...resolved.values()].map(t => ({
      name: t.name,
      description: t.deprecated
        ? `[DEPRECATED since ${t.deprecatedSince}, use ${t.replacedBy}] ${t.description}`
        : t.description,
      input_schema: t.inputSchema,
    }));
  }
}
```

### 策略 2：URL/命名空间版本（Namespace Versioning）

工具名带版本号，新旧并存：

```typescript
// 旧版本
{ name: "search_v1", description: "搜索（旧版，即将废弃）" }

// 新版本
{ name: "search_v2", description: "搜索（当前版本，支持分页）" }
```

优点：LLM 可以明确选择版本
缺点：工具列表膨胀，消耗 Token

适用场景：过渡期较长、新旧接口差异大

### 策略 3：适配器模式（Adapter Pattern）

对外只暴露新接口，内部兼容旧调用：

```typescript
// 新版工具：search_web v2（支持 pagination）
async function searchWebV2(params: {
  query: string;
  limit?: number;
  cursor?: string;  // 新增
  region?: string;  // 新增
}): Promise<SearchResult> {
  return searchImpl(params);
}

// 旧版适配器：接受旧 schema，内部转换
async function searchWebV1Adapter(params: {
  query: string;
  count?: number;  // 旧参数名
}): Promise<OldSearchResult> {
  // 参数映射
  const result = await searchWebV2({
    query: params.query,
    limit: params.count,  // 字段名映射
  });

  // 返回值映射
  return {
    results: result.items,  // 结构映射
    total: result.total,
  };
}
```

---

## 实战：deprecation 自动提示

让 LLM 主动迁移到新版：

```typescript
// OpenClaw 风格：在工具描述里注入迁移提示
function buildDeprecationNotice(tool: ToolDefinition): string {
  if (!tool.deprecated) return tool.description;

  return `
⚠️ DEPRECATED (since v${tool.deprecatedSince}): This tool will be removed.
Migrate to: ${tool.replacedBy}
Migration guide: ${tool.migrationNote ?? 'See docs'}

${tool.description}
`.trim();
}

// 注册旧版（自动带 deprecation notice）
registry.register({
  name: "get_user",
  version: "1.0.0",
  deprecated: true,
  deprecatedSince: "2.0.0",
  replacedBy: "get_user_profile",
  migrationNote: "参数 user_id 改为 userId，返回值新增 avatar 字段",
  description: "获取用户信息",
  inputSchema: { ... }
});
```

当 LLM 看到带 `⚠️ DEPRECATED` 的工具描述，会倾向于改用新工具——这是"提示词即文档"的典型应用。

---

## 版本协商（Version Negotiation）

Agent 启动时声明自己兼容的工具版本范围：

```typescript
// Agent 配置
interface AgentConfig {
  toolVersionConstraints?: Record<string, string>;
  // 例如：{ "search_web": ">=1.5.0 <3.0.0" }
}

// 工具分发层按约束过滤
class ToolDispatcher {
  constructor(
    private registry: ToolRegistry,
    private agentConfig: AgentConfig,
  ) {}

  getCompatibleTools(): AnthropicTool[] {
    const constraints = this.agentConfig.toolVersionConstraints ?? {};

    return this.registry.getAllTools()
      .filter(tool => {
        const constraint = constraints[tool.name];
        if (!constraint) return !tool.deprecated; // 默认用最新非废弃版本
        return semverSatisfies(tool.version, constraint);
      })
      .map(tool => this.registry.toAnthropicTool(tool));
  }
}
```

---

## 破坏性变更的安全迁移流程

```
Phase 1: 双写（Dark Launch）
  - 部署新版工具，但不注入 LLM
  - 旧版调用时后台同时调新版，对比结果
  - 观察 1-3 天，确认新版行为一致

Phase 2: 灰度（Canary）
  - 10% Agent 会话注入新版工具描述
  - 监控调用成功率、错误率、用户满意度

Phase 3: 全量 + Deprecated 标记
  - 所有 Agent 切换到新版
  - 旧版打上 deprecated，描述里加迁移提示
  - 保留旧版 3 个月（给外部集成者时间）

Phase 4: 下线
  - 旧版调用返回 409 Deprecated（带迁移说明）
  - 从工具列表中完全移除
```

---

## OpenClaw 实战：工具版本中间件

```typescript
// OpenClaw 工具中间件：自动检测版本调用，记录迁移指标
class ToolVersionMiddleware implements ToolMiddleware {
  async execute(
    toolName: string,
    params: unknown,
    next: ToolHandler,
  ): Promise<ToolResult> {
    const tool = registry.resolve(toolName);

    // 记录废弃工具调用
    if (tool.deprecated) {
      metrics.increment('tool.deprecated.calls', { tool: toolName });
      console.warn(
        `[ToolVersion] Deprecated tool called: ${toolName}@${tool.version}` +
        ` → migrate to ${tool.replacedBy}`
      );
    }

    // 参数迁移（自动修复常见旧参数名）
    const migrated = this.migrateParams(toolName, params, tool.version);

    try {
      return await next(toolName, migrated);
    } catch (err) {
      // 调用失败时提示版本问题
      if (err instanceof SchemaValidationError && tool.deprecated) {
        return {
          error: `Tool ${toolName} is deprecated and may have schema changes. ` +
                 `Please use ${tool.replacedBy} instead.`,
          migrationNote: tool.migrationNote,
        };
      }
      throw err;
    }
  }

  private migrateParams(
    toolName: string,
    params: unknown,
    version: string,
  ): unknown {
    // 自动修复常见字段名变更（驼峰/下划线互转等）
    const migrations = PARAM_MIGRATIONS[toolName]?.[version] ?? [];
    let result = params as Record<string, unknown>;

    for (const { from, to } of migrations) {
      if (from in result && !(to in result)) {
        result = { ...result, [to]: result[from] };
        delete result[from];
      }
    }

    return result;
  }
}

// 参数迁移表
const PARAM_MIGRATIONS: Record<string, Record<string, Array<{from: string; to: string}>>> = {
  search_web: {
    "1.x": [
      { from: "count", to: "limit" },
      { from: "q", to: "query" },
    ],
  },
};
```

---

## pi-mono 的工具版本实践

pi-mono 用 **tool name namespace** 做版本隔离：

```typescript
// pi-mono toolRegistry 风格
export const tools = {
  // 当前版本（无版本后缀）
  search: defineSearchTool({ version: "2.0.0" }),

  // 兼容旧版（命名保留）
  search_legacy: defineSearchLegacyTool({
    version: "1.5.0",
    deprecated: true,
    replacedBy: "search",
  }),
};
```

配合 system prompt 的动态组装（课 82 Dynamic System Prompt），
可以根据 Agent 版本动态注入不同的工具集：

```typescript
// 老 Agent（通过 User-Agent 或配置识别）→ 注入兼容工具列表
// 新 Agent → 注入最新工具列表
const agentVersion = ctx.session.metadata?.agentVersion ?? "latest";
const tools = registry.getToolsForAgent(agentVersion);
```

---

## 关键指标监控

| 指标 | 含义 | 告警阈值 |
|-----|------|--------|
| `tool.deprecated.calls` | 废弃工具调用次数 | >100/天 开始催迁移 |
| `tool.version.mismatch` | Schema 不匹配错误 | 任何值都要排查 |
| `tool.migration.success` | 自动参数迁移成功率 | <95% 说明迁移规则有问题 |
| `tool.sunset.countdown` | 距下线天数 | <30 天触发高优告警 |

---

## 总结

| 模式 | 适用场景 |
|-----|---------|
| Parallel Versioning | 过渡期，新旧工具同时存在 |
| Namespace Versioning | 新旧接口差异大，长期并存 |
| Adapter Pattern | 内部演进，对外接口稳定 |
| Version Negotiation | 多 Agent 版本同时运行的平台 |

**核心原则：**
1. 工具 Schema 是 Agent 的 API 合约，改它要像改 HTTP API 一样慎重
2. deprecated ≠ 立即删除，给 LLM 和人类集成者足够的迁移时间
3. 自动参数迁移适配器可以买时间，但不是长久之计
4. 监控废弃工具调用量——降为零才能安全下线

> OpenClaw 的 Skills 系统是天然的工具版本载体：每个 SKILL.md 可以声明版本，
> Gateway 重启时加载最新版，旧版 skill 文件可以保留在 `legacy/` 目录作为参考。
