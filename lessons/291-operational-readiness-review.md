# 291. Agent 运行就绪评审（Operational Readiness Review, ORR）

前两课我们讲了 **Recovery Drill / Game Day** 和 **Resilience Scorecard**。它们回答的是：系统坏了以后，恢复路径有没有演练过、覆盖率够不够。

今天补上发布前最后一道很实用的门：**Operational Readiness Review（ORR，运行就绪评审）**。

ORR 的目标不是写一堆形式主义 checklist，而是把“这个 Agent 能不能交给生产环境自动跑”变成一组可执行证据：监控有没有、回滚有没有、权限有没有收紧、失败路径有没有演练、on-call/runbook 有没有准备好。

> 重点：代码能跑，只代表功能 ready；ORR 通过，才代表运行 ready。

---

## 1. 为什么 Agent 更需要 ORR？

普通后端服务发布失败，通常是接口 500、延迟升高、任务积压。Agent 的风险更复杂：

1. **会调用工具**：发消息、提交代码、改配置、部署、查数据库，副作用范围比普通 API 大。
2. **行为会漂移**：prompt、模型、工具 schema、memory、外部网页变化，都会改变运行结果。
3. **故障不总是显性**：Agent 可能“成功地做错事”，HTTP 200 也不代表正确。
4. **恢复路径跨系统**：取消、补偿、回滚、人工接管、重复投递处理都要提前准备。
5. **自主等级不同**：observe-only、dry-run、canary、limited、full autonomy 的就绪要求不应该一样。

所以发布前要问的不是一句“测试过了吗”，而是一组明确问题：

- 这个 Agent 最坏能影响什么？
- 出错时谁能停它？
- 能不能证明不会重复执行副作用？
- 关键指标有没有 SLO 和告警？
- 回滚/暂停/恢复有没有实际演练证据？
- 权限是不是最小化，而不是用了万能 token？

---

## 2. 核心设计：Readiness Requirement + Evidence Gate

ORR 可以建模成一组 `ReadinessRequirement`。每条要求都绑定：

- `category`：observability / safety / recovery / security / operations
- `risk`：low / medium / high / critical
- `requiredFor`：哪些自主等级必须满足
- `evidence`：需要哪些证据 key
- `maxAgeDays`：证据多久过期

```ts
type AutonomyLevel = 'observe' | 'dry_run' | 'canary' | 'limited' | 'full';

type ReadinessRequirement = {
  id: string;
  category: 'observability' | 'safety' | 'recovery' | 'security' | 'operations';
  risk: 'low' | 'medium' | 'high' | 'critical';
  requiredFor: AutonomyLevel[];
  evidenceKeys: string[];
  maxAgeDays: number;
};

type EvidenceItem = {
  key: string;
  value: string;
  collectedAt: string;
  source: 'test' | 'drill' | 'config' | 'manual_review' | 'monitoring';
};
```

ORR Gate 的输出不要只给 boolean，应该给结构化结果：

```ts
type ReadinessDecision = {
  decision: 'pass' | 'pass_with_conditions' | 'block';
  missing: string[];
  expired: string[];
  conditions: string[];
  allowedAutonomy: AutonomyLevel;
};
```

---

## 3. learn-claude-code：最小 Python ORR Gate

教学版先用内存列表实现，适合放在 release script 或 CI 里跑。

```python
# learn_claude_code/operational_readiness.py
from dataclasses import dataclass
from datetime import datetime, timezone, timedelta
from typing import Literal

Autonomy = Literal['observe', 'dry_run', 'canary', 'limited', 'full']
Decision = Literal['pass', 'pass_with_conditions', 'block']

AUTONOMY_ORDER = ['observe', 'dry_run', 'canary', 'limited', 'full']

@dataclass(frozen=True)
class Requirement:
    id: str
    category: str
    risk: str
    required_for: tuple[Autonomy, ...]
    evidence_keys: tuple[str, ...]
    max_age_days: int

@dataclass(frozen=True)
class Evidence:
    key: str
    value: str
    collected_at: datetime
    source: str

DEFAULT_REQUIREMENTS = [
    Requirement(
        id='metrics_and_alerts',
        category='observability',
        risk='high',
        required_for=('canary', 'limited', 'full'),
        evidence_keys=('slo_defined', 'alerts_configured', 'dashboard_url'),
        max_age_days=30,
    ),
    Requirement(
        id='kill_switch',
        category='safety',
        risk='critical',
        required_for=('limited', 'full'),
        evidence_keys=('kill_switch_tested', 'operator_can_pause'),
        max_age_days=14,
    ),
    Requirement(
        id='rollback_or_compensation',
        category='recovery',
        risk='critical',
        required_for=('canary', 'limited', 'full'),
        evidence_keys=('rollback_runbook', 'rollback_drill_passed'),
        max_age_days=30,
    ),
    Requirement(
        id='least_privilege',
        category='security',
        risk='high',
        required_for=('dry_run', 'canary', 'limited', 'full'),
        evidence_keys=('scopes_reviewed', 'secret_refs_only'),
        max_age_days=30,
    ),
    Requirement(
        id='idempotency',
        category='operations',
        risk='high',
        required_for=('canary', 'limited', 'full'),
        evidence_keys=('idempotency_keys', 'duplicate_delivery_tested'),
        max_age_days=30,
    ),
]

class ORRGate:
    def __init__(self, requirements: list[Requirement], now: datetime):
        self.requirements = requirements
        self.now = now

    def evaluate(self, target: Autonomy, evidence: list[Evidence]) -> dict:
        by_key = {item.key: item for item in evidence}
        missing = []
        expired = []
        conditions = []

        active_requirements = [
            req for req in self.requirements
            if target in req.required_for
        ]

        for req in active_requirements:
            for key in req.evidence_keys:
                item = by_key.get(key)
                if item is None:
                    missing.append(f'{req.id}:{key}')
                    continue

                age = self.now - item.collected_at
                if age > timedelta(days=req.max_age_days):
                    expired.append(f'{req.id}:{key}')

        if missing:
            decision: Decision = 'block'
            allowed = self._downgrade(target)
        elif expired:
            decision = 'pass_with_conditions'
            allowed = target
            conditions.append('refresh_expired_evidence_before_full_autonomy')
        else:
            decision = 'pass'
            allowed = target

        return {
            'decision': decision,
            'targetAutonomy': target,
            'allowedAutonomy': allowed,
            'missing': missing,
            'expired': expired,
            'conditions': conditions,
        }

    def _downgrade(self, target: Autonomy) -> Autonomy:
        idx = AUTONOMY_ORDER.index(target)
        return AUTONOMY_ORDER[max(0, idx - 1)]  # type: ignore[return-value]


if __name__ == '__main__':
    now = datetime.now(timezone.utc)
    evidence = [
        Evidence('slo_defined', 'agent_turn_success_rate >= 99%', now, 'manual_review'),
        Evidence('alerts_configured', 'pager:agent-prod', now, 'monitoring'),
        Evidence('dashboard_url', 'https://grafana/agent-prod', now, 'monitoring'),
        Evidence('scopes_reviewed', 'write:message only', now, 'manual_review'),
        Evidence('secret_refs_only', 'no raw secrets in prompt', now, 'config'),
    ]

    result = ORRGate(DEFAULT_REQUIREMENTS, now).evaluate('limited', evidence)
    print(result)
    # 缺少 kill_switch / rollback / idempotency 证据，所以 limited 会被 block，最多降到 canary/dry_run。
```

这个小例子已经体现 ORR 的关键原则：**缺证据不是“提醒一下”，而是自动降低自主等级或阻断发布。**

---

## 4. pi-mono：生产版 Readiness Gate 中间件

生产环境里，ORR 不应该只在发布脚本里跑一次。更好的方式是接到 Agent Runtime / Release Controller 里：

```ts
// pi-mono/readiness/ReadinessGate.ts
type EvidenceStore = {
  getEvidence(agentId: string): Promise<EvidenceItem[]>;
  appendDecision(decision: ReadinessDecision & { agentId: string; runId: string }): Promise<void>;
};

type ReadinessPolicy = {
  requirementsFor(agentId: string): Promise<ReadinessRequirement[]>;
};

const autonomyRank: AutonomyLevel[] = ['observe', 'dry_run', 'canary', 'limited', 'full'];

export class ReadinessGate {
  constructor(
    private readonly policy: ReadinessPolicy,
    private readonly evidence: EvidenceStore,
    private readonly now = () => new Date(),
  ) {}

  async evaluate(input: {
    agentId: string;
    runId: string;
    targetAutonomy: AutonomyLevel;
  }): Promise<ReadinessDecision> {
    const requirements = await this.policy.requirementsFor(input.agentId);
    const items = await this.evidence.getEvidence(input.agentId);
    const byKey = new Map(items.map((item) => [item.key, item]));

    const missing: string[] = [];
    const expired: string[] = [];

    for (const req of requirements) {
      if (!req.requiredFor.includes(input.targetAutonomy)) continue;

      for (const key of req.evidenceKeys) {
        const item = byKey.get(key);
        if (!item) {
          missing.push(`${req.id}:${key}`);
          continue;
        }

        const ageMs = this.now().getTime() - new Date(item.collectedAt).getTime();
        const maxAgeMs = req.maxAgeDays * 24 * 60 * 60 * 1000;
        if (ageMs > maxAgeMs) expired.push(`${req.id}:${key}`);
      }
    }

    const decision: ReadinessDecision = missing.length > 0
      ? {
          decision: 'block',
          missing,
          expired,
          conditions: ['downgrade_autonomy_or_collect_missing_evidence'],
          allowedAutonomy: this.downgrade(input.targetAutonomy),
        }
      : expired.length > 0
        ? {
            decision: 'pass_with_conditions',
            missing,
            expired,
            conditions: ['refresh_expired_evidence_within_24h'],
            allowedAutonomy: input.targetAutonomy,
          }
        : {
            decision: 'pass',
            missing,
            expired,
            conditions: [],
            allowedAutonomy: input.targetAutonomy,
          };

    await this.evidence.appendDecision({ ...decision, agentId: input.agentId, runId: input.runId });
    return decision;
  }

  private downgrade(level: AutonomyLevel): AutonomyLevel {
    const idx = autonomyRank.indexOf(level);
    return autonomyRank[Math.max(0, idx - 1)];
  }
}
```

接入 Agent 执行入口：

```ts
// pi-mono/runtime/startAgentRun.ts
const readiness = await readinessGate.evaluate({
  agentId,
  runId,
  targetAutonomy: requestedAutonomy,
});

if (readiness.decision === 'block') {
  return {
    status: 'blocked',
    reason: 'operational_readiness_failed',
    allowedAutonomy: readiness.allowedAutonomy,
    missing: readiness.missing,
  };
}

const run = await agentRuntime.start({
  agentId,
  runId,
  autonomy: readiness.allowedAutonomy,
});
```

注意这里的设计：

- ORR 不直接修改 Agent 行为细节，只决定 **允许的自主等级**。
- Runtime 根据 `allowedAutonomy` 再过滤工具、是否 dry-run、是否需要审批。
- 每次评审结果写入 EvidenceStore，方便审计和复盘。

---

## 5. OpenClaw 实战：课程 Cron 的 ORR 清单

拿我们这个 Agent 开发课程 cron 举例，它每 3 小时会：写 lesson、更新 README/TOOLS、发 Telegram、git commit、push。

这已经是一个有外部副作用的自动 Agent。它的 ORR 可以这样设计：

```json
{
  "agentId": "agent-course-cron",
  "targetAutonomy": "limited",
  "requirements": [
    {
      "id": "telegram_delivery_receipt",
      "category": "operations",
      "evidenceKeys": ["message_id_recorded"],
      "maxAgeDays": 7
    },
    {
      "id": "git_push_preflight",
      "category": "safety",
      "evidenceKeys": ["gh_account_checked", "pr_state_checked", "pull_rebase_done", "diff_check_passed"],
      "maxAgeDays": 1
    },
    {
      "id": "content_deduplication",
      "category": "quality",
      "evidenceKeys": ["tools_topic_checked", "lesson_number_incremented"],
      "maxAgeDays": 1
    },
    {
      "id": "completion_gate",
      "category": "operations",
      "evidenceKeys": ["lesson_file_exists", "readme_updated", "tools_updated", "git_pushed"],
      "maxAgeDays": 1
    }
  ]
}
```

在 OpenClaw 里可以把 ORR 作为最终完成闸门：

```bash
# 伪命令：真实项目里可以写成 scripts/check-course-orr.ts
python scripts/check_course_orr.py \
  --lesson lessons/291-operational-readiness-review.md \
  --require README.md \
  --require ../TOOLS.md \
  --require-message-id \
  --require-git-clean
```

这个例子很贴近真实自动化：

- `message_id_recorded` 防止“我以为发了”。
- `pull_rebase_done` 防止基于旧 main 推送。
- `diff_check_passed` 防止 markdown 里有明显冲突/空白错误。
- `lesson_number_incremented` 防止重复覆盖上一课。
- `git_clean` 防止改了一半就宣称完成。

---

## 6. 常见 ORR 要求模板

可以直接把下面这些作为生产 Agent 发布 checklist：

### Observability

- 有核心 SLO：成功率、P95 延迟、工具失败率、成本预算。
- 有 dashboard 链接。
- 有告警规则和冷却期。
- 日志包含 runId / actor / tool / decision / evidence。

### Safety

- Kill Switch 已测试。
- Pause / Resume / Cancel 可用。
- 高风险工具默认 dry-run 或审批。
- Blast Radius Budget 已配置。

### Recovery

- 回滚或补偿 runbook 存在。
- 至少一次恢复演练通过。
- 幂等键和 Outbox/Effect Journal 覆盖副作用。
- Dead Letter Queue 或失败档案可查。

### Security

- 工具权限最小化。
- Secret 只以 SecretRef 出现，不进入 prompt。
- 数据访问带 actor/tenant/scope/purpose。
- 缓存/记忆有权限边界。

### Operations

- 负责人/on-call 明确。
- 升级路径明确。
- 版本、配置、prompt、工具 schema 有 fingerprint。
- 发布证据包归档。

---

## 7. ORR 和前面课程的关系

ORR 不是一个孤立功能，而是把前面很多能力串起来：

- **Runbook**：证明出问题时有剧本。
- **Recovery Drill**：证明剧本跑得通。
- **Resilience Scorecard**：证明覆盖率够。
- **Kill Switch**：证明能停。
- **Blast Radius**：证明影响范围有限。
- **Release Evidence Bundle**：证明发布过程可审计。
- **Completion Gates**：证明每次任务真的完成。

可以把 ORR 理解为：**Agent 上生产前的总闸门**。

---

## 8. 实战建议

1. **不要做成人工表格**：Checklist 最终一定会过期；要把它变成代码 Gate。
2. **证据必须有时间戳**：三个月前通过的演练，不一定能证明今天还 ready。
3. **按自主等级分层**：observe 可以宽松，full autonomy 必须严格。
4. **缺证据先降级，不要硬上**：从 full 降到 canary/dry-run，通常比直接 block 更实用。
5. **ORR 结果要入审计日志**：以后事故复盘要能看到当时为什么允许它跑。

---

## 9. 小结

今天这课的核心：

- 功能测试通过 ≠ 运行就绪。
- ORR 把生产运行要求变成可执行证据门。
- 缺证据时自动 block 或降低自主等级。
- learn-claude-code 可以用 Python 小 Gate 教学。
- pi-mono 应该把 ReadinessGate 接进 Release Controller / Agent Runtime。
- OpenClaw 这类 always-on Agent 尤其需要 ORR，因为它会长期自动执行外部副作用。

一句话：**成熟 Agent 不是“能跑就上线”，而是能证明自己监控得到、停得下来、恢复得了、权限收得住，才允许更高自主等级。**
