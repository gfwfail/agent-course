# 340. Agent 补偿关闭证据与学习回灌（Compensation Closeout & Learning Backfill）

上一课讲了 **Downstream Impact Containment & Compensation**：错误发布已经触发下游动作时，Agent 要先冻结影响范围，再按 reversible / compensatable / manual_review / irreversible 规划补偿。

今天继续收尾：**补偿动作执行完以后，Agent 怎么证明“真的结束了”，并把这次事故变成下次动作前的自动保护？**

核心思想：

> Compensation 解决这一次的影响；Closeout Evidence 证明影响已关闭；Learning Backfill 把新规则回写到 Runbook、Policy 和 Regression Case。成熟 Agent 不是“补偿完就忘”，而是把事故变成以后执行路径里的闸门。

---

## 1. 为什么补偿后还要 Closeout？

很多系统做到这里就停了：

```text
ImpactCase -> CompensationPlan -> execute steps -> done
```

问题是，“执行过补偿”不等于“影响已关闭”。

比如：

- 回滚部署执行了，但健康检查还没通过。
- GitHub 评论更正了，但 reviewer 仍基于旧评论 approve。
- Telegram 课程发了撤回通知，但 README / TOOLS / 远端 main 没同步。
- Cron 冻结过，但没有证明后续 run 不再消费旧证据。
- 手工复核完成了，但没有回写可自动检查的新规则。

所以 closeout 必须回答四个问题：

```text
1. 每个受影响对象是否有 terminal evidence？
2. 每个冻结范围是否可以解冻？
3. 这次事故暴露了哪些缺失规则？
4. 新规则是否已经进入未来执行路径，而不只是写在复盘里？
```

完整闭环应该是：

```text
Compensation Plan
  -> Step Evidence
  -> Closeout Gate
  -> Learning Backfill
  -> Regression Replay
  -> Unfreeze / Keep Frozen
  -> Release Ledger Append
```

注意：Closeout 不是“写一段总结”。它是一个 gate：证据不够就不能关闭 case，学习没有回灌就不能解冻高风险自动化。

---

## 2. 数据结构：CloseoutBundle

把关闭证明做成结构化对象，方便审计和自动复跑。

```json
{
  "closeoutId": "closeout_340",
  "impactCaseId": "impact_339",
  "compensationPlanId": "comp_339",
  "state": "ready_to_close",
  "stepEvidence": [
    {
      "stepId": "freeze:repo:gfwfail/agent-course:push",
      "target": "repo:gfwfail/agent-course:push",
      "terminalState": "frozen_then_unfrozen",
      "evidenceRefs": ["ev_freeze_001", "ev_unfreeze_001"]
    },
    {
      "stepId": "compensate:telegram:12171",
      "target": "telegram:12171",
      "terminalState": "corrected",
      "evidenceRefs": ["ev_message_edit_001", "ev_release_ledger_001"]
    }
  ],
  "learningBackfill": {
    "runbookPatches": ["rb_git_push_preflight_v4"],
    "policyRules": ["policy_external_release_requires_closeout"],
    "regressionCases": ["regression_old_release_downstream_impact"]
  },
  "unfreezeDecision": {
    "allowed": true,
    "reason": "all affected objects have terminal evidence and regression replay passed"
  }
}
```

关键字段：

- `terminalState`：每个对象必须进入明确终态，不能停在“已尝试”。
- `evidenceRefs`：关闭 case 靠证据引用，不靠自然语言承诺。
- `learningBackfill`：把教训拆成 Runbook、Policy、Regression 三类，不混成一篇复盘。
- `unfreezeDecision`：解冻也要有原因和证据，不能默认恢复自动化。

---

## 3. learn-claude-code：最小 Closeout Gate

教学版用 JSON 文件做一个关闭闸门：检查补偿步骤是否都有 terminal evidence，再生成学习回灌补丁。

```python
# learn-claude-code/compensation_closeout.py
from dataclasses import dataclass, asdict
from pathlib import Path
from typing import Literal
import json

TerminalState = Literal[
    "rolled_back",
    "compensated",
    "corrected",
    "manual_reviewed",
    "irreversible_acknowledged",
    "frozen_then_unfrozen",
]

@dataclass
class StepEvidence:
    step_id: str
    target: str
    terminal_state: TerminalState | None
    evidence_refs: list[str]

@dataclass
class LearningBackfill:
    runbook_patches: list[str]
    policy_rules: list[str]
    regression_cases: list[str]

@dataclass
class CloseoutBundle:
    closeout_id: str
    impact_case_id: str
    compensation_plan_id: str
    step_evidence: list[StepEvidence]
    learning_backfill: LearningBackfill

class CloseoutGate:
    def __init__(self, root: Path):
        self.root = root
        self.root.mkdir(parents=True, exist_ok=True)

    def missing_terminal_evidence(self, bundle: CloseoutBundle) -> list[str]:
        missing: list[str] = []
        for step in bundle.step_evidence:
            if step.terminal_state is None:
                missing.append(f"{step.step_id}: missing terminal_state")
            if not step.evidence_refs:
                missing.append(f"{step.step_id}: missing evidence_refs")
        return missing

    def missing_learning_backfill(self, bundle: CloseoutBundle) -> list[str]:
        gaps: list[str] = []
        backfill = bundle.learning_backfill
        if not backfill.runbook_patches:
            gaps.append("no runbook patch")
        if not backfill.policy_rules:
            gaps.append("no policy rule")
        if not backfill.regression_cases:
            gaps.append("no regression case")
        return gaps

    def decide(self, bundle: CloseoutBundle) -> dict:
        evidence_gaps = self.missing_terminal_evidence(bundle)
        learning_gaps = self.missing_learning_backfill(bundle)
        allowed = not evidence_gaps and not learning_gaps

        decision = {
            "closeout_id": bundle.closeout_id,
            "impact_case_id": bundle.impact_case_id,
            "allowed": allowed,
            "action": "close_and_unfreeze" if allowed else "keep_contained",
            "evidence_gaps": evidence_gaps,
            "learning_gaps": learning_gaps,
        }

        path = self.root / "closeout_decisions.jsonl"
        with path.open("a", encoding="utf-8") as f:
            f.write(json.dumps(decision, ensure_ascii=False, sort_keys=True) + "\n")

        return decision

if __name__ == "__main__":
    bundle = CloseoutBundle(
        closeout_id="closeout_340",
        impact_case_id="impact_339",
        compensation_plan_id="comp_339",
        step_evidence=[
            StepEvidence(
                step_id="freeze:repo:gfwfail/agent-course:push",
                target="repo:gfwfail/agent-course:push",
                terminal_state="frozen_then_unfrozen",
                evidence_refs=["ev_freeze_001", "ev_unfreeze_001"],
            ),
            StepEvidence(
                step_id="compensate:telegram:12171",
                target="telegram:12171",
                terminal_state="corrected",
                evidence_refs=["ev_message_edit_001", "ev_release_ledger_001"],
            ),
        ],
        learning_backfill=LearningBackfill(
            runbook_patches=["rb_git_push_preflight_v4"],
            policy_rules=["policy_external_release_requires_closeout"],
            regression_cases=["regression_old_release_downstream_impact"],
        ),
    )

    print(CloseoutGate(Path(".agent_state")).decide(bundle))
```

这个例子故意很朴素，但有两个关键点：

- `allowed=false` 时不是“失败”，而是保持 containment，避免影响继续扩散。
- 学习回灌是 closeout 的必要条件，不是可选项。

---

## 4. pi-mono：CloseoutMiddleware

生产版可以把 closeout 放进工具中间件：凡是要解冻、恢复 cron、继续 push、重新发送外部消息，都必须先证明对应 ImpactCase 可关闭。

```ts
type CloseoutDecision =
  | {
      action: "close_and_unfreeze";
      closeoutId: string;
      evidenceRefs: string[];
      learningRefs: string[];
      reason: string;
    }
  | {
      action: "keep_contained";
      closeoutId: string;
      gaps: string[];
      reason: string;
    };

type ToolContext = {
  runId: string;
  actorId: string;
  toolName: string;
  args: Record<string, unknown>;
  risk: "low" | "medium" | "high";
  impactCaseId?: string;
};

type ToolResult = {
  ok: boolean;
  output?: unknown;
  blocked?: boolean;
  reason?: string;
};

class CloseoutService {
  async evaluate(input: {
    impactCaseId: string;
    actorId: string;
    target: string;
  }): Promise<CloseoutDecision> {
    const bundle = await loadCloseoutBundle(input.impactCaseId);
    const gaps = [
      ...findMissingTerminalEvidence(bundle),
      ...findMissingLearningBackfill(bundle),
      ...await replayRegressionCases(bundle.learningBackfill.regressionCases),
    ];

    if (gaps.length > 0) {
      return {
        action: "keep_contained",
        closeoutId: bundle.closeoutId,
        gaps,
        reason: "impact case is not safe to close",
      };
    }

    return {
      action: "close_and_unfreeze",
      closeoutId: bundle.closeoutId,
      evidenceRefs: bundle.stepEvidence.flatMap((step) => step.evidenceRefs),
      learningRefs: [
        ...bundle.learningBackfill.runbookPatches,
        ...bundle.learningBackfill.policyRules,
        ...bundle.learningBackfill.regressionCases,
      ],
      reason: "terminal evidence and regression replay passed",
    };
  }
}

function requiresCloseout(ctx: ToolContext): boolean {
  return (
    ctx.risk === "high" &&
    ["git.push", "cron.resume", "message.send", "deploy.promote"].includes(ctx.toolName) &&
    typeof ctx.impactCaseId === "string"
  );
}

function closeoutMiddleware(service: CloseoutService) {
  return async (ctx: ToolContext, next: () => Promise<ToolResult>): Promise<ToolResult> => {
    if (!requiresCloseout(ctx)) {
      return next();
    }

    const decision = await service.evaluate({
      impactCaseId: ctx.impactCaseId!,
      actorId: ctx.actorId,
      target: ctx.toolName,
    });

    await appendAuditEvent({
      type: "closeout_decision",
      runId: ctx.runId,
      toolName: ctx.toolName,
      decision,
    });

    if (decision.action === "keep_contained") {
      return {
        ok: false,
        blocked: true,
        reason: `impact case not closed: ${decision.gaps.join("; ")}`,
      };
    }

    return next();
  };
}
```

这里的重点不是 TypeScript 语法，而是 **恢复自动化之前要复核 closeout**。否则 Agent 很容易出现这种事故链：

```text
旧证据导致外部发布错误
  -> 下游被影响
  -> Agent 补偿
  -> 没证明补偿完整
  -> cron 恢复
  -> 同类错误再次发生
```

---

## 5. OpenClaw 实战：课程 Cron 的 Closeout Checklist

以我们这个课程 cron 为例，一次发布不是只有“发群”：

```text
write lesson
  -> update README
  -> update TOOLS
  -> send Telegram
  -> git commit
  -> git push
  -> verify remote
  -> write daily memory
```

如果其中某步用了错误证据，比如 README 目录没更新但消息已发群，closeout 不能只说“补一下 README”。更稳的 checklist 是：

```text
CloseoutBundle for agent-course cron

Terminal evidence:
- lesson file exists and title matches Telegram topic
- README includes the exact lesson link
- TOOLS.md 已讲内容 includes the exact topic
- Telegram messageId recorded in memory
- git commit contains lesson + README + TOOLS update
- remote main points to pushed commit
- git status is clean or only contains unrelated pre-existing changes

Learning backfill:
- 如果漏了 README：新增 preflight check: README must include new lesson
- 如果漏了 TOOLS：新增 preflight check: TOOLS topic must be updated
- 如果消息发错：新增 regression: external message content must match lesson title
- 如果 push 前证据过期：新增 freshness gate before git push
```

关键是：**closeout checklist 要和具体工作流绑定**。课程 cron 的 closeout，不应该套用部署系统的 checklist；部署系统也不应该套用发群消息的 checklist。

---

## 6. 常见坑

第一，只有总结，没有 gate。
“下次注意”不是学习回灌。必须变成 Runbook patch、Policy rule 或 Regression case。

第二，补偿步骤没有终态。
`attempted`、`sent`、`started` 不是终态。终态应该是 `verified`、`corrected`、`rolled_back`、`manual_reviewed` 这类可审计状态。

第三，解冻太早。
只要还有 affected object 没有 terminal evidence，就应该 keep contained，而不是恢复全部自动化。

第四，把所有事故都塞进 MEMORY.md。
长期记忆能提醒 Agent，但真正可靠的是执行前自动检查。记忆是辅助，gate 才是保护。

---

## 7. 一句话总结

> Agent 的事故闭环不是“发现、补偿、结束”，而是“发现、补偿、证明关闭、学习回灌、回归验证、再解冻”。能把事故经验写回执行路径的 Agent，才会越跑越稳。
