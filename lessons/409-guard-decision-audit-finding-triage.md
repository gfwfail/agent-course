# 409. Agent 回归护栏审计发现的分级处置与修复队列（Guard Decision Audit Finding Triage & Remediation Queue）

上一课讲了 **Guard Pack Active Decision Sampling Audit & Evidence Budget**：Active Guard Pack 上线后，要对真实 guard decision 做分层采样审计，用 hash / summary / declassified / raw 的证据预算控制成本和隐私风险，并绑定 sampledBecause、counterfactual、outcomeRef 和 audit receipt。

今天继续往后走：**审计抽中了样本，不等于问题已经被处理。**

一句话：**Decision audit sample 发现异常后，要把它升级成结构化 Audit Finding，按 false_allow、false_block、explainability_gap、evidence_overread、policy_stale 等类型分级，进入带 owner、SLA、dedupeKey、repairPath 和 releaseGate 的修复队列；成熟 Agent 不只是会抽查，还会把抽查结果变成可关闭的工程工作流。**

---

## 1. 为什么 sample 不是终点

Decision sampling audit 会产出这样的东西：

~~~json
{
  "sampleId": "sample_20260526_001",
  "decision": "allow",
  "riskSurface": "message_egress",
  "reasonCode": "public_lesson_content",
  "evidenceMode": "declassified",
  "outcomeRef": "telegram_message_12728"
}
~~~

审计器或者人工 reviewer 发现：

~~~text
这条 Telegram 课程允许发送是对的，但 reasonCode 太粗。
它没有证明 lesson 内容经过 secret scan，也没有记录群聊上下文是否为 shared context。
~~~

这不是马上事故，但它是护栏质量问题。

如果只是把它留在审计日志里，后续会变成几种坏味道：

- 同类问题反复出现，但没有去重；
- false allow 和 explainability gap 混在一起，优先级不清；
- 发现了 evidence overread，却没有限制后续 raw access；
- 需要修 guard，却没人知道是改 classifier、scope 还是 prompt；
- 修完以后没有 release gate，下一版 guard 仍然可能复发。

所以 audit sample 之后必须有 **Finding Triage**。

---

## 2. Audit Finding 的核心分类

一个审计发现不要只写“有问题”。至少要分成几类：

~~~text
false_allow
  护栏放行了本应 warn/dry_run/block 的动作。
  风险最高，可能需要冻结同类动作或立即加临时 guard。

false_block
  护栏阻断了本应 allow/warn 的动作。
  影响体验和自动化效率，通常走收紧 scope 或降级。

explainability_gap
  决策可能正确，但 reasonCode/evidenceRefs/counterfactual 不足以解释。
  影响审计、回放和合规。

evidence_overread
  审计或 guard 读取了超过必要范围的证据。
  这是隐私和权限问题，不一定影响原动作正确性。

policy_stale
  规则仍在执行，但业务上下文变了。
  例如 repo、群聊、账号、工具 schema、ownerRole 已变化。

outcome_mismatch
  guard decision 和后续现实不一致。
  例如 guard 记录 allow，但外部系统实际失败；或记录 block，但消息已发出。
~~~

分类的价值是：不同 finding 走不同恢复路径，不能都丢给“修一下策略”。

---

## 3. 最小数据结构：GuardAuditFinding

~~~ts
type FindingKind =
  | "false_allow"
  | "false_block"
  | "explainability_gap"
  | "evidence_overread"
  | "policy_stale"
  | "outcome_mismatch";

type Severity = "low" | "medium" | "high" | "critical";

type GuardAuditFinding = {
  findingId: string;
  dedupeKey: string;
  sourceSampleIds: string[];
  guardPackVersion: number;
  riskSurface: string;
  action: string;
  kind: FindingKind;
  severity: Severity;
  summary: string;
  evidenceRefs: string[];
  affectedGuardIds: string[];
  suggestedRepairPath:
    | "tighten_scope"
    | "loosen_scope"
    | "repair_classifier"
    | "update_reason_codes"
    | "reduce_evidence_access"
    | "reconcile_outcome"
    | "manual_review";
  ownerRole: string;
  ackBy: string;
  resolveBy: string;
  releaseGate: {
    requiresReplay: boolean;
    requiresShadow: boolean;
    requiresCanary: boolean;
    requiredReceipts: string[];
  };
  status: "open" | "acknowledged" | "in_repair" | "verified" | "closed";
};
~~~

关键字段是 dedupeKey 和 releaseGate：

- dedupeKey 避免同一个 reasonCode 缺口被开 200 个 ticket；
- releaseGate 规定修完以后必须通过哪些验证才能关闭；
- suggestedRepairPath 把“问题是什么”和“该怎么修”分开；
- ownerRole 让 finding 不会停在无人队列里。

---

## 4. Triage 不只是 severity 排序

很多系统只会按 critical/high/medium 排序，但 Agent 护栏问题还要看四个维度：

~~~text
blastRadius
  同类动作影响多少 tenant、群聊、repo、tool、risk surface？

reversibility
  如果错放，能不能撤回？Telegram 消息和 git push 都属于外部副作用。

confidence
  审计发现是强证据，还是只是 explainability 不足？

timeSensitivity
  是否正在 active traffic 中持续发生？
~~~

一个 false_allow 如果只影响 sandbox dry-run，可能是 high 但不 urgent。

一个 evidence_overread 如果发生在 message_egress 的 auto_auditor，可能没有立刻造成业务事故，但必须尽快收口，因为它会扩大敏感数据面。

所以 triage 输出不只是 severity，还应该输出 action：

~~~text
freeze_surface
  暂停同类高风险动作，等人工或修复通过。

open_repair_ticket
  进入 guard repair 队列。

tighten_sampling
  临时提高同类样本采样率和 outcome backfill。

reduce_evidence_mode
  立即把 raw/declassified 降到 summary/hash_only。

close_as_expected
  审计发现不成立，记录 reviewer receipt。
~~~

---

## 5. learn-claude-code：纯函数分级器

教学版先把 triage 做成纯函数。输入 audit sample、review result 和风险配置，输出 finding 或 close receipt。

~~~py
from dataclasses import dataclass
from typing import Literal


FindingKind = Literal[
    "false_allow",
    "false_block",
    "explainability_gap",
    "evidence_overread",
    "policy_stale",
    "outcome_mismatch",
]

Severity = Literal["low", "medium", "high", "critical"]
RepairPath = Literal[
    "tighten_scope",
    "loosen_scope",
    "repair_classifier",
    "update_reason_codes",
    "reduce_evidence_access",
    "reconcile_outcome",
    "manual_review",
]


@dataclass(frozen=True)
class AuditReview:
    sample_id: str
    risk_surface: str
    action: str
    decision: str
    expected_decision: str | None
    finding_kind: FindingKind | None
    evidence_mode: str
    reason_complete: bool
    outcome_matched: bool
    external_side_effect: bool
    active_traffic: bool
    affected_count_24h: int


@dataclass(frozen=True)
class TriagePolicy:
    critical_surfaces: set[str]
    external_actions: set[str]
    stale_threshold_24h: int


def dedupe_key(review: AuditReview) -> str:
    return ":".join(
        [
            review.risk_surface,
            review.action,
            review.finding_kind or "none",
            review.expected_decision or review.decision,
        ]
    )


def choose_repair_path(kind: FindingKind) -> RepairPath:
    if kind == "false_allow":
        return "tighten_scope"
    if kind == "false_block":
        return "loosen_scope"
    if kind == "explainability_gap":
        return "update_reason_codes"
    if kind == "evidence_overread":
        return "reduce_evidence_access"
    if kind == "outcome_mismatch":
        return "reconcile_outcome"
    if kind == "policy_stale":
        return "repair_classifier"
    return "manual_review"


def classify_severity(review: AuditReview, policy: TriagePolicy) -> Severity:
    if review.finding_kind == "false_allow":
        if review.risk_surface in policy.critical_surfaces:
            return "critical"
        if review.external_side_effect:
            return "high"
        return "medium"

    if review.finding_kind == "outcome_mismatch":
        return "high" if review.external_side_effect else "medium"

    if review.finding_kind == "evidence_overread":
        return "high" if review.evidence_mode in {"raw", "declassified"} else "medium"

    if review.finding_kind == "false_block":
        return "medium" if review.active_traffic else "low"

    if review.finding_kind == "policy_stale":
        return "medium" if review.affected_count_24h >= policy.stale_threshold_24h else "low"

    return "low"


def triage_audit_review(review: AuditReview, policy: TriagePolicy) -> dict:
    if review.finding_kind is None and review.reason_complete and review.outcome_matched:
        return {
            "action": "close_as_expected",
            "sampleId": review.sample_id,
            "receipt": "audit_review_clean",
        }

    kind = review.finding_kind
    if kind is None:
        kind = "explainability_gap"

    normalized = AuditReview(**{**review.__dict__, "finding_kind": kind})
    severity = classify_severity(normalized, policy)

    immediate_actions: list[str] = []
    if severity == "critical":
        immediate_actions.append("freeze_surface")
    if kind in {"false_allow", "outcome_mismatch"}:
        immediate_actions.append("tighten_sampling")
    if kind == "evidence_overread":
        immediate_actions.append("reduce_evidence_mode")

    return {
        "action": "open_finding",
        "dedupeKey": dedupe_key(normalized),
        "kind": kind,
        "severity": severity,
        "repairPath": choose_repair_path(kind),
        "immediateActions": immediate_actions,
        "releaseGate": {
            "requiresReplay": kind in {"false_allow", "false_block", "policy_stale"},
            "requiresShadow": severity in {"high", "critical"},
            "requiresCanary": severity == "critical",
        },
    }
~~~

这段代码的重点不是算法复杂，而是把“审计发现”变成稳定、可测试、可重复的分级输出。

---

## 6. pi-mono：AuditFindingWorker + EventBus

在 pi-mono 这种事件驱动架构里，可以把审计样本、review 结果和 finding 队列接起来：

~~~ts
type GuardAuditReviewed = {
  type: "guard.audit.reviewed";
  sampleId: string;
  runId: string;
  riskSurface: string;
  action: string;
  decision: "allow" | "warn" | "dry_run" | "block";
  expectedDecision?: "allow" | "warn" | "dry_run" | "block";
  findingKind?: FindingKind;
  evidenceMode: "hash_only" | "summary" | "declassified" | "raw";
  reasonComplete: boolean;
  outcomeMatched: boolean;
  externalSideEffect: boolean;
  activeTraffic: boolean;
  affectedCount24h: number;
};

class AuditFindingWorker {
  constructor(
    private readonly triage: AuditTriageService,
    private readonly findings: FindingQueue,
    private readonly guardRuntime: GuardRuntimeController,
    private readonly eventBus: EventBus,
  ) {}

  async handle(event: GuardAuditReviewed) {
    const result = await this.triage.decide(event);

    if (result.action === "close_as_expected") {
      await this.eventBus.emit({
        type: "guard.audit.sample.closed",
        sampleId: event.sampleId,
        receipt: result.receipt,
      });
      return;
    }

    const finding = await this.findings.upsertByDedupeKey({
      dedupeKey: result.dedupeKey,
      sourceSampleId: event.sampleId,
      kind: result.kind,
      severity: result.severity,
      repairPath: result.repairPath,
      releaseGate: result.releaseGate,
    });

    for (const action of result.immediateActions) {
      if (action === "freeze_surface") {
        await this.guardRuntime.freezeSurface({
          riskSurface: event.riskSurface,
          action: event.action,
          reason: "audit_finding:" + finding.findingId,
        });
      }

      if (action === "reduce_evidence_mode") {
        await this.guardRuntime.reduceEvidenceMode({
          riskSurface: event.riskSurface,
          maxMode: "summary",
          reason: "audit_finding:" + finding.findingId,
        });
      }
    }

    await this.eventBus.emit({
      type: "guard.audit.finding.opened",
      findingId: finding.findingId,
      dedupeKey: result.dedupeKey,
      severity: result.severity,
      repairPath: result.repairPath,
    });
  }
}
~~~

这里的关键点：

- upsertByDedupeKey 防止重复 ticket；
- freezeSurface / reduceEvidenceMode 是即时风险控制；
- finding.opened 事件让后续 Repair Worker 接手；
- releaseGate 写进 ticket，不靠口头约定。

---

## 7. OpenClaw 课程 Cron 实战类比

拿这门课的 cron 来看，第 408 课之后，审计器可能发现这些问题：

~~~text
样本 A：Telegram 发送 allow，内容确实是课程，但没有记录 secret scan receipt
  kind = explainability_gap
  repairPath = update_reason_codes
  releaseGate = replay

样本 B：git push allow，repo 是 gfwfail/agent-course，但 branch 不是 main
  kind = false_allow
  severity = critical
  immediateAction = freeze_surface
  repairPath = tighten_scope
  releaseGate = replay + shadow + canary

样本 C：memory write allow，但 session 是 shared group context
  kind = false_allow
  severity = critical
  immediateAction = freeze_surface

样本 D：auto_auditor 为了检查 lesson 文案读取了 raw MEMORY.md
  kind = evidence_overread
  severity = high
  immediateAction = reduce_evidence_mode
  repairPath = reduce_evidence_access
~~~

这些都不是“日志里记一下”就完事。

课程 cron 的正确 closeout 应该带上：

~~~text
- audit sample 是否关闭；
- finding 是否打开或去重合并；
- 如果有 critical finding，同类外部副作用是否冻结；
- 修复是否绑定 replay/shadow/canary gate；
- 最终是否有 finding close receipt。
~~~

这就是从“我看到了异常”升级到“异常进入可追踪修复流”。

---

## 8. 常见坑

**坑 1：所有 finding 都走人工 review。**

人工是稀缺资源。false_allow critical 要人工，explainability_gap 可以自动开 ticket，evidence_overread 可以先自动降 evidence mode。

**坑 2：没有 dedupeKey。**

一次策略缺陷会在高流量系统里生成几百个 sample。没有去重，队列会被同一个问题淹没。

**坑 3：只修 guard，不处理即时风险。**

critical false_allow 不能等修复发版。要先 freeze surface 或切到 Last Known Good，再慢慢修。

**坑 4：修复 ticket 没有 releaseGate。**

“已修复”必须绑定 replay、shadow、canary、receipt。否则 finding 关闭只是状态字段变了。

**坑 5：把 evidence_overread 当低优先级。**

过度读取证据本身就是安全问题。Agent 系统最容易忽视的泄漏面，往往来自审计和调试工具。

---

## 9. 落地清单

给 Guard Decision Audit 加 finding triage，可以按 8 步做：

1. 定义 Audit Finding 分类：false_allow、false_block、explainability_gap、evidence_overread、policy_stale、outcome_mismatch；
2. 每个 audit review 必须输出 kind、severity、dedupeKey、repairPath；
3. finding queue 用 upsertByDedupeKey 合并重复问题；
4. critical false_allow 自动 freeze 对应 riskSurface/action；
5. evidence_overread 自动降低 evidence mode，并要求权限修复；
6. finding ticket 写入 ownerRole、ackBy、resolveBy；
7. releaseGate 明确 replay / shadow / canary / receipt 要求；
8. finding 关闭前必须验证 outcome backfill 和 repair receipt。

这套机制和前几课连起来就是：

~~~text
decision sampling audit
  -> audit review
  -> finding triage
  -> immediate risk control
  -> repair queue
  -> replay / shadow / canary release gate
  -> finding close receipt
~~~

---

## 10. 总结

这一课的核心：

1. Audit sample 只是发现入口，不是处理闭环；
2. 审计发现要结构化成 GuardAuditFinding，并按 kind / severity / repairPath 路由；
3. false_allow、evidence_overread、outcome_mismatch 要能触发即时风险控制；
4. learn-claude-code 可以用纯函数 triage，让分级策略可测试；
5. pi-mono 可以用 AuditFindingWorker + EventBus，把 finding 接到修复队列；
6. OpenClaw 课程 cron 的 Telegram、Git push、memory write 都适合做 finding triage 的真实练习场。

成熟 Agent 的审计不是“抽查一下安心”，而是把每个异常样本变成有 owner、有 SLA、有去重、有修复门、有关闭收据的工程对象。
