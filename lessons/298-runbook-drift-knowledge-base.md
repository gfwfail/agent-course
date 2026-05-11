# 298. Agent 运行知识库与 Runbook 漂移检测（Runbook Drift & Operational Knowledge Base）

上一课讲了整改验证与复发监控：整改项不是贴个 PR 就结束，而是要验证、观察、确认没有复发。

但闭环还有最后一个容易被忽略的坑：**事故修完了，Runbook 和运行知识库没更新。**

结果下一次值班 Agent 还是按旧剧本执行：

- 旧命令已经改名
- 旧 API endpoint 已经下线
- 旧告警阈值不再适用
- 旧恢复步骤少了新审批/新证据要求
- 复盘里明明写了新规则，但执行器根本不知道

今天讲 Runbook Drift & Operational Knowledge Base。

一句话：

> 把事故经验沉淀成可执行知识，并定期验证 Runbook 与真实系统没有漂移。

---

## 1. Runbook Drift 是什么

Runbook 漂移指的是：**文档/剧本描述的世界，和生产真实世界不一致。**

常见漂移：

```text
structural drift   : 工具、命令、API、路径变了
policy drift       : 审批、权限、冻结窗口规则变了
environment drift  : prod/staging 配置不一致
knowledge drift    : 事故复盘产生的新规则没写回知识库
ownership drift    : owner/team/channel 已失效
threshold drift    : SLO/告警阈值和当前业务规模不匹配
```

成熟 Agent 不能只“会读 Runbook”，还要会问：

- 这个 Runbook 最近验证过吗？
- 它依赖的工具 schema 还存在吗？
- 它的风险策略和当前 Policy-as-Code 一致吗？
- 最近事故有没有要求修改它？
- 执行前的 preflight 能不能证明它仍然可用？

---

## 2. 最小数据模型：OperationalKnowledgeEntry

```json
{
  "id": "kb_inc_20260512_retry_idempotency",
  "kind": "incident_lesson",
  "scope": "tool:send_message",
  "source": {
    "incidentId": "inc_20260512_001",
    "reviewId": "review_inc_20260512_001",
    "actionIds": ["act_001", "act_002"]
  },
  "rule": {
    "trigger": "retry_after_delivery_unknown",
    "requiredCheck": "lookup_delivery_receipt_before_retry",
    "defaultDecision": "defer_or_manual_review"
  },
  "appliesToRunbooks": ["rb_message_delivery_recovery"],
  "verifiedAt": "2026-05-12T23:30:00Z",
  "expiresAt": "2026-08-12T23:30:00Z"
}
```

Runbook 本身也要带依赖指纹：

```json
{
  "runbookId": "rb_message_delivery_recovery",
  "version": "2026-05-12.1",
  "owner": "platform-oncall",
  "dependsOn": {
    "tools": ["message.send", "message.poll"],
    "policies": ["side_effect_approval", "idempotency_required"],
    "knowledgeEntries": ["kb_inc_20260512_retry_idempotency"]
  },
  "lastValidatedAt": "2026-05-12T23:30:00Z"
}
```

关键是：知识库不要只存 Markdown。它至少要结构化到 Agent 可以执行检查。

---

## 3. learn-claude-code：Python 教学版

下面是一个最小 Drift Checker：验证 Runbook 依赖的工具、策略和事故知识是否仍然存在。

```python
# learn_claude_code/runbook_drift_checker.py
from __future__ import annotations

import time
from dataclasses import dataclass, field
from typing import Literal

DriftKind = Literal["missing_tool", "missing_policy", "missing_knowledge", "stale_validation"]


@dataclass
class Runbook:
    id: str
    tools: list[str]
    policies: list[str]
    knowledge_entries: list[str]
    last_validated_at: float
    max_validation_age_seconds: int = 30 * 24 * 3600


@dataclass
class RegistrySnapshot:
    tools: set[str]
    policies: set[str]
    knowledge_entries: set[str]


@dataclass
class DriftFinding:
    kind: DriftKind
    ref: str
    severity: Literal["warn", "block"]
    message: str


class RunbookDriftChecker:
    def check(self, runbook: Runbook, snapshot: RegistrySnapshot) -> list[DriftFinding]:
        findings: list[DriftFinding] = []

        for tool in runbook.tools:
            if tool not in snapshot.tools:
                findings.append(DriftFinding(
                    kind="missing_tool",
                    ref=tool,
                    severity="block",
                    message=f"Runbook depends on missing tool: {tool}",
                ))

        for policy in runbook.policies:
            if policy not in snapshot.policies:
                findings.append(DriftFinding(
                    kind="missing_policy",
                    ref=policy,
                    severity="block",
                    message=f"Runbook depends on missing policy: {policy}",
                ))

        for entry in runbook.knowledge_entries:
            if entry not in snapshot.knowledge_entries:
                findings.append(DriftFinding(
                    kind="missing_knowledge",
                    ref=entry,
                    severity="warn",
                    message=f"Linked knowledge entry not found: {entry}",
                ))

        age = time.time() - runbook.last_validated_at
        if age > runbook.max_validation_age_seconds:
            findings.append(DriftFinding(
                kind="stale_validation",
                ref=runbook.id,
                severity="warn",
                message=f"Runbook validation is stale: {int(age / 86400)} days old",
            ))

        return findings

    def decision(self, findings: list[DriftFinding]) -> Literal["allow", "dry_run", "block"]:
        if any(f.severity == "block" for f in findings):
            return "block"
        if findings:
            return "dry_run"
        return "allow"
```

使用：

```python
checker = RunbookDriftChecker()
runbook = Runbook(
    id="rb_message_delivery_recovery",
    tools=["message.send", "message.poll"],
    policies=["side_effect_approval", "idempotency_required"],
    knowledge_entries=["kb_inc_20260512_retry_idempotency"],
    last_validated_at=time.time() - 45 * 24 * 3600,
)

snapshot = RegistrySnapshot(
    tools={"message.send", "message.poll", "git.push"},
    policies={"side_effect_approval", "idempotency_required"},
    knowledge_entries={"kb_inc_20260512_retry_idempotency"},
)

findings = checker.check(runbook, snapshot)
print(checker.decision(findings))  # dry_run: 依赖存在，但验证过期
```

这就是 Runbook 执行前最小闸门：

```text
no drift      -> allow
warn drift    -> dry_run / require fresh validation
block drift   -> stop and escalate
```

---

## 4. pi-mono：TypeScript 生产版中间件

生产系统里，Runbook Drift Checker 应该挂在执行器前面。

```ts
// packages/runtime/src/runbooks/RunbookDriftMiddleware.ts
import { z } from "zod";

const RunbookSchema = z.object({
  id: z.string(),
  version: z.string(),
  dependsOn: z.object({
    tools: z.array(z.string()).default([]),
    policies: z.array(z.string()).default([]),
    knowledgeEntries: z.array(z.string()).default([]),
  }),
  lastValidatedAt: z.string().datetime(),
  maxValidationAgeDays: z.number().int().positive().default(30),
});

type DriftDecision = {
  action: "allow" | "dry_run" | "block";
  findings: Array<{
    kind: "missing_tool" | "missing_policy" | "missing_knowledge" | "stale_validation";
    ref: string;
    severity: "warn" | "block";
    message: string;
  }>;
};

export class RunbookDriftMiddleware {
  constructor(
    private readonly registries: {
      hasTool(name: string): Promise<boolean>;
      hasPolicy(name: string): Promise<boolean>;
      hasKnowledgeEntry(id: string): Promise<boolean>;
    },
    private readonly audit: { record(event: unknown): Promise<void> },
  ) {}

  async check(rawRunbook: unknown): Promise<DriftDecision> {
    const runbook = RunbookSchema.parse(rawRunbook);
    const findings: DriftDecision["findings"] = [];

    for (const tool of runbook.dependsOn.tools) {
      if (!(await this.registries.hasTool(tool))) {
        findings.push({
          kind: "missing_tool",
          ref: tool,
          severity: "block",
          message: `Tool no longer exists: ${tool}`,
        });
      }
    }

    for (const policy of runbook.dependsOn.policies) {
      if (!(await this.registries.hasPolicy(policy))) {
        findings.push({
          kind: "missing_policy",
          ref: policy,
          severity: "block",
          message: `Policy no longer exists: ${policy}`,
        });
      }
    }

    for (const entry of runbook.dependsOn.knowledgeEntries) {
      if (!(await this.registries.hasKnowledgeEntry(entry))) {
        findings.push({
          kind: "missing_knowledge",
          ref: entry,
          severity: "warn",
          message: `Knowledge entry is missing: ${entry}`,
        });
      }
    }

    const ageMs = Date.now() - Date.parse(runbook.lastValidatedAt);
    const maxAgeMs = runbook.maxValidationAgeDays * 24 * 60 * 60 * 1000;
    if (ageMs > maxAgeMs) {
      findings.push({
        kind: "stale_validation",
        ref: runbook.id,
        severity: "warn",
        message: `Runbook validation is older than ${runbook.maxValidationAgeDays} days`,
      });
    }

    const action = findings.some((f) => f.severity === "block")
      ? "block"
      : findings.length > 0
        ? "dry_run"
        : "allow";

    await this.audit.record({
      type: "runbook_drift_checked",
      runbookId: runbook.id,
      version: runbook.version,
      action,
      findings,
      checkedAt: new Date().toISOString(),
    });

    return { action, findings };
  }
}
```

执行器里：

```ts
const decision = await driftMiddleware.check(runbook);

if (decision.action === "block") {
  throw new Error(`Runbook drift blocks execution: ${JSON.stringify(decision.findings)}`);
}

const mode = decision.action === "dry_run" ? "dry_run" : "execute";
await runbookRunner.run(runbook, { mode });
```

重点：Runbook 漂移不是执行失败后才发现，而是执行前就做 preflight。

---

## 5. OpenClaw 实战：把 MEMORY / lessons / Runbook 串起来

OpenClaw 里可以用很轻量的文件系统实现运行知识库：

```text
.openclaw/workspace/
  ops-kb/
    incidents/
      inc_20260512_001.json
    lessons/
      kb_inc_20260512_retry_idempotency.json
    runbooks/
      rb_message_delivery_recovery.json
    drift-reports/
      2026-05-12.json
```

Cron 或 heartbeat 定期跑：

```bash
python scripts/check_runbook_drift.py \
  --runbooks ops-kb/runbooks \
  --knowledge ops-kb/lessons \
  --tool-schema .openclaw/tool-schema.json \
  --policy policies/tool-policy.json \
  --out ops-kb/drift-reports/$(date +%F).json
```

更实用的做法：每次事故复盘关闭时，强制要求：

```text
Incident Review completed
  -> generate knowledge entry
  -> link affected runbooks
  -> patch runbook dependencies
  -> run drift checker
  -> archive evidence
```

如果复盘产生了新规则，但没有任何 Runbook 引用它，就要告警：

```text
warning: knowledge entry kb_inc_20260512_retry_idempotency is orphaned
reason : incident lesson exists, but no runbook or policy consumes it
risk   : same mistake can repeat because execution path never learned it
```

这一步很关键：

> 写进复盘不算学会；进入执行路径才算学会。

---

## 6. 设计要点

### 6.1 知识库要分层

建议至少四类：

```text
facts      : 当前系统事实，如 endpoint、owner、env、tool schema
rules      : 必须遵守的策略，如审批、幂等、冻结窗口
lessons    : 事故经验，如触发信号、根因、新检查项
runbooks   : 可执行流程，如恢复、回滚、巡检、发布
```

不要把所有东西都塞进一个 MEMORY.md。MEMORY 适合人读，执行器需要结构化文件。

### 6.2 Drift 分级处理

```text
missing critical tool/policy      -> block
stale validation                  -> dry_run + fresh validation
missing optional knowledge link   -> warn
owner/channel stale               -> require confirmation
threshold suspicious              -> manual review
```

### 6.3 执行前检查，不是执行后复盘

Runbook Drift Checker 应该在这些时机跑：

- cron 自动任务启动前
- 高风险工具调用前
- 事故指挥中心升级时
- 发布/回滚前
- 复盘 action 关闭前
- 每周定期扫描全量 Runbook

---

## 7. 常见坑

**坑 1：知识库只有自然语言。**

自然语言适合解释，不适合强制执行。关键字段必须结构化。

**坑 2：Runbook 没有依赖声明。**

没有 `dependsOn.tools/policies/knowledgeEntries`，就没法自动判断漂移。

**坑 3：复盘和执行系统断开。**

复盘写得再好，如果没有进入 Policy、Runbook、Regression Test，就只是文档。

**坑 4：过期不算错误。**

运行环境变化很快。`lastValidatedAt` 太旧，本身就是风险信号。

---

## 8. 小结

今天这课的核心：

- Runbook Drift = 剧本描述和真实系统不一致
- Operational KB 要结构化，能被 Agent 执行检查
- Runbook 要声明依赖：tools / policies / knowledge entries
- 执行前用 Drift Checker 做 allow / dry_run / block
- 事故经验只有进入执行路径，才算真正被系统学会

一句话收尾：

> 成熟 Agent 的记忆不是“我知道了”，而是“下次执行前，我会自动检查这条经验是否适用”。
