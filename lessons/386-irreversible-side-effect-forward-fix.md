# 386. Agent 不可逆副作用的 Forward-Fix 合约（Irreversible Side Effect Forward-Fix Contract）

上一课讲了 Outcome Barrier：外部副作用没有进入 matched/repaired 这类可验证终态之前，下游动作不能放行。

今天继续讲一个更现实的问题：如果副作用已经发生，而且撤不回，怎么办？

很多 Agent 事故不能靠 rollback 解决：

- Telegram 消息已经发出，不能当作没发；
- Git commit 已经 push 到远端，不能随便 force push；
- 邮件已经送达，对方可能已经看到；
- 支付、退款、订单、发货已经进入第三方系统；
- memory 或知识库已经被错误样本污染；
- cron 已经触发多轮执行，下游已经引用了错误状态。

这类问题的重点不是“回到过去”，而是 **forward-fix**：承认事实已经发生，用可审计的补偿动作把系统带回可信状态。

一句话：**不可逆副作用不能回滚，只能用 Forward-Fix Contract 明确补偿目标、证据、屏障和关闭条件。**

## 1. 不要把补偿写成“再试一次”

很多系统的补偿逻辑写得太随意：

    send failed -> retry
    push mismatch -> retry
    deploy wrong -> redeploy
    memory polluted -> edit memory

这不够。retry 解决的是“动作没成功”，forward-fix 解决的是“动作成功了，但业务状态不可信”。

例子：

    Telegram 发错群
      不是 retry 发同一条消息
      而是发纠正说明、记录 messageId、阻止这条错误消息进入学习样本

    Git push 了错误 commit
      不是 force push 删除历史
      而是 revert commit 或追加修复 commit，并验证远端 main 包含修复

    课程目录写了错误主题
      不是只改 README
      而是同时修正 lesson、README、TOOLS、memory，并重新提交证据

所以 Forward-Fix Contract 要回答三个问题：

1. 已经发生了什么事实？
2. 我们要通过什么补偿动作恢复可信？
3. 哪些证据证明补偿已经完成，可以释放下游？

## 2. 先把副作用分类

执行补偿前，先给副作用打分类，避免所有问题都走同一套重试。

最小分类可以这样设计：

    reversible
      能安全撤销。比如未发布的本地文件改动、未提交的暂存区。

    compensatable
      撤不回，但能用新的动作抵消影响。比如发纠正消息、revert commit、补发账单。

    irreversible_observed
      撤不回，也只能记录和通知。比如邮件已读、外部用户已经看到错误内容。

    unsafe_unknown
      不知道远端到底发生了什么。必须先对账，不能盲目补偿。

关键点：**pending 不能直接补偿。**

如果 Telegram API 超时，你还不知道消息是否发出，这时不能马上再发一条“纠正消息”。正确流程是：

    tool timeout
      -> reconciliation 查远端
      -> outcome = matched / duplicate / missing / unsafe_unknown
      -> barrier 判断是否允许 forward-fix
      -> 执行补偿

否则补偿动作本身会制造第二个事故。

## 3. ForwardFixContract 的结构

可以把补偿合约写成这样的结构：

    {
      "contractId": "ffc_agent_course_386",
      "originalEffectId": "effect_telegram_12594",
      "classification": "compensatable",
      "reason": "published content contains wrong lesson number",
      "desiredState": {
        "telegram": "correction message visible in same group",
        "git": "remote main contains fix commit",
        "memory": "daily note references corrected message"
      },
      "compensationPlan": [
        {
          "actionId": "send_correction",
          "tool": "message.send",
          "idempotencyKey": "correction:agent-course:386:telegram",
          "requiredEvidence": ["correctionMessageId"]
        },
        {
          "actionId": "commit_fix",
          "tool": "git.commit_push",
          "idempotencyKey": "fix:agent-course:386:git",
          "requiredEvidence": ["commitSha", "remoteSha"]
        }
      ],
      "releaseBarrier": {
        "allowedOutcomes": ["forward_fixed"],
        "blocks": ["mark_completed", "learning_ingest", "notify_success"]
      }
    }

注意这里不是让 LLM 临场说“应该修好了”。合约把目标状态、补偿动作、幂等键、证据和释放屏障都写清楚。

## 4. learn-claude-code：从 dispatch map 加一层补偿壳

learn-claude-code 的 s02 很适合教学：工具执行就是一个 dispatch map。

原始形态是：

    TOOL_HANDLERS = {
        "bash":       lambda **kw: run_bash(kw["command"]),
        "read_file":  lambda **kw: run_read(kw["path"], kw.get("limit")),
        "write_file": lambda **kw: run_write(kw["path"], kw["content"]),
        "edit_file":  lambda **kw: run_edit(kw["path"], kw["old_text"], kw["new_text"]),
    }

    handler = TOOL_HANDLERS.get(block.name)
    output = handler(**block.input) if handler else f"Unknown tool: {block.name}"

教学版可以加一个很薄的 forward-fix wrapper：

    SIDE_EFFECT_TOOLS = {"bash", "write_file", "edit_file"}

    def execute_with_forward_fix(tool_name: str, args: dict, call_id: str) -> str:
        handler = TOOL_HANDLERS.get(tool_name)
        if handler is None:
            return f"Unknown tool: {tool_name}"

        effect_id = f"effect_{call_id}"
        try:
            output = handler(**args)
            record_effect(effect_id, tool_name, args, output, outcome="needs_reconcile")
            return output
        except Exception as exc:
            contract = maybe_create_forward_fix_contract(
                effect_id=effect_id,
                tool_name=tool_name,
                args=args,
                error=str(exc),
            )
            return f"Error: {exc}\nForwardFixContract: {contract.contract_id}"

    def maybe_create_forward_fix_contract(effect_id, tool_name, args, error):
        if tool_name not in SIDE_EFFECT_TOOLS:
            return NoopContract(effect_id)

        return ForwardFixContract(
            original_effect_id=effect_id,
            classification="unsafe_unknown",
            desired_state="reconcile before retry or compensation",
            release_barrier=["mark_completed", "learning_ingest"],
        )

这里重点不是代码多复杂，而是让每次副作用都有 effect_id。没有 effect_id，后面就无法把原始动作、补偿动作和关闭证据串起来。

## 5. pi-mono：在 tool event stream 上做补偿编排

pi-mono 的 packages/agent/src/agent-loop.ts 已经把工具执行事件拆得比较清楚：

    stream.push({
      type: "tool_execution_start",
      toolCallId: toolCall.id,
      toolName: toolCall.name,
      args: toolCall.arguments,
    });

    result = await tool.execute(...);

    stream.push({
      type: "tool_execution_end",
      toolCallId: toolCall.id,
      toolName: toolCall.name,
      result,
    });

这类 event stream 很适合挂 ForwardFixOrchestrator：

    type ForwardFixContract = {
      contractId: string;
      originalEffectId: string;
      classification: "reversible" | "compensatable" | "irreversible_observed" | "unsafe_unknown";
      desiredState: Record<string, string>;
      compensationPlan: CompensationAction[];
      releaseBarrier: {
        allowedOutcomes: ("reverted" | "forward_fixed" | "accepted_observed")[];
        blocks: string[];
      };
    };

    class ForwardFixOrchestrator {
      async onToolExecutionEnd(event: ToolExecutionEndEvent) {
        const effect = await this.ledger.record(event);
        const outcome = await this.reconciler.check(effect);

        if (outcome === "matched") return;
        if (outcome === "pending") {
          await this.barriers.block(effect.id, ["mark_completed", "learning_ingest"]);
          return;
        }

        const contract = await this.contracts.createFrom(effect, outcome);
        await this.queue.enqueue(contract.compensationPlan);
        await this.barriers.blockUntil(contract.contractId, contract.releaseBarrier);
      }
    }

工程上建议把它做成 Agent.subscribe 外层组件或 middleware，而不是塞进每个工具内部。工具只负责执行，orchestrator 负责记录、分类、补偿和释放。

## 6. OpenClaw 课程 cron 的实战映射

拿我们这个课程 cron 举例，它每 3 小时做一组外部副作用：

1. 写 lesson 文件；
2. 更新 README 目录；
3. 更新 TOOLS 已讲内容；
4. 给 Telegram Rust 学习小组发课；
5. git commit；
6. git push；
7. 写 daily memory 收据。

这里至少有三类不可逆或半不可逆副作用：

- Telegram 消息：发出去后不能当作没发生；
- Git push：公开历史不能靠随手 force push 解决；
- TOOLS/MEMORY：会影响后续选题和自动化判断。

所以完成条件不能只是“message tool 返回成功”和“git push exit 0”。更稳的关闭条件是：

    telegram messageId exists
    lesson file exists with expected title
    README contains lesson 386 link
    TOOLS contains lesson 386 topic
    git commit created
    remote main contains commit sha
    memory records messageId + commit sha

如果其中一个环节错了，比如 Telegram 发了但 Git push 失败，ForwardFixContract 应该阻止“已完成”结论，并给出补偿：

    originalEffect: telegram_message_sent
    failedStep: git_push
    compensation:
      - retry push after pull --rebase
      - if content wrong, send correction message
      - if repo cannot be fixed now, write memory incident and mark lesson unpublished in README
    release:
      - remote main contains fixed commit
      - Telegram correction message exists when needed

这样 Agent 不会因为前半段成功就把整个任务当成功。

## 7. 幂等键：补偿动作也要防重复

补偿动作经常发生在错误处理路径，最容易重复执行。

所以每个 compensation action 都要有 idempotencyKey：

    correction:telegram:-5115329245:lesson-386:v1
    revert:git:gfwfail/agent-course:<bad_sha>
    memory-fix:agent-course:2026-05-23:lesson-386

执行前先查 ledger：

    if ledger.has_success(idempotency_key):
        return existing_receipt

    receipt = execute_compensation(action)
    ledger.record_success(idempotency_key, receipt)
    return receipt

没有幂等键，补偿路径可能从“修复事故”变成“重复发三条纠错消息”。

## 8. 关闭收据：ForwardFixReceipt

补偿完成后，不要只返回一句“已修复”。要生成关闭收据：

    {
      "receiptId": "ffr_agent_course_386",
      "contractId": "ffc_agent_course_386",
      "originalEffectId": "effect_telegram_12594",
      "finalOutcome": "forward_fixed",
      "compensations": [
        {
          "actionId": "send_correction",
          "idempotencyKey": "correction:agent-course:386:telegram",
          "evidence": {"messageId": "12601"}
        },
        {
          "actionId": "commit_fix",
          "idempotencyKey": "fix:agent-course:386:git",
          "evidence": {"commitSha": "abc123", "remoteVerified": true}
        }
      ],
      "blockedDownstreamReleased": ["mark_completed"],
      "stillBlocked": ["learning_ingest"],
      "closedAt": "2026-05-23T05:30:00Z"
    }

注意 finalOutcome 不一定都是 success：

- forward_fixed：已用补偿动作恢复可信；
- accepted_observed：不可修，只能记录和通知，系统接受现实；
- escalated_manual_review：补偿风险太高，交给人；
- abandoned_with_incident：无法修复，只能关闭执行并保留事故。

这比“失败了再重试”更贴近真实生产。

## 9. 常见坑

第一，补偿前没有对账。

不知道远端事实时就补偿，很容易制造重复副作用。

第二，把 force push 当成默认修复。

公开协作仓库里，错误 commit 已经 push 后，默认应该 revert 或 forward fix，除非明确允许重写历史。

第三，补偿完成后没有释放屏障。

修复做完了，但 barrier 还挡着，下游永远卡住；或者 barrier 过早释放，学习系统把错误过程当成功样本。

第四，只修数据，不修记忆。

Agent 系统里的 TOOLS、MEMORY、lesson index、effect ledger 都会影响未来行为。修外部世界时，也要修内部状态。

## 10. 落地清单

给生产 Agent 加 Forward-Fix Contract，可以先做 6 件事：

1. 给所有外部副作用生成 effect_id；
2. 把副作用分成 reversible / compensatable / irreversible_observed / unsafe_unknown；
3. 对 unsafe_unknown 强制先 reconciliation；
4. 为每类副作用定义 compensation template；
5. 每个补偿动作绑定 idempotencyKey 和 requiredEvidence；
6. 用 ForwardFixReceipt 释放 Outcome Barrier。

最终链路应该是：

    effect executed
      -> reconciliation
      -> outcome barrier blocks downstream
      -> forward-fix contract created
      -> compensation actions executed idempotently
      -> forward-fix receipt verified
      -> downstream released or manual review

成熟 Agent 不是假装所有副作用都能回滚，而是知道哪些事实已经发生，并能带证据地把系统往前修回可信状态。

