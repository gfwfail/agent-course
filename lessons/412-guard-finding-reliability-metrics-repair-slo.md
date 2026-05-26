# 412. Agent 护栏审计发现的可靠性指标与修复 SLO（Guard Finding Reliability Metrics & Repair SLO）

上一课讲了 **Guard Finding Repair Execution Receipts & Close Gate**：worker 修完补丁以后，必须用 execution receipt、replay、shadow/canary、outcome backfill 和 mitigation 收口来证明 finding 可以关闭。

今天继续往后一层：**单个 finding 能关闭，不代表整个护栏系统健康。**

一句话：**Guard finding 要有可靠性指标和修复 SLO，把 false allow、false block、积压、超时、修复质量和复发风险变成可度量、可升级、可复盘的运营面板；否则团队只会看到“关了多少 ticket”，看不到护栏是不是越来越可靠。**

---

## 1. 为什么 close gate 之后还要看指标

上一课的 close gate 解决的是单个 finding：

~~~text
finding_101 -> repair receipt -> replay/shadow/canary -> close_verified
~~~

这很重要，但它回答不了这些问题：

1. 这周 false allow 是下降了，还是只是没人抽样？
2. critical finding 从发现到缓解用了多久？
3. repair queue 里是否堆了很多低优先级但高 blast radius 的问题？
4. 哪个 guardId 经常出问题，说明规则设计本身有债务？
5. close_verified 后是否经常 reopen，说明 close gate 太松？

成熟 Agent 系统不能只看“当前有没有红灯”。它要看趋势、预算和 SLO burn。

---

## 2. 找对指标：不要只数 ticket

Guard finding 指标至少分五类。

**第一类：发现质量**

~~~text
sampledDecisions
auditFindings
findingRate = auditFindings / sampledDecisions
falseAllowRate
falseBlockRate
unknownOutcomeRate
~~~

如果 sampledDecisions 很少，findingRate 低没有意义。先看样本量，再看比例。

**第二类：响应速度**

~~~text
timeToTriage = triagedAt - detectedAt
timeToMitigate = mitigatedAt - detectedAt
timeToRepair = repairedAt - detectedAt
timeToClose = closedAt - detectedAt
~~~

对 false_allow，timeToMitigate 通常比 timeToClose 更关键。因为漏挡可能已经影响外部世界，必须先 freeze、降级、切 LKG 或收窄 scope。

**第三类：积压风险**

~~~text
openCritical
openHigh
overdueFindings
blockedByOwner
blockedByEvidence
blockedByReleaseGate
guardDebtByRiskSurface
~~~

积压不是数量问题，而是风险面问题。10 个 low explainability gap 不一定比 1 个 critical false_allow 更急。

**第四类：修复质量**

~~~text
closeVerified
extendVerification
reopenAfterClose
adjacentRegressionFound
canaryRollbackCount
mitigationLeftOpen
~~~

如果 reopenAfterClose 高，说明 close gate 或验证矩阵不够硬。

**第五类：护栏热点**

~~~text
findingsByGuardId
findingsByRiskSurface
findingsByActionClass
findingsByRuntimeGroup
findingsByTenant
~~~

这类指标帮助你发现“哪个护栏总在坏”，而不是每次都把它当独立事故。

---

## 3. SLO 不是平均值，是按风险分层的承诺

不要给所有 finding 一个统一 SLA。护栏 finding 的 SLO 应该按 kind 和 severity 分层。

一个保守版本可以这样定：

~~~text
critical false_allow:
  triage <= 15m
  mitigate <= 30m
  repair candidate <= 24h
  close <= 72h

high false_block:
  triage <= 1h
  mitigate <= 4h
  repair candidate <= 48h
  close <= 5d

medium explainability_gap:
  triage <= 1d
  repair candidate <= 7d
  close <= 14d

policy_stale:
  triage <= 2d
  repair candidate <= 14d
  close <= 30d
~~~

这里有个关键点：**mitigate SLO 和 close SLO 要分开。**

很多团队只追 close time，结果为了好看会草率关闭。正确做法是先保证风险被控制，再慢慢完成修复、回放和 canary。

---

## 4. Metric Event：指标从事件流聚合，不从状态表猜

不要每天扫数据库状态表拼指标。状态表会丢掉过程：

- finding 曾经 overdue 后来被改了 deadline；
- finding 先 extend_verification 后 close；
- mitigation 曾经开过 6 小时；
- owner 曾经超时没有 ack；
- canary rollback 发生过但最终又修好了。

可靠做法是把每个关键动作写成事件。

~~~ts
type GuardFindingMetricEvent =
  | {
      type: "finding.detected";
      findingId: string;
      guardId: string;
      kind: "false_allow" | "false_block" | "explainability_gap" | "evidence_overread" | "policy_stale" | "outcome_mismatch";
      severity: "critical" | "high" | "medium" | "low";
      riskSurface: string;
      actionClass: string;
      tenantId?: string;
      detectedAt: string;
    }
  | {
      type: "finding.triaged";
      findingId: string;
      owner: string;
      releaseGate: "replay" | "shadow" | "canary" | "manual_review";
      triagedAt: string;
    }
  | {
      type: "finding.mitigated";
      findingId: string;
      mitigationKind: "freeze_surface" | "lkg_rollback" | "scope_narrow" | "sampling_boost";
      mitigatedAt: string;
    }
  | {
      type: "finding.close_decided";
      findingId: string;
      decision: "close_verified" | "extend_verification" | "reopen_finding" | "escalate_owner" | "manual_review";
      decidedAt: string;
    };
~~~

状态表回答“现在是什么”。事件流回答“怎么变成现在这样”。

---

## 5. learn-claude-code：纯函数计算 SLO 状态

教学版先不要接数据库。用纯函数把 finding 事件聚合成 SLO 状态。

~~~py
from dataclasses import dataclass
from datetime import datetime, timedelta, timezone
from typing import Literal


Kind = Literal[
    "false_allow",
    "false_block",
    "explainability_gap",
    "evidence_overread",
    "policy_stale",
    "outcome_mismatch",
]
Severity = Literal["critical", "high", "medium", "low"]
SloState = Literal["healthy", "burning", "breached", "missing_data", "closed"]


@dataclass(frozen=True)
class FindingSnapshot:
    finding_id: str
    kind: Kind
    severity: Severity
    detected_at: datetime
    triaged_at: datetime | None
    mitigated_at: datetime | None
    repaired_at: datetime | None
    closed_at: datetime | None
    reopened_after_close: bool


@dataclass(frozen=True)
class SloPolicy:
    triage_by: timedelta
    mitigate_by: timedelta | None
    repair_by: timedelta
    close_by: timedelta


def policy_for(finding: FindingSnapshot) -> SloPolicy:
    if finding.severity == "critical" and finding.kind == "false_allow":
        return SloPolicy(
            triage_by=timedelta(minutes=15),
            mitigate_by=timedelta(minutes=30),
            repair_by=timedelta(hours=24),
            close_by=timedelta(hours=72),
        )

    if finding.severity in ("critical", "high"):
        return SloPolicy(
            triage_by=timedelta(hours=1),
            mitigate_by=timedelta(hours=4),
            repair_by=timedelta(hours=48),
            close_by=timedelta(days=5),
        )

    return SloPolicy(
        triage_by=timedelta(days=1),
        mitigate_by=None,
        repair_by=timedelta(days=7),
        close_by=timedelta(days=14),
    )


def deadline_state(now: datetime, start: datetime, deadline: timedelta, done_at: datetime | None) -> SloState:
    if done_at is not None:
        return "closed"

    elapsed = now - start
    if elapsed > deadline:
        return "breached"

    if elapsed > deadline * 0.8:
        return "burning"

    return "healthy"


def finding_slo_state(finding: FindingSnapshot, now: datetime) -> dict[str, SloState]:
    if finding.reopened_after_close:
        return {
            "triage": "breached",
            "mitigation": "breached",
            "repair": "breached",
            "close": "breached",
        }

    policy = policy_for(finding)
    mitigation_state: SloState = "closed"
    if policy.mitigate_by is not None:
        mitigation_state = deadline_state(
            now,
            finding.detected_at,
            policy.mitigate_by,
            finding.mitigated_at,
        )

    return {
        "triage": deadline_state(now, finding.detected_at, policy.triage_by, finding.triaged_at),
        "mitigation": mitigation_state,
        "repair": deadline_state(now, finding.detected_at, policy.repair_by, finding.repaired_at),
        "close": deadline_state(now, finding.detected_at, policy.close_by, finding.closed_at),
    }


now = datetime(2026, 5, 26, 11, 30, tzinfo=timezone.utc)
finding = FindingSnapshot(
    finding_id="finding_412",
    kind="false_allow",
    severity="critical",
    detected_at=datetime(2026, 5, 26, 11, 5, tzinfo=timezone.utc),
    triaged_at=datetime(2026, 5, 26, 11, 10, tzinfo=timezone.utc),
    mitigated_at=None,
    repaired_at=None,
    closed_at=None,
    reopened_after_close=False,
)

print(finding_slo_state(finding, now))
~~~

输出会显示 mitigation 已经接近或超过 SLO。这个信号应该触发升级，而不是等到 close deadline。

---

## 6. pi-mono：用 EventBus 聚合 Guard Reliability Snapshot

生产版可以把 finding 事件流聚合成一个只读 snapshot，给 dashboard、cron、release gate 使用。

~~~ts
type GuardReliabilitySnapshot = {
  windowStart: string;
  windowEnd: string;
  sampledDecisions: number;
  findings: {
    total: number;
    byKind: Record<string, number>;
    bySeverity: Record<string, number>;
    byRiskSurface: Record<string, number>;
  };
  slo: {
    openCritical: number;
    burning: number;
    breached: number;
    blockedByOwner: number;
    blockedByEvidence: number;
  };
  quality: {
    closeVerified: number;
    extendVerification: number;
    reopenAfterClose: number;
    canaryRollback: number;
    mitigationLeftOpen: number;
  };
  hotGuards: Array<{
    guardId: string;
    findings: number;
    falseAllows: number;
    falseBlocks: number;
    reopenAfterClose: number;
  }>;
};

class GuardFindingMetricsProjector {
  constructor(
    private readonly events: EventStore,
    private readonly snapshots: SnapshotStore,
    private readonly slo: GuardFindingSloPolicy,
  ) {}

  async project(windowStart: Date, windowEnd: Date): Promise<GuardReliabilitySnapshot> {
    const events = await this.events.find({
      topicPrefix: "guard.finding.",
      from: windowStart,
      to: windowEnd,
    });

    const accumulator = new GuardReliabilityAccumulator(this.slo);

    for (const event of events) {
      accumulator.apply(event);
    }

    const snapshot = accumulator.snapshot(windowStart, windowEnd);

    await this.snapshots.put({
      key: `guard-reliability:${windowStart.toISOString()}:${windowEnd.toISOString()}`,
      value: snapshot,
      createdAt: new Date(),
    });

    return snapshot;
  }
}
~~~

注意这里叫 projector，不叫 monitor。它只负责把事件流投影成事实视图，不直接发告警、不直接改状态。告警和 release gate 是下一层消费者。

---

## 7. SLO Burn 触发什么动作

指标没有动作，就是漂亮日志。Guard finding SLO 至少要驱动四类动作。

**第一，升级 owner。**

~~~text
critical false_allow mitigate SLO burning -> page guard owner
critical false_allow mitigate SLO breached -> page owner + freeze risk surface
~~~

**第二，阻断发布。**

~~~text
openCritical > 0 -> block guard pack promotion
breached high finding > 0 -> require manual release approval
reopenAfterCloseRate > threshold -> harden close gate before next promotion
~~~

**第三，调整抽样。**

~~~text
unknownOutcomeRate high -> increase outcome backfill sampling
falseAllowRate rising -> increase audit sample on affected risk surface
falseBlockRate rising -> run counterfactual review on blocked good actions
~~~

**第四，生成修复债务。**

~~~text
same guardId >= 3 findings in 7d -> create guard design debt ticket
same riskSurface repeated falseAllow -> require policy redesign review
mitigationLeftOpen > 0 after close -> create mitigation cleanup task
~~~

成熟系统不是“报警更多”，而是每个报警都有下一步。

---

## 8. OpenClaw 实战：课程 cron 也需要这样的指标

拿 Agent 课程 cron 举例，它有三个真实副作用：

1. 发 Telegram；
2. 写 lessons/README/TOOLS；
3. git commit/push。

如果给它加 guard finding 指标，可以这样设计：

~~~json
{
  "riskSurface": "agent_course_cron",
  "guards": [
    "topic.dedup",
    "telegram.delivery_once",
    "git.remote_contains_commit",
    "tools.memory_updated"
  ],
  "slo": {
    "false_allow_critical": {
      "triage": "15m",
      "mitigate": "30m",
      "close": "72h"
    }
  }
}
~~~

如果某次课程重复发送，属于 telegram.delivery_once 的 false_allow。

正确处理不是只删一条消息，而是：

1. detected：记录 finding；
2. triaged：确认 guardId 和 root cause；
3. mitigated：冻结重复发送路径或开启 delivery ledger；
4. repaired：补 outbox/idempotency/reconcile；
5. close_decided：经过 replay、dry-run 和远端 messageId 对账；
6. metrics：统计 delivery_once 这个 guard 最近 7 天是否还有类似 finding。

这样下一次 cron 运行前，release gate 可以问：

~~~text
agent_course_cron 是否存在 open critical finding？
delivery_once 是否在过去 7 天 reopenAfterClose？
git.remote_contains_commit 是否有 breached SLO？
~~~

有任何硬阻断，就不要继续外发，先进入恢复路径。

---

## 9. 常见坑

**坑 1：只看 close 数量。**

close 数量上升可能是好事，也可能是 close gate 变松。要配合 reopenAfterClose、extendVerification 和 canaryRollback 看。

**坑 2：把 false allow 和 false block 混成一个错误率。**

false allow 是漏挡，可能造成外部损害；false block 是误挡，可能影响体验和效率。它们的 SLO、owner 和缓解动作不同。

**坑 3：平均修复时间掩盖 critical。**

10 个 low 很快关闭，会把平均值拉漂亮，但 1 个 critical false_allow 超时仍然是严重事故。SLO 要按 severity 切片。

**坑 4：指标从当前状态反推。**

状态表会丢掉曾经发生过的 extend、rollback、owner timeout。指标必须从事件流聚合。

**坑 5：指标不反向影响发布。**

如果 openCritical 不阻断 guard pack promotion，dashboard 就只是装饰。可靠性指标必须进入 release gate。

---

## 10. 落地清单

给 Guard Finding 加可靠性指标，可以按 8 步做：

1. 为 finding.detected / triaged / mitigated / repaired / close_decided 写事件；
2. 按 kind + severity 定义 triage、mitigate、repair、close SLO；
3. 把 mitigate SLO 和 close SLO 分开；
4. 用 projector 从事件流生成 GuardReliabilitySnapshot；
5. dashboard 展示 sampledDecisions、findingRate、openCritical、breached、hotGuards；
6. release gate 读取 snapshot，open critical 或 SLO breach 时阻断 promotion；
7. 同 guardId 多次 finding 自动生成 design debt ticket；
8. reopenAfterClose 高于阈值时，先强化 close gate，再继续放量。

---

## 11. 总结

这一课的核心：

1. close gate 证明单个 finding 可以关，reliability metrics 证明整套护栏是否变健康；
2. 指标要覆盖发现质量、响应速度、积压风险、修复质量和护栏热点；
3. SLO 要按 kind/severity 分层，mitigate 和 close 必须分开；
4. 指标要从事件流聚合，不要从当前状态表猜过程；
5. learn-claude-code 可以用纯函数训练 SLO 判断；
6. pi-mono 可以用 EventBus projector 生成 GuardReliabilitySnapshot；
7. OpenClaw 的 Telegram/Git/memory 课程 cron 也需要这种指标，因为它有真实外部副作用。

成熟 Agent 不是“出了 finding 能修”，而是能持续回答：哪些护栏最常失效、风险控制有没有超时、修复质量是否变好，以及下一次发布是否应该被这些事实挡住。
