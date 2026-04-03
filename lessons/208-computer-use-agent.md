# 208 - Agent Computer Use（计算机使用能力）

> **核心思想：** 给 Agent 一双"眼睛"和"手"——截图观察界面、模拟键鼠操作，让 Agent 像人一样使用电脑。

---

## 为什么需要 Computer Use？

传统 Agent 通过 API、文件、数据库与世界交互。但现实中大量系统：

- 没有 API（老旧 ERP、政务网站）
- API 权限受限
- 界面操作比 API 调用更直观

**Computer Use 让 Agent 突破"有无 API"的限制，直接操作任何 GUI 应用。**

---

## 核心交互模型

```
┌─────────────────────────────────────────┐
│           Computer Use Loop             │
│                                         │
│  [观察] 截图 → base64 → LLM            │
│     ↓                                   │
│  [思考] LLM 分析界面，决定下一步操作   │
│     ↓                                   │
│  [行动] 执行鼠标/键盘/滚动指令         │
│     ↓                                   │
│  [验证] 再次截图确认结果               │
│     ↓ (循环直到任务完成)               │
└─────────────────────────────────────────┘
```

---

## Anthropic Computer Use API

Claude 原生支持三种 Computer Use 工具：

```typescript
const computerUseTools = [
  {
    type: "computer_20241022",      // 截图 + 鼠标 + 键盘
    name: "computer",
    display_width_px: 1280,
    display_height_px: 800,
    display_number: 1,
  },
  {
    type: "text_editor_20241022",   // 查看/编辑文件
    name: "str_replace_editor",
  },
  {
    type: "bash_20241022",          // 执行 shell 命令
    name: "bash",
  },
];
```

**三大工具分工：**
- `computer`：截图观察 + 鼠标点击 + 键盘输入
- `str_replace_editor`：精确编辑文本文件
- `bash`：执行系统命令（替代 computer 处理可命令化的任务）

---

## TypeScript 完整实现

### 1. 工具执行器

```typescript
import Anthropic from "@anthropic-ai/sdk";
import { execSync } from "child_process";
import { readFileSync, writeFileSync } from "fs";

const client = new Anthropic();

// 截图工具（macOS）
async function takeScreenshot(): Promise<string> {
  execSync("screencapture -x /tmp/screenshot.png");
  const imageBuffer = readFileSync("/tmp/screenshot.png");
  return imageBuffer.toString("base64");
}

// 鼠标点击
function mouseClick(x: number, y: number, button: "left" | "right" = "left") {
  execSync(`cliclick ${button === "right" ? "rc" : "c"}:${x},${y}`);
}

// 键盘输入
function typeText(text: string) {
  // 转义特殊字符
  const escaped = text.replace(/'/g, "'\\''");
  execSync(`osascript -e 'tell application "System Events" to keystroke "${escaped}"'`);
}

// 特殊按键
function pressKey(key: string) {
  execSync(`osascript -e 'tell application "System Events" to key code "${key}"'`);
}

// 滚动
function scroll(x: number, y: number, direction: "up" | "down", amount: number = 3) {
  const delta = direction === "up" ? amount : -amount;
  execSync(`cliclick kd:cmd kp:${delta > 0 ? "up" : "down"} ku:cmd`);
}
```

### 2. Computer Use Agent 主循环

```typescript
interface ComputerAction {
  type: "screenshot" | "left_click" | "right_click" | "type" | "key" | "scroll";
  coordinate?: [number, number];
  text?: string;
  direction?: "up" | "down";
  amount?: number;
}

async function executeComputerAction(action: ComputerAction): Promise<string> {
  switch (action.type) {
    case "screenshot": {
      const screenshot = await takeScreenshot();
      return screenshot; // base64
    }
    case "left_click":
      mouseClick(action.coordinate![0], action.coordinate![1], "left");
      return "Clicked";
    case "right_click":
      mouseClick(action.coordinate![0], action.coordinate![1], "right");
      return "Right-clicked";
    case "type":
      typeText(action.text!);
      return "Typed";
    case "key":
      pressKey(action.text!);
      return "Key pressed";
    case "scroll":
      scroll(
        action.coordinate![0],
        action.coordinate![1],
        action.direction!,
        action.amount
      );
      return "Scrolled";
    default:
      throw new Error(`Unknown action: ${action.type}`);
  }
}

async function runComputerUseAgent(task: string): Promise<void> {
  console.log(`🖥️ Starting Computer Use Agent: ${task}`);

  const messages: Anthropic.MessageParam[] = [
    { role: "user", content: task },
  ];

  let iterations = 0;
  const MAX_ITERATIONS = 50; // 防止无限循环

  while (iterations < MAX_ITERATIONS) {
    iterations++;

    const response = await client.messages.create({
      model: "claude-opus-4-5", // Computer Use 建议用最强模型
      max_tokens: 4096,
      tools: computerUseTools as any,
      messages,
    });

    console.log(`\n--- Iteration ${iterations} ---`);

    // 处理工具调用
    const toolUses = response.content.filter(
      (block) => block.type === "tool_use"
    );
    const textBlocks = response.content.filter(
      (block) => block.type === "text"
    );

    // 打印 Agent 的思考
    for (const block of textBlocks) {
      if (block.type === "text") {
        console.log(`💭 Agent: ${block.text}`);
      }
    }

    // 任务完成
    if (response.stop_reason === "end_turn" && toolUses.length === 0) {
      console.log("✅ Task completed!");
      break;
    }

    // 执行工具调用
    const toolResults: Anthropic.ToolResultBlockParam[] = [];

    for (const toolUse of toolUses) {
      if (toolUse.type !== "tool_use") continue;

      console.log(`🔧 Action: ${toolUse.name}`, toolUse.input);

      try {
        let result: Anthropic.ToolResultBlockParam;

        if (toolUse.name === "computer") {
          const action = toolUse.input as ComputerAction;
          
          if (action.type === "screenshot") {
            const screenshot = await takeScreenshot();
            result = {
              type: "tool_result",
              tool_use_id: toolUse.id,
              content: [
                {
                  type: "image",
                  source: {
                    type: "base64",
                    media_type: "image/png",
                    data: screenshot,
                  },
                },
              ],
            };
          } else {
            await executeComputerAction(action);
            // 操作后自动截图，让 LLM 验证结果
            const screenshot = await takeScreenshot();
            result = {
              type: "tool_result",
              tool_use_id: toolUse.id,
              content: [
                {
                  type: "image",
                  source: {
                    type: "base64",
                    media_type: "image/png",
                    data: screenshot,
                  },
                },
              ],
            };
          }
        } else if (toolUse.name === "bash") {
          const { command } = toolUse.input as { command: string };
          const output = execSync(command, { encoding: "utf-8", timeout: 30000 });
          result = {
            type: "tool_result",
            tool_use_id: toolUse.id,
            content: output || "(no output)",
          };
        } else if (toolUse.name === "str_replace_editor") {
          // 处理文件编辑工具
          result = await handleTextEditor(toolUse);
        } else {
          result = {
            type: "tool_result",
            tool_use_id: toolUse.id,
            content: `Unknown tool: ${toolUse.name}`,
            is_error: true,
          };
        }

        toolResults.push(result);
      } catch (error) {
        toolResults.push({
          type: "tool_result",
          tool_use_id: toolUse.id,
          content: `Error: ${error}`,
          is_error: true,
        });
      }
    }

    // 更新消息历史
    messages.push({ role: "assistant", content: response.content });
    messages.push({ role: "user", content: toolResults });
  }

  if (iterations >= MAX_ITERATIONS) {
    console.warn("⚠️ Max iterations reached");
  }
}

async function handleTextEditor(toolUse: Anthropic.ToolUseBlock): Promise<Anthropic.ToolResultBlockParam> {
  const { command, path, old_str, new_str } = toolUse.input as any;
  
  switch (command) {
    case "view": {
      const content = readFileSync(path, "utf-8");
      return {
        type: "tool_result",
        tool_use_id: toolUse.id,
        content,
      };
    }
    case "str_replace": {
      const content = readFileSync(path, "utf-8");
      const updated = content.replace(old_str, new_str);
      writeFileSync(path, updated);
      return {
        type: "tool_result",
        tool_use_id: toolUse.id,
        content: "File updated",
      };
    }
    case "create": {
      writeFileSync(path, new_str || "");
      return {
        type: "tool_result",
        tool_use_id: toolUse.id,
        content: "File created",
      };
    }
    default:
      return {
        type: "tool_result",
        tool_use_id: toolUse.id,
        content: `Unknown command: ${command}`,
        is_error: true,
      };
  }
}
```

### 3. 使用示例

```typescript
// 示例 1：自动填写表单
await runComputerUseAgent(
  "打开 Safari，访问 https://example.com/form，填写姓名为'张三'，邮箱为'test@example.com'，点击提交按钮"
);

// 示例 2：截图分析
await runComputerUseAgent(
  "截图当前屏幕，告诉我现在打开了哪些应用，并把结果写入 /tmp/apps.txt"
);

// 示例 3：数据提取
await runComputerUseAgent(
  "打开 Chrome，访问公司内网报表页面，把销售数据截图并提取到 CSV 文件"
);
```

---

## OpenClaw 中的 Computer Use

OpenClaw 已内置浏览器控制能力，通过 `browser` 工具实现类似 Computer Use 的效果：

```typescript
// OpenClaw browser 工具 = 专为 Web 优化的 Computer Use
{
  action: "snapshot",    // 观察页面结构（比截图更高效）
  action: "act",         // 执行操作（click/type/fill）
  action: "screenshot",  // 视觉截图（兜底）
}
```

**OpenClaw browser vs 原生 Computer Use 对比：**

| 维度 | OpenClaw browser | 原生 Computer Use |
|------|-----------------|-------------------|
| Web 场景 | ✅ 首选（DOM 感知） | ⚠️ 纯视觉，较慢 |
| 桌面应用 | ❌ 不支持 | ✅ 唯一选择 |
| Token 消耗 | 低（结构化 snapshot） | 高（图片每次 ~1000 tokens） |
| 精度 | 高（aria refs） | 中（坐标定位） |
| 稳定性 | 高（DOM 不变） | 低（UI 像素变化即失效） |

---

## Python 版本（简洁实现）

```python
import anthropic
import base64
import subprocess
from pathlib import Path

client = anthropic.Anthropic()

def take_screenshot() -> str:
    subprocess.run(["screencapture", "-x", "/tmp/screenshot.png"])
    return base64.b64encode(Path("/tmp/screenshot.png").read_bytes()).decode()

def run_computer_use(task: str) -> str:
    messages = [{"role": "user", "content": task}]
    
    tools = [{
        "type": "computer_20241022",
        "name": "computer",
        "display_width_px": 1280,
        "display_height_px": 800,
        "display_number": 1,
    }]
    
    while True:
        response = client.messages.create(
            model="claude-opus-4-5",
            max_tokens=4096,
            tools=tools,
            messages=messages,
        )
        
        if response.stop_reason == "end_turn":
            # 提取最终文本回复
            for block in response.content:
                if block.type == "text":
                    return block.text
            return "Task completed"
        
        tool_results = []
        for block in response.content:
            if block.type == "tool_use" and block.name == "computer":
                action = block.input
                
                if action["action"] == "screenshot":
                    screenshot = take_screenshot()
                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": [{
                            "type": "image",
                            "source": {
                                "type": "base64",
                                "media_type": "image/png",
                                "data": screenshot,
                            }
                        }]
                    })
                elif action["action"] == "left_click":
                    x, y = action["coordinate"]
                    subprocess.run(["cliclick", f"c:{x},{y}"])
                    screenshot = take_screenshot()
                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": [{
                            "type": "image",
                            "source": {"type": "base64", "media_type": "image/png", "data": screenshot}
                        }]
                    })
        
        messages.append({"role": "assistant", "content": response.content})
        messages.append({"role": "user", "content": tool_results})
```

---

## 关键设计原则

### 1. 操作后必须截图验证

```typescript
// ❌ 错误：点击后不验证
mouseClick(x, y);
return "Clicked";

// ✅ 正确：点击后截图，让 LLM 确认结果
mouseClick(x, y);
const screenshot = await takeScreenshot();
return { screenshot }; // LLM 看到结果再决定下一步
```

### 2. Token 预算控制（截图很贵）

```typescript
// 每张 1280x800 截图 ≈ 1000-1500 tokens
// 50 次迭代 × 2张/次 = 10万+ tokens

// 策略：优先用 bash/API，最后才截图
const PREFER_BASH = true; // 能用命令解决的不截图
```

### 3. 安全沙箱

```typescript
// Computer Use 风险极高！必须沙箱隔离
const ALLOWED_APPS = ["Safari", "Chrome", "TextEdit"];
const BLOCKED_COMMANDS = ["rm", "sudo", "shutdown"];

function validateAction(action: ComputerAction): boolean {
  if (action.type === "key" && action.text?.includes("cmd+q")) {
    return false; // 禁止关闭应用
  }
  return true;
}
```

### 4. 人机协作检查点

```typescript
// 危险操作前暂停，等人类确认
const DANGEROUS_PATTERNS = [
  /delete/i, /remove/i, /submit.*payment/i, /confirm.*purchase/i
];

function needsHumanApproval(text: string): boolean {
  return DANGEROUS_PATTERNS.some(p => p.test(text));
}

// 暂停并等待 Telegram 确认
if (needsHumanApproval(action.text ?? "")) {
  await sendTelegramConfirmation(`⚠️ 即将执行危险操作: ${action.text}`);
}
```

---

## 实际应用场景

| 场景 | 工具选择 | 原因 |
|------|---------|------|
| 操作 Web 应用 | OpenClaw browser | DOM 感知，效率高 |
| 操作桌面 GUI | Computer Use | 唯一选择 |
| 自动化测试 | Playwright/Puppeteer | 专业工具更稳定 |
| 无 API 的老系统 | Computer Use | 视觉操作突破限制 |
| 数据录入 | Computer Use + bash | 混合使用 |

---

## 与 OpenClaw 的集成

OpenClaw 的 `browser` 工具在内部使用 Playwright，原理与 Computer Use 相同：

```typescript
// OpenClaw 内部流程（概念）
const snapshot = await browser({ action: "snapshot" }); // 观察
// LLM 分析 snapshot 决定操作
await browser({ action: "act", request: { kind: "click", ref: "e12" } }); // 行动
```

**关键区别：** OpenClaw 用结构化的 ARIA/DOM 树代替截图，
把每次迭代的 token 消耗从 ~1500 降到 ~200。

---

## 总结

```
Computer Use = 截图观察 + 工具执行 + 循环验证

核心挑战：
  - Token 消耗高（截图贵）
  - 坐标定位脆（UI 一变就失效）
  - 安全风险大（能操作任何东西）

最佳实践：
  1. Web 用 browser snapshot，桌面才用 Computer Use
  2. 操作后立即截图验证
  3. 危险操作必须人工确认
  4. 设置最大迭代次数防止无限循环
  5. 优先 bash/API，截图是最后手段
```

**Computer Use 不是银弹，是 API 失效时的终极武器。**
