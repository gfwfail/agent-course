# 387. Agent 补偿动作 Runbook 与演练闸门（Compensation Runbook & Drill Gate）

上一课讲了 Forward-Fix Contract：不可逆副作用已经发生时，不要假装 rollback，而是用补偿动作把系统带回可信状态。

今天继续补上一个工程上经常漏掉的环节：**补偿动作不能等事故发生时才第一次运行。**

很多团队写了很漂亮的补偿方案：

- 发错 Telegram 消息后，发送更正通知；
- push 错 commit 后，创建 revert 或修复 commit；
- memory 污染后，写 tombstone 并阻止继续检索；
- 部署错版本后，发布补丁并验证流量回到正确版本；
- 订单状态错写后，补发对账事件并通知下游。

但真出事时才发现：

- runbook 缺参数；
- 补偿工具权限不够；
- 幂等键生成不稳定；
- 证据读取路径坏了；
- 演练环境和生产环境差太远；
- 补偿动作本身又制造了第二个外部副作用事故。

一句话：**Forward-Fix Contract 定义“要修成什么样”，Compensation Runbook + Drill Gate 证明“真的会修，而且不会修出第二个坑”。**

## 1. Runbook 不是自然语言 checklist

自然语言 checklist 适合人读，但不适合 Agent 自动执行。

错误写法：

    如果发错消息，去群里发一条更正，然后更新记录。

这句话对 Agent 来说太松：

- “发错消息”怎么判断？
- 更正消息发到哪个 target？
- 原 messageId 如何引用？
- 可以发几次？
- 发完后如何验证？
- 验证失败能不能重试？
- 哪些下游动作必须继续阻断？

可执行 Runbook 至少要结构化：

    {
      "runbookId": "rb_agent_course_telegram_correction_v1",
      "appliesTo": {
        "effectKind": "outbound_message",
        "classification": "compensatable",
        "channel": "telegram"
      },
      "inputs": ["originalMessageId", "target", "correctionText", "sourceEffectId"],
      "preconditions": [
        "reconciliation.status == payload_mismatch",
        "target == originalEffect.target",
        "correctionTextHash == approvedArtifact.hash"
      ],
      "steps": [
        {
          "stepId": "send_correction",
          "tool": "message.send",
          "idempotencyKeyTemplate": "correction:{sourceEffectId}:{correctionTextHash}",
          "maxAttempts": 1,
          "evidence": ["correctionMessageId", "sentPayloadHash"]
        },
        {
          "stepId": "verify_visible",
          "tool": "message.poll",
          "mode": "read_only",
          "evidence": ["visible", "observedPayloadHash"]
        }
      ],
      "successCriteria": [
        "correctionMessageId exists",
        "observedPayloadHash == sentPayloadHash",
        "releaseBarrier outcome == forward_fixed"
      ],
      "forbiddenActions": ["delete_without_approval", "send_unbounded_retry"]
    }

关键点：Runbook 既描述步骤，也描述边界。

补偿动作本身也是副作用，所以它必须比普通动作更保守。

## 2. Drill Gate：补偿前先演练

Drill Gate 的目标不是“模拟一切”，而是定期证明 runbook 仍然可执行。

最小演练可以分三层：

    static_check
      校验 runbook schema、必填 input、idempotencyKeyTemplate、successCriteria、forbiddenActions。

    dry_run
      不执行外部写，只生成 planned steps、argsHash、payloadHash、permission decision。

    sandbox_probe
      在沙箱 target 或 fake provider 上真实执行补偿流程，验证证据、幂等、屏障释放。

生产事故现场不一定有时间做完整 sandbox_probe，但系统平时必须周期性做。

如果最近没有通过演练，高风险补偿不要自动执行：

    drillReceipt.status == pass
      -> allow automatic compensation within scope

    drillReceipt.status == stale
      -> require approval or sandbox probe first

    drillReceipt.status == fail
      -> freeze compensation, open repair ticket

## 3. DrillReceipt：演练也要留收据

演练结果不能只写一句“OK”。

建议记录：

    {
      "receiptId": "drill_rb_agent_course_telegram_correction_20260523",
      "runbookId": "rb_agent_course_telegram_correction_v1",
      "runbookHash": "sha256:...",
      "mode": "sandbox_probe",
      "scope": {
        "channel": "telegram",
        "target": "sandbox:agent-course"
      },
      "checkedAt": "2026-05-23T18:30:00+10:00",
      "results": {
        "schema": "pass",
        "idempotency": "pass",
        "permission": "pass",
        "evidence": "pass",
        "barrierRelease": "pass"
      },
      "validUntil": "2026-05-30T18:30:00+10:00",
      "allowedProductionScope": {
        "effectKind": "outbound_message",
        "maxRisk": "medium",
        "requiresSameTarget": true
      }
    }

注意两个字段：

- runbookHash：runbook 改了，旧演练收据不能继续证明新 runbook；
- allowedProductionScope：演练通过的是某个范围，不是给所有补偿动作发免死金牌。

## 4. learn-claude-code：给 BackgroundManager 加补偿演练队列

learn-claude-code 的 s08_background_tasks.py 有一个很适合教学的 BackgroundManager：

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

这个结构可以直接扩成“补偿演练任务”。

教学版不要真的发 Telegram，只跑 dry-run 和 fake provider：

    from dataclasses import dataclass
    from typing import Literal
    import hashlib
    import json
    import time

    Mode = Literal["static_check", "dry_run", "sandbox_probe"]

    @dataclass(frozen=True)
    class CompensationRunbook:
        runbook_id: str
        required_inputs: list[str]
        steps: list[dict]
        success_criteria: list[str]
        forbidden_actions: list[str]

    def stable_hash(value: dict) -> str:
        raw = json.dumps(value, sort_keys=True, separators=(",", ":")).encode()
        return "sha256:" + hashlib.sha256(raw).hexdigest()

    def drill_runbook(runbook: CompensationRunbook, mode: Mode, sample_input: dict) -> dict:
        missing = [key for key in runbook.required_inputs if key not in sample_input]
        forbidden = [
            step["tool"] for step in runbook.steps
            if step.get("tool") in runbook.forbidden_actions
        ]

        if missing or forbidden:
            return {
                "runbookId": runbook.runbook_id,
                "status": "fail",
                "missingInputs": missing,
                "forbiddenSteps": forbidden,
            }

        planned_steps = []
        for step in runbook.steps:
            args = {
                "stepId": step["stepId"],
                "tool": step["tool"],
                "input": sample_input,
            }
            planned_steps.append({
                "stepId": step["stepId"],
                "tool": step["tool"],
                "argsHash": stable_hash(args),
                "mode": mode,
            })

        return {
            "runbookId": runbook.runbook_id,
            "runbookHash": stable_hash(runbook.__dict__),
            "status": "pass",
            "mode": mode,
            "plannedSteps": planned_steps,
            "validUntil": int(time.time()) + 7 * 24 * 3600,
        }

这里的核心不是 fake provider 多真实，而是训练 Agent 形成习惯：

1. 补偿步骤先变成结构化 plan；
2. 每一步都有 argsHash；
3. 演练通过才允许自动补偿；
4. 演练收据过期后自动降级到 approval。

## 5. pi-mono：用 EventBus 挂 Drill Worker

pi-mono 的 coding-agent 里有一个轻量 EventBus：

    export interface EventBus {
      emit(channel: string, data: unknown): void;
      on(channel: string, handler: (data: unknown) => void): () => void;
    }

    export function createEventBus(): EventBusController {
      const emitter = new EventEmitter();
      return {
        emit: (channel, data) => {
          emitter.emit(channel, data);
        },
        on: (channel, handler) => {
          const safeHandler = async (data: unknown) => {
            try {
              await handler(data);
            } catch (err) {
              console.error("Event handler error (" + channel + "):", err);
            }
          };
          emitter.on(channel, safeHandler);
          return () => emitter.off(channel, safeHandler);
        },
        clear: () => {
          emitter.removeAllListeners();
        },
      };
    }

这类事件总线非常适合挂一个 CompensationDrillWorker：

    type DrillReceipt = {
      receiptId: string;
      runbookId: string;
      runbookHash: string;
      mode: "static_check" | "dry_run" | "sandbox_probe";
      status: "pass" | "fail";
      validUntil: string;
      allowedProductionScope: {
        effectKind: string;
        maxRisk: "low" | "medium" | "high";
      };
    };

    class CompensationDrillWorker {
      constructor(
        private bus: EventBus,
        private runbooks: RunbookStore,
        private receipts: DrillReceiptStore,
        private sandbox: CompensationSandbox,
      ) {}

      start() {
        this.bus.on("forward_fix.contract_created", async (event) => {
          const runbook = await this.runbooks.findFor(event);
          const latest = await this.receipts.latest(runbook.id);

          if (!latest || latest.status !== "pass" || latest.validUntil < new Date().toISOString()) {
            const receipt = await this.sandbox.drill(runbook, "dry_run", event.sampleInput);
            await this.receipts.save(receipt);
          }

          this.bus.emit("compensation.drill_checked", {
            contractId: event.contractId,
            runbookId: runbook.id,
          });
        });
      }
    }

工程上建议把 Drill Worker 做成旁路组件，而不是塞进工具内部。

工具只负责执行；Runbook Store 负责版本；Drill Worker 负责证明补偿路径可用；ForwardFixOrchestrator 决定是否执行真实补偿。

## 6. OpenClaw 课程 cron 的实战映射

拿我们这个 Agent 开发课程 cron 举例，最容易出问题的补偿路径有三个：

    Telegram 课程发错
      runbook: send_correction_message
      drill: sandbox group 或 dry-run payload hash
      evidence: correctionMessageId, originalMessageId, payloadHash

    Git push 内容错
      runbook: append_fix_commit_or_revert
      drill: local temp repo dry-run
      evidence: fixCommitSha, remoteHead, lsRemoteContainsFix

    TOOLS 已讲内容写错
      runbook: patch_topic_index_and_memory
      drill: local file fixture
      evidence: rg topic, git diff, memory entry

本次课程发布前，最低限度的“人工版 Drill Gate”就是：

1. 查 README 最后一课，确认新编号是 387；
2. 查 TOOLS 已讲内容，确认主题没重复；
3. 写文件后跑 git diff --check；
4. Telegram 发送后拿 messageId；
5. push 后用 git ls-remote 验证远端 main 包含 commit；
6. 最后把 messageId、commit、主题写入 daily memory。

这其实就是一个手工 DrillReceipt。

以后自动化程度更高时，可以把这些检查固化成 agent-course 的 runbook：

    runbookId: rb_agent_course_publish_lesson_v1
    compensationRunbooks:
      - rb_agent_course_send_correction_v1
      - rb_agent_course_append_fix_commit_v1
      - rb_agent_course_patch_topic_index_v1
    drillRequiredBefore:
      - telegram_send
      - git_push
      - memory_update

这样课程 cron 不只是会发课，还知道“发错后怎么证明能修”。

## 7. 实战建议

补偿 Runbook 最容易踩的坑：

- 只写自然语言，不写 machine-readable input 和 successCriteria；
- 不记录 runbookHash，导致旧演练证明新步骤；
- sandbox 太假，完全绕过权限、幂等和证据路径；
- 演练只测 happy path，不测 duplicate、payload_mismatch、permission denied；
- 补偿动作没有 forbiddenActions，事故现场被 Agent 自由发挥；
- 演练收据不过期，半年没跑还继续自动补偿。

一个成熟做法：

    ForwardFixContract created
      -> select CompensationRunbook
      -> check latest DrillReceipt
      -> if pass and in scope: execute compensation
      -> if stale: dry-run or approval
      -> if fail: freeze and open repair ticket
      -> after compensation: reconciliation + release barrier

## 8. 小结

Forward-Fix Contract 解决“补偿目标是什么”。

Compensation Runbook 解决“补偿步骤怎么执行”。

Drill Gate 解决“这些步骤最近有没有被证明还能执行”。

Agent 进入生产以后，最危险的不是它不会修，而是它临场 improvisation。补偿路径要像消防演练一样提前跑过、留收据、限定范围。这样真正出事故时，Agent 执行的是经过验证的恢复流程，而不是在压力下生成一段看似合理的新动作。
