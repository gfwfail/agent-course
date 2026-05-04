# 242. Agent 观察快照与执行前复核（Observation Snapshot & Pre-Action Revalidation）

> 核心思想：Agent 做副作用操作前，不能只相信几分钟前看到的世界。把关键观察冻结成快照，并在执行前按风险等级复核，才能避免“我刚才看是这样的”导致错误提交、误删、误发、误部署。

## 为什么需要这一层？

很多生产事故不是 Agent 不会推理，而是它基于**过期观察**行动：

- 10 分钟前 PR 还是 open，准备 push 时已经 merged；
- 读取库存时有货，付款前库存被别人锁走；
- 看到用户权限是 admin，执行前权限已被撤销；
- 监控检查时服务异常，准备告警时服务已恢复；
- 本地 diff 检查干净，提交前其他进程改了文件。

所以成熟 Agent 的动作链应该是：

```text
observe → snapshot → plan → pre-action revalidate → act → verify
```

注意它和“数据新鲜度管理”不同：Freshness 关心“数据过期了吗”；Pre-action Revalidation 关心“马上要产生副作用了，关键前提还成立吗”。

## 设计一个 Observation Snapshot

一个快照不是把所有上下文全塞进去，而是记录会影响动作安全性的前提条件。

```ts
export type ObservationSnapshot = {
  id: string;
  runId: string;
  observedAt: string;
  purpose: string;
  assumptions: Assumption[];
};

export type Assumption = {
  key: string;
  value: unknown;
  source: 'tool' | 'memory' | 'user' | 'cache';
  risk: 'low' | 'medium' | 'high' | 'critical';
  revalidate: 'never' | 'if_stale' | 'always_before_action';
  maxAgeMs?: number;
  check: () => Promise<unknown>;
};
```

例子：准备 `git push` 前，关键前提不是“我想 push”，而是：

```ts
const snapshot = createSnapshot({
  purpose: 'push lesson 242 to GitHub',
  assumptions: [
    {
      key: 'branch_is_main_and_remote_current',
      value: { branch: 'main', behind: 0, ahead: 1 },
      source: 'tool',
      risk: 'high',
      revalidate: 'always_before_action',
      check: () => gitStatus()
    },
    {
      key: 'github_account',
      value: 'gfwfail',
      source: 'tool',
      risk: 'medium',
      revalidate: 'always_before_action',
      check: () => ghCurrentUser()
    }
  ]
});
```

## learn-claude-code：Python 教学版

教学版可以先做一个最小复核器：动作前把 high/critical 前提重新查一遍。

```py
from dataclasses import dataclass
from datetime import datetime, timezone
from typing import Awaitable, Callable, Any

@dataclass
class Assumption:
    key: str
    value: Any
    risk: str
    revalidate: str
    check: Callable[[], Awaitable[Any]]

@dataclass
class ObservationSnapshot:
    run_id: str
    purpose: str
    observed_at: datetime
    assumptions: list[Assumption]

class PreActionRevalidator:
    async def validate(self, snapshot: ObservationSnapshot):
        failures = []

        for assumption in snapshot.assumptions:
            if assumption.revalidate != "always_before_action":
                continue

            current = await assumption.check()
            if current != assumption.value:
                failures.append({
                    "key": assumption.key,
                    "expected": assumption.value,
                    "current": current,
                    "risk": assumption.risk,
                })

        if failures:
            raise RuntimeError({
                "type": "PRE_ACTION_REVALIDATION_FAILED",
                "purpose": snapshot.purpose,
                "failures": failures,
                "llmHint": "重新观察现实状态，基于最新结果重新规划；不要继续执行旧计划。"
            })
```

用法：

```py
async def safe_action(snapshot, action):
    await PreActionRevalidator().validate(snapshot)
    result = await action()
    return result
```

这段代码很朴素，但已经避免一类大坑：Agent 不会拿旧状态硬干。

## pi-mono：TypeScript 生产中间件

生产版更适合做成 Tool Middleware，凡是带副作用的工具自动触发复核。

```ts
type ToolEffect = 'read' | 'write' | 'external' | 'destructive';

type ToolCall = {
  name: string;
  args: unknown;
  effect: ToolEffect;
  snapshot?: ObservationSnapshot;
};

export class PreActionRevalidationMiddleware {
  async beforeToolCall(call: ToolCall) {
    if (call.effect === 'read') return;
    if (!call.snapshot) {
      throw new Error(`Missing observation snapshot for side-effect tool: ${call.name}`);
    }

    const failures = [];

    for (const a of call.snapshot.assumptions) {
      if (a.revalidate !== 'always_before_action') continue;
      if (!['high', 'critical'].includes(a.risk)) continue;

      const current = await a.check();
      if (JSON.stringify(current) !== JSON.stringify(a.value)) {
        failures.push({ key: a.key, expected: a.value, current, risk: a.risk });
      }
    }

    if (failures.length > 0) {
      return {
        blocked: true,
        error: {
          code: 'PRE_ACTION_REVALIDATION_FAILED',
          message: '关键前提已变化，禁止执行副作用工具。',
          failures,
          suggestedAction: 'refresh_observation_and_replan'
        }
      };
    }
  }
}
```

工具声明时标注副作用等级：

```ts
registry.register({
  name: 'github.push',
  effect: 'external',
  requiredSnapshot: true,
  handler: pushToGitHub,
});
```

这样 LLM 不需要每次都记得“push 前检查一下”，系统会强制它遵守。

## OpenClaw 实战模式

OpenClaw 里可以把这个模式落成一条简单铁律：

```text
任何外部副作用前，如果前提来自 live system，就重新查 primary source。
```

例子：课程 cron 推送前：

1. 写 lesson/README/TOOLS；
2. `git diff --check`；
3. `gh auth switch --user gfwfail`；
4. **执行前复核**：`gh auth status`、`git status`、`git log -1`；
5. 再 `git push`；
6. 发送 Telegram；
7. 记录 messageId 和 commit hash。

对于 GitHub PR 工作流，复核点更关键：

```bash
# 不要相信记忆里的 PR 状态
# push/merge 前必须 live check

gh pr list --state all --limit 5
gh pr view <id> --json state,mergeable,statusCheckRollup
```

这不是保守，是工程化：副作用越接近发生，观察就越要新。

## 复核策略怎么定？

不用所有东西都复核，否则 Agent 会慢死。按风险分层：

| 风险 | 例子 | 策略 |
|---|---|---|
| low | 生成草稿、格式化文本 | 不复核 |
| medium | 写本地文件、更新 README | stale 才复核 |
| high | git push、发群消息、创建 PR | action 前必复核 |
| critical | 删除数据、部署生产、转账 | 复核 + 人工审批 + 执行后 verify |

## 常见坑

1. **只记录结果，不记录来源**  
   `status=open` 没用，要知道它来自 `gh pr view`，什么时候查的。

2. **复核太晚或太早**  
   复核要贴近副作用执行前，不能在计划阶段查完就算数。

3. **复核失败后继续旧计划**  
   复核失败不是“再试一次工具”，而是“重新观察并重新规划”。

4. **只复核业务状态，不复核身份状态**  
   GitHub 当前账号、Telegram 目标群、环境变量 profile 都属于关键前提。

## 一句话总结

Observation Snapshot 让 Agent 知道“我当时是基于哪些事实做计划”；Pre-action Revalidation 让 Agent 在真正动手前确认“这些事实现在还成立”。

靠谱 Agent 的副作用不是靠自信执行，而是靠最后一秒的现实复核。🫡
