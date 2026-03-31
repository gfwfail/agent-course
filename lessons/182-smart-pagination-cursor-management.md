# 182 - Agent 智能分页与游标管理（Smart Pagination & Cursor Management）

> 真实 API 从来不一次返回所有数据。Agent 如果不会翻页，就只能看到冰山一角。

---

## 为什么 Agent 需要分页管理？

大多数 API（GitHub Issues、数据库查询、文件列表、搜索结果）都有分页。Agent 面对分页有三种常见失误：

1. **只取第一页** — 以为已经"看完了"
2. **无限翻页** — 没有停止条件，耗尽 token / 钱 / 时间
3. **忽略游标** — 用 offset 翻页，数据漂移导致重复/遗漏

智能分页的目标：**自动翻页、有策略地停止、不浪费 token**。

---

## 两种分页模式对比

```
Offset-based:  ?page=2&limit=20  →  数据插入时结果漂移
Cursor-based:  ?after=abc123     →  稳定、高效、推荐
```

**Cursor 分页**是主流 API（GitHub、Stripe、Linear）的标准，Agent 必须首选。

---

## 核心设计：分页工具中间件

```typescript
// pi-mono 风格：PaginationMiddleware 透明包装任何 list 工具
interface PageResult<T> {
  items: T[];
  nextCursor?: string;
  hasMore: boolean;
  totalCount?: number;
}

interface PaginationPolicy {
  maxPages: number;         // 最多翻几页
  maxItems: number;         // 最多取多少条
  stopCondition?: (items: any[], accumulated: any[]) => boolean; // 自定义停止
  strategy: 'eager' | 'lazy'; // eager=全部取完再返回，lazy=流式返回每页
}

class PaginationMiddleware {
  async fetchAll<T>(
    fetcher: (cursor?: string) => Promise<PageResult<T>>,
    policy: PaginationPolicy
  ): Promise<T[]> {
    const accumulated: T[] = [];
    let cursor: string | undefined;
    let pageCount = 0;

    while (true) {
      const result = await fetcher(cursor);
      accumulated.push(...result.items);
      pageCount++;

      // 停止条件检查（优先级从高到低）
      if (!result.hasMore) break;
      if (pageCount >= policy.maxPages) {
        console.warn(`[Pagination] 达到 maxPages=${policy.maxPages}，停止`);
        break;
      }
      if (accumulated.length >= policy.maxItems) {
        console.warn(`[Pagination] 达到 maxItems=${policy.maxItems}，停止`);
        break;
      }
      if (policy.stopCondition?.(result.items, accumulated)) {
        console.log(`[Pagination] 自定义停止条件触发`);
        break;
      }

      cursor = result.nextCursor;
    }

    return accumulated;
  }
}
```

---

## 实战：GitHub Issues 自动翻页

```typescript
// tools/github-issues.ts
const paginationMW = new PaginationMiddleware();

async function listAllIssues(repo: string, label?: string) {
  const issues = await paginationMW.fetchAll(
    async (cursor) => {
      const resp = await octokit.graphql(`
        query($repo: String!, $after: String) {
          repository(owner: "org", name: $repo) {
            issues(first: 100, after: $after, states: OPEN) {
              nodes { number title labels { nodes { name } } }
              pageInfo { hasNextPage endCursor }
            }
          }
        }
      `, { repo, after: cursor });

      return {
        items: resp.repository.issues.nodes,
        nextCursor: resp.repository.issues.pageInfo.endCursor,
        hasMore: resp.repository.issues.pageInfo.hasNextPage,
      };
    },
    {
      maxPages: 10,       // 最多 10 页 = 1000 条
      maxItems: 500,      // 超过 500 条就停
      strategy: 'eager',
      // 找到 bug 标签就停（不用翻完所有页）
      stopCondition: (items) =>
        label ? items.some(i => i.labels.nodes.some((l: any) => l.name === label)) : false,
    }
  );

  return issues;
}
```

---

## Lazy 模式：流式返回，边翻边给 LLM

有时候不需要全部数据，LLM 找到答案就可以停了：

```typescript
async function* fetchPagesLazy<T>(
  fetcher: (cursor?: string) => Promise<PageResult<T>>,
  maxPages = 5
): AsyncGenerator<T[]> {
  let cursor: string | undefined;
  let page = 0;

  while (page < maxPages) {
    const result = await fetcher(cursor);
    yield result.items;  // 立即返回这一页，LLM 可以先处理
    page++;

    if (!result.hasMore || !result.nextCursor) break;
    cursor = result.nextCursor;
  }
}

// Agent 使用：找到目标立即停止，不浪费后续请求
for await (const page of fetchPagesLazy(githubFetcher)) {
  const found = page.find(issue => issue.title.includes('critical'));
  if (found) {
    return found; // 提前退出，后续页不再请求！
  }
}
```

---

## OpenClaw 实战：Read 工具的分页哲学

OpenClaw 的 `Read` 工具天然支持分页：

```json
{
  "tool": "Read",
  "params": {
    "file_path": "/path/to/large-file.log",
    "offset": 1,
    "limit": 100
  }
}
```

这是最简单的 offset 分页模式。工具描述中明确说：
> "output is truncated to 2000 lines or 50KB — use offset/limit for large files"

**正确的 Agent 行为**：
```
读第 1-100 行 → 没找到 → 读 101-200 行 → 找到错误日志 → 停止
```

**错误的 Agent 行为**：
```
读第 1-100 行 → 没找到 → 放弃（以为文件就这么多）
读第 1-100 行 → 没找到 → 继续读到 10000 行（浪费 token）
```

---

## Token 感知分页：动态调整每页大小

```typescript
class TokenAwarePaginator {
  private tokenBudget: number;
  private tokensUsed: number = 0;
  private readonly tokenPerItem: number = 200; // 预估每条消耗 token

  constructor(tokenBudget: number) {
    this.tokenBudget = tokenBudget;
  }

  pageSize(): number {
    const remaining = this.tokenBudget - this.tokensUsed;
    // 剩余 budget 越少，每页取的越少
    if (remaining > 10000) return 100;
    if (remaining > 5000)  return 50;
    if (remaining > 2000)  return 20;
    return 10;
  }

  recordPage(itemCount: number) {
    this.tokensUsed += itemCount * this.tokenPerItem;
  }

  hasbudget(): boolean {
    return this.tokenBudget - this.tokensUsed > this.tokenPerItem * 5;
  }
}

// 使用
const paginator = new TokenAwarePaginator(8000); // 8k token 预算给分页
while (paginator.hasBudget()) {
  const page = await fetchPage(cursor, paginator.pageSize());
  paginator.recordPage(page.items.length);
  // ... 处理
}
```

---

## 游标状态持久化（长任务必备）

大型爬取任务中途挂掉怎么办？保存游标！

```typescript
// learn-claude-code 风格：checkpoint 游标
interface CursorCheckpoint {
  toolName: string;
  params: Record<string, any>;
  lastCursor: string;
  itemsProcessed: number;
  createdAt: number;
}

class CursorStateManager {
  private store: Map<string, CursorCheckpoint> = new Map();

  save(key: string, checkpoint: CursorCheckpoint) {
    this.store.set(key, checkpoint);
    // 实际应持久化到 Redis/DB
    fs.writeFileSync(`./checkpoints/${key}.json`, JSON.stringify(checkpoint));
  }

  resume(key: string): string | undefined {
    const checkpoint = this.store.get(key) 
      ?? JSON.parse(fs.readFileSync(`./checkpoints/${key}.json`, 'utf8'));
    
    if (checkpoint) {
      console.log(`[Resume] 从游标 ${checkpoint.lastCursor} 继续，已处理 ${checkpoint.itemsProcessed} 条`);
      return checkpoint.lastCursor;
    }
    return undefined;
  }
}

// Agent 调用示例
const stateManager = new CursorStateManager();
const resumeCursor = stateManager.resume('github-issues-sync');

const allIssues = await paginationMW.fetchAll(
  async (cursor) => fetchGithubPage(cursor ?? resumeCursor),
  { maxPages: 100, maxItems: 10000, strategy: 'eager' }
);
```

---

## 工具定义：告诉 LLM 如何翻页

```typescript
// 在工具 schema 中明确声明分页能力
const listIssuesTool = {
  name: "list_issues",
  description: `列出 GitHub Issues。支持游标分页，如果 hasMore=true，
    用返回的 nextCursor 调用下一页。最多调用 5 次避免浪费。`,
  inputSchema: {
    type: "object",
    properties: {
      repo: { type: "string" },
      cursor: { 
        type: "string", 
        description: "分页游标，第一次调用不传，后续传上次返回的 nextCursor"
      },
      limit: { 
        type: "number", 
        default: 50,
        description: "每页数量，建议 20-100" 
      }
    }
  }
};
```

---

## 完整流程图

```
用户: "找出所有 open 的 P0 bug"
         ↓
LLM: 调用 list_issues(label="P0", limit=50)
         ↓
工具返回: { items: [...50条], hasMore: true, nextCursor: "abc" }
         ↓
LLM: 还有更多，继续翻页 → list_issues(cursor="abc")
         ↓
工具返回: { items: [...30条], hasMore: false }
         ↓
LLM: 全部取完，共 80 条 P0 bug，开始分析...
```

---

## 总结

| 场景 | 推荐策略 |
|------|---------|
| 搜索类（找到就停） | Lazy 模式 + 自定义 stopCondition |
| 同步类（全量更新） | Eager 模式 + 游标 checkpoint |
| Token 敏感 | TokenAwarePaginator 动态 pageSize |
| 大文件读取 | Read 工具 offset/limit |
| API 调用 | Cursor-based，避免 offset 漂移 |

**核心原则：分页要有策略，翻页要会停。**

Agent 不是爬虫，不要把所有数据都塞进 context。找到需要的，及时停止。
