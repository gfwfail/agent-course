# 445. Agent 护栏长期化接管与 Sunset Review（Guard Pattern Long-term Guard Supersession & Sunset Review）

> 关键词：long-term guard、supersession、sunset review、residual constraint、operational debt

上一课讲 PostReleaseMonitoringLease 到期后不要自动关闭，要通过 Renewal Gate 判断是续期、归档、重开 family lock，还是把残余约束升级成长周期护栏。

今天补这个分支：**当监控租约决定 supersede_by_long_term_guard 时，系统到底应该怎么接管？**

很多团队会把“临时监控”变成“永久 if 判断”：

- incident 结束了，但某个 residual constraint 还没完全消失；
- 工程师顺手在生产路径加一个 guard；
- 半年后没人知道它为什么存在，也没人敢删；
- 规则越来越多，false block 越来越高，Agent 行为越来越保守。

成熟 Agent 的做法不是把残余约束塞进热路径就完事，而是生成一个 LongTermGuardContract：

> 临时监控退出热路径前，必须把仍需保留的约束转成有 owner、有范围、有指标、有复查时间、有下线条件的长期护栏。

---

## 1. 为什么长期护栏必须有合同

PostReleaseMonitoringLease 的职责是事故后的短中期观察。它不是永久策略系统。

如果 residual constraint 一直没有关闭，通常说明三件事之一：

1. 这个风险确实长期存在，需要产品级或平台级护栏；
2. 证据不足，团队不敢解除；
3. 旧事故记忆变成了心理负担，规则已经失去现实意义。

三种情况不能混在一起。

长期护栏合同至少要回答：

- **risk**：它防的具体风险是什么；
- **scope**：它只管哪些 tenant、channel、tool、actionClass；
- **owner**：谁负责维护和解释；
- **metrics**：怎么证明它还有效；
- **sunset**：什么时候复查、什么条件下可以删除或降级；
- **fallback**：它误伤时怎么处理；
- **lineage**：它从哪个 incident、lease、receipt 接管而来。

没有这些字段，长期护栏会变成“谁也不敢删的祖传代码”。

---

## 2. 状态机：supersede 不是 archive

监控租约被长期护栏接管时，建议拆成两个对象：

~~~text
PostReleaseMonitoringLease
  observing
  -> renewal_due
  -> supersession_ready
  -> superseded

LongTermGuardContract
  proposed
  -> active
  -> sunset_due
  -> renewed
  -> downgraded
  -> retired
  -> breached
~~~

重点：

- superseded 不等于 archived，它说明 lease 退出热路径，但责任转移到长期 guard；
- LongTermGuardContract active 后，要进入正常 policy / guard 发布系统，而不是继续挂在 incident worker 上；
- sunset_due 是复查点，不是自动删除；
- retired 必须有 SunsetReceipt，证明下线条件满足；
- breached 说明长期护栏触发了新的事故或修复流程。

这样做的收益是边界清楚：

- incident lease 负责事故恢复后的有限观察；
- long-term guard 负责长期风险控制；
- sunset review 负责防止长期规则失控。

---

## 3. learn-claude-code：长期护栏接管闸门

教学版先写纯函数：输入 lease、residual constraints 和观测摘要，输出能否生成 LongTermGuardContract。

~~~python
# learn_claude_code/guard_patterns/long_term_guard_supersession.py
from dataclasses import dataclass
from typing import Literal

Decision = Literal[
    "create_long_term_guard",
    "extend_monitoring",
    "manual_review",
    "reopen_family_lock",
]


@dataclass(frozen=True)
class ResidualConstraint:
    constraint_id: str
    risk_surface: str
    severity: int
    owner: str | None
    false_block_rate: float
    false_allow_rate: float
    can_be_scoped: bool
    proposed_scope: tuple[str, ...]
    sunset_after_days: int


@dataclass(frozen=True)
class SupersessionInput:
    lease_id: str
    family_id: str
    clean_days: int
    recurrence_hits: int
    dependency_revalidated: bool
    residual_constraints: tuple[ResidualConstraint, ...]


@dataclass(frozen=True)
class LongTermGuardSpec:
    guard_id: str
    lineage_lease_id: str
    family_id: str
    risk_surfaces: tuple[str, ...]
    scope: tuple[str, ...]
    owner: str
    review_after_days: int
    max_false_block_rate: float
    max_false_allow_rate: float


@dataclass(frozen=True)
class SupersessionDecision:
    decision: Decision
    reason: str
    guard_spec: LongTermGuardSpec | None = None
    followups: tuple[str, ...] = ()


def decide_long_term_guard_supersession(
    data: SupersessionInput,
) -> SupersessionDecision:
    if data.recurrence_hits > 0:
        return SupersessionDecision("reopen_family_lock", "recurrence_during_supersession")

    if not data.dependency_revalidated:
        return SupersessionDecision(
            "extend_monitoring",
            "dependency_not_revalidated",
            followups=("run_dependency_probe",),
        )

    if data.clean_days < 14:
        return SupersessionDecision("extend_monitoring", "needs_minimum_clean_window")

    if not data.residual_constraints:
        return SupersessionDecision("manual_review", "no_residual_constraints_to_supersede")

    owners = {c.owner for c in data.residual_constraints if c.owner}
    if len(owners) != 1:
        return SupersessionDecision(
            "manual_review",
            "missing_or_ambiguous_owner",
            followups=tuple(c.constraint_id for c in data.residual_constraints if not c.owner),
        )

    unscoped = [c.constraint_id for c in data.residual_constraints if not c.can_be_scoped]
    if unscoped:
        return SupersessionDecision(
            "manual_review",
            "residual_constraint_not_scoped",
            followups=tuple(unscoped),
        )

    severe = [c for c in data.residual_constraints if c.severity >= 8]
    review_after_days = min(c.sunset_after_days for c in data.residual_constraints)

    spec = LongTermGuardSpec(
        guard_id=f"ltg.{data.family_id}.{data.lease_id}",
        lineage_lease_id=data.lease_id,
        family_id=data.family_id,
        risk_surfaces=tuple(sorted({c.risk_surface for c in data.residual_constraints})),
        scope=tuple(sorted({s for c in data.residual_constraints for s in c.proposed_scope})),
        owner=next(iter(owners)),
        review_after_days=min(review_after_days, 30 if severe else 90),
        max_false_block_rate=max(c.false_block_rate for c in data.residual_constraints),
        max_false_allow_rate=max(c.false_allow_rate for c in data.residual_constraints),
    )
    return SupersessionDecision("create_long_term_guard", "eligible_for_supersession", spec)
~~~

注意这里没有让函数直接“关闭 lease”。它只生成长期护栏 spec。真正切换要靠事务 worker 写收据。

---

## 4. LongTermGuardContract 应该长什么样

建议合同最少包含这些字段：

~~~json
{
  "guardId": "ltg.guard.family.restore.release.postrel_443",
  "lineage": {
    "incidentId": "inc.guard.restore.2026-05",
    "leaseId": "postrel_443",
    "finalReceiptId": "archive_or_supersession_receipt_445"
  },
  "scope": {
    "channels": ["telegram"],
    "tools": ["message.send", "git.push"],
    "actionClasses": ["external_side_effect", "repo_publish"]
  },
  "policy": {
    "mode": "warn_or_block",
    "maxFalseBlockRate": 0.01,
    "maxFalseAllowRate": 0.001,
    "onBreach": "open_guard_repair_case"
  },
  "owner": {
    "team": "course_cron",
    "reviewer": "agent_runtime_owner"
  },
  "sunset": {
    "reviewAfterDays": 30,
    "retireWhen": [
      "zero_high_severity_hits_for_30_days",
      "dependency_fingerprint_revalidated",
      "replacement_guard_active_or_risk_removed"
    ],
    "maxRenewals": 3
  }
}
~~~

关键不是 JSON 本身，而是它把“为什么存在、管哪里、谁负责、何时复查、怎么下线”写进可执行数据。

---

## 5. pi-mono：Supersession Worker

生产版可以用事件流串起来：

1. RenewalWorker 输出 supersede_by_long_term_guard；
2. SupersessionProjector 聚合 residual constraints；
3. Gate 生成 LongTermGuardSpec；
4. Worker 在事务里写 LongTermGuardContract + SupersessionReceipt；
5. Outbox 发布 long_term_guard.activated；
6. Lease 状态从 supersession_ready 变成 superseded。

示例：

~~~ts
// packages/runtime/src/guard/LongTermGuardSupersessionWorker.ts
type SupersessionDecision =
  | { type: "extend_monitoring"; reason: string; followups: string[] }
  | { type: "manual_review"; reason: string; followups: string[] }
  | { type: "reopen_family_lock"; reason: string }
  | { type: "create_long_term_guard"; reason: string; spec: LongTermGuardSpec };

type LongTermGuardSpec = {
  guardId: string;
  familyId: string;
  lineageLeaseId: string;
  scope: {
    channels: string[];
    tools: string[];
    actionClasses: string[];
  };
  owner: string;
  reviewAfter: Date;
  maxRenewals: number;
  metrics: {
    maxFalseBlockRate: number;
    maxFalseAllowRate: number;
  };
};

export async function handleSupersessionDue(
  leaseId: string,
  deps: {
    store: GuardStore;
    projector: SupersessionProjector;
    gate: SupersessionGate;
    outbox: Outbox;
    now: () => Date;
  },
) {
  await deps.store.transaction(async (tx) => {
    const lease = await tx.lockMonitoringLease(leaseId);
    if (lease.status !== "supersession_ready") {
      return;
    }

    const summary = await deps.projector.buildSummary(tx, lease);
    const decision = deps.gate.decide(lease, summary, deps.now());

    if (decision.type === "create_long_term_guard") {
      const contract = await tx.insertLongTermGuardContract({
        ...decision.spec,
        status: "active",
        createdAt: deps.now(),
      });

      const receipt = await tx.insertSupersessionReceipt({
        leaseId,
        guardId: contract.guardId,
        reason: decision.reason,
        residualConstraintIds: summary.residualConstraintIds,
        createdAt: deps.now(),
      });

      await tx.markLeaseSuperseded(leaseId, receipt.receiptId);
      await deps.outbox.publish(tx, {
        type: "long_term_guard.activated",
        guardId: contract.guardId,
        receiptId: receipt.receiptId,
      });
      return;
    }

    await tx.recordSupersessionDecision({
      leaseId,
      decision,
      decidedAt: deps.now(),
    });
  });
}
~~~

这里最重要的是 lockMonitoringLease。同一个 lease 不能同时被 archive worker 和 supersession worker 接管，否则会出现两个系统都以为自己负责。

---

## 6. Sunset Review：长期护栏也不能永久有效

LongTermGuardContract active 后，必须有 Sunset Review。

Sunset Review 的输入：

- 最近窗口命中次数；
- false block / false allow；
- 被替代规则是否已经 active；
- 依赖 fingerprint 是否变化；
- owner 是否仍有效；
- 合同已经续期几次；
- 是否仍有真实风险样本支持。

一个简单决策表：

| 信号 | 决策 |
| --- | --- |
| 高危命中仍存在 | renew_guard |
| 命中为 0，但替代护栏未上线 | renew_with_lower_sensitivity |
| false block 超预算 | open_repair_case |
| owner 缺失 | manual_review |
| 风险已消失且替代护栏 active | retire_guard |
| 超过 maxRenewals 仍说不清价值 | manual_review_or_retire |

不要让长期护栏只会增加不会减少。生产系统的安全性不是规则越多越好，而是每条规则都能解释它今天为什么还该存在。

---

## 7. OpenClaw 课程 Cron 里的落地例子

拿我们这个 Agent 开发课程 cron 来类比：

- 短期 lease：每 3 小时发课后检查 Telegram messageId、README、lesson 文件、TOOLS 已讲列表、git commit；
- residual constraint：不能重复选题、不能漏写目录、不能忘记推送；
- 如果这些约束连续很多轮都需要保留，就不要让每个 cron 自己临时检查；
- 应该升级成 LongTermGuardContract：发布前统一跑 topic dedupe、README index、TOOLS memory、remote head 对账；
- 每 30 天做 sunset review：如果课程系统换成自动 index generator，README 手工检查护栏就可以降级或退休。

这就是“从事故恢复监控”升级到“平台长期能力”。

---

## 8. 常见坑

第一，直接把 residual constraint 写进 if：

~~~ts
if (topic.includes("guard") && lessonNumber > 400) {
  block();
}
~~~

这类规则没有 lineage、scope、owner、reviewAfter，未来没人敢删。

第二，长期护栏没有指标：

只知道它拦了多少次，不知道拦对多少次、误伤多少次、风险是否还存在。

第三，supersession 没有原子收据：

lease 被标记 archived，但 long-term guard 没激活成功，中间出现保护空窗。

第四，sunset review 只看时间：

到期自动续一年，等于把临时恐惧固化成永久复杂度。

---

## 9. 这节课的核心

PostReleaseMonitoringLease 的 supersede_by_long_term_guard 不是“继续监控换个名字”。

它是一次责任转移：

- 从 incident recovery 转到 guard policy；
- 从临时观察转到长期合同；
- 从人工记忆转到可执行 scope / owner / metrics / sunset；
- 从“先别删”转到“到期必须重新证明还值得保留”。

成熟 Agent 不只会把风险挡住，还会管理风险控制本身的生命周期。否则护栏系统最后也会变成需要被护栏保护的复杂度来源。
