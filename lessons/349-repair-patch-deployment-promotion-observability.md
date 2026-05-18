# 349. Agent 修复补丁部署晋级与发布观测（Repair Patch Deployment Promotion & Observability）

上一课讲了 Mainline Sync：worker patch 通过 intake 之后，要先同步最新 main，再重选 replay cases、重跑 verification，证明修复在真实主线基线上仍然成立。

今天继续往后走：**补丁合进主线不等于事故修好了；自动修复还需要一个部署晋级链路，把 merged commit 变成 canary、stable、promoted 或 rollback 的可审计状态。**

成熟链路应该长这样：

~~~text
Patch Intake
  -> Mainline Sync
  -> Replay Confirmation
  -> Merge
  -> Deployment Promotion
  -> Post-Deploy Observation
  -> Promote / Rollback / Hold
~~~

核心原则：**修复补丁的终点不是 merge，而是线上观测证明问题已消失，并且没有引入新的风险。**

## 1. 为什么 merge 之后还要晋级观测

很多 Agent 自动修复系统会卡在“测试通过 + PR 合并”。这对代码仓库来说够了，但对真实系统不够。

合并之后仍然有几类风险：

- 修复没有部署到实际运行的环境。
- 部署成功了，但只有部分 worker / region / tenant 生效。
- 回归测试通过了，真实流量下还是触发同类错误。
- 修复了原 bug，却让延迟、错误率、成本或工具拒绝率变差。
- Agent 只记得“我 merge 了”，但几小时后没人能证明线上是否真的恢复。

所以 Deployment Promotion 不是 DevOps 装饰，而是自动修复的闭环：**代码证据证明可以发布，运行证据证明已经修好。**

## 2. learn-claude-code：最小部署晋级状态机

教学版可以先不接 Kubernetes、Vercel 或 Laravel Cloud，只把部署状态建模清楚。

~~~python
from dataclasses import dataclass
from enum import Enum

class PromotionAction(str, Enum):
    PROMOTE = "promote"
    HOLD = "hold"
    ROLLBACK = "rollback"
    ESCALATE = "escalate"

@dataclass
class Deployment:
    ticket_id: str
    commit_sha: str
    environment: str
    version: str
    rollout_percent: int

@dataclass
class Observation:
    probe: str
    status: str  # pass | fail | unknown
    error_rate: float
    p95_ms: int
    sample_size: int

@dataclass
class PromotionDecision:
    action: PromotionAction
    reasons: list[str]

def decide_promotion(
    deployment: Deployment,
    observations: list[Observation],
    min_samples: int = 50,
) -> PromotionDecision:
    reasons: list[str] = []

    if deployment.rollout_percent < 10:
        reasons.append("rollout is too small to trust")

    weak = [o.probe for o in observations if o.sample_size < min_samples]
    failed = [o.probe for o in observations if o.status == "fail"]
    unknown = [o.probe for o in observations if o.status == "unknown"]
    unhealthy = [
        o.probe for o in observations
        if o.error_rate > 0.02 or o.p95_ms > 2000
    ]

    if failed or unhealthy:
        return PromotionDecision(
            PromotionAction.ROLLBACK,
            reasons + [f"bad probes: {', '.join(failed + unhealthy)}"],
        )

    if unknown:
        return PromotionDecision(
            PromotionAction.ESCALATE,
            reasons + [f"unknown probes: {', '.join(unknown)}"],
        )

    if weak or reasons:
        return PromotionDecision(
            PromotionAction.HOLD,
            reasons + [f"insufficient samples: {', '.join(weak)}"],
        )

    return PromotionDecision(PromotionAction.PROMOTE, ["all probes healthy"])
~~~

这个例子故意把四种结果分开：

- promote：观测足够，指标健康，可以扩大流量或关闭工单。
- hold：没有坏信号，但样本不够，继续观察。
- rollback：已有明确坏信号，自动回滚或冻结。
- escalate：观测链路本身不可信，交给人或更强 Agent 处理。

这比“部署成功”更接近生产事实。

## 3. pi-mono：把 Agent.subscribe 变成发布观测入口

pi-mono 的 Agent 可以通过 subscribe 监听事件；生产版可以在事件外层挂一个 Promotion Observer，把 agent 事件、工具结果和部署探针统一写入 release evidence。

~~~ts
type DeployStage = "merged" | "deployed" | "canary" | "stable";

type ReleaseCandidate = {
  ticketId: string;
  commitSha: string;
  stage: DeployStage;
  environment: string;
  evidence: string[];
};

type ProbeResult = {
  name: string;
  status: "pass" | "fail" | "unknown";
  errorRate: number;
  p95Ms: number;
  sampleSize: number;
};

type PromotionDecision =
  | { action: "promote"; nextStage: DeployStage; evidence: string[] }
  | { action: "hold"; reason: string; evidence: string[] }
  | { action: "rollback"; reason: string; evidence: string[] }
  | { action: "manual_review"; reason: string; evidence: string[] };

function decideReleasePromotion(
  candidate: ReleaseCandidate,
  probes: ProbeResult[],
): PromotionDecision {
  const evidence = [
    ...candidate.evidence,
    "commit:" + candidate.commitSha,
    "env:" + candidate.environment,
  ];

  const failed = probes.filter((p) => p.status === "fail" || p.errorRate > 0.02);
  if (failed.length > 0) {
    return {
      action: "rollback",
      reason: "failed post-deploy probes: " + failed.map((p) => p.name).join(", "),
      evidence,
    };
  }

  const unknown = probes.filter((p) => p.status === "unknown");
  if (unknown.length > 0) {
    return {
      action: "manual_review",
      reason: "probe state unknown: " + unknown.map((p) => p.name).join(", "),
      evidence,
    };
  }

  const weak = probes.filter((p) => p.sampleSize < 50);
  if (weak.length > 0) {
    return {
      action: "hold",
      reason: "not enough samples: " + weak.map((p) => p.name).join(", "),
      evidence,
    };
  }

  return {
    action: "promote",
    nextStage: candidate.stage === "canary" ? "stable" : "deployed",
    evidence,
  };
}
~~~

结合 Agent 事件流时，不要只记录最终 action。要记录完整证据链：

~~~text
repair_ticket_created
patch_intake_pass
mainline_replay_pass
commit_merged
deployment_started
canary_probe_pass
promotion_decision_promote
stable_probe_pass
repair_ticket_verified_resolved
~~~

这样以后问“这个自动修复为什么被认为已经恢复”，答案不是聊天摘要，而是一串可复查事件。

## 4. OpenClaw 课程 Cron 的实战映射

这个课程 cron 本身也可以看成一个小型发布系统：

~~~text
写 lesson
  -> 更新 README / TOOLS
  -> git diff --check
  -> commit / push
  -> Telegram 发布
  -> 记录 messageId / commitSha
~~~

如果按 Deployment Promotion 思路增强，它还应该补三类观测：

- GitHub remote head 是否包含本次 commit。
- Telegram message 是否发送成功，并记录 messageId。
- memory / TOOLS 是否写入了本次主题，避免下一轮重复。

也就是说，cron 的完成条件不应该是“我运行了 push”，而是：

~~~json
{
  "commitPushed": true,
  "remoteContainsCommit": true,
  "telegramMessageId": "12183",
  "lessonIndexed": true,
  "topicRemembered": true
}
~~~

这就是 proof-carrying execution 的落地：每个副作用动作都带证明。

## 5. 设计部署晋级 Gate 的几个细节

第一，观测窗口要和风险匹配。文档 typo 修复可以只检查 remote commit；支付、权限、外发消息相关修复至少要 canary + error budget + rollback plan。

第二，promote 必须绑定版本。不要只写“已部署”，要写 commit、artifact、environment、stage、policyVersion。

第三，rollback 不是失败羞耻，而是自动修复的正常分支。一个成熟 Agent 看到坏 probe 应该快速回滚、冻结 ticket、保留证据，而不是继续补丁叠补丁。

第四，hold 要有超时。否则“继续观察”会变成永远没人收口的半完成状态。

## 6. 小结

自动修复链路可以拆成三段：

- 修复前：Regression Pack 找到问题并派单。
- 修复中：Patch Intake + Mainline Sync 证明补丁可信。
- 修复后：Deployment Promotion + Observation 证明线上恢复。

今天的重点：**merge 只是代码仓库状态，promote 才是运行系统状态。成熟 Agent 不把“合了”当“好了”，它要用线上观测证明修复已经生效。**
