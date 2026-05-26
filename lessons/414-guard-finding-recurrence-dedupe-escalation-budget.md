# 414. Agent 护栏复发事件的合并、抑制与升级预算（Guard Finding Recurrence Dedupe & Escalation Budget）

上一课讲了 **Guard Finding Recurrence Detection & Regression Replay**：Guard finding 关闭后，要把 close_verified finding 固化成 Recurrence Signature 和 Regression Replay Case，后续新失败先判断 repeat、adjacent、regression 还是 new。

今天继续往后一层：**复发检测命中以后，不要让系统被重复信号打爆，也不要把严重复发当普通 ticket 处理。**

一句话：**Guard recurrence 需要 Recurrence Dedupe Window、Escalation Budget 和 Suppression Receipt；同一个 sourceFinding 的 repeat/regression 在短窗口内合并成一个 Reopen Case，超过预算自动升级 owner / freeze release / require incident command，避免告警风暴，同时保证真正变严重的问题不会被 dedupe 吞掉。**

---

## 1. 为什么复发命中后还需要一层治理

上一课的输出是：

~~~text
sample_failed -> recurrence_detected
matchKind: repeat | adjacent | regression | new
nextAction: reopen | linked_finding | block_release | new_finding
~~~

这还不够。

生产里一个 guard 复发，通常不是只来一条信号：

1. 同一个坏规则在 5 分钟内命中 80 个 audit samples；
2. replay matrix 里 12 个 case 同时失败；
3. 多个 runtime group 都报同一个 regression；
4. canary、shadow、active 三条链路同时生成 reopen；
5. 每个失败都给 owner 发消息，最后 owner 只看到噪音。

如果每条都开 ticket，会造成 ticket storm；如果简单 dedupe，又可能把“风险正在扩大”的信号吞掉。

所以复发治理要同时做两件事：

~~~text
合并重复信号，避免刷屏；
保留强度变化，及时升级。
~~~

这就是 Recurrence Dedupe & Escalation Budget。

---

## 2. 三个核心对象

最小实现只需要三个对象。

~~~text
RecurrenceSignal:
  单条复发信号，来自 audit sample、replay case、runtime drift 或 canary probe。

ReopenCase:
  被合并后的可处理工作流，一个 sourceFinding + matchKind + window 只开一个。

EscalationBudget:
  判断这个复发窗口还能静默合并，还是必须升级。
~~~

一个稳定的 dedupe key 可以这样设计：

~~~text
recurrence:{sourceFindingId}:{matchKind}:{riskSurface}:{windowStart}
~~~

注意：不要只用 sourceFindingId。repeat 和 regression 的处置级别不同，不能互相吞掉。

---

## 3. Escalation Budget 不是通知预算

很多系统把 budget 理解成“最多通知几次”。这太浅。

这里的预算控制的是 **风险扩大程度**：

~~~text
maxSignalsPerWindow
maxRuntimeGroups
maxAffectedTenants
maxFailedReplayCases
maxMinutesOpenBeforeEscalate
releaseFreezeOnRegression
ownerPageOnRepeatCount
~~~

当预算被突破，动作必须升级：

~~~text
repeat 小范围命中 -> reopen 原 finding，合并证据
repeat 超过阈值 -> 升级 owner，缩短 SLO
regression 命中一个 replay case -> block release lane
regression 多 runtime group 命中 -> freeze guard pack，开 incident
adjacent 持续增长 -> 扩 replay matrix，升级 scope owner
~~~

Dedupe 负责少开重复工单；budget 负责别把真事故静音。

---

## 4. learn-claude-code：纯函数合并与升级判定

教学版先写成纯函数。输入是新信号、已有 ReopenCase 和预算，输出是合并、升级还是新建。

~~~py
from dataclasses import dataclass
from typing import Literal


MatchKind = Literal["repeat", "adjacent", "regression"]
Decision = Literal["create_case", "merge_signal", "merge_and_escalate", "freeze_and_incident"]


@dataclass(frozen=True)
class RecurrenceSignal:
    signal_id: str
    source_finding_id: str
    signature_id: str
    match_kind: MatchKind
    risk_surface: str
    runtime_group: str
    tenant_id: str
    replay_case_id: str | None
    occurred_at_minute: int


@dataclass(frozen=True)
class ReopenCase:
    case_id: str
    dedupe_key: str
    source_finding_id: str
    match_kind: MatchKind
    risk_surface: str
    window_start_minute: int
    signal_ids: tuple[str, ...]
    runtime_groups: tuple[str, ...]
    tenant_ids: tuple[str, ...]
    failed_replay_case_ids: tuple[str, ...]
    severity: Literal["P3", "P2", "P1", "P0"]
    status: Literal["open", "escalated", "incident"]


@dataclass(frozen=True)
class EscalationBudget:
    window_minutes: int = 30
    max_signals_per_window: int = 10
    max_runtime_groups: int = 2
    max_affected_tenants: int = 5
    max_failed_replay_cases: int = 3
    release_freeze_on_regression: bool = True


@dataclass(frozen=True)
class RecurrenceDecision:
    decision: Decision
    dedupe_key: str
    severity: str
    reason: str


def window_start(minute: int, size: int) -> int:
    return minute - (minute % size)


def dedupe_key(signal: RecurrenceSignal, budget: EscalationBudget) -> str:
    return ":".join([
        "recurrence",
        signal.source_finding_id,
        signal.match_kind,
        signal.risk_surface,
        str(window_start(signal.occurred_at_minute, budget.window_minutes)),
    ])


def add_unique(values: tuple[str, ...], value: str | None) -> tuple[str, ...]:
    if value is None or value in values:
        return values
    return values + (value,)


def decide_recurrence(signal: RecurrenceSignal, existing: ReopenCase | None, budget: EscalationBudget) -> RecurrenceDecision:
    key = dedupe_key(signal, budget)

    if existing is None:
        if signal.match_kind == "regression" and budget.release_freeze_on_regression:
            return RecurrenceDecision(
                decision="freeze_and_incident",
                dedupe_key=key,
                severity="P1",
                reason="first regression recurrence blocks release lane",
            )
        return RecurrenceDecision(
            decision="create_case",
            dedupe_key=key,
            severity="P2" if signal.match_kind == "repeat" else "P3",
            reason="first recurrence signal in this dedupe window",
        )

    signal_count = len(existing.signal_ids) + 1
    runtime_count = len(add_unique(existing.runtime_groups, signal.runtime_group))
    tenant_count = len(add_unique(existing.tenant_ids, signal.tenant_id))
    replay_count = len(add_unique(existing.failed_replay_case_ids, signal.replay_case_id))

    budget_breached = (
        signal_count > budget.max_signals_per_window
        or runtime_count > budget.max_runtime_groups
        or tenant_count > budget.max_affected_tenants
        or replay_count > budget.max_failed_replay_cases
    )

    if signal.match_kind == "regression" and budget_breached:
        return RecurrenceDecision(
            decision="freeze_and_incident",
            dedupe_key=key,
            severity="P0",
            reason="regression recurrence exceeded blast-radius budget",
        )

    if budget_breached:
        return RecurrenceDecision(
            decision="merge_and_escalate",
            dedupe_key=key,
            severity="P1",
            reason="recurrence signal volume exceeded escalation budget",
        )

    return RecurrenceDecision(
        decision="merge_signal",
        dedupe_key=key,
        severity=existing.severity,
        reason="same recurrence window, evidence merged without opening duplicate case",
    )
~~~

这个函数的关键点是：**dedupe 不是丢弃信号，而是把信号并入同一个 case，并持续重新计算预算。**

---

## 5. pi-mono：RecurrenceDedupeWorker

在 pi-mono 里，建议把它做成事件 worker，而不是塞进 guard checker。

~~~ts
type RecurrenceDetectedEvent = {
  type: "guard_finding.recurrence_detected";
  signalId: string;
  sourceFindingId: string;
  signatureId: string;
  matchKind: "repeat" | "adjacent" | "regression";
  riskSurface: string;
  runtimeGroup: string;
  tenantId: string;
  replayCaseId?: string;
  occurredAt: string;
};

type ReopenCaseUpdatedEvent = {
  type: "guard_finding.reopen_case_updated";
  caseId: string;
  dedupeKey: string;
  decision: "created" | "merged" | "escalated" | "incident_opened";
  signalCount: number;
  runtimeGroupCount: number;
  tenantCount: number;
  failedReplayCaseCount: number;
  severity: "P3" | "P2" | "P1" | "P0";
  suppressionReceiptId: string;
};
~~~

Worker 的流程：

~~~ts
class RecurrenceDedupeWorker {
  constructor(
    private readonly cases: ReopenCaseStore,
    private readonly budgets: EscalationBudgetStore,
    private readonly incidentBus: IncidentBus,
    private readonly events: EventBus,
  ) {}

  async handle(event: RecurrenceDetectedEvent) {
    const budget = await this.budgets.forRiskSurface(event.riskSurface);
    const key = buildDedupeKey(event, budget);

    const existing = await this.cases.findOpenByDedupeKey(key);
    const decision = decideRecurrence(event, existing, budget);

    const updated = await this.cases.upsertWithSignal({
      dedupeKey: key,
      sourceFindingId: event.sourceFindingId,
      matchKind: event.matchKind,
      signalId: event.signalId,
      runtimeGroup: event.runtimeGroup,
      tenantId: event.tenantId,
      replayCaseId: event.replayCaseId,
      severity: decision.severity,
      status: decision.decision === "freeze_and_incident" ? "incident" : "open",
    });

    if (decision.decision === "freeze_and_incident") {
      await this.incidentBus.openOrUpdate({
        dedupeKey: "guard-recurrence:" + event.sourceFindingId + ":" + event.matchKind,
        severity: decision.severity,
        title: "Guard recurrence " + event.matchKind + " exceeded budget",
        evidenceRefs: [updated.caseId, event.signalId],
        freezeReleaseLane: event.matchKind === "regression",
      });
    }

    await this.events.publish({
      type: "guard_finding.reopen_case_updated",
      caseId: updated.caseId,
      dedupeKey: key,
      decision: mapDecision(decision.decision),
      signalCount: updated.signalCount,
      runtimeGroupCount: updated.runtimeGroupCount,
      tenantCount: updated.tenantCount,
      failedReplayCaseCount: updated.failedReplayCaseCount,
      severity: decision.severity,
      suppressionReceiptId: updated.suppressionReceiptId,
    });
  }
}
~~~

这里有个工程细节：upsertWithSignal 必须是原子操作。否则两个 recurrence signal 同时到达，会开出两个 ReopenCase。

生产实现可以用：

~~~text
Postgres unique index: (dedupe_key, status=open)
Redis lock: recurrence:{dedupeKey}
Event idempotency ledger: signalId unique
~~~

---

## 6. Suppression Receipt：被抑制的也要可审计

Dedupe 最危险的地方是“看起来没发生”。

所以每次 merge_signal 都要生成 Suppression Receipt：

~~~json
{
  "receiptId": "suppress_rec_20260527_0330_001",
  "signalId": "sig_8841",
  "dedupeKey": "recurrence:finding_telegram_042:repeat:course_publish:29478240",
  "mergedIntoCaseId": "reopen_case_991",
  "reason": "same source finding, match kind, risk surface, and window",
  "budgetState": {
    "signalCount": 7,
    "runtimeGroupCount": 1,
    "tenantCount": 1,
    "failedReplayCaseCount": 0
  },
  "nextEscalationAt": {
    "signalCount": 11,
    "runtimeGroupCount": 3
  }
}
~~~

这张 receipt 证明两件事：

1. 信号没有被丢；
2. 为什么这次没有开新 case 或升级。

以后复盘时，能回答“为什么当时没有打断发布”。

---

## 7. OpenClaw 课程 Cron 的真实例子

拿我们这个课程 cron 做例子。

假设上一课提到的 recurrence signature 命中：

~~~text
sourceFindingId: finding_telegram_outbox_042
matchKind: repeat
riskSurface: course_publish
问题：messageId 缺失，但 git commit 已推送，memory 写了已发布
~~~

如果 3 分钟内出现这些信号：

~~~text
audit sample: lessons file exists, Telegram messageId missing
git probe: remote main contains commit
memory probe: daily note says 已发布
README probe: index includes lesson
TOOLS probe: 已讲内容 includes topic
~~~

它们不应该开 5 个问题。应该合并成一个：

~~~text
ReopenCase:
  dedupeKey: recurrence:finding_telegram_outbox_042:repeat:course_publish:2026-05-26T17:30Z
  signalCount: 5
  severity: P2
  nextAction: reconcile Telegram / Git / memory / README / TOOLS state
~~~

但如果又发现：

~~~text
runtimeGroup: cron-primary
runtimeGroup: cron-retry-worker
runtimeGroup: manual-resume
matchKind: regression
failedReplayCaseIds: 4
~~~

这就不能继续静默合并。系统应该：

~~~text
freeze agent-course release lane
open incident
page owner
stop future Telegram/Git side effects until guard replay passes
~~~

这就是 budget 的价值：正常重复合并，风险扩大升级。

---

## 8. 实战落地清单

1. recurrence_detected 事件必须带 sourceFindingId、matchKind、riskSurface、runtimeGroup；
2. dedupeKey 至少包含 sourceFindingId、matchKind、riskSurface、windowStart；
3. repeat、adjacent、regression 不要共用一个 dedupe bucket；
4. merge_signal 不是丢弃，要追加 evidenceRefs；
5. 每次抑制都生成 Suppression Receipt；
6. Escalation Budget 同时看 signal、runtime group、tenant、replay case；
7. regression 默认比 repeat 更激进，通常要 block release lane；
8. ReopenCase upsert 要有唯一约束和 signalId 幂等；
9. budget breach 要发事件，不要只改字段；
10. case close 时保留 suppression receipts，方便后续复盘。

---

## 9. 常见坑

**坑一：dedupeKey 太粗。**

只按 sourceFindingId 合并，会把 repeat 和 regression 吞在一起。结果 regression 已经需要 freeze release，系统却还当普通 repeat。

**坑二：dedupeKey 太细。**

把 signalId 放进 dedupeKey，等于没有 dedupe。每条信号都会开新 case。

**坑三：抑制不留 receipt。**

没有 receipt，复盘时没人知道信号是被处理了、被忽略了，还是压根没进系统。

**坑四：budget 只看数量。**

5 条信号来自同一个 runtime 和 5 条信号来自 5 个 runtime，不是一个级别。必须看 blast radius。

**坑五：升级只发通知。**

严重 regression 的升级动作应该包含 release freeze、owner routing、incident command、replay gate，而不是只发一条消息。

---

## 10. 记住这一句

**成熟 Agent 的复发处理，不是每个失败都开一个 ticket，也不是把重复失败静音；而是把同类复发合并进一个可审计 ReopenCase，用预算判断风险有没有扩大，到了阈值就自动升级、冻结发布、进入事故处置。**
