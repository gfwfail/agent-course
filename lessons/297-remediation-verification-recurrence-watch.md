# 297. Agent 整改验证与复发监控（Remediation Verification & Recurrence Watch）

上一课讲了 CAPA：事故复盘后，要把 root cause 变成有 owner、有 due date、有证据的整改行动项。

但还有一个坑：**action item 被关闭了，不代表风险真的消失了。**

今天讲生产 Agent 的下一层闭环：Remediation Verification & Recurrence Watch。

一句话：

> 整改项关闭前要验证证据；关闭后还要进入观察期，确认同类信号没有复发。

---

## 1. 为什么“已关闭”不够

很多事故会二次发生，不是因为没人修，而是因为“修复完成”的定义太松：

- 只贴了 PR 链接，但没有确认合并和部署
- 测试通过了，但不是针对原始事故路径
- 配置改了，但生产环境漂移后又变回去
- 告警静音了，事故表面消失，根因还在
- 关闭后没有观察窗口，复发信号没人看

所以整改闭环应该分两段：

```text
CAPA action open
  -> submit evidence
  -> verify evidence quality
  -> close action
  -> start recurrence watch window
  -> watch incident signals / SLO / regression tests
  -> mark verified_resolved or reopen/escalate
```

核心原则：

- **Evidence validation**：证据要能证明“修了正确的问题”
- **Environment validation**：证据要来自正确环境，如 main、prod、当前部署版本
- **Recurrence watch**：关闭后继续观察一段时间
- **Auto reopen**：观察期内同类信号复发，自动重开整改项

---

## 2. 最小数据模型：RemediationVerification

```json
{
  "verificationId": "ver_act_20260512_001",
  "actionId": "act_inc_20260511_001_01",
  "incidentId": "inc_20260511_001",
  "status": "watching",
  "requiredEvidence": ["merged_pr", "deployed_sha", "regression_test", "slo_snapshot"],
  "evidence": [
    { "kind": "merged_pr", "url": "https://github.com/acme/agent/pull/42", "sha": "abc123" },
    { "kind": "regression_test", "command": "pytest tests/test_retry_idempotency.py", "passed": true }
  ],
  "watch": {
    "startedAt": "2026-05-12T20:30:00Z",
    "until": "2026-05-19T20:30:00Z",
    "signals": ["duplicate_message_detected", "idempotency_violation"],
    "maxAllowedOccurrences": 0
  },
  "decision": {
    "result": "accepted_for_watch",
    "reason": "PR is merged, deployed SHA matches production, regression test covers original retry path."
  }
}
```

状态机建议：

```text
pending_evidence -> rejected | accepted_for_watch -> verified_resolved | reopened
```

不要让整改项直接从 `open` 跳到 `closed`。更安全的语义是：

- `closed_pending_watch`：证据通过，但还在观察期
- `verified_resolved`：观察期结束，没有复发信号
- `reopened`：观察期内复发，回到 action tracker

---

## 3. learn-claude-code：Python 教学版

下面是一个最小验证器：检查证据是否齐全，并根据观察期信号决定关闭还是重开。

```python
# learn_claude_code/remediation_verifier.py
from __future__ import annotations

import time
from dataclasses import dataclass, field
from typing import Literal

VerificationStatus = Literal[
    "pending_evidence",
    "rejected",
    "watching",
    "verified_resolved",
    "reopened",
]


@dataclass
class Evidence:
    kind: str
    value: str
    metadata: dict = field(default_factory=dict)


@dataclass
class WatchSpec:
    until: float
    signals: list[str]
    max_allowed_occurrences: int = 0


@dataclass
class Verification:
    action_id: str
    incident_id: str
    required_evidence: list[str]
    evidence: list[Evidence]
    watch: WatchSpec
    status: VerificationStatus = "pending_evidence"
    reason: str = ""


class RemediationVerifier:
    def validate_evidence(self, verification: Verification) -> Verification:
        supplied = {item.kind for item in verification.evidence}
        missing = [kind for kind in verification.required_evidence if kind not in supplied]

        if missing:
            verification.status = "rejected"
            verification.reason = f"missing evidence: {', '.join(missing)}"
            return verification

        # 生产版这里会调用 GitHub、CI、部署平台、监控系统做 live check。
        for item in verification.evidence:
            if item.kind == "regression_test" and not item.metadata.get("passed"):
                verification.status = "rejected"
                verification.reason = "regression test did not pass"
                return verification

            if item.kind == "deployed_sha" and item.metadata.get("environment") != "prod":
                verification.status = "rejected"
                verification.reason = "deployed SHA is not from prod environment"
                return verification

        verification.status = "watching"
        verification.reason = "evidence accepted; recurrence watch started"
        return verification

    def evaluate_watch(self, verification: Verification, signal_counts: dict[str, int]) -> Verification:
        total = sum(signal_counts.get(signal, 0) for signal in verification.watch.signals)

        if total > verification.watch.max_allowed_occurrences:
            verification.status = "reopened"
            verification.reason = f"recurrence detected: {total} matching signal(s)"
            return verification

        if time.time() >= verification.watch.until:
            verification.status = "verified_resolved"
            verification.reason = "watch window completed without recurrence"
            return verification

        verification.status = "watching"
        verification.reason = "still inside watch window"
        return verification
```

用法：

```python
verifier = RemediationVerifier()
verification = Verification(
    action_id="act_001",
    incident_id="inc_001",
    required_evidence=["merged_pr", "deployed_sha", "regression_test"],
    evidence=[
        Evidence("merged_pr", "https://github.com/acme/agent/pull/42", {"merged": True}),
        Evidence("deployed_sha", "abc123", {"environment": "prod"}),
        Evidence("regression_test", "pytest tests/test_retry_idempotency.py", {"passed": True}),
    ],
    watch=WatchSpec(
        until=time.time() + 7 * 24 * 3600,
        signals=["duplicate_message_detected", "idempotency_violation"],
    ),
)

verification = verifier.validate_evidence(verification)
verification = verifier.evaluate_watch(verification, {"duplicate_message_detected": 0})
print(verification.status, verification.reason)
```

这个教学版的重点不是复杂，而是把“关闭整改项”拆成两个可测试动作：

1. 证据是否够？
2. 观察期是否真的没有复发？

---

## 4. pi-mono：TypeScript 生产版中间件

在 pi-mono 里，可以把它做成 IncidentAction 的关闭门控：

```ts
// packages/runtime/src/incident/RemediationVerificationGate.ts
import { z } from "zod";

const EvidenceSchema = z.object({
  kind: z.enum(["merged_pr", "deployed_sha", "regression_test", "slo_snapshot", "runbook_patch"]),
  ref: z.string(),
  metadata: z.record(z.unknown()).default({}),
});

const VerificationRequestSchema = z.object({
  actionId: z.string(),
  incidentId: z.string(),
  requiredEvidence: z.array(EvidenceSchema.shape.kind),
  evidence: z.array(EvidenceSchema),
  watchDays: z.number().int().min(1).max(30).default(7),
  recurrenceSignals: z.array(z.string()),
});

type VerificationDecision =
  | { type: "reject"; reason: string }
  | { type: "accept_for_watch"; watchUntil: Date; reason: string };

export class RemediationVerificationGate {
  constructor(
    private readonly github: { isMerged(ref: string): Promise<boolean> },
    private readonly deploys: { isShaInProd(sha: string): Promise<boolean> },
    private readonly tests: { passed(ref: string): Promise<boolean> },
  ) {}

  async evaluate(input: unknown): Promise<VerificationDecision> {
    const req = VerificationRequestSchema.parse(input);
    const supplied = new Set(req.evidence.map((e) => e.kind));

    for (const kind of req.requiredEvidence) {
      if (!supplied.has(kind)) {
        return { type: "reject", reason: `missing evidence: ${kind}` };
      }
    }

    for (const item of req.evidence) {
      if (item.kind === "merged_pr" && !(await this.github.isMerged(item.ref))) {
        return { type: "reject", reason: `PR is not merged: ${item.ref}` };
      }

      if (item.kind === "deployed_sha" && !(await this.deploys.isShaInProd(item.ref))) {
        return { type: "reject", reason: `SHA is not deployed to prod: ${item.ref}` };
      }

      if (item.kind === "regression_test" && !(await this.tests.passed(item.ref))) {
        return { type: "reject", reason: `regression test failed: ${item.ref}` };
      }
    }

    return {
      type: "accept_for_watch",
      watchUntil: new Date(Date.now() + req.watchDays * 24 * 3600 * 1000),
      reason: "all required evidence verified",
    };
  }
}
```

再加一个观察任务：

```ts
// packages/runtime/src/incident/RecurrenceWatchJob.ts
export class RecurrenceWatchJob {
  constructor(
    private readonly incidents: IncidentStore,
    private readonly metrics: MetricsClient,
    private readonly actions: ActionTracker,
  ) {}

  async run(now = new Date()) {
    const watches = await this.incidents.listActiveRecurrenceWatches();

    for (const watch of watches) {
      const count = await this.metrics.countSignals({
        names: watch.signals,
        since: watch.startedAt,
        until: now,
        labels: { incidentRootCause: watch.rootCauseKey },
      });

      if (count > watch.maxAllowedOccurrences) {
        await this.actions.reopen(watch.actionId, {
          reason: "recurrence signal detected during watch window",
          evidence: [{ kind: "metric_signal_count", value: String(count) }],
        });
        continue;
      }

      if (now >= watch.until) {
        await this.actions.markVerifiedResolved(watch.actionId, {
          reason: "watch window completed without recurrence",
        });
      }
    }
  }
}
```

关键点：

- `closeAction()` 不直接关闭，先调用 `RemediationVerificationGate`
- `accept_for_watch` 后写入 `RecurrenceWatch`
- Watch job 是幂等的：重复运行不会重复 reopen
- reopen 必须带 metric/test/log evidence

---

## 5. OpenClaw 实战：课程 cron 的整改闭环

拿 OpenClaw 课程 cron 举例：如果曾经发生“课程重复发送”，整改项不能只写：

> 已加 idempotency，关闭。

更可靠的关闭证据应该是：

```json
{
  "actionId": "act_course_duplicate_delivery",
  "requiredEvidence": [
    "merged_pr",
    "deployed_sha",
    "regression_test",
    "outbox_audit_sample"
  ],
  "watch": {
    "days": 7,
    "signals": [
      "same_lesson_sent_twice",
      "same_idempotency_key_committed_twice"
    ],
    "maxAllowedOccurrences": 0
  }
}
```

Agent 在关闭前要做 live checks：

```bash
# 1. PR 必须已 merge
gh pr view 42 --json state,mergeCommit --jq '{state, sha: .mergeCommit.oid}'

# 2. 当前部署版本必须包含 merge commit
openclaw deployment status --json | jq '.gitSha'

# 3. 回归测试必须覆盖原始事故路径
pytest tests/test_course_delivery_idempotency.py

# 4. Outbox 审计里不能出现重复 idempotencyKey
jq -r '.idempotencyKey' state/outbox.jsonl | sort | uniq -d
```

如果这些检查有一个失败，整改项不能关闭，只能回到 `blocked` 或 `in_progress`。

---

## 6. 实战建议

**1）证据要绑定原始事故路径**

不要只要求“有测试”，要要求测试能复现并覆盖原始失败条件：

```text
bad: npm test passed
good: retry after message timeout does not send duplicate Telegram message
```

**2）观察期按风险分层**

- P0/P1：7-30 天
- P2：3-7 天
- P3：1-3 天

**3）复发信号要具体**

不要写“系统异常”，要写可查询的指标、日志模式、错误码或回归测试名。

**4）自动重开要少而准**

重开不是为了制造噪音，而是为了防止“已关闭”的假安全感。最好用 dedupeKey：

```text
reopen:${actionId}:${rootCauseKey}:${watchWindowStart}
```

**5）关闭状态分两层**

- `closed_pending_watch`：修复证据通过
- `verified_resolved`：观察期通过

这样报表里能看清：哪些事故只是“修了”，哪些是真的“稳定了”。

---

## 7. 小结

今天这课的核心：

- CAPA action 不是贴个 PR 就结束
- 关闭前要验证证据是否完整、真实、来自正确环境
- 关闭后进入 recurrence watch window
- 观察期内同类信号复发，自动 reopen
- 最终状态应该是 `verified_resolved`，不是简单 `closed`

成熟 Agent 的事故闭环不是“复盘写完、任务关闭”，而是：**证据证明修复，时间证明没有复发。**
