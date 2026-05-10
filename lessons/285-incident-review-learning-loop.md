# 285. Agent 事故复盘与根因知识库（Incident Review & Learning Loop）

> 真正成熟的生产 Agent，不是“永远不出错”，而是每次出错后都能把事故变成可复用的知识：下次更早发现、更少误判、更快恢复。

前几课我们讲了变更窗口、发布验证、证据包、Runbook。今天补上运维闭环最后一块：**事故复盘与根因知识库**。

核心思想：

> Incident Review 不是写一篇复盘文档给人看，而是把时间线、根因、触发条件、修复动作和预防规则结构化，回流到 Agent 的监控、策略、Runbook 和记忆系统里。

---

## 1. 为什么 Agent 需要结构化复盘？

很多团队的事故复盘停在 Markdown：

```md
原因：缓存 key 版本没更新，导致旧数据命中。
措施：以后注意。
```

人看完可能有印象，但 Agent 不会自动学会。下一次遇到相似信号，它仍然可能：

- 继续相信 stale cache；
- 继续选择错误 runbook；
- 继续忽略关键告警；
- 继续把“已修复”当成事实，而不是去验证。

所以复盘要升级成机器可读对象：

```json
{
  "incidentId": "inc-2026-05-10-cache-stale",
  "impact": "wrong_recommendation",
  "rootCause": "cache_key_schema_version_missing",
  "triggerSignals": ["cache_hit_rate_spike", "source_version_mismatch"],
  "failedGuardrails": ["pre_action_revalidation_missing"],
  "newRules": [
    "cache key must include schemaVersion",
    "external_side_effect must bypass stale cache"
  ],
  "runbookPatches": ["add sourceVersion probe before publish"]
}
```

这样 Agent 后续才能检索、匹配、提醒、阻断或自动修补。

---

## 2. 复盘闭环的 5 个阶段

一个可执行的 Incident Learning Loop 建议分五步：

1. **Capture**：自动收集时间线、工具调用、证据包、告警、用户反馈。
2. **Classify**：把事故归类：数据过期、权限越界、工具失败、模型误判、发布缺陷、流程缺口。
3. **Explain**：提取 root cause、contributing factors、failed guardrails。
4. **Patch**：生成可执行改进：策略规则、测试用例、Runbook step、监控探针、记忆条目。
5. **Verify**：用回放 / 单元测试 / dry-run 证明补丁真的防住同类事故。

关键点：复盘不是最后一步，**验证补丁有效**才是最后一步。

---

## 3. learn-claude-code：最小 Incident Store

教学版用 JSONL 存复盘记录，并提供一个相似事故检索器。真实系统可以替换成 SQLite、Postgres 或向量库。

```python
# learn_claude_code/incident_learning.py
from __future__ import annotations

import json
import time
from dataclasses import dataclass, asdict, field
from pathlib import Path
from typing import Literal

IncidentClass = Literal[
    "stale_data",
    "tool_failure",
    "permission_gap",
    "model_misjudgment",
    "release_regression",
    "process_gap",
]


@dataclass
class IncidentReview:
    incident_id: str
    klass: IncidentClass
    title: str
    impact: str
    root_cause: str
    trigger_signals: list[str]
    failed_guardrails: list[str]
    new_rules: list[str]
    runbook_patches: list[str] = field(default_factory=list)
    test_cases: list[str] = field(default_factory=list)
    created_at: float = field(default_factory=time.time)


class IncidentStore:
    def __init__(self, path: str):
        self.path = Path(path)
        self.path.parent.mkdir(parents=True, exist_ok=True)

    def append(self, review: IncidentReview) -> None:
        with self.path.open("a", encoding="utf-8") as f:
            f.write(json.dumps(asdict(review), ensure_ascii=False) + "\n")

    def search_similar(self, *, signals: list[str], klass: IncidentClass | None = None) -> list[IncidentReview]:
        signal_set = set(signals)
        matches: list[tuple[int, IncidentReview]] = []

        if not self.path.exists():
            return []

        for line in self.path.read_text(encoding="utf-8").splitlines():
            item = IncidentReview(**json.loads(line))
            if klass and item.klass != klass:
                continue

            score = len(signal_set.intersection(item.trigger_signals))
            if score > 0:
                matches.append((score, item))

        return [item for _, item in sorted(matches, key=lambda pair: pair[0], reverse=True)]


# 使用：事故结束后写入结构化复盘
store = IncidentStore("memory/incidents.jsonl")
store.append(IncidentReview(
    incident_id="inc-cache-schema-version-missing",
    klass="stale_data",
    title="缓存 key 缺少 schemaVersion 导致旧结果命中",
    impact="Agent 基于旧工具结果给出错误发布建议",
    root_cause="cache key builder did not include schemaVersion",
    trigger_signals=["source_version_mismatch", "unexpected_cache_hit"],
    failed_guardrails=["cache_validation", "pre_action_revalidation"],
    new_rules=[
        "cache key must include schemaVersion",
        "external_side_effect must revalidate sourceVersion before execution",
    ],
    runbook_patches=["add cache sourceVersion probe before publish step"],
    test_cases=["replay old cache entry with new schema and expect blocked"],
))
```

这个版本很朴素，但它完成了最重要的一步：把“以后注意”变成 Agent 可以搜索和执行的结构化知识。

---

## 4. pi-mono：Incident Learning Middleware

生产版不要把复盘当孤立文档，而要接到运行时中间件：当风险信号出现时，自动检索类似事故，把教训注入决策层。

```ts
// pi-mono/packages/agent-runtime/src/incidents/incident-learning.ts
export type IncidentClass =
  | "stale_data"
  | "tool_failure"
  | "permission_gap"
  | "model_misjudgment"
  | "release_regression"
  | "process_gap";

export interface IncidentReview {
  incidentId: string;
  klass: IncidentClass;
  title: string;
  impact: string;
  rootCause: string;
  triggerSignals: string[];
  failedGuardrails: string[];
  newRules: string[];
  runbookPatches: string[];
  testCases: string[];
  createdAt: string;
}

export interface IncidentRepository {
  searchSimilar(input: {
    klass?: IncidentClass;
    signals: string[];
    limit: number;
  }): Promise<IncidentReview[]>;
}

export interface PolicyDecision {
  allow: boolean;
  reasons: string[];
  requiredChecks: string[];
}

export class IncidentLearningMiddleware {
  constructor(private readonly repo: IncidentRepository) {}

  async beforeRiskyAction(input: {
    action: string;
    risk: "low" | "medium" | "high";
    signals: string[];
    baseDecision: PolicyDecision;
  }): Promise<PolicyDecision> {
    if (input.risk === "low") return input.baseDecision;

    const similar = await this.repo.searchSimilar({
      signals: input.signals,
      limit: 3,
    });

    if (similar.length === 0) return input.baseDecision;

    const requiredChecks = new Set(input.baseDecision.requiredChecks);
    const reasons = [...input.baseDecision.reasons];

    for (const incident of similar) {
      reasons.push(`similar_incident:${incident.incidentId}:${incident.rootCause}`);

      for (const guardrail of incident.failedGuardrails) {
        requiredChecks.add(`verify_guardrail:${guardrail}`);
      }

      for (const rule of incident.newRules) {
        requiredChecks.add(`rule:${rule}`);
      }
    }

    return {
      allow: input.baseDecision.allow,
      reasons,
      requiredChecks: [...requiredChecks],
    };
  }
}
```

这里的关键不是“让模型想起以前的事故”，而是让 Policy / Runbook / Tool Middleware 强制执行额外检查。

LLM 可以负责解释相似性，但阻断动作应该由确定性策略完成。

---

## 5. OpenClaw 实战：把复盘写回长期能力

OpenClaw 里可以用三层落地：

### 5.1 Daily memory：记录原始事故

```md
## Incident - 2026-05-10
- Symptom: 课程 cron push 前 README 已落后远端
- Root cause: 未先 pull 最新 main
- Fix: git pull --rebase --autostash origin main
- Prevention: push 前必须 gh pr list + pull + diff --check
```

### 5.2 MEMORY.md / TOOLS.md：沉淀规则

适合长期保留的规则才提升到 `MEMORY.md` 或项目 `TOOLS.md`，例如：

```md
- 每次 git push 前必须检查 PR 状态、pull 最新 main、跑 diff --check。
```

### 5.3 Runbook / Gate：把教训变成闸门

比如课程 cron 的完成闸门可以写成：

```json
{
  "gate": "agent_course_publish",
  "required": [
    "lesson_file_exists",
    "readme_contains_lesson",
    "tools_contains_topic",
    "telegram_message_id_recorded",
    "git_remote_synced",
    "git_diff_check_passed",
    "git_push_succeeded"
  ]
}
```

这就是复盘闭环：

> 事故 → 结构化记录 → 长期规则 → 自动闸门 → 回放验证。

---

## 6. 一个很实用的设计细节：Action Item 要可验证

坏的 action item：

```md
以后发布前更仔细检查。
```

好的 action item：

```json
{
  "type": "completion_gate",
  "name": "require_git_diff_check_before_push",
  "verifyCommand": "git diff --check",
  "blocksOnFailure": true
}
```

判断标准很简单：

- 能不能被 CI / Cron / Agent 自动执行？
- 失败时能不能阻断副作用？
- 有没有 owner 和证据？
- 能不能回放一次旧事故证明它会失败？

如果不能验证，它就只是愿望，不是修复。

---

## 7. 常见坑

### 坑 1：复盘只写给人看

人类总结很好，但 Agent 需要结构化字段：rootCause、signals、rules、tests、runbookPatches。

### 坑 2：把“模型提示词更新”当唯一修复

Prompt 可以提醒，但真正的防线应该落到 Policy、Tool Middleware、Completion Gate、Runbook。

### 坑 3：没有验证 action item

复盘写了 10 条措施，但没有任何测试或 gate，下一次还是靠记忆和运气。

### 坑 4：所有事故都进长期记忆

不是所有细节都值得永久保存。建议：

- 原始细节进 daily memory / incident store；
- 可复用规律进 MEMORY.md / policy；
- 敏感数据只存引用和 hash，不存原文。

---

## 8. 小结

今天这课可以记住一句话：

> 成熟 Agent 的复盘不是“写总结”，而是把事故转成可搜索、可执行、可验证的系统改进。

落地 Checklist：

- [ ] 每次事故生成结构化 IncidentReview。
- [ ] 记录 trigger signals、root cause、failed guardrails。
- [ ] Action item 必须能变成 rule / gate / runbook patch / test。
- [ ] 高风险动作前检索相似事故并追加 required checks。
- [ ] 用回放证明新规则能挡住旧事故。

复盘真正的价值不是解释过去，而是让下一次事故在发生前就被拦下来。
