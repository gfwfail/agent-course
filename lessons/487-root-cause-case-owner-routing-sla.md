# Agent 根因案件负责人路由与修复 SLA

> Root-Cause Case Owner Routing & Repair SLA Gate

第 486 课讲了长期回归套件失败如何先聚类，再判断根因优先级。但聚类只是把噪音变成了“几个真正要处理的问题”，还没有解决下一个关键问题：**这个根因案件到底交给谁，多久必须响应，超时后怎么升级？**

如果 `open_root_cause_case` 之后只是发一条群消息，系统很快会退化成人肉排班：

- 工具契约坏了，没人知道该找工具 owner 还是测试 owner；
- release blocking 失败已经卡住发布，但根因 ticket 没有 ack；
- side effect 风险被 quarantine 了，却没有恢复负责人；
- flake 反复重跑，没人判断是否应该降级或修 seed。

所以第 487 课补上 **Root-Cause Case Owner Routing & Repair SLA Gate**：把失败聚类转成可负责、可超时、可升级、可关闭的根因案件。

一句话：**聚类告诉你“同一个问题在哪里”，Owner Routing 告诉你“谁必须把它修完”。**

## 核心模型

把第 486 课的 `ClusterTriageReceipt` 往后接 5 个对象：

~~~text
ClusterTriageReceipt
  -> RootCauseCase
  -> OwnerRouteDecision
  -> RepairSlaLease
  -> EscalationDecision
  -> RootCauseCaseCloseoutReceipt
~~~

分工：

- `RootCauseCase`：一组同源失败的正式案件；
- `OwnerRouteDecision`：根据 risk surface、tool、runtime、assertion 选择 owner；
- `RepairSlaLease`：绑定 ack 截止时间、修复截止时间和升级策略；
- `EscalationDecision`：超时、无 owner、修复无进展时怎么升级；
- `RootCauseCaseCloseoutReceipt`：修复关闭时证明 replay、seed、发布闸门和 owner ack 都对齐。

关键点：**根因案件不是 issue 标题，而是一条带 owner、SLA、证据和关闭条件的工程契约。**

## learn-claude-code：Owner Routing 纯函数

教学版先做成纯函数。输入 cluster，输出 owner、SLA 和升级策略。

~~~python
# learn_claude_code/root_cause_owner_routing.py
from dataclasses import dataclass
from typing import Literal

RiskSurface = Literal[
    "tool_call",
    "memory",
    "side_effect",
    "release_gate",
    "ui",
]

OwnerRole = Literal[
    "tool_owner",
    "memory_owner",
    "release_engineer",
    "safety_engineer",
    "test_infra_owner",
    "manual_triage",
]

Severity = Literal["low", "medium", "high", "critical"]
EscalationAction = Literal["notify_owner", "reroute_backup", "page_release_owner", "manual_review"]


@dataclass(frozen=True)
class FailureCluster:
    cluster_key: str
    risk_surfaces: set[RiskSurface]
    tool_name: str | None
    blocks_release_count: int
    signal_count: int
    max_flake_score: float
    runtime_fingerprints: set[str]


@dataclass(frozen=True)
class OwnerRouteDecision:
    owner_role: OwnerRole
    severity: Severity
    ack_deadline_minutes: int
    repair_deadline_hours: int
    escalation_action: EscalationAction
    reason: str


def classify_severity(cluster: FailureCluster) -> Severity:
    if cluster.blocks_release_count > 0 and "side_effect" in cluster.risk_surfaces:
        return "critical"
    if cluster.blocks_release_count > 0:
        return "high"
    if cluster.signal_count >= 5:
        return "high"
    if cluster.max_flake_score > 0.75:
        return "low"
    return "medium"


def choose_owner(cluster: FailureCluster) -> OwnerRole:
    if "side_effect" in cluster.risk_surfaces:
        return "safety_engineer"
    if "release_gate" in cluster.risk_surfaces:
        return "release_engineer"
    if "memory" in cluster.risk_surfaces:
        return "memory_owner"
    if "tool_call" in cluster.risk_surfaces and cluster.tool_name:
        return "tool_owner"
    if cluster.max_flake_score > 0.75:
        return "test_infra_owner"
    return "manual_triage"


def build_sla(cluster: FailureCluster) -> OwnerRouteDecision:
    severity = classify_severity(cluster)
    owner = choose_owner(cluster)

    if severity == "critical":
        return OwnerRouteDecision(
            owner_role=owner,
            severity=severity,
            ack_deadline_minutes=10,
            repair_deadline_hours=4,
            escalation_action="page_release_owner",
            reason="release blocking + side effect risk requires immediate response",
        )

    if severity == "high":
        return OwnerRouteDecision(
            owner_role=owner,
            severity=severity,
            ack_deadline_minutes=30,
            repair_deadline_hours=24,
            escalation_action="reroute_backup",
            reason="release or broad regression impact",
        )

    if severity == "low":
        return OwnerRouteDecision(
            owner_role=owner,
            severity=severity,
            ack_deadline_minutes=240,
            repair_deadline_hours=72,
            escalation_action="notify_owner",
            reason="likely flaky, track but avoid noisy escalation",
        )

    return OwnerRouteDecision(
        owner_role=owner,
        severity=severity,
        ack_deadline_minutes=120,
        repair_deadline_hours=48,
        escalation_action="reroute_backup",
        reason="normal regression case",
    )
~~~

这里有两个细节：

- `owner_role` 不直接写人名，先写角色；角色再通过 on-call 表、代码 ownership、工具 registry 解析成具体负责人；
- `flake_score` 高不等于没人管，而是进入 `test_infra_owner`，因为“测试不稳定”本身就是生产问题。

## pi-mono：RootCauseCaseRouter

生产版要把 owner routing 放在事务里完成：创建 case、绑定 owner、写 SLA lease、发 outbox 通知，要么全部成功，要么都不成功。

~~~ts
// packages/agent-runtime/src/regression/RootCauseCaseRouter.ts
export type RiskSurface = "tool_call" | "memory" | "side_effect" | "release_gate" | "ui";
export type OwnerRole =
  | "tool_owner"
  | "memory_owner"
  | "release_engineer"
  | "safety_engineer"
  | "test_infra_owner"
  | "manual_triage";
export type Severity = "low" | "medium" | "high" | "critical";

export interface FailureCluster {
  clusterKey: string;
  riskSurfaces: RiskSurface[];
  toolName?: string;
  blocksReleaseCount: number;
  signalCount: number;
  maxFlakeScore: number;
  runtimeFingerprints: string[];
}

export interface RootCauseCase {
  caseId: string;
  clusterKey: string;
  status: "open" | "acknowledged" | "repairing" | "verifying" | "closed" | "escalated";
  severity: Severity;
  ownerRole: OwnerRole;
  ownerUserId: string;
  createdAt: string;
}

export interface RepairSlaLease {
  caseId: string;
  ownerUserId: string;
  ackBy: string;
  repairBy: string;
  escalationPolicy: "notify_owner" | "reroute_backup" | "page_release_owner" | "manual_review";
  leaseToken: string;
}

export class RootCauseCaseRouter {
  constructor(
    private readonly store: RootCauseStore,
    private readonly ownership: OwnershipResolver,
    private readonly clock: Clock,
  ) {}

  classifySeverity(cluster: FailureCluster): Severity {
    if (cluster.blocksReleaseCount > 0 && cluster.riskSurfaces.includes("side_effect")) {
      return "critical";
    }
    if (cluster.blocksReleaseCount > 0 || cluster.signalCount >= 5) return "high";
    if (cluster.maxFlakeScore > 0.75) return "low";
    return "medium";
  }

  chooseOwnerRole(cluster: FailureCluster): OwnerRole {
    if (cluster.riskSurfaces.includes("side_effect")) return "safety_engineer";
    if (cluster.riskSurfaces.includes("release_gate")) return "release_engineer";
    if (cluster.riskSurfaces.includes("memory")) return "memory_owner";
    if (cluster.riskSurfaces.includes("tool_call") && cluster.toolName) return "tool_owner";
    if (cluster.maxFlakeScore > 0.75) return "test_infra_owner";
    return "manual_triage";
  }

  slaFor(severity: Severity) {
    if (severity === "critical") {
      return { ackMinutes: 10, repairHours: 4, escalationPolicy: "page_release_owner" as const };
    }
    if (severity === "high") {
      return { ackMinutes: 30, repairHours: 24, escalationPolicy: "reroute_backup" as const };
    }
    if (severity === "low") {
      return { ackMinutes: 240, repairHours: 72, escalationPolicy: "notify_owner" as const };
    }
    return { ackMinutes: 120, repairHours: 48, escalationPolicy: "reroute_backup" as const };
  }

  async route(cluster: FailureCluster): Promise<{ case: RootCauseCase; sla: RepairSlaLease }> {
    const severity = this.classifySeverity(cluster);
    const ownerRole = this.chooseOwnerRole(cluster);
    const owner = await this.ownership.resolve({
      role: ownerRole,
      toolName: cluster.toolName,
      riskSurfaces: cluster.riskSurfaces,
    });
    const sla = this.slaFor(severity);
    const now = this.clock.now();

    return this.store.transaction(async (tx) => {
      const existing = await tx.findOpenCaseByClusterKey(cluster.clusterKey);
      if (existing) {
        await tx.appendClusterEvidence(existing.caseId, cluster);
        return {
          case: existing,
          sla: await tx.getSlaLease(existing.caseId),
        };
      }

      const rootCase = await tx.createRootCauseCase({
        caseId: crypto.randomUUID(),
        clusterKey: cluster.clusterKey,
        status: "open",
        severity,
        ownerRole,
        ownerUserId: owner.userId,
        createdAt: now.toISOString(),
      });

      const lease = await tx.createRepairSlaLease({
        caseId: rootCase.caseId,
        ownerUserId: owner.userId,
        ackBy: this.clock.addMinutes(now, sla.ackMinutes).toISOString(),
        repairBy: this.clock.addHours(now, sla.repairHours).toISOString(),
        escalationPolicy: sla.escalationPolicy,
        leaseToken: crypto.randomUUID(),
      });

      await tx.enqueueNotification({
        type: "root_cause_case_assigned",
        targetUserId: owner.userId,
        caseId: rootCase.caseId,
        severity,
      });

      return { case: rootCase, sla: lease };
    });
  }
}
~~~

注意 `findOpenCaseByClusterKey`：同一个 cluster 再次失败时不要新开 case，而是把证据 append 到现有 case。这样第 486 课的“聚类去噪”不会在第 487 课又变成“工单重复”。

## SLA 不是提醒，是状态机

一个根因 case 至少要有这几个状态：

~~~text
open
  -> acknowledged
  -> repairing
  -> verifying
  -> closed
  -> escalated
~~~

每个状态都有 deadline：

- `open` 必须在 `ackBy` 前变成 `acknowledged`；
- `repairing` 必须在 `repairBy` 前进入 `verifying` 或写明阻塞原因；
- `verifying` 必须跑过 regression seed / replay matrix；
- `closed` 必须写 `RootCauseCaseCloseoutReceipt`；
- `escalated` 必须带 `EscalationDecision`，不能只说“找人看看”。

SLA 的价值不是催人，而是让系统知道 **自动化到哪里已经不该继续假装自动化**。

## OpenClaw 实战：Cron 课程任务也是 Root-Cause Case

OpenClaw 的课程 Cron 很适合类比这个模式：

~~~text
cron run
  -> 写 lesson
  -> 更新 README / TOOLS
  -> 发 Telegram
  -> git commit / push
  -> final closeout
~~~

如果其中一个步骤失败，也应该能路由：

- Telegram 发送失败：owner 是 channel/tool owner，SLA 是尽快补发或写失败说明；
- git push 失败：owner 是 repo credential owner，先检查 `gh auth switch --user gfwfail`；
- README 更新失败：owner 是课程维护者，必须补齐目录；
- TOOLS 更新失败：owner 是记忆维护者，否则下次会重复选题。

一个轻量 OpenClaw 文件式状态可以这样存：

~~~json
{
  "caseId": "rcase_487_telegram_send_failed",
  "clusterKey": "course_cron|telegram|-5115329245|send_failed",
  "ownerRole": "tool_owner",
  "status": "open",
  "ackBy": "2026-06-05T02:40:00Z",
  "repairBy": "2026-06-05T03:30:00Z",
  "closeoutRequires": [
    "message_id_present",
    "git_commit_pushed",
    "README_contains_lesson",
    "TOOLS_contains_topic"
  ]
}
~~~

这就是把“自动化任务失败了”升级成“可恢复的工程案件”，而不是让下一轮 cron 靠运气重试。

## Closeout：关闭案件前必须证明什么

根因 case 关闭时，至少要证明四件事：

~~~ts
export interface RootCauseCaseCloseoutReceipt {
  caseId: string;
  clusterKey: string;
  closedBy: string;
  fixCommitSha?: string;
  replayRunId: string;
  replayPassed: boolean;
  affectedSeeds: string[];
  seedsUpdated: boolean;
  releaseGateUnblocked: boolean;
  ownerAckedCloseout: boolean;
  residualRisk?: string;
  closedAt: string;
}

export function canCloseRootCauseCase(receipt: RootCauseCaseCloseoutReceipt): boolean {
  if (!receipt.replayPassed) return false;
  if (!receipt.seedsUpdated) return false;
  if (!receipt.releaseGateUnblocked) return false;
  if (!receipt.ownerAckedCloseout) return false;
  return true;
}
~~~

这里的原则是：**修复合并不等于根因关闭。** 只有回放通过、受影响 seed 更新、release gate 解除、owner 确认，case 才能 close。

如果有残余风险，就不要假装没有风险：

- 低风险：写 `residualRisk`，进入观察窗口；
- 中风险：延长验证，不能 close；
- 高风险：保持 release gate blocked；
- critical：升级 incident。

## 常见坑

1. **只按 tool name 路由**
   同一个工具失败可能是权限、schema、运行时、外部 API、测试数据问题。tool name 只能作为信号之一。

2. **SLA 只发提醒不改变状态**
   超时必须产生 `EscalationDecision`，并改变 case 的可执行路径。

3. **同一 cluster 反复开新 case**
   必须用 `clusterKey` 做去重，把新证据 append 到旧 case。

4. **close 只看 PR merged**
   PR merged 只说明代码变了，不说明回归套件、release gate、外部现实都恢复了。

5. **manual_triage 没有 deadline**
   人工兜底也要有 ack 和 repair deadline，否则就是黑洞。

## 和前几课的关系

- 第 483 课 Burn-in：新 seed 先烧出稳定性；
- 第 484 课 Repair Reactivation：失败 seed 修复后要重新证明；
- 第 485 课 Re-entry：再激活 seed 回长期套件；
- 第 486 课 Failure Clustering：长期套件失败先聚类；
- 第 487 课 Owner Routing：聚类后把根因案件交给正确 owner，并用 SLA 推到关闭。

这条链路组合起来，长期回归系统才不是“测试失败了喊一嗓子”，而是：

~~~text
失败信号
  -> 聚类
  -> 根因案件
  -> owner + SLA
  -> 修复验证
  -> closeout receipt
  -> 回归资产更新
~~~

## 小结

Root-Cause Case Owner Routing 的核心不是“自动分配 issue”，而是把根因修复变成一条可审计状态机：

- 谁负责；
- 多久 ack；
- 多久修；
- 超时怎么升级；
- 什么证据允许关闭；
- 关闭后哪些回归资产要更新。

成熟 Agent 的测试系统不只是发现失败，而是能把失败推到正确的人、正确的队列、正确的截止时间，并最终用证据关闭。没有 owner 和 SLA 的根因 case，只是换了个名字的告警噪音。
