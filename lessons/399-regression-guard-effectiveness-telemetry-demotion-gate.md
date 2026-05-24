# 399. Agent 回归护栏效果遥测与自动降级闸门（Regression Guard Effectiveness Telemetry & Demotion Gate）

上一课讲了 **Regression Candidate Promotion & Guardrail Lifecycle**：事故里沉淀出来的防复发规则，不能直接变成永久 if，而要经过 candidate、shadow、canary、active、demoted、retired 的生命周期管理。

今天继续往下讲：**Guard 进入 active 以后，也不能默认永远正确。**

很多 Agent 系统的护栏会经历三个阶段：

- 刚上线时很有用，因为它精准挡住了刚发生过的事故；
- 运行一段时间后，业务、工具、模型、流程都变了，它开始偶尔误伤；
- 再往后，没人敢删，因为没人知道删了会不会出事。

一句话：**Regression Guard 不是“写进去就安全”，而是要持续采集 effectiveness telemetry，用数据决定保留、降级、收紧或退役。**

---

## 1. 为什么 active guard 还要被监控

看 OpenClaw 课程 cron 的例子：

上一课我们可能把 closeout guard 提升为 active，要求每次课程发布后检查：

- lesson 文件存在；
- README 目录包含链接；
- TOOLS 已讲内容已更新；
- Telegram messageId 已记录；
- origin/main 包含 commit；
- worktree 干净。

这套 guard 初期很合理。但几周后可能发生变化：

- 课程发布改成 PR 流程，不再直接 push main；
- Telegram 外发改成延迟 outbox，messageId 不会立刻出现；
- README 改成自动生成目录，不需要手工 patch；
- 网络抖动导致 remote_main_contains_commit 偶发 unknown；
- 某条检查从防事故变成了高频误伤。

如果没有效果遥测，系统只会越来越保守，最后每个正常任务都被历史事故规则拖慢。

---

## 2. Guard Effectiveness 的最小指标

一个上线后的 guard 至少要记录五类数据：

- **trigger_count**：触发了多少次；
- **prevented_failure_count**：其中多少次后来证明真的防住了事故；
- **false_positive_count**：其中多少次被人工或后续证据证明是误伤；
- **unknown_count**：触发后没有足够证据判断对错；
- **latency_cost_ms**：每次检查平均增加多少延迟。

可以建一个简单的快照模型：

~~~ts
type GuardEffectivenessSnapshot = {
  guardId: string;
  packVersion: number;
  windowStartedAt: string;
  windowEndedAt: string;
  triggerCount: number;
  preventedFailureCount: number;
  falsePositiveCount: number;
  unknownCount: number;
  avgLatencyMs: number;
  p95LatencyMs: number;
  lastIncidentAt?: string;
  lastFalsePositiveAt?: string;
};
~~~

这里最关键的是：**trigger 不等于有用。**

一个 guard 一个月触发 100 次，如果 95 次都是 false positive，它不是“很活跃”，而是在伤害系统吞吐。

---

## 3. Demotion Gate 的输出

护栏效果闸门不要只输出 pass/fail，而要把生命周期动作明确化：

- **keep_active**：继续 active；
- **tighten_scope**：规则本身有用，但 scope 太宽；
- **demote_canary**：误伤率偏高，先退回小范围；
- **demote_shadow**：只保留观察，不再阻断；
- **retire_guard**：长期无触发、路径消失或成本高于收益；
- **manual_review**：证据冲突，需要人判断。

这跟普通测试不同。普通测试问“这次能不能过”，Demotion Gate 问的是“这条护栏还值不值得留在热路径里”。

---

## 4. learn-claude-code：用纯函数做降级判定

教学版可以先写一个纯函数，根据窗口指标给出生命周期动作。

~~~python
from dataclasses import dataclass
from typing import Literal

Decision = Literal[
    "keep_active",
    "tighten_scope",
    "demote_canary",
    "demote_shadow",
    "retire_guard",
    "manual_review",
]

@dataclass(frozen=True)
class GuardSnapshot:
    trigger_count: int
    prevented_failure_count: int
    false_positive_count: int
    unknown_count: int
    avg_latency_ms: int
    p95_latency_ms: int
    days_since_last_trigger: int
    scope_too_broad: bool
    code_path_exists: bool

def decide_guard_lifecycle(snapshot: GuardSnapshot) -> tuple[Decision, list[str]]:
    if not snapshot.code_path_exists:
        return "retire_guard", ["code_path_removed"]

    if snapshot.trigger_count == 0:
        if snapshot.days_since_last_trigger >= 60:
            return "retire_guard", ["no_trigger_for_60_days"]
        return "keep_active", ["no_recent_trigger_but_within_ttl"]

    judged = snapshot.prevented_failure_count + snapshot.false_positive_count
    unknown_rate = snapshot.unknown_count / snapshot.trigger_count
    false_positive_rate = (
        snapshot.false_positive_count / judged if judged > 0 else 0.0
    )
    prevention_rate = snapshot.prevented_failure_count / snapshot.trigger_count

    if unknown_rate > 0.40:
        return "demote_shadow", [f"unknown_rate:{unknown_rate:.2f}"]

    if false_positive_rate > 0.20:
        if snapshot.scope_too_broad:
            return "tighten_scope", [f"false_positive_rate:{false_positive_rate:.2f}"]
        return "demote_canary", [f"false_positive_rate:{false_positive_rate:.2f}"]

    if snapshot.p95_latency_ms > 1500 and prevention_rate < 0.05:
        return "demote_shadow", [
            f"p95_latency_ms:{snapshot.p95_latency_ms}",
            f"prevention_rate:{prevention_rate:.2f}",
        ]

    if snapshot.prevented_failure_count == 0 and snapshot.days_since_last_trigger >= 30:
        return "manual_review", ["no_confirmed_prevention_for_30_days"]

    return "keep_active", [
        f"prevention_rate:{prevention_rate:.2f}",
        f"false_positive_rate:{false_positive_rate:.2f}",
    ]
~~~

这个函数有两个刻意设计：

1. **unknown 太高先退 shadow**：不知道对错的阻断，不应该长期 active；
2. **误伤高但 scope 过宽时先 tighten_scope**：有些 guard 不是坏，而是管太多。

---

## 5. pi-mono：把 guard 判断写成遥测事件

生产版里，不要只在日志里写 `guard blocked`。每次 guard 评估都应该产出结构化事件：

~~~ts
type GuardEvaluationEvent = {
  eventId: string;
  guardId: string;
  packVersion: number;
  workflow: string;
  runId: string;
  tier: "shadow" | "canary" | "active";
  decision: "allow" | "warn" | "block" | "unknown";
  reasonCodes: string[];
  latencyMs: number;
  evidenceRefs: string[];
  createdAt: Date;
};

type GuardOutcomeEvent = {
  eventId: string;
  guardEvaluationId: string;
  outcome:
    | "prevented_failure"
    | "false_positive"
    | "confirmed_safe"
    | "unknown";
  decidedBy: "system" | "human";
  evidenceRefs: string[];
  createdAt: Date;
};
~~~

EvaluationEvent 记录“当时 guard 怎么判断”；OutcomeEvent 记录“后来事实证明这个判断对不对”。

没有 OutcomeEvent，系统只能知道 guard 很忙，不能知道 guard 有用。

---

## 6. pi-mono：周期性 Demotion Worker

可以用一个定时 worker 汇总最近 7 天或 30 天的 guard 数据，输出生命周期动作：

~~~ts
type GuardLifecycleDecision = {
  kind:
    | "keep_active"
    | "tighten_scope"
    | "demote_canary"
    | "demote_shadow"
    | "retire_guard"
    | "manual_review";
  reasons: string[];
};

async function runGuardDemotionGate(
  tx: DbTransaction,
  guardId: string,
  windowDays: number,
) {
  const snapshot = await buildGuardEffectivenessSnapshot(tx, {
    guardId,
    windowDays,
  });

  const decision = decideGuardLifecycle(snapshot);

  await tx.guardLifecycleReview.create({
    data: {
      guardId,
      windowDays,
      decision: decision.kind,
      reasons: decision.reasons,
      snapshotHash: hashCanonical(snapshot),
      reviewedAt: new Date(),
    },
  });

  if (decision.kind === "keep_active") {
    return;
  }

  await tx.outbox.create({
    data: {
      type: "guard_lifecycle_action_requested",
      key: `${guardId}:${decision.kind}`,
      payload: {
        guardId,
        action: decision.kind,
        reasons: decision.reasons,
      },
    },
  });
}
~~~

注意这里先写 review，再通过 outbox 请求动作。这样即使 worker 崩了，也能知道为什么系统准备降级某条 guard。

---

## 7. OpenClaw 课程 cron 的落地方式

对 agent-course cron，可以把 closeout guard 的遥测做得很具体：

~~~yaml
guardId: agent-course-closeout-active-v1
telemetryWindow: 30d
signals:
  - name: lesson_file_exists
    outcomeSource: postcheck
  - name: readme_has_lesson_link
    outcomeSource: postcheck
  - name: tools_has_topic
    outcomeSource: postcheck
  - name: telegram_message_id_recorded
    outcomeSource: message.send result
  - name: remote_main_contains_commit
    outcomeSource: git ls-remote
demotionPolicy:
  falsePositiveRateAbove: 0.20
  unknownRateAbove: 0.40
  p95LatencyAboveMs: 1500
  noConfirmedPreventionDays: 30
actions:
  tighten_scope:
    - move remote_main_contains_commit to async reconcile
  demote_shadow:
    - stop blocking final reply
    - keep writing would_block telemetry
  retire_guard:
    - remove from active pack
    - keep lesson closeout as README checklist
~~~

比如 `remote_main_contains_commit` 如果经常因为 GitHub 网络返回 unknown，但最后 git push 实际成功，就应该从 active blocker 移到异步 reconcile。它仍然有价值，但不该卡住整节课。

---

## 8. 常见坑

- **只记录 block，不记录 allow**：没有 allow 基线，就算不出误伤率。
- **没有 outcome 追踪**：无法区分 prevented_failure 和 false_positive。
- **unknown 被当成 safe**：unknown 是证据缺口，不是成功。
- **所有 guard 一套阈值**：支付、发消息、读文件、git push 的误伤容忍度不同。
- **降级不留收据**：后来事故复发时，不知道是哪次 review 把 guard 降了。
- **退役直接删除知识**：退役热路径 guard，不代表删除事故记录；历史 lesson、incident review、regression pack 仍要保留。

---

## 9. 小结

今天的核心：

> Active Regression Guard 必须持续产出 evaluation/outcome 遥测，用 trigger、prevented failure、false positive、unknown 和 latency 数据驱动 keep、tighten、demote、retire。

上一课解决“防复发规则怎么上线”；这一课解决“上线以后怎么证明它仍然值得留在执行路径里”。

成熟 Agent 不只是会把事故变成护栏，还会定期审计护栏本身：有用就保留，太宽就收紧，误伤就降级，过期就退役。
