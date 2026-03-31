# 185 - Agent 密钥管理与环境变量安全（Secrets Management & Environment Security）

> "硬编码 API Key 进代码，等于在公厕墙上写密码。" —— 每一个翻过 git blame 的人

---

## 为什么这是必须掌握的技能

Agent 几乎必然要调用外部 API：LLM 服务、数据库、第三方工具……每个调用都需要凭证。
错误处理密钥会导致：

- 密钥泄露 → 账单爆炸（有人一觉醒来 AWS $50,000 账单）
- 密钥轮换 → Agent 停摆
- 多环境混用 → 测试 Agent 误操作生产数据

本节讲清楚 **密钥生命周期管理** 的每个环节，并给出 TypeScript / Python 可直接用的实现。

---

## 核心概念：密钥的生命周期

```
创建 → 注入 → 使用 → 轮换 → 吊销
  ↑                              ↓
  └──────────── 审计 ────────────┘
```

Agent 的特殊挑战：密钥不只在启动时需要，**工具调用时实时需要**，而且可能动态扩展新工具。

---

## 层级一：绝对禁止清单

```typescript
// ❌ 绝对不能这样写
const client = new Anthropic({ apiKey: "sk-ant-api03-xxxx" });

// ❌ 不能写在代码注释里
// API Key: sk-ant-...

// ❌ 不能写在测试文件里
const TEST_KEY = "sk-live-xxx"; // 单元测试用

// ❌ 不能 console.log 整个 config
console.log("Config:", config); // config 里可能有 key
```

```bash
# 检查有没有泄露（CI 必备）
git log --all -p | grep -E "(sk-ant|AIza|AKIA|ghp_)" | head -5
```

---

## 层级二：环境变量 + .env 分层

### 文件结构
```
.env.example      # ✅ 提交到 git，只含变量名（无值）
.env.local        # ✅ 本地开发，gitignore
.env.test         # ✅ 测试用假密钥，可提交
.env.production   # ❌ 不存在！生产密钥走 Vault/云服务
```

### .env.example（提交到 git）
```bash
# LLM
ANTHROPIC_API_KEY=
OPENAI_API_KEY=

# Database
DATABASE_URL=
REDIS_URL=

# Third-party tools
GITHUB_TOKEN=
TELEGRAM_BOT_TOKEN=
```

### TypeScript - 启动时强校验
```typescript
// src/config/secrets.ts
import { z } from "zod";

const SecretsSchema = z.object({
  ANTHROPIC_API_KEY: z.string().min(1).startsWith("sk-ant-"),
  DATABASE_URL: z.string().url(),
  REDIS_URL: z.string().url().optional(),
  GITHUB_TOKEN: z.string().startsWith("ghp_").optional(),
});

type Secrets = z.infer<typeof SecretsSchema>;

let _secrets: Secrets | null = null;

export function getSecrets(): Secrets {
  if (_secrets) return _secrets;

  const result = SecretsSchema.safeParse(process.env);
  if (!result.success) {
    const missing = result.error.issues
      .map((i) => `  - ${i.path.join(".")}: ${i.message}`)
      .join("\n");
    throw new Error(`❌ 密钥校验失败，请检查 .env：\n${missing}`);
  }

  _secrets = result.data;
  return _secrets;
}

// 使用
const { ANTHROPIC_API_KEY } = getSecrets();
```

### Python 版本
```python
# config/secrets.py
import os
from dataclasses import dataclass
from typing import Optional

@dataclass
class Secrets:
    ANTHROPIC_API_KEY: str
    DATABASE_URL: str
    REDIS_URL: Optional[str] = None
    GITHUB_TOKEN: Optional[str] = None

_secrets: Optional[Secrets] = None

def get_secrets() -> Secrets:
    global _secrets
    if _secrets:
        return _secrets

    missing = []
    required = ["ANTHROPIC_API_KEY", "DATABASE_URL"]
    
    for key in required:
        if not os.getenv(key):
            missing.append(key)
    
    if missing:
        raise EnvironmentError(
            f"❌ 缺少必要环境变量: {', '.join(missing)}\n"
            f"请复制 .env.example 到 .env.local 并填写"
        )

    _secrets = Secrets(
        ANTHROPIC_API_KEY=os.environ["ANTHROPIC_API_KEY"],
        DATABASE_URL=os.environ["DATABASE_URL"],
        REDIS_URL=os.getenv("REDIS_URL"),
        GITHUB_TOKEN=os.getenv("GITHUB_TOKEN"),
    )
    return _secrets
```

---

## 层级三：Agent 工具层的密钥注入

Agent 工具调用时不应该自己去读 `process.env`，而应该通过**依赖注入**获取密钥：

```typescript
// tools/github-tool.ts
interface GitHubToolContext {
  githubToken: string; // 注入，不是自己读环境变量
}

export function createGitHubTool(ctx: GitHubToolContext) {
  return {
    name: "github_create_issue",
    description: "在 GitHub 仓库创建 Issue",
    inputSchema: {
      type: "object",
      properties: {
        repo: { type: "string" },
        title: { type: "string" },
        body: { type: "string" },
      },
      required: ["repo", "title"],
    },
    async execute({ repo, title, body }: { repo: string; title: string; body?: string }) {
      const res = await fetch(`https://api.github.com/repos/${repo}/issues`, {
        method: "POST",
        headers: {
          Authorization: `Bearer ${ctx.githubToken}`, // 使用注入的密钥
          "Content-Type": "application/json",
        },
        body: JSON.stringify({ title, body }),
      });
      return res.json();
    },
  };
}

// Agent 初始化时注入
const secrets = getSecrets();
const tools = [
  createGitHubTool({ githubToken: secrets.GITHUB_TOKEN! }),
  // ... 其他工具
];
```

**好处**：
1. 工具可以独立单元测试（传入假 token）
2. 密钥来源集中在一处
3. 工具不依赖全局 `process.env`

---

## 层级四：运行时密钥轮换

生产环境密钥必须支持轮换，Agent 不能重启：

```typescript
// config/secret-store.ts
import EventEmitter from "events";

class SecretStore extends EventEmitter {
  private store = new Map<string, string>();
  private ttlMap = new Map<string, NodeJS.Timeout>();

  set(key: string, value: string, ttlMs?: number) {
    this.store.set(key, value);
    
    if (ttlMs) {
      // 清除旧定时器
      const old = this.ttlMap.get(key);
      if (old) clearTimeout(old);
      
      // TTL 到期前 10s 发出警告
      const warnTimer = setTimeout(() => {
        this.emit("expiring", key);
      }, ttlMs - 10_000);
      
      this.ttlMap.set(key, warnTimer);
    }
    
    this.emit("updated", key);
  }

  get(key: string): string {
    const val = this.store.get(key);
    if (!val) throw new Error(`Secret '${key}' not found`);
    return val;
  }

  // 支持轮换：外部调用此方法更新密钥
  rotate(key: string, newValue: string) {
    console.log(`🔄 轮换密钥: ${key}`);
    this.set(key, newValue);
  }
}

export const secretStore = new SecretStore();

// 监听密钥即将过期
secretStore.on("expiring", (key) => {
  console.warn(`⚠️ 密钥即将过期: ${key}，请及时轮换`);
  // 可接入告警通知
});
```

```typescript
// 工具使用 SecretStore 而不是硬绑定密钥
export function createDynamicGitHubTool() {
  return {
    name: "github_create_issue",
    async execute({ repo, title }: { repo: string; title: string }) {
      // 每次调用时实时获取，支持热轮换
      const token = secretStore.get("GITHUB_TOKEN");
      // ...
    },
  };
}
```

---

## 层级五：防止密钥泄露进日志

Agent 最大的泄露风险：**工具参数/响应被原样记录到日志**。

```typescript
// middleware/audit-log.ts
const SECRET_PATTERNS = [
  /sk-ant-[a-zA-Z0-9-_]{10,}/g,     // Anthropic
  /ghp_[a-zA-Z0-9]{36}/g,           // GitHub PAT
  /AKIA[0-9A-Z]{16}/g,              // AWS Access Key
  /AIza[0-9A-Za-z-_]{35}/g,         // Google API Key
  /Bearer [a-zA-Z0-9._-]{20,}/g,    // Generic Bearer
];

export function maskSecrets(text: string): string {
  let masked = text;
  for (const pattern of SECRET_PATTERNS) {
    masked = masked.replace(pattern, "[REDACTED]");
  }
  return masked;
}

// 包装 JSON 序列化
export function safeStringify(obj: unknown): string {
  return maskSecrets(JSON.stringify(obj, null, 2));
}

// 工具调用日志中间件
export function withSecretMasking<T extends (...args: any[]) => any>(
  toolName: string,
  fn: T
): T {
  return (async (...args: any[]) => {
    const maskedArgs = args.map((a) =>
      typeof a === "object" ? JSON.parse(maskSecrets(JSON.stringify(a))) : a
    );
    console.log(`[Tool] ${toolName} args:`, safeStringify(maskedArgs));

    const result = await fn(...args);

    console.log(`[Tool] ${toolName} result:`, safeStringify(result));
    return result;
  }) as T;
}
```

---

## 层级六：OpenClaw 的密钥管理实践

OpenClaw（本课程运行的平台）使用的几个模式可直接参考：

### 1. TOOLS.md 作为本地密钥备注
```markdown
## 各服务凭证
- AWS Access Key: AKIAZMON3HGVXUPJLEHN（billing only）
- Laravel Cloud Token: 149|15UZ...
```
注意：TOOLS.md 在 workspace 里，**不提交公开仓库**。

### 2. 系统提示注入密钥上下文
OpenClaw 把 TOOLS.md 注入到 Agent context 而不是代码里，
Agent 需要时从 context 读取，不落磁盘日志。

### 3. 分权原则（最小权限）
```markdown
- AWS Billing: ViewOnlyAccess + AWSBillingReadOnlyAccess
- Laravel Forge: server:view, server:delete-keys（不含 create）
```
每个密钥只有完成任务的最小权限。

---

## 完整 Agent 密钥管理架构

```
┌─────────────────────────────────────────────┐
│              Secret Sources                  │
│   .env.local  │  Vault  │  Cloud Secrets     │
└──────────────────────┬──────────────────────┘
                       ↓
            ┌─────────────────┐
            │  SecretStore    │ ← 支持热轮换
            │  + Validation   │ ← Zod 强校验
            │  + TTL 管理     │ ← 过期预警
            └────────┬────────┘
                     ↓
        ┌────────────────────────┐
        │   Dependency Injection  │
        │   (工具初始化时注入)     │
        └────────────┬───────────┘
                     ↓
        ┌────────────────────────┐
        │   Tool Execution       │
        │   + Secret Masking     │ ← 日志脱敏
        │   + Audit Log          │ ← 操作审计
        └────────────────────────┘
```

---

## 关键原则总结

| 原则 | 说明 |
|------|------|
| **启动时校验** | Agent 启动即检查所有必要密钥，快速失败 |
| **依赖注入** | 工具通过参数接收密钥，不自己读环境变量 |
| **日志脱敏** | 所有日志输出前过 maskSecrets() |
| **最小权限** | 每个密钥只授予必要的操作权限 |
| **支持轮换** | 设计上允许密钥热更新，无需重启 Agent |
| **审计追踪** | 记录谁在什么时间用了什么密钥做了什么 |
| **永不提交** | .env.local / production secrets 进 .gitignore |

---

## OpenClaw 实战示例

本课程使用的 OpenClaw 本身就是这套模式的活例子：

```bash
# openclaw 的密钥通过系统级环境变量注入
# 查看当前 gateway 配置（密钥已脱敏展示）
openclaw gateway status
```

Agent 在 TOOLS.md 里记录服务凭证，通过 Bootstrap 注入到 context，
工具调用时从 context 取值，而不是 `process.env.XXXX`。
这样即使 Agent 崩溃，密钥也不会出现在 core dump 里。

---

## 参考资料

- [The Twelve-Factor App - Config](https://12factor.net/config)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [HashiCorp Vault Agent Injector](https://developer.hashicorp.com/vault/docs/agent-and-proxy/agent)
