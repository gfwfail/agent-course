# 463. Agent 取消重试后的重放收据链关闭闸门（Replay Receipt Chain Closeout Gate）

上一课讲了 **Replay Permit Consumption Gate**：ReplayPermit 不是长期通行证，而是一次性、可过期、按 step 消费、能写 ConsumptionReceipt 的执行许可。

但生产系统里还有最后一个坑：**每个 step 都消费成功，不代表这次 retry 可以关闭。**

因为 retry 的最终问题不是“工具有没有跑完”，而是：

~~~text
这次重放是否形成了一条完整、连续、可验证的证据链？
~~~

如果没有关闭闸门，常见事故是：

- step 1 rerun 成功，step 2 reuse 旧 Telegram receipt，但最终报告没有引用旧 receipt；
- git push 成功了，但使用的是旧 idempotencyKey，无法证明本轮 retry 和远端 commit 对上；
- permit 被消费了，但 consumption receipt 没有写入 ledger；
- drift report 的 hash 和 replay plan 绑定不上，审计时不知道本轮 retry 基于哪次现实探测；
- final answer 发出去了，但还有一个 external step 处于 unknown；
- 后续再次 retry 时，只看到“上次 completed”，不知道哪些副作用该 reuse、哪些要重新探测。

所以今天补上 **Replay Receipt Chain Closeout Gate**：

~~~text
DriftReport
  -> ReplayPlan
  -> ReplayPermit
  -> ConsumptionReceipt[]
  -> ToolResultReceipt[]
  -> RealityRecheck[]
  -> ReplayCloseoutReceipt | ReplayCloseoutBlocked
~~~

一句话：**retry 的终点不是 completed，而是 closeout receipt 能证明每个 permit、每个工具结果、每个外部现实都对得上。**

---

## 1. 为什么需要收据链关闭

前三课已经把 retry 做得比较安全了：

- Retry Replay Guard：决定 rerun / reuse_receipt / reconcile / block；
- Pre-Replay Reality Drift Probe：retry 前重新探测外部现实；
- Replay Permit Consumption Gate：执行时按 step 消费 permit。

但这三层解决的是“能不能开始”和“能不能执行”。关闭闸门解决的是“能不能宣布结束”。

关闭时至少要检查六件事：

~~~text
1. DriftReport hash 是否等于 ReplayPermit 绑定的 driftReportHash？
2. ReplayPlan hash 是否等于 ReplayPermit 绑定的 planHash？
3. permit 中每个 required step 是否都有 ConsumptionReceipt？
4. 每个 rerun 外部副作用是否有 ToolResultReceipt + RealityRecheck？
5. 每个 reuse_receipt 是否引用了旧 closeout 里的 matched receipt？
6. 是否还有 unknown / pending / manual_review 没被收口？
~~~

这一步很像数据库事务的 commit 校验：不是每条 SQL 没报错就行，还要证明事务日志完整、约束满足、可恢复。

---

## 2. 最小数据模型

一个关闭闸门可以先用四类输入：

~~~json
{
  "retryRunId": "run_retry_789",
  "retryOfRunId": "run_cancelled_456",
  "permitId": "replay_permit_123",
  "planHash": "sha256:plan",
  "driftReportHash": "sha256:drift",
  "consumptionReceipts": [
    {
      "stepId": "send_telegram",
      "decision": "returned_cached_receipt",
      "receiptRef": "telegram:-5115329245:13120"
    },
    {
      "stepId": "git_push",
      "decision": "allowed_rerun",
      "toolResultRef": "git:commit:b4f91a9"
    }
  ],
  "realityRechecks": [
    {
      "stepId": "send_telegram",
      "status": "matched",
      "remoteRef": "telegram:-5115329245:13120"
    },
    {
      "stepId": "git_push",
      "status": "matched",
      "remoteRef": "origin/main:b4f91a9"
    }
  ]
}
~~~

关闭输出不要只写 boolean，应该写结构化 receipt：

~~~json
{
  "kind": "ReplayCloseoutReceipt",
  "retryRunId": "run_retry_789",
  "retryOfRunId": "run_cancelled_456",
  "permitId": "replay_permit_123",
  "decision": "allow_close",
  "chainHash": "sha256:receipt-chain",
  "closedAt": "2026-06-02T09:30:00+11:00",
  "terminalEffects": [
    "telegram:-5115329245:13120",
    "origin/main:b4f91a9"
  ]
}
~~~

如果不能关闭，也要写 blocked receipt：

~~~json
{
  "kind": "ReplayCloseoutBlocked",
  "retryRunId": "run_retry_789",
  "decision": "hold_reconcile",
  "reasons": ["missing_consumption_receipt:git_push", "external_reality_unknown:telegram"]
}
~~~

注意：blocked 也要落账。否则下一次 retry 又会从一团雾里开始。

---

## 3. learn-claude-code：教学版关闭判定器

教学版可以先写纯函数：输入 permit step、消费回执、现实复查结果，输出 allow_close / hold_reconcile / manual_review。

~~~python
# learn_claude_code/runtime/replay_closeout_gate.py
from dataclasses import dataclass
from typing import Literal

Action = Literal["rerun", "reuse_receipt", "skip", "manual_review", "block"]
Decision = Literal["allow_close", "hold_reconcile", "manual_review", "block_close"]
RealityStatus = Literal["matched", "missing", "mismatch", "unknown", "not_required"]


@dataclass(frozen=True)
class PermitStep:
    step_id: str
    tool_name: str
    action: Action
    external: bool
    required: bool = True
    receipt_ref: str | None = None


@dataclass(frozen=True)
class ReplayPermit:
    permit_id: str
    plan_hash: str
    drift_report_hash: str
    steps: list[PermitStep]


@dataclass(frozen=True)
class ReplayExecution:
    permit_id: str
    plan_hash: str
    drift_report_hash: str
    consumption_step_ids: set[str]
    reality_by_step: dict[str, RealityStatus]


@dataclass(frozen=True)
class CloseoutResult:
    decision: Decision
    reasons: list[str]


def evaluate_replay_closeout(permit: ReplayPermit, execution: ReplayExecution) -> CloseoutResult:
    reasons: list[str] = []

    if execution.permit_id != permit.permit_id:
        reasons.append("permit_id_mismatch")

    if execution.plan_hash != permit.plan_hash:
        reasons.append("plan_hash_mismatch")

    if execution.drift_report_hash != permit.drift_report_hash:
        reasons.append("drift_report_hash_mismatch")

    for step in permit.steps:
        if step.action in ("block", "manual_review"):
            reasons.append(f"step_not_executable:{step.step_id}:{step.action}")
            continue

        if step.required and step.action != "skip" and step.step_id not in execution.consumption_step_ids:
            reasons.append(f"missing_consumption_receipt:{step.step_id}")

        if step.external:
            status = execution.reality_by_step.get(step.step_id, "unknown")
            if status != "matched":
                reasons.append(f"external_reality_{status}:{step.step_id}")

        if step.action == "reuse_receipt" and not step.receipt_ref:
            reasons.append(f"reuse_receipt_missing_ref:{step.step_id}")

    if not reasons:
        return CloseoutResult("allow_close", [])

    if any(r.startswith("step_not_executable") for r in reasons):
        return CloseoutResult("manual_review", reasons)

    if any("mismatch" in r for r in reasons):
        return CloseoutResult("block_close", reasons)

    return CloseoutResult("hold_reconcile", reasons)
~~~

两个测试非常值钱：

~~~python
def test_closeout_requires_consumption_receipt_for_required_step():
    permit = ReplayPermit(
        "permit_1",
        "sha256:plan",
        "sha256:drift",
        [PermitStep("git_push", "git.push", "rerun", external=True)],
    )

    result = evaluate_replay_closeout(
        permit,
        ReplayExecution(
            "permit_1",
            "sha256:plan",
            "sha256:drift",
            consumption_step_ids=set(),
            reality_by_step={"git_push": "matched"},
        ),
    )

    assert result.decision == "hold_reconcile"
    assert "missing_consumption_receipt:git_push" in result.reasons


def test_closeout_blocks_on_external_mismatch():
    permit = ReplayPermit(
        "permit_1",
        "sha256:plan",
        "sha256:drift",
        [PermitStep("send_telegram", "telegram.sendMessage", "reuse_receipt", external=True, receipt_ref="telegram:13120")],
    )

    result = evaluate_replay_closeout(
        permit,
        ReplayExecution(
            "permit_1",
            "sha256:plan",
            "sha256:drift",
            consumption_step_ids={"send_telegram"},
            reality_by_step={"send_telegram": "mismatch"},
        ),
    )

    assert result.decision == "block_close"
    assert "external_reality_mismatch:send_telegram" in result.reasons
~~~

这段代码故意很小：它只教一个核心观念，**retry closeout 是证据完整性检查，不是流程状态检查。**

---

## 4. pi-mono：生产版 Closeout Worker

在生产版里，关闭闸门应该是一个独立 worker，不要塞进 prompt 或 final answer 生成逻辑。

~~~ts
type ReplayCloseoutDecision =
  | "allow_close"
  | "hold_reconcile"
  | "manual_review"
  | "block_close";

interface ReplayCloseoutInput {
  retryRunId: string;
  retryOfRunId: string;
  permitId: string;
  expectedPlanHash: string;
  expectedDriftReportHash: string;
}

interface ReplayCloseoutReceipt {
  kind: "ReplayCloseoutReceipt";
  retryRunId: string;
  retryOfRunId: string;
  permitId: string;
  decision: ReplayCloseoutDecision;
  reasons: string[];
  chainHash: string;
  terminalRefs: string[];
  createdAt: string;
}

class ReplayReceiptChainCloseoutWorker {
  constructor(
    private readonly permits: ReplayPermitStore,
    private readonly consumptions: ConsumptionReceiptStore,
    private readonly toolResults: ToolResultReceiptStore,
    private readonly realityProbe: RealityProbeService,
    private readonly receipts: ReplayCloseoutReceiptStore,
    private readonly outbox: Outbox,
  ) {}

  async closeout(input: ReplayCloseoutInput): Promise<ReplayCloseoutReceipt> {
    return await this.receipts.transaction(async (tx) => {
      const permit = await this.permits.getForUpdate(input.permitId, tx);
      if (!permit) throw new Error("permit_not_found");

      const consumptionReceipts = await this.consumptions.listByPermit(input.permitId, tx);
      const toolResults = await this.toolResults.listByRun(input.retryRunId, tx);
      const reality = await this.recheckExternalReality(permit, toolResults);

      const evaluation = evaluateReplayCloseout({
        permit,
        expectedPlanHash: input.expectedPlanHash,
        expectedDriftReportHash: input.expectedDriftReportHash,
        consumptionReceipts,
        toolResults,
        reality,
      });

      const receipt: ReplayCloseoutReceipt = {
        kind: "ReplayCloseoutReceipt",
        retryRunId: input.retryRunId,
        retryOfRunId: input.retryOfRunId,
        permitId: input.permitId,
        decision: evaluation.decision,
        reasons: evaluation.reasons,
        terminalRefs: evaluation.terminalRefs,
        chainHash: hashReceiptChain({
          permit,
          consumptionReceipts,
          toolResults,
          reality,
        }),
        createdAt: new Date().toISOString(),
      };

      await this.receipts.insert(receipt, tx);

      if (receipt.decision === "allow_close") {
        await this.outbox.enqueue("retry.closeout.closed", receipt, tx);
      } else {
        await this.outbox.enqueue("retry.closeout.blocked", receipt, tx);
      }

      return receipt;
    });
  }

  private async recheckExternalReality(
    permit: ReplayPermit,
    toolResults: ToolResultReceipt[],
  ): Promise<RealityCheck[]> {
    const externalSteps = permit.steps.filter((step) => step.external);
    return Promise.all(
      externalSteps.map((step) =>
        this.realityProbe.check({
          stepId: step.stepId,
          toolName: step.toolName,
          expectedRef: step.receiptRef ?? toolResults.find((r) => r.stepId === step.stepId)?.externalRef,
        }),
      ),
    );
  }
}
~~~

几个生产细节：

- closeout worker 要用事务锁住 permit，避免两个 closeout 同时写不同结论；
- chainHash 必须覆盖 permit、consumption、tool result、reality check；
- allow_close 之后才能释放 downstream gate；
- hold_reconcile 应该自动开 reconcile ticket，不应该让 run 静默停在 completed；
- block_close 不能自动重试，通常说明现实和证据冲突。

---

## 5. OpenClaw 实战：课程 cron 的关闭清单

拿这套课程 cron 举例，一次成功发布不只是“我发了消息、写了文件、git push 了”。

更可靠的 closeout 应该像这样：

~~~json
{
  "run": "agent-course-463",
  "steps": [
    {
      "stepId": "write_lesson",
      "type": "local",
      "evidence": "lessons/463-cancelled-run-replay-receipt-chain-closeout-gate.md exists"
    },
    {
      "stepId": "update_readme",
      "type": "local",
      "evidence": "README.md contains 463 entry"
    },
    {
      "stepId": "send_telegram",
      "type": "external",
      "evidence": "Telegram messageId captured"
    },
    {
      "stepId": "git_push",
      "type": "external",
      "evidence": "origin/main contains commit hash"
    },
    {
      "stepId": "update_tools",
      "type": "memory",
      "evidence": "TOOLS.md contains topic title"
    }
  ],
  "closeout": "allow_close"
}
~~~

如果某一步失败，下一轮 retry 不应该重新发一遍 Telegram 或盲目 push，而是先读上一轮 closeout/blocked receipt：

~~~text
if send_telegram matched:
  reuse_receipt(messageId)
else if send_telegram unknown:
  probe Telegram by content hash before sending

if git_push matched:
  reuse_receipt(commitHash)
else if git_push missing:
  rerun with same idempotency key

if README updated but TOOLS missing:
  rerun only memory update step
~~~

这就是收据链的实际价值：它让 retry 从“凭感觉补一下”变成“按证据续跑”。

---

## 6. 常见坑

第一，**把 completed 当 closeout**。completed 只说明 Agent loop 停了，不说明外部现实匹配。

第二，**只存最后结论，不存链路 hash**。没有 chainHash，后面谁改了 consumption receipt 或 reality check，你很难发现。

第三，**reuse_receipt 不复查现实**。旧 receipt 可能当时 matched，现在消息被删、commit 被 force push、部署 job 被回滚。关闭前仍要 recheck。

第四，**blocked 不落账**。失败也要写 ReplayCloseoutBlocked，否则下一轮没有起点。

第五，**关闭闸门和 final answer 绑死**。final answer 可以失败或被截断，closeout receipt 应该由 worker 写入持久化账本。

---

## 7. 记忆口诀

取消重试的闭环可以记成五句话：

~~~text
ReplayPlan 决定怎么续跑；
DriftProbe 确认旧世界还在；
ReplayPermit 限制谁能执行；
ConsumptionReceipt 记录每次放行；
CloseoutReceipt 证明整条链可以关闭。
~~~

成熟 Agent 的 retry，不是“再跑一次直到看起来成功”，而是每一次续跑都能证明：**我基于哪份现实、使用哪张许可、消费哪些 step、触发或复用了哪些副作用，最后为什么可以安全关闭。**
