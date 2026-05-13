# 314. Agent 策略证据要求与带证明行动（Policy Evidence Requirements & Proof-Carrying Actions）

上一课我们讲了 **Policy Obligation Queue**：策略允许动作之后，后置义务要可靠履行。

今天再往前补一层：**Policy 不应该只说 allow/deny，还应该声明“这个动作要带什么证据才能执行和完成”**。

这叫 **Proof-Carrying Action**：每个高风险动作都必须携带可验证证据包，否则工具层拒绝执行。

---

## 1. 问题：Agent 经常“看起来检查过了”，但证明不了

典型事故链路：

1. Agent 说“我检查过可以推送”
2. 实际只看了一眼 `git status`
3. 没跑测试、没确认目标 remote、没记录 diff
4. 出问题后无法回答：当时凭什么觉得可以？

成熟 Agent 不能靠一句自然语言承诺。

高风险动作必须带结构化证明：

```json
{
  "action": "git.push",
  "intentId": "intent_20260514_course",
  "requiredEvidence": ["clean_worktree_before", "diff_summary", "commit_sha", "remote_url"],
  "evidence": {
    "clean_worktree_before": { "ok": true, "source": "git status --porcelain" },
    "diff_summary": { "files": ["README.md", "lessons/314-...md"] },
    "commit_sha": { "value": "abc123" },
    "remote_url": { "value": "git@github.com:gfwfail/agent-course.git" }
  }
}
```

Policy 决定：**这个动作至少要哪些 proof**。
Tool Dispatcher 决定：**proof 不完整就不执行**。

---

## 2. 核心设计：EvidenceRequirement

把证据要求从 prompt 移到代码里：

```ts
type EvidenceRequirement = {
  id: string;
  required: boolean;
  freshnessMs?: number;
  validator: (evidence: EvidenceItem, ctx: ActionContext) => boolean;
};

type PolicyDecision = {
  action: "allow" | "deny" | "require_approval" | "dry_run";
  reasonCodes: string[];
  evidenceRequirements: EvidenceRequirement[];
};
```

不同动作的证据要求不同：

| 动作 | 必需证据 |
|---|---|
| 发 Telegram 群消息 | target chat、message preview、dedupe key |
| git push | clean worktree、commit sha、remote url、push result |
| 部署生产 | diff、测试结果、审批、回滚方案、健康检查 |
| 删除数据 | owner intent、scope、dry-run result、backup/tombstone |

**重点：Evidence 是工具层事实，不是 LLM 自己说“我觉得”。**

---

## 3. learn-claude-code：Python 教学版

最小可用实现：策略返回 evidence requirements，执行器执行前检查。

```python
from dataclasses import dataclass
from time import time

@dataclass
class EvidenceItem:
    id: str
    ok: bool
    observed_at: float
    value: dict
    source: str

@dataclass
class Requirement:
    id: str
    required: bool = True
    max_age_sec: int | None = None

class EvidenceBag:
    def __init__(self):
        self.items: dict[str, EvidenceItem] = {}

    def add(self, item: EvidenceItem):
        self.items[item.id] = item

    def check(self, req: Requirement) -> tuple[bool, str]:
        item = self.items.get(req.id)
        if not item:
            return (not req.required, f"missing evidence: {req.id}")
        if not item.ok:
            return False, f"evidence failed: {req.id}"
        if req.max_age_sec and time() - item.observed_at > req.max_age_sec:
            return False, f"stale evidence: {req.id}"
        return True, "ok"


def policy_for(action: str) -> list[Requirement]:
    if action == "git.push":
        return [
            Requirement("clean_worktree_before", max_age_sec=60),
            Requirement("commit_sha", max_age_sec=300),
            Requirement("remote_url", max_age_sec=300),
        ]
    if action == "message.send.group":
        return [
            Requirement("target_chat_verified", max_age_sec=300),
            Requirement("message_preview", max_age_sec=300),
            Requirement("dedupe_key", max_age_sec=3600),
        ]
    return []


def execute_action(action: str, evidence: EvidenceBag, fn):
    failures = []
    for req in policy_for(action):
        ok, reason = evidence.check(req)
        if not ok:
            failures.append(reason)

    if failures:
        return {"status": "blocked", "reasons": failures}

    result = fn()
    evidence.add(EvidenceItem(
        id=f"{action}.result",
        ok=result["ok"],
        observed_at=time(),
        value=result,
        source="tool_result",
    ))
    return {"status": "done", "result": result}
```

这段代码解决一个关键问题：

> Agent 不能直接“想执行就执行”，必须先交出动作需要的 proof。

---

## 4. pi-mono：生产版中间件

在生产系统里，Evidence Check 应该是 Tool Middleware，而不是散落在业务代码里。

```ts
export type EvidenceItem = {
  id: string;
  ok: boolean;
  observedAt: string;
  source: "tool" | "policy" | "approval" | "test" | "receipt";
  valueHash?: string;
  value?: unknown;
};

export type ToolCallContext = {
  runId: string;
  actorId: string;
  action: string;
  evidence: Map<string, EvidenceItem>;
};

export class ProofCarryingActionMiddleware {
  constructor(private policy: PolicyEngine, private audit: AuditLog) {}

  async beforeToolCall(ctx: ToolCallContext, tool: ToolDefinition, args: unknown) {
    const decision = await this.policy.decide({
      actorId: ctx.actorId,
      action: ctx.action,
      toolName: tool.name,
      args,
    });

    if (decision.action === "deny") {
      throw new PolicyDeniedError(decision.reasonCodes);
    }

    const missing = decision.evidenceRequirements.filter((req) => {
      const item = ctx.evidence.get(req.id);
      if (!item) return req.required;
      if (!item.ok) return true;
      if (req.freshnessMs) {
        const age = Date.now() - Date.parse(item.observedAt);
        return age > req.freshnessMs;
      }
      return false;
    });

    if (missing.length > 0) {
      await this.audit.append({
        type: "tool.blocked.missing_evidence",
        runId: ctx.runId,
        toolName: tool.name,
        missing: missing.map((m) => m.id),
        reasonCodes: decision.reasonCodes,
      });

      throw new MissingEvidenceError(missing.map((m) => m.id));
    }

    await this.audit.append({
      type: "tool.allowed.with_proof",
      runId: ctx.runId,
      toolName: tool.name,
      evidenceIds: decision.evidenceRequirements.map((r) => r.id),
    });
  }
}
```

这个中间件有三个好处：

1. **LLM 不需要记住所有规则**：策略引擎统一返回 proof requirements
2. **工具调用天然可审计**：每次 allow 都知道依赖哪些 evidence
3. **高风险动作可复盘**：出事后能还原当时证据链

---

## 5. OpenClaw 实战：课程 Cron 就应该这样跑

以这个课程自动发布任务为例，高风险点有两个：

- 给 Telegram 群发消息
- git commit / push 到 GitHub

推荐 Evidence Bundle：

```json
{
  "intent": "publish_agent_course_lesson",
  "lessonFile": "lessons/314-policy-evidence-requirements-proof-carrying-actions.md",
  "evidence": [
    "topic_not_duplicate",
    "lesson_file_written",
    "readme_updated",
    "tools_updated",
    "telegram_message_id",
    "git_commit_sha",
    "git_push_success"
  ]
}
```

执行顺序：

1. 检查 `TOOLS.md` 已讲列表，生成 `topic_not_duplicate`
2. 写 lesson 文件，生成 `lesson_file_written`
3. 更新 README，生成 `readme_updated`
4. 发群消息，记录 provider message id
5. commit 后记录 commit sha
6. push 后记录 remote 结果
7. final reply 前检查 evidence bundle 完整

这就是 **Completion Gates + Policy Evidence Requirements** 的组合拳。

---

## 6. 常见坑

### 坑 1：让 LLM 自己声明证据

错误做法：

```json
{ "checked_tests": "yes" }
```

正确做法：

```json
{
  "id": "tests_passed",
  "ok": true,
  "source": "exec:npm test",
  "observedAt": "2026-05-14T09:30:00+11:00",
  "valueHash": "sha256:..."
}
```

证据必须来自工具结果、审批回执、外部系统 receipt，而不是模型嘴巴。

### 坑 2：证据没有新鲜度

“昨天测试通过”不能证明“现在可以部署”。

给 requirement 加 `freshnessMs`，副作用前强制复核。

### 坑 3：只检查执行前，不记录执行后

Proof-Carrying Action 应该有两类 proof：

- **pre-proof**：执行前为什么允许做
- **post-proof**：执行后怎么证明做成了

例如 git push：

- pre-proof：commit sha、remote url、clean worktree
- post-proof：push exit code、remote branch head、GitHub commit URL

---

## 7. 一句话总结

**成熟 Agent 的动作不是“我觉得可以”，而是“我带着证据执行，并留下证据证明执行结果”。**

Policy Evidence Requirements 把安全从 prompt 约束升级成工具层契约；Proof-Carrying Actions 则让每一次副作用都有凭有据、可阻断、可复盘。