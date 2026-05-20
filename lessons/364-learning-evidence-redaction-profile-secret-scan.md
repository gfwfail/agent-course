# 364. Agent 学习证据脱敏剖面与 Secret 扫描闸门（Learning Evidence Redaction Profile & Secret Scan Gate）

上一课讲了 Learning Rule Evidence Retention：学习规则不能只保存结论，还要保存 sourceRunId、evidenceIds、decisionEventIds、prompt/tool/policy hash 与 replayPack。

今天继续补一个更容易出事故的细节：**学习证据进入长期规则前，必须先过脱敏剖面和 Secret 扫描闸门。**

因为学习系统最危险的地方不是“忘记了”，而是“记住了不该记住的东西”：

- 用户私聊内容被沉淀成全局规则；
- API token、cookie、订单号、邮箱、手机号被写进长期记忆；
- raw tool output 被直接塞进 prompt pack；
- 复盘里的敏感 payload 被课程、日志、GitHub、Telegram 二次传播；
- 一条本来正确的学习规则，因为证据泄漏，变成长期隐私风险。

一句话：**Learning Rule 可以长期存在，但它引用的证据必须先被降密、脱敏、扫描，并声明可用边界。**

## 1. 为什么 Retention 不等于安全

Retention Bundle 解决的是“未来还能不能审计和重放”。

但它本身不保证安全：

```json
{
  "ruleId": "learning:course:push-proof-before-commit",
  "retention": {
    "rawEvidenceRefs": ["telegram-message-raw", "shell-output-raw"],
    "summaryEvidenceRefs": ["lesson-summary"],
    "proofRefs": ["sha256:..."],
    "redactionProfile": "private-summary-with-proof-hashes"
  }
}
```

这里至少还有三个问题：

1. redactionProfile 只是字符串，谁证明它真的执行过？
2. rawEvidenceRefs 是否含 secret，谁扫描过？
3. 这份 evidence summary 能不能注入 LLM、发群、进 Git，边界在哪里？

所以生产系统里要把 redactionProfile 从“说明文字”升级成一个可执行的闸门。

## 2. 学习证据的四个视图

一条学习规则通常需要四种 evidence view：

| 视图 | 内容 | 可用于 |
| --- | --- | --- |
| raw | 原始证据，可能含敏感字段 | 严格审计、事故取证 |
| redacted | 删除或遮盖敏感字段后的证据 | 普通审计、回放输入 |
| summary | 抽取事实 claim，不含原文 payload | prompt 注入、规则说明 |
| proof | hash、签名、时间戳、schema/version | 长期证明、漂移检测 |

关键设计：**学习规则默认只能引用 summary + proof；raw 只能用来审计，不能直接进 prompt。**

## 3. Redaction Profile 应该长什么样

不要只写：

```json
{ "redactionProfile": "safe" }
```

这没有工程意义。更好的 profile 是可执行策略：

```json
{
  "profileId": "learning-rule-summary-v1",
  "allowRawInPrompt": false,
  "allowRawInExternalMessage": false,
  "fields": {
    "user.email": "hash",
    "user.phone": "mask",
    "message.text": "claim_extract",
    "toolOutput.stdout": "secret_scan_then_summarize",
    "env.*": "drop",
    "headers.authorization": "drop"
  },
  "requiredScanners": ["secret-patterns-v3", "pii-basic-v2", "prompt-injection-v1"],
  "maxSensitivityAfterRedaction": "internal",
  "proof": {
    "storePayloadHash": true,
    "storeRedactedHash": true,
    "storeProfileVersion": true
  }
}
```

这里的重点是：

- 每个字段怎么处理是明确的；
- 扫描器版本要入证据；
- raw 是否能进入 prompt / 外发是硬约束；
- 最终敏感度必须降到目标等级；
- proof 记录原始 hash 和脱敏后 hash，未来能证明没偷换。

## 4. learn-claude-code：教学版 Redaction Gate

learn-claude-code 的 `s05_skill_loading.py` 已经体现了一个重要原则：不要把所有内容一次性塞进 system prompt，而是用两层加载，先注入 metadata，需要时再加载完整内容。

学习证据也应该这样：默认只给 LLM summary，需要审计时才通过受控工具读取 raw。

教学版可以用纯函数实现一个 Redaction Gate：

```python
# learn-claude-code/learning_redaction_gate.py
from dataclasses import dataclass
import hashlib
import json
import re
from typing import Literal

View = Literal["raw", "redacted", "summary", "proof"]

SECRET_PATTERNS = [
    re.compile(r"sk-[A-Za-z0-9_-]{20,}"),
    re.compile(r"ghp_[A-Za-z0-9_]{20,}"),
    re.compile(r"AKIA[0-9A-Z]{16}"),
    re.compile(r"(?i)(api[_-]?key|token|secret|password)\s*[:=]\s*[^\s]+"),
]

@dataclass
class Evidence:
    evidence_id: str
    payload: dict
    sensitivity: str
    source: str

@dataclass
class RedactionProfile:
    profile_id: str
    allow_raw_in_prompt: bool
    allow_raw_external: bool
    max_sensitivity_after_redaction: str

@dataclass
class RedactedEvidence:
    evidence_id: str
    view: View
    payload: dict
    payload_hash: str
    redacted_hash: str
    profile_id: str
    scanners: list[str]
    findings: list[str]

def canonical_hash(value: object) -> str:
    data = json.dumps(value, ensure_ascii=False, sort_keys=True, separators=(",", ":"))
    return "sha256:" + hashlib.sha256(data.encode("utf-8")).hexdigest()

def scan_text(text: str) -> list[str]:
    findings = []
    for pattern in SECRET_PATTERNS:
        if pattern.search(text):
            findings.append(pattern.pattern)
    return findings

def redact_payload(payload: dict) -> tuple[dict, list[str]]:
    text = json.dumps(payload, ensure_ascii=False)
    findings = scan_text(text)

    redacted = json.loads(json.dumps(payload))
    for key in ("token", "password", "secret", "api_key"):
        if key in redacted:
            redacted[key] = "[REDACTED]"

    if "message" in redacted and isinstance(redacted["message"], str):
        redacted["message"] = summarize_claim(redacted["message"])

    if "stdout" in redacted:
        redacted["stdout"] = summarize_claim(str(redacted["stdout"]))

    return redacted, findings

def summarize_claim(text: str) -> str:
    # 教学版：真实系统应使用结构化 claim extraction，而不是简单截断。
    cleaned = re.sub(r"\s+", " ", text).strip()
    return cleaned[:280]

def build_learning_evidence_view(
    evidence: Evidence,
    profile: RedactionProfile,
    target_view: View,
) -> RedactedEvidence:
    raw_hash = canonical_hash(evidence.payload)
    redacted_payload, findings = redact_payload(evidence.payload)
    redacted_hash = canonical_hash(redacted_payload)

    if target_view == "raw":
        raise ValueError("learning rules cannot request raw evidence for prompt injection")

    if findings and target_view in ("summary", "redacted"):
        # 有 secret 命中时，summary 仍可保留事实，但不能保留原文片段。
        redacted_payload = {
            "claims": ["source evidence contained sensitive material"],
            "source": evidence.source,
            "sensitivity": "restricted",
        }
        redacted_hash = canonical_hash(redacted_payload)

    if target_view == "proof":
        redacted_payload = {
            "evidenceId": evidence.evidence_id,
            "payloadHash": raw_hash,
            "redactedHash": redacted_hash,
            "profileId": profile.profile_id,
            "scannerVersions": ["secret-patterns-v3", "pii-basic-v2"],
        }

    return RedactedEvidence(
        evidence_id=evidence.evidence_id,
        view=target_view,
        payload=redacted_payload,
        payload_hash=raw_hash,
        redacted_hash=redacted_hash,
        profile_id=profile.profile_id,
        scanners=["secret-patterns-v3", "pii-basic-v2"],
        findings=findings,
    )
```

这段代码的重点不是 regex 多强，而是边界清楚：

- Learning Rule 不允许请求 raw evidence 注入 prompt；
- 扫描命中 secret 后，只保留 claim，不保留原文；
- profileId、scanner version、payload hash、redacted hash 都写入 proof；
- 未来审计可以验证“当时这份 summary 是从哪份 raw 降密来的”。

## 5. pi-mono：LearningRedactionMiddleware

pi-mono 的 `resource-loader.ts`、`system-prompt.ts`、`skills.ts` 体现了一个生产框架常见模式：

1. 资源先被加载成结构化对象；
2. 再由 prompt builder 选择哪些内容进入 system prompt；
3. skill 也是先 metadata、后 full body。

学习规则也应该走同样模式：在进入 prompt pack 前，先过 LearningRedactionMiddleware。

```ts
type EvidenceViewMode = "raw" | "redacted" | "summary" | "proof";

type EvidenceRef = {
  id: string;
  source: "message" | "tool_result" | "memory" | "artifact" | "decision_event";
  sensitivity: "public" | "internal" | "private" | "secret" | "restricted";
  payloadHash: string;
};

type RedactionProfile = {
  id: string;
  version: string;
  allowRawInPrompt: boolean;
  allowRawExternal: boolean;
  requiredScanners: string[];
  maxPromptSensitivity: "public" | "internal" | "private";
};

type LearningRule = {
  id: string;
  status: "shadow" | "canary" | "active" | "retired";
  text: string;
  evidenceRefs: EvidenceRef[];
  redactionProfileId: string;
};

type RedactionResult = {
  allowed: boolean;
  viewMode: EvidenceViewMode;
  promptClaims: string[];
  proofRefs: string[];
  blockedReason?: string;
  audit: {
    profileId: string;
    scannerVersions: Record<string, string>;
    findings: Array<{ evidenceId: string; kind: string; severity: "low" | "medium" | "high" }>;
  };
};

class LearningRedactionMiddleware {
  constructor(
    private profiles: Map<string, RedactionProfile>,
    private evidenceStore: {
      loadSummary(id: string): Promise<{ claims: string[]; redactedHash: string; proofRef: string }>;
      scan(id: string, scanners: string[]): Promise<Array<{ kind: string; severity: "low" | "medium" | "high" }>>;
    },
    private audit: (event: string, payload: unknown) => void,
  ) {}

  async prepareForPrompt(rule: LearningRule): Promise<RedactionResult> {
    const profile = this.profiles.get(rule.redactionProfileId);
    if (!profile) {
      return { allowed: false, viewMode: "proof", promptClaims: [], proofRefs: [], blockedReason: "missing_redaction_profile", audit: { profileId: "missing", scannerVersions: {}, findings: [] } };
    }

    if (profile.allowRawInPrompt) {
      return { allowed: false, viewMode: "proof", promptClaims: [], proofRefs: [], blockedReason: "raw_prompt_not_allowed_for_learning", audit: { profileId: profile.id, scannerVersions: {}, findings: [] } };
    }

    const promptClaims: string[] = [];
    const proofRefs: string[] = [];
    const findings: RedactionResult["audit"]["findings"] = [];

    for (const ref of rule.evidenceRefs) {
      const scan = await this.evidenceStore.scan(ref.id, profile.requiredScanners);
      findings.push(...scan.map((item) => ({ evidenceId: ref.id, ...item })));

      if (scan.some((item) => item.severity === "high")) {
        proofRefs.push(ref.payloadHash);
        continue;
      }

      const summary = await this.evidenceStore.loadSummary(ref.id);
      promptClaims.push(...summary.claims);
      proofRefs.push(summary.proofRef);
    }

    const allowed = promptClaims.length > 0 && findings.every((item) => item.severity !== "high");

    this.audit("learning.redaction.prepare_prompt", {
      ruleId: rule.id,
      profileId: profile.id,
      allowed,
      findings,
      proofRefs,
    });

    return {
      allowed,
      viewMode: "summary",
      promptClaims,
      proofRefs,
      blockedReason: allowed ? undefined : "high_sensitivity_evidence_requires_review",
      audit: {
        profileId: profile.id,
        scannerVersions: Object.fromEntries(profile.requiredScanners.map((name) => [name, "current"])),
        findings,
      },
    };
  }
}
```

这层 middleware 不需要知道 LLM 怎么想。它只回答一个问题：

> 这条学习规则能不能带着哪些脱敏 claim 进入 prompt？

如果不能，就降级为 proof-only 或 manual_review。

## 6. OpenClaw 实战：课程 Cron 的学习规则怎么落地

以这个课程 Cron 为例，长期规则里可能有这些经验：

- 发布前必须检查已讲内容，避免重复；
- GitHub push 前要切换到 gfwfail；
- 课程内容要写入 lessons，并更新 README；
- Telegram messageId、git commit、memory log 要形成证据链。

这些规则是可以长期记住的，但它们的证据不应该把所有 shell output、Telegram raw message、完整 TOOLS.md 片段都长期暴露。

更稳的 Learning Rule 记录应该像这样：

```json
{
  "ruleId": "learning:agent-course:publish-checklist",
  "status": "active",
  "ruleText": "Agent 课程发布前必须检查已讲内容、写 lesson、更新 README、发 Telegram、commit/push，并记录 messageId 与 commit。",
  "evidence": {
    "summaryClaims": [
      "2026-05-20 多次课程发布都要求先查 TOOLS.md 已讲内容。",
      "成功发布记录包含 Telegram messageId、lesson file、README update、git commit。"
    ],
    "proofRefs": [
      "sha256:lesson-363-summary",
      "sha256:git-commit-85980e4",
      "sha256:telegram-message-12326"
    ],
    "redactionProfile": {
      "id": "learning-rule-summary-v1",
      "version": "2026-05-20",
      "rawAllowedInPrompt": false,
      "externalAllowed": "summary_only"
    },
    "scannerEvidence": {
      "secret-patterns-v3": "passed",
      "pii-basic-v2": "summary_only",
      "prompt-injection-v1": "passed"
    }
  }
}
```

这里最重要的是：长期规则记住的是流程和证据指针，不是把原始上下文整包塞进长期记忆。

## 7. 常见坑

**坑 1：脱敏只靠 LLM 自觉**

让 LLM “注意不要泄漏 secret”不是工程控制。正确做法是先由代码生成 redacted/summary/proof 三种 view，再让 LLM 只能看到允许的 view。

**坑 2：summary 里复制了原文敏感片段**

很多系统所谓 summary 只是截断原文。学习证据 summary 应该是 claim extraction：提取事实，不复制 payload。

**坑 3：扫描器没有版本**

今天 scanner 没发现，不代表未来也没问题。scanner version 必须写进 proof，升级 scanner 后可以重新扫描旧证据并触发 quarantine。

**坑 4：raw 证据直接进入 prompt pack**

学习规则越长期，越不能依赖 raw prompt。raw 只服务严格审计；日常推理用 summary + proof。

**坑 5：脱敏后没有 hash**

没有 payloadHash 和 redactedHash，未来就无法证明 summary 对应哪份 raw，也无法发现 redacted artifact 被偷偷改过。

## 8. 落地清单

给学习系统加 Redaction Gate，先做这 9 件事：

- [ ] Learning Rule schema 增加 redactionProfileId；
- [ ] Evidence Store 支持 raw、redacted、summary、proof 四种 view；
- [ ] raw view 默认禁止进入 prompt 和外部消息；
- [ ] summary 用 claim extraction，不用原文截断；
- [ ] secret/PII/prompt-injection scanner 版本写入 proof；
- [ ] payloadHash、redactedHash、profileVersion 长期保留；
- [ ] 高风险 scanner finding 自动 quarantine 或 manual_review；
- [ ] prompt pack 只接收 LearningRedactionMiddleware 的输出；
- [ ] scanner 升级后能重扫旧 learning evidence。

## 9. 总结

Learning Evidence Redaction Profile 解决的是一个长期安全问题：**Agent 学到的规则可以留下来，但敏感证据不能跟着规则一起无限期扩散。**

成熟 Agent 的学习链路应该是：

```text
raw evidence
  -> secret / PII / injection scan
  -> redacted evidence
  -> claim summary
  -> proof hashes
  -> learning rule prompt pack
```

最后记住一句：**学习系统不是记忆越多越好，而是能把经验留下、把敏感负担卸掉、还能证明这件事做对了。**
