# 401. Agent 回归护栏包 Schema 兼容与迁移闸门（Regression Guard Pack Schema Compatibility & Migration Gate）

上一课讲了 **Regression Guard Dependency Graph**：guard 多起来以后，不能谁最后执行谁说了算，要用依赖图和冲突仲裁收敛成可审计决策。

今天继续补一个生产里很现实的问题：**Guard Pack 本身也会升级**。

一句话：**Regression Guard Pack 是运行时安全契约；升级它的 schema 时，旧 guard 要能被迁移、被标记能力缺口，并且高风险动作必须通过兼容闸门后才能继续执行。**

---

## 1. 为什么 Guard Pack schema 会变成事故源

Guard Pack 一开始可能只有这些字段：

~~~json
{
  "guardId": "prevent_duplicate_push",
  "tier": "active",
  "decision": "block",
  "reasonCodes": ["duplicate_push"]
}
~~~

后来系统越来越成熟，会继续加字段：

- scope：只作用在某些 repo / tenant / channel；
- evidenceRequirements：命中时需要哪些证据；
- failClosed：评估失败时是否保守 block；
- dependencies：依赖哪些 guard 先跑；
- obligations：allow/dry_run 后必须履行哪些后置义务；
- migrationVersion：这条 guard 从哪个 schema 迁过来。

危险点在于：**旧 guard 能被 JSON parser 读出来，不等于它在新执行器里仍然安全。**

最常见的坏味道：

1. 新执行器默认缺失 scope 为 all，导致旧 guard 误伤全量流量；
2. 新执行器默认缺失 failClosed 为 false，导致异常时放行；
3. 新仲裁器要求 dependencies，但旧 guard 没声明，顺序不稳定；
4. 旧 evidenceRefs 没有 freshness，结果高风险副作用用了过期证据。

所以 Guard Pack schema 升级不能只是“字段可选”。它要有迁移、能力标记和执行前闸门。

---

## 2. 最小模型：GuardPackEnvelope

把 guard pack 外面包一层 envelope，不要让 runner 直接读裸数组：

~~~ts
type GuardPackEnvelope = {
  packId: string;
  schemaVersion: number;
  guardPackVersion: number;
  createdAt: string;
  minRunnerVersion: string;
  migration?: {
    fromSchemaVersion: number;
    migratedAt: string;
    migrationId: string;
    degradedCapabilities: string[];
  };
  guards: RegressionGuard[];
};

type RegressionGuard = {
  guardId: string;
  tier: "shadow" | "canary" | "active" | "retired";
  scope: {
    repos: string[];
    tenants: string[];
    channels: string[];
    riskSurfaces: string[];
  };
  decision: "allow" | "warn" | "dry_run" | "block";
  failClosed: boolean;
  evidenceRequirements: string[];
  dependsOn: string[];
  reasonCodes: string[];
};
~~~

这里的重点不是字段多，而是把三个事实显式化：

- schemaVersion：这包 guard 按哪个结构解释；
- minRunnerVersion：低版本 runner 不能硬跑新包；
- degradedCapabilities：迁移后仍缺什么能力，后面闸门要看。

兼容不是假装旧数据完美。兼容是把“不完美”写成机器能判断的状态。

---

## 3. learn-claude-code：最小迁移器

教学版可以先做一个纯函数：输入旧 guard pack，输出当前 schema，并记录降级能力。

~~~py
from dataclasses import dataclass, field
from datetime import datetime, timezone


CURRENT_SCHEMA = 3


@dataclass(frozen=True)
class MigrationResult:
    pack: dict
    degraded_capabilities: list[str] = field(default_factory=list)


def migrate_guard_pack(raw: dict) -> MigrationResult:
    version = raw.get("schemaVersion", 1)
    pack = dict(raw)
    degraded: list[str] = []

    if version == 1:
        guards = []
        for guard in pack.get("guards", []):
            migrated = dict(guard)

            # v1 没有 scope，不能默认扩大到全世界；先迁成 explicit narrow scope。
            migrated["scope"] = {
                "repos": guard.get("repos", []),
                "tenants": guard.get("tenants", []),
                "channels": guard.get("channels", []),
                "riskSurfaces": guard.get("riskSurfaces", ["unknown"]),
            }
            if not migrated["scope"]["repos"] and not migrated["scope"]["tenants"]:
                degraded.append(f"{guard['guardId']}:missing_explicit_scope")

            # v1 没有 failClosed，保守迁成 True。
            migrated["failClosed"] = True
            migrated["evidenceRequirements"] = guard.get("evidenceRequirements", [])
            migrated["dependsOn"] = guard.get("dependsOn", [])
            guards.append(migrated)

        pack["guards"] = guards
        pack["schemaVersion"] = 2
        version = 2

    if version == 2:
        for guard in pack.get("guards", []):
            if not guard.get("evidenceRequirements"):
                degraded.append(f"{guard['guardId']}:missing_evidence_requirements")

        pack["schemaVersion"] = 3

    if pack.get("schemaVersion") != CURRENT_SCHEMA:
        raise ValueError(f"unsupported guard pack schema: {pack.get('schemaVersion')}")

    pack["migration"] = {
        "fromSchemaVersion": raw.get("schemaVersion", 1),
        "migratedAt": datetime.now(timezone.utc).isoformat(),
        "migrationId": f"guard-pack-migration-v{CURRENT_SCHEMA}",
        "degradedCapabilities": sorted(set(degraded)),
    }
    return MigrationResult(pack=pack, degraded_capabilities=sorted(set(degraded)))
~~~

这里有两个关键选择：

1. 缺 failClosed 时迁成 True，安全优先；
2. 缺 scope / evidence 时不瞎补，写入 degradedCapabilities。

这样 runner 后面可以做更细的决策：低风险 read-only 可以 warn，高风险外部副作用必须 block 或 manual_review。

---

## 4. 兼容闸门：读得懂，不代表能执行

迁移成功后，还要过 Compatibility Gate。

~~~py
def compatibility_gate(pack: dict, action: dict) -> dict:
    degraded = pack.get("migration", {}).get("degradedCapabilities", [])
    side_effect = action["sideEffect"] in ("external", "irreversible")
    high_risk = action["risk"] in ("high", "critical")

    missing_scope = [d for d in degraded if d.endswith(":missing_explicit_scope")]
    missing_evidence = [d for d in degraded if d.endswith(":missing_evidence_requirements")]

    if side_effect and missing_scope:
        return {
            "decision": "block",
            "reason": "guard_pack_migrated_without_explicit_scope",
            "degradedCapabilities": missing_scope,
        }

    if high_risk and missing_evidence:
        return {
            "decision": "manual_review",
            "reason": "guard_pack_migrated_without_evidence_requirements",
            "degradedCapabilities": missing_evidence,
        }

    if degraded:
        return {
            "decision": "warn",
            "reason": "guard_pack_migrated_with_degraded_capabilities",
            "degradedCapabilities": degraded,
        }

    return {"decision": "allow", "reason": "guard_pack_schema_compatible"}
~~~

这个 gate 的价值是把兼容策略从“全局能不能用”变成“这次动作能不能用”：

- 读文件、总结日志：可以 warn 后继续；
- 发消息、git push、部署：缺 scope 直接 block；
- 人工恢复、补偿动作：缺 evidenceRequirements 要 manual_review；
- 完整新 schema：allow。

成熟系统不会因为旧包能解析就信任它，也不会因为旧包不完美就全站瘫痪。它按风险面降级。

---

## 5. pi-mono：在 GuardPackLoader 外层做 migrate + gate

生产版可以把这层放在 guard runner 之前：

~~~ts
type CompatibilityDecision =
  | { decision: "allow"; reason: string }
  | { decision: "warn"; reason: string; degradedCapabilities: string[] }
  | { decision: "manual_review"; reason: string; degradedCapabilities: string[] }
  | { decision: "block"; reason: string; degradedCapabilities: string[] };

class GuardPackLoader {
  constructor(
    private readonly store: GuardPackStore,
    private readonly migrations: GuardPackMigrationRegistry,
    private readonly compatibility: GuardPackCompatibilityGate,
    private readonly events: EventBus,
  ) {}

  async loadForAction(action: AgentAction): Promise<GuardPackEnvelope> {
    const raw = await this.store.currentFor(action.workflowId);
    const migrated = await this.migrations.toCurrent(raw);
    const decision = this.compatibility.evaluate(migrated, action);

    this.events.emit("guard_pack.compatibility_checked", {
      actionId: action.id,
      packId: migrated.packId,
      schemaVersion: migrated.schemaVersion,
      guardPackVersion: migrated.guardPackVersion,
      decision: decision.decision,
      reason: decision.reason,
      degradedCapabilities: "degradedCapabilities" in decision
        ? decision.degradedCapabilities
        : [],
    });

    if (decision.decision === "block") {
      throw new Error("GuardPack blocked: " + decision.reason);
    }

    if (decision.decision === "manual_review") {
      await this.store.createReviewCase({
        actionId: action.id,
        packId: migrated.packId,
        reason: decision.reason,
      });
      throw new Error("GuardPack requires manual review: " + decision.reason);
    }

    return migrated;
  }
}
~~~

这里 runner 不需要知道所有 schema 历史。它只拿“当前 schema + compatibility decision”。历史复杂度集中在 migration registry 和 compatibility gate。

这也符合 pi-mono 的生产习惯：把副作用和事件流做成边界清楚的 middleware，而不是散落到每个工具里。

---

## 6. OpenClaw 课程 Cron 的落地方式

拿这个每 3 小时课程任务举例，Guard Pack schema gate 可以这么用：

1. lesson guard：检查新课编号、主题是否重复；
2. delivery guard：检查 Telegram target 和 payload；
3. git guard：检查 gh 账号、远端、diff、push 前 PR 状态；
4. memory guard：检查 TOOLS 和 daily memory 是否更新。

如果 guard pack 从旧 schema 迁移过来：

- 缺 scope：不能发群、不能 push，只允许生成草稿；
- 缺 evidenceRequirements：不能自动 closeout，要人工 review；
- 缺 dependencies：可以 dry-run 生成 arbitration preview；
- 全部兼容：正常发布。

这能避免一个很隐蔽的问题：**课程任务本身看起来成功，但执行它的护栏其实已经因为 schema 演进失效。**

---

## 7. 常见坑

1. **把 optional 字段当兼容策略**：optional 只是不报错，不代表安全。
2. **迁移时默认扩大 scope**：缺 scope 时默认 all，是最危险的迁移。
3. **只在启动时迁移**：长任务和后台 worker 也要在 action 前检查当前 pack。
4. **没有记录 degradedCapabilities**：以后出问题只能猜旧数据缺了什么。
5. **低版本 runner 偷跑新 pack**：minRunnerVersion 必须强制校验。

---

## 8. 实战检查清单

- [ ] Guard Pack 有 schemaVersion、guardPackVersion、minRunnerVersion；
- [ ] 旧 schema 通过 migration registry 迁到当前 schema；
- [ ] 迁移不会静默扩大 scope；
- [ ] 缺失能力写入 degradedCapabilities；
- [ ] 高风险动作前跑 compatibility gate；
- [ ] block / manual_review / warn 都写事件；
- [ ] CI 里有旧 Guard Pack fixture 回放；
- [ ] 迁移后再跑 dependency graph 和 conflict arbitration。

---

## 总结

Guard Pack 是运行时安全契约，不是普通配置文件。

今天这课的核心：

- schema 能解析，不代表能安全执行；
- 迁移要保守，尤其不能静默扩大 scope；
- 旧 guard 缺失的能力要写进 degradedCapabilities；
- 高风险动作必须通过 compatibility gate；
- 迁移后还要重新跑依赖图和冲突仲裁。

成熟 Agent 的护栏系统，不只是 guard 本身可靠，还要保证 **guard 的版本、schema、迁移和执行器之间的契约持续可靠**。

