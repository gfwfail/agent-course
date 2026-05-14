# 317. Agent 证据索引与审计查询（Evidence Index & Audit Query）

上一课讲了 **Evidence Retention Lifecycle**：证据要有保留期、删除期、legal hold 和销毁证明。

今天继续补证据系统的下一块：**证据不能只“存起来”，还要能被安全地查出来。**

这叫 **Evidence Index & Audit Query**：每条证据写入时同步生成可查询索引，审计人员、策略门控、事故复盘和自动验证器都通过受控查询拿到最小必要证据，而不是直接扫原始日志、翻聊天记录、grep 敏感文件。

一句话：**Evidence Store 负责保存事实，Evidence Index 负责让事实可追踪、可筛选、可证明。**

---

## 1. 为什么只存 Evidence 不够？

很多 Agent 系统一开始会把证据写成文件：

- `evidence/ev_001.raw.json`
- `evidence/ev_001.meta.json`
- `evidence/tombstones.jsonl`
- `audit/tool-calls.jsonl`

这能留痕，但遇到真实问题会很痛：

- “昨晚所有 git push 前有没有跑测试？”
- “某个 Telegram 消息是谁发的？基于哪个用户请求？”
- “这次部署失败前，policy 为什么 allow？”
- “哪些 raw_tool_output 已经过期但还没删？”
- “这个 incident 相关的证据链是否完整？”

如果每次都 grep 文件，两个问题很快出现：

1. **慢**：证据越来越多，扫全量文件不可持续
2. **危险**：查询者容易直接看到 raw evidence，绕过最小披露和权限控制

所以生产系统要把 evidence 分成两层：

- **Raw Evidence**：原始证据，受权限和保留策略保护
- **Evidence Index**：可查询索引，只放 metadata、hash、时间、主体、动作、关联关系

---

## 2. Evidence Index 应该索引什么？

不要把原文塞进索引。索引要回答的是“有没有、在哪里、和谁相关、是否可信”，不是复刻 raw data。

推荐字段：

```json
{
  "evidenceId": "ev_20260514_push_01",
  "kind": "git_push_receipt",
  "actor": "agent:cron-agent-course",
  "subject": "repo:gfwfail/agent-course",
  "action": "git.push",
  "risk": "medium",
  "status": "ok",
  "createdAt": "2026-05-14T08:30:00Z",
  "runId": "run_abc123",
  "intentId": "intent_lesson_317",
  "policyDecisionId": "pd_789",
  "artifactRef": "artifact://evidence/ev_20260514_push_01.raw.json",
  "contentHash": "sha256:...",
  "sensitivity": "internal",
  "retentionState": "active",
  "tags": ["agent-course", "telegram", "git"]
}
```

核心设计原则：

- `actor`：谁执行的
- `subject`：影响了什么对象
- `action`：做了什么动作
- `runId / intentId`：为什么做、属于哪个任务
- `policyDecisionId`：当时策略如何判断
- `artifactRef + contentHash`：原始证据在哪里、是否被篡改
- `sensitivity`：查询时如何脱敏
- `retentionState`：active / expired / tombstoned / legal_hold

---

## 3. learn-claude-code：SQLite 教学版索引

先写一个最小 EvidenceIndex。SQLite 足够教学，也很适合单机 Agent、Cron、桌面 OpenClaw。

```python
# evidence_index.py
from __future__ import annotations

from dataclasses import dataclass
from datetime import datetime, timezone
import json
import sqlite3
from typing import Literal

Risk = Literal["low", "medium", "high", "critical"]
Status = Literal["ok", "failed", "blocked", "expired", "tombstoned"]

@dataclass
class EvidenceIndexEntry:
    evidence_id: str
    kind: str
    actor: str
    subject: str
    action: str
    risk: Risk
    status: Status
    created_at: str
    run_id: str
    intent_id: str | None
    policy_decision_id: str | None
    artifact_ref: str
    content_hash: str
    sensitivity: str
    tags: list[str]

class EvidenceIndex:
    def __init__(self, path: str = "./evidence-index.sqlite3"):
        self.db = sqlite3.connect(path)
        self.db.row_factory = sqlite3.Row
        self._init_schema()

    def _init_schema(self) -> None:
        self.db.execute("""
        CREATE TABLE IF NOT EXISTS evidence_index (
          evidence_id TEXT PRIMARY KEY,
          kind TEXT NOT NULL,
          actor TEXT NOT NULL,
          subject TEXT NOT NULL,
          action TEXT NOT NULL,
          risk TEXT NOT NULL,
          status TEXT NOT NULL,
          created_at TEXT NOT NULL,
          run_id TEXT NOT NULL,
          intent_id TEXT,
          policy_decision_id TEXT,
          artifact_ref TEXT NOT NULL,
          content_hash TEXT NOT NULL,
          sensitivity TEXT NOT NULL,
          tags_json TEXT NOT NULL
        )
        """)
        self.db.execute("CREATE INDEX IF NOT EXISTS idx_evidence_run ON evidence_index(run_id)")
        self.db.execute("CREATE INDEX IF NOT EXISTS idx_evidence_action_time ON evidence_index(action, created_at)")
        self.db.execute("CREATE INDEX IF NOT EXISTS idx_evidence_subject ON evidence_index(subject)")
        self.db.commit()

    def add(self, entry: EvidenceIndexEntry) -> None:
        self.db.execute("""
        INSERT OR REPLACE INTO evidence_index (
          evidence_id, kind, actor, subject, action, risk, status,
          created_at, run_id, intent_id, policy_decision_id,
          artifact_ref, content_hash, sensitivity, tags_json
        ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
        """, (
            entry.evidence_id,
            entry.kind,
            entry.actor,
            entry.subject,
            entry.action,
            entry.risk,
            entry.status,
            entry.created_at,
            entry.run_id,
            entry.intent_id,
            entry.policy_decision_id,
            entry.artifact_ref,
            entry.content_hash,
            entry.sensitivity,
            json.dumps(entry.tags, ensure_ascii=False),
        ))
        self.db.commit()

    def query(self, *, action: str | None = None, subject: str | None = None,
              run_id: str | None = None, since: str | None = None, limit: int = 50) -> list[dict]:
        clauses = []
        args = []

        if action:
            clauses.append("action = ?")
            args.append(action)
        if subject:
            clauses.append("subject = ?")
            args.append(subject)
        if run_id:
            clauses.append("run_id = ?")
            args.append(run_id)
        if since:
            clauses.append("created_at >= ?")
            args.append(since)

        where = "WHERE " + " AND ".join(clauses) if clauses else ""
        rows = self.db.execute(f"""
          SELECT * FROM evidence_index
          {where}
          ORDER BY created_at DESC
          LIMIT ?
        """, [*args, limit]).fetchall()

        return [dict(row) for row in rows]
```

使用时，EvidenceStore 写 raw/meta，EvidenceIndex 写可查询摘要：

```python
index = EvidenceIndex()

index.add(EvidenceIndexEntry(
    evidence_id="ev_20260514_push_01",
    kind="git_push_receipt",
    actor="agent:agent-course-cron",
    subject="repo:gfwfail/agent-course",
    action="git.push",
    risk="medium",
    status="ok",
    created_at=datetime.now(timezone.utc).isoformat(),
    run_id="run_lesson_317",
    intent_id="intent_publish_course",
    policy_decision_id="pd_allow_001",
    artifact_ref="artifact://evidence/ev_20260514_push_01.raw.json",
    content_hash="sha256:abc...",
    sensitivity="internal",
    tags=["course", "git", "telegram"],
))

print(index.query(action="git.push", subject="repo:gfwfail/agent-course"))
```

这一步的价值不是技术复杂，而是把“找证据”从人工 grep 变成可控 API。

---

## 4. 审计查询不能直接返回 raw

查询接口默认返回 **Proof Summary**，不是 raw evidence。

```python
class EvidenceQueryService:
    def __init__(self, index: EvidenceIndex):
        self.index = index

    def search(self, *, actor: str, purpose: str, **filters) -> list[dict]:
        rows = self.index.query(**filters)
        return [self._project(row, actor=actor, purpose=purpose) for row in rows]

    def _project(self, row: dict, *, actor: str, purpose: str) -> dict:
        # 默认最小披露：不返回 artifact_ref 原始路径，除非 purpose 允许
        can_open_raw = purpose in {"incident_review", "security_audit"}

        result = {
            "evidenceId": row["evidence_id"],
            "kind": row["kind"],
            "subject": row["subject"],
            "action": row["action"],
            "status": row["status"],
            "risk": row["risk"],
            "createdAt": row["created_at"],
            "runId": row["run_id"],
            "contentHash": row["content_hash"],
            "sensitivity": row["sensitivity"],
        }

        if can_open_raw and row["sensitivity"] != "secret":
            result["artifactRef"] = row["artifact_ref"]

        return result
```

注意这里的默认策略：

- 普通验证只看 `status/hash/time/runId`
- 事故复盘可以拿 `artifactRef`
- secret 级证据永远不通过普通查询返回原文位置
- 查询本身也要写 audit log

成熟 Agent 的证据查询不是“谁都能搜日志”，而是“按目的授权后的最小披露”。

---

## 5. pi-mono：生产版 EvidenceQueryMiddleware

在 pi-mono 里，可以把证据索引放在 Tool Middleware / Policy Middleware 旁边。

```ts
type EvidenceQuery = {
  actor: string;
  purpose: "completion_gate" | "incident_review" | "security_audit" | "user_export";
  filters: {
    action?: string;
    subject?: string;
    runId?: string;
    since?: string;
    risk?: "low" | "medium" | "high" | "critical";
  };
  limit?: number;
};

type EvidenceSummary = {
  evidenceId: string;
  kind: string;
  subject: string;
  action: string;
  status: string;
  risk: string;
  createdAt: string;
  runId: string;
  contentHash: string;
  artifactRef?: string;
};

class EvidenceQueryMiddleware {
  constructor(
    private readonly index: EvidenceIndexRepository,
    private readonly policy: EvidenceAccessPolicy,
    private readonly audit: AuditLog,
  ) {}

  async search(query: EvidenceQuery): Promise<EvidenceSummary[]> {
    const decision = await this.policy.decide(query);

    await this.audit.append({
      event: "evidence_query_requested",
      actor: query.actor,
      purpose: query.purpose,
      filters: query.filters,
      decision: decision.action,
      reasons: decision.reasons,
      at: new Date().toISOString(),
    });

    if (decision.action === "deny") {
      throw new Error(`evidence_query_denied: ${decision.reasons.join(",")}`);
    }

    const rows = await this.index.search(query.filters, query.limit ?? 50);

    return rows.map((row) => ({
      evidenceId: row.evidenceId,
      kind: row.kind,
      subject: row.subject,
      action: row.action,
      status: row.status,
      risk: row.risk,
      createdAt: row.createdAt,
      runId: row.runId,
      contentHash: row.contentHash,
      artifactRef: decision.allowArtifactRef && row.sensitivity !== "secret"
        ? row.artifactRef
        : undefined,
    }));
  }
}
```

这个中间件应该强制三件事：

1. 查询前过 policy
2. 查询结果按 purpose 投影
3. 查询动作本身进入审计日志

---

## 6. OpenClaw 实战：课程 Cron 完成闸门

以这个课程 Cron 为例，一次自动发布至少产生这些证据：

- lesson 文件创建证据
- README 更新证据
- TOOLS.md 已讲内容更新证据
- Telegram messageId 投递回执
- git commit hash
- git push 回执

完成前不要只说“我觉得发完了”，而是查索引：

```json
{
  "runId": "run_agent_course_317",
  "requiredEvidence": [
    { "action": "file.write", "subject": "lessons/317-evidence-index-audit-query.md" },
    { "action": "file.update", "subject": "README.md" },
    { "action": "file.update", "subject": "TOOLS.md" },
    { "action": "message.send", "subject": "telegram:-5115329245" },
    { "action": "git.commit", "subject": "repo:gfwfail/agent-course" },
    { "action": "git.push", "subject": "repo:gfwfail/agent-course" }
  ]
}
```

完成闸门逻辑：

```python
def verify_completion(index: EvidenceIndex, run_id: str, required: list[dict]) -> bool:
    rows = index.query(run_id=run_id, limit=200)
    seen = {(r["action"], r["subject"]): r for r in rows if r["status"] == "ok"}

    missing = []
    for item in required:
        key = (item["action"], item["subject"])
        if key not in seen:
            missing.append(item)

    if missing:
        raise RuntimeError(f"missing required evidence: {missing}")

    return True
```

这就是从“任务做完了”升级成“任务完成条件有证据支持”。

---

## 7. 常见坑

**坑 1：索引里塞原文**

索引是为了查询，不是为了绕过 Evidence Store。敏感正文、完整工具输出、用户私密内容不要进索引。

**坑 2：只有 evidenceId，没有关联 ID**

没有 `runId / intentId / policyDecisionId`，事故复盘时证据就串不起来。

**坑 3：查询不审计**

看证据也是一种访问行为，尤其是安全、用户隐私、审批记录，必须记录谁因为什么查了什么。

**坑 4：删除 raw 后不更新索引状态**

Evidence Sweeper 删除 raw evidence 后，要把索引状态改成 `tombstoned` 或 `expired`，否则查询结果会指向不存在的 artifact。

---

## 8. 最小落地清单

如果你现在要给 Agent 加 Evidence Index，先做这 6 个字段就够：

- `evidenceId`
- `runId`
- `action`
- `subject`
- `status`
- `contentHash`

然后逐步补：

- `intentId`
- `policyDecisionId`
- `sensitivity`
- `retentionState`
- `tags`
- `artifactRef`

不要一开始就追求大而全。先让关键副作用能被查、能被验证、能被复盘。

---

## 9. 总结

今天的核心：**证据系统不能只会写，还要能安全地查。**

成熟 Agent 的 Evidence Index 应该做到：

1. raw evidence 和 query index 分离
2. 索引只存 metadata、hash、关联关系，不存敏感原文
3. 每条证据绑定 runId / intentId / policyDecisionId
4. 查询按 purpose 最小披露
5. 查询本身进入审计日志
6. 完成闸门通过索引验证 required evidence

一句话收尾：

> Evidence Store 让 Agent 留下证据；Evidence Index 让 Agent 能在正确权限下找回证据。

下一步可以继续讲：**Evidence Chain Verification** —— 怎么验证一整条证据链没有断、没有缺、没有被篡改。
