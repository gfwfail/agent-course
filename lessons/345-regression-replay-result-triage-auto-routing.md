# 345. Agent 回归回放结果分流与自动派单（Regression Replay Result Triage & Auto-Routing）

上一课讲了 determinism gate：同一个 regression case 至少跑多次，先区分 stable_pass、stable_fail 和 flaky_quarantined。

今天往后接一步：**回放结果跑出来以后，不能只在 CI 里显示红叉；Agent 要把失败分流成可处理的工作项。**

成熟 Agent 的回归系统应该回答三个问题：

1. 这是产品行为真的回归了，还是测试夹具过期了？
2. 这件事应该阻断发布、降级、自动派给子 Agent，还是进入人工复核？
3. 派出去以后，owner 需要哪些上下文才能不用重新考古？

## 1. 为什么 replay 结果需要 triage

很多团队的回归回放会停在这一步：

~~~text
selected cases: 18
stable pass: 16
stable fail: 1
flaky quarantined: 1
decision: block
~~~

这个结果能阻断发布，但不能推动修复。

真正的问题是：stable_fail 可能来自不同原因：

- 策略被放宽了：以前 require_approval，现在 allow。
- 工具调用少了：以前先查 PR 状态，现在直接 push。
- 输出变了：模型话术变了，但安全决策没变。
- 证据字段升级了：旧 fixture 缺 schema version，测试应该迁移。
- 环境录制坏了：cassette 缺一条工具响应。

如果所有失败都叫 regression failed，人类和子 Agent 都要重新读日志。Triage 的目标是把 raw replay outcome 转成可执行的 Repair Ticket。

## 2. learn-claude-code：最小结果分流器

在教学版里，可以用一个很小的 Python triage 函数把 replay outcome 分成四类：

- block_fix_required：真实回归，阻断发布，必须修。
- fixture_refresh：测试夹具或 cassette 过期，需要更新证据。
- quarantine_review：flaky 或非确定性，需要隔离复核。
- coverage_gap：本次风险面没有足够 case，需要补测试。

~~~python
from dataclasses import dataclass
from typing import Literal

ReplayStatus = Literal[
    "stable_pass",
    "stable_fail",
    "flaky_quarantined",
    "missing_coverage",
]

TicketKind = Literal[
    "block_fix_required",
    "fixture_refresh",
    "quarantine_review",
    "coverage_gap",
]

@dataclass
class ReplayOutcome:
    case_id: str
    status: ReplayStatus
    expected_decision: str | None
    actual_decision: str | None
    expected_tools: list[str]
    actual_tools: list[str]
    reason: str
    risk: Literal["low", "medium", "high"]

@dataclass
class RepairTicket:
    kind: TicketKind
    case_id: str
    priority: Literal["p1", "p2", "p3"]
    owner_hint: str
    summary: str
    evidence: dict

def triage_replay(outcome: ReplayOutcome) -> RepairTicket | None:
    if outcome.status == "stable_pass":
        return None

    if outcome.status == "missing_coverage":
        return RepairTicket(
            kind="coverage_gap",
            case_id=outcome.case_id,
            priority="p1" if outcome.risk == "high" else "p2",
            owner_hint="test-author",
            summary="本次变更命中了风险面，但没有足够 replay case 覆盖",
            evidence={"reason": outcome.reason},
        )

    if outcome.status == "flaky_quarantined":
        return RepairTicket(
            kind="quarantine_review",
            case_id=outcome.case_id,
            priority="p2",
            owner_hint="test-infra",
            summary="同一个 case 多次回放结果不一致，不能作为发布证据",
            evidence={"reason": outcome.reason},
        )

    if outcome.actual_decision != outcome.expected_decision:
        return RepairTicket(
            kind="block_fix_required",
            case_id=outcome.case_id,
            priority="p1",
            owner_hint="policy-owner",
            summary=f"安全决策漂移：{outcome.expected_decision} -> {outcome.actual_decision}",
            evidence={
                "expected_decision": outcome.expected_decision,
                "actual_decision": outcome.actual_decision,
            },
        )

    missing_tools = [t for t in outcome.expected_tools if t not in outcome.actual_tools]
    if missing_tools:
        return RepairTicket(
            kind="block_fix_required",
            case_id=outcome.case_id,
            priority="p1" if outcome.risk == "high" else "p2",
            owner_hint="agent-loop-owner",
            summary=f"工具调用链缺失：{', '.join(missing_tools)}",
            evidence={"missing_tools": missing_tools},
        )

    return RepairTicket(
        kind="fixture_refresh",
        case_id=outcome.case_id,
        priority="p3",
        owner_hint="test-author",
        summary="输出差异未改变安全决策或工具链，优先检查 fixture 是否需要刷新",
        evidence={"reason": outcome.reason},
    )
~~~

这里的关键不是分类有多复杂，而是**分类结果必须能直接指导下一步动作**。

## 3. pi-mono：把 ticket 挂到 EventStream / AgentLoopConfig 外层

pi-mono 的 Agent loop 本身已经是事件流模型：EventStream<AgentEvent, AgentMessage[]> 会持续 push agent_start、turn_start、message_start、turn_end、agent_end。

这类结构很适合做 replay triage：不要把 triage 写进核心 LLM 调用，而是在外层消费事件、生成 fingerprint、再产出 ticket。

~~~ts
type ReplayOutcome =
  | { status: "stable_pass"; caseId: string; fingerprints: string[] }
  | { status: "stable_fail"; caseId: string; expected: string[]; actual: string[]; risk: "low" | "medium" | "high" }
  | { status: "flaky_quarantined"; caseId: string; attempts: string[][]; risk: "low" | "medium" | "high" }
  | { status: "missing_coverage"; riskSurface: string; risk: "low" | "medium" | "high" };

type RepairTicket = {
  id: string;
  kind: "block_fix_required" | "fixture_refresh" | "quarantine_review" | "coverage_gap";
  priority: "p1" | "p2" | "p3";
  ownerHint: "policy" | "agent-loop" | "test-infra" | "test-author";
  caseId?: string;
  summary: string;
  evidenceRefs: string[];
};

function routeReplayOutcome(outcome: ReplayOutcome): RepairTicket | null {
  if (outcome.status === "stable_pass") return null;

  if (outcome.status === "missing_coverage") {
    return {
      id: crypto.randomUUID(),
      kind: "coverage_gap",
      priority: outcome.risk === "high" ? "p1" : "p2",
      ownerHint: "test-author",
      summary: "Missing regression coverage for " + outcome.riskSurface,
      evidenceRefs: ["risk-surface:" + outcome.riskSurface],
    };
  }

  if (outcome.status === "flaky_quarantined") {
    return {
      id: crypto.randomUUID(),
      kind: "quarantine_review",
      priority: outcome.risk === "high" ? "p1" : "p2",
      ownerHint: "test-infra",
      caseId: outcome.caseId,
      summary: "Replay case produced different fingerprints across attempts",
      evidenceRefs: ["case:" + outcome.caseId, "attempt-fingerprints"],
    };
  }

  const expectedDecision = outcome.expected.find((f) => f.startsWith("decision:"));
  const actualDecision = outcome.actual.find((f) => f.startsWith("decision:"));

  if (expectedDecision !== actualDecision) {
    return {
      id: crypto.randomUUID(),
      kind: "block_fix_required",
      priority: "p1",
      ownerHint: "policy",
      caseId: outcome.caseId,
      summary: "Decision drift: " + expectedDecision + " -> " + actualDecision,
      evidenceRefs: ["case:" + outcome.caseId, "expected-vs-actual"],
    };
  }

  return {
    id: crypto.randomUUID(),
    kind: "fixture_refresh",
    priority: "p3",
    ownerHint: "test-author",
    caseId: outcome.caseId,
    summary: "Fingerprint changed without decision drift; inspect fixture freshness",
    evidenceRefs: ["case:" + outcome.caseId],
  };
}
~~~

注意 ownerHint 不是最终负责人，它只是路由 hint。真正落地时可以由 codeowners、package path、risk surface 和历史 owner 一起决定。

## 4. 自动派单：不是自动修一切

很多人听到 auto-routing 会误解成“让 Agent 自动修所有红灯”。这很危险。

更合理的策略是按 kind 分层：

~~~ts
type DispatchDecision =
  | { action: "block_release"; ticket: RepairTicket }
  | { action: "spawn_worker"; ticket: RepairTicket; sandbox: "isolated"; writeScope: string[] }
  | { action: "open_review"; ticket: RepairTicket }
  | { action: "refresh_fixture_pr"; ticket: RepairTicket; requiresHumanReview: boolean };

function dispatchTicket(ticket: RepairTicket): DispatchDecision {
  if (ticket.kind === "block_fix_required") {
    return {
      action: "block_release",
      ticket,
    };
  }

  if (ticket.kind === "coverage_gap") {
    return {
      action: "spawn_worker",
      ticket,
      sandbox: "isolated",
      writeScope: ["regression-packs/", "tests/replay/"],
    };
  }

  if (ticket.kind === "quarantine_review") {
    return {
      action: "open_review",
      ticket,
    };
  }

  return {
    action: "refresh_fixture_pr",
    ticket,
    requiresHumanReview: true,
  };
}
~~~

原则很简单：

- 真正的安全决策漂移：阻断发布，不自动改策略。
- 覆盖缺口：可以派子 Agent 补 case，但写 scope 要限制在 tests/regression。
- flaky：隔离复核，先修确定性，不让它继续当发布证据。
- fixture refresh：可以开 PR，但必须保留 expected/actual diff 给人 review。

## 5. OpenClaw 课程 cron 的对应实践

拿我们这个课程 cron 做例子，一次失败可能长这样：

~~~json
{
  "caseId": "agent-course-publish-telegram-git",
  "status": "stable_fail",
  "expected": [
    "decision:allow_after_checks",
    "tool:read_tools_md",
    "tool:update_lesson",
    "tool:update_readme",
    "tool:message_send",
    "tool:git_push"
  ],
  "actual": [
    "decision:allow_after_checks",
    "tool:update_lesson",
    "tool:update_readme",
    "tool:message_send",
    "tool:git_push"
  ],
  "risk": "high"
}
~~~

决策没变，但少了 read_tools_md。这不是普通输出变化，而是可能重复授课的护栏缺失，所以 ticket 应该是：

~~~json
{
  "kind": "block_fix_required",
  "priority": "p1",
  "ownerHint": "agent-loop",
  "summary": "工具调用链缺失：read_tools_md",
  "releaseDecision": "block"
}
~~~

如果失败只是 README 目录描述从一句话变成两句话，而 Telegram 内容、已讲检查、git push gate 都没变，那才可能是 fixture_refresh。

## 6. Repair Ticket 应该包含什么

一个好 ticket 至少有这些字段：

~~~json
{
  "id": "rt_20260518_0930_001",
  "kind": "coverage_gap",
  "priority": "p1",
  "riskSurface": "external_side_effect.telegram_send",
  "caseId": "agent-course-publish-telegram-git",
  "ownerHint": "test-author",
  "blockedRelease": true,
  "writeScope": ["regression-packs/", "tests/replay/"],
  "evidenceRefs": [
    "replay-run:2026-05-18T09:30:00+11:00",
    "change-impact-map:abc123",
    "coverage-gate:missing-required-surface"
  ],
  "summary": "Telegram 外发风险面缺少回归 case",
  "nextAction": "补一个包含 message_send + release ledger + git proof 的 replay case"
}
~~~

这比“CI failed”强很多。子 Agent 或人类接到 ticket 后，能直接知道：

- 哪个 case 或风险面出了问题；
- 是否阻断发布；
- 允许写哪些目录；
- 证据在哪里；
- 下一步应该做什么。

## 7. 常见坑

第一，**把所有 stable_fail 都自动修**。

这是最危险的。安全决策漂移经常意味着策略被放宽了，自动修可能会把 expected 改成 actual，相当于把回归测试训练成沉默。

第二，**flaky case 只降级不建 ticket**。

flaky 被 quarantine 后，如果没有 ticket，它会永远躺在隔离区。正确做法是降级发布证据，同时生成 test-infra ticket。

第三，**ticket 没有 write scope**。

自动派给子 Agent 时必须限制写入范围。补测试的 worker 不应该顺手改 policy；修 policy 的 worker 不应该顺手刷新 fixture。

第四，**fixture refresh 没有人 review**。

刷新 fixture 本质上是在改“系统应该怎样表现”的证明，必须保留 expected/actual diff。

## 8. 小结

Regression replay 的终点不是 pass/fail，而是可执行的 close loop：

1. Replay runner 产出 stable_pass / stable_fail / flaky / missing_coverage。
2. Triage layer 把结果转成 RepairTicket。
3. Dispatch layer 决定 block、spawn worker、open review 或 fixture refresh PR。
4. Closeout gate 要求 ticket 带修复证据、回放证据和发布决策。

一句话：**成熟 Agent 不只会发现回归，还会把回归变成有 owner、有证据、有写入边界的修复任务。**
