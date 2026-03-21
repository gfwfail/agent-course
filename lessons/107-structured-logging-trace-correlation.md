# 107 - 结构化日志与链路关联（Structured Logging & Trace Correlation）

> 当 Agent 出了问题，你第一件事是什么？看日志。  
> 如果日志是 `console.log("error:", e)`，那你完蛋了。  
> 结构化日志 + 链路关联，让调试从"大海捞针"变成"精确制导"。

---

## 为什么需要结构化日志

```
# ❌ 传统日志 —— 调试噩梦
[2026-03-21 16:30:01] ERROR tool failed
[2026-03-21 16:30:01] retrying...
[2026-03-21 16:30:02] done

# ✅ 结构化日志 —— 一眼定位
{"ts":"2026-03-21T16:30:01Z","level":"error","service":"agent","traceId":"abc123","spanId":"def456","tool":"web_search","userId":"u_789","error":"timeout after 5000ms","attempt":1,"duration_ms":5001}
{"ts":"2026-03-21T16:30:01Z","level":"info","traceId":"abc123","msg":"retrying tool","attempt":2,"backoff_ms":1000}
{"ts":"2026-03-21T16:30:02Z","level":"info","traceId":"abc123","msg":"tool success","attempt":2,"duration_ms":800}
```

结构化日志的核心价值：
- **机器可读**：JSON 格式直接送 ELK/Loki/CloudWatch
- **可过滤**：`jq '.[] | select(.traceId=="abc123")'`
- **可聚合**：统计 P99 工具耗时、错误率
- **关联性**：一个 traceId 串起所有日志

---

## 核心概念

### 关联 ID 体系

```
Session
  └── traceId (本次对话的唯一标识)
        ├── spanId (单个操作的标识)
        └── parentSpanId (父操作，形成树)

用户请求 → traceId: "trace_abc"
  ├── LLM 推理    spanId: "span_001"  parentSpanId: null
  ├── tool_call   spanId: "span_002"  parentSpanId: "span_001"
  │     └── sub-agent spanId: "span_003"  parentSpanId: "span_002"
  └── tool_call   spanId: "span_004"  parentSpanId: "span_001"
```

### 日志级别设计

```typescript
enum LogLevel {
  DEBUG = 0,   // 开发调试，生产关闭
  INFO  = 1,   // 正常业务流程
  WARN  = 2,   // 可恢复的异常
  ERROR = 3,   // 需要人工介入
  FATAL = 4,   // 服务崩溃
}
```

---

## 实现：Agent 结构化日志系统

### 1. 核心 Logger 设计

```typescript
// logger.ts

interface LogContext {
  traceId?: string;
  spanId?: string;
  parentSpanId?: string;
  sessionId?: string;
  userId?: string;
  tool?: string;
  model?: string;
  [key: string]: unknown;  // 扩展字段
}

interface LogEntry {
  ts: string;
  level: string;
  msg: string;
  service: string;
  version: string;
  duration_ms?: number;
  error?: string;
  stack?: string;
  [key: string]: unknown;
}

class StructuredLogger {
  private context: LogContext = {};
  private static instance: StructuredLogger;
  
  // 子 Logger 继承父 context，添加新字段
  child(additionalContext: LogContext): StructuredLogger {
    const child = new StructuredLogger();
    child.context = { ...this.context, ...additionalContext };
    return child;
  }

  private write(level: string, msg: string, extra?: Record<string, unknown>) {
    const entry: LogEntry = {
      ts: new Date().toISOString(),
      level,
      msg,
      service: process.env.SERVICE_NAME ?? "agent",
      version: process.env.SERVICE_VERSION ?? "unknown",
      ...this.context,
      ...extra,
    };
    
    // 生产环境 JSON，开发环境可读格式
    if (process.env.NODE_ENV === "production") {
      process.stdout.write(JSON.stringify(entry) + "\n");
    } else {
      const color = { debug: "\x1b[90m", info: "\x1b[36m", warn: "\x1b[33m", error: "\x1b[31m" }[level] ?? "";
      console.error(`${color}[${level.toUpperCase()}]\x1b[0m ${msg}`, 
        Object.keys(extra ?? {}).length ? extra : "");
    }
  }

  debug(msg: string, extra?: Record<string, unknown>) { this.write("debug", msg, extra); }
  info(msg: string, extra?: Record<string, unknown>)  { this.write("info",  msg, extra); }
  warn(msg: string, extra?: Record<string, unknown>)  { this.write("warn",  msg, extra); }
  
  error(msg: string, err?: Error, extra?: Record<string, unknown>) {
    this.write("error", msg, {
      error: err?.message,
      stack: err?.stack,
      ...extra,
    });
  }

  // 计时工具：自动记录耗时
  timer(operation: string): { done: (extra?: Record<string, unknown>) => void } {
    const start = Date.now();
    return {
      done: (extra = {}) => {
        this.info(`${operation} completed`, { 
          duration_ms: Date.now() - start,
          ...extra 
        });
      }
    };
  }
}

export const logger = new StructuredLogger();
```

### 2. Trace Context 传播

```typescript
// trace-context.ts
import { AsyncLocalStorage } from "node:async_hooks";
import { randomUUID } from "node:crypto";

interface TraceContext {
  traceId: string;
  spanId: string;
  parentSpanId?: string;
}

// AsyncLocalStorage 是关键：自动在异步调用链中传播 context
const traceStorage = new AsyncLocalStorage<TraceContext>();

export function runWithTrace<T>(
  fn: () => Promise<T>,
  parentCtx?: TraceContext
): Promise<T> {
  const ctx: TraceContext = {
    traceId: parentCtx?.traceId ?? `trace_${randomUUID().slice(0, 8)}`,
    spanId: `span_${randomUUID().slice(0, 8)}`,
    parentSpanId: parentCtx?.spanId,
  };
  return traceStorage.run(ctx, fn);
}

export function getCurrentTrace(): TraceContext | undefined {
  return traceStorage.getStore();
}

export function getTraceLogger() {
  const ctx = getCurrentTrace();
  return logger.child(ctx ?? {});
}

// 使用示例
async function handleUserRequest(userId: string, message: string) {
  return runWithTrace(async () => {
    const log = getTraceLogger().child({ userId });
    log.info("request received", { message_length: message.length });
    
    // 工具调用自动继承 traceId
    const result = await callTool("web_search", { query: message });
    log.info("request completed", { result_length: result.length });
    return result;
  });
}
```

### 3. 工具调用日志中间件

```typescript
// tool-logging-middleware.ts

type ToolFn = (params: unknown) => Promise<unknown>;

function withToolLogging(toolName: string, fn: ToolFn): ToolFn {
  return async (params: unknown) => {
    const log = getTraceLogger().child({ tool: toolName });
    const timer = log.timer(`tool:${toolName}`);
    
    log.debug("tool invoked", { params: sanitizeParams(params) });
    
    try {
      const result = await fn(params);
      timer.done({ status: "success", result_size: JSON.stringify(result).length });
      return result;
    } catch (err) {
      log.error("tool failed", err as Error, { 
        params: sanitizeParams(params),
        status: "error"
      });
      throw err;
    }
  };
}

// 敏感参数脱敏
function sanitizeParams(params: unknown): unknown {
  const SENSITIVE_KEYS = ["password", "token", "api_key", "secret"];
  
  if (typeof params !== "object" || params === null) return params;
  
  return Object.fromEntries(
    Object.entries(params as Record<string, unknown>).map(([k, v]) => [
      k,
      SENSITIVE_KEYS.some(s => k.toLowerCase().includes(s)) ? "[REDACTED]" : v,
    ])
  );
}

// 批量注册所有工具
const TOOLS = { web_search, code_exec, file_read, ... };
const loggedTools = Object.fromEntries(
  Object.entries(TOOLS).map(([name, fn]) => [name, withToolLogging(name, fn)])
);
```

### 4. Sub-agent 跨进程 Trace 传播

```typescript
// 父 Agent 发送子任务时，注入 trace context
async function spawnSubAgent(task: string) {
  const ctx = getCurrentTrace();
  
  // 通过环境变量或任务 metadata 传递 trace context
  return sessionsSpawn({
    task,
    env: {
      PARENT_TRACE_ID: ctx?.traceId ?? "",
      PARENT_SPAN_ID:  ctx?.spanId ?? "",
    },
  });
}

// 子 Agent 启动时恢复 trace context
function initFromParent() {
  const parentTraceId = process.env.PARENT_TRACE_ID;
  const parentSpanId  = process.env.PARENT_SPAN_ID;
  
  if (parentTraceId) {
    return runWithTrace(mainLoop, {
      traceId: parentTraceId,         // 继承父 traceId！
      spanId: `span_${randomUUID().slice(0, 8)}`,
      parentSpanId: parentSpanId,
    });
  }
  return runWithTrace(mainLoop);
}
```

---

## OpenClaw 中的实际应用

OpenClaw 的 session 日志系统天然是结构化的：
```
日志流: session_key → traceId
每条工具调用: spanId
子 Agent: 继承 traceId，生成新 spanId
```

查询某次对话的全部日志：
```bash
# 从文件日志中提取某个 traceId 的完整链路
cat /var/log/openclaw/agent.jsonl | \
  jq 'select(.traceId == "trace_abc123")' | \
  jq -s 'sort_by(.ts)'
```

---

## learn-claude-code 中的模式

Claude Code 的工具日志设计关键点：

```python
# Python 版：使用 contextvars 实现异步 trace 传播
import contextvars
import uuid
import json
import time
from datetime import datetime, timezone

trace_id_var = contextvars.ContextVar("trace_id", default=None)
span_id_var  = contextvars.ContextVar("span_id",  default=None)

class AgentLogger:
    def __init__(self, **base_ctx):
        self.base_ctx = base_ctx
    
    def _emit(self, level: str, msg: str, **extra):
        entry = {
            "ts":       datetime.now(timezone.utc).isoformat(),
            "level":    level,
            "msg":      msg,
            "traceId":  trace_id_var.get(),
            "spanId":   span_id_var.get(),
            **self.base_ctx,
            **extra,
        }
        print(json.dumps(entry, ensure_ascii=False))
    
    def info(self, msg, **extra):  self._emit("info",  msg, **extra)
    def error(self, msg, **extra): self._emit("error", msg, **extra)
    def warn(self, msg, **extra):  self._emit("warn",  msg, **extra)

# 工具调用装饰器
def log_tool_call(tool_name: str):
    def decorator(fn):
        async def wrapper(*args, **kwargs):
            log = AgentLogger(tool=tool_name)
            start = time.monotonic()
            try:
                result = await fn(*args, **kwargs)
                log.info("tool success", duration_ms=int((time.monotonic()-start)*1000))
                return result
            except Exception as e:
                log.error("tool failed", error=str(e), 
                          duration_ms=int((time.monotonic()-start)*1000))
                raise
        return wrapper
    return decorator

@log_tool_call("web_search")
async def web_search(query: str) -> str:
    ...
```

---

## 日志聚合查询实战

### 用 jq 分析本地日志

```bash
# 统计各工具调用次数
cat agent.jsonl | jq -r 'select(.tool) | .tool' | sort | uniq -c | sort -rn

# 找出耗时超过 3 秒的工具调用
cat agent.jsonl | jq 'select(.duration_ms > 3000) | {ts, tool, duration_ms, traceId}'

# 统计错误率（按工具）
cat agent.jsonl | jq -s '
  group_by(.tool)[] |
  {
    tool: .[0].tool,
    total: length,
    errors: [.[] | select(.level=="error")] | length,
    error_rate: ([.[] | select(.level=="error")] | length) / length * 100
  }
'

# 还原某次请求的完整调用链
TRACE_ID="trace_abc123"
cat agent.jsonl | jq --arg t "$TRACE_ID" 'select(.traceId==$t)' | jq -s 'sort_by(.ts)'
```

### Loki 查询（生产环境）

```logql
# 查某用户最近1小时的错误
{service="agent"} | json | level="error" | userId="u_123" | __error__=""

# P99 工具耗时
quantile_over_time(0.99, {service="agent"} | json | tool="web_search" | duration_ms [1h])

# 错误率趋势
sum(rate({service="agent"} | json | level="error" [5m])) 
  / sum(rate({service="agent"} | json [5m]))
```

---

## 最佳实践总结

| 原则 | 做法 |
|------|------|
| **一定要有 traceId** | 每次用户请求生成，贯穿整个调用链 |
| **AsyncLocalStorage** | 不要手动传参，用 ALS 自动传播 |
| **结构优先** | 所有字段都是 JSON key，不要拼接字符串 |
| **不要记录 PII** | 敏感字段 `[REDACTED]`，参考第93课 |
| **计时要精确** | `Date.now()` 打点，`duration_ms` 字段 |
| **子 Agent 继承 traceId** | 通过 env 或 metadata 传递 |
| **日志采样** | DEBUG 级别生产环境按 1% 采样 |

---

## 与其他课程的关系

- **第87课 OpenTelemetry**：OTel 专注 Span/Metric 导出到 Jaeger/Prometheus，本课专注日志格式和查询
- **第93课 PII 脱敏**：日志写入前的数据清洗
- **第58课 Observability**：整体可观测性三大支柱：Logs + Metrics + Traces

---

## 一句话总结

> **结构化日志 = 给未来的自己留的侦探线索。**  
> traceId 是案件编号，spanId 是每个线索，AsyncLocalStorage 是自动跟踪的侦探助手。  
> 出了问题，`jq 'select(.traceId=="xxx")'`，案情一目了然。
