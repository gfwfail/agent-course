# 383. Agent 外部副作用预览与干运行收据（External Side Effect Preview & Dry-run Receipt）

上一课讲了 Cold Evidence Retrieval Drill：冷证据放进仓库以后，还要定期取回、验 hash、验 schema、验权限租约。

今天换一个更贴近日常执行的问题：Agent 要发 Telegram、push Git、部署、发邮件、创建订单、调用支付接口之前，怎么证明自己知道“马上要对外做什么”？

很多事故不是模型不会调用工具，而是它太会调用工具了：

- 还没确认目标群，就把消息发出去了；
- git push 前没看 diff，推了不该推的文件；
- 部署前没说明环境和版本，结果发到 production；
- 邮件内容里混了内部证据；
- retry 时重复创建了外部资源；
- 用户问“看看能不能做”，Agent 直接做了。

所以今天讲 External Side Effect Preview & Dry-run Receipt。

一句话：**所有外部副作用，在真正执行前先生成预览和干运行收据；收据要能被 policy、用户、审计和后续执行共同验证。**

## 1. 什么叫外部副作用

外部副作用不是“危险操作”的同义词。只要动作离开了当前进程、当前文件和当前私有上下文，就算外部副作用。

常见类型：

- outbound_message：Telegram、Slack、Discord、Email；
- remote_write：Git push、GitHub comment、issue edit；
- deployment：发布、重启、迁移；
- payment_or_order：付款、退款、下单、发货；
- data_export：上传文件、共享链接、发报表；
- scheduler_change：新增 cron、取消任务、改触发时间。

这些动作有一个共同点：做完以后不能只靠本地回滚恢复。即使能撤回，也已经产生了外部痕迹。

所以成熟 Agent 不能只问“工具参数合法不合法”，还要问：

- 目标是谁？
- 将要发送/写入/部署什么？
- 数据敏感度是多少？
- 是否会重复执行？
- 是否有用户或 policy 授权？
- 执行后的证据怎么绑定回这次预览？

## 2. PreviewSpec：把“将要做什么”结构化

不要让 Agent 只在自然语言里说“我准备发一条消息”。预览必须结构化，后续工具执行才能校验它。

一个最小 PreviewSpec：

    {
      "previewId": "preview_20260523_0630_383",
      "actionType": "outbound_message",
      "target": {
        "channel": "telegram",
        "chatId": "-5115329245",
        "audience": "Rust学习小组"
      },
      "payloadSummary": "发布第383课：外部副作用预览与干运行收据",
      "payloadHash": "sha256:...",
      "risk": "external_write",
      "sensitivity": "public_course_content",
      "idempotencyKey": "agent-course-383-telegram",
      "requiresApproval": false,
      "expiresAt": "2026-05-23T06:45:00+10:00"
    }

注意几个字段：

- payloadSummary 给人看；
- payloadHash 给机器校验；
- idempotencyKey 防重复发送；
- expiresAt 防止旧预览被 replay；
- sensitivity 决定能不能外发 raw 内容；
- requiresApproval 不是硬编码，由 policy 根据 risk/target/sensitivity 算出来。

## 3. DryRunReceipt：干运行不是打印日志

很多系统的 dry-run 只是“不执行”。这不够。

Agent 的 dry-run 要产出 DryRunReceipt，证明系统已经完成这些检查：

- 工具参数解析完成；
- 目标和 payload 已规范化；
- policy 已评估；
- 敏感信息已扫描；
- 幂等键已生成；
- 执行计划已冻结；
- 后续真实执行必须匹配这张收据。

收据示例：

    {
      "receiptId": "dryrun_383_telegram_001",
      "previewId": "preview_20260523_0630_383",
      "decision": "allowed",
      "policyVersion": "egress-policy@2026-05-20",
      "normalizedArgsHash": "sha256:args...",
      "payloadHash": "sha256:payload...",
      "requiredObligations": [
        "record_message_id",
        "append_memory_summary",
        "verify_remote_commit"
      ],
      "validUntil": "2026-05-23T06:45:00+10:00"
    }

真正执行时，工具层要重新计算 argsHash/payloadHash。只要和 dry-run receipt 不一致，就必须拒绝执行，重新预览。

## 4. learn-claude-code：教学版干运行闸门

教学版先做成纯函数。输入 action request，输出 preview 和 receipt；执行函数必须带 receipt。

    from dataclasses import dataclass
    from hashlib import sha256
    import json
    import time

    @dataclass(frozen=True)
    class ActionRequest:
        action_type: str
        target: dict
        payload: dict
        actor: str
        risk: str

    def canonical_hash(value) -> str:
        encoded = json.dumps(
            value,
            sort_keys=True,
            separators=(",", ":"),
            ensure_ascii=False,
        ).encode("utf-8")
        return "sha256:" + sha256(encoded).hexdigest()

    def build_preview(request: ActionRequest) -> dict:
        payload_hash = canonical_hash(request.payload)
        args_hash = canonical_hash({
            "actionType": request.action_type,
            "target": request.target,
            "payloadHash": payload_hash,
            "actor": request.actor,
        })

        return {
            "previewId": "preview_" + args_hash[-12:],
            "actionType": request.action_type,
            "target": request.target,
            "payloadSummary": request.payload.get("summary", ""),
            "payloadHash": payload_hash,
            "normalizedArgsHash": args_hash,
            "risk": request.risk,
            "expiresAt": int(time.time()) + 900,
        }

    def dry_run(request: ActionRequest, policy) -> dict:
        preview = build_preview(request)
        decision = policy.evaluate(preview)

        return {
            "receiptId": "dryrun_" + preview["previewId"].removeprefix("preview_"),
            "preview": preview,
            "decision": decision["action"],
            "reasonCodes": decision["reasonCodes"],
            "requiredObligations": decision.get("obligations", []),
            "validUntil": preview["expiresAt"],
        }

这个函数的重点不是 hash 本身，而是把“将要执行什么”冻结成机器可比较的结构。

## 5. 执行前复核：receipt 必须匹配当前动作

执行函数不要相信 LLM 说“我已经 dry-run 过了”。它要自己验 receipt。

    def execute_with_receipt(request: ActionRequest, receipt: dict, tool):
        if receipt["decision"] != "allowed":
            raise ValueError("dry_run_not_allowed")

        if time.time() > receipt["validUntil"]:
            raise ValueError("dry_run_receipt_expired")

        current_preview = build_preview(request)
        expected = receipt["preview"]

        if current_preview["normalizedArgsHash"] != expected["normalizedArgsHash"]:
            raise ValueError("dry_run_args_changed")

        if current_preview["payloadHash"] != expected["payloadHash"]:
            raise ValueError("dry_run_payload_changed")

        result = tool.execute(request.target, request.payload)

        return {
            "receiptId": receipt["receiptId"],
            "executionStatus": "executed",
            "externalId": result["externalId"],
            "obligations": receipt["requiredObligations"],
        }

这样可以挡住一个很常见的问题：预览时是 A，执行时参数偷偷变成 B。

## 6. pi-mono：SideEffectPreviewMiddleware

生产版更适合放在 Tool Dispatch 外层，让所有外部写工具都走同一条路径。

    type SideEffectKind =
      | "outbound_message"
      | "remote_write"
      | "deployment"
      | "payment_or_order"
      | "data_export";

    type DryRunReceipt = {
      receiptId: string;
      previewId: string;
      decision: "allowed" | "requires_approval" | "denied";
      normalizedArgsHash: string;
      payloadHash: string;
      validUntil: string;
      obligations: string[];
    };

    async function previewExternalSideEffect(call: ToolCall): Promise<DryRunReceipt> {
      const normalized = normalizeToolCall(call);
      const preview = {
        previewId: createId("preview"),
        kind: call.meta.sideEffectKind as SideEffectKind,
        target: normalized.target,
        payloadSummary: summarizePayload(normalized.payload),
        payloadHash: canonicalHash(normalized.payload),
        normalizedArgsHash: canonicalHash(normalized),
        sensitivity: classifySensitivity(normalized.payload),
        idempotencyKey: buildIdempotencyKey(call, normalized),
      };

      const decision = await policy.evaluate("external_side_effect.preview", preview);

      return receiptStore.append({
        receiptId: createId("dryrun"),
        previewId: preview.previewId,
        decision: decision.action,
        normalizedArgsHash: preview.normalizedArgsHash,
        payloadHash: preview.payloadHash,
        validUntil: addMinutes(new Date(), 15).toISOString(),
        obligations: decision.obligations,
      });
    }

执行阶段：

    async function executeExternalSideEffect(call: ToolCall, receipt: DryRunReceipt) {
      const normalized = normalizeToolCall(call);

      assertNotExpired(receipt.validUntil);
      assertEqual(receipt.normalizedArgsHash, canonicalHash(normalized));
      assertEqual(receipt.payloadHash, canonicalHash(normalized.payload));

      if (receipt.decision === "requires_approval") {
        throw new Error("approval_required_before_external_side_effect");
      }

      if (receipt.decision !== "allowed") {
        throw new Error("external_side_effect_denied");
      }

      return idempotency.runOnce(normalized.idempotencyKey, async () => {
        const result = await toolRegistry.execute(call);
        await obligationQueue.enqueueAll(receipt.obligations, result);
        return result;
      });
    }

这里有两个关键点：

- dry-run receipt 是 append-only，不是内存变量；
- execute 阶段重新计算 hash，不靠模型自觉。

## 7. OpenClaw 实战：课程 Cron 的副作用预览

拿今天这个课程 cron 举例，至少有三个外部副作用：

- 发送 Telegram 群消息；
- git commit + push 到 GitHub；
- 更新 TOOLS/memory 作为长期上下文。

执行前的预览可以写成：

    {
      "courseNo": 383,
      "sideEffects": [
        {
          "kind": "outbound_message",
          "target": "telegram:-5115329245",
          "summary": "发布第383课课程正文",
          "idempotencyKey": "agent-course-383-telegram"
        },
        {
          "kind": "remote_write",
          "target": "github:gfwfail/agent-course@main",
          "summary": "新增 lesson 文件并更新 README",
          "idempotencyKey": "agent-course-383-git"
        },
        {
          "kind": "local_memory_update",
          "target": "TOOLS.md + memory/2026-05-23.md",
          "summary": "记录已讲主题和完成证据",
          "idempotencyKey": "agent-course-383-memory"
        }
      ]
    }

这张预览的价值是：后续如果 Telegram 发了但 git push 失败，Agent 能明确知道哪一个副作用已完成、哪一个需要补偿或重试，而不是凭对话记忆猜。

## 8. 和审批网关的区别

Dry-run receipt 不是 Approval Gateway 的替代品。

它们的关系是：

    dry-run preview
      说明将要做什么。

    policy decision
      判断是否允许、是否需要审批。

    approval gateway
      如果需要人工确认，收集确认。

    execute with receipt
      确保执行的动作和预览一致。

低风险动作可以 preview 后自动执行；高风险动作必须 preview -> approval -> execute。没有 preview 的 approval 也不可靠，因为人不知道自己到底批准了什么。

## 9. 最小检查清单

做外部副作用工具时，至少问 10 个问题：

- [ ] 工具是否声明 sideEffectKind？
- [ ] preview 是否包含 target、payloadSummary、payloadHash？
- [ ] dry-run receipt 是否 append-only 保存？
- [ ] execute 前是否重新计算 argsHash/payloadHash？
- [ ] receipt 是否有过期时间？
- [ ] 是否有 idempotencyKey 防重复执行？
- [ ] policy 是否基于 sensitivity/target/risk 判断？
- [ ] requires_approval 是否会阻断真实执行？
- [ ] 执行结果是否绑定 receiptId？
- [ ] post-action obligation 是否入队并可重试？

## 10. 这一课的关键

Agent 的外部动作要从“模型决定了就调用工具”升级为四段式：

    preview
      结构化说明将要发生什么。

    dry-run receipt
      冻结参数、payload、policy decision 和 obligations。

    execute with receipt
      真实执行前复核 receipt 与当前动作一致。

    closeout proof
      记录 externalId、messageId、commit hash、部署版本等结果证据。

记住一句话：

**没有 dry-run receipt 的外部副作用，本质上是在让 Agent 盲开生产按钮。**
