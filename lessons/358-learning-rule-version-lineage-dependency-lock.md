# 358. Agent 学习规则版本血缘与依赖锁定（Learning Rule Version Lineage & Dependency Lock）

上一课讲了 Learning Rollback Forensics：学习规则一旦被回滚，不能只 disable，而要生成 Rollback Case、分类 cause、创建 Repair Ticket、补 regression case，再重新走 shadow / canary / active。

今天继续补一块生产系统里很容易漏掉的东西：**学习规则每次修复、发布、回滚，都必须留下版本血缘和依赖锁。**

Agent 的学习规则不是孤立文本，它依赖很多外部条件：

- system prompt / policy 版本；
- 工具 schema 和工具行为版本；
- 证据文件、消息、commit、ticket 是否还存在；
- counterfactual replay 用的是哪组历史事件；
- canary 晋级时观察的是哪一个 SLO 窗口。

如果这些依赖没有锁住，半年后你看到一条 active learning，只知道“它好像能防某个坑”，但不知道它从哪学来、哪版生效、依赖什么、怎么验证、出事回到哪。

一句话：**学习规则不是一句记忆，而是一条可追溯、可复现、可回滚的发布链。**

## 1. 为什么 learning 需要版本血缘

看一个反例：

~~~json
{
  "learningId": "learn.git.push.pr-state-check",
  "rule": "git push 前检查 PR 状态",
  "stage": "active",
  "updatedAt": "2026-05-20T14:30:00Z"
}
~~~

这条记录太薄了。它没有回答几个关键问题：

- v1 是谁创建的？
- v2 改了什么？
- v3 为什么从 canary 晋级 active？
- 如果 v3 误伤，要回到 v2 还是禁用全部规则？
- v3 依赖的 `gh pr list` 输出格式是否已经变了？
- v3 的证据里是否包含老板明确要求“这次直接 push main”的例外？

成熟做法是把 learning 拆成两层：

- **Learning Rule**：稳定 ID，表示这类经验；
- **Learning Version**：不可变版本，表示某次具体规则内容、依赖和验证证据。

## 2. Learning Version Manifest

一个最小 Manifest 可以这样设计：

~~~json
{
  "learningId": "learn.git.push.pr-state-check",
  "version": 4,
  "parentVersion": 3,
  "stage": "canary",
  "createdBy": "repair-ticket:lrt-2026-05-20-358",
  "changeReason": "narrow direct-main exception for explicit cron instruction",
  "ruleHash": "sha256:8a91...",
  "scope": {
    "capabilities": ["git_push"],
    "repos": ["gfwfail/agent-course"],
    "denyWhen": ["branch_is_merged_pr"],
    "allowWhen": ["explicit_user_instruction_direct_main", "cron_course_publish"]
  },
  "dependencyLock": {
    "policyVersion": "policy.git-safety@2026-05-20",
    "toolSchemas": {
      "gh.pr.list": "sha256:5e41...",
      "git.status": "sha256:91ab..."
    },
    "promptPack": "system-prompt@2026-05-20T00:30+11",
    "evidenceRefs": [
      "memory://2026-05-19.md#agent-course-cron-21-30",
      "git://gfwfail/agent-course@d52038e",
      "telegram://-5115329245/12278"
    ],
    "replayPack": "regression-pack://git-push-safety@v7"
  },
  "verification": {
    "counterfactualReplay": "passed",
    "matchedCases": 42,
    "preventedBadAllows": 3,
    "falsePositives": 0,
    "unknowns": 1
  }
}
~~~

这里最重要的是 **dependencyLock**。它不是形式主义，而是为了以后能判断：

- 当前工具 schema 变了没有；
- 当前 policy 变了没有；
- 证据是否还可访问；
- replay pack 是否覆盖了这次规则依赖的风险面；
- 新版本能不能在同一条件下复现当时的判断。

## 3. learn-claude-code：纯函数版血缘记录器

教学版可以先不接数据库，只做文件式 Manifest。重点是两个动作：生成不可变版本；对依赖做 hash，留下 lock。

~~~python
from dataclasses import dataclass, asdict
from hashlib import sha256
import json
import time
from pathlib import Path

def stable_hash(value: object) -> str:
    encoded = json.dumps(value, sort_keys=True, ensure_ascii=False).encode("utf-8")
    return "sha256:" + sha256(encoded).hexdigest()

@dataclass(frozen=True)
class DependencyLock:
    policy_version: str
    tool_schemas: dict[str, str]
    prompt_pack: str
    evidence_refs: list[str]
    replay_pack: str

@dataclass(frozen=True)
class LearningVersionManifest:
    learning_id: str
    version: int
    parent_version: int | None
    stage: str
    created_by: str
    change_reason: str
    rule_hash: str
    scope: dict
    dependency_lock: DependencyLock
    verification: dict
    created_at: float

def build_learning_version(
    learning_id: str,
    version: int,
    parent_version: int | None,
    rule: dict,
    scope: dict,
    dependency_lock: DependencyLock,
    verification: dict,
    created_by: str,
    change_reason: str,
) -> LearningVersionManifest:
    return LearningVersionManifest(
        learning_id=learning_id,
        version=version,
        parent_version=parent_version,
        stage="shadow",
        created_by=created_by,
        change_reason=change_reason,
        rule_hash=stable_hash({"rule": rule, "scope": scope}),
        scope=scope,
        dependency_lock=dependency_lock,
        verification=verification,
        created_at=time.time(),
    )

def write_manifest(root: Path, manifest: LearningVersionManifest) -> Path:
    path = root / ".learning" / manifest.learning_id / f"v{manifest.version}.json"
    path.parent.mkdir(parents=True, exist_ok=True)

    if path.exists():
        raise ValueError(f"refuse to overwrite immutable learning version: {path}")

    path.write_text(
        json.dumps(asdict(manifest), indent=2, ensure_ascii=False) + "\n",
        encoding="utf-8",
    )
    return path
~~~

这个实现有个关键细节：**版本文件一旦存在就拒绝覆盖**。学习规则版本必须像 git commit 一样不可变。要改，就创建 v5，而不是悄悄改 v4。

learn-claude-code 里的 `s12_worktree_task_isolation.py` 已经用了类似思路：Task 是控制面，worktree 是执行面，`.worktrees/events.jsonl` 是生命周期事件。学习规则也一样：Rule 是控制面，Version Manifest 是执行面，lineage events 是证据面。

## 4. 依赖漂移检测：当前环境是否还能证明旧结论

有了 lock，下一步就是比较当前环境。

~~~python
@dataclass
class CurrentLearningRuntime:
    policy_version: str
    tool_schemas: dict[str, str]
    prompt_pack: str
    available_evidence_refs: set[str]
    replay_pack: str

def detect_dependency_drift(
    locked: DependencyLock,
    current: CurrentLearningRuntime,
) -> list[dict]:
    drifts = []

    if locked.policy_version != current.policy_version:
        drifts.append({
            "kind": "policy_changed",
            "locked": locked.policy_version,
            "current": current.policy_version,
            "action": "rerun_counterfactual_replay",
        })

    for tool_name, locked_hash in locked.tool_schemas.items():
        current_hash = current.tool_schemas.get(tool_name)
        if current_hash != locked_hash:
            drifts.append({
                "kind": "tool_schema_changed",
                "tool": tool_name,
                "locked": locked_hash,
                "current": current_hash,
                "action": "rerun_tool_contract_cases",
            })

    missing_refs = [
        ref for ref in locked.evidence_refs
        if ref not in current.available_evidence_refs
    ]
    if missing_refs:
        drifts.append({
            "kind": "evidence_missing",
            "refs": missing_refs,
            "action": "demote_to_shadow_or_refresh_evidence",
        })

    if locked.replay_pack != current.replay_pack:
        drifts.append({
            "kind": "replay_pack_changed",
            "locked": locked.replay_pack,
            "current": current.replay_pack,
            "action": "rerun_selected_regression_pack",
        })

    return drifts
~~~

这个函数不决定最终命运，只输出事实。上层 promotion gate 再决定 keep、shadow、hold、rollback 或 manual_review。

## 5. pi-mono：用 Agent.subscribe 做外层治理

pi-mono 的 `Agent` 暴露了 `subscribe(fn)`，这类事件订阅很适合做 learning governance：主 Agent Loop 继续负责执行，学习版本血缘在外层监听 `turn_end`、`tool_execution_end`、error 和 final response，生成 Manifest 和审计记录。

概念代码：

~~~ts
type LearningDependencyLock = {
  policyVersion: string;
  toolSchemas: Record<string, string>;
  promptPack: string;
  evidenceRefs: string[];
  replayPack: string;
};

type LearningVersionManifest = {
  learningId: string;
  version: number;
  parentVersion?: number;
  stage: "shadow" | "canary" | "active" | "rolled_back";
  createdBy: string;
  changeReason: string;
  ruleHash: string;
  scope: {
    capabilities: string[];
    allowWhen: string[];
    denyWhen: string[];
  };
  dependencyLock: LearningDependencyLock;
  verification: {
    counterfactualReplay: "passed" | "failed" | "unknown";
    matchedCases: number;
    preventedBadAllows: number;
    falsePositives: number;
    unknowns: number;
  };
};

type LearningLineageStore = {
  appendVersion(manifest: LearningVersionManifest): Promise<void>;
  appendEvent(event: {
    type: string;
    learningId: string;
    version: number;
    runId: string;
    evidenceRefs: string[];
    createdAt: string;
  }): Promise<void>;
};

function installLearningLineageSubscriber(agent: Agent, store: LearningLineageStore) {
  return agent.subscribe(async (event) => {
    if (event.type !== "turn_end") return;

    const candidate = extractLearningCandidate(event);
    if (!candidate) return;

    const manifest = await buildManifestFromCandidate(candidate, {
      policyVersion: currentPolicyVersion(),
      toolSchemas: currentToolSchemaHashes(),
      promptPack: currentPromptPackId(),
      evidenceRefs: candidate.evidenceRefs,
      replayPack: selectedReplayPackId(candidate.scope),
    });

    await store.appendVersion(manifest);
    await store.appendEvent({
      type: "learning.version.created",
      learningId: manifest.learningId,
      version: manifest.version,
      runId: candidate.runId,
      evidenceRefs: manifest.dependencyLock.evidenceRefs,
      createdAt: new Date().toISOString(),
    });
  });
}
~~~

这里不要把 lineage 逻辑塞进 prompt。越是生产系统，越应该把“该记录的证据”放到外层确定性代码里，而不是要求 LLM 自觉记住。

## 6. 和上一课的回滚取证怎么串起来

上一课的 Rollback Case 可以直接成为新版本的 `createdBy`：

~~~json
{
  "learningId": "learn.git.push.pr-state-check",
  "version": 5,
  "parentVersion": 4,
  "createdBy": "rollback-case:lr-2026-05-20-357",
  "changeReason": "fix rule_too_broad after blocked_good_runs exceeded",
  "stage": "shadow"
}
~~~

这带来三个好处：

- rollback case 能追到新版本；
- 新版本能追到被修复的旧版本；
- 如果 v5 继续误伤，可以证明它是否真的解决了 v4 的 root cause。

这就是“学习系统的 git history”。

## 7. OpenClaw 实战：课程 cron 的证据链

OpenClaw 课程 cron 很适合做这个练习，因为它每 3 小时都有清晰的外部副作用：

- Telegram messageId；
- lesson markdown 文件；
- README 目录更新；
- TOOLS.md 已讲内容更新；
- git commit；
- remote main 验证；
- daily memory 记录。

这套流程天然就是 evidence chain。对课程发布 Agent，可以给每条长期规则保留 Manifest：

~~~json
{
  "learningId": "learn.course.avoid-duplicate-topic",
  "version": 2,
  "parentVersion": 1,
  "stage": "active",
  "createdBy": "memory-maintenance:2026-05-20",
  "changeReason": "TOOLS.md topic list became canonical duplicate source",
  "dependencyLock": {
    "policyVersion": "agent-course-cron@2026-05-20",
    "toolSchemas": {
      "rg": "local-cli:ripgrep",
      "git": "local-cli:git",
      "openclaw_message": "openclaw-message-schema@2026-05"
    },
    "promptPack": "cron:agent-course-every-3h",
    "evidenceRefs": [
      "file:///Users/bot001/.openclaw/workspace/TOOLS.md#已讲内容",
      "file:///Users/bot001/.openclaw/workspace/agent-course/README.md",
      "telegram://-5115329245"
    ],
    "replayPack": "course-topic-dedup@v2"
  }
}
~~~

如果未来 TOOLS.md 格式改了，或者课程目录换仓库了，这条规则就不是“继续盲信”，而是触发 dependency drift，要求重新验证去重逻辑。

## 8. 实战检查清单

每条 active learning 至少要能回答：

- 它的 stable learningId 是什么？
- 当前 active 版本是多少？
- parentVersion 是谁？
- createdBy 指向哪个 incident / rollback / repair / memory maintenance？
- ruleHash 是否覆盖 rule + scope？
- dependencyLock 是否包含 policy、tool schema、prompt pack、evidence refs、replay pack？
- promotion 证据是否可追溯？
- 依赖漂移时，是降级 shadow、重新 replay，还是 manual_review？

少任何一项，这条 learning 都还只是“记忆文本”，不是生产规则。

## 9. 小结

今天这课的重点：

- Learning Rule 是长期 ID，Learning Version 是不可变发布物；
- 每个版本都要有 parentVersion、createdBy、changeReason 和 ruleHash；
- dependencyLock 记录 policy、tool schema、prompt、证据和 replay pack；
- 依赖漂移检测不直接拍脑袋，而是输出可审计的 drift facts；
- pi-mono 的 `Agent.subscribe` 很适合在 Agent Loop 外层做血缘记录；
- OpenClaw cron 的 Telegram messageId + git commit + memory 文件，就是天然的学习证据链。

成熟 Agent 的学习系统，不能只是“我记住了”。它要能说清楚：**我从哪学来、哪版生效、依赖什么、怎么验证、出事回到哪。**
