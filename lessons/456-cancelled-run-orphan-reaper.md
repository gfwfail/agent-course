# 456. Agent 取消后的孤儿任务收割器（Cancelled Run Orphan Reaper）

> 关键词：orphan task、cancel reaper、background cleanup、lease、reconciliation

上一课讲了可取消工具不能只抛 \`AbortError\`，而要收敛到明确终态。今天继续往下走一步：**如果 Agent Run 已经 cancelled，但它启动过的子进程、后台任务、sub-agent、外部 job 还在跑，怎么办？**

这类残留任务我建议叫 **orphan task**：

~~~text
parent run cancelled
  -> child process still alive
  -> background job still running
  -> sub-agent still consuming tokens
  -> external API job still mutating state
~~~

如果没有孤儿任务收割器，取消只是 UI 上“看起来停了”。真实世界里，后台动作可能继续写文件、继续发消息、继续烧钱，最后还会把结果写回一个已经取消的会话。

一句话：**取消协议负责判断当前工具终态；孤儿任务收割器负责把取消后仍在跑的后代任务找出来、处理掉、对账清楚。**

---

## 1. 为什么取消后会产生孤儿任务

Agent 系统里常见三种异步边界：

- **子进程边界**：shell、ffmpeg、browser driver、test runner；
- **任务队列边界**：background task、cron job、worker queue；
- **Agent 边界**：sub-agent、remote session、tool server 内部 job。

\`AbortSignal\` 通常只能通知当前调用栈。只要跨过异步边界，就可能出现信号没有继续传播，或者传播了但对方已经进入不可中断阶段。

典型事故：

- 用户取消“生成图片”，UI 停了，但远端生成任务完成后仍扣费；
- 取消“批量发消息”，父 run 结束了，worker 还在消费队列；
- 取消“代码修改”，sub-agent 仍在 fork workspace 里提交 patch；
- 取消“浏览器自动化”，browser context 没关，后续任务复用到脏页面。

所以生产级 Agent 需要一条后台安全网：

~~~text
CancelledRun
  -> OrphanScan
  -> ReaperLease
  -> StopAttempt
  -> RealityProbe
  -> ReapReceipt | ReconcileTicket
~~~

---

## 2. 孤儿任务收割器的数据模型

最小可用模型有五个对象：

- **RunDescendant**：父 run 启动过的后代，记录 kind、id、ownerRunId、startedAt、cancelPolicy；
- **OrphanScan**：扫描某个 cancelled run 的所有后代，判断 running、finished、unknown；
- **ReaperLease**：抢占式收割租约，避免多个 reaper 同时 kill 同一个任务；
- **StopAttempt**：对不同 kind 使用不同停止方式，例如 SIGTERM、queue cancel、sub-agent close；
- **ReapReceipt**：证明任务已经 stopped、completed_before_reap、not_found 或 needs_reconcile。

关键点：

- 不能只按进程名 kill，要按 \`ownerRunId\` 或 \`traceId\` 关联；
- stop 之后必须 probe，不能相信 kill API 的返回值；
- 收割器要幂等，重复跑不能误杀新任务；
- unknown 状态必须进入 reconcile ticket，不能静默吞掉。

---

## 3. learn-claude-code：教学版纯函数判定器

教学版可以先把复杂执行抽掉，只写一个分类器：输入后代状态和停止结果，输出收割决策。

~~~python
# learn_claude_code/runtime/orphan_reaper.py
from dataclasses import dataclass
from typing import Literal

DescendantKind = Literal["process", "background_task", "subagent", "external_job"]
ObservedState = Literal["running", "completed", "not_found", "unknown"]
StopOutcome = Literal["stopped", "already_done", "not_found", "stop_failed", "not_supported"]
ReapState = Literal["reaped", "completed_before_reap", "not_found", "needs_reconcile"]


@dataclass(frozen=True)
class RunDescendant:
    owner_run_id: str
    descendant_id: str
    kind: DescendantKind
    cancel_policy: Literal["kill", "request_stop", "detach_and_reconcile"]
    started_at_epoch: int


@dataclass(frozen=True)
class ReapInput:
    descendant: RunDescendant
    observed_state: ObservedState
    stop_outcome: StopOutcome | None
    probe_ref: str | None = None


@dataclass(frozen=True)
class ReapDecision:
    state: ReapState
    reason: str
    required_actions: tuple[str, ...] = ()


def decide_reap(input: ReapInput) -> ReapDecision:
    if input.observed_state == "completed":
        return ReapDecision(
            "completed_before_reap",
            "descendant_finished_after_parent_cancelled",
            ("attach_output_to_cancelled_run_audit",),
        )

    if input.observed_state == "not_found":
        return ReapDecision("not_found", "descendant_absent_during_scan")

    if input.observed_state == "unknown":
        return ReapDecision(
            "needs_reconcile",
            "cannot_observe_descendant_state",
            ("retry_probe", "open_reconcile_ticket"),
        )

    if input.descendant.cancel_policy == "detach_and_reconcile":
        return ReapDecision(
            "needs_reconcile",
            "descendant_policy_requires_detached_reconciliation",
            ("record_detached_job", "schedule_reconcile_probe"),
        )

    if input.stop_outcome in ("stopped", "already_done"):
        return ReapDecision(
            "reaped",
            "stop_confirmed_by_probe",
            ("write_reap_receipt",),
        )

    if input.stop_outcome == "not_found":
        return ReapDecision("not_found", "descendant_disappeared_before_stop")

    return ReapDecision(
        "needs_reconcile",
        "stop_failed_or_unsupported",
        ("escalate_reaper_failure", "block_child_output_writeback"),
    )
~~~

这个函数的重点是把“取消后还在跑”变成一个可测试的状态机。教学系统可以用它写单元测试，覆盖 \`running + stop_failed\`、\`completed after cancel\`、\`external_job detach\` 这些最容易漏的边界。

---

## 4. pi-mono：生产版 Reaper Worker

pi-mono 这种 TypeScript 生产实现里，建议把所有异步后代登记进统一 ledger。工具启动进程、创建 background task、spawn sub-agent 时，都写一条 \`RunDescendant\`。

~~~ts
type DescendantKind = "process" | "background_task" | "subagent" | "external_job";

type RunDescendant = {
  ownerRunId: string;
  descendantId: string;
  kind: DescendantKind;
  cancelPolicy: "kill" | "request_stop" | "detach_and_reconcile";
  startedAt: number;
  fenceToken: string;
};

type ReapReceipt = {
  ownerRunId: string;
  descendantId: string;
  state: "reaped" | "completed_before_reap" | "not_found" | "needs_reconcile";
  evidenceRef?: string;
  reapedAt: number;
};

class CancelledRunReaper {
  constructor(
    private readonly ledger: DescendantLedger,
    private readonly stopper: DescendantStopper,
    private readonly receipts: ReapReceiptStore,
  ) {}

  async reapCancelledRun(ownerRunId: string): Promise<ReapReceipt[]> {
    const descendants = await this.ledger.listByOwnerRun(ownerRunId);
    const receipts: ReapReceipt[] = [];

    for (const descendant of descendants) {
      const lease = await this.ledger.tryAcquireReaperLease(
        descendant.descendantId,
        descendant.fenceToken,
      );
      if (!lease.acquired) continue;

      const observed = await this.stopper.probe(descendant);
      const stopOutcome =
        observed.state === "running"
          ? await this.stopper.stop(descendant)
          : undefined;
      const afterStop = await this.stopper.probe(descendant);

      const receipt = classifyReap(descendant, observed, stopOutcome, afterStop);
      await this.receipts.writeOnce(receipt);
      receipts.push(receipt);
    }

    return receipts;
  }
}
~~~

这里有三个工程细节很重要：

- \`fenceToken\` 防止旧 reaper 租约误操作新任务；
- \`writeOnce\` 保证同一个 descendant 只产生一个最终收据；
- \`probe -> stop -> probe\` 三段式，避免把 API 返回当现实。

如果 stop 后仍然 running，状态不要写成 failed 就结束，而是写 \`needs_reconcile\`，交给后续补偿或人工升级。

---

## 5. OpenClaw：Cron / sub-agent 场景怎么落地

OpenClaw 的 Always-on 形态特别需要这个模式，因为它天然有 cron、sessions、subagents、browser、message 这些跨边界动作。

以这门课程 cron 为例，一次正常发布会跨过多个外部边界：

~~~text
cron run
  -> write lesson file
  -> update README / TOOLS
  -> send Telegram message
  -> git commit / push
~~~

如果 run 在发送 Telegram 后、git commit 前被取消，reaper 不能“撤回一切”。它要做的是：

- 扫描本 run 是否还有未结束的 child process，例如 git push、测试进程；
- 检查 Telegram message 是否已产生 \`messageId\`，写入副作用 ledger；
- 如果 lesson 文件已写但未提交，标记 \`needs_reconcile\`，下次 cron 先对账；
- 如果 sub-agent 仍在运行，关闭或改成 detached reconcile；
- 最后写一条 \`ReapReceipt\`，说明哪些已停、哪些已完成、哪些需要补账。

最小实现可以是一个 heartbeat/cron 后台任务：

~~~text
every 10 minutes:
  cancelled_runs = list_runs(status="cancelled", reaperReceiptMissing=true)
  for run in cancelled_runs:
    descendants = list_descendants(ownerRunId=run.id)
    reap each descendant with lease
    if any needs_reconcile:
      create reconciliation ticket
~~~

这比“用户取消了，所以当作没发生”可靠得多。

---

## 6. 实战检查清单

做 Agent 取消能力时，可以直接问这 6 个问题：

- 每个子进程、background task、sub-agent 是否登记了 \`ownerRunId\`？
- 取消父 run 时，是否能列出全部 descendants？
- 对每类 descendant，stop 方式是什么，是否可重复执行？
- stop 后是否有 reality probe？
- reaper 是否有 lease/fence，避免误杀新任务？
- unknown / stop_failed 是否进入 reconcile，而不是吞掉？

只要有一个答案是“不知道”，取消能力就还停留在 UI 层，不是生产级取消。

---

## 7. 关键 takeaway

取消不是一个按钮，而是一条后代任务治理链。

- 第 455 课解决“当前工具如何收敛到终态”；
- 第 456 课解决“取消后仍在跑的后代如何被收割和对账”。

成熟 Agent 不能只会停下当前调用栈。它还要能找到自己放出去的所有后代任务，带租约地停止它们，带证据地确认现实，最后把无法确认的部分转成可追踪的 reconcile ticket。

下一次你给 Agent 加 \`AbortController\` 时，顺手加一张 \`RunDescendant\` 表。否则取消只是前端动画，后台还在继续干活。
