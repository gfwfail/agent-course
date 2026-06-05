# Agent 基线漂移修复后的候选再晋级闸门

> Baseline Remediation Candidate Re-promotion Gate

第 492 课讲了 Normal Ops Baseline Drift Breach：新常态基线激活后，如果 Drift Sentinel 发现错误率、延迟、成本或工具失败率漂移，要先生成 Breach Report，再决定 rollback_baseline、reopen_observation、open_regression_case、extend_sentinel 或 ignore_noise。

今天补上下一步：漂移处理以后，不能永远停在旧基线，也不能修完配置就直接把新基线重新设为 active。要把修复后的 baseline 当成一个新的 candidate，带着原 breach 证据、修复 diff、回放结果和影子观察窗口重新晋级。

所以第 493 课是 **Baseline Remediation Candidate Re-promotion Gate**：把 baseline 漂移修复从“改配置”升级成“候选版本再发布”。成熟 Agent 的运维基线不是一份 YAML，而是一条有证据、有回滚点、有观察窗口的发布链路。

## 核心模型

把修复后的再晋级拆成 5 个对象：

~~~text
BaselineDriftCloseoutReceipt
  -> BaselineRemediationPatch
  -> RemediatedBaselineCandidate
  -> CandidateShadowValidationReport
  -> BaselineRepromotionReceipt
~~~

分工：

- BaselineDriftCloseoutReceipt：上一课关闭漂移 breach 时写下的动作和原因；
- BaselineRemediationPatch：修复 diff，说明改了哪些阈值、timeout、并发、采样率或告警策略；
- RemediatedBaselineCandidate：不可变候选版本，绑定 parentBaseline、breachId、patchHash 和 rollbackBaseline；
- CandidateShadowValidationReport：候选基线在 shadow 模式下的真实表现，对比 active baseline；
- BaselineRepromotionReceipt：最终晋级或拒绝收据，记录 promote_candidate、extend_shadow、tighten_patch、reject_candidate 或 manual_review。

一句话：**漂移修复不能直接改 active，要先把修复变成 candidate，在 shadow 里证明它比旧基线好，而且没有把风险藏起来。**

## learn-claude-code：再晋级判定纯函数

教学版保持纯函数，输入修复证据和 shadow 报告，输出下一步动作。

~~~python
# learn_claude_code/baseline_repromotion_gate.py
from dataclasses import dataclass
from typing import Literal

RepromotionDecision = Literal[
    "promote_candidate",
    "extend_shadow",
    "tighten_patch",
    "reject_candidate",
    "manual_review",
]


@dataclass
class BaselineDriftCloseoutReceipt:
    breach_id: str
    baseline_version: str
    action: str  # rollback_baseline | reopen_observation | open_regression_case
    closed: bool


@dataclass
class BaselineRemediationPatch:
    breach_id: str
    candidate_version: str
    parent_baseline_version: str
    patch_hash: str
    rollback_baseline_version: str
    changed_controls: list[str]
    risk_tier: str  # low | medium | high


@dataclass
class CandidateShadowValidationReport:
    breach_id: str
    candidate_version: str
    shadow_windows: int
    min_shadow_windows: int
    error_rate_delta: float
    p95_delta_ms: int
    cost_delta: float
    tool_failure_delta: float
    hidden_signal_count: int
    reproduced_breach_fixed: bool
    regression_replay_passed: bool


def decide_baseline_repromotion(
    closeout: BaselineDriftCloseoutReceipt,
    patch: BaselineRemediationPatch,
    report: CandidateShadowValidationReport,
) -> RepromotionDecision:
    same_breach = (
        closeout.breach_id == patch.breach_id == report.breach_id
        and patch.candidate_version == report.candidate_version
        and closeout.baseline_version == patch.parent_baseline_version
    )
    if not same_breach or not closeout.closed:
        return "manual_review"

    if report.shadow_windows < report.min_shadow_windows:
        return "extend_shadow"

    if not report.reproduced_breach_fixed:
        return "reject_candidate"

    if not report.regression_replay_passed:
        return "tighten_patch"

    worse_runtime = (
        report.error_rate_delta > 0.002
        or report.p95_delta_ms > 250
        or report.cost_delta > 0.10
        or report.tool_failure_delta > 0.005
    )

    if report.hidden_signal_count > 0:
        return "manual_review"

    if worse_runtime:
        return "tighten_patch"

    if patch.risk_tier == "high" and report.shadow_windows < report.min_shadow_windows * 2:
        return "extend_shadow"

    return "promote_candidate"
~~~

这里的关键不是阈值本身，而是 3 个硬门槛：

- `reproduced_breach_fixed`：原来触发 breach 的问题必须能被候选修掉；
- `regression_replay_passed`：历史正常样本不能被候选搞坏；
- `hidden_signal_count`：不能靠少采样、少告警、少记录来制造“看起来稳定”。

## pi-mono：BaselineRepromotionWorker

生产版 worker 应该把候选晋级做成事务：同一事务里写 receipt、切 pointer、注册 rollback point、安排 post-promotion sentinel。

~~~ts
// packages/agent-runtime/src/ops/BaselineRepromotionWorker.ts
export type RepromotionDecision =
  | "promote_candidate"
  | "extend_shadow"
  | "tighten_patch"
  | "reject_candidate"
  | "manual_review";

export interface BaselineDriftCloseoutReceipt {
  breachId: string;
  baselineVersion: string;
  action: "rollback_baseline" | "reopen_observation" | "open_regression_case";
  closed: boolean;
}

export interface BaselineRemediationPatch {
  breachId: string;
  candidateVersion: string;
  parentBaselineVersion: string;
  patchHash: string;
  rollbackBaselineVersion: string;
  changedControls: string[];
  riskTier: "low" | "medium" | "high";
}

export interface CandidateShadowValidationReport {
  breachId: string;
  candidateVersion: string;
  shadowWindows: number;
  minShadowWindows: number;
  errorRateDelta: number;
  p95DeltaMs: number;
  costDelta: number;
  toolFailureDelta: number;
  hiddenSignalCount: number;
  reproducedBreachFixed: boolean;
  regressionReplayPassed: boolean;
}

export class BaselineRepromotionWorker {
  constructor(private readonly store: BaselineOpsStore) {}

  decide(
    closeout: BaselineDriftCloseoutReceipt,
    patch: BaselineRemediationPatch,
    report: CandidateShadowValidationReport,
  ): RepromotionDecision {
    const sameBreach =
      closeout.breachId === patch.breachId &&
      patch.breachId === report.breachId &&
      patch.candidateVersion === report.candidateVersion &&
      closeout.baselineVersion === patch.parentBaselineVersion;

    if (!sameBreach || !closeout.closed) return "manual_review";
    if (report.shadowWindows < report.minShadowWindows) return "extend_shadow";
    if (!report.reproducedBreachFixed) return "reject_candidate";
    if (!report.regressionReplayPassed) return "tighten_patch";
    if (report.hiddenSignalCount > 0) return "manual_review";

    const worseRuntime =
      report.errorRateDelta > 0.002 ||
      report.p95DeltaMs > 250 ||
      report.costDelta > 0.10 ||
      report.toolFailureDelta > 0.005;

    if (worseRuntime) return "tighten_patch";

    if (
      patch.riskTier === "high" &&
      report.shadowWindows < report.minShadowWindows * 2
    ) {
      return "extend_shadow";
    }

    return "promote_candidate";
  }

  async handle(report: CandidateShadowValidationReport) {
    return this.store.transaction(async (tx) => {
      const closeout = await tx.getDriftCloseout(report.breachId);
      const patch = await tx.getRemediationPatch(report.candidateVersion);
      const decision = this.decide(closeout, patch, report);

      const receipt = await tx.writeRepromotionReceipt({
        breachId: report.breachId,
        candidateVersion: report.candidateVersion,
        decision,
        report,
        decidedAt: Date.now(),
      });

      if (decision === "promote_candidate") {
        await tx.registerRollbackPoint({
          fromVersion: patch.candidateVersion,
          toVersion: patch.rollbackBaselineVersion,
          reason: "baseline_repromotion",
        });

        await tx.activateBaselinePointer({
          version: patch.candidateVersion,
          expectedParent: patch.parentBaselineVersion,
          receiptId: receipt.id,
        });

        await tx.createPostPromotionSentinel({
          baselineVersion: patch.candidateVersion,
          sourceBreachId: patch.breachId,
          minWindows: patch.riskTier === "high" ? 12 : 4,
        });
      }

      if (decision === "tighten_patch") {
        await tx.openPatchRevisionTask({
          breachId: report.breachId,
          candidateVersion: report.candidateVersion,
          changedControls: patch.changedControls,
        });
      }

      return receipt;
    });
  }
}
~~~

注意 `activateBaselinePointer` 里要带 `expectedParent`。这相当于 CAS，防止另一个 worker 已经切了 active 指针，而当前 worker 还拿旧上下文覆盖生产基线。

## OpenClaw：课程 Cron 里的实战类比

这套课程 Cron 其实也有“基线”：

- README.md 目录必须包含最新课程；
- TOOLS.md 的已讲内容必须同步；
- Telegram 群组必须发出同一课；
- Git commit/push 必须成功；
- 课程编号必须单调递增，不能重复。

如果某次自动任务发现 README 更新了但 TOOLS 没更新，这就是 baseline drift。修复后不要只补一行 TOOLS 就算结束，还应该做一次 candidate shadow validation：

~~~text
checklist:
  - lesson_file_exists: lessons/493-*.md
  - readme_has_entry: true
  - tools_has_topic: true
  - telegram_sent: true
  - git_pushed: true
  - no_duplicate_topic: true
decision:
  promote_candidate: all true
  tighten_patch: missing readme/tools/git evidence
  manual_review: telegram sent but git failed
~~~

这就是 OpenClaw 自动化里的“运维基线再晋级”：不是凭感觉说完成，而是每个外部副作用都回到证据清单里对账。

## 落地建议

1. Baseline 修复永远生成新版本，不要原地改 active。
2. Shadow 报告必须同时比较质量、延迟、成本、工具失败和信号完整性。
3. 原 breach 必须可复现、可证明已修复，否则不能晋级。
4. 晋级时写 rollback point，并立刻挂 post-promotion sentinel。
5. 高风险 baseline 至少需要双倍 shadow window，宁可慢一点，也不要把坏基线再次推回生产。

## 小结

第 492 课解决的是“漂移报警后怎么选动作”。第 493 课解决的是“动作之后怎么把修复后的新基线重新安全上线”。

成熟 Agent 的常态运维不是一条直线，而是一个闭环：发现漂移 -> 最小处置 -> 修复候选 -> shadow 验证 -> 重新晋级 -> 晋级后哨兵。只有这样，基线才不是写在配置里的愿望，而是持续被生产证据证明的运行契约。
