# 227. Agent 运行时指纹与可复现执行（Runtime Fingerprint & Reproducible Runs）

> 关键词：runtime fingerprint、run manifest、reproducibility、environment drift、debug provenance

上一课讲了 **数据新鲜度与过期管理**：上下文里的观察必须带时间戳，旧数据不能假装是新事实。

今天继续往生产排障方向走一步：**Agent 每次执行都应该记录运行时指纹**。

很多 Agent bug 最痛苦的地方不是“出错”，而是：

```txt
昨天晚上 02:13 出过一次错
现在本地复现不了
模型换了、工具版本换了、配置也热更新过
日志里只剩一句：tool failed
```

一句话：**Agent 的每次 run 都要保存一份 Run Manifest，让未来的你知道当时到底是在什么环境下做的决策。**

---

## 1. 为什么普通日志不够？

传统服务里，日志通常记录 requestId、错误堆栈、耗时。

但 Agent 系统多了很多“会改变行为”的维度：

1. 使用的模型和采样参数；
2. system prompt / skill / policy 版本；
3. 工具 schema 版本与启用工具集合；
4. 用户权限、渠道能力、审批策略；
5. 当前 git commit / package version；
6. 时区、当前日期、语言偏好；
7. memory / context snapshot 版本；
8. 子 Agent / ACP harness / browser 节点环境；
9. 外部配置热更新后的 effective config；
10. feature flags 和实验分组。

如果这些没有被记录，同一个用户输入第二天再跑，可能已经不是同一个实验了。

所以生产 Agent 需要把“这次运行的世界状态”固化成一份 **Run Manifest**。

---

## 2. 核心模型：RunManifest

一个最小可用的运行时指纹长这样：

```ts
type RunManifest = {
  runId: string;
  createdAt: string;
  agent: {
    name: string;
    version?: string;
    gitSha?: string;
  };
  model: {
    provider: string;
    name: string;
    temperature?: number;
    reasoning?: string;
  };
  prompt: {
    systemHash: string;
    skills: Array<{ name: string; hash: string }>;
  };
  tools: Array<{
    name: string;
    schemaHash: string;
    version?: string;
    permissions: string[];
  }>;
  runtime: {
    channel: string;
    timezone: string;
    cwd?: string;
    node?: string;
  };
  configHash: string;
  memorySnapshotId?: string;
};
```

注意两点：

- 不要把大段 prompt / memory 原文全塞进去，默认存 **hash + snapshot id**；
- 不要记录密钥原文，secret 永远只能记录 `present: true`、`scope` 或 hash。

Manifest 的目标不是替代日志，而是回答一个问题：**这次 Agent 到底是谁、带着什么工具、站在什么环境里做的决定？**

---

## 3. learn-claude-code：Python 教学版

教学版可以先做一个轻量 `RuntimeFingerprint`，在每次 Agent Loop 开始时生成。

```python
# learn-claude-code/runtime_fingerprint.py
from dataclasses import dataclass, asdict
from datetime import datetime, timezone
import hashlib
import json
import os
import subprocess
from typing import Any


def sha256_text(text: str) -> str:
    return hashlib.sha256(text.encode("utf-8")).hexdigest()[:16]


def git_sha(cwd: str) -> str | None:
    try:
        return subprocess.check_output(
            ["git", "rev-parse", "HEAD"],
            cwd=cwd,
            text=True,
            stderr=subprocess.DEVNULL,
        ).strip()
    except Exception:
        return None


@dataclass
class RuntimeFingerprint:
    run_id: str
    created_at: str
    model: str
    system_hash: str
    tool_schema_hash: str
    git_sha: str | None
    timezone: str
    cwd: str

    @classmethod
    def capture(cls, *, run_id: str, model: str, system_prompt: str, tools: list[dict[str, Any]], cwd: str):
        tool_schema_json = json.dumps(tools, sort_keys=True, ensure_ascii=False)
        return cls(
            run_id=run_id,
            created_at=datetime.now(timezone.utc).isoformat(),
            model=model,
            system_hash=sha256_text(system_prompt),
            tool_schema_hash=sha256_text(tool_schema_json),
            git_sha=git_sha(cwd),
            timezone=os.environ.get("TZ", "UTC"),
            cwd=cwd,
        )

    def save(self, path: str):
        os.makedirs(os.path.dirname(path), exist_ok=True)
        with open(path, "w", encoding="utf-8") as f:
            json.dump(asdict(self), f, ensure_ascii=False, indent=2)
```

Agent Loop 入口：

```python
# learn-claude-code/agent_loop.py
import uuid
from runtime_fingerprint import RuntimeFingerprint


async def run_agent(user_message: str, model: str, system_prompt: str, tools: list[dict], cwd: str):
    run_id = str(uuid.uuid4())

    fingerprint = RuntimeFingerprint.capture(
        run_id=run_id,
        model=model,
        system_prompt=system_prompt,
        tools=tools,
        cwd=cwd,
    )
    fingerprint.save(f"runs/{run_id}/manifest.json")

    # 后续 messages / tool calls / final answer 都带同一个 run_id
    result = await call_llm(
        model=model,
        system=system_prompt,
        messages=[{"role": "user", "content": user_message}],
        tools=tools,
        metadata={"run_id": run_id},
    )

    return {"run_id": run_id, "result": result}
```

这样以后排障至少能回答：

- 当时用的是哪个模型？
- system prompt 有没有变？
- 工具 schema 有没有变？
- 代码是不是同一个 commit？

---

## 4. pi-mono：生产版 RunManifest 中间件

生产版不要让业务代码手动保存 manifest，而应该把它做成 run lifecycle hook。

```ts
// pi-mono/runtime/run-manifest.ts
import { createHash } from "crypto";

function hashJson(value: unknown): string {
  return createHash("sha256")
    .update(JSON.stringify(value, Object.keys(value as any).sort()))
    .digest("hex")
    .slice(0, 16);
}

export type RunManifest = {
  runId: string;
  createdAt: string;
  agentVersion: string;
  gitSha?: string;
  model: {
    provider: string;
    name: string;
    temperature?: number;
    reasoning?: string;
  };
  promptHash: string;
  toolSchemaHash: string;
  enabledTools: string[];
  configHash: string;
  channel: string;
  timezone: string;
  memorySnapshotId?: string;
};

export interface RunManifestStore {
  put(manifest: RunManifest): Promise<void>;
  get(runId: string): Promise<RunManifest | null>;
}

export async function captureRunManifest(ctx: {
  runId: string;
  agentVersion: string;
  gitSha?: string;
  model: RunManifest["model"];
  systemPrompt: string;
  tools: Array<{ name: string; schema: unknown }>;
  effectiveConfig: unknown;
  channel: string;
  timezone: string;
  memorySnapshotId?: string;
}): Promise<RunManifest> {
  return {
    runId: ctx.runId,
    createdAt: new Date().toISOString(),
    agentVersion: ctx.agentVersion,
    gitSha: ctx.gitSha,
    model: ctx.model,
    promptHash: hashJson({ system: ctx.systemPrompt }),
    toolSchemaHash: hashJson(ctx.tools.map(t => ({ name: t.name, schema: t.schema }))),
    enabledTools: ctx.tools.map(t => t.name).sort(),
    configHash: hashJson(ctx.effectiveConfig),
    channel: ctx.channel,
    timezone: ctx.timezone,
    memorySnapshotId: ctx.memorySnapshotId,
  };
}
```

接到 Agent Runner：

```ts
// pi-mono/agent/runner.ts
export class AgentRunner {
  constructor(private manifestStore: RunManifestStore) {}

  async run(input: UserInput, ctx: RunContext) {
    const manifest = await captureRunManifest({
      runId: ctx.runId,
      agentVersion: ctx.agentVersion,
      gitSha: ctx.gitSha,
      model: ctx.model,
      systemPrompt: ctx.systemPrompt,
      tools: ctx.enabledTools,
      effectiveConfig: ctx.effectiveConfig,
      channel: ctx.channel,
      timezone: ctx.timezone,
      memorySnapshotId: ctx.memorySnapshotId,
    });

    await this.manifestStore.put(manifest);

    ctx.logger.info("run_manifest_captured", {
      runId: manifest.runId,
      promptHash: manifest.promptHash,
      toolSchemaHash: manifest.toolSchemaHash,
      configHash: manifest.configHash,
      enabledTools: manifest.enabledTools,
    });

    return this.executeLoop(input, ctx);
  }
}
```

这里有个工程细节：manifest 必须在 **第一次 LLM 调用前** 写入。

原因很简单：如果第一轮 LLM 调用就崩了，你也要知道它是在什么配置下崩的。

---

## 5. Reproduce CLI：从 runId 复现

有了 manifest，下一步就是提供一个复现入口。

```ts
// pi-mono/cli/reproduce-run.ts
export async function reproduceRun(runId: string, options: { dryRun?: boolean }) {
  const manifest = await manifestStore.get(runId);
  if (!manifest) throw new Error(`Run manifest not found: ${runId}`);

  const current = await captureCurrentManifestSkeleton();

  const diffs = compareManifest(manifest, current);
  if (diffs.length > 0) {
    console.warn("Runtime differs from original run:");
    for (const diff of diffs) {
      console.warn(`- ${diff.path}: was ${diff.before}, now ${diff.after}`);
    }
  }

  if (options.dryRun) {
    return { status: "planned", original: manifest, diffs };
  }

  // 真正复现时，可以选择：
  // 1. checkout 原 gitSha
  // 2. 使用原模型配置
  // 3. 加载原 memory snapshot
  // 4. 回放原 tool cassette
  return replayRecordedRun(runId, manifest);
}
```

复现不一定要 100% 自动，哪怕第一版只输出 diff，也已经很有价值：

```txt
Runtime differs from original run:
- model.name: was claude-3-5-sonnet, now gpt-5.5
- toolSchemaHash: was a13f..., now 9c22...
- configHash: was 1a9b..., now 781d...
```

这比“我这边复现不了”强太多。

---

## 6. OpenClaw 实战：把每次关键任务变成可追踪 run

OpenClaw 里天然已经有很多运行时信息：

- `/status` 或 `session_status`：当前模型、reasoning、host、workspace；
- `git rev-parse HEAD`：代码版本；
- workspace 文件：`SOUL.md`、`USER.md`、`TOOLS.md`、Skills；
- session / sub-agent：每个任务都有会话上下文；
- cron / heartbeat：周期任务有明确 job id；
- `memory/YYYY-MM-DD.md`：人工可读运行记录。

最小落地方式：给重要 cron 任务写一个 run manifest。

```json
{
  "runId": "agent-course-2026-05-03T02:30:00Z",
  "job": "Agent 开发课程",
  "createdAt": "2026-05-03T02:30:00Z",
  "model": "openai-codex/gpt-5.5",
  "workspace": "/Users/bot001/.openclaw/workspace",
  "repo": {
    "path": "agent-course",
    "branch": "main",
    "gitShaBefore": "..."
  },
  "inputs": {
    "telegramGroup": "-5115329245",
    "lessonNumber": 227
  },
  "outputs": {
    "lessonFile": "lessons/227-runtime-fingerprint-reproducible-runs.md",
    "commit": "...",
    "messageId": "..."
  }
}
```

这个文件可以放在：

```txt
memory/runs/<runId>.json
```

好处是：以后老板问“第 227 课那次 cron 到底有没有推送成功？用哪个 commit？”不用翻聊天记录，直接查 run manifest。

---

## 7. Manifest 设计 Checklist

生产落地时记住 8 条：

1. **先记录，后执行**：第一轮 LLM / 第一笔外部副作用前写 manifest；
2. **记录 effective config**：不要只记录配置文件路径，要记录合并后的配置 hash；
3. **prompt 存 hash，快照另存**：避免日志爆炸和敏感信息泄露；
4. **tool schema 必须 hash**：工具描述变化会直接影响 LLM 行为；
5. **模型参数必须记录**：model、temperature、reasoning、top_p 都会影响输出；
6. **外部副作用带 runId**：commit、PR、消息、支付操作都能反查来源；
7. **支持 diff**：复现时先告诉你当前环境和原环境哪里不同；
8. **secret 不落盘**：只记录密钥是否存在、scope、版本，不记录值。

---

## 8. 和之前几课的关系

- **工具调用回放**：记录调用输入输出；
- **时间旅行调试**：记录每步状态；
- **审计日志**：记录谁做了什么；
- **运行时指纹**：记录当时世界长什么样。

它们组合起来才是完整生产排障链路：

```txt
RunManifest → 知道环境
Trace/Logs → 知道过程
Tool Replay → 复现外部交互
State Snapshot → 回到中间状态
Audit Log → 证明权限与责任
```

---

## 9. 小结

Agent 的行为不是只由用户输入决定的，而是由：

```txt
用户输入 + 模型 + prompt + 工具 schema + 配置 + 权限 + memory + 当前环境
```

共同决定。

所以每次重要执行都应该留下运行时指纹。

**没有 Run Manifest 的 Agent bug，只能靠玄学复盘；有 Run Manifest 的 Agent bug，才有机会工程化复现。**
