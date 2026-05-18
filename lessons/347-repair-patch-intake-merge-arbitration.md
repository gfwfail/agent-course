# 347. Agent 修复补丁接收与合并仲裁（Repair Patch Intake & Merge Arbitration）

上一课讲了 RepairTicket 和 Worktree Lane：失败回归不要一句“修一下”丢给子 Agent，而是用写入边界、禁止范围和验证命令约束修复。

今天继续往后走：**子 Agent 修完以后，主控 Agent 不能直接信最终回复，也不能把 patch 一把合进去；它要接收补丁、验证边界、仲裁冲突，再决定 merge、retry 还是 escalate。**

成熟链路应该长这样：

~~~text
RepairTicket
  -> Worktree Lane
  -> Worker Patch
  -> Patch Intake Gate
  -> Conflict Arbitration
  -> Verification Replay
  -> Merge / Retry / Escalate
~~~

核心原则：**worker 负责产出候选修复，controller 负责证明候选修复可以进入主线。**

## 1. 为什么修完不能直接 merge

子 Agent 的最终回复通常会说：

~~~text
已修复，跑了测试，通过。
~~~

这句话不能当证据。主控至少要确认五件事：

- touchedFiles 是否都在 writeScope 里。
- forbiddenScope 有没有被修改。
- patch 有没有只改 fixture、注释或 snapshot 来掩盖行为回归。
- verification 命令是否真的执行，结果是否来自当前 patch。
- 多个 worker 的 patch 是否改了同一块逻辑，合并后会不会互相抵消。

所以 Repair Patch Intake 不是简单 git apply，而是一个小型准入流程。

## 2. learn-claude-code：最小 Patch Intake Gate

教学版可以先用文件路径做第一层防线。Worker 完成后，controller 读取 diff 文件列表，对照 ticket 的 writeScope / forbiddenScope。

~~~python
from dataclasses import dataclass
from fnmatch import fnmatch

@dataclass
class RepairTicket:
    id: str
    case_id: str
    write_scope: list[str]
    forbidden_scope: list[str]
    verification: list[str]

@dataclass
class WorkerPatch:
    ticket_id: str
    touched_files: list[str]
    commands_run: list[str]
    summary: str

@dataclass
class IntakeResult:
    decision: str
    reasons: list[str]

def matches_any(path: str, patterns: list[str]) -> bool:
    return any(fnmatch(path, pattern) for pattern in patterns)

def intake_patch(ticket: RepairTicket, patch: WorkerPatch) -> IntakeResult:
    reasons: list[str] = []

    for path in patch.touched_files:
        if matches_any(path, ticket.forbidden_scope):
            reasons.append(f"forbidden file changed: {path}")

        if not matches_any(path, ticket.write_scope):
            reasons.append(f"outside write scope: {path}")

    for command in ticket.verification:
        if command not in patch.commands_run:
            reasons.append(f"missing verification: {command}")

    if reasons:
        return IntakeResult(decision="reject_or_retry", reasons=reasons)

    return IntakeResult(decision="accept_for_replay", reasons=[])
~~~

这只是第一层。它不能证明业务正确，但能快速挡住越权修改和没跑验证的 patch。

如果结合前面课程的 worktree 事件，可以把 intake 写进生命周期：

~~~text
worktree_run_done
  -> diff_collect
  -> patch_intake
  -> replay_verify
  -> merge_candidate
~~~

这样每个修复 lane 都有可审计状态，而不是靠聊天记录猜修到了哪一步。

## 3. pi-mono：用事件证据约束 Patch Intake

pi-mono 的 Agent.subscribe 可以观察工具调用。上一课我们用它记录 verificationSeen；今天把它升级成 Patch Evidence。

~~~ts
type RepairTicket = {
  id: string;
  caseId: string;
  writeScope: string[];
  forbiddenScope: string[];
  verification: string[];
};

type PatchEvidence = {
  ticketId: string;
  touchedFiles: Set<string>;
  verificationSeen: Set<string>;
  toolErrors: string[];
  finalSummary?: string;
};

function pathMatches(path: string, patterns: string[]): boolean {
  return patterns.some((pattern) => {
    const prefix = pattern.replace("/**", "");
    return path === prefix || path.startsWith(prefix + "/");
  });
}

function intakePatch(ticket: RepairTicket, evidence: PatchEvidence): string[] {
  const failures: string[] = [];

  for (const file of evidence.touchedFiles) {
    if (pathMatches(file, ticket.forbiddenScope)) {
      failures.push("forbidden file changed: " + file);
    }

    if (!pathMatches(file, ticket.writeScope)) {
      failures.push("outside write scope: " + file);
    }
  }

  for (const command of ticket.verification) {
    if (!evidence.verificationSeen.has(command)) {
      failures.push("missing verification: " + command);
    }
  }

  if (evidence.toolErrors.length > 0) {
    failures.push("tool errors: " + evidence.toolErrors.join(", "));
  }

  return failures;
}
~~~

这里有个工程重点：**finalSummary 只能当说明，不能当证据。** 真正的证据来自 diff、工具事件、测试输出和 replay 结果。

## 4. 合并仲裁：多个 patch 同时回来怎么办

如果回归系统一次派出多个 RepairTicket，controller 可能同时收到几个 patch。最危险的做法是按完成时间一个个 merge，因为先完成的不一定优先级最高，也不一定和其他 patch 兼容。

更好的做法是引入 Merge Candidate：

~~~ts
type MergeCandidate = {
  ticketId: string;
  priority: "p1" | "p2" | "p3";
  riskSurface: string;
  touchedFiles: string[];
  replayCommands: string[];
  intakeFailures: string[];
};

type MergeDecision =
  | { action: "merge"; candidate: MergeCandidate }
  | { action: "retry"; candidate: MergeCandidate; reason: string }
  | { action: "hold_for_rebase"; candidates: MergeCandidate[]; reason: string }
  | { action: "escalate"; candidates: MergeCandidate[]; reason: string };

function overlaps(a: MergeCandidate, b: MergeCandidate): boolean {
  return a.touchedFiles.some((file) => b.touchedFiles.includes(file));
}

function arbitrate(candidates: MergeCandidate[]): MergeDecision[] {
  const decisions: MergeDecision[] = [];
  const clean = candidates.filter((c) => c.intakeFailures.length === 0);

  for (const candidate of candidates) {
    if (candidate.intakeFailures.length > 0) {
      decisions.push({
        action: "retry",
        candidate,
        reason: candidate.intakeFailures.join("; "),
      });
    }
  }

  for (const candidate of clean) {
    const conflicts = clean.filter((other) => {
      return other.ticketId !== candidate.ticketId && overlaps(candidate, other);
    });

    if (conflicts.length > 0) {
      decisions.push({
        action: "hold_for_rebase",
        candidates: [candidate, ...conflicts],
        reason: "overlapping touched files",
      });
      continue;
    }

    decisions.push({ action: "merge", candidate });
  }

  return decisions;
}
~~~

实际生产里，冲突判断不能只看文件名，还要看：

- 同一个 policy / middleware / prompt 区域是否被改。
- 两个 patch 是否改变同一个 replay case 的 expected decision。
- 一个 patch 是否刷新 fixture，另一个 patch 是否修真实逻辑。
- 合并后是否还通过完整 selected replay set。

这就是 arbitration 的价值：它不是解决 git conflict，而是解决“哪些候选修复可以一起进入主线”。

## 5. OpenClaw：主控 Agent 的补丁接收流程

在 OpenClaw 里，worker 完成后主控 Agent 不应该只读最终汇报。它应该按固定流程收口：

~~~text
1. 读取 worker 改动文件列表和 diff
2. 对照 RepairTicket 检查 writeScope / forbiddenScope
3. 跑 ticket.verification
4. 跑本轮 selected regression replay
5. 如果有多个 worker patch，做 overlap arbitration
6. 生成 MergeDecision evidence
7. 再 commit / PR / merge
~~~

一个主控 prompt 可以这样写：

~~~text
接收 RepairTicket rt-347 的 worker patch。

不要直接相信 worker 最终回复。请执行：
- 收集 touched files
- 检查是否越过 writeScope
- 检查 forbiddenScope 是否被修改
- 重新运行 verification commands
- 重新运行 selected replay cases
- 如果和其他候选 patch 修改同一文件，标记 hold_for_rebase

只有当 intake=pass 且 replay=pass 时，才允许进入 merge_candidate。
最终输出 MergeDecision：merge / retry / hold_for_rebase / escalate。
~~~

这里的重点是：**自动修复可以快，但合并判断必须慢半拍。** 快的是 worker 并行探索，稳的是 controller 统一验收。

## 6. 实战建议

可以先做一个很小的版本：

- 每个 RepairTicket 必须声明 writeScope、forbiddenScope、verification。
- Worker 完成后自动收集 changed files。
- 任何 forbiddenScope 修改直接 reject。
- verification 没跑直接 retry。
- replay 没恢复直接 retry 或 escalate。
- 多个 patch 改同一文件时，不自动 merge，先 rebase 或人工看一眼。

等这套跑顺了，再加更细的语义检查，比如 AST diff、policy decision diff、fixture anti-cheat、CODEOWNERS owner routing。

一句话总结：**Repair Patch Intake 把“子 Agent 说修好了”变成“主控系统证明这个 patch 可以合”。成熟 Agent 的自动修复不是自动相信，而是自动验收。**
