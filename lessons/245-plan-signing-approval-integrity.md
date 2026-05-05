# 第 245 课：Agent 执行计划签名与审批防篡改（Plan Signing & Approval Integrity）

> 核心思想：人工审批不是“弹个按钮”就安全。真正安全的审批要绑定**具体计划、参数、风险、时间和执行主体**。审批通过后，Agent 执行前必须校验计划 hash；只要计划或参数变了，就必须重新审批。

---

## 1. 为什么普通审批还不够？

很多 Agent 系统会做 Human-in-the-loop：危险操作前问一句“是否确认？”。这比完全自动化安全，但仍然有几个坑：

- 审批时展示的是摘要，执行时参数已经变了；
- 用户批准了 `deploy staging`，Agent 最后执行成了 `deploy production`；
- 子 Agent 重新规划后复用了旧 approval；
- 审批消息没有记录风险、证据和过期时间；
- 计划被压缩/重写后，没人知道审批对应的是哪个版本。

所以审批必须从“确认意图”升级为“确认不可变计划”。

一句话：**审批批准的不是一句话，而是一份带 hash 的执行计划。**

---

## 2. ApprovalRecord：审批应该绑定什么？

一个生产级审批记录至少包含：

```json
{
  "approvalId": "appr_01HX...",
  "planHash": "sha256:8d1f...",
  "actor": "telegram:67431246",
  "risk": "high",
  "expiresAt": "2026-05-05T09:10:00Z",
  "approvedBy": "telegram:67431246",
  "approvedAt": "2026-05-05T08:45:00Z",
  "allowedTools": ["git.push", "deploy.trigger"],
  "status": "approved"
}
```

其中最关键的是 `planHash`。它应该由这些字段稳定生成：

- 工具名；
- canonicalized 参数；
- 目标环境；
- 风险等级；
- dry-run 证据；
- 预期副作用；
- 执行主体/权限作用域。

如果任何字段变化，hash 就变化，旧审批自动失效。

---

## 3. learn-claude-code：Python 教学版 PlanSigner

教学版用标准库实现：先把计划 canonicalize，再计算 SHA256。

```python
from dataclasses import dataclass, asdict
from datetime import datetime, timedelta, timezone
import hashlib
import json
import secrets

@dataclass(frozen=True)
class ToolStep:
    tool: str
    args: dict
    expected_effect: str

@dataclass(frozen=True)
class ExecutionPlan:
    actor: str
    target: str
    risk: str
    steps: list[ToolStep]
    evidence: list[str]

@dataclass
class ApprovalRecord:
    approval_id: str
    plan_hash: str
    approved_by: str
    expires_at: str
    status: str = "approved"


def canonical_json(value: object) -> str:
    return json.dumps(value, sort_keys=True, separators=(",", ":"), ensure_ascii=False)


def plan_hash(plan: ExecutionPlan) -> str:
    # dataclass -> dict 后排序，避免字段顺序导致 hash 不稳定
    payload = canonical_json(asdict(plan))
    return "sha256:" + hashlib.sha256(payload.encode("utf-8")).hexdigest()


def create_approval(plan: ExecutionPlan, approved_by: str, ttl_minutes: int = 15) -> ApprovalRecord:
    return ApprovalRecord(
        approval_id="appr_" + secrets.token_hex(8),
        plan_hash=plan_hash(plan),
        approved_by=approved_by,
        expires_at=(datetime.now(timezone.utc) + timedelta(minutes=ttl_minutes)).isoformat(),
    )


def assert_approval_valid(plan: ExecutionPlan, approval: ApprovalRecord, actor: str):
    if approval.status != "approved":
        raise PermissionError("approval is not approved")

    if datetime.fromisoformat(approval.expires_at) < datetime.now(timezone.utc):
        raise PermissionError("approval expired; request a new approval")

    if approval.approved_by != actor:
        raise PermissionError("approval actor mismatch")

    current_hash = plan_hash(plan)
    if current_hash != approval.plan_hash:
        raise PermissionError(
            f"plan changed after approval: expected {approval.plan_hash}, got {current_hash}"
        )


# 示例：审批 git push 到 main 前的不可变计划
plan = ExecutionPlan(
    actor="telegram:67431246",
    target="github:gfwfail/agent-course:main",
    risk="medium",
    steps=[
        ToolStep(
            tool="git.push",
            args={"remote": "origin", "branch": "main"},
            expected_effect="push one lesson commit",
        )
    ],
    evidence=["git status clean before commit", "lesson file exists", "README updated"],
)

approval = create_approval(plan, approved_by="telegram:67431246")
assert_approval_valid(plan, approval, actor="telegram:67431246")
```

教学版重点不是加密强度，而是流程：

1. 计划先 canonicalize；
2. 审批绑定 hash；
3. 执行前重新计算 hash；
4. hash 不一致就停。

---

## 4. pi-mono：ApprovalMiddleware 生产版

生产里不要让每个工具自己检查审批，应该放在 Tool Middleware 层统一做。

```ts
import { createHash } from 'node:crypto';
import { z } from 'zod';

type Risk = 'low' | 'medium' | 'high' | 'critical';

type ToolCall = {
  name: string;
  args: Record<string, unknown>;
  actor: string;
  scope: string;
  risk: Risk;
};

type ApprovalRecord = {
  id: string;
  planHash: string;
  approvedBy: string;
  expiresAt: string;
  allowedTools: string[];
};

const ApprovalInput = z.object({
  approvalId: z.string().optional(),
});

function stableStringify(value: unknown): string {
  return JSON.stringify(value, Object.keys(value as object).sort());
}

function canonicalizeToolCall(call: ToolCall) {
  return {
    name: call.name,
    args: call.args,
    actor: call.actor,
    scope: call.scope,
    risk: call.risk,
  };
}

function hashPlan(call: ToolCall): string {
  const canonical = stableStringify(canonicalizeToolCall(call));
  return 'sha256:' + createHash('sha256').update(canonical).digest('hex');
}

export function withApprovalIntegrity(store: ApprovalStore) {
  return async (call: ToolCall, next: () => Promise<unknown>) => {
    if (call.risk === 'low') return next();

    const parsed = ApprovalInput.parse(call.args);
    if (!parsed.approvalId) {
      return {
        ok: false,
        needsApproval: true,
        planHash: hashPlan(call),
        message: 'This action requires approval for the exact plan hash.',
      };
    }

    const approval: ApprovalRecord = await store.get(parsed.approvalId);
    if (!approval.allowedTools.includes(call.name)) {
      throw new Error(`approval does not allow tool ${call.name}`);
    }

    if (new Date(approval.expiresAt).getTime() < Date.now()) {
      throw new Error('approval expired');
    }

    const currentHash = hashPlan(call);
    if (currentHash !== approval.planHash) {
      throw new Error(`approved plan hash mismatch: ${currentHash}`);
    }

    return next();
  };
}
```

生产版还要补三件事：

- `stableStringify` 要支持深层对象排序，不能只排第一层；
- `approvalId` 最好从控制通道传入，不混在业务参数里；
- 审批记录要进入 Effect Journal，方便复盘“谁批准了什么”。

---

## 5. OpenClaw 实战：native approval 也要看完整命令

OpenClaw 里有 native approval card，这很好，但 Agent 仍然要遵守两条纪律：

1. **展示完整命令/脚本**：包括 `&&`、管道、重定向、多行 shell；
2. **审批只覆盖当前命令**：allow-once 不能被解释为后续命令都允许。

一个实用的 OpenClaw 执行前检查表：

```text
[approval preflight]
- command preview is complete: yes
- risk level: medium/high/critical
- target repo/server/env: explicit
- current observation refreshed: yes
- plan hash / command hash recorded: yes
- approval is not expired: yes
- command did not change after approval: yes
```

如果批准后 Agent 重新生成了命令，即使语义“差不多”，也必须重新审批。因为 shell 里一个参数就可能改变整个副作用边界。

---

## 6. 常见错误

### 错误 1：审批摘要，不审批参数

```text
用户看到：是否部署？
实际执行：deploy --env production --force
```

正确做法：审批界面必须显示目标环境、命令、参数和风险。

### 错误 2：旧审批复用到新计划

```text
plan A: git push feature/foo
plan B: git push main
```

两个计划都叫“push code”，但副作用完全不同。hash 必须不同。

### 错误 3：子 Agent 拿父 Agent 审批乱用

子 Agent 可以继承上下文，但不能继承无限审批权。审批应该绑定：

- parentRunId；
- childRunId；
- allowedTools；
- maxUses；
- expiresAt。

---

## 7. 设计 checklist

做审批系统时，问自己 8 个问题：

- 审批是否绑定 canonical plan hash？
- 执行前是否重新计算 hash？
- 计划变化是否强制重新审批？
- 审批是否有 TTL？
- 审批是否绑定 actor / scope / target？
- 是否记录 dry-run 证据？
- 是否限制 allowedTools / maxUses？
- 是否进入审计日志或 Effect Journal？

只要其中任意一项缺失，审批就还停留在“用户点了个同意按钮”的水平。

---

## 8. 一句话总结

审批不是信任 Agent 的记忆，而是验证一份不可变计划。  
**Plan Signing & Approval Integrity = 给人工确认加上工程化防篡改。**
