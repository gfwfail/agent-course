# 224. Agent 运行手册自动化与安全执行（Runbook Automation & Safe Operations）

> 关键词：runbook、incident response、approval gate、dry-run、rollback、operational safety

上一课讲了 **会话状态迁移与兼容升级**：长会话 Agent 升级时，旧状态必须先 migration，再继续执行。

今天讲一个非常落地的生产能力：**让 Agent 按运行手册（Runbook）处理运维任务，但不让它乱动生产。**

很多团队都有这样的文档：

```txt
如果队列堆积：
1. 查看 queue depth
2. 查看 worker 是否在线
3. 如果 worker 崩了，重启 worker
4. 如果 DB 慢，先不要重启，升级人工
```

人执行时靠经验；Agent 执行时如果只把整段文档塞给 LLM，很容易变成：

```txt
LLM 看懂大概意思
→ 直接调用 restart_service
→ 没确认环境
→ 没 dry-run
→ 没记录证据
→ 事故扩大
```

所以今天的核心是：**Runbook 不能只是自然语言文档，它要变成可验证、可审批、可回滚的操作图。**

一句话：**Agent 做运维，不是“更大胆地自动化”，而是“把人类 SOP 变成带护栏的执行系统”。**

---

## 1. Runbook Agent 解决什么问题？

传统告警自动化通常是硬编码脚本：

```bash
if queue_depth > 10000; then systemctl restart worker; fi
```

它快，但不懂上下文。

纯 LLM Agent 则相反：

```txt
告警来了 → LLM 自己判断 → 自己找工具 → 自己执行
```

它灵活，但风险太大。

Runbook Agent 要走中间路线：

- **LLM 负责理解现场和选择分支**；
- **Runbook DSL 负责限制可执行步骤**；
- **Policy/Approval 负责控制副作用**；
- **Evidence Log 负责记录为什么这么做**；
- **Rollback Plan 负责失败后的退路**。

一个生产可用的 Runbook 至少要包含：

```yaml
id: queue-backlog
trigger: queue_depth_high
steps:
  - id: inspect_queue
    tool: get_queue_metrics
    effect: read
  - id: inspect_workers
    tool: list_workers
    effect: read
  - id: restart_workers
    tool: restart_worker_group
    effect: write
    requiresApproval: true
    precondition: "workers_down == true && db_healthy == true"
    rollback: "scale_workers_previous_count"
```

注意：**LLM 不能凭空发明操作步骤，只能在 Runbook 允许的步骤里选择。**

---

## 2. 核心模型：Runbook = DAG + Policy + Evidence

把 Runbook 看成一个受控 DAG：

```txt
告警输入
  → 收集证据（read-only）
  → 判断分支（LLM/规则）
  → dry-run 生成操作计划
  → 审批高风险步骤
  → 执行副作用
  → 验证结果
  → 必要时 rollback 或升级人工
```

关键设计有 5 条：

1. **默认只读**：诊断阶段只允许 read 工具。
2. **副作用分级**：write / external / destructive 分开处理。
3. **执行前 dry-run**：先生成计划，不直接动生产。
4. **每个结论绑定证据**：为什么判断 worker down？引用哪个工具结果？
5. **失败路径一等公民**：每个写操作要么可回滚，要么必须升级人工。

反模式：

```ts
await llm.decideAndExecute(alert, allTools);
```

正确模式：

```ts
const runbook = registry.get(alert.type);
const evidence = await collectReadOnlyEvidence(runbook, alert);
const plan = await planner.propose(runbook, evidence);
await policy.validate(plan);
await approvalGate.confirmIfNeeded(plan);
await executor.execute(plan);
```

---

## 3. learn-claude-code：Python 教学版

教学版先用 dict 定义 Runbook，不急着搞完整 DSL。

```python
# learn-claude-code/runbook.py
from dataclasses import dataclass, field
from typing import Any, Callable, Literal

Effect = Literal["read", "write", "destructive"]


@dataclass
class Step:
    id: str
    tool: str
    effect: Effect = "read"
    requires_approval: bool = False
    precondition: str | None = None
    rollback: str | None = None


@dataclass
class Runbook:
    id: str
    description: str
    steps: list[Step]


QUEUE_BACKLOG = Runbook(
    id="queue-backlog",
    description="队列堆积排查与恢复",
    steps=[
        Step("inspect_queue", "get_queue_metrics"),
        Step("inspect_workers", "list_workers"),
        Step("inspect_db", "get_db_health"),
        Step(
            "restart_workers",
            "restart_worker_group",
            effect="write",
            requires_approval=True,
            precondition="workers_down and db_healthy",
            rollback="restore_worker_count",
        ),
    ],
)
```

执行器分两段：先只读收集证据，再生成计划。

```python
# learn-claude-code/runbook_executor.py
from typing import Any

class RunbookExecutor:
    def __init__(self, tools, llm, approval_gate):
        self.tools = tools
        self.llm = llm
        self.approval_gate = approval_gate

    async def collect_evidence(self, runbook, alert: dict[str, Any]):
        evidence = {}
        for step in runbook.steps:
            if step.effect != "read":
                continue
            result = await self.tools.call(step.tool, {"alert": alert})
            evidence[step.id] = result
        return evidence

    async def propose_plan(self, runbook, alert, evidence):
        # LLM 只能选择 runbook 里已有 step id，不能发明工具
        allowed = [s.id for s in runbook.steps]
        prompt = {
            "task": "根据证据选择下一步 runbook steps",
            "allowedSteps": allowed,
            "alert": alert,
            "evidence": evidence,
            "output": "JSON: {steps:[...], reason, citations:[...]}"
        }
        return await self.llm.json(prompt)

    async def execute(self, runbook, alert):
        evidence = await self.collect_evidence(runbook, alert)
        plan = await self.propose_plan(runbook, alert, evidence)

        steps_by_id = {s.id: s for s in runbook.steps}
        for step_id in plan["steps"]:
            step = steps_by_id[step_id]

            if step.effect != "read":
                await self.approval_gate.require(
                    reason=plan["reason"],
                    step=step,
                    evidence=evidence,
                )

            result = await self.tools.call(step.tool, {"alert": alert, "evidence": evidence})
            evidence[step.id] = result

        return evidence
```

这里最重要的不是代码复杂度，而是边界：

- LLM 只能输出 step id；
- Tool executor 才能真正调用工具；
- write 步骤一定经过 approval gate；
- evidence 会持续累计，后续验证和复盘都靠它。

---

## 4. pi-mono：TypeScript 生产版

生产版建议把 Runbook 做成强类型 DSL + Policy Engine。

```ts
// pi-mono/agent/runbook/types.ts
import { z } from 'zod';

export const EffectSchema = z.enum(['read', 'write', 'external', 'destructive']);
export type Effect = z.infer<typeof EffectSchema>;

export const RunbookStepSchema = z.object({
  id: z.string(),
  tool: z.string(),
  effect: EffectSchema.default('read'),
  argsSchema: z.record(z.any()).optional(),
  requiresApproval: z.boolean().default(false),
  precondition: z.string().optional(),
  rollbackStepId: z.string().optional(),
});

export const RunbookSchema = z.object({
  id: z.string(),
  version: z.number(),
  trigger: z.string(),
  steps: z.array(RunbookStepSchema),
});

export type Runbook = z.infer<typeof RunbookSchema>;
```

Planner 输出必须被 schema 校验：

```ts
// pi-mono/agent/runbook/planner.ts
const PlanSchema = z.object({
  steps: z.array(z.object({
    stepId: z.string(),
    args: z.record(z.any()).default({}),
    reason: z.string(),
    evidenceIds: z.array(z.string()).default([]),
  })),
});

export async function planRunbook(input: {
  runbook: Runbook;
  evidence: Record<string, unknown>;
  llm: JsonLLM;
}) {
  const allowedStepIds = new Set(input.runbook.steps.map(s => s.id));

  const raw = await input.llm.json({
    system: 'You are a runbook planner. Choose only allowed stepId values.',
    runbook: input.runbook,
    evidence: input.evidence,
    outputSchema: PlanSchema.shape,
  });

  const plan = PlanSchema.parse(raw);

  for (const item of plan.steps) {
    if (!allowedStepIds.has(item.stepId)) {
      throw new Error(`Planner selected unknown runbook step: ${item.stepId}`);
    }
  }

  return plan;
}
```

Policy Engine 拦截高风险步骤：

```ts
// pi-mono/agent/runbook/policy.ts
export async function validatePlan(plan: Plan, runbook: Runbook, ctx: RunContext) {
  const byId = new Map(runbook.steps.map(s => [s.id, s]));

  for (const item of plan.steps) {
    const step = byId.get(item.stepId)!;

    if (step.effect === 'destructive') {
      throw new Error(`Destructive step requires human escalation: ${step.id}`);
    }

    if (step.effect !== 'read' && ctx.mode === 'diagnoseOnly') {
      throw new Error(`Write step not allowed in diagnoseOnly mode: ${step.id}`);
    }

    if ((step.effect === 'write' || step.effect === 'external') && !step.requiresApproval) {
      throw new Error(`Write/external step must declare requiresApproval: ${step.id}`);
    }
  }
}
```

执行器只负责照计划跑，不让 LLM 越权：

```ts
// pi-mono/agent/runbook/executor.ts
export async function executeRunbookPlan(input: {
  runbook: Runbook;
  plan: Plan;
  tools: ToolRegistry;
  approval: ApprovalGateway;
  audit: AuditLog;
}) {
  const steps = new Map(input.runbook.steps.map(s => [s.id, s]));

  for (const item of input.plan.steps) {
    const step = steps.get(item.stepId)!;

    await input.audit.append({ type: 'RUNBOOK_STEP_PLANNED', step, reason: item.reason });

    if (step.requiresApproval) {
      await input.approval.require({
        title: `Approve runbook step: ${step.id}`,
        risk: step.effect,
        reason: item.reason,
        evidenceIds: item.evidenceIds,
      });
    }

    const result = await input.tools.call(step.tool, item.args);
    await input.audit.append({ type: 'RUNBOOK_STEP_RESULT', stepId: step.id, result });
  }
}
```

生产版的关键点：**LLM 是 planner，不是 root shell。**

---

## 5. OpenClaw 实战：Heartbeat/Cron 里的运维 Runbook

OpenClaw 天然适合 Runbook 自动化，因为它有：

- `exec` / API 工具：收集指标、查日志；
- `message`：把审批或结果发到 Telegram；
- `sessions_spawn`：把长任务丢给隔离子会话；
- `memory/*.md`：记录执行证据和复盘；
- native approvals：危险命令可以挂起等人确认。

一个实用模式：

```txt
Cron/Heartbeat 收到告警
  → 读取 runbooks/queue-backlog.yml
  → 先执行 read-only 诊断
  → 写入 memory/incidents/INC-xxx.md
  → 如果需要 write 操作：发审批消息
  → 审批通过后执行
  → 再跑验证步骤
  → 更新 incident 文件，必要时总结给老板
```

Runbook 文件可以这样放：

```yaml
# runbooks/queue-backlog.yml
id: queue-backlog
version: 1
trigger: queue_depth_high
steps:
  - id: queue_metrics
    tool: exec
    effect: read
    command: "php artisan queue:monitor --json"

  - id: worker_status
    tool: exec
    effect: read
    command: "supervisorctl status"

  - id: restart_workers
    tool: exec
    effect: write
    requiresApproval: true
    command: "sudo supervisorctl restart worker:*"
    verify: "supervisorctl status"
```

注意一个铁律：**Runbook 里的命令也要分 effect。**

`queue:monitor` 是 read；`restart` 是 write；`rm -rf`、删库、改 DNS 这类必须 destructive，默认不自动执行。

---

## 6. 生产 Checklist

做 Runbook Agent 时，至少检查这 8 项：

1. **Runbook 有版本号**：`id + version`，执行记录保存当时版本。
2. **步骤有 effect 标记**：read/write/external/destructive。
3. **LLM 输出只允许 stepId**：不能自由生成 shell 命令。
4. **写操作先 dry-run**：把计划给人看，别直接动。
5. **审批携带证据**：人看到原因和指标，而不是一句“建议重启”。
6. **操作后必须 verify**：执行成功不等于问题解决。
7. **失败有 rollback/escalate**：没退路就不要自动执行。
8. **全程 audit log**：谁触发、看了什么、做了什么、结果如何。

---

## 7. 常见坑

### 坑 1：把 Runbook 文档直接塞给 LLM

这样 LLM 可能“理解”出文档里没有的步骤。文档可以作为解释，但真正执行必须走 DSL。

### 坑 2：诊断和修复混在一起

先 read-only 诊断，再进入 write 修复阶段。两段式能显著降低误操作。

### 坑 3：审批消息太抽象

不要发：

```txt
是否重启 worker？
```

要发：

```txt
队列深度 42,000，3/4 worker offline，DB health ok。
建议执行 restart_worker_group。
风险：write，可回滚：restore_worker_count。
```

### 坑 4：没有 verify

`restart` 命令返回 0，只代表命令执行成功，不代表业务恢复。必须再查 queue depth、worker status、error rate。

---

## 8. 总结

Runbook Automation 的本质不是让 Agent 自由发挥，而是把人类 SOP 工程化：

```txt
自然语言经验
  → 结构化 Runbook
  → 只读诊断
  → 受控计划
  → 审批执行
  → 结果验证
  → 审计复盘
```

learn-claude-code 里可以用 Python dict 快速教学；pi-mono 里应该做强类型 Runbook DSL + Policy Engine；OpenClaw 里则可以用 Cron/Heartbeat + approvals + memory incident log 把它跑起来。

最后记一句：**好的运维 Agent 不是不犯错，而是默认不被允许犯大错。**
