# 455. Agent 可取消工具的收敛协议（Cancellable Tool Convergence Protocol）

> 关键词：cancellable tool、AbortSignal、cleanup receipt、terminal state、cancel convergence

上一课讲了披露例外租约要持续续期和监控。今天换回 Agent 运行时里一个更基础、也更容易被低估的问题：**用户按了停止、任务超时、上游 AbortSignal 触发以后，工具到底算成功、失败、取消，还是未知？**

很多 Agent 框架一开始会这样处理取消：

~~~text
user presses Esc
abort controller fires
tool throws AbortError
agent says: cancelled
~~~

这在纯计算任务里勉强够用，但在真实工具里很危险。因为工具可能已经：

- 写了一半文件；
- 启动了后台进程；
- 发出了外部 API 请求；
- 拿到了锁但还没释放；
- 生成了临时文件但没登记；
- 子任务还在另一个线程继续跑。

所以“取消”不是一个异常，而是一条收敛协议：

~~~text
CancelRequest
  -> CooperativeStop
  -> CleanupReceipt
  -> RealityProbe
  -> TerminalState(cancelled | completed | failed | unknown_needs_reconcile)
~~~

一句话：**可取消工具不能只会停；它必须证明自己停到了哪个终态。**

---

## 1. 为什么 AbortError 不够

pi-mono 里已经有很好的底层形态：TUI 的 `CancellableLoader` 用 `AbortController` 把 Escape 转成 `AbortSignal`；工具执行函数也可以接收 `signal?: AbortSignal`，比如 edit tool 在读写文件时把 signal 传给 executor。

这解决了“取消信号如何传播”，但还没解决“取消后状态如何收敛”：

- 如果读文件阶段取消：通常没有副作用，可以直接 cancelled；
- 如果写文件阶段取消：文件可能已经被 shell 截断或部分写入，需要 reality probe；
- 如果后台任务取消：线程/子进程可能继续运行，需要 task status 和 cleanup receipt；
- 如果外部 API 已发出：本地取消不等于远端没执行，需要 reconciliation。

所以生产级工具要区分两层：

- **transport cancellation**：AbortSignal、timeout、用户中断；
- **semantic convergence**：清理了什么、验证了什么、最后世界处于什么状态。

没有第二层，Agent 很容易把“我不等了”误报成“事情没发生”。

---

## 2. 收敛协议对象

建议给每个可取消工具定义这五类对象：

- **CancelRequest**：谁触发、原因、时间、是否用户可见；
- **ToolPhase**：工具当前阶段，至少区分 before_effect / effect_in_progress / after_effect；
- **CleanupReceipt**：取消后释放了哪些资源，哪些释放失败；
- **RealityProbe**：取消后重新观察外部现实，例如文件 hash、进程状态、远端状态；
- **TerminalState**：最终状态，只允许 completed、cancelled_clean、failed、unknown_needs_reconcile。

核心原则：

- 取消发生在副作用前，可以 clean cancel；
- 取消发生在副作用中，必须 probe；
- probe 失败不能假装成功，进入 unknown_needs_reconcile；
- cleanup receipt 要能重复写，避免取消处理自身崩溃后丢证据。

状态机可以这样写：

~~~text
running
  -> cancel_requested
  -> stopping
  -> cleanup_attempted
  -> reality_probed
  -> cancelled_clean
  -> completed_despite_cancel
  -> failed_after_cancel
  -> unknown_needs_reconcile
~~~

注意：completed_despite_cancel 是合法终态。用户取消时，工具可能已经完成了副作用。Agent 应该诚实报告“取消来晚了，动作已经完成”，而不是为了迎合用户说“已取消”。

---

## 3. learn-claude-code：取消收敛判定器

教学版先不接真实线程，写一个纯函数：输入取消发生阶段、cleanup 结果、reality probe，输出终态和下一步动作。

~~~python
# learn_claude_code/runtime/cancel_convergence.py
from dataclasses import dataclass
from typing import Literal

ToolPhase = Literal[
    "before_effect",
    "effect_in_progress",
    "after_effect",
    "background_running",
]

ProbeOutcome = Literal[
    "no_effect_observed",
    "effect_completed",
    "partial_effect_observed",
    "probe_failed",
]

TerminalState = Literal[
    "cancelled_clean",
    "completed_despite_cancel",
    "failed_after_cancel",
    "unknown_needs_reconcile",
]


@dataclass(frozen=True)
class CancelRequest:
    run_id: str
    reason: str
    requested_by: str
    requested_at_epoch: int


@dataclass(frozen=True)
class CleanupReceipt:
    lock_released: bool
    temp_files_removed: bool
    child_process_stopped: bool
    cleanup_error: str | None = None


@dataclass(frozen=True)
class RealityProbe:
    outcome: ProbeOutcome
    observed_ref: str | None = None
    evidence_hash: str | None = None


@dataclass(frozen=True)
class CancelDecision:
    terminal_state: TerminalState
    reason: str
    required_actions: tuple[str, ...] = ()


def converge_cancelled_tool(
    phase: ToolPhase,
    cleanup: CleanupReceipt,
    probe: RealityProbe | None,
) -> CancelDecision:
    cleanup_ok = (
        cleanup.lock_released
        and cleanup.temp_files_removed
        and cleanup.child_process_stopped
        and cleanup.cleanup_error is None
    )

    if phase == "before_effect" and cleanup_ok:
        return CancelDecision("cancelled_clean", "cancelled_before_side_effect")

    if not cleanup_ok:
        return CancelDecision(
            "unknown_needs_reconcile",
            "cleanup_incomplete",
            ("retry_cleanup", "write_reconcile_ticket"),
        )

    if probe is None:
        return CancelDecision(
            "unknown_needs_reconcile",
            "side_effect_phase_requires_reality_probe",
            ("run_reality_probe",),
        )

    if probe.outcome == "no_effect_observed":
        return CancelDecision("cancelled_clean", "cleanup_complete_and_no_effect_observed")

    if probe.outcome == "effect_completed":
        return CancelDecision(
            "completed_despite_cancel",
            "cancel_arrived_after_effect_completed",
            ("notify_user_effect_already_happened",),
        )

    if probe.outcome == "partial_effect_observed":
        return CancelDecision(
            "failed_after_cancel",
            "partial_effect_requires_forward_fix",
            ("open_forward_fix", "block_downstream_actions"),
        )

    return CancelDecision(
        "unknown_needs_reconcile",
        "probe_failed",
        ("retry_probe", "escalate_if_probe_budget_exhausted"),
    )
~~~

这个函数的价值不是复杂，而是把取消后的模糊状态压成四个可处理终态。Agent 后续就不会把 unknown 当 cancelled。

---

## 4. pi-mono：把 AbortSignal 包成收敛工具

pi-mono 的现有形态已经给了两个关键积木：

- `CancellableLoader`：用户按 Escape 后触发 `AbortController.abort()`；
- tool `execute(..., signal?: AbortSignal)`：工具执行链能接收取消信号。

生产版可以在工具外包一层 wrapper，统一写 phase、cleanup receipt 和 terminal state：

~~~ts
type Phase = "before_effect" | "effect_in_progress" | "after_effect";
type TerminalState =
  | "completed"
  | "cancelled_clean"
  | "completed_despite_cancel"
  | "failed_after_cancel"
  | "unknown_needs_reconcile";

type CleanupReceipt = {
  lockReleased: boolean;
  tempRemoved: boolean;
  childStopped: boolean;
  error?: string;
};

type RealityProbe = {
  outcome: "no_effect_observed" | "effect_completed" | "partial_effect_observed" | "probe_failed";
  evidenceHash?: string;
};

function convergeCancellation(
  phase: Phase,
  cleanup: CleanupReceipt,
  probe: RealityProbe,
): TerminalState {
  const cleanupOk =
    cleanup.lockReleased &&
    cleanup.tempRemoved &&
    cleanup.childStopped &&
    cleanup.error === undefined;

  if (phase === "before_effect" && cleanupOk) return "cancelled_clean";
  if (!cleanupOk) return "unknown_needs_reconcile";
  if (probe.outcome === "no_effect_observed") return "cancelled_clean";
  if (probe.outcome === "effect_completed") return "completed_despite_cancel";
  if (probe.outcome === "partial_effect_observed") return "failed_after_cancel";
  return "unknown_needs_reconcile";
}

async function runCancellableTool<T>(args: {
  runId: string;
  signal: AbortSignal;
  execute: (markEffectStarted: () => void) => Promise<T>;
  cleanup: () => Promise<CleanupReceipt>;
  probe: () => Promise<RealityProbe>;
  writeReceipt: (receipt: unknown) => Promise<void>;
}): Promise<{ state: TerminalState; result?: T }> {
  let phase: Phase = "before_effect";

  const markEffectStarted = () => {
    phase = "effect_in_progress";
  };

  try {
    const result = await args.execute(markEffectStarted);
    phase = "after_effect";
    return { state: "completed", result };
  } catch (error) {
    if (!args.signal.aborted) throw error;

    const cleanup = await args.cleanup().catch((e) => ({
      lockReleased: false,
      tempRemoved: false,
      childStopped: false,
      error: String(e),
    }));

    const probe =
      phase === "before_effect"
        ? { outcome: "no_effect_observed" as const }
        : await args.probe().catch(() => ({ outcome: "probe_failed" as const }));

    const state = convergeCancellation(phase, cleanup, probe);
    await args.writeReceipt({
      runId: args.runId,
      cancelledAt: Date.now(),
      phase,
      cleanup,
      probe,
      terminalState: state,
    });

    return { state };
  }
}
~~~

关键点：工具内部只负责在真正副作用开始前调用 `markEffectStarted()`。外层 wrapper 负责统一收敛、写收据、决定要不要阻断下游。

---

## 5. OpenClaw：长任务和 Cron 的取消对账

OpenClaw 场景里，取消不只来自用户按 Esc，还可能来自：

- cron 超时；
- session 被新指令 interrupt；
- sub-agent 被 kill；
- browser/exec 长任务失联；
- Telegram 消息发出前任务被改道。

实战做法：

1. 每个长任务启动时写 `runId` 和 `phase=before_effect`；
2. 任何外部副作用前把 phase 切到 `effect_in_progress`；
3. 收到取消后先 stop/drain，再写 CleanupReceipt；
4. 对已经开始的副作用做 RealityProbe，例如 git status、远端 message id、文件 hash、进程状态；
5. 只有 terminalState 不是 unknown 时，才允许后续 Agent 基于这个结果继续推理；
6. unknown_needs_reconcile 要进入任务队列或 daily memory，不能只在日志里飘过。

本次课程 cron 自己就是例子：如果发送 Telegram 后 git commit 失败，不能说“课程失败”；如果 git commit 成功但 Telegram 发送未知，也不能重复发。每个外部动作都要有可对账收据。

---

## 6. 实战 Checklist

- [ ] 工具 schema 或 registry 标注是否有副作用；
- [ ] 工具执行链支持 `AbortSignal` 或等价取消信号；
- [ ] 副作用前后有明确 phase；
- [ ] 取消后一定写 CleanupReceipt；
- [ ] 副作用已开始时一定做 RealityProbe；
- [ ] terminal state 禁止只有 cancelled/success 两种；
- [ ] unknown_needs_reconcile 会阻断下游动作并进入对账队列；
- [ ] 用户可见回复区分“已取消”和“取消来晚，动作已完成”。

成熟 Agent 的取消不是把异常吞掉，而是把运行时从不确定状态带回可解释、可恢复、可审计的终态。
