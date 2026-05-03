# 233. Agent 完成判定与验收闸门（Completion Gates & Done Criteria）

> 核心思想：Agent 不应该靠“感觉差不多了”结束任务，而应该靠一组可执行的 Done Criteria 通过验收。完成判定要工程化：每个任务有可检查的验收项，每个验收项有证据，失败就继续修，无法修才明确标记 blocked。

很多 Agent Demo 看起来聪明，生产里却经常翻车：

- 用户要求“写文件 + 更新目录 + push”，Agent 写了文件就结束；
- 代码改完没跑测试，却回复“已完成”；
- 外部消息发出去了，但本地 repo 没提交；
- 多步骤任务做到 80%，最后漏掉最无聊但最关键的验证。

这类问题不是模型不聪明，而是系统没有“完成闸门”。

---

## 1. 完成闸门的最小模型

一个任务应该拆成三层：

```text
Task
  ├─ intent: 用户到底要什么
  ├─ doneCriteria: 完成条件列表
  └─ gates: 可执行验收闸门
```

示例：

```ts
type GateStatus = 'passed' | 'failed' | 'blocked';

type CompletionGate = {
  id: string;
  description: string;
  required: boolean;
  check: () => Promise<{
    status: GateStatus;
    evidence?: string;
    reason?: string;
  }>;
};
```

关键点：

1. **required gate 不通过，不能说完成**；
2. **每个 gate 都要有 evidence**，例如测试日志、git diff、messageId、文件路径；
3. **blocked 必须说明缺什么输入**，不能伪装成成功；
4. **完成前最后跑一次闸门**，防止中途状态变化。

---

## 2. learn-claude-code：Python 教学版

在教学版里，可以先不用复杂框架，用普通函数表达完成条件：

```python
from dataclasses import dataclass
from typing import Callable, Awaitable, Literal

GateStatus = Literal["passed", "failed", "blocked"]

@dataclass
class GateResult:
    status: GateStatus
    evidence: str = ""
    reason: str = ""

@dataclass
class CompletionGate:
    id: str
    description: str
    required: bool
    check: Callable[[], Awaitable[GateResult]]

async def run_completion_gates(gates: list[CompletionGate]) -> tuple[bool, list[GateResult]]:
    results: list[GateResult] = []

    for gate in gates:
        result = await gate.check()
        results.append(result)

        if gate.required and result.status != "passed":
            return False, results

    return True, results
```

用在 Agent Loop 里：

```python
async def finish_task(task, gates: list[CompletionGate]):
    ok, results = await run_completion_gates(gates)

    if ok:
        return {
            "status": "completed",
            "evidence": [r.evidence for r in results if r.evidence],
        }

    return {
        "status": "not_done",
        "failed_gates": [
            {"status": r.status, "reason": r.reason, "evidence": r.evidence}
            for r in results
            if r.status != "passed"
        ],
    }
```

这会强迫 Agent 在回复前先问自己：

> 用户要求的每一件事，都有证据证明完成了吗？

---

## 3. pi-mono：生产版 CompletionGate 中间件

生产系统可以把它做成 Task Runner 的中间件：

```ts
type GateResult = {
  status: 'passed' | 'failed' | 'blocked';
  evidence?: string;
  reason?: string;
};

type CompletionGate<TContext> = {
  id: string;
  required: boolean;
  run: (ctx: TContext) => Promise<GateResult>;
};

export class CompletionGateRunner<TContext> {
  constructor(private gates: CompletionGate<TContext>[]) {}

  async verify(ctx: TContext) {
    const results: Record<string, GateResult> = {};

    for (const gate of this.gates) {
      const result = await gate.run(ctx);
      results[gate.id] = result;

      if (gate.required && result.status !== 'passed') {
        return {
          ok: false,
          failedGate: gate.id,
          results,
        };
      }
    }

    return { ok: true, results };
  }
}
```

给“改代码任务”配置 gates：

```ts
const codeChangeGates: CompletionGate<RunContext>[] = [
  {
    id: 'diff-exists',
    required: true,
    run: async (ctx) => {
      const diff = await ctx.shell('git diff --stat');
      return diff.stdout.trim()
        ? { status: 'passed', evidence: diff.stdout }
        : { status: 'failed', reason: '没有检测到代码变更' };
    },
  },
  {
    id: 'tests-pass',
    required: true,
    run: async (ctx) => {
      const result = await ctx.shell('npm test', { timeoutMs: 120_000 });
      return result.code === 0
        ? { status: 'passed', evidence: 'npm test passed' }
        : { status: 'failed', reason: result.stderr.slice(-1000) };
    },
  },
  {
    id: 'working-tree-clean-or-explained',
    required: true,
    run: async (ctx) => {
      const status = await ctx.shell('git status --short');
      return status.stdout.trim()
        ? { status: 'blocked', reason: `还有未处理变更:\n${status.stdout}` }
        : { status: 'passed', evidence: 'working tree clean' };
    },
  },
];
```

然后 Task Runner 在最终回复前统一执行：

```ts
const gates = new CompletionGateRunner(codeChangeGates);
const verification = await gates.verify(ctx);

if (!verification.ok) {
  return ctx.respond({
    status: 'blocked',
    message: `任务还没完成，卡在 gate: ${verification.failedGate}`,
    evidence: verification.results,
  });
}

return ctx.respond({ status: 'completed', evidence: verification.results });
```

这比在 prompt 里写“请确保完成所有步骤”靠谱得多。

---

## 4. OpenClaw 实战：课程 cron 的完成闸门

以这个课程 cron 为例，完成条件不是“写了一篇文章”，而是：

```text
required gates:
1. lesson file exists
2. README contains lesson link
3. TOOLS.md 已讲内容包含新主题
4. Telegram group message sent and returned messageId
5. git commit exists
6. git push succeeded
```

可以写成一个轻量检查脚本：

```bash
LESSON="lessons/233-completion-gates-done-criteria.md"
TOPIC="Agent 完成判定与验收闸门"

[ -f "$LESSON" ] || { echo "missing lesson"; exit 1; }
grep -q "233-completion-gates-done-criteria" README.md || { echo "README missing"; exit 1; }
grep -q "$TOPIC" ../TOOLS.md || { echo "TOOLS missing"; exit 1; }
git log -1 --oneline || { echo "no commit"; exit 1; }
git status --short
```

OpenClaw 里实际工作流可以是：

1. `read`/`exec` 检查上一课编号和已讲列表；
2. `write` 新 lesson；
3. `edit` 更新 README/TOOLS；
4. `message` 发群；
5. `exec` git add/commit/push；
6. 最后跑一次 grep/status 作为 completion gate。

重点是：**最终回复不能早于 gates 通过。**

---

## 5. 常见坑

### 坑 1：把“动作成功”误认为“任务完成”

`write file ok` 只证明文件写了，不证明 README 更新了、消息发了、git 推了。

### 坑 2：只做正向检查，不做反向检查

比如测试通过了，但 `git status --short` 还有未提交文件。完成闸门应该检查“没有遗留”。

### 坑 3：blocked 不透明

不要说“遇到问题了”。要说：

```json
{
  "status": "blocked",
  "gate": "tests-pass",
  "reason": "npm test 缺少 DATABASE_URL",
  "neededInput": "提供测试数据库或允许跳过集成测试"
}
```

### 坑 4：让 LLM 自己记 gates

Gate 应该在代码/任务状态里，不应该只存在于模型上下文里。上下文会被裁剪，代码不会忘。

---

## 6. 设计 Checklist

- [ ] 用户要求是否被转成明确 done criteria？
- [ ] 每个 required criteria 是否有可执行 gate？
- [ ] 每个 gate 是否返回 evidence？
- [ ] 最终回复前是否重新跑 gate？
- [ ] failed/blocked 是否阻止“完成”状态？
- [ ] blocked 是否说明缺什么输入？
- [ ] gate 结果是否写入日志，方便复盘？

---

## 总结

成熟 Agent 的结束语不应该是“我觉得完成了”，而应该是：

> 我通过了哪些验收闸门，因此可以证明任务完成。

Completion Gates 把“完成”从主观判断变成工程事实。越是长任务、自动化任务、会产生外部副作用的任务，越需要这层闸门。
