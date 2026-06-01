# 462. Agent 取消重试后的重放许可消费闸门（Replay Permit Consumption Gate）

上一课讲了 **Pre-Replay Reality Drift Probe**：retry 前不能只相信旧 closeout receipt，要重新探测外部现实，确认 Telegram 消息、git remote、部署 job、队列任务、workspace hash 仍然匹配。

探测完成以后，系统会生成一个 ReplayPlan 或 ReplayPermit。但这里还有一个生产事故高发点：**有了 permit 不等于可以随便用 permit。**

如果 ReplayPermit 没有消费语义，常见问题是：

- 同一个 permit 被两个 worker 同时消费，导致重复恢复；
- permit 过期后仍被后台任务继续使用；
- permit 绑定的是 run A，却被 run B 的工具调用拿来放行；
- permit 允许复用 Telegram receipt，但工具层又重新 sendMessage；
- permit 只允许执行 step 3，Agent 却跳到 step 5 直接 push。

所以今天补一个 **Replay Permit Consumption Gate**：

~~~text
DriftReport
  -> ReplayPlan
  -> ReplayPermit
  -> PermitConsumptionLease
  -> ToolDispatchGate
  -> ConsumptionReceipt | ReplayBlockedReceipt
~~~

一句话：**ReplayPermit 不是一张长期通行证，而是一次性、可过期、按 step 消费、能写回执的执行许可。**

---

## 1. 为什么 permit 要被消费

Retry Replay Guard 负责决定哪些 step 可以 rerun、哪些 step 必须 reuse_receipt、哪些 step 要 block。Pre-Replay Drift Probe 负责确认旧收据没有过期。

但真正执行时，系统还要回答几个运行时问题：

~~~text
1. 谁正在消费这个 permit？
2. 当前 tool call 是否属于 permit 允许的 step？
3. 这个 step 是否已经被消费过？
4. reuse_receipt 是否真的没有重新触发外部副作用？
5. permit 过期、漂移或被抢占后，工具层是否会立刻拒绝？
~~~

这些问题不能交给 LLM 临场自觉。permit 必须变成工具分发层的硬约束。

关键原则：

- **single-use**：外部副作用 step 的许可只能消费一次；
- **step-scoped**：permit 只放行声明过的 step；
- **run-bound**：permit 只能被指定 newRunId 使用；
- **lease-bound**：消费 permit 要持有当前 lease 和 fenceToken；
- **receipt-backed**：每次消费都写 ConsumptionReceipt，后续 retry 能复盘。

---

## 2. 最小数据模型

一个 ReplayPermit 至少应该包含：

~~~json
{
  "permitId": "replay_permit_123",
  "retryOfRunId": "run_cancelled_456",
  "newRunId": "run_retry_789",
  "planHash": "sha256:plan",
  "driftReportHash": "sha256:drift",
  "expiresAt": "2026-06-02T06:45:00+11:00",
  "steps": [
    {
      "stepId": "write_lesson",
      "toolName": "file.write",
      "action": "rerun",
      "consume": "multi_local"
    },
    {
      "stepId": "send_telegram",
      "toolName": "telegram.sendMessage",
      "action": "reuse_receipt",
      "receiptRef": "telegram:-5115329245:13120",
      "consume": "single_external"
    },
    {
      "stepId": "git_push",
      "toolName": "git.push",
      "action": "rerun",
      "idempotencyKey": "course-462-push",
      "consume": "single_external"
    }
  ]
}
~~~

再配一个消费回执：

~~~json
{
  "permitId": "replay_permit_123",
  "runId": "run_retry_789",
  "stepId": "send_telegram",
  "decision": "returned_cached_receipt",
  "receiptRef": "telegram:-5115329245:13120",
  "consumedAt": "2026-06-02T06:32:10+11:00",
  "fenceToken": 17
}
~~~

注意：reuse_receipt 也算消费。它消费的是“允许把旧工具结果注入本轮 run”的许可，而不是重新调用外部工具。

---

## 3. learn-claude-code：教学版消费判定器

教学版可以先写成纯函数：输入 permit step、当前 tool call、已有 consumption receipts，输出 allow、return_cached_result 或 deny。

~~~python
# learn_claude_code/runtime/replay_permit_gate.py
from dataclasses import dataclass
from typing import Literal
import time

ReplayAction = Literal["rerun", "reuse_receipt", "skip", "manual_review", "block"]
Decision = Literal["allow", "return_cached_result", "deny"]
ConsumeMode = Literal["multi_local", "single_external"]


@dataclass(frozen=True)
class PermitStep:
    step_id: str
    tool_name: str
    action: ReplayAction
    consume: ConsumeMode
    receipt_ref: str | None = None
    idempotency_key: str | None = None


@dataclass(frozen=True)
class ReplayPermit:
    permit_id: str
    new_run_id: str
    expires_at: float
    steps: list[PermitStep]


@dataclass(frozen=True)
class ToolCall:
    run_id: str
    step_id: str
    tool_name: str


@dataclass(frozen=True)
class ConsumptionReceipt:
    permit_id: str
    step_id: str
    decision: Decision


@dataclass(frozen=True)
class GateResult:
    decision: Decision
    reason: str
    receipt_ref: str | None = None
    idempotency_key: str | None = None


def evaluate_permit_call(
    permit: ReplayPermit,
    call: ToolCall,
    receipts: list[ConsumptionReceipt],
    now: float | None = None,
) -> GateResult:
    now = now or time.time()

    if call.run_id != permit.new_run_id:
        return GateResult("deny", "permit_bound_to_different_run")

    if now > permit.expires_at:
        return GateResult("deny", "permit_expired")

    step = next((s for s in permit.steps if s.step_id == call.step_id), None)
    if step is None:
        return GateResult("deny", "step_not_in_permit")

    if step.tool_name != call.tool_name:
        return GateResult("deny", "tool_name_mismatch")

    already_consumed = any(
        r.permit_id == permit.permit_id and r.step_id == step.step_id
        for r in receipts
    )

    if step.consume == "single_external" and already_consumed:
        return GateResult("deny", "single_use_step_already_consumed")

    if step.action == "reuse_receipt":
        return GateResult(
            "return_cached_result",
            "external_effect_reused_from_previous_receipt",
            receipt_ref=step.receipt_ref,
        )

    if step.action in ("manual_review", "block"):
        return GateResult("deny", f"step_action_{step.action}")

    if step.action == "skip":
        return GateResult("deny", "step_marked_skip")

    return GateResult("allow", "step_allowed_by_replay_permit", idempotency_key=step.idempotency_key)
~~~

两个测试最关键：

~~~python
def test_reuse_receipt_never_calls_external_tool():
    permit = ReplayPermit(
        "permit_1",
        "run_retry",
        expires_at=9999999999,
        steps=[
            PermitStep(
                "send_telegram",
                "telegram.sendMessage",
                "reuse_receipt",
                "single_external",
                receipt_ref="telegram:13120",
            )
        ],
    )

    result = evaluate_permit_call(
        permit,
        ToolCall("run_retry", "send_telegram", "telegram.sendMessage"),
        receipts=[],
    )

    assert result.decision == "return_cached_result"
    assert result.receipt_ref == "telegram:13120"


def test_single_external_step_cannot_be_consumed_twice():
    permit = ReplayPermit(
        "permit_1",
        "run_retry",
        expires_at=9999999999,
        steps=[PermitStep("git_push", "git.push", "rerun", "single_external")],
    )

    result = evaluate_permit_call(
        permit,
        ToolCall("run_retry", "git_push", "git.push"),
        receipts=[ConsumptionReceipt("permit_1", "git_push", "allow")],
    )

    assert result.decision == "deny"
    assert result.reason == "single_use_step_already_consumed"
~~~

教学版的目标不是复杂，而是把“permit 会被消费”这个观念刻进 runtime。

---

## 4. pi-mono：生产版 ToolDispatchGate

生产版要把消费闸门放在工具分发前，并且用数据库事务锁住 permit step。否则两个 worker 同时通过检查，仍然可能重复执行。

~~~ts
type ReplayAction = "rerun" | "reuse_receipt" | "skip" | "manual_review" | "block";
type ConsumeMode = "multi_local" | "single_external";

type ReplayPermitStep = {
  stepId: string;
  toolName: string;
  action: ReplayAction;
  consume: ConsumeMode;
  receiptRef?: string;
  idempotencyKey?: string;
};

type ReplayPermit = {
  permitId: string;
  newRunId: string;
  planHash: string;
  driftReportHash: string;
  expiresAt: number;
  steps: ReplayPermitStep[];
};

class ReplayPermitConsumptionGate {
  constructor(
    private readonly permits: ReplayPermitStore,
    private readonly receipts: ConsumptionReceiptStore,
    private readonly effectReceipts: EffectReceiptStore,
  ) {}

  async beforeToolCall(ctx: ToolContext, call: ToolCall): Promise<ToolDecision> {
    return this.permits.transaction(async (tx) => {
      const permit = await tx.getPermitForRun(ctx.runId, { forUpdate: true });
      if (!permit) return { type: "allow" };

      if (Date.now() > permit.expiresAt) {
        await tx.writeBlockedReceipt(permit.permitId, call.stepId, "permit_expired");
        return { type: "deny", reason: "permit_expired" };
      }

      const step = permit.steps.find((s) => s.stepId === call.stepId);
      if (!step) {
        await tx.writeBlockedReceipt(permit.permitId, call.stepId, "step_not_in_permit");
        return { type: "deny", reason: "step_not_in_permit" };
      }

      if (step.toolName !== call.toolName) {
        await tx.writeBlockedReceipt(permit.permitId, call.stepId, "tool_name_mismatch");
        return { type: "deny", reason: "tool_name_mismatch" };
      }

      const consumed = await tx.hasConsumptionReceipt(permit.permitId, step.stepId);
      if (step.consume === "single_external" && consumed) {
        return { type: "deny", reason: "single_use_step_already_consumed" };
      }

      if (step.action === "reuse_receipt") {
        const receipt = await this.effectReceipts.get(step.receiptRef!);
        await tx.writeConsumptionReceipt({
          permitId: permit.permitId,
          runId: ctx.runId,
          stepId: step.stepId,
          decision: "returned_cached_receipt",
          receiptRef: step.receiptRef,
          fenceToken: ctx.fenceToken,
        });

        return {
          type: "return_cached_result",
          result: receipt.toolResult,
          reason: "replay_reuses_verified_external_effect",
        };
      }

      if (step.action === "manual_review" || step.action === "block" || step.action === "skip") {
        await tx.writeBlockedReceipt(permit.permitId, step.stepId, step.action);
        return { type: "deny", reason: "step_action_" + step.action };
      }

      await tx.writeConsumptionReceipt({
        permitId: permit.permitId,
        runId: ctx.runId,
        stepId: step.stepId,
        decision: "allowed_tool_call",
        idempotencyKey: step.idempotencyKey,
        fenceToken: ctx.fenceToken,
      });

      return { type: "allow", idempotencyKey: step.idempotencyKey };
    });
  }
}
~~~

这里有三个生产要点：

- forUpdate 或等价 CAS 必须有，不然 single-use 只是纸面规则；
- return_cached_result 必须写 ConsumptionReceipt，不然后续无法解释为什么本轮没有发消息；
- idempotencyKey 要从 permit 进入真实 tool call，不能让工具自己临时生成。

---

## 5. OpenClaw 课程 Cron：实战映射

拿这门课程 cron 举例，一次任务大概是：

~~~text
1. 写 lessons/462-xxx.md
2. 更新 README.md
3. 更新 TOOLS.md 已讲内容
4. 发 Telegram 群消息
5. git add / commit / push
6. 写 daily memory
~~~

如果任务在第 4 步发完 Telegram 后取消，下一轮 retry 不能重新发同一课。正确做法是：

~~~text
send_telegram:
  action = reuse_receipt
  receiptRef = telegram:-5115329245:<messageId>
  consume = single_external

git_push:
  action = rerun
  idempotencyKey = agent-course-462
  consume = single_external
~~~

工具分发层看到 send_telegram 时，应该返回旧 Telegram receipt 给 Agent，让 Agent 知道“群已经发过”，然后继续补 README、TOOLS、commit、push。它不应该再次调用 Telegram send。

这就是 Replay Permit Consumption Gate 的价值：**把续跑从“模型记得不要重复做”变成“runtime 不允许重复做”。**

---

## 6. 常见坑

第一，把 permit 当配置读，不写消费回执。这样事故复盘时只能看到“应该没执行”，看不到“实际有没有执行”。

第二，只在外部工具内部做判断。Telegram 工具有判断，Git 工具没有判断，最后规则会碎成一地。闸门应该在统一 dispatch 层。

第三，permit 没有过期时间。retry 前的漂移探针结果是有时效的，permit 也必须有 TTL。

第四，permit 没绑定 runId。任何后台任务都能拿来用，等于绕开了 retry chain。

第五，reuse_receipt 没有伪装成工具结果返回。后续步骤拿不到 messageId、commit hash、deployment id，就会诱导 Agent 再调用一次真实工具。

---

## 7. 工程检查清单

- ReplayPermit 是否绑定 retryOfRunId、newRunId、planHash、driftReportHash？
- permit 是否有 expiresAt？
- 外部副作用 step 是否 single-use？
- 工具分发层是否在事务中检查和写入 ConsumptionReceipt？
- reuse_receipt 是否返回 cached tool result，而不是重新调用工具？
- deny/block/manual_review 是否也写 BlockedReceipt？
- 每个真实外部工具是否能接收 permit 提供的 idempotencyKey？
- permit 消费后是否能被 closeout / audit / 下次 retry 读取？

---

## 8. 记住

成熟 Agent 的 retry 不是“再跑一次”。

更准确地说：

~~~text
retry = drift probe + replay plan + single-use permit + consumption receipt
~~~

ReplayPermit 的价值不在于“允许执行”，而在于把允许范围压到最小、把每次使用写成证据、把重复副作用挡在工具调用之前。

当 permit 变成一次性、按 step、带租约、带回执的工程对象，取消后的续跑才真正可控。
