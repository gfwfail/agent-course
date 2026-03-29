# 170 - Agent 审批网关与多级授权

> Approval Gateway & Multi-Level Authorization：危险工具调用先审后执行，多级审批链 + Telegram 内联按钮实战

---

## 问题背景

Agent 拿到一把锤子，看啥都像钉子。当工具可以删数据库、发邮件、转账，你需要一道**人工审批关卡**——但不能让每个操作都打断用户。

目标：
- 低风险工具：自动执行
- 中风险工具：记录日志但放行
- 高风险工具：挂起，等人工审批
- 关键操作：多级审批（L1 → L2 → 执行）

---

## 核心架构

```
Tool Call
    ↓
[Risk Classifier]      ← 评估风险等级
    ↓
[Approval Gateway]
  ├── LOW    → 直接执行
  ├── MEDIUM → 记录 + 执行
  ├── HIGH   → 挂起 → Telegram 审批 → 继续/拒绝
  └── CRITICAL → 多级审批 → 执行
```

---

## 实现：审批网关（TypeScript）

### 风险注册表

```typescript
// approval-gateway.ts
type RiskLevel = 'LOW' | 'MEDIUM' | 'HIGH' | 'CRITICAL';

interface ToolRiskPolicy {
  level: RiskLevel;
  description: string;
  autoApproveCondition?: (params: unknown) => boolean;
  requireApprovers?: number; // CRITICAL: 需要几个人审批
}

const TOOL_RISK_REGISTRY: Record<string, ToolRiskPolicy> = {
  read_file:       { level: 'LOW',      description: '读取文件' },
  web_search:      { level: 'LOW',      description: '搜索网页' },
  send_email:      { level: 'MEDIUM',   description: '发送邮件' },
  delete_record:   { level: 'HIGH',     description: '删除数据库记录' },
  deploy_to_prod:  { level: 'CRITICAL', description: '生产部署', requireApprovers: 2 },
  transfer_funds:  { level: 'CRITICAL', description: '转账操作', requireApprovers: 2 },
  
  // 条件自动审批：删除少量数据可以自动通过
  bulk_delete: {
    level: 'HIGH',
    description: '批量删除',
    autoApproveCondition: (params: any) => params.count < 10,
  },
};
```

### 挂起状态机

```typescript
// approval-store.ts（Redis 存储）
import Redis from 'ioredis';

interface PendingApproval {
  id: string;
  sessionId: string;
  toolName: string;
  params: unknown;
  riskLevel: RiskLevel;
  requestedAt: number;
  approvals: string[];   // 已审批人 ID
  requiredApprovals: number;
  status: 'PENDING' | 'APPROVED' | 'REJECTED' | 'TIMEOUT';
  timeoutMs: number;
}

class ApprovalStore {
  private redis: Redis;
  private TIMEOUT_MS = 5 * 60 * 1000; // 5 分钟

  constructor(redis: Redis) {
    this.redis = redis;
  }

  async create(approval: Omit<PendingApproval, 'id' | 'requestedAt' | 'status' | 'approvals'>): Promise<PendingApproval> {
    const id = `approval:${crypto.randomUUID()}`;
    const record: PendingApproval = {
      ...approval,
      id,
      requestedAt: Date.now(),
      approvals: [],
      status: 'PENDING',
    };
    
    // TTL = 超时时间 + 30s 缓冲
    await this.redis.setex(id, Math.ceil(approval.timeoutMs / 1000) + 30, JSON.stringify(record));
    return record;
  }

  async get(id: string): Promise<PendingApproval | null> {
    const raw = await this.redis.get(id);
    return raw ? JSON.parse(raw) : null;
  }

  async addApproval(id: string, approverId: string): Promise<PendingApproval | null> {
    const approval = await this.get(id);
    if (!approval || approval.status !== 'PENDING') return null;

    approval.approvals.push(approverId);
    
    // 达到所需审批数 → 自动转为 APPROVED
    if (approval.approvals.length >= approval.requiredApprovals) {
      approval.status = 'APPROVED';
    }

    const ttl = await this.redis.ttl(id);
    await this.redis.setex(id, ttl > 0 ? ttl : 30, JSON.stringify(approval));
    return approval;
  }

  async reject(id: string, reason: string): Promise<void> {
    const approval = await this.get(id);
    if (!approval) return;
    approval.status = 'REJECTED';
    await this.redis.setex(id, 60, JSON.stringify({ ...approval, rejectReason: reason }));
  }
}
```

### 审批网关主逻辑

```typescript
// approval-gateway.ts
export class ApprovalGateway {
  private store: ApprovalStore;
  private notifier: ApprovalNotifier; // 发 Telegram 通知

  constructor(store: ApprovalStore, notifier: ApprovalNotifier) {
    this.store = store;
    this.notifier = notifier;
  }

  async executeWithApproval(
    toolName: string,
    params: unknown,
    executor: () => Promise<unknown>,
    context: { sessionId: string; userId: string }
  ): Promise<unknown> {
    const policy = TOOL_RISK_REGISTRY[toolName];
    if (!policy) return executor(); // 未注册的工具直接执行

    // LOW: 直接执行
    if (policy.level === 'LOW') return executor();

    // MEDIUM: 记录日志后执行
    if (policy.level === 'MEDIUM') {
      console.log(`[AUDIT] ${toolName} called by ${context.userId}`, params);
      return executor();
    }

    // 条件自动审批
    if (policy.autoApproveCondition?.(params)) {
      console.log(`[AUTO-APPROVED] ${toolName}`);
      return executor();
    }

    // HIGH / CRITICAL: 挂起，等待审批
    const pending = await this.store.create({
      sessionId: context.sessionId,
      toolName,
      params,
      riskLevel: policy.level,
      requiredApprovals: policy.requireApprovers ?? 1,
      timeoutMs: 5 * 60 * 1000,
    });

    // 发送 Telegram 审批通知
    await this.notifier.sendApprovalRequest(pending);

    // 轮询等待审批结果（最多 5 分钟）
    const result = await this.waitForApproval(pending.id);

    if (result.status === 'APPROVED') {
      return executor();
    } else if (result.status === 'TIMEOUT') {
      throw new Error(`[APPROVAL TIMEOUT] ${toolName} 审批超时，操作已取消`);
    } else {
      throw new Error(`[APPROVAL REJECTED] ${toolName} 被拒绝：${result.rejectReason ?? '无原因'}`);
    }
  }

  private async waitForApproval(id: string, intervalMs = 3000): Promise<PendingApproval> {
    const deadline = Date.now() + 5 * 60 * 1000;

    while (Date.now() < deadline) {
      const approval = await this.store.get(id);
      
      if (!approval) throw new Error('审批记录不存在');
      if (approval.status !== 'PENDING') return approval;
      
      await new Promise(r => setTimeout(r, intervalMs));
    }

    // 超时处理
    await this.store.reject(id, 'TIMEOUT');
    return { ...(await this.store.get(id))!, status: 'TIMEOUT' };
  }
}
```

---

## Telegram 内联按钮审批通知

```typescript
// approval-notifier.ts（OpenClaw message tool 风格）
class ApprovalNotifier {
  async sendApprovalRequest(approval: PendingApproval): Promise<void> {
    const risk_emoji = { HIGH: '🔶', CRITICAL: '🔴' }[approval.riskLevel] ?? '⚠️';
    const policy = TOOL_RISK_REGISTRY[approval.toolName];

    const text = `
${risk_emoji} **工具调用审批请求**

🔧 工具：\`${approval.toolName}\`
📝 描述：${policy?.description ?? '未知'}
⚠️ 风险：${approval.riskLevel}
📋 参数：\`\`\`json
${JSON.stringify(approval.params, null, 2).slice(0, 500)}
\`\`\`

需要 ${approval.requiredApprovals} 人审批，5 分钟后超时。
审批 ID：\`${approval.id}\`
    `.trim();

    // 通过 OpenClaw message tool 发送（支持 inline buttons）
    await sendTelegramMessage({
      target: ADMIN_CHAT_ID,
      message: text,
      // OpenClaw capabilities: inlineButtons
      replyMarkup: {
        inline_keyboard: [[
          { text: '✅ 批准', callback_data: `approve:${approval.id}` },
          { text: '❌ 拒绝', callback_data: `reject:${approval.id}` },
        ]]
      }
    });
  }
}
```

### OpenClaw Callback 处理（在 Agent 中监听）

```typescript
// 在 OpenClaw 中处理 inline button 回调
// 当用户点击 ✅ 批准 时，OpenClaw 发送 callback_data 给 Agent

async function handleCallback(callbackData: string, approverId: string) {
  const [action, approvalId] = callbackData.split(':');
  
  if (action === 'approve') {
    const approval = await approvalStore.addApproval(approvalId, approverId);
    
    if (approval?.status === 'APPROVED') {
      await sendTelegramMessage(`✅ 已批准！工具调用将继续执行。`);
    } else if (approval) {
      const remaining = approval.requiredApprovals - approval.approvals.length;
      await sendTelegramMessage(`✅ ${approverId} 已批准，还需 ${remaining} 人审批。`);
    }
  } else if (action === 'reject') {
    await approvalStore.reject(approvalId, `${approverId} 手动拒绝`);
    await sendTelegramMessage(`❌ 已拒绝，工具调用取消。`);
  }
}
```

---

## 集成到 Tool 分发层（中间件模式）

```typescript
// tool-middleware.ts
function withApprovalGateway(gateway: ApprovalGateway) {
  return async (
    toolName: string,
    params: unknown,
    next: () => Promise<unknown>,
    ctx: ToolContext
  ): Promise<unknown> => {
    return gateway.executeWithApproval(toolName, params, next, {
      sessionId: ctx.sessionId,
      userId: ctx.userId,
    });
  };
}

// 注册中间件
toolRegistry.use(withApprovalGateway(approvalGateway));
toolRegistry.use(withAuditLog());
toolRegistry.use(withRateLimit());
```

---

## OpenClaw 实战模式

OpenClaw 中的 `capabilities: inlineButtons` 正是为此设计的：

```
Agent 想执行危险操作
    ↓
ApprovalGateway 挂起
    ↓
发送 Telegram 消息 + inline buttons [✅批准] [❌拒绝]
    ↓
老板点击按钮 → callback_data 触发
    ↓
ApprovalStore 更新状态
    ↓
waitForApproval() 解除挂起
    ↓
Agent 继续执行（或取消）
```

这就是 **人机协作（HITL）的最小可行实现**——Agent 自主运行，人只在关键节点介入。

---

## 超时降级策略

```typescript
// 超时策略：可配置为自动拒绝或自动批准（低风险场景）
interface TimeoutPolicy {
  action: 'AUTO_APPROVE' | 'AUTO_REJECT' | 'ESCALATE';
  escalateTo?: string; // 升级给谁
}

const TIMEOUT_POLICIES: Record<RiskLevel, TimeoutPolicy> = {
  LOW:      { action: 'AUTO_APPROVE' },
  MEDIUM:   { action: 'AUTO_APPROVE' },
  HIGH:     { action: 'AUTO_REJECT' },    // 超时 → 拒绝，安全优先
  CRITICAL: { action: 'ESCALATE', escalateTo: 'SUPER_ADMIN' },
};
```

---

## 关键设计要点

| 场景 | 策略 |
|------|------|
| 超时未响应 | HIGH 自动拒绝，CRITICAL 升级 |
| 审批人不在线 | TTL 兜底 + 事后审计日志 |
| 多级审批死锁 | 任一审批人可拒绝，需所有人批准才通过 |
| 审批幂等 | 同一人不能重复批准（approvals 去重） |
| 参数篡改 | 审批 ID 绑定参数哈希，防止批准后换参数 |

---

## 小结

1. **Risk Registry**：工具 → 风险等级的映射表，支持条件自动审批
2. **ApprovalStore**：Redis 存储挂起状态，TTL 自动超时
3. **waitForApproval**：Agent Loop 在此挂起，轮询直到有结果
4. **Telegram Inline Buttons**：OpenClaw `capabilities: inlineButtons` 实现零延迟审批
5. **中间件注入**：不改工具逻辑，在分发层透明拦截

> 核心原则：**Agent 应该能做任何事，但某些事必须先问人。** 审批网关是这条边界的技术实现。

---

*下一课预告：Agent 流量镜像与双写测试（Traffic Mirroring & Dual-Write Testing）*
