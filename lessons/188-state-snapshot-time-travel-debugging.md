# 188 - Agent 状态快照与时间旅行调试（State Snapshot & Time-Travel Debugging）

> **核心思想：** Agent 的 bug 往往在第 17 步工具调用后才爆发——你根本不知道它怎么走到那里的。状态快照让你能「回到任意时间点」重放、检查、修复，把不可复现的 bug 变成可调试的问题。

---

## 为什么 Agent 调试这么难？

传统程序出 bug：打断点、看调用栈、复现。

Agent 出 bug：

```
用户: "帮我整理本月账单并发邮件给财务"
Agent: [调用了 11 个工具, 生成了 3 份报告, 写了 2 个文件]
Agent: 💥 TypeError: Cannot read property 'amount' of undefined
```

**你面对的问题：**
- 不知道第几步开始出错
- 重跑一遍结果不一定一样（LLM 非确定性）
- 中间工具有副作用（已发邮件、已写文件）
- token 成本高，不能无限重试

**时间旅行调试的核心能力：**

| 能力 | 说明 |
|------|------|
| Snapshot | 每步执行前保存完整状态 |
| Replay | 从任意快照重放后续步骤 |
| Branch | 从快照创建分支，测试不同决策路径 |
| Diff | 对比两个快照之间的状态变化 |
| Inspect | 检查任意时间点的 messages / tool results |

---

## 核心数据结构

```typescript
// TypeScript - 状态快照结构
interface AgentSnapshot {
  id: string;           // 快照 ID（步骤编号）
  timestamp: number;    // 时间戳
  step: number;         // 第几步
  label?: string;       // 可选标签（如 "before_email_send"）

  // Agent 核心状态
  messages: Message[];          // 完整对话历史
  toolResults: ToolResult[];    // 已执行工具结果
  variables: Record<string, any>; // 业务变量（账单数据、用户偏好等）

  // 元数据
  tokenUsage: { input: number; output: number };
  parentSnapshotId?: string;    // 用于分支追踪
}

interface TimelineEntry {
  snapshotId: string;
  action: "tool_call" | "llm_response" | "user_input" | "error";
  description: string;
  durationMs: number;
}
```

---

## TypeScript 实现：SnapshotManager

```typescript
import { createHash } from "crypto";
import { writeFileSync, readFileSync, existsSync, mkdirSync } from "fs";
import path from "path";

class AgentSnapshotManager {
  private snapshots = new Map<string, AgentSnapshot>();
  private timeline: TimelineEntry[] = [];
  private snapshotDir: string;
  private currentStep = 0;

  constructor(sessionId: string, persist = true) {
    this.snapshotDir = `./snapshots/${sessionId}`;
    if (persist) mkdirSync(this.snapshotDir, { recursive: true });
  }

  // 保存快照
  save(state: Omit<AgentSnapshot, "id" | "timestamp" | "step">, label?: string): string {
    const snapshot: AgentSnapshot = {
      ...state,
      id: `snap_${this.currentStep}_${Date.now()}`,
      timestamp: Date.now(),
      step: this.currentStep++,
      label,
    };

    this.snapshots.set(snapshot.id, snapshot);
    this.persist(snapshot);
    return snapshot.id;
  }

  // 从快照恢复
  restore(snapshotId: string): AgentSnapshot {
    const snapshot = this.snapshots.get(snapshotId) ?? this.loadFromDisk(snapshotId);
    if (!snapshot) throw new Error(`Snapshot ${snapshotId} not found`);
    return snapshot;
  }

  // 创建分支（从某个快照开岔，测试不同决策）
  branch(fromSnapshotId: string, branchLabel: string): AgentSnapshot {
    const base = this.restore(fromSnapshotId);
    return {
      ...structuredClone(base),
      id: `branch_${branchLabel}_${Date.now()}`,
      timestamp: Date.now(),
      parentSnapshotId: fromSnapshotId,
      label: branchLabel,
    };
  }

  // Diff 两个快照
  diff(snapshotA: string, snapshotB: string): SnapshotDiff {
    const a = this.restore(snapshotA);
    const b = this.restore(snapshotB);

    return {
      messagesAdded: b.messages.length - a.messages.length,
      toolsExecuted: b.toolResults.length - a.toolResults.length,
      tokenDelta: {
        input: b.tokenUsage.input - a.tokenUsage.input,
        output: b.tokenUsage.output - a.tokenUsage.output,
      },
      variableChanges: this.diffVariables(a.variables, b.variables),
      timeDeltaMs: b.timestamp - a.timestamp,
    };
  }

  // 列出时间线
  getTimeline(): TimelineEntry[] {
    return this.timeline;
  }

  // 记录时间线事件
  recordEvent(action: TimelineEntry["action"], description: string, durationMs: number) {
    this.timeline.push({
      snapshotId: `snap_${this.currentStep - 1}`,
      action,
      description,
      durationMs,
    });
  }

  private persist(snapshot: AgentSnapshot) {
    const filePath = path.join(this.snapshotDir, `${snapshot.id}.json`);
    writeFileSync(filePath, JSON.stringify(snapshot, null, 2));
  }

  private loadFromDisk(snapshotId: string): AgentSnapshot | null {
    const filePath = path.join(this.snapshotDir, `${snapshotId}.json`);
    if (!existsSync(filePath)) return null;
    const snap = JSON.parse(readFileSync(filePath, "utf-8")) as AgentSnapshot;
    this.snapshots.set(snap.id, snap);
    return snap;
  }

  private diffVariables(a: Record<string, any>, b: Record<string, any>) {
    const changes: Array<{ key: string; from: any; to: any }> = [];
    const allKeys = new Set([...Object.keys(a), ...Object.keys(b)]);
    for (const key of allKeys) {
      if (JSON.stringify(a[key]) !== JSON.stringify(b[key])) {
        changes.push({ key, from: a[key], to: b[key] });
      }
    }
    return changes;
  }
}
```

---

## 集成到 Agent Loop

```typescript
class DebuggableAgent {
  private snapshots: AgentSnapshotManager;
  private messages: Message[] = [];
  private variables: Record<string, any> = {};

  constructor(sessionId: string) {
    this.snapshots = new AgentSnapshotManager(sessionId);
  }

  async run(userInput: string): Promise<string> {
    this.messages.push({ role: "user", content: userInput });

    // 📸 快照：任务开始
    const startSnap = this.snapshots.save({
      messages: [...this.messages],
      toolResults: [],
      variables: { ...this.variables },
      tokenUsage: { input: 0, output: 0 },
    }, "task_start");

    let tokenUsage = { input: 0, output: 0 };
    const toolResults: ToolResult[] = [];
    let iteration = 0;

    while (iteration < 20) {
      // 📸 快照：每次 LLM 调用前
      const preCallSnap = this.snapshots.save({
        messages: [...this.messages],
        toolResults: [...toolResults],
        variables: { ...this.variables },
        tokenUsage: { ...tokenUsage },
      }, `before_llm_call_${iteration}`);

      const t0 = Date.now();
      const response = await callLLM(this.messages);
      const llmMs = Date.now() - t0;

      this.snapshots.recordEvent("llm_response", `iteration ${iteration}`, llmMs);
      tokenUsage.input += response.usage.input_tokens;
      tokenUsage.output += response.usage.output_tokens;

      if (!response.toolCalls?.length) {
        // 最终回复
        const finalSnap = this.snapshots.save({
          messages: [...this.messages],
          toolResults,
          variables: { ...this.variables },
          tokenUsage,
        }, "task_complete");

        // 输出调试摘要
        this.printDebugSummary(startSnap, finalSnap);
        return response.content;
      }

      // 执行工具
      for (const toolCall of response.toolCalls) {
        // 📸 快照：危险工具执行前（可打 label）
        const isDangerous = ["send_email", "delete_file", "make_payment"].includes(toolCall.name);
        if (isDangerous) {
          this.snapshots.save({
            messages: [...this.messages],
            toolResults: [...toolResults],
            variables: { ...this.variables },
            tokenUsage: { ...tokenUsage },
          }, `before_dangerous_${toolCall.name}`);
        }

        const t1 = Date.now();
        const result = await executeTool(toolCall);
        const toolMs = Date.now() - t1;

        this.snapshots.recordEvent("tool_call", `${toolCall.name}(${JSON.stringify(toolCall.args)})`, toolMs);
        toolResults.push(result);
        this.messages.push({ role: "tool", content: result.output });
      }

      iteration++;
    }

    throw new Error("Max iterations exceeded");
  }

  private printDebugSummary(startId: string, endId: string) {
    const diff = this.snapshots.diff(startId, endId);
    console.log(`\n📊 Debug Summary:`);
    console.log(`  Steps: ${diff.messagesAdded} messages, ${diff.toolsExecuted} tools`);
    console.log(`  Tokens: +${diff.tokenDelta.input} in / +${diff.tokenDelta.output} out`);
    console.log(`  Duration: ${diff.timeDeltaMs}ms`);
    console.log(`  Timeline:`);
    this.snapshots.getTimeline().forEach((e, i) =>
      console.log(`    ${i + 1}. [${e.action}] ${e.description} (${e.durationMs}ms)`)
    );
  }
}
```

---

## Python 版本：简洁实现

```python
import json
import copy
import time
from pathlib import Path
from dataclasses import dataclass, field, asdict
from typing import Optional, Any

@dataclass
class Snapshot:
    id: str
    step: int
    timestamp: float
    messages: list
    tool_results: list
    variables: dict
    token_usage: dict
    label: Optional[str] = None
    parent_id: Optional[str] = None

class SnapshotManager:
    def __init__(self, session_id: str, persist_dir: str = "./snapshots"):
        self.snapshots: dict[str, Snapshot] = {}
        self.step = 0
        self.persist_dir = Path(persist_dir) / session_id
        self.persist_dir.mkdir(parents=True, exist_ok=True)

    def save(self, messages, tool_results, variables, token_usage, label=None) -> str:
        snap = Snapshot(
            id=f"snap_{self.step}_{int(time.time() * 1000)}",
            step=self.step,
            timestamp=time.time(),
            messages=copy.deepcopy(messages),
            tool_results=copy.deepcopy(tool_results),
            variables=copy.deepcopy(variables),
            token_usage=dict(token_usage),
            label=label,
        )
        self.step += 1
        self.snapshots[snap.id] = snap
        # 持久化
        with open(self.persist_dir / f"{snap.id}.json", "w") as f:
            json.dump(asdict(snap), f, indent=2)
        return snap.id

    def restore(self, snap_id: str) -> Snapshot:
        if snap_id in self.snapshots:
            return self.snapshots[snap_id]
        # 从磁盘加载
        path = self.persist_dir / f"{snap_id}.json"
        data = json.loads(path.read_text())
        snap = Snapshot(**data)
        self.snapshots[snap_id] = snap
        return snap

    def branch(self, from_id: str, label: str) -> Snapshot:
        base = self.restore(from_id)
        branch = copy.deepcopy(base)
        branch.id = f"branch_{label}_{int(time.time() * 1000)}"
        branch.parent_id = from_id
        branch.label = label
        return branch

    def diff(self, id_a: str, id_b: str) -> dict:
        a, b = self.restore(id_a), self.restore(id_b)
        return {
            "messages_added": len(b.messages) - len(a.messages),
            "tools_executed": len(b.tool_results) - len(a.tool_results),
            "token_delta": {
                "input": b.token_usage.get("input", 0) - a.token_usage.get("input", 0),
                "output": b.token_usage.get("output", 0) - a.token_usage.get("output", 0),
            },
            "time_delta_ms": (b.timestamp - a.timestamp) * 1000,
            "variable_changes": [
                {"key": k, "from": a.variables.get(k), "to": b.variables.get(k)}
                for k in set(list(a.variables) + list(b.variables))
                if a.variables.get(k) != b.variables.get(k)
            ],
        }
```

---

## 实战场景：从失败快照重放

```typescript
// 场景：Agent 在第 8 步出错，但前 7 步都成功了
// 不需要从头重跑，直接从快照 7 继续

async function debugReplay(sessionId: string, fromSnapLabel: string) {
  const manager = new AgentSnapshotManager(sessionId);

  // 找到目标快照（按 label 查找）
  const allSnaps = manager.getAllByLabel(fromSnapLabel);
  const snap = allSnaps[0];

  console.log(`🔄 从快照 [${snap.label}] 重放，步骤 ${snap.step}`);
  console.log(`   Messages: ${snap.messages.length} 条`);
  console.log(`   Tools already done: ${snap.toolResults.length} 个`);
  console.log(`   Variables:`, snap.variables);

  // 注入修复后的状态（比如修正了有 bug 的变量）
  const fixedVariables = {
    ...snap.variables,
    invoiceData: fixBuggyInvoiceData(snap.variables.invoiceData), // 修复 bug
  };

  // 从修复后的状态重建 messages，继续运行
  const agent = new DebuggableAgent(sessionId + "_replay");
  await agent.runFromSnapshot({
    ...snap,
    variables: fixedVariables,
  });
}
```

---

## OpenClaw 中的时间旅行调试

OpenClaw 的 `exec` + `process` 工具天然支持类似模式：

```typescript
// OpenClaw 风格：长任务 + 快照检查点
async function longTaskWithCheckpoints(task: string) {
  const checkpoints: string[] = [];

  // 阶段 1：数据收集
  const result1 = await exec({ command: "gather_data.sh", yieldMs: 5000 });
  checkpoints.push(saveCheckpoint("data_gathered", result1));

  // 阶段 2：数据分析（可能耗时）
  const proc = await exec({ command: "analyze.py", background: true });
  // ... 等待 ...
  const result2 = await process({ action: "poll", sessionId: proc.id });
  checkpoints.push(saveCheckpoint("analysis_done", result2));

  // 阶段 3：生成报告（如果上一步出错，可以从 checkpoint 2 重来）
  try {
    await generateReport(result2);
  } catch (e) {
    console.log(`❌ 出错，回滚到快照: ${checkpoints[1]}`);
    const state = restoreCheckpoint(checkpoints[1]);
    await generateReportWithFix(state);
  }
}
```

---

## 快照存储策略

```typescript
// 不是每一步都要存，按策略控制
enum SnapshotPolicy {
  ALL           = "all",           // 每步都存（调试模式）
  MILESTONES    = "milestones",    // 只存关键里程碑
  BEFORE_DANGER = "before_danger", // 只在危险操作前存
  ADAPTIVE      = "adaptive",      // 根据任务复杂度自适应
}

class AdaptiveSnapshotPolicy {
  shouldSnapshot(context: {
    step: number;
    toolName?: string;
    isLlmCall: boolean;
    estimatedRemainingSteps: number;
  }): boolean {
    // 前 3 步总是存（热身阶段）
    if (context.step < 3) return true;

    // 危险工具前一定存
    const dangerousTools = ["send_email", "delete", "payment", "deploy"];
    if (context.toolName && dangerousTools.some(t => context.toolName!.includes(t))) return true;

    // 任务后半段每 3 步存一次
    if (context.step % 3 === 0) return true;

    // 剩余步数少（快完成了），每步都存
    if (context.estimatedRemainingSteps < 3) return true;

    return false;
  }
}
```

---

## 对比：和 Checkpoint / Event Sourcing 的区别

| 模式 | 粒度 | 目的 | 存储 |
|------|------|------|------|
| **Checkpoint** | 粗粒度（任务阶段） | 崩溃恢复 | 持久化为主 |
| **Event Sourcing** | 细粒度（每个事件） | 状态重建 + 审计 | 事件日志 |
| **Snapshot（本课）** | 中粒度（每步） | **调试 + 分支测试** | 内存 + 磁盘 |

三者互补：
- **Event Sourcing** 记录"发生了什么"
- **Snapshot** 记录"那一刻的完整状态"
- **Checkpoint** 记录"任务进度到哪了"

生产环境推荐组合：**Checkpoint（粗）+ Snapshot（危险操作前）+ Event Log（全量）**

---

## 关键指标监控

```typescript
// 快照本身也是可观测数据
interface SnapshotMetrics {
  totalSnapshots: number;
  avgSnapshotSizeKb: number;
  snapshotOverheadMs: number;    // 快照操作耗时占总耗时比例
  replayCount: number;           // 被重放了几次
  branchCount: number;           // 创建了几个分支
  mostReplayedLabel: string;     // 哪个快照最常被重放（= bug 最多的地方）
}
```

---

## 小结

| 要点 | 说明 |
|------|------|
| **快照时机** | 任务开始、危险操作前、每 N 步、任务结束 |
| **分支调试** | 从同一快照跑不同路径，对比结果 |
| **Diff 工具** | 快速定位「从哪步开始出问题」 |
| **持久化** | 快照写磁盘，跨进程重启后仍可调试 |
| **存储控制** | 自适应策略，调试模式全量，生产模式精简 |
| **副作用保护** | 危险工具前必存快照，失败可从上一步重试 |

**一句话记忆：** 快照 = Agent 的「存档点」，时间旅行调试 = 「读档」重玩，找到 bug 的那一步。

---

*下一课预告：Agent 响应质量自动评分（Response Quality Auto-Scoring）——如何让 Agent 知道自己回答得好不好。*
