# 第 24 课：Tool Policy Pipeline - 工具策略管道

> 如何通过多层策略精确控制 Agent 可用的工具

## 为什么需要工具策略？

在生产环境中，我们不能让 Agent 随意使用所有工具。不同场景需要不同的工具权限：

```
场景 A: 私人助理 → 完整工具集（邮件、日历、文件、执行命令）
场景 B: 客服机器人 → 只能查询知识库、发消息
场景 C: 代码审查 → 只能读文件、写评论，不能执行代码
场景 D: 公开群聊 → 基础工具，禁止敏感操作
```

工具策略管道就是解决这个问题的系统。

## 核心概念：策略管道

工具策略不是简单的 "允许/禁止" 列表，而是一个**多层过滤管道**：

```
所有工具
   ↓
[全局策略] → 过滤
   ↓
[Provider 策略] → 过滤
   ↓
[Agent 策略] → 过滤
   ↓
[Profile 策略] → 过滤
   ↓
[群组策略] → 过滤
   ↓
[发送者策略] → 过滤
   ↓
最终可用工具
```

每一层都可以进一步收紧权限（deny wins）。

## 实现：策略管道类型定义

```typescript
// 策略管道步骤
type ToolPolicyPipelineStep = {
    policy: ToolPolicyLike | undefined;  // 策略配置
    label: string;                         // 调试标签
    stripPluginOnlyAllowlist?: boolean;   // 是否移除插件专属工具
};

// 单个策略配置
type ToolPolicyConfig = {
    allow?: string[];      // 允许的工具列表
    alsoAllow?: string[];  // 额外允许（合并到 allow）
    deny?: string[];       // 禁止的工具列表
    profile?: ToolProfileId;  // 预设配置
};

// 预设 Profile
type ToolProfileId = "minimal" | "coding" | "messaging" | "full";
```

## 构建策略管道

```typescript
function buildDefaultToolPolicyPipelineSteps(params: {
    profilePolicy?: ToolPolicyLike;        // 用户 Profile 策略
    profile?: string;
    providerProfilePolicy?: ToolPolicyLike; // Provider Profile 策略
    providerProfile?: string;
    globalPolicy?: ToolPolicyLike;         // 全局策略
    globalProviderPolicy?: ToolPolicyLike; // 全局 Provider 策略
    agentPolicy?: ToolPolicyLike;          // Agent 策略
    agentProviderPolicy?: ToolPolicyLike;  // Agent Provider 策略
    groupPolicy?: ToolPolicyLike;          // 群组策略
    agentId?: string;
}): ToolPolicyPipelineStep[] {
    const steps: ToolPolicyPipelineStep[] = [];
    
    // 1. 全局策略（最宽松的限制）
    if (params.globalPolicy) {
        steps.push({
            policy: params.globalPolicy,
            label: 'global'
        });
    }
    
    // 2. Provider 级别策略
    if (params.globalProviderPolicy) {
        steps.push({
            policy: params.globalProviderPolicy,
            label: `global-provider`
        });
    }
    
    // 3. Agent 级别策略
    if (params.agentPolicy) {
        steps.push({
            policy: params.agentPolicy,
            label: `agent:${params.agentId}`
        });
    }
    
    // 4. Profile 策略
    if (params.profilePolicy) {
        steps.push({
            policy: params.profilePolicy,
            label: `profile:${params.profile}`
        });
    }
    
    // 5. 群组策略（最严格）
    if (params.groupPolicy) {
        steps.push({
            policy: params.groupPolicy,
            label: 'group'
        });
    }
    
    return steps;
}
```

## 应用策略管道

```typescript
function applyToolPolicyPipeline(params: {
    tools: AnyAgentTool[];
    toolMeta: (tool: AnyAgentTool) => { pluginId: string } | undefined;
    warn: (message: string) => void;
    steps: ToolPolicyPipelineStep[];
}): AnyAgentTool[] {
    let filteredTools = [...params.tools];
    
    for (const step of params.steps) {
        if (!step.policy) continue;
        
        const before = filteredTools.length;
        filteredTools = applyPolicy(filteredTools, step.policy);
        const after = filteredTools.length;
        
        if (before !== after) {
            params.warn(
                `[${step.label}] filtered ${before - after} tools ` +
                `(${before} → ${after})`
            );
        }
    }
    
    return filteredTools;
}

function applyPolicy(
    tools: AnyAgentTool[], 
    policy: ToolPolicyLike
): AnyAgentTool[] {
    // deny 优先级最高
    if (policy.deny?.length) {
        tools = tools.filter(t => !matchesPattern(t.name, policy.deny!));
    }
    
    // 如果有 allow 列表，只保留匹配的
    if (policy.allow?.length) {
        const effectiveAllow = [
            ...policy.allow,
            ...(policy.alsoAllow || [])
        ];
        tools = tools.filter(t => matchesPattern(t.name, effectiveAllow));
    }
    
    return tools;
}

// 支持通配符匹配
function matchesPattern(toolName: string, patterns: string[]): boolean {
    return patterns.some(pattern => {
        if (pattern === '*') return true;
        if (pattern.endsWith('*')) {
            return toolName.startsWith(pattern.slice(0, -1));
        }
        return toolName === pattern;
    });
}
```

## 实战：OpenClaw 的 Profile 系统

OpenClaw 预定义了几个工具 Profile：

```typescript
const TOOL_PROFILES = {
    // 最小化：只有基础交互
    minimal: {
        allow: [
            'read', 'write', 'edit',
            'web_search', 'web_fetch',
            'message'
        ]
    },
    
    // 编码模式：文件 + 执行
    coding: {
        allow: [
            'read', 'write', 'edit',
            'exec', 'process',
            'web_search', 'web_fetch',
            'browser'
        ]
    },
    
    // 消息模式：通信为主
    messaging: {
        allow: [
            'read', 'web_search', 'web_fetch',
            'message', 'tts',
            'memory_search', 'memory_get'
        ]
    },
    
    // 完整模式：所有工具
    full: {
        allow: ['*']
    }
};
```

## 配置示例

```yaml
# openclaw.yaml

# 全局工具策略
tools:
  profile: full  # 基础 profile
  deny:
    - gateway    # 全局禁止 gateway 工具
  
  # 按 Provider 调整
  byProvider:
    anthropic:
      alsoAllow:
        - sessions_spawn  # 只允许 Anthropic 模型使用子代理
    openai:
      deny:
        - exec  # OpenAI 模型禁止执行命令

# Agent 级别策略
agents:
  customer-support:
    tools:
      profile: messaging
      deny:
        - exec
        - write
        - edit
      allow:
        - read
        - web_search
        - message
  
  code-reviewer:
    tools:
      profile: coding
      deny:
        - exec  # 只能看，不能执行
      alsoAllow:
        - message  # 可以发评论

# 群组策略
groups:
  public-chat:
    tools:
      allow:
        - web_search
        - web_fetch
        - message
      deny:
        - exec
        - write
        - edit
        - gateway
```

## 按发送者精细控制

```yaml
groups:
  team-chat:
    tools:
      allow: ['*']  # 默认全部允许
      
    # 按发送者覆盖
    bySender:
      # 管理员完整权限
      'id:12345':
        allow: ['*']
      
      # 特定用户限制
      'username:guest':
        allow:
          - web_search
          - message
        deny:
          - exec
          - write
      
      # 通配符：所有其他人
      '*':
        deny:
          - gateway
          - elevated
```

## 运行时策略检查

```typescript
class ToolPolicyChecker {
    private pipeline: ToolPolicyPipelineStep[];
    
    constructor(config: ToolsConfig) {
        this.pipeline = buildDefaultToolPolicyPipelineSteps({
            globalPolicy: config,
            // ... 其他策略层
        });
    }
    
    // 获取当前上下文可用的工具
    getAvailableTools(
        allTools: AnyAgentTool[],
        context: {
            agentId?: string;
            senderId?: string;
            groupId?: string;
            provider?: string;
        }
    ): AnyAgentTool[] {
        // 构建上下文相关的策略管道
        const contextPipeline = this.buildContextPipeline(context);
        
        return applyToolPolicyPipeline({
            tools: allTools,
            toolMeta: (t) => ({ pluginId: 'core' }),
            warn: console.warn,
            steps: contextPipeline
        });
    }
    
    // 检查特定工具是否可用
    isToolAllowed(
        toolName: string,
        context: { agentId?: string; senderId?: string; }
    ): boolean {
        const dummyTool = { name: toolName } as AnyAgentTool;
        const available = this.getAvailableTools([dummyTool], context);
        return available.length > 0;
    }
}
```

## 调试策略管道

```typescript
function debugToolPolicies(
    tools: AnyAgentTool[],
    steps: ToolPolicyPipelineStep[]
): void {
    console.log(`\n=== Tool Policy Pipeline Debug ===`);
    console.log(`Initial tools: ${tools.length}`);
    
    let current = [...tools];
    
    for (const step of steps) {
        if (!step.policy) {
            console.log(`\n[${step.label}] (no policy, skipped)`);
            continue;
        }
        
        const before = current.map(t => t.name);
        current = applyPolicy(current, step.policy);
        const after = current.map(t => t.name);
        
        const removed = before.filter(t => !after.includes(t));
        
        console.log(`\n[${step.label}]`);
        console.log(`  Policy: allow=${step.policy.allow?.join(',') || '*'}`);
        console.log(`  Policy: deny=${step.policy.deny?.join(',') || 'none'}`);
        console.log(`  Result: ${before.length} → ${after.length}`);
        if (removed.length) {
            console.log(`  Removed: ${removed.join(', ')}`);
        }
    }
    
    console.log(`\n=== Final: ${current.length} tools ===`);
    console.log(current.map(t => t.name).join('\n'));
}
```

## 最佳实践

### 1. 最小权限原则
```typescript
// ❌ 错误：默认全部允许
tools: { allow: ['*'] }

// ✅ 正确：按需开放
tools: {
    profile: 'minimal',
    alsoAllow: ['specific_tool_needed']
}
```

### 2. 使用 Profile 作为基线
```typescript
// ❌ 错误：从头定义
tools: {
    allow: ['read', 'write', 'edit', 'exec', ...]
}

// ✅ 正确：基于 Profile 调整
tools: {
    profile: 'coding',
    alsoAllow: ['custom_tool'],
    deny: ['dangerous_tool']
}
```

### 3. deny 用于安全边界
```typescript
// deny 优先级最高，用于硬性安全限制
tools: {
    profile: 'full',
    deny: [
        'gateway',     // 绝对不允许修改配置
        'elevated',    // 绝对不允许提权
        'sessions_*'   // 绝对不允许跨会话
    ]
}
```

### 4. 层级分离关注点
```typescript
// 全局：安全边界
global.tools.deny = ['gateway', 'elevated'];

// Agent：功能定位
agent.tools.profile = 'coding';

// 群组：社交边界
group.tools.deny = ['exec', 'write'];

// 发送者：个人权限
sender.tools.allow = ['*']; // VIP 用户
```

## 总结

工具策略管道是 Agent 安全的核心机制：

| 层级 | 关注点 | 示例 |
|------|--------|------|
| 全局 | 安全边界 | 禁止 gateway |
| Provider | 模型能力 | GPT 禁止执行 |
| Agent | 功能定位 | 客服只能消息 |
| Profile | 预设集合 | minimal/coding |
| 群组 | 社交边界 | 公开群禁止写 |
| 发送者 | 个人权限 | VIP 全权限 |

记住：**deny 永远优先**，安全策略不可被下级覆盖。

## 下节预告

下一课我们将学习 **Agent Orchestration Patterns** - 如何编排多个 Agent 协作完成复杂任务。

---

💡 **思考题**：如果你要为一个企业级 AI 助理设计工具策略，你会怎么划分层级？哪些工具应该全局禁止？
