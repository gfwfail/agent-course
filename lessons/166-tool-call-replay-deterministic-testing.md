# 166 - Agent 工具调用重放与确定性测试（Tool Call Replay & Deterministic Testing）

> 核心思路：把生产环境的真实工具调用录制下来，回放时用录像代替真实 API，Agent 行为变得完全可重现、可 diff、可回归测试。

---

## 🎯 解决什么问题

Agent 测试最难的地方不是单元逻辑，而是**工具调用的不确定性**：
- 网络 API 每次返回不同内容
- LLM 输出本身有随机性
- 时间戳、ID、顺序都在变

传统 Mock 要手写假数据，但真实场景太复杂，Mock 容易写错，反而测不到真正的 Bug。

**解法：Record & Replay**
1. **Record 阶段**：生产流量中拦截工具调用，将 `(工具名, 参数, 返回值)` 序列化到文件
2. **Replay 阶段**：测试时用录像文件顶替真实工具，Agent 行为完全可预测

```
生产环境                            测试环境
┌─────────────────┐                ┌─────────────────┐
│  Agent Loop     │                │  Agent Loop     │
│  ↓              │  录制           │  ↓              │
│  Tool Dispatch  │──────────────→ │  Replay Engine  │
│  ↓              │   cassette.json│  ↓              │
│  真实 API 调用   │                │  从录像返回数据   │
└─────────────────┘                └─────────────────┘
```

---

## 🏗️ 核心实现

### TypeScript 录制器（pi-mono 风格）

```typescript
// tools/replay/recorder.ts

interface ToolCallRecord {
  id: string;          // UUID，保证顺序
  tool: string;        // 工具名
  input: unknown;      // 入参（序列化）
  output: unknown;     // 出参（序列化）
  durationMs: number;
  timestamp: string;
  error?: string;      // 若工具报错，记录错误
}

interface Cassette {
  sessionId: string;
  recordedAt: string;
  model: string;
  records: ToolCallRecord[];
}

export class ToolRecorder {
  private records: ToolCallRecord[] = [];
  private counter = 0;

  /** 包装任意工具函数，自动录制 */
  wrap<T extends (input: unknown) => Promise<unknown>>(
    toolName: string,
    fn: T
  ): T {
    return (async (input: unknown) => {
      const start = Date.now();
      const id = `${toolName}#${++this.counter}`;
      try {
        const output = await fn(input);
        this.records.push({
          id,
          tool: toolName,
          input,
          output,
          durationMs: Date.now() - start,
          timestamp: new Date().toISOString(),
        });
        return output;
      } catch (err: any) {
        this.records.push({
          id,
          tool: toolName,
          input,
          output: null,
          durationMs: Date.now() - start,
          timestamp: new Date().toISOString(),
          error: err.message,
        });
        throw err;
      }
    }) as T;
  }

  /** 序列化为 cassette 文件 */
  save(sessionId: string, model: string): Cassette {
    return {
      sessionId,
      recordedAt: new Date().toISOString(),
      model,
      records: this.records,
    };
  }
}
```

### Replay 引擎

```typescript
// tools/replay/replayer.ts

export class ToolReplayer {
  private queue: Map<string, ToolCallRecord[]> = new Map();

  constructor(cassette: Cassette) {
    // 按工具名建队列（FIFO）
    for (const record of cassette.records) {
      if (!this.queue.has(record.tool)) {
        this.queue.set(record.tool, []);
      }
      this.queue.get(record.tool)!.push(record);
    }
  }

  /** 生成一个可以注入 Tool Registry 的假工具 */
  mockTool(toolName: string) {
    return async (input: unknown): Promise<unknown> => {
      const records = this.queue.get(toolName);
      if (!records || records.length === 0) {
        throw new Error(`[Replay] No more records for tool: ${toolName}`);
      }

      const record = records.shift()!;

      // 可选：校验入参是否匹配（strict mode）
      const inputMatch = JSON.stringify(input) === JSON.stringify(record.input);
      if (!inputMatch) {
        console.warn(
          `[Replay] Input mismatch for ${toolName}:\n` +
          `  Expected: ${JSON.stringify(record.input)}\n` +
          `  Got:      ${JSON.stringify(input)}`
        );
      }

      // 模拟原始延迟（可关闭）
      // await sleep(record.durationMs);

      if (record.error) {
        throw new Error(record.error);
      }
      return record.output;
    };
  }

  /** 校验所有录像是否都被消费（防止 Agent 少调了工具） */
  assertAllConsumed() {
    const remaining: string[] = [];
    for (const [tool, records] of this.queue.entries()) {
      if (records.length > 0) {
        remaining.push(`${tool}: ${records.length} unused`);
      }
    }
    if (remaining.length > 0) {
      throw new Error(`[Replay] Unconsumed records:\n${remaining.join('\n')}`);
    }
  }
}
```

### 注入到 Agent Loop

```typescript
// agent/run-with-replay.ts

import { readFileSync } from 'fs';
import { ToolReplayer } from './replay/replayer';
import { ToolRegistry } from './tool-registry';
import { runAgentLoop } from './loop';

export async function runWithReplay(
  cassetteFile: string,
  userMessage: string
) {
  const cassette = JSON.parse(readFileSync(cassetteFile, 'utf-8'));
  const replayer = new ToolReplayer(cassette);

  // 用 Replay mock 替换真实工具
  const registry = new ToolRegistry();
  for (const toolName of registry.listTools()) {
    registry.override(toolName, replayer.mockTool(toolName));
  }

  const result = await runAgentLoop({ message: userMessage, registry });

  // 回放结束后校验
  replayer.assertAllConsumed();

  return result;
}
```

---

## 🐍 Python 版本（asyncio）

```python
# replay/cassette.py

import json
import asyncio
from dataclasses import dataclass, field, asdict
from typing import Any, Callable, Optional
from datetime import datetime, timezone
from collections import defaultdict

@dataclass
class ToolCallRecord:
    id: str
    tool: str
    input: Any
    output: Any
    duration_ms: int
    timestamp: str
    error: Optional[str] = None

@dataclass
class Cassette:
    session_id: str
    recorded_at: str
    model: str
    records: list[ToolCallRecord] = field(default_factory=list)

    def save(self, path: str):
        with open(path, 'w') as f:
            json.dump({
                'sessionId': self.session_id,
                'recordedAt': self.recorded_at,
                'model': self.model,
                'records': [asdict(r) for r in self.records],
            }, f, indent=2)

    @staticmethod
    def load(path: str) -> 'Cassette':
        with open(path) as f:
            data = json.load(f)
        return Cassette(
            session_id=data['sessionId'],
            recorded_at=data['recordedAt'],
            model=data['model'],
            records=[ToolCallRecord(**r) for r in data['records']],
        )


class ToolRecorder:
    def __init__(self):
        self._records: list[ToolCallRecord] = []
        self._counter = 0

    def wrap(self, tool_name: str, fn: Callable) -> Callable:
        async def wrapped(input_: Any) -> Any:
            self._counter += 1
            record_id = f"{tool_name}#{self._counter}"
            start = asyncio.get_event_loop().time()
            try:
                output = await fn(input_)
                self._records.append(ToolCallRecord(
                    id=record_id,
                    tool=tool_name,
                    input=input_,
                    output=output,
                    duration_ms=int((asyncio.get_event_loop().time() - start) * 1000),
                    timestamp=datetime.now(timezone.utc).isoformat(),
                ))
                return output
            except Exception as e:
                self._records.append(ToolCallRecord(
                    id=record_id,
                    tool=tool_name,
                    input=input_,
                    output=None,
                    duration_ms=int((asyncio.get_event_loop().time() - start) * 1000),
                    timestamp=datetime.now(timezone.utc).isoformat(),
                    error=str(e),
                ))
                raise

        return wrapped

    def to_cassette(self, session_id: str, model: str) -> Cassette:
        return Cassette(
            session_id=session_id,
            recorded_at=datetime.now(timezone.utc).isoformat(),
            model=model,
            records=self._records,
        )


class ToolReplayer:
    def __init__(self, cassette: Cassette):
        self._queues: dict[str, list[ToolCallRecord]] = defaultdict(list)
        for r in cassette.records:
            self._queues[r.tool].append(r)

    def mock_tool(self, tool_name: str) -> Callable:
        async def mock(input_: Any) -> Any:
            queue = self._queues.get(tool_name, [])
            if not queue:
                raise RuntimeError(f"[Replay] No more records for: {tool_name}")
            record = queue.pop(0)
            if record.error:
                raise RuntimeError(record.error)
            return record.output
        return mock

    def assert_all_consumed(self):
        leftovers = {k: len(v) for k, v in self._queues.items() if v}
        if leftovers:
            raise AssertionError(f"[Replay] Unconsumed records: {leftovers}")
```

---

## 🧪 测试示例（Vitest / pytest）

```typescript
// tests/replay.test.ts
import { describe, it, expect } from 'vitest';
import { runWithReplay } from '../agent/run-with-replay';

describe('Agent 回归测试 - 邮件摘要任务', () => {
  it('应当正确摘要 3 封邮件', async () => {
    const result = await runWithReplay(
      'cassettes/email-summary-2026-03-01.json',
      '帮我摘要今天的邮件'
    );

    // 断言最终输出包含关键信息
    expect(result.text).toMatch(/3 封邮件/);
    expect(result.toolCalls).toHaveLength(4); // list + 3x get
  });
});
```

```python
# tests/test_replay.py
import pytest
from replay.cassette import Cassette, ToolReplayer

@pytest.mark.asyncio
async def test_weather_agent_replay():
    cassette = Cassette.load("cassettes/weather-query-2026-03-01.json")
    replayer = ToolReplayer(cassette)
    
    # 注入 mock 工具到 Agent
    agent = WeatherAgent(tools={
        "get_weather": replayer.mock_tool("get_weather"),
        "send_message": replayer.mock_tool("send_message"),
    })
    
    result = await agent.run("今天北京天气怎么样？")
    
    assert "晴" in result.text or "多云" in result.text
    replayer.assert_all_consumed()  # 确保所有录像都被使用
```

---

## 🔧 OpenClaw 实战：录制 Cron 任务

OpenClaw 的 Cron 任务在隔离 session 中运行，天然适合录制。可以加一个环境变量：

```typescript
// openclaw/tool-middleware.ts（示意）

const RECORD_MODE = process.env.AGENT_RECORD === '1';
const REPLAY_FILE = process.env.AGENT_CASSETTE;

export function buildToolMiddleware(toolName: string, fn: ToolFn): ToolFn {
  if (RECORD_MODE) {
    return globalRecorder.wrap(toolName, fn);
  }
  if (REPLAY_FILE) {
    return globalReplayer.mockTool(toolName);
  }
  return fn;
}
```

录制：
```bash
AGENT_RECORD=1 openclaw run --task "每日报表任务"
# 任务结束后自动保存 cassette.json
```

回放（CI 中）：
```bash
AGENT_CASSETTE=cassettes/daily-report.json \
  vitest run tests/daily-report.test.ts
```

---

## 🎯 与 Mock 测试的区别

| 维度 | Mock（手写） | Replay（录制） |
|------|-------------|--------------|
| 数据来源 | 手写假数据 | 真实生产数据 |
| 维护成本 | 高（场景变了要改 Mock） | 低（重新录制即可） |
| 覆盖率 | 取决于你能想到多少场景 | 自动覆盖真实路径 |
| 适合阶段 | 纯逻辑单元测试 | 端到端回归测试 |
| 调试体验 | 差（假数据看不出真问题） | 好（真实报错可复现） |

> 两者互补：Unit Test 用 Mock，Regression 用 Replay。

---

## 💡 Cassette 文件管理建议

```
cassettes/
├── prod/                    # 生产录制（自动，按日期）
│   ├── 2026-03-01-email-summary.json
│   └── 2026-03-02-weather.json
├── curated/                 # 精选 Cassette（手动挑出典型场景）
│   ├── edge-case-empty-inbox.json
│   └── error-api-timeout.json
└── .gitattributes           # 大文件用 Git LFS
```

- **自动录制**：Cron 任务默认录制，按日轮转，保留 7 天
- **精选场景**：发现 Bug 时立即录制当时的 Cassette，提交到 `curated/`
- **CI 回归**：只跑 `curated/` 目录，稳定不依赖网络

---

## 🔑 关键要点

1. **录制 = 最廉价的测试数据收集**：真实场景比你手写的 Mock 复杂 10 倍
2. **Replay 让 CI 无需网络**：离线运行，速度快，不消耗 API 配额
3. **用 `assertAllConsumed()` 防止 Agent 少做事**：确保所有工具都被调用
4. **录制时脱敏**：PII Masking（第 93 课）先处理再保存 Cassette
5. **与 CI/CD 集成**（第 158 课）：每次部署前跑 Cassette 回归，行为漂移自动拦截

---

*下一课预告：Agent 工具调用热路径优化（Hot Path Optimization）——找到最频繁的工具调用模式，针对性缓存和并发，用最小改动拿最大收益。*
