# 287. Agent 全局熔断与紧急停止（Kill Switch & Side-Effect Circuit Breaker）

> Agent 越自动化，越需要一个能“一键停手”的安全刹车。权限、审批、Runbook 都是正常路径的护栏；Kill Switch 是系统异常、事故扩大、凭证疑似泄露、外部 API 失控时的最后防线。

前几课我们讲了事故复盘、回归测试和发布证据包。今天补上运行时安全里非常关键的一层：**全局熔断与紧急停止**。

核心思想：

> 所有副作用工具在真正执行前，都必须检查一个可动态更新的 Kill Switch / Circuit Breaker 状态；一旦进入 freeze，只允许只读诊断和恢复动作，禁止继续发消息、部署、删改数据、创建订单、推送代码等外部写操作。

---

## 1. 为什么 Agent 需要 Kill Switch？

普通服务的熔断，通常是为了保护下游：API 慢了就停止调用。

Agent 的熔断还要保护“现实世界”：

- LLM 误判导致重复发消息；
- cron 循环触发重复部署；
- 工具参数补全错了，把生产当测试；
- 凭证疑似泄露，需要立刻停止所有写操作；
- 上游 API 返回脏数据，继续执行会扩大事故；
- 人类明确说“停一下”，所有后台任务都要尊重。

如果没有 Kill Switch，Agent 可能还在“按计划完成任务”，但计划本身已经不安全了。

---

## 2. Kill Switch 的状态模型

不要只用一个布尔值 `disabled=true`。生产里至少需要区分 scope、reason、TTL 和允许的恢复动作。

```json
{
  "id": "ks-2026-05-11-001",
  "scope": "global",
  "state": "freeze",
  "reason": "suspected_duplicate_deploy_loop",
  "createdBy": "operator:admin",
  "createdAt": "2026-05-11T00:30:00+11:00",
  "expiresAt": "2026-05-11T01:30:00+11:00",
  "allowReadOnly": true,
  "allowedRecoveryTools": ["deploy.status", "runbook.rollback", "incident.create"],
  "blockedRiskLevels": ["medium", "high", "critical"]
}
```

建议的状态：

- `normal`：正常执行；
- `degraded`：允许低风险动作，高风险必须审批；
- `freeze`：阻断副作用，只允许只读诊断和白名单恢复；
- `locked`：严重事件，连恢复动作也必须人工审批。

---

## 3. learn-claude-code：最小 Kill Switch Guard

教学版可以先做成一个 JSON 文件 + 工具执行前检查器。

```python
# learn_claude_code/kill_switch.py
from __future__ import annotations

import json
from dataclasses import dataclass
from datetime import datetime, timezone
from pathlib import Path
from typing import Literal

Risk = Literal["low", "medium", "high", "critical"]
SwitchState = Literal["normal", "degraded", "freeze", "locked"]


@dataclass
class ToolCall:
    name: str
    risk: Risk
    side_effect: bool


@dataclass
class KillSwitch:
    state: SwitchState
    reason: str
    expires_at: str | None
    allow_read_only: bool
    allowed_recovery_tools: list[str]
    blocked_risk_levels: list[Risk]

    def expired(self) -> bool:
        if not self.expires_at:
            return False
        expires = datetime.fromisoformat(self.expires_at.replace("Z", "+00:00"))
        return datetime.now(timezone.utc) > expires.astimezone(timezone.utc)


def load_switch(path: str = "runtime/kill-switch.json") -> KillSwitch:
    p = Path(path)
    if not p.exists():
        return KillSwitch("normal", "", None, True, [], [])
    return KillSwitch(**json.loads(p.read_text(encoding="utf-8")))


def preflight(tool: ToolCall, switch: KillSwitch) -> None:
    if switch.expired():
        return

    if switch.state == "normal":
        return

    if tool.name in switch.allowed_recovery_tools:
        return

    if switch.state == "locked":
        raise PermissionError(f"kill_switch_locked: {switch.reason}")

    if switch.state == "freeze" and tool.side_effect:
        raise PermissionError(f"side_effect_blocked_by_kill_switch: {switch.reason}")

    if switch.state == "degraded" and tool.risk in switch.blocked_risk_levels:
        raise PermissionError(f"risk_blocked_by_degraded_mode: {tool.risk}")


# demo
if __name__ == "__main__":
    switch = load_switch()
    preflight(ToolCall("git.push", "high", side_effect=True), switch)
    print("allowed")
```

这个例子很小，但原则很重要：**检查发生在工具执行层，不是靠 prompt 提醒模型“请不要做危险事”。**

---

## 4. pi-mono：副作用工具熔断中间件

生产版建议做成 Tool Middleware，所有工具统一经过一条 preflight pipeline。

```ts
// pi-mono/packages/agent-runtime/src/safety/kill-switch-middleware.ts
export type Risk = "low" | "medium" | "high" | "critical";
export type KillSwitchState = "normal" | "degraded" | "freeze" | "locked";

export interface ToolMeta {
  name: string;
  risk: Risk;
  sideEffect: boolean;
  recoveryTool?: boolean;
}

export interface KillSwitchRecord {
  state: KillSwitchState;
  reason: string;
  scope: "global" | "tenant" | "session" | "tool";
  expiresAt?: string;
  allowReadOnly: boolean;
  allowedRecoveryTools: string[];
  blockedRiskLevels: Risk[];
}

export interface KillSwitchStore {
  getEffectiveSwitch(input: {
    tenantId?: string;
    sessionId: string;
    toolName: string;
  }): Promise<KillSwitchRecord | null>;
}

export class KillSwitchBlocked extends Error {
  constructor(
    public readonly code: string,
    public readonly switchRecord: KillSwitchRecord,
  ) {
    super(`${code}: ${switchRecord.reason}`);
  }
}

function isExpired(record: KillSwitchRecord): boolean {
  return !!record.expiresAt && Date.now() > Date.parse(record.expiresAt);
}

export function createKillSwitchMiddleware(store: KillSwitchStore) {
  return async function killSwitchMiddleware(ctx: {
    tenantId?: string;
    sessionId: string;
    tool: ToolMeta;
  }, next: () => Promise<unknown>) {
    const record = await store.getEffectiveSwitch({
      tenantId: ctx.tenantId,
      sessionId: ctx.sessionId,
      toolName: ctx.tool.name,
    });

    if (!record || record.state === "normal" || isExpired(record)) {
      return next();
    }

    if (record.allowedRecoveryTools.includes(ctx.tool.name) || ctx.tool.recoveryTool) {
      return next();
    }

    if (record.state === "locked") {
      throw new KillSwitchBlocked("kill_switch_locked", record);
    }

    if (record.state === "freeze" && ctx.tool.sideEffect) {
      throw new KillSwitchBlocked("side_effect_frozen", record);
    }

    if (record.state === "degraded" && record.blockedRiskLevels.includes(ctx.tool.risk)) {
      throw new KillSwitchBlocked("risk_blocked_in_degraded_mode", record);
    }

    return next();
  };
}
```

这层可以挂在所有工具前面：`message.send`、`git.push`、`deploy.create`、`db.write`、`payment.refund` 都统一受控。

---

## 5. OpenClaw 落地：Cron / Heartbeat / Sub-agent 都要尊重同一个刹车

OpenClaw 这种 always-on Agent 最容易踩的坑是：主会话停了，但 cron 或 sub-agent 还在跑。

建议做法：

1. 在 workspace 放一个运行时开关，例如 `runtime/kill-switch.json`；
2. 所有高风险 cron 开始时先读它；
3. 发 Telegram、push git、部署、删改文件前再次读它；
4. sub-agent 继承同一个 `runId/sessionId`，工具层查 effective switch；
5. 如果被阻断，写 `incident` 或 `daily memory`，不要静默失败。

伪代码：

```ts
async function guardedSideEffect(toolCall: ToolCall, execute: () => Promise<ToolResult>) {
  const decision = await killSwitchGuard.check({
    sessionId: toolCall.sessionId,
    tenantId: toolCall.tenantId,
    toolName: toolCall.name,
    risk: toolCall.risk,
    sideEffect: toolCall.sideEffect,
  });

  if (decision.action === "block") {
    await auditLog.append({
      event: "tool_blocked_by_kill_switch",
      toolName: toolCall.name,
      reason: decision.reason,
      switchId: decision.switchId,
    });
    return {
      ok: false,
      error: "Blocked by active kill switch",
      recoverable: true,
    };
  }

  return execute();
}
```

重点是：Kill Switch 不是某个 Agent 的“自觉”，而是所有副作用工具共享的硬闸门。

---

## 6. 设计细节：别让 Kill Switch 变成另一个事故源

几个实战建议：

- **必须有 TTL**：避免事故后忘记恢复，导致系统长期假死；
- **必须审计**：谁开启、为什么、影响哪些 scope、什么时候解除；
- **必须支持局部 scope**：能只冻结某个 tenant/session/tool，就不要全局停机；
- **恢复动作白名单**：rollback、status、incident.create 这类恢复工具不能被普通 freeze 一起锁死；
- **解除也要审批**：从 `locked` 回到 `normal` 应该比开启更严格；
- **副作用前二次检查**：计划阶段检查过不算，真正执行前还要重新检查一次；
- **和 Circuit Breaker 分层**：Kill Switch 是人工/策略强制刹车，Circuit Breaker 是指标驱动自动熔断，两者可以共享执行闸门。

---

## 7. 一句话总结

> 成熟 Agent 不只是会自动执行，还必须能被可靠地停下来。Kill Switch 把“停手”从聊天指令变成工具层强约束：只读诊断继续，副作用立刻冻结，恢复动作受控放行。

下一次你给 Agent 加任何外部写工具时，都问一句：**它在全局 freeze 时会不会真的停手？**
