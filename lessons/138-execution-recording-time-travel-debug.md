# 138 - Agent 执行录制与时间旅行调试（Execution Recording & Time-Travel Debugging）

> **核心思想：** 把 Agent 的每一步执行都录下来，然后像视频一样回放、暂停、修改、重跑——调试不再靠猜，而是靠重现。

---

## 为什么需要时间旅行调试？

传统 Agent 调试的痛点：

```
用户: "昨天那个查询失败了，啥原因？"
你:   😰 日志太少 / LLM 输出不确定 / 重跑可能得到不同结果
```

时间旅行调试解决的问题：
- **不可复现性**：LLM 是概率性的，同样输入不一定得到同样输出
- **调试困难**：出错步骤往往在一长串工具调用中间，不知道从哪开始看
- **成本高**：重跑整个流程耗 Token、耗时、耗钱
- **局部修复**：找到 bug 后只想修第 5 步，不想重跑前 4 步

---

## 核心概念

```
录制 → 回放 → 暂停 → 修改 → 继续
  ↓
Snapshot(状态) + EventLog(事件)  =  可回放的执行轨迹
```

### 三个关键组件

1. **Recorder（录制器）** — 拦截所有工具调用，记录输入/输出/时序
2. **Replayer（回放器）** — 重放录制，对 LLM 请求使用 mock，工具调用用录制结果
3. **Patcher（补丁器）** — 在回放时修改特定步骤的输入或输出，观察影响

---

## 实现：TypeScript（参考 pi-mono 风格）

### 数据结构

```typescript
// 执行轨迹的基本单元
interface ExecutionEvent {
  id: string;
  timestamp: number;
  stepIndex: number;
  type: 'llm_request' | 'llm_response' | 'tool_call' | 'tool_result' | 'agent_start' | 'agent_end';
  data: unknown;
}

interface ToolCallEvent extends ExecutionEvent {
  type: 'tool_call';
  data: {
    toolName: string;
    toolInput: Record<string, unknown>;
    requestId: string;
  };
}

interface ToolResultEvent extends ExecutionEvent {
  type: 'tool_result';
  data: {
    toolName: string;
    result: unknown;
    durationMs: number;
    error?: string;
    requestId: string; // 与 tool_call 配对
  };
}

// 完整录制
interface ExecutionRecording {
  id: string;
  sessionId: string;
  startTime: number;
  endTime?: number;
  initialPrompt: string;
  events: ExecutionEvent[];
  metadata: {
    model: string;
    totalTokens: number;
    totalCost: number;
    finalResult?: string;
  };
}
```

### Recorder：透明拦截

```typescript
class ExecutionRecorder {
  private events: ExecutionEvent[] = [];
  private stepIndex = 0;
  private recording: ExecutionRecording;

  constructor(sessionId: string, initialPrompt: string) {
    this.recording = {
      id: crypto.randomUUID(),
      sessionId,
      startTime: Date.now(),
      initialPrompt,
      events: this.events,
      metadata: { model: '', totalTokens: 0, totalCost: 0 }
    };
  }

  // 包装工具——无感知录制
  wrapTool<T extends Record<string, unknown>>(
    toolName: string,
    originalFn: (input: T) => Promise<unknown>
  ): (input: T) => Promise<unknown> {
    return async (input: T) => {
      const requestId = crypto.randomUUID();
      const callEvent: ToolCallEvent = {
        id: crypto.randomUUID(),
        timestamp: Date.now(),
        stepIndex: this.stepIndex++,
        type: 'tool_call',
        data: { toolName, toolInput: input, requestId }
      };
      this.events.push(callEvent);

      const start = Date.now();
      try {
        const result = await originalFn(input);
        const resultEvent: ToolResultEvent = {
          id: crypto.randomUUID(),
          timestamp: Date.now(),
          stepIndex: this.stepIndex++,
          type: 'tool_result',
          data: {
            toolName,
            result,
            durationMs: Date.now() - start,
            requestId
          }
        };
        this.events.push(resultEvent);
        return result;
      } catch (error) {
        const resultEvent: ToolResultEvent = {
          id: crypto.randomUUID(),
          timestamp: Date.now(),
          stepIndex: this.stepIndex++,
          type: 'tool_result',
          data: {
            toolName,
            result: null,
            durationMs: Date.now() - start,
            error: String(error),
            requestId
          }
        };
        this.events.push(resultEvent);
        throw error;
      }
    };
  }

  // 记录 LLM 调用
  recordLLMRequest(messages: unknown[], model: string): string {
    const requestId = crypto.randomUUID();
    this.events.push({
      id: crypto.randomUUID(),
      timestamp: Date.now(),
      stepIndex: this.stepIndex++,
      type: 'llm_request',
      data: { messages, model, requestId }
    });
    return requestId;
  }

  recordLLMResponse(requestId: string, response: unknown, usage: { inputTokens: number; outputTokens: number }) {
    this.events.push({
      id: crypto.randomUUID(),
      timestamp: Date.now(),
      stepIndex: this.stepIndex++,
      type: 'llm_response',
      data: { response, usage, requestId }
    });
    this.recording.metadata.totalTokens += usage.inputTokens + usage.outputTokens;
  }

  finish(finalResult?: string): ExecutionRecording {
    this.recording.endTime = Date.now();
    this.recording.metadata.finalResult = finalResult;
    return this.recording;
  }
}
```

### Replayer：确定性重放

```typescript
interface ReplayPatch {
  // 修改特定 stepIndex 的工具结果
  patchToolResult?: {
    stepIndex: number;
    newResult: unknown;
  };
  // 从某步开始重新实际执行（而非使用录制结果）
  reExecuteFromStep?: number;
}

class ExecutionReplayer {
  constructor(
    private recording: ExecutionRecording,
    private realTools: Map<string, (input: unknown) => Promise<unknown>>
  ) {}

  async replay(patch?: ReplayPatch): Promise<{
    events: ExecutionEvent[];
    divergedAt?: number;  // 与原录制产生分歧的步骤
  }> {
    const replayedEvents: ExecutionEvent[] = [];
    const toolCallEvents = this.recording.events.filter(e => e.type === 'tool_call') as ToolCallEvent[];
    const toolResultMap = new Map<string, ToolResultEvent>();

    // 建立 requestId → result 映射
    for (const event of this.recording.events) {
      if (event.type === 'tool_result') {
        const e = event as ToolResultEvent;
        toolResultMap.set(e.data.requestId, e);
      }
    }

    let divergedAt: number | undefined;

    for (const callEvent of toolCallEvents) {
      const originalResult = toolResultMap.get(callEvent.data.requestId);
      
      // 检查是否需要 patch 这一步
      const shouldPatch = patch?.patchToolResult?.stepIndex === callEvent.stepIndex;
      const shouldReExecute = patch?.reExecuteFromStep !== undefined &&
        callEvent.stepIndex >= patch.reExecuteFromStep;

      let result: unknown;
      let error: string | undefined;
      const start = Date.now();

      if (shouldPatch) {
        // 使用 patch 提供的新结果
        result = patch!.patchToolResult!.newResult;
        divergedAt = divergedAt ?? callEvent.stepIndex;
        console.log(`[Replay] Patched step ${callEvent.stepIndex}: ${callEvent.data.toolName}`);
      } else if (shouldReExecute) {
        // 重新真实执行
        const realTool = this.realTools.get(callEvent.data.toolName);
        if (realTool) {
          try {
            result = await realTool(callEvent.data.toolInput);
            divergedAt = divergedAt ?? callEvent.stepIndex;
          } catch (e) {
            error = String(e);
          }
        }
      } else {
        // 使用录制的结果（完全确定性）
        result = originalResult?.data.result;
        error = originalResult?.data.error;
      }

      replayedEvents.push({
        ...callEvent,
        timestamp: Date.now() // 重放时用新时间戳
      });

      replayedEvents.push({
        id: crypto.randomUUID(),
        timestamp: Date.now(),
        stepIndex: callEvent.stepIndex + 0.5, // 插在 call 后
        type: 'tool_result',
        data: {
          toolName: callEvent.data.toolName,
          result,
          durationMs: Date.now() - start,
          error,
          requestId: callEvent.data.requestId
        }
      } as ToolResultEvent);
    }

    return { events: replayedEvents, divergedAt };
  }

  // 只回放到指定步骤（用于断点调试）
  async replayUntilStep(targetStep: number): Promise<ExecutionEvent[]> {
    const result = await this.replay();
    return result.events.filter(e => e.stepIndex <= targetStep);
  }
}
```

### 时间旅行调试器：交互式调试

```typescript
class TimeTravelDebugger {
  private recordings = new Map<string, ExecutionRecording>();
  private store: { save: (r: ExecutionRecording) => Promise<void>; load: (id: string) => Promise<ExecutionRecording | null> };

  constructor(storeImpl: typeof TimeTravelDebugger.prototype['store']) {
    this.store = storeImpl;
  }

  async saveRecording(recording: ExecutionRecording): Promise<void> {
    this.recordings.set(recording.id, recording);
    await this.store.save(recording);
    console.log(`[TTDebug] 录制已保存: ${recording.id} (${recording.events.length} events)`);
  }

  // 找到出错的步骤
  findErrorStep(recording: ExecutionRecording): ToolResultEvent | null {
    const errors = recording.events.filter(
      e => e.type === 'tool_result' && (e as ToolResultEvent).data.error
    ) as ToolResultEvent[];
    return errors[0] ?? null;
  }

  // 找到最慢的工具调用
  findSlowestStep(recording: ExecutionRecording): { toolName: string; durationMs: number; stepIndex: number } | null {
    const results = recording.events.filter(e => e.type === 'tool_result') as ToolResultEvent[];
    if (results.length === 0) return null;
    const slowest = results.reduce((a, b) => a.data.durationMs > b.data.durationMs ? a : b);
    return { toolName: slowest.data.toolName, durationMs: slowest.data.durationMs, stepIndex: slowest.stepIndex };
  }

  // 生成执行摘要
  summarize(recording: ExecutionRecording): string {
    const toolCalls = recording.events.filter(e => e.type === 'tool_call') as ToolCallEvent[];
    const toolResults = recording.events.filter(e => e.type === 'tool_result') as ToolResultEvent[];
    const errors = toolResults.filter(e => e.data.error);
    const totalDuration = recording.endTime ? recording.endTime - recording.startTime : 0;

    const toolStats = new Map<string, { count: number; totalMs: number; errors: number }>();
    for (const result of toolResults) {
      const stat = toolStats.get(result.data.toolName) ?? { count: 0, totalMs: 0, errors: 0 };
      stat.count++;
      stat.totalMs += result.data.durationMs;
      if (result.data.error) stat.errors++;
      toolStats.set(result.data.toolName, stat);
    }

    const lines = [
      `📼 执行录制 ${recording.id.slice(0, 8)}`,
      `⏱️  总耗时: ${totalDuration}ms`,
      `🔧 工具调用: ${toolCalls.length} 次 (${errors.length} 个错误)`,
      `💰 Token 消耗: ${recording.metadata.totalTokens}`,
      '',
      '工具调用统计:',
      ...Array.from(toolStats.entries()).map(
        ([name, stat]) => `  ${name}: ${stat.count}次, 平均${Math.round(stat.totalMs / stat.count)}ms${stat.errors ? ` ❌${stat.errors}错误` : ''}`
      )
    ];

    return lines.join('\n');
  }
}
```

---

## 实现：Python（参考 learn-claude-code 风格）

```python
import json
import time
import uuid
from dataclasses import dataclass, field, asdict
from typing import Any, Callable, Optional
from pathlib import Path
import anthropic

@dataclass
class ToolCallRecord:
    step_index: int
    request_id: str
    tool_name: str
    tool_input: dict
    result: Any = None
    error: Optional[str] = None
    duration_ms: int = 0
    timestamp: float = field(default_factory=time.time)

@dataclass
class AgentRecording:
    id: str = field(default_factory=lambda: str(uuid.uuid4()))
    session_id: str = ""
    initial_prompt: str = ""
    tool_calls: list[ToolCallRecord] = field(default_factory=list)
    llm_messages: list[dict] = field(default_factory=list)  # 完整对话历史
    start_time: float = field(default_factory=time.time)
    end_time: Optional[float] = None
    total_input_tokens: int = 0
    total_output_tokens: int = 0

    def save(self, path: Path):
        path.write_text(json.dumps(asdict(self), indent=2, default=str))

    @classmethod
    def load(cls, path: Path) -> "AgentRecording":
        data = json.loads(path.read_text())
        tool_calls = [ToolCallRecord(**tc) for tc in data.pop('tool_calls', [])]
        recording = cls(**{k: v for k, v in data.items() if k != 'tool_calls'})
        recording.tool_calls = tool_calls
        return recording


class RecordingAgentLoop:
    """带录制功能的 Agent Loop"""

    def __init__(self, tools: dict[str, Callable], model: str = "claude-3-5-sonnet-20241022"):
        self.tools = tools
        self.model = model
        self.client = anthropic.Anthropic()

    def run(self, prompt: str, session_id: str = "") -> AgentRecording:
        recording = AgentRecording(session_id=session_id, initial_prompt=prompt)
        messages = [{"role": "user", "content": prompt}]
        step_index = 0

        tool_schemas = [
            {"name": name, "description": f"Tool: {name}", "input_schema": {"type": "object", "properties": {}}}
            for name in self.tools
        ]

        while True:
            response = self.client.messages.create(
                model=self.model,
                max_tokens=4096,
                tools=tool_schemas,
                messages=messages
            )

            recording.total_input_tokens += response.usage.input_tokens
            recording.total_output_tokens += response.usage.output_tokens
            recording.llm_messages.append({
                "role": "assistant",
                "content": [block.model_dump() for block in response.content]
            })

            if response.stop_reason != "tool_use":
                break

            # 执行工具调用并录制
            tool_results = []
            for block in response.content:
                if block.type != "tool_use":
                    continue

                record = ToolCallRecord(
                    step_index=step_index,
                    request_id=block.id,
                    tool_name=block.name,
                    tool_input=block.input
                )

                start = time.time()
                try:
                    if block.name in self.tools:
                        record.result = self.tools[block.name](block.input)
                    else:
                        record.error = f"Unknown tool: {block.name}"
                except Exception as e:
                    record.error = str(e)
                finally:
                    record.duration_ms = int((time.time() - start) * 1000)

                recording.tool_calls.append(record)
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": str(record.error) if record.error else json.dumps(record.result)
                })
                step_index += 1

            messages.append({"role": "user", "content": tool_results})

        recording.end_time = time.time()
        return recording


def replay_recording(
    recording: AgentRecording,
    tools: dict[str, Callable],
    patch_step: Optional[int] = None,
    patch_result: Any = None
) -> list[ToolCallRecord]:
    """
    回放录制，可选修改特定步骤的结果。
    LLM 调用完全跳过——使用录制的响应。
    """
    replayed = []

    for record in recording.tool_calls:
        if patch_step is not None and record.step_index == patch_step:
            # 注入 patch
            replayed.append(ToolCallRecord(
                step_index=record.step_index,
                request_id=record.request_id,
                tool_name=record.tool_name,
                tool_input=record.tool_input,
                result=patch_result,
                duration_ms=0
            ))
            print(f"[Replay] Step {record.step_index} PATCHED: {record.tool_name}")
        else:
            # 使用录制结果（确定性，不实际调用）
            replayed.append(record)
            print(f"[Replay] Step {record.step_index} replayed: {record.tool_name} ({record.duration_ms}ms)")

    return replayed
```

---

## OpenClaw 集成：录制存到文件系统

```typescript
// OpenClaw 中的用法：用文件系统存录制
import * as fs from 'fs/promises';
import * as path from 'path';

const RECORDINGS_DIR = path.join(process.env.HOME!, '.openclaw/workspace/recordings');

const fileStore = {
  async save(recording: ExecutionRecording): Promise<void> {
    await fs.mkdir(RECORDINGS_DIR, { recursive: true });
    const file = path.join(RECORDINGS_DIR, `${recording.id}.json`);
    await fs.writeFile(file, JSON.stringify(recording, null, 2));
  },
  async load(id: string): Promise<ExecutionRecording | null> {
    try {
      const file = path.join(RECORDINGS_DIR, `${id}.json`);
      const data = await fs.readFile(file, 'utf-8');
      return JSON.parse(data);
    } catch {
      return null;
    }
  }
};

// 在 Cron 任务中自动录制
const ttDebugger = new TimeTravelDebugger(fileStore);

// 每次执行后保存录制
const recording = recorder.finish(finalResponse);
await ttDebugger.saveRecording(recording);

// 查看摘要
console.log(ttDebugger.summarize(recording));
// 找错误
const errorStep = ttDebugger.findErrorStep(recording);
if (errorStep) {
  console.log(`❌ 在步骤 ${errorStep.stepIndex} 出错: ${errorStep.data.toolName}`);
  console.log(`   错误信息: ${errorStep.data.error}`);
  
  // 修复工具后，只重跑这一步
  const replayer = new ExecutionReplayer(recording, realTools);
  const { events, divergedAt } = await replayer.replay({
    reExecuteFromStep: errorStep.stepIndex
  });
  console.log(`重放完成，从步骤 ${divergedAt} 开始产生分歧`);
}
```

---

## 典型调试工作流

```
1. 生产中某次 Agent 执行失败
        ↓
2. 从录制存储中加载该次 Recording
   const recording = await ttDebugger.store.load(failedExecutionId)
        ↓
3. 查看摘要，定位出错步骤
   const errorStep = ttDebugger.findErrorStep(recording)
   // → Step 7: search_database (timeout after 3000ms)
        ↓
4. 分析步骤 7 的输入，发现 SQL 查询太慢
   const callEvent = recording.events.find(e => e.stepIndex === 7)
   // → toolInput: { query: "SELECT * FROM orders WHERE..." }
        ↓
5. 修复 SQL，测试新结果
   const newResult = await optimizedSearch({ query: "..." })
        ↓
6. 用 patch 重放：只替换步骤 7，其余完全确定性复现
   const replayer = new ExecutionReplayer(recording, realTools)
   const { events } = await replayer.replay({
     patchToolResult: { stepIndex: 7, newResult }
   })
        ↓
7. 确认修复有效，推上线 ✅
```

---

## 关键设计原则

| 原则 | 说明 |
|------|------|
| **确定性优先** | 回放时对 LLM 使用录制结果，避免随机性 |
| **外科手术式修改** | 只改有问题的那一步，其余完全用录制数据 |
| **零成本回放** | 不调用 LLM API，不重跑工具，完全从录制还原 |
| **渐进重执行** | 可以设置从某步开始"接管"，之后实际执行 |
| **录制开销低** | 只存 JSON，不存二进制，压缩后通常 < 100KB |

---

## 小结

时间旅行调试把 Agent 的执行从"黑盒"变成"可回放的录像"：

- **生产故障**：复现率从 10% 提升到 100%（录制数据完全确定性）
- **调试效率**：不用重跑整个流程，直接跳到出错步骤
- **修复验证**：patch 一步，立刻看全局影响
- **性能分析**：`findSlowestStep()` 精确定位瓶颈

> 💡 **一句话**：录制不是为了审计，是为了让调试从"薛定谔的猫"变成"可控的实验"。
