# 137 - Agent 工具版本管理与向后兼容（Tool Versioning & Backward Compatibility）

> "工具也是 API，API 就要版本管理。"

---

## 问题背景

生产环境中，Agent 工具会随着业务发展不断演化：

- `search_user` 要新增 `include_deleted` 参数
- `send_email` 的参数从单个 `to` 改为支持数组 `recipients`
- `get_order` 返回结构要重构，但老 LLM 调用方还在用旧格式

**不做版本管理的后果：**

```typescript
// 今天的工具
{ name: "send_email", parameters: { to: "string", body: "string" } }

// 半年后的工具
{ name: "send_email", parameters: { recipients: "string[]", body: "string", cc: "string[]" } }

// 旧的 Agent prompt 里有 "to" 字段的调用记录
// → LLM 根据历史经验仍然传 to="user@example.com"
// → 工具收不到 recipients → 静默失败 💀
```

**核心矛盾：** LLM 的 few-shot 记忆 vs 工具 Schema 的演进速度

---

## 解决方案：三层版本管理策略

```
┌─────────────────────────────────────┐
│  Layer 1: Schema 版本命名空间        │  send_email_v1 / send_email_v2
├─────────────────────────────────────┤
│  Layer 2: 参数向后兼容层             │  旧参数自动映射到新参数
├─────────────────────────────────────┤
│  Layer 3: 灰度迁移 + 废弃告警        │  渐进切换，主动提示升级
└─────────────────────────────────────┘
```

---

## 实战代码

### 1. 工具版本注册表

```typescript
// tool-registry-versioned.ts
interface ToolVersion<T = Record<string, unknown>> {
  version: number;
  schema: ToolSchema;
  handler: (params: T) => Promise<unknown>;
  deprecated?: boolean;
  deprecatedSince?: string;
  migratesTo?: number;      // 推荐迁移到哪个版本
  migrator?: (oldParams: unknown) => T;  // 旧参数 → 新参数自动转换
}

class VersionedToolRegistry {
  // toolName → Map<version, ToolVersion>
  private tools = new Map<string, Map<number, ToolVersion>>();

  register(toolName: string, version: ToolVersion) {
    if (!this.tools.has(toolName)) {
      this.tools.set(toolName, new Map());
    }
    this.tools.get(toolName)!.set(version.version, version);
  }

  // 给 LLM 暴露工具列表，默认只暴露最新版本
  // 可选：同时暴露 v1（带 deprecated 描述）
  getToolsForLLM(opts = { includeDeprecated: false }): ToolSchema[] {
    const schemas: ToolSchema[] = [];
    
    for (const [toolName, versions] of this.tools) {
      const sorted = [...versions.entries()].sort(([a], [b]) => b - a);
      const [latestVersion, latest] = sorted[0];
      
      // 最新版本始终暴露
      schemas.push({
        ...latest.schema,
        name: `${toolName}_v${latestVersion}`,
      });

      // 废弃版本可选暴露（兼容旧 prompt）
      if (opts.includeDeprecated) {
        for (const [ver, tool] of sorted.slice(1)) {
          if (tool.deprecated) {
            schemas.push({
              ...tool.schema,
              name: `${toolName}_v${ver}`,
              description: `[DEPRECATED → use ${toolName}_v${latestVersion}] ${tool.schema.description}`,
            });
          }
        }
      }
    }
    return schemas;
  }

  // 执行工具调用（含自动迁移）
  async execute(toolName: string, version: number, params: unknown): Promise<unknown> {
    // 解析版本号（兼容无版本后缀的老调用）
    const resolved = this.resolveVersion(toolName, version);
    const tool = this.tools.get(toolName)?.get(resolved);
    if (!tool) throw new Error(`Tool ${toolName} v${resolved} not found`);

    let finalParams = params;

    // 如果调用的是旧版本，自动用 migrator 转换参数
    if (resolved < this.getLatestVersion(toolName)) {
      const latest = this.getLatest(toolName)!;
      if (latest.migrator) {
        finalParams = latest.migrator(params);
        console.warn(
          `[ToolRegistry] Auto-migrated ${toolName} v${resolved} → v${this.getLatestVersion(toolName)}`
        );
      }
    }

    // 废弃告警
    if (tool.deprecated) {
      console.warn(
        `[ToolRegistry] DEPRECATED: ${toolName} v${resolved} ` +
        `(since ${tool.deprecatedSince}). Migrate to v${tool.migratesTo}.`
      );
    }

    return tool.handler(finalParams as never);
  }

  private getLatestVersion(toolName: string): number {
    const versions = this.tools.get(toolName);
    if (!versions) throw new Error(`Unknown tool: ${toolName}`);
    return Math.max(...versions.keys());
  }

  private getLatest(toolName: string): ToolVersion | undefined {
    const v = this.getLatestVersion(toolName);
    return this.tools.get(toolName)?.get(v);
  }

  private resolveVersion(toolName: string, requestedVersion: number): number {
    // 若请求版本不存在，返回最近的更高版本
    const versions = [...(this.tools.get(toolName)?.keys() ?? [])].sort();
    return versions.find(v => v >= requestedVersion) ?? versions[versions.length - 1];
  }
}
```

---

### 2. 注册工具版本（send_email 演进示例）

```typescript
const registry = new VersionedToolRegistry();

// ── V1：旧版本（已废弃）──────────────────────────────
registry.register("send_email", {
  version: 1,
  deprecated: true,
  deprecatedSince: "2025-01",
  migratesTo: 2,
  schema: {
    name: "send_email",
    description: "发送邮件（旧版，仅支持单收件人）",
    parameters: {
      type: "object",
      properties: {
        to:      { type: "string", description: "收件人邮箱" },
        subject: { type: "string", description: "主题" },
        body:    { type: "string", description: "正文" },
      },
      required: ["to", "subject", "body"],
    },
  },
  handler: async (params: { to: string; subject: string; body: string }) => {
    // 旧逻辑保留但实际会被 migrator 接管
    return sendMail([params.to], params.subject, params.body);
  },
});

// ── V2：新版本（当前）────────────────────────────────
registry.register("send_email", {
  version: 2,
  schema: {
    name: "send_email",
    description: "发送邮件，支持多收件人和抄送",
    parameters: {
      type: "object",
      properties: {
        recipients: { type: "array", items: { type: "string" }, description: "收件人列表" },
        cc:         { type: "array", items: { type: "string" }, description: "抄送列表（可选）" },
        subject:    { type: "string", description: "主题" },
        body:       { type: "string", description: "正文" },
        template_id:{ type: "string", description: "邮件模板 ID（可选）" },
      },
      required: ["recipients", "subject", "body"],
    },
  },
  // 🔑 migrator：v1 参数 → v2 参数自动转换
  migrator: (old: unknown) => {
    const o = old as { to: string; subject: string; body: string };
    return {
      recipients: [o.to],   // string → string[]
      subject: o.subject,
      body: o.body,
      cc: [],
    };
  },
  handler: async (params: {
    recipients: string[];
    cc?: string[];
    subject: string;
    body: string;
    template_id?: string;
  }) => {
    return sendMail(params.recipients, params.subject, params.body, params.cc);
  },
});
```

---

### 3. Agent Loop 集成

```typescript
// 解析 LLM 返回的工具调用（工具名可能带或不带版本后缀）
function parseToolCall(rawName: string): { toolName: string; version: number } {
  const match = rawName.match(/^(.+?)(?:_v(\d+))?$/);
  return {
    toolName: match![1],
    version: match![2] ? parseInt(match![2]) : 999, // 无版本号 → 用最新
  };
}

async function runAgentLoop(messages: Message[]) {
  // 给 LLM 只暴露最新版本工具
  const tools = registry.getToolsForLLM({ includeDeprecated: false });

  const response = await anthropic.messages.create({
    model: "claude-opus-4-5",
    tools,
    messages,
  });

  for (const block of response.content) {
    if (block.type !== "tool_use") continue;

    const { toolName, version } = parseToolCall(block.name);
    
    try {
      const result = await registry.execute(toolName, version, block.input);
      // 将结果追加到 messages...
    } catch (err) {
      // 版本不兼容 → 返回错误提示，LLM 可重试
      appendToolError(messages, block.id, `Tool version error: ${err.message}`);
    }
  }
}
```

---

### 4. OpenClaw / pi-mono 实战模式

OpenClaw 的 Skills 系统天然支持版本化——通过文件路径隔离：

```
skills/
  send-email/
    v1/SKILL.md      ← 旧版（保留兼容）
    v2/SKILL.md      ← 当前版本
    SKILL.md         ← 软链接 → v2/SKILL.md
```

pi-mono 中用 `ToolRegistry` 实现同样模式：

```typescript
// pi-mono: src/tools/registry.ts
export class ToolRegistry {
  private versions: Map<string, Map<number, RegisteredTool>> = new Map();

  // 工具名解析：支持 "search_user"、"search_user_v2"、"search_user@2"
  resolve(nameWithVersion: string): RegisteredTool {
    const [name, ver] = nameWithVersion.split(/[_@]v?(\d+)$/).filter(Boolean);
    const version = ver ? parseInt(ver) : this.getLatestVersion(name);
    return this.getOrThrow(name, version);
  }
}
```

---

## 参数兼容性原则（The 3 Rules）

| 操作 | 兼容性 | 处理方式 |
|------|--------|---------|
| **新增可选参数** | ✅ 向后兼容 | 直接加，默认值处理 |
| **删除/重命名参数** | ❌ 破坏性变更 | 必须 +1 版本号，写 migrator |
| **修改参数类型** | ❌ 破坏性变更 | 必须 +1 版本号，写 migrator |

```typescript
// ✅ 向后兼容的加法：直接在 v2 schema 加可选参数即可
// 不需要新版本号
{
  name: "search_user",
  parameters: {
    query: { type: "string" },
    limit: { type: "number", default: 10 },      // 新加，可选
    include_deleted: { type: "boolean" },         // 新加，可选
  }
}

// ❌ 破坏性变更：to → recipients（类型变了）
// 必须 bump 到 v2 + 写 migrator
```

---

## 废弃生命周期（Deprecation Lifecycle）

```
v1 发布
  │
  ├── v2 发布（推荐迁移）
  │     └── v1 标记 deprecated，两个版本共存
  │
  ├── 4 周观察期（监控 v1 调用量）
  │     └── 打印告警日志，统计迁移进度
  │
  ├── v1 从 LLM 工具列表移除（LLM 不再主动调用）
  │     └── 但执行层仍支持（老 prompt 里的硬编码调用不报错）
  │
  └── 3 个月后 v1 完全下线（migrator 保留作历史记录）
```

---

## 监控指标

```typescript
// 埋点：工具版本调用分布
metrics.histogram("tool_call_version", 1, {
  tool: toolName,
  version: String(version),
  is_deprecated: String(tool.deprecated ?? false),
  was_migrated: String(wasMigrated),
});

// 告警：废弃工具调用量占比 > 10%
if (deprecatedCallRatio > 0.1) {
  alert.warn(`${toolName} v${version} still receiving ${deprecatedCallRatio * 100}% calls`);
}
```

---

## 关键总结

1. **工具 Schema = API 契约**，破坏性变更必须 bump 版本号
2. **migrator 是桥梁**：让旧调用自动转换成新格式，LLM 零感知
3. **废弃 ≠ 删除**：先从工具列表移除，执行层继续兜底
4. **监控驱动迁移**：看调用量数据，而不是凭感觉下线旧版本
5. **向加法兼容**：能加可选参数就别改类型，优先保持向后兼容

> OpenClaw 和 pi-mono 的工具演进历史就是最好的教材——每次 breaking change 都有对应的 migrator，老 session 无感知升级。

---

*Lesson 137 / Agent 开发课程*
