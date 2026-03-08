# 第六课：工具 Schema 设计

> 工具定义的好坏，直接决定了 Agent 的智商上限。

## 为什么这课重要？

你可能觉得工具就是写个函数、加个描述，没什么技术含量。错了。

**工具 Schema 是 Agent 和外部世界的接口协议**。设计不好的工具会让 LLM：
- 传错参数
- 调用时机不对
- 产生幻觉（以为工具能做实际做不到的事）
- 返回结果无法理解

## JSON Schema 基础

所有主流 LLM Provider 都用 JSON Schema 描述工具参数：

```python
# Anthropic 格式
TOOLS = [{
    "name": "bash",
    "description": "Run a shell command.",
    "input_schema": {
        "type": "object",
        "properties": {
            "command": {"type": "string"}
        },
        "required": ["command"]
    }
}]

# OpenAI 格式（略有不同）
tools = [{
    "type": "function",
    "function": {
        "name": "bash",
        "description": "Run a shell command.",
        "parameters": {
            "type": "object",
            "properties": {
                "command": {"type": "string"}
            },
            "required": ["command"]
        }
    }
}]
```

两者核心结构一样：`name` + `description` + `properties` + `required`。

## 设计原则 1：Description 是你的武器

**Description 不是给人看的，是给 LLM 看的**。

### 差的 Description

```python
{"name": "read_file", "description": "Read file."}
```

问题：
- LLM 不知道能读什么文件
- 不知道返回格式
- 不知道限制

### 好的 Description

```python
{
    "name": "Read",
    "description": "Read the contents of a file. Supports text files and images. "
                   "Output is truncated to 2000 lines or 50KB. "
                   "Use offset/limit for large files.",
    "input_schema": {
        "type": "object",
        "properties": {
            "file_path": {
                "description": "Path to the file (relative or absolute)",
                "type": "string"
            },
            "offset": {
                "description": "Line number to start reading from (1-indexed)",
                "type": "number"
            },
            "limit": {
                "description": "Maximum number of lines to read",
                "type": "number"
            }
        },
        "required": []
    }
}
```

**关键点**：
1. 说明支持什么（text + images）
2. 说明限制（2000 lines / 50KB）
3. 说明如何处理大文件（offset/limit）
4. 每个参数都有 description

## 设计原则 2：类型要精确

```python
# 差：什么都是 string
"status": {"type": "string"}

# 好：用 enum 限制取值
"status": {
    "type": "string",
    "enum": ["pending", "running", "completed", "failed"],
    "description": "Task status"
}

# 好：用 number 而不是 string 表示数字
"limit": {
    "type": "number",
    "description": "Max results (1-100)"
}

# 好：用 boolean 而不是 string "true"/"false"
"recursive": {
    "type": "boolean",
    "description": "Search subdirectories"
}
```

### 数组和对象

```python
# 数组参数
"tags": {
    "type": "array",
    "items": {"type": "string"},
    "description": "Filter by tags"
}

# 对象参数
"options": {
    "type": "object",
    "properties": {
        "timeout": {"type": "number"},
        "retries": {"type": "number"}
    }
}
```

## 设计原则 3：Required 要谨慎

只把**真正必须**的参数放进 `required`：

```python
# 差：所有参数都 required
"required": ["path", "offset", "limit"]

# 好：只有核心参数 required
"required": ["path"]  # offset 和 limit 可选
```

为什么？因为 LLM 会尝试满足所有 required 字段。如果不必要的字段是 required，LLM 可能会瞎编值。

## 设计原则 4：一个工具做一件事

### 差：上帝工具

```python
{
    "name": "file_operation",
    "description": "Perform file operations",
    "input_schema": {
        "properties": {
            "operation": {"enum": ["read", "write", "delete", "copy", "move"]},
            "source": {"type": "string"},
            "destination": {"type": "string"},
            "content": {"type": "string"}
        }
    }
}
```

问题：
- 参数组合复杂（read 不需要 destination，write 不需要 source...）
- LLM 容易搞混
- 错误处理困难

### 好：单一职责

```python
TOOLS = [
    {"name": "read_file", ...},
    {"name": "write_file", ...},
    {"name": "edit_file", ...},  # 精确替换
    {"name": "delete_file", ...}
]
```

**每个工具只做一件事**。LLM 更容易选对工具，你更容易处理错误。

## 设计原则 5：返回结果要结构化

工具返回给 LLM 的内容同样重要：

```python
# 差：返回原始 dump
def list_files(path):
    return str(os.listdir(path))
# 输出: "['file1.txt', 'file2.py', 'dir1']"

# 好：结构化返回
def list_files(path):
    items = []
    for name in os.listdir(path):
        full_path = os.path.join(path, name)
        items.append({
            "name": name,
            "type": "dir" if os.path.isdir(full_path) else "file",
            "size": os.path.getsize(full_path) if os.path.isfile(full_path) else None
        })
    return json.dumps({"path": path, "items": items, "count": len(items)})
```

LLM 理解结构化数据比理解随意字符串容易得多。

## 实战案例：Edit 工具的设计演进

### V1：简单粗暴

```python
{"name": "edit", "description": "Edit file", 
 "input_schema": {"properties": {"path": {}, "content": {}}}}
```

问题：每次都要写完整文件内容，小改动也要传几 MB。

### V2：添加 diff

```python
{"name": "edit", "description": "Edit file with diff",
 "input_schema": {"properties": {"path": {}, "diff": {}}}}
```

问题：LLM 经常生成错误的 diff 格式。

### V3：精确替换（Claude Code 方案）

```python
{
    "name": "Edit",
    "description": "Edit a file by replacing exact text. "
                   "The old_string must match exactly (including whitespace). "
                   "Use this for precise, surgical edits.",
    "input_schema": {
        "type": "object",
        "properties": {
            "path": {"type": "string", "description": "Path to the file"},
            "old_string": {"type": "string", "description": "Exact text to find and replace"},
            "new_string": {"type": "string", "description": "New text"}
        },
        "required": ["path", "old_string", "new_string"]
    }
}
```

**这是目前最可靠的方案**：
- 必须找到精确匹配才能替换
- 如果找不到，工具报错，LLM 知道要重试
- 避免了 diff 解析的复杂性

## TypeScript 中的类型安全 Schema

在 pi-mono / OpenClaw 中，用 TypeBox 或 Zod 定义 Schema：

```typescript
import { Type } from '@sinclair/typebox';

const ReadFileSchema = Type.Object({
    path: Type.String({ description: "File path" }),
    offset: Type.Optional(Type.Number({ description: "Start line" })),
    limit: Type.Optional(Type.Number({ description: "Max lines" }))
});

// 自动生成 JSON Schema + TypeScript 类型
type ReadFileInput = Static<typeof ReadFileSchema>;
```

好处：
1. **编译时类型检查**：传错参数类型会报错
2. **自动生成 JSON Schema**：不用手写
3. **IDE 自动补全**：开发体验好

## 工具数量的权衡

工具越多，LLM 选择越难。经验法则：

| 工具数量 | 效果 |
|---------|------|
| 5-10 | 最佳，LLM 很少选错 |
| 10-20 | 可以，需要好的 description |
| 20-50 | 勉强，考虑分组或动态加载 |
| 50+ | 危险，需要 Skill 系统按需加载 |

**Skill 系统**（第 5 课讲过）就是为了解决工具爆炸问题。

## 常见错误

### 1. Description 里写代码

```python
# 差
"description": "def read_file(path): return open(path).read()"
```

LLM 不需要看实现，只需要知道怎么用。

### 2. 参数名歧义

```python
# 差
"file": {}  # 是路径？是内容？是文件对象？

# 好
"file_path": {}
"file_content": {}
```

### 3. 没有错误说明

```python
# 差
"description": "Delete file"

# 好
"description": "Delete file. Returns error if file doesn't exist or permission denied."
```

LLM 需要知道可能的失败情况。

## 总结

好的工具 Schema：
1. **Description 详尽**：告诉 LLM 能做什么、限制是什么
2. **类型精确**：用 enum、number、boolean 而不是万能 string
3. **Required 最小化**：只包含真正必须的参数
4. **单一职责**：一个工具做一件事
5. **返回结构化**：让 LLM 容易理解结果

工具是 Agent 的手和脚。设计好工具，Agent 就能做更多事。

## 思考题

1. 为什么 Claude Code 的 Edit 工具用 `old_string` + `new_string` 而不是 diff？
2. 如果你要设计一个「发送邮件」工具，参数应该有哪些？哪些是 required？
3. 工具返回错误时，应该返回什么格式让 LLM 最容易理解？

---

下一课：Agent 安全与护栏 - 如何防止 Agent 做危险的事
