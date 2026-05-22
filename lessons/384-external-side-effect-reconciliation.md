# 384. Agent 外部副作用执行后的对账与分歧修复（External Side Effect Reconciliation & Divergence Repair）

上一课讲了 External Side Effect Preview & Dry-run Receipt：外部副作用执行前，要先冻结 target、payloadHash、argsHash、policy decision 和 obligations。

今天继续往后走一步：执行完以后，怎么确认现实世界真的变成了我们以为的样子？

生产里最麻烦的事故，往往不是“工具直接报错”，而是这种半成功状态：

- Telegram 工具返回超时，但消息其实发出去了；
- Git push 返回成功，本地 memory 没写完成记录；
- 本地 ledger 写了 executed，远端 provider 没有 messageId；
- retry 时 idempotencyKey 不稳定，发了两条重复消息；
- 部署 API 返回 accepted，几分钟后实际 rollout failed；
- obligation 队列执行了一半，closeout proof 缺关键证据。

所以今天讲 External Side Effect Reconciliation & Divergence Repair。

一句话：**外部副作用不能只看工具返回值；要用本地 Effect Ledger 和远端 Reality Snapshot 对账，发现分歧后按可恢复策略修复。**

## 1. 为什么 dry-run 之后还需要对账

Dry-run receipt 解决的是“执行前有没有看清楚”。

Reconciliation 解决的是“执行后世界是否真的一致”。

二者不是重复，而是一前一后：

    PreviewSpec
      冻结将要做什么。

    DryRunReceipt
      证明执行前检查通过。

    ExecutionReceipt
      记录工具实际返回什么。

    RealitySnapshot
      从远端重新读取事实。

    ReconciliationReport
      判断 expected 与 actual 是否一致。

只保存 ExecutionReceipt 不够，因为工具返回值可能不完整、延迟、超时，甚至和远端最终状态不一致。

成熟 Agent 要养成一个习惯：**凡是外部写操作，执行后都要能重新读回来验证。**

## 2. Effect Ledger：先把本地期待写清楚

对账的前提是本地有 ledger。没有 ledger，就不知道该对什么账。

最小 Effect Ledger entry：

    {
      "effectId": "effect_agent_course_384_telegram",
      "receiptId": "dryrun_384_telegram_001",
      "kind": "outbound_message",
      "target": "telegram:-5115329245",
      "idempotencyKey": "agent-course-384-telegram",
      "expected": {
        "payloadHash": "sha256:...",
        "mustHaveExternalId": true,
        "mustBeVisible": true
      },
      "status": "executing",
      "attempt": 1,
      "createdAt": "2026-05-23T09:30:00+10:00"
    }

注意 expected 不是自然语言描述，而是可验证条件。

例如：

- Telegram：必须拿到 messageId，消息文本 hash 匹配；
- GitHub：remote main 必须包含 commit sha；
- Email：provider accepted id 存在，并且收件人/subject hash 匹配；
- Deployment：目标 environment 的 active version 等于 expected sha；
- Memory write：文件内容包含 lesson id 和完成证据。

## 3. RealitySnapshot：从远端重新采样现实

对账不能用刚才工具的返回值冒充 reality。

RealitySnapshot 必须来自独立的 read path：

    {
      "effectId": "effect_agent_course_384_git",
      "observedAt": "2026-05-23T09:35:12+10:00",
      "source": "git ls-remote origin main",
      "actual": {
        "remoteHead": "abc123...",
        "containsExpectedCommit": true
      }
    }

这一步很关键。

如果 Git push 工具说 success，但 ls-remote 看不到 commit，说明远端现实并没有接受这次写入。

如果 Telegram send 返回 timeout，但 poll/read 能看到同一 idempotencyKey 的消息，说明不应该重发，而应该补写 externalId。

## 4. 分歧类型：不要只有成功/失败

对账结果至少要分成几类：

    matched
      本地期待和远端现实一致。

    pending
      远端最终一致性还没完成，可以稍后再查。

    local_missing_remote_present
      远端已经发生，本地 ledger 缺回执。

    local_present_remote_missing
      本地以为成功，远端没有。

    duplicate_external_effect
      远端出现重复副作用。

    payload_mismatch
      远端存在动作，但 payload/target 和预期不一致。

    obligation_incomplete
      主动作成功，但后置义务没闭环。

分类越细，修复策略越稳。

把所有问题都当成 failed，会导致重复发送、重复部署、重复扣款。

## 5. learn-claude-code：教学版对账器

教学版可以做成纯函数：输入 ledger entry 和 reality snapshot，输出 report。

    from dataclasses import dataclass
    from typing import Literal

    Status = Literal[
        "matched",
        "pending",
        "local_missing_remote_present",
        "local_present_remote_missing",
        "duplicate_external_effect",
        "payload_mismatch",
        "obligation_incomplete",
    ]

    @dataclass(frozen=True)
    class LedgerEntry:
        effect_id: str
        kind: str
        idempotency_key: str
        expected_payload_hash: str
        external_id: str | None
        obligations: list[str]
        completed_obligations: list[str]

    @dataclass(frozen=True)
    class RealitySnapshot:
        found: bool
        external_id: str | None
        payload_hash: str | None
        duplicate_count: int
        final: bool

    def reconcile(entry: LedgerEntry, reality: RealitySnapshot) -> dict:
        missing_obligations = [
            item for item in entry.obligations
            if item not in entry.completed_obligations
        ]

        if reality.duplicate_count > 1:
            return {
                "effectId": entry.effect_id,
                "status": "duplicate_external_effect",
                "repair": "dedupe_ledger_and_escalate_if_irreversible",
            }

        if not reality.found and not reality.final:
            return {
                "effectId": entry.effect_id,
                "status": "pending",
                "repair": "schedule_recheck",
            }

        if entry.external_id is None and reality.found:
            return {
                "effectId": entry.effect_id,
                "status": "local_missing_remote_present",
                "repair": "backfill_external_id",
            }

        if entry.external_id is not None and not reality.found:
            return {
                "effectId": entry.effect_id,
                "status": "local_present_remote_missing",
                "repair": "retry_with_same_idempotency_key_or_escalate",
            }

        if reality.payload_hash != entry.expected_payload_hash:
            return {
                "effectId": entry.effect_id,
                "status": "payload_mismatch",
                "repair": "freeze_followup_side_effects_and_open_review",
            }

        if missing_obligations:
            return {
                "effectId": entry.effect_id,
                "status": "obligation_incomplete",
                "missingObligations": missing_obligations,
                "repair": "enqueue_missing_obligations",
            }

        return {
            "effectId": entry.effect_id,
            "status": "matched",
            "repair": "none",
        }

这个函数没有调用 Telegram 或 Git。它只判断“本地期待”和“远端事实”的关系。

真正的工程里，read path、repair path、audit path 可以分开测试。

## 6. 修复策略：先补账，再重试

发现分歧后，不要马上重试外部写。

正确顺序：

1. 先判断远端是否已经发生；
2. 如果已经发生，补本地 ledger；
3. 如果没发生，再用同一个 idempotencyKey 重试；
4. 如果 payload mismatch 或 duplicate，冻结后续副作用，进入人工复核；
5. 如果只是 obligation 缺失，补跑 obligation，不重复主动作。

特别是消息、邮件、支付、订单这类动作：**宁可补账，不要盲目重发。**

## 7. pi-mono：SideEffectReconciler

生产版可以把 reconciler 做成后台 worker，也可以在外部写工具完成后同步触发一次轻量对账。

    type ReconciliationStatus =
      | "matched"
      | "pending"
      | "local_missing_remote_present"
      | "local_present_remote_missing"
      | "duplicate_external_effect"
      | "payload_mismatch"
      | "obligation_incomplete";

    type EffectLedgerEntry = {
      effectId: string;
      receiptId: string;
      kind: "outbound_message" | "remote_write" | "deployment" | "data_export";
      target: string;
      idempotencyKey: string;
      expectedPayloadHash: string;
      externalId?: string;
      obligations: string[];
      completedObligations: string[];
    };

    type RealitySnapshot = {
      observedAt: string;
      source: string;
      found: boolean;
      final: boolean;
      externalId?: string;
      payloadHash?: string;
      duplicateCount: number;
    };

    async function reconcileEffect(entry: EffectLedgerEntry): Promise<ReconciliationReport> {
      const snapshot = await realityReaders.read(entry.kind, entry);
      const report = classifyDivergence(entry, snapshot);

      await reconciliationStore.append({
        ...report,
        effectId: entry.effectId,
        receiptId: entry.receiptId,
        observedAt: snapshot.observedAt,
        snapshotSource: snapshot.source,
      });

      await repairRouter.route(report, entry, snapshot);

      return report;
    }

    function classifyDivergence(
      entry: EffectLedgerEntry,
      snapshot: RealitySnapshot,
    ): { status: ReconciliationStatus; repairAction: string } {
      if (snapshot.duplicateCount > 1) {
        return {
          status: "duplicate_external_effect",
          repairAction: "freeze_and_open_review",
        };
      }

      if (!snapshot.found && !snapshot.final) {
        return {
          status: "pending",
          repairAction: "schedule_recheck",
        };
      }

      if (!entry.externalId && snapshot.found) {
        return {
          status: "local_missing_remote_present",
          repairAction: "backfill_external_id",
        };
      }

      if (entry.externalId && !snapshot.found) {
        return {
          status: "local_present_remote_missing",
          repairAction: "retry_same_idempotency_key",
        };
      }

      if (snapshot.payloadHash !== entry.expectedPayloadHash) {
        return {
          status: "payload_mismatch",
          repairAction: "freeze_and_open_review",
        };
      }

      const missing = entry.obligations.filter(
        (name) => !entry.completedObligations.includes(name),
      );

      if (missing.length > 0) {
        return {
          status: "obligation_incomplete",
          repairAction: "enqueue_missing_obligations",
        };
      }

      return {
        status: "matched",
        repairAction: "none",
      };
    }

这里的关键不是代码复杂，而是职责分离：

- realityReaders 只负责读远端事实；
- classifyDivergence 只负责分类；
- repairRouter 只负责执行修复；
- reconciliationStore 负责留审计证据。

这样以后加 Slack、GitHub、Stripe、Laravel Cloud、K8s，都不用改 LLM prompt。

## 8. OpenClaw 实战：课程 Cron 的三段对账

拿今天这个课程 cron 举例，它至少有三类副作用：

    telegram_send
      预期：Rust学习小组出现第384课消息，并拿到 messageId。

    git_push
      预期：gfwfail/agent-course main 包含新 commit。

    memory_update
      预期：TOOLS.md 已讲内容和 memory/2026-05-23.md 都记录第384课。

执行后不能只说“我发了”。要做三段对账：

    Telegram RealitySnapshot
      openclaw_message 返回 messageId，必要时 poll 群消息确认。

    Git RealitySnapshot
      git ls-remote origin main，确认 remote head 包含本地 commit。

    Memory RealitySnapshot
      rg lesson slug / title，确认 TOOLS.md 和 daily memory 都有记录。

如果 Telegram 已发但 git push 失败，修复策略不是重发 Telegram，而是补完成 git push，然后把同一个 messageId 绑定到 closeout proof。

如果 git push 成功但 memory 没写，修复策略是补 memory，不重写 lesson。

如果发现 README 没包含 lesson 384，那是 obligation_incomplete，补 README 并 amend/new commit，而不是重新生成课程。

## 9. 和 Outbox / Obligation 的区别

之前讲过 Outbox、Policy Obligation Queue、Proof-Carrying Actions。

它们和今天的关系是：

    Outbox
      负责可靠投递。

    Obligation Queue
      负责动作后的必做事项。

    Proof-Carrying Action
      负责执行时携带证据。

    Reconciliation
      负责执行后校验本地记录与远端现实是否一致。

可以把 Reconciliation 理解成最后的“现实校验层”：不信模型、不信内存、不信单次工具返回，只信可重复读取的事实。

## 10. 最小检查清单

给外部副作用加对账时，至少检查 8 件事：

- [ ] 每个外部写是否有 effectId？
- [ ] ledger 是否保存 receiptId、idempotencyKey、expectedPayloadHash？
- [ ] 是否有独立 read path 采样 RealitySnapshot？
- [ ] 是否区分 pending、missing、duplicate、payload_mismatch？
- [ ] retry 是否复用同一个 idempotencyKey？
- [ ] remote present 时是否优先 backfill，而不是重发？
- [ ] obligation 缺失时是否只补 obligation？
- [ ] 对账报告是否 append-only 保存，方便审计和 replay？

## 11. 这一课的关键

Agent 做外部副作用，不能停在“工具调用成功”。

真正可靠的闭环是：

    dry-run receipt
      执行前知道自己要做什么。

    execution receipt
      执行时记录工具返回什么。

    reality snapshot
      执行后重新读取世界发生了什么。

    reconciliation report
      判断本地记录和远端现实是否一致。

    repair action
      只修缺口，不重复制造副作用。

记住一句话：

**成熟 Agent 不只会执行外部动作，还会对账；不只会重试，还知道什么时候应该补账。**
