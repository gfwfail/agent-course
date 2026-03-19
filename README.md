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
- [59-agent-middleware-patterns](lessons/59-agent-middleware-patterns.md) - Agent Middleware Patterns（中间件模式）：给工具调用加装拦截器，统一管理日志/鉴权/限流/重试
- [60-feature-flags-runtime-behavior](lessons/60-feature-flags-runtime-behavior.md) - Feature Flags & 动态行为开关：运行时控制 Agent 工具权限与行为，30秒内紧急关闭问题功能
- [61-task-queue-priority-scheduling](lessons/61-task-queue-priority-scheduling.md) - Task Queue & Priority Scheduling：任务队列与优先级调度，让 Agent 在多任务环境中有条不紊地处理并发请求
- [62-conversational-state-machine](lessons/62-conversational-state-machine.md) - 对话状态机与多轮上下文管理：让 Agent 在复杂多轮对话中不迷路，支持任务嵌套、插队与恢复
- [63-agent-swarm-intelligence](lessons/63-agent-swarm-intelligence.md) - Agent Swarm Intelligence（蜂群智能）：多个专业化小 Agent 协作涌现出比单一大 Agent 更强的能力
- [64-time-aware-agents](lessons/64-time-aware-agents.md) - 时间感知 Agent（Time-Aware Agents）：让 LLM 跨越"时间盲"障碍，实现时区感知、截止时间规划与时机决策
- [65-agent-workflow-dsl](lessons/65-agent-workflow-dsl.md) - Agent Workflow DSL（声明式工作流管道）：用 YAML/代码定义 Agent 执行流，实现自动并行、条件分支、可版本化的工作流
- [66-knowledge-graph-agent-memory](lessons/66-knowledge-graph-agent-memory.md) - Knowledge Graph for Agent Memory（知识图谱增强记忆）：用图结构存储实体关系，让 Agent 推理"关联"而不只是"相似"
- [67-semantic-tool-selection](lessons/67-semantic-tool-selection.md) - Semantic Tool Selection（语义工具选择）：用向量相似度从大型工具集中智能预筛，只给 LLM 最相关的 Top-K 工具
- [68-sliding-window-summarization](lessons/68-sliding-window-summarization.md) - Sliding Window + Dynamic Summarization（滑动窗口与动态摘要）：超长对话的 Token 成本控制，保留最近原文 + 异步压缩历史为摘要
- [69-backpressure-flow-control](lessons/69-backpressure-flow-control.md) - Agent 背压与流量控制（Backpressure & Flow Control）：信号量、令牌桶、有界队列、自适应背压，让 Agent 在流量峰值下优雅降级而不崩溃
- [70-trust-hierarchies-permission-delegation](lessons/70-trust-hierarchies-permission-delegation.md) - Agent Trust Hierarchies & Permission Delegation（信任层级与权限委托）：数据≠指令，子代理权限不升级，防 Prompt Injection 的来源标记法
- [71-agent-rollback-state-recovery](lessons/71-agent-rollback-state-recovery.md) - Agent Rollback & State Recovery（回滚与状态恢复）：Saga 模式处理分布式副作用，每步配补偿操作，失败时自动回滚

- [72-multi-agent-negotiation](lessons/72-multi-agent-negotiation.md) - Multi-Agent Negotiation（多 Agent 协商与共识机制）：投票/拍卖/对话协商三种模式，Agent 间无中心化协调
- [73-agent-self-healing](lessons/73-agent-self-healing.md) - Agent Self-Healing（Agent 自愈机制）：错误分类、自动诊断、修复动作注册、断点续传，让 Agent 遇错自愈而非遇错即停
- [74-contract-testing-agents](lessons/74-contract-testing-agents.md) - Agent 契约测试（Contract Testing）：用 JSON Schema 契约保障 Agent、Tool、LLM 之间接口稳定，独立迭代不互相破坏
- [75-agent-bootstrapping-self-initialization](lessons/75-agent-bootstrapping-self-initialization.md) - Agent Bootstrapping & Self-Initialization（Agent 自举与自初始化）：从 SOUL.md 到 MEMORY.md，系统化让 Agent 每次启动都"认识自己"
- [76-structured-thinking-patterns](lessons/76-structured-thinking-patterns.md) - Structured Thinking Patterns（结构化思维模式）：CoT / ToT / GoT 三种推理架构，从线性思维链到树形探索到图状网状推理
- [77-token-budget-management](lessons/77-token-budget-management.md) - Token Budget Management（Token 预算管理）：给 Agent 思考设预算，自适应分配 budget_tokens，在质量与成本间找最优平衡点
- [78-cron-driven-agent-automation](lessons/78-cron-driven-agent-automation.md) - Cron 驱动的自主 Agent 工作流：让 Agent 按计划主动执行任务，at/every/cron 三种调度模式 + 隔离子会话 + 幂等性保障
- [79-output-formatting-multiplatform](lessons/79-output-formatting-multiplatform.md) - Agent 输出格式化与多平台渲染：Telegram/Discord/CLI/Web 平台能力差异、Formatter 分层架构、结构化输出 + 渲染层解耦、长消息自动切割
- [80-cold-start-optimization](lessons/80-cold-start-optimization.md) - Agent 冷启动优化（Cold-Start Optimization）：HTTP 连接池预热、工具预加载、LLM Warmup Ping、技能文件缓存、流式首 Token 时间优化
- [81-adaptive-sampling-temperature](lessons/81-adaptive-sampling-temperature.md) - Agent 自适应采样与温度控制（Adaptive Sampling & Temperature Control）：按任务类型动态调整 temperature/top_p，意图分类路由 + 失败反馈降温重试 + 工具元数据携带采样偏好
- [82-dynamic-system-prompt](lessons/82-dynamic-system-prompt.md) - Agent 动态系统提示词（Dynamic System Prompt）：运行时按需组装 system prompt，分层注入角色定义/用户上下文/记忆片段/权限工具/渠道规则，Token 预算控制
- [83-dag-execution-engine](lessons/83-dag-execution-engine.md) - Agent DAG 执行引擎（Directed Acyclic Graph）：用有向无环图建模工作流，拓扑排序 + 并行波次执行，依赖结果自动注入，消除不必要的串行等待
- [84-prompt-caching-cache-control](lessons/84-prompt-caching-cache-control.md) - Prompt Caching（提示词缓存）：用 cache_control 把静态 system prompt / 工具列表"钉"在服务器端，缓存命中时成本降低 90%，延迟降低 85%
- [85-tool-call-prediction-prefetching](lessons/85-tool-call-prediction-prefetching.md) - Tool Call Prediction & Prefetching（工具调用预测与预取）：在 LLM 思考期间预判下一步工具并提前执行，命中时工具延迟归零；静态规则 + 转移矩阵概率预测，仅对幂等只读操作预取
- [86-context-externalization-cross-session](lessons/86-context-externalization-cross-session.md) - Context Externalization & Cross-Session Sharing（上下文外部化与跨会话共享）：把 Agent context 从进程内存搬到 Redis/DB，实现崩溃恢复、多实例共享、用户跨会话记忆；追加写 + 乐观锁 + 自动压缩
- [87-collaborative-blackboard-pattern](lessons/87-collaborative-blackboard-pattern.md) - Collaborative Blackboard Pattern（协作记忆黑板模式）：多 Agent 围绕共享黑板协作；slot 粒度设计、乐观锁写入、Pull/Push 两种控制器、OpenClaw 文件系统黑板实现
- [88-opentelemetry-distributed-tracing](lessons/88-opentelemetry-distributed-tracing.md) - OpenTelemetry 分布式追踪实战：给 Agent Turn/LLM 调用/工具执行埋 Span；W3C traceparent 跨 Sub-agent 传播；采样策略；本地 Jaeger 搭建；日志/指标/链路三合一可观测性
- [89-webhook-driven-architecture](lessons/89-webhook-driven-architecture.md) - Agent Webhook 驱动架构（Webhook-Driven Architecture）：让事件主动找 Agent 而不是轮询；验签防伪造、幂等键去重、202 快速返回；OpenClaw Cron at 模式实现 Webhook → 隔离 Sub-agent 流水线
- [90-prompt-template-engine](lessons/90-prompt-template-engine.md) - Agent Prompt Template Engine（提示词模板引擎）：Prompt 是代码，变量插值/Fragment 复用/条件渲染/版本管理/单元测试，像工程化代码一样管理 Prompt
- [91-multi-model-fallback-chain](lessons/91-multi-model-fallback-chain.md) - Agent 多模型回退链（Multi-Model Fallback Chain）：主模型挂了/限速/超预算自动切换，健康监控+冷却期+任务类型路由，用户无感知
- [92-streaming-response-aggregation](lessons/92-streaming-response-aggregation.md) - Agent 流式响应聚合（Streaming Response Aggregation）：多 Sub-agent 并行流式输出的 Fan-In 聚合；优先级队列、取消传播、Partial Error 策略；用户实时看到进展而非等最慢的那个

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
