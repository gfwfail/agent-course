# 360. Agent 学习规则 Schema 迁移与兼容闸门（Learning Rule Schema Migration & Compatibility Gate）

上一课讲了 Learning Rule Sunsetting：学习规则不能直接删除，要通过 deprecated / retired / archived 留下退场理由、影响指标和清理证据。

今天继续补一个更底层、但上线后必然会遇到的问题：**学习规则自己的 Schema 会变。**

早期 learning 可能只是一段文字：

~~~json
{
  "rule": "git push 前检查 PR 状态"
}
~~~

后来你会加 scope、evidenceRefs、riskSurface、expiresAt、verification、dependencyLock、releaseStage。再后来，policy 系统要求每条规则声明 decisionImpact，retrieval 系统要求每条规则声明 retrievalTags。

如果没有迁移层，Agent 很容易犯两种错：

- 把旧规则当新规则读，缺字段就用危险默认值，导致高风险动作被错误放行；
- 为了兼容旧字段，到处写 if oldField exists，半年后没人知道哪条字段才可信。

一句话：**学习规则是长期资产，Schema 升级必须像数据库 migration 一样可版本化、可验证、可降级。**

## 1. 为什么 learning rule 需要 schemaVersion

看一个危险反例：

~~~ts
function loadLearningRule(raw: any) {
  return {
    id: raw.id,
    rule: raw.rule,
    scope: raw.scope ?? { all: true },
    decisionImpact: raw.decisionImpact ?? "allow",
  };
}
~~~

这里最危险的是默认值：

- 旧规则没有 scope，就默认全局生效；
- 旧规则没有 decisionImpact，就默认可以影响 allow；
- 旧规则没有 evidenceRefs，却被当成可信学习；
- 旧规则没有 expiresAt，就永不过期。

兼容不是“猜一个默认值继续跑”。兼容应该先回答：

- 这条规则原始 Schema 是哪一版；
- 它能不能迁移到当前 Schema；
- 迁移过程有没有 synthetic/default 字段；
- 迁移后的规则能不能用于当前 purpose；
- 高风险动作是否要求原生新 Schema，而不是迁移兼容版。

## 2. 目标 Schema：把规则、证据、影响范围拆开

一个可演进的 Learning Rule Envelope 可以这样设计：

~~~json
{
  "schemaName": "learning.rule",
  "schemaVersion": 4,
  "learningId": "learn.git.push.pr-state-check",
  "version": 6,
  "stage": "active",
  "rule": {
    "condition": "before_git_push",
    "action": "run_pr_state_preflight",
    "rationale": "avoid pushing to merged or closed PR branches"
  },
  "scope": {
    "capabilities": ["git_push"],
    "repos": ["*"],
    "riskSurface": ["external_side_effect", "source_control"]
  },
  "decisionImpact": {
    "canAllow": false,
    "canDeny": true,
    "canRequireApproval": true
  },
  "retrieval": {
    "tags": ["git", "push", "pr", "safety"],
    "priority": 80
  },
  "evidenceRefs": [
    "memory://MEMORY.md#git-push-rule",
    "incident://wrong-branch-push-2026-05"
  ],
  "compatibility": {
    "nativeSchemaVersion": 4,
    "migratedFrom": null,
    "syntheticFields": []
  }
}
~~~

注意 compatibility 这块。它明确告诉后续闸门：

- nativeSchemaVersion：原始写入时就是哪一版；
- migratedFrom：如果是旧规则迁移上来的，记录来源版本；
- syntheticFields：哪些字段不是历史事实，而是迁移时补出来的。

这能避免“读得懂”被误当成“完全可信”。

## 3. learn-claude-code：Migration Registry 教学版

教学版先用纯函数实现。核心规则：**读入口统一迁移，业务代码只接触最新版对象。**

~~~python
from dataclasses import dataclass, field
from typing import Any, Callable

CURRENT_SCHEMA_VERSION = 4

@dataclass(frozen=True)
class MigrationResult:
    value: dict[str, Any]
    synthetic_fields: list[str] = field(default_factory=list)
    warnings: list[str] = field(default_factory=list)

Migration = Callable[[dict[str, Any]], MigrationResult]

class LearningRuleMigrator:
    def __init__(self) -> None:
        self._migrations: dict[int, Migration] = {}

    def register(self, from_version: int, migration: Migration) -> None:
        self._migrations[from_version] = migration

    def migrate_to_current(self, raw: dict[str, Any]) -> MigrationResult:
        original_version = int(raw.get("schemaVersion", 1))
        version = original_version
        value = dict(raw)
        synthetic_fields: list[str] = []
        warnings: list[str] = []

        while version < CURRENT_SCHEMA_VERSION:
            migration = self._migrations.get(version)
            if migration is None:
                raise ValueError(f"missing migration v{version}->v{version + 1}")

            result = migration(value)
            value = result.value
            synthetic_fields.extend(result.synthetic_fields)
            warnings.extend(result.warnings)
            version = int(value["schemaVersion"])

        value.setdefault("compatibility", {})
        value["compatibility"] = {
            **value["compatibility"],
            "nativeSchemaVersion": original_version,
            "migratedFrom": None if original_version == CURRENT_SCHEMA_VERSION else original_version,
            "syntheticFields": synthetic_fields,
        }

        return MigrationResult(value=value, synthetic_fields=synthetic_fields, warnings=warnings)

def migrate_v1_to_v2(raw: dict[str, Any]) -> MigrationResult:
    migrated = {
        "schemaName": "learning.rule",
        "schemaVersion": 2,
        "learningId": raw["id"],
        "rule": {"condition": "unknown", "action": raw["rule"], "rationale": "legacy rule"},
        "scope": {"capabilities": ["unknown"], "riskSurface": ["unknown"]},
    }
    return MigrationResult(
        value=migrated,
        synthetic_fields=["rule.condition", "rule.rationale", "scope.capabilities", "scope.riskSurface"],
        warnings=["v1 rule has unknown scope; high-risk decisions must reject it"],
    )

def migrate_v2_to_v3(raw: dict[str, Any]) -> MigrationResult:
    raw = dict(raw)
    raw["schemaVersion"] = 3
    raw["decisionImpact"] = {
        "canAllow": False,
        "canDeny": True,
        "canRequireApproval": True,
    }
    return MigrationResult(
        value=raw,
        synthetic_fields=["decisionImpact"],
        warnings=["decisionImpact synthesized as deny-only for safety"],
    )

def migrate_v3_to_v4(raw: dict[str, Any]) -> MigrationResult:
    raw = dict(raw)
    raw["schemaVersion"] = 4
    raw["retrieval"] = {
        "tags": list(set(raw.get("scope", {}).get("capabilities", []))),
        "priority": 50,
    }
    raw.setdefault("evidenceRefs", [])
    return MigrationResult(
        value=raw,
        synthetic_fields=["retrieval", "evidenceRefs"],
        warnings=["missing evidenceRefs means this rule cannot support side effects"],
    )

migrator = LearningRuleMigrator()
migrator.register(1, migrate_v1_to_v2)
migrator.register(2, migrate_v2_to_v3)
migrator.register(3, migrate_v3_to_v4)
~~~

这里有两个重点：

1. migration 是链式的，v1 先到 v2，再到 v3，再到 v4；
2. 每次补字段都记录 synthetic_fields 和 warnings，而不是悄悄假装原始数据本来就有。

## 4. 兼容闸门：能读，不代表能用于所有动作

迁移只是第一步。更关键的是 purpose gate：迁移后的规则能不能用于当前动作。

~~~python
def can_use_learning_rule(
    rule: dict[str, Any],
    *,
    purpose: str,
    risk_surface: str,
) -> tuple[bool, str]:
    compatibility = rule.get("compatibility", {})
    synthetic = set(compatibility.get("syntheticFields", []))
    native_version = int(compatibility.get("nativeSchemaVersion", 1))
    evidence_refs = rule.get("evidenceRefs", [])

    if purpose == "external_side_effect":
        if native_version < 4:
            return False, "side_effect_requires_native_schema_v4"
        if "evidenceRefs" in synthetic or not evidence_refs:
            return False, "side_effect_requires_real_evidence_refs"
        if "scope.riskSurface" in synthetic:
            return False, "side_effect_requires_explicit_risk_surface"

    if risk_surface == "security_decision":
        impact = rule.get("decisionImpact", {})
        if impact.get("canAllow") is True and "decisionImpact" in synthetic:
            return False, "synthetic_allow_decision_forbidden"

    return True, "ok"
~~~

这就是兼容闸门的核心：

- 普通检索 / explain 可以使用迁移规则；
- 高风险副作用必须要求原生新 Schema；
- synthetic 字段只能辅助理解，不能支撑关键放行；
- 如果旧规则想继续影响高风险动作，必须走 re-author / replay / canary，生成新版本。

## 5. pi-mono：把 migration 放在中间件外层

在生产系统里，不要让每个 Agent step 自己处理旧字段。更好的做法是把它放在 Memory / Policy / Retrieval 的边界。

~~~ts
type LearningRuleV4 = {
  schemaName: "learning.rule";
  schemaVersion: 4;
  learningId: string;
  version: number;
  stage: "shadow" | "canary" | "active" | "deprecated" | "retired";
  rule: {
    condition: string;
    action: string;
    rationale: string;
  };
  scope: {
    capabilities: string[];
    riskSurface: string[];
  };
  decisionImpact: {
    canAllow: boolean;
    canDeny: boolean;
    canRequireApproval: boolean;
  };
  retrieval: {
    tags: string[];
    priority: number;
  };
  evidenceRefs: string[];
  compatibility: {
    nativeSchemaVersion: number;
    migratedFrom: number | null;
    syntheticFields: string[];
  };
};

type MigrationOutcome = {
  rule: LearningRuleV4;
  warnings: string[];
};

class LearningRuleStore {
  constructor(
    private readonly rawStore: { load(id: string): Promise<unknown> },
    private readonly migrator: { migrate(raw: unknown): MigrationOutcome },
    private readonly audit: { write(event: unknown): Promise<void> },
  ) {}

  async loadCurrent(id: string): Promise<LearningRuleV4> {
    const raw = await this.rawStore.load(id);
    const outcome = this.migrator.migrate(raw);

    if (outcome.rule.compatibility.migratedFrom !== null) {
      await this.audit.write({
        type: "learning_rule.migrated_read",
        learningId: outcome.rule.learningId,
        version: outcome.rule.version,
        fromSchema: outcome.rule.compatibility.migratedFrom,
        toSchema: outcome.rule.schemaVersion,
        syntheticFields: outcome.rule.compatibility.syntheticFields,
        warnings: outcome.warnings,
        at: new Date().toISOString(),
      });
    }

    return outcome.rule;
  }
}
~~~

pi-mono 的思路可以放在 Agent.subscribe / middleware 外层：Agent loop 拿到的是标准 V4 rule，事件流里记录 migrated_read。这样后续排查时能回答：

- 哪些旧规则仍被读取；
- 哪些字段是迁移补出来的；
- 哪些 action 因兼容闸门被拒绝；
- 是否需要批量 re-author 某类旧规则。

## 6. OpenClaw 实战：文件式 learning migration

OpenClaw 这种长期运行 Agent 很适合用文件做最小闭环：

~~~text
.openclaw/
  learning/
    rules/
      learn.git.push.pr-state-check/
        v1.json
        v2.json
    schema/
      learning.rule.v1.json
      learning.rule.v2.json
      learning.rule.v3.json
      learning.rule.v4.json
    migrations/
      v1_to_v2.py
      v2_to_v3.py
      v3_to_v4.py
    audit/
      migration-events.jsonl
~~~

每次 heartbeat / cron 可以做一个轻量巡检：

~~~python
def scan_legacy_rules(rule_paths: list[str], migrator: LearningRuleMigrator) -> list[dict]:
    report = []
    for path in rule_paths:
        raw = load_json(path)
        result = migrator.migrate_to_current(raw)
        rule = result.value

        allowed, reason = can_use_learning_rule(
            rule,
            purpose="external_side_effect",
            risk_surface="source_control",
        )

        report.append({
            "path": path,
            "learningId": rule["learningId"],
            "nativeSchemaVersion": rule["compatibility"]["nativeSchemaVersion"],
            "syntheticFields": rule["compatibility"]["syntheticFields"],
            "sideEffectEligible": allowed,
            "reason": reason,
        })
    return report
~~~

这个 report 可以直接变成运维动作：

- 只是 explain 用的旧规则：继续兼容；
- 影响 side effect 的旧规则：创建 re-author ticket；
- 缺 evidenceRefs 的旧规则：降级 deprecated；
- 迁移失败的旧规则：进入 quarantine，禁止进入 prompt。

## 7. 常见坑

**坑一：默认全局 scope。**

旧规则缺 scope 时，不能默认全局。应该默认 unknown，并禁止高风险自动决策。

**坑二：迁移时覆盖原始文件。**

迁移读取和迁移写入要分开。读路径可以动态迁移，真正写入新版本必须生成新的 immutable version manifest。

**坑三：migration 没有测试。**

每个 migration 都要有 golden case：输入旧 JSON，输出新 JSON，synthetic_fields 和 warnings 必须稳定。

**坑四：把 synthetic evidence 当真实证据。**

evidenceRefs: [] 和“没有 evidenceRefs 字段”不是一回事。前者是明确没有证据，后者是旧 Schema 不知道证据概念。

**坑五：只升不降。**

有些外部系统只支持旧格式。Schema Registry 应该支持 downcast，但 downcast 后必须标注 lossOfInformation，不能用于审计闭环。

## 8. 什么时候该强制 re-author

这些情况不要靠 migration 继续撑：

- 规则会影响 canAllow=true 的安全决策；
- 规则会触发外部副作用，比如发消息、push、部署、付款；
- 迁移补出了 scope / riskSurface / decisionImpact；
- 原始 evidenceRefs 缺失；
- 旧规则来自低权限上下文或无法确认来源；
- dependencyLock 已经漂移，replay pack 也过期。

强制 re-author 的意思是：让规则重新走质量闸门、冲突检查、反事实评估、shadow/canary，再生成当前 Schema 的新版本。

## 9. 落地清单

生产里最小可用版本：

1. 每条 learning rule 必须有 schemaName 和 schemaVersion；
2. 读入口统一通过 Migration Registry；
3. migration 结果记录 nativeSchemaVersion、migratedFrom、syntheticFields；
4. 高风险 purpose 使用 Compatibility Gate；
5. migrated read 写审计日志；
6. migration golden tests 进 CI；
7. 定期扫描旧规则，生成 re-author / deprecate / quarantine 建议。

## 10. 关键 takeaway

学习规则不是 prompt 里的几句话，而是会长期影响 Agent 行为的生产资产。

Schema 迁移的目标不是“旧数据还能读”，而是：

- 旧规则能被诚实理解；
- 缺失字段不会被危险默认值掩盖；
- synthetic 字段不会支撑高风险放行；
- 真正重要的规则会被重新验证、重新发布、重新留证。

成熟 Agent 的学习系统，必须能在系统升级后继续安全地理解旧经验，也能清楚地说：**这条旧经验我读得懂，但现在还不能拿它做关键决策。**
