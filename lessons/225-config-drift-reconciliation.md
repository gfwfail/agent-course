# 225. Agent 配置漂移检测与自动修复（Configuration Drift Detection & Reconciliation）

> 关键词：configuration drift、desired state、reconcile loop、safe remediation、GitOps

上一课讲了 **Runbook 自动化与安全执行**：Agent 可以按 SOP 处理运维，但必须先诊断、再计划、再审批、再验证。

今天讲一个和运维 Agent 强相关的能力：**配置漂移检测与自动修复**。

生产系统最常见的问题之一不是“没人配置”，而是：

```txt
Git 里写的是 A
Terraform 里写的是 A
K8s / SaaS 控制台里实际变成了 B
告警来了，大家才发现有人手动改过
```

这就叫 **Configuration Drift（配置漂移）**。

传统做法是定时脚本对比配置；Agent 的价值是：它不只是发现 diff，还能理解 diff 的风险、解释原因、生成修复计划，并在安全边界内自动 reconcile。

一句话：**Agent 不应该只会“执行命令”，还要能持续判断“现实世界是否还符合期望状态”。**

---

## 1. 为什么 Agent 需要 Drift Detection？

很多团队会把配置写在 Git：

- 模型路由配置；
- 工具权限配置；
- cron / heartbeat 调度；
- webhook endpoint；
- K8s replicas / env vars；
- SaaS 平台开关；
- Prompt / Skill 版本。

但实际运行态可能被这些事情改变：

1. 人在控制台手动 hotfix；
2. 旧版本 Agent 写回了旧配置；
3. 部署失败只更新了一半；
4. 多实例并发写入互相覆盖；
5. 紧急开关打开后没人关；
6. 外部平台默认值变化。

如果 Agent 只在用户问时才检查，就太晚了。

Drift Agent 的目标是：

```txt
定期读取 desired state
→ 拉取 actual state
→ 归一化并对比
→ 分类风险
→ 生成 remediation plan
→ 安全范围内自动修复
→ 高风险变更升级人工
→ 写入 evidence / audit log
```

这其实就是 Kubernetes controller 的 reconcile loop 思想，只是对象从 Pod 扩展到 Agent 配置、工具权限、Prompt、外部 SaaS 等所有“期望状态”。

---

## 2. 核心模型：Desired State vs Actual State

不要让 LLM 直接比较两大段 JSON。

正确做法是先把状态标准化：

```ts
type ConfigSnapshot = {
  resourceId: string;
  resourceType: "model_route" | "tool_policy" | "cron" | "webhook";
  version: string;
  fields: Record<string, unknown>;
  ignoredFields?: string[];
  source: "git" | "runtime" | "api";
  fetchedAt: string;
};
```

然后 diff 结果也结构化：

```ts
type Drift = {
  path: string;
  desired: unknown;
  actual: unknown;
  severity: "info" | "warning" | "danger";
  autoFixable: boolean;
  reason: string;
};
```

这里有个关键点：**不是所有 diff 都是漂移**。

例如：

- `lastSeenAt`、`updatedAt` 是运行时字段，应忽略；
- 平台自动补全的默认值，不一定要修；
- 人工临时 override 可能有 TTL，过期前不算异常；
- secret 值不能直接比较明文，只能比较 hash / version / presence。

所以 Drift Detection 必须有 **normalizer + ignore rules + severity rules**，不能只是 `JSON.stringify(a) !== JSON.stringify(b)`。

---

## 3. learn-claude-code：Python 教学版

教学版先做一个小型 reconcile loop：读取 Git 里的 desired JSON，调用工具拿 actual JSON，对比后输出计划。

```python
# learn-claude-code/config_drift.py
from dataclasses import dataclass
from typing import Any, Literal

Severity = Literal["info", "warning", "danger"]


@dataclass
class Drift:
    path: str
    desired: Any
    actual: Any
    severity: Severity
    auto_fixable: bool
    reason: str


IGNORE_PATHS = {
    "updatedAt",
    "lastSeenAt",
    "runtime.metrics",
}

DANGER_PATHS = {
    "toolPolicy.allowDestructive",
    "auth.allowedUsers",
    "modelRoute.fallbackModel",
}


def get_path(obj: dict[str, Any], path: str) -> Any:
    cur: Any = obj
    for part in path.split("."):
        if not isinstance(cur, dict):
            return None
        cur = cur.get(part)
    return cur


def flatten(obj: dict[str, Any], prefix: str = "") -> dict[str, Any]:
    out: dict[str, Any] = {}
    for key, value in obj.items():
        path = f"{prefix}.{key}" if prefix else key
        if isinstance(value, dict):
            out.update(flatten(value, path))
        else:
            out[path] = value
    return out


def detect_drift(desired: dict[str, Any], actual: dict[str, Any]) -> list[Drift]:
    desired_flat = flatten(desired)
    actual_flat = flatten(actual)
    all_paths = sorted(set(desired_flat) | set(actual_flat))

    drifts: list[Drift] = []
    for path in all_paths:
        if path in IGNORE_PATHS:
            continue

        desired_value = desired_flat.get(path)
        actual_value = actual_flat.get(path)
        if desired_value == actual_value:
            continue

        severity: Severity = "warning"
        auto_fixable = True
        reason = "actual state differs from desired state"

        if path in DANGER_PATHS:
            severity = "danger"
            auto_fixable = False
            reason = "security or model behavior sensitive field changed"

        drifts.append(
            Drift(
                path=path,
                desired=desired_value,
                actual=actual_value,
                severity=severity,
                auto_fixable=auto_fixable,
                reason=reason,
            )
        )

    return drifts
```

再加一个 reconcile 执行器：

```python
# learn-claude-code/reconcile.py
class ConfigReconciler:
    def __init__(self, tools, approval_gate, audit_log):
        self.tools = tools
        self.approval_gate = approval_gate
        self.audit_log = audit_log

    async def run(self, resource_id: str):
        desired = await self.tools.call("read_desired_config", {"id": resource_id})
        actual = await self.tools.call("read_actual_config", {"id": resource_id})
        drifts = detect_drift(desired, actual)

        if not drifts:
            return {"status": "clean", "resourceId": resource_id}

        auto = [d for d in drifts if d.auto_fixable]
        manual = [d for d in drifts if not d.auto_fixable]

        if manual:
            await self.approval_gate.escalate(
                title="Dangerous config drift detected",
                resource_id=resource_id,
                drifts=[d.__dict__ for d in manual],
            )

        for drift in auto:
            await self.tools.call(
                "patch_actual_config",
                {
                    "id": resource_id,
                    "path": drift.path,
                    "value": drift.desired,
                    "reason": drift.reason,
                },
            )
            await self.audit_log.write(
                "config_drift_fixed",
                {"resourceId": resource_id, "drift": drift.__dict__},
            )

        verify = await self.tools.call("read_actual_config", {"id": resource_id})
        remaining = detect_drift(desired, verify)
        return {
            "status": "fixed" if not remaining else "partial",
            "fixed": len(auto),
            "manual": len(manual),
            "remaining": [d.__dict__ for d in remaining],
        }
```

这个教学版的重点是边界清楚：

- read desired / actual 是只读；
- diff 是确定性代码，不靠 LLM 猜；
- danger 字段只升级，不自动改；
- patch 后必须 verify；
- audit log 永远记录修了什么、为什么修。

---

## 4. pi-mono：TypeScript 生产版

生产版可以把 Drift Detection 做成一个独立服务：

```ts
// pi-mono/packages/agent-runtime/src/drift/types.ts
export type Severity = "info" | "warning" | "danger";

export interface ResourceRef {
  type: "tool_policy" | "model_route" | "cron" | "webhook";
  id: string;
}

export interface Snapshot {
  ref: ResourceRef;
  source: "desired" | "actual";
  version?: string;
  data: Record<string, unknown>;
  fetchedAt: string;
}

export interface DriftRule {
  path: string;
  ignore?: boolean;
  severity?: Severity;
  autoFixable?: boolean;
  normalize?: (value: unknown) => unknown;
}

export interface DriftFinding {
  ref: ResourceRef;
  path: string;
  desired: unknown;
  actual: unknown;
  severity: Severity;
  autoFixable: boolean;
  reason: string;
}
```

Detector 不依赖具体平台：

```ts
// drift-detector.ts
export class DriftDetector {
  constructor(private readonly rules: DriftRule[]) {}

  detect(desired: Snapshot, actual: Snapshot): DriftFinding[] {
    const desiredFlat = flatten(desired.data);
    const actualFlat = flatten(actual.data);
    const paths = new Set([...Object.keys(desiredFlat), ...Object.keys(actualFlat)]);
    const findings: DriftFinding[] = [];

    for (const path of [...paths].sort()) {
      const rule = this.matchRule(path);
      if (rule?.ignore) continue;

      const normalize = rule?.normalize ?? ((v: unknown) => v);
      const expected = normalize(desiredFlat[path]);
      const observed = normalize(actualFlat[path]);

      if (deepEqual(expected, observed)) continue;

      findings.push({
        ref: desired.ref,
        path,
        desired: redactIfSensitive(path, expected),
        actual: redactIfSensitive(path, observed),
        severity: rule?.severity ?? "warning",
        autoFixable: rule?.autoFixable ?? true,
        reason: `actual ${path} does not match desired state`,
      });
    }

    return findings;
  }

  private matchRule(path: string): DriftRule | undefined {
    return this.rules.find((rule) => path === rule.path || path.startsWith(`${rule.path}.`));
  }
}
```

Reconciler 负责策略，不负责 diff 细节：

```ts
// reconciler.ts
export class ConfigReconciler {
  constructor(
    private readonly desiredStore: DesiredStateStore,
    private readonly actualStore: ActualStateStore,
    private readonly detector: DriftDetector,
    private readonly policy: DriftPolicy,
    private readonly audit: AuditLog,
  ) {}

  async reconcile(ref: ResourceRef): Promise<ReconcileResult> {
    const desired = await this.desiredStore.get(ref);
    const actual = await this.actualStore.get(ref);
    const findings = this.detector.detect(desired, actual);

    if (findings.length === 0) {
      return { status: "clean", ref, findings: [] };
    }

    const decision = await this.policy.decide({ ref, findings });

    if (decision.action === "escalate") {
      await this.audit.append("drift.escalated", { ref, findings, reason: decision.reason });
      return { status: "needs_approval", ref, findings };
    }

    for (const patch of decision.patches) {
      await this.actualStore.patch(ref, patch, {
        idempotencyKey: `reconcile:${ref.type}:${ref.id}:${patch.path}`,
      });
      await this.audit.append("drift.patch_applied", { ref, patch });
    }

    const verifyActual = await this.actualStore.get(ref);
    const remaining = this.detector.detect(desired, verifyActual);

    return {
      status: remaining.length === 0 ? "fixed" : "partial",
      ref,
      findings,
      remaining,
    };
  }
}
```

Policy 示例：

```ts
export class DriftPolicy {
  async decide(input: { ref: ResourceRef; findings: DriftFinding[] }) {
    const dangerous = input.findings.filter((f) => f.severity === "danger" || !f.autoFixable);

    if (dangerous.length > 0) {
      return {
        action: "escalate" as const,
        reason: "dangerous or non-auto-fixable drift detected",
      };
    }

    return {
      action: "patch" as const,
      patches: input.findings.map((f) => ({ path: f.path, value: f.desired })),
    };
  }
}
```

生产版要特别注意三点：

1. **敏感字段脱敏**：日志里不能出现 secret 明文；
2. **幂等 patch**：同一 drift 重试不能重复制造副作用；
3. **verify after write**：修复后必须重新读 actual，而不是相信 patch 返回成功。

---

## 5. OpenClaw：Heartbeat / Cron 实战

OpenClaw 里这类任务非常适合放在 Heartbeat 或 Cron：

```txt
每 30 分钟：
1. 读取 workspace/config/desired/*.json
2. 查询 gateway / node / SaaS 实际配置
3. 生成 drift report
4. info/warning 自动写 memory/drift-state.json
5. danger 级别通知老板确认
6. autoFixable 的小漂移自动修，并写审计日志
```

一个轻量文件状态可以这样设计：

```json
{
  "lastRunAt": "2026-05-03T06:30:00+11:00",
  "resources": {
    "tool_policy:telegram-main": {
      "status": "clean",
      "lastCleanAt": "2026-05-03T06:30:00+11:00"
    },
    "cron:agent-course": {
      "status": "fixed",
      "lastFinding": "schedule.enabled drifted false -> true"
    }
  }
}
```

通知策略不要太吵：

- `info`：只记日志；
- `warning + autoFixable`：自动修复，摘要里说；
- `warning + non-autoFixable`：进待办，不立即打断；
- `danger`：立即通知人工确认；
- 同一个 drift 用 hash 去重，避免每 30 分钟刷屏。

这和第 221 课“通知路由与噪声预算”可以组合起来。

---

## 6. 常见坑

### 坑 1：把运行时字段也当漂移

`updatedAt`、`lastSeenAt`、`metrics.*` 这种字段天生会变化，必须 ignore。

### 坑 2：自动修复权限字段

像 `allowedUsers`、`allowDestructive`、`webhookUrl`、`modelRoute.primary` 这类字段影响安全或成本，最好默认升级人工。

### 坑 3：只 diff，不解释

Agent 不是 `diff` 命令。它应该输出：

```txt
发现 tool_policy.allowExternal 从 false 变成 true。
这是 danger，因为它扩大了外部写入权限。
建议：不要自动回滚，先确认是否为临时授权。
```

### 坑 4：修完不验证

外部 API 可能接受 patch 但异步生效；也可能被另一个控制器马上改回去。必须 verify。

### 坑 5：没有 owner

每类配置要有 owner：谁能批准？谁能忽略？谁能改 desired state？否则 Agent 发现问题也不知道找谁。

---

## 7. 设计 Checklist

做 Drift Agent 时，至少检查这 8 项：

- [ ] desired state 是否有版本和来源？
- [ ] actual state 是否来自权威 API，而不是缓存？
- [ ] 是否有 normalizer，避免格式差异误报？
- [ ] 是否有 ignore rules，跳过运行时字段？
- [ ] 是否有 severity rules，把安全/成本字段标为 danger？
- [ ] auto-fix 是否幂等？
- [ ] patch 后是否重新读取 verify？
- [ ] 通知是否去重，避免噪音？

---

## 8. 总结

配置漂移检测的本质不是“发现 JSON 不一样”，而是：

> **持续把现实世界拉回期望状态，同时知道什么时候不能自动拉。**

传统脚本只能看到差异；Agent 可以理解差异的风险、上下文和修复路径。

但边界必须清楚：

- diff 用确定性代码；
- LLM 负责解释、归因、生成建议；
- policy 决定是否自动修；
- 高风险字段走人工确认；
- 所有修复必须 audit + verify。

好的 Agent 不是永远不出错，而是能持续发现系统偏离轨道，并用安全、可审计的方式把它拉回来。
