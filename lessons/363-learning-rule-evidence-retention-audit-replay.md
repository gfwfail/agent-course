# 363. Agent 学习规则证据保留与可审计重放（Learning Rule Evidence Retention & Audit Replay）

上一课讲了学习规则的租户隔离与主体绑定：每条学习都要知道“从哪里学来、能影响谁”。

今天继续补一层生产能力：**学习规则不能只保存结论，还要保存能被审计、能被重放的证据包。**

很多系统会把复盘结果写成一句话：

~~~text
以后发课程前要检查已讲内容，避免重复。
~~~

这句话有用，但还不够。半年后如果它误伤了一个新课程，工程师会问：

- 这条学习是从哪次事故或任务来的？
- 当时有哪些原始输入、工具结果和决策事件？
- 这条规则通过了哪些评估？
- 它有没有被人手动改过？
- 现在还能不能用同一批证据重放出相同结论？

如果回答不了，学习规则就从“工程资产”退化成“神秘 prompt”。

一句话：**Learning Rule 必须带 Evidence Retention，未来才能 audit replay。**

## 1. 为什么学习规则的 evidenceRefs 不够

前面几课已经要求 Learning Candidate 携带 evidenceRefs。但只存几个 ID 还不够。

问题在于 evidence 可能会变化：

- 原始日志被归档，默认查询不到；
- 外部消息被编辑或删除；
- 工具 schema 升级，旧参数解释不同；
- policy 版本升级，旧决策上下文消失；
- LLM 生成候选时的 prompt 没有保存；
- 人工 review 只留了“通过”，没有留原因。

所以学习规则需要一个 **Retention Bundle**：

~~~json
{
  "learningId": "learn.agent-course.no-duplicate-topic",
  "version": "2026-05-20.363",
  "retentionBundle": {
    "sourceRunId": "cron:3eba6ee3-2d11-4afa-aa7f-a47b63226982",
    "evidenceIds": ["ev.tools.topics.read", "ev.lessons.scan", "ev.telegram.sent"],
    "decisionEventIds": ["decision.learning.candidate", "decision.learning.activate"],
    "promptSnapshotHash": "sha256:8f9d...",
    "toolSchemaHash": "sha256:a31c...",
    "policyHash": "sha256:6b20...",
    "replayPackId": "replay.learning.no-duplicate-topic.v1",
    "retainUntil": "2027-05-20T00:00:00Z",
    "minimumRetainedFields": ["scope", "rule", "evidenceRefs", "decisionImpact", "subjectBinding"],
    "redactionProfile": "private-summary-with-proof-hashes"
  }
}
~~~

这不是为了把所有 raw data 永久塞进上下文，而是为了保证未来能回答：

> 这条学习为什么存在？它现在还配不配继续影响决策？

## 2. Retention 的三层：raw、summary、proof

学习证据保留不要一刀切。生产里建议分三层：

- raw：原始输入、工具结果、消息回执、diff、测试输出；
- summary：脱敏后的事实摘要，适合注入 LLM 或给普通审计看；
- proof：hash、签名、时间戳、schema 版本、policy 版本，适合长期验证。

关键规则：

1. raw 不一定长期保留，尤其含隐私或 secret；
2. summary 必须能解释学习结论；
3. proof 必须能证明 summary 没有被悄悄篡改；
4. replay 需要的最小字段必须在 retention policy 里声明；
5. 如果 raw 过期，只能做 summary-level replay，不能假装还能做 full replay。

这样学习规则退场、迁移、漂移检测时，系统知道自己手里证据的可信等级。

## 3. learn-claude-code：Retention Bundle 教学版

教学版可以用纯函数检查学习规则是否携带足够证据。

~~~python
from dataclasses import dataclass
from datetime import datetime, timezone
from enum import Enum

class ReplayLevel(str, Enum):
    FULL = "full"
    SUMMARY = "summary"
    PROOF_ONLY = "proof_only"

@dataclass(frozen=True)
class RetentionBundle:
    source_run_id: str
    evidence_ids: list[str]
    decision_event_ids: list[str]
    prompt_snapshot_hash: str | None
    tool_schema_hash: str | None
    policy_hash: str | None
    replay_pack_id: str | None
    retain_until: datetime
    minimum_retained_fields: set[str]
    redaction_profile: str

@dataclass(frozen=True)
class LearningRule:
    learning_id: str
    rule_text: str
    scope: str
    decision_impact: str
    subject_binding: str
    retention: RetentionBundle | None

REQUIRED_FIELDS = {
    "scope",
    "rule",
    "evidenceRefs",
    "decisionImpact",
    "subjectBinding",
}

def retention_gate(rule: LearningRule, now: datetime) -> tuple[bool, str, ReplayLevel | None]:
    if rule.retention is None:
        return False, "missing_retention_bundle", None

    retention = rule.retention

    if retention.retain_until <= now:
        return False, "retention_expired", None

    if not retention.evidence_ids:
        return False, "missing_evidence_ids", None

    if not retention.decision_event_ids:
        return False, "missing_decision_events", None

    missing = REQUIRED_FIELDS - retention.minimum_retained_fields
    if missing:
        return False, "missing_minimum_fields:" + ",".join(sorted(missing)), None

    if retention.prompt_snapshot_hash and retention.tool_schema_hash and retention.policy_hash:
        if retention.replay_pack_id:
            return True, "full_replay_available", ReplayLevel.FULL
        return True, "summary_replay_available", ReplayLevel.SUMMARY

    return True, "proof_only_audit_available", ReplayLevel.PROOF_ONLY

now = datetime.now(timezone.utc)
~~~

注意这里的返回不是简单 allow / deny，而是给出 replay level。

有些学习规则可以继续作为低风险提示注入，但不能再参与外部副作用决策；原因不是规则一定错，而是证据已经不足以支撑高风险用途。

## 4. Audit Replay：重放学习产生过程，而不是只重放工具

学习规则的 audit replay 至少要重放四个阶段：

1. source reconstruction：还原当时用户输入、工具结果、上下文摘要；
2. candidate extraction：重新生成或验证 Learning Candidate；
3. gate decisions：重跑质量、冲突、主体、权限、兼容闸门；
4. outcome comparison：比较旧 decision 和新 decision 是否一致。

最小 replay report 可以长这样：

~~~json
{
  "learningId": "learn.agent-course.no-duplicate-topic",
  "replayPackId": "replay.learning.no-duplicate-topic.v1",
  "replayedAt": "2026-05-20T05:30:00Z",
  "level": "full",
  "checks": {
    "sourceReconstructed": true,
    "candidateStable": true,
    "qualityGate": "pass",
    "conflictGate": "pass",
    "subjectGate": "pass",
    "permissionBoundaryGate": "pass"
  },
  "decision": {
    "previous": "active",
    "current": "active",
    "fingerprintChanged": false
  }
}
~~~

如果 candidateStable 变成 false，不一定马上删除规则，但至少应该降到 shadow 或 manual_review。成熟系统不能让不可重放的学习继续长期控制生产行为。

## 5. pi-mono：LearningRetentionMiddleware

在 pi-mono 里，Retention Gate 适合放在学习规则进入 prompt pack、policy pack、tool decision 前的统一入口。

~~~ts
type ReplayLevel = "full" | "summary" | "proof_only";

type RetentionBundle = {
  sourceRunId: string;
  evidenceIds: string[];
  decisionEventIds: string[];
  promptSnapshotHash?: string;
  toolSchemaHash?: string;
  policyHash?: string;
  replayPackId?: string;
  retainUntil: string;
  minimumRetainedFields: string[];
  redactionProfile: string;
};

type LearningRule = {
  learningId: string;
  stage: "shadow" | "canary" | "active" | "deprecated" | "retired";
  riskUse: "prompt_hint" | "tool_selection" | "external_side_effect" | "security_decision";
  retention?: RetentionBundle;
  text: string;
};

type RetentionDecision =
  | { allow: true; replayLevel: ReplayLevel; maxRiskUse: LearningRule["riskUse"] }
  | { allow: false; reason: string };

const REQUIRED_FIELDS = [
  "scope",
  "rule",
  "evidenceRefs",
  "decisionImpact",
  "subjectBinding",
];

function retentionDecision(rule: LearningRule, now = new Date()): RetentionDecision {
  const retention = rule.retention;

  if (!retention) {
    return { allow: false, reason: "missing_retention_bundle" };
  }

  if (new Date(retention.retainUntil).getTime() <= now.getTime()) {
    return { allow: false, reason: "retention_expired" };
  }

  for (const field of REQUIRED_FIELDS) {
    if (!retention.minimumRetainedFields.includes(field)) {
      return { allow: false, reason: `missing_minimum_field:${field}` };
    }
  }

  if (retention.evidenceIds.length === 0 || retention.decisionEventIds.length === 0) {
    return { allow: false, reason: "missing_evidence_or_decision_events" };
  }

  const hasProofHashes =
    Boolean(retention.promptSnapshotHash) &&
    Boolean(retention.toolSchemaHash) &&
    Boolean(retention.policyHash);

  if (hasProofHashes && retention.replayPackId) {
    return { allow: true, replayLevel: "full", maxRiskUse: "security_decision" };
  }

  if (hasProofHashes) {
    return { allow: true, replayLevel: "summary", maxRiskUse: "tool_selection" };
  }

  return { allow: true, replayLevel: "proof_only", maxRiskUse: "prompt_hint" };
}

function riskRank(use: LearningRule["riskUse"]): number {
  return {
    prompt_hint: 1,
    tool_selection: 2,
    external_side_effect: 3,
    security_decision: 4,
  }[use];
}

export function filterRetainedLearning(rules: LearningRule[], now = new Date()) {
  const inject: LearningRule[] = [];
  const excluded: Array<{ learningId: string; reason: string }> = [];

  for (const rule of rules) {
    const decision = retentionDecision(rule, now);

    if (!decision.allow) {
      excluded.push({ learningId: rule.learningId, reason: decision.reason });
      continue;
    }

    if (riskRank(rule.riskUse) > riskRank(decision.maxRiskUse)) {
      excluded.push({
        learningId: rule.learningId,
        reason: `retention_level_too_weak_for:${rule.riskUse}`,
      });
      continue;
    }

    inject.push(rule);
  }

  return { inject, excluded };
}
~~~

这里最重要的设计是：**证据保留等级决定学习规则能影响多高风险的决策。**

proof_only 可以当普通提示；summary 可以辅助工具选择；full replay 才能参与安全决策或外部副作用前置判断。

## 6. OpenClaw 实战：课程 Cron 的学习证据保留

拿这个课程 cron 举例，每次更新“已讲内容”时，至少要留下这些证据：

- TOOLS.md 读取片段：证明当前主题没有重复；
- lessons 目录扫描：证明课程序号连续；
- lesson 文件 diff：证明写入了什么；
- README diff：证明目录已更新；
- Telegram messageId：证明课程已外发；
- git commit SHA 和远端 ls-remote：证明仓库已推送；
- memory/YYYY-MM-DD.md：证明本次任务的运行摘要。

如果某条学习规则说：

~~~text
课程 cron 每次发布前必须检查 TOOLS.md 已讲内容，避免重复。
~~~

那它的 retention bundle 应该能追到：

- 哪一次重复风险让这条规则出现；
- 当时检查了哪些文件；
- 后续几次 influence event 是否真的阻止了重复；
- 如果 TOOLS.md 结构变了，replay 是否还能跑。

这就是为什么成熟 Agent 的 memory 不是“写一句经验”就结束，而是要保留足够证据让未来的自己复查。

## 7. 最小落地 Checklist

给学习系统加 Retention Gate 时，先做这 8 件事：

- [ ] Learning Rule schema 增加 retentionBundle；
- [ ] 每条学习绑定 sourceRunId 和 decisionEventIds；
- [ ] evidenceIds 区分 raw / summary / proof；
- [ ] 保存 promptSnapshotHash、toolSchemaHash、policyHash；
- [ ] retention policy 声明 retainUntil 和 redactionProfile；
- [ ] 按 replay level 限制可影响的 riskUse；
- [ ] 定期跑 audit replay，把 unstable 规则降到 shadow；
- [ ] raw 过期时降级 replay level，不假装 full replay 仍可用。

## 8. 常见坑

**坑 1：只保存结论，不保存来源**

这会让学习无法审计。任何 active 学习至少要能追到 sourceRunId、evidenceIds 和 decisionEventIds。

**坑 2：raw data 一删，规则还当高风险证据用**

raw 可以过期，但过期后必须降级。不能用 summary-only 的规则去授权转账、删库、发公开声明。

**坑 3：重放只跑最终结果，不跑 gate**

Audit replay 要重放学习产生过程：候选提取、质量闸门、冲突仲裁、主体绑定、权限边界都要重跑。

**坑 4：Retention 变成无限保存敏感数据**

Retention 不是“永远存 raw”。正确做法是 raw 最小化、summary 脱敏、proof 长期保留，并声明用途边界。

## 总结

Learning Rule Evidence Retention 解决的是一个长期问题：**当 Agent 说“我学会了”，系统必须能证明它为什么学、从哪学、现在还能不能靠这条经验行动。**

成熟 Agent 的学习不是黑盒记忆，而是可审计、可降级、可重放的工程资产。
