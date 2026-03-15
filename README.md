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
- [21-multimodal-input](lessons/21-multimodal-input.md) - Multi-Modal Input：多模态输入处理
- [22-cost-optimization](lessons/22-cost-optimization.md) - Cost Optimization：Agent 成本优化策略
- [23-structured-output](lessons/23-structured-output.md) - Structured Output：结构化输出
- [24-tool-policy-pipeline](lessons/24-tool-policy-pipeline.md) - Tool Policy Pipeline：工具策略管道
- [25-mcp-protocol](lessons/25-mcp-protocol.md) - MCP (Model Context Protocol)：Agent 工具交互的标准化协议
- [26-idempotency-retry](lessons/26-idempotency-retry.md) - Idempotency & Retry：幂等性与重试模式
- [27-dynamic-tool-loading](lessons/27-dynamic-tool-loading.md) - Dynamic Tool Loading：动态工具加载
- [28-graceful-degradation](lessons/28-graceful-degradation.md) - Graceful Degradation：Agent 优雅降级
- [29-context-injection](lessons/29-context-injection.md) - Context Injection：上下文注入
- [30-tool-timeout-cancellation](lessons/30-tool-timeout-cancellation.md) - Tool Timeout & Cancellation：工具超时与取消机制
- [31-agent-reflection](lessons/31-agent-reflection.md) - Agent Reflection：Agent 自我反思
- [32-agent-handoff](lessons/32-agent-handoff.md) - Agent Handoff：Agent 交接机制
- [33-subagents-task-dispatch](lessons/33-subagents-task-dispatch.md) - Subagents 任务分发与协作
- [34-rag-retrieval-augmented-generation](lessons/34-rag-retrieval-augmented-generation.md) - RAG：检索增强生成，让 Agent 拥有外部长期记忆
- [35-prompt-chaining](lessons/35-prompt-chaining.md) - Prompt Chaining：提示链，让 Agent 完成复杂任务
- [36-workflow-orchestration](lessons/36-workflow-orchestration.md) - Workflow Orchestration：工作流编排，让 Agent 按依赖关系协调复杂任务
- [37-long-running-tasks-checkpointing](lessons/37-long-running-tasks-checkpointing.md) - Long-Running Tasks & Checkpointing：长任务与断点续传
- [38-agent-security-authentication](lessons/38-agent-security-authentication.md) - Agent Security & Authentication：Agent 安全与认证
- [39-agent-evaluation-framework](lessons/39-agent-evaluation-framework.md) - Agent 评估框架（Evals）：让 Agent 质量可度量
- [40-agent-fsm](lessons/40-agent-fsm.md) - Agent FSM：用有限状态机让 Agent 行为可预测

### 进阶篇
- [41-tool-composition-patterns](lessons/41-tool-composition-patterns.md) - Tool Composition Patterns：工具组合模式，Pipeline/Fan-out/Reduce/条件分支/重试循环
- [42-output-validation-self-correction](lessons/42-output-validation-self-correction.md) - Output Validation & Self-Correction：输出校验与自我纠错
- [43-planning-task-decomposition](lessons/43-planning-task-decomposition.md) - Planning & Task Decomposition：任务规划与分解，ReAct/Plan-and-Execute/自适应规划

### 安全篇
- [44-prompt-injection-defense](lessons/44-prompt-injection-defense.md) - Prompt Injection 攻击与防御：保护 Agent 不被外部内容劫持

### 实战篇
- [45-sandbox-code-execution](lessons/45-sandbox-code-execution.md) - Agent Sandbox 代码执行：让 Agent 安全运行代码
- [46-production-deployment-scaling](lessons/46-production-deployment-scaling.md) - Agent 生产部署与水平扩容：从本地跑通到生产可用
- [47-hot-reload-live-config](lessons/47-hot-reload-live-config.md) - Agent 热更新与动态配置：不重启就能更新 Agent 行为
- [48-multi-tenant-agent-architecture](lessons/48-multi-tenant-agent-architecture.md) - 多租户 Agent 架构：数据隔离、权限控制、资源公平调度
- [49-agent-to-agent-protocol](lessons/49-agent-to-agent-protocol.md) - Agent-to-Agent (A2A) 协议：让不同 Agent 框架互联互通
- [50-context-distillation](lessons/50-context-distillation.md) - Context Distillation：上下文蒸馏，用 LLM 把长对话提炼成结构化知识
- [51-agent-versioning-ab-testing](lessons/51-agent-versioning-ab-testing.md) - Agent Versioning & A/B Testing：行为版本管理与灰度测试，像部署代码一样部署 Agent 行为
- [52-intent-recognition-routing](lessons/52-intent-recognition-routing.md) - Intent Recognition & Routing：意图识别与路由，让 Agent 听懂"你想干什么"并精准分发
- [53-agent-memory-types](lessons/53-agent-memory-types.md) - Agent 记忆类型：Episodic / Semantic / Procedural 三种记忆的实现与应用
- [54-semantic-caching](lessons/54-semantic-caching.md) - 语义缓存：用向量相似度降低 API 调用成本
- [55-agent-persona-session-identity](lessons/55-agent-persona-session-identity.md) - Agent Persona & Session Identity：角色持久化与身份一致性，SOUL.md 机制与跨平台人格统一
- [56-circuit-breaker-pattern](lessons/56-circuit-breaker-pattern.md) - Agent Circuit Breaker Pattern：熔断器模式，工具持续失败时的快速失败与优雅降级
- [57-behavior-trees](lessons/57-behavior-trees.md) - Agent Behavior Trees（行为树）：比 FSM 更灵活、更可复用的 Agent 行为建模方式
- [58-event-sourcing-agent-state](lessons/58-event-sourcing-agent-state.md) - Event Sourcing for Agent State（事件溯源）：让 Agent 的每一步都可回溯、可重放、可审计

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
