# 第 246 课：Agent 环境漂移检测与配置指纹（Environment Drift Detection & Config Fingerprinting）

> 关键词：环境漂移、配置指纹、部署前复核、密钥版本、运行时一致性

很多 Agent 事故不是“代码写错”，而是：

- 本地是 Node 22，生产是 Node 20；
- staging 开了 feature flag，production 没开；
- 工具列表热更新了，但审批时看到的是旧工具；
- `.env` 里密钥换了，Agent 还拿旧 token 执行；
- Docker image、依赖 lockfile、数据库 schema 版本不一致。

所以成熟 Agent 在执行副作用前，不能只问“计划对不对”，还要问：**我现在所在的环境，还是刚才验证过的那个环境吗？**

这就是本课：**环境漂移检测与配置指纹**。

---

## 1. 核心概念：把运行环境变成一枚可比较的指纹

环境指纹不是把所有配置原文塞进日志，而是提取可验证摘要：

```ts
type EnvFingerprint = {
  app: string;
  env: "dev" | "staging" | "prod";
  gitSha: string;
  lockfileHash: string;
  runtime: {
    node?: string;
    python?: string;
  };
  toolSchemaHash: string;
  configHash: string;       // 脱敏后的配置摘要
  secretVersionHash: string; // 只记录版本/引用，不记录 secret 原文
  createdAt: string;
};
```

关键点：

1. **可比较**：两次 fingerprint 不一样，就说明环境可能漂移。
2. **可审计**：日志里能知道“当时是哪个 git sha、哪个工具 schema、哪个配置版本”。
3. **不泄密**：hash 配置结构和 secret version，不能 hash secret 原文后到处传播，更不能明文记录。

---

## 2. 漂移检测放在哪些位置？

建议放 4 个闸门：

| 时机 | 目的 |
|---|---|
| Agent 启动时 | 建立 baseline fingerprint |
| 生成计划后 | 把 fingerprint 绑定到计划/审批 |
| 执行副作用前 | 重新采样，确认环境没漂移 |
| 最终回复前 | 把执行环境写入 evidence，方便追溯 |

执行危险工具前的判断：

```text
if currentFingerprint != approvedFingerprint:
    stop
    explain drift
    require revalidation or re-approval
```

这和上一课“计划签名”配套：计划没变但环境变了，也不能继续。

---

## 3. learn-claude-code：Python 教学版

下面是一个简化版环境指纹器：

```python
# learn_claude_code/env_fingerprint.py
from __future__ import annotations

import hashlib
import json
import os
import platform
import subprocess
from dataclasses import dataclass, asdict
from pathlib import Path
from typing import Any

SENSITIVE_KEYS = {"api_key", "token", "secret", "password"}


def sha256_text(text: str) -> str:
    return hashlib.sha256(text.encode("utf-8")).hexdigest()


def file_hash(path: str) -> str:
    p = Path(path)
    if not p.exists():
        return "missing"
    return sha256_text(p.read_text(errors="ignore"))


def git_sha() -> str:
    try:
        return subprocess.check_output(
            ["git", "rev-parse", "HEAD"],
            text=True,
            stderr=subprocess.DEVNULL,
        ).strip()
    except Exception:
        return "unknown"


def sanitize_config(config: dict[str, Any]) -> dict[str, Any]:
    clean: dict[str, Any] = {}
    for key, value in sorted(config.items()):
        lowered = key.lower()
        if any(s in lowered for s in SENSITIVE_KEYS):
            # 只记录“存在”和长度，不记录值本身
            clean[key] = {"present": bool(value), "length": len(str(value or ""))}
        else:
            clean[key] = value
    return clean


@dataclass(frozen=True)
class EnvFingerprint:
    app: str
    env: str
    git_sha: str
    lockfile_hash: str
    python_version: str
    tool_schema_hash: str
    config_hash: str
    secret_version_hash: str


def collect_fingerprint(tool_schemas: list[dict[str, Any]]) -> EnvFingerprint:
    app_config = {
        "MODEL_PROVIDER": os.getenv("MODEL_PROVIDER"),
        "DATABASE_URL": os.getenv("DATABASE_URL"),
        "OPENAI_API_KEY": os.getenv("OPENAI_API_KEY"),
        "FEATURE_FLAGS": os.getenv("FEATURE_FLAGS"),
    }

    secret_versions = {
        # 生产里建议来自 Secret Manager 的 version id，而不是 secret value
        "OPENAI_API_KEY_VERSION": os.getenv("OPENAI_API_KEY_VERSION", "local"),
    }

    return EnvFingerprint(
        app="learn-claude-code",
        env=os.getenv("APP_ENV", "dev"),
        git_sha=git_sha(),
        lockfile_hash=file_hash("requirements.txt"),
        python_version=platform.python_version(),
        tool_schema_hash=sha256_text(json.dumps(tool_schemas, sort_keys=True)),
        config_hash=sha256_text(json.dumps(sanitize_config(app_config), sort_keys=True)),
        secret_version_hash=sha256_text(json.dumps(secret_versions, sort_keys=True)),
    )


def assert_no_drift(approved: EnvFingerprint, current: EnvFingerprint) -> None:
    if approved == current:
        return

    old = asdict(approved)
    new = asdict(current)
    changed = {
        key: {"approved": old[key], "current": new[key]}
        for key in old
        if old[key] != new[key]
    }
    raise RuntimeError(f"ENV_DRIFT_DETECTED: {json.dumps(changed, ensure_ascii=False)}")
```

在 Agent Loop 里用法：

```python
# 计划生成时
approved_fingerprint = collect_fingerprint(tool_schemas)
plan = {
    "goal": user_goal,
    "steps": steps,
    "envFingerprint": approved_fingerprint,
}

# 执行外部副作用前
current_fingerprint = collect_fingerprint(tool_schemas)
assert_no_drift(plan["envFingerprint"], current_fingerprint)

# 只有没漂移，才真正执行
await tool_call.execute()
```

---

## 4. pi-mono：TypeScript 生产版中间件

生产系统里建议做成工具中间件，透明拦截高风险工具：

```ts
// pi-mono/packages/agent-runtime/src/env/EnvFingerprint.ts
import crypto from "node:crypto";
import fs from "node:fs";
import { execSync } from "node:child_process";

type ToolSchema = {
  name: string;
  inputSchema: unknown;
  risk?: "low" | "medium" | "high" | "critical";
};

export type EnvFingerprint = {
  app: string;
  env: string;
  gitSha: string;
  lockfileHash: string;
  nodeVersion: string;
  toolSchemaHash: string;
  configHash: string;
  secretVersionHash: string;
};

function sha256(value: unknown): string {
  return crypto
    .createHash("sha256")
    .update(typeof value === "string" ? value : JSON.stringify(value))
    .digest("hex");
}

function readHash(path: string): string {
  return fs.existsSync(path) ? sha256(fs.readFileSync(path, "utf8")) : "missing";
}

function currentGitSha(): string {
  try {
    return execSync("git rev-parse HEAD", { encoding: "utf8" }).trim();
  } catch {
    return "unknown";
  }
}

function sanitizedConfig() {
  return {
    modelProvider: process.env.MODEL_PROVIDER,
    databaseHost: process.env.DATABASE_HOST,
    featureFlags: process.env.FEATURE_FLAGS,
    // secret 只记录是否存在，不记录值
    openaiKeyPresent: Boolean(process.env.OPENAI_API_KEY),
  };
}

export function collectEnvFingerprint(toolSchemas: ToolSchema[]): EnvFingerprint {
  return {
    app: process.env.APP_NAME ?? "pi-mono-agent",
    env: process.env.APP_ENV ?? "dev",
    gitSha: currentGitSha(),
    lockfileHash: readHash("pnpm-lock.yaml"),
    nodeVersion: process.version,
    toolSchemaHash: sha256(
      toolSchemas.map((tool) => ({
        name: tool.name,
        inputSchema: tool.inputSchema,
        risk: tool.risk,
      })),
    ),
    configHash: sha256(sanitizedConfig()),
    secretVersionHash: sha256({
      openai: process.env.OPENAI_API_KEY_VERSION ?? "local",
      anthropic: process.env.ANTHROPIC_API_KEY_VERSION ?? "local",
    }),
  };
}
```

中间件：

```ts
// pi-mono/packages/agent-runtime/src/middleware/EnvDriftMiddleware.ts
import { collectEnvFingerprint, type EnvFingerprint } from "../env/EnvFingerprint";

class EnvDriftError extends Error {
  constructor(public readonly diff: Record<string, unknown>) {
    super("ENV_DRIFT_DETECTED");
  }
}

function diffFingerprint(approved: EnvFingerprint, current: EnvFingerprint) {
  const diff: Record<string, unknown> = {};

  for (const key of Object.keys(approved) as Array<keyof EnvFingerprint>) {
    if (approved[key] !== current[key]) {
      diff[key] = {
        approved: approved[key],
        current: current[key],
      };
    }
  }

  return diff;
}

export function createEnvDriftMiddleware(options: {
  toolSchemas: Array<{ name: string; inputSchema: unknown; risk?: string }>;
}) {
  return async function envDriftMiddleware(ctx: ToolContext, next: () => Promise<ToolResult>) {
    const risk = ctx.tool.risk ?? "low";

    // 低风险只记录，不阻断；高风险必须强校验
    if (risk !== "high" && risk !== "critical") {
      return next();
    }

    const approved = ctx.run.approvedEnvFingerprint as EnvFingerprint | undefined;
    const current = collectEnvFingerprint(options.toolSchemas);

    if (!approved) {
      throw new EnvDriftError({ reason: "missing_approved_fingerprint", current });
    }

    const diff = diffFingerprint(approved, current);
    if (Object.keys(diff).length > 0) {
      ctx.logger.warn({ diff, tool: ctx.tool.name }, "Environment drift detected");
      throw new EnvDriftError(diff);
    }

    return next();
  };
}
```

Agent 运行时：

```ts
const envFingerprint = collectEnvFingerprint(toolSchemas);

const run = await agent.createRun({
  userId,
  goal,
  approvedEnvFingerprint: envFingerprint,
});

// 后续所有 high/critical 工具调用都会自动校验 fingerprint
await agent.execute(run.id);
```

---

## 5. OpenClaw 实战：为什么 git push / 发群消息前要 live check？

OpenClaw 的 Always-on Agent 很容易遇到这种漂移：

- Cron 触发时 repo 是最新的；
- 写课、改 README 用了 10 分钟；
- 执行 `git push` 前，远端 main 已经被别的任务更新；
- 如果不复核，就可能冲突、覆盖、或者把内容推到错误状态。

所以课程 cron 的安全流程应该是：

```bash
# 1. 采样当前状态
pwd
git status --short
git rev-parse HEAD
find lessons -maxdepth 1 -name '*.md' | sort -V | tail

# 2. 修改课程文件与 README

# 3. 副作用前重新复核
gh auth switch --user gfwfail
gh pr list --state all --limit 5
git pull --rebase origin main
git diff --check

# 4. 再提交推送
git add .
git commit -m "Add lesson 246 env drift fingerprinting"
git push
```

发 Telegram 也一样：发送前应该确认：

- 群 ID 是否还是目标群；
- 内容是否是本课；
- 没有泄露 token/secret；
- 没有重复上一课主题。

这就是“副作用前环境复核”的实际价值。

---

## 6. 常见坑

### 坑 1：把 secret 原文 hash 后到处记录

不要这么做。secret hash 也可能成为离线撞库目标。更好的方式：

- 记录 Secret Manager 的 version id；
- 或记录“present / missing / length bucket”；
- 审计系统里只展示版本，不展示值。

### 坑 2：所有漂移都强制失败

不是所有变化都同等危险。

建议策略：

| 漂移字段 | 风险 |
|---|---|
| gitSha | medium/high |
| lockfileHash | high |
| toolSchemaHash | critical |
| configHash | high |
| secretVersionHash | high/critical |
| nodeVersion/pythonVersion | medium |

低风险工具可以只记录 warning，高风险副作用必须阻断。

### 坑 3：fingerprint 太大，塞爆上下文

LLM 不需要看到完整 fingerprint。上下文里只注入摘要：

```json
{
  "env": "prod",
  "gitSha": "abc1234",
  "toolSchemaHash": "sha256:7f2...",
  "driftPolicy": "block high/critical tools"
}
```

完整内容放数据库/日志/审计记录。

---

## 7. 一句话总结

**Agent 环境漂移检测 = 给“我以为的运行环境”做指纹，执行副作用前重新比对。**

代码正确只是第一步；环境一致，才敢执行。

成熟 Agent 不只会说“我已经检查过了”，而是能证明：

> 我刚才检查的世界，和我现在要动手的世界，还是同一个世界。
