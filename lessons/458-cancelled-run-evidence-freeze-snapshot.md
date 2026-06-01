# 458. Agent 取消后的证据冻结快照（Cancelled Run Evidence Freeze Snapshot）

上一课讲了取消后的外部副作用对账：run 被 cancelled 以后，已经发出去的消息、git push、支付、部署不能当作没发生。

今天补一个更靠前、也更容易被忽略的环节：**取消发生的第一件事，不是清理现场，而是冻结证据。**

因为很多 Agent 的取消流程会立刻做这些事：

~~~text
abort stream
kill child process
delete temp files
clear tool buffer
mark run cancelled
return control to user
~~~

这些动作看起来很干净，但生产事故里最怕的就是“干净到没有证据”。如果工具执行到一半、外部请求刚发出、sub-agent 刚提交 patch、日志 buffer 还没 flush，取消后的 cleanup 可能把判断真相所需的上下文一起删掉。

一句话：**取消不是先扫地；取消要先拍照、封存、再清理。**

---

## 1. 为什么要先冻结证据

取消现场最容易丢的证据包括：

- 当前 run 的状态：active tool、active step、last assistant delta；
- 工具阶段：prepared、dispatched、acknowledged、cleanup_started；
- 外部副作用尝试：messageId、requestId、idempotencyKey、payloadHash；
- 子任务信息：pid、sub-agent id、worktree path、job id；
- 临时文件：patch、stdout/stderr tail、生成物 manifest；
- 用户可见输出：已经 stream 出去的 token、已经发出的通知；
- 时间线：cancel requested、abort delivered、cleanup started、cleanup finished。

如果没有这些证据，后续对账会退化成猜：

~~~text
Telegram 消息到底发没发？
git push 是在取消前还是取消后完成？
工具超时是 provider 没收到，还是收到了但本地断线？
sub-agent 的 patch 是有效产物，还是应该丢弃？
cleanup 删除的临时文件里有没有唯一的 remoteRef？
~~~

所以取消协议里应该多一个很小但很硬的步骤：

~~~text
CancelRequest
  -> EvidenceFreezeSnapshot
  -> cleanup / orphan reaper / side-effect reconciliation
  -> terminal cancellation receipt
~~~

冻结快照不是为了永久保存所有原始数据，而是为了给后续判断留下最小可验证事实。

---

## 2. 最小数据模型

建议把取消快照做成不可变对象：

- **snapshotId**：一次冻结的唯一 ID；
- **runId**：被取消的 run；
- **cancelRequestId**：对应哪次取消请求；
- **capturedAt**：冻结时间；
- **phaseVector**：每个工具、子任务、副作用的当前阶段；
- **artifactRefs**：stdout tail、patch、temp manifest、transcript slice 的引用；
- **effectRefs**：EffectLedger 中 cancel window 附近的 effectId；
- **redactionProfile**：保存前做过哪些脱敏；
- **hash**：快照内容哈希，防止后续被无声改写；
- **retentionUntil**：保留到什么时候，避免无限堆积。

注意两个原则：

1. 快照要在 cleanup 之前写入。
2. 快照只保存最小必要证据，敏感原文能 hash 就不要 raw。

---

## 3. learn-claude-code：教学版冻结判定器

教学版可以先写一个纯函数：根据当前 run state，决定哪些证据必须冻结、哪些可以摘要、哪些应该脱敏。

~~~python
# learn_claude_code/runtime/cancel_evidence_freeze.py
from dataclasses import dataclass
from typing import Literal

Phase = Literal["prepared", "running", "dispatched", "acknowledged", "cleanup_started"]
Sensitivity = Literal["public", "internal", "secret"]
CaptureMode = Literal["raw", "tail", "hash_only", "omit"]


@dataclass(frozen=True)
class EvidenceItem:
    key: str
    phase: Phase
    sensitivity: Sensitivity
    bytes_estimate: int
    required_for_reconcile: bool


@dataclass(frozen=True)
class CaptureDecision:
    key: str
    mode: CaptureMode
    reason: str


def decide_cancel_freeze(items: list[EvidenceItem]) -> list[CaptureDecision]:
    decisions: list[CaptureDecision] = []

    for item in items:
        if item.sensitivity == "secret":
            decisions.append(CaptureDecision(
                item.key,
                "hash_only",
                "secret_material_must_not_be_frozen_raw",
            ))
            continue

        if item.required_for_reconcile:
            mode: CaptureMode = "raw" if item.bytes_estimate <= 8192 else "tail"
            decisions.append(CaptureDecision(
                item.key,
                mode,
                "needed_by_orphan_reaper_or_side_effect_reconciliation",
            ))
            continue

        if item.phase in ("dispatched", "acknowledged"):
            decisions.append(CaptureDecision(
                item.key,
                "tail",
                "late_phase_evidence_may_contain_remote_reference",
            ))
            continue

        decisions.append(CaptureDecision(
            item.key,
            "omit",
            "not_needed_for_cancel_closeout",
        ))

    return decisions
~~~

这个函数故意很朴素，但它把关键策略写清楚了：

- secret 不进 raw 快照；
- 对账必需证据优先保留；
- 大对象只保留 tail 或 manifest；
- dispatched / acknowledged 阶段更可能包含 remoteRef；
- 其余东西不要无限保存。

---

## 4. pi-mono：生产版 EvidenceFreezeService

生产实现里，EvidenceFreezeService 应该挂在 run lifecycle 的 cancel path 上，而且要早于 cleanup worker。

~~~ts
type ToolPhase = "prepared" | "running" | "dispatched" | "acknowledged" | "cleanup_started";

type EvidenceFreezeSnapshot = {
  snapshotId: string;
  runId: string;
  cancelRequestId: string;
  capturedAt: number;
  phaseVector: Array<{
    subjectId: string;
    subjectType: "tool" | "subagent" | "effect" | "artifact";
    phase: ToolPhase;
    ref?: string;
    payloadHash?: string;
  }>;
  artifactRefs: string[];
  effectIds: string[];
  redactionProfile: "hash_secrets_tail_large_outputs";
  snapshotHash: string;
  retentionUntil: number;
};

class CancelEvidenceFreezeService {
  constructor(
    private readonly runs: RunStore,
    private readonly tools: ToolExecutionStore,
    private readonly effects: EffectLedger,
    private readonly artifacts: ArtifactStore,
    private readonly snapshots: EvidenceSnapshotStore,
    private readonly hasher: SnapshotHasher,
  ) {}

  async freezeBeforeCleanup(runId: string, cancelRequestId: string): Promise<EvidenceFreezeSnapshot> {
    const existing = await this.snapshots.findByCancelRequest(cancelRequestId);
    if (existing) return existing;

    const run = await this.runs.get(runId);
    const toolExecutions = await this.tools.listActiveOrRecent(runId);
    const effectAttempts = await this.effects.listCancelWindow(runId);
    const artifactRefs = await this.artifacts.captureMinimalRefs({
      runId,
      includeStdoutTail: true,
      includePatchManifest: true,
      redactSecrets: true,
    });

    const draft = {
      snapshotId: crypto.randomUUID(),
      runId,
      cancelRequestId,
      capturedAt: Date.now(),
      phaseVector: [
        ...toolExecutions.map((tool) => ({
          subjectId: tool.executionId,
          subjectType: "tool" as const,
          phase: tool.phase,
          ref: tool.providerRequestId,
          payloadHash: tool.inputHash,
        })),
        ...effectAttempts.map((effect) => ({
          subjectId: effect.effectId,
          subjectType: "effect" as const,
          phase: effect.phase,
          ref: effect.remoteRef,
          payloadHash: effect.payloadHash,
        })),
      ],
      artifactRefs,
      effectIds: effectAttempts.map((effect) => effect.effectId),
      redactionProfile: "hash_secrets_tail_large_outputs" as const,
      retentionUntil: run.startedAt + 7 * 24 * 60 * 60 * 1000,
    };

    const snapshotHash = this.hasher.hash(draft);

    return this.snapshots.writeOnce({
      ...draft,
      snapshotHash,
    });
  }
}
~~~

几个实现细节很重要：

- **writeOnce**：同一个 cancelRequestId 重试时返回同一份快照；
- **captureMinimalRefs**：保存引用和 tail，不要把所有大文件塞进 DB；
- **redactSecrets**：冻结证据不是泄密豁免；
- **phaseVector**：后续 reaper / reconciler 不必重新猜取消时每个对象在哪个阶段；
- **snapshotHash**：后续 closeout 可以证明快照没有被 cleanup 或人工修补改写。

---

## 5. OpenClaw：课程 Cron 的真实用法

拿这个课程 cron 举例，一次发布至少会触碰：

~~~text
lesson file
README directory
TOOLS.md taught list
Telegram group message
git commit / push
memory daily log
~~~

如果 run 在中间取消，最坏的情况不是“没发成功”，而是“发到一半但证据丢了”：

- Telegram 已经发出，但 messageId 没记住；
- lesson 文件写了，但 README 没加目录；
- README 改了，但 TOOLS.md 没加已讲内容；
- commit 成功但 push 失败；
- push 成功但本地 final 没来得及汇报。

这个 cron 的取消冻结快照可以很简单：

~~~text
snapshot:
  latestLessonFile: lessons/458-cancelled-run-evidence-freeze-snapshot.md
  readmeEntryPresent: true/false
  toolsTaughtEntryPresent: true/false
  telegramMessageId: optional
  gitHead: local sha
  gitRemoteHead: origin/main sha
  dirtyFiles:
    - README.md
    - lessons/458-...
    - ../TOOLS.md
~~~

下一次 cron 启动时，先看这个 snapshot：

- 如果 Telegram 已发但 git 没推，补 commit/push，不要重发；
- 如果 git 已推但 TOOLS 没更新，补 TOOLS 并追加修复 commit；
- 如果 lesson/README/TOOLS 不一致，先 reconcile 第 458 课，不要直接选第 459 课；
- 如果 snapshotHash 不匹配，进入人工复核，避免基于被污染证据继续自动化。

这就是 Always-on Agent 的实用边界：可以自动恢复，但恢复必须建立在冻结证据上。

---

## 6. 和前几课的关系

取消治理链路现在可以串起来：

~~~text
455 Cancellable Tool Convergence
  当前工具要收敛到明确终态

456 Cancelled Run Orphan Reaper
  取消后还在跑的后代要被找到并收口

457 Cancelled Run Side Effect Reconciliation
  已经出门的外部副作用要和现实对账

458 Cancelled Run Evidence Freeze Snapshot
  在 cleanup 前冻结最小证据，支撑后续所有判断
~~~

注意顺序上，458 的动作应该发生得更早：

~~~text
cancel requested
  -> freeze evidence snapshot
  -> deliver abort signal
  -> cleanup local resources
  -> reap descendants
  -> reconcile side effects
  -> write terminal cancellation receipt
~~~

否则后面的 reaper 和 reconciler 可能拿不到足够证据。

---

## 7. 实战检查清单

给 Agent 加取消能力时，直接问这 8 个问题：

- cleanup 前是否会写 EvidenceFreezeSnapshot？
- 快照是否 writeOnce，重复取消不会覆盖第一现场？
- 是否记录 tool phase、effect phase、sub-agent id 和 artifact refs？
- 是否保存 idempotencyKey / remoteRef / payloadHash？
- stdout、patch、临时文件是否至少有 tail 或 manifest？
- secret 是否只 hash 或脱敏保存？
- snapshot 是否有 hash 和 retentionUntil？
- closeout 是否会校验 snapshotHash，而不是相信可变日志？

如果答案是否定的，取消后的“对账”很可能只是补救时的猜测。

---

## 8. 关键 takeaway

取消是一个破坏性很强的控制动作。它会中断工具、停止流、清理资源、终止后代任务。越是这种动作，越要先留下最小证据快照。

成熟 Agent 的取消流程不是：

~~~text
cancel -> cleanup -> hope
~~~

而是：

~~~text
cancel -> freeze evidence -> cleanup -> reconcile -> receipt
~~~

先冻结证据，后清理现场。这样 cancelled 才能从一个 UI 状态，变成一个可审计、可恢复、可证明的工程终态。
