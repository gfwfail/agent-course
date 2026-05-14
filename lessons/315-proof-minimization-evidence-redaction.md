# 315. Agent 证据包最小披露与证明摘要（Proof Minimization & Evidence Redaction）

上一课我们讲了 **Proof-Carrying Actions**：高风险动作必须携带证据包，工具层验证 proof 后才允许执行。

今天补一个非常容易被忽略的安全点：**证据包不是越完整越好，而是要“足够证明，但尽量少暴露”。**

这叫 **Proof Minimization**：证明动作安全所需的最小事实进入上下文；原始敏感材料只存指针、hash 或审计归档，不直接喂给 LLM、不直接发到群里。

---

## 1. 问题：证据包也可能变成泄密通道

很多团队做完 evidence bag 之后，会犯一个新错误：

1. 为了证明跑过测试，把完整 CI log 塞进上下文
2. 为了证明配置检查，把 `.env` diff 原样记录
3. 为了证明消息已发送，把完整用户内容写进公开审计
4. 为了证明 API 调用成功，把 response body 全量归档

结果 proof 是有了，但敏感信息也跟着扩散了。

成熟 Agent 的证据应该分三层：

```json
{
  "proof": {
    "id": "test_result",
    "ok": true,
    "summary": "unit tests passed: 128 passed, 0 failed",
    "contentHash": "sha256:8b3f...",
    "artifactRef": "artifact://runs/2026-05-14/test.log",
    "redaction": "raw log stored, secrets masked, prompt only sees summary"
  }
}
```

LLM 只需要知道：**这个证据是否通过、由谁观察、什么时候观察、摘要是什么、原文在哪里可审计复查。**

不需要知道：完整 secret、完整用户隐私、完整生产日志。

---

## 2. 核心设计：ProofView 而不是 RawEvidence

把证据拆成两种对象：

```ts
type RawEvidence = {
  id: string;
  ok: boolean;
  observedAt: string;
  source: string;
  rawValue: unknown;
};

type ProofView = {
  id: string;
  ok: boolean;
  observedAt: string;
  source: string;
  summary: string;
  contentHash: string;
  artifactRef?: string;
  sensitivity: "public" | "internal" | "sensitive" | "secret";
  allowedInPrompt: boolean;
};
```

原则很简单：

- **RawEvidence** 给审计和复查系统看
- **ProofView** 给 LLM、Policy、消息输出看
- secret 字段永远不进入 ProofView，只能变成 `present=true`、`hash`、`SecretRef`
- 外部发送时再过一次 channel redaction

这样既能证明，又不把 proof 变成新风险。

---

## 3. learn-claude-code：Python 教学版

最小实现：先把 raw evidence 归档，再生成可注入上下文的 proof view。

```python
import hashlib
import json
import re
from dataclasses import dataclass
from datetime import datetime, timezone
from pathlib import Path
from typing import Any

SECRET_PATTERNS = [
    re.compile(r"(AKIA[0-9A-Z]{16})"),
    re.compile(r"(ghp_[A-Za-z0-9_]{20,})"),
    re.compile(r"(sk-[A-Za-z0-9]{20,})"),
]

@dataclass
class RawEvidence:
    id: str
    ok: bool
    source: str
    value: Any
    observed_at: str

@dataclass
class ProofView:
    id: str
    ok: bool
    source: str
    observed_at: str
    summary: str
    content_hash: str
    artifact_ref: str
    sensitivity: str
    allowed_in_prompt: bool


def canonical_hash(value: Any) -> str:
    body = json.dumps(value, sort_keys=True, ensure_ascii=False).encode()
    return "sha256:" + hashlib.sha256(body).hexdigest()


def mask_secrets(text: str) -> str:
    masked = text
    for pattern in SECRET_PATTERNS:
        masked = pattern.sub("[REDACTED_SECRET]", masked)
    return masked


def summarize(raw: RawEvidence) -> tuple[str, str, bool]:
    text = json.dumps(raw.value, ensure_ascii=False, default=str)
    masked = mask_secrets(text)

    has_secret = "[REDACTED_SECRET]" in masked
    sensitivity = "secret" if has_secret else "internal"

    # 教学版用简单摘要；生产版可以按证据类型写专门 summarizer
    short = masked[:240].replace("\n", " ")
    if len(masked) > 240:
        short += "..."

    return short, sensitivity, not has_secret


def archive_and_minimize(raw: RawEvidence, artifact_dir: Path) -> ProofView:
    artifact_dir.mkdir(parents=True, exist_ok=True)
    content_hash = canonical_hash(raw.value)
    artifact_path = artifact_dir / f"{raw.id}-{content_hash.removeprefix('sha256:')[:12]}.json"
    artifact_path.write_text(json.dumps(raw.__dict__, ensure_ascii=False, indent=2, default=str))

    summary, sensitivity, allowed = summarize(raw)

    return ProofView(
        id=raw.id,
        ok=raw.ok,
        source=raw.source,
        observed_at=raw.observed_at,
        summary=summary,
        content_hash=content_hash,
        artifact_ref=f"artifact://evidence/{artifact_path.name}",
        sensitivity=sensitivity,
        allowed_in_prompt=allowed,
    )


raw = RawEvidence(
    id="ci_result",
    ok=True,
    source="pytest",
    observed_at=datetime.now(timezone.utc).isoformat(),
    value={"passed": 128, "failed": 0, "log_tail": "all good"},
)

proof = archive_and_minimize(raw, Path(".evidence"))
print(proof)
```

关键点：

- 原始证据写入 artifact，支持审计复查
- prompt 只拿 `ProofView`
- 如果检测到 secret，默认 `allowed_in_prompt=False`
- hash 绑定 raw value，防止摘要和原文对不上

---

## 4. pi-mono：生产版中间件

生产系统里，Proof Minimization 应该放在 evidence pipeline 中，而不是靠业务代码自觉。

```ts
export type RawEvidence = {
  id: string;
  ok: boolean;
  source: "tool" | "test" | "approval" | "receipt";
  observedAt: string;
  value: unknown;
};

export type ProofView = {
  id: string;
  ok: boolean;
  source: RawEvidence["source"];
  observedAt: string;
  summary: string;
  contentHash: string;
  artifactRef?: string;
  sensitivity: "public" | "internal" | "sensitive" | "secret";
  allowedAudiences: Array<"policy" | "llm" | "operator" | "external">;
};

export class ProofMinimizationMiddleware {
  constructor(
    private artifactStore: ArtifactStore,
    private redactor: Redactor,
    private audit: AuditLog,
  ) {}

  async record(raw: RawEvidence): Promise<ProofView> {
    const contentHash = await sha256Canonical(raw.value);

    const artifactRef = await this.artifactStore.put({
      kind: "raw_evidence",
      id: raw.id,
      contentHash,
      value: raw.value,
      retentionDays: 90,
    });

    const redacted = await this.redactor.project(raw.value, {
      purpose: "proof_view",
      mode: "summary",
    });

    const proof: ProofView = {
      id: raw.id,
      ok: raw.ok,
      source: raw.source,
      observedAt: raw.observedAt,
      summary: redacted.summary,
      contentHash,
      artifactRef,
      sensitivity: redacted.sensitivity,
      allowedAudiences: redacted.sensitivity === "secret"
        ? ["policy", "operator"]
        : ["policy", "llm", "operator"],
    };

    await this.audit.append({
      type: "evidence.minimized",
      evidenceId: raw.id,
      contentHash,
      artifactRef,
      sensitivity: proof.sensitivity,
      allowedAudiences: proof.allowedAudiences,
    });

    return proof;
  }

  forAudience(proofs: ProofView[], audience: ProofView["allowedAudiences"][number]) {
    return proofs.filter((proof) => proof.allowedAudiences.includes(audience));
  }
}
```

这个中间件让证据流变成：

```text
Tool Result / Test Result
        ↓
RawEvidence
        ↓ archive + hash + redact + summarize
ProofView
        ↓
Policy / LLM / Operator / External Message
```

不同 audience 看到不同视图，避免“一份 proof 到处复制”。

---

## 5. OpenClaw 实战：课程 Cron 的证据最小披露

以这个课程 Cron 为例，执行链路里至少有这些 proof：

| 动作 | RawEvidence | ProofView |
|---|---|---|
| 写 lesson 文件 | 完整文件内容 | 文件路径 + sha256 + 标题 |
| 更新 README | diff 全文 | 变更文件列表 + diffstat |
| 发 Telegram | 完整消息 | target chat + message hash + message id |
| git push | push 输出 | remote + commit sha + pushed ok |

最终汇报不应该贴完整 raw evidence，而应该贴：

```json
{
  "lesson": "315-proof-minimization-evidence-redaction.md",
  "commit": "abc1234",
  "telegramMessageId": 11825,
  "proofs": [
    { "id": "lesson_file", "ok": true, "hash": "sha256:..." },
    { "id": "telegram_receipt", "ok": true, "hash": "sha256:..." },
    { "id": "git_push", "ok": true, "hash": "sha256:..." }
  ]
}
```

这样老板能知道事情完成了；审计系统能复查原文；群聊和 prompt 不会被塞一堆敏感材料。

---

## 6. 工程落地清单

做 Proof Minimization 时，建议加这几个规则：

1. **默认不把 RawEvidence 注入 LLM**：只能注入 ProofView
2. **每个 ProofView 必须有 contentHash**：摘要可读，hash 可验
3. **敏感字段只留存在性和指针**：例如 `token_present=true`，不要存 token
4. **外部消息二次投影**：Telegram/Slack/Email 看到的是 external view
5. **高风险动作保留 artifactRef**：方便事故复盘时人工复查
6. **审计日志记录披露决策**：谁看到了哪种 proof view

一句话：

> Evidence 负责证明事实；Minimization 负责控制事实的扩散半径。

---

## 7. 小结

今天的核心：

- Proof-Carrying Action 解决“执行前有没有证据”
- Proof Minimization 解决“证据有没有过度暴露”
- RawEvidence 用于审计，ProofView 用于推理和沟通
- hash + artifactRef 让摘要可复查、可证明、可追责

成熟 Agent 不只是“带着证据行动”，还要懂得：**证明够用就好，敏感信息别乱跑。**
