# 346. Agent 修复工单与 Worktree 隔离（Repair Ticket & Worktree Isolation）

上一课讲了 replay triage：把 stable_fail、flaky_quarantined、missing_coverage 分流成 RepairTicket。

今天继续往后走一步：**RepairTicket 不能只是一个 issue 标题，它必须能安全地启动修复工作，而且每个修复都要有清晰写入边界。**

成熟 Agent 的自动修复链路应该是：

~~~text
Regression Replay
  -> RepairTicket
  -> Worktree Lane
  -> Bounded Worker Prompt
  -> Verification Gate
  -> Merge / Closeout / Escalate
~~~

核心原则：**ticket 是控制平面，worktree 是执行平面，verification 是合并闸门。**

## 1. 为什么不能直接让 Agent “修一下”

回归失败后，最危险的派单方式是：

~~~text
这个 replay case 挂了，你去修一下。
~~~

这种指令太宽，子 Agent 很容易做三件坏事：

- 顺手改了测试，让红变绿，但产品行为没修。
- 在同一个目录里和别的修复互相覆盖。
- 修完没有证明 expected decision、tool chain、evidence refs 是否恢复。

所以 RepairTicket 至少要带这些字段：

- caseId：哪条回归失败。
- riskSurface：命中的风险面，例如 side_effect_policy、evidence_egress、tool_dispatch。
- writeScope：允许修改哪些文件或模块。
- forbiddenScope：明确不能碰哪些文件。
- expectedRepair：修行为、刷 fixture、补 coverage，还是隔离 flaky。
- verification：修完必须跑哪些 replay/test。
- evidenceRefs：失败证据在哪里。

## 2. learn-claude-code：把 ticket 绑定到 worktree

learn-claude-code 的 s12_worktree_task_isolation.py 已经有很适合教学的结构：

- `.tasks/task_12.json` 是任务控制面。
- `.worktrees/index.json` 是执行目录登记。
- `task_bind_worktree()` 把两者绑定。
- `worktree_events()` 记录生命周期事件。

我们可以在 RepairTicket 生成后，创建一个隔离 lane：

~~~python
from dataclasses import dataclass, asdict
from pathlib import Path
import json

@dataclass
class RepairTicket:
    id: str
    kind: str
    case_id: str
    risk_surface: str
    priority: str
    write_scope: list[str]
    forbidden_scope: list[str]
    verification: list[str]
    evidence_refs: list[str]

def create_repair_task(ticket: RepairTicket, tasks_dir: Path) -> Path:
    tasks_dir.mkdir(parents=True, exist_ok=True)
    path = tasks_dir / f"repair_{ticket.id}.json"
    path.write_text(json.dumps({
        **asdict(ticket),
        "status": "pending",
        "owner": "repair-worker",
        "worktree": "",
    }, indent=2), encoding="utf-8")
    return path

def build_worker_prompt(ticket: RepairTicket) -> str:
    return f"""
你是回归修复 worker。你不是独自在代码库里工作，可能有其他 worker 并行修改别的范围。

任务：
- 修复 replay case: {ticket.case_id}
- 风险面: {ticket.risk_surface}
- 工单类型: {ticket.kind}

允许写入：
{chr(10).join('- ' + p for p in ticket.write_scope)}

禁止修改：
{chr(10).join('- ' + p for p in ticket.forbidden_scope)}

必须验证：
{chr(10).join('- ' + cmd for cmd in ticket.verification)}

证据：
{chr(10).join('- ' + ref for ref in ticket.evidence_refs)}

完成时只汇报：
1. 改了哪些文件
2. replay 是否恢复
3. 还有哪些风险没关闭
""".strip()
~~~

这里的重点不是 Python 文件写入，而是结构：**LLM 收到的是一个带边界的修复合同，而不是一句模糊要求。**

在 s12 的模型里，这个 ticket 接下来会进入：

~~~text
task_create -> worktree_create -> task_bind_worktree -> worktree_run -> worktree_events
~~~

这样每个修复都有独立目录、独立生命周期事件、独立 closeout。

## 3. pi-mono：用 Agent.subscribe 观察修复过程

pi-mono 的 agent 包暴露了 `AgentEvent`：

~~~ts
type AgentEvent =
  | { type: "agent_start" }
  | { type: "agent_end"; messages: AgentMessage[] }
  | { type: "turn_start" }
  | { type: "turn_end"; message: AgentMessage; toolResults: ToolResultMessage[] }
  | { type: "tool_execution_start"; toolCallId: string; toolName: string; args: any }
  | { type: "tool_execution_end"; toolCallId: string; toolName: string; result: any; isError: boolean };
~~~

`Agent` 本身支持订阅事件：

~~~ts
agent.subscribe((event) => {
  // 外层系统可以记录工具调用、验证命令、最终消息
});
~~~

所以 Repair Worker 不需要侵入核心 loop。外层 controller 可以包一层：

~~~ts
type RepairTicket = {
  id: string;
  caseId: string;
  riskSurface: string;
  writeScope: string[];
  forbiddenScope: string[];
  verification: string[];
  evidenceRefs: string[];
};

type RepairEvidence = {
  ticketId: string;
  touchedFiles: Set<string>;
  toolCalls: string[];
  verificationSeen: Set<string>;
  errors: string[];
};

function attachRepairObserver(agent: Agent, ticket: RepairTicket): RepairEvidence {
  const evidence: RepairEvidence = {
    ticketId: ticket.id,
    touchedFiles: new Set(),
    toolCalls: [],
    verificationSeen: new Set(),
    errors: [],
  };

  agent.subscribe((event) => {
    if (event.type === "tool_execution_start") {
      evidence.toolCalls.push(event.toolName);

      const raw = JSON.stringify(event.args ?? {});
      for (const command of ticket.verification) {
        if (raw.includes(command)) {
          evidence.verificationSeen.add(command);
        }
      }
    }

    if (event.type === "tool_execution_end" && event.isError) {
      evidence.errors.push(event.toolName);
    }
  });

  return evidence;
}

function verifyRepairEvidence(ticket: RepairTicket, evidence: RepairEvidence): string[] {
  const failures: string[] = [];

  for (const command of ticket.verification) {
    if (!evidence.verificationSeen.has(command)) {
      failures.push("missing verification: " + command);
    }
  }

  if (evidence.errors.length > 0) {
    failures.push("tool errors: " + evidence.errors.join(", "));
  }

  return failures;
}
~~~

这和前几课的 EventStream 思路一致：核心 agent loop 只负责执行，外层系统负责治理、审计和合并判断。

## 4. OpenClaw：spawn worker 前要先写清楚 ownership

OpenClaw 的子 Agent 很适合处理这种修复，但 prompt 必须像工单，不像闲聊。

一个好的派单模板：

~~~text
你负责修复 RepairTicket rt-2026-05-18-346。

你不是独自在代码库里工作，其他 worker 可能同时处理别的 ticket。
不要 revert 别人的改动；发现冲突时调整自己的实现并在最终汇报。

写入范围：
- packages/agent/src/policy/**
- packages/agent/test/policy-replay/**

禁止修改：
- packages/web-ui/**
- docs/**
- regression-packs/cases/egress-raw-secret.json

目标：
- 恢复 replay case egress-raw-secret 的 expectedDecision=require_approval
- 保留 reasonCode=evidence.raw_secret
- 不要通过降低 fixture 期望来让测试通过

验证：
- npm test -- policy-replay
- npm test -- egress-raw-secret

最终汇报：
- changed files
- commands run
- replay result
- remaining risk
~~~

这段话里最重要的不是“请修复”，而是：

- ownership：谁负责哪个范围。
- non-revert：不能回滚别人。
- anti-cheat：不能改低 fixture 期望。
- verification：必须跑什么。
- closeout format：回来要带什么证据。

## 5. 合并闸门：不要相信“修好了”，只相信证据

Repair Worker 完成后，主控 Agent 应该检查四件事：

1. Diff 是否只落在 writeScope。
2. forbiddenScope 是否零改动。
3. verification 命令是否真实执行并通过。
4. replay fingerprint 是否回到 expected。

可以把 closeout gate 写成很直接的规则：

~~~ts
type CloseoutDecision =
  | { action: "merge_ready"; evidenceRefs: string[] }
  | { action: "return_to_worker"; reasons: string[] }
  | { action: "manual_review"; reasons: string[] };

function decideRepairCloseout(input: {
  scopeViolations: string[];
  verificationFailures: string[];
  replayRestored: boolean;
  risk: "low" | "medium" | "high";
}): CloseoutDecision {
  const reasons = [
    ...input.scopeViolations,
    ...input.verificationFailures,
  ];

  if (!input.replayRestored) {
    reasons.push("replay fingerprint not restored");
  }

  if (reasons.length === 0) {
    return { action: "merge_ready", evidenceRefs: ["diff", "verification", "replay"] };
  }

  if (input.risk === "high" || input.scopeViolations.length > 0) {
    return { action: "manual_review", reasons };
  }

  return { action: "return_to_worker", reasons };
}
~~~

这一步让自动修复不会变成自动乱改。

## 6. 实战建议

把 RepairTicket 接入自动修复时，建议先从三类 ticket 开始：

- coverage_gap：让 worker 只补测试，不改产品代码。
- fixture_refresh：让 worker 只迁移 fixture，不改策略逻辑。
- block_fix_required：只允许改明确 owner 的小模块，并强制 replay gate。

不要一开始就让 Agent 对整个仓库自由修复。自由度太高时，LLM 会倾向于找最短路径让测试变绿，而不是修复真实行为。

今天这课的 takeaway：

**回归系统发现问题只是第一步；能把问题变成隔离的、有边界的、可验证的修复 lane，才是 Agent 自动维护代码库的关键。**
