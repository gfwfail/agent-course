# 300. Agent 运维知识检索与执行前上下文注入（Operational Knowledge Retrieval & Pre-Execution Context Injection）

上一课讲了运维知识的生命周期：知识会过期、会降权、会归档。
今天讲更关键的一步：**Agent 真正执行生产操作前，如何自动检索相关运维知识，并把它注入到决策上下文里**。

一句话：

> Runbook 不是给人看的文档，而应该变成 Agent 执行前的自动安全上下文。

如果 Agent 要重启服务、清理队列、回滚部署、处理账单异常，它不应该只靠当前对话和模型记忆，而要先问：

- 这类操作过去出过什么事故？
- 有没有相关 Runbook？
- 有没有刚更新的策略或禁令？
- 有没有类似服务的特殊坑？
- 这些知识是否仍然 active、未过期、适用于当前环境？

---

## 1. 问题：有知识，但执行时没想起来

很多团队已经有：

- 事故复盘文档
- Runbook
- SRE Wiki
- GitHub issue
- Slack 历史讨论
- 老工程师脑内经验

但 Agent 执行时经常出现两个问题：

1. **检索缺失**：知识存在，但没有被拉进上下文。
2. **上下文污染**：拉了一堆无关文档，LLM 反而分不清主次。

所以我们要做的不是简单 RAG，而是一个面向生产执行的 **Ops Knowledge Retrieval Gate**：

```text
User Request / Cron Task
        ↓
Intent + Risk Classification
        ↓
Retrieve relevant operational knowledge
        ↓
Filter by freshness / service / environment / operation type
        ↓
Inject compact context pack
        ↓
Plan / Dry-run / Approval / Execute
```

核心原则：**只给 Agent 当前操作真正需要、且仍然可信的知识。**

---

## 2. 知识条目结构：可检索，也可执行前判断

一个运维知识条目不能只是 Markdown。它至少需要携带元数据：

```json
{
  "id": "kb-mysql-restart-2026-05",
  "title": "MySQL 重启前必须检查复制延迟",
  "service": "mysql",
  "env": ["prod", "staging"],
  "operationTypes": ["restart", "failover", "maintenance"],
  "risk": "high",
  "status": "active",
  "reviewAfter": "2026-06-01",
  "expiresAt": "2026-08-01",
  "tags": ["database", "replication", "incident-2026-04-18"],
  "content": "重启 primary 前必须检查 replica lag < 5s，并确认最近 10 分钟无写入峰值。"
}
```

关键字段：

- `service`：适用哪个系统
- `env`：适用环境
- `operationTypes`：适用操作类型
- `risk`：是否需要强注入/审批
- `status`：只允许 active 进入执行上下文
- `reviewAfter` / `expiresAt`：避免旧知识污染决策

这和普通 RAG 最大区别是：**检索结果不是“看起来相关”就能用，还要过执行安全过滤。**

---

## 3. learn-claude-code：最小 Python 教学版

教学版先用本地 JSON 文件实现，不依赖向量数据库。

```python
# learn-claude-code/ops_knowledge.py
from dataclasses import dataclass
from datetime import date
from typing import Literal

Status = Literal["draft", "active", "stale", "superseded", "archived"]

@dataclass
class KnowledgeEntry:
    id: str
    title: str
    service: str
    env: list[str]
    operation_types: list[str]
    risk: str
    status: Status
    review_after: date | None
    expires_at: date | None
    tags: list[str]
    content: str

@dataclass
class ExecutionIntent:
    service: str
    env: str
    operation_type: str
    query: str
    side_effect: bool


def is_fresh(entry: KnowledgeEntry, today: date) -> bool:
    if entry.status != "active":
        return False
    if entry.expires_at and entry.expires_at < today:
        return False
    return True


def score(entry: KnowledgeEntry, intent: ExecutionIntent) -> int:
    s = 0
    if entry.service == intent.service:
        s += 5
    if intent.env in entry.env:
        s += 3
    if intent.operation_type in entry.operation_types:
        s += 4
    for tag in entry.tags:
        if tag.lower() in intent.query.lower():
            s += 1
    if entry.risk in ("high", "critical") and intent.side_effect:
        s += 2
    return s


def retrieve_ops_context(entries: list[KnowledgeEntry], intent: ExecutionIntent, limit: int = 5) -> list[KnowledgeEntry]:
    today = date.today()
    candidates = [e for e in entries if is_fresh(e, today)]
    ranked = sorted(candidates, key=lambda e: score(e, intent), reverse=True)
    return [e for e in ranked if score(e, intent) > 0][:limit]


def build_context_pack(entries: list[KnowledgeEntry]) -> str:
    if not entries:
        return "No active operational knowledge matched this execution."

    lines = ["<operational_knowledge>"]
    for e in entries:
        lines.append(f"- [{e.risk}] {e.title} ({e.id})")
        lines.append(f"  {e.content}")
    lines.append("</operational_knowledge>")
    return "\n".join(lines)
```

Agent Loop 中使用：

```python
intent = ExecutionIntent(
    service="mysql",
    env="prod",
    operation_type="restart",
    query=user_request,
    side_effect=True,
)

entries = retrieve_ops_context(kb_entries, intent)
ops_context = build_context_pack(entries)

system_prompt = f"""
You are an operational agent.
Before planning or executing, obey the following active operational knowledge:

{ops_context}

If knowledge conflicts with the user request, stop and ask for confirmation.
"""
```

这个版本虽然简单，但已经有 3 个生产级思想：

1. 先识别意图，再检索知识。
2. 只注入 active 且未过期的条目。
3. 知识以明确 XML block 注入，降低被普通上下文淹没的概率。

---

## 4. pi-mono：生产版中间件设计

在 pi-mono 里，不应该让每个工具自己去查知识。更好的方式是做成 Agent Loop 的中间件：

```ts
// pi-mono/packages/agent/src/middleware/OpsKnowledgeMiddleware.ts
export type OperationType =
  | 'deploy'
  | 'rollback'
  | 'restart'
  | 'scale'
  | 'data_fix'
  | 'billing_adjustment'
  | 'incident_response';

export interface ExecutionIntent {
  service: string;
  environment: 'dev' | 'staging' | 'prod';
  operationType: OperationType;
  sideEffect: boolean;
  userRequest: string;
}

export interface OpsKnowledgeEntry {
  id: string;
  title: string;
  service: string;
  environments: string[];
  operationTypes: OperationType[];
  risk: 'low' | 'medium' | 'high' | 'critical';
  status: 'draft' | 'active' | 'stale' | 'superseded' | 'archived';
  expiresAt?: string;
  content: string;
}

export interface OpsKnowledgeStore {
  search(intent: ExecutionIntent, limit: number): Promise<OpsKnowledgeEntry[]>;
}

export class OpsKnowledgeMiddleware {
  constructor(private readonly store: OpsKnowledgeStore) {}

  async beforePlan(ctx: AgentContext): Promise<AgentContext> {
    const intent = await this.classifyIntent(ctx);

    if (!intent.sideEffect) return ctx;

    const entries = await this.store.search(intent, 8);
    const usable = entries.filter((entry) => this.isUsable(entry, intent));

    return ctx.withSystemContext({
      key: 'ops_knowledge',
      priority: 'critical',
      content: this.renderContextPack(usable),
      metadata: {
        count: usable.length,
        ids: usable.map((e) => e.id),
      },
    });
  }

  private isUsable(entry: OpsKnowledgeEntry, intent: ExecutionIntent): boolean {
    if (entry.status !== 'active') return false;
    if (!entry.environments.includes(intent.environment)) return false;
    if (!entry.operationTypes.includes(intent.operationType)) return false;
    if (entry.expiresAt && new Date(entry.expiresAt) < new Date()) return false;
    return true;
  }

  private renderContextPack(entries: OpsKnowledgeEntry[]): string {
    if (entries.length === 0) {
      return '<ops_knowledge>No active matched ops knowledge.</ops_knowledge>';
    }

    return [
      '<ops_knowledge priority="critical">',
      ...entries.map((e) => [
        `<entry id="${e.id}" risk="${e.risk}">`,
        `<title>${e.title}</title>`,
        `<content>${e.content}</content>`,
        '</entry>',
      ].join('\n')),
      '</ops_knowledge>',
    ].join('\n');
  }

  private async classifyIntent(ctx: AgentContext): Promise<ExecutionIntent> {
    // 生产版可用轻量模型分类，也可先用规则兜底：
    // - tool metadata 推断 service / operationType
    // - user/session env 推断 environment
    // - write/external/delete/deploy 标记 sideEffect
    return ctx.intent as ExecutionIntent;
  }
}
```

注意 `priority: 'critical'`：这类上下文应该比普通检索资料更靠前、更稳定。

生产实现建议：

- 用 BM25 + embedding 混合检索，避免只靠向量相似度。
- `risk=critical` 条目即使相关性分不高，也要强制进入上下文。
- 知识条目 ID 写进审计日志，方便复盘“当时 Agent 看到了什么”。
- 如果高风险操作没有匹配到任何知识，可以降级为 dry-run 或要求人工确认。

---

## 5. OpenClaw 实战：文件式 ops-kb + 执行前闸门

OpenClaw 很适合用文件系统做轻量运维知识库：

```text
.openclaw/workspace/ops-kb/
  index.json
  mysql-restart-replica-lag.md
  forge-deploy-rollback.md
  telegram-rate-limit.md
```

`index.json` 保存元数据：

```json
[
  {
    "id": "forge-deploy-rollback",
    "file": "forge-deploy-rollback.md",
    "service": "forge",
    "env": ["prod"],
    "operationTypes": ["deploy", "rollback"],
    "risk": "high",
    "status": "active",
    "expiresAt": "2026-08-01"
  }
]
```

Heartbeat / Cron / 手动任务执行前可以先跑一个小脚本生成上下文包：

```bash
python scripts/build_ops_context.py \
  --service forge \
  --env prod \
  --operation deploy \
  --out /tmp/ops-context.md
```

然后 Agent 在执行前读取 `/tmp/ops-context.md`，把它当作硬约束。

更进一步，可以把执行证据也写下来：

```json
{
  "runId": "cron-2026-05-12T05:30:00Z",
  "operation": "deploy",
  "service": "forge",
  "knowledgeInjected": [
    "forge-deploy-rollback",
    "incident-2026-04-forge-env-drift"
  ],
  "decision": "dry_run_then_execute",
  "approvedBy": "human"
}
```

这样事故复盘时就能回答：

> Agent 当时到底有没有看到那条 Runbook？

如果没看到，是检索问题；如果看到了还做错，是策略/执行器问题。

---

## 6. 三个容易踩的坑

### 坑 1：只按语义相似度检索

“重启 Redis”可能检索到一堆 Redis 性能优化文章，但漏掉“生产 Redis 禁止白天重启”的关键禁令。

解决：元数据过滤必须先于语义排序，`service/env/operationType/status` 是硬条件。

### 坑 2：把过期知识也注入

旧 Runbook 比没有 Runbook 更危险，因为它会给 Agent 错误确定性。

解决：`expiresAt` 过期直接禁用；`reviewAfter` 到期则降权或要求人工确认。

### 坑 3：上下文包太长

把 20 篇 Wiki 全塞进去，LLM 只会更迷糊。

解决：注入摘要 + 禁令 + checklist，而不是全文；全文只作为可按需读取的引用。

---

## 7. 实战检查清单

做执行前运维知识注入时，至少检查：

- [ ] 是否先识别了 service / env / operationType / sideEffect？
- [ ] 是否过滤掉 draft/stale/archived/expired 知识？
- [ ] 是否把 critical 禁令强制注入？
- [ ] 是否限制上下文包长度？
- [ ] 是否把 injected knowledge IDs 写入审计日志？
- [ ] 高风险操作无知识命中时，是否降级为 dry-run 或人工确认？

---

## 总结

运维知识检索不是普通 RAG，它是生产执行链路的一道安全闸门。

成熟 Agent 的执行流程应该是：

```text
先理解任务 → 再找相关运维知识 → 过滤过期和无关内容 → 注入关键约束 → 再计划和执行
```

记住一句话：

> 知识库的价值，不在于它存在，而在于关键操作发生前，Agent 能不能自动想起正确的那几条。
