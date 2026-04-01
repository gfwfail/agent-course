# 192 - Agent MCP 服务端实现（MCP Server Implementation）

> 前面我们学了如何消费 MCP 工具。今天反过来——**自己写一个 MCP Server**，把你的工具暴露给 Claude Desktop、OpenClaw、任何 MCP 客户端。  
> 一次实现，到处可用。

---

## 为什么要写 MCP Server？

```
你写了个超好用的工具（查数据库、调内部 API、执行脚本）
↓
以前：每个 Agent 都要复制粘贴工具定义
↓
MCP Server：写一次，所有支持 MCP 的客户端都能用
```

支持 MCP 的客户端：
- Claude Desktop
- OpenClaw（通过 mcporter 工具）
- Cursor / Windsurf
- 其他 Agent 框架（LangChain、CrewAI...）

---

## MCP 协议基础（30 秒回顾）

MCP（Model Context Protocol）定义了三种能力：

| 能力 | 作用 | LLM 视角 |
|------|------|----------|
| **Tools** | 可调用的函数 | tool_use |
| **Resources** | 可读取的数据源 | 文件/URL 内容 |
| **Prompts** | 预设提示模板 | 复用的 prompt 片段 |

传输协议：
- **stdio**：本地进程，最常用
- **HTTP/SSE**：远程服务，适合多客户端共享

---

## 实战：用 TypeScript 写一个 MCP Server

### 1. 安装依赖

```bash
npm init -y
npm install @modelcontextprotocol/sdk zod
npm install -D typescript @types/node tsx
```

### 2. 最简单的 MCP Server

```typescript
// src/server.ts
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";
import { z } from "zod";

// ─── 1. 创建 Server 实例 ───────────────────────────────────────
const server = new Server(
  {
    name: "my-agent-tools",
    version: "1.0.0",
  },
  {
    capabilities: {
      tools: {},      // 声明支持 Tools
      resources: {},  // 声明支持 Resources（可选）
    },
  }
);

// ─── 2. 定义工具列表 ───────────────────────────────────────────
server.setRequestHandler(ListToolsRequestSchema, async () => {
  return {
    tools: [
      {
        name: "get_weather",
        description: "获取指定城市的当前天气",
        inputSchema: {
          type: "object",
          properties: {
            city: {
              type: "string",
              description: "城市名称（中文或英文）",
            },
            unit: {
              type: "string",
              enum: ["celsius", "fahrenheit"],
              description: "温度单位，默认摄氏度",
            },
          },
          required: ["city"],
        },
      },
      {
        name: "query_database",
        description: "查询内部数据库，返回 JSON 结果",
        inputSchema: {
          type: "object",
          properties: {
            sql: {
              type: "string",
              description: "只读 SELECT 语句",
            },
          },
          required: ["sql"],
        },
      },
    ],
  };
});

// ─── 3. 实现工具调用 ──────────────────────────────────────────
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;

  switch (name) {
    case "get_weather": {
      const { city, unit = "celsius" } = args as {
        city: string;
        unit?: string;
      };
      // 实际项目里调真实天气 API
      const temp = unit === "celsius" ? "22°C" : "72°F";
      return {
        content: [
          {
            type: "text",
            text: JSON.stringify({
              city,
              temperature: temp,
              condition: "晴天",
              humidity: "65%",
            }),
          },
        ],
      };
    }

    case "query_database": {
      const { sql } = args as { sql: string };
      // 安全检查：只允许 SELECT
      if (!sql.trim().toUpperCase().startsWith("SELECT")) {
        return {
          content: [{ type: "text", text: "错误：只允许 SELECT 语句" }],
          isError: true,
        };
      }
      // 实际项目里连真实数据库
      return {
        content: [
          {
            type: "text",
            text: JSON.stringify({ rows: [], sql, message: "查询成功（mock）" }),
          },
        ],
      };
    }

    default:
      return {
        content: [{ type: "text", text: `未知工具：${name}` }],
        isError: true,
      };
  }
});

// ─── 4. 启动 ──────────────────────────────────────────────────
async function main() {
  const transport = new StdioServerTransport();
  await server.connect(transport);
  console.error("MCP Server 已启动（stdio 模式）");
}

main().catch(console.error);
```

---

## 3. 接入 Claude Desktop

在 `~/Library/Application Support/Claude/claude_desktop_config.json` 添加：

```json
{
  "mcpServers": {
    "my-agent-tools": {
      "command": "node",
      "args": ["/path/to/your/dist/server.js"],
      "env": {
        "DATABASE_URL": "postgresql://..."
      }
    }
  }
}
```

重启 Claude Desktop → 工具栏出现你的工具 ✅

---

## 4. 接入 OpenClaw（通过 mcporter）

```bash
# 注册本地 MCP Server
mcporter add my-tools \
  --transport stdio \
  --command "node /path/to/dist/server.js"

# 验证工具可见
mcporter call my-tools list_tools
```

---

## 进阶：暴露 Resources（让 LLM 读取动态数据）

```typescript
import {
  ListResourcesRequestSchema,
  ReadResourceRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";

// 声明 resources 能力（在 Server 构造时已设置）

// 列出可用资源
server.setRequestHandler(ListResourcesRequestSchema, async () => {
  return {
    resources: [
      {
        uri: "db://users/recent",
        name: "最近注册用户",
        description: "过去 24 小时注册的用户列表",
        mimeType: "application/json",
      },
      {
        uri: "file://config/app.yaml",
        name: "应用配置",
        mimeType: "text/yaml",
      },
    ],
  };
});

// 读取资源内容
server.setRequestHandler(ReadResourceRequestSchema, async (request) => {
  const { uri } = request.params;

  if (uri === "db://users/recent") {
    const users = await fetchRecentUsers(); // 你的业务逻辑
    return {
      contents: [
        {
          uri,
          mimeType: "application/json",
          text: JSON.stringify(users),
        },
      ],
    };
  }

  throw new Error(`未知资源：${uri}`);
});
```

LLM 可以主动请求读取这些资源，获得动态数据，而不需要显式 tool_use。

---

## 进阶：HTTP/SSE 模式（多客户端共享）

```typescript
import { SSEServerTransport } from "@modelcontextprotocol/sdk/server/sse.js";
import express from "express";

const app = express();
const transports = new Map<string, SSEServerTransport>();

// SSE 端点：客户端订阅
app.get("/sse", async (req, res) => {
  const transport = new SSEServerTransport("/message", res);
  const sessionId = transport.sessionId;
  transports.set(sessionId, transport);

  // 为每个连接创建独立 Server 实例（或共享，看业务需求）
  const srv = createServer(); // 复用上面的 server 创建逻辑
  await srv.connect(transport);

  req.on("close", () => {
    transports.delete(sessionId);
  });
});

// Message 端点：客户端发送请求
app.post("/message", express.json(), async (req, res) => {
  const sessionId = req.query.sessionId as string;
  const transport = transports.get(sessionId);
  if (!transport) {
    res.status(404).json({ error: "会话不存在" });
    return;
  }
  await transport.handlePostMessage(req, res);
});

app.listen(3000, () => {
  console.log("MCP Server (HTTP/SSE) 运行在 http://localhost:3000");
});
```

客户端配置：
```json
{
  "mcpServers": {
    "my-tools": {
      "url": "http://your-server:3000/sse"
    }
  }
}
```

---

## Python 版（用 FastMCP，更简洁）

```python
# server.py
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("my-agent-tools")

@mcp.tool()
def get_weather(city: str, unit: str = "celsius") -> dict:
    """获取指定城市的当前天气"""
    # 调用真实天气 API
    return {
        "city": city,
        "temperature": "22°C" if unit == "celsius" else "72°F",
        "condition": "晴天"
    }

@mcp.tool()
def query_database(sql: str) -> dict:
    """查询内部数据库（只允许 SELECT）"""
    if not sql.strip().upper().startswith("SELECT"):
        raise ValueError("只允许 SELECT 语句")
    # 执行真实查询
    return {"rows": [], "sql": sql}

@mcp.resource("db://users/recent")
def recent_users() -> str:
    """最近注册的用户（JSON 格式）"""
    return '{"users": []}'

if __name__ == "__main__":
    mcp.run()  # 默认 stdio 模式
    # mcp.run(transport="sse", port=3000)  # HTTP/SSE 模式
```

Python 版注册到 Claude Desktop：
```json
{
  "mcpServers": {
    "my-tools": {
      "command": "python",
      "args": ["/path/to/server.py"]
    }
  }
}
```

---

## OpenClaw Skills 与 MCP Server 的关系

```
OpenClaw Skills                 MCP Server
─────────────────               ──────────────────────
仅供 OpenClaw 自用              任何 MCP 客户端可用
直接读文件/调 API               通过标准协议通信
无需网络                        可跨机器部署
随 OpenClaw 更新                独立部署/版本管理
```

**最佳实践**：
- 团队内部工具 → MCP Server（多人共享）
- 个人配置/技巧 → OpenClaw Skills（本地快速迭代）

---

## 工具 Schema 设计最佳实践

```typescript
// ❌ 描述太模糊，LLM 不知道传什么
{
  name: "process",
  description: "处理数据",
  inputSchema: { type: "object", properties: { data: { type: "string" } } }
}

// ✅ 描述清晰，约束明确
{
  name: "send_notification",
  description: "向指定用户发送站内通知。用于任务完成、告警提醒等场景。不适用于营销推送。",
  inputSchema: {
    type: "object",
    properties: {
      userId: {
        type: "string",
        description: "目标用户 ID，格式：usr_xxxxxxxx",
      },
      message: {
        type: "string",
        description: "通知内容，不超过 200 字",
        maxLength: 200,
      },
      priority: {
        type: "string",
        enum: ["normal", "high", "urgent"],
        description: "优先级。urgent 会触发短信提醒",
        default: "normal",
      },
    },
    required: ["userId", "message"],
  },
}
```

**黄金法则**：description 写给 LLM 看，不是给人看。要告诉 LLM *什么时候用*、*不适合什么场景*、*参数格式举例*。

---

## 错误处理规范

```typescript
// MCP 工具错误分两种：

// 1. 工具级错误（业务逻辑错误，LLM 可以看到并重试）
return {
  content: [{ type: "text", text: "用户不存在：usr_123" }],
  isError: true,  // ← 标记为错误，但不抛异常
};

// 2. 协议级错误（服务器故障，客户端会报错）
throw new Error("数据库连接失败");  // ← 抛异常，触发 MCP 错误响应

// 最佳实践：业务错误用 isError: true，系统错误才 throw
```

---

## 测试你的 MCP Server

```bash
# 方法一：MCP Inspector（官方调试工具）
npx @modelcontextprotocol/inspector node dist/server.js

# 方法二：直接用 stdio 测试
echo '{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{}}' | node dist/server.js

# 方法三：mcporter 命令行
mcporter call my-tools get_weather '{"city": "Sydney"}'
```

---

## 完整项目结构

```
my-mcp-server/
├── src/
│   ├── server.ts        # 入口，Server 配置和启动
│   ├── tools/
│   │   ├── weather.ts   # 天气工具
│   │   └── database.ts  # 数据库工具
│   ├── resources/
│   │   └── users.ts     # 用户资源
│   └── index.ts         # 工具注册汇总
├── package.json
├── tsconfig.json
└── README.md            # 写清楚：工具列表、配置方式、环境变量
```

---

## 总结

| 知识点 | 要点 |
|--------|------|
| **MCP = 工具标准化协议** | 写一次，所有客户端都能用 |
| **三种能力** | Tools（调用）/ Resources（读取）/ Prompts（模板） |
| **两种传输** | stdio（本地）/ HTTP+SSE（远程） |
| **TypeScript SDK** | `@modelcontextprotocol/sdk` |
| **Python FastMCP** | 装饰器语法，更简洁 |
| **Schema 质量决定 LLM 使用效果** | description 写给 LLM 看 |
| **isError vs throw** | 业务错误 vs 系统错误 |

> **一句话记住**：MCP Server 就是把你的工具从"只有我能用"变成"所有 AI 都能用"的适配器层。Schema 写得好，LLM 自己会知道什么时候叫你。

---

*下节预告：Agent 跨框架互操作与工具生态对接（Cross-Framework Interoperability）——当你的 MCP Server 遇到 LangChain、AutoGen、CrewAI，怎么无缝对接？*
