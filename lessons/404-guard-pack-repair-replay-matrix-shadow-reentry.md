# 404. Agent 回归护栏修复候选的回放矩阵与影子重入闸门（Guard Pack Repair Replay Matrix & Shadow Re-entry Gate）

上一课讲了 **Guard Pack Incident Forensics & Root-Cause Repair**：Guard Pack 出事故后，回滚只是止血；真正的修复要把错误 decision、guardTrace、证据包、runtime snapshot 和 root cause 绑定成 Forensics Case。

今天继续讲修复后的关键一步：**修复候选不能因为修了根因就直接回 canary。**

一句话：**Guard Pack repair candidate 必须先通过 Replay Matrix 覆盖原事故、相邻风险面、历史回归包和反向安全样本，再进入只读 Shadow Re-entry；只有影子窗口证明 false positive、false negative、unknown 和 latency 都在预算内，才能重新进入 canary。**

---

## 1. 为什么修复候选不能直接上线

Guard Pack 的修复很容易出现二次事故：

~~~text
事故：v42 把安全的 git push 错误拦成 manual_review
修复：放宽 guard 条件
新事故：某些危险 push 也被放过去了
~~~

这是典型的“修了误伤，制造漏挡”。

护栏修复和普通 bugfix 不一样。普通业务 bug 可以靠单测覆盖主路径；Guard Pack 是安全契约，修复必须证明四件事：

- 原事故不会复发；
- 相邻风险面没有被放宽；
- 历史事故回归包仍然通过；
- 线上真实流量里，候选包只读跟跑时没有异常差异。

这就是 **Replay Matrix + Shadow Re-entry Gate**。

---

## 2. 最小模型：Replay Matrix

Replay Matrix 不是“跑全部测试”。它是按风险面组织的验证矩阵。

~~~ts
type ReplayMatrix = {
  matrixId: string;
  forensicsCaseId: string;
  candidatePackVersion: number;
  basePackVersion: number;
  rows: ReplayRow[];
  requiredOutcome: {
    maxFalsePositiveCount: number;
    maxFalseNegativeCount: number;
    maxUnknownCount: number;
    maxLatencyP95Ms: number;
  };
};

type ReplayRow = {
  rowId: string;
  source:
    | "incident_fixture"
    | "neighbor_surface"
    | "historical_regression"
    | "negative_control"
    | "latency_probe";
  riskSurface:
    | "external_side_effect"
    | "credential_access"
    | "memory_write"
    | "message_egress"
    | "git_push"
    | "deployment"
    | "low_risk_readonly";
  fixtureRefs: string[];
  expectedDecision: "allow" | "warn" | "dry_run" | "block" | "manual_review";
  strictness: "must_match" | "may_be_stricter" | "budgeted_diff";
};
~~~

几个设计点：

1. **incident_fixture**：原事故样本，必须修好；
2. **neighbor_surface**：相邻风险面，防止修复条件太宽；
3. **historical_regression**：旧事故，防止旧坑回来；
4. **negative_control**：明确低风险样本，防止 manual_review 风暴；
5. **latency_probe**：护栏不能为了安全把决策路径拖死。

Replay Matrix 的价值是让“这次修复到底证明了什么”可以被审计，而不是只看 CI 绿了。

---

## 3. learn-claude-code：纯函数回放矩阵闸门

教学版可以把每条 replay 结果压成结构化 diff，再统一判定是否允许进入 shadow。

~~~py
from dataclasses import dataclass
from typing import Literal


Decision = Literal["allow", "warn", "dry_run", "block", "manual_review"]
Strictness = Literal["must_match", "may_be_stricter", "budgeted_diff"]
GateDecision = Literal["enter_shadow", "block_repair", "manual_review"]

RANK = {
    "allow": 0,
    "warn": 1,
    "dry_run": 2,
    "manual_review": 3,
    "block": 4,
}


@dataclass(frozen=True)
class ReplayResult:
    row_id: str
    source: str
    expected: Decision
    actual: Decision
    strictness: Strictness
    latency_ms: int
    evidence_complete: bool


@dataclass(frozen=True)
class ReplayBudget:
    max_false_positive_count: int
    max_false_negative_count: int
    max_unknown_count: int
    max_latency_p95_ms: int


def percentile_95(values: list[int]) -> int:
    if not values:
        return 0
    ordered = sorted(values)
    index = int((len(ordered) - 1) * 0.95)
    return ordered[index]


def classify_diff(result: ReplayResult) -> str:
    if not result.evidence_complete:
        return "unknown"

    if result.strictness == "must_match":
        if result.actual == result.expected:
            return "match"
        if RANK[result.actual] > RANK[result.expected]:
            return "false_positive"
        return "false_negative"

    if result.strictness == "may_be_stricter":
        if RANK[result.actual] >= RANK[result.expected]:
            return "match"
        return "false_negative"

    if result.actual == result.expected:
        return "match"
    if RANK[result.actual] > RANK[result.expected]:
        return "budgeted_false_positive"
    return "budgeted_false_negative"


def decide_replay_matrix(results: list[ReplayResult], budget: ReplayBudget) -> dict:
    counts = {
        "false_positive": 0,
        "false_negative": 0,
        "unknown": 0,
        "budgeted_false_positive": 0,
        "budgeted_false_negative": 0,
    }

    failures = []
    for result in results:
        diff = classify_diff(result)
        if diff in counts:
            counts[diff] += 1
        if diff in ("false_negative", "unknown"):
            failures.append({"rowId": result.row_id, "diff": diff})

    latency_p95 = percentile_95([result.latency_ms for result in results])

    false_positive_total = counts["false_positive"] + counts["budgeted_false_positive"]
    false_negative_total = counts["false_negative"] + counts["budgeted_false_negative"]

    if false_negative_total > budget.max_false_negative_count:
        return {
            "decision": "block_repair",
            "reason": "candidate would loosen safety decisions",
            "counts": counts,
            "failures": failures,
        }

    if counts["unknown"] > budget.max_unknown_count:
        return {
            "decision": "manual_review",
            "reason": "candidate lacks complete evidence for replay rows",
            "counts": counts,
            "failures": failures,
        }

    if false_positive_total > budget.max_false_positive_count:
        return {
            "decision": "manual_review",
            "reason": "candidate may create over-blocking or review storm",
            "counts": counts,
        }

    if latency_p95 > budget.max_latency_p95_ms:
        return {
            "decision": "block_repair",
            "reason": "candidate guard path is too slow",
            "latencyP95Ms": latency_p95,
        }

    return {
        "decision": "enter_shadow",
        "reason": "replay matrix passed within budget",
        "counts": counts,
        "latencyP95Ms": latency_p95,
    }
~~~

注意这里的策略：**false negative 默认比 false positive 更危险**。护栏修复宁可先 manual_review，也不能把本该 block 的高风险动作放过去。

---

## 4. pi-mono：把修复候选接到事件流

生产实现里，Replay Matrix 不应该是一个临时脚本，而应该是 Guard Pack 发布流水线的一环。

~~~ts
type GuardPackRepairCandidate = {
  candidateId: string;
  forensicsCaseId: string;
  candidatePackVersion: number;
  basePackVersion: number;
  changedGuardIds: string[];
  rootCauseKind: string;
};

type ReplayMatrixReceipt = {
  candidateId: string;
  matrixId: string;
  decision: "enter_shadow" | "block_repair" | "manual_review";
  counts: Record<string, number>;
  latencyP95Ms: number;
  evidenceRefs: string[];
  createdAt: string;
};

class GuardPackRepairGate {
  constructor(
    private readonly replayRunner: ReplayRunner,
    private readonly eventBus: EventBus,
    private readonly receiptStore: ReceiptStore,
  ) {}

  async evaluate(candidate: GuardPackRepairCandidate): Promise<ReplayMatrixReceipt> {
    const matrix = await this.replayRunner.buildMatrix({
      forensicsCaseId: candidate.forensicsCaseId,
      changedGuardIds: candidate.changedGuardIds,
      rootCauseKind: candidate.rootCauseKind,
    });

    const results = await this.replayRunner.run({
      matrix,
      packVersion: candidate.candidatePackVersion,
      compareWith: candidate.basePackVersion,
    });

    const receipt = decideReplayMatrix(matrix, results);
    await this.receiptStore.append(receipt);

    await this.eventBus.publish({
      type: "guard_pack.repair_replay_matrix_evaluated",
      candidateId: candidate.candidateId,
      matrixId: matrix.matrixId,
      decision: receipt.decision,
      evidenceRefs: receipt.evidenceRefs,
    });

    return receipt;
  }
}
~~~

这层要坚持两个原则：

- **receipt append-only**：每次 replay 都留下收据，后面 shadow/canary 才知道自己基于哪次验证；
- **matrix build 可解释**：必须能说清楚为什么这些 fixtures 被选中，尤其是 neighbor_surface。

---

## 5. Shadow Re-entry：只读重入，不产生副作用

Replay Matrix 通过，只代表离线样本通过。下一步是 Shadow Re-entry：候选 Guard Pack 跟跑真实流量，但不控制真实动作。

~~~ts
type ShadowReentryPolicy = {
  candidateId: string;
  candidatePackVersion: number;
  primaryPackVersion: number;
  window: {
    minDecisions: number;
    maxDurationMinutes: number;
  };
  budgets: {
    maxDangerousLoosening: number;
    maxOverBlockingRate: number;
    maxUnknownRate: number;
    maxLatencyP95Ms: number;
  };
};

type ShadowDiff =
  | "same_decision"
  | "candidate_stricter"
  | "candidate_looser_safe"
  | "candidate_looser_dangerous"
  | "candidate_unknown"
  | "candidate_error";

function classifyShadowDiff(primary: Decision, shadow: Decision, risk: "low" | "medium" | "high"): ShadowDiff {
  if (primary === shadow) return "same_decision";
  if (RANK[shadow] > RANK[primary]) return "candidate_stricter";
  if (RANK[shadow] < RANK[primary] && risk === "high") return "candidate_looser_dangerous";
  if (RANK[shadow] < RANK[primary]) return "candidate_looser_safe";
  return "candidate_unknown";
}
~~~

Shadow Re-entry 的输出不是“上线/不上线”这么粗，而是：

- promote_to_canary：样本足够，差异在预算内；
- extend_shadow：样本不足或差异集中在低风险面；
- block_reentry：出现高风险放宽、unknown 爆炸或 latency 退化；
- manual_review：差异解释不清，但还没到自动阻断。

---

## 6. OpenClaw 实战：课程 Cron 也能套这个思路

以这个课程 Cron 自己为例：

1. **Replay Matrix**：发课前检查 lesson 文件、README、TOOLS、Telegram 文案、git 状态；
2. **incident_fixture**：上一课已讲主题不能重复；
3. **neighbor_surface**：不能误改 Rust 课程仓库或其他项目；
4. **negative_control**：README 旧目录不能被重排；
5. **Shadow Re-entry**：真实发送前先 dry-run 文案长度、敏感内容和目标群组；
6. **Receipt**：发送后的 messageId、commit sha、远端 main 验证写入 memory。

这不是把简单事情复杂化，而是让外部副作用有证据链。对 Agent 来说，发群消息、push 代码、部署服务，本质都是现实世界副作用；越自动化，越需要 replay 和 shadow 这种低成本刹车。

---

## 7. 实战清单

做 Guard Pack 修复时，至少加这几个检查：

1. **原事故样本必须通过**：否则不是修复；
2. **相邻风险面必须覆盖**：否则容易修出漏挡；
3. **历史回归包必须重放**：否则旧事故会复活；
4. **低风险 negative control 必须存在**：否则会制造 manual_review 风暴；
5. **latency 单独预算**：安全规则慢到影响主路径，也是一种事故；
6. **shadow 只读跟跑**：候选包先观察真实流量，不直接控制副作用；
7. **每个阶段写 receipt**：没有收据，就没有可审计的晋级理由。

---

## 8. 小结

第 403 课解决的是：**坏 Guard Pack 为什么坏，应该怎么修。**

第 404 课解决的是：**修复候选凭什么重新进入生产路径。**

成熟 Agent 的 Guard Pack 修复流程应该是：

~~~text
forensics case
  -> root cause
  -> repair candidate
  -> replay matrix
  -> shadow re-entry
  -> canary
  -> active
~~~

真正可靠的护栏系统，不靠“这次我觉得修好了”，而靠离线回放、线上影子、预算闸门和可审计收据，一步一步把修复候选重新带回生产。
