# 398. Agent 回归候选晋级与护栏生命周期（Regression Candidate Promotion & Guardrail Lifecycle）

上一课讲了 **Post-Resumption Observation Window**：人工恢复执行后，不能马上 close，要观察 case、queue、side-effect、policy、user-facing 信号，并把复发风险沉淀成 Regression Guard。

今天继续往下讲：**观察窗口里发现的回归点，不能直接变成永久规则。**

很多团队会在事故后加一堆 if、黑名单、临时检查、特殊 runbook。短期看能止血，长期看会让 Agent 变成一坨“事故纪念碑”：

- 临时 guard 永久存在，没人知道还能不能删；
- 一个边缘 case 被提升成全局阻断，误伤正常任务；
- 规则只记录了结论，没有记录来源和适用范围；
- 旧代码路径已经消失，guard 还在热路径上消耗 token 和延迟；
- 新模型、新工具、新流程上线后，旧 guard 反而拦错地方。

一句话：**Regression Guard 不是写完就结束；它要先作为候选进入晋级闸门，再带着 owner、scope、TTL、证据和复核计划进入生命周期管理。**

---

## 1. 为什么不能直接固化回归护栏

看 OpenClaw 课程 cron 的例子：

1. 某次 git push 超时；
2. 人工确认远端 commit 已存在；
3. 恢复后观察窗口发现：README 已更新，但 TOOLS 去重列表缺本课；
4. 我们想加一个 guard：最终完成前必须检查 lesson、README、TOOLS、Telegram、remote commit。

这个 guard 看起来合理，但它有几个需要明确的问题：

- 只适用于 agent-course 课程 cron，不能影响别的 git 任务；
- Telegram messageId 检查只适用于已发课场景，不适用于草稿生成；
- remote commit 检查依赖网络，失败时应该 hold_reconcile，而不是误判业务失败；
- 如果以后课程发布改成 PR 流程，remote main 检查就要换成 PR merge 状态；
- guard 的证据来自一次事故，强度还不够支持全局 enforce。

所以正确做法不是“立刻加永久规则”，而是创建 **Regression Candidate**。

---

## 2. Regression Candidate 的最小模型

候选对象要保存“这条 guard 从哪来、想防什么、影响哪里、什么时候复核”：

~~~ts
type RegressionCandidate = {
  candidateId: string;
  sourceWindowId: string;
  sourceCaseId: string;
  failureSignature: string;
  proposedGuard: {
    name: string;
    kind: "preflight" | "postcheck" | "replay" | "policy" | "runbook";
    scope: {
      agent: string;
      workflow: string;
      channel?: string;
      repo?: string;
      toolNames: string[];
    };
    expectedDecision: "enforce" | "warn" | "shadow" | "manual_review";
  };
  evidenceRefs: string[];
  owner: string;
  ttlDays: number;
  reviewAt: string;
  status:
    | "candidate"
    | "shadow"
    | "canary"
    | "active"
    | "demoted"
    | "retired";
};
~~~

这里最重要的是 **scope** 和 **expectedDecision**。

没有 scope 的 guard 会变成全局误伤；没有 decision tier 的 guard 会把“提醒一下”和“阻断执行”混在一起。

---

## 3. 晋级闸门看什么

Regression Candidate 进入 active 前，至少过五个检查：

- **scope check**：是否只影响源 workflow，是否避免跨租户、跨频道、跨 repo；
- **evidence check**：是否有原始事故、恢复窗口、复核结果和 replay 证据；
- **false-positive check**：拿最近成功样本回放，确认不会拦住正常任务；
- **cost check**：是否把高延迟网络检查放进热路径，能不能异步或缓存；
- **expiry check**：是否有 owner、reviewAt、ttl，过期后能自动降级或删除。

晋级不是二选一。常见输出应该是：

- promote_shadow：先旁路记录，不影响执行；
- promote_canary：小流量或单 workflow enforce；
- promote_active：证据足够，进入正式 guard pack；
- hold_more_evidence：继续观察；
- reject_overbroad：范围太大，容易误伤；
- manual_review：风险或证据冲突需要人判断。

---

## 4. learn-claude-code：用纯函数做晋级判定

教学版可以把晋级逻辑写成一个纯函数。它不需要数据库，只要输入候选、证据和回放结果。

~~~python
from dataclasses import dataclass
from typing import Literal

Decision = Literal[
    "promote_shadow",
    "promote_canary",
    "promote_active",
    "hold_more_evidence",
    "reject_overbroad",
    "manual_review",
]

@dataclass(frozen=True)
class CandidateScope:
    agent: str
    workflow: str
    repo: str | None
    tool_names: tuple[str, ...]
    affects_all_workflows: bool

@dataclass(frozen=True)
class EvidenceBundle:
    incident_count: int
    observation_window_closed: bool
    replay_passed: bool
    independent_success_samples: int
    has_owner: bool
    ttl_days: int

@dataclass(frozen=True)
class ReplayResult:
    false_positive_rate: float
    added_latency_ms: int
    unknown_rate: float

def decide_regression_promotion(
    scope: CandidateScope,
    evidence: EvidenceBundle,
    replay: ReplayResult,
) -> tuple[Decision, list[str]]:
    if scope.affects_all_workflows:
        return "reject_overbroad", ["scope_affects_all_workflows"]

    if not evidence.has_owner or evidence.ttl_days <= 0:
        return "manual_review", ["missing_owner_or_ttl"]

    if not evidence.observation_window_closed:
        return "hold_more_evidence", ["observation_window_not_closed"]

    if not evidence.replay_passed:
        return "hold_more_evidence", ["replay_not_passed"]

    if replay.false_positive_rate > 0.02:
        return "reject_overbroad", [f"false_positive_rate:{replay.false_positive_rate}"]

    if replay.unknown_rate > 0.10:
        return "promote_shadow", [f"unknown_rate:{replay.unknown_rate}"]

    if replay.added_latency_ms > 500:
        return "promote_shadow", [f"latency_too_high:{replay.added_latency_ms}ms"]

    if evidence.incident_count >= 2 and evidence.independent_success_samples >= 20:
        return "promote_active", ["sufficient_incidents_and_success_replay"]

    return "promote_canary", ["single_incident_or_limited_samples"]
~~~

这个函数故意把“单次事故”默认推向 canary，而不是 active。成熟 Agent 可以从事故学习，但不能让一次偶然样本直接统治所有未来行为。

---

## 5. pi-mono：Guard Pack 版本化发布

生产版不要把 guard 散落在各个 worker 的 if 里，而是发布成版本化 **Guard Pack**：

~~~ts
type GuardPack = {
  packId: string;
  version: number;
  workflow: string;
  guards: Array<{
    guardId: string;
    candidateId: string;
    name: string;
    tier: "shadow" | "canary" | "active";
    scopeHash: string;
    predicateRef: string;
    evidenceRefs: string[];
    owner: string;
    expiresAt: Date;
  }>;
  manifestHash: string;
  publishedAt: Date;
};

async function publishGuardPack(
  tx: DbTransaction,
  candidateId: string,
  decision: "promote_shadow" | "promote_canary" | "promote_active",
) {
  const candidate = await tx.regressionCandidate.findUniqueOrThrow({
    where: { candidateId },
  });

  const tier =
    decision === "promote_active"
      ? "active"
      : decision === "promote_canary"
        ? "canary"
        : "shadow";

  const pack = await tx.guardPack.create({
    data: {
      workflow: candidate.workflow,
      version: { increment: 1 },
      guards: {
        create: {
          candidateId,
          name: candidate.guardName,
          tier,
          scopeHash: hashCanonical(candidate.scope),
          predicateRef: candidate.predicateRef,
          evidenceRefs: candidate.evidenceRefs,
          owner: candidate.owner,
          expiresAt: candidate.expiresAt,
        },
      },
    },
  });

  await tx.outbox.create({
    data: {
      type: "guard_pack_published",
      key: pack.packId,
      payload: {
        workflow: pack.workflow,
        version: pack.version,
        tier,
        candidateId,
      },
    },
  });

  return pack;
}
~~~

Tool Dispatch 或 workflow runner 只读取当前 workflow 的 Guard Pack。这样 guard 能版本化、能回滚、能审计，也能在 shadow/canary/active 之间移动。

---

## 6. OpenClaw 课程 cron 的落地方式

对课程发布任务，一个合适的 Regression Candidate 可以是：

~~~yaml
candidateId: regression-agent-course-closeout-v1
failureSignature: course_published_with_incomplete_closeout
guard:
  kind: postcheck
  workflow: agent-course-cron
  scope:
    repo: gfwfail/agent-course
    channel: telegram:-5115329245
    toolNames:
      - openclaw_message.send
      - git.push
      - file.patch
  checks:
    - lesson_file_exists
    - readme_has_lesson_link
    - tools_has_topic
    - telegram_message_id_recorded
    - remote_main_contains_commit
    - worktree_clean_after_commit
decisionTier: canary
owner: agent-course-maintainer
ttlDays: 30
reviewAt: 2026-06-24T00:00:00+10:00
~~~

注意这里不是写“所有 cron 都必须 remote_main_contains_commit”。它只绑定 agent-course 发布工作流。范围越清楚，guard 越能放心自动执行。

---

## 7. 生命周期：晋级、降级、删除都要有收据

Guard 的生命周期至少包含：

1. **candidate**：从事故或观察窗口生成，只记录建议；
2. **shadow**：只记录 would_block / would_warn，不影响执行；
3. **canary**：只在小范围或低风险路径 enforce；
4. **active**：正式阻断或要求复核；
5. **demoted**：误伤、过期、依赖变化后降级；
6. **retired**：证据过期或代码路径消失后删除热路径影响。

每次状态变化都要写收据：

~~~ts
type GuardLifecycleReceipt = {
  receiptId: string;
  guardId: string;
  from: "candidate" | "shadow" | "canary" | "active" | "demoted" | "retired";
  to: "shadow" | "canary" | "active" | "demoted" | "retired";
  reason: string;
  decisionId: string;
  evidenceRefs: string[];
  actor: "system" | "human";
  createdAt: Date;
};
~~~

没有 lifecycle receipt 的 guard，本质上就是一条没人负责的隐形规则。

---

## 8. 常见坑

- **把事故样本当真理**：一次失败只能支持候选或 canary，不能直接支持全局 active。
- **只加不删**：guard 没有 TTL 和 reviewAt，会慢慢拖垮系统。
- **scope 写太宽**：从课程 cron 学到的规则，不应该影响所有 git workflow。
- **阻断条件依赖慢网络**：慢检查放 postcheck 或异步 reconcile，不要塞进所有工具调用前置路径。
- **没有 shadow 数据**：不知道误伤率，就不知道该不该 enforce。
- **没有 owner**：规则出问题时没人能判断是修、降级还是删除。

---

## 9. 小结

今天的核心：

> Regression Guard 要先作为 Candidate 被证据、scope、误伤率、成本和 TTL 审核，再用 Guard Pack 版本化发布，并在 shadow / canary / active / retired 生命周期里持续管理。

恢复观察窗口回答“这次恢复后稳不稳定”；回归候选晋级回答“这条防复发规则值不值得进入未来的运行路径”。

成熟 Agent 不是事故后到处加 if，而是把每条防复发规则做成有来源、有范围、有证据、有期限、能晋级也能退休的工程资产。
