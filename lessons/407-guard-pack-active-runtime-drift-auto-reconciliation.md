# 407. Agent 回归护栏包激活后的运行时漂移监控与自动重对齐（Guard Pack Active Runtime Drift & Auto-Reconciliation）

上一课讲了 **Guard Pack Active Promotion & LKG Consistency Gate**：canary 100% 通过以后，仍然要证明所有 runtime group 一致加载、risk surface 样本或豁免完整、rollback probe 可用、freeze window 稳定，最后才能用 compare-and-swap 原子更新 Last Known Good。

今天继续往后走：**LKG 更新成功，也不代表 Guard Pack 会永远保持一致。**

一句话：**Active Guard Pack 需要持续监控 runtime drift，用版本、schema、executor、decision fingerprint 和探针结果判断是否跑偏，再自动 refresh、quarantine、rollback 或升级人工复核；成熟 Agent 不只会把护栏发布成功，还会持续证明生产里的每个执行者仍然在执行同一份安全契约。**

---

## 1. 为什么 active 后还会漂移

很多团队以为：

~~~text
LKG = v43
active = v43
所有事情结束
~~~

这对普通配置可能勉强够用。对 Agent Guard Pack 不够，因为 Agent runtime 是分散的：

- interactive session 可能常驻几小时，内存里还拿着旧 policy snapshot；
- cron worker 是新进程，但 subagent worker 可能复用了旧 executor；
- tool schema 更新了，guard pack 里的 action matcher 还按旧字段判断；
- prompt pack、memory injector、retrieval index 可能各自缓存不同版本；
- message egress、git executor、deployment executor 可能由不同进程解释同一条 guard；
- rollback 后又 forward-fix，某些 runtime 可能跳过了中间的 reload receipt。

所以 active 后要监控的不是“Guard 有没有挡住坏动作”，那是上一阶段的 effectiveness telemetry；这里要监控的是：

> 同一份 active Guard Pack，在所有 runtime 里是否仍然被同一种方式解释和执行。

这叫 **Active Runtime Drift Monitor**。

---

## 2. 漂移不是只有版本号不一致

最浅层的 drift 是版本不一致：

~~~json
{
  "runtimeId": "cron-worker-03",
  "activeGuardPackVersion": 42,
  "lkgVersion": 43
}
~~~

但更隐蔽的是：版本号一样，解释结果不一样。

比如两个 runtime 都声称使用 v43：

~~~text
runtime A: git_push + main + no_pr -> block
runtime B: git_push + main + no_pr -> warn
~~~

这可能来自：

- executor build hash 不同；
- schema migrator 版本不同；
- action classifier 依赖的 tool name 变了；
- policy cache 中 risk surface 映射过期；
- prompt pack 注入的安全规则不是同一份；
- runtime 使用了不同的 fail-open / fail-closed 默认值。

所以 drift monitor 至少要记录五个 fingerprint：

~~~ts
type GuardRuntimeFingerprint = {
  runtimeId: string;
  runtimeGroup: RuntimeGroup;
  guardPackVersion: number;
  guardPackHash: string;
  executorBuildHash: string;
  schemaVersion: string;
  policySnapshotHash: string;
  toolSchemaHash: string;
  promptPackHash: string;
  loadedAt: string;
};
~~~

版本号只是入口，hash 和 probe decision 才能证明“同一份契约被同样执行”。

---

## 3. 最小模型：DriftProbe

Runtime drift monitor 可以定期向每组 runtime 发送只读 probe。

~~~ts
type DriftProbe = {
  probeId: string;
  guardPackVersion: number;
  riskSurface: RiskSurface;
  action: string;
  fixture: Record<string, unknown>;
  expectedDecision: "allow" | "warn" | "dry_run" | "block";
  expectedReasonCode: string;
};

type DriftProbeResult = {
  probeId: string;
  runtimeId: string;
  observedDecision: "allow" | "warn" | "dry_run" | "block";
  observedReasonCode: string;
  fingerprint: GuardRuntimeFingerprint;
  latencyMs: number;
  evaluatedAt: string;
};
~~~

注意：probe 必须是只读的，不允许真的发消息、push git、部署或写 memory。

生产里常见 probe：

- message_egress：私密证据是否会被挡在群聊外；
- git_push：main 分支直接 push 是否被 block；
- credential_access：低权限上下文读 secret 是否被 block；
- deployment：未通过 rollback probe 的发布是否被 hold；
- memory_write：group chat 是否禁止写长期私人记忆。

这些 probe 不只是测试 guard 本身，也测试 runtime 是否加载了正确 schema、tool metadata 和 prompt policy。

---

## 4. learn-claude-code：纯函数漂移判定器

教学版先做一个纯函数。输入 expected probe、runtime result 和预算，输出 keep、refresh_runtime、quarantine_runtime、rollback_lkg 或 manual_review。

~~~py
from dataclasses import dataclass
from typing import Literal


Decision = Literal[
    "keep_active",
    "refresh_runtime",
    "quarantine_runtime",
    "rollback_lkg",
    "manual_review",
]


@dataclass(frozen=True)
class Fingerprint:
    runtime_id: str
    runtime_group: str
    guard_pack_version: int
    guard_pack_hash: str
    executor_build_hash: str
    schema_version: str
    policy_snapshot_hash: str
    tool_schema_hash: str
    prompt_pack_hash: str


@dataclass(frozen=True)
class DriftProbeResult:
    expected_decision: str
    observed_decision: str
    expected_reason_code: str
    observed_reason_code: str
    fingerprint: Fingerprint
    latency_ms: int


@dataclass(frozen=True)
class DriftBudget:
    lkg_version: int
    lkg_hash: str
    allowed_executor_hashes: set[str]
    allowed_schema_versions: set[str]
    max_latency_ms: int
    fail_closed_surfaces: set[str]


def classify_drift(
    result: DriftProbeResult,
    budget: DriftBudget,
    risk_surface: str,
) -> dict:
    fp = result.fingerprint
    failures = []

    if fp.guard_pack_version != budget.lkg_version:
        failures.append("guard_pack_version_skew")
    if fp.guard_pack_hash != budget.lkg_hash:
        failures.append("guard_pack_hash_mismatch")
    if fp.executor_build_hash not in budget.allowed_executor_hashes:
        failures.append("executor_build_drift")
    if fp.schema_version not in budget.allowed_schema_versions:
        failures.append("schema_version_drift")
    if result.observed_decision != result.expected_decision:
        failures.append("decision_mismatch")
    if result.observed_reason_code != result.expected_reason_code:
        failures.append("reason_code_mismatch")
    if result.latency_ms > budget.max_latency_ms:
        failures.append("latency_budget_exceeded")

    if not failures:
        return {"decision": "keep_active", "failures": []}

    unsafe = (
        "decision_mismatch" in failures
        and risk_surface in budget.fail_closed_surfaces
        and result.observed_decision in {"allow", "warn"}
    )

    if unsafe:
        return {
            "decision": "quarantine_runtime",
            "reason": "fail-closed surface produced a weaker decision than expected",
            "failures": failures,
            "runtimeId": fp.runtime_id,
        }

    if "guard_pack_hash_mismatch" in failures:
        return {
            "decision": "rollback_lkg",
            "reason": "runtime claims active version but hash differs from LKG",
            "failures": failures,
        }

    if {"executor_build_drift", "schema_version_drift"} & set(failures):
        return {
            "decision": "refresh_runtime",
            "reason": "runtime can be reloaded before changing LKG",
            "failures": failures,
            "runtimeId": fp.runtime_id,
        }

    return {
        "decision": "manual_review",
        "reason": "drift needs human classification",
        "failures": failures,
    }
~~~

这里有个重要边界：**runtime drift 优先修 runtime，不要马上改 Guard Pack。**

如果 LKG 没错，只是某个 worker 没 reload，正确动作是 refresh 或 quarantine 这个 runtime；只有 LKG 指针、hash 或全局解释都错了，才考虑 rollback。

---

## 5. pi-mono：用 Agent.subscribe 把 drift 变成事件流

pi-mono 的 packages/agent/src/agent.ts 提供了 Agent.subscribe(fn)，packages/mom/src/agent.ts 里已经用它监听 tool_execution_start / tool_execution_end，记录工具名称、参数、结果和耗时。

这类事件流适合扩展成 Guard Drift Monitor：

~~~ts
type GuardDecisionObserved = {
  type: "guard.decision.observed";
  runId: string;
  runtimeId: string;
  runtimeGroup: RuntimeGroup;
  riskSurface: RiskSurface;
  action: string;
  decision: "allow" | "warn" | "dry_run" | "block";
  reasonCode: string;
  fingerprint: GuardRuntimeFingerprint;
  latencyMs: number;
};

class GuardRuntimeDriftMonitor {
  constructor(
    private readonly registry: RuntimeRegistry,
    private readonly probeStore: DriftProbeStore,
    private readonly controller: DriftReconciliationController,
  ) {}

  attach(agent: Agent, runtime: RuntimeIdentity) {
    agent.subscribe(async (event) => {
      if (event.type !== "tool_execution_start") return;

      const probe = await this.probeStore.matchReadonlyProbe({
        toolName: event.toolName,
        args: event.args,
        runtimeGroup: runtime.group,
      });

      if (!probe) return;

      const result = await this.registry.evaluateGuardReadonly({
        runtimeId: runtime.id,
        probe,
      });

      await this.controller.reconcile(result);
    });
  }
}
~~~

这里重点不是把所有工具都拦住，而是抽样跑只读 probe：

1. 真实运行事件提供 runtime 活跃信号；
2. monitor 按 runtime group 匹配适合的 probe；
3. guard executor 用 readonly fixture 计算 decision；
4. controller 对比 expected decision 和 fingerprint；
5. 只对异常 runtime 执行 refresh / quarantine / rollback。

这样做比“每分钟扫全量 runtime”更省成本，也更接近真实流量。

---

## 6. 自动重对齐：不要把 reload 做成盲重启

发现 drift 后，常见错误是直接重启全部 worker。

更好的方式是生成 **Reconciliation Plan**：

~~~ts
type ReconciliationPlan = {
  planId: string;
  driftKind:
    | "version_skew"
    | "hash_mismatch"
    | "executor_drift"
    | "schema_drift"
    | "decision_mismatch"
    | "latency_drift";
  affectedRuntimeIds: string[];
  action:
    | "refresh_policy_cache"
    | "reload_executor"
    | "quarantine_runtime"
    | "rollback_lkg"
    | "manual_review";
  preconditions: string[];
  postChecks: DriftProbe[];
  receiptRequired: boolean;
};
~~~

执行时要有顺序：

~~~text
detect drift
  -> classify blast radius
  -> choose minimal reconciliation action
  -> apply action with fence token
  -> rerun readonly probes
  -> write reconciliation receipt
  -> release or keep quarantine
~~~

如果只是 policySnapshotHash 过期，刷新 cache 就够了。

如果是 executorBuildHash 不在 allowlist，需要 reload executor。

如果是 fail-closed risk surface 出现 allow，先 quarantine runtime，再查根因。

如果多个 runtime group 的 guardPackHash 都和 LKG 不一致，才进入 rollback LKG 或 incident forensics。

---

## 7. OpenClaw 课程 Cron 实战类比

拿这个课程 cron 举例，每 3 小时会做三类外部副作用：

1. 发 Telegram 群消息；
2. 写 lessons/XXX-topic.md、README.md、TOOLS.md；
3. git commit && git push 到课程仓库。

如果我们给它加 Active Drift Monitor，可以设计这些 probe：

~~~json
[
  {
    "riskSurface": "message_egress",
    "fixture": {
      "target": "-5115329245",
      "containsSecret": true
    },
    "expectedDecision": "block",
    "expectedReasonCode": "secret_to_group_chat"
  },
  {
    "riskSurface": "git_push",
    "fixture": {
      "branch": "main",
      "repo": "gfwfail/agent-course",
      "ghUser": "metaclaw01"
    },
    "expectedDecision": "block",
    "expectedReasonCode": "wrong_github_identity"
  },
  {
    "riskSurface": "memory_write",
    "fixture": {
      "file": "MEMORY.md",
      "context": "group_chat"
    },
    "expectedDecision": "block",
    "expectedReasonCode": "private_memory_in_shared_context"
  }
]
~~~

这不是为了每次真的泄密、真的 push，而是在副作用前用只读 fixture 验证：

- 当前 runtime 知道 Telegram 群是外部公开-ish 场景；
- 当前 git identity 是 gfwfail；
- 当前课程编号、README、TOOLS.md 更新策略一致；
- 当前 session 没把私人长期记忆带进群聊输出。

如果 probe 发现 git_push 规则在某个 runtime 里从 block 变成 warn，应该先 quarantine 那个 runtime，不要让它继续执行真实 push。

---

## 8. 常见坑

**坑 1：只看 guardPackVersion。**

版本号一致不代表执行一致。必须看 hash、executor build、schema、tool schema、prompt pack 和 probe decision。

**坑 2：把 drift 当成 guard 效果问题。**

效果问题问的是“这个 guard 是否该存在”；drift 问的是“同一个 guard 是否被同样执行”。两个问题不要混在一个自动修复器里。

**坑 3：发现漂移就全局 rollback。**

如果只有一个 cron worker 过期，全局 rollback 会把健康 runtime 也拖回旧版本。先局部 refresh / quarantine。

**坑 4：probe 有副作用。**

Drift probe 必须 readonly。任何会发消息、写文件、push、部署、扣费的 probe 都应该变成 dry-run fixture。

**坑 5：没有 reconciliation receipt。**

自动刷新 cache、reload worker、quarantine runtime 都是安全动作，也要留收据。否则下次事故不知道谁在什么时候改变了运行时状态。

---

## 9. 落地清单

给你的 Agent 系统加 Active Runtime Drift Monitor，可以从 6 件事开始：

1. 每个 runtime 上报 GuardRuntimeFingerprint；
2. 每个高风险面准备 2-3 个 readonly drift probe；
3. probe 输出 expected / observed decision 和 reason code；
4. drift 判定器区分 version、hash、executor、schema、decision、latency；
5. reconciliation plan 优先局部 refresh / quarantine，再考虑 rollback；
6. 每次自动重对齐都写 receipt，并 rerun probe 验证。

这套机制补上了 Guard Pack 发布链路的最后一块：**发布成功只是一个时点，持续一致才是生产安全。**

---

## 10. 总结

这一课的核心：

1. LKG promotion 成功后，runtime 仍可能因为缓存、热更新、schema 和 executor 差异产生漂移；
2. Drift Monitor 要看版本、hash、executor、schema、tool schema、prompt pack 和 probe decision；
3. learn-claude-code 里可以先做纯函数 drift classifier；
4. pi-mono 可以用 Agent.subscribe 的事件流触发 readonly probe；
5. OpenClaw 课程 cron 可以把 Telegram、Git、memory 写入都做成副作用前的 drift probe；
6. 修 drift 要优先局部 refresh / quarantine，并用 receipt 证明重对齐完成。

成熟 Agent 的护栏不是“上线那一刻正确”，而是上线之后每个 runtime 都持续证明：我仍然在执行同一份安全契约。
