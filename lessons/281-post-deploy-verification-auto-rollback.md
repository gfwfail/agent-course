# 281. Agent 部署后验证与自动回滚（Post-Deploy Verification & Auto-Rollback）

> 核心思想：部署成功只代表“新版本被放上去了”，不代表“新版本真的可用”。成熟 Agent 要在发布后自动验证关键路径，一旦证据不达标，就按预案回滚，而不是等用户报警。

很多自动化系统把 `deploy` 当成最后一步：

```text
测试通过 → 部署 → 发消息：已完成
```

但生产里真正危险的是：

- CI 绿了，生产环境变量缺失
- 容器启动了，关键工具调用 500
- Agent 能回复，但不能执行审批 / 记忆 / 外部 API
- 只看健康检查 OK，业务路径已经坏了

所以部署流水线应该变成：

```text
部署 → 等待稳定 → 业务探针 → 对比基线 → 自动判定 → 晋级 / 回滚 / 升级人工
```

这就是 **Post-Deploy Verification & Auto-Rollback**。

---

## 1. 把“验证”设计成可执行契约

不要只写一句“部署后检查一下”。要把验证目标变成结构化契约：

```json
{
  "releaseId": "agent-2026-05-10-0630",
  "environment": "prod",
  "candidateSha": "abc123",
  "previousSha": "def456",
  "stabilizationSeconds": 90,
  "probes": [
    { "name": "health", "type": "http", "url": "/health", "slo": { "status": 200, "p95Ms": 300 } },
    { "name": "agent_turn", "type": "synthetic", "scenario": "simple_question", "slo": { "successRate": 0.99 } },
    { "name": "tool_dispatch", "type": "synthetic", "scenario": "readonly_tool", "slo": { "successRate": 0.98 } }
  ],
  "rollback": {
    "strategy": "redeploy_previous_sha",
    "maxRollbackSeconds": 180,
    "requiresApproval": false
  }
}
```

关键点：

- `health` 只能证明进程活着，不能证明 Agent 可用
- `synthetic scenario` 要覆盖真实业务关键路径
- `previousSha` 必须在部署前记录，否则回滚时找不到安全版本
- 验证结果要产出证据：probe 输出、时间、失败原因、rollback id

---

## 2. learn-claude-code：Python 教学版

先写一个最小可测的验证器。它不关心底层是 K8s、Fly、Laravel Cloud 还是 GitHub Actions，只关心探针结果是否满足 SLO。

```python
from dataclasses import dataclass
from typing import Callable
import time

@dataclass
class ProbeResult:
    name: str
    ok: bool
    latency_ms: int
    reason: str | None = None

@dataclass
class VerificationDecision:
    outcome: str  # promote / rollback / manual_review
    reason: str
    failed_probes: list[str]

class PostDeployVerifier:
    def __init__(self, probes: list[Callable[[], ProbeResult]]):
        self.probes = probes

    def verify(self, stabilization_seconds: int = 30) -> VerificationDecision:
        time.sleep(stabilization_seconds)

        results = [probe() for probe in self.probes]
        failed = [r.name for r in results if not r.ok]

        if not failed:
            return VerificationDecision("promote", "all_probes_passed", [])

        critical_failed = [
            r.name for r in results
            if not r.ok and r.name in {"agent_turn", "tool_dispatch"}
        ]
        if critical_failed:
            return VerificationDecision("rollback", "critical_probe_failed", critical_failed)

        return VerificationDecision("manual_review", "non_critical_probe_failed", failed)


def rollback_to_previous(previous_sha: str) -> str:
    # 教学版只返回动作；生产版这里会调用 deploy provider
    return f"rollback_requested:{previous_sha}"
```

把它接到部署流程：

```python
def deploy_with_verification(candidate_sha: str, previous_sha: str, verifier: PostDeployVerifier):
    deploy_id = deploy(candidate_sha)
    decision = verifier.verify(stabilization_seconds=90)

    if decision.outcome == "promote":
        mark_release_good(deploy_id)
        return {"status": "promoted", "deployId": deploy_id}

    if decision.outcome == "rollback":
        rollback_id = rollback_to_previous(previous_sha)
        return {
            "status": "rolled_back",
            "deployId": deploy_id,
            "rollbackId": rollback_id,
            "failedProbes": decision.failed_probes,
        }

    create_incident_review(deploy_id, decision.failed_probes)
    return {"status": "manual_review", "deployId": deploy_id}
```

这段代码的重点不是“怎么调用云厂商”，而是把发布后的判断变成**确定性状态机**。

---

## 3. pi-mono：TypeScript 生产版 Release Controller

生产系统里建议把部署拆成三个接口：

- `DeployProvider`：执行发布 / 回滚
- `ProbeRunner`：跑验证探针
- `ReleaseController`：按策略决策

```ts
type ProbeResult = {
  name: string;
  ok: boolean;
  latencyMs: number;
  reason?: string;
  evidence?: unknown;
};

type ReleasePlan = {
  releaseId: string;
  env: "staging" | "prod";
  candidateSha: string;
  previousSha: string;
  stabilizationMs: number;
  rollback: { enabled: boolean; requiresApproval: boolean };
};

interface DeployProvider {
  deploy(sha: string): Promise<{ deploymentId: string }>;
  rollback(toSha: string): Promise<{ rollbackId: string }>;
}

interface ProbeRunner {
  run(plan: ReleasePlan): Promise<ProbeResult[]>;
}

class ReleaseController {
  constructor(
    private deployProvider: DeployProvider,
    private probeRunner: ProbeRunner,
    private audit: { append(event: unknown): Promise<void> },
  ) {}

  async release(plan: ReleasePlan) {
    await this.audit.append({ type: "release_started", plan });

    const deployment = await this.deployProvider.deploy(plan.candidateSha);
    await sleep(plan.stabilizationMs);

    const probes = await this.probeRunner.run(plan);
    const failed = probes.filter((p) => !p.ok);

    await this.audit.append({
      type: "post_deploy_verification_finished",
      releaseId: plan.releaseId,
      deploymentId: deployment.deploymentId,
      probes,
    });

    if (failed.length === 0) {
      await this.audit.append({ type: "release_promoted", releaseId: plan.releaseId });
      return { outcome: "promoted", deployment, probes } as const;
    }

    const criticalFailed = failed.some((p) =>
      ["agent_turn", "tool_dispatch", "approval_flow"].includes(p.name),
    );

    if (criticalFailed && plan.rollback.enabled && !plan.rollback.requiresApproval) {
      const rollback = await this.deployProvider.rollback(plan.previousSha);
      await this.audit.append({
        type: "release_rolled_back",
        releaseId: plan.releaseId,
        rollback,
        failed,
      });
      return { outcome: "rolled_back", deployment, rollback, failed } as const;
    }

    await this.audit.append({ type: "release_needs_manual_review", releaseId: plan.releaseId, failed });
    return { outcome: "manual_review", deployment, failed } as const;
  }
}
```

生产细节：

- 回滚也必须写审计日志，不能悄悄发生
- 自动回滚只适合低争议、可逆、已验证 previousSha 的发布
- 高风险数据迁移失败不要自动回滚数据库，应该进入人工 runbook
- 所有探针都要带 `evidence`，方便复盘

---

## 4. OpenClaw 实战：课程 Cron 也可以套这个模式

OpenClaw 的“部署”不一定是上线服务，也可能是一次外部副作用：

- 发 Telegram 课程
- 写 lesson 文件
- 更新 README / TOOLS.md
- git commit + push

这类任务也能做“发布后验证”：

```ts
async function verifyCourseRelease(input: {
  lessonPath: string;
  readmePath: string;
  topic: string;
  commitSha: string;
  telegramMessageId?: string;
}) {
  const gates = [
    await fileExists(input.lessonPath),
    await readmeContains(input.readmePath, input.lessonPath),
    await gitCommitExists(input.commitSha),
    Boolean(input.telegramMessageId),
  ];

  if (gates.every(Boolean)) {
    return { outcome: "promoted", reason: "all_completion_gates_passed" };
  }

  // 课程发布这种副作用不适合删除消息自动回滚；更适合补偿修复
  return {
    outcome: "compensate",
    reason: "course_release_incomplete",
    missing: gates.map((ok, i) => (!ok ? i : null)).filter((x) => x !== null),
  };
}
```

这里的重点是：**不是所有失败都叫 rollback**。

- 服务部署失败：可以回滚 previousSha
- 消息发错群：不能假装没发生，只能补发/更正
- DB migration 失败：先冻结写入，再人工恢复
- 文件漏更新：补偿提交即可

成熟 Agent 要先判断副作用类型，再决定 `rollback / compensate / manual_review`。

---

## 5. 常见坑

1. **只查 `/health`**  
   `/health` 绿了，Agent Loop、工具、审批、记忆都可能坏。

2. **没有 previousSha**  
   等出事再找“上一个好版本”，已经晚了。

3. **探针太重**  
   发布后验证不是全量 E2E，应该是少量关键路径合成请求。

4. **自动回滚所有失败**  
   数据迁移、外部消息、支付回调这类副作用可能不能安全回滚。

5. **回滚后不再验证**  
   回滚也是一次部署，也要跑 post-rollback verification。

---

## 6. Checklist

- [ ] 部署前记录 `candidateSha` 和 `previousKnownGoodSha`
- [ ] 每个环境定义最小业务探针集合
- [ ] 探针输出结构化 evidence，而不是只打印日志
- [ ] 自动回滚只用于可逆发布
- [ ] 不可逆副作用使用补偿动作和人工 runbook
- [ ] 回滚后再次验证
- [ ] 发布、验证、回滚全链路写入审计日志

---

## 总结

Agent 自动化的发布闭环不是：

```text
deploy 完成 = 成功
```

而是：

```text
deploy 完成 + 关键路径验证通过 + 可审计证据完整 = 成功
```

让 Agent 负责部署可以提高速度；让 Agent 负责部署后验证和安全回滚，才是真的进入生产级自动化。
