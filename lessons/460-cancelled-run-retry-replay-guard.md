# 460. Agent 取消后的重试重放护栏（Cancelled Run Retry Replay Guard）

上一组课把取消恢复链路串到了 closeout：

~~~text
CancelRequest
  -> orphan reaper
  -> side-effect reconciliation
  -> evidence freeze
  -> snapshot closeout / retention cleanup
~~~

今天补一个经常被忽略的后续问题：**取消后的 run 要不要允许重试、续跑或自动重放？**

很多 Agent 系统会把 cancelled run 当成普通 failed run，用户点一下 retry，或者 cron 下一轮自动继续。但如果上一轮已经发过 Telegram、push 过 git、创建过支付、触发过部署，直接 replay 就可能把外部副作用再执行一遍。

所以需要一个专门的 **Retry Replay Guard**：

~~~text
RetryRequest
  -> PreviousRunCloseoutReceipt
  -> EffectReplayPolicy
  -> ReplayPlan
  -> ReplayPermit | ReplayBlockedReceipt
~~~

一句话：**取消后的重试不是从头再跑，而是先证明哪些步骤可重放、哪些必须复用旧结果、哪些必须等人工确认。**

---

## 1. 为什么 retry 不能等于 rerun

普通代码任务里，retry 通常只是再执行一次函数。但 Agent run 会穿过很多外部边界：

- 消息已发送：再跑一次可能重复通知；
- git push 已成功：再跑一次可能空提交、冲突提交或覆盖后续修改；
- 支付已创建：再跑一次可能重复扣款；
- 部署已触发：再跑一次可能把已回滚版本重新上线；
- sub-agent 已产出 patch：再跑一次可能基于旧 workspace 重写新结果。

取消恢复链路已经能回答“上一轮停在哪里”。Retry Replay Guard 要回答的是下一句：

~~~text
基于上一轮终态，本轮能不能继续？
如果能继续，是从哪个 checkpoint 继续？
哪些外部副作用必须复用 receipt，不能重新调用工具？
~~~

没有这层护栏，幂等键、对账收据、冻结快照都会变成事后审计材料，而不是运行前阻断条件。

---

## 2. 最小数据模型

建议把 retry 拆成四个对象：

- **RetryRequest**：谁请求重试、目标 run、重试原因、期望模式；
- **PreviousRunCloseoutReceipt**：上一轮取消后的最终收口证明；
- **ReplayPlan**：每个 step 的动作，分为 rerun、reuse_receipt、skip、manual_review；
- **ReplayPermit**：真正允许执行的新 run 许可，绑定 allowedSteps 和 forbiddenEffectIds。

一个 ReplayPlan 可以长这样：

~~~json
{
  "retryOfRunId": "run_456",
  "newRunId": "run_789",
  "steps": [
    {
      "stepId": "write_lesson",
      "action": "rerun",
      "reason": "local_file_not_committed"
    },
    {
      "stepId": "send_telegram",
      "action": "reuse_receipt",
      "effectId": "telegram:13095",
      "reason": "message_already_sent_and_matched"
    },
    {
      "stepId": "git_push",
      "action": "manual_review",
      "reason": "remote_head_changed_after_cancel"
    }
  ],
  "forbiddenEffectIds": ["telegram:13095"],
  "requiresFreshRealityProbe": true
}
~~~

关键点是：ReplayPlan 不是日志，而是执行前的约束。工具分发层必须读它，看到 forbiddenEffectId 时直接拒绝重复副作用。

---

## 3. learn-claude-code：教学版重放判定器

learn-claude-code 的教学实现适合把它写成纯函数：输入上一轮 step closeout，输出这次 retry 的计划。

~~~python
# learn_claude_code/runtime/retry_replay_guard.py
from dataclasses import dataclass
from typing import Literal

StepOutcome = Literal[
    "not_started",
    "local_only",
    "effect_matched",
    "effect_unknown",
    "effect_mismatch",
    "terminal_failed",
]
ReplayAction = Literal["rerun", "reuse_receipt", "skip", "manual_review", "block"]


@dataclass(frozen=True)
class PreviousStep:
    step_id: str
    outcome: StepOutcome
    effect_id: str | None = None
    idempotency_key: str | None = None
    remote_changed_after_cancel: bool = False


@dataclass(frozen=True)
class ReplayStep:
    step_id: str
    action: ReplayAction
    reason: str
    forbidden_effect_id: str | None = None


def decide_replay_step(step: PreviousStep) -> ReplayStep:
    if step.outcome == "not_started":
        return ReplayStep(step.step_id, "rerun", "step_never_started")

    if step.outcome == "local_only":
        return ReplayStep(step.step_id, "rerun", "local_change_can_be_recomputed")

    if step.outcome == "effect_matched":
        return ReplayStep(
            step.step_id,
            "reuse_receipt",
            "external_effect_already_verified",
            forbidden_effect_id=step.effect_id,
        )

    if step.outcome == "effect_unknown":
        return ReplayStep(step.step_id, "manual_review", "effect_state_unknown")

    if step.outcome == "effect_mismatch":
        return ReplayStep(step.step_id, "block", "effect_mismatch_requires_forward_fix")

    if step.remote_changed_after_cancel:
        return ReplayStep(step.step_id, "manual_review", "remote_changed_after_cancel")

    return ReplayStep(step.step_id, "rerun", "terminal_failure_without_external_effect")


def build_replay_plan(previous_steps: list[PreviousStep]) -> list[ReplayStep]:
    return [decide_replay_step(step) for step in previous_steps]
~~~

教学版要强调两个测试：

- 已经 matched 的外部副作用不能重新执行，只能 reuse receipt；
- unknown / mismatch 不能靠 retry 掩盖，必须先 reconcile 或 forward-fix。

---

## 4. pi-mono：生产版 ReplayGuard 中间件

pi-mono 这种 TypeScript 生产实现里，ReplayGuard 应该放在 tool dispatch 之前，而不是放在业务工具内部。这样 Telegram、GitHub、支付、部署工具都能共享同一套规则。

~~~ts
type ReplayAction = "rerun" | "reuse_receipt" | "skip" | "manual_review" | "block";

type ReplayPlanStep = {
  stepId: string;
  toolName: string;
  action: ReplayAction;
  idempotencyKey?: string;
  forbiddenEffectId?: string;
  receiptRef?: string;
  reason: string;
};

type ReplayPermit = {
  retryOfRunId: string;
  newRunId: string;
  planHash: string;
  steps: ReplayPlanStep[];
  expiresAt: number;
};

class ReplayGuardMiddleware {
  constructor(
    private readonly permits: ReplayPermitStore,
    private readonly receipts: EffectReceiptStore,
  ) {}

  async beforeToolCall(ctx: ToolContext, call: ToolCall): Promise<ToolDecision> {
    const permit = await this.permits.getForRun(ctx.runId);
    if (!permit) return { type: "allow" };

    const step = permit.steps.find((s) => s.stepId === call.stepId);
    if (!step) {
      return { type: "deny", reason: "step_not_in_replay_permit" };
    }

    if (step.action === "reuse_receipt") {
      const receipt = await this.receipts.get(step.receiptRef!);
      return {
        type: "return_cached_result",
        result: receipt.toolResult,
        reason: "replay_reuses_verified_external_effect",
      };
    }

    if (step.action === "manual_review" || step.action === "block") {
      return { type: "deny", reason: step.reason };
    }

    if (step.forbiddenEffectId && call.effectId === step.forbiddenEffectId) {
      return { type: "deny", reason: "external_effect_replay_forbidden" };
    }

    return { type: "allow", idempotencyKey: step.idempotencyKey };
  }
}
~~~

这里的重点是 **return_cached_result**。如果上一轮 Telegram 已经发出并对账 matched，新 run 不应该再调用 sendMessage，而是把上一轮 receipt 作为工具结果注入后续步骤，让 Agent 继续生成 README、commit 或总结。

---

## 5. OpenClaw 课程 Cron：怎么落地

这门课程 cron 本身就是很好的例子。一次发布有几个步骤：

~~~text
1. 写 lesson 文件
2. 更新 README
3. 更新 TOOLS
4. 发 Telegram
5. git add / commit / push
6. 写 memory
~~~

如果 run 在第 4 步之后取消，下一轮不能简单重跑第 4 步。正确做法：

- 先读取上一轮 cancel closeout receipt；
- 如果 Telegram messageId 已 matched，把 send Telegram 标记为 reuse_receipt；
- 如果 git commit 未产生，允许重新执行本地文件检查和 commit；
- 如果远端 main 已经变化，先 pull/rebase，再决定是否继续；
- 如果 TOOLS 已经追加该主题，禁止再追加同一条；
- 最后写 ReplayPermit，后续工具调用必须按 permit 执行。

一个实用规则：

~~~text
local deterministic step -> 可以 rerun
external matched effect -> reuse receipt
external unknown effect -> reconcile first
external mismatch effect -> forward-fix first
remote changed after cancel -> manual review or fresh merge gate
~~~

这能防止最烦人的事故：群里发了两遍课、Git 里出现两个相似提交、README 和 TOOLS 对不上。

---

## 6. 工程检查清单

做取消后 retry 时，我建议至少检查：

- retry 前必须有上一轮 closeout receipt；
- replay plan 必须覆盖所有外部副作用 step；
- matched effect 默认 reuse，不默认 rerun；
- unknown / mismatch 默认阻断，不默认继续；
- ReplayPermit 要有 TTL，避免旧许可长期有效；
- tool dispatch 层要能拒绝 forbiddenEffectId；
- cached tool result 要保留原 receiptRef，方便审计；
- 新 run 的最终 summary 要说明哪些步骤是重跑，哪些是复用。

成熟 Agent 的 retry，不是“再试一次”。它是一次带证据的续跑：知道过去发生了什么，知道哪些现实不能重复，也知道从哪里继续才不会把系统越修越乱。
