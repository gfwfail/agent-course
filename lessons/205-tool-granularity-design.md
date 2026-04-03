# 205 - Agent 工具粒度设计：原子工具 vs 组合工具

> **核心问题**：工具应该做一件事还是多件事？拆太细 LLM 迷失，合太粗失去灵活性——找到那条线，是工具设计的真正难题。

---

## 为什么粒度决定成败

设计 Agent 工具时，最容易踩的坑不是参数校验，不是错误处理，而是**粒度**：

| 问题 | 症状 |
|------|------|
| 工具太细 | LLM 需要调用 7 个工具才能完成一件事；每步骤都要决策；容易中途出错 |
| 工具太粗 | 无法灵活组合；参数爆炸（50 个可选字段）；难以测试；复用性差 |
| 粒度刚好 | LLM 1-3 次调用搞定任务；工具语义清晰；错误隔离 |

**粒度不是技术问题，是认知负担问题**。工具列表就是给 LLM 的 API——好的 API 设计原则同样适用。

---

## 两个极端的反例

### ❌ 过度拆分（原子地狱）

```typescript
// 查一封邮件需要 6 次工具调用
tools = [
  list_mailboxes,       // 先列出邮箱
  select_mailbox,       // 选择 INBOX
  search_emails,        // 搜索邮件
  get_email_headers,    // 获取标题
  get_email_body,       // 获取正文
  get_email_attachments // 获取附件
]
```

LLM 需要维护中间状态，每一步都可能"偏航"。

### ❌ 过度合并（上帝工具）

```typescript
// 一个工具做所有事
manage_email({
  action: "list" | "read" | "send" | "delete" | "archive" | "search" | "label" | ...,
  mailbox?: string,
  email_id?: string,
  subject?: string,
  body?: string,
  to?: string[],
  cc?: string[],
  // ...30 个可选参数
})
```

Schema 巨大，LLM 经常猜错 `action`，参数组合爆炸，测试用例无穷无尽。

---

## 粒度设计的黄金法则

### 法则 1：以用户意图为单位，不以系统操作为单位

```typescript
// ❌ 以系统操作为单位（数据库 CRUD）
read_user_row(user_id)
update_user_row(user_id, fields)

// ✅ 以用户意图为单位
get_user_profile(user_id)           // 返回业务所需的聚合信息
update_user_preferences(user_id, prefs) // 语义明确的操作
```

### 法则 2：一个工具解决一个"完整的小问题"

判断标准：**这个工具独立调用，用户能理解它做了什么吗？**

```typescript
// ❌ 不完整
search_web(query)  // 只搜索，LLM 还要再调用 fetch_page 才能用结果

// ✅ 完整
search_and_extract(query, max_results = 5)
// 搜索 + 提取摘要，一次调用得到可用的信息
```

### 法则 3：组合不是工具的职责，是 Agent Loop 的职责

工具做**原子操作**，Agent Loop 负责**编排**。不要为了"省 LLM 调用次数"在工具里写业务逻辑。

```typescript
// ❌ 工具里做编排
async function send_report(user_id: string) {
  const user = await get_user(user_id)       // 工具里调工具
  const data = await fetch_report_data(user)
  await send_email(user.email, data)
}

// ✅ 工具做原子操作，Agent 编排
// LLM 调用顺序：get_user → fetch_report_data → send_email
```

---

## 实战：粒度设计的决策矩阵

```typescript
interface ToolGranularityAnalysis {
  tool: string;
  callFrequency: 'always_together' | 'sometimes_separate' | 'independent';
  semanticUnit: 'atomic' | 'natural_composite' | 'forced_composite';
  errorBoundary: 'clean' | 'messy';  // 失败时影响范围
  recommendation: 'split' | 'merge' | 'keep';
}

function analyzeGranularity(tools: Tool[]): ToolGranularityAnalysis[] {
  return tools.map(tool => {
    // 如果两个工具总是一起被调用 → 考虑合并
    // 如果一个工具有 > 5 个可选参数 → 考虑拆分
    // 如果工具名称需要用 "and" 连接 → 一定要拆
    const hasTooManyOptional = tool.schema.optional_params > 5;
    const hasAndInName = tool.name.includes('_and_');
    
    return {
      tool: tool.name,
      recommendation: hasTooManyOptional || hasAndInName ? 'split' : 'keep',
      // ...
    };
  });
}
```

### 合并的信号（应该做成一个工具）

- 两个操作总是连续发生（co-occurrence > 80%）
- 分开调用会产生无效的中间状态
- 两个操作构成**业务上的原子事务**（要么都做，要么都不做）

```typescript
// 发订单：扣库存 + 创建订单 必须原子，应该合并
create_order_with_inventory_deduction({
  user_id, items, payment_method
})
// 不要拆成 deduct_inventory + create_order（中间状态危险）
```

### 拆分的信号（应该变成多个工具）

- 工具名称里有 "and" / "or"
- 可选参数超过 5 个
- 某些参数组合根本没有意义（互斥参数）
- 测试用例数量爆炸

```typescript
// ❌ 需要拆分的工具
manage_file({
  action: 'read' | 'write' | 'delete' | 'move' | 'copy' | 'list',
  path: string,
  content?: string,   // 只在 write 时有意义
  destination?: string, // 只在 move/copy 时有意义
  recursive?: boolean,  // 只在 delete/list 时有意义
})

// ✅ 拆分后
read_file(path: string)
write_file(path: string, content: string)
delete_file(path: string, recursive?: boolean)
move_file(src: string, dst: string)
list_directory(path: string)
```

---

## OpenClaw Skills 的粒度实践

OpenClaw 的 Skills 体系本身就是一个粒度设计范本：

```typescript
// Skills = 高内聚的工具包，不是单个工具
// 每个 Skill 处理一个"领域"，域内工具可以细粒度

// weather skill
get_current_weather(location)
get_forecast(location, days)
// 两个独立工具，不合并成 get_weather(type: 'current'|'forecast')

// gog (Google Workspace) skill  
list_emails(filter)
read_email(id)
send_email(to, subject, body)
// 各司其职，LLM 按需调用
```

**Skills 的粒度原则**：
- Skill 级别 → 领域隔离（一个 Skill 一个领域）
- 工具级别 → 意图隔离（一个工具一个明确意图）

---

## pi-mono 的工具注册实践

```typescript
// pi-mono 中的工具设计：语义明确，参数最小化
import { Tool } from '@pi-mono/core';

// ✅ 好的粒度：参数少，意图清晰
const searchEmailsTool: Tool = {
  name: 'search_emails',
  description: 'Search emails by keyword in subject or body. Returns list of matching emails with id, subject, date, sender.',
  inputSchema: {
    type: 'object',
    properties: {
      query: { type: 'string', description: 'Search keyword' },
      limit: { type: 'number', description: 'Max results, default 10' }
    },
    required: ['query']
  },
  handler: async ({ query, limit = 10 }) => {
    return await emailService.search(query, limit);
  }
};

// 工具注册时检查粒度
function registerTool(tool: Tool) {
  // 警告：可选参数过多
  const optionalCount = Object.keys(tool.inputSchema.properties || {})
    .filter(k => !tool.inputSchema.required?.includes(k)).length;
  
  if (optionalCount > 5) {
    console.warn(`⚠️ Tool '${tool.name}' has ${optionalCount} optional params. Consider splitting.`);
  }
  
  // 警告：名称包含 and
  if (tool.name.includes('_and_')) {
    console.warn(`⚠️ Tool '${tool.name}' might be doing too much. Check if it should be split.`);
  }
  
  registry.register(tool);
}
```

---

## 层级工具设计模式

当需求真的很复杂时，用**两层工具**代替一个超级工具：

```typescript
// Layer 1: 原子工具（细粒度，可复用）
const atomicTools = [
  get_user(user_id),
  get_orders(user_id, status?),
  calculate_refund(order_id),
  process_refund(order_id, amount),
  send_notification(user_id, template, vars),
];

// Layer 2: 场景工具（组合原子工具，处理常见完整场景）
const sceneTools = [
  // 这个工具内部调用多个原子工具，对 LLM 暴露简单接口
  handle_refund_request(order_id: string) {
    const order = await get_orders(order_id);
    const amount = await calculate_refund(order_id);
    await process_refund(order_id, amount);
    await send_notification(order.user_id, 'refund_success', { amount });
    return { success: true, refunded: amount };
  }
];

// LLM 看到两层工具，优先选场景工具
// 场景工具不够用时，再用原子工具自由组合
```

这就是 **Facade Pattern** 在工具设计中的应用：
- 场景工具 = 高层 Facade，处理 80% 的常见情况
- 原子工具 = 底层 Primitives，处理 20% 的特殊情况

---

## 实战案例：重构"上帝工具"

**Before：一个管理所有数据库操作的上帝工具**

```typescript
// 这是真实踩过的坑
database_operation({
  table: string,
  action: 'select' | 'insert' | 'update' | 'delete' | 'count' | 'exists',
  where?: Record<string, any>,
  data?: Record<string, any>,
  fields?: string[],
  limit?: number,
  offset?: number,
  order_by?: string,
  order_dir?: 'asc' | 'desc',
  join?: { table: string, on: string }[],
  group_by?: string[],
})
// LLM 调用错误率：~35%，参数组合太复杂
```

**After：按意图拆分**

```typescript
// 统一查询接口（但参数大幅精简）
query_records(table: string, filter: SimpleFilter, options?: QueryOptions)
get_record_by_id(table: string, id: string)
count_records(table: string, filter?: SimpleFilter)
record_exists(table: string, filter: SimpleFilter)

// 写操作（明确意图，权限分开）
create_record(table: string, data: Record<string, any>)
update_record(table: string, id: string, changes: Partial<Record<string, any>>)
delete_record(table: string, id: string)

// LLM 调用错误率：~4%，每个工具语义清晰
```

---

## 粒度 Checklist

在合并或拆分工具前，过一遍这个清单：

```
合并工具前问自己：
□ 这两个操作是否构成不可分割的原子事务？
□ 分开调用是否会产生危险的中间状态？
□ 合并后的工具名称是否仍然语义清晰？
□ 合并后可选参数是否 ≤ 5 个？

拆分工具前问自己：
□ 拆分后，每个子工具是否有独立的使用价值？
□ LLM 是否需要分别调用这些子工具来完成真实任务？
□ 现在的工具是否经常被"误用"（参数填错）？
□ 工具的 description 是否需要写很长才能解释清楚？
```

---

## 总结

| 维度 | 原子工具 | 组合工具 |
|------|---------|---------|
| 灵活性 | ✅ 高 | ❌ 低 |
| LLM 调用次数 | ❌ 多 | ✅ 少 |
| 错误隔离 | ✅ 清晰 | ❌ 混乱 |
| 测试难度 | ✅ 容易 | ❌ 组合爆炸 |
| 适用场景 | 通用 Primitives | 高频业务场景 |

**最佳实践：原子工具 + 场景 Facade 两层架构**

- 底层：原子工具，覆盖所有操作
- 顶层：场景工具，覆盖 80% 的高频任务
- LLM 优先用场景工具，场景工具不够用时组合原子工具

> 工具粒度没有绝对的正确答案，但有一个可测量的指标：**LLM 在真实任务中的工具调用成功率**。如果成功率低于 90%，先检查粒度是否合理。

---

## 延伸阅读

- [Lesson 06 - Tool Schema Design](./06-tool-schema-design.md)
- [Lesson 36 - Tool Composition Patterns](./36-tool-composition-patterns.md)
- [Lesson 98 - Plugin Architecture](./98-plugin-architecture.md)
- [Lesson 176 - Tool Versioning & Backward Compatibility](./176-tool-versioning-backward-compatibility.md)
