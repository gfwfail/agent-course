# 234. Agent 变更风险评分与发布门控（Change Risk Scoring & Release Gates）

> 核心思想：Agent 做完代码/配置变更后，不应该只问“测试过了吗”，还要问“这次变更有多危险，应该走什么发布路径”。把 diff、文件类型、外部副作用、测试覆盖、历史失败率转成风险分数，再按分数自动决定：直接提交、需要人工 review、灰度发布，还是禁止发布。

很多生产事故不是因为 Agent 没改对代码，而是因为它低估了风险：

- 改了 migration / billing / auth，却只跑了前端 build；
- 删除了配置字段，测试通过但线上环境缺变量；
- 小 PR 改到了支付回调，应该人工确认却直接 deploy；
- cron 自动任务在夜里推了高风险变更，没人看。

所以 Agent 的发布系统要有“风险雷达”。

---

## 1. 风险评分的输入

一个实用的 ChangeRisk 可以先看五类信号：

```text
risk = files + blastRadius + sideEffects + testEvidence + history
```

建议权重：

| 信号 | 例子 | 加分 |
|---|---|---:|
| 敏感文件 | migration、auth、billing、payment、deploy config | +30 |
| 外部副作用 | 发消息、扣款、部署、删数据、写生产 DB | +35 |
| 影响范围 | 公共组件、API schema、数据库字段、队列任务 | +20 |
| 验证不足 | 没有测试/没跑 build/只做静态检查 | +20 |
| 历史失败 | 最近同类 PR 回滚/CI flaky/线上报错 | +15 |

分数不是为了“精确预测事故”，而是为了把发布策略自动分层：

```text
0-29   low      自动提交/常规 PR
30-59  medium   必须跑完整测试 + 人工 review
60-79  high     必须灰度/feature flag/approval
80+    critical 禁止自动发布，升级给人
```

---

## 2. learn-claude-code：Python 教学版

教学版可以从 git diff 文件名开始，不需要复杂模型：

```python
from dataclasses import dataclass
from typing import Literal

RiskLevel = Literal['low', 'medium', 'high', 'critical']

@dataclass
class RiskReport:
    score: int
    level: RiskLevel
    reasons: list[str]
    required_gates: list[str]

SENSITIVE_PATTERNS = {
    'migration': 30,
    'payment': 35,
    'billing': 35,
    'auth': 30,
    '.env': 40,
    'deploy': 25,
    'workflow': 20,
}

def score_changed_files(files: list[str], tests_ran: list[str]) -> RiskReport:
    score = 0
    reasons: list[str] = []

    for file in files:
        lower = file.lower()
        for pattern, points in SENSITIVE_PATTERNS.items():
            if pattern in lower:
                score += points
                reasons.append(f'{file} 命中敏感模式 {pattern} +{points}')

    if not tests_ran:
        score += 20
        reasons.append('没有测试证据 +20')

    if any('migration' in f.lower() for f in files) and 'migration:test' not in tests_ran:
        score += 20
        reasons.append('改了 migration 但没跑迁移测试 +20')

    level: RiskLevel = 'low'
    if score >= 80:
        level = 'critical'
    elif score >= 60:
        level = 'high'
    elif score >= 30:
        level = 'medium'

    gates = ['unit-tests']
    if level in ('medium', 'high', 'critical'):
        gates += ['full-build', 'human-review']
    if level in ('high', 'critical'):
        gates += ['staging-verify', 'rollback-plan']
    if level == 'critical':
        gates += ['manual-release-approval']

    return RiskReport(score, level, reasons, gates)
```

接入 Agent Loop：

```python
async def before_publish(ctx):
    files = await ctx.git_changed_files()
    report = score_changed_files(files, ctx.tests_ran)

    ctx.memory['risk_report'] = report

    if report.level == 'critical':
        return {
            'status': 'blocked',
            'reason': 'critical risk requires human approval',
            'risk': report,
        }

    ctx.required_gates.extend(report.required_gates)
    return {'status': 'ok', 'risk': report}
```

这一步的价值：Agent 不是“改完就跑”，而是先知道自己改到了危险区域。

---

## 3. pi-mono：生产版 RiskPolicy + Gate Router

生产版建议把风险评分做成中间件，输出统一结构：

```ts
type RiskLevel = 'low' | 'medium' | 'high' | 'critical';

type ChangeRiskReport = {
  score: number;
  level: RiskLevel;
  reasons: string[];
  requiredGates: string[];
  releaseMode: 'auto_pr' | 'review_required' | 'canary' | 'manual_only';
};

type ChangeContext = {
  changedFiles: string[];
  toolCalls: Array<{ name: string; sideEffect?: 'none' | 'external' | 'destructive' }>;
  evidence: { tests: string[]; build?: boolean; screenshots?: string[] };
};

export class ChangeRiskScorer {
  score(ctx: ChangeContext): ChangeRiskReport {
    let score = 0;
    const reasons: string[] = [];

    const add = (points: number, reason: string) => {
      score += points;
      reasons.push(`${reason} +${points}`);
    };

    for (const file of ctx.changedFiles) {
      const f = file.toLowerCase();
      if (f.includes('migration')) add(30, `migration file: ${file}`);
      if (f.includes('payment') || f.includes('billing')) add(35, `money path: ${file}`);
      if (f.includes('auth') || f.includes('permission')) add(30, `auth path: ${file}`);
      if (f.includes('.github/workflows') || f.includes('deploy')) add(25, `deployment path: ${file}`);
    }

    for (const call of ctx.toolCalls) {
      if (call.sideEffect === 'external') add(20, `external side effect: ${call.name}`);
      if (call.sideEffect === 'destructive') add(45, `destructive side effect: ${call.name}`);
    }

    if (ctx.evidence.tests.length === 0) add(20, 'no test evidence');
    if (!ctx.evidence.build) add(10, 'no build evidence');

    const level: RiskLevel =
      score >= 80 ? 'critical' : score >= 60 ? 'high' : score >= 30 ? 'medium' : 'low';

    return {
      score,
      level,
      reasons,
      requiredGates: this.gatesFor(level),
      releaseMode: this.releaseModeFor(level),
    };
  }

  private gatesFor(level: RiskLevel) {
    return {
      low: ['diff-check'],
      medium: ['diff-check', 'unit-tests', 'build', 'review'],
      high: ['diff-check', 'unit-tests', 'build', 'staging', 'rollback-plan', 'approval'],
      critical: ['manual-security-review', 'manual-release-approval'],
    }[level];
  }

  private releaseModeFor(level: RiskLevel): ChangeRiskReport['releaseMode'] {
    return {
      low: 'auto_pr',
      medium: 'review_required',
      high: 'canary',
      critical: 'manual_only',
    }[level];
  }
}
```

然后 Task Runner 根据 releaseMode 路由：

```ts
const risk = new ChangeRiskScorer().score(ctx.change);
ctx.audit.write('change_risk_scored', risk);

if (risk.releaseMode === 'manual_only') {
  return ctx.stop({
    status: 'blocked',
    message: '变更风险为 critical，禁止自动发布',
    evidence: risk,
  });
}

ctx.completionGates.addMany(risk.requiredGates);
ctx.release.mode = risk.releaseMode;
```

重点：风险评分不是替代测试，而是决定“需要哪些测试和审批”。

---

## 4. OpenClaw 实战：PR 前风险摘要

OpenClaw 里最常见的场景是：Agent 改代码、开 PR、跑 CI、发给老板。

可以在 PR body 里固定加一段：

```md
## Change Risk
- Level: high
- Score: 70
- Reasons:
  - database migration changed +30
  - payment flow touched +35
  - build passed but no integration test +5
- Required gates:
  - npm run build ✅
  - migration dry-run ✅
  - human review pending
- Release mode: canary
```

对 cron/heartbeat 自动任务尤其重要：

- low：可以自动写 lesson、更新 README、push；
- medium：可以开 PR，但最终 merge 等人；
- high：必须截图/测试证据齐全，再请求 approval；
- critical：只做诊断，不做外部写入。

这和前几课的 Completion Gates 可以组合：

```text
ChangeRiskScorer 决定要过哪些门
CompletionGateRunner 负责逐个验门
EffectJournal 记录门过没过、发布做没做
```

---

## 5. 常见坑

1. **只按文件数量评估风险**：改一行支付回调比改 500 行文案危险；
2. **只看测试是否通过**：测试覆盖不到的外部副作用仍然危险；
3. **风险分数不解释原因**：Agent 和 reviewer 都不知道为什么被拦；
4. **没有 releaseMode**：评分只是日志，没有改变行为；
5. **不记录风险报告**：事故后无法复盘当时为什么放行。

---

## 6. 小结

Agent 发布变更要三问：

```text
改了什么？风险多高？这个风险等级需要哪些门？
```

风险评分的目标不是追求数学精确，而是把“凭经验觉得危险”变成稳定、可审计、可自动执行的发布策略。

成熟 Agent 不是永远大胆自动化，而是知道什么时候该减速、什么时候必须等人。
