# 136 - Agent 依赖注入与工具工厂（Dependency Injection & Tool Factory）

> "好的工具不是写死的，是注入进来的。"

---

## 问题背景

绝大多数 Agent 教程里，工具的依赖是这样写的：

```typescript
// ❌ 硬编码依赖 —— 无法测试，无法替换
async function get_user_orders(userId: string) {
  const db = new MySQLConnection("prod-db.internal:3306"); // 直接 new！
  const http = axios.create({ baseURL: "https://api.stripe.com" }); // 直接创建！
  
  const orders = await db.query(`SELECT * FROM orders WHERE user_id = ?`, [userId]);
  const payments = await http.get(`/v1/charges?customer=${userId}`);
  return { orders, payments };
}
```

这种写法有三个致命问题：

1. **无法测试**：每次调用都真正连接数据库和 Stripe
2. **无法替换**：测试环境想用 SQLite，生产用 MySQL？改代码
3. **无法复用**：10 个工具，10 份连接池，资源浪费

**解法：依赖注入（DI）+ 工具工厂（Tool Factory）**

---

## 核心模式

### 1. 容器（Container）

```typescript
// di-container.ts
type Factory<T> = (container: Container) => T;

class Container {
  private bindings = new Map<string, Factory<any>>();
  private singletons = new Map<string, any>();

  // 注册工厂
  bind<T>(token: string, factory: Factory<T>): void {
    this.bindings.set(token, factory);
  }

  // 注册单例（连接池这种只创建一次）
  singleton<T>(token: string, factory: Factory<T>): void {
    this.bind(token, (c) => {
      if (!this.singletons.has(token)) {
        this.singletons.set(token, factory(c));
      }
      return this.singletons.get(token);
    });
  }

  // 解析依赖
  resolve<T>(token: string): T {
    const factory = this.bindings.get(token);
    if (!factory) throw new Error(`No binding for token: ${token}`);
    return factory(this);
  }
}
```

### 2. 工具工厂（Tool Factory）

```typescript
// tool-factory.ts
interface ToolDeps {
  db: DatabaseClient;
  http: HttpClient;
  cache: CacheClient;
  logger: Logger;
}

type ToolFactory<TInput, TOutput> = (deps: ToolDeps) => {
  name: string;
  description: string;
  schema: object;
  execute: (input: TInput) => Promise<TOutput>;
};

// ✅ 工具定义：只描述行为，不管依赖从哪来
const createGetUserOrdersTool: ToolFactory<
  { userId: string },
  { orders: Order[]; payments: Payment[] }
> = (deps) => ({
  name: "get_user_orders",
  description: "获取用户的订单和支付记录",
  schema: {
    type: "object",
    properties: {
      userId: { type: "string", description: "用户 ID" }
    },
    required: ["userId"]
  },
  execute: async ({ userId }) => {
    const cacheKey = `orders:${userId}`;
    
    // 使用注入的 cache
    const cached = await deps.cache.get(cacheKey);
    if (cached) return JSON.parse(cached);

    // 使用注入的 db
    const orders = await deps.db.query(
      `SELECT * FROM orders WHERE user_id = ?`,
      [userId]
    );

    // 使用注入的 http
    const { data: payments } = await deps.http.get(`/charges`, {
      params: { customer: userId }
    });

    const result = { orders, payments };
    await deps.cache.set(cacheKey, JSON.stringify(result), { ttl: 60 });
    
    deps.logger.info("get_user_orders", { userId, orderCount: orders.length });
    return result;
  }
});
```

---

## 完整实现

### 3. 环境配置绑定

```typescript
// bootstrap.ts — 生产环境
function createProductionContainer(): Container {
  const c = new Container();

  // 数据库连接池（单例）
  c.singleton("db", () => new MySQLClient({
    host: process.env.DB_HOST!,
    user: process.env.DB_USER!,
    password: process.env.DB_PASSWORD!,
    database: process.env.DB_NAME!,
    pool: { min: 2, max: 10 }
  }));

  // HTTP 客户端（单例）
  c.singleton("http", () => axios.create({
    baseURL: process.env.STRIPE_BASE_URL!,
    headers: { Authorization: `Bearer ${process.env.STRIPE_KEY}` },
    timeout: 10_000
  }));

  // Redis 缓存（单例）
  c.singleton("cache", () => new RedisClient({
    host: process.env.REDIS_HOST!,
    port: 6379
  }));

  // Logger
  c.bind("logger", () => new PinoLogger({ level: "info" }));

  return c;
}

// bootstrap.test.ts — 测试环境（完全替换！）
function createTestContainer(): Container {
  const c = new Container();

  c.bind("db", () => new InMemoryDatabase());     // SQLite / Map
  c.bind("http", () => new MockHttpClient());      // 录播回放
  c.bind("cache", () => new LocalCache());         // 本地 Map
  c.bind("logger", () => new SilentLogger());      // 不输出日志

  return c;
}
```

### 4. 工具注册中心整合

```typescript
// tool-registry.ts
class DIToolRegistry {
  private tools: Map<string, ReturnType<ToolFactory<any, any>>> = new Map();

  constructor(private container: Container) {}

  register<TInput, TOutput>(factory: ToolFactory<TInput, TOutput>): void {
    const deps = {
      db:     this.container.resolve<DatabaseClient>("db"),
      http:   this.container.resolve<HttpClient>("http"),
      cache:  this.container.resolve<CacheClient>("cache"),
      logger: this.container.resolve<Logger>("logger"),
    };
    const tool = factory(deps);
    this.tools.set(tool.name, tool);
  }

  // 生成 Anthropic tools 数组
  toAnthropicTools(): Anthropic.Tool[] {
    return [...this.tools.values()].map(t => ({
      name: t.name,
      description: t.description,
      input_schema: t.schema as Anthropic.Tool["input_schema"]
    }));
  }

  // 执行工具
  async execute(name: string, input: unknown): Promise<unknown> {
    const tool = this.tools.get(name);
    if (!tool) throw new Error(`Tool not found: ${name}`);
    return tool.execute(input as any);
  }
}
```

### 5. Agent Loop 集成

```typescript
// agent.ts
async function runAgent(userMessage: string, env: "production" | "test") {
  // 根据环境选容器
  const container = env === "production"
    ? createProductionContainer()
    : createTestContainer();

  // 注册工具
  const registry = new DIToolRegistry(container);
  registry.register(createGetUserOrdersTool);
  registry.register(createSearchProductsTool);
  registry.register(createSendNotificationTool);

  const client = new Anthropic();
  const messages: Anthropic.MessageParam[] = [
    { role: "user", content: userMessage }
  ];

  while (true) {
    const response = await client.messages.create({
      model: "claude-opus-4-5",
      max_tokens: 4096,
      tools: registry.toAnthropicTools(), // ← 自动生成
      messages
    });

    if (response.stop_reason === "end_turn") break;

    if (response.stop_reason === "tool_use") {
      const toolResults = await Promise.all(
        response.content
          .filter(b => b.type === "tool_use")
          .map(async (block) => {
            if (block.type !== "tool_use") return null;
            const result = await registry.execute(block.name, block.input);
            return {
              type: "tool_result" as const,
              tool_use_id: block.id,
              content: JSON.stringify(result)
            };
          })
      );

      messages.push({ role: "assistant", content: response.content });
      messages.push({ role: "user", content: toolResults.filter(Boolean) as any });
    }
  }
}
```

---

## OpenClaw Skills 里的 DI 模式

OpenClaw 的 Skills 系统本质上就是一种轻量级 DI：

```
skill 目录结构:
├── SKILL.md          ← 描述 (what)
├── scripts/          ← 实现 (how)  
└── references/       ← 注入的上下文 (dependencies)
```

当 Agent 加载一个 Skill 时，它把 `references/` 里的文档"注入"到执行上下文中 —— 这就是文档级别的依赖注入。

```typescript
// OpenClaw 内部 (简化)
async function loadSkill(skillName: string) {
  const skillMd = await readFile(`skills/${skillName}/SKILL.md`);
  const refs = await loadReferences(`skills/${skillName}/references/`);
  
  // 把依赖注入到 Agent 执行上下文
  return {
    instructions: skillMd,
    context: refs, // ← 依赖注入
  };
}
```

---

## pi-mono 中的实现参考

pi-mono 使用 `ToolContext` 对象作为依赖载体：

```typescript
// pi-mono/src/tools/base.ts (简化)
export interface ToolContext {
  session: Session;
  storage: StorageAdapter;
  messaging: MessagingAdapter;
  config: Config;
}

export abstract class BaseTool<TInput, TOutput> {
  constructor(protected ctx: ToolContext) {} // ← 构造器注入
  
  abstract execute(input: TInput): Promise<TOutput>;
}

// 具体工具
export class SendMessageTool extends BaseTool<
  { channel: string; text: string },
  { messageId: string }
> {
  async execute({ channel, text }) {
    return this.ctx.messaging.send(channel, text); // 使用注入的 adapter
  }
}
```

---

## 测试效果

```typescript
// get_user_orders.test.ts
test("正确聚合订单和支付记录", async () => {
  const container = createTestContainer();
  
  // 注入测试数据
  const db = container.resolve<InMemoryDatabase>("db");
  db.seed("orders", [
    { id: "ord_1", user_id: "usr_123", amount: 99 }
  ]);
  
  const http = container.resolve<MockHttpClient>("http");
  http.mock("GET", "/charges", {
    data: [{ id: "ch_1", customer: "usr_123", amount: 99 }]
  });

  // 注册并执行工具
  const registry = new DIToolRegistry(container);
  registry.register(createGetUserOrdersTool);
  
  const result = await registry.execute("get_user_orders", {
    userId: "usr_123"
  });

  expect(result).toEqual({
    orders: [{ id: "ord_1", user_id: "usr_123", amount: 99 }],
    payments: [{ id: "ch_1", customer: "usr_123", amount: 99 }]
  });
  
  // ✅ 零真实网络调用，运行 <5ms
});
```

---

## 关键收益对比

| 指标 | 硬编码依赖 | DI + Tool Factory |
|------|-----------|-------------------|
| 单元测试 | ❌ 需要真实 DB/API | ✅ 完全 Mock |
| 环境切换 | ❌ 改代码 | ✅ 换容器 |
| 连接池 | ❌ 每个工具各一份 | ✅ 全局共享 |
| 工具复用 | ❌ 依赖耦合死 | ✅ 换注入即换行为 |
| CI 速度 | ❌ 需要启动外部服务 | ✅ 纯内存运行 |

---

## 一句话总结

> **工具工厂**描述"做什么"，**依赖注入容器**决定"用什么做"。两者分离，Agent 工具就能像乐高一样自由组合、替换和测试。

---

*Lesson 136 | Agent 开发课程 | 2026-03-25*
