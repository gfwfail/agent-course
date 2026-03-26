# 143 - Agent 工具调用上下文传播（Tool Call Context Propagation）

> **核心思路：** Agent 处理一个请求时，userId、tenantId、traceId、auth token、feature flags 这些"环境元数据"需要在整个工具调用链中保持可访问——但你不希望每个工具函数都显式接收这些参数。**上下文传播**就是让这些数据像"空气"一样流动，工具只管干活，框架负责搬运上下文。

---

## 为什么需要上下文传播？

没有上下文传播时的痛点：

```typescript
// ❌ 每个工具都要显式传 userId、traceId...噩梦
async function searchDatabase(query: string, userId: string, traceId: string, tenantId: string) { ... }
async function callExternalAPI(url: string, userId: string, traceId: string, authToken: string) { ... }
async function writeLog(msg: string, userId: string, traceId: string, level: string) { ... }
```

有了上下文传播：

```typescript
// ✅ 工具只关心业务参数，上下文自动获取
async function searchDatabase(query: string) {
  const { userId, traceId, tenantId } = AgentContext.current();
  // ...
}
```

**好处：**
| 好处 | 说明 |
|------|------|
| 工具接口简洁 | LLM schema 只含业务参数，不被运维字段污染 |
| 自动日志关联 | 所有日志自动带 traceId，无需手动传 |
| 权限检查统一 | 任意层级的工具都能获取当前用户身份 |
| 测试友好 | 测试时注入 mock context，工具代码零修改 |
| Sub-agent 透明继承 | 子代理自动继承父代理的 context |

---

## 核心实现：AsyncLocalStorage

Node.js 的 `AsyncLocalStorage` 是上下文传播的基础设施。它像一个"隐式参数"，在整个异步调用链中自动传递：

```typescript
// context-store.ts
import { AsyncLocalStorage } from 'async_hooks';

export interface AgentContextData {
  // 身份
  userId: string;
  tenantId: string;
  sessionId: string;
  
  // 追踪
  traceId: string;
  spanId: string;
  requestId: string;
  
  // 权限
  authToken?: string;
  permissions: Set<string>;
  
  // 功能开关
  featureFlags: Record<string, boolean>;
  
  // 计量
  startedAt: number;
  budgetTokens?: number;
}

const storage = new AsyncLocalStorage<AgentContextData>();

export const AgentContext = {
  // 启动一个新 context（在 Agent Loop 入口调用）
  run<T>(ctx: AgentContextData, fn: () => Promise<T>): Promise<T> {
    return storage.run(ctx, fn);
  },

  // 在任意嵌套层获取当前 context
  current(): AgentContextData {
    const ctx = storage.getStore();
    if (!ctx) throw new Error('AgentContext.current() called outside of context');
    return ctx;
  },

  // 安全版本（不抛异常）
  get(): AgentContextData | undefined {
    return storage.getStore();
  },

  // 局部更新 context（不影响父级）
  patch<T>(patch: Partial<AgentContextData>, fn: () => Promise<T>): Promise<T> {
    const current = AgentContext.current();
    return storage.run({ ...current, ...patch }, fn);
  },
};
```

---

## Agent Loop 入口注入

```typescript
// agent-loop.ts
import { AgentContext } from './context-store';
import { randomUUID } from 'crypto';

export async function runAgent(request: AgentRequest): Promise<AgentResponse> {
  // 在 Agent Loop 最外层建立 context
  return AgentContext.run({
    userId: request.userId,
    tenantId: request.tenantId,
    sessionId: request.sessionId,
    traceId: request.traceId ?? randomUUID(),
    spanId: randomUUID(),
    requestId: randomUUID(),
    authToken: request.authToken,
    permissions: new Set(request.permissions ?? []),
    featureFlags: request.featureFlags ?? {},
    startedAt: Date.now(),
    budgetTokens: request.budgetTokens,
  }, async () => {
    // 从这里开始，整个调用链都能访问 context
    return executeAgentLoop(request.messages);
  });
}
```

---

## 工具函数中使用 Context

```typescript
// tools/database.ts
import { AgentContext } from '../context-store';

export async function queryDatabase(sql: string): Promise<QueryResult> {
  const { userId, traceId, tenantId } = AgentContext.current();
  
  // 自动多租户隔离
  const connection = await getConnection(tenantId);
  
  // 自动日志关联
  logger.info('DB query', { traceId, userId, sql: sql.slice(0, 200) });
  
  // 执行查询（自动在 tenant schema 下）
  const result = await connection.query(sql);
  
  return result;
}

// tools/external-api.ts
export async function callExternalAPI(url: string, body: unknown): Promise<unknown> {
  const { authToken, traceId, featureFlags } = AgentContext.current();
  
  // 自动注入 auth header
  const response = await fetch(url, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${authToken}`,
      'X-Trace-Id': traceId,             // 分布式追踪透传
      'Content-Type': 'application/json',
    },
    body: JSON.stringify(body),
  });
  
  // 功能开关：新 API 灰度
  if (featureFlags['new_api_response_format']) {
    return parseNewFormat(await response.json());
  }
  return response.json();
}
```

---

## 工具中间件自动注入 Context

更优雅的方式是在**工具调度层**自动处理，工具代码完全不感知：

```typescript
// tool-dispatcher.ts
import { AgentContext } from './context-store';

type ToolFn = (params: Record<string, unknown>) => Promise<unknown>;

export function wrapWithContext(toolFn: ToolFn): ToolFn {
  return async (params) => {
    const ctx = AgentContext.current();
    
    // 记录工具调用（自动带 context）
    const spanId = randomUUID();
    logger.info('tool_call_start', {
      tool: toolFn.name,
      traceId: ctx.traceId,
      spanId,
      userId: ctx.userId,
      params: sanitizeParams(params),  // 脱敏
    });
    
    const startMs = Date.now();
    
    try {
      // 在子 span 中执行工具，spanId 更新
      const result = await AgentContext.patch({ spanId }, () => toolFn(params));
      
      logger.info('tool_call_end', {
        tool: toolFn.name,
        traceId: ctx.traceId,
        spanId,
        durationMs: Date.now() - startMs,
        success: true,
      });
      
      return result;
    } catch (err) {
      logger.error('tool_call_error', {
        tool: toolFn.name,
        traceId: ctx.traceId,
        spanId,
        durationMs: Date.now() - startMs,
        error: String(err),
      });
      throw err;
    }
  };
}

// 注册时自动包装
export class ToolRegistry {
  private tools = new Map<string, ToolFn>();
  
  register(name: string, fn: ToolFn) {
    this.tools.set(name, wrapWithContext(fn));  // 自动注入 context 追踪
  }
  
  async dispatch(name: string, params: Record<string, unknown>) {
    const tool = this.tools.get(name);
    if (!tool) throw new Error(`Unknown tool: ${name}`);
    return tool(params);
  }
}
```

---

## Sub-agent 跨进程传播

当父 Agent 派发 Sub-agent 时，context 需要序列化传递：

```typescript
// parent agent 派发 sub-agent
async function spawnSubAgent(task: string): Promise<string> {
  const ctx = AgentContext.current();
  
  // 序列化 context 传给子代理（只传安全字段）
  const propagatedCtx = {
    traceId: ctx.traceId,        // 追踪 ID 继承，让整条链路可查
    parentSpanId: ctx.spanId,    // 标记父子关系
    userId: ctx.userId,
    tenantId: ctx.tenantId,
    // authToken 不传！子代理用自己的凭据
    featureFlags: ctx.featureFlags,  // 功能开关继承
  };
  
  const result = await callSubAgent({
    task,
    contextHeaders: propagatedCtx,  // 通过 HTTP header 或消息字段传递
  });
  
  return result;
}

// sub-agent 入口恢复 context
export async function subAgentEntrypoint(request: SubAgentRequest) {
  const parentCtx = request.contextHeaders;
  
  return AgentContext.run({
    ...parentCtx,
    spanId: randomUUID(),          // 新 span
    sessionId: randomUUID(),        // 新会话
    authToken: getServiceToken(),   // 用自己的 token
    permissions: getServicePermissions(),
    startedAt: Date.now(),
  }, async () => {
    return runSubAgentLoop(request.task);
  });
}
```

---

## OpenClaw 实战：权限感知工具

OpenClaw 的 Skills 体系中，工具权限检查可以完全依赖 context：

```typescript
// skills/dangerous-tool/index.ts

// 权限检查中间件
function requirePermission(permission: string) {
  return function(toolFn: ToolFn): ToolFn {
    return async (params) => {
      const { permissions, userId } = AgentContext.current();
      
      if (!permissions.has(permission)) {
        throw new Error(
          `Permission denied: ${permission} required. ` +
          `User ${userId} has: [${[...permissions].join(', ')}]`
        );
      }
      
      return toolFn(params);
    };
  };
}

// 使用：工具定义完全干净，权限在外层声明
export const deleteUserData = requirePermission('admin:delete')(
  async ({ userId }: { userId: string }) => {
    // 直接干活，不用检查权限
    await db.deleteUser(userId);
    return { deleted: userId };
  }
);
```

---

## Context 调试：Inspector

```typescript
// context-inspector.ts - 开发时使用

export function contextSnapshot(): string {
  const ctx = AgentContext.get();
  if (!ctx) return '(no context)';
  
  return JSON.stringify({
    userId: ctx.userId,
    tenantId: ctx.tenantId,
    traceId: ctx.traceId.slice(0, 8) + '...',
    permissions: [...ctx.permissions],
    featureFlags: Object.entries(ctx.featureFlags)
      .filter(([, v]) => v)
      .map(([k]) => k),
    ageMs: Date.now() - ctx.startedAt,
  }, null, 2);
}

// 添加到 Agent 的 debug 工具
export const debugContext = async () => {
  return { context: contextSnapshot() };
};
```

---

## Python 版本（learn-claude-code 风格）

Python 用 `contextvars` 实现同等效果：

```python
# context_store.py
from contextvars import ContextVar
from dataclasses import dataclass, field
from typing import Optional
import uuid

@dataclass
class AgentContext:
    user_id: str
    tenant_id: str
    trace_id: str = field(default_factory=lambda: str(uuid.uuid4()))
    span_id: str = field(default_factory=lambda: str(uuid.uuid4()))
    auth_token: Optional[str] = None
    permissions: set = field(default_factory=set)
    feature_flags: dict = field(default_factory=dict)

_current_context: ContextVar[Optional[AgentContext]] = ContextVar(
    'agent_context', 
    default=None
)

def set_context(ctx: AgentContext) -> None:
    _current_context.set(ctx)

def get_context() -> AgentContext:
    ctx = _current_context.get()
    if ctx is None:
        raise RuntimeError("No AgentContext in current task")
    return ctx

# 使用 contextmanager 注入
from contextlib import contextmanager

@contextmanager
def agent_context(ctx: AgentContext):
    token = _current_context.set(ctx)
    try:
        yield ctx
    finally:
        _current_context.reset(token)

# 工具函数中使用
def query_database(sql: str) -> dict:
    ctx = get_context()
    logger.info(f"DB query trace={ctx.trace_id} user={ctx.user_id}")
    return db.execute(sql, tenant=ctx.tenant_id)
```

---

## 关键设计原则

```
┌─────────────────────────────────────────────────────────┐
│                    Agent Request                        │
│    userId + tenantId + traceId + permissions            │
└────────────────────────┬────────────────────────────────┘
                         │ AgentContext.run(ctx, ...)
                         ▼
              ┌──────────────────────┐
              │     Agent Loop       │
              │  (context 自动可用)   │
              └──────┬───────────────┘
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
    Tool A        Tool B       Sub-Agent
   (inherit)    (inherit)    (patch+propagate)
        │            │            │
        └────────────┴────────────┘
              所有层共享 context
              工具函数无需显式传参
```

**三个黄金规则：**
1. **注入一次**：在 Agent Loop 最外层 `run(ctx, ...)` 注入，之后从不手动传
2. **读取任意层**：工具、中间件、日志器都用 `AgentContext.current()` 读取
3. **跨进程序列化**：Sub-agent 收到传播的 traceId，但用自己的 authToken

---

## 总结

| 问题 | 解决方案 |
|------|----------|
| 工具参数爆炸 | AsyncLocalStorage 隐式传递环境上下文 |
| 日志无法关联 | 中间件自动把 traceId 注入每条日志 |
| 多租户隔离 | 工具从 context 读 tenantId，无需调用方传 |
| Sub-agent 追踪 | 序列化 traceId 跨进程传播，链路可查 |
| 权限检查分散 | 权限装饰器从 context 读 permissions 统一校验 |

上下文传播是"隐形的胶水"——它让 Agent 各层之间保持一致性，同时保持每个工具函数的简洁与可测试性。
