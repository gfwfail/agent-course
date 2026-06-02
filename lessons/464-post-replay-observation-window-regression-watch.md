# 464. Agent 取消重试后的观察窗口与回归监控（Post-Replay Observation Window & Regression Watch）

上一课讲了 **Replay Receipt Chain Closeout Gate**：retry 的终点不是 completed，而是 closeout receipt 能证明 DriftReport、ReplayPlan、ReplayPermit、ConsumptionReceipt、ToolResultReceipt 和 RealityRecheck 全部对得上。

但生产系统里还有一个尾巴：**closeout 通过，只能证明关闭那一刻链路一致；它不能证明关闭后的几分钟、几小时里不会有迟到副作用、下游回潮或旧任务复活。**

典型事故：

- retry closeout 已经 allow_close，但上一次取消前发出的异步 webhook 过了 10 分钟才到；
- Telegram 消息和 git push 都匹配了，但 background task 后来又写了一次 summary；
- 下游 queue 在 closeout 后消费了旧 runId 的事件，把状态从 retry_result 覆盖回 cancelled；
- orphan reaper 当时 matched，后来一个子进程被 supervisor 拉起来继续跑；
- 监控只看最终 completed，没有把 closeout 后的回归信号绑定回这次 retry。

所以今天补上 **Post-Replay Observation Window & Regression Watch**：

~~~text
ReplayCloseoutReceipt
  -> ObservationLease
  -> SignalProbe[]
  -> RegressionFinding[]
  -> ObservationDecision
  -> StableCloseReceipt | ReopenReplayCase
~~~

一句话：**retry closeout 后要留一段观察窗口，把迟到副作用、状态回潮、旧 run 复活和下游覆盖都变成可探测、可重开、可关闭的工程对象。**

---

## 1. 为什么 closeout 后还要观察

ReplayCloseoutReceipt 解决的是“证据链是否完整”。

Observation Window 解决的是“外部世界是否继续稳定”。

这两个问题不一样：

~~~text
closeout gate:
  当前证据链完整吗？

observation window:
  关闭后有没有新的现实变化证明我们关早了？
~~~

最小观察信号可以先看四类：

~~~text
1. late_effect：旧 runId 或旧 idempotencyKey 的副作用迟到
2. state_regression：状态从 retry terminal 倒回 cancelled/pending/unknown
3. descendant_resurrection：已收割的子任务、进程、job 又出现
4. downstream_overwrite：下游系统用旧 payload 覆盖新结果
~~~

这一步很像数据库迁移后的观察期：migration 成功不代表业务读写路径稳定，必须看一段真实流量。

---

## 2. 最小数据模型

closeout 后生成一张观察租约：

~~~json
{
  "kind": "ObservationLease",
  "leaseId": "obs_464",
  "retryRunId": "run_retry_789",
  "retryOfRunId": "run_cancelled_456",
  "closeoutReceiptHash": "sha256:closeout",
  "watchUntil": "2026-06-02T13:00:00+11:00",
  "signals": [
    "late_effect",
    "state_regression",
    "descendant_resurrection",
    "downstream_overwrite"
  ],
  "reopenPolicy": {
    "late_effect": "reconcile",
    "state_regression": "reopen_replay",
    "descendant_resurrection": "reap_again",
    "downstream_overwrite": "manual_review"
  }
}
~~~

观察结束时不要只写“没事”，而是写稳定关闭收据：

~~~json
{
  "kind": "StableCloseReceipt",
  "leaseId": "obs_464",
  "retryRunId": "run_retry_789",
  "decision": "close_stable",
  "checkedSignals": [
    "late_effect",
    "state_regression",
    "descendant_resurrection",
    "downstream_overwrite"
  ],
  "closedAt": "2026-06-02T13:00:00+11:00"
}
~~~

如果发现回归，要生成 reopen case：

~~~json
{
  "kind": "ReopenReplayCase",
  "leaseId": "obs_464",
  "retryRunId": "run_retry_789",
  "reason": "late_effect",
  "evidenceRef": "effect:telegram:old-run:13124",
  "nextAction": "reconcile"
}
~~~

关键点：**观察窗口不是告警文本，而是一张有期限、有信号、有重开策略的租约。**

---

## 3. learn-claude-code：教学版观察判定器

教学版可以先做纯函数：输入 closeout 后采集到的 signal，输出 close_stable / extend_window / reopen_replay / manual_review。

~~~python
# learn_claude_code/runtime/post_replay_observation.py
from dataclasses import dataclass
from typing import Literal

SignalKind = Literal[
    "late_effect",
    "state_regression",
    "descendant_resurrection",
    "downstream_overwrite",
]
Severity = Literal["info", "warning", "critical"]
Decision = Literal["close_stable", "extend_window", "reopen_replay", "manual_review"]


@dataclass(frozen=True)
class ObservationLease:
    lease_id: str
    retry_run_id: str
    closeout_receipt_hash: str
    watch_signals: set[SignalKind]
    min_probe_count: int


@dataclass(frozen=True)
class SignalProbe:
    signal: SignalKind
    matched: bool
    severity: Severity
    evidence_ref: str | None = None


@dataclass(frozen=True)
class ObservationResult:
    decision: Decision
    reasons: list[str]
    evidence_refs: list[str]


def evaluate_post_replay_observation(
    lease: ObservationLease,
    probes: list[SignalProbe],
) -> ObservationResult:
    reasons: list[str] = []
    evidence_refs: list[str] = []

    if len(probes) < lease.min_probe_count:
        return ObservationResult("extend_window", ["insufficient_probe_count"], [])

    seen = {probe.signal for probe in probes}
    missing = lease.watch_signals - seen
    if missing:
        return ObservationResult(
            "extend_window",
            [f"missing_signal_probe:{signal}" for signal in sorted(missing)],
            [],
        )

    for probe in probes:
        if not probe.matched:
            continue

        if probe.evidence_ref:
            evidence_refs.append(probe.evidence_ref)

        if probe.signal == "downstream_overwrite":
            reasons.append("downstream_overwrite_requires_human_review")
        elif probe.severity == "critical":
            reasons.append(f"critical_regression:{probe.signal}")
        else:
            reasons.append(f"observed_regression:{probe.signal}")

    if not reasons:
        return ObservationResult("close_stable", [], [])

    if any(reason == "downstream_overwrite_requires_human_review" for reason in reasons):
        return ObservationResult("manual_review", reasons, evidence_refs)

    return ObservationResult("reopen_replay", reasons, evidence_refs)
~~~

两个测试最有用：

~~~python
def test_close_stable_when_all_watch_signals_are_clean():
    lease = ObservationLease(
        "obs_1",
        "run_retry_1",
        "sha256:closeout",
        {"late_effect", "state_regression"},
        min_probe_count=2,
    )

    result = evaluate_post_replay_observation(
        lease,
        [
            SignalProbe("late_effect", matched=False, severity="info"),
            SignalProbe("state_regression", matched=False, severity="info"),
        ],
    )

    assert result.decision == "close_stable"


def test_reopen_when_late_effect_arrives_after_closeout():
    lease = ObservationLease(
        "obs_1",
        "run_retry_1",
        "sha256:closeout",
        {"late_effect"},
        min_probe_count=1,
    )

    result = evaluate_post_replay_observation(
        lease,
        [SignalProbe("late_effect", matched=True, severity="critical", evidence_ref="effect:old-run")],
    )

    assert result.decision == "reopen_replay"
    assert result.evidence_refs == ["effect:old-run"]
~~~

这和 learn-claude-code 的 BackgroundManager.drain_notifications() 是同一类思想：后台结果不能丢在对话外，要被重新注入 agent loop。Observation Window 做的是生产版注入：把 closeout 后迟到的现实信号重新注入恢复流程。

---

## 4. pi-mono：事件流里的观察 Worker

pi-mono 的 Agent.subscribe() 已经能把 agent 运行事件旁路出来：

~~~ts
agent.subscribe((event) => {
  // message_start / message_end / turn_end / agent_end ...
});
~~~

生产版可以在 agent_end 或 replay closeout 后创建观察租约，然后由 Worker 订阅外部信号：

~~~ts
type ObservationDecision =
  | { type: "close_stable"; leaseId: string }
  | { type: "extend_window"; leaseId: string; reasons: string[] }
  | { type: "reopen_replay"; leaseId: string; reasons: string[]; evidenceRefs: string[] }
  | { type: "manual_review"; leaseId: string; reasons: string[]; evidenceRefs: string[] };

type SignalProbe = {
  leaseId: string;
  retryRunId: string;
  signal: "late_effect" | "state_regression" | "descendant_resurrection" | "downstream_overwrite";
  matched: boolean;
  severity: "info" | "warning" | "critical";
  evidenceRef?: string;
};

async function closeObservationWindow(
  lease: ObservationLease,
  probes: SignalProbe[],
  tx: DatabaseTransaction,
): Promise<ObservationDecision> {
  const decision = evaluateObservationLease(lease, probes);

  await tx.observationReceipts.insert({
    leaseId: lease.id,
    retryRunId: lease.retryRunId,
    closeoutReceiptHash: lease.closeoutReceiptHash,
    decision: decision.type,
    reasons: "reasons" in decision ? decision.reasons : [],
    evidenceRefs: "evidenceRefs" in decision ? decision.evidenceRefs : [],
    createdAt: new Date(),
  });

  if (decision.type === "reopen_replay") {
    await tx.replayCases.insert({
      retryRunId: lease.retryRunId,
      sourceLeaseId: lease.id,
      reasons: decision.reasons,
      evidenceRefs: decision.evidenceRefs,
      status: "pending",
    });
  }

  return decision;
}
~~~

注意两个实现细节：

- 观察租约要绑定 closeoutReceiptHash，否则无法证明观察的是哪次 closeout；
- reopen case 要带 sourceLeaseId，否则下一轮 retry 不知道自己是被哪个回归信号拉起来的。

---

## 5. OpenClaw 实战：课程 Cron 的观察窗口

拿我们这个课程 Cron 举例，retry closeout 后可以观察这些信号：

~~~text
telegram_message:
  新 messageId 是否只出现一次？
  是否有旧 retry 又补发了一条？

git_remote:
  origin/main 是否停在本轮 commit 或更新后的合法 commit？
  是否有旧 run 又 push 了旧目录状态？

workspace:
  README.md / TOOLS.md / lesson 文件是否被旧进程覆盖？

memory:
  daily memory 是否出现同一课多个冲突 messageId？
~~~

最小 Runbook：

~~~text
1. closeout 成功后写 ObservationLease
2. 观察窗口内定期 probe Telegram / git / workspace / memory
3. 无异常：写 StableCloseReceipt
4. 发现迟到旧副作用：写 ReopenReplayCase
5. 发现下游覆盖或证据冲突：manual_review，不再自动扩散
~~~

这也是为什么课程任务里每次都记录 messageId、commit hash、lesson path 和 TOOLS 主题：这些不是流水账，是观察窗口判断“有没有回潮”的锚点。

---

## 6. 常见坑

**坑 1：只靠 fixed sleep。**

sleep 10 分钟不等于观察。观察要有信号、有证据、有结论。

**坑 2：观察窗口没有租约。**

没有 watchUntil、owner、signals，后续 worker 不知道什么时候该关，也不知道该看什么。

**坑 3：发现异常只发告警。**

告警不是恢复流程。要生成 ReopenReplayCase 或 manual_review case，才能进入可关闭的工作流。

**坑 4：不绑定 closeout hash。**

同一个 retryRunId 可能多次 closeout 尝试，观察必须绑定具体 closeout receipt。

---

## 7. 记忆口诀

~~~text
Closeout 证明当时对，
Observation 证明后来稳。

迟到副作用要重开，
状态回潮要回放，
下游覆盖要人工审。
~~~

成熟 Agent 的 retry 不是 closeout 就真的结束，而是 closeout 后还能观察现实一段时间；稳定就写 StableCloseReceipt，回归就带证据重开。
