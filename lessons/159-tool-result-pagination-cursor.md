# 159 - Agent 工具结果分页与游标

> **Agent Tool Result Pagination & Cursor**  
> 大型数据集的分页策略、游标设计、上下文预算感知加载

---

## 为什么需要分页？

Agent 调用 `search_emails()`，返回 2000 封邮件——直接塞进上下文？Context 爆炸。

工具结果分页解决三个核心问题：

1. **Context 预算溢出** — 大结果集超过 LLM 窗口
2. **信息过载** — LLM 难以从海量数据中聚焦关键信息
3. **延迟与成本** — 全量拉取慢、贵、不必要

正确姿势：**工具返回 cursor，Agent 按需翻页**。

---

## 核心架构

```
┌─────────────────────────────────────────────────────┐
│                   Agent Loop                        │
│                                                     │
│  call tool → get page + cursor → decide: more?      │
│                    ↓                                │
│            [context budget check]                   │
│                    ↓                                │
│          yes: call tool(cursor=...)                 │
│          no:  summarize & stop                      │
└─────────────────────────────────────────────────────┘
```

---

## 工具设计：标准分页 Schema

```typescript
// 分页工具的标准输入/输出结构
interface PaginatedInput {
  cursor?: string;   // undefined = 第一页
  limit?: number;    // 每页大小，默认 20
  filter?: Record<string, unknown>;
}

interface PaginatedOutput<T> {
  items: T[];
  cursor: string | null;     // null = 最后一页
  total?: number;            // 可选，总数
  hasMore: boolean;
  pageInfo: {
    currentPage: number;
    itemsReturned: number;
    contextBudgetUsed: number;  // 估算 token 消耗
  };
}
```

### 游标编码策略

```typescript
// 方案A：偏移量游标（简单但有漂移风险）
function encodeOffsetCursor(offset: number): string {
  return Buffer.from(JSON.stringify({ offset })).toString('base64');
}

// 方案B：键集游标（稳定，推荐用于实时数据）
function encodeKeyCursor(lastId: string, lastCreatedAt: string): string {
  return Buffer.from(JSON.stringify({ 
    lastId, 
    lastCreatedAt,
    v: 1  // 版本号，便于迁移
  })).toString('base64');
}

function decodeCursor(cursor: string): { lastId: string; lastCreatedAt: string } {
  return JSON.parse(Buffer.from(cursor, 'base64').toString('utf-8'));
}
```

键集游标（keyset/cursor-based pagination）比 OFFSET 更稳定：

```sql
-- OFFSET 方式（有问题：数据插入时会漂移）
SELECT * FROM emails ORDER BY created_at DESC LIMIT 20 OFFSET 40;

-- 键集方式（稳定：基于上一页最后一条记录）
SELECT * FROM emails 
WHERE (created_at, id) < (:lastCreatedAt, :lastId)
ORDER BY created_at DESC, id DESC 
LIMIT 20;
```

---

## 实战：邮件搜索 Agent 工具

```typescript
// tools/search-emails.ts
import { db } from '../db';
import { encodeKeyCursor, decodeCursor } from '../cursor';
import { estimateTokens } from '../utils/token-counter';

const SEARCH_EMAILS_SCHEMA = {
  name: 'search_emails',
  description: '搜索邮件。返回分页结果，使用 cursor 获取下一页。',
  input_schema: {
    type: 'object',
    properties: {
      query: { type: 'string', description: '搜索关键词' },
      cursor: { 
        type: 'string', 
        description: '分页游标，来自上一次调用的 cursor 字段。首次调用不传。' 
      },
      limit: { 
        type: 'number', 
        description: '每页条数，默认10，最大50',
        default: 10
      }
    },
    required: ['query']
  }
};

async function searchEmails(input: {
  query: string;
  cursor?: string;
  limit?: number;
}) {
  const limit = Math.min(input.limit ?? 10, 50);
  
  // 解码游标
  let whereClause = '';
  let params: unknown[] = [input.query, limit];
  
  if (input.cursor) {
    const { lastId, lastCreatedAt } = decodeCursor(input.cursor);
    whereClause = 'AND (created_at, id) < (?, ?)';
    params = [input.query, lastCreatedAt, lastId, limit];
  }

  const emails = await db.query(`
    SELECT id, subject, sender, preview, created_at
    FROM emails
    WHERE MATCH(subject, body) AGAINST(? IN BOOLEAN MODE)
    ${whereClause}
    ORDER BY created_at DESC, id DESC
    LIMIT ?
  `, params);

  // 生成下一页游标
  const nextCursor = emails.length === limit
    ? encodeKeyCursor(
        emails[emails.length - 1].id,
        emails[emails.length - 1].created_at
      )
    : null;

  // 估算 token 消耗，帮助 LLM 决策是否继续翻页
  const tokenEstimate = estimateTokens(JSON.stringify(emails));

  return {
    items: emails,
    cursor: nextCursor,
    hasMore: nextCursor !== null,
    pageInfo: {
      itemsReturned: emails.length,
      estimatedTokens: tokenEstimate,
      hint: nextCursor 
        ? `还有更多结果。如需继续，用 cursor="${nextCursor}" 调用。如已找到目标，无需翻页。`
        : '已是最后一页。'
    }
  };
}
```

---

## Agent Loop 中的翻页决策

关键在于 **LLM 自己决定是否翻页**，而不是强制遍历所有页。

```typescript
// agent-loop.ts 中的分页感知逻辑
const PAGINATION_SYSTEM_HINT = `
当工具返回分页结果时（包含 cursor 和 hasMore 字段）：
1. 先评估当前页是否已满足用户需求
2. 如果已找到目标信息 → 直接回答，不要继续翻页
3. 如果需要更多数据 AND estimatedTokens < 50000 → 使用 cursor 翻下一页
4. 如果 estimatedTokens 接近上限 → 总结当前结果，告知用户数据量太大
5. 最多翻 5 页，避免无限循环
`;
```

### 上下文预算感知分页

```typescript
// 在 Agent Loop 中追踪已用 token
class PaginationBudgetTracker {
  private usedTokens = 0;
  private readonly budget: number;
  private pageCount = 0;
  private readonly maxPages: number;

  constructor(contextBudget = 80000, maxPages = 5) {
    this.budget = contextBudget;
    this.maxPages = maxPages;
  }

  canFetchMore(estimatedNextPageTokens: number): boolean {
    return (
      this.usedTokens + estimatedNextPageTokens < this.budget &&
      this.pageCount < this.maxPages
    );
  }

  recordFetch(tokens: number) {
    this.usedTokens += tokens;
    this.pageCount++;
  }

  getSummary() {
    return {
      pagesLoaded: this.pageCount,
      tokensUsed: this.usedTokens,
      budgetRemaining: this.budget - this.usedTokens,
      budgetExhausted: this.usedTokens >= this.budget
    };
  }
}

// 在工具执行后注入给 LLM 的元信息
function wrapPaginatedResult(result: PaginatedOutput<unknown>, tracker: PaginationBudgetTracker) {
  tracker.recordFetch(result.pageInfo.contextBudgetUsed ?? 0);
  
  return {
    ...result,
    _meta: {
      ...tracker.getSummary(),
      recommendation: tracker.canFetchMore(2000)
        ? (result.hasMore ? 'more_available' : 'complete')
        : 'budget_limit_approaching'
    }
  };
}
```

---

## OpenClaw 实战：Heartbeat 中的分页工具

OpenClaw 的 Heartbeat 检查邮件时就是个典型的分页场景：

```typescript
// openclaw/tools/gmail-tool.ts（简化示意）
export const gmailSearchTool = {
  name: 'gmail_search',
  async execute({ query, cursor, limit = 10 }) {
    const gog = await getGogClient();
    
    // gog CLI 支持 --page-token 参数
    const args = ['gmail', 'messages', 'list', 
      '--query', query,
      '--max-results', String(limit)
    ];
    if (cursor) args.push('--page-token', cursor);
    
    const result = await gog.run(args);
    
    return {
      items: result.messages ?? [],
      cursor: result.nextPageToken ?? null,   // Gmail API 原生支持
      hasMore: !!result.nextPageToken,
      pageInfo: {
        itemsReturned: result.messages?.length ?? 0,
        hint: result.nextPageToken
          ? `Gmail 还有更多邮件，pageToken="${result.nextPageToken}"`
          : '已是最后一页'
      }
    };
  }
};
```

---

## pi-mono 中的流式分页模式

pi-mono 的实现更进一步：**流式分页**——边翻页边向用户输出，不等全部加载完。

```typescript
// pi-mono 风格的流式分页（伪代码）
async function* streamPaginatedResults<T>(
  fetcher: (cursor?: string) => Promise<PaginatedOutput<T>>,
  options: { maxPages?: number; tokenBudget?: number } = {}
) {
  const { maxPages = 5, tokenBudget = 60000 } = options;
  let cursor: string | undefined;
  let page = 0;
  let tokensUsed = 0;

  while (page < maxPages) {
    const result = await fetcher(cursor);
    tokensUsed += estimateTokens(JSON.stringify(result.items));
    
    yield result.items;  // 🔑 立即 yield，不等后续页
    
    if (!result.hasMore || tokensUsed > tokenBudget) {
      return;
    }
    
    cursor = result.cursor!;
    page++;
  }
}

// 使用方式
for await (const batch of streamPaginatedResults(searchEmails)) {
  // 每批数据立即处理，用户实时看到进展
  await processEmailBatch(batch);
}
```

---

## 常见坑与最佳实践

### ❌ 反模式

```typescript
// 坏：强制遍历所有页，不管 LLM 是否需要
while (hasMore) {
  const result = await fetchNextPage(cursor);
  allItems.push(...result.items);
  cursor = result.cursor;
  hasMore = result.hasMore;
}
// 把所有数据一次性给 LLM → Context 爆炸
```

### ✅ 正确姿势

```typescript
// 好：给 LLM 提供分页能力，让它自己决定何时停止
// 工具只返回一页 + cursor + hint
// LLM 根据任务需求决定是否翻页
```

### 游标安全性

```typescript
// 游标要防篡改：HMAC 签名
import { createHmac } from 'crypto';

function signCursor(data: object, secret: string): string {
  const payload = JSON.stringify(data);
  const sig = createHmac('sha256', secret).update(payload).digest('hex');
  return Buffer.from(JSON.stringify({ payload, sig })).toString('base64');
}

function verifyCursor(cursor: string, secret: string): object {
  const { payload, sig } = JSON.parse(Buffer.from(cursor, 'base64').toString());
  const expected = createHmac('sha256', secret).update(payload).digest('hex');
  if (sig !== expected) throw new Error('Invalid cursor signature');
  return JSON.parse(payload);
}
```

---

## 总结

| 问题 | 方案 |
|------|------|
| Context 溢出 | 每页限制大小 + token 预算追踪 |
| 数据漂移 | 键集游标代替 OFFSET |
| 无限翻页 | 最大页数限制 + LLM 自主决策 |
| 游标伪造 | HMAC 签名验证 |
| 用户等待 | 流式分页（yield per batch） |

**核心原则：工具提供分页能力，LLM 决定分页策略。** 不要在代码里强制遍历，让 Agent 根据任务目标智能决定"够了"。

---

> **下一课预告：** Agent 工具调用幂等键与重复提交防护（Idempotency Key & Duplicate Submission Guard）
