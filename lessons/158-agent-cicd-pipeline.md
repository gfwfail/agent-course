# 158 - Agent CI/CD 流水线：自动化测试与持续部署

> Agent 开发不只是写代码——每次改动都可能让 LLM 行为漂移。CI/CD 是你抵御"悄悄坏掉"的最后防线。

---

## 🤔 为什么 Agent CI/CD 比普通软件更难？

普通软件：输入 → 确定性输出 → 断言 ✅/❌

Agent：输入 → LLM 生成 → 工具调用 → 再生成 → 输出（**非确定性**）

| 挑战 | 普通 CI | Agent CI |
|------|---------|----------|
| 输出断言 | `assert result == "hello"` | 需要 LLM 评判或语义相似度 |
| 测试成本 | 几乎 0 | 每次跑 = 真实 API 费用 |
| 回归检测 | 精确 diff | 行为漂移（prompt 改了 1 行就变了） |
| 部署验证 | 健康检查 `/health` | 需要金丝雀 + 行为基线对比 |

---

## 🏗️ Agent CI/CD 全貌

```
代码推送
   │
   ├─ 1. 单元测试（Mock 工具，不花 API 费）
   ├─ 2. 集成测试（真实 LLM，沙箱工具）
   ├─ 3. Eval 基线对比（行为回归检测）
   ├─ 4. 金丝雀部署（5% 流量 + 自动监控）
   └─ 5. 全量上线（或自动回滚）
```

---

## 📦 第一层：单元测试（Mock 一切，0 费用）

```typescript
// tests/unit/tool-dispatch.test.ts
import { AgentLoop } from "../../src/agent";
import { MockToolRegistry } from "../mocks/tool-registry";

describe("AgentLoop 工具分发", () => {
  let agent: AgentLoop;
  let mockRegistry: MockToolRegistry;

  beforeEach(() => {
    mockRegistry = new MockToolRegistry();
    // 注册 Mock 工具（不需要真实 API）
    mockRegistry.register("search_web", async (params) => ({
      results: [{ title: "Mock Result", url: "https://example.com" }]
    }));
    mockRegistry.register("read_file", async ({ path }) => ({
      content: `Mock content for ${path}`
    }));

    agent = new AgentLoop({
      model: "mock", // 使用 Mock LLM
      tools: mockRegistry,
    });
  });

  it("应该调用正确的工具", async () => {
    const response = await agent.run("搜索 TypeScript 教程");
    expect(mockRegistry.getCallCount("search_web")).toBe(1);
    expect(mockRegistry.getLastCall("search_web").params.query).toContain("TypeScript");
  });

  it("工具失败时应该优雅降级", async () => {
    mockRegistry.failNext("search_web", new Error("网络超时"));
    const response = await agent.run("搜索 TypeScript");
    // Agent 不应该崩溃，应该告知用户
    expect(response.content).toContain("无法搜索");
  });
});
```

---

## 🧪 第二层：集成测试（真实 LLM + 沙箱工具）

```typescript
// tests/integration/full-flow.test.ts
// 只在 CI 环境中跑（花真实 API 费）

const RUN_INTEGRATION = process.env.CI === "true";

(RUN_INTEGRATION ? describe : describe.skip)("完整 Agent 流程", () => {
  it("文件读写任务", async () => {
    const agent = new AgentLoop({
      model: process.env.TEST_MODEL || "claude-3-haiku-20240307", // 用便宜模型
      tools: createSandboxToolRegistry("/tmp/agent-test"), // 沙箱目录
      maxTokens: 1000,
      timeout: 30_000,
    });

    const result = await agent.run(
      "创建一个文件 /tmp/agent-test/hello.txt，内容是 'Hello CI'"
    );

    // 验证工具调用结果
    const fileContent = fs.readFileSync("/tmp/agent-test/hello.txt", "utf-8");
    expect(fileContent.trim()).toBe("Hello CI");
  }, 60_000); // 给 LLM 足够时间
});
```

---

## 📊 第三层：Eval 基线对比（行为回归检测）

这是 Agent CI/CD 最核心的部分——检测 **行为漂移**。

```typescript
// evals/runner.ts

interface EvalCase {
  id: string;
  input: string;
  expectedBehavior: {
    toolsCalled?: string[];        // 必须调用的工具
    toolsNotCalled?: string[];     // 不应该调用的工具
    outputContains?: string[];     // 输出应包含
    outputNotContains?: string[];  // 输出不应包含
    maxToolCalls?: number;         // 工具调用上限（防止循环）
    semanticSimilarTo?: string;    // 语义相似度基线
  };
}

const EVAL_CASES: EvalCase[] = [
  {
    id: "simple-query",
    input: "今天北京天气怎么样？",
    expectedBehavior: {
      toolsCalled: ["get_weather"],
      outputContains: ["温度", "天气"],
    },
  },
  {
    id: "no-hallucination",
    input: "2099年的股票价格是多少？",
    expectedBehavior: {
      toolsNotCalled: ["get_stock_price"], // 未来数据，不应该去查
      outputNotContains: ["¥", "$100"],   // 不应该编造价格
    },
  },
  {
    id: "multi-step-task",
    input: "搜索最新的 TypeScript 版本，然后写一个 Hello World",
    expectedBehavior: {
      toolsCalled: ["search_web"],
      maxToolCalls: 5, // 防止 Agent 陷入循环
    },
  },
];

async function runEvals(baselineFile?: string) {
  const results: EvalResult[] = [];

  for (const evalCase of EVAL_CASES) {
    const agent = createProductionAgent();
    const startTime = Date.now();
    
    try {
      const response = await agent.run(evalCase.input);
      const toolCalls = agent.getToolCallHistory();

      const result = assertEvalCase(evalCase, response, toolCalls);
      results.push({
        id: evalCase.id,
        passed: result.passed,
        duration: Date.now() - startTime,
        details: result.failures,
      });
    } catch (e) {
      results.push({ id: evalCase.id, passed: false, error: e.message });
    }
  }

  // 与基线对比
  if (baselineFile && fs.existsSync(baselineFile)) {
    const baseline = JSON.parse(fs.readFileSync(baselineFile, "utf-8"));
    const regression = detectRegression(baseline, results);
    if (regression.length > 0) {
      console.error("⚠️ 行为回归检测到！", regression);
      process.exit(1); // 阻断部署
    }
  }

  // 保存为新基线
  fs.writeFileSync("evals/baseline.json", JSON.stringify(results, null, 2));
  return results;
}

function detectRegression(
  baseline: EvalResult[],
  current: EvalResult[]
): string[] {
  const regressions: string[] = [];
  for (const curr of current) {
    const base = baseline.find((b) => b.id === curr.id);
    if (base?.passed && !curr.passed) {
      regressions.push(`${curr.id}: 之前通过，现在失败！`);
    }
  }
  return regressions;
}
```

---

## 🚀 第四层：GitHub Actions 完整流水线

```yaml
# .github/workflows/agent-ci.yml
name: Agent CI/CD Pipeline

on:
  push:
    branches: [main, dev]
  pull_request:
    branches: [main]

env:
  ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
  TEST_MODEL: "claude-3-haiku-20240307"  # CI 用便宜模型

jobs:
  # === 第一层：单元测试（无 API 费用）===
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: "22" }
      - run: npm ci
      - run: npm run test:unit
        env:
          ANTHROPIC_API_KEY: "" # 单元测试不需要真实 key

  # === 第二层：集成测试（有 API 费用，只在 main/PR 跑）===
  integration-tests:
    runs-on: ubuntu-latest
    needs: unit-tests
    if: github.ref == 'refs/heads/main' || github.event_name == 'pull_request'
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: "22" }
      - run: npm ci
      - run: npm run test:integration
        env:
          CI: "true"
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}

  # === 第三层：Eval 回归检测 ===
  eval-regression:
    runs-on: ubuntu-latest
    needs: unit-tests
    if: github.event_name == 'pull_request'
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
      - run: npm ci
      
      # 下载 main 分支的基线
      - name: Download baseline
        uses: actions/download-artifact@v4
        with:
          name: eval-baseline
          path: evals/
        continue-on-error: true  # 首次运行没有基线，不报错
      
      - name: Run Evals
        run: npm run eval
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
      
      # 上传新基线（供下次对比）
      - name: Upload baseline
        uses: actions/upload-artifact@v4
        with:
          name: eval-baseline
          path: evals/baseline.json
          retention-days: 30

  # === 第四层：部署到 Staging ===
  deploy-staging:
    runs-on: ubuntu-latest
    needs: [integration-tests, eval-regression]
    if: github.ref == 'refs/heads/main'
    environment: staging
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to staging
        run: |
          # 部署到 staging 环境
          fly deploy --app my-agent-staging --image ghcr.io/myorg/agent:${{ github.sha }}
        env:
          FLY_API_TOKEN: ${{ secrets.FLY_API_TOKEN }}
      
      # 金丝雀烟雾测试
      - name: Smoke test staging
        run: |
          sleep 10  # 等待部署完成
          curl -f https://my-agent-staging.fly.dev/health || exit 1
          npm run eval:smoke -- --url https://my-agent-staging.fly.dev

  # === 第五层：生产发布（手动审批）===
  deploy-production:
    runs-on: ubuntu-latest
    needs: deploy-staging
    environment: production  # 需要人工审批
    steps:
      - name: Deploy to production
        run: |
          fly deploy --app my-agent-prod --image ghcr.io/myorg/agent:${{ github.sha }}
```

---

## 🔄 OpenClaw 中的 CI 思路

OpenClaw 的技能系统其实就是一种轻量级 CI 机制：

```typescript
// 每个 Skill 有隐式的"契约"
// skills/weather/SKILL.md 定义了期望的工具调用行为

// 可以为 Skill 写 Eval
// evals/skills/weather.eval.ts
export const weatherSkillEvals: EvalCase[] = [
  {
    id: "weather-basic",
    input: "北京天气怎么样",
    triggerSkill: "weather",
    expectedBehavior: {
      toolsCalled: ["web_fetch"],    // 应该调用天气 API
      outputContains: ["°C", "天气"],
    },
  },
];

// 技能更新时自动跑 Eval
// openclaw skill test weather
```

---

## 💰 控制 CI 成本的 5 个技巧

```typescript
// 1. 按分支控制测试深度
const testLevel = process.env.GITHUB_REF?.includes("main") 
  ? "full"      // main 分支：完整测试
  : "quick";    // PR：快速测试

// 2. 用便宜模型跑 CI（行为一致性优先，不需要最聪明的模型）
const ciModel = process.env.CI 
  ? "claude-3-haiku-20240307"  // $0.25/M tokens
  : "claude-sonnet-4";         // $3/M tokens

// 3. 缓存 LLM 响应（相同输入复用）
const cachedAgent = new AgentLoop({
  model: ciModel,
  responseCache: new FileCache(".ci-cache/llm-responses"),
  cacheTTL: 24 * 60 * 60 * 1000, // 24小时
});

// 4. 并发跑 Eval（但限制总并发数）
const results = await pLimit(3)(  // 最多 3 个并发
  EVAL_CASES.map((c) => () => runEvalCase(c))
);

// 5. 只在 diff 影响到的文件时跑相关 Eval
const changedFiles = getChangedFiles();
const relevantEvals = EVAL_CASES.filter((e) => 
  e.relatedFiles?.some((f) => changedFiles.includes(f))
);
```

---

## 📈 关键 CI 指标监控

```typescript
// 每次 CI 运行后上报指标
async function reportCIMetrics(results: EvalResult[]) {
  const metrics = {
    passRate: results.filter(r => r.passed).length / results.length,
    avgDuration: avg(results.map(r => r.duration)),
    totalCost: sum(results.map(r => r.tokenCost ?? 0)),
    regressions: results.filter(r => r.isRegression).length,
    timestamp: Date.now(),
    commit: process.env.GITHUB_SHA,
  };

  // 写入 Prometheus / Grafana
  await pushMetrics("agent_ci", metrics);

  // 历史趋势告警
  if (metrics.passRate < 0.9) {
    await notify(`🚨 CI 通过率跌至 ${(metrics.passRate * 100).toFixed(1)}%！`);
  }
  if (metrics.regressions > 0) {
    await notify(`⚠️ 检测到 ${metrics.regressions} 个行为回归，阻断部署`);
  }
}
```

---

## 🎯 总结

| 层次 | 目的 | 成本 | 触发时机 |
|------|------|------|---------|
| 单元测试 | 逻辑正确性 | 免费 | 每次 push |
| 集成测试 | 端到端流程 | 低（便宜模型） | PR + main |
| Eval 回归 | 行为不漂移 | 中等 | PR |
| Staging 验证 | 部署可用 | 低（烟雾测试） | main 合并 |
| 生产发布 | 上线把关 | 低（手动审批） | 按需 |

**核心思想**：普通软件 CI 验证代码逻辑，Agent CI 还要验证 **LLM 行为**。把不确定性量化、基线化、自动化——才能真正放心地持续交付 Agent。

---

## 📚 延伸阅读

- [Lesson 14 - Agent Testing](./14-agent-testing.md)
- [Lesson 39 - Agent 评估框架（Evals）](./39-agent-evaluation-framework.md)
- [Lesson 100 - Agent 影子测试与金丝雀验证](./100-shadow-testing-canary.md)
