# 211 - Agent 工具返回值设计最佳实践（Tool Return Value Design Best Practices）

> 工具的输入设计决定 LLM 能不能"用对"工具，工具的输出设计决定 LLM 能不能"理解"结果。  
> 大多数教程只讲输入 Schema，输出设计几乎没人讲——但这才是 Agent 智能的关键。

## 核心洞察

工具的返回值不是给人看的，是给 LLM 看的。  
**LLM 读你的工具结果，就像用户读你的 API 文档一样。**

这意味着：
- 返回值太啰嗦 → Token 浪费，LLM 注意力分散
- 返回值太简单 → LLM 缺少上下文，判断出错
- 返回值结构不清晰 → LLM 幻觉式"自动补全"
- 错误信息不够具体 → LLM 不知道该怎么恢复

## 反模式（Anti-Patterns）

### ❌ 反模式 1：直接返回原始 API 响应

```typescript
// 坏的：把原始 HTTP 响应丢给 LLM
async function getUserTool(userId: string) {
  const response = await fetch(`/api/users/${userId}`);
  return await response.json(); // 🚨 可能含几十个无关字段！
}

// LLM 收到的：
// {
//   "id": "u_123", "email": "...", "created_at": "...",
//   "stripe_customer_id": "cus_xxx",  // LLM 不需要知道
//   "internal_flags": {...},           // 内部字段泄露
//   "legacy_field_deprecated": null,   // 噪音
//   ... 还有 40 个字段
// }
```

### ❌ 反模式 2：只返回布尔值或 ID

```typescript
// 坏的：成功返回 true，失败返回 false
async function sendEmailTool(to: string, subject: string, body: string) {
  const ok = await emailService.send({ to, subject, body });
  return ok; // 🚨 LLM 不知道为什么失败，无法决策下一步
}
```

### ❌ 反模式 3：错误信息不够具体

```typescript
// 坏的：
if (!user) {
  return { error: "Failed" }; // LLM 无法判断是重试还是放弃
}
```

### ❌ 反模式 4：返回裸字符串（大段文本）

```typescript
// 坏的：
async function searchDocsTool(query: string) {
  const results = await search(query);
  return results.map(r => r.content).join('\n\n'); 
  // 🚨 LLM 无法知道哪段来自哪个文档，无法引用来源
}
```

---

## 最佳实践

### ✅ 原则 1：精选字段，按需裁剪

```typescript
// 好的：只返回 LLM 需要的字段
async function getUserTool(userId: string): Promise<ToolResult<UserInfo>> {
  const user = await db.users.findById(userId);
  
  if (!user) {
    return {
      success: false,
      error: { code: "USER_NOT_FOUND", message: `用户 ${userId} 不存在` }
    };
  }
  
  // 只暴露 LLM 需要推理的字段
  return {
    success: true,
    data: {
      id: user.id,
      name: user.name,
      email: user.email,
      role: user.role,
      subscription: user.subscription_tier, // 简化版
      created_at: user.created_at,
    }
    // 🚫 不返回：stripe_id, internal_flags, password_hash 等
  };
}
```

### ✅ 原则 2：统一的结果信封（Result Envelope）

定义一个统一的返回结构，LLM 学会一次，所有工具都适用：

```typescript
// TypeScript 版
interface ToolResult<T = unknown> {
  success: boolean;
  data?: T;
  error?: {
    code: string;         // 机器可读的错误码
    message: string;      // 人类可读的错误信息
    recoverable: boolean; // LLM 是否应该重试
    suggestion?: string;  // LLM 下一步怎么做
  };
  meta?: {
    timestamp: string;    // 结果新鲜度
    source?: string;      // 数据来源
    cached?: boolean;     // 是否命中缓存
    truncated?: boolean;  // 是否被截断
  };
}
```

```python
# Python 版
from dataclasses import dataclass
from typing import Generic, TypeVar, Optional
import json

T = TypeVar('T')

@dataclass
class ToolError:
    code: str
    message: str
    recoverable: bool = True
    suggestion: Optional[str] = None

@dataclass 
class ToolMeta:
    timestamp: str
    source: Optional[str] = None
    cached: bool = False
    truncated: bool = False

@dataclass
class ToolResult(Generic[T]):
    success: bool
    data: Optional[T] = None
    error: Optional[ToolError] = None
    meta: Optional[ToolMeta] = None
    
    def to_json(self) -> str:
        return json.dumps(self.__dict__, default=str, ensure_ascii=False)
```

### ✅ 原则 3：错误信息要有行动指引

```typescript
// 好的：告诉 LLM 下一步该怎么做
async function createOrderTool(params: OrderParams): Promise<ToolResult<Order>> {
  try {
    const order = await orderService.create(params);
    return {
      success: true,
      data: { orderId: order.id, status: order.status, total: order.total },
      meta: { timestamp: new Date().toISOString(), source: 'order-service' }
    };
  } catch (err) {
    if (err.code === 'INSUFFICIENT_FUNDS') {
      return {
        success: false,
        error: {
          code: 'INSUFFICIENT_FUNDS',
          message: `账户余额不足。当前余额: ¥${err.balance}，订单金额: ¥${params.amount}`,
          recoverable: false,
          suggestion: '请通知用户充值或选择其他支付方式，不要重试此操作'
        }
      };
    }
    if (err.code === 'TIMEOUT') {
      return {
        success: false,
        error: {
          code: 'SERVICE_TIMEOUT',
          message: '订单服务暂时不可用',
          recoverable: true,
          suggestion: '可等待 5 秒后重试一次，如仍失败请告知用户稍后再试'
        }
      };
    }
    // ... 其他错误
  }
}
```

### ✅ 原则 4：搜索/列表结果带来源元信息

```typescript
// 好的：每条结果都有元信息，LLM 可以引用
async function searchDocsTool(query: string, limit = 5): Promise<ToolResult<SearchResults>> {
  const results = await vectorSearch(query, limit);
  
  return {
    success: true,
    data: {
      query,
      total_found: results.total,
      items: results.hits.map(hit => ({
        id: hit.id,
        title: hit.title,
        snippet: hit.content.slice(0, 300), // 控制长度
        url: hit.url,
        score: Math.round(hit.score * 100) / 100, // 相关度分数
        updated_at: hit.updated_at,
      })),
    },
    meta: {
      timestamp: new Date().toISOString(),
      truncated: results.hits.some(h => h.content.length > 300),
    }
  };
}
```

### ✅ 原则 5：长内容主动截断并告知

```typescript
const MAX_CONTENT_TOKENS = 2000; // 约 8000 字符

async function readFileTool(path: string): Promise<ToolResult<FileContent>> {
  const content = await fs.readFile(path, 'utf-8');
  const MAX_CHARS = 8000;
  const truncated = content.length > MAX_CHARS;
  
  return {
    success: true,
    data: {
      path,
      content: truncated ? content.slice(0, MAX_CHARS) : content,
      total_chars: content.length,
      shown_chars: Math.min(content.length, MAX_CHARS),
    },
    meta: {
      timestamp: new Date().toISOString(),
      truncated,
      // 截断时给 LLM 提示
      ...(truncated && {
        note: `文件过长（${content.length} 字符），已截断显示前 ${MAX_CHARS} 字符。如需查看后续内容，请使用 read_file_range 工具指定行范围。`
      })
    }
  };
}
```

---

## OpenClaw 实战：工具返回值对比

OpenClaw 的 `exec` 工具输出设计就是一个好例子：

```
# LLM 收到的结构化结果（非原始 stdout）
{
  "exitCode": 0,
  "stdout": "...",
  "stderr": "",
  "truncated": false,
  "durationMs": 234
}
```

如果只返回 stdout 字符串，LLM 就不知道命令是否成功（exitCode）、执行耗时（durationMs），以及是否有截断。

---

## 设计检查清单

写完一个工具的返回值，过一遍这个 checklist：

```
□ 字段是否都是 LLM 推理需要的？（去掉所有内部字段）
□ 成功路径返回值是否足够让 LLM 决定下一步？
□ 失败路径是否包含：错误码 + 可读消息 + recoverable 标记 + 建议？
□ 列表/搜索结果是否有来源/分数/元信息？
□ 长内容是否主动截断并说明如何获取完整内容？
□ 时间戳是否有（让 LLM 判断数据新鲜度）？
□ 返回的 JSON 字段名是否语义清晰（不用 d/v/f 等缩写）？
```

---

## 与相关课程的区别

| 课程 | 关注点 |
|------|--------|
| 04 - 工具结果处理管道 | 工具返回后如何变换再注入 LLM（外部管道） |
| 72 - 工具结果转换管道 | 截断/脱敏/格式化的中间件 |
| **本课** | **工具函数本身应该返回什么**（源头设计） |

好的工具返回值设计 = 源头干净，下游省力。

---

## 一句话总结

> **设计工具返回值时，始终问自己：LLM 读到这个结果，能不能做出正确决策？**  
> 如果你需要人工解读才能理解这个返回值，LLM 也会"猜"——而猜错的代价你来承担。
