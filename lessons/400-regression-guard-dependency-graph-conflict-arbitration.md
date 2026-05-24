# 400. Agent 回归护栏依赖图与冲突仲裁（Regression Guard Dependency Graph & Conflict Arbitration）

上一课讲了 **Regression Guard Effectiveness Telemetry**：active guard 不能默认永久正确，要持续用 evaluation/outcome 遥测决定 keep、tighten、demote 或 retire。

今天补一个上线后更容易踩的坑：**guard 多了以后会互相影响**。

一句话：**Regression Guard 不能只是一堆独立 if；它们要进入依赖图，先按证据、范围、风险和优先级排序，再用冲突仲裁器给出一个可解释的最终决策。**

---

## 1. 为什么 guard 会互相打架

单个 guard 很容易理解：

- stale evidence guard：证据过期就 block；
- side effect guard：外部副作用必须 dry-run；
- regression guard：命中历史事故模式就 block；
- cost guard：超预算就降级；
- user override guard：老板批准后 allow。

问题是这些 guard 同时命中时，系统不能简单地“谁最后执行谁说了算”。

典型冲突：

1. human_override 想 allow，但 regression_guard 发现这是历史事故复现；
2. cost_guard 想 demote to dry-run，但 security_guard 要求直接 block；
3. guard_A 依赖 fresh evidence，guard_B 已经判定 evidence stale；
4. guard_C 已经被 demote，但旧 policy cache 还把它当 active。

成熟系统要回答的是：**哪些 guard 先跑、哪些 guard 的结果能覆盖别人、冲突时保守到什么程度、最终原因怎么审计。**

---

## 2. 最小模型：GuardDependencyGraph

不要让每个 guard 自己决定世界。先把 guard 声明成节点：

~~~ts
type GuardTier = "shadow" | "canary" | "active" | "retired";
type GuardDecision = "allow" | "warn" | "dry_run" | "block" | "unknown";

type GuardNode = {
  guardId: string;
  tier: GuardTier;
  riskSurface: "security" | "external_side_effect" | "cost" | "quality";
  priority: number;
  dependsOn: string[];
  overrides: string[];
  failClosed: boolean;
};

type GuardEvaluation = {
  guardId: string;
  decision: GuardDecision;
  reasonCodes: string[];
  evidenceRefs: string[];
  latencyMs: number;
};
~~~

这里有几个关键字段：

- dependsOn：这个 guard 的判断依赖哪些 guard 已经完成；
- overrides：这个 guard 在冲突时能覆盖哪些低优先级 guard；
- failClosed：guard 自己 unknown / timeout 时是否保守 block；
- priority：同一层级冲突时用来排序。

注意：priority 不是权限。它只是仲裁里的排序信号。真正的覆盖关系要靠 overrides 和风险面规则。

---

## 3. learn-claude-code：最小依赖图排序器

教学版可以先写一个纯函数：输入 guard 节点，输出执行顺序；有环就拒绝发布。

~~~py
from dataclasses import dataclass


@dataclass(frozen=True)
class GuardNode:
    guard_id: str
    priority: int
    depends_on: tuple[str, ...]
    overrides: tuple[str, ...]
    fail_closed: bool = True


def order_guards(nodes: list[GuardNode]) -> list[str]:
    by_id = {node.guard_id: node for node in nodes}
    visiting: set[str] = set()
    visited: set[str] = set()
    ordered: list[str] = []

    def visit(guard_id: str):
        if guard_id in visited:
            return
        if guard_id in visiting:
            raise ValueError(f"guard dependency cycle: {guard_id}")
        if guard_id not in by_id:
            raise ValueError(f"missing guard dependency: {guard_id}")

        visiting.add(guard_id)
        node = by_id[guard_id]
        for dep in sorted(node.depends_on):
            visit(dep)
        visiting.remove(guard_id)
        visited.add(guard_id)
        ordered.append(guard_id)

    for node in sorted(nodes, key=lambda n: (-n.priority, n.guard_id)):
        visit(node.guard_id)

    return ordered
~~~

这段代码的重点不是算法多复杂，而是两个生产约束：

1. **发布前就发现 dependency cycle**，不要等请求进来才卡死；
2. **missing dependency 直接失败**，不要把“没跑过”当 allow。

learn-claude-code 里的 EventBus 已经演示了把生命周期事件写成 append-only log。这里可以把 guard_graph_validated、guard_cycle_rejected 也写进去，方便回放。

---

## 4. 冲突仲裁：最终决策不能靠最后一个 guard

所有 guard 都跑完后，要把多个评估压成一个最终动作：

~~~py
RANK = {
    "allow": 0,
    "warn": 1,
    "dry_run": 2,
    "unknown": 3,
    "block": 4,
}


def arbitrate(
    evaluations: list[dict],
    nodes: dict[str, GuardNode],
) -> dict:
    effective = []

    for ev in evaluations:
        node = nodes[ev["guard_id"]]
        decision = ev["decision"]

        if decision == "unknown" and node.fail_closed:
            decision = "block"

        effective.append({
            **ev,
            "effective_decision": decision,
            "priority": node.priority,
            "overrides": set(node.overrides),
        })

    winners = []
    for candidate in effective:
        overridden = any(
            candidate["guard_id"] in other["overrides"]
            and RANK[other["effective_decision"]] >= RANK[candidate["effective_decision"]]
            for other in effective
        )
        if not overridden:
            winners.append(candidate)

    final = max(
        winners,
        key=lambda ev: (RANK[ev["effective_decision"]], ev["priority"]),
    )

    return {
        "decision": final["effective_decision"],
        "winning_guard_id": final["guard_id"],
        "reason_codes": final["reasonCodes"],
        "conflicting_guard_ids": [
            ev["guard_id"] for ev in effective if ev["guard_id"] != final["guard_id"]
        ],
    }
~~~

仲裁原则很简单：

- block > unknown > dry_run > warn > allow；
- unknown + failClosed 先变成 block；
- 只有声明了 overrides 的 guard 才能覆盖别人；
- 最终输出必须带 winning_guard_id 和 conflicting_guard_ids。

这样以后出事故时，审计者不是看一堆日志猜“为什么放行”，而是直接看到哪条 guard 赢了。

---

## 5. pi-mono：用 EventBus 发布 Guard Arbitration 事件

pi-mono 里已经有一个很轻的 EventBus：

~~~ts
export interface EventBus {
  emit(channel: string, data: unknown): void;
  on(channel: string, handler: (data: unknown) => void): () => void;
}
~~~

可以在 guard runner 外层加一层 arbitration middleware：

~~~ts
type GuardArbitrationEvent = {
  eventId: string;
  runId: string;
  graphVersion: number;
  guardPackVersion: number;
  evaluationIds: string[];
  decision: GuardDecision;
  winningGuardId: string;
  conflictingGuardIds: string[];
  reasonCodes: string[];
  createdAt: Date;
};

async function runGuardGraph(
  ctx: AgentRunContext,
  graph: GuardNode[],
  guards: Record<string, GuardHandler>,
) {
  const order = orderGuardGraph(graph);
  const evaluations: GuardEvaluation[] = [];

  for (const guardId of order) {
    const handler = guards[guardId];
    const started = Date.now();

    try {
      const result = await handler.evaluate(ctx);
      evaluations.push({
        ...result,
        guardId,
        latencyMs: Date.now() - started,
      });
    } catch (err) {
      evaluations.push({
        guardId,
        decision: "unknown",
        reasonCodes: ["guard_handler_error"],
        evidenceRefs: [],
        latencyMs: Date.now() - started,
      });
    }
  }

  const arbitration = arbitrateGuardDecisions(graph, evaluations);

  ctx.events.emit("guard.arbitrated", {
    eventId: crypto.randomUUID(),
    runId: ctx.runId,
    graphVersion: ctx.guardGraphVersion,
    guardPackVersion: ctx.guardPackVersion,
    evaluationIds: evaluations.map((ev) => ev.guardId),
    decision: arbitration.decision,
    winningGuardId: arbitration.winningGuardId,
    conflictingGuardIds: arbitration.conflictingGuardIds,
    reasonCodes: arbitration.reasonCodes,
    createdAt: new Date(),
  } satisfies GuardArbitrationEvent);

  return arbitration;
}
~~~

这里要注意：guard.arbitrated 是一等事件，不是 debug log。它应该进入审计、回放和 telemetry。

---

## 6. OpenClaw 课程 Cron 的类比

这套课程发布任务其实也有 guard graph：

1. 选题去重：不能重复 TOOLS.md 已讲主题；
2. 文件生成：lesson 必须写入 lessons；
3. 目录更新：README 必须指向 lesson；
4. 外部发布：Telegram messageId 要可追踪；
5. Git 发布：commit/push 后远端 main 要能验证；
6. 记忆更新：TOOLS.md 和 daily memory 要记录。

如果 Telegram 发布成功，但 Git push 失败，这不是“课程成功一半”。外部副作用已经发生，系统必须进入补账或修复。

如果 Git push 成功，但 TOOLS.md 没更新，下次 cron 可能重复选题。

所以这类 always-on Agent 不应该只写线性脚本，而应该把关键 gate 和依赖显式建图。哪个 gate 赢、哪个 gate 阻断、哪个副作用已经发生，都要有事件。

---

## 7. 常见坑

**坑 1：只按代码顺序执行 guard**

代码顺序不是策略。guard 顺序要从 graphVersion 里来。

**坑 2：把 allow 当作强信号**

allow 通常只是“这个 guard 没发现问题”。它不能覆盖高风险 block。

**坑 3：human override 覆盖所有 guard**

人工批准也要有 scope、TTL 和 allowedActions。它可以覆盖某些 business guard，但不应该绕过 security / evidence invalidation guard。

**坑 4：guard timeout 默认 allow**

高风险 guard timeout 必须 fail closed。低风险质量 guard 可以 warn 或 shadow。

**坑 5：没有记录冲突**

如果只记录最终 block/allow，后面就无法改进 guard。冲突本身就是高价值训练数据。

---

## 8. 实战检查清单

- [ ] Guard Pack 里每个 guard 都有 tier、riskSurface、priority；
- [ ] guard 依赖发布前做 cycle detection；
- [ ] missing dependency 直接拒绝发布；
- [ ] unknown + failClosed 会转成 block；
- [ ] overrides 显式声明，不能靠最后执行覆盖；
- [ ] 每次 run 产出 guard.arbitrated 事件；
- [ ] arbitration 事件记录 winningGuardId 和 conflictingGuardIds；
- [ ] human override 有 scope、TTL、allowedActions；
- [ ] 降级/退役 guard 后刷新 graphVersion，并做运行时一致性检查。

---

## 9. 一句话收尾

> Regression Guard 多起来以后，真正的风险不是少一条 if，而是多条 guard 在不同缓存、不同顺序、不同版本里互相打架。用依赖图和冲突仲裁把最终决策变成可解释、可回放、可审计的工程结果。
