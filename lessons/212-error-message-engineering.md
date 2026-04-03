# 212 - Agent 错误消息工程化（Error Message Engineering）

> 工具返回的错误消息，决定了 LLM 能不能自愈。烂错误消息让 Agent 原地打转，好错误消息让 Agent 自动修复。

---

## 为什么错误消息工程化至关重要

大多数 Agent 的工具错误处理是这样的：

```typescript
// ❌ 烂错误消息
try {
  await db.query(sql);
} catch (e) {
  return { error: e.message }; // "syntax error at position 42"
}
```

LLM 看到 `"syntax error at position 42"` 能做什么？什么都做不了。它不知道：
- 是哪条 SQL 出错了？
- 出错的具体原因是什么？
- 应该怎么修？
- 能重试吗？

**结论：工具错误消息不是给开发者看的，是给 LLM 看的。**

---

## 错误分类三段式

每个错误必须回答三个问题：

| 维度 | 含义 | 示例 |
|------|------|------|
| **发生了什么** | 事实描述 | SQL syntax error near "FORM" |
| **为什么** | 根因 | 关键字 FROM 拼错了 |
| **怎么修** | 行动建议 | 把 "FORM" 改为 "FROM" |

```typescript
// ✅ 好错误消息的三段结构
interface StructuredError {
  // 可恢复性：LLM 是否应该重试
  recoverable: boolean;
  
  // 错误类型：方便路由处理策略  
  type: 'validation' | 'permission' | 'not_found' | 'rate_limit' | 'server_error' | 'user_error';
  
  // 面向 LLM 的人类可读消息
  message: string;
  
  // 具体的修复建议
  suggestion?: string;
  
  // 原始错误（调试用，不注入 LLM）
  raw?: unknown;
}
```

---

## 可恢复性分级

```typescript
enum RecoveryStrategy {
  // LLM 可以直接重试（参数修正后）
  RETRY_WITH_FIX = 'retry_with_fix',
  
  // 等一下再重试（限流/临时故障）
  RETRY_AFTER_WAIT = 'retry_after_wait',
  
  // 需要用户介入（权限/账户问题）
  ESCALATE_TO_USER = 'escalate_to_user',
  
  // 放弃这条路，换个策略
  ABORT_AND_REROUTE = 'abort_and_reroute',
}

function classifyError(error: unknown): RecoveryStrategy {
  if (error instanceof ValidationError) return RecoveryStrategy.RETRY_WITH_FIX;
  if (error instanceof RateLimitError) return RecoveryStrategy.RETRY_AFTER_WAIT;
  if (error instanceof PermissionError) return RecoveryStrategy.ESCALATE_TO_USER;
  if (error instanceof NotFoundError) return RecoveryStrategy.ABORT_AND_REROUTE;
  return RecoveryStrategy.ESCALATE_TO_USER; // 未知错误走人工
}
```

---

## 实战：错误消息模板系统

```typescript
// error-templates.ts

const ERROR_TEMPLATES = {
  // 数据库类
  sql_syntax: (detail: { sql: string; position: number; hint: string }) => ({
    recoverable: true,
    type: 'validation' as const,
    message: `SQL 语法错误：在位置 ${detail.position} 附近 "${detail.hint}"`,
    suggestion: `检查 SQL 语句：\n\`\`\`sql\n${detail.sql}\n\`\`\`\n常见问题：关键字拼写错误、缺少引号、括号不匹配`,
  }),

  // 文件类
  file_not_found: (detail: { path: string; searched: string[] }) => ({
    recoverable: false,
    type: 'not_found' as const,
    message: `文件不存在：${detail.path}`,
    suggestion: `尝试以下路径：\n${detail.searched.map(p => `- ${p}`).join('\n')}\n或使用 list_files 工具确认目录结构`,
  }),

  // API 类
  rate_limit: (detail: { retryAfter: number; limit: string }) => ({
    recoverable: true,
    type: 'rate_limit' as const,
    message: `API 限流：${detail.limit}，请 ${detail.retryAfter} 秒后重试`,
    suggestion: `等待 ${detail.retryAfter} 秒后可重试。如频繁触发，考虑减少并发请求`,
    retryAfterMs: detail.retryAfter * 1000,
  }),

  // 权限类
  permission_denied: (detail: { resource: string; required: string }) => ({
    recoverable: false,
    type: 'permission' as const,
    message: `权限不足：访问 "${detail.resource}" 需要 "${detail.required}" 权限`,
    suggestion: `此操作需要用户手动授权，请告知用户需要配置相应权限`,
  }),
};

// 使用
function buildSqlError(rawError: Error, sql: string): StructuredError {
  const match = rawError.message.match(/syntax error at or near "(.+?)" at position (\d+)/);
  if (match) {
    return ERROR_TEMPLATES.sql_syntax({
      sql,
      position: parseInt(match[2]),
      hint: match[1],
    });
  }
  // fallback
  return {
    recoverable: true,
    type: 'validation',
    message: `SQL 执行失败：${rawError.message}`,
    suggestion: '请检查 SQL 语法，或尝试简化查询',
  };
}
```

---

## 工具层错误转换中间件

```typescript
// tool-error-transformer.ts

type ToolFn<T> = (...args: any[]) => Promise<T>;

function withErrorTransform<T>(
  toolName: string,
  fn: ToolFn<T>,
  transformer: (error: unknown) => StructuredError
): ToolFn<T | StructuredError> {
  return async (...args) => {
    try {
      return await fn(...args);
    } catch (error) {
      const structured = transformer(error);
      
      // 记录原始错误（内部日志）
      console.error(`[${toolName}] raw error:`, error);
      
      // 返回结构化错误（给 LLM）
      return {
        success: false,
        ...structured,
        // 不包含 raw 字段，避免污染 LLM 上下文
      };
    }
  };
}

// 注册工具时包装
const execute_sql = withErrorTransform(
  'execute_sql',
  async ({ query }: { query: string }) => {
    const result = await db.query(query);
    return { success: true, rows: result.rows, rowCount: result.rowCount };
  },
  (error) => {
    if (error instanceof DatabaseError) {
      return buildSqlError(error, '');
    }
    return {
      recoverable: false,
      type: 'server_error',
      message: '数据库连接失败',
      suggestion: '请稍后重试，或检查数据库服务状态',
    };
  }
);
```

---

## Agent Loop 中的错误处理策略

```typescript
// agent-loop.ts（核心修改部分）

async function handleToolResult(
  toolName: string,
  result: ToolResult | StructuredError
): Promise<string> {
  // 检测是否是结构化错误
  if ('recoverable' in result && result.success === false) {
    return buildErrorContext(toolName, result as StructuredError);
  }
  return JSON.stringify(result);
}

function buildErrorContext(toolName: string, error: StructuredError): string {
  const lines = [
    `工具 "${toolName}" 执行失败`,
    `错误类型：${error.type}`,
    `可恢复：${error.recoverable ? '是（可以重试）' : '否（需要换策略）'}`,
    ``,
    `错误详情：${error.message}`,
  ];
  
  if (error.suggestion) {
    lines.push(``, `修复建议：`, error.suggestion);
  }
  
  if (error.type === 'rate_limit' && 'retryAfterMs' in error) {
    lines.push(``, `⏱️ 将在 ${(error as any).retryAfterMs / 1000} 秒后自动重试`);
  }
  
  return lines.join('\n');
}
```

---

## Python 版本

```python
# error_engineering.py

from dataclasses import dataclass, field
from typing import Optional, Literal
from enum import Enum
import functools

class ErrorType(str, Enum):
    VALIDATION = "validation"
    PERMISSION = "permission"
    NOT_FOUND = "not_found"
    RATE_LIMIT = "rate_limit"
    SERVER_ERROR = "server_error"
    USER_ERROR = "user_error"

@dataclass
class StructuredError:
    recoverable: bool
    type: ErrorType
    message: str
    suggestion: Optional[str] = None
    retry_after_ms: Optional[int] = None
    success: bool = field(default=False, init=False)


def with_error_transform(transformer):
    """工具错误转换装饰器"""
    def decorator(fn):
        @functools.wraps(fn)
        async def wrapper(*args, **kwargs):
            try:
                return await fn(*args, **kwargs)
            except Exception as error:
                structured = transformer(error)
                print(f"[{fn.__name__}] raw error: {error}")  # 内部日志
                return structured  # 返回给 LLM
        return wrapper
    return decorator


# 具体工具示例
def file_error_transformer(error: Exception) -> StructuredError:
    if isinstance(error, FileNotFoundError):
        path = str(error.filename) if hasattr(error, 'filename') else '未知路径'
        return StructuredError(
            recoverable=False,
            type=ErrorType.NOT_FOUND,
            message=f"文件不存在：{path}",
            suggestion=f"请使用 list_files 工具确认目录结构，或检查路径是否正确"
        )
    if isinstance(error, PermissionError):
        return StructuredError(
            recoverable=False,
            type=ErrorType.PERMISSION,
            message="文件权限不足",
            suggestion="此文件需要更高权限才能访问，请告知用户"
        )
    return StructuredError(
        recoverable=True,
        type=ErrorType.SERVER_ERROR,
        message=f"文件操作失败：{str(error)}",
        suggestion="请重试，如问题持续请检查磁盘空间"
    )


@with_error_transform(file_error_transformer)
async def read_file(path: str) -> dict:
    with open(path, 'r', encoding='utf-8') as f:
        content = f.read()
    return {"success": True, "content": content, "size": len(content)}
```

---

## OpenClaw 中的错误消息实践

OpenClaw 的工具返回值设计已经体现了这个原则。看 `exec` 工具的返回：

```
// ✅ exec 工具返回失败时
{
  "exit_code": 1,
  "stdout": "",
  "stderr": "npm ERR! code ENOENT\nnpm ERR! syscall open\nnpm ERR! path /project/package.json",
  // OpenClaw 在这里的设计就是：
  // - 包含 exit_code（可恢复性判断）
  // - 完整 stderr（根因）
  // - 不做截断（让 LLM 自己判断）
}
```

**改进建议**：在你自己的工具中，额外追加一个 `hint` 字段：

```typescript
// 在 exec 工具的错误处理包装里
if (stderr.includes('ENOENT') && stderr.includes('package.json')) {
  return {
    ...baseResult,
    hint: '项目根目录缺少 package.json，请先运行 npm init 或切换到正确目录',
  };
}
```

---

## 常见错误模式与建议模板

```typescript
// 常见错误自动建议映射
const ERROR_HINTS: Array<{ pattern: RegExp; hint: string }> = [
  {
    pattern: /Cannot read propert(y|ies) .+ of (undefined|null)/,
    hint: '变量未初始化，请检查数据是否存在再访问其属性',
  },
  {
    pattern: /ECONNREFUSED/,
    hint: '连接被拒绝，目标服务可能未启动或端口错误',
  },
  {
    pattern: /JWT.*expired/i,
    hint: 'Token 已过期，需要重新获取认证凭据',
  },
  {
    pattern: /duplicate key.*violates unique constraint/i,
    hint: '数据重复，该记录已存在，可以改用 upsert 操作',
  },
  {
    pattern: /too many connections/i,
    hint: '连接池已满，请等待现有连接释放或增大连接池上限',
  },
];

function autoHint(errorMessage: string): string | undefined {
  for (const { pattern, hint } of ERROR_HINTS) {
    if (pattern.test(errorMessage)) return hint;
  }
  return undefined;
}
```

---

## 错误消息设计 Checklist

**工具返回错误时，必须包含：**

- [ ] ✅ `success: false` 明确标记失败
- [ ] ✅ `recoverable` 字段告诉 LLM 是否可重试
- [ ] ✅ `message` 用人话解释发生了什么
- [ ] ✅ `suggestion` 给出具体的修复方向
- [ ] ❌ 不包含原始 stack trace（噪音，浪费 token）
- [ ] ❌ 不包含敏感信息（密码、密钥、内部 IP）
- [ ] ❌ 不使用纯数字错误码（LLM 不知道 1045 是什么）

---

## 与其他模式的关系

| 模式 | 关注点 |
|------|--------|
| **Tool Return Value Design（211）** | 成功结果的信封设计 |
| **Error Message Engineering（212）** | 失败结果的信封设计 |
| **Tool Hallucination Detection（前作）** | 检测 LLM 乱调工具 |
| **Output Validation & Self-Correction（前作）** | 验证工具执行结果 |
| **Graceful Degradation（28）** | 工具不可用时的降级策略 |

---

## 总结

> 错误消息是工具与 LLM 之间的沟通语言。
> 好的错误消息 = 发生了什么 + 为什么 + 怎么修。
> LLM 不是靠猜来修复错误的，它靠的是你给它的信息质量。

**核心设计原则：**
1. **错误消息面向 LLM 而非开发者**
2. **每个错误必须有可恢复性标记**
3. **修复建议要具体可操作**
4. **敏感信息绝不进入 LLM 上下文**
5. **用中间件统一处理，不散落在每个工具里**
