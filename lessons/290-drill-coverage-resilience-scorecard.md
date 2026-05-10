# 290. Agent 演练覆盖率与恢复成熟度评分（Drill Coverage & Resilience Scorecard）

上一课讲了 **Recovery Drill / Game Day**：主动注入故障，验证恢复路径能不能跑通。今天继续补上一个容易被忽略的问题：**你怎么知道演练覆盖够不够？**

只做一次 timeout 演练，不代表系统真的可靠。成熟做法是把演练结果沉淀成一张可计算的 **Resilience Scorecard**：覆盖了哪些工具、哪些故障类型、哪些恢复路径、哪些副作用边界；缺口是什么；是否允许继续发布。

> 重点：Game Day 证明“某个场景能恢复”，Scorecard 证明“关键场景大体都被验证过”。

---

## 1. 为什么需要演练覆盖率？

很多团队做演练会停在“今天演练通过了”，但 Agent 系统的问题是：

1. **工具很多**：message、git、deploy、browser、database、payment，每个恢复语义不同。
2. **故障类型很多**：timeout、rate limit、权限吊销、重复投递、数据损坏、子 Agent 崩溃。
3. **风险等级不同**：只读查询失败和生产写操作失败，不应该同等看待。
4. **恢复路径会漂移**：Runbook 改了、工具 schema 改了、权限策略改了，旧演练结果会过期。
5. **“没测过”比“失败”更危险**：失败至少可见，没覆盖的高风险路径会在事故时第一次执行。

所以我们需要一个明确的矩阵：

- 高风险工具是否都演练过？
- 每类关键故障是否都覆盖？
- 演练证据是否还新鲜？
- 是否验证了 no duplicate side effects？
- 是否验证了人工接管 / pause / rollback？

---

## 2. 核心设计：Coverage Matrix + Freshness + Gate

可以把每次演练结果标准化成 `DrillResult`：

```ts
type DrillResult = {
  drillId: string;
  tool: string;
  risk: 'low' | 'medium' | 'high' | 'critical';
  fault: 'timeout' | 'rate_limit' | 'permission_revoked' | 'corrupt_data' | 'duplicate_delivery' | 'crash';
  recoveryPath: 'retry' | 'degrade' | 'rollback' | 'manual_review' | 'pause_resume';
  passed: boolean;
  evidence: string[];
  executedAt: string;
  policyVersion: string;
};
```

Scorecard 做三件事：

1. **覆盖矩阵**：`tool × fault × recoveryPath` 是否有通过记录。
2. **新鲜度检查**：超过 TTL 的演练结果降权或失效。
3. **发布门控**：critical/high 缺口未补齐时，阻断发布或要求人工确认。

---

## 3. learn-claude-code：最小 Python 评分器

教学版不用复杂数据库，直接用一组演练结果计算分数和缺口。

```python
# learn_claude_code/drill_scorecard.py
from dataclasses import dataclass
from datetime import datetime, timezone, timedelta

@dataclass(frozen=True)
class Requirement:
    tool: str
    risk: str
    fault: str
    recovery_path: str
    max_age_days: int

@dataclass(frozen=True)
class DrillResult:
    drill_id: str
    tool: str
    risk: str
    fault: str
    recovery_path: str
    passed: bool
    evidence: tuple[str, ...]
    executed_at: datetime
    policy_version: str

REQUIRED_EVIDENCE = {
    "retry": {"fault_detected", "retry_scheduled"},
    "degrade": {"fault_detected", "degraded_response"},
    "rollback": {"fault_detected", "rollback_started", "rollback_verified"},
    "manual_review": {"fault_detected", "operator_notified"},
    "pause_resume": {"pause_checkpointed", "resume_revalidated"},
}

class DrillScorecard:
    def __init__(self, requirements: list[Requirement], now: datetime):
        self.requirements = requirements
        self.now = now

    def _matches(self, req: Requirement, result: DrillResult) -> bool:
        return (
            result.tool == req.tool
            and result.risk == req.risk
            and result.fault == req.fault
            and result.recovery_path == req.recovery_path
        )

    def _valid(self, req: Requirement, result: DrillResult) -> bool:
        if not result.passed:
            return False
        if self.now - result.executed_at > timedelta(days=req.max_age_days):
            return False
        needed = REQUIRED_EVIDENCE.get(req.recovery_path, set())
        return needed.issubset(set(result.evidence))

    def evaluate(self, results: list[DrillResult]) -> dict:
        covered = []
        missing = []

        for req in self.requirements:
            candidates = [r for r in results if self._matches(req, r)]
            if any(self._valid(req, r) for r in candidates):
                covered.append(req)
            else:
                missing.append(req)

        score = round(len(covered) / max(len(self.requirements), 1) * 100, 1)
        critical_missing = [m for m in missing if m.risk in {"high", "critical"}]

        return {
            "score": score,
            "covered": len(covered),
            "required": len(self.requirements),
            "missing": [m.__dict__ for m in missing],
            "gate": "pass" if not critical_missing and score >= 85 else "block",
        }

if __name__ == "__main__":
    now = datetime.now(timezone.utc)
    requirements = [
        Requirement("message.send", "high", "timeout", "retry", 14),
        Requirement("git.push", "critical", "permission_revoked", "manual_review", 30),
        Requirement("deploy.release", "critical", "timeout", "rollback", 14),
    ]
    results = [
        DrillResult(
            "msg-timeout-001", "message.send", "high", "timeout", "retry", True,
            ("fault_detected", "retry_scheduled"), now - timedelta(days=2), "policy-v7"
        ),
    ]
    print(DrillScorecard(requirements, now).evaluate(results))
```

输出会告诉你不是“系统可靠”，而是：当前只覆盖了 1/3，`git.push` 和 `deploy.release` 的关键恢复路径还没验证，发布门控应该 block。

---

## 4. pi-mono：生产版 ResilienceScorecard

生产版建议把需求清单放在配置里，把演练结果来自 DrillRunner / Evidence Store。

```ts
// pi-mono/packages/agent-runtime/src/resilience/ResilienceScorecard.ts
export type Risk = 'low' | 'medium' | 'high' | 'critical';
export type FaultKind = 'timeout' | 'rate_limit' | 'permission_revoked' | 'corrupt_data' | 'duplicate_delivery' | 'crash';
export type RecoveryPath = 'retry' | 'degrade' | 'rollback' | 'manual_review' | 'pause_resume';

export interface CoverageRequirement {
  id: string;
  tool: string;
  risk: Risk;
  fault: FaultKind;
  recoveryPath: RecoveryPath;
  maxAgeDays: number;
  weight: number;
}

export interface DrillResult {
  drillId: string;
  tool: string;
  risk: Risk;
  fault: FaultKind;
  recoveryPath: RecoveryPath;
  passed: boolean;
  evidence: string[];
  executedAt: Date;
  policyVersion: string;
}

export interface ScorecardReport {
  score: number;
  gate: 'pass' | 'warn' | 'block';
  missing: CoverageRequirement[];
  stale: CoverageRequirement[];
  coveredWeight: number;
  totalWeight: number;
}

const REQUIRED_EVIDENCE: Record<RecoveryPath, string[]> = {
  retry: ['fault_detected', 'retry_scheduled'],
  degrade: ['fault_detected', 'degraded_response'],
  rollback: ['fault_detected', 'rollback_started', 'rollback_verified'],
  manual_review: ['fault_detected', 'operator_notified'],
  pause_resume: ['pause_checkpointed', 'resume_revalidated'],
};

export class ResilienceScorecard {
  constructor(
    private readonly requirements: CoverageRequirement[],
    private readonly now: Date = new Date(),
  ) {}

  evaluate(results: DrillResult[]): ScorecardReport {
    let coveredWeight = 0;
    let totalWeight = 0;
    const missing: CoverageRequirement[] = [];
    const stale: CoverageRequirement[] = [];

    for (const req of this.requirements) {
      totalWeight += req.weight;
      const candidates = results.filter((result) => this.matches(req, result));
      const fresh = candidates.filter((result) => this.isFresh(req, result));
      const valid = fresh.some((result) => this.hasEvidence(req, result));

      if (valid) {
        coveredWeight += req.weight;
        continue;
      }

      if (candidates.length > 0 && fresh.length === 0) stale.push(req);
      else missing.push(req);
    }

    const score = totalWeight === 0 ? 100 : Math.round((coveredWeight / totalWeight) * 1000) / 10;
    const criticalGap = [...missing, ...stale].some((req) => req.risk === 'critical');
    const highGap = [...missing, ...stale].some((req) => req.risk === 'high');

    return {
      score,
      gate: criticalGap || score < 80 ? 'block' : highGap || score < 90 ? 'warn' : 'pass',
      missing,
      stale,
      coveredWeight,
      totalWeight,
    };
  }

  private matches(req: CoverageRequirement, result: DrillResult): boolean {
    return result.passed
      && result.tool === req.tool
      && result.risk === req.risk
      && result.fault === req.fault
      && result.recoveryPath === req.recoveryPath;
  }

  private isFresh(req: CoverageRequirement, result: DrillResult): boolean {
    const ageMs = this.now.getTime() - result.executedAt.getTime();
    return ageMs <= req.maxAgeDays * 24 * 60 * 60 * 1000;
  }

  private hasEvidence(req: CoverageRequirement, result: DrillResult): boolean {
    const evidence = new Set(result.evidence);
    return REQUIRED_EVIDENCE[req.recoveryPath].every((event) => evidence.has(event));
  }
}
```

注意这里用了 `weight`：

- `deploy.release + rollback` 权重可以是 10。
- `message.send + retry` 权重可以是 4。
- 低风险只读工具权重可以是 1。

这样分数不会被大量低风险演练“刷高”。

---

## 5. OpenClaw 落地：课程 Cron 的自检清单

OpenClaw 这种 always-on Agent，最小可落地版本可以是一个 JSON 文件：

```json
{
  "requirements": [
    {
      "id": "course-message-timeout-retry",
      "tool": "message.send",
      "risk": "high",
      "fault": "timeout",
      "recoveryPath": "retry",
      "maxAgeDays": 14,
      "weight": 4
    },
    {
      "id": "course-git-push-auth-manual-review",
      "tool": "git.push",
      "risk": "critical",
      "fault": "permission_revoked",
      "recoveryPath": "manual_review",
      "maxAgeDays": 30,
      "weight": 8
    }
  ]
}
```

每次课程 cron / 发布 cron 可以做一个轻量 gate：

1. 读取最近 DrillResult JSONL。
2. 计算 scorecard。
3. 如果 gate = `pass`：继续自动发布。
4. 如果 gate = `warn`：继续但附带告警和缺口列表。
5. 如果 gate = `block`：停止高风险副作用，只生成草稿和报告。

这跟普通测试不同：测试验证代码逻辑，Scorecard 验证**恢复能力是否还被定期证明**。

---

## 6. 实战建议

1. **先覆盖最高风险工具**：deploy、payment、git push、外部消息、配置写入。
2. **每个高风险工具至少覆盖三类故障**：timeout、权限失败、重复/半成功副作用。
3. **演练结果必须有 TTL**：三个月前通过，不代表今天仍然可靠。
4. **缺口要进入 backlog**：missing coverage 不是备注，是工程债。
5. **分数不要追求 100**：关键路径 90+ 比低风险路径刷满更有价值。

---

## 7. 小结

Recovery Drill 是一次演练，Resilience Scorecard 是持续治理。

成熟 Agent 不是靠一句“我们做过 Game Day”来证明可靠，而是能随时回答：

- 哪些高风险副作用已经演练过？
- 哪些恢复路径过期了？
- 当前恢复覆盖率是多少？
- 今天这次发布是否应该被阻断？

一句话：**演练让恢复路径可执行，评分卡让恢复能力可管理。**
