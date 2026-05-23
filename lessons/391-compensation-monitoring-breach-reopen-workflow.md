# 391. Agent 补偿监控租约违约后的重开工作流（Compensation Monitoring Breach & Reopen Workflow）

上一课讲到：补偿关闭以后，残余风险不能靠一句“后续观察”糊过去，要写成 Residual Risk Register 和 Monitoring Lease。

今天继续往下走一步：lease 真的 breach 了怎么办？

很多系统只做到告警：

~~~text
lease breached -> send alert
~~~

这不够。成熟 Agent 应该做到：

~~~text
lease breached
  -> classify breach
  -> freeze risky downstream
  -> create reopen case
  -> attach evidence
  -> route to reconcile / compensation / incident
  -> prove it was reopened exactly once
~~~

监控租约不是提醒器，而是恢复系统的入口。

## 1. 为什么 breach 不能只发告警

补偿 closeout 之后，我们通常已经释放了下游屏障：

- 任务状态从 recovering 变成 closed；
- 用户或系统被告知“已修复”；
- 临时冻结被解除；
- 证据被压缩进冷存储；
- 只留下有限时间的 residual risk watch。

这时候如果 lease breach 只是发一条消息，很容易出现三类问题。

第一，没人接手。群里看到“指标异常”，但没有结构化 case、owner、下一步动作，最后变成口头提醒。

第二，重复重开。cron 每 5 分钟扫一次，连续三次命中同一个信号，就创建三个 incident、三条补偿任务、三次通知。

第三，错误路线。有些 breach 需要 reopen reconcile，有些要 reopen compensation，有些只是 manual review。全都扔给同一个告警渠道，会让恢复流程变慢。

所以 breach handler 的核心不是“发通知”，而是生成一个可执行的 Reopen Case。

## 2. Reopen Case 的最小结构

一个 Reopen Case 至少要绑定五类信息：

~~~json
{
  "caseId": "reopen_2026_05_24_course_391",
  "sourceLeaseId": "lease_risk_course_publish_consistency",
  "riskId": "risk_course_publish_partial_failure",
  "breachKind": "evidence_mismatch",
  "severity": "medium",
  "dedupeKey": "reopen:risk_course_publish_partial_failure:evidence_mismatch:2026-05-24",
  "observedSignals": [
    {
      "name": "git_remote_missing_commit",
      "status": "breach",
      "observedAt": "2026-05-24T20:30:00Z",
      "evidenceRef": "evidence://git/ls-remote/main"
    }
  ],
  "route": "reconcile",
  "blockedDownstream": ["publish_next_lesson"],
  "status": "opened"
}
~~~

关键字段：

- sourceLeaseId：从哪条监控租约触发；
- breachKind：不是所有 breach 都一样；
- dedupeKey：保证同一个风险窗口只重开一次；
- observedSignals：不能只有“异常了”，要带证据引用；
- route：进入 reconcile、compensation、incident 还是 manual_review；
- blockedDownstream：重开期间哪些动作必须暂停。

一句话：Reopen Case 是 breach 从“观察信号”升级为“可处理工作”的边界对象。

## 3. learn-claude-code：教学版纯函数

在 learn-claude-code 里，可以先用一个纯函数把 breach signals 转成 reopen decision。

~~~python
from dataclasses import dataclass
from typing import Literal

BreachKind = Literal[
    "signal_noise",
    "evidence_mismatch",
    "remote_state_regression",
    "compensation_regression",
    "policy_violation",
]

Route = Literal["ignore", "manual_review", "reconcile", "compensation", "incident"]

@dataclass(frozen=True)
class WatchSignal:
    name: str
    status: Literal["ok", "warn", "breach"]
    evidence_ref: str

@dataclass(frozen=True)
class MonitoringLease:
    lease_id: str
    risk_id: str
    on_breach: Route
    expires_at: str

@dataclass(frozen=True)
class ReopenDecision:
    should_reopen: bool
    breach_kind: BreachKind
    route: Route
    dedupe_key: str
    blocked_downstream: list[str]
    reason: str

def classify_breach(signals: list[WatchSignal]) -> BreachKind:
    names = {signal.name for signal in signals if signal.status == "breach"}

    if "policy_guard_failed" in names:
        return "policy_violation"
    if "remote_state_regressed" in names:
        return "remote_state_regression"
    if "compensation_post_check_failed" in names:
        return "compensation_regression"
    if "evidence_hash_mismatch" in names:
        return "evidence_mismatch"

    return "signal_noise"

def decide_reopen(
    lease: MonitoringLease,
    signals: list[WatchSignal],
    window_start: str,
) -> ReopenDecision:
    breached = [signal for signal in signals if signal.status == "breach"]

    if not breached:
        return ReopenDecision(
            should_reopen=False,
            breach_kind="signal_noise",
            route="ignore",
            dedupe_key=f"noop:{lease.lease_id}:{window_start}",
            blocked_downstream=[],
            reason="no breach signals",
        )

    breach_kind = classify_breach(breached)
    route_by_kind: dict[BreachKind, Route] = {
        "signal_noise": "manual_review",
        "evidence_mismatch": "reconcile",
        "remote_state_regression": "reconcile",
        "compensation_regression": "compensation",
        "policy_violation": "incident",
    }

    route = route_by_kind[breach_kind]
    dedupe_key = f"reopen:{lease.risk_id}:{breach_kind}:{window_start}"
    blocked = ["release_downstream", "archive_evidence"]

    if route in ("compensation", "incident"):
        blocked.append("publish_success_notice")

    return ReopenDecision(
        should_reopen=True,
        breach_kind=breach_kind,
        route=route,
        dedupe_key=dedupe_key,
        blocked_downstream=blocked,
        reason=f"{len(breached)} breach signal(s) matched {breach_kind}",
    )
~~~

这里刻意不直接调用外部系统。教学版先把判断逻辑变成可测试的纯函数：

- 输入：lease + signals；
- 输出：是否重开、重开路线、幂等键、阻断范围；
- 没有副作用，方便单元测试。

真正执行 reopen，可以由外层 runner 根据 ReopenDecision 去做。

## 4. pi-mono：BreachReopenWorker

在 pi-mono 里，比较自然的实现是一个事件驱动 worker：

~~~ts
type BreachKind =
  | "signal_noise"
  | "evidence_mismatch"
  | "remote_state_regression"
  | "compensation_regression"
  | "policy_violation";

type ReopenRoute = "manual_review" | "reconcile" | "compensation" | "incident";

interface MonitoringLeaseBreachedEvent {
  leaseId: string;
  riskId: string;
  signals: Array<{
    name: string;
    status: "breach";
    evidenceRef: string;
    observedAt: string;
  }>;
  windowStart: string;
}

interface ReopenCase {
  caseId: string;
  sourceLeaseId: string;
  riskId: string;
  breachKind: BreachKind;
  route: ReopenRoute;
  dedupeKey: string;
  evidenceRefs: string[];
  blockedDownstream: string[];
  status: "opened" | "already_open" | "routed";
}

class BreachReopenWorker {
  constructor(
    private readonly cases: ReopenCaseStore,
    private readonly blocker: DownstreamBlocker,
    private readonly router: ReopenRouter,
    private readonly audit: AuditLog,
  ) {}

  async handle(event: MonitoringLeaseBreachedEvent): Promise<ReopenCase> {
    const breachKind = this.classify(event.signals);
    const route = this.routeFor(breachKind);
    const dedupeKey = [
      "reopen",
      event.riskId,
      breachKind,
      event.windowStart,
    ].join(":");

    const existing = await this.cases.findByDedupeKey(dedupeKey);
    if (existing) {
      await this.audit.write({
        type: "reopen_case_deduped",
        dedupeKey,
        existingCaseId: existing.caseId,
      });
      return { ...existing, status: "already_open" };
    }

    const reopenCase = await this.cases.create({
      caseId: crypto.randomUUID(),
      sourceLeaseId: event.leaseId,
      riskId: event.riskId,
      breachKind,
      route,
      dedupeKey,
      evidenceRefs: event.signals.map((signal) => signal.evidenceRef),
      blockedDownstream: this.blockedDownstream(route),
      status: "opened",
    });

    await this.blocker.block(reopenCase.blockedDownstream, {
      reason: "monitoring_lease_breached",
      caseId: reopenCase.caseId,
    });

    await this.router.route(reopenCase);

    await this.audit.write({
      type: "reopen_case_routed",
      caseId: reopenCase.caseId,
      route,
      breachKind,
      evidenceRefs: reopenCase.evidenceRefs,
    });

    return { ...reopenCase, status: "routed" };
  }

  private classify(signals: MonitoringLeaseBreachedEvent["signals"]): BreachKind {
    const names = new Set(signals.map((signal) => signal.name));

    if (names.has("policy_guard_failed")) return "policy_violation";
    if (names.has("compensation_post_check_failed")) return "compensation_regression";
    if (names.has("remote_state_regressed")) return "remote_state_regression";
    if (names.has("evidence_hash_mismatch")) return "evidence_mismatch";

    return "signal_noise";
  }

  private routeFor(kind: BreachKind): ReopenRoute {
    return {
      signal_noise: "manual_review",
      evidence_mismatch: "reconcile",
      remote_state_regression: "reconcile",
      compensation_regression: "compensation",
      policy_violation: "incident",
    }[kind];
  }

  private blockedDownstream(route: ReopenRoute): string[] {
    const base = ["archive_case", "release_barrier"];
    return route === "incident"
      ? [...base, "send_success_notice", "run_next_side_effect"]
      : base;
  }
}
~~~

这段代码有几个生产细节：

- dedupeKey 先查再创建，防止重复 reopen；
- 创建 case 后先 block downstream，再 route；
- route 之后写 audit event，方便后续 closeout；
- route 不等于处理完成，只是把 case 放进正确恢复队列。

## 5. OpenClaw 课程 Cron 的实战映射

拿这个课程 Cron 举例，一次完整发布有多个外部事实：

- Telegram 群消息已发送；
- lesson 文件已创建；
- README 目录已更新；
- TOOLS 已讲内容已更新；
- git commit 已推送；
- memory 日志记录了 messageId 和 commit。

假设 closeout 已经通过，但监控租约在下一轮发现：

~~~yaml
leaseId: lease_agent_course_publish_391
riskId: risk_course_publish_partial_failure
watchSignals:
  - name: git_remote_missing_commit
    status: breach
    evidenceRef: evidence://git/ls-remote/main
  - name: telegram_message_present
    status: ok
    evidenceRef: evidence://telegram/message/12633
onBreach: reopen_reconcile
~~~

这时不能只发一句“push 好像失败了”。正确动作是：

1. 创建 Reopen Case；
2. 用 dedupeKey 防止重复创建；
3. 暂停下一课发布或至少阻断依赖“391 已完整发布”的动作；
4. 路由到 reconcile worker；
5. 重新检查本地 commit、远端 main、README、TOOLS、Telegram messageId；
6. 如果只是本地缺回执，补账；如果远端真的缺 commit，重新 push 或创建 forward-fix；
7. 最后再跑 closeout gate。

结构大概是：

~~~json
{
  "caseId": "reopen_course_391_remote_missing_commit",
  "route": "reconcile",
  "dedupeKey": "reopen:risk_course_publish_partial_failure:remote_state_regression:2026-05-24T20:30Z",
  "blockedDownstream": ["publish_lesson_392"],
  "requiredChecks": [
    "git log contains lesson 391 commit",
    "git ls-remote origin main contains commit",
    "README contains lesson 391",
    "TOOLS contains lesson 391 topic",
    "Telegram messageId exists or release ledger has failure"
  ]
}
~~~

这样第 392 课不会建立在一个“第 391 课也许发完了”的模糊状态上。

## 6. Breach 路由表

生产系统里建议把 breach kind 和 route 明确写成表，而不是散落在 if/else 里：

| breachKind | 典型信号 | route | 下游策略 |
|---|---|---|---|
| signal_noise | 单个低可信指标抖动 | manual_review | 不阻断或弱阻断 |
| evidence_mismatch | hash 不一致、证据缺口 | reconcile | 阻断 close/archive |
| remote_state_regression | 远端状态倒退、外部资源消失 | reconcile | 阻断依赖该事实的动作 |
| compensation_regression | 补偿后检查再次失败 | compensation | 重开补偿 runbook |
| policy_violation | 权限、隐私、安全策略失败 | incident | 冻结高风险副作用 |

表驱动的好处是：

- 审计时能解释为什么这样路由；
- 新增信号不需要到处改控制流；
- 可以给不同 tenant、channel、risk tier 配不同策略；
- 测试可以覆盖每个 breach kind。

## 7. 常见坑

第一，breach 没有证据引用。

没有 evidenceRef 的 breach 只能算日志噪音。重开 case 至少要能证明“我为什么重开”。

第二，dedupeKey 太粗。

如果 dedupeKey 只有 riskId，那么不同类型 breach 会互相吞掉。建议至少包含 riskId、breachKind、watch window。

第三，重开后不阻断下游。

这会导致系统一边修复一边继续释放依赖旧状态的动作。reopen case 创建后，相关 downstream barrier 要立即回到 blocked。

第四，把 manual_review 当失败。

有些信号可信度不够，应该进入人工复核，而不是自动补偿。自动化的价值不是一律自动执行，而是把证据和决策边界准备好。

第五，reopen 没有 closeout。

重开 case 不是终点。它后面还要走 reconcile/compensation/incident 的 closeout gate，否则系统会长期处于半开状态。

## 8. 本课小结

Monitoring Lease 的价值不在于“发现异常”，而在于异常发生时能稳定进入恢复流程。

一个靠谱的 breach reopen workflow 要做到：

- breach 分类，而不是统一告警；
- dedupeKey 幂等重开；
- Reopen Case 绑定 lease、risk、signals、route 和 evidence；
- 创建 case 后立即阻断相关下游；
- 按 route 进入 reconcile、compensation、incident 或 manual_review；
- 最后仍然要用 closeout gate 收口。

> Closeout 证明补偿完成；Residual Risk Register 记录仍需观察的风险；Monitoring Lease 负责发现复发；Breach Reopen Workflow 则保证复发后不是“喊一嗓子”，而是回到可审计、可幂等、可关闭的恢复系统。
