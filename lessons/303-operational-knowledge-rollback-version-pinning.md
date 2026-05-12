# 303. Agent 运维知识回滚与版本固定（Operational Knowledge Rollback & Version Pinning）

> 上一课讲“知识变更影响分析”：新知识进来前要跑回归门控。今天补上另一个生产必需能力：**知识更新后如果导致 Agent 行为异常，必须能快速回滚，并且关键 Runbook 可以固定在已验证的知识版本上执行**。

很多 Agent 事故不是代码部署导致的，而是“记忆 / Runbook / 策略知识”悄悄变了：

- 新复盘写入了一条过度宽泛的规则
- Runbook 依赖的知识条目被改成了新版本
- 某个缓存或长期记忆被错误摘要污染
- 多实例 Agent 中，一部分读到新知识，一部分还在旧知识

所以成熟 Agent 的知识库不能只是 `latest`。它需要：

1. **Versioned Knowledge Store**：每条知识都有不可变版本。
2. **Pinning Manifest**：高风险操作固定使用某个已验证版本集合。
3. **Rollback Plan**：新版本异常时能原子回退到旧版本。
4. **Behavior Replay Gate**：回滚前后重放关键案例，证明行为恢复。

---

## 1. 核心模型：KnowledgeRevision + Pin Manifest

不要覆盖写知识条目，而是追加新 revision：

```json
{
  "id": "kb.deploy.rollback-policy",
  "revision": 7,
  "status": "active",
  "supersedes": 6,
  "createdAt": "2026-05-12T14:30:00Z",
  "contentHash": "sha256:abc123...",
  "verifiedBy": ["regression.deploy.safe", "orr.production"],
  "risk": "high"
}
```

关键 Runbook 不直接读 `active latest`，而是读一份 pin manifest：

```json
{
  "runbook": "production.deploy",
  "pinSet": {
    "kb.deploy.rollback-policy": 6,
    "kb.change-window.freeze": 12,
    "kb.release.evidence": 4
  },
  "approvedAt": "2026-05-12T10:00:00Z",
  "approvedBy": "orr-gate",
  "expiresAt": "2026-05-19T10:00:00Z"
}
```

这样 Agent 做生产操作时，不会因为后台知识刚更新就突然改变行为。

---

## 2. learn-claude-code：最小 Python 教学版

文件式实现足够说明机制：

```python
from dataclasses import dataclass
from typing import Dict, List
import hashlib, json, time

@dataclass(frozen=True)
class KnowledgeRevision:
    id: str
    revision: int
    content: str
    status: str          # draft | active | rolled_back | archived
    supersedes: int | None
    content_hash: str

class KnowledgeStore:
    def __init__(self):
        self.revisions: Dict[str, List[KnowledgeRevision]] = {}

    def append(self, id: str, content: str, supersedes: int | None = None):
        rev = (max([r.revision for r in self.revisions.get(id, [])], default=0) + 1)
        h = "sha256:" + hashlib.sha256(content.encode()).hexdigest()
        item = KnowledgeRevision(id, rev, content, "active", supersedes, h)
        self.revisions.setdefault(id, []).append(item)
        return item

    def get_pinned(self, pin_set: Dict[str, int]) -> List[KnowledgeRevision]:
        result = []
        for id, rev in pin_set.items():
            matches = [r for r in self.revisions.get(id, []) if r.revision == rev]
            if not matches:
                raise ValueError(f"missing pinned knowledge: {id}@{rev}")
            result.append(matches[0])
        return result

    def rollback(self, id: str, target_rev: int):
        current = max(self.revisions[id], key=lambda r: r.revision)
        target = [r for r in self.revisions[id] if r.revision == target_rev][0]
        # 不是删除新版本，而是追加一条“回滚 revision”，审计链不断
        return self.append(
            id=id,
            content=target.content,
            supersedes=current.revision,
        )
```

关键点：**rollback 不是把历史改掉，而是追加一个新 revision，内容回到旧版本**。这样审计日志能回答：什么时候、为什么、从哪个错误版本回滚。

---

## 3. pi-mono：生产中间件设计

在 TypeScript 生产系统里，建议把知识读取放进中间件，而不是散落在业务代码里：

```ts
type KnowledgeRef = `${string}@${number}`;

type PinManifest = {
  runbook: string;
  pinSet: Record<string, number>;
  approvedAt: string;
  expiresAt: string;
};

type KnowledgeRevision = {
  id: string;
  revision: number;
  content: string;
  contentHash: string;
  status: "draft" | "active" | "rolled_back" | "archived";
  verifiedBy: string[];
};

class KnowledgeVersionMiddleware {
  constructor(
    private store: KnowledgeStore,
    private manifests: PinManifestStore,
    private replayGate: BehaviorReplayGate,
  ) {}

  async buildContext(runbook: string, purpose: "answer" | "side_effect") {
    const manifest = await this.manifests.get(runbook);

    if (purpose === "side_effect" && this.isExpired(manifest)) {
      throw new Error(`Pin manifest expired for ${runbook}; require re-approval`);
    }

    const revisions = await Promise.all(
      Object.entries(manifest.pinSet).map(([id, rev]) =>
        this.store.getRevision(id, rev),
      ),
    );

    for (const item of revisions) {
      if (!item.verifiedBy.length && purpose === "side_effect") {
        throw new Error(`Unverified knowledge blocked: ${item.id}@${item.revision}`);
      }
    }

    return revisions.map(r => ({
      ref: `${r.id}@${r.revision}` as KnowledgeRef,
      hash: r.contentHash,
      content: r.content,
    }));
  }

  private isExpired(manifest: PinManifest) {
    return Date.now() > Date.parse(manifest.expiresAt);
  }
}
```

这个中间件提供三个保护：

- 副作用操作必须使用未过期 pin manifest
- 未验证知识不能进入高风险动作上下文
- 上下文里带 `id@revision + hash`，后续审计能精确复现

---

## 4. 回滚流程：不是“切回去”这么简单

推荐流程：

```text
1. Detect
   发现新知识版本导致异常行为 / 回归失败 / 人工报告

2. Freeze
   暂停该知识影响范围内的自动副作用动作

3. Select Target
   选择最近一个通过回归门控的 revision

4. Replay
   用目标 revision 重放关键 Regression Cases

5. Append Rollback Revision
   追加 rollback revision，保留审计链

6. Re-pin
   更新受影响 Runbook 的 pin manifest

7. Verify
   再跑一次行为回放 + 小流量 canary

8. Unfreeze
   解除冻结，归档 evidence bundle
```

注意：**冻结要早于回滚**。因为知识异常期间继续自动执行，可能扩大事故半径。

---

## 5. OpenClaw 落地方式

OpenClaw 这种 Always-on Agent 里，知识来源很多：`MEMORY.md`、`TOOLS.md`、skills、daily memory、Runbook 文件、课程 cron 自己写的 lesson。建议给高风险工作流加一个轻量 manifest：

```json
{
  "workflow": "agent-course-publish",
  "knowledgePins": {
    "TOOLS.md#agent-course-known-topics": "sha256:...",
    "AGENTS.md#external-actions": "sha256:...",
    "github-skill": "sha256:..."
  },
  "requiredGates": [
    "lesson_file_exists",
    "readme_updated",
    "tools_known_topics_updated",
    "telegram_message_sent",
    "git_pushed"
  ]
}
```

执行前读取这些 hash，执行后再复核：

- 如果 `TOOLS.md` 课程列表在运行中变了，重新检查是否重复选题
- 如果 GitHub skill 或外部动作规则变了，重新做 push 前验证
- 如果群消息发送失败，不要假装完成，要记录 outbox / retry

这其实就是前面几课的组合：

- 数据新鲜度：执行前重新读关键文件
- 观察快照：冻结“我刚才看到的世界”
- 变更影响分析：知识更新要跑回归
- 本课：异常时能回滚并固定版本

---

## 6. 常见坑

### 坑 1：只保存 latest

只保存 latest 的知识库没法回答“上一次正常的知识是什么”。生产系统必须保留 revision。

### 坑 2：回滚用 delete

删除错误版本会破坏审计链。正确做法是追加 rollback revision。

### 坑 3：低风险和高风险同策略

普通问答可以读 latest active；生产副作用必须读 pinned verified revision。

### 坑 4：回滚后不跑 replay

回滚不是信仰动作。必须用 Regression Cases 证明目标版本确实恢复了预期行为。

---

## 7. 实战建议

如果你的 Agent 已经有长期记忆或运维知识库，下一步可以这样做：

1. 给每条知识加 `id / revision / contentHash / status`
2. 高风险 Runbook 引入 `pin manifest`
3. 所有知识变更追加写，不覆盖写
4. 每次回滚生成 evidence bundle
5. 每周清理过期 pin，但清理前必须重新验证并续签

一句话总结：

> 成熟 Agent 不只会学习新知识，还要能在新知识有毒时快速退回到“已证明安全”的旧世界。
