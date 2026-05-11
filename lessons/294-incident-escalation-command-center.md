# 294. Agent 事件升级与指挥中心（Incident Escalation & Command Center）

上一课讲了运行健康 SLO 与错误预算：Agent 什么时候还能自动跑，什么时候要降级。

今天更进一步：**当错误预算烧穿、Kill Switch 触发、关键工具连续失败时，Agent 不应该只是发一句“出问题了”，而要自动进入事件流程：分级、指派、升级、记录证据、等待解除。**

一句话：

> 成熟 Agent 的告警不是噪音，而是一条可追踪、可升级、可恢复的 incident lifecycle。

---

## 1. 为什么需要 Incident Command Center

很多 Agent 系统早期只有两种状态：

1. 正常回答
2. 报错给用户

这在 Demo 里够用，在生产里不够。

真实事故需要回答这些问题：

- 这是 P0、P1 还是普通 warning？
- 谁是 commander？谁负责修？
- 同一个事故有没有重复告警？
- 是否已经进入 read-only / freeze？
- 什么时候升级到老板、值班群、GitHub issue 或 Pager？
- 解除事故的证据是什么？

所以我们需要一个 Agent 侧的 Incident Command Center：

```text
signal -> classify -> dedupe -> open incident -> assign owner
       -> apply control state -> notify -> collect evidence
       -> escalate if stale -> resolve with proof
```

---

## 2. Incident 数据模型

最小可用结构：

```json
{
  "incidentId": "inc_20260511_001",
  "severity": "P1",
  "status": "open",
  "title": "message delivery failure rate above SLO",
  "commander": "agent-main",
  "owner": "delivery-worker",
  "openedAt": "2026-05-11T11:30:00Z",
  "lastSignalAt": "2026-05-11T11:34:00Z",
  "dedupeKey": "slo:telegram_delivery:P1",
  "controlMode": "degraded",
  "evidence": [
    { "type": "metric", "name": "delivery_error_rate", "value": 0.18 },
    { "type": "log", "path": "logs/outbox-2026-05-11.jsonl" }
  ],
  "timeline": [
    { "at": "2026-05-11T11:30:00Z", "event": "opened" },
    { "at": "2026-05-11T11:31:00Z", "event": "control.degraded" }
  ]
}
```

关键字段：

- `severity`：决定打扰谁、是否自动降级
- `dedupeKey`：同类事故合并，避免刷屏
- `commander`：负责协调的人或 Agent
- `controlMode`：事故期间系统运行模式
- `evidence`：解除事故必须有证据，不是“感觉好了”
- `timeline`：复盘和审计靠它

---

## 3. learn-claude-code：Python 教学版

先做一个文件式 Incident Store，适合教学项目：

```python
# learn_claude_code/incident_center.py
from __future__ import annotations

import json
import time
from dataclasses import dataclass, asdict, field
from pathlib import Path
from typing import Literal

Severity = Literal["P0", "P1", "P2", "P3"]
Status = Literal["open", "mitigated", "resolved"]


@dataclass
class Incident:
    incident_id: str
    severity: Severity
    status: Status
    title: str
    dedupe_key: str
    commander: str
    owner: str
    opened_at: float
    last_signal_at: float
    control_mode: str
    evidence: list[dict] = field(default_factory=list)
    timeline: list[dict] = field(default_factory=list)


class IncidentCenter:
    def __init__(self, path: str = "state/incidents.json"):
        self.path = Path(path)
        self.path.parent.mkdir(parents=True, exist_ok=True)
        if not self.path.exists():
            self.path.write_text("{}")

    def _load(self) -> dict[str, dict]:
        return json.loads(self.path.read_text())

    def _save(self, data: dict[str, dict]) -> None:
        tmp = self.path.with_suffix(".tmp")
        tmp.write_text(json.dumps(data, ensure_ascii=False, indent=2))
        tmp.replace(self.path)

    def open_or_update(
        self,
        *,
        severity: Severity,
        title: str,
        dedupe_key: str,
        evidence: list[dict],
    ) -> Incident:
        now = time.time()
        data = self._load()

        # dedupe：已有 open incident 就追加信号，不重复开事故
        for raw in data.values():
            if raw["dedupe_key"] == dedupe_key and raw["status"] != "resolved":
                raw["last_signal_at"] = now
                raw["evidence"].extend(evidence)
                raw["timeline"].append({"at": now, "event": "signal.updated"})
                self._save(data)
                return Incident(**raw)

        incident_id = f"inc_{time.strftime('%Y%m%d_%H%M%S')}"
        control_mode = "freeze" if severity == "P0" else "degraded" if severity == "P1" else "normal"

        incident = Incident(
            incident_id=incident_id,
            severity=severity,
            status="open",
            title=title,
            dedupe_key=dedupe_key,
            commander="agent-main",
            owner="unassigned",
            opened_at=now,
            last_signal_at=now,
            control_mode=control_mode,
            evidence=evidence,
            timeline=[
                {"at": now, "event": "opened"},
                {"at": now, "event": f"control.{control_mode}"},
            ],
        )
        data[incident_id] = asdict(incident)
        self._save(data)
        return incident

    def resolve(self, incident_id: str, proof: dict) -> Incident:
        data = self._load()
        raw = data[incident_id]
        raw["status"] = "resolved"
        raw["evidence"].append(proof)
        raw["timeline"].append({"at": time.time(), "event": "resolved"})
        self._save(data)
        return Incident(**raw)
```

使用方式：

```python
center = IncidentCenter()

incident = center.open_or_update(
    severity="P1",
    title="Telegram delivery error rate above SLO",
    dedupe_key="slo:telegram_delivery:P1",
    evidence=[{"type": "metric", "name": "error_rate", "value": 0.18}],
)

print(incident.incident_id, incident.control_mode)
```

这个版本已经具备四个核心能力：

- dedupe：重复信号合并
- severity：按严重等级决策
- control：自动给出运行模式
- timeline：留下审计链

---

## 4. pi-mono：中间件生产版

生产里不要让业务工具自己判断事故升级，应该挂在 middleware：

```ts
// pi-mono/src/incident/IncidentMiddleware.ts
export type Severity = 'P0' | 'P1' | 'P2' | 'P3'

export interface IncidentSignal {
  severity: Severity
  title: string
  dedupeKey: string
  evidence: Array<Record<string, unknown>>
}

export interface IncidentStore {
  openOrUpdate(signal: IncidentSignal): Promise<{
    incidentId: string
    severity: Severity
    controlMode: 'normal' | 'degraded' | 'freeze'
  }>
}

export interface ControlPlane {
  setMode(input: {
    mode: 'normal' | 'degraded' | 'freeze'
    reason: string
    ttlSeconds: number
  }): Promise<void>
}

export function createIncidentMiddleware(deps: {
  incidentStore: IncidentStore
  controlPlane: ControlPlane
  notifier: { notifyIncident(input: unknown): Promise<void> }
}) {
  return async function incidentMiddleware(ctx: any, next: () => Promise<any>) {
    try {
      const result = await next()

      if (result?.sloViolation) {
        const incident = await deps.incidentStore.openOrUpdate({
          severity: result.sloViolation.severity,
          title: result.sloViolation.title,
          dedupeKey: result.sloViolation.dedupeKey,
          evidence: result.sloViolation.evidence,
        })

        if (incident.controlMode !== 'normal') {
          await deps.controlPlane.setMode({
            mode: incident.controlMode,
            reason: `incident:${incident.incidentId}`,
            ttlSeconds: 30 * 60,
          })
        }

        await deps.notifier.notifyIncident({
          incidentId: incident.incidentId,
          severity: incident.severity,
          title: result.sloViolation.title,
          controlMode: incident.controlMode,
        })
      }

      return result
    } catch (error: any) {
      // 未捕获异常也统一进入 incident 流程，而不是散落在日志里
      const incident = await deps.incidentStore.openOrUpdate({
        severity: 'P1',
        title: `Unhandled tool error: ${ctx.toolName}`,
        dedupeKey: `tool_error:${ctx.toolName}:${error.name}`,
        evidence: [
          { type: 'tool', name: ctx.toolName },
          { type: 'error', message: error.message },
        ],
      })

      await deps.notifier.notifyIncident({
        incidentId: incident.incidentId,
        severity: incident.severity,
        title: `Unhandled tool error: ${ctx.toolName}`,
      })

      throw error
    }
  }
}
```

这样 Agent Loop 只管业务，事故处理统一横切：

```ts
const dispatcher = new ToolDispatcher()

dispatcher.use(createIncidentMiddleware({
  incidentStore,
  controlPlane,
  notifier,
}))
```

好处：

- 所有工具统一事故升级
- SLO violation 和 exception 走同一条 lifecycle
- control plane 可以自动 freeze/degrade
- 通知、审计、复盘都能复用同一份 Incident

---

## 5. OpenClaw：Cron / Heartbeat 实战

OpenClaw 这种 always-on Agent 特别适合做轻量事件指挥中心。

可以维护一个文件：

```text
memory/incidents/open/inc_20260511_001.json
```

Heartbeat 或 Cron 每次醒来做三件事：

```text
1. 扫描 open incidents
2. 检查是否 stale / 是否需要升级
3. 如果恢复条件满足，写入 proof 并 resolve
```

升级策略示例：

```json
{
  "P0": { "notifyImmediately": true, "escalateAfterMinutes": 5, "controlMode": "freeze" },
  "P1": { "notifyImmediately": true, "escalateAfterMinutes": 15, "controlMode": "degraded" },
  "P2": { "notifyImmediately": false, "escalateAfterMinutes": 60, "controlMode": "normal" },
  "P3": { "notifyImmediately": false, "escalateAfterMinutes": 240, "controlMode": "normal" }
}
```

课程 Cron 也能用这个模式：

- `git push` 失败两次：开 P2 incident，记录 stdout/stderr
- Telegram message 发送失败：开 P1 incident，因为外部交付失败
- README / lesson / TOOLS 任一 gate 缺失：开 P2 incident，不允许报完成
- GitHub 认证失败：开 P1 incident，等待人工修复凭证

---

## 6. 解除事故必须有 proof

不要允许 Agent 只因为“下一次没报错”就关闭事故。

正确做法是定义 resolve gate：

```python
def can_resolve(incident: dict, checks: list[dict]) -> tuple[bool, list[str]]:
    missing = []

    if not any(c["name"] == "slo_back_to_normal" and c["ok"] for c in checks):
        missing.append("SLO has not recovered")

    if incident["severity"] in ["P0", "P1"]:
        if not any(c["name"] == "side_effect_verified" and c["ok"] for c in checks):
            missing.append("side effect verification missing")

    if not any(c["name"] == "timeline_complete" and c["ok"] for c in checks):
        missing.append("timeline incomplete")

    return len(missing) == 0, missing
```

事故关闭时写入 proof：

```json
{
  "type": "resolve_proof",
  "checks": [
    { "name": "slo_back_to_normal", "ok": true, "value": "error_rate=0.01" },
    { "name": "side_effect_verified", "ok": true, "value": "messageId=11679" },
    { "name": "timeline_complete", "ok": true }
  ]
}
```

---

## 7. 常见坑

### 坑 1：只告警，不建模

只发 Telegram 告警，后面就丢了。

应该先写 Incident，再通知。通知只是 Incident 的一个 side effect。

### 坑 2：每个错误都开新事故

没有 `dedupeKey` 会导致告警风暴。

同一工具、同一 SLO、同一风险等级，在未 resolved 前应该合并。

### 坑 3：事故期间还保持 full autonomy

P0/P1 事故期间，Agent 应该自动缩小自主权：

- P0：freeze 大部分副作用，只允许恢复工具
- P1：degraded，只允许低风险动作
- P2：记录并观察

### 坑 4：关闭事故没有证据

“我觉得好了”不是 proof。

必须有指标恢复、投递回执、验证命令、人工确认或回滚证据。

---

## 8. 生产 Checklist

实现 Incident Command Center 时至少检查：

- [ ] 每类信号都有 severity mapping
- [ ] 每个 incident 有稳定 dedupeKey
- [ ] P0/P1 会自动写入 controlMode
- [ ] 通知发送失败不会丢 Incident 本体
- [ ] stale incident 会升级，而不是一直 open
- [ ] resolve 必须带 proof
- [ ] timeline append-only
- [ ] 复盘能从 Incident 自动生成初稿

---

## 9. 总结

Agent 事故管理的核心不是“会报警”，而是：

```text
信号可分级
事故可去重
责任可指派
权限可收缩
过程可追踪
恢复可证明
```

一个真正能长期运行的 Agent，必须既会执行任务，也会在自己或环境出问题时进入清晰的事件流程。

> 告警只是铃声，Incident Command Center 才是处理事故的大脑。
