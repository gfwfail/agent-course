# 389. Agent 补偿关闭对账与最终释放闸门（Compensation Closeout Reconciliation Gate）

上一课讲了 Compensation Ledger：补偿动作要按步骤幂等续跑，不能失败后整条 runbook 重放。

今天补最后一个生产闭环：**补偿步骤全部 succeeded，不代表事故已经可以关闭。**

因为补偿动作通常横跨多个外部系统。每个 step 成功，只说明“这一步工具认为自己完成了”；真正关闭之前，还要确认：

- 外部现实和 ledger 一致；
- 原始错误影响已经被覆盖或解释；
- 下游屏障可以释放；
- 没有残留 unknown / duplicate / stale evidence；
- 关闭收据可以被以后审计重放。

一句话：**Compensation closeout 不是把 runbook 跑完，而是用 reconciliation gate 证明补偿结果、外部现实、证据链和下游状态已经一致。**

## 1. 为什么 succeeded 还不够

看一个课程 cron 的例子：

1. 发群消息成功，拿到 messageId；
2. 写 lesson 文件成功；
3. 更新 README 成功；
4. 更新 TOOLS 成功；
5. git commit 成功；
6. git push 超时，但远端其实已经收到；
7. memory 写入成功；
8. Agent 看到本地步骤大多 succeeded，于是宣布完成。

这里风险不在某个步骤，而在最终事实有没有对齐：

- Telegram 是否真的有那条课；
- GitHub main 是否真的包含 commit；
- README 是否指向存在的 lesson；
- TOOLS 已讲内容是否和 README 主题一致；
- memory 记录的 commit/messageId 是否真实；
- 下游“下一课选题”是否可以基于第 389 课继续。

如果 closeout 只看本地 ledger，就会把“局部成功”错当“系统恢复完成”。

## 2. Closeout Gate 的最小模型

补偿关闭可以建成一个独立的 gate，而不是散在 final answer 里：

    {
      "closeoutId": "comp_closeout_agent_course_389",
      "contractId": "ffc_agent_course_389",
      "ledgerId": "comp_ledger_agent_course_389",
      "requiredRealityChecks": [
        "telegram_message_present",
        "remote_git_contains_commit",
        "readme_links_lesson",
        "tools_topic_recorded",
        "memory_receipt_matches"
      ],
      "releaseBarriers": [
        "next_lesson_topic_selection",
        "public_completion_report"
      ],
      "decision": "pending"
    }

它要输出的不是布尔值，而是一个可审计决策：

    allow_close
    hold_reconcile
    hold_manual_review
    reopen_compensation
    release_partial_with_warning

最重要的原则：

    all_steps_succeeded != closeout_allowed
    closeout_allowed == ledger_terminal + reality_matched + evidence_complete + barriers_released

## 3. Reality Check 清单

Closeout Gate 至少检查四层。

第一层：ledger 终态。

    no pending
    no unknown
    no failed_safe
    no failed_unsafe
    every succeeded step has evidenceHash

第二层：外部现实。

    messageId -> remote message exists
    commitSha -> origin/main contains commit
    file paths -> current working tree contains expected content
    published URL / ticket / deployment -> remote status matches expected state

第三层：证据一致性。

    evidenceHash matches payload
    receipt references real externalRef
    README topic == lesson title == TOOLS topic
    memory summary does not claim more than evidence proves

第四层：下游释放。

    blocked dependent actions can now proceed
    temporary freeze / barrier is removed
    watch window or follow-up probe is scheduled when required
    closeout receipt is immutable

只要其中一层不一致，就不是“再试一次最终回复”，而是进入 hold_reconcile 或 reopen_compensation。

## 4. learn-claude-code：文件式 closeout gate

learn-claude-code 的 s08_background_tasks.py 已经展示了 BackgroundManager：后台任务状态在 tasks 里，完成后把通知塞进 notification queue。

教学版可以继续保持简单：把 CompensationLedger 和 CloseoutGate 都落成 JSON 文件。

    from dataclasses import dataclass
    from pathlib import Path
    import json

    TERMINAL_STEP = {"succeeded", "failed_unsafe"}

    @dataclass
    class RealityCheck:
        name: str
        ok: bool
        evidence: str
        reason: str = ""

    def load_json(path: Path) -> dict:
        return json.loads(path.read_text())

    def gate_closeout(ledger: dict, checks: list[RealityCheck]) -> dict:
        non_terminal = [
            step["stepId"]
            for step in ledger["steps"]
            if step["status"] not in TERMINAL_STEP
        ]
        unsafe = [
            step["stepId"]
            for step in ledger["steps"]
            if step["status"] == "failed_unsafe"
        ]
        failed_checks = [check for check in checks if not check.ok]

        if unsafe:
            decision = "hold_manual_review"
        elif non_terminal:
            decision = "hold_reconcile"
        elif failed_checks:
            decision = "reopen_compensation"
        else:
            decision = "allow_close"

        return {
            "decision": decision,
            "nonTerminalSteps": non_terminal,
            "failedChecks": [c.name for c in failed_checks],
            "evidence": {c.name: c.evidence for c in checks},
        }

    def write_closeout_receipt(path: Path, receipt: dict) -> None:
        path.write_text(json.dumps(receipt, ensure_ascii=False, indent=2))

注意这里的 closeout gate 不执行补偿动作。它只判断“能不能关闭”。如果需要补动作，就返回 reopen_compensation，让 compensation runner 按 ledger 继续推进。

这能防止一个常见事故：finalizer 发现缺口后，自己临场发消息、改文件、push。关闭器不能既当法官又当施工队。

## 5. pi-mono：把 closeout 放在事件流收口处

pi-mono 的 mom 包里，工具统一从 createMomTools(executor) 暴露：

    export function createMomTools(executor: Executor): AgentTool<any>[] {
        return [
            createReadTool(executor),
            createBashTool(executor),
            createEditTool(executor),
            createWriteTool(executor),
            attachTool,
        ];
    }

事件处理在 EventsWatcher 里区分 immediate / one-shot / periodic：

    switch (event.type) {
        case "immediate":
            this.handleImmediate(filename, event);
            break;
        case "one-shot":
            this.handleOneShot(filename, event);
            break;
        case "periodic":
            this.handlePeriodic(filename, event);
            break;
    }

生产版可以在事件 worker 的最终阶段加 CloseoutReconciliationGate：

    type CloseoutDecision =
      | "allow_close"
      | "hold_reconcile"
      | "hold_manual_review"
      | "reopen_compensation";

    type RealityCheck = {
      name: string;
      ok: boolean;
      evidenceRef: string;
      reason?: string;
    };

    type CloseoutReceipt = {
      closeoutId: string;
      ledgerId: string;
      decision: CloseoutDecision;
      checks: RealityCheck[];
      releaseBarriers: string[];
      createdAt: string;
    };

    class CloseoutReconciliationGate {
      constructor(
        private ledgerStore: CompensationLedgerStore,
        private reality: RealityProbeRegistry,
        private receiptStore: CloseoutReceiptStore,
      ) {}

      async evaluate(closeoutId: string, ledgerId: string): Promise<CloseoutReceipt> {
        const ledger = await this.ledgerStore.get(ledgerId);
        const checks = await Promise.all(
          ledger.requiredRealityChecks.map((name) =>
            this.reality.run(name, ledger),
          ),
        );

        const decision = decideCloseout(ledger, checks);
        const receipt = {
          closeoutId,
          ledgerId,
          decision,
          checks,
          releaseBarriers: ledger.releaseBarriers,
          createdAt: new Date().toISOString(),
        };

        await this.receiptStore.append(receipt);
        return receipt;
      }
    }

这样每个 periodic cron、one-shot reminder、外部发布任务，在“看起来完成”以后，都要过同一个关闭闸门。业务逻辑不用到处复制“检查 messageId、查 git remote、验文件链接”的代码。

## 6. OpenClaw 课程 cron 的实战 closeout

拿我们现在这类 Agent 开发课程任务举例，真正的完成条件不是“写完课文”。

应该生成这样的 closeout checklist：

    closeoutId: agent_course_lesson_389
    ledger:
      - lesson_file_written
      - readme_updated
      - tools_updated
      - telegram_sent
      - git_commit_created
      - git_push_verified
      - memory_logged
    realityChecks:
      - lesson file exists and title is 389
      - README contains lessons/389-compensation-closeout-reconciliation-gate.md
      - TOOLS contains exact taught topic
      - Telegram send returned messageId
      - origin/main hash == local commit hash
      - git diff --check passes
      - git status has no unintended changes

如果 git push 返回超时，不能立刻重推，也不能宣布失败。应该先：

    git ls-remote origin main
      -> contains commit: mark git_push_verified succeeded
      -> does not contain commit: retry push with same commit
      -> auth error: hold_manual_review

如果 Telegram 发群成功但本地 memory 写失败，不能重发 Telegram。应该只补 memory receipt，并把原 messageId 写进去。

## 7. 释放屏障：什么时候允许下游继续

Closeout Gate 通过后，才释放下游屏障。

课程系统里的下游动作包括：

- 下一次 cron 选题可以把第 389 课视为已讲；
- README 目录可以作为公开索引；
- TOOLS 已讲内容可以用于去重；
- memory 可以作为后续审计依据；
- 最终汇报可以说“已发群、已推送”。

如果 closeout 失败，下游动作要被限制：

    hold_reconcile:
      可以继续只读对账，不能发最终成功报告

    reopen_compensation:
      可以执行 ledger 指定的缺口补偿步骤，不能重放已成功步骤

    hold_manual_review:
      停止自动副作用，保留草稿、证据和下一步建议

这就是 Outcome Barrier 的延伸：屏障不只挡住业务动作，也挡住 Agent 自己的“完成宣告”。

## 8. 常见坑

坑 1：把 closeout 写成最后一段自然语言。

自然语言总结不可重放，不能被下次任务读取，也不能驱动屏障释放。关闭结果要写成 receipt。

坑 2：只检查本地状态，不查外部现实。

git commit 本地存在不代表 push 成功；message tool 返回文本不代表群里真的能找到；README 更新不代表链接有效。

坑 3：发现缺口后 finalizer 自己修。

finalizer 应该返回 reopen_compensation，由 runbook + ledger 执行补偿。否则关闭器会绕过演练、幂等键和权限策略。

坑 4：释放屏障没有收据。

下游继续运行前，要能回答“谁基于哪些 checks 释放了哪个 barrier”。没有 release receipt，事故复盘时只能靠猜。

## 9. 最小落地清单

给你的 Agent 系统加补偿关闭闸门，可以先做 6 件事：

- 每个补偿 contract 声明 requiredRealityChecks；
- 每个 ledger step 成功时写 externalRef 和 evidenceHash；
- closeout 只读检查，不直接执行补偿；
- failed check 映射到 hold_reconcile / reopen_compensation / manual_review；
- allow_close 时写 CloseoutReceipt；
- 下游动作只认 CloseoutReceipt，不认“步骤全绿”。

## 10. 总结

Compensation Ledger 解决“补偿怎么安全重试”。

Closeout Reconciliation Gate 解决“什么时候能证明补偿真的结束”。

成熟 Agent 的恢复闭环，不是把步骤跑完就说完事，而是把本地账本、外部现实、证据链和下游屏障全部对齐，再用关闭收据释放后续动作。这样系统以后回看时，不只知道 Agent 做了什么，还知道为什么当时可以宣布恢复完成。
