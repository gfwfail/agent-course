# 163 - Agent 工具代价函数与最优路径选择（Tool Cost Function & Optimal Path Selection）

> "同样是查用户信息，可以走缓存、可以查数据库、可以调第三方 API。Agent 怎么知道该选哪个？"

---

## 为什么需要代价函数

Agent 在执行任务时，经常面临**多条等效路径**：

- 查用户数据 → Redis 缓存 / MySQL / 第三方 CRM API
- 发通知 → 站内消息 / Email / Telegram / SMS
- 做计算 → 本地执行 / 调用 Python 脚本 / 云函数

每条路径的**代价不同**：
- 延迟（latency）
- 金钱成本（token/API 费用）
- 副作用风险（有些操作不可逆）
- 资源消耗（CPU/内存/并发占用）

如果没有代价函数，LLM 只能靠直觉或工具名称猜测最优选择，**容易选贵的、慢的、或高风险的路径**。

---

## 核心思想：给工具标注代价元数据

每个工具携带 `cost` 元数据：

```typescript
interface ToolCost {
  latencyMs: { p50: number; p95: number };     // 历史延迟分布
  moneyCostPerCall: number;                      // 美元/次
  sideEffectRisk: 'none' | 'low' | 'medium' | 'high'; // 副作用
  resourceWeight: number;                        // 资源权重 0-1
}

interface CostAwareTool {
  name: string;
  description: string;
  schema: JSONSchema;
  cost: ToolCost;
  alternatives?: string[];  // 等效备选工具
}
```

---

## 代价函数设计

把多维代价归一化为单一分数，方便比较：

```typescript
interface CostWeights {
  latency: number;    // 对延迟的权重
  money: number;      // 对成本的权重
  sideEffect: number; // 对副作用的权重
  resource: number;   // 对资源的权重
}

const SIDE_EFFECT_SCORES = {
  none: 0,
  low: 0.25,
  medium: 0.6,
  high: 1.0,
} as const;

function computeCostScore(
  tool: CostAwareTool,
  weights: CostWeights,
  context: { urgency: 'low' | 'medium' | 'high' }
): number {
  // 紧急任务时延迟权重加倍
  const latencyWeight =
    context.urgency === 'high' ? weights.latency * 2 : weights.latency;

  // 归一化各维度（越高越差）
  const latencyScore = tool.cost.latencyMs.p95 / 10_000; // 10s 为上限
  const moneyScore = Math.min(tool.cost.moneyCostPerCall / 0.01, 1); // 0.01 USD 为上限
  const sideEffectScore = SIDE_EFFECT_SCORES[tool.cost.sideEffectRisk];
  const resourceScore = tool.cost.resourceWeight;

  return (
    latencyScore * latencyWeight +
    moneyScore * weights.money +
    sideEffectScore * weights.sideEffect +
    resourceScore * weights.resource
  );
}
```

---

## 最优路径选择器

```typescript
class OptimalToolSelector {
  private tools: Map<string, CostAwareTool> = new Map();
  private weights: CostWeights = {
    latency: 0.4,
    money: 0.3,
    sideEffect: 0.2,
    resource: 0.1,
  };

  register(tool: CostAwareTool) {
    this.tools.set(tool.name, tool);
  }

  // 从一组候选工具中选最优
  selectBest(
    candidates: string[],
    context: { urgency: 'low' | 'medium' | 'high'; budget?: number }
  ): { tool: CostAwareTool; score: number } | null {
    let best: { tool: CostAwareTool; score: number } | null = null;

    for (const name of candidates) {
      const tool = this.tools.get(name);
      if (!tool) continue;

      // 预算硬过滤：超出 budget 的工具直接排除
      if (context.budget !== undefined &&
          tool.cost.moneyCostPerCall > context.budget) {
        continue;
      }

      const score = computeCostScore(tool, this.weights, context);
      if (!best || score < best.score) {
        best = { tool, score };
      }
    }

    return best;
  }

  // 查找某工具的所有等效替代方案并排序
  rankAlternatives(
    toolName: string,
    context: { urgency: 'low' | 'medium' | 'high' }
  ): Array<{ tool: CostAwareTool; score: number }> {
    const tool = this.tools.get(toolName);
    if (!tool?.alternatives) return [];

    const candidates = [toolName, ...tool.alternatives];
    return candidates
      .map((name) => {
        const t = this.tools.get(name);
        if (!t) return null;
        return { tool: t, score: computeCostScore(t, this.weights, context) };
      })
      .filter(Boolean)
      .sort((a, b) => a!.score - b!.score) as Array<{
        tool: CostAwareTool;
        score: number;
      }>;
  }
}
```

---

## 与工具中间件集成

代价选择注入到工具分发层，LLM 无感知：

```typescript
class CostAwareToolRegistry {
  private selector = new OptimalToolSelector();
  private tools: Map<string, Function> = new Map();

  register(tool: CostAwareTool, handler: Function) {
    this.selector.register(tool);
    this.tools.set(tool.name, handler);

    // 注册替代工具的 handler
    if (tool.alternatives) {
      // alternatives 共享同一逻辑，只是底层实现不同
    }
  }

  async dispatch(
    toolName: string,
    params: unknown,
    context: { urgency: 'low' | 'medium' | 'high'; budget?: number }
  ) {
    // 查找最优替代
    const ranked = this.selector.rankAlternatives(toolName, context);

    if (ranked.length > 0 && ranked[0].tool.name !== toolName) {
      const best = ranked[0];
      console.log(
        `[CostRouter] ${toolName} → ${best.tool.name} (score: ${best.score.toFixed(3)})`
      );
      toolName = best.tool.name;
    }

    const handler = this.tools.get(toolName);
    if (!handler) throw new Error(`Tool not found: ${toolName}`);

    return handler(params);
  }
}
```

---

## 实战：注册等效工具组

```typescript
const registry = new CostAwareToolRegistry();

// 方案一：Redis 缓存（快但可能过期）
registry.register(
  {
    name: 'get_user_cache',
    description: '从 Redis 缓存读取用户信息（可能过期）',
    schema: { type: 'object', properties: { userId: { type: 'string' } } },
    cost: {
      latencyMs: { p50: 2, p95: 10 },
      moneyCostPerCall: 0.000001,
      sideEffectRisk: 'none',
      resourceWeight: 0.05,
    },
    alternatives: ['get_user_db', 'get_user_api'],
  },
  async ({ userId }) => redis.get(`user:${userId}`)
);

// 方案二：MySQL 数据库（准确但较慢）
registry.register(
  {
    name: 'get_user_db',
    description: '从 MySQL 读取最新用户信息',
    schema: { type: 'object', properties: { userId: { type: 'string' } } },
    cost: {
      latencyMs: { p50: 15, p95: 80 },
      moneyCostPerCall: 0.00005,
      sideEffectRisk: 'none',
      resourceWeight: 0.3,
    },
    alternatives: ['get_user_cache', 'get_user_api'],
  },
  async ({ userId }) => db.query('SELECT * FROM users WHERE id = ?', [userId])
);

// 方案三：第三方 CRM API（最准确但贵且慢）
registry.register(
  {
    name: 'get_user_api',
    description: '从 CRM API 获取完整用户档案（实时）',
    schema: { type: 'object', properties: { userId: { type: 'string' } } },
    cost: {
      latencyMs: { p50: 200, p95: 800 },
      moneyCostPerCall: 0.002,
      sideEffectRisk: 'low',
      resourceWeight: 0.6,
    },
    alternatives: ['get_user_cache', 'get_user_db'],
  },
  async ({ userId }) => crmClient.getUser(userId)
);

// 使用：低紧急度 → 自动选 Redis；高紧急度且需要实时 → 动态选
const result = await registry.dispatch(
  'get_user_cache',
  { userId: '123' },
  { urgency: 'low', budget: 0.001 }
);
// 实际路由到 get_user_cache（最低代价）

const resultUrgent = await registry.dispatch(
  'get_user_cache',
  { userId: '123' },
  { urgency: 'high', budget: 0.01 }
);
// 延迟权重加倍 → 仍然选 Redis（它最快）
```

---

## 动态学习实际代价

历史代价比静态配置更准确：

```typescript
class AdaptiveCostLearner {
  private history: Map<string, number[]> = new Map();

  record(toolName: string, latencyMs: number) {
    if (!this.history.has(toolName)) {
      this.history.set(toolName, []);
    }
    const arr = this.history.get(toolName)!;
    arr.push(latencyMs);
    // 只保留最近 100 次
    if (arr.length > 100) arr.shift();
  }

  getP95(toolName: string): number | null {
    const arr = this.history.get(toolName);
    if (!arr || arr.length < 10) return null; // 样本不足

    const sorted = [...arr].sort((a, b) => a - b);
    return sorted[Math.floor(sorted.length * 0.95)];
  }

  // 将学习到的延迟注入工具元数据
  updateToolCost(tool: CostAwareTool): CostAwareTool {
    const p95 = this.getP95(tool.name);
    if (p95 === null) return tool;

    return {
      ...tool,
      cost: {
        ...tool.cost,
        latencyMs: {
          ...tool.cost.latencyMs,
          p95, // 用实测数据覆盖静态配置
        },
      },
    };
  }
}
```

---

## 在 OpenClaw/pi-mono 中应用

OpenClaw 的 `tool_use` 中间件层可以加入代价路由：

```typescript
// pi-mono: src/tools/costRouter.ts
export function costRouterMiddleware(
  registry: CostAwareToolRegistry,
  getContext: () => { urgency: 'low' | 'medium' | 'high' }
) {
  return async (toolName: string, params: unknown, next: Function) => {
    const context = getContext();
    // 透明路由到最优工具，LLM 无感知
    return registry.dispatch(toolName, params, context);
  };
}
```

对 LLM 暴露的工具列表保持不变，只在执行层做替换。这样 LLM 的 reasoning 不受影响，但实际执行走了最优路径。

---

## 与已讲内容的关系

| 课次 | 主题 | 关系 |
|------|------|------|
| 118 | Cost Attribution | 事后统计 vs 本课的事前预估 |
| 153 | Performance Profiling | 剖析历史 vs 本课指导选择 |
| 162 | Tool Discovery | 发现工具 vs 本课选最优工具 |
| 85 | Tool Prefetching | 预取下一步 vs 本课选最优当前步 |

---

## 关键结论

1. **代价函数 = 多维归一化评分**，延迟 / 金钱 / 副作用 / 资源
2. **工具注册时携带代价元数据**，不是在调用时猜测
3. **分发层透明路由**，LLM 不需要知道底层换了哪个工具
4. **动态学习实际延迟**，比静态配置更准确
5. **上下文影响权重**：紧急任务 → 延迟优先；省钱模式 → 成本优先

> 💡 Agent 不只要"能做"，还要"聪明地选择怎么做"。代价函数就是这个"聪明"的工程化实现。
