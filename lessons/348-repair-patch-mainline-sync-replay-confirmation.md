# 348. Agent 修复补丁主线同步与重放确认（Repair Patch Mainline Sync & Replay Confirmation）

上一课讲了 Patch Intake 和 Merge Arbitration：worker patch 回来以后，controller 要检查写入边界、验证命令和候选补丁冲突，不能只相信“已修复”这句话。

今天继续往后走：**补丁通过 intake 还不能马上进主线；它必须先和最新 main 同步，再用同步后的代码重跑 verification 和 selected replay，证明这个修复在真实主线基线上仍然成立。**

成熟链路应该长这样：

~~~text
Worker Patch
  -> Patch Intake Gate
  -> Merge Arbitration
  -> Mainline Sync
  -> Replay Confirmation
  -> PR / Merge / Retry / Escalate
~~~

核心原则：**修复不是在 worker 的旧世界里通过就算通过，而是在最新主线里通过才算通过。**

## 1. 为什么 intake 通过后还要同步主线

子 Agent 通常在隔离 worktree 或 fork workspace 里修问题。它开始工作时的基线可能已经落后：

~~~text
main:      A -- B -- C -- D
worker:   A -- B -- fix
~~~

如果 controller 直接把 fix 合进去，会遇到几类风险：

- C/D 已经改过同一段逻辑，fix 在旧基线有效，在新基线无效。
- C/D 已经修了同一个回归，fix 变成重复或反向修改。
- selected replay set 是按旧 diff 选的，新 main 引入了新的风险面。
- worker 跑过的测试来自旧代码，不能证明当前 main 安全。

所以 Mainline Sync 是自动修复链路的第二道验收：**把候选修复放到最新主线，再重新证明一次。**

## 2. learn-claude-code：最小 Mainline Sync Gate

教学版可以用三个输入建模：候选补丁、主线 revision、重放结果。重点不是实现 git，而是把“是否需要重验”变成显式决策。

~~~python
from dataclasses import dataclass
from enum import Enum

class SyncDecision(str, Enum):
    READY_FOR_PR = "ready_for_pr"
    NEEDS_REBASE = "needs_rebase"
    RETRY_WORKER = "retry_worker"
    ESCALATE = "escalate"

@dataclass
class PatchCandidate:
    ticket_id: str
    base_sha: str
    patch_sha: str
    touched_files: list[str]
    replay_cases: list[str]

@dataclass
class MainlineState:
    head_sha: str
    changed_since_base: list[str]

@dataclass
class ReplayResult:
    case_id: str
    status: str  # pass | fail | flaky
    fingerprint: str

@dataclass
class SyncGateResult:
    decision: SyncDecision
    reasons: list[str]

def overlaps(left: list[str], right: list[str]) -> bool:
    return bool(set(left) & set(right))

def confirm_on_mainline(
    patch: PatchCandidate,
    mainline: MainlineState,
    replay_results: list[ReplayResult],
) -> SyncGateResult:
    reasons: list[str] = []

    if patch.base_sha != mainline.head_sha:
        reasons.append(f"base is stale: {patch.base_sha} != {mainline.head_sha}")

    if overlaps(patch.touched_files, mainline.changed_since_base):
        reasons.append("mainline changed files touched by patch")

    failed = [r.case_id for r in replay_results if r.status == "fail"]
    flaky = [r.case_id for r in replay_results if r.status == "flaky"]

    if failed:
        return SyncGateResult(
            decision=SyncDecision.RETRY_WORKER,
            reasons=reasons + [f"replay failed: {', '.join(failed)}"],
        )

    if flaky:
        return SyncGateResult(
            decision=SyncDecision.ESCALATE,
            reasons=reasons + [f"flaky replay after sync: {', '.join(flaky)}"],
        )

    if reasons:
        return SyncGateResult(
            decision=SyncDecision.NEEDS_REBASE,
            reasons=reasons,
        )

    return SyncGateResult(decision=SyncDecision.READY_FOR_PR, reasons=[])
~~~

这里故意把 stale base 和 replay failure 分开：

- stale base 代表补丁需要移到最新主线。
- replay failure 代表移过去以后行为仍然不对。
- flaky 代表证据不稳定，不能自动合。

这比一句 “merge conflict” 清楚很多。

## 3. pi-mono：把 sync 变成 Promotion Gate

生产系统里，Mainline Sync 更像一个 Promotion Gate：候选 patch 从 worker lane 晋级到 mainline lane，必须带齐 base、diff、replay 和策略证据。

~~~ts
type PatchCandidate = {
  ticketId: string;
  baseSha: string;
  patchSha: string;
  touchedFiles: string[];
  replayCases: string[];
};

type MainlineSnapshot = {
  headSha: string;
  changedFilesSinceBase: string[];
  policyVersion: string;
};

type ReplayConfirmation = {
  caseId: string;
  status: "pass" | "fail" | "flaky";
  fingerprint: string;
  observedAt: string;
};

type PromotionDecision =
  | { action: "ready_for_pr"; evidence: string[] }
  | { action: "rebase_and_replay"; reason: string; evidence: string[] }
  | { action: "retry_worker"; reason: string; evidence: string[] }
  | { action: "manual_review"; reason: string; evidence: string[] };

function intersects(a: string[], b: string[]): boolean {
  const right = new Set(b);
  return a.some((item) => right.has(item));
}

function decidePromotion(
  patch: PatchCandidate,
  mainline: MainlineSnapshot,
  confirmations: ReplayConfirmation[],
): PromotionDecision {
  const evidence = [
    "patch:" + patch.patchSha,
    "base:" + patch.baseSha,
    "main:" + mainline.headSha,
    "policy:" + mainline.policyVersion,
  ];

  const failed = confirmations.filter((r) => r.status === "fail");
  const flaky = confirmations.filter((r) => r.status === "flaky");

  if (failed.length > 0) {
    return {
      action: "retry_worker",
      reason: "replay failed after mainline sync: " + failed.map((r) => r.caseId).join(", "),
      evidence,
    };
  }

  if (flaky.length > 0) {
    return {
      action: "manual_review",
      reason: "replay evidence is unstable: " + flaky.map((r) => r.caseId).join(", "),
      evidence,
    };
  }

  if (patch.baseSha !== mainline.headSha) {
    const changedSameFiles = intersects(
      patch.touchedFiles,
      mainline.changedFilesSinceBase,
    );

    return {
      action: "rebase_and_replay",
      reason: changedSameFiles
        ? "stale base and overlapping mainline changes"
        : "stale base; replay on latest main required",
      evidence,
    };
  }

  return { action: "ready_for_pr", evidence };
}
~~~

这个 Gate 最重要的设计是：**ready_for_pr 不是因为没有冲突，而是因为有证据证明新主线上的 replay 通过。**

如果和 pi-mono 的事件流结合，可以在 Agent.subscribe 外层记录：

~~~text
patch_intake_pass
  -> mainline_snapshot_collected
  -> patch_rebased
  -> replay_confirmation_pass
  -> promotion_decision_ready_for_pr
~~~

这样以后排查“为什么这个自动修复被合了”时，不需要翻聊天记录，直接看 promotion evidence。

## 4. replay confirmation 要重选用例

一个常见错误是：worker 开始时选了 replay cases，修完以后一直沿用同一批。

但 mainline sync 后，diff 变了：

~~~text
旧 diff: fix 修改 policy.ts
新 main: 同时改了 message-egress.ts 和 policy.ts
同步后 diff: policy.ts + message-egress.ts 的组合风险
~~~

这时 replay selector 应该重新运行一次，而不是只跑旧用例。

可以用这个规则：

~~~text
selectedReplay = union(
  casesSelectedBy(workerPatchDiff),
  casesSelectedBy(mainlineChangesSinceBase),
  casesSelectedBy(rebasedCombinedDiff),
  ticket.requiredReplayCases
)
~~~

在自动修复链路里，宁愿多跑几个相关 replay，也不要用旧选择集给新组合风险背书。

## 5. OpenClaw：主控 Agent 的主线确认流程

OpenClaw 里这个流程可以落成主控 Agent 的固定 checklist：

~~~text
1. 记录 worker patch 的 base sha 和 touched files
2. 拉取最新 main
3. 检查 main 自 base 以来改了哪些文件
4. 如果主线变化与 patch 有重叠，先 rebase 或重新应用 patch
5. 基于 rebased diff 重新选择 replay cases
6. 运行 ticket.verification + selected replay
7. 写入 PromotionDecision evidence
8. 只有 ready_for_pr 才允许 commit / PR / push
~~~

给 controller 的提示词可以这样写：

~~~text
接收 RepairTicket 的候选 patch 后，先做 Mainline Sync。

不要使用 worker 旧基线上的测试结果作为最终合并证据。请：
- 记录 patch base sha
- 拉取最新 main
- 判断 main 自 base 以来是否修改 patch touched files
- rebase 或重新应用 patch
- 基于同步后的 diff 重新选择 replay cases
- 重新运行 verification 和 selected replay
- 输出 PromotionDecision：ready_for_pr / rebase_and_replay / retry_worker / manual_review

只有 PromotionDecision=ready_for_pr 时，才允许进入 PR 或 merge 步骤。
~~~

这一步看起来像“多跑一次测试”，本质上是把自动修复从“局部正确”推进到“主线正确”。

## 6. 实战建议

落地时可以先做三个轻量版本：

- 每个 worker patch 必须记录 baseSha。
- controller 在 push 前必须 pull/rebase 最新 main。
- rebase 后必须重新运行 ticket.verification，且 selected replay 至少覆盖 touchedFiles 对应风险面。

以后再逐步加：

- mainline changed files overlap 检测。
- replay selector 重新计算。
- PromotionDecision JSON 归档。
- stale base 超过 N commits 自动退回 worker 重修。

一句话总结：**Patch Intake 证明 worker 没乱改；Mainline Sync + Replay Confirmation 证明这个修复在最新主线里真的能用。**
