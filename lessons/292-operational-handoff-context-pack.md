# 292. Agent 值班交接与运行上下文包（Operational Handoff & On-call Context Pack）

上一课讲了 **Operational Readiness Review（ORR）**：上线前要证明监控、回滚、Kill Switch、幂等和权限都准备好了。

但 Agent 真正进入生产后，还有一个很容易被忽视的问题：**交接**。

人类工程师换班时会写 handoff：当前系统状态、刚发生过什么、有哪些风险、下一步要盯什么。Always-on Agent 也一样。尤其是 cron、heartbeat、sub-agent、长任务并行运行时，如果没有结构化交接，下一个 run 很容易“醒来失忆”：重复执行、误判旧告警、漏掉等待中的人工审批。

今天这课讲：**Operational Handoff & On-call Context Pack（值班交接与运行上下文包）**。

> 重点：成熟 Agent 不只是会做任务，还要能把“我现在处于什么运行状态”交给下一个自己或下一个值班者。

---

## 1. 为什么 Agent 需要 Handoff？

Agent 的运行状态经常横跨多个 turn / process / session：

1. **长任务会中断**：部署、数据同步、PR review、批处理都可能跨小时。
2. **多个入口会并发**：Telegram、Cron、Webhook、GitHub event 可能同时触发。
3. **外部世界会变化**：PR 被合并、审批过期、缓存失效、环境漂移。
4. **人类需要接管**：高风险操作、事故处理、异常账单都需要清楚知道目前卡在哪。
5. **Agent 自己会重启**：上下文窗口清空后，必须从文件/DB 恢复关键状态。

没有 handoff 的常见事故：

- 上一个 run 已经发过消息，下一个 run 又发一遍。
- 明明 Kill Switch 已开启，新 run 不知道，还继续执行副作用。
- PR 已 merge，但 Agent 继续往旧分支 push。
- 事故处理中断后，下一次 heartbeat 只看到“没异常”，漏掉未完成的恢复步骤。

---

## 2. 核心设计：Handoff Pack = 状态摘要 + 待办 + 风险 + 证据

一个可用的 `HandoffPack` 至少包含：

```ts
type HandoffPack = {
  id: string;
  agentId: string;
  createdAt: string;
  expiresAt?: string;
  status: 'normal' | 'watching' | 'degraded' | 'incident' | 'paused';
  summary: string;
  activeWork: ActiveWorkItem[];
  blockers: Blocker[];
  risks: RiskNote[];
  nextActions: NextAction[];
  evidence: EvidenceRef[];
  controlState: {
    killSwitch?: 'off' | 'degraded' | 'freeze' | 'locked';
    autonomyLevel: 'observe' | 'dry_run' | 'canary' | 'limited' | 'full';
  };
};
```

建议把 handoff 当作 **运行时契约**，而不是普通日志：

- `summary` 给 LLM 快速恢复上下文。
- `activeWork` 记录正在推进的任务和 owner。
- `blockers` 记录需要人类或外部系统解除的阻塞。
- `risks` 记录“下次动手前必须复核”的风险。
- `nextActions` 记录下一步可执行动作，避免 Agent 醒来重新规划跑偏。
- `evidence` 链接 messageId、commit、PR、日志、dashboard、drill report。
- `controlState` 把当前自主等级和刹车状态显式写入。

---

## 3. learn-claude-code：最小 Python Handoff Store

教学版用 JSON 文件实现，适合本地 Agent、cron 和长任务恢复。

```python
# learn_claude_code/handoff_pack.py
from __future__ import annotations

from dataclasses import dataclass, asdict
from datetime import datetime, timezone
from pathlib import Path
import json
import uuid

@dataclass
class NextAction:
    action: str
    risk: str = 'low'
    requires_revalidation: bool = True

@dataclass
class EvidenceRef:
    kind: str
    ref: str
    observed_at: str

@dataclass
class HandoffPack:
    id: str
    agent_id: str
    created_at: str
    status: str
    summary: str
    active_work: list[str]
    blockers: list[str]
    risks: list[str]
    next_actions: list[NextAction]
    evidence: list[EvidenceRef]
    control_state: dict

class HandoffStore:
    def __init__(self, path: Path):
        self.path = path
        self.path.parent.mkdir(parents=True, exist_ok=True)

    def write(self, pack: HandoffPack) -> None:
        tmp = self.path.with_suffix('.tmp')
        tmp.write_text(json.dumps(asdict(pack), ensure_ascii=False, indent=2), encoding='utf-8')
        tmp.replace(self.path)  # 原子替换，避免写一半崩溃

    def read(self) -> HandoffPack | None:
        if not self.path.exists():
            return None
        raw = json.loads(self.path.read_text(encoding='utf-8'))
        raw['next_actions'] = [NextAction(**x) for x in raw.get('next_actions', [])]
        raw['evidence'] = [EvidenceRef(**x) for x in raw.get('evidence', [])]
        return HandoffPack(**raw)


def now() -> str:
    return datetime.now(timezone.utc).isoformat()


if __name__ == '__main__':
    store = HandoffStore(Path('memory/handoff/agent-course.json'))
    pack = HandoffPack(
        id=str(uuid.uuid4()),
        agent_id='agent-course-cron',
        created_at=now(),
        status='normal',
        summary='第292课已生成，等待发送 Telegram、git commit 和 push。',
        active_work=['agent-course lesson 292'],
        blockers=[],
        risks=['push 前必须确认 gh 当前账号和远端 main 最新'],
        next_actions=[
            NextAction('send telegram lesson summary', risk='medium'),
            NextAction('git diff --check && commit && push', risk='medium'),
        ],
        evidence=[EvidenceRef('lesson_file', 'lessons/292-operational-handoff-context-pack.md', now())],
        control_state={'killSwitch': 'off', 'autonomyLevel': 'limited'},
    )
    store.write(pack)

    restored = store.read()
    print(restored.summary if restored else 'no handoff')
```

几个关键点：

- 写文件用 `tmp.replace()`，保证崩溃时不会留下半截 JSON。
- `next_actions` 标出风险和是否需要重新验证，避免下个 run 直接信旧观察。
- `evidence` 不塞大段日志，只放可回查的引用。
- `control_state` 必须和任务状态一起恢复，不能只恢复业务待办。

---

## 4. pi-mono：生产版 Handoff Middleware

生产里 handoff 不应该靠每个业务 Agent 自觉写，而应该作为 runtime middleware。

```ts
// pi-mono/runtime/HandoffMiddleware.ts
type HandoffStatus = 'normal' | 'watching' | 'degraded' | 'incident' | 'paused';

type NextAction = {
  action: string;
  risk: 'low' | 'medium' | 'high' | 'critical';
  requiresRevalidation: boolean;
};

type HandoffPack = {
  id: string;
  runId: string;
  agentId: string;
  createdAt: string;
  status: HandoffStatus;
  summary: string;
  activeWork: string[];
  blockers: string[];
  risks: string[];
  nextActions: NextAction[];
  evidence: Array<{ kind: string; ref: string; observedAt: string }>;
  controlState: { killSwitch: string; autonomyLevel: string };
};

type HandoffStore = {
  loadLatest(agentId: string): Promise<HandoffPack | null>;
  save(pack: HandoffPack): Promise<void>;
};

type RuntimeContext = {
  runId: string;
  agentId: string;
  memory: Record<string, unknown>;
  evidence: Array<{ kind: string; ref: string; observedAt: string }>;
  controlState: { killSwitch: string; autonomyLevel: string };
  taskState: {
    activeWork: string[];
    blockers: string[];
    risks: string[];
    nextActions: NextAction[];
  };
};

export class HandoffMiddleware {
  constructor(private readonly store: HandoffStore) {}

  async beforeRun(ctx: RuntimeContext) {
    const latest = await this.store.loadLatest(ctx.agentId);
    if (!latest) return;

    ctx.memory.previousHandoff = {
      summary: latest.summary,
      status: latest.status,
      activeWork: latest.activeWork,
      blockers: latest.blockers,
      risks: latest.risks,
      nextActions: latest.nextActions,
      controlState: latest.controlState,
    };
  }

  async afterRun(ctx: RuntimeContext, result: { finalSummary: string; status: HandoffStatus }) {
    const pack: HandoffPack = {
      id: crypto.randomUUID(),
      runId: ctx.runId,
      agentId: ctx.agentId,
      createdAt: new Date().toISOString(),
      status: result.status,
      summary: result.finalSummary,
      activeWork: ctx.taskState.activeWork,
      blockers: ctx.taskState.blockers,
      risks: ctx.taskState.risks,
      nextActions: ctx.taskState.nextActions,
      evidence: ctx.evidence,
      controlState: ctx.controlState,
    };

    await this.store.save(pack);
  }
}
```

实际工程里可以加三条策略：

1. **只注入摘要，不注入原始日志**：避免 context 污染。
2. **过期 handoff 降权使用**：例如超过 24 小时只能作为参考，副作用前必须 live revalidate。
3. **incident handoff 高优先级注入**：事故状态不能被普通任务摘要覆盖。

---

## 5. OpenClaw 实战：用文件交接让 cron / heartbeat 不失忆

OpenClaw 已经天然适合做轻量 handoff：

- `memory/YYYY-MM-DD.md`：事件流水账。
- `MEMORY.md`：长期记忆。
- `HEARTBEAT.md`：周期性职责。
- `session_status`：当前模型、运行环境、时间和配置。
- `git status / messageId / commit sha`：外部副作用证据。

可以约定一个小文件：

```json
// memory/handoff/agent-course.json
{
  "agentId": "agent-course-cron",
  "status": "normal",
  "summary": "上一课已发布并推送，下一课应从 292 开始。",
  "lastLesson": 291,
  "lastTelegramMessageId": 11667,
  "lastCommit": "f762c29",
  "nextActions": [
    {
      "action": "生成 lessons/292-*.md 并更新 README/TOOLS",
      "risk": "medium",
      "requiresRevalidation": true
    }
  ],
  "risks": [
    "git push 前必须 gh pr list 并 pull --rebase 最新 main",
    "避免重复 TOOLS.md 已讲内容"
  ]
}
```

下一次 cron 醒来时，不需要重新从 300 行 README 里猜状态，可以：

1. 读 `memory/handoff/agent-course.json`。
2. 再用 `git status` / `README.md` / `TOOLS.md` live check 验证。
3. 生成新课。
4. 发送 Telegram 后记录 messageId。
5. commit/push 后更新 handoff。

这就是 **handoff + revalidation** 的组合：交接帮你恢复方向，复核帮你避免相信过期世界。

---

## 6. Handoff 的完成闸门

每次写 handoff 前，可以跑一个小 checklist：

- [ ] 有没有当前状态 `status`？
- [ ] 有没有一句人能看懂的 `summary`？
- [ ] 有没有未完成工作 `activeWork`？
- [ ] 有没有阻塞项 `blockers`？
- [ ] 有没有下一步动作 `nextActions`？
- [ ] 有没有副作用证据 `evidence`？
- [ ] 有没有控制状态 `killSwitch/autonomyLevel`？
- [ ] 是否标记了哪些观察下次必须重新验证？

如果任务涉及外部写操作，handoff 至少要记录：

- provider message id / PR URL / commit sha / deployment id
- idempotency key 或 dedupe key
- 最后一次 live check 时间
- 下一次继续前必须检查什么

---

## 7. 常见坑

**坑 1：把 handoff 写成流水账**

日志越长，越没人看，LLM 也抓不住重点。handoff 应该是结构化摘要，不是完整 transcript。

**坑 2：handoff 里没有证据引用**

“已发送消息”不够，必须写 messageId；“已推送代码”不够，必须写 commit sha。

**坑 3：恢复时直接执行 nextActions**

nextActions 是方向，不是免复核授权。凡是副作用动作，执行前都要重新检查现实世界。

**坑 4：只写业务状态，不写控制状态**

如果 Kill Switch / pause / autonomy level 没被交接，Agent 重启后可能绕过上一轮安全刹车。

---

## 8. 小结

今天的关键点：

- Handoff Pack 是 Agent 的值班交接单。
- 它应该包含状态摘要、待办、阻塞、风险、下一步、证据和控制状态。
- learn-claude-code 可以用 JSON 文件 + 原子写实现最小版本。
- pi-mono 里最好做成 Runtime Middleware，自动在 run 前注入、run 后保存。
- OpenClaw 可以用 `memory/handoff/*.json` 把 cron、heartbeat、长任务串起来。
- Handoff 负责恢复方向，Revalidation 负责确认现实。

一句话：**会交接的 Agent，才适合长期值班。**
