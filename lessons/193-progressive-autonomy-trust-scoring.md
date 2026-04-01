# 193 - Agent 渐进式自治与信任度量（Progressive Autonomy & Trust Scoring）

> Agent 上线第一天不能直接拿 root 权限乱跑——就像新员工入职不会第一天就签合同一样。  
> 今天讲怎么让 Agent **靠业绩赚取自治权**，用量化信任分动态解锁更高自主级别。

---

## 🎯 核心问题

大多数 Agent 系统是**静态权限**的：

```
SOUL.md → 固定规则 → 固定权限
```

问题显而易见：
- **过度保守**：每次写文件都要问人，用户烦死了
- **过度放权**：新部署的 Agent 没有任何 track record，却能直接删数据库
- **没有自愈**：Agent 犯了错，权限不变，下次继续犯

真实系统需要的是：**Agent 通过行为记录动态获得/失去自治权**。

---

## 🏗️ 架构设计

### 信任级别 × 操作类型矩阵

```
                READ   WRITE   DELETE   EXTERNAL   CRITICAL
SUPERVISED       ✅      ❓       ❌        ❌         ❌
ASSISTED         ✅      ✅       ❓        ❓         ❌
AUTONOMOUS       ✅      ✅       ✅        ✅         ❓
TRUSTED          ✅      ✅       ✅        ✅         ✅

✅ = 直接执行
❓ = 执行后上报 / 低风险自动执行
❌ = 必须等人工审批
```

### 信任分计算公式

```
TrustScore = base_score
           + Σ(success × weight)     # 成功加分
           - Σ(failure × penalty)    # 失败扣分
           × time_decay_factor       # 时间衰减
           × recency_boost           # 近期表现加权
```

---

## 💻 TypeScript 实现

```typescript
// trust-scoring.ts

import { createClient } from 'redis'

// ────────────────────────────────────────────────
// 1. 信任级别定义
// ────────────────────────────────────────────────
export type TrustLevel = 'SUPERVISED' | 'ASSISTED' | 'AUTONOMOUS' | 'TRUSTED'
export type OperationType = 'read' | 'write' | 'delete' | 'external' | 'critical'
export type AutonomyDecision = 'allow' | 'require_approval' | 'deny' | 'allow_and_report'

// 权限矩阵
const PERMISSION_MATRIX: Record<TrustLevel, Record<OperationType, AutonomyDecision>> = {
  SUPERVISED: {
    read:     'allow',
    write:    'require_approval',
    delete:   'deny',
    external: 'deny',
    critical: 'deny',
  },
  ASSISTED: {
    read:     'allow',
    write:    'allow',
    delete:   'require_approval',
    external: 'require_approval',
    critical: 'deny',
  },
  AUTONOMOUS: {
    read:     'allow',
    write:    'allow',
    delete:   'allow_and_report',
    external: 'allow',
    critical: 'require_approval',
  },
  TRUSTED: {
    read:     'allow',
    write:    'allow',
    delete:   'allow',
    external: 'allow',
    critical: 'allow_and_report',
  },
}

// 信任级别阈值
const TRUST_THRESHOLDS = {
  SUPERVISED:  0,    // 默认新 Agent
  ASSISTED:    30,   // 30分以上
  AUTONOMOUS:  60,   // 60分以上
  TRUSTED:     85,   // 85分以上
}

// ────────────────────────────────────────────────
// 2. 信任分管理器
// ────────────────────────────────────────────────
interface TaskRecord {
  id: string
  type: OperationType
  success: boolean
  impact: 'low' | 'medium' | 'high'    // 操作影响级别
  timestamp: number
  sessionId: string
}

const SCORE_WEIGHTS = {
  success: { low: 1, medium: 2, high: 5 },
  failure: { low: 3, medium: 8, high: 20 },   // 失败扣分比成功加分重
}

export class TrustScoreManager {
  private redis = createClient()
  private readonly KEY_PREFIX = 'agent:trust:'
  private readonly DECAY_DAYS = 30     // 30天内的记录有效
  private readonly MAX_SCORE = 100
  private readonly MIN_SCORE = 0

  async recordTask(agentId: string, record: TaskRecord): Promise<void> {
    const key = `${this.KEY_PREFIX}${agentId}:history`
    
    // 追加记录（Sorted Set，score = timestamp 方便按时间查询）
    await this.redis.zAdd(key, {
      score: record.timestamp,
      value: JSON.stringify(record),
    })
    
    // 清理 30 天前的记录
    const cutoff = Date.now() - this.DECAY_DAYS * 24 * 3600 * 1000
    await this.redis.zRemRangeByScore(key, '-inf', cutoff)
    
    // 实时更新信任分缓存
    const score = await this.calculateScore(agentId)
    await this.redis.setEx(
      `${this.KEY_PREFIX}${agentId}:score`,
      3600,  // 1小时缓存
      score.toString()
    )
  }

  async calculateScore(agentId: string): Promise<number> {
    const key = `${this.KEY_PREFIX}${agentId}:history`
    const now = Date.now()
    
    // 拿最近 90 天记录（窗口比衰减期宽，确保边缘数据正确衰减）
    const cutoff = now - 90 * 24 * 3600 * 1000
    const raw = await this.redis.zRangeByScore(key, cutoff, '+inf')
    const records: TaskRecord[] = raw.map(r => JSON.parse(r))

    if (records.length === 0) return 10  // 新 Agent 给个初始基础分

    let score = 10  // 基础分
    
    for (const record of records) {
      const ageDays = (now - record.timestamp) / (24 * 3600 * 1000)
      
      // 时间衰减：越旧的记录影响越小（指数衰减）
      const decayFactor = Math.exp(-ageDays / this.DECAY_DAYS)
      
      if (record.success) {
        const weight = SCORE_WEIGHTS.success[record.impact]
        score += weight * decayFactor
      } else {
        const penalty = SCORE_WEIGHTS.failure[record.impact]
        score -= penalty * decayFactor
      }
    }
    
    return Math.max(this.MIN_SCORE, Math.min(this.MAX_SCORE, Math.round(score)))
  }

  async getTrustLevel(agentId: string): Promise<TrustLevel> {
    // 先查缓存
    const cached = await this.redis.get(`${this.KEY_PREFIX}${agentId}:score`)
    const score = cached ? parseInt(cached) : await this.calculateScore(agentId)
    
    if (score >= TRUST_THRESHOLDS.TRUSTED)    return 'TRUSTED'
    if (score >= TRUST_THRESHOLDS.AUTONOMOUS) return 'AUTONOMOUS'
    if (score >= TRUST_THRESHOLDS.ASSISTED)   return 'ASSISTED'
    return 'SUPERVISED'
  }

  async getDecision(
    agentId: string,
    operation: OperationType
  ): Promise<{ decision: AutonomyDecision; level: TrustLevel; score: number }> {
    const level = await this.getTrustLevel(agentId)
    const score = parseInt(
      (await this.redis.get(`${this.KEY_PREFIX}${agentId}:score`)) || '10'
    )
    const decision = PERMISSION_MATRIX[level][operation]
    return { decision, level, score }
  }
}
```

---

## 🔌 工具中间件集成

```typescript
// trust-middleware.ts

import { TrustScoreManager } from './trust-scoring'
import { sendTelegramApproval } from './telegram'

const trustManager = new TrustScoreManager()

// 工具元数据：声明操作类型
interface ToolMeta {
  operationType: OperationType
  impact: 'low' | 'medium' | 'high'
  description: string
}

const TOOL_META: Record<string, ToolMeta> = {
  read_file:        { operationType: 'read',     impact: 'low',    description: '读取文件' },
  write_file:       { operationType: 'write',    impact: 'medium', description: '写入文件' },
  delete_file:      { operationType: 'delete',   impact: 'high',   description: '删除文件' },
  send_email:       { operationType: 'external', impact: 'high',   description: '发送邮件' },
  deploy_prod:      { operationType: 'critical', impact: 'high',   description: '生产部署' },
  query_db:         { operationType: 'read',     impact: 'low',    description: '查询数据库' },
  update_db:        { operationType: 'write',    impact: 'medium', description: '更新数据库' },
}

export function withTrustGuard(agentId: string) {
  return async function trustGuard<T>(
    toolName: string,
    toolFn: () => Promise<T>,
    params: Record<string, unknown>
  ): Promise<T> {
    const meta = TOOL_META[toolName]
    if (!meta) {
      // 未注册的工具默认当 write 处理
      console.warn(`[TrustGuard] Unknown tool: ${toolName}, defaulting to write policy`)
    }

    const opType = meta?.operationType ?? 'write'
    const impact = meta?.impact ?? 'medium'
    const { decision, level, score } = await trustManager.getDecision(agentId, opType)

    console.log(`[TrustGuard] ${toolName} | level=${level}(${score}) | decision=${decision}`)

    let result: T

    if (decision === 'deny') {
      // 直接拒绝，反馈给 LLM
      throw new Error(
        `[TrustGuard] 操作被拒绝：当前信任级别 ${level}(${score}分) 不允许执行 ${meta?.description ?? toolName}。` +
        `需要提升到 ${getRequiredLevel(opType)} 级别。请向用户说明情况并请求人工授权。`
      )
    }

    if (decision === 'require_approval') {
      // 挂起，等待人工审批（Telegram Inline Buttons）
      const approved = await requestApproval(agentId, toolName, params, level, score)
      if (!approved) {
        throw new Error(`[TrustGuard] 用户拒绝了操作：${meta?.description ?? toolName}`)
      }
    }

    // 执行工具
    const startTime = Date.now()
    try {
      result = await toolFn()
      const latencyMs = Date.now() - startTime

      // 记录成功
      await trustManager.recordTask(agentId, {
        id: `${toolName}-${Date.now()}`,
        type: opType,
        success: true,
        impact,
        timestamp: Date.now(),
        sessionId: agentId,
      })

      if (decision === 'allow_and_report') {
        // 执行完上报（不阻塞）
        reportExecution(agentId, toolName, params, true, latencyMs).catch(console.error)
      }

      return result
    } catch (err) {
      // 记录失败
      await trustManager.recordTask(agentId, {
        id: `${toolName}-${Date.now()}-fail`,
        type: opType,
        success: false,
        impact,
        timestamp: Date.now(),
        sessionId: agentId,
      })
      throw err
    }
  }
}

function getRequiredLevel(opType: OperationType): TrustLevel {
  const map: Record<OperationType, TrustLevel> = {
    read: 'SUPERVISED', write: 'ASSISTED',
    delete: 'AUTONOMOUS', external: 'AUTONOMOUS', critical: 'TRUSTED',
  }
  return map[opType]
}

async function requestApproval(
  agentId: string,
  toolName: string,
  params: Record<string, unknown>,
  level: TrustLevel,
  score: number
): Promise<boolean> {
  // 发 Telegram Inline Buttons 给管理员
  const message = 
    `🔐 *Agent 权限请求*\n\n` +
    `操作：\`${toolName}\`\n` +
    `参数：\`${JSON.stringify(params, null, 2).slice(0, 200)}\`\n` +
    `信任级别：${level} (${score}分)\n\n` +
    `是否批准？`

  return sendTelegramApproval(agentId, message, 60_000)  // 60秒超时
}

async function reportExecution(
  agentId: string, toolName: string,
  params: Record<string, unknown>,
  success: boolean, latencyMs: number
): Promise<void> {
  console.log(`[TrustReport] agent=${agentId} tool=${toolName} success=${success} latency=${latencyMs}ms`)
  // 实际项目中可以推送到监控系统
}
```

---

## 📊 信任分仪表板（查询接口）

```typescript
// trust-dashboard.ts

export async function getTrustReport(agentId: string) {
  const manager = new TrustScoreManager()
  const score = await manager.calculateScore(agentId)
  const level = await manager.getTrustLevel(agentId)

  // 离下一级别还差多少分？
  const nextLevel = getNextLevel(level)
  const threshold = nextLevel ? TRUST_THRESHOLDS[nextLevel] : null
  const gap = threshold ? threshold - score : 0

  return {
    agentId,
    score,
    level,
    nextLevel: nextLevel ?? 'MAX',
    pointsToNextLevel: Math.max(0, gap),
    permissions: PERMISSION_MATRIX[level],
    message: buildProgressMessage(level, score, nextLevel, gap),
  }
}

function getNextLevel(level: TrustLevel): TrustLevel | null {
  const ladder: TrustLevel[] = ['SUPERVISED', 'ASSISTED', 'AUTONOMOUS', 'TRUSTED']
  const idx = ladder.indexOf(level)
  return idx < ladder.length - 1 ? ladder[idx + 1] : null
}

function buildProgressMessage(
  level: TrustLevel, score: number,
  nextLevel: TrustLevel | null, gap: number
): string {
  const bars = '█'.repeat(Math.floor(score / 10)) + '░'.repeat(10 - Math.floor(score / 10))
  return nextLevel
    ? `[${bars}] ${score}/100 — 再积累 ${gap} 分升级到 ${nextLevel}`
    : `[${bars}] ${score}/100 — 最高信任级别！`
}
```

---

## 🐍 Python 版本

```python
# trust_scoring.py

import json
import math
import time
from dataclasses import dataclass
from enum import Enum
from typing import Optional
import redis

class TrustLevel(str, Enum):
    SUPERVISED = "SUPERVISED"
    ASSISTED   = "ASSISTED"
    AUTONOMOUS = "AUTONOMOUS"
    TRUSTED    = "TRUSTED"

class OperationType(str, Enum):
    READ     = "read"
    WRITE    = "write"
    DELETE   = "delete"
    EXTERNAL = "external"
    CRITICAL = "critical"

PERMISSION_MATRIX = {
    TrustLevel.SUPERVISED: {
        OperationType.READ: "allow", OperationType.WRITE: "require_approval",
        OperationType.DELETE: "deny", OperationType.EXTERNAL: "deny",
        OperationType.CRITICAL: "deny",
    },
    TrustLevel.ASSISTED: {
        OperationType.READ: "allow", OperationType.WRITE: "allow",
        OperationType.DELETE: "require_approval", OperationType.EXTERNAL: "require_approval",
        OperationType.CRITICAL: "deny",
    },
    TrustLevel.AUTONOMOUS: {
        OperationType.READ: "allow", OperationType.WRITE: "allow",
        OperationType.DELETE: "allow_and_report", OperationType.EXTERNAL: "allow",
        OperationType.CRITICAL: "require_approval",
    },
    TrustLevel.TRUSTED: {
        OperationType.READ: "allow", OperationType.WRITE: "allow",
        OperationType.DELETE: "allow", OperationType.EXTERNAL: "allow",
        OperationType.CRITICAL: "allow_and_report",
    },
}

TRUST_THRESHOLDS = {
    TrustLevel.SUPERVISED: 0,
    TrustLevel.ASSISTED:  30,
    TrustLevel.AUTONOMOUS: 60,
    TrustLevel.TRUSTED:   85,
}

@dataclass
class TaskRecord:
    id: str
    type: OperationType
    success: bool
    impact: str   # 'low' | 'medium' | 'high'
    timestamp: float
    session_id: str

class TrustScoreManager:
    DECAY_DAYS = 30
    SCORE_WEIGHTS = {
        "success": {"low": 1, "medium": 2, "high": 5},
        "failure": {"low": 3, "medium": 8, "high": 20},
    }

    def __init__(self):
        self.redis = redis.Redis(decode_responses=True)
        self.key_prefix = "agent:trust:"

    def record_task(self, agent_id: str, record: TaskRecord):
        key = f"{self.key_prefix}{agent_id}:history"
        self.redis.zadd(key, {json.dumps(record.__dict__): record.timestamp})
        
        # 清理过期记录
        cutoff = time.time() - self.DECAY_DAYS * 86400
        self.redis.zremrangebyscore(key, "-inf", cutoff)
        
        # 更新缓存
        score = self.calculate_score(agent_id)
        self.redis.setex(f"{self.key_prefix}{agent_id}:score", 3600, str(score))

    def calculate_score(self, agent_id: str) -> int:
        key = f"{self.key_prefix}{agent_id}:history"
        now = time.time()
        cutoff = now - 90 * 86400

        raw = self.redis.zrangebyscore(key, cutoff, "+inf")
        records = [TaskRecord(**json.loads(r)) for r in raw]

        if not records:
            return 10

        score = 10.0
        for record in records:
            age_days = (now - record.timestamp) / 86400
            decay = math.exp(-age_days / self.DECAY_DAYS)

            if record.success:
                weight = self.SCORE_WEIGHTS["success"][record.impact]
                score += weight * decay
            else:
                penalty = self.SCORE_WEIGHTS["failure"][record.impact]
                score -= penalty * decay

        return max(0, min(100, round(score)))

    def get_trust_level(self, agent_id: str) -> TrustLevel:
        cached = self.redis.get(f"{self.key_prefix}{agent_id}:score")
        score = int(cached) if cached else self.calculate_score(agent_id)

        if score >= TRUST_THRESHOLDS[TrustLevel.TRUSTED]:    return TrustLevel.TRUSTED
        if score >= TRUST_THRESHOLDS[TrustLevel.AUTONOMOUS]: return TrustLevel.AUTONOMOUS
        if score >= TRUST_THRESHOLDS[TrustLevel.ASSISTED]:   return TrustLevel.ASSISTED
        return TrustLevel.SUPERVISED

    def get_decision(self, agent_id: str, op: OperationType) -> dict:
        level = self.get_trust_level(agent_id)
        cached = self.redis.get(f"{self.key_prefix}{agent_id}:score")
        score = int(cached) if cached else 10
        decision = PERMISSION_MATRIX[level][op]
        return {"decision": decision, "level": level, "score": score}
```

---

## 🔗 OpenClaw 集成：SOUL.md + 信任中间件

在 OpenClaw 中，`SOUL.md` 定义了静态规则，信任中间件在运行时动态收窄/放宽这些规则：

```typescript
// openclaw-trust-integration.ts

// 在工具分发层注入信任守卫
export function wrapToolsWithTrust(tools: Record<string, Function>, agentId: string) {
  const guard = withTrustGuard(agentId)
  
  return new Proxy(tools, {
    get(target, toolName: string) {
      const original = target[toolName]
      if (typeof original !== 'function') return original

      return async (params: Record<string, unknown>) => {
        return guard(toolName, () => original(params), params)
      }
    }
  })
}

// Heartbeat 中检查信任分状态
async function heartbeatTrustCheck(agentId: string) {
  const report = await getTrustReport(agentId)
  
  if (report.score < 20) {
    // 低信任分告警
    await notify(`⚠️ Agent 信任分过低：${report.score}分 (${report.level})，请检查最近的失败记录`)
  }
  
  if (report.level !== previousLevel) {
    // 级别变化通知
    await notify(`🎖️ Agent 信任级别变更：${previousLevel} → ${report.level} (${report.score}分)`)
    previousLevel = report.level
  }
}
```

---

## 🎓 关键设计决策

| 设计点 | 决策 | 原因 |
|--------|------|------|
| **时间衰减** | 指数衰减，30天半衰期 | 旧行为不应永久影响当前权限 |
| **失败惩罚 > 成功奖励** | 失败扣3-20分，成功加1-5分 | 破坏比建设容易，风险非对称 |
| **级别阈值非线性** | 30/60/85 | 高级别需要更多证明 |
| **操作类型分级** | read/write/delete/external/critical | 与真实风险对应 |
| **allow_and_report** | 执行后异步上报 | 不阻塞高信任 Agent，但保留可审计性 |
| **缓存 TTL = 1h** | 信任分短期缓存 | 避免每次工具调用都重算，但保持动态 |

---

## 📌 总结

```
新 Agent (10分) → SUPERVISED (谨慎探索)
    ↓ 成功积累 +30
ASSISTED (60分) → 正常工作，高风险操作报批
    ↓ 继续表现 +30  
AUTONOMOUS (60分) → 独立执行，关键操作留记录
    ↓ 长期可靠 +25
TRUSTED (85分) → 完全自治，最高权限
    ↑ 失败时自动降级
```

**一句话总结：信任不是配置出来的，是赚来的。**  
Agent 写代码不靠猜，赚权限也不靠求——**靠业绩**。
