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
- [93-pii-masking-privacy-guards](lessons/93-pii-masking-privacy-guards.md) - Agent PII 脱敏与隐私护栏（PII Masking & Privacy Guards）：检测邮箱/手机/API Key 等敏感数据，工具结果注入 LLM 前清洗，日志敏感字段自动 REDACTED，Sub-agent 间数据最小化传递
- [94-llm-api-gateway-pattern](lessons/94-llm-api-gateway-pattern.md) - Agent LLM API 网关模式（LLM API Gateway Pattern）：集中代理所有 LLM 调用，统一处理限流/成本追踪/语义缓存/多 Provider 路由/密钥集中管理，业务 Agent 零感知底层模型
- [95-query-rewriting-intent-augmentation](lessons/95-query-rewriting-intent-augmentation.md) - Agent 查询改写与意图增强（Query Rewriting & Intent Augmentation）：用户说的不等于用户想的；代词解析/意图扩展/HyDE/多路展开，执行前先把模糊输入翻译成精确可执行的查询
- [96-request-deduplication-coalescing](lessons/96-request-deduplication-coalescing.md) - Agent 工具调用去重与请求合并（Request Deduplication & Coalescing）：多 Sub-agent 并发同一请求时去重共享 Promise，批量请求用 DataLoader 合并成一次 IN 查询，透明中间件无需改业务逻辑
- [97-adaptive-timeout-hedged-requests](lessons/97-adaptive-timeout-hedged-requests.md) - Agent 自适应超时与对冲请求（Adaptive Timeout & Hedged Requests）：基于 P95 历史延迟动态计算超时，P50 后发对冲请求取最快响应，消灭 P99 长尾延迟，延迟降低 40-60%
- [98-plugin-architecture](lessons/98-plugin-architecture.md) - Agent 插件化架构（Plugin Architecture）：工具集动态扩展而非写死；ToolRegistry 注册表、文件系统扫描动态加载、生命周期 hooks、权限沙箱隔离、插件间事件总线通信；OpenClaw Skills 即此模式的落地实现
- [99-nl2sql-nl2api](lessons/99-nl2sql-nl2api.md) - Agent NL2SQL & NL2API（自然语言转结构化查询）：Schema 注入策略、SQL 安全白名单验证、错误自修复循环（成功率 75%→95%）、Few-Shot 提升准确率、pi-mono 工具实现参考
- [100-automatic-prompt-optimization](lessons/100-automatic-prompt-optimization.md) - 🎉 Agent 自动提示词优化（Automatic Prompt Optimization）：Few-Shot 自动筛选 / 指令自动改写（Meta-LLM）/ A/B 测试 + Thompson Sampling 晋级，让 Agent 基于执行反馈持续自我进化
- [101-conversation-repair-clarification](lessons/101-conversation-repair-clarification.md) - Agent 对话修复与澄清（Conversation Repair & Clarification）：歧义检测 + 置信度驱动执行策略，一次只问最关键的问题，澄清限流防止烦死用户
- [102-tool-result-transformation-pipeline](lessons/102-tool-result-transformation-pipeline.md) - Agent 工具结果转换管道（Tool Result Transformation Pipeline）：工具原始输出在注入 LLM 前经过标准化/截断/脱敏/增强/错误包装管道，纯函数设计、按工具类型定制配置，消灭"脏数据入 LLM"问题
- [103-tool-call-hallucination-detection](lessons/103-tool-call-hallucination-detection.md) - Agent 工具调用幻觉检测与修复（Tool Call Hallucination Detection & Recovery）：拦截层+诊断+修复三层防御，自动修复类型错误/多余参数，不可修复的结构化反馈给 LLM 重试，把 throw 变成 tool_result
- [104-tool-mocking-test-doubles](lessons/104-tool-mocking-test-doubles.md) - Agent 工具 Mock 与测试替身（Tool Mocking & Test Doubles）：Stub/Mock/Spy/Record-Replay 四种模式，ChaosWrapper 注入故障，离线 CI 零成本，测试金字塔三层策略（Unit/Integration/E2E）
- [105-self-learning-experience-accumulation](lessons/105-self-learning-experience-accumulation.md) - Agent 自主学习与经验积累（Self-Learning & Experience Accumulation）：任务完成后 LLM 提炼经验、向量化存储、下次语义检索注入 prompt，TTL 过期 + 成功率反馈闭环，让 Agent 真正"越用越好"
- [106-distributed-locking-concurrency](lessons/106-distributed-locking-concurrency.md) - Agent 分布式锁与并发控制（Distributed Locking & Concurrency Control）：多实例 Agent 同时跑时防止资源冲突；Redis SET NX PX + Lua 原子释放；乐观锁 CAS；看门狗续期；Cron 单实例保障；死锁预防
- [107-structured-logging-trace-correlation](lessons/107-structured-logging-trace-correlation.md) - Agent 结构化日志与链路关联（Structured Logging & Trace Correlation）：JSON 格式日志设计、AsyncLocalStorage 自动传播 traceId/spanId、工具调用日志中间件、Sub-agent 跨进程 Trace 继承、jq/Loki 日志聚合查询实战
- [108-health-check-auto-restart](lessons/108-health-check-auto-restart.md) - Agent 健康检查与自动重启（Health Check & Auto-Restart）：Liveness/Readiness/Startup 三种探针、进程内 Watchdog 自救、PM2/systemd 外部守卫、原子 Checkpoint 状态保存、重启后任务恢复
- [109-batch-processing-mini-batch](lessons/109-batch-processing-mini-batch.md) - Agent 批处理与微批优化（Batch Processing & Mini-Batch Optimization）：Anthropic Batch API 省 50% 成本、微批聚合器、优先级批处理队列、内容审核实战、关键调优指标
- [110-shadow-testing-canary-validation](lessons/110-shadow-testing-canary-validation.md) - Agent 影子测试与金丝雀验证（Shadow Testing & Canary Validation）：新版并行影子运行、差异对比分析、分阶段流量切换（1%→10%→50%→100%）、自动晋级/回滚、Cron 定时健康检查
- [111-schema-driven-tool-generation](lessons/111-schema-driven-tool-generation.md) - Agent Schema 驱动工具自动生成（Schema-Driven Tool Auto-Generation）：从 OpenAPI spec 自动生成工具定义 + HTTP 执行器，告别手写重复代码；路径参数/Query/RequestBody 自动解析；工具过滤策略；MCP 协议即此模式的标准化落地
- [112-cross-process-state-broadcast](lessons/112-cross-process-state-broadcast.md) - Agent 跨进程状态广播（Cross-Process State Broadcast with Pub/Sub）：Redis Pub/Sub 作为多实例 Agent 消息总线；配置热更新广播、任务完成通知、全局限流协调；幂等消费 + 忽略自发事件；Pub/Sub vs Streams vs Kafka 场景选型
- [113-streaming-thinking-progressive-output](lessons/113-streaming-thinking-progressive-output.md) - Agent 流式思维与渐进输出（Streaming Thinking & Progressive Output）：Extended Thinking Token 流式展示、流式工具参数解析、并行工具渐进结果、首 Token 时间优化、流式中断取消、Streaming Markdown 渲染；让用户边看 AI 思考边看结果
- [114-load-shedding-priority-sla](lessons/114-load-shedding-priority-sla.md) - Agent 负载卸载与优先级 SLA（Load Shedding & Priority SLA）：过载时主动丢弃低优先级任务保障核心 SLA；优先级感知队列、抢占式调度、工具中间件集成、Cron 任务可跳过设计、优雅降级响应、多层防御体系（限流→背压→卸载）
- [115-tool-call-audit-compliance-logging](lessons/115-tool-call-audit-compliance-logging.md) - Agent 工具调用审计与合规日志（Tool Call Audit & Compliance Logging）：Append-Only 审计记录、哈希链防篡改、意图捕获、权限快照、参数脱敏、合规报告生成、数据保留策略
- [116-async-tool-execution-polling](lessons/116-async-tool-execution-polling.md) - Agent 异步工具执行与结果轮询（Async Tool Execution & Result Polling）：慢工具不阻塞 Agent Loop，Job ID + 指数退避轮询 + Redis 持久化，让 Agent 并发提交多任务、空档做其他事
- [117-graceful-shutdown-task-migration](lessons/117-graceful-shutdown-task-migration.md) - Agent 优雅关机与任务迁移（Graceful Shutdown & Task Migration）：SIGTERM 三步协议（停止接单→drain→迁移），AbortController 取消传播，Redis checkpoint + re-queue，K8s terminationGracePeriod 配置实战
- [118-cost-attribution-usage-quotas](lessons/118-cost-attribution-usage-quotas.md) - Agent 成本归因与用量配额（Cost Attribution & Usage Quotas）：按用户/功能/模型多维度追踪 Token 成本，Redis 累计配额，超限拒绝 + 阈值告警，Cron 定时报表，pi-mono ModelRouter hook 实战
- [119-multi-modal-output-rich-media](lessons/119-multi-modal-output-rich-media.md) - Agent 多模态输出与富媒体响应（Multi-Modal Output & Rich Media Response）：输出类型注册表 + 平台能力矩阵，TTS/图片/文件/Canvas UI 工具设计，AUTO_DELIVER + NO_REPLY 模式，渐进式多模态交付，OpenClaw tts 工具实战
- [120-tool-least-privilege](lessons/120-tool-least-privilege.md) - Agent 工具权限最小化原则（Tool Least Privilege）：三维权限模型（角色/会话/任务），动态工具过滤器，临时权限升级 + 人工审批 + TTL 自动过期，风险等级分级，pi-mono Agent Loop 实战
- [121-confidence-aware-routing-multi-candidate](lessons/121-confidence-aware-routing-multi-candidate.md) - Agent 置信度感知路由与多候选评估（Confidence-Aware Routing & Multi-Candidate Evaluation）：意图置信度打分、三档阈值路由（直接执行/候选确认/精准澄清）、置信度校准、OpenClaw 内联按钮集成
- [122-local-llm-hybrid-routing](lessons/122-local-llm-hybrid-routing.md) - Agent 本地大模型集成与混合路由（Local LLM Integration & Hybrid Routing）：Ollama + OpenAI SDK 兼容接入、三维路由决策（PII/预算/复杂度）、本地-云端成本对比、统一调用接口，节省 80-90% API 成本
- [123-realtime-monitoring-alerting](lessons/123-realtime-monitoring-alerting.md) - Agent 实时监控与告警（Real-time Monitoring & Alerting）：四大黄金信号、Prometheus 指标注册表、工具中间件自动埋点、告警引擎（冷却期+分级）、Telegram 告警渠道、Cron 定时巡检
- [124-infinite-loop-detection-escape](lessons/124-infinite-loop-detection-escape.md) - Agent 无限循环检测与逃出机制（Infinite Loop Detection & Escape）：哈希去重/LoopBudget 硬上限/语义相似度/调用图环检测/三层逃出策略（摘要/升级/终止）、Cron 任务循环保护
- [125-command-pattern-undo-redo](lessons/125-command-pattern-undo-redo.md) - Agent 命令模式与撤销机制（Command Pattern & Undo/Redo）：工具调用封装为 Command 对象、undo/redo 双栈、宏命令事务回滚、不可逆操作标记、与 Agent 工具分发层集成
- [126-declarative-tool-annotations](lessons/126-declarative-tool-annotations.md) - Agent 声明式工具注解（Declarative Tool Annotations & Decorator Pattern）：用 TypeScript 装饰器声明权限/缓存/重试/超时/限流元数据，运行时自动应用横切关注，从注解自动生成 LLM Schema，消灭样板代码
- [127-tool-parameter-auto-completion](lessons/127-tool-parameter-auto-completion.md) - Agent 工具参数自动补全与上下文推断（Tool Parameter Auto-Completion & Contextual Inference）：从对话历史/工具调用记录/用户偏好三层推断缺失参数，置信度打分，透明提示，只问最关键的一个
- [128-protocol-adapter-pattern](lessons/128-protocol-adapter-pattern.md) - Agent 协议适配器模式（Protocol Adapter Pattern）：统一 Anthropic/OpenAI/Gemini 消息格式，Agent 代码完全 Provider 无关，改一行配置切换底层模型，Adapter 无状态双向翻译
- [129-conversation-summary-memory-sync](lessons/129-conversation-summary-memory-sync.md) - Agent 对话摘要与长期记忆同步（Conversation Summary & Long-Term Memory Sync）：三层记忆模型（工作/情景/语义）、LLM 驱动摘要提取、Cron 定期蒸馏、MEMORY.md 更新策略
- [130-lazy-context-loading](lessons/130-lazy-context-loading.md) - Agent 懒加载上下文与按需数据注入（Lazy Context Loading & On-Demand Data Injection）：上下文注册表 + 目录模式、fetch_context 工具按需拉取、分层 TTL 缓存、预测性预加载、平均节省 70% token 消耗；OpenClaw Skills 即此模式的最佳实践
- [131-multi-hop-reasoning-tool-chain](lessons/131-multi-hop-reasoning-tool-chain.md) - Agent 多跳推理与工具链编排（Multi-Hop Reasoning & Tool Chain Orchestration）：线性链/扇出并发/条件分支三种模式、跳数上限防护、中间结果压缩、Promise.all 并发化同级工具调用
- [132-structured-concurrency](lessons/132-structured-concurrency.md) - Agent 结构化并发（Structured Concurrency）：TaskScope 任务树生命周期管理、AbortSignal 取消传播、Fail-Fast/Collect-All/Race 三种策略、零资源泄漏保证、与裸 Promise.all 的工程化升级
- [133-tool-call-budget-control](lessons/133-tool-call-budget-control.md) - Agent 工具调用预算控制（Tool Call Budget Control）：次数/成本/时间三维预算、四级状态机（ok→warning→throttle→exceeded）、工具中间件透明注入、tools=[] 强制收敛、自适应预算估算
- [134-tool-dry-run-mode](lessons/134-tool-dry-run-mode.md) - Agent 工具调用 Dry-Run 模式（Tool Call Dry-Run Mode）：执行前先演练、危险操作审批、DryRunToolRegistry + 两阶段执行、contextvar 线程安全控制、Telegram inline 审批集成
- [135-adaptive-concurrency-control](lessons/135-adaptive-concurrency-control.md) - Agent 自适应并发控制（Adaptive Concurrency Control）：AIMD 算法动态调节并发度、Vegas 延迟感知算法、按工具类型分层限速、与 Agent Loop 集成、比固定并发吞吐高 58%
- [136-dependency-injection-tool-factory](lessons/136-dependency-injection-tool-factory.md) - Agent 依赖注入与工具工厂（Dependency Injection & Tool Factory）：DI 容器/工具工厂/环境隔离/测试替换，让工具像乐高一样可组合可测试
- [137-tool-versioning-backward-compatibility](lessons/137-tool-versioning-backward-compatibility.md) - Agent 工具版本管理与向后兼容（Tool Versioning & Backward Compatibility）：Schema 版本命名空间/参数自动迁移/废弃生命周期/灰度下线，工具也是 API，要像 API 一样管理
- [138-tool-call-replay-offline-debugging](lessons/138-tool-call-replay-offline-debugging.md) - Agent 工具调用回放与离线调试（Tool Call Replay & Offline Debugging）：录制真实工具调用序列、离线回放跳过 LLM、差异对比新旧版本输出、CI Golden Test、生产 Bug 本地复现，建立 Agent 离线调试基础设施
- [138-execution-recording-time-travel-debug](lessons/138-execution-recording-time-travel-debug.md) - Agent 执行录制与时间旅行调试（Execution Recording & Time-Travel Debugging）：录制每次工具调用的输入/输出/时序，支持确定性回放、步骤级 Patch、渐进重执行；生产故障复现率 100%，调试不再靠猜
- [139-tool-result-conflict-arbitration](lessons/139-tool-result-conflict-arbitration.md) - Agent 工具结果冲突检测与仲裁（Tool Result Conflict Detection & Arbitration）：多工具/多数据源返回矛盾答案时，自动检测冲突、评分可信度（权威性+时效性）、仲裁决策，分数相近时升级人工，告知 LLM 不确定性
- [140-incremental-diff-change-detection](lessons/140-incremental-diff-change-detection.md) - Agent 增量 Diff 与变更检测（Incremental Diff & Change Detection）：指纹+快照+Diff 三层架构；ChangeSet 只传变化给 LLM；游标增量拉取替代全量扫描；适用于监控/同步/邮件/订单 Agent；OpenClaw Heartbeat 实战
- [141-memory-decay-active-forgetting](lessons/141-memory-decay-active-forgetting.md) - Agent 记忆衰减与主动遗忘（Memory Decay & Active Forgetting）：Ebbinghaus 遗忘曲线数学模型；重要性×使用频率=稳定性；间隔重复强化；LLM 二次裁判防误杀；Redis Sorted Set 管理衰减时间戳；Cron 驱动异步 GC
- [142-data-lineage-decision-provenance](lessons/142-data-lineage-decision-provenance.md) - Agent 数据血缘追踪与决策溯源（Data Lineage & Decision Provenance）：给每条数据打"出生证明"；LineageNode DAG 模型；工具调用/结果/推理/输出四类节点；祖先链溯源；Redis 持久化；Mermaid 可视化；合规报告生成
- [143-tool-call-context-propagation](lessons/143-tool-call-context-propagation.md) - Agent 工具调用上下文传播（Tool Call Context Propagation）：AsyncLocalStorage 隐式传递 userId/traceId/permissions；工具函数无需显式传参；中间件自动日志关联；Sub-agent 跨进程序列化传播；Python contextvars 实现；权限装饰器集成
- [144-chain-of-thought-verification](lessons/144-chain-of-thought-verification.md) - Agent 思维链验证与逻辑一致性检查（Chain-of-Thought Verification & Logical Consistency Check）：独立裁判 LLM 验证推理链；矛盾检测/隐含假设/逻辑跳跃三类问题；多路径一致性检查；OpenClaw 工具中间件集成；高风险工具调用前自动验证；置信度阈值分层策略
- [145-intelligent-error-classification-retry](lessons/145-intelligent-error-classification-retry.md) - Agent 智能错误分类与差异化重试（Intelligent Error Classification & Differential Retry）：六类错误分类（瞬时/限流/认证/永久/配额/逻辑）；Retry-After 精确等待；Auth Token 刷新；PERMANENT 错误携带 llmHint 让模型自修复；QUOTA 触发模型降级；TypeScript + Python 双版本实现
- [146-chaos-engineering-resilience-testing](lessons/146-chaos-engineering-resilience-testing.md) - Agent 混沌工程与弹性测试（Chaos Engineering & Resilience Testing）：ChaosToolRegistry 透明故障注入（超时/错误/脏数据/慢响应）、弹性测试套件（5大场景）、Python 中间件版本、Cron 驱动定期混沌测试、三条铁律（先爆炸/防回归/优雅降级）
- [147-timeout-budget-inheritance-cascading-cancellation](lessons/147-timeout-budget-inheritance-cascading-cancellation.md) - Agent 超时预算继承与级联取消（Timeout Budget Inheritance & Cascading Cancellation）：Deadline vs Timeout、DeadlineContext.child() 自动预算分配、AbortController 链、跨进程 deadline 传播、asyncio 版本、OpenClaw Cron 集成、三大陷阱
- [148-tool-health-probe-auto-degradation](lessons/148-tool-health-probe-auto-degradation.md) - Agent 工具健康探针与自动降级（Tool Health Probe & Auto-Degradation）：主动探针 vs 被动错误处理、HealthStatus 三态机、失败/恢复阈值、降级路由中间件、DB/API 探针实战、Python asyncio 版本、OpenClaw Cron 健康报告集成
- [149-multi-model-jury-ensemble-voting](lessons/149-multi-model-jury-ensemble-voting.md) - Agent 多模型合议制（Multi-Model Jury & Ensemble Voting）：并发派发多模型、LLM 裁判仲裁、语义投票、快速赢家三种策略、成本控制技巧、Python asyncio 版本、OpenClaw Cron 集成、适用场景分析
- [150-functional-error-handling-result-pattern](lessons/150-functional-error-handling-result-pattern.md) - Agent 函数式错误处理与 Result 模式（Functional Error Handling with Result/Either Pattern）：告别 try-catch 地狱、Ok/Err 类型设计、结构化 AgentError + llmHint、工具链组合 flatMap/andThen、并行工具 Result 聚合、Python dataclass 版本、OpenClaw 工具注册集成

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
- [151-context-aware-rate-limiting](lessons/151-context-aware-rate-limiting.md) - Agent 上下文感知速率限制（Context-Aware Rate Limiting）：动态配额 = 基准 RPS × 用户等级 × 系统负载 × 时间窗口 × 工具类型，令牌桶 + Redis Lua 原子操作，与 Backoff/Backpressure 三课形成完整限流体系
- [152-websocket-realtime-bidirectional](lessons/152-websocket-realtime-bidirectional.md) - Agent WebSocket 实时双向通信（WebSocket Real-Time Bidirectional Communication）：双向协议设计（run/cancel/input/thinking/tool/output）、Agent Loop 流式进度推送、Human-in-the-loop 挂起等待、AbortController 取消传播、React Hook 客户端实现、SSE vs WebSocket 选型、OpenClaw streamTo:"parent" 原理
- [153-tool-call-performance-profiling](lessons/153-tool-call-performance-profiling.md) - Agent 工具调用性能剖析（Tool Call Performance Profiling）：Profiling 中间件自动采集时序数据、瀑布图可视化串并行结构、关键路径算法找最长依赖链、跨会话 P95/P99 聚合统计、优化决策矩阵（缓存/并发/去重/Circuit Breaker），先剖析再优化别猜
- [154-explainability-decision-transparency](lessons/154-explainability-decision-transparency.md) - Agent 可解释性与决策透明化（Explainability & Decision Transparency）：DecisionTracker 记录工具选择上下文、置信度标注让 Agent 主动说"不确定"、分层展示（用户摘要/工程师详情/审计原始）、/explain 指令随时可查、pi-mono 中间件钩子、OpenClaw Heartbeat 决策记录
- [155-agent-api-design-best-practices](lessons/155-agent-api-design-best-practices.md) - Agent API 设计最佳实践（Agent API Design Best Practices）：SSE 流式 / 异步任务 / Webhook 三种响应模式选择，会话生命周期 API，错误码标准化，API 版本管理，202 Accepted + 轮询 + Retry-After，OpenClaw 实战参考
- [156-finetune-dataset-collection](lessons/156-finetune-dataset-collection.md) - Agent 微调数据集自动收集（Automatic Fine-tuning Dataset Collection）：从生产流量自动收集高质量对话，质量评分过滤（工具报错/用户纠正），JSONL 追加写，转换 OpenAI/Anthropic 微调格式，Cron 定期导出上传，小模型替代大模型降成本 10x
- [157-tool-dependency-injection](lessons/157-tool-dependency-injection.md) - Agent 工具依赖注入与 IoC 容器（Tool Dependency Injection & IoC Container）：工具接口与实现解耦，轻量 IoC Container（bind/resolve/生命周期），生产 vs 测试容器配置，构造函数注入，测试无需真实基础设施，装饰器语法，singleton/transient/scoped 生命周期对比，OpenClaw Skills 即 DI 的落地
- [158-agent-cicd-pipeline](lessons/158-agent-cicd-pipeline.md) - Agent CI/CD 流水线：自动化测试与持续部署（Agent CI/CD Pipeline）：五层流水线（单元/集成/Eval 回归/Staging/生产），行为漂移检测与基线对比，GitHub Actions 完整配置，控制 CI API 成本的 5 个技巧，CI 关键指标监控与告警
- [159-tool-result-pagination-cursor](lessons/159-tool-result-pagination-cursor.md) - Agent 工具结果分页与游标（Tool Result Pagination & Cursor）：键集游标 vs OFFSET、标准分页 Schema 设计、上下文预算感知翻页、LLM 自主决策是否继续翻页、流式分页 yield per batch、游标 HMAC 签名防篡改、OpenClaw Gmail 实战
- [160-tool-call-anomaly-detection](lessons/160-tool-call-anomaly-detection.md) - Agent 工具调用异常检测与行为基线（Tool Call Anomaly Detection & Behavioral Baseline）：Welford 在线算法增量建模均值/标准差、Z-Score 3σ实时偏差检测、N-gram 工具序列异常识别、异步中间件不阻塞主流程、Redis TTL 基线存储、Severity 分级告警、OpenClaw Heartbeat 巡检集成
- [161-adaptive-concurrency-control](lessons/161-adaptive-concurrency-control.md) - Agent 自适应并发控制（Adaptive Concurrency Control）：AIMD 算法（加法增加/乘法减少）动态调整工具并发上限、按工具类型隔离控制器、错误率+延迟+系统负载三维感知、与 OpenClaw 工具中间件集成、Python asyncio 版本、Grafana 关键指标、与限流/背压/并发执行三课的关系对比
- [162-tool-discovery-capability-negotiation](lessons/162-tool-discovery-capability-negotiation.md) - Agent 工具发现与能力协商（Tool Discovery & Capability Negotiation）：工具自声明可用性/约束/替代品、运行时按意图动态发现工具、双阶段可用性检查（发现+执行前）、透明降级到备选工具、OpenClaw Skills 即本地工具发现、MCP 标准化网络工具发现、LLM 工具列表动态注入替代硬编码
- [163-tool-cost-function-optimal-path](lessons/163-tool-cost-function-optimal-path.md) - Agent 工具代价函数与最优路径选择（Tool Cost Function & Optimal Path Selection）：多维代价归一化评分（延迟/金钱/副作用/资源）、工具注册时携带代价元数据、分发层透明路由对 LLM 无感知、动态学习实测延迟 P95、上下文影响权重（紧急→延迟优先/省钱→成本优先）
- [164-tool-output-noise-filtering](lessons/164-tool-output-noise-filtering.md) - Agent 工具输出噪声过滤与信号提取（Tool Output Noise Filtering & Signal Extraction）：规则提取（HTML/JSON/日志）+ LLM 语义提取双层管道、意图驱动过滤方向、相关性评分多文档筛选、Token 预算感知裁剪、实测节省 92-95% tokens
- [165-cross-session-knowledge-transfer](lessons/165-cross-session-knowledge-transfer.md) - Agent 跨会话知识传递与经验共享（Cross-Session Knowledge Transfer & Experience Sharing）：Knowledge Capsule 结构化打包、程序性/语义性/情节性三层分类、会话结束提炼+启动注入、置信度衰减+老化清理、多实例共享知识库、OpenClaw MEMORY.md 即此模式落地
- [166-tool-call-replay-deterministic-testing](lessons/166-tool-call-replay-deterministic-testing.md) - Agent 工具调用重放与确定性测试（Tool Call Replay & Deterministic Testing）：录制生产工具调用为 Cassette 文件、Replay 引擎代替真实 API 实现离线回归测试、assertAllConsumed() 防止 Agent 少做事、CI 无需网络零成本跑回归、与 PII Masking/CI-CD 流水线集成，真实数据比手写 Mock 复杂 10 倍
- [167-load-testing-capacity-planning](lessons/167-load-testing-capacity-planning.md) - Agent 压测与容量规划（Load Testing & Capacity Planning）：Agent 系统三大压测难点（有状态/延迟不可预测/Token 成本动态）、k6 脚本模拟并发会话、Locust Python 复杂场景建模、三大瓶颈识别（LLM API/工具执行/存储层）、容量规划公式、OpenClaw Cron 定时压测巡检、真实容量基线参考值
- [168-tool-call-debouncing](lessons/168-tool-call-debouncing.md) - Agent 工具调用去抖动（Tool Call Debouncing）：消灭重复触发，等安静期再执行；Debounce vs Throttle vs Dedup 三种模式对比；TypeScript/Python 双版本实现；流式 LLM 触发去抖、Webhook 批量到达去抖；Key 设计与取消处理踩坑；与限流/去重体系互补，无效 API 调用减少 70-90%

- [169-user-profiling-personalization](lessons/169-user-profiling-personalization.md) - Agent 多维度用户画像与个性化适配（User Profiling & Personalization）：从交互信号被动推断偏好（详略/专业度/领域/行为）、置信度门控注入 system prompt、Redis + TTL + 衰减存储、隐私最小化护栏（GDPR 被遗忘权）、OpenClaw MEMORY.md 人工版落地，让 Agent 越用越懂你
- [170-approval-gateway-multi-level-auth](lessons/170-approval-gateway-multi-level-auth.md) - Agent 审批网关与多级授权（Approval Gateway & Multi-Level Authorization）：危险工具调用挂起等人工审批、Risk Registry 四级风险策略、Redis 挂起状态机、waitForApproval 轮询、Telegram Inline Buttons 实时审批、CRITICAL 多级审批链、超时自动降级、中间件透明注入，Agent 自主运行但关键节点人类介入
