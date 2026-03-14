# 48 - 多租户 Agent 架构

> **核心问题**：当你的 Agent 平台要服务多个用户/团队/客户时，如何做到数据隔离、权限控制、资源公平分配？

---

## 什么是多租户 Agent 架构

多租户（Multi-Tenant）指同一套 Agent 基础设施同时服务多个"租户"（用户、团队、企业），但每个租户的：

- **数据**：互相不可见
- **权限**：各自独立配置
- **资源**：公平分配，互不干扰
- **行为**：可个性化定制

典型场景：SaaS AI 平台、企业内部多部门 Agent、API 服务商。

---

## 三种隔离模式

### 1. 进程级隔离（最强）

每个租户独立进程/容器，完全隔离。

```
Tenant A → Agent Process A → DB A
Tenant B → Agent Process B → DB B
```

**优点**：安全性最高，崩溃不互相影响  
**缺点**：资源消耗大，冷启动慢

### 2. 会话级隔离（平衡）

共享进程，但每个租户有独立会话上下文。

```python
# OpenClaw 的 Session 模型就是这种
class Session:
    tenant_id: str
    session_key: str
    memory: dict       # 租户私有内存
    tool_policy: dict  # 租户级工具权限
    model: str         # 租户可选不同模型
```

### 3. 上下文级隔离（最轻量）

同一会话内通过 prompt 注入区分租户，依赖模型理解。  
**不推荐用于生产**，有 prompt injection 风险。

---

## OpenClaw 的多租户实现

OpenClaw 天然支持多租户，核心是 **Session** 层：

```json
// config.yaml 片段
sessions:
  - key: "tenant-a-main"
    channel: telegram
    channelTarget: "123456"
    model: anthropic/claude-opus-4
    workspace: /tenants/a/workspace

  - key: "tenant-b-main"  
    channel: telegram
    channelTarget: "789012"
    model: anthropic/claude-haiku-3
    workspace: /tenants/b/workspace
```

每个 Session 有：
- 独立 workspace（文件系统隔离）
- 独立 model（成本控制）
- 独立 channel（通信隔离）
- 独立 memory（上下文隔离）

---

## 权限控制层设计

```typescript
// pi-mono 风格的租户权限中间件
interface TenantContext {
  tenantId: string;
  tier: 'free' | 'pro' | 'enterprise';
  allowedTools: string[];
  tokenBudget: number;
  modelAllowlist: string[];
}

// 工具调用前检查权限
async function dispatchTool(
  tool: string,
  params: unknown,
  ctx: TenantContext
): Promise<ToolResult> {
  // 1. 工具白名单检查
  if (!ctx.allowedTools.includes(tool)) {
    return { error: `Tool '${tool}' not available on ${ctx.tier} plan` };
  }
  
  // 2. Token 预算检查
  if (ctx.tokenBudget <= 0) {
    return { error: 'Token budget exhausted for this billing period' };
  }
  
  // 3. 执行工具
  const result = await executeTool(tool, params);
  
  // 4. 扣减预算
  await deductBudget(ctx.tenantId, result.tokensUsed);
  
  return result;
}
```

---

## 数据隔离策略

### 文件系统隔离

```bash
/workspaces/
  tenant-a/
    MEMORY.md
    memory/
    projects/
  tenant-b/
    MEMORY.md
    memory/
    projects/
```

Agent 的 `cwd` 锁定到租户目录，防止越权访问。

### 数据库隔离

```sql
-- 方案1：行级隔离（共享表，租户字段过滤）
SELECT * FROM agent_logs 
WHERE tenant_id = 'tenant-a';  -- 必须带租户过滤

-- 方案2：Schema 隔离（PostgreSQL）
SET search_path = tenant_a;
SELECT * FROM agent_logs;  -- 自动访问租户 A 的表

-- 方案3：独立数据库（最强隔离）
-- 连接字符串包含租户标识
```

### 向量数据库隔离（RAG 场景）

```python
# 每次查询必须带 tenant_id 过滤
results = vector_db.query(
    vector=query_embedding,
    filter={"tenant_id": {"$eq": current_tenant}},  # 关键！
    top_k=5
)
```

---

## 资源公平调度

多租户最大挑战：防止一个"吵闹租户"（noisy neighbor）耗尽资源。

```python
# 令牌桶限流（per-tenant）
class TenantRateLimiter:
    def __init__(self, tenant_id: str, rpm: int):
        self.tenant_id = tenant_id
        self.rpm = rpm
        self.tokens = rpm
        self.last_refill = time.time()
    
    async def acquire(self) -> bool:
        now = time.time()
        elapsed = now - self.last_refill
        
        # 按时间比例补充令牌
        self.tokens = min(
            self.rpm,
            self.tokens + elapsed * (self.rpm / 60)
        )
        self.last_refill = now
        
        if self.tokens >= 1:
            self.tokens -= 1
            return True
        return False  # 触发限流

# 每个租户独立的限流器
limiters: dict[str, TenantRateLimiter] = {
    'free-user': TenantRateLimiter('free', rpm=10),
    'pro-user': TenantRateLimiter('pro', rpm=60),
    'enterprise': TenantRateLimiter('ent', rpm=300),
}
```

---

## 租户上下文注入

每次 Agent 启动时，自动注入租户身份信息到系统提示词：

```python
def build_system_prompt(base_prompt: str, tenant: TenantContext) -> str:
    tenant_header = f"""
## 当前租户信息
- 租户 ID: {tenant.tenant_id}
- 套餐: {tenant.tier}
- 可用工具: {', '.join(tenant.allowedTools)}
- 数据目录: /workspaces/{tenant.tenantId}/

**重要**: 只能访问当前租户的数据，不得访问其他租户的文件或数据库记录。
"""
    return tenant_header + base_prompt
```

---

## 审计日志

多租户必须记录"谁做了什么"：

```typescript
interface AuditLog {
  timestamp: string;
  tenantId: string;
  sessionKey: string;
  action: 'tool_call' | 'file_read' | 'file_write' | 'external_request';
  resource: string;      // 操作的资源
  outcome: 'success' | 'denied' | 'error';
  tokensUsed?: number;
}

// 每次工具调用后写入审计日志
async function auditLog(entry: AuditLog) {
  await db.insert('audit_logs', entry);
  
  // 告警：访问异常
  if (entry.outcome === 'denied') {
    await alert(`Tenant ${entry.tenantId} attempted unauthorized access to ${entry.resource}`);
  }
}
```

---

## 实战：OpenClaw 多租户部署

```yaml
# 完整的多租户 OpenClaw 配置示例
sessions:
  # 免费用户 - 限制模型和工具
  - key: "free-user-001"
    model: anthropic/claude-haiku-3
    workspace: /tenants/free-001/
    tools:
      deny: [exec, browser, nodes]  # 禁用危险工具

  # Pro 用户 - 完整功能
  - key: "pro-user-002"
    model: anthropic/claude-sonnet-4-6
    workspace: /tenants/pro-002/
    # 全工具权限

  # 企业用户 - 自定义模型 + 优先级
  - key: "enterprise-003"
    model: anthropic/claude-opus-4
    workspace: /tenants/enterprise-003/
    priority: high
```

---

## 常见坑

| 问题 | 症状 | 解决方案 |
|------|------|----------|
| 内存泄漏跨租户 | A 用户的上下文污染 B | 每次请求重建 context，不共享可变状态 |
| 向量库无过滤 | RAG 返回其他租户数据 | 所有查询强制加 tenant_id filter |
| 日志混用 | 无法追踪单租户问题 | 所有日志带 tenant_id 字段 |
| 资源饥饿 | 大客户耗光 GPU/API 额度 | 按租户级别设独立限流器 |

---

## 总结

```
多租户 Agent 架构 = 隔离 + 权限 + 限流 + 审计

隔离层次：进程 > 会话 > 上下文
权限模型：工具白名单 + 数据过滤 + 预算控制
公平调度：令牌桶限流（per-tenant）
可观测性：审计日志 + 租户级监控
```

OpenClaw 的 Session 模型已经给了你多租户的基础骨架，
在此之上加权限中间件 + 数据过滤 + 限流器，就是生产级多租户平台。

---

*下一课预告：Agent 知识图谱集成 - 让 Agent 理解实体关系*
