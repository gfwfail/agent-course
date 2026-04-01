# 189 - Agent 自然语言策略引擎（Natural Language Policy Engine）

> 用自然语言声明行为约束，LLM 实时检查，支持热更新——比硬编码规则更灵活，比复杂 DSL 更易维护。

---

## 为什么需要自然语言策略？

传统 Agent 的行为约束往往写死在代码里：

```typescript
// 硬编码规则——难维护，改一条要改代码+重部署
if (tool === 'send_email' && hour >= 22) {
  throw new Error('不允许在夜间发送邮件');
}
if (tool === 'delete_file' && !path.startsWith('/tmp')) {
  throw new Error('只允许删除临时文件');
}
```

问题很明显：
- 每条新规则要改代码
- 规则之间的交互难以推理
- 非技术人员无法参与定义策略
- 无法动态更新（热更新困难）

**自然语言策略引擎**解决这些问题：把约束写成自然语言，让 LLM 在执行工具前检查是否违反策略。

---

## 核心架构

```
用户请求
    ↓
Agent Loop
    ↓
LLM 决定调用工具
    ↓
[PolicyEngine.check(context, tool, args)]  ← 拦截层
    ↓ ALLOW
工具执行
    ↓ DENY
返回 PolicyDenied 结果给 LLM，让其调整策略
```

### 策略定义格式

```typescript
interface Policy {
  id: string;
  name: string;
  description: string;        // 给 LLM 看的解释
  rule: string;               // 自然语言规则
  scope: 'global' | 'tool' | 'user';
  toolFilter?: string[];      // 仅对哪些工具生效
  severity: 'block' | 'warn' | 'audit';
  enabled: boolean;
}

// 示例策略
const policies: Policy[] = [
  {
    id: 'no-night-marketing',
    name: '禁止夜间营销',
    rule: '不要在晚上10点到早上8点之间发送任何营销类邮件或消息',
    scope: 'tool',
    toolFilter: ['send_email', 'send_message'],
    severity: 'block',
    enabled: true,
  },
  {
    id: 'read-before-write',
    name: '写前先读',
    rule: '修改任何文件之前，必须先在同一轮对话中读取过该文件，防止盲目覆盖',
    scope: 'global',
    severity: 'warn',
    enabled: true,
  },
  {
    id: 'no-prod-delete',
    name: '禁止删除生产数据',
    rule: '严禁删除包含 production、prod、live 关键词的文件或数据库记录',
    scope: 'tool',
    toolFilter: ['delete_file', 'db_execute'],
    severity: 'block',
    enabled: true,
  },
  {
    id: 'budget-guard',
    name: '成本预算守卫',
    rule: '如果本次会话已花费超过 $5，不得再调用任何外部 API 付费工具',
    scope: 'global',
    severity: 'block',
    enabled: true,
  },
];
```

---

## TypeScript 实现

```typescript
import Anthropic from '@anthropic-ai/sdk';

interface PolicyCheckContext {
  currentTime: string;
  sessionMessages: Array<{ role: string; toolName?: string }>;
  sessionCostUsd: number;
  userId: string;
  userRole: string;
}

interface PolicyCheckResult {
  decision: 'ALLOW' | 'DENY' | 'WARN';
  violatedPolicies: Array<{ id: string; name: string; reason: string }>;
  explanation: string;
}

class NLPolicyEngine {
  private client: Anthropic;
  private policies: Policy[] = [];
  private policyCache = new Map<string, PolicyCheckResult>();

  constructor() {
    this.client = new Anthropic();
  }

  // 热更新策略——无需重启
  updatePolicies(policies: Policy[]) {
    this.policies = policies;
    this.policyCache.clear(); // 清除缓存
    console.log(`[PolicyEngine] 已更新 ${policies.length} 条策略`);
  }

  async check(
    toolName: string,
    toolArgs: Record<string, unknown>,
    context: PolicyCheckContext
  ): Promise<PolicyCheckResult> {
    // 筛选适用的策略
    const applicablePolicies = this.policies.filter(p => {
      if (!p.enabled) return false;
      if (p.toolFilter && !p.toolFilter.includes(toolName)) return false;
      return true;
    });

    if (applicablePolicies.length === 0) {
      return { decision: 'ALLOW', violatedPolicies: [], explanation: '无适用策略' };
    }

    // 构建缓存 key（粗粒度：工具+参数摘要+时间段）
    const cacheKey = `${toolName}:${JSON.stringify(toolArgs)}:${context.currentTime.slice(0, 13)}`;
    if (this.policyCache.has(cacheKey)) {
      return this.policyCache.get(cacheKey)!;
    }

    // 用 Haiku 快速检查（省成本）
    const policiesText = applicablePolicies
      .map(p => `[${p.id}] ${p.name}: ${p.rule}`)
      .join('\n');

    const response = await this.client.messages.create({
      model: 'claude-haiku-4-5',
      max_tokens: 512,
      messages: [
        {
          role: 'user',
          content: `你是 Agent 策略检查器。判断即将执行的工具调用是否违反以下策略。

## 当前上下文
- 时间: ${context.currentTime}
- 用户角色: ${context.userRole}
- 会话花费: $${context.sessionCostUsd.toFixed(4)}
- 最近工具调用: ${context.sessionMessages.filter(m => m.toolName).map(m => m.toolName).slice(-5).join(', ') || '无'}

## 即将执行的工具
工具名称: ${toolName}
参数: ${JSON.stringify(toolArgs, null, 2)}

## 需要检查的策略
${policiesText}

## 要求
以 JSON 格式返回检查结果：
{
  "decision": "ALLOW" | "DENY" | "WARN",
  "violatedPolicies": [{"id": "策略ID", "name": "策略名", "reason": "违反原因"}],
  "explanation": "一句话说明决策原因"
}

只返回 JSON，不要其他内容。`,
        },
      ],
    });

    const text = response.content[0].type === 'text' ? response.content[0].text : '{}';
    
    let result: PolicyCheckResult;
    try {
      result = JSON.parse(text.trim());
    } catch {
      // 解析失败默认放行（fail-open，避免误伤）
      result = { decision: 'ALLOW', violatedPolicies: [], explanation: '策略检查解析失败，默认放行' };
    }

    // 将 DENY/WARN 结果缓存 60 秒（同样场景不重复调用 LLM）
    if (result.decision !== 'ALLOW') {
      this.policyCache.set(cacheKey, result);
      setTimeout(() => this.policyCache.delete(cacheKey), 60_000);
    }

    return result;
  }
}
```

---

## 集成到 Agent Loop 中间件

```typescript
// 工具调用中间件：策略检查层
class PolicyMiddleware {
  constructor(
    private engine: NLPolicyEngine,
    private getContext: () => PolicyCheckContext
  ) {}

  wrap(tools: Record<string, Function>): Record<string, Function> {
    const wrapped: Record<string, Function> = {};

    for (const [name, fn] of Object.entries(tools)) {
      wrapped[name] = async (args: Record<string, unknown>) => {
        const context = this.getContext();
        const result = await this.engine.check(name, args, context);

        if (result.decision === 'DENY') {
          // 返回结构化拒绝结果，让 LLM 知道原因并调整
          return {
            __policy_denied: true,
            violatedPolicies: result.violatedPolicies,
            explanation: result.explanation,
            suggestion: '请调整你的操作以符合当前策略约束。',
          };
        }

        if (result.decision === 'WARN') {
          console.warn(`[PolicyEngine] ⚠️ 工具 ${name} 触发警告:`, result.explanation);
          // 警告不阻止执行，但会记录审计日志
        }

        return fn(args);
      };
    }

    return wrapped;
  }
}

// 使用示例
const engine = new NLPolicyEngine();
engine.updatePolicies(policies);

const rawTools = { send_email, delete_file, write_file, db_execute };
const policyMiddleware = new PolicyMiddleware(engine, () => ({
  currentTime: new Date().toISOString(),
  sessionMessages: conversationHistory,
  sessionCostUsd: sessionCost,
  userId: 'user-123',
  userRole: 'operator',
}));

const tools = policyMiddleware.wrap(rawTools);
// tools 就是加了策略检查的工具集，Agent Loop 无需感知
```

---

## Python 版本

```python
import anthropic
import json
from dataclasses import dataclass
from typing import Literal
from datetime import datetime

@dataclass
class Policy:
    id: str
    name: str
    rule: str
    tool_filter: list[str] | None = None
    severity: Literal['block', 'warn', 'audit'] = 'block'
    enabled: bool = True

class NLPolicyEngine:
    def __init__(self):
        self.client = anthropic.Anthropic()
        self.policies: list[Policy] = []
    
    def update_policies(self, policies: list[Policy]):
        self.policies = policies
        print(f"[PolicyEngine] 已更新 {len(policies)} 条策略")
    
    async def check(self, tool_name: str, tool_args: dict, context: dict) -> dict:
        applicable = [
            p for p in self.policies
            if p.enabled and (p.tool_filter is None or tool_name in p.tool_filter)
        ]
        
        if not applicable:
            return {"decision": "ALLOW", "violatedPolicies": [], "explanation": "无适用策略"}
        
        policies_text = "\n".join(
            f"[{p.id}] {p.name}: {p.rule}" for p in applicable
        )
        
        response = self.client.messages.create(
            model="claude-haiku-4-5",
            max_tokens=512,
            messages=[{
                "role": "user",
                "content": f"""策略检查：工具 {tool_name}，参数 {json.dumps(tool_args)}
当前时间：{context.get('current_time', datetime.now().isoformat())}
策略列表：
{policies_text}

返回 JSON: {{"decision": "ALLOW"|"DENY"|"WARN", "violatedPolicies": [], "explanation": "..."}}"""
            }]
        )
        
        try:
            return json.loads(response.content[0].text.strip())
        except Exception:
            return {"decision": "ALLOW", "violatedPolicies": [], "explanation": "解析失败，默认放行"}
```

---

## OpenClaw 实战：SOUL.md 即策略文件

OpenClaw 本身就在用自然语言策略！看看 `SOUL.md`：

```markdown
## 原则
- 老板让干啥就干啥，不要自作主张换方案
- 遇到问题先汇报，等老板指示再操作
- 任何新方案必须先问老板，得到确认后才能执行

## 权限管理
- 只有被 @我 时才回复
- 老板：完全信任，可以执行任何指令
- 其他人：低权限，只能做基本交互
- 危险操作 → 必须跟老板确认后才能执行
```

这些**就是自然语言策略**！OpenClaw 在 system prompt 中注入这些规则，让 Claude 在每次决策前自动 "检查" 是否符合策略。

更进一步，可以把策略文件外化为热更新的 `POLICIES.md`：

```markdown
# POLICIES.md
## 活跃策略

1. **夜间静默**: 北京时间 23:00-08:00，不主动发送消息，仅响应紧急告警
2. **写前确认**: 修改生产配置前必须获得老板明确确认
3. **成本限额**: 单次对话 LLM 花费超过 $1 时，切换到 Haiku 节省成本  
4. **外发审查**: 发送邮件/推文等公开内容前，必须回显给老板预览
```

---

## 关键设计决策

### 1. Fail-Open vs Fail-Close

```typescript
// Fail-Open：策略检查失败时放行（防止误伤正常操作）
// 适合：内部工具、低风险场景
if (policyCheckFailed) return { decision: 'ALLOW' };

// Fail-Close：策略检查失败时拒绝（防止安全漏洞）
// 适合：涉及资金、数据删除的高风险工具
if (policyCheckFailed) return { decision: 'DENY', explanation: '策略检查服务不可用' };
```

### 2. 策略粒度

| 粒度 | 示例 | 成本 |
|------|------|------|
| 全局策略 | "不得泄露用户PII" | 每次工具调用都检查 |
| 工具级策略 | "send_email 夜间禁用" | 仅目标工具触发 |
| 用户级策略 | "VIP用户不限速" | 按用户身份检查 |
| 会话级策略 | "测试模式不写生产DB" | 会话开始时设置 |

### 3. 缓存策略（降低 LLM 调用成本）

```typescript
// 相同工具+相似参数+相同时段 → 缓存检查结果
// 避免每次工具调用都调用 Haiku（额外 $0.00025/次）
const cacheHitRate = 0.7; // 典型场景 70% 命中率
// 实际新增成本：0.3 × $0.00025 = $0.000075/次，可接受
```

---

## 实测效果

| 场景 | 传统硬编码 | NL 策略引擎 |
|------|-----------|------------|
| 新增一条规则 | 改代码 + 测试 + 部署（30min） | 修改策略文件（30s） |
| 规则更新生效 | 需要重部署 | 热更新，即时生效 |
| 规则可读性 | 代码逻辑 | 自然语言，产品经理可参与 |
| 规则之间交互推理 | 手动分析 | LLM 自动处理边界情况 |
| 额外成本 | 0 | ~$0.000075/次工具调用 |

---

## 何时用，何时不用

✅ **适合用 NL 策略引擎的场景：**
- 业务规则复杂且经常变化（合规要求、营销规则）
- 非技术人员需要参与定义策略
- 边界情况多、硬编码容易漏掉

❌ **不适合的场景：**
- 纯技术限制（内存/CPU 上限）→ 用代码守卫
- 极高频工具调用（每秒数千次）→ 缓存+降级到规则引擎
- 确定性要求极高（金融清算）→ 用显式规则 + NL 策略作补充

---

## 总结

**自然语言策略引擎 = 自然语言策略文件 + Haiku 实时检查 + 中间件透明注入**

三个核心收益：
1. **灵活性**：策略和代码分离，产品/运营可以直接修改规则
2. **热更新**：策略变更无需重部署
3. **可读性**：自然语言规则，任何人都能理解和参与维护

OpenClaw 的 SOUL.md 就是这个模式最好的落地案例——用自然语言定义 Agent 的行为边界，让 Claude 内化成自己的"职业道德"。

---

*相关课程：[08-safety-guardrails](08-safety-guardrails.md) | [24-tool-policy-pipeline](24-tool-policy-pipeline.md) | [120-tool-least-privilege](120-tool-least-privilege.md) | [170-approval-gateway-multi-level-auth](170-approval-gateway-multi-level-auth.md)*
