# 339. Agent 下游影响隔离与补偿计划（Downstream Impact Containment & Compensation）

上一课讲了 **Recipient Acknowledgement & Impact Tracking**：更正通知发出去以后，Agent 要追踪关键接收者是否看到，以及是否有人已经基于旧消息行动。

今天继续讲更难的一步：**如果下游真的已经行动了，Agent 怎么收口？**

比如旧部署指令被撤回前有人已经部署、错误 GitHub 评论被修正前 reviewer 已经 approve、错误账单结论已经被转发给客户、Runbook 旧参数已经被 cron 消费。核心思想：

> Ack 证明“关键人知道了”；Impact Containment 证明“影响范围被隔离了”；Compensation Plan 证明“已发生的下游动作有补偿、验证和关闭条件”。成熟 Agent 不只会道歉，还会把错误影响面压住并可审计地收尾。

---

## 1. 为什么只发更正不够？

很多系统的外部发布闭环停在这里：

```text
wrong release -> correction notice -> recipients acknowledged -> done
```

但现实里，旧消息可能已经触发下游动作：

```text
旧消息: "可以推 main"
  -> 人类执行 git push
  -> CI 部署生产
  -> 客户看到异常
  -> Agent 发更正
```

这时“大家看到更正了”只是止血的一部分，还必须回答：

- 哪些 workflow / PR / ticket / cron run / deployment 被旧消息影响？
- 哪些动作是可逆的？哪些只能补偿？
- 在确认范围前，哪些后续自动动作必须冻结？
- 补偿动作本身是否需要审批？
- 怎么证明补偿后影响已经关闭？

更完整的流程应该是：

```text
Impact Signal
  -> Impact Graph
  -> Containment Freeze
  -> Compensation Plan
  -> Execute / Approve
  -> Post-Compensation Verification
  -> Closeout Evidence
```

注意：这不是重新讲 Saga。Saga 关注“一个工作流内部每步怎么回滚”；本课关注 **错误发布已经外溢到多个下游系统以后，怎么先隔离影响，再规划补偿**。

---

## 2. 数据结构：ImpactCase

把下游影响建模成一个 case，而不是散落在聊天记录里的几句“好像有人做了”。

```json
{
  "impactCaseId": "impact_339",
  "releaseId": "rel_337",
  "noticeId": "notice_337",
  "rootCause": "published_old_deploy_instruction",
  "severity": "high",
  "state": "contained",
  "affectedObjects": [
    {
      "kind": "deployment",
      "id": "deploy_abc123",
      "source": "ci",
      "derivedFrom": "telegram:12168",
      "reversibility": "compensatable"
    },
    {
      "kind": "github_review",
      "id": "pr_42_review_9",
      "source": "github",
      "derivedFrom": "github_comment_551",
      "reversibility": "manual_review"
    }
  ],
  "containment": {
    "freezeScopes": ["repo:gfwfail/agent-course:deploy", "cron:agent-course"],
    "startedAt": "2026-05-17T05:45:00.000Z",
    "reason": "old release may have triggered downstream actions"
  },
  "compensationPlanId": "comp_339"
}
```

关键字段：

- `derivedFrom`：下游动作为什么被判定为受影响，必须能追溯到旧发布或旧证据。
- `reversibility`：`reversible`、`compensatable`、`manual_review`、`irreversible` 不能混在一起处理。
- `freezeScopes`：冻结要按范围，不是把整个 Agent 系统一刀切停掉。
- `state`：`detected -> contained -> compensating -> verifying -> closed`，每一步都要有证据。

---

## 3. learn-claude-code：文件式影响隔离器

教学版用 JSONL 实现一个最小可运行的 Impact Tracker：登记影响、生成冻结范围、生成补偿计划。

```python
# learn-claude-code/downstream_impact.py
from dataclasses import dataclass, asdict
from datetime import datetime, timezone
from pathlib import Path
from typing import Literal
import json

Severity = Literal["low", "medium", "high", "critical"]
Reversibility = Literal["reversible", "compensatable", "manual_review", "irreversible"]
ActionKind = Literal["freeze", "rollback", "compensate", "notify", "manual_review", "verify"]

@dataclass
class AffectedObject:
    kind: str
    object_id: str
    source: str
    derived_from: str
    reversibility: Reversibility

@dataclass
class ImpactCase:
    impact_case_id: str
    release_id: str
    notice_id: str
    root_cause: str
    severity: Severity
    affected_objects: list[AffectedObject]
    detected_at: str

@dataclass
class CompensationStep:
    step_id: str
    action: ActionKind
    target: str
    reason: str
    requires_approval: bool
    verification: str

class ImpactTracker:
    def __init__(self, root: Path):
        self.root = root
        self.root.mkdir(parents=True, exist_ok=True)
        self.cases_path = self.root / "impact_cases.jsonl"
        self.plans_path = self.root / "compensation_plans.jsonl"

    def append_jsonl(self, path: Path, row: dict) -> None:
        with path.open("a", encoding="utf-8") as f:
            f.write(json.dumps(row, ensure_ascii=False, sort_keys=True) + "\n")

    def create_case(self, case: ImpactCase) -> None:
        self.append_jsonl(self.cases_path, asdict(case))

    def freeze_scopes(self, case: ImpactCase) -> list[str]:
        scopes: set[str] = set()
        for obj in case.affected_objects:
            if obj.kind == "deployment":
                scopes.add(f"deploy:{obj.source}")
            elif obj.kind == "github_review":
                scopes.add(f"repo:{obj.source}:merge")
            elif obj.kind == "cron_run":
                scopes.add(f"cron:{obj.object_id}")
            else:
                scopes.add(f"object:{obj.kind}:{obj.object_id}")
        return sorted(scopes)

    def build_compensation_plan(self, case: ImpactCase) -> dict:
        steps: list[CompensationStep] = []

        for scope in self.freeze_scopes(case):
            steps.append(CompensationStep(
                step_id=f"freeze:{scope}",
                action="freeze",
                target=scope,
                reason="prevent affected downstream automation from expanding impact",
                requires_approval=False,
                verification=f"{scope} is in frozen/degraded mode",
            ))

        for obj in case.affected_objects:
            target = f"{obj.kind}:{obj.object_id}"
            if obj.reversibility == "reversible":
                action, reason = "rollback", "object can be safely reverted"
            elif obj.reversibility == "compensatable":
                action, reason = "compensate", "apply a forward fix instead of blind rollback"
            else:
                action, reason = "manual_review", f"reversibility={obj.reversibility}"

            steps.append(CompensationStep(
                step_id=f"{action}:{target}",
                action=action,
                target=target,
                reason=reason,
                requires_approval=action != "freeze",
                verification=f"{target} has terminal evidence before closeout",
            ))

        steps.append(CompensationStep(
            step_id="verify:closeout",
            action="verify",
            target=case.impact_case_id,
            reason="prove no affected object remains unhandled",
            requires_approval=False,
            verification="all steps completed and release ledger appended",
        ))

        plan = {
            "compensation_plan_id": f"comp_{case.impact_case_id}",
            "impact_case_id": case.impact_case_id,
            "created_at": datetime.now(timezone.utc).isoformat(),
            "steps": [asdict(step) for step in steps],
        }
        self.append_jsonl(self.plans_path, plan)
        return plan

if __name__ == "__main__":
    tracker = ImpactTracker(Path(".agent-impact"))
    case = ImpactCase(
        impact_case_id="impact_339",
        release_id="rel_337",
        notice_id="notice_337",
        root_cause="old instruction was used before correction notice",
        severity="high",
        affected_objects=[
            AffectedObject("deployment", "deploy_abc123", "ci", "telegram:12168", "compensatable"),
            AffectedObject("github_review", "review_9", "gfwfail/agent-course", "github_comment_551", "manual_review"),
        ],
        detected_at=datetime.now(timezone.utc).isoformat(),
    )
    tracker.create_case(case)
    print(json.dumps(tracker.build_compensation_plan(case), ensure_ascii=False, indent=2))
```

教学版强调三件事：

1. **先冻结，再补偿**：影响范围没收住前，不要让自动化继续跑。
2. **按可逆性分类**：rollback、forward fix、人工复核不是同一种动作。
3. **每步都有 verification**：补偿计划不是 TODO 列表，而是带关闭条件的执行契约。

---

## 4. pi-mono：DownstreamImpactMiddleware

生产版可以挂在 Release Ledger、Ack Tracker 和工具执行队列之间：一旦发现旧发布已触发下游动作，就给受影响范围加临时控制策略。

```ts
// pi-mono/packages/agent-safety/src/downstream-impact.ts
export type Severity = "low" | "medium" | "high" | "critical";
export type Reversibility = "reversible" | "compensatable" | "manual_review" | "irreversible";
export type ImpactState = "detected" | "contained" | "compensating" | "verifying" | "closed";

export interface AffectedObject {
  kind: "deployment" | "github_review" | "ticket" | "cron_run" | "message" | "custom";
  id: string;
  source: string;
  derivedFrom: string;
  reversibility: Reversibility;
}

export interface ImpactCase {
  impactCaseId: string;
  releaseId: string;
  noticeId: string;
  severity: Severity;
  state: ImpactState;
  affectedObjects: AffectedObject[];
}

export interface ControlPlane {
  freeze(scope: string, reason: string, ttlMs: number): Promise<{ freezeId: string }>;
}

export interface CompensationStep {
  id: string;
  action: "freeze" | "rollback" | "forward_fix" | "manual_review" | "verify";
  target: string;
  requiresApproval: boolean;
  verification: string;
}

export class DownstreamImpactMiddleware {
  constructor(
    private readonly controlPlane: ControlPlane,
    private readonly ledger: { append(event: unknown): Promise<void> },
  ) {}

  async contain(case_: ImpactCase): Promise<CompensationStep[]> {
    const steps: CompensationStep[] = [];

    for (const scope of this.deriveFreezeScopes(case_)) {
      const result = await this.controlPlane.freeze(
        scope,
        "impact_case=" + case_.impactCaseId + "; old release may have triggered downstream action",
        60 * 60 * 1000,
      );
      await this.ledger.append({
        type: "impact_scope_frozen",
        impactCaseId: case_.impactCaseId,
        scope,
        freezeId: result.freezeId,
      });
      steps.push({
        id: "freeze:" + scope,
        action: "freeze",
        target: scope,
        requiresApproval: false,
        verification: scope + " remains frozen until compensation is verified",
      });
    }

    for (const object of case_.affectedObjects) {
      steps.push(this.planForObject(object));
    }

    steps.push({
      id: "verify:closeout",
      action: "verify",
      target: case_.impactCaseId,
      requiresApproval: false,
      verification: "all affected objects have terminal evidence and freeze scopes are released intentionally",
    });

    await this.ledger.append({
      type: "compensation_plan_created",
      impactCaseId: case_.impactCaseId,
      steps,
    });

    return steps;
  }

  private deriveFreezeScopes(case_: ImpactCase): string[] {
    const scopes = new Set<string>();
    for (const object of case_.affectedObjects) {
      if (object.kind === "deployment") scopes.add("deploy:" + object.source);
      if (object.kind === "github_review") scopes.add("repo:" + object.source + ":merge");
      if (object.kind === "cron_run") scopes.add("cron:" + object.id);
      if (case_.severity === "critical") scopes.add("release:" + case_.releaseId + ":external_side_effects");
    }
    return [...scopes].sort();
  }

  private planForObject(object: AffectedObject): CompensationStep {
    const target = object.kind + ":" + object.id;
    if (object.reversibility === "reversible") {
      return { id: "rollback:" + target, action: "rollback", target, requiresApproval: true, verification: "previous known-good state restored" };
    }
    if (object.reversibility === "compensatable") {
      return { id: "forward_fix:" + target, action: "forward_fix", target, requiresApproval: true, verification: "compensating change landed and affected recipients notified" };
    }
    return { id: "manual_review:" + target, action: "manual_review", target, requiresApproval: true, verification: "owner signed off residual risk" };
  }
}
```

这层中间件的原则：

- `contain()` 可以自动冻结低风险范围，因为它减少伤害；
- `rollback / forward_fix` 默认要审批，因为它们是新的副作用；
- `critical` case 可以冻结更宽的 external side effects；
- 所有冻结和补偿计划都写回 Release Ledger，方便事后审计。

---

## 5. OpenClaw 实战：课程 Cron 怎么用

拿当前课程发布 Cron 举例，错误发布后的影响收口可以这样做：

```text
1. Release Verification 发现 Telegram 消息内容 drift 或旧内容不可信
2. Correction Notice 发到 Rust 学习小组
3. Ack Tracker 发现 owner/on-call 已确认
4. Impact Tracker 扫描：
   - 是否有人引用旧课程发了 README / PR？
   - 是否已经 git push 了错误 lesson？
   - TOOLS.md 已讲内容是否写入了错误主题？
5. 如果有影响：
   - freeze agent-course cron 下一轮 push
   - create compensation PR 或补偿 commit
   - 发更正说明
   - 重新跑 README / TOOLS / git ls-remote 验证
6. Release Ledger 追加 closeout evidence
```

关键不是“永远不出错”，而是错误发布发生后，Agent 不会继续机械执行下一轮自动化，把影响滚大。

---

## 6. 设计清单

- [ ] 每个更正通知是否会触发 Impact Scan？
- [ ] Impact Scan 是否能找到 PR、ticket、deployment、cron、message 等下游对象？
- [ ] 每个 affected object 是否标注 `derivedFrom`？
- [ ] 是否按 `reversible / compensatable / manual_review / irreversible` 分类？
- [ ] 冻结范围是否足够小，但能阻止影响扩散？
- [ ] 冻结动作是否有 TTL，避免永久卡死？
- [ ] 补偿动作是否默认需要审批？
- [ ] 每个补偿步骤是否有 verification condition？
- [ ] closeout 前是否确认没有未处理 affected object？
- [ ] closeout evidence 是否写回 Release Ledger？

---

## 7. 常见坑

**坑 1：更正通知发完就结束。**

更正只能改变未来读者的认知，不能自动撤销已经发生的下游动作。

**坑 2：把所有东西都 rollback。**

有些动作不能回滚，比如客户已经收到邮件、外部系统已经消费 webhook。正确做法是 forward fix、补偿通知、人工确认。

**坑 3：冻结范围太大。**

全局 freeze 会让系统不可用，也会让值班人员绕过系统。更好的做法是按 repo、cron、channel、tool purpose 冻结。

**坑 4：补偿计划没有关闭条件。**

“检查一下”不是 verification。好的关闭条件应该可执行、可审计，比如 `post_check passed`、`PR merged`、`message edited`、`owner signed off`。

---

## 8. 小结

- 发布更正以后，还要追踪旧发布是否已经触发下游动作。
- 影响收口第一步是 containment freeze，不是急着补偿。
- 下游对象要按可逆性分类：能回滚、能补偿、需人工、不可逆。
- 补偿计划必须带审批、验证和 closeout evidence。
- Release Ledger 负责把“发现影响 -> 冻结 -> 补偿 -> 验证关闭”串成可审计事实链。

一句话：

> 成熟 Agent 的外部发布闭环，不是“错了就改口”，而是发现错误影响后，能马上隔离、规划补偿、验证收口，并证明影响没有继续扩散。
