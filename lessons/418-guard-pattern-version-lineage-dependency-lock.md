# 418. Agent 护栏模式版本血缘与依赖锁定（Guard Pattern Version Lineage & Dependency Lock）

上一课讲了 **Guard Pattern Evidence Refresh & Predicate Repair**：pattern 变差时，先刷新证据、修 predicate、跑 replay，再决定晋级、观察、收窄或退役。

今天继续补一个生产系统必须有的底座：**修好的 pattern 不能原地覆盖，要像代码发布一样留下版本血缘和依赖锁。**

一句话：**Guard Pattern 要拆成稳定 patternId 和不可变 PatternVersion Manifest，记录 parentVersion、changeReason、predicateHash、dependencyLock、replayProof 和 activation scope；后续 outcome、回滚、审计都按版本归因，避免“旧事故用新版解释，新误伤找不到源头”。**

---

## 1. 为什么 Pattern 不能原地修改

Pattern KB 一旦参与 guard routing、replay selection、release gate 或 human escalation，它就不再是文档，而是运行时安全逻辑。

如果 repair worker 直接覆盖原 pattern，会出现四个问题：

1. 旧 outcome 无法解释：上周的 false match 到底是哪版 predicate 造成的；
2. 回滚没有目标：新版误伤后，只知道“退回去”，但不知道退回哪个 hash；
3. 依赖漂移不可见：tool schema、guard pack、risk surface 改过后，pattern 看起来还 active，实际语义已经变了；
4. 审计链断裂：messageId、commit、replay report、evidence refs 对不上同一个版本。

所以 pattern 必须 versioned：patternId 稳定不变，patternVersion 不可变；任何 predicate、scope、evidence、recommendedAction 的变化都生成新版本。

---

## 2. PatternVersion Manifest 的最小字段

一份可审计的 PatternVersion Manifest 至少要回答：从哪来、改了什么、依赖什么、用什么证据证明可以生效。

~~~text
PatternVersionManifest:
  patternId
  version
  parentVersion
  lifecycleState: shadow | canary | active | observe | retired
  createdBy
  changeReason:
    initial_mining
    evidence_refresh
    predicate_repair
    scope_tightening
    dependency_migration
    rollback
  predicateHash
  recommendedActionHash
  scope:
    riskSurfaces
    tenants
    actionClasses
  evidenceRefs
  dependencyLock:
    guardPackVersion
    toolSchemaHashes
    promptPackHash
    replayPackHash
    riskSurfaceCatalogHash
    policyHash
  replayProofRef
  activationWindow
  rollbackTargetVersion
~~~

关键点：**manifest 不保存大段 raw evidence，只保存 refs 和 hash。** Raw evidence 由 Evidence Store 管，pattern 只持有可校验引用。

---

## 3. learn-claude-code：纯函数创建 Manifest

教学版先用一个纯函数生成不可变 manifest。它不碰数据库，也不发布，只负责把候选版本规范化。

~~~py
from dataclasses import dataclass
from hashlib import sha256
from typing import Literal

LifecycleState = Literal["shadow", "canary", "active", "observe", "retired"]
ChangeReason = Literal[
    "initial_mining",
    "evidence_refresh",
    "predicate_repair",
    "scope_tightening",
    "dependency_migration",
    "rollback",
]


@dataclass(frozen=True)
class DependencyLock:
    guard_pack_version: str
    tool_schema_hashes: dict[str, str]
    prompt_pack_hash: str
    replay_pack_hash: str
    risk_surface_catalog_hash: str
    policy_hash: str


@dataclass(frozen=True)
class PatternVersionManifest:
    pattern_id: str
    version: int
    parent_version: int | None
    lifecycle_state: LifecycleState
    change_reason: ChangeReason
    predicate_hash: str
    recommended_action_hash: str
    evidence_refs: tuple[str, ...]
    dependency_lock: DependencyLock
    replay_proof_ref: str
    rollback_target_version: int | None


def stable_hash(value: str) -> str:
    return sha256(value.encode("utf-8")).hexdigest()


def create_pattern_manifest(
    *,
    pattern_id: str,
    next_version: int,
    parent_version: int | None,
    change_reason: ChangeReason,
    predicate_source: str,
    recommended_action: str,
    evidence_refs: list[str],
    dependency_lock: DependencyLock,
    replay_proof_ref: str,
    rollback_target_version: int | None,
) -> PatternVersionManifest:
    if next_version <= 0:
        raise ValueError("version must be positive")
    if change_reason != "initial_mining" and parent_version is None:
        raise ValueError("non-initial version requires parent_version")
    if not evidence_refs:
        raise ValueError("pattern version requires evidence refs")
    if not replay_proof_ref:
        raise ValueError("pattern version requires replay proof")

    return PatternVersionManifest(
        pattern_id=pattern_id,
        version=next_version,
        parent_version=parent_version,
        lifecycle_state="shadow",
        change_reason=change_reason,
        predicate_hash=stable_hash(predicate_source),
        recommended_action_hash=stable_hash(recommended_action),
        evidence_refs=tuple(sorted(set(evidence_refs))),
        dependency_lock=dependency_lock,
        replay_proof_ref=replay_proof_ref,
        rollback_target_version=parent_version,
    )
~~~

这里默认新版本先进入 shadow。即使 repair replay 通过，也不等于可以立刻 active；是否 active 由发布闸门决定。

---

## 4. learn-claude-code：依赖漂移判定

有了 dependency lock 后，每次 pattern 参与高风险决策前，都可以先做 drift check。

~~~py
from dataclasses import dataclass
from typing import Literal

DriftAction = Literal[
    "allow_evaluate",
    "allow_observe_only",
    "require_replay",
    "block_and_manual_review",
]


@dataclass(frozen=True)
class RuntimeDeps:
    guard_pack_version: str
    tool_schema_hashes: dict[str, str]
    prompt_pack_hash: str
    replay_pack_hash: str
    risk_surface_catalog_hash: str
    policy_hash: str


def decide_pattern_dependency_drift(
    locked: DependencyLock,
    runtime: RuntimeDeps,
    affects_external_side_effects: bool,
) -> tuple[DriftAction, list[str]]:
    reasons: list[str] = []

    if locked.policy_hash != runtime.policy_hash:
        reasons.append("policy_hash_changed")
    if locked.risk_surface_catalog_hash != runtime.risk_surface_catalog_hash:
        reasons.append("risk_surface_catalog_changed")
    if locked.guard_pack_version != runtime.guard_pack_version:
        reasons.append("guard_pack_version_changed")
    if locked.prompt_pack_hash != runtime.prompt_pack_hash:
        reasons.append("prompt_pack_hash_changed")
    if locked.replay_pack_hash != runtime.replay_pack_hash:
        reasons.append("replay_pack_hash_changed")

    changed_tools = [
        name for name, old_hash in locked.tool_schema_hashes.items()
        if runtime.tool_schema_hashes.get(name) != old_hash
    ]
    if changed_tools:
        reasons.append("tool_schema_changed:" + ",".join(sorted(changed_tools)))

    if not reasons:
        return "allow_evaluate", ["dependency_lock_matches_runtime"]

    hard_drift = any(
        r.startswith("policy_hash")
        or r.startswith("risk_surface_catalog")
        or r.startswith("tool_schema_changed")
        for r in reasons
    )
    if hard_drift and affects_external_side_effects:
        return "block_and_manual_review", reasons
    if hard_drift:
        return "require_replay", reasons
    return "allow_observe_only", reasons
~~~

这个判断的原则很简单：policy、risk surface、tool schema 变了，就是语义级漂移；影响外部副作用时不能继续 enforce。

---

## 5. pi-mono：PatternVersion 事件

生产版里，不要让 Pattern Store 默默更新一行记录。每个版本变化都应该发事件。

~~~ts
type PatternVersionManifest = {
  patternId: string;
  version: number;
  parentVersion?: number;
  lifecycleState: "shadow" | "canary" | "active" | "observe" | "retired";
  changeReason:
    | "initial_mining"
    | "evidence_refresh"
    | "predicate_repair"
    | "scope_tightening"
    | "dependency_migration"
    | "rollback";
  predicateHash: string;
  recommendedActionHash: string;
  scope: {
    riskSurfaces: string[];
    tenants: string[];
    actionClasses: string[];
  };
  evidenceRefs: string[];
  dependencyLock: {
    guardPackVersion: string;
    toolSchemaHashes: Record<string, string>;
    promptPackHash: string;
    replayPackHash: string;
    riskSurfaceCatalogHash: string;
    policyHash: string;
  };
  replayProofRef: string;
  rollbackTargetVersion?: number;
  createdAt: Date;
};

type PatternVersionCreated = {
  type: "guard_pattern.version_created";
  eventId: string;
  manifest: PatternVersionManifest;
};

type PatternVersionActivated = {
  type: "guard_pattern.version_activated";
  eventId: string;
  patternId: string;
  version: number;
  previousActiveVersion?: number;
  activationScope: PatternVersionManifest["scope"];
  activationReceiptRef: string;
  activatedAt: Date;
};
~~~

PatternVersionCreated 说明候选版本存在；PatternVersionActivated 说明某个 scope 里 runtime 已经开始使用它。这两个事件不要混在一起。

---

## 6. pi-mono：版本发布 Worker

发布 worker 只做一件事：检查 manifest 的 replay proof 和依赖锁，再把版本从 shadow/canary 推到 active。

~~~ts
class PatternVersionReleaseWorker {
  constructor(
    private readonly patterns: PatternStore,
    private readonly deps: RuntimeDependencyRegistry,
    private readonly replay: ReplayProofStore,
    private readonly eventBus: EventBus,
  ) {}

  async promoteToActive(patternId: string, version: number) {
    const manifest = await this.patterns.getManifest(patternId, version);
    const proof = await this.replay.get(manifest.replayProofRef);

    if (!proof.passed || proof.highRiskFalsePositiveCount > 0) {
      throw new Error("pattern replay proof is not promotable");
    }

    const runtimeDeps = await this.deps.snapshot();
    const drift = compareDependencyLock(manifest.dependencyLock, runtimeDeps);
    if (drift.hardDrift.length > 0) {
      await this.eventBus.publish({
        type: "guard_pattern.release_blocked",
        eventId: crypto.randomUUID(),
        patternId,
        version,
        reasons: drift.hardDrift,
        createdAt: new Date(),
      });
      return;
    }

    const previous = await this.patterns.getActiveVersion(patternId, manifest.scope);
    const receipt = await this.patterns.activateVersion({
      patternId,
      version,
      scope: manifest.scope,
      expectedPredicateHash: manifest.predicateHash,
    });

    await this.eventBus.publish({
      type: "guard_pattern.version_activated",
      eventId: crypto.randomUUID(),
      patternId,
      version,
      previousActiveVersion: previous?.version,
      activationScope: manifest.scope,
      activationReceiptRef: receipt.ref,
      activatedAt: new Date(),
    });
  }
}
~~~

注意 expectedPredicateHash：发布时再校验一次，防止候选版本被并发修改或存储层读取到了不一致内容。

---

## 7. OpenClaw 课程 Cron 的实战映射

这个课程 cron 也需要类似的版本血缘。每一课不是“发完就算”，而是要能证明这一版课程发布链完整：

~~~text
patternId: agent_course_topic_selection
version: lesson-418
parentVersion: lesson-417
dependencyLock:
  TOOLS.md taught list hash
  README.md catalog hash
  latest remote main commit
  Telegram target chat id
replayProofRef:
  rg 去重结果
  git diff --check
  git diff --cached --check
activationReceipt:
  Telegram messageId
  git commit sha
  git ls-remote confirmation
~~~

如果 README 写了第 418 课，但 Telegram 没发成功，这不是“半成功”。activation receipt 缺 messageId，版本就不能视为 active；下次 cron 要按 evidence 恢复或补发，而不是重新生成一个相似主题造成重复。

---

## 8. 常见坑

1. 只记录最新版本：审计时解释不了旧 outcome。
2. parentVersion 可选乱填：非 initial 版本必须有 parent。
3. dependency lock 只锁 guard pack：tool schema、policy、risk surface 变了也会改变 pattern 语义。
4. activation 和 creation 混在一起：候选版本存在，不代表运行时已经使用。
5. 回滚只改 active 指针：还要写 rollback reason、rollback target 和 activation receipt。

---

## 9. 落地清单

给 Guard Pattern KB 加版本血缘，可以按这个顺序做：

1. Pattern Store 拆成 Pattern 和 PatternVersion；
2. 每个版本写不可变 manifest；
3. predicate、recommendedAction、scope、evidence 变化都生成新版本；
4. dependencyLock 锁住 guard pack、tool schema、prompt pack、policy、risk surface、replay pack；
5. 发布前做 dependency drift check；
6. active 切换写 activation receipt；
7. outcome、rollback、repair case 全部引用具体 version。

---

## 10. 记住

Pattern KB 的版本化不是形式主义，而是把“经验”变成可发布、可回滚、可归因的工程资产：

~~~text
稳定 patternId 负责延续历史；不可变 version 负责解释事实；dependency lock 负责证明当前运行时还配得上这条经验。
~~~

成熟 Agent 不只会从事故里抽取 pattern，还能说清楚：这条 pattern 从哪版来，依赖什么，什么时候生效，为什么现在还能信，坏了退回哪里。
