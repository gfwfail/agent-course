# 306. Agent 策略差异预算与晋级门控（Policy Diff Budget & Promotion Gate）

上一课讲了 **Policy Shadow Mode**：新策略先跟跑，只记录 diff，不影响真实动作。

但 shadow 跑一段时间后，问题来了：

> 到底什么时候可以把 shadow policy 晋级成 primary policy？

不能靠“感觉差不多”。成熟 Agent 需要一个可执行的晋级门控：把 shadow diff 转成指标，再用 **Diff Budget** 决定是否允许发布。

---

## 1. 核心思想

Policy Diff Budget 的目标不是追求“新旧策略完全一致”。如果完全一致，新策略也没价值。

它真正关心三类差异：

1. **危险放宽**：旧策略要求审批/拒绝，新策略允许执行；
2. **过度收紧**：旧策略允许，新策略大量改成审批/拒绝，影响自动化效率；
3. **理由漂移**：action 一样，但 reason/risk 变化，说明策略解释口径变了。

一个实用的晋级门控可以这样定义：

- `dangerous_relaxation == 0`：绝不允许无审批放宽高风险动作；
- `tightening_rate <= 5%`：收紧可以有，但不能突然把系统变成人工审批机器；
- `sample_count >= 200`：样本太少不晋级；
- `shadow_error_rate <= 0.5%`：新策略自身不能频繁报错；
- `manual_review_diffs` 必须有人看过。

一句话：**shadow 负责观察，diff budget 负责给晋级一个硬门槛。**

---

## 2. learn-claude-code：最小 Python Diff Budget 检查器

教学版可以直接读取上一课产生的 `policy-shadow.jsonl`。

```python
# learn-claude-code/policy_diff_budget.py
from dataclasses import dataclass
import json
from pathlib import Path

STRICTNESS = {
    "allow": 0,
    "dry_run": 1,
    "require_approval": 2,
    "deny": 3,
}

@dataclass
class DiffStats:
    total: int = 0
    dangerous_relaxation: int = 0
    tightening: int = 0
    risk_downgrade: int = 0
    shadow_errors: int = 0


def classify(event: dict, stats: DiffStats) -> None:
    if event.get("type") == "policy_shadow_error":
        stats.shadow_errors += 1
        stats.total += 1
        return

    primary = event["primary"]
    shadow = event["shadow"]
    p = STRICTNESS[primary["action"]]
    s = STRICTNESS[shadow["action"]]

    stats.total += 1

    # shadow 比 primary 更宽松：require_approval -> allow 这类最危险
    if s < p:
        stats.dangerous_relaxation += 1

    # shadow 比 primary 更严格：可能安全，但会降低自动化率
    if s > p:
        stats.tightening += 1

    if primary.get("risk") == "high" and shadow.get("risk") in {"medium", "low"}:
        stats.risk_downgrade += 1


def evaluate(path: str) -> dict:
    stats = DiffStats()
    for line in Path(path).read_text().splitlines():
        if line.strip():
            classify(json.loads(line), stats)

    tightening_rate = stats.tightening / max(stats.total, 1)
    shadow_error_rate = stats.shadow_errors / max(stats.total, 1)

    passed = (
        stats.total >= 200
        and stats.dangerous_relaxation == 0
        and stats.risk_downgrade == 0
        and tightening_rate <= 0.05
        and shadow_error_rate <= 0.005
    )

    return {
        "passed": passed,
        "stats": stats.__dict__,
        "tighteningRate": round(tightening_rate, 4),
        "shadowErrorRate": round(shadow_error_rate, 4),
    }


if __name__ == "__main__":
    print(json.dumps(evaluate("policy-shadow.jsonl"), indent=2, ensure_ascii=False))
```

这里的重点是：**晋级不是一个按钮，而是一个有证据的判定。**

---

## 3. pi-mono：PolicyPromotionGate 中间件

生产版可以把 diff budget 做成发布流水线里的 gate。

```ts
// pi-mono/src/policy/policy-promotion-gate.ts
type PolicyAction = 'allow' | 'dry_run' | 'require_approval' | 'deny'

type Decision = {
  action: PolicyAction
  risk: 'low' | 'medium' | 'high'
  reason: string
  policyVersion: string
}

type ShadowDiff = {
  runId: string
  toolName: string
  primary: Decision
  shadow: Decision
  type: 'policy_shadow_diff' | 'policy_shadow_error'
}

type PromotionResult = {
  decision: 'promote' | 'hold' | 'manual_review'
  reasons: string[]
  metrics: Record<string, number>
}

const strictness: Record<PolicyAction, number> = {
  allow: 0,
  dry_run: 1,
  require_approval: 2,
  deny: 3,
}

export class PolicyPromotionGate {
  constructor(
    private minSamples = 200,
    private maxTighteningRate = 0.05,
    private maxShadowErrorRate = 0.005,
  ) {}

  evaluate(diffs: ShadowDiff[]): PromotionResult {
    let dangerousRelaxation = 0
    let tightening = 0
    let riskDowngrade = 0
    let shadowErrors = 0

    for (const diff of diffs) {
      if (diff.type === 'policy_shadow_error') {
        shadowErrors++
        continue
      }

      const p = strictness[diff.primary.action]
      const s = strictness[diff.shadow.action]

      if (s < p) dangerousRelaxation++
      if (s > p) tightening++

      if (diff.primary.risk === 'high' && diff.shadow.risk !== 'high') {
        riskDowngrade++
      }
    }

    const samples = diffs.length
    const tighteningRate = tightening / Math.max(samples, 1)
    const shadowErrorRate = shadowErrors / Math.max(samples, 1)
    const reasons: string[] = []

    if (samples < this.minSamples) reasons.push('not enough shadow samples')
    if (dangerousRelaxation > 0) reasons.push('dangerous relaxation detected')
    if (riskDowngrade > 0) reasons.push('high risk downgraded by shadow policy')
    if (tighteningRate > this.maxTighteningRate) reasons.push('too much automation loss')
    if (shadowErrorRate > this.maxShadowErrorRate) reasons.push('shadow policy unstable')

    if (dangerousRelaxation > 0 || riskDowngrade > 0) {
      return {
        decision: 'manual_review',
        reasons,
        metrics: { samples, dangerousRelaxation, tighteningRate, shadowErrorRate },
      }
    }

    return {
      decision: reasons.length === 0 ? 'promote' : 'hold',
      reasons,
      metrics: { samples, dangerousRelaxation, tighteningRate, shadowErrorRate },
    }
  }
}
```

这个 gate 可以放在：

- feature flag 从 `shadow` 切到 `primary` 前；
- policy version 发布前；
- 高风险工具权限放宽前；
- cron 自主权限升级前。

---

## 4. OpenClaw 实战：课程 Cron 的策略晋级门控

拿 OpenClaw 的课程 cron 举例：

当前流程里，课程 cron 会：

1. 写 lesson 文件；
2. 更新 README / TOOLS；
3. 发 Telegram 群消息；
4. git commit & push。

如果我们准备上线一条“更严格的发布策略”，不要直接替换主策略。先让它 shadow 跑 1-2 天，记录：

```json
{
  "type": "policy_shadow_diff",
  "operation": "git_push",
  "primary": {
    "action": "allow",
    "risk": "medium",
    "policyVersion": "course-policy@2026-05-01"
  },
  "shadow": {
    "action": "require_approval",
    "risk": "high",
    "policyVersion": "course-policy@2026-05-13"
  }
}
```

然后发布前跑一个 promotion gate：

```bash
python learn-claude-code/policy_diff_budget.py
```

如果结果是：

```json
{
  "passed": false,
  "stats": {
    "total": 96,
    "dangerous_relaxation": 0,
    "tightening": 44,
    "risk_downgrade": 0,
    "shadow_errors": 0
  },
  "tighteningRate": 0.4583
}
```

结论不是“新策略更安全所以直接上”，而是：

> hold。它确实更保守，但会让 45% 的自动发布进入审批，自动化损耗太大。

这时应该调策略：比如只对 `deploy_production` 和 `delete_resource` 要审批，课程 cron 的 `git_push` 只要求先跑 PR 状态检查和 diff check。

---

## 5. 生产 checklist

策略从 shadow 晋级 primary 前，至少检查：

- [ ] shadow 样本数足够，不是只跑了几个 happy path；
- [ ] 没有 `deny/require_approval -> allow` 的危险放宽；
- [ ] 没有 `high -> low/medium` 的风险降级；
- [ ] 收紧率在预算内，避免把自动化变成全人工；
- [ ] shadow error rate 很低；
- [ ] 所有 `manual_review` diff 有人看过；
- [ ] 晋级后保留 rollback target，可以一键切回旧策略。

---

## 6. 记忆点

Shadow mode 解决“新策略会怎么判断”；Diff Budget 解决“新策略能不能上线”。

成熟 Agent 的策略发布应该像代码发布一样：

> 先跟跑，再量化，再门控，最后可回滚地晋级。
