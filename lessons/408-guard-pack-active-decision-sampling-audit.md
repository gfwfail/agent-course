# 408. Agent 回归护栏包激活后的决策采样审计与证据预算（Guard Pack Active Decision Sampling Audit & Evidence Budget）

上一课讲了 **Guard Pack Active Runtime Drift & Auto-Reconciliation**：LKG promotion 成功后，还要持续证明每个 runtime 的 guard pack、executor、schema、tool schema、prompt pack 和只读 probe decision 都一致；发现漂移优先局部 refresh / quarantine，而不是盲目全局 rollback。

今天继续往后走：**runtime 没漂移，也不代表 active guard 的真实决策一直健康。**

一句话：**Active Guard Pack 需要对真实 guard decision 做分层采样审计，用 evidence budget 控制原始证据读取成本和隐私风险，再把 sampled decision、outcome、reason code、counterfactual 和 audit receipt 串起来；成熟 Agent 不只证明护栏版本一致，还要持续抽查它在真实流量里有没有挡对、放对、解释对。**

---

## 1. 为什么还需要决策采样

Runtime drift monitor 解决的是：

~~~text
同一份 Guard Pack 是否被所有 runtime 同样解释？
~~~

但它没有回答另一个问题：

~~~text
这份 Guard Pack 在真实生产决策里是否仍然表现健康？
~~~

比如所有 runtime 都一致执行 v43，没有 drift：

- 私密证据外发到群聊：block，一致；
- main 分支直接 push：block，一致；
- 普通 lesson 文件写入：allow，一致；
- 低风险 README 更新：allow，一致。

看起来没问题。

但真实流量里可能出现新形态：

- 文件里混入了 access token，但 classifier 没识别；
- Telegram 文案引用了 raw evidence 的一句话，策略只看 evidenceRef 没看 payload；
- git push 是 gfwfail 账号，但目标 repo 不是课程 repo；
- memory 写入发生在 cron session，分类器误判成 main session；
- allow 的理由码还是旧 reasonCode，审计时无法解释为什么放行。

这些不是 runtime drift，而是 **active decision quality** 问题。

所以 active 后要做决策采样审计：不是每次都全量人工复核，而是按风险、变化、历史异常和证据敏感度抽样。

---

## 2. 不要全量审计所有决策

最简单的做法是每个 guard decision 都存完整上下文：

~~~json
{
  "decision": "allow",
  "toolArgs": "...",
  "prompt": "...",
  "evidencePayload": "...",
  "messages": "..."
}
~~~

这在生产里很快会出问题：

- 成本高：每个工具调用都写大日志；
- 隐私风险高：raw evidence 进入审计库；
- 噪声大：低风险 allow 会淹没真正需要看的样本；
- LLM 复核成本不可控；
- 审计系统本身变成新的敏感数据仓库。

正确模型是 **分层采样 + 证据预算**：

~~~text
low risk allow        -> 只记录摘要和 hash，极低采样率
medium risk warn      -> 记录 reasonCode、evidenceRef，部分采样
high risk dry_run     -> 记录 declassified summary，高采样率
critical block/allow  -> 强制采样，必要时人工复核
policy changed area   -> 临时提高采样率
recent incident area  -> 提高采样率并绑定 regression refs
~~~

核心原则：**审计要足够看见问题，但不能把每次决策都变成新的数据泄漏面。**

---

## 3. 最小数据结构：DecisionAuditSample

一个可用的采样事件不用保存全部 raw payload。

~~~ts
type GuardDecision = "allow" | "warn" | "dry_run" | "block";

type DecisionAuditSample = {
  sampleId: string;
  runId: string;
  runtimeId: string;
  guardPackVersion: number;
  guardPackHash: string;
  riskSurface: string;
  action: string;
  decision: GuardDecision;
  reasonCode: string;
  guardIds: string[];
  evidenceRefs: string[];
  evidenceMode: "none" | "hash_only" | "summary" | "declassified" | "raw";
  payloadHash: string;
  sampledBecause: string[];
  counterfactual?: {
    previousLkgDecision: GuardDecision;
    shadowCandidateDecision?: GuardDecision;
  };
  outcomeRef?: string;
  createdAt: string;
};
~~~

这里有几个关键点：

- payloadHash 用来证明审计时看到的是同一个 payload，但默认不存 payload；
- evidenceMode 明确这次审计拿到了多少证据；
- sampledBecause 要能解释为什么这条被抽中；
- counterfactual 用来判断新旧 guard 的差异；
- outcomeRef 延迟绑定真实结果，比如消息是否发出、push 是否成功、用户是否纠正。

采样审计不是日志堆积，而是有目的地生成可复核样本。

---

## 4. Evidence Budget：谁能看多少证据

采样后最大的问题是：审计器要不要读取 raw evidence？

答案不能写死，要由 Evidence Budget 决定。

~~~ts
type EvidenceBudget = {
  actor: "auto_auditor" | "human_reviewer" | "incident_forensics";
  riskSurface: string;
  maxSamplesPerHour: number;
  maxRawReadsPerHour: number;
  allowedModes: Array<"hash_only" | "summary" | "declassified" | "raw">;
  requireApprovalForRaw: boolean;
};
~~~

举例：

~~~text
message_egress + auto_auditor:
  allowedModes = ["hash_only", "summary", "declassified"]
  rawReads = 0

git_push + auto_auditor:
  allowedModes = ["hash_only", "summary"]
  rawReads = 0

incident_forensics + human_reviewer:
  allowedModes = ["summary", "declassified", "raw"]
  rawReads = 5/hour
  requireApprovalForRaw = true
~~~

这跟前面证据课程的思路一致：Agent 可以做审计，但审计器不应该天然拥有无限 raw access。

---

## 5. learn-claude-code：纯函数采样判定器

教学版先把采样策略做成纯函数。输入 guard decision、风险配置和预算状态，输出 skip 或 sample，以及允许的证据模式。

~~~py
from dataclasses import dataclass
from typing import Literal
import hashlib


Decision = Literal["allow", "warn", "dry_run", "block"]
EvidenceMode = Literal["none", "hash_only", "summary", "declassified", "raw"]


@dataclass(frozen=True)
class GuardDecisionEvent:
    run_id: str
    risk_surface: str
    action: str
    decision: Decision
    reason_code: str
    guard_ids: tuple[str, ...]
    payload: str
    policy_changed: bool = False
    recent_incident: bool = False
    user_reported_issue: bool = False


@dataclass(frozen=True)
class SamplingPolicy:
    base_rate_per_thousand: int
    high_risk_surfaces: set[str]
    critical_actions: set[str]
    always_sample_decisions: set[Decision]


@dataclass(frozen=True)
class BudgetState:
    samples_used: int
    max_samples: int
    raw_reads_used: int
    max_raw_reads: int
    allowed_modes: tuple[EvidenceMode, ...]


def stable_bucket(seed: str) -> int:
    digest = hashlib.sha256(seed.encode("utf-8")).hexdigest()
    return int(digest[:8], 16) % 1000


def decide_audit_sample(
    event: GuardDecisionEvent,
    policy: SamplingPolicy,
    budget: BudgetState,
) -> dict:
    if budget.samples_used >= budget.max_samples:
        return {"action": "skip", "reason": "sample_budget_exhausted"}

    reasons: list[str] = []

    if event.decision in policy.always_sample_decisions:
        reasons.append("decision_requires_sampling")
    if event.risk_surface in policy.high_risk_surfaces:
        reasons.append("high_risk_surface")
    if event.action in policy.critical_actions:
        reasons.append("critical_action")
    if event.policy_changed:
        reasons.append("policy_changed_surface")
    if event.recent_incident:
        reasons.append("recent_incident_surface")
    if event.user_reported_issue:
        reasons.append("user_reported_issue")

    bucket = stable_bucket(f"{event.run_id}:{event.risk_surface}:{event.action}")
    if bucket < policy.base_rate_per_thousand:
        reasons.append("baseline_sample")

    if not reasons:
        return {"action": "skip", "reason": "not_selected"}

    mode: EvidenceMode = "hash_only"
    if "declassified" in budget.allowed_modes and (
        event.risk_surface in policy.high_risk_surfaces
        or event.decision in {"dry_run", "block"}
    ):
        mode = "declassified"
    elif "summary" in budget.allowed_modes:
        mode = "summary"

    payload_hash = hashlib.sha256(event.payload.encode("utf-8")).hexdigest()

    return {
        "action": "sample",
        "sampledBecause": reasons,
        "evidenceMode": mode,
        "payloadHash": payload_hash,
        "reasonCode": event.reason_code,
        "guardIds": list(event.guard_ids),
    }
~~~

这里故意用 stable_bucket，不是随机数。这样同一个 runId / riskSurface / action 在重放时会得到同样采样结果，方便离线 replay 和审计复现。

---

## 6. pi-mono：用事件流做旁路审计

pi-mono 的 packages/mom/src/agent.ts 里已经对 session.subscribe 做了工具执行事件监听：tool_execution_start 记录参数和开始时间，tool_execution_end 记录结果和耗时。

Active Decision Sampling Audit 可以沿用这个模式，但放在 guard decision 事件旁路：

~~~ts
type GuardDecisionObserved = {
  type: "guard.decision.observed";
  runId: string;
  runtimeId: string;
  guardPackVersion: number;
  guardPackHash: string;
  riskSurface: string;
  action: string;
  decision: "allow" | "warn" | "dry_run" | "block";
  reasonCode: string;
  guardIds: string[];
  evidenceRefs: string[];
  payloadHash: string;
};

class DecisionAuditSubscriber {
  constructor(
    private readonly sampler: DecisionSampler,
    private readonly evidence: EvidenceViewService,
    private readonly auditStore: AuditStore,
  ) {}

  attach(agent: Agent) {
    agent.subscribe(async (event) => {
      if (event.type !== "guard.decision.observed") return;

      const sample = await this.sampler.decide(event as GuardDecisionObserved);
      if (sample.action === "skip") return;

      const evidenceView = await this.evidence.view({
        refs: event.evidenceRefs,
        mode: sample.evidenceMode,
        purpose: "guard_decision_audit",
      });

      await this.auditStore.append({
        ...sample,
        runId: event.runId,
        riskSurface: event.riskSurface,
        action: event.action,
        decision: event.decision,
        reasonCode: event.reasonCode,
        guardIds: event.guardIds,
        evidenceViewRef: evidenceView.viewId,
      });
    });
  }
}
~~~

重点是旁路：主执行路径只发出 guard decision event，审计器异步消费。审计失败不能让普通低风险工具调用全局卡死；但 critical action 可以配置成同步 barrier，必须写入 audit receipt 后才继续。

---

## 7. OpenClaw 课程 Cron 实战类比

拿这门课的 cron 来看，一次成功运行至少有这些 guard decision：

1. 是否允许读取 TOOLS.md 已讲列表；
2. 是否允许生成 lesson 文件；
3. 是否允许向 Telegram 群发送课程内容；
4. 是否允许切换 gfwfail 账号；
5. 是否允许 push 到 gfwfail/agent-course main；
6. 是否允许把本次结果写入 memory/TOOLS.md。

不是每条都需要同样审计。

可以这样采样：

~~~json
{
  "message_egress": {
    "sample": "always",
    "evidenceMode": "declassified",
    "why": "群聊外发，必须证明没有 secret/raw 私人记忆"
  },
  "git_push": {
    "sample": "always",
    "evidenceMode": "summary",
    "why": "外部副作用，必须证明 repo、branch、identity 正确"
  },
  "file_write": {
    "sample": "20%",
    "evidenceMode": "hash_only",
    "why": "本地可回滚，低风险但要抽查"
  },
  "memory_write": {
    "sample": "always_if_shared_context",
    "evidenceMode": "summary",
    "why": "防止把私人长期记忆写到不该写的位置"
  }
}
~~~

这能补上一个很实际的缺口：我们不仅知道“这次发群、写文件、push 成功了”，还知道每个关键动作在执行前被哪个 guard 放行、为什么放行、抽样审计看到了多少证据。

---

## 8. 常见坑

**坑 1：采样只看随机比例。**

高风险动作、刚改过的 policy、近期事故风险面、用户纠错过的路径，都应该提高采样率。随机 1% 不等于安全。

**坑 2：审计器默认读取 raw evidence。**

审计也是权限边界。默认用 hash、summary、declassified view；只有 incident forensics 才按审批读取 raw。

**坑 3：只审计 block，不审计 allow。**

真正危险的常常是错误 allow。block 误伤影响体验，allow 放错可能造成外部副作用或数据泄漏。

**坑 4：没有 outcome 绑定。**

单条 decision 只能说明当时为什么放行；绑定后续 outcome，才能判断 guard 是否真的挡住了坏结果、有没有误伤好结果。

**坑 5：采样不可重放。**

采样要用稳定 hash bucket，而不是不可复现随机数。否则 incident replay 时解释不了为什么当时抽中或没抽中。

---

## 9. 落地清单

给 active Guard Pack 加决策采样审计，可以从 7 件事开始：

1. guard executor 每次输出 guard.decision.observed 事件；
2. 采样策略同时看 risk surface、decision、action、policy change、incident tag；
3. 用 stable hash bucket 做可重放采样；
4. Evidence Budget 限制每类审计 actor 的 raw 读取能力；
5. audit sample 默认存 hash / summary / declassified view，不默认存 raw；
6. critical external side effect 绑定同步 audit receipt；
7. 后续 outcome 回填到 sample，用来评估 false allow、false block 和 unknown。

这套机制和前几课连起来就是：

~~~text
Guard Pack active
  -> runtime drift monitor 证明执行一致
  -> decision sampling audit 抽查真实决策质量
  -> outcome backfill 证明挡对/放对
  -> effectiveness telemetry 决定保留、收紧、降级或修复
~~~

---

## 10. 总结

这一课的核心：

1. Runtime 没漂移，只能证明执行一致，不能证明真实决策一直健康；
2. Active Guard Pack 要对真实 guard decision 做分层采样审计；
3. Evidence Budget 决定审计器能看 hash、summary、declassified 还是 raw；
4. learn-claude-code 可以用纯函数采样判定器和 stable bucket 做可重放抽样；
5. pi-mono 可以用 Agent.subscribe 旁路消费 guard decision event；
6. OpenClaw 课程 cron 里的 Telegram、Git push、memory write 都适合作为 always-sample 的关键副作用。

成熟 Agent 的护栏不是“版本一致就放心”，而是持续抽样真实决策：为什么放行、为什么阻断、用了多少证据、结果是否证明当时的判断是对的。
