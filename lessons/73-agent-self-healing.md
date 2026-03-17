# 73. Agent Self-Healing（Agent 自愈机制）

> 让 Agent 在遇到错误时自动诊断、修复并恢复执行

## 为什么需要自愈？

生产环境中，Agent 会遇到各种意外：
- API 临时不可用
- 网络抖动
- 配置过期（token 失效、URL 变更）
- 依赖服务升级导致接口变化
- 资源耗尽（磁盘满、内存不足）

传统做法是报错退出，等人来修。但一个成熟的 Agent 应该能：

1. **检测** - 识别问题类型
2. **诊断** - 分析根因
3. **修复** - 尝试自动修复
4. **恢复** - 从断点继续执行
5. **学习** - 记录经验避免重复

## 核心架构

```
┌─────────────────────────────────────────────────────┐
│                   Agent Loop                         │
│  ┌─────────────────────────────────────────────┐    │
│  │              Tool Execution                  │    │
│  │  ┌─────────┐    ┌─────────┐    ┌─────────┐  │    │
│  │  │ Execute │───▶│ Detect  │───▶│ Classify│  │    │
│  │  └─────────┘    │  Error  │    │  Error  │  │    │
│  │       ▲         └─────────┘    └────┬────┘  │    │
│  │       │                             │       │    │
│  │       │         ┌─────────┐    ┌────▼────┐  │    │
│  │       └─────────│ Recover │◀───│ Heal    │  │    │
│  │                 └─────────┘    └─────────┘  │    │
│  └─────────────────────────────────────────────┘    │
│                        │                             │
│                   ┌────▼────┐                        │
│                   │ Journal │  (记录修复经验)         │
│                   └─────────┘                        │
└─────────────────────────────────────────────────────┘
```

## 实现一：错误分类器

第一步是精确分类错误，不同类型需要不同的修复策略：

```typescript
// 错误分类枚举
enum ErrorCategory {
  TRANSIENT = 'transient',      // 临时性错误，重试即可
  RECOVERABLE = 'recoverable',  // 可恢复，需要特定修复动作
  TERMINAL = 'terminal',        // 终态错误，无法自动修复
  UNKNOWN = 'unknown'           // 未知错误，需要人工介入
}

interface ClassifiedError {
  category: ErrorCategory
  code: string
  message: string
  suggestedAction?: HealingAction
  context?: Record<string, unknown>
}

// 错误分类器
class ErrorClassifier {
  private patterns: Map<RegExp, ErrorCategory> = new Map([
    // 临时性错误 - 重试
    [/ECONNRESET|ETIMEDOUT|ENOTFOUND/i, ErrorCategory.TRANSIENT],
    [/rate.?limit|429|too many requests/i, ErrorCategory.TRANSIENT],
    [/503|service unavailable/i, ErrorCategory.TRANSIENT],
    
    // 可恢复错误 - 需要修复动作
    [/401|unauthorized|token.?expired/i, ErrorCategory.RECOVERABLE],
    [/ENOSPC|no space left/i, ErrorCategory.RECOVERABLE],
    [/ENOMEM|out of memory/i, ErrorCategory.RECOVERABLE],
    [/certificate.?expired/i, ErrorCategory.RECOVERABLE],
    
    // 终态错误 - 无法自动修复
    [/403|forbidden|permission denied/i, ErrorCategory.TERMINAL],
    [/404|not found/i, ErrorCategory.TERMINAL],
    [/invalid.?api.?key/i, ErrorCategory.TERMINAL],
  ])
  
  classify(error: Error): ClassifiedError {
    const message = error.message || String(error)
    
    for (const [pattern, category] of this.patterns) {
      if (pattern.test(message)) {
        return {
          category,
          code: this.extractCode(error),
          message,
          suggestedAction: this.suggestAction(category, message)
        }
      }
    }
    
    return {
      category: ErrorCategory.UNKNOWN,
      code: 'UNKNOWN',
      message
    }
  }
  
  private suggestAction(category: ErrorCategory, message: string): HealingAction | undefined {
    if (category === ErrorCategory.TRANSIENT) {
      return { type: 'retry', params: { maxAttempts: 3, backoffMs: 1000 } }
    }
    
    if (/token.?expired|401/i.test(message)) {
      return { type: 'refresh_credentials' }
    }
    
    if (/ENOSPC/i.test(message)) {
      return { type: 'cleanup_disk' }
    }
    
    return undefined
  }
}
```

## 实现二：自愈动作注册表

定义各种自愈动作，Agent 可以根据错误类型自动选择：

```typescript
interface HealingAction {
  type: string
  params?: Record<string, unknown>
}

interface HealingResult {
  success: boolean
  action: HealingAction
  details?: string
}

// 自愈动作注册表
class HealingRegistry {
  private healers: Map<string, (params?: any) => Promise<HealingResult>> = new Map()
  
  constructor() {
    this.registerBuiltinHealers()
  }
  
  private registerBuiltinHealers() {
    // 重试修复器
    this.healers.set('retry', async (params) => {
      // 重试逻辑由调用方处理，这里只返回指示
      return {
        success: true,
        action: { type: 'retry', params },
        details: `Will retry with backoff: ${params?.backoffMs}ms`
      }
    })
    
    // 凭证刷新修复器
    this.healers.set('refresh_credentials', async () => {
      try {
        // 尝试刷新 token
        const newToken = await this.refreshAuthToken()
        return {
          success: true,
          action: { type: 'refresh_credentials' },
          details: 'Token refreshed successfully'
        }
      } catch (e) {
        return {
          success: false,
          action: { type: 'refresh_credentials' },
          details: `Failed to refresh: ${e}`
        }
      }
    })
    
    // 磁盘清理修复器
    this.healers.set('cleanup_disk', async () => {
      try {
        // 清理临时文件、日志等
        const freedMB = await this.cleanupTempFiles()
        return {
          success: freedMB > 100, // 至少清理 100MB
          action: { type: 'cleanup_disk' },
          details: `Freed ${freedMB}MB`
        }
      } catch (e) {
        return {
          success: false,
          action: { type: 'cleanup_disk' },
          details: `Cleanup failed: ${e}`
        }
      }
    })
    
    // 服务重连修复器
    this.healers.set('reconnect_service', async (params) => {
      const { serviceName, endpoint } = params || {}
      try {
        await this.reconnect(serviceName, endpoint)
        return {
          success: true,
          action: { type: 'reconnect_service', params },
          details: `Reconnected to ${serviceName}`
        }
      } catch (e) {
        return {
          success: false,
          action: { type: 'reconnect_service', params },
          details: `Reconnection failed: ${e}`
        }
      }
    })
  }
  
  async heal(action: HealingAction): Promise<HealingResult> {
    const healer = this.healers.get(action.type)
    if (!healer) {
      return {
        success: false,
        action,
        details: `No healer registered for: ${action.type}`
      }
    }
    return healer(action.params)
  }
  
  // 允许注册自定义修复器
  register(type: string, healer: (params?: any) => Promise<HealingResult>) {
    this.healers.set(type, healer)
  }
}
```

## 实现三：自愈工具执行器

将自愈能力集成到工具执行流程中：

```typescript
class SelfHealingToolExecutor {
  private classifier = new ErrorClassifier()
  private registry = new HealingRegistry()
  private journal = new HealingJournal()
  
  async execute<T>(
    toolName: string,
    fn: () => Promise<T>,
    options: { maxHealAttempts?: number; checkpoint?: () => void } = {}
  ): Promise<T> {
    const { maxHealAttempts = 3, checkpoint } = options
    let attempts = 0
    let lastError: Error | null = null
    
    while (attempts < maxHealAttempts) {
      try {
        const result = await fn()
        
        // 成功后保存检查点
        if (checkpoint) checkpoint()
        
        // 如果之前有修复过，记录成功经验
        if (attempts > 0 && lastError) {
          this.journal.recordSuccess(toolName, lastError, attempts)
        }
        
        return result
        
      } catch (error) {
        attempts++
        lastError = error as Error
        
        // 1. 分类错误
        const classified = this.classifier.classify(lastError)
        console.log(`[SelfHealing] Error classified: ${classified.category}`)
        
        // 2. 终态错误，直接抛出
        if (classified.category === ErrorCategory.TERMINAL) {
          this.journal.recordFailure(toolName, lastError, 'terminal')
          throw lastError
        }
        
        // 3. 未知错误，尝试让 LLM 诊断
        if (classified.category === ErrorCategory.UNKNOWN) {
          const llmDiagnosis = await this.llmDiagnose(toolName, lastError)
          if (llmDiagnosis.canHeal) {
            classified.suggestedAction = llmDiagnosis.action
          } else {
            this.journal.recordFailure(toolName, lastError, 'unknown')
            throw lastError
          }
        }
        
        // 4. 尝试自愈
        if (classified.suggestedAction) {
          console.log(`[SelfHealing] Attempting: ${classified.suggestedAction.type}`)
          const healResult = await this.registry.heal(classified.suggestedAction)
          
          if (healResult.success) {
            console.log(`[SelfHealing] Healed: ${healResult.details}`)
            
            // 特殊处理：重试需要等待
            if (classified.suggestedAction.type === 'retry') {
              const backoff = classified.suggestedAction.params?.backoffMs || 1000
              await this.sleep(backoff * attempts) // 指数退避
            }
            
            continue // 重试执行
          }
        }
        
        // 5. 自愈失败，记录并抛出
        this.journal.recordFailure(toolName, lastError, 'heal_failed')
        throw lastError
      }
    }
    
    // 超过最大尝试次数
    this.journal.recordFailure(toolName, lastError!, 'max_attempts')
    throw lastError!
  }
  
  // 让 LLM 诊断未知错误
  private async llmDiagnose(
    toolName: string, 
    error: Error
  ): Promise<{ canHeal: boolean; action?: HealingAction }> {
    const prompt = `
Tool "${toolName}" failed with error:
${error.message}
${error.stack}

Available healing actions: retry, refresh_credentials, cleanup_disk, reconnect_service

Can this error be automatically healed? If yes, which action?
Respond in JSON: { "canHeal": boolean, "action": { "type": string, "params": object } | null }
`
    
    // 调用 LLM 获取诊断结果
    const response = await this.llm.complete(prompt, { responseFormat: 'json' })
    return JSON.parse(response)
  }
}
```

## 实现四：修复经验日志

记录修复经验，用于分析和改进：

```typescript
interface HealingRecord {
  timestamp: Date
  toolName: string
  error: string
  errorCategory: ErrorCategory
  healingAction?: HealingAction
  outcome: 'success' | 'terminal' | 'unknown' | 'heal_failed' | 'max_attempts'
  attempts: number
}

class HealingJournal {
  private records: HealingRecord[] = []
  private persistPath: string
  
  constructor(persistPath = './healing-journal.json') {
    this.persistPath = persistPath
    this.load()
  }
  
  recordSuccess(toolName: string, error: Error, attempts: number) {
    this.addRecord({
      timestamp: new Date(),
      toolName,
      error: error.message,
      errorCategory: ErrorCategory.RECOVERABLE,
      outcome: 'success',
      attempts
    })
  }
  
  recordFailure(toolName: string, error: Error, reason: string) {
    this.addRecord({
      timestamp: new Date(),
      toolName,
      error: error.message,
      errorCategory: ErrorCategory.UNKNOWN,
      outcome: reason as any,
      attempts: 0
    })
  }
  
  // 分析修复成功率
  getSuccessRate(toolName?: string): number {
    const filtered = toolName 
      ? this.records.filter(r => r.toolName === toolName)
      : this.records
    
    if (filtered.length === 0) return 0
    
    const successes = filtered.filter(r => r.outcome === 'success').length
    return successes / filtered.length
  }
  
  // 获取常见失败模式
  getFailurePatterns(): Map<string, number> {
    const patterns = new Map<string, number>()
    
    for (const record of this.records) {
      if (record.outcome !== 'success') {
        const key = `${record.toolName}:${record.outcome}`
        patterns.set(key, (patterns.get(key) || 0) + 1)
      }
    }
    
    return patterns
  }
  
  // 生成改进建议
  generateInsights(): string[] {
    const insights: string[] = []
    const patterns = this.getFailurePatterns()
    
    for (const [pattern, count] of patterns) {
      if (count > 5) {
        const [tool, reason] = pattern.split(':')
        insights.push(
          `Tool "${tool}" has ${count} ${reason} failures. Consider adding specific healer.`
        )
      }
    }
    
    return insights
  }
}
```

## 实现五：断点续传

对于长时间运行的任务，支持从断点恢复：

```typescript
interface Checkpoint {
  taskId: string
  step: number
  state: Record<string, unknown>
  timestamp: Date
}

class CheckpointManager {
  private checkpoints: Map<string, Checkpoint> = new Map()
  
  save(taskId: string, step: number, state: Record<string, unknown>) {
    this.checkpoints.set(taskId, {
      taskId,
      step,
      state,
      timestamp: new Date()
    })
    
    // 持久化到磁盘
    this.persist(taskId)
  }
  
  restore(taskId: string): Checkpoint | null {
    return this.checkpoints.get(taskId) || this.loadFromDisk(taskId)
  }
  
  clear(taskId: string) {
    this.checkpoints.delete(taskId)
    this.removeFromDisk(taskId)
  }
}

// 使用示例：带断点的批量处理
async function processWithCheckpoint(items: Item[]) {
  const taskId = `batch-${Date.now()}`
  const checkpointMgr = new CheckpointManager()
  const executor = new SelfHealingToolExecutor()
  
  // 尝试从断点恢复
  const checkpoint = checkpointMgr.restore(taskId)
  let startIndex = checkpoint?.step || 0
  let results = (checkpoint?.state?.results as any[]) || []
  
  if (checkpoint) {
    console.log(`Resuming from step ${startIndex}`)
  }
  
  for (let i = startIndex; i < items.length; i++) {
    const item = items[i]
    
    const result = await executor.execute(
      'processItem',
      () => processItem(item),
      {
        maxHealAttempts: 3,
        checkpoint: () => {
          // 每成功处理一项，保存检查点
          checkpointMgr.save(taskId, i + 1, { results })
        }
      }
    )
    
    results.push(result)
  }
  
  // 全部完成，清理检查点
  checkpointMgr.clear(taskId)
  return results
}
```

## OpenClaw 中的自愈实践

OpenClaw 在多个层面实现了自愈：

```typescript
// 1. Gateway 级别的自愈
class Gateway {
  private async executeWithHealing(handler: () => Promise<void>) {
    try {
      await handler()
    } catch (error) {
      // 自动重连 Telegram
      if (error.code === 'ETELEGRAM_DISCONNECTED') {
        await this.telegram.reconnect()
        await handler() // 重试
      }
      
      // 自动刷新 OAuth token
      if (error.code === 'GOOGLE_AUTH_EXPIRED') {
        await this.googleAuth.refresh()
        await handler()
      }
    }
  }
}

// 2. 工具级别的自愈
const webFetchTool = {
  name: 'web_fetch',
  execute: async (url: string) => {
    return withSelfHealing(async () => {
      const response = await fetch(url)
      if (!response.ok) throw new Error(`HTTP ${response.status}`)
      return response.text()
    }, {
      // 自动处理常见错误
      healers: {
        'HTTP 429': async () => { await sleep(5000); return true },
        'ECONNRESET': async () => true, // 自动重试
        'HTTP 503': async () => { await sleep(2000); return true }
      }
    })
  }
}

// 3. Session 级别的自愈
class Session {
  async run() {
    while (true) {
      try {
        await this.processMessages()
      } catch (error) {
        // Session 崩溃后自动恢复
        console.error('Session error, recovering...', error)
        await this.loadLastCheckpoint()
        // 通知用户
        await this.notify('Session recovered from error')
      }
    }
  }
}
```

## 最佳实践

### 1. 分层自愈

```
┌─────────────────────────────────────┐
│  Application Layer                  │  ← 业务级自愈（重试业务逻辑）
├─────────────────────────────────────┤
│  Agent Layer                        │  ← 会话级自愈（恢复对话状态）
├─────────────────────────────────────┤
│  Tool Layer                         │  ← 工具级自愈（重试API调用）
├─────────────────────────────────────┤
│  Infrastructure Layer               │  ← 基础设施自愈（重连服务）
└─────────────────────────────────────┘
```

### 2. 自愈边界

不是所有错误都应该自愈：

```typescript
// ✅ 应该自愈
- 网络临时故障
- 服务暂时不可用
- Token 过期（可刷新）
- 资源临时不足

// ❌ 不应该自愈
- 权限不足（需要人工授权）
- 配置错误（需要人工修复）
- 数据校验失败（业务逻辑问题）
- API Key 无效（需要人工更换）
```

### 3. 自愈超时

避免无限自愈：

```typescript
const HEALING_CONFIG = {
  maxAttempts: 3,           // 最多尝试 3 次
  maxDurationMs: 60_000,    // 最多花 60 秒
  backoffMultiplier: 2,     // 指数退避
  alertThreshold: 2         // 2 次失败后告警
}
```

### 4. 自愈监控

```typescript
// 记录自愈指标
metrics.increment('healing.attempt', { tool, action })
metrics.increment('healing.success', { tool, action })
metrics.increment('healing.failure', { tool, action, reason })

// 告警规则
if (healingFailureRate > 0.5) {
  alert('High healing failure rate', { tool, rate: healingFailureRate })
}
```

## 总结

Agent 自愈机制的核心要点：

| 组件 | 职责 |
|-----|-----|
| **ErrorClassifier** | 精确分类错误类型 |
| **HealingRegistry** | 注册和执行修复动作 |
| **CheckpointManager** | 支持断点续传 |
| **HealingJournal** | 记录经验，持续改进 |
| **LLM Diagnosis** | 处理未知错误 |

自愈不是万能的，但它能让 Agent 更可靠：
- **减少人工干预** - 80% 的临时性错误自动恢复
- **提高可用性** - 从"遇错即停"到"遇错自愈"
- **积累经验** - 通过日志分析不断改进

---

下节预告：**Agent Capability Discovery（能力发现）** - 让 Agent 动态发现和协商可用能力
