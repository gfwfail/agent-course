# 第三课：错误处理与恢复机制

> Agent 在真实世界运行时，错误是常态而非例外。优雅地处理错误、自动恢复，是区分玩具和生产级 Agent 的关键。

## 为什么错误处理如此重要？

Agent 运行中会遇到各种错误：
- **API 超时** - 模型服务偶尔会慢或不响应
- **工具执行失败** - shell 命令出错、文件不存在、网络请求失败
- **模型拒绝** - 安全过滤触发、格式错误
- **上下文溢出** - 对话太长超过 token 限制
- **外部依赖故障** - 数据库、第三方服务不可用

如果不处理这些错误，Agent 要么崩溃，要么卡住不动。

## 核心设计原则

### 1. 错误分类

```typescript
// pi-mono 风格的错误分类
enum AgentErrorType {
  // 可重试的错误
  RETRYABLE = 'retryable',     // API 超时、速率限制
  
  // 可恢复的错误  
  RECOVERABLE = 'recoverable', // 工具失败，但可以尝试其他方法
  
  // 致命错误
  FATAL = 'fatal',             // 认证失败、配置错误
  
  // 用户错误
  USER_ERROR = 'user_error',   // 输入无效、权限不足
}
```

### 2. 重试策略

```typescript
// 指数退避重试
async function retryWithBackoff<T>(
  fn: () => Promise<T>,
  options: {
    maxRetries: number;
    baseDelayMs: number;
    maxDelayMs: number;
  }
): Promise<T> {
  let lastError: Error;
  
  for (let i = 0; i <= options.maxRetries; i++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error as Error;
      
      // 检查是否是可重试的错误
      if (!isRetryable(error) || i === options.maxRetries) {
        throw error;
      }
      
      // 指数退避 + 随机抖动
      const delay = Math.min(
        options.baseDelayMs * Math.pow(2, i) + Math.random() * 1000,
        options.maxDelayMs
      );
      
      console.log(`Retry ${i + 1}/${options.maxRetries} after ${delay}ms`);
      await sleep(delay);
    }
  }
  
  throw lastError!;
}

// 判断错误是否可重试
function isRetryable(error: unknown): boolean {
  if (error instanceof APIError) {
    // 429 (Rate Limit) 和 5xx 可重试
    return error.status === 429 || error.status >= 500;
  }
  if (error instanceof Error) {
    // 超时和网络错误可重试
    return error.message.includes('timeout') || 
           error.message.includes('ECONNRESET');
  }
  return false;
}
```

## 实战：工具执行错误处理

### Claude Code 的做法

```python
# learn-claude-code 的简化实现
async def execute_tool(tool_name: str, params: dict) -> ToolResult:
    try:
        handler = get_tool_handler(tool_name)
        result = await handler.execute(params)
        return ToolResult(success=True, output=result)
        
    except ToolExecutionError as e:
        # 工具执行失败 - 返回错误信息让模型自己决定下一步
        return ToolResult(
            success=False,
            output=f"Tool execution failed: {e.message}",
            error_type="recoverable"
        )
        
    except PermissionError as e:
        # 权限问题 - 可能需要用户介入
        return ToolResult(
            success=False,
            output=f"Permission denied: {e}. Please grant access.",
            error_type="user_action_required"
        )
```

关键点：**把错误信息返回给模型，让它自己决定如何恢复**。

### OpenClaw 的实现

```typescript
// OpenClaw 的工具执行包装
async function executeToolSafely(
  tool: Tool,
  params: unknown,
  context: AgentContext
): Promise<ToolResult> {
  const timeout = tool.timeoutMs ?? 30000;
  
  try {
    // 带超时的执行
    const result = await Promise.race([
      tool.execute(params, context),
      sleep(timeout).then(() => {
        throw new TimeoutError(`Tool ${tool.name} timed out after ${timeout}ms`);
      })
    ]);
    
    return { success: true, data: result };
    
  } catch (error) {
    // 记录错误用于调试
    context.logger.error('Tool execution failed', {
      tool: tool.name,
      params,
      error: serializeError(error)
    });
    
    // 返回结构化的错误信息
    return {
      success: false,
      error: formatErrorForAgent(error),
      recoverable: isRecoverableToolError(error)
    };
  }
}
```

## 上下文溢出处理

当对话太长时，需要智能压缩：

```typescript
// OpenClaw 的 Context Compact 策略
async function handleContextOverflow(
  messages: Message[],
  maxTokens: number
): Promise<Message[]> {
  const currentTokens = countTokens(messages);
  
  if (currentTokens <= maxTokens) {
    return messages;
  }
  
  // 策略 1: 保留最重要的消息
  const systemMessages = messages.filter(m => m.role === 'system');
  const recentMessages = messages.slice(-10); // 保留最近 10 条
  
  // 策略 2: 压缩中间的对话历史
  const middleMessages = messages.slice(
    systemMessages.length, 
    -recentMessages.length
  );
  
  // 让模型总结中间部分
  const summary = await summarizeConversation(middleMessages);
  
  return [
    ...systemMessages,
    { role: 'system', content: `Previous conversation summary:\n${summary}` },
    ...recentMessages
  ];
}
```

## 模型拒绝处理

```typescript
// 处理模型拒绝（安全过滤、格式错误等）
async function handleModelResponse(response: ModelResponse): Promise<void> {
  // 检查是否被安全过滤
  if (response.finish_reason === 'content_filter') {
    // 记录并通知用户
    logger.warn('Response filtered by content policy');
    return sendToUser('抱歉，我无法回答这个问题。');
  }
  
  // 检查是否格式错误（如 JSON 解析失败）
  if (response.tool_use) {
    try {
      JSON.parse(response.tool_use.input);
    } catch {
      // 格式错误 - 要求模型重试
      return retryWithPrompt(
        '你的 JSON 格式有误，请重新生成正确的格式。',
        response
      );
    }
  }
}
```

## 优雅降级

当某个功能不可用时，提供替代方案：

```typescript
// 分层的服务降级
async function searchWithFallback(query: string): Promise<SearchResult> {
  // 尝试主服务
  try {
    return await primarySearchService.search(query);
  } catch (e) {
    logger.warn('Primary search failed, trying fallback', e);
  }
  
  // 尝试备用服务
  try {
    return await fallbackSearchService.search(query);
  } catch (e) {
    logger.warn('Fallback search failed, using cache', e);
  }
  
  // 最后使用缓存
  const cached = await searchCache.get(query);
  if (cached) {
    return { ...cached, stale: true };
  }
  
  // 真的不行了
  throw new ServiceUnavailableError('All search services are down');
}
```

## 错误恢复检查点

对于长时间运行的任务，保存检查点：

```typescript
// 任务检查点
interface TaskCheckpoint {
  taskId: string;
  step: number;
  state: Record<string, unknown>;
  timestamp: number;
}

async function runLongTask(task: Task): Promise<void> {
  // 尝试从检查点恢复
  const checkpoint = await loadCheckpoint(task.id);
  let currentStep = checkpoint?.step ?? 0;
  let state = checkpoint?.state ?? {};
  
  const steps = task.getSteps();
  
  for (let i = currentStep; i < steps.length; i++) {
    try {
      state = await steps[i].execute(state);
      
      // 保存检查点
      await saveCheckpoint({
        taskId: task.id,
        step: i + 1,
        state,
        timestamp: Date.now()
      });
      
    } catch (error) {
      if (isRetryable(error)) {
        // 等待后重试当前步骤
        await sleep(5000);
        i--; // 重试
        continue;
      }
      throw error;
    }
  }
  
  // 完成后清理检查点
  await deleteCheckpoint(task.id);
}
```

## 实际案例：OpenClaw 的 Agent Loop 错误处理

```typescript
// 简化的 Agent Loop
async function agentLoop(session: Session): Promise<void> {
  while (!session.shouldStop) {
    try {
      // 1. 获取模型响应
      const response = await getModelResponse(session);
      
      // 2. 处理响应
      if (response.toolCalls) {
        for (const call of response.toolCalls) {
          const result = await executeToolSafely(call);
          session.addToolResult(result);
        }
        // 继续循环让模型处理结果
        continue;
      }
      
      // 3. 发送最终响应
      await session.sendResponse(response.content);
      break;
      
    } catch (error) {
      // 错误分类处理
      if (error instanceof ContextOverflowError) {
        await session.compactContext();
        continue; // 压缩后重试
      }
      
      if (error instanceof RateLimitError) {
        await sleep(error.retryAfter * 1000);
        continue; // 等待后重试
      }
      
      if (error instanceof AuthenticationError) {
        session.fail('认证失败，请检查配置');
        break; // 致命错误，停止
      }
      
      // 未知错误 - 记录并通知用户
      logger.error('Unexpected error in agent loop', error);
      session.fail('发生意外错误，请稍后重试');
      break;
    }
  }
}
```

## 总结

| 错误类型 | 处理策略 | 示例 |
|---------|---------|------|
| 可重试 | 指数退避重试 | API 超时、速率限制 |
| 可恢复 | 返回给模型决定 | 工具执行失败 |
| 用户相关 | 提示用户操作 | 权限不足 |
| 致命 | 记录并停止 | 配置错误 |
| 资源溢出 | 智能压缩 | 上下文过长 |

核心思想：
1. **分类处理** - 不同错误用不同策略
2. **优雅降级** - 总有 Plan B
3. **检查点** - 长任务可恢复
4. **透明度** - 错误信息返回给模型，让它自己决定

下节课我们讲 **Agent 可观测性 (Observability)** - 如何追踪、调试和监控 Agent 的行为。
