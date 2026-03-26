# 145 - Agent 智能错误分类与差异化重试（Intelligent Error Classification & Differential Retry）

> 不是所有错误都该重试，不是所有重试都该等一样久。

---

## 核心思路

大多数 Agent 错误处理只有一个策略：**等一会儿，再试一次**。这很原始。

生产环境中，工具调用会遇到各种错误：

| 错误类型 | 示例 | 正确策略 |
|---------|------|---------|
| **瞬时错误** | 网络超时、503 | 指数退避重试 |
| **限流错误** | 429 Too Many Requests | 读 Retry-After，精确等待 |
| **认证错误** | 401 Unauthorized | 先刷 Token，再重试一次 |
| **永久错误** | 400 Bad Request | 不重试，把错误描述给 LLM |
| **配额错误** | 额度不足 | 降级到便宜模型 或 终止 |
| **逻辑错误** | 工具内部异常 | 包装后让 LLM 自我纠错 |

**差异化重试**：根据错误分类，选择对应策略，而不是一刀切。

---

## 实现：TypeScript（pi-mono 风格）

```typescript
// error-classifier.ts

export enum ErrorClass {
  TRANSIENT = 'TRANSIENT',       // 重试 with backoff
  RATE_LIMIT = 'RATE_LIMIT',     // 等 Retry-After
  AUTH = 'AUTH',                  // 刷 token 后重试
  PERMANENT = 'PERMANENT',        // 不重试，告诉 LLM
  QUOTA = 'QUOTA',                // 降级或终止
  LOGIC = 'LOGIC',                // 包装给 LLM 自修复
}

export interface ClassifiedError {
  class: ErrorClass
  message: string
  retryAfterMs?: number           // RATE_LIMIT 专属
  shouldRefreshAuth?: boolean     // AUTH 专属
  llmHint?: string                // 给 LLM 的修复建议
}

export function classifyError(err: unknown): ClassifiedError {
  const e = err as any
  const status = e?.status ?? e?.statusCode ?? e?.response?.status

  // 429 限流
  if (status === 429) {
    const retryAfter = e?.headers?.['retry-after']
    const retryAfterMs = retryAfter
      ? parseInt(retryAfter) * 1000
      : 60_000  // 默认等 60s
    return {
      class: ErrorClass.RATE_LIMIT,
      message: `Rate limited. Retry after ${retryAfterMs}ms`,
      retryAfterMs,
    }
  }

  // 401/403 认证
  if (status === 401 || status === 403) {
    return {
      class: ErrorClass.AUTH,
      message: `Auth error (${status}). Will refresh token.`,
      shouldRefreshAuth: true,
    }
  }

  // 400 Bad Request / 422 Unprocessable
  if (status === 400 || status === 422) {
    return {
      class: ErrorClass.PERMANENT,
      message: e?.message ?? 'Bad request',
      llmHint: `The tool call failed with ${status}. Check parameter types and required fields.`,
    }
  }

  // 402 / 配额关键词
  if (status === 402 || /quota|billing|insufficient/i.test(e?.message ?? '')) {
    return {
      class: ErrorClass.QUOTA,
      message: `Quota/billing error: ${e?.message}`,
      llmHint: 'API quota exceeded. Consider switching to a cheaper model.',
    }
  }

  // 5xx / 网络超时 → 瞬时
  if (status >= 500 || /timeout|ECONNRESET|ETIMEDOUT|ENOTFOUND/i.test(e?.code ?? '')) {
    return {
      class: ErrorClass.TRANSIENT,
      message: `Transient error (${status ?? e?.code}): ${e?.message}`,
    }
  }

  // 默认：工具逻辑错误
  return {
    class: ErrorClass.LOGIC,
    message: e?.message ?? String(err),
    llmHint: `Tool threw an unexpected error: "${e?.message}". Try adjusting the parameters.`,
  }
}
```

---

## 重试执行器

```typescript
// retry-executor.ts
import { classifyError, ErrorClass } from './error-classifier'

interface RetryOptions {
  maxAttempts?: number
  baseDelayMs?: number
  maxDelayMs?: number
  refreshAuth?: () => Promise<void>
  onFallback?: () => Promise<unknown>  // QUOTA 降级
}

export async function withSmartRetry<T>(
  fn: () => Promise<T>,
  opts: RetryOptions = {}
): Promise<T> {
  const {
    maxAttempts = 4,
    baseDelayMs = 500,
    maxDelayMs = 30_000,
    refreshAuth,
    onFallback,
  } = opts

  let attempt = 0

  while (attempt < maxAttempts) {
    try {
      return await fn()
    } catch (err) {
      const classified = classifyError(err)
      attempt++

      console.log(`[retry] attempt=${attempt} class=${classified.class} msg=${classified.message}`)

      switch (classified.class) {
        case ErrorClass.PERMANENT:
        case ErrorClass.LOGIC:
          // 不重试，直接抛出（带 llmHint）
          throw Object.assign(new Error(classified.message), {
            llmHint: classified.llmHint,
            noRetry: true,
          })

        case ErrorClass.QUOTA:
          if (onFallback) {
            console.log('[retry] quota hit, trying fallback')
            return onFallback() as Promise<T>
          }
          throw new Error(`Quota exceeded: ${classified.message}`)

        case ErrorClass.AUTH:
          if (refreshAuth && attempt === 1) {
            console.log('[retry] refreshing auth token...')
            await refreshAuth()
            continue  // 立刻重试，不等待
          }
          throw new Error(`Auth failed after refresh: ${classified.message}`)

        case ErrorClass.RATE_LIMIT:
          const waitMs = classified.retryAfterMs ?? 60_000
          console.log(`[retry] rate limited, waiting ${waitMs}ms`)
          await sleep(Math.min(waitMs, maxDelayMs))
          break

        case ErrorClass.TRANSIENT:
          if (attempt >= maxAttempts) throw err
          const backoff = Math.min(baseDelayMs * 2 ** (attempt - 1), maxDelayMs)
          const jitter = Math.random() * backoff * 0.2
          console.log(`[retry] transient, backoff ${Math.round(backoff + jitter)}ms`)
          await sleep(backoff + jitter)
          break
      }
    }
  }

  throw new Error(`Max attempts (${maxAttempts}) exceeded`)
}

const sleep = (ms: number) => new Promise(r => setTimeout(r, ms))
```

---

## 与 Agent Tool Dispatcher 集成

```typescript
// tool-dispatcher.ts
import { withSmartRetry } from './retry-executor'

export async function dispatchTool(
  toolName: string,
  params: Record<string, unknown>,
  context: AgentContext
): Promise<ToolResult> {
  try {
    const result = await withSmartRetry(
      () => executeTool(toolName, params),
      {
        maxAttempts: 4,
        refreshAuth: () => context.auth.refresh(),
        onFallback: toolName === 'llm_call'
          ? () => executeTool('llm_call_cheap', params)  // 降级到便宜模型
          : undefined,
      }
    )
    return { success: true, data: result }

  } catch (err: any) {
    // PERMANENT / LOGIC 错误：包装 llmHint 返回给 LLM，让它自我纠错
    if (err.noRetry && err.llmHint) {
      return {
        success: false,
        error: err.message,
        hint: err.llmHint,   // LLM 下一轮会读到这个
      }
    }
    throw err
  }
}
```

---

## Python 版本（learn-claude-code 风格）

```python
# error_classifier.py
import re
import time
from enum import Enum
from dataclasses import dataclass, field
from typing import Optional, Callable, Any
import httpx

class ErrorClass(Enum):
    TRANSIENT  = "TRANSIENT"
    RATE_LIMIT = "RATE_LIMIT"
    AUTH       = "AUTH"
    PERMANENT  = "PERMANENT"
    QUOTA      = "QUOTA"
    LOGIC      = "LOGIC"

@dataclass
class ClassifiedError:
    cls: ErrorClass
    message: str
    retry_after_ms: Optional[int] = None
    should_refresh_auth: bool = False
    llm_hint: Optional[str] = None

def classify_error(err: Exception) -> ClassifiedError:
    status = None
    if isinstance(err, httpx.HTTPStatusError):
        status = err.response.status_code

    if status == 429:
        retry_after = err.response.headers.get("retry-after", "60")
        return ClassifiedError(
            cls=ErrorClass.RATE_LIMIT,
            message=f"Rate limited",
            retry_after_ms=int(retry_after) * 1000,
        )

    if status in (401, 403):
        return ClassifiedError(
            cls=ErrorClass.AUTH,
            message=f"Auth error {status}",
            should_refresh_auth=True,
        )

    if status in (400, 422):
        return ClassifiedError(
            cls=ErrorClass.PERMANENT,
            message=str(err),
            llm_hint=f"Tool call failed ({status}). Check parameter schema.",
        )

    if status == 402 or re.search(r"quota|billing", str(err), re.I):
        return ClassifiedError(
            cls=ErrorClass.QUOTA,
            message=str(err),
            llm_hint="API quota exceeded. Switch to a cheaper model.",
        )

    if (status and status >= 500) or re.search(r"timeout|connection", str(err), re.I):
        return ClassifiedError(cls=ErrorClass.TRANSIENT, message=str(err))

    return ClassifiedError(
        cls=ErrorClass.LOGIC,
        message=str(err),
        llm_hint=f'Unexpected tool error: "{err}". Try different parameters.',
    )


def with_smart_retry(
    fn: Callable[[], Any],
    max_attempts: int = 4,
    base_delay: float = 0.5,
    max_delay: float = 30.0,
    refresh_auth: Optional[Callable] = None,
    on_fallback: Optional[Callable] = None,
) -> Any:
    import random

    for attempt in range(1, max_attempts + 1):
        try:
            return fn()
        except Exception as err:
            c = classify_error(err)
            print(f"[retry] attempt={attempt} class={c.cls.value} msg={c.message}")

            if c.cls in (ErrorClass.PERMANENT, ErrorClass.LOGIC):
                raise RuntimeError(c.message) from err  # 附带 llm_hint

            if c.cls == ErrorClass.QUOTA:
                if on_fallback:
                    return on_fallback()
                raise

            if c.cls == ErrorClass.AUTH:
                if refresh_auth and attempt == 1:
                    refresh_auth()
                    continue
                raise

            if c.cls == ErrorClass.RATE_LIMIT:
                wait = min((c.retry_after_ms or 60_000) / 1000, max_delay)
                time.sleep(wait)

            elif c.cls == ErrorClass.TRANSIENT:
                if attempt >= max_attempts:
                    raise
                backoff = min(base_delay * (2 ** (attempt - 1)), max_delay)
                jitter = random.uniform(0, backoff * 0.2)
                time.sleep(backoff + jitter)

    raise RuntimeError(f"Max attempts ({max_attempts}) exceeded")
```

---

## 在 OpenClaw 中的应用

OpenClaw 的工具执行天然适合这套模式。在 `SKILL.md` 中调用外部 API 时：

```typescript
// 在 skill 中调用外部 API
const result = await withSmartRetry(
  () => fetch('https://api.example.com/data', {
    headers: { Authorization: `Bearer ${token}` }
  }).then(r => {
    if (!r.ok) throw Object.assign(new Error(r.statusText), {
      status: r.status,
      headers: Object.fromEntries(r.headers),
    })
    return r.json()
  }),
  {
    refreshAuth: async () => {
      token = await refreshAccessToken()
    }
  }
)
```

---

## 关键设计原则

1. **先分类，再决策** — 错误类型决定重试策略，不是凭感觉
2. **Retry-After 优先** — 429 一定读响应头，不要自己猜等多久
3. **Auth 刷新只试一次** — 避免无限刷 token 循环
4. **PERMANENT/LOGIC 错误给 LLM** — 让模型自我纠错，不要吞错误
5. **QUOTA 错误触发降级** — 切便宜模型，而不是死等

---

## 小结

```
错误发生
   ↓
classify_error() → 分类
   ↓
TRANSIENT  → 指数退避重试
RATE_LIMIT → 读 Retry-After 精确等待
AUTH       → 刷 Token，重试一次
PERMANENT  → 不重试，把 llmHint 给 LLM
QUOTA      → 降级到便宜模型
LOGIC      → 包装错误描述，让 LLM 自修复
```

这套模式让 Agent 面对真实 API 的各种脾气时，既不会无脑死循环，也不会过早放弃。

---

*下一课：待定*
