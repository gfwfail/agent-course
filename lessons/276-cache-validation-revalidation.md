# 276. Agent 缓存校验器与自动再验证（Cache Validation & Revalidation）

> 上一课我们讲了缓存血缘：缓存答案要知道自己从哪里来。今天继续往生产可靠性推进一步：**缓存命中以后，不要立刻相信它；先校验它还像不像一个可用事实，必要时自动回源再验证。**

一句话：**Cache hit 只是“找到了旧答案”，Validation 才决定“这个旧答案今天还能不能用”。**

## 1. 为什么命中还要校验？

很多缓存事故不是 miss，而是 hit：

- 上游 API schema 变了，旧缓存结构还能反序列化，但字段语义变了；
- LLM 摘要缓存里混入了过时结论；
- 价格、库存、权限、部署状态这类事实变化太快；
- 缓存值被手工改过，value hash 对不上；
- 业务规则升级后，旧缓存不再满足新 policy。

所以成熟 Agent 的读取路径不是：

```text
get(key) -> hit -> use
```

而是：

```text
get(key) -> validate envelope/schema/hash/freshness/policy ->
  valid: use
  suspicious: revalidate from origin
  invalid: evict + rebuild or block
```

## 2. 四层校验模型

### 2.1 Envelope 校验：缓存外壳完整吗？

每个缓存条目至少要有：

- `schemaVersion`
- `valueHash`
- `createdAt / observedAt`
- `sourceVersion`
- `purpose`
- `risk`
- `lineage`

缺任何关键字段，直接当 invalid，不要让 LLM 猜。

### 2.2 Schema 校验：结构还能读吗？

用 JSON Schema / Zod / Pydantic 验证 value：

- 字段类型；
- 必填字段；
- enum 是否仍合法；
- 数值范围是否合理；
- 新旧 schema migration 是否成功。

### 2.3 Invariant 校验：业务上合理吗？

结构正确不代表事实合理，例如：

- `totalCost >= 0`
- `currency in ["USD", "AUD"]`
- `deployment.status` 只能从 running/cancelled/failed 等状态集合来；
- `expiresAt > observedAt`
- `tenantId` 必须等于当前 scope。

这些不变量失败，比 miss 更危险，因为它会给 Agent 一个看似可信的错误事实。

### 2.4 Freshness / Policy 校验：这个用途还能用吗？

同一缓存对不同用途的要求不同：

- `answer`：可以接受几分钟旧数据，并标注 caveat；
- `planning`：可以用 stale 数据做候选计划；
- `external_side_effect`：必须回源确认；
- `security_decision`：通常禁止使用派生缓存。

校验器必须知道 `purpose`，不能只看 TTL。

## 3. learn-claude-code：Python 教学版

```python
# learn-claude-code/cache_validation.py
from dataclasses import dataclass
from datetime import datetime, timezone, timedelta
from typing import Literal, Callable, Any
import hashlib
import json

Purpose = Literal["answer", "planning", "external_side_effect", "security_decision"]
Decision = Literal["use", "revalidate", "evict", "block"]


def stable_hash(value: Any) -> str:
    body = json.dumps(value, sort_keys=True, ensure_ascii=False, separators=(",", ":"))
    return "sha256:" + hashlib.sha256(body.encode()).hexdigest()


@dataclass
class CacheEnvelope:
    key: str
    value: dict
    value_hash: str
    schema_version: int
    observed_at: str
    source_version: str
    purpose: Purpose
    risk: Literal["low", "medium", "high"]


@dataclass
class ValidationResult:
    decision: Decision
    reasons: list[str]


class CacheValidator:
    def __init__(self, *, current_schema: int, max_age_by_purpose: dict[Purpose, timedelta]):
        self.current_schema = current_schema
        self.max_age_by_purpose = max_age_by_purpose
        self.invariants: list[Callable[[CacheEnvelope], str | None]] = []

    def add_invariant(self, check: Callable[[CacheEnvelope], str | None]) -> None:
        self.invariants.append(check)

    def validate(self, env: CacheEnvelope, requested_purpose: Purpose) -> ValidationResult:
        reasons: list[str] = []

        if env.value_hash != stable_hash(env.value):
            return ValidationResult("evict", ["value_hash_mismatch"])

        if env.schema_version < self.current_schema:
            return ValidationResult("revalidate", ["schema_version_old"])

        if requested_purpose in ("external_side_effect", "security_decision"):
            return ValidationResult("revalidate", [f"purpose_requires_live_check:{requested_purpose}"])

        observed_at = datetime.fromisoformat(env.observed_at)
        age = datetime.now(timezone.utc) - observed_at
        if age > self.max_age_by_purpose[requested_purpose]:
            reasons.append("stale_for_purpose")

        for check in self.invariants:
            if reason := check(env):
                return ValidationResult("evict", [reason])

        if reasons:
            return ValidationResult("revalidate", reasons)
        return ValidationResult("use", ["valid"])
```

加两个业务不变量：

```python
validator = CacheValidator(
    current_schema=3,
    max_age_by_purpose={
        "answer": timedelta(minutes=15),
        "planning": timedelta(minutes=5),
        "external_side_effect": timedelta(seconds=0),
        "security_decision": timedelta(seconds=0),
    },
)

validator.add_invariant(lambda env: "negative_total" if env.value.get("total", 0) < 0 else None)
validator.add_invariant(lambda env: "tenant_mismatch" if env.value.get("tenant") != "metadraw" else None)
```

教学版重点：

1. hash 不匹配直接 evict；
2. schema 旧了先 revalidate，不要直接喂给 LLM；
3. 高风险用途强制 live check；
4. 业务 invariant 失败要比普通过期更严肃。

## 4. pi-mono：Validation Middleware

```ts
// pi-mono/cache/CacheValidationMiddleware.ts
type Purpose = "answer" | "planning" | "external_side_effect" | "security_decision";
type Decision = "use" | "revalidate" | "evict" | "block";

type CacheEnvelope<T> = {
  key: string;
  value: T;
  valueHash: string;
  schemaVersion: number;
  observedAt: string;
  sourceVersion: string;
  tenantId: string;
  purpose: Purpose;
  risk: "low" | "medium" | "high";
};

type ValidationContext = {
  requestedPurpose: Purpose;
  tenantId: string;
  now: Date;
};

type ValidationResult = {
  decision: Decision;
  reasons: string[];
  requireLiveCheck?: boolean;
};

export class CacheValidationMiddleware {
  constructor(
    private hash: (value: unknown) => string,
    private currentSchemaVersion: number,
    private origin: { fetchFresh<T>(key: string): Promise<CacheEnvelope<T>> },
    private audit: (event: string, payload: Record<string, unknown>) => void,
  ) {}

  async read<T>(entry: CacheEnvelope<T> | null, ctx: ValidationContext): Promise<T | null> {
    if (!entry) return null;

    const result = this.validate(entry, ctx);
    this.audit("cache.validation", {
      key: entry.key,
      decision: result.decision,
      reasons: result.reasons,
      purpose: ctx.requestedPurpose,
    });

    if (result.decision === "use") return entry.value;

    if (result.decision === "evict") {
      // caller 可在外层删除 key，这里只表达决策
      return null;
    }

    if (result.decision === "block") {
      throw new Error(`cache blocked: ${result.reasons.join(",")}`);
    }

    // revalidate：回源取新值，再跑一次校验；新值仍不通过就拒绝使用
    const fresh = await this.origin.fetchFresh<T>(entry.key);
    const freshResult = this.validate(fresh, ctx);
    if (freshResult.decision !== "use") {
      this.audit("cache.revalidation_failed", {
        key: entry.key,
        reasons: freshResult.reasons,
      });
      return null;
    }

    return fresh.value;
  }

  private validate<T>(entry: CacheEnvelope<T>, ctx: ValidationContext): ValidationResult {
    if (entry.valueHash !== this.hash(entry.value)) {
      return { decision: "evict", reasons: ["value_hash_mismatch"] };
    }

    if (entry.tenantId !== ctx.tenantId) {
      return { decision: "block", reasons: ["tenant_mismatch"] };
    }

    if (entry.schemaVersion < this.currentSchemaVersion) {
      return { decision: "revalidate", reasons: ["schema_version_old"] };
    }

    if (["external_side_effect", "security_decision"].includes(ctx.requestedPurpose)) {
      return {
        decision: "revalidate",
        reasons: [`purpose_requires_live_check:${ctx.requestedPurpose}`],
        requireLiveCheck: true,
      };
    }

    const ageMs = ctx.now.getTime() - new Date(entry.observedAt).getTime();
    if (ageMs > 15 * 60_000) {
      return { decision: "revalidate", reasons: ["stale_for_purpose"] };
    }

    return { decision: "use", reasons: ["valid"] };
  }
}
```

生产版要注意：

- `revalidate` 要走 miss budget，不能无限回源；
- `block` 和 `evict` 都要进审计；
- fresh value 写回缓存时必须更新 hash、observedAt、lineage；
- 对高风险用途，回源失败不能偷偷降级成 stale。

## 5. OpenClaw 实战：课程 Cron 的校验清单

这类定时课程任务也可以看成一个缓存/状态读取流程：

1. 读取 TOOLS.md 已讲内容；
2. 读取 lessons 最新编号；
3. 生成新课程；
4. 更新 README 和 TOOLS；
5. 发 Telegram；
6. git commit/push。

执行前后可以加 validation gates：

```json
{
  "lesson": 276,
  "requiredChecks": [
    "lesson_file_exists",
    "readme_contains_276",
    "tools_contains_topic",
    "telegram_message_id_present",
    "git_status_clean_after_push"
  ]
}
```

如果 README 没更新，不能算完成；如果 Telegram 发送失败，不能只提交 repo；如果 push 前远端 main 变化，必须 rebase 后再验证。

这就是 Cache Validation 的思路迁移到 Agent 自动化：**不要相信中间状态，所有关键状态在使用前/完成前都要可执行校验。**

## 6. Checklist

- [ ] 缓存 envelope 是否带 schemaVersion/valueHash/observedAt/sourceVersion/purpose/risk？
- [ ] valueHash 是否每次读取都验证？
- [ ] schema 旧版本是否有 migration 或 revalidate 路径？
- [ ] 是否有业务 invariant，而不是只靠类型检查？
- [ ] 高风险 purpose 是否强制 live check？
- [ ] revalidate 是否受 miss budget / circuit breaker 保护？
- [ ] revalidate 失败时是否明确 block，而不是静默使用 stale？
- [ ] 所有 validation decision 是否进入审计日志？

## 7. 今日总结

缓存系统的成熟度，可以用一句话判断：

> 你是否能解释每一次 cache hit 为什么可以被当前用途使用？

做不到，就说明缓存只是“快”；做得到，才说明缓存“可信”。

下一次给 Agent 加缓存时，不要只写 `get/set`，先写 `validate/revalidate`。**缓存命中不是终点，可信命中才是终点。**
