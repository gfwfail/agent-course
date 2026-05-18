# 350. Agent 修复关闭、SLO 归档与学习记忆（Repair Closeout, SLO Archive & Learning Memory）

上一课讲了 Deployment Promotion：补丁合并之后，要通过 canary、stable、post-deploy probes 和 error budget 观测，证明线上问题真的消失。

今天继续把自动修复闭环收尾：**线上观测通过还不等于工单可以被忘掉；Agent 需要把修复结果、SLO 证据、长期学习和后续检查一起归档，才能安全关闭 repair ticket。**

成熟链路应该长这样：

~~~text
Repair Ticket
  -> Patch Intake
  -> Mainline Sync
  -> Deployment Promotion
  -> Post-Deploy Observation
  -> Repair Closeout
  -> SLO Archive
  -> Learning Memory
  -> Follow-up Watch
~~~

核心原则：**关闭不是“状态改成 done”，而是证明问题已恢复、风险已收口、知识已回灌、未来会自动防复发。**

## 1. 为什么 promote 之后还要 closeout

很多自动修复系统会在 promote 成功时直接关 ticket。这会留下几个隐患：

- 线上指标恢复了，但没有保存当时的 SLO 证据。
- ticket 关了，但 runbook、policy、regression pack 没更新。
- 只记录“修好了”，没记录为什么确认已修好。
- 后续 24 小时复发时，Agent 找不到上一次修复的判断依据。
- 同类问题下次再来，系统还是从零分析。

所以 closeout gate 要回答四个问题：

- **resolved**：原始失败信号是否消失？
- **contained**：临时冻结、降级、限流是否已经解除或保留原因明确？
- **learned**：新的测试、策略、runbook 或 memory 是否已经回灌？
- **watched**：后续观察窗口是否已安排，复发时知道找谁、看什么、回滚到哪里？

这四项少一项，都只是“暂时看起来好了”。

## 2. learn-claude-code：最小 Closeout Gate

教学版先把 closeout 建模成一个纯函数：输入 repair ticket、SLO 快照、学习回灌状态，输出 close / hold / escalate。

~~~python
from dataclasses import dataclass
from enum import Enum

class CloseoutAction(str, Enum):
    CLOSE = "close"
    HOLD = "hold"
    ESCALATE = "escalate"

@dataclass
class RepairTicket:
    id: str
    severity: str
    root_cause: str
    fix_commit: str
    rollback_ref: str | None

@dataclass
class SloSnapshot:
    error_rate: float
    p95_ms: int
    sample_size: int
    observation_minutes: int

@dataclass
class LearningBackfill:
    regression_case_added: bool
    runbook_updated: bool
    policy_updated: bool
    memory_written: bool

@dataclass
class CloseoutDecision:
    action: CloseoutAction
    reasons: list[str]

def decide_closeout(
    ticket: RepairTicket,
    slo: SloSnapshot,
    learning: LearningBackfill,
) -> CloseoutDecision:
    reasons: list[str] = []

    if slo.sample_size < 100:
        reasons.append("not enough production samples")

    if slo.observation_minutes < 30:
        reasons.append("observation window too short")

    if slo.error_rate > 0.01:
        reasons.append("error rate still above recovery threshold")

    if slo.p95_ms > 1500:
        reasons.append("latency still degraded")

    missing_learning = []
    if not learning.regression_case_added:
        missing_learning.append("regression case")
    if not learning.runbook_updated:
        missing_learning.append("runbook")
    if not learning.memory_written:
        missing_learning.append("memory")

    if missing_learning:
        reasons.append("missing learning backfill: " + ", ".join(missing_learning))

    if ticket.severity in {"P0", "P1"} and ticket.rollback_ref is None:
        return CloseoutDecision(
            CloseoutAction.ESCALATE,
            reasons + ["high severity ticket has no rollback reference"],
        )

    if reasons:
        return CloseoutDecision(CloseoutAction.HOLD, reasons)

    return CloseoutDecision(CloseoutAction.CLOSE, ["recovered and learning archived"])
~~~

这个 gate 的重点不是复杂，而是把“关单”从主观判断变成可测试条件。

如果有人问为什么不能关闭，reasons 就是下一步工作清单。

## 3. pi-mono：CloseoutBundle 作为长期证据

生产版不要只在 ticket 里写一句 resolved。应该产出一个 CloseoutBundle，既给人读，也给后续 Agent 复用。

~~~ts
type CloseoutBundle = {
  ticketId: string;
  fixCommit: string;
  deployedVersion: string;
  resolvedAt: string;
  sloArchive: {
    window: { startedAt: string; endedAt: string };
    errorRate: number;
    p95Ms: number;
    sampleSize: number;
    probeEvidenceIds: string[];
  };
  learning: {
    regressionCases: string[];
    runbookPatches: string[];
    policyPatches: string[];
    memoryRefs: string[];
  };
  followUpWatch: {
    until: string;
    probes: string[];
    rollbackRef: string;
  };
};

type CloseoutResult =
  | { status: "closed"; bundle: CloseoutBundle }
  | { status: "hold"; missing: string[] }
  | { status: "manual_review"; reason: string };

function validateCloseoutBundle(bundle: CloseoutBundle): CloseoutResult {
  const missing: string[] = [];

  if (bundle.sloArchive.sampleSize < 100) {
    missing.push("sloArchive.sampleSize");
  }

  if (bundle.sloArchive.probeEvidenceIds.length === 0) {
    missing.push("sloArchive.probeEvidenceIds");
  }

  if (bundle.learning.regressionCases.length === 0) {
    missing.push("learning.regressionCases");
  }

  if (bundle.learning.runbookPatches.length === 0) {
    missing.push("learning.runbookPatches");
  }

  if (!bundle.followUpWatch.rollbackRef) {
    return {
      status: "manual_review",
      reason: "follow-up watch has no rollback reference",
    };
  }

  if (missing.length > 0) {
    return { status: "hold", missing };
  }

  return { status: "closed", bundle };
}
~~~

CloseoutBundle 解决的是长期可解释性：

- 事故为什么被判定恢复？
- 哪个版本修复了它？
- 哪些 probe 证明线上正常？
- 哪些 regression case 防止下次复发？
- 如果 24 小时内复发，应该回滚到哪里？

这比 ticket comment 更稳定，也更适合 Agent 自动读取。

## 4. OpenClaw：把 closeout 接到 cron / memory / Git 证据

OpenClaw 的课程 cron 本身就是一个小型自动执行系统。一次课程成功后，不能只说“发了”，而要形成 closeout 证据：

~~~json
{
  "lesson": "350-repair-closeout-slo-learning-memory",
  "git": {
    "commit": "abc123",
    "remoteContainsCommit": true
  },
  "telegram": {
    "target": "-5115329245",
    "messageId": "12184"
  },
  "learning": {
    "readmeUpdated": true,
    "toolsTopicRecorded": true,
    "dailyMemoryWritten": true
  },
  "followUpWatch": {
    "nextCronMustAvoidTopic": true
  }
}
~~~

这就是 Repair Closeout 在日常自动化里的简化版：

- Git 证据证明 artifact 已发布。
- Telegram messageId 证明外部动作已完成。
- README / TOOLS / memory 证明知识已回灌。
- 下一轮 cron 的去重检查就是 follow-up watch。

## 5. 实战检查清单

做自动修复系统时，可以把 closeout gate 固化成这几个条件：

- 必须有 fix commit、deploy version、rollback ref。
- 必须有 post-deploy SLO window，不接受单点成功。
- 必须有 regression case 或明确说明为什么不能自动回归。
- 必须更新 runbook / policy / memory 中至少一个长期知识位置。
- 必须安排复发观察窗口，且超时后自动关闭或升级。
- P0/P1 关闭必须保留人工可读摘要和机器可读 bundle。

一句话总结：**自动修复的最后一步不是关闭 ticket，而是把“这次为什么修好、以后怎么避免、复发怎么处理”变成系统能执行的记忆。**
