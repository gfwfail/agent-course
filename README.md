# Agent 开发课程

从零构建 AI Agent 的系统化课程。

## 课程目录

### 基础篇
- [01-streaming-output](lessons/01-streaming-output.md) - 流式输出：Agent 实时响应的秘密
- [02-memory-system](lessons/02-memory-system.md) - 记忆系统：让 Agent 拥有长期记忆
- [03-error-handling](lessons/03-error-handling.md) - 错误处理与恢复机制
- [04-tool-results-processing](lessons/04-tool-results-processing.md) - 工具结果处理：Agent 如何理解工具的回答
- [05-model-routing](lessons/05-model-routing.md) - 模型路由：智能选择最优 LLM
- [06-tool-schema-design](lessons/06-tool-schema-design.md) - 工具 Schema 设计：让 LLM 正确调用工具
- [07-context-window-management](lessons/07-context-window-management.md) - Context Window 管理：在有限空间内高效工作
- [08-safety-guardrails](lessons/08-safety-guardrails.md) - Safety & Guardrails：让 Agent 有能力但不搞破坏
- [09-prompt-engineering](lessons/09-prompt-engineering.md) - Prompt Engineering：系统提示词设计
- [10-session-management](lessons/10-session-management.md) - Session Management：会话管理
- [11-observability](lessons/11-observability.md) - Observability：可观测性
- [12-concurrent-tool-execution](lessons/12-concurrent-tool-execution.md) - 并发工具执行：让 Agent 一心多用
- [13-state-persistence](lessons/13-state-persistence.md) - State Persistence：状态持久化
- [14-agent-testing](lessons/14-agent-testing.md) - Agent Testing：如何测试 AI Agent
- [15-rate-limiting-backoff](lessons/15-rate-limiting-backoff.md) - Rate Limiting & Backoff：限流与退避策略
- [16-human-in-the-loop](lessons/16-human-in-the-loop.md) - Human-in-the-Loop：人机协作模式
- [17-tool-validation-type-safety](lessons/17-tool-validation-type-safety.md) - Tool Validation & Type Safety：工具参数验证与类型安全
- [18-agent-debugging](lessons/18-agent-debugging.md) - Agent Debugging & Troubleshooting：Agent 调试与排错
- [19-caching-strategies](lessons/19-caching-strategies.md) - Caching Strategies：Agent 缓存策略
- [20-event-driven-architecture](lessons/20-event-driven-architecture.md) - Event-Driven Architecture：事件驱动架构

### 进阶篇
- (更新中...)

### 实战篇
- (更新中...)

## 资源

- [learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) - Python 教学实现
- [pi-mono](https://github.com/badlogic/pi-mono) - TypeScript 生产实现
- [OpenClaw](https://github.com/openclaw/openclaw) - Always-on Agent 平台

## 关于

每 3 小时更新，涵盖 Agent 开发的核心知识点。

## 已讲内容 (Telegram 群组)

之前的 12 课基础内容已在群里讲完，本 repo 记录进阶内容：
- Agent Loop 核心循环
- Tool Dispatch 工具分发
- TodoWrite 计划能力
- Subagents 子代理
- Skills 技能加载
- Context Compact 上下文压缩
- Task System 任务系统
- Background Tasks 后台任务
- Agent Teams 多 Agent
- Team Protocols 协议
- Autonomous Agents 自主工作
- Worktree Isolation 隔离执行
- pi-mono 架构解析
- OpenClaw + pi SDK 集成
- Lovable 原理
