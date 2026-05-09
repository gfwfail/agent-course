# 282. Agent 发布证据包与审计归档（Release Evidence Bundle & Audit Archive）

> 核心思想：一次发布不只要“成功”，还要能在事后证明它为什么可以发布、发布了什么、验证了什么、失败时如何恢复。成熟 Agent 每次外部变更都应该产出一份可复盘的 Evidence Bundle。

上一课我们讲了部署后的验证与自动回滚。但还有一个经常被忽略的问题：

```text
发布成功了 → 过了两天线上出问题 → 问：当时为什么放行？验证证据在哪？用了哪个模型/配置/工具版本？
```

如果答案散落在聊天记录、CI 日志、终端输出和脑子里，那这次发布就不可审计。

所以要把发布闭环升级为：

```text
计划 → 风险评分 → 审批/门控 → 部署 → 验证 → 回滚策略 → 证据包归档
```

这就是 **Release Evidence Bundle & Audit Archive**。

---

## 1. Evidence Bundle 里应该有什么？

一份最小可用证据包包含 8 类信息：

```json
{
  "releaseId": "agent-2026-05-10-0930",
  "actor": "cron:agent-course",
  "repo": "gfwfail/agent-course",
  "git": {
    "branch": "main",
    "beforeSha": "18b6ef4",
    "afterSha": "abc1234",
    "diffStat": "+210 -0"
  },
  "intent": {
    "userRequest": "每3小时发布一节 Agent 开发课",
    "normalizedIntent": "publish_lesson_and_push_repo"
  },
  "risk": {
    "score": 22,
    "level": "low",
    "reasons": ["docs_only", "telegram_message", "git_push"]
  },
  "gates": [
    { "name": "duplicate_topic_check", "status": "passed" },
    { "name": "lesson_file_exists", "status": "passed" },
    { "name": "git_diff_check", "status": "passed" }
  ],
  "sideEffects": [
    { "type": "telegram_message", "target": "-5115329245", "messageId": "11475" },
    { "type": "git_push", "remote": "origin", "commit": "abc1234" }
  ],
  "artifacts": [
    { "path": "lessons/282-release-evidence-bundle-audit-archive.md", "sha256": "..." }
  ]
}
```

注意：证据包不是日志大杂烩，而是**发布决策的结构化摘要**。

它要回答四个问题：

1. 为什么要做？—— intent
2. 做了什么？—— git / sideEffects / artifacts
3. 凭什么认为安全？—— risk / gates / verification
4. 出事后怎么追？—— releaseId / actor / timestamps / hashes

---

## 2. learn-claude-code：Python 教学版

先做一个最小 EvidenceBundle 生成器。教学版可以直接写 JSON 文件。

```python
from dataclasses import dataclass, asdict
from datetime import datetime, timezone
from pathlib import Path
import hashlib
import json

@dataclass
class GateEvidence:
    name: str
    status: str              # passed / failed / blocked
    detail: str | None = None

@dataclass
class SideEffectEvidence:
    type: str                # telegram_message / git_push / deploy / email
    target: str
    external_id: str | None

@dataclass
class ReleaseEvidenceBundle:
    release_id: str
    actor: str
    created_at: str
    intent: dict
    git: dict
    risk: dict
    gates: list[GateEvidence]
    side_effects: list[SideEffectEvidence]
    artifacts: list[dict]


def sha256_file(path: Path) -> str:
    h = hashlib.sha256()
    with path.open("rb") as f:
        for chunk in iter(lambda: f.read(1024 * 1024), b""):
            h.update(chunk)
    return h.hexdigest()


def build_bundle(release_id: str, lesson_path: Path, message_id: str, commit: str):
    return ReleaseEvidenceBundle(
        release_id=release_id,
        actor="openclaw-cron:agent-course",
        created_at=datetime.now(timezone.utc).isoformat(),
        intent={
            "normalizedIntent": "publish_agent_course_lesson",
            "topic": "Release Evidence Bundle & Audit Archive",
        },
        git={"repo": "gfwfail/agent-course", "commit": commit},
        risk={"level": "low", "reasons": ["docs_only", "known_channel"]},
        gates=[
            GateEvidence("topic_not_duplicate", "passed"),
            GateEvidence("lesson_file_written", "passed", str(lesson_path)),
            GateEvidence("git_diff_check", "passed"),
        ],
        side_effects=[
            SideEffectEvidence("telegram_message", "-5115329245", message_id),
            SideEffectEvidence("git_push", "origin/main", commit),
        ],
        artifacts=[{"path": str(lesson_path), "sha256": sha256_file(lesson_path)}],
    )


def archive_bundle(bundle: ReleaseEvidenceBundle, archive_dir: Path):
    archive_dir.mkdir(parents=True, exist_ok=True)
    path = archive_dir / f"{bundle.release_id}.json"
    path.write_text(
        json.dumps(asdict(bundle), ensure_ascii=False, indent=2),
        encoding="utf-8",
    )
    return path
```

关键点：

- `release_id` 是所有日志、消息、commit、artifact 的关联键
- `sha256` 防止 artifact 后续被悄悄改掉
- `gates` 记录的是“为什么放行”，不是只记录“我做完了”
- `side_effects.external_id` 保存外部系统回执，比如 Telegram message id / deployment id / PR id

---

## 3. pi-mono：TypeScript 生产版 Evidence Middleware

生产系统里不要让业务代码到处手写证据。更好的方式是做一个中间件：每次 release/run 自动收集证据。

```ts
type GateStatus = "passed" | "failed" | "blocked";

type EvidenceEvent =
  | { type: "intent_recorded"; intent: unknown }
  | { type: "gate_checked"; name: string; status: GateStatus; detail?: unknown }
  | { type: "side_effect_committed"; effectType: string; target: string; externalId?: string }
  | { type: "artifact_written"; path: string; sha256: string }
  | { type: "git_pushed"; remote: string; commit: string };

type EvidenceBundle = {
  releaseId: string;
  actor: string;
  startedAt: string;
  finishedAt?: string;
  events: EvidenceEvent[];
  summary?: {
    outcome: "completed" | "failed" | "rolled_back" | "manual_review";
    commit?: string;
    messageIds?: string[];
  };
};

class EvidenceRecorder {
  private bundle: EvidenceBundle;

  constructor(releaseId: string, actor: string) {
    this.bundle = {
      releaseId,
      actor,
      startedAt: new Date().toISOString(),
      events: [],
    };
  }

  record(event: EvidenceEvent) {
    this.bundle.events.push(event);
  }

  finish(summary: EvidenceBundle["summary"]) {
    this.bundle.finishedAt = new Date().toISOString();
    this.bundle.summary = summary;
    return this.bundle;
  }
}
```

把它挂到工具分发层：

```ts
async function withEvidence<T>(
  recorder: EvidenceRecorder,
  action: () => Promise<T>,
): Promise<T> {
  try {
    const result = await action();
    recorder.record({
      type: "gate_checked",
      name: "action_completed_without_throw",
      status: "passed",
    });
    return result;
  } catch (error) {
    recorder.record({
      type: "gate_checked",
      name: "action_completed_without_throw",
      status: "failed",
      detail: String(error),
    });
    throw error;
  }
}
```

更进一步：把 `message.send`、`git.push`、`deploy.rollback` 这类副作用工具都包一层：

```ts
async function sendTelegramWithEvidence(
  recorder: EvidenceRecorder,
  send: (target: string, text: string) => Promise<{ messageId: string }>,
  target: string,
  text: string,
) {
  const receipt = await send(target, text);
  recorder.record({
    type: "side_effect_committed",
    effectType: "telegram_message",
    target,
    externalId: receipt.messageId,
  });
  return receipt;
}
```

这样 Agent 不需要“记得写日志”，基础设施会强制留下证据。

---

## 4. OpenClaw 落地：课程 Cron 的证据包

以这个课程 cron 为例，完成标准不应该只是“发群 + push”。至少要有这些验收点：

```text
releaseId: lesson-282-2026-05-10-0930

Gates:
- 已检查 TOOLS.md / README，主题未重复
- lessons/282-xxx.md 已写入
- README.md 已追加目录
- TOOLS.md 已追加已讲内容
- Telegram message 返回 messageId
- gh auth 当前用户为 gfwfail
- git diff --check 通过
- git commit 成功
- git push 成功

Side effects:
- Telegram group -5115329245: messageId=...
- GitHub origin/main: commit=...
```

文件式归档可以放在：

```text
agent-course/.evidence/lesson-282-2026-05-10-0930.json
```

如果不想把证据包提交进公开 repo，也可以放到私有工作区：

```text
~/.openclaw/workspace/memory/evidence/agent-course/lesson-282-2026-05-10-0930.json
```

选择规则很简单：

- 公开、无敏感信息、对读者有价值 → 可进 repo
- 含外部 message id、操作者、内部路径、审计细节 → 放私有 archive
- 含 token/API key/用户隐私 → 只存脱敏摘要，敏感原文不进证据包

---

## 5. 常见坑

### 坑 1：只保存最终状态，不保存放行理由

```json
{ "status": "success" }
```

这没用。它不能解释为什么成功、谁判断成功、哪些检查通过。

更好的记录：

```json
{
  "status": "success",
  "passedGates": ["topic_not_duplicate", "diff_check", "push_confirmed"],
  "commit": "abc1234",
  "messageId": "11480"
}
```

### 坑 2：证据包里塞敏感数据

证据包是审计材料，不是秘密垃圾桶。

应该记录：

```json
{ "secretRef": "github:gfwfail", "used": true }
```

不要记录：

```json
{ "githubToken": "ghp_xxx" }
```

### 坑 3：证据包在成功后才创建

如果中途失败，最需要证据。正确做法是：

```text
start release → 创建 bundle 草稿 → 每个 gate/side effect 追加事件 → finish/failed 都归档
```

---

## 6. 实战 Checklist

做任何 Agent 发布/外部副作用前，问这 6 个问题：

- 有没有唯一 `releaseId/runId`？
- 有没有记录用户意图和标准化任务？
- 有没有记录风险评分与放行 gate？
- 有没有记录外部副作用回执？
- 有没有 artifact hash，证明交付物没被改？
- 失败/回滚路径是否也会归档？

---

## 总结

Evidence Bundle 是 Agent 发布系统的“黑匣子”。

- `日志` 记录发生过什么
- `验证` 判断能不能继续
- `证据包` 证明为什么当时这样判断

成熟 Agent 不只会自动执行，还要能把每次执行变成可审计、可复盘、可追责的事实链。

一句话：**没有证据包的自动化，只是跑得很快的失忆症。**
