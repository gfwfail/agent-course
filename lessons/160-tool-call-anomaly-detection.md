# 160 - Agent 工具调用异常检测与行为基线

**Tool Call Anomaly Detection & Behavioral Baseline**

---

## 核心问题

Agent 在生产环境中运行久了，你怎么知道它"行为正常"？

- 某个工具突然被调用 50 次/分钟（正常是 2 次）
- 某个参数值出现了从未见过的极端值
- 某个工具组合从没出现过，但现在频繁出现

这些可能是：**Prompt 注入攻击、配置错误、Agent 陷入循环、用户滥用**。

传统监控只看"是否报错"，行为基线检测看"是否异常"——即使没有报错，异常行为也值得告警。

---

## 核心架构：三层模型

```
历史数据 → 基线建模 → 实时对比 → 异常评分 → 告警/拦截
```

### 基线指标

| 维度 | 指标 |
|------|------|
| 调用频率 | 每分钟调用次数（rolling window） |
| 参数分布 | 数值型参数的均值/标准差 |
| 工具组合 | 同一 session 内工具序列的 N-gram |
| 调用深度 | 嵌套调用层级 |
| 错误率 | 每个工具的失败比例 |

---

## TypeScript 实现

### 1. 基线存储（Redis）

```typescript
// baseline-store.ts
import Redis from 'ioredis'

const redis = new Redis()

interface ToolBaseline {
  callsPerMinute: { mean: number; stdDev: number; samples: number }
  errorRate: { mean: number; stdDev: number }
  paramStats: Record<string, { mean: number; stdDev: number }>
  lastUpdated: number
}

export async function updateBaseline(
  toolName: string,
  callsPerMin: number,
  isError: boolean,
  numericParams: Record<string, number>
) {
  const key = `baseline:tool:${toolName}`
  const raw = await redis.get(key)
  const baseline: ToolBaseline = raw ? JSON.parse(raw) : {
    callsPerMinute: { mean: 0, stdDev: 1, samples: 0 },
    errorRate: { mean: 0, stdDev: 0.1 },
    paramStats: {}
  }

  // Welford's online algorithm（增量计算均值/方差，不需要存历史数据）
  baseline.callsPerMinute = welfordUpdate(
    baseline.callsPerMinute,
    callsPerMin
  )
  baseline.errorRate = welfordUpdate(
    baseline.errorRate,
    isError ? 1 : 0
  )

  for (const [param, value] of Object.entries(numericParams)) {
    baseline.paramStats[param] = welfordUpdate(
      baseline.paramStats[param] ?? { mean: 0, stdDev: 1, samples: 0 },
      value
    )
  }

  baseline.lastUpdated = Date.now()
  await redis.set(key, JSON.stringify(baseline), 'EX', 86400 * 7) // 7天TTL
}

function welfordUpdate(
  stats: { mean: number; stdDev: number; samples: number },
  newValue: number
) {
  const n = (stats.samples ?? 0) + 1
  const delta = newValue - stats.mean
  const mean = stats.mean + delta / n
  const delta2 = newValue - mean
  const m2 = (stats.stdDev ** 2) * (n - 1) + delta * delta2
  return {
    mean,
    stdDev: n > 1 ? Math.sqrt(m2 / (n - 1)) : 1,
    samples: n
  }
}
```

**Welford 在线算法**：不存历史数据，O(1) 增量更新均值和标准差。

### 2. 异常评分引擎

```typescript
// anomaly-detector.ts

interface AnomalyReport {
  score: number          // 0-1，越高越异常
  reasons: string[]
  severity: 'low' | 'medium' | 'high' | 'critical'
}

export async function detectAnomaly(
  toolName: string,
  callsPerMin: number,
  isError: boolean,
  numericParams: Record<string, number>
): Promise<AnomalyReport> {
  const key = `baseline:tool:${toolName}`
  const raw = await redis.get(key)
  
  // 新工具，样本不足，跳过检测
  if (!raw) return { score: 0, reasons: [], severity: 'low' }
  
  const baseline: ToolBaseline = JSON.parse(raw)
  const reasons: string[] = []
  let maxZScore = 0

  // Z-Score 检测：超过 3σ 就是统计异常
  const freqZ = zScore(callsPerMin, baseline.callsPerMinute)
  if (freqZ > 3) {
    reasons.push(`调用频率异常: ${callsPerMin}/min（基线 ${baseline.callsPerMinute.mean.toFixed(1)} ± ${baseline.callsPerMinute.stdDev.toFixed(1)}）`)
    maxZScore = Math.max(maxZScore, freqZ)
  }

  const errZ = zScore(isError ? 1 : 0, baseline.errorRate)
  if (errZ > 2.5 && baseline.errorRate.samples > 10) {
    reasons.push(`错误率异常上升`)
    maxZScore = Math.max(maxZScore, errZ)
  }

  for (const [param, value] of Object.entries(numericParams)) {
    const stats = baseline.paramStats[param]
    if (!stats || stats.samples < 10) continue
    const pZ = zScore(value, stats)
    if (pZ > 4) {
      reasons.push(`参数 ${param} 异常: ${value}（基线 ${stats.mean.toFixed(2)} ± ${stats.stdDev.toFixed(2)}）`)
      maxZScore = Math.max(maxZScore, pZ)
    }
  }

  // 归一化评分
  const score = Math.min(1, maxZScore / 10)
  const severity = score > 0.8 ? 'critical'
    : score > 0.5 ? 'high'
    : score > 0.3 ? 'medium'
    : 'low'

  return { score, reasons, severity }
}

function zScore(
  value: number,
  stats: { mean: number; stdDev: number }
): number {
  if (stats.stdDev === 0) return 0
  return Math.abs(value - stats.mean) / stats.stdDev
}
```

### 3. 工具调用序列异常（N-gram）

```typescript
// sequence-detector.ts

// 记录 session 内的调用序列，检测罕见的工具组合
export async function checkSequenceAnomaly(
  sessionId: string,
  toolName: string
): Promise<boolean> {
  const seqKey = `seq:${sessionId}`
  
  // 追加当前工具到序列
  await redis.rpush(seqKey, toolName)
  await redis.expire(seqKey, 3600)
  
  const seq = await redis.lrange(seqKey, -3, -1) // 取最近3个
  if (seq.length < 2) return false
  
  const bigram = seq.slice(-2).join('→')
  const count = parseInt(await redis.get(`ngram:${bigram}`) ?? '0')
  const total = parseInt(await redis.get('ngram:total') ?? '1')
  
  // 出现概率 < 0.1% 视为异常序列
  const probability = count / total
  return probability < 0.001 && total > 100
}

// 后台异步更新 N-gram 统计
export async function recordSequence(tools: string[]) {
  const pipeline = redis.pipeline()
  for (let i = 0; i < tools.length - 1; i++) {
    const bigram = `${tools[i]}→${tools[i+1]}`
    pipeline.incr(`ngram:${bigram}`)
  }
  pipeline.incr('ngram:total')
  await pipeline.exec()
}
```

### 4. 中间件集成（OpenClaw / pi-mono）

```typescript
// anomaly-middleware.ts

export function anomalyDetectionMiddleware() {
  // 滑动窗口计数器
  const callWindows = new Map<string, number[]>()

  return async (toolName: string, params: any, next: () => Promise<any>) => {
    const now = Date.now()
    
    // 计算最近60秒调用频率
    const window = callWindows.get(toolName) ?? []
    const recentCalls = window.filter(t => now - t < 60_000)
    recentCalls.push(now)
    callWindows.set(toolName, recentCalls)
    const callsPerMin = recentCalls.length

    // 执行工具
    let isError = false
    let result: any
    try {
      result = await next()
    } catch (err) {
      isError = true
      throw err
    } finally {
      // 提取数值型参数
      const numericParams = extractNumericParams(params)

      // 异步检测（不阻塞工具执行）
      detectAnomaly(toolName, callsPerMin, isError, numericParams)
        .then(async report => {
          if (report.severity === 'critical' || report.severity === 'high') {
            console.error(`[ANOMALY] ${toolName}`, report)
            // 推送告警（可接 Telegram/PagerDuty）
            await sendAlert(toolName, report)
          }
          // 异步更新基线
          await updateBaseline(toolName, callsPerMin, isError, numericParams)
        })
        .catch(() => {}) // 检测失败不影响主流程
    }

    return result
  }
}

function extractNumericParams(params: any): Record<string, number> {
  const result: Record<string, number> = {}
  for (const [k, v] of Object.entries(params ?? {})) {
    if (typeof v === 'number') result[k] = v
    else if (typeof v === 'string' && !isNaN(Number(v))) result[k] = Number(v)
  }
  return result
}

async function sendAlert(toolName: string, report: AnomalyReport) {
  // 对接 Telegram 告警
  const msg = [
    `🚨 工具异常告警`,
    `工具: \`${toolName}\``,
    `严重度: ${report.severity.toUpperCase()}`,
    `评分: ${(report.score * 100).toFixed(0)}%`,
    `原因:`,
    ...report.reasons.map(r => `  • ${r}`)
  ].join('\n')
  
  // 调用 OpenClaw message tool 推送
  // await message({ action: 'send', target: 'alert-channel', message: msg })
  console.error(msg)
}
```

---

## Python 版本（简化）

```python
# anomaly_detector.py
import redis
import json
import math
from dataclasses import dataclass, field
from typing import Optional

r = redis.Redis()

@dataclass
class RunningStats:
    mean: float = 0.0
    m2: float = 0.0
    samples: int = 0
    
    def update(self, value: float):
        self.samples += 1
        delta = value - self.mean
        self.mean += delta / self.samples
        delta2 = value - self.mean
        self.m2 += delta * delta2
    
    @property
    def std_dev(self) -> float:
        if self.samples < 2:
            return 1.0
        return math.sqrt(self.m2 / (self.samples - 1))
    
    def z_score(self, value: float) -> float:
        if self.std_dev == 0:
            return 0.0
        return abs(value - self.mean) / self.std_dev

def detect_anomaly(tool_name: str, calls_per_min: float) -> Optional[str]:
    key = f"baseline:tool:{tool_name}"
    raw = r.get(key)
    if not raw:
        return None
    
    stats = RunningStats(**json.loads(raw))
    z = stats.z_score(calls_per_min)
    
    if z > 3:
        return f"⚠️ {tool_name} 调用频率异常 (z={z:.1f}σ)"
    return None
```

---

## OpenClaw 实战：Heartbeat 巡检

在 `HEARTBEAT.md` 加入定期基线巡检：

```markdown
## 工具异常巡检
每次 heartbeat 检查 Redis 中的异常记录：
- 查看过去1小时是否有 high/critical 级别告警
- 异常工具列表是否有持续上升趋势
- 基线样本是否充足（samples < 100 的工具跳过检测）
```

---

## 关键设计决策

| 决策 | 选择 | 理由 |
|------|------|------|
| 基线算法 | Welford 在线算法 | O(1) 内存，不需要存历史数据 |
| 异常判断 | Z-Score（3σ法则） | 统计意义明确，无需调参 |
| 执行方式 | 异步，不阻塞主流程 | 检测失败不能影响工具执行 |
| 存储 | Redis + TTL | 内存快，自动过期，不需要持久化 |
| 序列检测 | N-gram 频率统计 | 检测罕见工具组合，发现 Prompt 注入 |

---

## 什么时候用

✅ **适合场景：**
- 多用户生产环境，需要发现滥用行为
- 有外部触发的工具（Webhook 驱动），需要防异常流量
- 工具调用成本高（LLM API），需要及时发现循环调用

❌ **不适合场景：**
- 开发/测试环境（频繁变更，基线无意义）
- 样本量 < 100 的新工具（统计不稳定）
- 实时性要求极高且异常检测耗时敏感（用同步模式会加延迟）

---

## 总结

```
正常监控：工具报错了 → 告警
行为基线：工具行为异常了（即使没报错）→ 告警
```

三步走：
1. **积累基线**：Welford 算法在线更新，Redis 存储，无需历史数据
2. **实时检测**：Z-Score 判断频率/参数/错误率偏差，N-gram 检测异常序列
3. **异步告警**：不阻塞主流程，severity 分级，高危直接推 Telegram

> 越早建立基线，越早发现问题。从第一天上线就开始收集。

---

*Lesson 160 / Agent 开发课程*  
*下一课：待定*
