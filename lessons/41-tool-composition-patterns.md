# 41 - Tool Composition Patterns：工具组合模式

> **核心思想**：Agent 的真正威力不在于有多少工具，而在于如何把简单工具**组合**成复杂能力。

---

## 什么是工具组合？

单个工具能力有限。但 `web_search` + `web_fetch` + `summarize` 三个工具组合起来，就能做到**「搜索 → 抓取 → 总结」**的完整研究流程。

工具组合模式（Tool Composition Patterns）就是把这种组合固化成**可复用的结构**。

---

## 五大核心模式

### 1. Pipeline（顺序管道）

最简单的组合：A → B → C，上一步的输出是下一步的输入。

```python
# learn-claude-code 风格（Python）
async def research_pipeline(query: str) -> str:
    # Step 1: 搜索
    search_results = await tool_call("web_search", {"query": query})
    
    # Step 2: 抓取第一个结果
    url = extract_first_url(search_results)
    content = await tool_call("web_fetch", {"url": url})
    
    # Step 3: 总结
    summary = await tool_call("summarize", {"text": content, "max_words": 200})
    
    return summary
```

**OpenClaw 中的例子**：`memory_search` → `memory_get` → 注入上下文，就是一个典型 Pipeline。

---

### 2. Fan-out（扇出并行）

同一个输入，**并行**发给多个工具，然后等所有结果。

```typescript
// pi-mono 风格（TypeScript）
async function parallelResearch(topics: string[]): Promise<ResearchResult[]> {
  // 并发搜索多个主题
  const searches = await Promise.all(
    topics.map(topic => 
      callTool("web_search", { query: topic })
    )
  );
  
  // 并发抓取所有结果
  const contents = await Promise.all(
    searches.map(result => 
      callTool("web_fetch", { url: result.topUrl })
    )
  );
  
  return contents;
}
```

**OpenClaw 中的例子**：当你让 Claude 同时查天气、查日历、读邮件，它会把这三个工具调用放在同一个 `<function_calls>` block 里，就是 Fan-out。

---

### 3. Reduce（聚合归并）

Fan-out 的配套模式：把多个工具结果**合并**成一个输出。

```python
async def aggregate_news(sources: list[str]) -> str:
    # Fan-out: 并行抓取多个新闻源
    articles = await asyncio.gather(*[
        fetch_article(url) for url in sources
    ])
    
    # Reduce: 聚合成一份摘要
    combined = "\n\n---\n\n".join(articles)
    return await summarize(combined, style="bullet_points")
```

**关键**：Reduce 阶段通常需要一次 LLM 调用来做「理解性聚合」，而不只是字符串拼接。

---

### 4. Conditional Branch（条件分支）

根据第一个工具的结果，决定下一步调用哪个工具。

```typescript
// OpenClaw / pi-mono 模式
async function smartFileHandler(path: string) {
  // 先检查文件类型
  const fileInfo = await callTool("read_file_metadata", { path });
  
  if (fileInfo.type === "pdf") {
    return await callTool("pdf_extract", { path });
  } else if (fileInfo.type === "image") {
    return await callTool("image_analyze", { path });
  } else {
    return await callTool("read_text", { path });
  }
}
```

**OpenClaw 中的例子**：Agent 先调用 `memory_search`，如果找到相关记忆就用 `memory_get` 读取；如果没找到，就直接用工作记忆回答。

---

### 5. Retry Loop（重试循环）

工具失败时，用不同参数或方式重试。

```python
async def resilient_search(query: str, max_retries: int = 3) -> str:
    strategies = [
        {"query": query},                           # 原始查询
        {"query": f"{query} site:github.com"},      # 限定来源
        {"query": " ".join(query.split()[:3])},     # 缩短查询
    ]
    
    for attempt, params in enumerate(strategies):
        try:
            result = await tool_call("web_search", params)
            if result and len(result) > 100:
                return result
        except ToolError as e:
            if attempt == max_retries - 1:
                raise
            await asyncio.sleep(2 ** attempt)  # 指数退避
    
    return "搜索失败，请稍后重试"
```

---

## 组合模式的元模式：Orchestrator

上面 5 种模式可以**嵌套组合**，形成复杂的工具编排。OpenClaw 的技能系统本质上就是一个元模式：

```
用户请求
    ↓
[Skill 选择] (Conditional Branch)
    ↓
[SKILL.md 加载] (Pipeline)
    ↓
[并发工具调用] (Fan-out)
    ↓
[结果聚合] (Reduce)
    ↓
[失败重试] (Retry Loop)
    ↓
最终回答
```

---

## 实战：pi-mono 的工具组合实现

pi-mono 用 TypeScript 的类型系统让工具组合**类型安全**：

```typescript
// 定义工具组合器
type ToolChain<TInput, TOutput> = {
  steps: Array<{
    tool: string;
    transform: (input: any) => any;
  }>;
  execute: (input: TInput) => Promise<TOutput>;
};

function createPipeline<T, R>(
  tools: string[],
  transforms: Array<(x: any) => any>
): ToolChain<T, R> {
  return {
    steps: tools.map((tool, i) => ({ tool, transform: transforms[i] })),
    async execute(input: T): Promise<R> {
      let result: any = input;
      for (const step of this.steps) {
        const toolInput = step.transform(result);
        result = await callTool(step.tool, toolInput);
      }
      return result as R;
    }
  };
}

// 使用
const researchChain = createPipeline<string, Summary>(
  ["web_search", "web_fetch", "summarize"],
  [
    (query) => ({ query }),
    (searchResult) => ({ url: searchResult.results[0].url }),
    (content) => ({ text: content, maxWords: 300 })
  ]
);

const summary = await researchChain.execute("Claude 4 新特性");
```

---

## 设计原则

### ✅ 正确做法

1. **工具原子化**：每个工具只做一件事，组合由 Agent 完成
2. **明确数据流**：每一步的输入输出类型要清晰
3. **在 Fan-out 中最大化并发**：能并行的绝不串行
4. **Reduce 时让 LLM 做语义聚合**：不要只做字符串拼接

### ❌ 常见错误

1. **Fat Tool 反模式**：把一个工具做成「搜索+抓取+总结」全包，失去了灵活性
2. **盲目串行**：把可以并行的工具串行执行，浪费时间
3. **忘记错误传播**：Pipeline 中一个工具失败，整个链路崩溃

---

## 总结

| 模式 | 适用场景 | 并发? |
|------|----------|-------|
| Pipeline | 有依赖的顺序步骤 | ❌ |
| Fan-out | 独立的多路并行 | ✅ |
| Reduce | 多结果聚合 | 部分 |
| Conditional Branch | 根据结果选路 | ❌ |
| Retry Loop | 容错与恢复 | ❌ |

> **一句话总结**：工具是积木，组合模式是搭法。学会这五种搭法，Agent 能力直接上一个台阶。

---

*下一节预告：Agent 输出通道适配（Output Channel Adaptation）——如何让同一个 Agent 在 Telegram、Discord、API、语音等不同渠道优雅输出。*
