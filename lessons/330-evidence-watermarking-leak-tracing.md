# 330 - Agent 证据水印与泄漏追踪：Evidence Watermarking & Leak Tracing

> 证据访问控制解决“谁能看”，水印解决“如果泄漏了，怎么知道是哪一次访问泄漏的”。

前几讲我们做了 Evidence 授权、审计、异常检测、委托访问。今天补上一个生产系统非常实用的能力：**Evidence Watermarking & Leak Tracing**。

当 Agent 给不同子代理、审计员、外部系统展示同一份 evidence 时，不要把完全相同的文本/JSON 发出去。应该为每一次 view 生成一个不可见或低干扰的水印，把 `evidenceId + actor + runId + purpose + grantId` 绑定进去。以后如果截图、日志、复制文本泄漏出来，就能反查是哪一次访问、哪个委托、哪个任务链路泄漏的。

---

## 核心问题

没有水印的系统里，泄漏调查通常是这样：

```text
发现一段 evidence 内容出现在不该出现的地方
  ↓
只能查“哪些人有权限看过”
  ↓
候选人几十个 / 几百个
  ↓
无法定位具体来源，只能扩大封禁或人工排查
```

有水印后：

```text
每次 evidence view 都加入 trace watermark
  ↓
泄漏文本 / 截图 / 日志被发现
  ↓
extract watermark 或 fingerprint
  ↓
定位 accessEventId / grantId / actor / runId
  ↓
撤销相关 capability，冻结同类访问，触发 incident workflow
```

关键原则：

1. **每次 view 一个水印**：同一 evidence，不同 actor/run/purpose 得到不同 watermark。
2. **水印不代替脱敏**：raw 不该给就不给；水印只是泄漏追踪，不是授权。
3. **水印要可审计**：watermarkId 必须写入 access audit event。
4. **多层水印**：文本用 zero-width / footer；JSON 用 `_trace`; 文件用 metadata；截图场景用细微可见 footer。
5. **泄漏响应自动化**：定位后立即 revoke grant、降级 raw access、创建 incident。

---

## learn-claude-code：最小文本水印器

教学版先实现一个“低干扰文本水印”。用 HMAC 生成不可伪造的 watermark token，再用 zero-width 字符编码，插入到文本末尾。

```python
# learn-claude-code/evidence_watermark.py
from __future__ import annotations

import base64
import hashlib
import hmac
import json
import time
from dataclasses import dataclass

ZERO = "\u200b"  # zero width space
ONE = "\u200c"   # zero width non-joiner
PREFIX = "\u2063EW"  # invisible separator + marker


@dataclass
class WatermarkContext:
    evidence_id: str
    actor: str
    tenant: str
    purpose: str
    run_id: str
    grant_id: str | None = None


def sign_payload(payload: dict, secret: bytes) -> str:
    body = json.dumps(payload, sort_keys=True, separators=(",", ":")).encode()
    sig = hmac.new(secret, body, hashlib.sha256).digest()[:12]
    token = base64.urlsafe_b64encode(body + b"." + sig).decode().rstrip("=")
    return token


def encode_zero_width(token: str) -> str:
    bits = "".join(f"{byte:08b}" for byte in token.encode())
    return PREFIX + "".join(ONE if bit == "1" else ZERO for bit in bits)


def apply_text_watermark(text: str, ctx: WatermarkContext, secret: bytes) -> tuple[str, str]:
    payload = {
        "evidenceId": ctx.evidence_id,
        "actor": ctx.actor,
        "tenant": ctx.tenant,
        "purpose": ctx.purpose,
        "runId": ctx.run_id,
        "grantId": ctx.grant_id,
        "iat": int(time.time()),
    }
    token = sign_payload(payload, secret)
    watermark_id = hashlib.sha256(token.encode()).hexdigest()[:16]
    return text + encode_zero_width(token), watermark_id
```

读取证据时，先授权、再投影、再加水印、最后写审计：

```python
def read_evidence_view(evidence_db, audit_log, *, evidence_id, ctx, secret):
    payload = evidence_db[evidence_id]

    # 前置：access gate 已经决定只能给 summary
    view_text = payload["summary"]
    watermarked, watermark_id = apply_text_watermark(view_text, ctx, secret)

    audit_log.append({
        "event": "evidence_view_issued",
        "evidenceId": evidence_id,
        "actor": ctx.actor,
        "tenant": ctx.tenant,
        "purpose": ctx.purpose,
        "runId": ctx.run_id,
        "grantId": ctx.grant_id,
        "watermarkId": watermark_id,
        "decision": "summary_only",
        "ts": time.time(),
    })

    return watermarked
```

> 生产里不要只依赖 zero-width：很多平台会清洗不可见字符。要配合 visible footer、结构化 `_trace` 字段、文件 metadata、截图角标等多种方式。

---

## pi-mono：EvidenceWatermarkMiddleware

生产版建议把水印做成 Evidence View Pipeline 的 middleware，而不是业务代码里到处手写。

```typescript
// pi-mono/src/evidence/EvidenceWatermark.ts
export type WatermarkMode = "none" | "zero_width" | "visible_footer" | "json_trace";

export interface EvidenceWatermarkContext {
  evidenceId: string;
  tenantId: string;
  actorId: string;
  purpose: string;
  runId: string;
  grantId?: string;
  accessEventId: string;
  decision: "summary_only" | "raw";
}

export interface WatermarkResult<T> {
  view: T;
  watermarkId: string;
  mode: WatermarkMode;
}
```

Middleware 根据输出类型选择水印策略：

```typescript
// pi-mono/src/middleware/EvidenceWatermarkMiddleware.ts
import crypto from "node:crypto";

export class EvidenceWatermarkMiddleware {
  constructor(private readonly secret: string) {}

  apply<T extends string | Record<string, unknown>>(
    view: T,
    ctx: EvidenceWatermarkContext,
  ): WatermarkResult<T> {
    const payload = {
      evidenceId: ctx.evidenceId,
      tenantId: ctx.tenantId,
      actorId: ctx.actorId,
      purpose: ctx.purpose,
      runId: ctx.runId,
      grantId: ctx.grantId,
      accessEventId: ctx.accessEventId,
      decision: ctx.decision,
      issuedAt: new Date().toISOString(),
    };

    const encoded = Buffer.from(JSON.stringify(payload)).toString("base64url");
    const sig = crypto
      .createHmac("sha256", this.secret)
      .update(encoded)
      .digest("base64url")
      .slice(0, 24);

    const token = `${encoded}.${sig}`;
    const watermarkId = crypto.createHash("sha256").update(token).digest("hex").slice(0, 16);

    if (typeof view === "string") {
      const footer = `\n\n[trace:${watermarkId}]`;
      return { view: (view + footer) as T, watermarkId, mode: "visible_footer" };
    }

    return {
      view: { ...view, _trace: { watermarkId, accessEventId: ctx.accessEventId } } as T,
      watermarkId,
      mode: "json_trace",
    };
  }
}
```

注意：`watermarkId` 给审计和用户可见足够了；完整 token 不一定要暴露。服务端用 `watermarkId -> accessEventId` 索引反查。

---

## 泄漏反查：Leak Tracing

水印的价值在 incident 时体现。发现疑似泄漏内容后，系统尝试三层匹配：

1. **显式 trace**：提取 `[trace:xxxx]` 或 JSON `_trace.watermarkId`。
2. **不可见字符**：解析 zero-width token。
3. **模糊指纹**：如果 trace 被删，用内容 shingle / SimHash 匹配最近 issued views。

```typescript
// pi-mono/src/evidence/LeakTracer.ts
export interface LeakTraceResult {
  matched: boolean;
  confidence: number;
  watermarkId?: string;
  accessEventId?: string;
  actorId?: string;
  grantId?: string;
  recommendedAction: "ignore" | "review" | "revoke_grant" | "freeze_actor";
}

export async function traceLeak(text: string, auditIndex: AuditIndex): Promise<LeakTraceResult> {
  const explicit = text.match(/\[trace:([a-f0-9]{16})\]/)?.[1];
  if (explicit) {
    const event = await auditIndex.findByWatermarkId(explicit);
    return {
      matched: Boolean(event),
      confidence: event ? 1.0 : 0,
      watermarkId: explicit,
      accessEventId: event?.id,
      actorId: event?.actorId,
      grantId: event?.grantId,
      recommendedAction: event?.grantId ? "revoke_grant" : "review",
    };
  }

  const fuzzy = await auditIndex.findSimilarIssuedView(text);
  return {
    matched: fuzzy.score > 0.86,
    confidence: fuzzy.score,
    accessEventId: fuzzy.event?.id,
    actorId: fuzzy.event?.actorId,
    grantId: fuzzy.event?.grantId,
    recommendedAction: fuzzy.score > 0.94 ? "review" : "ignore",
  };
}
```

---

## OpenClaw 落地方式

OpenClaw 里可以把这个模式用在三类地方：

1. **Sub-agent 交接**：父 Agent 不把 evidence raw 放进 prompt，只传 `capabilityRef`；子 Agent redeem 出来的 view 自动带 trace。
2. **群聊/外部输出**：任何含 evidence 的 Telegram/Discord 消息加短 trace footer，便于事后追踪。
3. **课程 Cron / 自动任务**：发布前记录 `runId + lesson + evidenceRefs + outputHash`，形成可审计链路。

一个简单约定：

```json
{
  "evidenceView": {
    "summary": "...",
    "trace": "wm_8f3a91c0e12d44aa",
    "accessEventId": "eva_20260515_001"
  }
}
```

这样即使内容被转发，至少能定位“这段内容来自哪一次 Agent run”。

---

## 容易踩坑

1. **把水印当安全边界**：错。安全边界还是授权、脱敏、租户隔离；水印只是追责。
2. **只做不可见水印**：平台复制/渲染/清洗可能丢失，要多层策略。
3. **水印包含敏感信息**：不要把 actor/email/tenant 明文塞进 footer；footer 只放短 ID，详情服务端查。
4. **没有索引**：生成 watermarkId 却不能按它查 audit event，等于没做。
5. **没有响应动作**：能定位泄漏但不 revoke/freeze/alert，价值少一半。

---

## 小结

Evidence Watermarking 的本质是：**每一次证据展示都带出生证明**。

- 授权回答“这次能不能看”；
- 审计回答“谁在什么时候看过”；
- 委托回答“能否把查看能力交给别人”；
- 水印回答“如果内容流出，能不能定位来源”。

成熟 Agent 系统不是假设证据永远不会泄漏，而是让每一次证据流转都可追踪、可撤销、可复盘。
