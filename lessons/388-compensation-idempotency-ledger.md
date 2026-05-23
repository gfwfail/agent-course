# 388. Agent 补偿动作幂等账本与分段重试（Compensation Idempotency Ledger & Partial Retry）

上一课讲了 Compensation Runbook + Drill Gate：补偿动作要结构化，还要提前演练。

今天继续补一个生产系统最容易踩的坑：**补偿动作不能靠“再跑一遍 runbook”来重试。**

原因很简单：补偿 runbook 里面通常有多个副作用步骤。

- 第一步发了更正消息；
- 第二步更新了 README；
- 第三步提交并 push；
- 第四步写 memory；
- 第五步释放下游屏障。

如果第三步超时，Agent 不知道前两步是否成功，这时直接从头再跑，就可能重复发更正消息、重复创建 commit、重复写外部状态。

一句话：**补偿动作必须有 Compensation Ledger；重试时按 step outcome 续跑，而不是整条 runbook 重放。**

## 1. 为什么普通 retry 不够

普通工具调用失败时，retry 通常是这样：

    call tool
      -> timeout
      -> call tool again

这对只读工具还可以，对补偿动作很危险。

补偿动作的特点是：

1. 它本身就在修事故，风险比普通动作高；
2. 它经常跨多个系统，比如 Telegram、Git、memory、数据库；
3. 每一步可能部分成功，但工具返回值丢了；
4. 下一步是否能执行，依赖前一步的真实外部状态；
5. 重复执行可能制造第二个事故。

所以补偿重试要从“函数重试”升级为“账本驱动的分段恢复”。

最小原则：

    already_succeeded -> skip
    unknown           -> reconcile first
    failed_safe       -> retry this step only
    failed_unsafe     -> block and require repair

## 2. CompensationLedger 的最小结构

可以把每个补偿 runbook 的执行写成一条 ledger：

    {
      "ledgerId": "comp_ledger_agent_course_388",
      "contractId": "ffc_agent_course_388",
      "runbookId": "rb_telegram_git_memory_correction_v1",
      "runbookHash": "sha256:...",
      "status": "running",
      "steps": [
        {
          "stepId": "send_correction",
          "idempotencyKey": "correction:effect_12602:hash_abc",
          "status": "succeeded",
          "attempts": 1,
          "externalRef": "telegram:-5115329245:12610",
          "evidenceHash": "sha256:..."
        },
        {
          "stepId": "commit_fix",
          "idempotencyKey": "gitfix:agent-course:388:hash_def",
          "status": "unknown",
          "attempts": 1,
          "lastError": "push timeout",
          "reconcileWith": "git ls-remote origin main"
        }
      ],
      "releaseBarrier": "blocked_until_all_terminal"
    }

关键字段：

- stepId：必须来自 runbook，不能由 LLM 临场发明；
- idempotencyKey：同一补偿意图重复执行时必须稳定；
- externalRef：外部世界的引用，比如 messageId、commit sha、ticket id；
- status：区分 succeeded、failed_safe、unknown、failed_unsafe；
- reconcileWith：unknown 时先对账，不能盲重试；
- evidenceHash：证明这一步确实完成，不只看工具文本。

## 3. 分段重试算法

补偿执行器不应该说“runbook failed, retry all”。

它应该按 ledger 逐步推进：

    def resume_compensation(ledger, runbook):
        for step in runbook.steps:
            entry = ledger.get(step.step_id)

            if entry.status == "succeeded":
                continue

            if entry.status == "unknown":
                outcome = reconcile(step, entry)
                ledger.update(step.step_id, outcome)
                if outcome.status == "succeeded":
                    continue
                if outcome.status == "failed_unsafe":
                    return block("manual repair required")

            if entry.status in ["pending", "failed_safe"]:
                result = execute_once(step, idempotency_key=entry.idempotency_key)
                ledger.update(step.step_id, classify(result))

        return release_if_all_terminal(ledger)

这里的重点是 execute_once。

即使进入 retry，也只能执行当前 step 一次，然后重新写 ledger。不要在工具内部无限 retry，更不要让 LLM 一边猜一边循环。

## 4. learn-claude-code：把 BackgroundManager 变成补偿续跑器

learn-claude-code 的 s08_background_tasks.py 里，BackgroundManager 用 task_id 记录后台任务状态：

    class BackgroundManager:
        def __init__(self):
            self.tasks = {}
            self._notification_queue = []
            self._lock = threading.Lock()

        def run(self, command: str) -> str:
            task_id = str(uuid.uuid4())[:8]
            self.tasks[task_id] = {"status": "running", "result": None, "command": command}
            thread = threading.Thread(
                target=self._execute, args=(task_id, command), daemon=True
            )
            thread.start()
            return f"Background task {task_id} started: {command[:80]}"

教学版可以沿用这个结构，但 task 里不要只放 command，要放 ledger：

    TERMINAL = {"succeeded", "failed_unsafe"}

    class CompensationLedger:
        def __init__(self, ledger_id: str, steps: list[dict]):
            self.ledger_id = ledger_id
            self.steps = {
                step["stepId"]: {
                    "status": "pending",
                    "attempts": 0,
                    "idempotencyKey": step["idempotencyKey"],
                    "externalRef": None,
                }
                for step in steps
            }

        def next_step(self):
            for step_id, entry in self.steps.items():
                if entry["status"] not in TERMINAL:
                    return step_id, entry
            return None, None

    def resume_once(ledger: CompensationLedger, runbook: dict) -> str:
        step_id, entry = ledger.next_step()
        if step_id is None:
            return "compensation completed"

        if entry["status"] == "unknown":
            outcome = reconcile(step_id, entry)
            entry.update(outcome)
            return f"reconciled {step_id}: {entry['status']}"

        entry["attempts"] += 1
        result = execute_step_once(runbook[step_id], entry["idempotencyKey"])
        entry.update(classify_result(result))
        return f"executed {step_id}: {entry['status']}"

这样 BackgroundManager 每次醒来只推进一个 step。进程重启后，只要 ledger 落盘，就能从最后一个非终态步骤继续，而不是把整条补偿链重跑一遍。

## 5. pi-mono：在事件流外层做 ledger middleware

pi-mono 的 mom 包把事件、工具和 Agent runner 分得比较清楚。

工具集合在 packages/mom/src/tools/index.ts 里统一创建：

    export function createMomTools(executor: Executor): AgentTool<any>[] {
        return [
            createReadTool(executor),
            createBashTool(executor),
            createEditTool(executor),
            createWriteTool(executor),
            attachTool,
        ];
    }

事件系统在 packages/mom/src/events.ts 里处理 one-shot / periodic / immediate：

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

这给生产版一个很清楚的位置：不要把幂等逻辑塞进每个 tool；在 tool 外层或事件 worker 外层加 CompensationLedgerMiddleware。

    type StepStatus = "pending" | "succeeded" | "unknown" | "failed_safe" | "failed_unsafe";

    type LedgerStep = {
      stepId: string;
      idempotencyKey: string;
      status: StepStatus;
      attempts: number;
      externalRef?: string;
      evidenceHash?: string;
      lastError?: string;
    };

    class CompensationLedgerMiddleware {
      async run(contractId: string, runbook: Runbook) {
        const ledger = await this.store.loadOrCreate(contractId, runbook);

        for (const step of runbook.steps) {
          const entry = ledger.step(step.stepId);

          if (entry.status === "succeeded") continue;

          if (entry.status === "unknown") {
            const reconciled = await this.reconciler.check(step, entry);
            await this.store.update(contractId, step.stepId, reconciled);
            if (reconciled.status === "succeeded") continue;
            if (reconciled.status === "failed_unsafe") return this.block(ledger);
          }

          const result = await this.executor.executeOnce(step, entry.idempotencyKey);
          await this.store.update(contractId, step.stepId, this.classify(result));
          return this.scheduleResume(contractId);
        }

        return this.releaseBarrier(contractId);
      }
    }

注意 scheduleResume：每次只推进一步，然后交还控制权。这样可以避免一个补偿任务长期占住 agent turn，也能让外部事件、人工审批、对账结果插进来。

## 6. OpenClaw 课程 Cron 的真实落点

这门课的 cron 本身就是一个很好的例子。

一次课程发布至少包含这些外部副作用：

1. 写 lessons/388-xxx.md；
2. 更新 README.md；
3. 更新 TOOLS.md 已讲内容；
4. 发 Telegram 到 Rust 学习小组；
5. git add / commit / push；
6. 写 memory/YYYY-MM-DD.md；
7. 验证远端 main 包含 commit。

如果 Telegram 发成功，但 git push 失败，不能重发 Telegram。

如果 commit 成功但 push timeout，不能直接再 commit 一次；要先：

    git status
    git log -1 --oneline
    git ls-remote origin main

确认远端有没有这个 commit。

这就是课程 cron 的 ledger 化：

    {
      "stepId": "telegram_publish",
      "status": "succeeded",
      "externalRef": "messageId:12610"
    }

    {
      "stepId": "git_push",
      "status": "unknown",
      "reconcileWith": "git ls-remote origin main"
    }

    {
      "stepId": "memory_log",
      "status": "pending",
      "blockedBy": ["git_push"]
    }

这样 Agent 恢复时不会凭记忆乱补，而是按账本继续。

## 7. 实战检查清单

设计补偿动作时，至少检查这 8 个问题：

1. 每个 step 有没有稳定 stepId？
2. 幂等键是否由业务事实生成，而不是随机数？
3. 每个外部副作用有没有 externalRef？
4. timeout 后能不能对账？
5. unknown 状态是否禁止直接重试？
6. succeeded step 是否永远不会再次执行？
7. ledger 是否落盘，进程重启后能恢复？
8. release barrier 是否只在所有关键 step terminal 后释放？

成熟 Agent 的补偿不是“失败了再努力一次”，而是带着账本恢复：知道哪些已经做过、哪些需要对账、哪些可以重试、哪些必须停下来让人修。
