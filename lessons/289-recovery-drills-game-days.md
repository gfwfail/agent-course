# 289. Agent 恢复演练与 Game Day（Recovery Drills & Game Days）

上一课讲的是 **Blast Radius**：让 Agent 的自主行动半径可控。今天继续往生产化推进一步：**不要等事故发生才第一次验证恢复流程**。

成熟的 Agent 系统必须定期做 Recovery Drill / Game Day：主动模拟工具失败、缓存损坏、权限吊销、消息重复投递、子 Agent 崩溃等场景，验证系统能不能按预期降级、恢复、告警、留证据。

> 重点：演练不是“搞破坏”，而是用受控故障证明恢复路径真的能跑通。

---

## 1. 为什么 Agent 特别需要恢复演练？

传统服务的故障多是接口超时、数据库挂了、队列堆积。Agent 多了几类特殊风险：

1. **LLM 决策不确定**：同一个错误，模型可能换一种修复路径。
2. **工具副作用复杂**：发消息、推代码、部署、改配置都不是纯函数。
3. **长任务跨时间**：cron、sub-agent、checkpoint、lease 可能跨多次唤醒。
4. **上下文会腐烂**：缓存、记忆、工具观察可能 stale 或冲突。
5. **恢复路径也会调用工具**：如果恢复 runbook 本身没测过，事故时更容易二次事故。

所以只写 Runbook 不够，必须定期验证：

- 失败能不能被检测到？
- 任务能不能安全暂停/重试/接管？
- 已发生副作用能不能通过 outbox/evidence 查清楚？
- 是否会重复发送、重复扣款、重复部署？
- 人工接管入口是否真的有效？

---

## 2. 核心设计：DrillSpec + FaultInjector + RecoveryGate

把演练写成结构化规格，而不是临时脚本：

```ts
type DrillSpec = {
  id: string;
  title: string;
  target: 'tool' | 'cache' | 'queue' | 'subagent' | 'memory' | 'delivery';
  fault: {
    kind: 'timeout' | 'rate_limit' | 'corrupt_data' | 'duplicate_delivery' | 'permission_revoked' | 'crash';
    scope: 'dry_run' | 'sandbox' | 'canary';
    durationMs?: number;
  };
  expected: {
    detectsWithinMs: number;
    maxDuplicateSideEffects: number;
    requiredSignals: string[];
    requiredEvidence: string[];
    finalState: 'recovered' | 'degraded' | 'manual_review';
  };
};
```

三件套：

- **DrillSpec**：描述要模拟什么故障、影响范围、预期结果。
- **FaultInjector**：只在演练范围内注入故障，避免污染真实生产流量。
- **RecoveryGate**：演练结束后检查证据，不达标就开 issue / 告警 / 阻断发布。

---

## 3. learn-claude-code：最小 Python 教学版

教学版用一个假的工具调用器模拟 timeout 和 retry，重点看“演练规格”和“验收闸门”。

```python
# learn_claude_code/recovery_drill.py
from dataclasses import dataclass
from time import monotonic

@dataclass
class DrillSpec:
    id: str
    fault_kind: str
    detects_within_ms: int
    max_duplicate_side_effects: int
    final_state: str

class EvidenceLog:
    def __init__(self):
        self.events = []

    def append(self, event: str, **meta):
        self.events.append({"event": event, **meta})

    def has(self, event: str) -> bool:
        return any(e["event"] == event for e in self.events)

class FaultInjector:
    def __init__(self, spec: DrillSpec):
        self.spec = spec
        self.used = False

    def maybe_fail(self, tool_name: str):
        if self.used:
            return
        self.used = True
        if self.spec.fault_kind == "timeout":
            raise TimeoutError(f"drill timeout injected for {tool_name}")

class AgentRunner:
    def __init__(self, fault: FaultInjector, evidence: EvidenceLog):
        self.fault = fault
        self.evidence = evidence
        self.side_effects = 0

    def send_message(self, text: str):
        # 幂等保护：发送前先写 outbox key
        self.evidence.append("outbox_prepared", key="lesson:289")
        self.fault.maybe_fail("send_message")
        self.side_effects += 1
        self.evidence.append("message_sent", text=text)

    def run(self):
        try:
            self.send_message("hello")
            return "recovered"
        except TimeoutError as e:
            self.evidence.append("fault_detected", error=str(e))
            self.evidence.append("retry_scheduled", backoff_ms=1000)
            return "degraded"

def run_drill(spec: DrillSpec):
    evidence = EvidenceLog()
    runner = AgentRunner(FaultInjector(spec), evidence)

    started = monotonic()
    state = runner.run()
    elapsed_ms = int((monotonic() - started) * 1000)

    assert elapsed_ms <= spec.detects_within_ms
    assert runner.side_effects <= spec.max_duplicate_side_effects
    assert evidence.has("fault_detected")
    assert evidence.has("retry_scheduled")
    assert state == spec.final_state

    return {"drill": spec.id, "state": state, "events": evidence.events}

if __name__ == "__main__":
    result = run_drill(DrillSpec(
        id="send-message-timeout",
        fault_kind="timeout",
        detects_within_ms=500,
        max_duplicate_side_effects=0,
        final_state="degraded",
    ))
    print(result)
```

这里故意要求 `max_duplicate_side_effects=0`：因为 timeout 发生在真正发送前，系统应该只留下 outbox prepared 和 retry scheduled，不应该假装发送成功。

---

## 4. pi-mono：生产版 DrillRunner 中间件

生产系统里不要把故障注入散落在业务代码里，应该挂到工具网关 / middleware 层。

```ts
// pi-mono/packages/agent-runtime/src/drills/DrillRunner.ts
import { randomUUID } from 'node:crypto';

export type FaultKind =
  | 'timeout'
  | 'rate_limit'
  | 'corrupt_data'
  | 'duplicate_delivery'
  | 'permission_revoked';

export interface DrillSpec {
  id: string;
  targetTool?: string;
  faultKind: FaultKind;
  scope: 'dry_run' | 'sandbox' | 'canary';
  expected: {
    requiredEvents: string[];
    maxSideEffects: number;
    finalState: 'recovered' | 'degraded' | 'manual_review';
  };
}

export interface ToolCall {
  name: string;
  args: unknown;
  risk: 'low' | 'medium' | 'high' | 'critical';
}

export interface ToolResult {
  ok: boolean;
  data?: unknown;
  error?: { code: string; message: string; recoverable: boolean };
  meta?: Record<string, unknown>;
}

export class DrillContext {
  readonly runId = randomUUID();
  readonly events: string[] = [];
  sideEffects = 0;

  constructor(readonly spec: DrillSpec) {}

  record(event: string) {
    this.events.push(event);
  }
}

export class DrillFaultMiddleware {
  constructor(private readonly ctx: DrillContext | null) {}

  async around(call: ToolCall, next: () => Promise<ToolResult>): Promise<ToolResult> {
    if (!this.ctx) return next();

    const spec = this.ctx.spec;
    const matched = !spec.targetTool || spec.targetTool === call.name;
    if (!matched) return next();

    this.ctx.record(`fault_injected:${spec.faultKind}`);

    if (spec.faultKind === 'timeout') {
      return {
        ok: false,
        error: {
          code: 'DRILL_TIMEOUT',
          message: `Recovery drill ${spec.id} injected timeout for ${call.name}`,
          recoverable: true,
        },
        meta: { drillId: spec.id, injected: true },
      };
    }

    if (spec.faultKind === 'permission_revoked') {
      return {
        ok: false,
        error: {
          code: 'DRILL_PERMISSION_REVOKED',
          message: `Recovery drill ${spec.id} revoked permission for ${call.name}`,
          recoverable: false,
        },
        meta: { drillId: spec.id, injected: true },
      };
    }

    return next();
  }
}

export function verifyDrill(ctx: DrillContext, finalState: string) {
  const missing = ctx.spec.expected.requiredEvents.filter(
    event => !ctx.events.includes(event),
  );

  if (missing.length > 0) {
    throw new Error(`Drill ${ctx.spec.id} missing events: ${missing.join(', ')}`);
  }

  if (ctx.sideEffects > ctx.spec.expected.maxSideEffects) {
    throw new Error(
      `Drill ${ctx.spec.id} side effects exceeded: ${ctx.sideEffects}`,
    );
  }

  if (finalState !== ctx.spec.expected.finalState) {
    throw new Error(
      `Drill ${ctx.spec.id} expected ${ctx.spec.expected.finalState}, got ${finalState}`,
    );
  }
}
```

生产要点：

- 演练默认只跑 `dry_run/sandbox/canary`，不要直接打全量生产。
- `meta.drillId` 必须进入日志、trace、evidence bundle。
- 故障注入由 middleware 统一控制，业务工具不需要知道自己在被演练。
- 演练失败要当成真实缺陷处理，不要默默吞掉。

---

## 5. OpenClaw 实战：Cron 定期做“小范围演练”

OpenClaw 这种 always-on Agent 很适合用 cron 做轻量恢复演练：

```md
# recovery-drills/daily-outbox-timeout.md
id: daily-outbox-timeout
scope: dry_run
target: message.send
fault: timeout
expected:
  finalState: degraded
  requiredEvidence:
    - outbox_prepared
    - fault_detected
    - retry_scheduled
    - no_external_message_id
  maxDuplicateSideEffects: 0
```

Cron 触发时不要真的往群里发测试消息，而是跑 dry-run 工具链：

```bash
openclaw cron at "tomorrow 03:10" \
  --title "Recovery drill: outbox timeout" \
  --prompt "Run recovery-drills/daily-outbox-timeout.md in dry-run mode, verify evidence, update memory/drill-state.json only if gates pass."
```

演练结果写入本地状态文件：

```json
{
  "lastDrills": {
    "daily-outbox-timeout": {
      "lastRunAt": "2026-05-11T06:30:00+11:00",
      "status": "passed",
      "evidencePath": "artifacts/drills/daily-outbox-timeout-2026-05-11.json"
    }
  }
}
```

这样心跳或发布前 gate 可以快速判断：

- 最近 24h outbox 恢复演练是否通过？
- 最近 7d sub-agent lease 接管是否通过？
- 最近 30d break-glass / kill switch 是否演练过？

---

## 6. 常见演练场景清单

建议从这些开始，覆盖 Agent 最容易出事的路径：

1. **消息发送 timeout**：确认 outbox 不重复投递。
2. **工具 429 限流**：确认 retry-after、生效退避、不会 tight loop。
3. **缓存 hash mismatch**：确认 quarantine + live refetch。
4. **子 Agent 崩溃**：确认 lease 过期后可接管，旧 worker fencing token 失效。
5. **Git push 前状态变化**：确认 pre-action revalidation 能阻断 stale branch。
6. **权限临时吊销**：确认 capability revocation 立即生效。
7. **Kill switch 打开**：确认副作用工具全部 freeze，只允许恢复工具。
8. **重复 webhook / cron**：确认 inbox dedupe key 生效。

---

## 7. Checklist

做 Recovery Drill 时检查这 8 项：

- [ ] 演练有明确 `scope`，默认 dry-run/sandbox。
- [ ] 故障注入点在 middleware，不污染业务代码。
- [ ] 每次演练有 `drillId/runId`，能串起日志和 evidence。
- [ ] 验收不靠感觉，使用 RecoveryGate 自动断言。
- [ ] 副作用有 outbox/effect journal/idempotency key 保护。
- [ ] 演练失败会产生可见任务，而不是只写一行日志。
- [ ] 演练频率按风险分层：高风险路径更频繁。
- [ ] 演练本身也受 blast radius 和 kill switch 约束。

---

## 8. 一句话总结

**Runbook 说明你打算怎么恢复，Recovery Drill 证明你真的恢复得了。**

Agent 越自主，越要定期演练失败路径；否则你拥有的不是自动化系统，而是一套未经验证的乐观假设。
