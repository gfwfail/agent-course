# 410. Agent 护栏审计发现修复队列的调度与冲突预算（Guard Finding Repair Queue Scheduling & Conflict Budget）

上一课讲了 **Guard Decision Audit Finding Triage & Remediation Queue**：Decision audit sample 发现异常后，要升级成结构化 Audit Finding，按 false_allow、false_block、explainability_gap、evidence_overread、policy_stale、outcome_mismatch 分级，进入带 owner、SLA、dedupeKey、repairPath 和 releaseGate 的修复队列。

今天继续往后走：**finding 进了修复队列，不代表可以全部同时修、同时发、同时改同一个护栏包。**

一句话：**Guard finding repair queue 要有批处理调度、冲突预算和 release lane；按 riskSurface、affectedGuardIds、writeScope、releaseGate 和 blast radius 把修复分批，避免多个“正确修复”组合后互相覆盖、重复放量或扩大事故面。**

---

## 1. 为什么修复队列也需要调度

上一课的 triage 可能连续开出几类 finding：

~~~json
[
  {
    "findingId": "finding_101",
    "kind": "false_allow",
    "riskSurface": "message_egress",
    "affectedGuardIds": ["egress.shared_context"],
    "repairPath": "tighten_scope",
    "severity": "critical"
  },
  {
    "findingId": "finding_102",
    "kind": "explainability_gap",
    "riskSurface": "message_egress",
    "affectedGuardIds": ["egress.reason_codes"],
    "repairPath": "update_reason_codes",
    "severity": "medium"
  },
  {
    "findingId": "finding_103",
    "kind": "evidence_overread",
    "riskSurface": "message_egress",
    "affectedGuardIds": ["evidence.access_budget"],
    "repairPath": "reduce_evidence_access",
    "severity": "high"
  }
]
~~~

看起来它们是三个 ticket，但它们都碰同一个 riskSurface：message_egress。

如果系统简单地“谁有 worker 谁先修”，会出现几个问题：

- false_allow 正在加严 scope，explainability_gap 同时改 reason code，最终 replay 很难判断是哪一个修复生效；
- evidence_overread 需要降低证据访问，但另一个补丁为了补 reason code 又扩大 evidenceRefs；
- 两个修复都要求 shadow/canary，结果同一批真实流量被多个 candidate 同时影响；
- 一个 critical 修复还没稳定，medium 修复就把 guard pack schema 又改了一遍；
- 最后 close receipt 证明不了“这个 finding 是被哪个补丁修好的”。

所以 repair queue 不能只是 FIFO。它要像发布系统一样有调度。

---

## 2. 三个核心概念

**Repair Batch**

同一批可以一起修、一起 replay、一起 shadow 的 finding 集合。一个 batch 内的 finding 必须写入范围兼容，验证证据可以合并。

**Release Lane**

不同风险面走不同车道。例如 message_egress、git_push、memory_write 不应该共享同一个 canary 预算。每条 lane 有并发数、blast radius 和 freeze 状态。

**Conflict Budget**

不是所有冲突都绝对禁止。两个 explainability_gap 可能可以合并；两个 critical false_allow 通常不能并发。Conflict Budget 用来限制同一时间同一 riskSurface / guardId / writeScope 能承受多少变化。

---

## 3. 最小数据结构

~~~ts
type FindingKind =
  | "false_allow"
  | "false_block"
  | "explainability_gap"
  | "evidence_overread"
  | "policy_stale"
  | "outcome_mismatch";

type Severity = "low" | "medium" | "high" | "critical";

type RepairFinding = {
  findingId: string;
  dedupeKey: string;
  kind: FindingKind;
  severity: Severity;
  riskSurface: string;
  affectedGuardIds: string[];
  writeScope: string[];
  repairPath:
    | "tighten_scope"
    | "loosen_scope"
    | "repair_classifier"
    | "update_reason_codes"
    | "reduce_evidence_access"
    | "reconcile_outcome"
    | "manual_review";
  releaseGate: {
    requiresReplay: boolean;
    requiresShadow: boolean;
    requiresCanary: boolean;
  };
  openedAt: string;
  resolveBy: string;
};

type ReleaseLanePolicy = {
  riskSurface: string;
  maxConcurrentBatches: number;
  maxCriticalPerBatch: number;
  maxGuardOverlap: number;
  frozen: boolean;
  canaryBudgetPercent: number;
};

type RepairBatch = {
  batchId: string;
  lane: string;
  findingIds: string[];
  affectedGuardIds: string[];
  writeScope: string[];
  requiredGates: {
    replay: boolean;
    shadow: boolean;
    canary: boolean;
  };
  conflictBudgetUsed: {
    criticalCount: number;
    guardOverlap: number;
    canaryBudgetPercent: number;
  };
  status: "scheduled" | "in_repair" | "verifying" | "ready_for_release";
};
~~~

关键点：

- finding 自己声明 affectedGuardIds 和 writeScope；
- lane policy 决定同一 riskSurface 能跑多少批；
- batch 合并 releaseGate，不能因为合批降低验证要求；
- conflictBudgetUsed 是后续审计和回滚的证据。

---

## 4. 调度规则

一个实用的调度器可以从五条规则开始：

~~~text
规则 1：critical false_allow 独占 lane
  同一 riskSurface 下，critical false_allow 不和其他修复合批。

规则 2：相同 affectedGuardIds 不并发
  两个 batch 如果碰同一个 guardId，除非都是 reason code 级别修复，否则必须串行。

规则 3：writeScope 交叉需要仲裁
  同时改 guard-pack.yaml / policy.ts / evidence-budget.ts 时，要选择一个 owner batch。

规则 4：releaseGate 只升级不降级
  batch 内任何 finding 要 canary，整个 batch 就要 canary。

规则 5：frozen lane 只允许解冻修复
  如果 message_egress 已冻结，只能调度 tighten_scope、reduce_evidence_access、reconcile_outcome 这类直接降低风险的修复。
~~~

这套规则的目标不是让队列最满，而是让每个修复的因果关系可证明。

---

## 5. learn-claude-code：纯函数调度器

教学版先写一个纯函数：输入 findings、lane policy、当前 running batches，输出下一批可调度 batch。

~~~py
from dataclasses import dataclass
from typing import Literal


Severity = Literal["low", "medium", "high", "critical"]
RepairPath = Literal[
    "tighten_scope",
    "loosen_scope",
    "repair_classifier",
    "update_reason_codes",
    "reduce_evidence_access",
    "reconcile_outcome",
    "manual_review",
]


@dataclass(frozen=True)
class Finding:
    finding_id: str
    kind: str
    severity: Severity
    risk_surface: str
    affected_guard_ids: tuple[str, ...]
    write_scope: tuple[str, ...]
    repair_path: RepairPath
    requires_replay: bool
    requires_shadow: bool
    requires_canary: bool


@dataclass(frozen=True)
class LanePolicy:
    risk_surface: str
    max_concurrent_batches: int
    max_critical_per_batch: int
    max_guard_overlap: int
    frozen: bool
    canary_budget_percent: int


@dataclass(frozen=True)
class RunningBatch:
    batch_id: str
    risk_surface: str
    affected_guard_ids: tuple[str, ...]
    write_scope: tuple[str, ...]


LOWER_RISK_REPAIRS = {
    "tighten_scope",
    "reduce_evidence_access",
    "reconcile_outcome",
}


def overlaps(left: tuple[str, ...], right: tuple[str, ...]) -> int:
    return len(set(left) & set(right))


def can_run_in_frozen_lane(finding: Finding) -> bool:
    return finding.repair_path in LOWER_RISK_REPAIRS


def conflicts_with_running(finding: Finding, running: list[RunningBatch]) -> bool:
    for batch in running:
        if batch.risk_surface != finding.risk_surface:
            continue
        if overlaps(finding.affected_guard_ids, batch.affected_guard_ids) > 0:
            return True
        if overlaps(finding.write_scope, batch.write_scope) > 0:
            return True
    return False


def schedule_repair_batch(
    findings: list[Finding],
    policy: LanePolicy,
    running: list[RunningBatch],
) -> dict:
    lane_running = [b for b in running if b.risk_surface == policy.risk_surface]
    if len(lane_running) >= policy.max_concurrent_batches:
        return {"action": "wait", "reason": "lane_concurrency_full"}

    candidates = [
        f
        for f in findings
        if f.risk_surface == policy.risk_surface
        and not conflicts_with_running(f, lane_running)
        and (not policy.frozen or can_run_in_frozen_lane(f))
    ]

    if not candidates:
        return {"action": "wait", "reason": "no_eligible_findings"}

    candidates.sort(
        key=lambda f: (
            f.severity != "critical",
            f.severity not in {"critical", "high"},
            f.finding_id,
        )
    )

    first = candidates[0]

    if first.severity == "critical" and first.kind == "false_allow":
        selected = [first]
    else:
        selected = []
        used_guards: set[str] = set()
        critical_count = 0

        for finding in candidates:
            next_critical = critical_count + (1 if finding.severity == "critical" else 0)
            next_overlap = len(used_guards & set(finding.affected_guard_ids))

            if next_critical > policy.max_critical_per_batch:
                continue
            if next_overlap > policy.max_guard_overlap:
                continue
            if overlaps(
                tuple(scope for f in selected for scope in f.write_scope),
                finding.write_scope,
            ) > 0:
                continue

            selected.append(finding)
            used_guards.update(finding.affected_guard_ids)
            critical_count = next_critical

    return {
        "action": "schedule_batch",
        "lane": policy.risk_surface,
        "findingIds": [f.finding_id for f in selected],
        "requiredGates": {
            "replay": any(f.requires_replay for f in selected),
            "shadow": any(f.requires_shadow for f in selected),
            "canary": any(f.requires_canary for f in selected),
        },
        "conflictBudgetUsed": {
            "criticalCount": sum(1 for f in selected if f.severity == "critical"),
            "affectedGuardCount": len({g for f in selected for g in f.affected_guard_ids}),
            "canaryBudgetPercent": policy.canary_budget_percent
            if any(f.requires_canary for f in selected)
            else 0,
        },
    }
~~~

这段代码没有依赖 LLM。好处是：调度策略可以被 replay、单测和审计，而不是藏在 prompt 里。

---

## 6. pi-mono：RepairQueueScheduler + EventBus

在 pi-mono 里，RepairQueueScheduler 可以订阅 finding.opened / finding.updated / batch.completed 事件，定期尝试调度下一批。

~~~ts
class RepairQueueScheduler {
  constructor(
    private readonly findings: FindingRepository,
    private readonly lanes: ReleaseLaneRepository,
    private readonly batches: RepairBatchRepository,
    private readonly eventBus: EventBus,
  ) {}

  async tick(riskSurface: string) {
    const policy = await this.lanes.getPolicy(riskSurface);
    const openFindings = await this.findings.listOpen({ riskSurface });
    const running = await this.batches.listRunning({ riskSurface });

    const decision = scheduleRepairBatch(openFindings, policy, running);

    if (decision.action === "wait") {
      await this.eventBus.emit({
        type: "guard.repair.scheduler.waited",
        riskSurface,
        reason: decision.reason,
      });
      return;
    }

    const batch = await this.batches.create({
      lane: decision.lane,
      findingIds: decision.findingIds,
      requiredGates: decision.requiredGates,
      conflictBudgetUsed: decision.conflictBudgetUsed,
      status: "scheduled",
    });

    await this.findings.markScheduled(decision.findingIds, batch.batchId);

    await this.eventBus.emit({
      type: "guard.repair.batch.scheduled",
      batchId: batch.batchId,
      lane: decision.lane,
      findingIds: decision.findingIds,
      requiredGates: decision.requiredGates,
      conflictBudgetUsed: decision.conflictBudgetUsed,
    });
  }
}
~~~

Worker 收到 batch.scheduled 后再去 spawn 修复任务，而不是直接从 finding queue 抢 ticket。

这样有几个好处：

- 每个 worker 只负责一个 batch；
- batch 明确拥有 writeScope；
- replay/shadow/canary 要求在派工前已经确定；
- scheduler.waited 也是证据，能解释为什么队列暂时没有推进。

---

## 7. OpenClaw 课程 Cron 实战类比

拿这门课的 cron 做例子，假设审计发现三个问题：

~~~text
finding A：Telegram 课程发送缺少 secret scan receipt
  riskSurface = message_egress
  repairPath = update_reason_codes
  severity = medium

finding B：Git push 前没有检查 PR 状态
  riskSurface = git_push
  repairPath = tighten_scope
  severity = critical

finding C：写 TOOLS.md 已讲内容时没有去重
  riskSurface = memory_write
  repairPath = repair_classifier
  severity = medium
~~~

这三个 finding 可以进三条不同 release lane：

~~~text
message_egress lane
  可以修 reason code，不影响 git push。

git_push lane
  critical false_allow，独占 lane，先 replay 再 shadow，再 canary。

memory_write lane
  修去重逻辑，可以和 message_egress 并行，但不能和另一个 TOOLS.md 写入修复并行。
~~~

如果还有一个 finding D 也是 git_push，要求改同一个 pre_push_gate.ts，那就不能和 B 同时修。它要等 B 的 replay / shadow / canary 收据出来，再基于新主线重新调度。

这就是调度器的价值：它不是“多派几个 worker”，而是让并发修复仍然保持因果清晰。

---

## 8. 常见坑

**坑 1：队列按时间先进先出。**

FIFO 对普通任务可以，对 safety finding 不够。critical false_allow 必须优先，而且经常要独占 lane。

**坑 2：多个 worker 同时改同一个 guard。**

最后 merge 时也许能解决文本冲突，但 replay 证据已经失真。修复系统最怕“看起来都通过，但不知道谁修了什么”。

**坑 3：合批时降低 releaseGate。**

一个 batch 里只要有一个 finding 要 canary，整个 batch 就要 canary。验证要求只能取并集，不能取平均。

**坑 4：frozen lane 继续排普通优化。**

lane 冻结时只能调度降低风险的修复。reason code 美化、文案调整、非阻塞优化都应该等解冻。

**坑 5：没有 waited receipt。**

队列没动也要有原因：lane full、frozen、writeScope conflict、等待人工、等待主线同步。否则 later review 看不出系统是卡住还是有意等待。

---

## 9. 落地清单

给 Guard Finding Repair Queue 加调度，可以按 8 步做：

1. 每个 finding 必须声明 riskSurface、affectedGuardIds、writeScope、releaseGate；
2. 为高风险面建立 release lane：message_egress、git_push、memory_write、tool_execution；
3. 给每条 lane 配 maxConcurrentBatches、maxCriticalPerBatch、maxGuardOverlap、canaryBudget；
4. critical false_allow 默认独占 lane；
5. frozen lane 只允许降低风险的 repairPath；
6. batch 的 releaseGate 取所有 finding 的并集；
7. batch.scheduled / scheduler.waited / batch.completed 都写 EventBus receipt；
8. finding close 前反查 batchId、repair commit、replay/shadow/canary 收据。

连起来的流程是：

~~~text
audit finding opened
  -> repair queue scheduler
  -> lane / conflict budget decision
  -> repair batch scheduled
  -> worker patch
  -> replay / shadow / canary
  -> finding close receipt
~~~

---

## 10. 总结

这一课的核心：

1. 修复队列不能只是 FIFO，它要理解 riskSurface、guardId、writeScope 和 releaseGate；
2. critical false_allow 通常要独占 lane，先降低风险，再谈合批效率；
3. Conflict Budget 让系统知道哪些修复可以并发，哪些必须串行；
4. learn-claude-code 可以用纯函数调度器，把队列决策变成可测试逻辑；
5. pi-mono 可以用 RepairQueueScheduler + EventBus，把 batch.scheduled / waited / completed 都变成收据；
6. OpenClaw 的 Telegram、Git push、memory write 都可以映射成不同 release lane，用来练习真实副作用调度。

成熟 Agent 的自动修复，不是开很多 worker 把队列清空，而是让每个修复在正确的车道、正确的预算、正确的验证门里前进。
