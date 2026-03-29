# 167. Agent 压测与容量规划（Load Testing & Capacity Planning）

> Agent 系统不是普通 REST API——它有状态、烧 Token、调工具、跑子代理。
> 不压测就上生产，等于开着眼睛跳崖。

---

## 为什么 Agent 压测比普通 API 难？

| 普通 API | Agent 系统 |
|----------|-----------|
| 请求无状态 | 会话有状态（memory/context） |
| 延迟可预测 ms 级 | 延迟不可预测 1s–60s |
| 成本固定 | 成本随 Token 数波动 |
| 失败可直接重试 | 重试可能导致重复工具调用 |
| 并发无上限 | 受 LLM API 并发/RPM 双重限制 |

所以 Agent 压测要关注三个维度：

1. **吞吐量**：每秒能处理多少并发会话？
2. **延迟分布**：P50/P95/P99 首 Token 时间、完整响应时间？
3. **成本效率**：每请求平均 Token 消耗，总费用预估？

---

## 核心指标定义

```typescript
interface AgentLoadMetrics {
  // 吞吐量
  sessionsPerSecond: number;       // 新建会话/s
  toolCallsPerSecond: number;      // 工具调用/s
  
  // 延迟
  timeToFirstToken_p50: number;    // ms
  timeToFirstToken_p99: number;    // ms
  totalResponseTime_p95: number;   // ms（含所有工具调用）
  
  // 资源
  avgInputTokens: number;          // 每会话平均输入 Token
  avgOutputTokens: number;         // 每会话平均输出 Token
  concurrentSessions: number;      // 峰值并发会话数
  
  // 错误
  llmRateLimitRate: number;        // LLM 429 错误率
  toolTimeoutRate: number;         // 工具超时率
  sessionErrorRate: number;        // 会话级错误率
}
```

---

## 方法一：k6 压测 Agent HTTP API

```javascript
// load-test-agent.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate, Trend, Counter } from 'k6/metrics';

// 自定义指标
const firstTokenTime = new Trend('agent_first_token_ms');
const totalResponseTime = new Trend('agent_total_response_ms');
const tokenCount = new Counter('agent_total_tokens');
const errorRate = new Rate('agent_error_rate');

export const options = {
  stages: [
    { duration: '1m',  target: 5  },  // 预热：爬坡到 5 并发
    { duration: '3m',  target: 20 },  // 稳定：20 并发
    { duration: '2m',  target: 50 },  // 压力：50 并发
    { duration: '1m',  target: 80 },  // 峰值：80 并发
    { duration: '2m',  target: 0  },  // 降温
  ],
  thresholds: {
    // SLA 红线
    'agent_first_token_ms{p:95}': ['<3000'],   // P95 首 Token < 3s
    'agent_total_response_ms{p:95}': ['<30000'], // P95 完整响应 < 30s
    'agent_error_rate': ['<0.01'],               // 错误率 < 1%
  },
};

// 模拟真实用户消息场景
const testMessages = [
  "帮我查一下今天天气",
  "搜索最新的 AI 新闻",
  "计算 1234567 * 9876543",
  "列出我的待办事项",
  "发送邮件给 test@example.com 说 hello",
];

export default function () {
  const msg = testMessages[Math.floor(Math.random() * testMessages.length)];
  const startTime = Date.now();

  // 发起 Agent 请求（SSE 流式）
  const res = http.post(
    'http://localhost:3000/api/agent/run',
    JSON.stringify({
      message: msg,
      sessionId: `load-test-${__VU}-${__ITER}`,  // 每个 VU 独立会话
    }),
    {
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${__ENV.API_KEY}`,
      },
      timeout: '60s',
    }
  );

  const elapsed = Date.now() - startTime;
  
  const ok = check(res, {
    'status 200': (r) => r.status === 200,
    'has response': (r) => r.body && r.body.length > 0,
  });

  if (!ok) {
    errorRate.add(1);
    return;
  }
  
  errorRate.add(0);
  totalResponseTime.add(elapsed);
  
  // 从响应头提取首 Token 时间（需要服务端注入）
  const ftt = res.headers['X-First-Token-Ms'];
  if (ftt) firstTokenTime.add(parseInt(ftt));

  // 统计 Token 消耗（需要服务端在响应中返回）
  try {
    const body = JSON.parse(res.body);
    if (body.usage) {
      tokenCount.add(body.usage.total_tokens);
    }
  } catch (_) {}

  // 模拟真实用户思考间隔（1–3 秒）
  sleep(1 + Math.random() * 2);
}

// 压测结束后输出容量报告
export function handleSummary(data) {
  const p95ResponseTime = data.metrics['agent_total_response_ms']?.values?.['p(95)'];
  const rps = data.metrics['http_reqs']?.values?.rate;
  const errRate = data.metrics['agent_error_rate']?.values?.rate;
  
  console.log('\n==== Agent 容量规划报告 ====');
  console.log(`峰值 RPS: ${rps?.toFixed(2)}`);
  console.log(`P95 响应时间: ${p95ResponseTime?.toFixed(0)}ms`);
  console.log(`错误率: ${(errRate * 100)?.toFixed(2)}%`);
  
  return {
    'stdout': JSON.stringify(data, null, 2),
    './load-test-report.json': JSON.stringify(data),
  };
}
```

```bash
# 运行压测
k6 run -e API_KEY=your_key load-test-agent.js

# 并行压测多个场景
k6 run --scenario chat load-test-agent.js \
       --scenario search search-load-test.js
```

---

## 方法二：Locust Python 版（更适合复杂 Agent 场景）

```python
# locustfile.py
import time
import json
import random
from locust import HttpUser, task, between, events
from locust.runners import MasterRunner

class AgentUser(HttpUser):
    wait_time = between(1, 3)  # 思考间隔
    
    def on_start(self):
        """会话初始化——模拟用户登录"""
        self.session_id = f"test-{self.user_id}-{int(time.time())}"
        self.token_count = 0
        
    @task(3)  # 权重 3：最常见的简单查询
    def simple_query(self):
        self._run_agent("今天天气怎么样？")
    
    @task(2)  # 权重 2：需要工具调用
    def tool_calling(self):
        self._run_agent("帮我搜索最新的 OpenAI 新闻，总结前三条")
    
    @task(1)  # 权重 1：复杂多轮对话
    def multi_turn(self):
        messages = [
            "我想订一张从上海到北京的机票",
            "明天出发，下午的航班",
            "经济舱就好，帮我比较几个选项",
        ]
        for msg in messages:
            self._run_agent(msg)
            time.sleep(random.uniform(0.5, 1.5))
    
    def _run_agent(self, message: str):
        start = time.time()
        first_token_time = None
        
        with self.client.post(
            "/api/agent/run",
            json={"message": message, "sessionId": self.session_id},
            headers={"Authorization": f"Bearer {self.api_key}"},
            stream=True,
            catch_response=True,
            timeout=60,
        ) as response:
            if response.status_code != 200:
                response.failure(f"HTTP {response.status_code}")
                return
            
            # 流式读取，记录首 Token 时间
            for chunk in response.iter_lines():
                if chunk and first_token_time is None:
                    first_token_time = (time.time() - start) * 1000  # ms
                    # 上报自定义指标
                    events.request.fire(
                        request_type="agent_first_token",
                        name="first_token_latency",
                        response_time=first_token_time,
                        response_length=0,
                    )
            response.success()

@events.test_stop.add_listener
def on_test_stop(environment, **kwargs):
    """压测结束时输出容量建议"""
    stats = environment.stats
    total_rps = stats.total.current_rps
    p95 = stats.total.get_response_time_percentile(0.95)
    failures = stats.total.fail_ratio
    
    print("\n========== 容量规划建议 ==========")
    
    if failures > 0.05:
        print(f"⚠️  错误率 {failures:.1%} 超标，当前配置无法支撑此负载")
    elif p95 > 30000:
        print(f"⚠️  P95 响应时间 {p95/1000:.1f}s 超过 SLA，需要扩容")
    else:
        print(f"✅  当前配置可稳定支撑 {total_rps:.1f} RPS")
        print(f"   P95 响应时间: {p95/1000:.1f}s")
```

```bash
# 运行 Locust（Web UI 模式）
locust -f locustfile.py --host=http://localhost:3000

# 无头模式（CI/CD）
locust -f locustfile.py --host=http://localhost:3000 \
  --users 50 --spawn-rate 5 --run-time 5m \
  --headless --csv=results
```

---

## 方法三：识别 Agent 系统的三大瓶颈

```typescript
// bottleneck-analyzer.ts
import { AgentLoadMetrics } from './types';

type Bottleneck = 'LLM_API' | 'TOOL_EXECUTION' | 'MEMORY_STORAGE' | 'NONE';

interface CapacityReport {
  bottleneck: Bottleneck;
  maxConcurrentSessions: number;
  estimatedMonthlyCost: number;
  recommendations: string[];
}

function analyzeBottleneck(metrics: AgentLoadMetrics): CapacityReport {
  const recommendations: string[] = [];
  
  // 瓶颈 1：LLM API 限流（429 错误率 > 2%）
  if (metrics.llmRateLimitRate > 0.02) {
    const requiredRPM = metrics.sessionsPerSecond * 60 * 3; // 每会话约 3 次 LLM 调用
    const tierNeeded = Math.ceil(requiredRPM / 10000); // Anthropic Tier 每级 ~10k RPM
    
    return {
      bottleneck: 'LLM_API',
      maxConcurrentSessions: estimateMaxSessions(metrics),
      estimatedMonthlyCost: estimateMonthlyCost(metrics),
      recommendations: [
        `升级 Anthropic API 额度到 Tier ${tierNeeded}`,
        '实现多 API Key 轮转（第 96 课：去重合并）',
        '开启 Prompt Caching 降低重复 Token 消耗（第 84 课）',
        '对简单任务路由到 claude-haiku 降低成本（第 91 课：多模型回退）',
      ],
    };
  }
  
  // 瓶颈 2：工具执行慢（工具超时率 > 1% 或 P95 > 10s）
  if (metrics.toolTimeoutRate > 0.01 || metrics.totalResponseTime_p95 > 10000) {
    return {
      bottleneck: 'TOOL_EXECUTION',
      maxConcurrentSessions: estimateMaxSessions(metrics),
      estimatedMonthlyCost: estimateMonthlyCost(metrics),
      recommendations: [
        '实现工具调用预取（第 85 课：预测预取）',
        '工具结果加语义缓存（第 27 课：语义缓存）',
        '慢工具改为异步执行（第 130 课：异步工具轮询）',
        '并行执行无依赖工具（第 37 课：并发工具执行）',
      ],
    };
  }
  
  // 瓶颈 3：内存/数据库（会话错误率高但 LLM/工具正常）
  if (metrics.sessionErrorRate > 0.05 && metrics.llmRateLimitRate < 0.01) {
    return {
      bottleneck: 'MEMORY_STORAGE',
      maxConcurrentSessions: estimateMaxSessions(metrics),
      estimatedMonthlyCost: estimateMonthlyCost(metrics),
      recommendations: [
        '会话存储从内存迁移到 Redis（第 86 课：上下文外部化）',
        '启用连接池，防止 DB 连接耗尽',
        '对历史消息实现压缩/归档（第 35 课：滑动窗口摘要）',
      ],
    };
  }
  
  return {
    bottleneck: 'NONE',
    maxConcurrentSessions: estimateMaxSessions(metrics),
    estimatedMonthlyCost: estimateMonthlyCost(metrics),
    recommendations: ['当前系统健康，可继续扩大压测规模验证上限'],
  };
}

function estimateMaxSessions(m: AgentLoadMetrics): number {
  // 基于 LLM API 限制估算最大并发会话
  // Anthropic claude-sonnet 默认 ~40 RPM
  const llmCallsPerSession = 3; // 平均每会话 3 次 LLM 调用
  const rpm = 40; // 你的 API tier 的 RPM
  return Math.floor(rpm / llmCallsPerSession);
}

function estimateMonthlyCost(m: AgentLoadMetrics): number {
  // Anthropic claude-sonnet-4-5 定价（示例）
  const inputCostPer1M = 3.0;   // $3/M input tokens
  const outputCostPer1M = 15.0; // $15/M output tokens
  
  const monthlyInputTokens = m.avgInputTokens * m.sessionsPerSecond * 86400 * 30;
  const monthlyOutputTokens = m.avgOutputTokens * m.sessionsPerSecond * 86400 * 30;
  
  return (
    (monthlyInputTokens / 1_000_000) * inputCostPer1M +
    (monthlyOutputTokens / 1_000_000) * outputCostPer1M
  );
}
```

---

## 容量规划公式

```
最大并发会话数 = min(
  LLM_RPM / (每会话平均LLM调用次数),
  最大DB连接数 / 每会话DB连接数,
  可用内存MB / 每会话内存占用MB
)

月度成本估算（美元）= 
  日活用户数 × 每用户日均会话数 × 
  (平均输入Token × $input/M + 平均输出Token × $output/M) / 1,000,000

扩容触发阈值：
  - LLM 429 率 > 1%    → 升级 API Tier 或增加 Key 轮转
  - P95 响应 > 15s      → 检查工具瓶颈，考虑异步化
  - 错误率 > 0.5%       → 立即告警，触发自动扩容
```

---

## OpenClaw Cron 定时压测（每日凌晨自动跑）

```typescript
// cron-load-test.ts
// 在 OpenClaw Cron 中配置：每天 03:00 AM 跑轻量压测，结果推 Telegram

import { exec } from 'child_process';
import { promisify } from 'util';

const execAsync = promisify(exec);

async function scheduledLoadTest() {
  console.log('🔥 开始日常容量巡检...');
  
  // 轻量压测：5 并发跑 2 分钟
  const { stdout } = await execAsync(
    'k6 run --vus 5 --duration 2m --out json=./results.json load-test-agent.js',
    { timeout: 3 * 60 * 1000 }
  );
  
  // 解析关键指标
  const results = parseK6Results('./results.json');
  const report = analyzeBottleneck(results);
  
  // 只在发现问题时告警
  if (report.bottleneck !== 'NONE') {
    await sendTelegramAlert(
      `⚠️ Agent 容量告警\n` +
      `瓶颈：${report.bottleneck}\n` +
      `建议：\n${report.recommendations.map(r => `• ${r}`).join('\n')}`
    );
  } else {
    console.log('✅ 容量巡检通过，系统健康');
  }
}
```

---

## 真实场景：OpenClaw 容量基线（参考值）

```
测试环境：Mac Studio M2 Ultra，本地运行
模型：claude-sonnet-4-5（Anthropic Tier 1）

结果：
  最大稳定并发会话：12（受 RPM 40 限制）
  P95 首 Token 时间：1.2s
  P95 完整响应时间：8.4s（含 2 次工具调用）
  每会话平均 Token：1,200 input + 400 output
  每会话平均成本：$0.0096

扩容至 100 并发需要：
  升级至 Anthropic Tier 3（RPM 300+）
  启用 Prompt Caching（减少 60% input tokens）
  启用多 Key 轮转（至少 3 个 API Key）
  预计成本：$0.004/会话（Caching 后降低 60%）
```

---

## 小结

```
压测三件套：
  k6          → 轻量、脚本化、适合 CI/CD 集成
  Locust      → Python 生态，复杂场景建模更灵活
  自定义工具   → 深度 Agent 指标采集（Token/工具调用链）

关注三大瓶颈：
  LLM API 限流  → 升级 Tier / 多 Key 轮转 / Caching
  工具执行慢    → 异步化 / 预取 / 缓存
  存储层       → Redis / 连接池 / 上下文压缩

先测再上线，别等生产炸了再回头找原因。
```

---

> 相关课程：
> - [第 84 课：Prompt Caching](84-prompt-caching-cache-control.md) - 减少 Token 消耗降成本
> - [第 85 课：工具调用预取](85-tool-call-prediction-prefetching.md) - 消灭工具调用等待
> - [第 130 课：异步工具执行](130-async-tool-execution-polling.md) - 慢工具不阻塞 Agent Loop
> - [第 153 课：性能剖析](153-tool-call-performance-profiling.md) - 精确找到系统瓶颈
> - [第 161 课：自适应并发控制](161-adaptive-concurrency-control.md) - 动态调整工具并发
