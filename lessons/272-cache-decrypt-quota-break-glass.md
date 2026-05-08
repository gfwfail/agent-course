# 272. Agent 缓存解密配额与 Break-Glass 应急访问（Cache Decrypt Quotas & Break-Glass Access）

> 上一课我们把 decrypt 变成了带授权的高风险动作。今天继续补生产里的最后一块：**即使有授权，也不能无限解密；真正紧急时可以 break-glass，但必须更严格地留痕和事后复盘**。

一句话：**缓存解密权限要有额度、速率、异常检测；应急越权可以存在，但不能静悄悄发生。**

## 1. 为什么授权还不够？

`KeyAccessGrant` 能回答“这次能不能解密”。但生产事故经常发生在“每一次看起来都合法，合起来却很危险”：

- 某个 Agent 一分钟内解密 500 个用户邮箱；
- 重加密 worker 因 bug 反复读同一批密文；
- 子 Agent 拿到合法 grant 后批量扫缓存；
- 线上事故排查需要临时查看敏感字段，但没有应急流程。

所以要在 grant 外面加一层 `DecryptQuotaGuard`：它不替代授权，而是限制**频率、数量、范围和应急路径**。

## 2. 配额模型：按 actor / purpose / field 分层

推荐把解密额度拆成四个维度：

```json
{
  "actor": "agent:main",
  "tenant": "acme",
  "purpose": "answer",
  "keyClass": "user_profile",
  "field": "email",
  "window": "1m",
  "maxDecrypts": 20,
  "breakGlassAllowed": false
}
```

几个经验值：

- `answer`：小额度，按用户交互自然速率走；
- `planning`：通常不允许解 sensitive 字段；
- `reencrypt`：额度大，但只允许后台 worker，且明文不能进 LLM；
- `external_side_effect`：默认 0，必须 live check 或重新授权；
- `break_glass`：临时越权，只给人类操作者，短 TTL，强审计。

## 3. learn-claude-code：Python 教学版

```python
# learn-claude-code/decrypt_quota_guard.py
from __future__ import annotations

from dataclasses import dataclass
from datetime import datetime, timedelta, timezone
from collections import defaultdict, deque
from typing import Literal

Purpose = Literal["answer", "planning", "reencrypt", "external_side_effect", "break_glass"]


@dataclass(frozen=True)
class DecryptQuota:
    max_decrypts: int
    window_seconds: int
    break_glass_allowed: bool = False


@dataclass(frozen=True)
class DecryptRequest:
    actor: str
    tenant: str
    purpose: Purpose
    key_class: str
    field: str
    grant_id: str
    break_glass_reason: str | None = None


class DecryptQuotaGuard:
    def __init__(self, quotas: dict[tuple[str, str, str], DecryptQuota]):
        self.quotas = quotas
        self.events: dict[tuple[str, str, str, str, str], deque[datetime]] = defaultdict(deque)

    def check(self, req: DecryptRequest, now: datetime | None = None) -> None:
        now = now or datetime.now(timezone.utc)
        quota_key = (req.purpose, req.key_class, req.field)
        quota = self.quotas.get(quota_key)
        if quota is None:
            raise PermissionError(f"no decrypt quota for {quota_key}")

        if req.purpose == "break_glass":
            if not quota.break_glass_allowed or not req.break_glass_reason:
                raise PermissionError("break-glass requires explicit reason and policy allow")

        event_key = (req.actor, req.tenant, req.purpose, req.key_class, req.field)
        bucket = self.events[event_key]
        cutoff = now - timedelta(seconds=quota.window_seconds)
        while bucket and bucket[0] < cutoff:
            bucket.popleft()

        if len(bucket) >= quota.max_decrypts:
            raise PermissionError("decrypt quota exceeded")

        bucket.append(now)
```

这个版本用内存队列表达滑动窗口。真实生产里可以换成 Redis Sorted Set 或令牌桶，核心不变：**授权通过以后，还要过速率和范围限制**。

## 4. pi-mono：TypeScript 中间件形态

```ts
// pi-mono/cache/DecryptQuotaMiddleware.ts
type Purpose = "answer" | "planning" | "reencrypt" | "external_side_effect" | "break_glass";

type DecryptRequest = {
  actor: string;
  tenant: string;
  purpose: Purpose;
  keyClass: string;
  field: string;
  grantId: string;
  breakGlassReason?: string;
};

type QuotaDecision = { allow: true } | { allow: false; reason: string; retryAfterMs?: number };

type QuotaStore = {
  consume(key: string, limit: number, windowMs: number): Promise<{ ok: boolean; retryAfterMs?: number }>;
};

type QuotaPolicy = {
  limitFor(req: DecryptRequest): { limit: number; windowMs: number; breakGlassAllowed: boolean } | null;
};

export class DecryptQuotaMiddleware {
  constructor(private store: QuotaStore, private policy: QuotaPolicy) {}

  async check(req: DecryptRequest): Promise<QuotaDecision> {
    const quota = this.policy.limitFor(req);
    if (!quota) return { allow: false, reason: "no_quota_policy" };

    if (req.purpose === "break_glass") {
      if (!quota.breakGlassAllowed || !req.breakGlassReason) {
        return { allow: false, reason: "break_glass_denied" };
      }
    }

    const key = ["decrypt", req.tenant, req.actor, req.purpose, req.keyClass, req.field].join(":");
    const used = await this.store.consume(key, quota.limit, quota.windowMs);

    audit("cache_decrypt_quota", {
      actor: req.actor,
      tenant: req.tenant,
      purpose: req.purpose,
      keyClass: req.keyClass,
      field: req.field,
      grantId: req.grantId,
      decision: used.ok ? "allow" : "deny",
      reason: used.ok ? undefined : "quota_exceeded",
      retryAfterMs: used.retryAfterMs,
      breakGlassReason: req.breakGlassReason,
    });

    return used.ok ? { allow: true } : { allow: false, reason: "quota_exceeded", retryAfterMs: used.retryAfterMs };
  }
}

function audit(event: string, payload: Record<string, unknown>) {
  console.log(JSON.stringify({ event, ts: new Date().toISOString(), ...payload }));
}
```

它应该放在 `KeyAccessMiddleware` 后面、真正 `crypto.decrypt()` 前面：

```ts
await grants.authorize(req);     // 这次有没有权限
await decryptQuota.check(req);   // 这类权限有没有被滥用
return crypto.decrypt(field, aad);
```

## 5. OpenClaw 实战：文件式 policy + JSONL 审计

OpenClaw 这种 always-on Agent 很适合把 policy 和审计都文件化，简单但非常可靠：

```json
{
  "answer:user_profile:email": { "limit": 20, "windowMs": 60000 },
  "answer:billing:invoice_total": { "limit": 10, "windowMs": 60000 },
  "reencrypt:*:*": { "limit": 5000, "windowMs": 3600000 },
  "external_side_effect:*:*": { "limit": 0, "windowMs": 60000 },
  "break_glass:user_profile:email": {
    "limit": 5,
    "windowMs": 300000,
    "breakGlassAllowed": true,
    "requiresReason": true
  }
}
```

审计日志建议 append-only：

```jsonl
{"event":"cache_decrypt_quota","ts":"2026-05-08T17:30:00Z","actor":"agent:main","tenant":"acme","purpose":"answer","field":"email","decision":"allow"}
{"event":"cache_decrypt_quota","ts":"2026-05-08T17:31:02Z","actor":"agent:main","tenant":"acme","purpose":"answer","field":"email","decision":"deny","reason":"quota_exceeded"}
```

Heartbeat 或 cron 可以每天扫一次：

- 哪个 actor 解密量突然升高；
- 哪些字段频繁触发 deny；
- break-glass 有没有 reason；
- break-glass 之后有没有工单/复盘记录；
- reencrypt worker 是否只在维护窗口高频解密。

## 6. Break-Glass 的三条铁律

Break-glass 不是后门，而是**受控应急通道**：

1. **必须写原因**：`incident_id` / `ticket_id` / 人类操作者；
2. **必须短 TTL**：分钟级，过期自动失效；
3. **必须事后复盘**：所有 break-glass 事件进入日报或安全审计。

如果一个系统没有 break-glass，真出事故时人会绕过系统；如果 break-glass 没审计，它就会变成长期后门。

## 7. 常见坑

- **只限制 grant，不限制 decrypt 次数**：批量滥用仍然合法；
- **重加密 worker 没单独额度**：后台任务和用户回答互相抢额度；
- **break-glass 没 reason**：事后完全不知道为什么越权；
- **deny 不审计**：安全团队看不到攻击尝试；
- **额度 key 太粗**：只按 tenant 限制会误伤正常用户，也抓不出单个 actor 异常。

## 8. 小结

今天的模式可以压成四句话：

1. decrypt 先授权，再过配额；
2. 配额按 actor、tenant、purpose、keyClass、field 分层；
3. break-glass 可以存在，但必须短 TTL、强原因、强审计；
4. deny 事件和 allow 事件一样重要，都要进入安全观测。

成熟 Agent 的缓存安全不是“能不能解密”这一个布尔值，而是一套运行时治理：谁解、解多少、为什么解、异常时谁知道。
