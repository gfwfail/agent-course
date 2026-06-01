# 457. Agent 取消后的外部副作用对账闸门（Cancelled Run Side Effect Reconciliation Gate）

> 关键词：cancel side effect、effect ledger、reality probe、forward fix、downstream gate

上一课讲了取消后的孤儿任务收割器：父 run cancelled 以后，要把还在跑的子进程、background task、sub-agent 和外部 job 找出来收口。

今天继续补上另一个生产事故高发点：**run 被取消时，已经发生或可能已经发生的外部副作用怎么办？**

外部副作用包括：

~~~text
send message
create PR / push git commit
charge payment
deploy release
write customer-facing database row
call third-party API
~~~

这些动作一旦跨出本地进程，取消按钮就不等于回滚按钮。成熟 Agent 要做的是：把取消时刻附近的副作用冻结在 ledger 里，逐个 probe 现实，判断 matched、missing、partial、duplicate、unknown，再决定 close、retry、forward-fix、block downstream 或升级人工。

一句话：**孤儿任务收割器处理“还在跑的后代”；取消副作用对账闸门处理“已经出门的现实影响”。**

---

## 1. 为什么取消不能假装副作用没发生

很多系统把取消实现成：

~~~text
abort signal received
mark run = cancelled
stop streaming output
return to user
~~~

这只能停住 UI 和当前调用栈，不能改变已经发生的外部现实。

典型坏例子：

- Telegram 消息已经发出，但本地 run 因取消没有保存 messageId；
- git push 已经完成，但 README/TOOLS 的本地状态还没记录 commit；
- 支付 API 返回超时，用户取消了，但远端其实已经扣款；
- deploy job 已创建，父 run cancelled 后没人检查它最后是成功还是失败；
- sub-agent 提交了 patch，但父 run 取消后主线程把它当成“没发生”。

这些状态最危险的地方在于：**本地以为取消了，远端以为成功了。**

所以取消后的副作用治理需要一条单独的闸门：

~~~text
CancelRequest
  -> freeze EffectLedger slice
  -> RealityProbe each effect
  -> classify divergence
  -> SideEffectReconciliationReceipt
  -> DownstreamReleaseGate | ForwardFixCase | ManualReview
~~~

---

## 2. 最小数据模型

建议把所有外部副作用都登记成 EffectAttempt：

- **effectId**：本地唯一 ID；
- **ownerRunId**：属于哪个 Agent run；
- **kind**：message、git_push、payment、deploy、db_write、third_party_job；
- **idempotencyKey**：远端幂等键；
- **payloadHash**：要发出去的内容哈希；
- **phase**：prepared、dispatched、acknowledged、unknown_after_cancel；
- **remoteRef**：远端返回的 messageId、commitSha、paymentId、deployId；
- **obligations**：后续必须完成的事，例如写 README、保存 receipt、通知用户、清缓存。

取消发生时，不要只看 run 状态，要按时间窗口冻结副作用切片：

~~~text
run.startedAt <= effect.dispatchedAt <= cancel.finalizedAt + graceWindow
~~~

为什么要加 graceWindow？因为取消信号传播有延迟，某个 worker 可能在父 run 标记 cancelled 后几秒才真正停下。

---

## 3. learn-claude-code：教学版纯函数对账器

教学版先写一个纯函数，把本地 ledger 和远端 reality snapshot 分类清楚。

~~~python
# learn_claude_code/runtime/cancel_side_effect_reconcile.py
from dataclasses import dataclass
from typing import Literal

EffectKind = Literal["message", "git_push", "payment", "deploy", "db_write", "third_party_job"]
LocalPhase = Literal["prepared", "dispatched", "acknowledged", "unknown_after_cancel"]
RemoteState = Literal["present", "missing", "duplicate", "partial", "unknown"]
Decision = Literal[
    "close_matched",
    "retry_if_idempotent",
    "open_forward_fix",
    "block_downstream",
    "manual_review",
]


@dataclass(frozen=True)
class EffectAttempt:
    effect_id: str
    owner_run_id: str
    kind: EffectKind
    phase: LocalPhase
    idempotency_key: str
    payload_hash: str
    remote_ref: str | None = None


@dataclass(frozen=True)
class RealitySnapshot:
    remote_state: RemoteState
    remote_ref: str | None
    payload_hash: str | None
    reversible: bool


@dataclass(frozen=True)
class ReconcileDecision:
    decision: Decision
    reason: str
    required_actions: tuple[str, ...]


def reconcile_cancelled_effect(
    effect: EffectAttempt,
    reality: RealitySnapshot,
) -> ReconcileDecision:
    if reality.remote_state == "present" and reality.payload_hash == effect.payload_hash:
        return ReconcileDecision(
            "close_matched",
            "remote_effect_matches_local_intent",
            ("write_reconciliation_receipt", "release_safe_downstream"),
        )

    if reality.remote_state == "missing":
        if effect.phase == "prepared":
            return ReconcileDecision(
                "close_matched",
                "prepared_effect_never_left_process",
                ("mark_not_dispatched",),
            )
        return ReconcileDecision(
            "retry_if_idempotent",
            "dispatched_effect_missing_remotely",
            ("retry_with_same_idempotency_key", "probe_again"),
        )

    if reality.remote_state in ("duplicate", "partial"):
        return ReconcileDecision(
            "open_forward_fix",
            f"remote_effect_is_{reality.remote_state}",
            ("create_forward_fix_case", "block_dependent_actions"),
        )

    if reality.remote_state == "present" and reality.payload_hash != effect.payload_hash:
        return ReconcileDecision(
            "manual_review",
            "remote_payload_mismatch",
            ("freeze_effect_family", "attach_evidence_packet"),
        )

    return ReconcileDecision(
        "block_downstream",
        "remote_state_unknown_after_cancel",
        ("schedule_reality_probe", "hold_downstream_release_gate"),
    )
~~~

这个函数的关键不是“自动修所有问题”，而是把取消后的模糊状态变成有限集合：

- 远端确实有，且 payload 匹配：关闭；
- 远端没有，本地只是 prepared：关闭；
- 远端没有，但本地已经 dispatched：用同一个幂等键重试或继续 probe；
- 远端 partial / duplicate：进入 forward-fix；
- payload mismatch：人工复核；
- unknown：阻断下游。

---

## 4. pi-mono：生产版取消副作用 Reconciler

生产实现里，副作用对账不应该散落在各个工具 wrapper 里。建议统一挂在 run lifecycle 上：只要 run 进入 cancelled、interrupted、timed_out，都触发一次 side-effect reconciliation。

~~~ts
type EffectKind = "message" | "git_push" | "payment" | "deploy" | "db_write" | "third_party_job";

type EffectAttempt = {
  effectId: string;
  ownerRunId: string;
  kind: EffectKind;
  phase: "prepared" | "dispatched" | "acknowledged" | "unknown_after_cancel";
  idempotencyKey: string;
  payloadHash: string;
  remoteRef?: string;
  dispatchedAt?: number;
};

type ReconciliationReceipt = {
  effectId: string;
  ownerRunId: string;
  outcome:
    | "matched"
    | "not_dispatched"
    | "retry_scheduled"
    | "forward_fix_opened"
    | "downstream_blocked"
    | "manual_review";
  evidenceRef: string;
  reconciledAt: number;
};

class CancelledRunSideEffectReconciler {
  constructor(
    private readonly effects: EffectLedger,
    private readonly probes: RealityProbeRegistry,
    private readonly receipts: ReconciliationReceiptStore,
    private readonly gates: DownstreamGateStore,
    private readonly forwardFix: ForwardFixQueue,
  ) {}

  async reconcileCancelledRun(ownerRunId: string): Promise<ReconciliationReceipt[]> {
    const attempts = await this.effects.listCancelWindow(ownerRunId);
    const written: ReconciliationReceipt[] = [];

    for (const attempt of attempts) {
      const already = await this.receipts.findFinal(attempt.effectId);
      if (already) {
        written.push(already);
        continue;
      }

      const probe = this.probes.forKind(attempt.kind);
      const reality = await probe.snapshot(attempt);
      const decision = classifyCancelledEffect(attempt, reality);

      if (decision.outcome === "downstream_blocked") {
        await this.gates.blockByEffect(attempt.effectId, decision.reason);
      }

      if (decision.outcome === "forward_fix_opened") {
        await this.forwardFix.enqueue({
          effectId: attempt.effectId,
          ownerRunId,
          reason: decision.reason,
          evidenceRef: reality.evidenceRef,
        });
      }

      const receipt = await this.receipts.writeOnce({
        effectId: attempt.effectId,
        ownerRunId,
        outcome: decision.outcome,
        evidenceRef: reality.evidenceRef,
        reconciledAt: Date.now(),
      });
      written.push(receipt);
    }

    return written;
  }
}
~~~

几个工程约束：

- EffectLedger.prepare() 必须发生在真实副作用之前；
- EffectLedger.ack() 必须保存远端 ref，不要只保存在内存；
- 对账 worker 要 writeOnce，重复执行不能创建重复补偿；
- downstream gate 必须默认 fail-closed：没有最终 receipt 就不放行；
- 对 payment、deploy、message 这种外部动作，优先 probe 远端现实，不要相信本地异常类型。

---

## 5. OpenClaw：课程 Cron 的真实例子

这门课程 cron 每次发布都至少有三类外部副作用：

~~~text
Telegram message -> messageId
Git commit/push   -> commitSha / remote main
Memory update     -> TOOLS.md 已讲内容
~~~

如果 run 在中途取消，下一次 cron 不能直接“重新讲一课”，否则可能出现：

- 群里已经发了第 457 课，但 repo 没提交；
- repo 已经 push 了 lesson，但 TOOLS 没更新，下一次选题重复；
- Telegram 发了两遍，因为本地没保存 messageId；
- README 更新了，但 lesson 文件缺失或 commit 没推上远端。

一个实用的 OpenClaw 对账策略：

~~~text
before selecting next topic:
  inspect latest lesson file
  inspect README last index
  inspect TOOLS taught list
  inspect git remote HEAD
  optionally inspect Telegram message receipt

if partial publish detected:
  reconcile existing topic first
  do not create a new topic until previous effect family is matched
~~~

也就是说，这个 cron 的“取消副作用对账闸门”不是抽象概念，它就是每次开课前检查 lesson/README/TOOLS/git/Telegram 是否互相一致。

---

## 6. 实战检查清单

给 Agent 加取消能力时，直接检查这 7 项：

- 所有外部副作用是否先写 EffectAttempt 再执行？
- 是否有稳定 idempotencyKey，取消后重试不会重复扣款/重复发消息？
- run 取消时是否冻结了 cancel window 内的 effects？
- 每类 effect 是否有 reality probe，而不是只看工具返回？
- unknown 是否阻断下游，而不是当成功或失败处理？
- partial / duplicate 是否进入 forward-fix，而不是盲目 rollback？
- receipt 是否 writeOnce，重复对账不会制造第二个补偿动作？

只要有任何一项缺失，取消就可能把系统带进“本地取消、远端成功”的裂缝。

---

## 7. 关键 takeaway

取消不是删除历史。对 Agent 来说，取消只是一个新的事实：**从这个时刻开始，所有已经发生、正在发生、可能发生的副作用都必须进入对账。**

- 第 455 课：当前工具如何收敛到终态；
- 第 456 课：取消后仍在跑的后代如何被收割；
- 第 457 课：取消前后已经出门的外部副作用如何对账、阻断和 forward-fix。

成熟 Agent 的取消协议，最终要回答三个问题：

- 后代任务还在跑吗？
- 外部现实已经改变了吗？
- 下游是否可以继续相信这个 run 的结果？

如果这三个问题没有收据，cancelled 只是一个 UI 状态，不是一个可信终态。
