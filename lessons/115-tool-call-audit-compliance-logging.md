# 115 - Agent 工具调用审计与合规日志（Tool Call Audit & Compliance Logging）

> 结构化日志告诉你发生了什么，审计日志告诉你谁做了什么、为什么做、能不能做——这是合规的命根子。

---

## 为什么审计日志不同于普通日志？

Lesson 107 讲了结构化日志与链路追踪，聚焦**调试**和**性能分析**。审计日志的目标完全不同：

| 维度 | 结构化日志 | 审计日志 |
|------|-----------|---------|
| 目的 | 调试/监控 | 合规/追责 |
| 写入者 | 系统自动 | 强制埋点 |
| 可变性 | 可覆盖 | 追加写、不可篡改 |
| 保留期 | 短期（7-90天） | 长期（1-7年） |
| 查询场景 | 实时排查 | 事后取证 |
| 敏感字段 | 部分脱敏 | 完整记录意图 |

合规场景（GDPR、SOC2、金融审计）要求你能回答：
- **谁**发起了这个工具调用？（用户/Agent/Cron Job？）
- **什么时候**？（精确时间戳，UTC）
- **调用了什么**？（工具名、参数摘要）
- **为什么**？（调用意图/上下文）
- **结果**？（成功/失败/被拒绝）
- **有没有授权**？（权限检查结果）

---

## 核心设计原则

### 1. Append-Only 追加写

审计记录一旦写入，不允许修改或删除（物理删除走合规申请流程）。

### 2. 意图记录（Intent Capture）

不只记录"调了什么参数"，还记录"LLM 为什么调"——从 assistant message 中提取意图描述。

### 3. 权限上下文

每条审计记录附带调用时的权限快照：用户角色、工具授权状态、是否绕过了某些检查。

### 4. 防篡改哈希链

每条记录包含上一条记录的哈希，形成链式结构，任何中间记录被篡改都能被检测到。

---

## 代码实现

### TypeScript：审计日志核心

```typescript
// audit/audit-logger.ts
import { createHash } from 'crypto';
import { appendFileSync } from 'fs';

interface AuditEntry {
  id: string;
  timestamp: string;          // ISO 8601 UTC
  sessionId: string;
  userId?: string;
  agentId: string;
  action: 'tool_call' | 'tool_result' | 'permission_denied' | 'session_start' | 'session_end';
  toolName?: string;
  paramsSummary?: string;     // 脱敏后的参数摘要，不记录原始敏感值
  intent?: string;            // LLM 的调用意图
  outcome: 'success' | 'failure' | 'denied';
  durationMs?: number;
  errorCode?: string;
  permissions: {
    granted: string[];
    checked: string[];
  };
  prevHash: string;           // 前一条记录的哈希（防篡改链）
  hash: string;               // 本条记录的哈希
}

class AuditLogger {
  private lastHash = '0000000000000000'; // 创世哈希
  private logPath: string;

  constructor(logPath: string) {
    this.logPath = logPath;
  }

  private computeHash(entry: Omit<AuditEntry, 'hash'>): string {
    const content = JSON.stringify(entry);
    return createHash('sha256').update(content).digest('hex').slice(0, 16);
  }

  private sanitizeParams(toolName: string, params: Record<string, unknown>): string {
    // 敏感字段用 [REDACTED] 替换，只保留结构摘要
    const SENSITIVE_KEYS = ['password', 'token', 'secret', 'api_key', 'credit_card'];
    const sanitized = Object.fromEntries(
      Object.entries(params).map(([k, v]) => [
        k,
        SENSITIVE_KEYS.some(s => k.toLowerCase().includes(s)) ? '[REDACTED]' : v
      ])
    );
    // 截断长字符串
    return JSON.stringify(sanitized, null, 0).slice(0, 500);
  }

  log(event: {
    sessionId: string;
    userId?: string;
    agentId: string;
    action: AuditEntry['action'];
    toolName?: string;
    params?: Record<string, unknown>;
    intent?: string;
    outcome: AuditEntry['outcome'];
    durationMs?: number;
    errorCode?: string;
    permissions?: { granted: string[]; checked: string[] };
  }): void {
    const partial: Omit<AuditEntry, 'hash'> = {
      id: `audit-${Date.now()}-${Math.random().toString(36).slice(2, 8)}`,
      timestamp: new Date().toISOString(),
      sessionId: event.sessionId,
      userId: event.userId,
      agentId: event.agentId,
      action: event.action,
      toolName: event.toolName,
      paramsSummary: event.params
        ? this.sanitizeParams(event.toolName ?? '', event.params)
        : undefined,
      intent: event.intent,
      outcome: event.outcome,
      durationMs: event.durationMs,
      errorCode: event.errorCode,
      permissions: event.permissions ?? { granted: [], checked: [] },
      prevHash: this.lastHash,
    };

    const hash = this.computeHash(partial);
    const entry: AuditEntry = { ...partial, hash };

    // Append-Only 写入
    appendFileSync(this.logPath, JSON.stringify(entry) + '\n', 'utf8');
    this.lastHash = hash;
  }
}

export const auditLogger = new AuditLogger('/var/log/agent-audit.jsonl');
```

### 工具调用中间件：自动注入审计

```typescript
// audit/audit-middleware.ts
import { auditLogger } from './audit-logger';

type ToolFn = (params: Record<string, unknown>) => Promise<unknown>;

/**
 * 包装任意工具函数，自动记录审计日志
 */
export function withAudit(
  toolName: string,
  fn: ToolFn,
  context: {
    sessionId: string;
    userId?: string;
    agentId: string;
    requiredPermissions?: string[];
  }
): ToolFn {
  return async (params) => {
    const start = Date.now();
    
    // 权限检查（从 context 获取当前会话权限）
    const userPermissions = await getSessionPermissions(context.sessionId);
    const required = context.requiredPermissions ?? [];
    const denied = required.filter(p => !userPermissions.includes(p));

    if (denied.length > 0) {
      auditLogger.log({
        sessionId: context.sessionId,
        userId: context.userId,
        agentId: context.agentId,
        action: 'permission_denied',
        toolName,
        params,
        outcome: 'denied',
        permissions: {
          granted: userPermissions,
          checked: required,
        },
      });
      throw new Error(`Permission denied: missing [${denied.join(', ')}]`);
    }

    try {
      const result = await fn(params);
      const durationMs = Date.now() - start;

      auditLogger.log({
        sessionId: context.sessionId,
        userId: context.userId,
        agentId: context.agentId,
        action: 'tool_result',
        toolName,
        params,
        outcome: 'success',
        durationMs,
        permissions: {
          granted: userPermissions,
          checked: required,
        },
      });

      return result;
    } catch (err: unknown) {
      const durationMs = Date.now() - start;
      const error = err instanceof Error ? err : new Error(String(err));

      auditLogger.log({
        sessionId: context.sessionId,
        userId: context.userId,
        agentId: context.agentId,
        action: 'tool_result',
        toolName,
        params,
        outcome: 'failure',
        durationMs,
        errorCode: error.message,
        permissions: {
          granted: userPermissions,
          checked: required,
        },
      });

      throw err;
    }
  };
}

async function getSessionPermissions(sessionId: string): Promise<string[]> {
  // 实际场景从 Redis/DB 查询会话权限
  return ['read', 'write', 'tool:web_search'];
}
```

### 从 LLM 消息中提取意图

```typescript
// audit/intent-extractor.ts

/**
 * 在 Agent Loop 中，每次 LLM 返回 tool_use block 时，
 * 同时从 assistant message 的 text 部分提取意图。
 */
export function extractIntent(assistantMessage: {
  content: Array<{ type: string; text?: string; id?: string; name?: string }>;
  toolUseId: string;
}): string {
  // 找到 tool_use 对应的文字说明
  const textBlocks = assistantMessage.content
    .filter(b => b.type === 'text' && b.text)
    .map(b => b.text!);

  if (textBlocks.length === 0) return '(no intent text)';

  // 取最后一段文字作为意图说明（LLM 通常在 tool_use 前解释原因）
  const fullText = textBlocks.join(' ');
  // 截取最后 200 字符
  return fullText.slice(-200).trim();
}

// 在 Agent Loop 里的用法：
async function agentLoop(messages: Message[]) {
  const response = await anthropic.messages.create({ ... });

  for (const block of response.content) {
    if (block.type === 'tool_use') {
      const intent = extractIntent({
        content: response.content,
        toolUseId: block.id,
      });

      // 记录 tool_call 审计（调用前）
      auditLogger.log({
        sessionId: currentSessionId,
        agentId: 'main-agent',
        action: 'tool_call',
        toolName: block.name,
        params: block.input as Record<string, unknown>,
        intent,
        outcome: 'success', // 调用前先记录意图
      });

      // 执行工具...
    }
  }
}
```

### 防篡改链验证

```typescript
// audit/verify-chain.ts
import { createHash } from 'crypto';
import { createReadStream } from 'fs';
import { createInterface } from 'readline';

interface AuditEntry {
  prevHash: string;
  hash: string;
  [key: string]: unknown;
}

export async function verifyAuditChain(logPath: string): Promise<{
  valid: boolean;
  totalRecords: number;
  firstTamperedId?: string;
}> {
  const rl = createInterface({ input: createReadStream(logPath) });
  let expectedPrevHash = '0000000000000000';
  let lineNum = 0;

  for await (const line of rl) {
    if (!line.trim()) continue;
    lineNum++;

    const entry: AuditEntry = JSON.parse(line);

    // 验证 prevHash
    if (entry.prevHash !== expectedPrevHash) {
      return {
        valid: false,
        totalRecords: lineNum,
        firstTamperedId: entry.id as string,
      };
    }

    // 重新计算哈希验证
    const { hash, ...rest } = entry;
    const expectedHash = createHash('sha256')
      .update(JSON.stringify(rest))
      .digest('hex')
      .slice(0, 16);

    if (hash !== expectedHash) {
      return {
        valid: false,
        totalRecords: lineNum,
        firstTamperedId: entry.id as string,
      };
    }

    expectedPrevHash = hash;
  }

  return { valid: true, totalRecords: lineNum };
}

// 使用示例
const result = await verifyAuditChain('/var/log/agent-audit.jsonl');
console.log(result);
// { valid: true, totalRecords: 8742 }
```

---

## Python 实现：合规报告生成

```python
# audit/compliance_report.py
import json
from datetime import datetime, timedelta
from collections import defaultdict
from pathlib import Path

def generate_compliance_report(
    log_path: str,
    start_time: datetime,
    end_time: datetime,
) -> dict:
    """生成指定时间段的合规报告"""
    entries = []

    with open(log_path) as f:
        for line in f:
            if not line.strip():
                continue
            entry = json.loads(line)
            ts = datetime.fromisoformat(entry['timestamp'].replace('Z', '+00:00'))
            if start_time <= ts <= end_time:
                entries.append(entry)

    # 统计维度
    by_user = defaultdict(list)
    by_tool = defaultdict(int)
    permission_denials = []
    failures = []

    for e in entries:
        if e.get('userId'):
            by_user[e['userId']].append(e)
        if e.get('toolName'):
            by_tool[e['toolName']] += 1
        if e['outcome'] == 'denied':
            permission_denials.append(e)
        if e['outcome'] == 'failure':
            failures.append(e)

    return {
        'period': {
            'start': start_time.isoformat(),
            'end': end_time.isoformat(),
        },
        'summary': {
            'total_events': len(entries),
            'unique_users': len(by_user),
            'permission_denials': len(permission_denials),
            'tool_failures': len(failures),
        },
        'top_tools': sorted(
            [{'tool': k, 'calls': v} for k, v in by_tool.items()],
            key=lambda x: -x['calls']
        )[:10],
        'denial_breakdown': [
            {
                'userId': e.get('userId'),
                'tool': e.get('toolName'),
                'timestamp': e['timestamp'],
                'missing_permissions': [
                    p for p in e['permissions']['checked']
                    if p not in e['permissions']['granted']
                ],
            }
            for e in permission_denials
        ],
    }

# 生成最近24小时报告
report = generate_compliance_report(
    '/var/log/agent-audit.jsonl',
    start_time=datetime.utcnow() - timedelta(hours=24),
    end_time=datetime.utcnow(),
)
print(json.dumps(report, indent=2, ensure_ascii=False))
```

---

## OpenClaw 中的审计实践

OpenClaw 本身的工具策略管道（Lesson 42）是注入审计钩子的天然位置：

```typescript
// openclaw 工具拦截点（伪代码，展示思路）
const auditedTools = Object.fromEntries(
  Object.entries(tools).map(([name, fn]) => [
    name,
    withAudit(name, fn, {
      sessionId: session.key,
      userId: session.userId,
      agentId: 'openclaw-main',
      requiredPermissions: TOOL_PERMISSIONS[name] ?? [],
    }),
  ])
);
```

在 **pi-mono** 中，Tool Dispatch 层（`packages/core/src/tools/dispatcher.ts`）同样适合在这里插入审计中间件，而不是在每个工具内部重复埋点。

---

## 存储选型建议

| 场景 | 推荐存储 | 理由 |
|------|---------|------|
| 开发/小规模 | JSONL 文件 | 零依赖，grep 友好 |
| 生产/中规模 | PostgreSQL (append-only table) | SQL 查询，行级权限 |
| 企业合规 | AWS CloudTrail / Datadog Audit | 托管不可变存储 |
| 高吞吐 | Kafka → S3 | 流式写入，长期冷存储 |

**关键约束：**
- 写入账号只有 `INSERT` 权限，无 `UPDATE/DELETE`
- 启用数据库审计（记录谁动了审计表）
- 定期导出到冷存储（S3 Object Lock / Glacier）

---

## 合规保留策略

```typescript
// 不同数据类型的保留期
const RETENTION_POLICY = {
  'tool_call':         '90d',   // 普通工具调用
  'permission_denied': '1y',    // 权限拒绝
  'session_start':     '2y',    // 会话记录
  'data_access':       '7y',    // 涉及用户数据的访问（GDPR）
  'financial':         '7y',    // 财务相关操作
} as const;

// 过期归档（不是删除！）
async function archiveExpiredRecords() {
  const cutoff = new Date(Date.now() - 90 * 24 * 60 * 60 * 1000);
  
  // 查询过期的普通记录
  const expired = await db.query(
    `SELECT * FROM audit_log WHERE action = 'tool_call' AND timestamp < $1`,
    [cutoff]
  );

  // 写入冷存储（S3）
  await uploadToS3(`archive/audit-${cutoff.toISOString()}.jsonl`, expired);

  // 从热存储删除（保留索引记录）
  await db.query(
    `UPDATE audit_log SET archived = true WHERE id = ANY($1)`,
    [expired.map(e => e.id)]
  );
}
```

---

## 关键要点

1. **审计 ≠ 日志**：审计关注意图、授权、追责；日志关注调试、性能
2. **Append-Only**：审计记录不可修改——用数据库权限和存储策略强制保证
3. **哈希链**：防篡改的低成本方案，任何中间记录被改都会导致链断裂
4. **意图捕获**：记录 LLM 的文字解释，而不只是参数——这是事后取证的关键
5. **敏感字段处理**：审计记录本身也需要脱敏，参数摘要而非完整原文
6. **工具中间件**：在 Dispatcher 层统一注入，不要在每个工具内重复

---

## 小结

| 审计日志组件 | 作用 |
|-------------|------|
| Append-Only 写入 | 防止事后删改 |
| 哈希链 | 检测任意位置的篡改 |
| 意图提取 | 记录"为什么调用" |
| 权限快照 | 记录"有没有权限" |
| 参数脱敏 | 审计记录本身的安全 |
| 合规报告 | 将原始记录转为可交付文件 |

> **一句话记忆**：审计日志是给律师、审计师、法院看的——不是给你自己 debug 用的。设计的时候想想：如果明天被审计，这条记录能不能自证清白？
