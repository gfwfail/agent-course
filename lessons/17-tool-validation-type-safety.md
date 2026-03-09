# 第 17 课：Tool Validation & Type Safety - 工具参数验证与类型安全

> LLM 不是程序员，它会犯错。你的工具必须能优雅地处理错误输入。

## 为什么需要验证？

LLM 调用工具时，本质上是在"猜"参数应该是什么。它会：

1. **传错类型** - 该传数字传了字符串 `"100"` 而不是 `100`
2. **漏掉必填字段** - 忘记传 `path` 参数
3. **传无效值** - status 应该是 `pending/completed`，传了 `done`
4. **格式不对** - 日期应该是 `2024-01-15`，传了 `Jan 15, 2024`

如果你的工具直接崩溃，Agent 就会陷入困境。好的验证让 Agent 能从错误中恢复。

## 验证的三层防线

```
                     ┌─────────────────┐
                     │  LLM 调用工具   │
                     └────────┬────────┘
                              │
               ┌──────────────▼──────────────┐
        层 1   │     Schema 验证            │ ← 结构正确吗？
               │   (JSON Schema / TypeBox)  │
               └──────────────┬──────────────┘
                              │
               ┌──────────────▼──────────────┐
        层 2   │     业务规则验证            │ ← 值合理吗？
               │   (自定义校验逻辑)          │
               └──────────────┬──────────────┘
                              │
               ┌──────────────▼──────────────┐
        层 3   │     运行时安全检查          │ ← 操作安全吗？
               │   (路径检查、权限检查)      │
               └──────────────┬──────────────┘
                              │
                     ┌────────▼────────┐
                     │   执行工具逻辑   │
                     └─────────────────┘
```

## 层 1：Schema 验证

### TypeScript 方案：TypeBox + AJV

pi-mono 使用 TypeBox 定义 Schema，AJV 做验证：

```typescript
// packages/ai/src/utils/validation.ts
import AjvModule from "ajv";
import addFormatsModule from "ajv-formats";

const Ajv = (AjvModule as any).default || AjvModule;
const addFormats = (addFormatsModule as any).default || addFormatsModule;

// 创建 AJV 实例
let ajv = new Ajv({
  allErrors: true,      // 收集所有错误，不是遇到第一个就停
  strict: false,        // 宽松模式，允许额外属性
  coerceTypes: true,    // 🔑 关键：自动类型转换！
});
addFormats(ajv);        // 支持 email、uri、date 等格式

export function validateToolArguments(tool: Tool, toolCall: ToolCall): any {
  // 编译 Schema
  const validate = ajv.compile(tool.parameters);
  
  // 克隆参数（AJV 会原地修改做类型强转）
  const args = structuredClone(toolCall.arguments);
  
  // 验证
  if (validate(args)) {
    return args;  // 返回强转后的参数
  }
  
  // 格式化错误信息（对 LLM 友好）
  const errors = validate.errors
    ?.map((err: any) => {
      const path = err.instancePath 
        ? err.instancePath.substring(1) 
        : err.params.missingProperty || "root";
      return `  - ${path}: ${err.message}`;
    })
    .join("\n");
  
  throw new Error(
    `Validation failed for tool "${toolCall.name}":\n${errors}\n\n` +
    `Received arguments:\n${JSON.stringify(toolCall.arguments, null, 2)}`
  );
}
```

**关键配置 `coerceTypes: true`**：

```typescript
// LLM 传了 "100"（字符串），Schema 期望 number
// coerceTypes 会自动转换成 100（数字）

// 支持的转换：
// "100"  → 100        (string → number)
// "true" → true       (string → boolean)
// 100    → "100"      (number → string)
// null   → ""         (null → string)
```

这很重要！LLM 经常把数字当字符串传，有了类型强转就不会报错。

### 定义工具 Schema

```typescript
// packages/coding-agent/src/core/tools/read.ts
import { Type } from "@sinclair/typebox";

const readSchema = Type.Object({
  path: Type.String({ 
    description: "Path to the file to read" 
  }),
  offset: Type.Optional(Type.Number({ 
    description: "Line number to start from (1-indexed)" 
  })),
  limit: Type.Optional(Type.Number({ 
    description: "Maximum lines to read" 
  })),
});

// 自动生成 TypeScript 类型
type ReadInput = Static<typeof readSchema>;
// 等价于：{ path: string; offset?: number; limit?: number }
```

TypeBox 的好处：**一处定义，双重用途**
1. 生成 JSON Schema 给 LLM 看
2. 生成 TypeScript 类型给你的代码用

### Python 方案：Pydantic

```python
from pydantic import BaseModel, Field, field_validator
from typing import Optional, Literal

class TodoItem(BaseModel):
    id: str
    text: str = Field(..., min_length=1)  # 不能为空
    status: Literal["pending", "in_progress", "completed"]
    
    @field_validator("id")
    @classmethod
    def validate_id(cls, v):
        if not v.strip():
            raise ValueError("id cannot be empty")
        return v.strip()

class TodoInput(BaseModel):
    items: list[TodoItem] = Field(..., max_length=20)
    
    @field_validator("items")
    @classmethod
    def validate_single_in_progress(cls, items):
        in_progress = [i for i in items if i.status == "in_progress"]
        if len(in_progress) > 1:
            raise ValueError("Only one task can be in_progress")
        return items

# 使用
def run_todo(raw_input: dict) -> str:
    try:
        validated = TodoInput(**raw_input)
        # validated.items 现在是类型安全的
        return process_todos(validated.items)
    except ValidationError as e:
        return f"Validation error: {e}"
```

## 层 2：业务规则验证

Schema 验证只能检查结构，业务规则需要自定义逻辑：

```python
# learn-claude-code/agents/s03_todo_write.py

class TodoManager:
    def update(self, items: list) -> str:
        # 规则 1: 最多 20 个 todo
        if len(items) > 20:
            raise ValueError("Max 20 todos allowed")
        
        validated = []
        in_progress_count = 0
        
        for i, item in enumerate(items):
            text = str(item.get("text", "")).strip()
            status = str(item.get("status", "pending")).lower()
            item_id = str(item.get("id", str(i + 1)))
            
            # 规则 2: text 不能为空
            if not text:
                raise ValueError(f"Item {item_id}: text required")
            
            # 规则 3: status 必须是有效值
            if status not in ("pending", "in_progress", "completed"):
                raise ValueError(f"Item {item_id}: invalid status '{status}'")
            
            # 规则 4: 同时只能有一个 in_progress
            if status == "in_progress":
                in_progress_count += 1
                
            validated.append({"id": item_id, "text": text, "status": status})
        
        if in_progress_count > 1:
            raise ValueError("Only one task can be in_progress at a time")
        
        self.items = validated
        return self.render()
```

**业务规则 vs Schema 规则**：

| Schema 验证 | 业务验证 |
|------------|---------|
| 类型是否正确 | 值是否在合理范围 |
| 必填字段是否存在 | 多个字段间的关系 |
| 枚举值是否有效 | 状态机转换是否合法 |
| 格式是否正确 | 权限是否足够 |

## 层 3：运行时安全检查

验证通过不代表操作安全：

```python
# 路径安全检查
def safe_path(p: str) -> Path:
    """确保路径不会逃出工作目录"""
    path = (WORKDIR / p).resolve()
    if not path.is_relative_to(WORKDIR):
        raise ValueError(f"Path escapes workspace: {p}")
    return path

# 命令安全检查
def run_bash(command: str) -> str:
    dangerous = ["rm -rf /", "sudo", "shutdown", "reboot", "> /dev/"]
    if any(d in command for d in dangerous):
        return "Error: Dangerous command blocked"
    # ... 执行命令
```

```typescript
// TypeScript 版本
function resolveReadPath(path: string, cwd: string): string {
  const absolutePath = resolve(cwd, path);
  
  // 检查路径是否在允许的目录内
  if (!absolutePath.startsWith(cwd)) {
    throw new Error(`Access denied: ${path} is outside workspace`);
  }
  
  return absolutePath;
}
```

## 错误信息设计

错误信息要对 LLM 友好，告诉它怎么修正：

### ❌ 不好的错误信息

```
Error: Invalid input
```

LLM 不知道哪里错了，只能瞎猜。

### ✅ 好的错误信息

```
Validation failed for tool "todo":
  - items[2].status: must be one of: pending, in_progress, completed
  - items[3].text: is required

Received arguments:
{
  "items": [
    {"id": "1", "text": "Task 1", "status": "pending"},
    {"id": "2", "text": "Task 2", "status": "pending"},
    {"id": "3", "text": "Task 3", "status": "done"},    // ← 这里错了
    {"id": "4", "status": "pending"}                     // ← 缺 text
  ]
}
```

LLM 看到这个，就知道：
1. `items[2].status` 应该用 `completed` 而不是 `done`
2. `items[3]` 缺少 `text` 字段

## 实战：完整的验证流程

```typescript
// 完整的工具定义 + 验证
import { Type, Static } from "@sinclair/typebox";

// 1. 定义 Schema
const editSchema = Type.Object({
  path: Type.String({ description: "Path to file" }),
  oldText: Type.String({ description: "Text to find (exact match)" }),
  newText: Type.String({ description: "Replacement text" }),
});

type EditInput = Static<typeof editSchema>;

// 2. 创建工具
function createEditTool(cwd: string): AgentTool<typeof editSchema> {
  return {
    name: "edit",
    description: "Replace exact text in file",
    parameters: editSchema,
    
    execute: async (toolCallId, input: EditInput) => {
      // 层 1: Schema 验证在调用前已完成
      
      // 层 2: 业务规则
      if (input.oldText === input.newText) {
        return { 
          content: [{ type: "text", text: "Warning: oldText equals newText, no change made" }]
        };
      }
      
      // 层 3: 安全检查
      const absolutePath = resolveReadPath(input.path, cwd);
      
      // 执行操作
      const content = await readFile(absolutePath, "utf-8");
      
      // 层 2: 更多业务规则
      if (!content.includes(input.oldText)) {
        const preview = content.substring(0, 500);
        return {
          content: [{
            type: "text",
            text: `Error: oldText not found in file.\n\nFile preview:\n${preview}...`
          }]
        };
      }
      
      const newContent = content.replace(input.oldText, input.newText);
      await writeFile(absolutePath, newContent);
      
      return {
        content: [{ type: "text", text: `Edited ${input.path}` }]
      };
    }
  };
}
```

## 类型强转策略

不同框架有不同的强转策略：

### 宽松策略（推荐用于 LLM）

```typescript
const ajv = new Ajv({
  coerceTypes: true,   // 自动转换类型
  removeAdditional: true,  // 删除未定义的属性
  useDefaults: true,   // 自动填充默认值
});
```

### 严格策略（用于安全敏感场景）

```typescript
const ajv = new Ajv({
  coerceTypes: false,  // 不转换，类型必须精确匹配
  removeAdditional: false,  // 保留额外属性
  strict: true,
});
```

## 常见陷阱

### 1. 可选参数的默认值

```typescript
// ❌ 错误：undefined 和 missing 不一样
const readSchema = Type.Object({
  limit: Type.Optional(Type.Number()),
});
// input: {} → limit 是 undefined
// input: { limit: undefined } → 也是 undefined

// ✅ 正确：显式设置默认值
const readSchema = Type.Object({
  limit: Type.Optional(Type.Number({ default: 100 })),
});
```

### 2. 数组可能是空的

```typescript
// ❌ 假设数组总有元素
function processItems(input: { items: string[] }) {
  const first = input.items[0];  // 可能 undefined！
}

// ✅ 检查数组长度
const itemsSchema = Type.Array(Type.String(), { minItems: 1 });
```

### 3. 字符串可能是空的

```python
# ❌ 只检查类型
def read_file(path: str):
    return open(path).read()  # path="" 会报错

# ✅ 检查非空
class ReadInput(BaseModel):
    path: str = Field(..., min_length=1)
```

## 总结

1. **三层防线**：Schema → 业务规则 → 运行时安全
2. **开启类型强转**：LLM 经常传错类型，`coerceTypes: true` 能救你
3. **错误信息要详细**：告诉 LLM 哪里错了、怎么改
4. **TypeBox/Pydantic**：一处定义，双重用途（Schema + 类型）
5. **永远不信任输入**：即使 LLM 说它会传正确的参数

验证是 Agent 稳定性的基石。花时间在验证上，能省下调试的时间。

---

下节课我们讲 **Configuration Management**（配置管理）—— 如何让 Agent 灵活适应不同环境。
