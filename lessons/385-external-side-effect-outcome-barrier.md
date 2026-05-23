# 385. Agent 外部副作用结果屏障与下游放行闸门（External Side Effect Outcome Barrier & Downstream Release Gate）

上一课讲了 External Side Effect Reconciliation：外部副作用执行后，要用本地 Effect Ledger 和远端 Reality Snapshot 对账，区分 matched、pending、duplicate、payload_mismatch 等状态。

今天继续往后走一步：对账还没结束时，下游动作能不能继续？

现实里很多事故不是“第一步错了”，而是第一步还没确认，第二步已经开始扩散：

- Telegram 发送超时，Agent 还没确认消息是否存在，就继续写“已发布”；
- Git push 返回 accepted，还没确认远端 main 包含 commit，就通知用户“已上线”；
- 部署 API 返回 queued，还没 rollout 成功，就开始跑数据迁移；
- 邮件还在 pending，Agent 就把 customer ticket 标成 replied；
- 外部订单创建结果未确认，下游库存已经扣减；
- obligation 还没完成，学习规则已经把这次执行当成成功样本。

所以今天讲 External Side Effect Outcome Barrier & Downstream Release Gate。

一句话：**外部副作用没有进入 terminal matched 状态之前，任何依赖它的下游动作都不能被放行。**

## 1. 为什么需要 Outcome Barrier

Reconciliation 负责回答：

    本地 ledger 和远端现实是否一致？

Outcome Barrier 负责回答：

    这个结果是否足够确定，可以让下游继续？

这两个问题不一样。

对账可能得到 pending，这不是失败，但也不是成功。它的正确处理不是继续往后跑，而是把依赖它的动作挡住：

    side_effect executed
      -> reconciliation pending
      -> downstream blocked
      -> recheck later
      -> matched
      -> downstream released

没有 barrier，Agent 很容易把“工具调用已返回”误当成“业务结果已完成”。

生产系统里要记住一条硬规则：

**外部副作用的完成条件，不是工具返回 success，而是 outcome 达到可验证终态。**

## 2. Terminal outcome：先定义什么叫“完成”

不同副作用的终态不一样，不能统一写成 succeeded。

最小 OutcomeState 可以这样分：

    pending
      远端还在最终一致性窗口里，稍后再查。

    matched
      远端事实和本地期待一致，且必要 obligation 已完成。

    repaired
      发生过分歧，但已经补账或补偿，并有修复证据。

    failed_terminal
      已确认失败，继续重试也不会改变结果。

    unsafe_terminal
      出现重复、payload mismatch、越权等危险终态，需要冻结或人工复核。

Outcome Barrier 只允许特定下游依赖通过。

例如：

- 普通通知依赖 matched 或 repaired；
- 训练样本回灌只依赖 matched，不能接受 repaired；
- 事故报告可以依赖 failed_terminal 或 unsafe_terminal；
- 补偿动作可以依赖 unsafe_terminal，但不能依赖 pending；
- “已完成”回复只能依赖 matched/repaired，不能依赖 tool_success。

## 3. 把依赖关系写进 BarrierSpec

不要让 LLM 临场判断“这个应该可以继续吧”。把下游动作和前置 outcome 绑定成结构化 BarrierSpec。

示例：

    {
      "barrierId": "barrier_agent_course_385_publish",
      "dependsOn": [
        {
          "effectId": "effect_385_telegram",
          "allowedOutcomes": ["matched"],
          "reason": "message must exist before memory says published"
        },
        {
          "effectId": "effect_385_git_push",
          "allowedOutcomes": ["matched"],
          "reason": "remote repo must contain lesson commit"
        }
      ],
      "downstreamAction": {
        "type": "append_memory_summary",
        "target": "memory/2026-05-23.md"
      },
      "onBlocked": "wait_or_schedule_recheck",
      "onUnsafe": "freeze_and_review"
    }

BarrierSpec 的价值是把“先后顺序”从口头约定变成可执行契约。

它和普通 DAG 依赖的区别在于：普通 DAG 常常只看任务是否结束；Outcome Barrier 看的是外部现实是否达到允许的业务终态。

## 4. learn-claude-code：教学版 Barrier Gate

教学版可以先做成纯函数：输入 barrier spec 和 outcome ledger，输出 release / block / escalate。

    from dataclasses import dataclass
    from typing import Literal

    Outcome = Literal[
        "pending",
        "matched",
        "repaired",
        "failed_terminal",
        "unsafe_terminal",
        "unknown",
    ]

    Decision = Literal["release", "block", "escalate"]

    @dataclass(frozen=True)
    class Dependency:
        effect_id: str
        allowed_outcomes: set[Outcome]
        reason: str

    @dataclass(frozen=True)
    class BarrierSpec:
        barrier_id: str
        downstream_action: str
        dependencies: list[Dependency]

    def decide_barrier(spec: BarrierSpec, outcomes: dict[str, Outcome]) -> dict:
        blocked = []
        unsafe = []
        missing = []

        for dep in spec.dependencies:
            outcome = outcomes.get(dep.effect_id, "unknown")

            if outcome == "unknown":
                missing.append({
                    "effectId": dep.effect_id,
                    "reason": "missing outcome",
                })
                continue

            if outcome == "unsafe_terminal":
                unsafe.append({
                    "effectId": dep.effect_id,
                    "outcome": outcome,
                    "reason": dep.reason,
                })
                continue

            if outcome not in dep.allowed_outcomes:
                blocked.append({
                    "effectId": dep.effect_id,
                    "outcome": outcome,
                    "required": sorted(dep.allowed_outcomes),
                    "reason": dep.reason,
                })

        if unsafe:
            return {
                "barrierId": spec.barrier_id,
                "decision": "escalate",
                "unsafe": unsafe,
                "next": "freeze_downstream_and_open_review",
            }

        if missing or blocked:
            return {
                "barrierId": spec.barrier_id,
                "decision": "block",
                "missing": missing,
                "blocked": blocked,
                "next": "schedule_reconciliation_recheck",
            }

        return {
            "barrierId": spec.barrier_id,
            "decision": "release",
            "downstreamAction": spec.downstream_action,
        }

这个函数不执行 Telegram、Git、部署或写 memory。它只判断“下游现在能不能继续”。

这很重要：Barrier Gate 应该容易单测、容易 replay、容易审计。

## 5. pi-mono：DownstreamReleaseGate

生产版可以把 barrier 放在任务队列、工具分发和 obligation worker 之间。

    type Outcome =
      | "pending"
      | "matched"
      | "repaired"
      | "failed_terminal"
      | "unsafe_terminal"
      | "unknown";

    type BarrierDependency = {
      effectId: string;
      allowedOutcomes: Outcome[];
      reason: string;
    };

    type BarrierSpec = {
      barrierId: string;
      downstreamAction: {
        type: string;
        target: string;
        idempotencyKey: string;
      };
      dependencies: BarrierDependency[];
      onBlocked: "wait_or_schedule_recheck" | "dead_letter";
      onUnsafe: "freeze_and_review";
    };

    class DownstreamReleaseGate {
      constructor(
        private readonly outcomeStore: OutcomeStore,
        private readonly queue: TaskQueue,
        private readonly audit: AuditLog,
      ) {}

      async evaluate(spec: BarrierSpec): Promise<void> {
        const outcomes = await this.outcomeStore.getMany(
          spec.dependencies.map((item) => item.effectId),
        );

        const decision = decideBarrier(spec, outcomes);

        await this.audit.append({
          type: "side_effect.barrier_evaluated",
          barrierId: spec.barrierId,
          decision: decision.kind,
          dependencies: decision.dependencies,
          observedAt: new Date().toISOString(),
        });

        if (decision.kind === "release") {
          await this.queue.enqueue({
            ...spec.downstreamAction,
            idempotencyKey: spec.downstreamAction.idempotencyKey,
            releasedByBarrierId: spec.barrierId,
          });
          return;
        }

        if (decision.kind === "block") {
          await this.queue.schedule({
            type: "side_effect.recheck_barrier",
            barrierId: spec.barrierId,
            runAfterMs: 60_000,
          });
          return;
        }

        await this.queue.enqueue({
          type: "incident.open_review",
          target: spec.barrierId,
          idempotencyKey: "review:" + spec.barrierId,
        });
      }
    }

这里有三个重点：

- release 不是直接执行下游，而是把下游动作幂等入队；
- block 不是失败，而是安排重新对账；
- unsafe 不继续猜，直接冻结和复核。

## 6. Barrier 要和 idempotency 一起用

Outcome Barrier 不是幂等的替代品。

它挡的是“什么时候可以继续”，幂等键挡的是“继续时会不会重复”。

下游动作被 barrier 多次 evaluate 很正常，所以 release 也必须幂等：

    idempotencyKey = "downstream:" + barrierId + ":" + actionType

否则会出现一个新问题：

1. barrier 第一次看到 matched，enqueue memory_update；
2. audit 写入超时；
3. worker 重试 evaluate；
4. 第二次又 enqueue memory_update；
5. 下游写了两份完成记录。

所以 barrier release 必须遵守两个规则：

- 同一个 barrierId + downstreamAction 只能 release 一次；
- release 后即使再次 evaluate，也只能返回 already_released。

## 7. OpenClaw 实战：课程 Cron 的结果屏障

拿今天这个课程 cron 举例。它有几个外部/持久副作用：

    telegram_send
      要求：messageId 存在，内容 hash 对得上。

    git_push
      要求：remote main 包含新 commit。

    tools_update
      要求：TOOLS.md 已讲内容包含第385课主题。

    memory_update
      要求：daily memory 写入 messageId、commit、远端验证。

这些动作之间有依赖：

    lesson_file + README
      -> git commit
      -> git push matched
      -> Telegram message matched
      -> memory closeout
      -> final reply

如果 Telegram 已发，但 git push 还 pending，不能在 memory 里写“课程完整发布成功”。更准确的状态是：

    telegram matched
    git pending
    final closeout blocked by git outcome

如果 git push 成功，但 Telegram poll 发现 duplicate_external_effect，不能继续写“成功”。应该进入 unsafe_terminal，冻结下游自动发布，开 review。

这就是 Outcome Barrier 的价值：让 Agent 不把半完成状态包装成完成。

## 8. 和前几课的关系

最近几课连起来是一条完整链路：

    Preview & Dry-run Receipt
      执行前冻结计划。

    Effect Ledger & Execution Receipt
      执行时记录事实。

    Reconciliation
      执行后读取远端现实并分类。

    Outcome Barrier
      对账未达终态时挡住下游扩散。

    Repair / Compensation
      发现危险分歧时补账、补偿或升级。

这不是为了把系统做复杂，而是为了防止 Agent 在自动化里最常见的一类错误：**上一步还没证明完成，下一步已经对外宣布完成。**

## 9. 最小检查清单

给外部副作用加 Outcome Barrier，至少检查 8 件事：

- [ ] 是否为每个外部写定义 terminal outcome？
- [ ] pending 是否明确不能放行普通下游？
- [ ] BarrierSpec 是否列出 effectId 和 allowedOutcomes？
- [ ] unsafe_terminal 是否直接冻结或升级？
- [ ] release 是否幂等，能防止重复入队？
- [ ] barrier evaluation 是否 append-only 审计？
- [ ] blocked 状态是否有 recheck 计划，而不是静默卡死？
- [ ] final reply / memory / learning backfill 是否都依赖 barrier release？

## 10. 这一课的关键

Agent 自动化不怕分步骤，怕的是步骤之间没有硬边界。

外部副作用尤其如此：工具返回、远端现实、业务完成、下游放行，是四个不同概念。

记住一句话：

**成熟 Agent 不会把 pending 当成功；它会等外部现实进入可验证终态，再放行下游动作。**
