# 第12课：并发工具执行 - 让 Agent 一心多用

> 当 Agent 需要调用多个独立工具时，串行执行太慢。本课讲解如何实现并发工具执行，大幅提升 Agent 响应速度。

## 为什么需要并发执行？

想象一个场景：用户问"北京和上海今天的天气怎么样？"

**串行执行**：
```
查北京天气 (2秒) → 查上海天气 (2秒) → 总共 4 秒
```

**并发执行**：
```
查北京天气 ─┐
            ├─> 总共约 2 秒
查上海天气 ─┘
```

这就是并发的威力。在实际 Agent 中，涉及多个 API 调用、文件读取、数据库查询时，并发执行能节省大量时间。

## 识别独立工具调用

并发的前提是**独立性**。两个工具调用独立，意味着：
- 后一个不依赖前一个的结果
- 没有共享状态冲突

```typescript
// ❌ 有依赖，必须串行
const user = await getUser(userId);
const posts = await getUserPosts(user.id);  // 依赖 user.id

// ✅ 无依赖，可并发
const weather = fetchWeather("Beijing");
const news = fetchNews("tech");
// 两个调用互不影响
```

Claude 在系统提示词中明确要求识别这种模式：

```
IF there are no dependencies between the calls, 
make all of the independent calls in the same block
```

## 核心实现模式

### 1. Promise.all 模式（TypeScript）

```typescript
// pi-mono 风格的并发工具执行
async function executeToolsConcurrently(
  toolCalls: ToolCall[]
): Promise<ToolResult[]> {
  // 分组：独立的 vs 有依赖的
  const { independent, dependent } = groupByDependency(toolCalls);
  
  // 并发执行独立调用
  const independentResults = await Promise.all(
    independent.map(call => executeTool(call))
  );
  
  // 串行执行有依赖的（用上一步结果）
  const allResults = [...independentResults];
  for (const call of dependent) {
    const result = await executeTool(call, allResults);
    allResults.push(result);
  }
  
  return allResults;
}
```

### 2. 依赖分析

如何判断工具调用之间有没有依赖？

```typescript
interface ToolCall {
  id: string;
  name: string;
  arguments: Record<string, any>;
  // 显式声明依赖（如果有）
  dependsOn?: string[];  // 其他 tool call 的 id
}

function groupByDependency(calls: ToolCall[]) {
  const independent: ToolCall[] = [];
  const dependent: ToolCall[] = [];
  
  for (const call of calls) {
    if (!call.dependsOn || call.dependsOn.length === 0) {
      independent.push(call);
    } else {
      dependent.push(call);
    }
  }
  
  return { independent, dependent };
}
```

### 3. 拓扑排序处理复杂依赖

当有多层依赖时，需要拓扑排序：

```typescript
function executeInOrder(calls: ToolCall[]): Promise<ToolResult[]> {
  // 构建依赖图
  const graph = buildDependencyGraph(calls);
  
  // 拓扑排序，得到执行层级
  const levels = topologicalSort(graph);
  
  // 逐层执行，每层内部并发
  const results: ToolResult[] = [];
  
  for (const level of levels) {
    const levelResults = await Promise.all(
      level.map(call => executeTool(call, results))
    );
    results.push(...levelResults);
  }
  
  return results;
}

// 例如：
// Level 0: [getUser, getConfig]  ← 并发
// Level 1: [getUserPosts]        ← 等 Level 0 完成
// Level 2: [formatPosts]         ← 等 Level 1 完成
```

## OpenClaw 的实现

OpenClaw 在处理多个工具调用时的逻辑：

```typescript
// 简化版 OpenClaw 工具执行
async function handleToolCalls(
  calls: ToolUse[],
  context: AgentContext
): Promise<ToolResult[]> {
  // 按工具类型分组
  const readCalls = calls.filter(c => c.name === 'Read');
  const execCalls = calls.filter(c => c.name === 'exec');
  const otherCalls = calls.filter(c => 
    c.name !== 'Read' && c.name !== 'exec'
  );
  
  // 读取操作天然可以并发（只读不写）
  const readResults = await Promise.all(
    readCalls.map(c => executeRead(c))
  );
  
  // exec 需要注意：某些命令可能有副作用
  // 但如果是不同目录/不同资源，也可并发
  const execResults = await Promise.all(
    execCalls.map(c => executeExec(c))
  );
  
  // 其他工具逐个处理（保守策略）
  const otherResults = [];
  for (const call of otherCalls) {
    otherResults.push(await executeTool(call, context));
  }
  
  return [...readResults, ...execResults, ...otherResults];
}
```

## 并发控制

并发不是越多越好，需要限制：

### 1. 并发数限制

```typescript
import pLimit from 'p-limit';

const limit = pLimit(5);  // 最多 5 个并发

async function executeWithLimit(calls: ToolCall[]) {
  return Promise.all(
    calls.map(call => 
      limit(() => executeTool(call))
    )
  );
}
```

### 2. 超时控制

```typescript
async function executeWithTimeout(
  call: ToolCall,
  timeoutMs: number = 30000
): Promise<ToolResult> {
  const controller = new AbortController();
  const timeoutId = setTimeout(() => controller.abort(), timeoutMs);
  
  try {
    const result = await executeTool(call, {
      signal: controller.signal
    });
    return result;
  } catch (error) {
    if (error.name === 'AbortError') {
      return {
        toolUseId: call.id,
        content: `Tool execution timed out after ${timeoutMs}ms`,
        isError: true
      };
    }
    throw error;
  } finally {
    clearTimeout(timeoutId);
  }
}
```

### 3. 错误隔离

一个工具失败不应影响其他并发工具：

```typescript
async function executeIsolated(calls: ToolCall[]): Promise<ToolResult[]> {
  const results = await Promise.allSettled(
    calls.map(call => executeTool(call))
  );
  
  return results.map((result, index) => {
    if (result.status === 'fulfilled') {
      return result.value;
    } else {
      // 失败的返回错误信息，但不中断其他
      return {
        toolUseId: calls[index].id,
        content: `Error: ${result.reason.message}`,
        isError: true
      };
    }
  });
}
```

## LLM 的并发请求模式

Claude 等 LLM 会在一次响应中返回多个工具调用：

```json
{
  "content": [
    {
      "type": "tool_use",
      "id": "toolu_01",
      "name": "Read",
      "input": { "path": "src/main.ts" }
    },
    {
      "type": "tool_use", 
      "id": "toolu_02",
      "name": "Read",
      "input": { "path": "package.json" }
    },
    {
      "type": "tool_use",
      "id": "toolu_03",
      "name": "web_search",
      "input": { "query": "TypeScript best practices" }
    }
  ]
}
```

这三个调用没有依赖关系，应该并发执行后一起返回：

```typescript
// 一次性返回所有结果给 LLM
const results = await Promise.all([
  readFile("src/main.ts"),
  readFile("package.json"),
  webSearch("TypeScript best practices")
]);

// 构造 tool_result 消息
const toolResults = results.map((result, i) => ({
  type: "tool_result",
  tool_use_id: toolCalls[i].id,
  content: result
}));
```

## 实战：批量文件操作

一个常见场景：同时读取多个文件

```typescript
// 用户问："帮我看看 src 目录下所有的 .ts 文件"

async function readMultipleFiles(paths: string[]): Promise<FileContent[]> {
  // 限制并发，避免打开太多文件
  const limit = pLimit(10);
  
  const results = await Promise.allSettled(
    paths.map(path => 
      limit(async () => ({
        path,
        content: await fs.readFile(path, 'utf-8')
      }))
    )
  );
  
  return results
    .filter((r): r is PromiseFulfilledResult<FileContent> => 
      r.status === 'fulfilled'
    )
    .map(r => r.value);
}

// 调用
const tsFiles = await glob('src/**/*.ts');
const contents = await readMultipleFiles(tsFiles);
```

## 性能对比

假设每个工具调用平均 500ms：

| 工具数量 | 串行耗时 | 并发耗时 | 提升 |
|---------|---------|---------|------|
| 2 | 1000ms | 500ms | 2x |
| 5 | 2500ms | 500ms | 5x |
| 10 | 5000ms | 500ms | 10x |

当然，这假设网络和服务端没有瓶颈。实际中会有递减效应。

## 最佳实践

### DO ✅

1. **读取操作尽量并发** - 只读操作天然安全
2. **设置合理的并发上限** - 通常 5-10 个
3. **使用 Promise.allSettled** - 隔离单个失败
4. **添加超时机制** - 防止单个调用卡死整体

### DON'T ❌

1. **有副作用的操作盲目并发** - 可能导致竞态条件
2. **无限并发** - 会压垮下游服务
3. **忽略错误处理** - Promise.all 会因单个失败全部失败
4. **复杂依赖不做排序** - 会导致数据不一致

## 总结

| 概念 | 说明 |
|-----|------|
| 独立性判断 | 后者不依赖前者结果 |
| Promise.all | 并发执行，全部完成后返回 |
| Promise.allSettled | 并发执行，单个失败不影响其他 |
| 并发限制 | 使用 p-limit 等库控制并发数 |
| 拓扑排序 | 处理复杂依赖关系 |

**下一课预告**：Caching Strategies - 如何缓存 LLM 响应和工具结果，进一步提升性能。

---

## 练习

1. 实现一个支持依赖分析的工具执行器
2. 添加并发数限制和超时控制
3. 用 Promise.allSettled 处理部分失败的情况
4. 测量并发 vs 串行的实际性能差异
