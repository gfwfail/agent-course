# 138 - Agent 工具调用回放与离线调试（Tool Call Replay & Offline Debugging）

> 线上出了 Bug，你要怎么复现？重新跑一遍 Agent 既费钱又不稳定。**录制 → 回放** 才是工程师的正确姿势。

---

## 为什么需要回放调试？

Agent 运行时的问题很难复现：
- LLM 输出有随机性，同样的输入不一定产生同样的工具调用
- 真实用户数据不能随便拿来重跑（隐私/成本）
- 调试某个工具的 Bug 却要等 30 秒 LLM 响应，效率极低

**解法：把 Agent 的工具调用序列录下来，离线回放。**

```
Live Session:  User → LLM → [tool_call_1, tool_call_2, ...] → Response
                                      ↓ 录制
Replay Session: 直接注入 tool_call_1, tool_call_2 → 跳过 LLM → 分析工具行为
```

---

## 核心设计

### 1. 录制层（Recorder）

```typescript
// recorder.ts
export interface ToolCallRecord {
  id: string;           // tool_use block id
  name: string;         // 工具名
  input: unknown;       // 原始入参
  output: unknown;      // 真实返回值
  durationMs: number;   // 耗时
  timestamp: string;    // ISO 时间
  error?: string;       // 如果报错
}

export interface SessionRecording {
  sessionId: string;
  startedAt: string;
  model: string;
  toolCalls: ToolCallRecord[];
}

export class ToolRecorder {
  private records: ToolCallRecord[] = [];

  // 包装任意工具执行器
  wrap<T extends (input: unknown) => Promise<unknown>>(
    name: string,
    fn: T
  ): T {
    return (async (input: unknown) => {
      const id = crypto.randomUUID();
      const start = Date.now();
      try {
        const output = await fn(input);
        this.records.push({
          id, name, input, output,
          durationMs: Date.now() - start,
          timestamp: new Date().toISOString(),
        });
        return output;
      } catch (err) {
        this.records.push({
          id, name, input, output: null,
          durationMs: Date.now() - start,
          timestamp: new Date().toISOString(),
          error: String(err),
        });
        throw err;
      }
    }) as T;
  }

  save(path: string): void {
    const recording: SessionRecording = {
      sessionId: crypto.randomUUID(),
      startedAt: new Date().toISOString(),
      model: process.env.MODEL ?? 'unknown',
      toolCalls: this.records,
    };
    fs.writeFileSync(path, JSON.stringify(recording, null, 2));
    console.log(`📼 录制完成，共 ${this.records.length} 次工具调用`);
  }
}
```

---

### 2. 回放层（Replayer）

```typescript
// replayer.ts
export class ToolReplayer {
  private cursor = 0;

  constructor(private recording: SessionRecording) {}

  // 替换真实工具执行器：直接返回录制的结果
  buildMockRegistry(): Record<string, (input: unknown) => Promise<unknown>> {
    const registry: Record<string, (input: unknown) => Promise<unknown>> = {};

    for (const record of this.recording.toolCalls) {
      // 同名工具可能被多次调用，按顺序消费
      registry[record.name] = async (_input: unknown) => {
        const rec = this.findNext(record.name);
        if (!rec) throw new Error(`回放耗尽：没有更多 ${record.name} 的记录`);

        // 模拟真实耗时（可选：加速模式直接跳过）
        // await sleep(rec.durationMs);

        if (rec.error) throw new Error(`[录制错误] ${rec.error}`);
        return rec.output;
      };
    }

    return registry;
  }

  private findNext(name: string): ToolCallRecord | null {
    for (let i = this.cursor; i < this.recording.toolCalls.length; i++) {
      if (this.recording.toolCalls[i].name === name) {
        this.cursor = i + 1;
        return this.recording.toolCalls[i];
      }
    }
    return null;
  }

  // 差异对比：对比录制结果 vs 新代码结果
  async diffRun(
    newRegistry: Record<string, (input: unknown) => Promise<unknown>>
  ): Promise<void> {
    console.log('\n🔍 差异对比模式\n');
    for (const record of this.recording.toolCalls) {
      const fn = newRegistry[record.name];
      if (!fn) {
        console.log(`⚠️  工具 ${record.name} 在新注册表中不存在`);
        continue;
      }
      try {
        const newOutput = await fn(record.input);
        const old = JSON.stringify(record.output);
        const newStr = JSON.stringify(newOutput);
        if (old === newStr) {
          console.log(`✅ ${record.name} - 输出一致`);
        } else {
          console.log(`❌ ${record.name} - 输出不同！`);
          console.log('   录制:', old.slice(0, 200));
          console.log('   新版:', newStr.slice(0, 200));
        }
      } catch (err) {
        console.log(`💥 ${record.name} - 新版抛出错误: ${err}`);
      }
    }
  }
}
```

---

### 3. 接入 Agent Loop

```typescript
// agent.ts - 录制模式
const recorder = new ToolRecorder();

const tools = {
  search_web:   recorder.wrap('search_web',   searchWeb),
  read_file:    recorder.wrap('read_file',    readFile),
  execute_code: recorder.wrap('execute_code', executeCode),
};

await runAgentLoop({ tools, messages });
recorder.save('./recordings/session-2026-03-26.json');

// ---

// agent.ts - 回放模式
const recording = JSON.parse(fs.readFileSync('./recordings/session-2026-03-26.json', 'utf8'));
const replayer = new ToolReplayer(recording);
const mockTools = replayer.buildMockRegistry();

// 完全跳过 LLM，只跑工具逻辑
for (const record of recording.toolCalls) {
  console.log(`▶️  回放 ${record.name}`);
  const result = await mockTools[record.name](record.input);
  console.log('结果:', JSON.stringify(result).slice(0, 100));
}
```

---

### 4. Python 版（learn-claude-code 风格）

```python
# recorder.py
import json, time, functools
from dataclasses import dataclass, asdict
from pathlib import Path
from typing import Any, Callable

@dataclass
class ToolCallRecord:
    name: str
    input: Any
    output: Any
    duration_ms: int
    timestamp: str
    error: str | None = None

class ToolRecorder:
    def __init__(self):
        self.records: list[ToolCallRecord] = []

    def wrap(self, name: str, fn: Callable) -> Callable:
        @functools.wraps(fn)
        def wrapped(input_data):
            start = time.time()
            try:
                result = fn(input_data)
                self.records.append(ToolCallRecord(
                    name=name, input=input_data, output=result,
                    duration_ms=int((time.time() - start) * 1000),
                    timestamp=time.strftime('%Y-%m-%dT%H:%M:%SZ'),
                ))
                return result
            except Exception as e:
                self.records.append(ToolCallRecord(
                    name=name, input=input_data, output=None,
                    duration_ms=int((time.time() - start) * 1000),
                    timestamp=time.strftime('%Y-%m-%dT%H:%M:%SZ'),
                    error=str(e),
                ))
                raise
        return wrapped

    def save(self, path: str):
        Path(path).write_text(json.dumps(
            [asdict(r) for r in self.records], indent=2, ensure_ascii=False
        ))
        print(f"📼 录制完成，共 {len(self.records)} 次工具调用 → {path}")


class ToolReplayer:
    def __init__(self, records: list[dict]):
        self.records = records
        self.cursor = 0

    def next(self, name: str) -> Any:
        for i in range(self.cursor, len(self.records)):
            if self.records[i]['name'] == name:
                self.cursor = i + 1
                rec = self.records[i]
                if rec.get('error'):
                    raise RuntimeError(f"[录制错误] {rec['error']}")
                return rec['output']
        raise StopIteration(f"回放耗尽：没有更多 {name} 的记录")
```

---

### 5. OpenClaw 场景：自动录制失败任务

```typescript
// 在 cron job 的 isolated sub-agent 里
// 用环境变量控制录制/回放模式
const MODE = process.env.AGENT_MODE ?? 'live'; // 'live' | 'record' | 'replay'

let tools = buildRealTools();

if (MODE === 'record') {
  const recorder = new ToolRecorder();
  tools = wrapAllTools(tools, recorder);
  process.on('exit', () => recorder.save(`./recordings/${Date.now()}.json`));
}

if (MODE === 'replay') {
  const file = process.env.REPLAY_FILE!;
  const recording = JSON.parse(fs.readFileSync(file, 'utf8'));
  const replayer = new ToolReplayer(recording);
  tools = replayer.buildMockRegistry();
}

await runAgentLoop({ tools });
```

---

## 实战技巧

| 场景 | 做法 |
|------|------|
| 复现线上 Bug | 生产录制，本地回放，加断点 |
| 新工具版本测试 | `replayer.diffRun(newRegistry)` 差异对比 |
| CI 单元测试 | 录制 fixture，回放作为 golden test |
| 性能分析 | 解析 `durationMs`，找慢工具 |
| 成本分析 | 统计各工具调用次数，估算 API 费用 |

---

## 关键注意事项

1. **顺序敏感**：同名工具按调用顺序消费，乱序回放会出错
2. **副作用工具**：写文件、发消息等工具在回放时应该 mock 掉，避免重复执行
3. **时间相关**：回放时 `Date.now()` 仍是当前时间，注意时间窗口逻辑
4. **录制文件安全**：可能包含敏感数据，加密存储或自动脱敏

---

## 小结

```
录制 = 给 Agent 装监控摄像头
回放 = 不用 LLM 就能复现工具行为
差异对比 = 改了代码后验证工具输出没变
```

工具调用回放是 Agent 调试工具链的**基础设施**。结合第 104 课（工具 Mock）和第 88 课（OpenTelemetry），能建立完整的 Agent 可观测性和离线调试体系。

> 下次线上出 Bug，第一步不是加 `console.log`，而是把那次 session 的录制文件拿来回放。
