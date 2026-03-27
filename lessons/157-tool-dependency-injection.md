# 157 - Agent 工具依赖注入与 IoC 容器（Tool Dependency Injection & IoC Container）

> 把工具实现从 Agent 核心解耦，让测试、换环境、换模型都变得轻松。

---

## 🤔 为什么需要依赖注入？

典型反模式：工具硬编码在 Agent 里

```typescript
// ❌ 糟糕：Agent 直接 new 工具，无法测试、无法替换
class Agent {
  async run(prompt: string) {
    const db = new PostgresDatabase({ host: "prod.db" });  // 硬编码
    const email = new SendgridEmailSender();               // 硬编码
    // ...
  }
}
```

问题：
- 测试必须连真实数据库
- 换环境要改代码
- Mock 困难，测试覆盖率低
- 多租户时无法注入不同配置

---

## 🧩 核心概念：IoC Container

控制反转（Inversion of Control）：**不是 Agent 创建工具，而是容器注入工具**。

```
┌─────────────────────────────────┐
│         IoC Container           │
│  ┌──────────────────────────┐   │
│  │  bindings (interface→impl)│   │
│  └──────────────────────────┘   │
│        ↓ inject                 │
│  ┌──────────────────────────┐   │
│  │     Agent / Tools        │   │
│  └──────────────────────────┘   │
└─────────────────────────────────┘
```

---

## 🛠 实现：轻量 IoC Container

```typescript
// container.ts

type Factory<T> = () => T;
type Lifecycle = "singleton" | "transient";

interface Binding<T> {
  factory: Factory<T>;
  lifecycle: Lifecycle;
  instance?: T;
}

export class Container {
  private bindings = new Map<string, Binding<unknown>>();

  // 注册绑定
  bind<T>(token: string, factory: Factory<T>, lifecycle: Lifecycle = "singleton"): this {
    this.bindings.set(token, { factory, lifecycle });
    return this;
  }

  // 解析依赖
  resolve<T>(token: string): T {
    const binding = this.bindings.get(token);
    if (!binding) throw new Error(`[IoC] No binding for: ${token}`);

    if (binding.lifecycle === "singleton") {
      if (!binding.instance) {
        binding.instance = binding.factory();
      }
      return binding.instance as T;
    }

    // transient: 每次新建
    return binding.factory() as T;
  }

  // 检查是否已注册
  has(token: string): boolean {
    return this.bindings.has(token);
  }
}

// 全局容器单例
export const container = new Container();
```

---

## 🔌 定义工具接口与实现

```typescript
// interfaces.ts

export interface DatabaseTool {
  query(sql: string, params?: unknown[]): Promise<unknown[]>;
}

export interface EmailTool {
  send(to: string, subject: string, body: string): Promise<void>;
}

export interface CacheTool {
  get(key: string): Promise<string | null>;
  set(key: string, value: string, ttlSec?: number): Promise<void>;
}

// ---- 生产实现 ----
import { Pool } from "pg";

export class PostgresDatabaseTool implements DatabaseTool {
  private pool: Pool;
  constructor(connectionString: string) {
    this.pool = new Pool({ connectionString });
  }
  async query(sql: string, params?: unknown[]): Promise<unknown[]> {
    const result = await this.pool.query(sql, params as any[]);
    return result.rows;
  }
}

export class SendgridEmailTool implements EmailTool {
  constructor(private apiKey: string) {}
  async send(to: string, subject: string, body: string): Promise<void> {
    // 调用 Sendgrid API...
    console.log(`[Sendgrid] Send to ${to}: ${subject}`);
  }
}
```

---

## 🏭 容器配置：生产 vs 测试

```typescript
// container.prod.ts
import { container } from "./container";
import { PostgresDatabaseTool, SendgridEmailTool } from "./interfaces";
import Redis from "ioredis";

export function configureProd() {
  container
    .bind("db", () => new PostgresDatabaseTool(process.env.DATABASE_URL!))
    .bind("email", () => new SendgridEmailTool(process.env.SENDGRID_KEY!))
    .bind("cache", () => {
      const redis = new Redis(process.env.REDIS_URL!);
      return {
        get: (k: string) => redis.get(k),
        set: (k: string, v: string, ttl = 300) => redis.set(k, v, "EX", ttl),
      };
    });
}

// container.test.ts
import { Container } from "./container";

export function configureTest(): Container {
  const c = new Container();

  // 内存 Mock DB
  const rows = new Map<string, unknown[]>();
  c.bind("db", () => ({
    query: async (sql: string) => {
      console.log(`[MockDB] ${sql}`);
      return rows.get(sql) ?? [];
    },
  }));

  // 无操作 Email
  c.bind("email", () => ({
    send: async (to: string, subject: string) => {
      console.log(`[MockEmail] → ${to}: ${subject}`);
    },
  }));

  // 内存 Cache
  const store = new Map<string, string>();
  c.bind("cache", () => ({
    get: async (k: string) => store.get(k) ?? null,
    set: async (k: string, v: string) => { store.set(k, v); },
  }));

  return c;
}
```

---

## 🤖 Agent 使用注入的工具

```typescript
// agent.ts
import { container } from "./container";
import type { DatabaseTool, EmailTool, CacheTool } from "./interfaces";

export class DataAgent {
  private db: DatabaseTool;
  private email: EmailTool;
  private cache: CacheTool;

  // 构造函数注入（也可用属性注入）
  constructor(c = container) {
    this.db = c.resolve<DatabaseTool>("db");
    this.email = c.resolve<EmailTool>("email");
    this.cache = c.resolve<CacheTool>("cache");
  }

  async getUserReport(userId: string): Promise<string> {
    // 先查缓存
    const cacheKey = `report:${userId}`;
    const cached = await this.cache.get(cacheKey);
    if (cached) return cached;

    // 查数据库
    const rows = await this.db.query(
      "SELECT * FROM orders WHERE user_id = $1",
      [userId]
    );

    const report = `用户 ${userId} 共有 ${rows.length} 条订单`;

    // 写缓存
    await this.cache.set(cacheKey, report, 60);

    return report;
  }

  async notifyUser(userId: string, message: string): Promise<void> {
    const users = await this.db.query(
      "SELECT email FROM users WHERE id = $1",
      [userId]
    );
    if (users.length > 0) {
      const user = users[0] as { email: string };
      await this.email.send(user.email, "通知", message);
    }
  }
}
```

---

## ✅ 测试：完全不需要真实基础设施

```typescript
// agent.test.ts
import { DataAgent } from "./agent";
import { configureTest } from "./container.test";

describe("DataAgent", () => {
  it("getUserReport - 无缓存时查 DB", async () => {
    const c = configureTest();

    // 注入测试数据
    const db = c.resolve<any>("db");
    db.query = async () => [{ id: 1 }, { id: 2 }, { id: 3 }];

    const agent = new DataAgent(c);
    const report = await agent.getUserReport("user-42");

    expect(report).toContain("3 条订单");
  });

  it("getUserReport - 命中缓存不查 DB", async () => {
    const c = configureTest();
    const cache = c.resolve<any>("cache");
    await cache.set("report:user-99", "缓存结果", 60);

    let dbCalled = false;
    const db = c.resolve<any>("db");
    db.query = async () => { dbCalled = true; return []; };

    const agent = new DataAgent(c);
    const report = await agent.getUserReport("user-99");

    expect(report).toBe("缓存结果");
    expect(dbCalled).toBe(false);  // DB 未被调用 ✅
  });
});
```

---

## 🔗 在 OpenClaw Skills 中的应用

OpenClaw 的 Skills 系统本质上就是一种 DI：技能文件（SKILL.md）定义接口，工具实现在运行时注入。

```typescript
// openclaw skill 伪代码：技能就是"接口声明"
// SKILL.md 告诉 Agent 如何使用工具（接口）
// 实际工具（gh CLI、gog CLI 等）是注入的"实现"

// pi-mono 的工具注册也遵循类似模式:
// tools/registry.ts
export class ToolRegistry {
  private tools = new Map<string, ToolDefinition>();

  register(tool: ToolDefinition) {
    this.tools.set(tool.name, tool);  // 注册 = bind
  }

  dispatch(name: string, params: unknown) {
    const tool = this.tools.get(name);   // 解析 = resolve
    if (!tool) throw new Error(`Unknown tool: ${name}`);
    return tool.handler(params);
  }
}
```

---

## 🏗 进阶：装饰器语法（TypeScript + reflect-metadata）

```typescript
import "reflect-metadata";

// 注入 Token
export const TOKENS = {
  DB: "DatabaseTool",
  EMAIL: "EmailTool",
} as const;

// @Injectable 装饰器
function Injectable() {
  return (target: any) => {
    // 自动注册到全局容器
    const tokens: string[] = Reflect.getMetadata("inject:tokens", target) ?? [];
    container.bind(
      target.name,
      () => {
        const deps = tokens.map(t => container.resolve(t));
        return new target(...deps);
      }
    );
  };
}

function Inject(token: string) {
  return (target: any, _: string | symbol, index: number) => {
    const tokens = Reflect.getMetadata("inject:tokens", target) ?? [];
    tokens[index] = token;
    Reflect.defineMetadata("inject:tokens", tokens, target);
  };
}

// 使用装饰器
@Injectable()
class ReportAgent {
  constructor(
    @Inject(TOKENS.DB) private db: DatabaseTool,
    @Inject(TOKENS.EMAIL) private email: EmailTool
  ) {}
}
```

---

## 📊 生命周期对比

| 生命周期 | 描述 | 适用场景 |
|---------|------|---------|
| **singleton** | 容器内只创建一次 | DB 连接池、Redis 客户端 |
| **transient** | 每次 resolve 新建 | 无状态工具、请求级别隔离 |
| **scoped** | 每个请求/会话一个实例 | 多租户、会话级工具 |

---

## 🎯 总结

| 不用 DI | 用 DI |
|--------|-------|
| 工具硬编码，难以测试 | 注入 Mock，测试轻松 |
| 换环境要改代码 | 只改容器配置 |
| 多租户难以隔离 | 每租户独立容器 |
| 依赖隐藏在代码深处 | 依赖关系一目了然 |

**三行哲学**：
1. **面向接口编程**，不依赖具体实现
2. **容器管理生命周期**，不自己 new
3. **测试传入 Mock 容器**，不动生产代码

> OpenClaw 的 Skills、pi-mono 的 ToolRegistry、learn-claude-code 的工具分发——本质都是 DI 的变体。理解了 DI，就理解了 Agent 工具系统的灵魂。
