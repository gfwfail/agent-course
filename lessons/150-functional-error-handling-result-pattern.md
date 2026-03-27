# 150. Agent 函数式错误处理与 Result 模式（Functional Error Handling with Result/Either Pattern）

> 告别 try-catch 地狱，用类型安全的 Result 模式让工具错误变成可组合的数据——错误处理也能优雅

---

## 为什么 try-catch 不够好？

传统的 try-catch 有几个问题：

1. **隐式错误**：函数签名看不出会不会抛异常
2. **遗漏处理**：忘了 catch，程序崩溃
3. **难以组合**：链式调用时每步都要包 try-catch
4. **上下文丢失**：catch 到的 Error 往往不知道发生在哪一步

Agent 工具调用链中，这些问题会被放大——一个工具失败可能悄悄传播，最终 LLM 拿到的是乱七八糟的错误消息。

---

## Result 模式：错误变成值

```typescript
// Result<T, E> = Ok<T> | Err<E>
// 错误不再是异常，而是函数的返回值的一部分

type Ok<T> = { ok: true; value: T };
type Err<E> = { ok: false; error: E };
type Result<T, E = AgentError> = Ok<T> | Err<E>;

// 构造函数
const ok = <T>(value: T): Ok<T> => ({ ok: true, value });
const err = <E>(error: E): Err<E> => ({ ok: false, error });
```

调用方必须显式处理两种情况，编译器会帮你检查。

---

## Agent 错误类型体系

```typescript
// 结构化的 Agent 错误类型
type AgentErrorCode =
  | "TOOL_TIMEOUT"
  | "TOOL_RATE_LIMITED"
  | "TOOL_PERMISSION_DENIED"
  | "TOOL_INVALID_PARAMS"
  | "TOOL_EXTERNAL_ERROR"
  | "TOOL_NOT_FOUND"
  | "LLM_CONTEXT_OVERFLOW"
  | "LLM_CONTENT_FILTERED";

interface AgentError {
  code: AgentErrorCode;
  message: string;
  retryable: boolean;
  retryAfterMs?: number;
  llmHint?: string;      // 给 LLM 看的错误说明
  cause?: unknown;        // 原始错误
}

// 工厂函数
const AgentErrors = {
  timeout: (toolName: string, ms: number): AgentError => ({
    code: "TOOL_TIMEOUT",
    message: `Tool "${toolName}" timed out after ${ms}ms`,
    retryable: true,
    llmHint: `The tool ${toolName} took too long. Try with a simpler request or break it into smaller steps.`,
  }),

  rateLimited: (toolName: string, retryAfterMs: number): AgentError => ({
    code: "TOOL_RATE_LIMITED",
    message: `Tool "${toolName}" is rate limited`,
    retryable: true,
    retryAfterMs,
    llmHint: `Rate limited. Please wait ${retryAfterMs / 1000}s before retrying.`,
  }),

  permissionDenied: (toolName: string, reason: string): AgentError => ({
    code: "TOOL_PERMISSION_DENIED",
    message: `Tool "${toolName}" permission denied: ${reason}`,
    retryable: false,
    llmHint: `You don't have permission to use ${toolName}. ${reason}`,
  }),
};
```

---

## 工具包装器：把 throw 变成 Result

```typescript
// 把任意可能抛出的函数包装成返回 Result 的函数
async function wrapTool<T>(
  toolName: string,
  fn: () => Promise<T>,
  timeoutMs = 10_000
): Promise<Result<T>> {
  const timer = new Promise<never>((_, reject) =>
    setTimeout(() => reject(new Error("TIMEOUT")), timeoutMs)
  );

  try {
    const value = await Promise.race([fn(), timer]);
    return ok(value);
  } catch (e: unknown) {
    if (e instanceof Error) {
      if (e.message === "TIMEOUT") {
        return err(AgentErrors.timeout(toolName, timeoutMs));
      }
      if (e.message.includes("429")) {
        const retryAfter = parseInt(
          (e as any).headers?.["retry-after"] ?? "5"
        ) * 1000;
        return err(AgentErrors.rateLimited(toolName, retryAfter));
      }
      if (e.message.includes("403") || e.message.includes("permission")) {
        return err(AgentErrors.permissionDenied(toolName, e.message));
      }
    }
    return err({
      code: "TOOL_EXTERNAL_ERROR",
      message: `Tool "${toolName}" failed: ${String(e)}`,
      retryable: false,
      llmHint: `An unexpected error occurred in ${toolName}. The external service may be unavailable.`,
      cause: e,
    });
  }
}
```

---

## Result 组合：链式调用不再繁琐

```typescript
// map: 成功时变换值
function mapResult<T, U, E>(
  result: Result<T, E>,
  fn: (value: T) => U
): Result<U, E> {
  return result.ok ? ok(fn(result.value)) : result;
}

// flatMap / andThen: 成功时继续执行另一个可能失败的操作
async function andThen<T, U, E>(
  result: Result<T, E>,
  fn: (value: T) => Promise<Result<U, E>>
): Promise<Result<U, E>> {
  return result.ok ? fn(result.value) : result;
}

// mapError: 变换错误类型
function mapError<T, E, F>(
  result: Result<T, E>,
  fn: (error: E) => F
): Result<T, F> {
  return result.ok ? result : err(fn(result.error));
}

// 实际使用示例
async function fetchUserAndPosts(userId: string): Promise<Result<UserWithPosts>> {
  // 1. 获取用户
  const userResult = await wrapTool("get_user", () => apiGetUser(userId));
  if (!userResult.ok) return userResult;

  // 2. 获取帖子（依赖用户数据）
  const postsResult = await wrapTool("get_posts", () =>
    apiGetPosts(userResult.value.id)
  );
  if (!postsResult.ok) return postsResult;

  // 3. 组合结果
  return ok({ ...userResult.value, posts: postsResult.value });
}
```

---

## 与 Agent Loop 集成：Result → LLM 工具响应

```typescript
// OpenClaw / Claude 工具执行器中的集成
interface ToolExecutionResult {
  content: string;
  isError?: boolean;
}

function resultToToolResponse(result: Result<unknown>): ToolExecutionResult {
  if (result.ok) {
    return {
      content: JSON.stringify(result.value, null, 2),
    };
  }

  const error = result.error;
  return {
    content: JSON.stringify({
      error: error.code,
      message: error.message,
      hint: error.llmHint,
      retryable: error.retryable,
      ...(error.retryAfterMs && { retryAfterSeconds: error.retryAfterMs / 1000 }),
    }),
    isError: true,
  };
}

// 工具中间件：统一转换
class ResultAwareToolRegistry {
  private tools = new Map<string, (...args: any[]) => Promise<Result<any>>>();

  register(name: string, fn: (...args: any[]) => Promise<Result<any>>) {
    this.tools.set(name, fn);
  }

  async execute(name: string, params: unknown): Promise<ToolExecutionResult> {
    const tool = this.tools.get(name);
    if (!tool) {
      return {
        content: JSON.stringify({
          error: "TOOL_NOT_FOUND",
          hint: `Tool "${name}" does not exist. Available tools: ${[...this.tools.keys()].join(", ")}`,
        }),
        isError: true,
      };
    }

    const result = await tool(params);
    return resultToToolResponse(result);
  }
}
```

---

## 并行工具调用的 Result 聚合

```typescript
// 多个工具并行执行，收集所有结果（成功+失败）
async function runToolsParallel<T extends Record<string, () => Promise<Result<any>>>>(
  tools: T
): Promise<{ [K in keyof T]: Awaited<ReturnType<T[K]>> }> {
  const entries = Object.entries(tools);
  const results = await Promise.allSettled(
    entries.map(([_, fn]) => fn())
  );

  return Object.fromEntries(
    entries.map(([key, _], i) => {
      const settled = results[i];
      if (settled.status === "fulfilled") {
        return [key, settled.value];
      }
      return [key, err({
        code: "TOOL_EXTERNAL_ERROR" as const,
        message: String(settled.reason),
        retryable: false,
        llmHint: "Unexpected error during parallel execution",
      })];
    })
  ) as any;
}

// 使用示例
const results = await runToolsParallel({
  weather: () => wrapTool("weather", () => fetchWeather("Sydney")),
  news: () => wrapTool("news", () => fetchNews("tech")),
  stocks: () => wrapTool("stocks", () => fetchStocks(["AAPL"])),
});

// 每个结果独立处理，不因某个失败而整体崩溃
const summary = {
  weather: results.weather.ok ? results.weather.value : `unavailable: ${results.weather.error.llmHint}`,
  news: results.news.ok ? results.news.value : `unavailable: ${results.news.error.llmHint}`,
  stocks: results.stocks.ok ? results.stocks.value : `unavailable: ${results.stocks.error.llmHint}`,
};
```

---

## Python 版本：使用 dataclass 实现

```python
from dataclasses import dataclass
from typing import TypeVar, Generic, Callable, Awaitable
import asyncio

T = TypeVar('T')
E = TypeVar('E')

@dataclass
class Ok(Generic[T]):
    value: T
    ok: bool = True

@dataclass  
class Err(Generic[E]):
    error: E
    ok: bool = False

Result = Ok[T] | Err[E]

# 工具包装器
async def wrap_tool(
    tool_name: str,
    fn: Callable[[], Awaitable[T]],
    timeout_secs: float = 10.0
) -> Result:
    try:
        value = await asyncio.wait_for(fn(), timeout=timeout_secs)
        return Ok(value)
    except asyncio.TimeoutError:
        return Err({
            "code": "TOOL_TIMEOUT",
            "message": f"Tool '{tool_name}' timed out after {timeout_secs}s",
            "retryable": True,
            "llm_hint": f"The tool {tool_name} took too long. Try breaking it into smaller steps."
        })
    except Exception as e:
        return Err({
            "code": "TOOL_EXTERNAL_ERROR",
            "message": str(e),
            "retryable": False,
            "llm_hint": f"Unexpected error in {tool_name}: {str(e)}"
        })

# 链式操作
async def and_then(result: Result, fn: Callable) -> Result:
    if result.ok:
        return await fn(result.value)
    return result

# 使用示例
async def main():
    user_result = await wrap_tool("get_user", lambda: fetch_user("123"))
    
    if not user_result.ok:
        print(f"Error: {user_result.error['llm_hint']}")
        return
    
    posts_result = await wrap_tool(
        "get_posts",
        lambda: fetch_posts(user_result.value["id"])
    )
    
    match posts_result:
        case Ok(value=posts):
            print(f"Got {len(posts)} posts")
        case Err(error=e):
            print(f"Failed to get posts: {e['llm_hint']}")
```

---

## OpenClaw 实战：工具注册时强制 Result 签名

```typescript
// 在 OpenClaw skills 里，把工具统一改造成返回 Result
// skills/my-skill/tools.ts

import { ok, err, Result } from "./result";

export const searchTool = {
  name: "search_web",
  description: "Search the web for information",
  inputSchema: {
    type: "object",
    properties: {
      query: { type: "string", description: "Search query" },
    },
    required: ["query"],
  },
  // 工具实现返回 Result 而非直接 throw
  async execute(params: { query: string }): Promise<Result<SearchResult[]>> {
    if (!params.query.trim()) {
      return err({
        code: "TOOL_INVALID_PARAMS" as const,
        message: "Query cannot be empty",
        retryable: false,
        llmHint: "Please provide a non-empty search query.",
      });
    }

    return wrapTool("search_web", () => braveSearch(params.query), 8000);
  },
};
```

---

## 核心收益对比

| 维度 | try-catch | Result 模式 |
|------|-----------|-------------|
| 类型安全 | ❌ 运行时才知道 | ✅ 编译时检查 |
| 可组合性 | ❌ 嵌套地狱 | ✅ flatMap 链式 |
| 错误遗漏 | ❌ 容易忘 catch | ✅ 必须处理 |
| 并行错误 | ❌ Promise.all 全崩 | ✅ 独立收集 |
| LLM 提示 | ❌ 原始 Error | ✅ 结构化 llmHint |
| 可测试性 | ❌ 需要模拟异常 | ✅ 返回值即数据 |

---

## 关键要点

1. **错误是值，不是异常**：`Result<T, E>` 让错误变成可组合的数据
2. **显式优于隐式**：函数签名直接告诉你"我可能失败"
3. **llmHint 是桥梁**：错误结构里的 `llmHint` 让 LLM 能自主决策重试/降级
4. **并行不怕单点失败**：`Promise.allSettled` + Result 聚合，一个工具挂了不影响其他
5. **中间件天然适配**：工具注册表统一 `execute` 接口，一处转换，全局受益

---

*下一课预告：Agent 声明式管道与函数组合（Declarative Pipeline & Function Composition）*
