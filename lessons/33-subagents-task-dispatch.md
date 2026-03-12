# 33-Subagents 任务分发与协作

## 什么是 Subagents？

Subagents 是 Agent 的子代理，用于**并行处理任务**、**复杂任务分解**和**专业化分工**。

OpenClaw 的 `sessions_spawn` 工具可以创建子代理：

```javascript
{
  "task": "分析用户数据并生成报告",
  "runtime": "subagent",
  "agentId": "data-analyzer-01",
  "model": "gpt-4o-mini"
}
```

## Subagents 的三种模式

### 1. `run` 模式（一次性）

```javascript
sessions_spawn({
  task: "计算本月销售额统计",
  runtime: "subagent",
  mode: "run",
  runTimeoutSeconds: 120
});
```

- 任务完成后自动退出
- 适合一次性计算任务
- 可设置超时时间

### 2. `session` 模式（持久化）

```javascript
sessions_spawn({
  task: "监控数据库连接池状态",
  runtime: "subagent",
  mode: "session",
  thread: true
});
```

- 持久运行，可长期接收指令
- 适合监控、定时任务
- 支持线程绑定（Discord/Telegram）

### 3. `acp` 模式（Code Pairing）

```javascript
sessions_spawn({
  task: "修复登录功能的 bug",
  runtime: "acp",
  agentId: "codex",
  cwd: "/app/src/auth"
});
```

- 使用 Codex/ Claude Code 等专业编码 Agent
- 支持独立工作区（`cwd`）
- 适合代码开发任务

## 实战案例：Agent 团队协作

### 场景：博客文章生成流程

```
用户请求
   ↓
Editor Agent (规划内容)
   ↓
Researcher Agent (收集资料) ──┐
   ↓                          │
 Writer Agent (撰写初稿) ←────┘
   ↓
Reviewer Agent (审阅修改)
   ↓
Published Article
```

### OpenClaw 实现代码

```javascript
// 主 Agent 启动子代理团队
function writeBlogPost({ topic, length }) {
  const editorId = sessions_spawn({
    task: `规划一篇 ${length} 字的博客文章，主题是：${topic}`,
    runtime: "subagent",
    agentId: "editor-agent"
  });

  // 分别启动研究员和写作员
  const researcherId = sessions_spawn({
    task: `收集关于 ${topic} 的最新资料和数据`,
    runtime: "subagent",
    agentId: "researcher-agent"
  });

  const writerId = sessions_spawn({
    task: `根据资料撰写初稿`,
    runtime: "subagent",
    agentId: "writer-agent"
  });

  // 等待子代理完成
  const editorResult = waitSession(editorId);
  const researchResult = waitSession(researcherId);
  const draftResult = waitSession(writerId);

  // 审阅员合并内容并润色
  const reviewerId = sessions_spawn({
    task: `整合编辑、研究和初稿内容，生成最终文章`,
    runtime: "subagent",
    agentId: "reviewer-agent"
  });

  return waitSession(reviewerId);
}
```

## OpenClaw Subagents API

```javascript
// 列出当前子代理
subagents({ action: "list" });

// 向特定子代理发送指令
subagents({
  action: "steer",
  target: "<sessionKey>",
  message: "继续完成上一个任务"
});

// 终止子代理
subagents({
  action: "kill",
  target: "<sessionKey>"
});
```

## 最佳实践

### 1. 任务分解原则

```
复杂任务
   ↓
拆分为子任务（并行）
   ↓
分配给专业化 Agent
   ↓
汇总结果
   ↓
最终输出
```

### 2. 错误处理

```javascript
try {
  const result = sessions_spawn({
    task: "分析用户数据",
    runtime: "subagent",
    timeoutSeconds: 60
  });
} catch (error) {
  // 子代理失败时的回退方案
  return fallbackAnalysis();
}
```

### 3. 状态共享

```javascript
// 子代理之间共享数据
const sharedState = { topic: "Agent 开发", level: "intermediate" };

sessions_spawn({
  task: `根据状态编写课程：${JSON.stringify(sharedState)}`,
  runtime: "subagent",
  attachments: [{
    name: "state.json",
    content: JSON.stringify(sharedState),
    encoding: "utf8"
  }]
});
```

## pi-mono 实现参考

pi-mono 使用 `AgentRuntime.spawn()` 创建子代理：

```typescript
const researcher = await runtime.spawn({
  id: "researcher",
  instructions: `收集关于 ${topic} 的最新资料`
});

const writer = await runtime.spawn({
  id: "writer",
  instructions: `根据资料撰写文章`
});

// 等待所有子代理完成
const [researchResult, draft] = await Promise.all([
  researcher.run({ input: "开始研究" }),
  writer.run({ input: "开始写作" })
]);
```

## 调试 Subagents

```bash
# 列出所有子代理
openclaw sessions list --kinds subagent

# 查看子代理历史
openclaw sessions history --sessionKey <key>

# 杀死卡住的子代理
openclaw subagents kill --target <sessionKey>
```

## 小结

Subagents 是实现 Agent 协作的关键工具：

1. **并行处理**：多个任务同时进行
2. **专业化分工**：不同 Agent 专攻不同领域
3. **任务分解**：复杂问题拆解为简单子任务
4. **资源隔离**：每个子代理独立的上下文和状态

OpenClaw 提供了完整的 Subagents 管理能力，让 Agent 协作变得简单高效！
