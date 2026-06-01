# 461. Agent 取消重试前的现实漂移探针（Pre-Replay Reality Drift Probe）

上一课讲了 **Retry Replay Guard**：取消后的 retry 不能直接 rerun，要把步骤拆成 rerun、reuse_receipt、manual_review 或 block。

但这里还有一个更细的坑：**上一轮 closeout 的结论，只代表 closeout 当时的现实，不代表 retry 此刻的现实。**

例如：

- closeout 时 git remote head 是 A，retry 时已经有人 push 到 B；
- closeout 时 Telegram 消息存在，retry 时消息被管理员删除了；
- closeout 时部署 job 是 cancelled，retry 前平台自动恢复成 running；
- closeout 时 sub-agent patch 没合并，retry 前用户手动改了同一个文件；
- closeout 时 payment intent 是 requires_capture，retry 前 webhook 把它推进到 succeeded。

所以 Retry Replay Guard 前面应该再放一个 **Pre-Replay Reality Drift Probe**：

~~~text
RetryRequest
  -> PreviousRunCloseoutReceipt
  -> RealityProbePlan
  -> DriftReport
  -> ReplayPlan | ReplayBlockedReceipt
~~~

一句话：**retry 不是只看旧收据，而是先重新探测外部现实，确认旧收据仍然有效。**

---

## 1. 为什么 closeout receipt 会过期

取消恢复链路里，closeout receipt 很重要。它证明取消后的 orphan、side effect、evidence、retention 都收口到了某个终态。

但外部系统不会因为 Agent 写了 receipt 就冻结：

- GitHub 分支会继续被别人 push；
- Telegram 消息可能被编辑、删除、撤回；
- 队列任务可能被其他 worker 抢走；
- cron 下一轮可能已经补跑；
- 账单、支付、部署平台可能有异步 webhook；
- 本地 workspace 也可能被人继续编辑。

因此 retry 前至少要回答三个问题：

~~~text
1. closeout receipt 依赖的 remoteRef 还在吗？
2. closeout receipt 里的 payloadHash / version / status 还匹配吗？
3. 如果已经漂移，漂移是否仍在 replay permit 允许范围内？
~~~

如果这一步缺失，系统会拿着过期收据继续执行，最常见的事故是：旧分支状态下生成的 ReplayPlan，在新 remote head 上重复发消息、重复 push、覆盖新改动。

---

## 2. 最小数据模型

建议把 retry 前探针拆成四个对象：

- **RealityProbePlan**：需要重新检查哪些外部事实；
- **RealitySnapshot**：当前探测到的事实；
- **DriftReport**：旧收据和当前事实的差异；
- **ReplayReadinessDecision**：允许继续、要求重建计划、阻断或升级人工。

一个 DriftReport 可以长这样：

~~~json
{
  "retryOfRunId": "run_456",
  "closeoutReceiptId": "closeout_abc",
  "checkedAt": "2026-06-02T03:30:00+11:00",
  "items": [
    {
      "kind": "git_remote",
      "ref": "origin/main",
      "expected": "c37be93",
      "actual": "f91a221",
      "drift": "changed",
      "severity": "manual_review"
    },
    {
      "kind": "telegram_message",
      "ref": "telegram:-5115329245:13109",
      "expectedPayloadHash": "sha256:old",
      "actualPayloadHash": "sha256:old",
      "drift": "matched",
      "severity": "ok"
    }
  ],
  "decision": "rebuild_replay_plan"
}
~~~

注意这里的重点不是“探测到了什么”，而是把漂移变成可执行决策：

- matched：旧收据还能用；
- changed_safe：轻微变化，可重建 ReplayPlan；
- changed_risky：需要人工确认；
- missing：旧收据引用的现实不存在了；
- conflict：现实和旧收据矛盾，必须阻断或 forward-fix；
- unknown：探针失败，不能假装 matched。

---

## 3. learn-claude-code：教学版漂移判定器

教学版可以先写成纯函数：输入上一轮 closeout 里的 expected facts 和本次 probe 的 actual facts，输出 DriftReport。

~~~python
# learn_claude_code/runtime/pre_replay_drift_probe.py
from dataclasses import dataclass
from typing import Literal

DriftKind = Literal["matched", "changed_safe", "changed_risky", "missing", "conflict", "unknown"]
Decision = Literal["allow_replay", "rebuild_replay_plan", "manual_review", "block_replay"]


@dataclass(frozen=True)
class ExpectedFact:
    fact_id: str
    kind: str
    ref: str
    version: str | None
    payload_hash: str | None
    strict: bool = True


@dataclass(frozen=True)
class ActualFact:
    fact_id: str
    exists: bool
    version: str | None
    payload_hash: str | None
    probe_error: str | None = None


@dataclass(frozen=True)
class DriftItem:
    fact_id: str
    drift: DriftKind
    reason: str


def compare_fact(expected: ExpectedFact, actual: ActualFact | None) -> DriftItem:
    if actual is None:
        return DriftItem(expected.fact_id, "unknown", "probe_not_returned")

    if actual.probe_error:
        return DriftItem(expected.fact_id, "unknown", actual.probe_error)

    if not actual.exists:
        return DriftItem(expected.fact_id, "missing", "external_fact_missing")

    if expected.version and actual.version != expected.version:
        if expected.strict:
            return DriftItem(expected.fact_id, "changed_risky", "version_changed")
        return DriftItem(expected.fact_id, "changed_safe", "version_changed_but_non_strict")

    if expected.payload_hash and actual.payload_hash != expected.payload_hash:
        return DriftItem(expected.fact_id, "conflict", "payload_hash_mismatch")

    return DriftItem(expected.fact_id, "matched", "fact_still_matches_closeout")


def decide_replay_readiness(items: list[DriftItem]) -> Decision:
    drifts = {item.drift for item in items}

    if "conflict" in drifts or "missing" in drifts:
        return "block_replay"

    if "unknown" in drifts or "changed_risky" in drifts:
        return "manual_review"

    if "changed_safe" in drifts:
        return "rebuild_replay_plan"

    return "allow_replay"


def build_drift_report(
    expected: list[ExpectedFact],
    actual_by_id: dict[str, ActualFact],
) -> tuple[list[DriftItem], Decision]:
    items = [compare_fact(fact, actual_by_id.get(fact.fact_id)) for fact in expected]
    return items, decide_replay_readiness(items)
~~~

这个教学版有两个关键测试：

~~~python
def test_payload_mismatch_blocks_replay():
    expected = ExpectedFact("telegram_msg", "telegram", "msg:1", None, "sha256:old")
    actual = ActualFact("telegram_msg", True, None, "sha256:new")

    items, decision = build_drift_report([expected], {"telegram_msg": actual})

    assert items[0].drift == "conflict"
    assert decision == "block_replay"


def test_non_strict_version_change_rebuilds_plan():
    expected = ExpectedFact("repo_head", "git", "origin/main", "a1", None, strict=False)
    actual = ActualFact("repo_head", True, "b2", None)

    items, decision = build_drift_report([expected], {"repo_head": actual})

    assert items[0].drift == "changed_safe"
    assert decision == "rebuild_replay_plan"
~~~

这类测试的价值很高：它把“retry 前必须重新看一眼现实”变成确定性规则，而不是靠 Agent 临场谨慎。

---

## 4. pi-mono：生产版 Probe Worker

pi-mono 这种 TypeScript 生产实现里，PreReplayDriftProbe 不应该塞进每个工具内部，而应该放在 retry orchestration 的入口。

~~~ts
type DriftKind =
  | "matched"
  | "changed_safe"
  | "changed_risky"
  | "missing"
  | "conflict"
  | "unknown";

type ReplayReadinessDecision =
  | "allow_replay"
  | "rebuild_replay_plan"
  | "manual_review"
  | "block_replay";

type ProbeTarget = {
  factId: string;
  kind: "git_remote" | "telegram_message" | "deployment" | "queue_job" | "workspace_file";
  ref: string;
  expectedVersion?: string;
  expectedPayloadHash?: string;
  strict: boolean;
};

type DriftItem = {
  factId: string;
  kind: ProbeTarget["kind"];
  drift: DriftKind;
  reason: string;
  observedVersion?: string;
  observedPayloadHash?: string;
};

class PreReplayDriftProbe {
  constructor(
    private readonly probes: RealityProbeRegistry,
    private readonly reports: DriftReportStore,
  ) {}

  async evaluate(input: {
    retryOfRunId: string;
    closeoutReceiptId: string;
    targets: ProbeTarget[];
  }): Promise<{ decision: ReplayReadinessDecision; items: DriftItem[] }> {
    const snapshots = await Promise.all(
      input.targets.map(async (target) => {
        const probe = this.probes.get(target.kind);
        return probe.snapshot(target).catch((error) => ({
          exists: false,
          probeError: String(error),
        }));
      }),
    );

    const items = input.targets.map((target, index) =>
      compareProbeTarget(target, snapshots[index]),
    );
    const decision = decideReplayReadiness(items);

    await this.reports.save({
      retryOfRunId: input.retryOfRunId,
      closeoutReceiptId: input.closeoutReceiptId,
      decision,
      items,
      checkedAt: new Date(),
    });

    return { decision, items };
  }
}
~~~

然后 retry orchestration 入口要这样串：

~~~ts
class RetryOrchestrator {
  constructor(
    private readonly driftProbe: PreReplayDriftProbe,
    private readonly replayPlanner: ReplayPlanner,
    private readonly queue: RunQueue,
  ) {}

  async requestRetry(request: RetryRequest) {
    const closeout = await loadCloseoutReceipt(request.retryOfRunId);
    const probeTargets = buildProbeTargets(closeout);

    const drift = await this.driftProbe.evaluate({
      retryOfRunId: request.retryOfRunId,
      closeoutReceiptId: closeout.receiptId,
      targets: probeTargets,
    });

    if (drift.decision === "block_replay") {
      return { status: "blocked", reason: "pre_replay_drift_conflict", drift };
    }

    if (drift.decision === "manual_review") {
      return { status: "manual_review", reason: "pre_replay_drift_risky", drift };
    }

    const plan = await this.replayPlanner.build({
      request,
      closeout,
      drift,
      forceRebuild: drift.decision === "rebuild_replay_plan",
    });

    await this.queue.enqueueReplay(plan);
    return { status: "queued", planId: plan.planId };
  }
}
~~~

工程上要注意：probe 失败不能默认 allow。网络超时、权限失败、API 限流都应该先变成 unknown，再由策略决定 manual_review 或延迟重试。

---

## 5. OpenClaw 课程 Cron 的落地例子

拿这个课程 cron 来看，一次 run 有这些外部事实：

- Telegram 群消息：messageId、chatId、payloadHash；
- git remote：origin/main 的 head sha；
- 本地 repo：lesson 文件、README、TOOLS 的 content hash；
- GitHub auth：当前账号必须是 gfwfail；
- memory 记录：今天的课程记录是否已经写入。

如果上一轮在“发 Telegram 成功、git push 前被取消”，retry 前不能只看本地文件。正确流程是：

~~~text
1. 读取上一轮 closeout receipt；
2. probe Telegram messageId 是否存在、内容 hash 是否匹配；
3. probe origin/main 是否仍是上一轮基线；
4. probe lesson/README/TOOLS 是否被其他 run 改过；
5. 生成 DriftReport；
6. 再生成 ReplayPlan：
   - Telegram matched -> reuse_receipt，不再发送；
   - git head changed -> rebuild plan 或 manual_review；
   - local file changed -> 重新 diff，再提交；
   - message missing -> block 或补发纠正消息，不能静默当成功。
~~~

这样 cron 即使被取消、重启、重复触发，也不会因为“旧收据看起来完整”而覆盖新提交或重复轰炸群聊。

---

## 6. 常见踩坑

**坑 1：把 closeout receipt 当永久真相。**
Receipt 是证据，不是锁。外部现实只要还会变化，就必须在 replay 前重新探测。

**坑 2：只 probe 成功路径。**
只检查“消息是否发过”不够，还要检查“消息内容是不是上一轮那条”“有没有被编辑/删除”“下游是否已经基于它继续扩散”。

**坑 3：probe unknown 时默认继续。**
unknown 是一种风险状态，不是 matched。成熟系统宁可延迟重试或人工确认，也不要在看不见现实的时候继续制造副作用。

**坑 4：DriftReport 不参与 ReplayPlan。**
如果漂移报告只是日志，执行层不读它，那它没有工程价值。ReplayPlanner 必须把 DriftReport 转成 allowedSteps、forbiddenEffects 和 requiredManualReview。

---

## 7. 这一课的工程原则

取消恢复链路到这里，已经有一条更完整的 retry 协议：

~~~text
CancelRequest
  -> OrphanReaper
  -> SideEffectReconciliation
  -> EvidenceFreezeSnapshot
  -> SnapshotCloseout
  -> RetryReplayGuard
  -> PreReplayRealityDriftProbe
  -> ReplayPermit
~~~

核心原则很简单：

**retry 前不要相信旧世界。先重新探测现实，再决定旧收据还能不能指导新执行。**

成熟 Agent 的重试不是“再跑一次”，而是带着旧证据、现实探针、漂移报告和执行许可，证明这一次续跑仍然站在当前世界上。
