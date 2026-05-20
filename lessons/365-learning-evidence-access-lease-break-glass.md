# 365. Agent 学习证据访问租约与 Break-glass 审计（Learning Evidence Access Lease & Break-glass Audit）

上一课讲了 Learning Evidence Redaction Profile：学习规则引用证据前，必须先经过脱敏剖面、Secret 扫描、claim extraction 和 proof hash。

今天补上另一个生产级细节：**raw evidence 不是永远不能读，而是只能通过短 TTL 访问租约读取，并且每次读取都必须留下审计事件。**

原因很简单：学习系统要能审计、能取证、能修复误学。但如果任何 Agent 都能随手读取 raw evidence，前面做的脱敏和权限边界就会被绕开。

一句话：**raw evidence 要像生产数据库 break-glass 一样管理：默认不可见，临时开窗，绑定用途，到期失效，全程留痕。**

## 1. 为什么 Redaction 之后还需要 Access Lease

Redaction Gate 解决的是“进入长期规则和 prompt 的内容是否安全”。

但真实系统还会遇到这些场景：

- 学习规则误伤用户，需要查看原始 evidence 判断是不是摘要提错了；
- drift detector 发现 evidence hash 变了，需要核对 raw 与 redacted 的对应关系；
- 事故复盘要证明某条规则当时为什么被发布；
- 安全团队要抽查 secret scanner 是否漏扫；
- 租户要求删除或导出相关 evidence。

这些场景都可能需要 raw evidence。问题不是“永远不准读”，而是“谁、为什么、读哪份、读多久、读后发生了什么”必须可审计。

## 2. Access Lease 的最小字段

不要只做一个 `canReadRaw: true`。

生产里应该把 raw evidence 访问抽象成租约：

```json
{
  "leaseId": "lease_01HX...",
  "evidenceId": "ev_2026_05_20_12327",
  "requestedBy": "agent:rollback-forensics",
  "subject": "tenant:rskins",
  "purpose": "rollback_forensics",
  "allowedViews": ["raw", "proof"],
  "expiresAt": "2026-05-20T11:45:00Z",
  "maxReads": 1,
  "approval": {
    "mode": "break_glass",
    "approver": "operator:owner",
    "reason": "verify summary extraction after false positive"
  },
  "constraints": {
    "noPromptInjection": true,
    "noExternalMessage": true,
    "noPersistentCopy": true
  }
}
```

关键点：

- `purpose` 必须枚举，不能写自由发挥的“debug”；
- `expiresAt` 必须短，通常 5 到 30 分钟；
- `maxReads` 限制重复读取；
- `subject` 绑定租户/用户/频道，避免跨主体复用；
- `constraints` 声明 raw 不能进 prompt、不能外发、不能复制到长期记忆；
- break-glass 不是跳过审计，而是更强审计。

## 3. 访问决策不是权限开关，而是事件链

一次 raw evidence 读取至少产生三类事件：

```json
{"type":"evidence.lease.requested","leaseId":"lease_1","purpose":"rollback_forensics"}
{"type":"evidence.lease.granted","leaseId":"lease_1","expiresAt":"2026-05-20T11:45:00Z"}
{"type":"evidence.raw.read","leaseId":"lease_1","evidenceId":"ev_1","bytes":2048,"result":"success"}
```

如果失败，也要记录：

```json
{"type":"evidence.raw.read_denied","reason":"lease_expired","leaseId":"lease_1"}
```

这和 pi-mono 的 `agent-loop.ts` 很像：工具调用不是只执行函数，而是发出 `tool_execution_start`、`tool_execution_end`、`toolResult`。Access Lease 也应该把敏感读取变成可重放的事件，而不是藏在某个 helper 里。

## 4. learn-claude-code：教学版 AccessLeaseStore

learn-claude-code 的 `s12_worktree_task_isolation.py` 用 `EventBus` 记录 worktree 生命周期事件；`s08_background_tasks.py` 用任务 ID 管理后台任务状态。

学习证据访问可以用同样的教学模型：**租约是控制面，evidence store 是数据面，事件日志是审计面。**

```python
# learn-claude-code/learning_evidence_access_lease.py
from dataclasses import dataclass
import json
import time
import uuid
from pathlib import Path
from typing import Literal

Purpose = Literal["rollback_forensics", "scanner_audit", "subject_export", "legal_hold"]
View = Literal["raw", "redacted", "summary", "proof"]

@dataclass
class EvidenceLease:
    lease_id: str
    evidence_id: str
    requested_by: str
    subject: str
    purpose: Purpose
    allowed_views: list[View]
    expires_at: float
    max_reads: int
    reads: int
    no_prompt_injection: bool = True
    no_external_message: bool = True
    no_persistent_copy: bool = True

class AuditLog:
    def __init__(self, path: Path):
        self.path = path
        self.path.parent.mkdir(parents=True, exist_ok=True)
        if not self.path.exists():
            self.path.write_text("")

    def emit(self, event: str, payload: dict):
        record = {"type": event, "ts": time.time(), **payload}
        with self.path.open("a", encoding="utf-8") as f:
            f.write(json.dumps(record, ensure_ascii=False, sort_keys=True) + "\n")

class AccessLeaseStore:
    def __init__(self, audit: AuditLog):
        self.leases: dict[str, EvidenceLease] = {}
        self.audit = audit

    def request_break_glass(
        self,
        evidence_id: str,
        requested_by: str,
        subject: str,
        purpose: Purpose,
        ttl_seconds: int = 900,
    ) -> EvidenceLease:
        if ttl_seconds > 1800:
            raise ValueError("raw evidence lease ttl must be <= 30 minutes")

        lease = EvidenceLease(
            lease_id="lease_" + uuid.uuid4().hex[:12],
            evidence_id=evidence_id,
            requested_by=requested_by,
            subject=subject,
            purpose=purpose,
            allowed_views=["raw", "proof"],
            expires_at=time.time() + ttl_seconds,
            max_reads=1,
            reads=0,
        )
        self.leases[lease.lease_id] = lease
        self.audit.emit("evidence.lease.granted", lease.__dict__)
        return lease

    def authorize_read(
        self,
        lease_id: str,
        evidence_id: str,
        view: View,
        target: Literal["audit", "prompt", "external_message", "memory"],
    ) -> EvidenceLease:
        lease = self.leases.get(lease_id)
        if not lease:
            self.audit.emit("evidence.raw.read_denied", {"leaseId": lease_id, "reason": "lease_not_found"})
            raise PermissionError("lease not found")

        reason = None
        if lease.evidence_id != evidence_id:
            reason = "wrong_evidence"
        elif time.time() > lease.expires_at:
            reason = "lease_expired"
        elif lease.reads >= lease.max_reads:
            reason = "max_reads_exceeded"
        elif view not in lease.allowed_views:
            reason = "view_not_allowed"
        elif view == "raw" and target in ("prompt", "external_message", "memory"):
            reason = "raw_target_forbidden"

        if reason:
            self.audit.emit("evidence.raw.read_denied", {
                "leaseId": lease_id,
                "evidenceId": evidence_id,
                "view": view,
                "target": target,
                "reason": reason,
            })
            raise PermissionError(reason)

        lease.reads += 1
        self.audit.emit("evidence.raw.read", {
            "leaseId": lease_id,
            "evidenceId": evidence_id,
            "view": view,
            "target": target,
            "purpose": lease.purpose,
            "requestedBy": lease.requested_by,
        })
        return lease
```

这个教学版有三个重点：

- raw evidence 读取必须带 `lease_id`；
- 租约只允许读指定 evidence，不能横向扩散；
- 即使有租约，raw 也不能进入 prompt、外部消息或长期记忆。

## 5. pi-mono：EvidenceAccessMiddleware

pi-mono 的 `agent-loop.ts` 在工具调用前后推送事件，`mom/src/agent.ts` 订阅这些事件并写日志、回频道 thread。这个结构很适合加敏感读取审计。

生产版可以做一个中间件，包住 evidence tool：

```ts
type EvidenceView = "raw" | "redacted" | "summary" | "proof";
type ReadTarget = "audit" | "prompt" | "external_message" | "memory";

interface EvidenceLease {
	leaseId: string;
	evidenceId: string;
	subject: string;
	purpose: "rollback_forensics" | "scanner_audit" | "subject_export" | "legal_hold";
	allowedViews: EvidenceView[];
	expiresAt: number;
	maxReads: number;
	reads: number;
	constraints: {
		noPromptInjection: boolean;
		noExternalMessage: boolean;
		noPersistentCopy: boolean;
	};
}

interface AuditSink {
	emit(event: string, payload: Record<string, unknown>): void;
}

function authorizeEvidenceRead(
	lease: EvidenceLease | undefined,
	input: { evidenceId: string; view: EvidenceView; target: ReadTarget },
	audit: AuditSink,
): EvidenceLease {
	const deny = (reason: string): never => {
		audit.emit("evidence.read_denied", {
			leaseId: lease?.leaseId ?? "missing",
			evidenceId: input.evidenceId,
			view: input.view,
			target: input.target,
			reason,
		});
		throw new Error(`Evidence read denied: ${reason}`);
	};

	if (!lease) deny("lease_required");
	if (lease.evidenceId !== input.evidenceId) deny("wrong_evidence");
	if (Date.now() > lease.expiresAt) deny("lease_expired");
	if (lease.reads >= lease.maxReads) deny("max_reads_exceeded");
	if (!lease.allowedViews.includes(input.view)) deny("view_not_allowed");

	if (input.view === "raw" && input.target === "prompt" && lease.constraints.noPromptInjection) {
		deny("raw_prompt_forbidden");
	}
	if (input.view === "raw" && input.target === "external_message" && lease.constraints.noExternalMessage) {
		deny("raw_external_forbidden");
	}
	if (input.view === "raw" && input.target === "memory" && lease.constraints.noPersistentCopy) {
		deny("raw_memory_forbidden");
	}

	lease.reads += 1;
	audit.emit("evidence.read_allowed", {
		leaseId: lease.leaseId,
		evidenceId: input.evidenceId,
		view: input.view,
		target: input.target,
		purpose: lease.purpose,
		subject: lease.subject,
	});
	return lease;
}
```

接入点建议放在两个地方：

1. evidence tool 内部：没有 lease 就不返回 raw；
2. prompt builder 前：如果内容 view 是 raw，直接拒绝进入 system/user context。

这样即使某个工具误返回 raw，也会在 prompt 注入前被第二道门拦住。

## 6. OpenClaw 实战：Cron 课程的 raw evidence 边界

以这个课程 cron 为例，每 3 小时会做四件有副作用的事：

- 发 Telegram；
- 写 lesson；
- 更新 README / TOOLS；
- git commit / push。

这些动作会产生很多 evidence：

- Telegram messageId；
- git commit hash；
- lesson 文件 diff；
- `git diff --check` / push 结果；
- memory 里的课程记录。

正常情况下，长期课程学习只需要 summary + proof：

```json
{
  "lesson": 365,
  "topic": "Learning Evidence Access Lease & Break-glass Audit",
  "proof": {
    "telegramMessageId": 12345,
    "commit": "abc1234",
    "files": [
      "lessons/365-learning-evidence-access-lease-break-glass.md",
      "README.md",
      "../TOOLS.md"
    ]
  }
}
```

如果未来要查 raw shell output 或 raw Telegram payload，就应该开一个短 TTL lease，并把用途绑定到 `rollback_forensics` 或 `scanner_audit`。查完以后只能把结论写回 summary，不能把 raw payload 复制进下一课、README 或群消息。

## 7. 常见坑

1. **把 break-glass 当超级管理员**

   错。break-glass 只是在紧急情况下临时开窗，审计强度应该更高，不是更低。

2. **租约只绑定用户，不绑定 evidence**

   这会导致拿到一次授权后横向读取所有 raw evidence。租约必须绑定 evidenceId 或非常小的 scope。

3. **过期只在 UI 检查**

   工具层、store 层、prompt builder 层都要检查。UI 不是安全边界。

4. **读 raw 后把内容写进 memory**

   这是最常见的二次泄漏。memory 只能写结论、hash、引用 ID，不写 raw payload。

5. **审计日志也泄漏 raw**

   audit event 记录 evidenceId、hash、bytes、reason，不记录原始内容。

## 8. 生产 Checklist

- [ ] raw evidence 读取必须带 leaseId；
- [ ] lease 绑定 evidenceId / subject / purpose；
- [ ] TTL 默认 5-30 分钟；
- [ ] maxReads 默认 1；
- [ ] raw 禁止进入 prompt、外部消息、长期 memory；
- [ ] 所有 allow / deny 都写 append-only audit event；
- [ ] break-glass 需要 reason 和 approver；
- [ ] prompt builder 对 raw view 做二次拦截；
- [ ] 审计日志只写 metadata，不写 raw payload；
- [ ] 租约过期后不能续命，只能重新申请。

## 9. 核心 takeaway

学习系统越成熟，越不能只靠“脱敏后再说”。

真正安全的 Agent 学习链路应该是：

```text
raw evidence
  -> redaction profile
  -> summary/proof for learning rule
  -> access lease for rare raw inspection
  -> append-only audit
  -> no raw prompt / no raw external / no raw memory
```

**成熟 Agent 不是永远不看原始证据，而是每一次看原始证据都能解释、限时、限域、留痕、到期失效。**
