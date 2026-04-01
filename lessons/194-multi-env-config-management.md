# 194 - Agent 多环境配置管理（Multi-Environment Configuration Management）

> 同一个 Agent，开发环境用假数据、Staging 用沙箱 API、生产用真钱真接口。  
> 如果配置管理没做好，轻则测试污染生产，重则把 $10,000 的订单发给了测试用户。

---

## 一、为什么 Agent 的多环境配置比普通服务更难？

普通 Web 服务换个环境，通常只是换数据库地址。  
Agent 换环境，要换的东西多得多：

| 维度 | Dev | Staging | Prod |
|------|-----|---------|------|
| LLM 模型 | `haiku`（便宜快） | `sonnet`（接近生产） | `sonnet`/`opus` |
| 工具权限 | 全开（方便调试） | 受限（模拟生产） | 最小权限 |
| 外部 API | Mock / Sandbox | Sandbox | 真实接口 |
| 费用限制 | 无 | 低（防意外超支） | 高 |
| 日志级别 | DEBUG（带工具详情） | INFO | WARN + Audit |
| 自动重试 | 1次（快速失败） | 3次 | 5次 + 指数退避 |
| 人工审批 | 跳过（开发效率） | 模拟（测试流程） | 严格执行 |

---

## 二、配置分层设计

核心思路：**base config + env overlay + runtime override**，三层叠加，后者覆盖前者。

```typescript
// config/schema.ts
import { z } from "zod";

export const AgentConfigSchema = z.object({
  env: z.enum(["development", "staging", "production"]),
  
  llm: z.object({
    model: z.string(),
    maxTokens: z.number().int().positive(),
    temperature: z.number().min(0).max(2),
    budgetCents: z.number().nonnegative(), // 0 = unlimited
  }),
  
  tools: z.object({
    allowList: z.array(z.string()).optional(), // undefined = all tools
    requireApprovalRisk: z.enum(["HIGH", "CRITICAL", "NEVER"]),
    timeoutMs: z.number().int().positive(),
    maxRetries: z.number().int().min(0).max(10),
  }),
  
  external: z.object({
    useMocks: z.boolean(),
    sandboxMode: z.boolean(),
    webhookBaseUrl: z.string().url(),
  }),
  
  logging: z.object({
    level: z.enum(["debug", "info", "warn", "error"]),
    includeToolParams: z.boolean(),  // prod 要关掉，防 PII 泄露
    includeToolResults: z.boolean(),
  }),
  
  rateLimit: z.object({
    requestsPerMinute: z.number().int().positive(),
    burstMultiplier: z.number().min(1),
  }),
});

export type AgentConfig = z.infer<typeof AgentConfigSchema>;
```

```typescript
// config/base.ts - 所有环境共享的默认值
export const baseConfig = {
  llm: {
    maxTokens: 4096,
    temperature: 0.7,
    budgetCents: 0,
  },
  tools: {
    requireApprovalRisk: "HIGH" as const,
    timeoutMs: 30_000,
    maxRetries: 3,
  },
  external: {
    useMocks: false,
    sandboxMode: false,
    webhookBaseUrl: "https://agent.example.com",
  },
  logging: {
    level: "info" as const,
    includeToolParams: false,
    includeToolResults: false,
  },
  rateLimit: {
    requestsPerMinute: 60,
    burstMultiplier: 2,
  },
};
```

```typescript
// config/envs/development.ts
import type { DeepPartial } from "../types";
import type { AgentConfig } from "../schema";

export const developmentOverride: DeepPartial<AgentConfig> = {
  env: "development",
  llm: {
    model: "claude-haiku-4-5",   // 最便宜
    budgetCents: 50,             // $0.50 上限，防手滑
    temperature: 1.0,            // 更随机，方便测边界
  },
  tools: {
    requireApprovalRisk: "NEVER", // 开发时不弹审批框
    maxRetries: 1,                // 快速失败
  },
  external: {
    useMocks: true,
    sandboxMode: true,
    webhookBaseUrl: "http://localhost:3000",
  },
  logging: {
    level: "debug",
    includeToolParams: true,    // 开发时看完整参数
    includeToolResults: true,
  },
  rateLimit: {
    requestsPerMinute: 300,     // 开发不限速
  },
};
```

```typescript
// config/envs/production.ts
export const productionOverride: DeepPartial<AgentConfig> = {
  env: "production",
  llm: {
    model: "claude-sonnet-4-5",
    budgetCents: 0,              // 生产不限额（由监控告警）
  },
  tools: {
    requireApprovalRisk: "HIGH", // HIGH 及以上需审批
    timeoutMs: 60_000,           // 生产给更多时间
    maxRetries: 5,
  },
  external: {
    useMocks: false,
    sandboxMode: false,
    webhookBaseUrl: "https://agent.example.com",
  },
  logging: {
    level: "warn",
    includeToolParams: false,    // 生产不记工具参数（防 PII）
    includeToolResults: false,
  },
  rateLimit: {
    requestsPerMinute: 120,
  },
};
```

---

## 三、ConfigLoader：合并 + 验证 + 缓存

```typescript
// config/loader.ts
import { merge } from "lodash";
import { AgentConfigSchema, type AgentConfig } from "./schema";
import { baseConfig } from "./base";
import { developmentOverride } from "./envs/development";
import { stagingOverride } from "./envs/staging";
import { productionOverride } from "./envs/production";

const ENV_OVERRIDES = {
  development: developmentOverride,
  staging: stagingOverride,
  production: productionOverride,
};

let _config: AgentConfig | null = null;

export function loadConfig(
  envOverride?: string,
  runtimePatch?: Partial<AgentConfig>
): AgentConfig {
  // 允许 CI 注入 TEST_ENV 覆盖 NODE_ENV
  const env = (envOverride ?? process.env.TEST_ENV ?? process.env.NODE_ENV ?? "development") as keyof typeof ENV_OVERRIDES;
  
  const envConfig = ENV_OVERRIDES[env] ?? ENV_OVERRIDES.development;
  
  // 三层合并：base → env → runtime patch
  const merged = merge({}, baseConfig, envConfig, runtimePatch ?? {});
  
  // Zod 验证：配置错误在启动时立刻爆出来，不要等到运行中
  const result = AgentConfigSchema.safeParse(merged);
  if (!result.success) {
    console.error("❌ Agent config validation failed:");
    console.error(result.error.format());
    process.exit(1); // 快速失败，不要带着破配置跑
  }
  
  return result.data;
}

// 单例模式：进程内只加载一次
export function getConfig(): AgentConfig {
  if (!_config) {
    _config = loadConfig();
  }
  return _config;
}

// 测试时重置（不然测试用例互相污染）
export function resetConfig(): void {
  _config = null;
}
```

---

## 四、与 Agent 工具层集成

配置加载好之后，工具中间件按配置调整行为：

```typescript
// tools/middleware/env-aware.ts
import { getConfig } from "../../config/loader";
import type { Tool, ToolCall, ToolResult } from "../types";

/**
 * 环境感知中间件：
 * - Dev: 用 Mock 替换真实工具
 * - Prod: 检查预算限制
 * - 所有环境: 统一超时 + 重试
 */
export function createEnvAwareMiddleware() {
  const config = getConfig();
  
  return async function envAwareMiddleware(
    call: ToolCall,
    next: (call: ToolCall) => Promise<ToolResult>
  ): Promise<ToolResult> {
    // 1. Mock 模式：直接返回假数据
    if (config.external.useMocks) {
      const mock = await getMockResult(call);
      if (mock) {
        log("debug", `[MOCK] ${call.name}`, call.params);
        return mock;
      }
    }
    
    // 2. 预算检查（仅非零预算时）
    if (config.llm.budgetCents > 0) {
      const spent = await getBudgetSpent();
      if (spent >= config.llm.budgetCents) {
        return {
          error: `Budget limit reached: ${spent}/${config.llm.budgetCents} cents`,
          budgetExceeded: true,
        };
      }
    }
    
    // 3. 带超时 + 重试执行
    return executeWithRetry(
      () => Promise.race([
        next(call),
        timeout(config.tools.timeoutMs, `Tool ${call.name} timed out`),
      ]),
      config.tools.maxRetries
    );
  };
}

async function executeWithRetry<T>(
  fn: () => Promise<T>,
  maxRetries: number
): Promise<T> {
  let lastError: Error;
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (err) {
      lastError = err as Error;
      if (attempt < maxRetries) {
        const delay = Math.min(1000 * 2 ** attempt, 30_000); // 指数退避，上限 30s
        await sleep(delay);
      }
    }
  }
  throw lastError!;
}
```

---

## 五、Mock 注册表 —— Dev 环境的核心

```typescript
// config/mocks/registry.ts
type MockFn = (params: Record<string, unknown>) => Promise<unknown>;

const mockRegistry = new Map<string, MockFn>();

export function registerMock(toolName: string, fn: MockFn): void {
  mockRegistry.set(toolName, fn);
}

export async function getMockResult(
  call: { name: string; params: Record<string, unknown> }
): Promise<unknown | null> {
  const fn = mockRegistry.get(call.name);
  return fn ? fn(call.params) : null; // 没有 Mock 就走真实工具
}

// ---- 注册 Dev Mock ----
registerMock("send_email", async ({ to, subject }) => ({
  messageId: `mock-${Date.now()}`,
  status: "delivered",
  _note: `[DEV MOCK] Would send to ${to}: "${subject}"`,
}));

registerMock("charge_user", async ({ amount, userId }) => ({
  transactionId: `mock-tx-${Date.now()}`,
  amount,
  _note: `[DEV MOCK] $${amount} charge to ${userId} NOT actually processed`,
}));

registerMock("list_files", async ({ path }) => ({
  files: ["mock-file-1.txt", "mock-file-2.json"],
  _note: `[DEV MOCK] Listing ${path}`,
}));
```

---

## 六、OpenClaw 实战 —— SOUL.md 是环境感知配置的最佳落地

OpenClaw 本身就是多环境配置管理的典范：

```
workspace/
├── SOUL.md          → 身份配置（"base config"：价值观、语气、语言）
├── USER.md          → 用户上下文（"env override"：权限、偏好、频道）
├── HEARTBEAT.md     → 运行时任务列表（"runtime patch"：当前任务注入）
└── TOOLS.md         → 工具配置（本地端点、API Key、设备别名）
```

- **SOUL.md** = base config，定义不变的核心行为
- **USER.md** = env override，按用户/频道定制权限
- **HEARTBEAT.md** = runtime patch，每次心跳动态注入当前任务

每次启动，OpenClaw 把三层合并成最终的 system prompt —— 这正是 `loadConfig()` 的文本化版本。

---

## 七、CI/CD 中的环境配置验证

```yaml
# .github/workflows/config-check.yml
name: Config Validation

on: [push, pull_request]

jobs:
  validate-configs:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Install deps
        run: npm ci
      
      - name: Validate all env configs
        run: |
          for env in development staging production; do
            echo "Checking $env config..."
            TEST_ENV=$env node -e "
              const { loadConfig } = require('./config/loader');
              loadConfig('$env');
              console.log('✅ $env config OK');
            "
          done
      
      - name: Check prod config has no mocks
        run: |
          node -e "
            const { loadConfig } = require('./config/loader');
            const cfg = loadConfig('production');
            if (cfg.external.useMocks) {
              console.error('❌ Production config has useMocks=true!');
              process.exit(1);
            }
            console.log('✅ Production mock check passed');
          "
```

---

## 八、常见陷阱

| 陷阱 | 症状 | 解法 |
|------|------|------|
| 开发时用了 `production` 配置 | 测试产生真实费用/副作用 | `NODE_ENV` 强制校验 + 启动时打印当前环境 |
| 配置合并用了引用（没深拷贝） | 修改一个环境配置污染另一个 | `lodash.merge` 或结构化克隆 |
| Zod 验证放在热路径 | 每次工具调用都验证，性能损耗 | 启动时验证一次，缓存结果 |
| Mock 忘记注册新工具 | Dev 环境调用真实 API | Mock 注册检查脚本（CI 运行） |
| 生产配置包含日志敏感信息 | 工具参数进日志泄露 PII | `includeToolParams: false` + 日志脱敏（第 93 课） |

---

## 九、一句话总结

> **Base → Env Overlay → Runtime Patch，三层叠加，Zod 验证，启动时快速失败。**  
> 配置错误必须在启动时暴露，不能等到生产环境凌晨三点爆。

---

## 配套阅读

- **第 93 课**：PII 脱敏与隐私护栏（`includeToolParams` 为何在 Prod 必须关）
- **第 185 课**：密钥管理与环境变量安全（配置分层 + 密钥存储结合使用）
- **第 189 课**：自然语言策略引擎（SOUL.md 作为 base config 的哲学来源）
- **第 98 课**：插件化架构（工具 Mock 注册表与插件注册表同构）
