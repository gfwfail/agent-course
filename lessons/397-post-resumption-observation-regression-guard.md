# 397. Agent 恢复执行后的观察窗口与回归护栏（Post-Resumption Observation Window & Regression Guard）

上一课讲了 **Resumption Gate**：人工决策不能直接恢复执行，必须先变成一次性 Resumption Permit，并校验有效期、证据、目标、租约和 fenceToken。

今天讲下一步：**恢复动作执行成功，也不代表案件可以马上关闭。**

因为人工恢复通常是在系统已经出现异常、DLQ 或 unsafe failure 之后发生的。这个时候最危险的不是“能不能跑完一次”，而是：

- 旧问题是否又被触发；
- 人工 override 是否放宽过头；
- 恢复动作是否制造了新的副作用；
- 下游队列是否真的重新稳定；
- 这次人工恢复是否应该沉淀成回归用例。

一句话：**Resumption Receipt 只能证明恢复动作被允许并执行过；Post-Resumption Observation Window 要证明恢复后的真实系统没有复发、没有扩散、没有隐形回归。**

---

## 1. 为什么恢复后还要观察窗口

看一个课程 cron 例子：

1. git push 超时，任务进入 DLQ；
2. 人工确认远端其实已经收到 commit；
3. Resumption Gate 生成 permit，允许继续发最终消息和写 memory；
4. Agent 成功执行后宣布完成。

如果这里直接 close，有几个坑：

- 下一轮 cron 可能因为 TOOLS 没更新完整又选了重复主题；
- README 指向新 lesson，但远端 main 还没同步到最新；
- Telegram 消息发出去了，但 git push 其实后来被 force update 覆盖；
- 人工 override 放行了本次，但同类 timeout 下次还会再次进 DLQ；
- DLQ case 被关闭了，但原来的 dispatch ticket 还在队列里，可能被旧 worker 拿到。

所以恢复后需要一个短观察窗口：不用无限等，但要拿到足够样本证明系统重新稳定。

---

## 2. Observation Window 的最小模型

恢复观察窗口不是“睡一会再看”，而是一个结构化对象：

~~~ts
type PostResumptionWindow = {
  windowId: string;
  caseId: string;
  permitId: string;
  resumptionReceiptId: string;
  startedAt: string;
  expiresAt: string;
  sampleBudget: {
    minEvents: number;
    maxEvents: number;
    maxDurationMs: number;
  };
  watchSignals: string[];
  regressionGuards: string[];
  decision:
    | "observing"
    | "close_stable"
    | "extend_window"
    | "rollback_resumption"
    | "reopen_escalation"
    | "manual_review";
};
~~~

它要回答三个问题：

1. **稳定了吗？** 队列、外部副作用、关键状态有没有重新对齐；
2. **复发了吗？** 同一个 failure signature 有没有再次出现；
3. **该固化吗？** 这次人工恢复暴露出的测试、policy、runbook 缺口是否要写入 regression pack。

---

## 3. 观察哪些信号

不同系统信号不同，但生产 Agent 至少看五类：

- **case signal**：原 escalation case 没有新错误、没有重复 resume；
- **queue signal**：旧 dispatch ticket 被取消或 fenced，队列没有重复消费；
- **side-effect signal**：外部副作用对账仍然 matched，没有 duplicate/payload mismatch；
- **policy signal**：本次 override 没有被其他 action 复用；
- **user-facing signal**：最终输出、消息、PR、部署状态与用户可见事实一致。

如果是 OpenClaw 课程 cron，观察窗口可以检查：

- Telegram messageId 存在；
- origin/main contains commit；
- README 目录能链接到 lesson 文件；
- TOOLS 已讲内容包含本课；
- 下一课去重扫描不会再选同一主题；
- 工作区没有残留 staged diff。

---

## 4. learn-claude-code：用纯函数做窗口判定

教学版可以先不接真实队列，只写一个窗口判定器。它输入事件样本，输出观察决策。

~~~python
from dataclasses import dataclass
from typing import Literal

Decision = Literal[
    "observing",
    "close_stable",
    "extend_window",
    "rollback_resumption",
    "reopen_escalation",
    "manual_review",
]

@dataclass(frozen=True)
class ObservationEvent:
    name: str
    ok: bool
    severity: Literal["info", "warn", "error", "critical"]
    signature: str
    evidence_ref: str

@dataclass(frozen=True)
class ObservationWindow:
    min_events: int
    max_events: int
    recurrence_signatures: set[str]
    forbidden_signatures: set[str]

def evaluate_post_resumption_window(
    window: ObservationWindow,
    events: list[ObservationEvent],
) -> tuple[Decision, list[str]]:
    reasons: list[str] = []

    critical = [e for e in events if not e.ok and e.severity == "critical"]
    if critical:
        return "rollback_resumption", [f"critical:{e.signature}" for e in critical]

    forbidden = [
        e for e in events
        if not e.ok and e.signature in window.forbidden_signatures
    ]
    if forbidden:
        return "reopen_escalation", [f"forbidden:{e.signature}" for e in forbidden]

    recurrence = [
        e for e in events
        if not e.ok and e.signature in window.recurrence_signatures
    ]
    if recurrence:
        return "extend_window", [f"recurrence:{e.signature}" for e in recurrence]

    warn = [e for e in events if not e.ok and e.severity == "warn"]
    if len(warn) >= 3:
        return "manual_review", [f"warn_budget_exceeded:{len(warn)}"]

    if len(events) < window.min_events:
        return "observing", [f"need_more_events:{len(events)}/{window.min_events}"]

    if len(events) > window.max_events:
        return "manual_review", ["window_budget_exceeded"]

    return "close_stable", ["stable_sample_collected"]
~~~

这个函数的重点是：**恢复观察的输出不是“看起来没事”，而是 close / extend / rollback / reopen / review 的明确分流。**

---

## 5. pi-mono：把观察挂到事件流，不阻塞主 worker

pi-mono 的 agent runtime 已经有 Agent、Agent.subscribe、tool result 和事件流的概念。生产版不要让恢复 worker 阻塞在那里等观察结束，而是：

1. resumption worker 写入 ResumptionReceipt；
2. 创建 PostResumptionWindow；
3. 订阅相关事件流；
4. window worker 按信号更新样本；
5. 达到 close 条件后写 CloseStableReceipt；
6. 达到风险条件后发 rollback/reopen outbox。

示意代码：

~~~ts
type ObservationDecision =
  | "observing"
  | "close_stable"
  | "extend_window"
  | "rollback_resumption"
  | "reopen_escalation"
  | "manual_review";

type ObservationSample = {
  windowId: string;
  eventName: string;
  ok: boolean;
  severity: "info" | "warn" | "error" | "critical";
  signature: string;
  evidenceRef: string;
  observedAt: Date;
};

async function recordObservationSample(
  tx: DbTransaction,
  sample: ObservationSample,
) {
  await tx.observationSample.create({ data: sample });

  const window = await tx.postResumptionWindow.findUniqueOrThrow({
    where: { windowId: sample.windowId },
    include: { samples: true },
  });

  const decision = evaluatePostResumptionWindow({
    window,
    samples: [...window.samples, sample],
  });

  await tx.postResumptionWindow.update({
    where: { windowId: window.windowId },
    data: { decision: decision.kind, decisionReason: decision.reason },
  });

  if (decision.kind === "rollback_resumption") {
    await tx.outbox.create({
      data: {
        topic: "resumption.rollback.requested",
        dedupeKey: `rollback:${window.windowId}`,
        payload: { windowId: window.windowId, reason: decision.reason },
      },
    });
  }

  if (decision.kind === "reopen_escalation") {
    await tx.outbox.create({
      data: {
        topic: "escalation.reopen.requested",
        dedupeKey: `reopen:${window.caseId}:${decision.signature}`,
        payload: { caseId: window.caseId, windowId: window.windowId },
      },
    });
  }
}
~~~

这里有两个关键点：

- 观察窗口用 outbox 通知后续动作，保持幂等；
- rollback/reopen 有自己的 dedupeKey，防止每条异常事件都创建一轮恢复事故。

---

## 6. Regression Guard：把人工恢复变成未来保护

观察窗口 close_stable 之后，不应该只写“关闭”。它还要产出一个 Regression Candidate：

~~~json
{
  "candidateId": "reg_post_resume_git_push_timeout_397",
  "sourceCaseId": "esc_agent_course_397",
  "failureSignature": "git_push_timeout_remote_present",
  "expectedFutureBehavior": "reconcile_remote_before_retry_push",
  "fixtures": [
    "local_push_timeout.log",
    "origin_main_contains_commit.snapshot",
    "dirty_tree_after_resume.snapshot"
  ],
  "guardLevel": "pre_side_effect",
  "expiresAt": "2026-06-25T00:00:00Z"
}
~~~

这条 guard 的意思是：

> 下次 git push timeout 时，先查远端 commit 是否存在，再决定 retry / reconcile / manual review；不要盲目重推，也不要直接 close。

成熟 Agent 的恢复不是“这次救回来了”，而是“同类错误下次更难发生”。

---

## 7. OpenClaw 实战：课程 cron 的恢复观察清单

这类 cron 发课任务可以把观察窗口压缩成 6 个检查：

~~~text
post_resumption_window:
  - git ls-remote origin main contains commit
  - git status --short is clean
  - lesson file exists and title matches README
  - TOOLS.md contains exact topic
  - Telegram send returned messageId
  - memory receipt mentions messageId and commitSha
~~~

如果全部通过：close_stable。

如果 git 远端不含 commit：reopen_escalation 或 retry push。

如果 Telegram 发了但 git 没推：rollback 不现实，要进入 forward-fix，把 git 侧补齐。

如果 TOOLS/README 缺一个：reopen compensation，只补缺口，不重发群消息。

---

## 8. 落地原则

最后给一个判断标准：

~~~text
Resumption Gate:
  Can we safely resume this one action?

Post-Resumption Observation Window:
  Did the system stay healthy after resuming?

Regression Guard:
  Will the same failure be caught earlier next time?
~~~

三者不要混在一起。恢复许可负责“能不能继续”，观察窗口负责“继续后有没有稳定”，回归护栏负责“以后怎么不再踩同一个坑”。

成熟 Agent 的人工恢复闭环，不是拿到批准就继续跑，而是：**批准受限、执行留痕、恢复后观察、稳定后固化。**
