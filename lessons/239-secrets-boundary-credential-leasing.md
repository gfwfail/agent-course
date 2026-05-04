# 第 239 课：Agent 密钥边界与短期凭证租赁（Secrets Boundary & Credential Leasing）

> 核心：**LLM 不应该“知道”密钥，只应该请求一个受限、短期、可审计的能力。**

很多 Agent Demo 会把 `API_KEY`、数据库密码、GitHub Token 直接塞进 system prompt、工具参数或上下文文件。短期看很方便，长期看这是生产事故预备役：

- prompt injection 可能诱导模型复述密钥；
- tool log / trace / session history 会把密钥永久保存；
- sub-agent 拿到超出任务所需的全量权限；
- 一次泄露等于长期凭证失效，轮换成本很高。

成熟做法是加一层 **Secrets Boundary**：模型只看到 `secretRef` / `capability`，真正密钥只在工具执行层短暂解析，并且通过 **Credential Lease** 控制范围、TTL 和审计。

---

## 1. 反模式：把 secret 当普通上下文

```ts
// ❌ 反模式：LLM prompt 能直接看到 token
const systemPrompt = `
You can deploy Laravel Cloud.
LARAVEL_CLOUD_TOKEN=${process.env.LARAVEL_CLOUD_TOKEN}
`;
```

问题不是“模型会不会故意泄露”，而是：

1. 用户输入、网页内容、issue body 都可能包含 prompt injection；
2. Agent 会把上下文传给 sub-agent、日志、评估器、摘要器；
3. 未来任何一次 debug / replay 都可能再次暴露密钥。

**原则：Prompt 里永远不放 secret value，只放能力说明。**

```ts
// ✅ 只告诉模型能力，不告诉它密钥
const systemPrompt = `
You may request capability: laravel-cloud:deploy.
The tool layer will resolve credentials if policy allows it.
Never ask user to paste secrets into chat.
`;
```

---

## 2. 数据模型：SecretRef + CredentialLease

### SecretRef：稳定引用，不是密钥本身

```ts
type SecretRef = {
  provider: 'vault' | 'env' | 'openclaw-tools';
  key: string;                  // e.g. "laravel_cloud/api_token"
  scope: string;                // e.g. "deploy:mysterybox:production"
};
```

### CredentialLease：一次任务的短期授权

```ts
type CredentialLease = {
  leaseId: string;
  secretRef: SecretRef;
  grantedToRunId: string;
  allowedTools: string[];       // ['laravelCloud.deploy']
  expiresAt: string;            // ISO timestamp
  maxUses: number;
  used: number;
  reason: string;
};
```

关键点：

- `SecretRef` 可以进入上下文；
- `CredentialLease` 可以进入审计日志；
- **secret value 只存在于工具执行函数的局部变量里**，用完即丢；
- lease 过期、工具不匹配、次数超限都拒绝执行。

---

## 3. learn-claude-code：Python 最小教学版

```python
# learn-claude-code / secrets_boundary.py
from dataclasses import dataclass
from datetime import datetime, timedelta, timezone
import os
import uuid

@dataclass
class SecretRef:
    provider: str
    key: str
    scope: str

@dataclass
class CredentialLease:
    lease_id: str
    secret_ref: SecretRef
    run_id: str
    allowed_tool: str
    expires_at: datetime
    max_uses: int = 1
    used: int = 0

class SecretBoundary:
    def __init__(self):
        self.leases: dict[str, CredentialLease] = {}

    def grant(self, run_id: str, secret_ref: SecretRef, tool: str, ttl_seconds=300):
        # 真实生产里这里还要检查 operator / task / env policy
        lease = CredentialLease(
            lease_id=str(uuid.uuid4()),
            secret_ref=secret_ref,
            run_id=run_id,
            allowed_tool=tool,
            expires_at=datetime.now(timezone.utc) + timedelta(seconds=ttl_seconds),
        )
        self.leases[lease.lease_id] = lease
        return lease.lease_id

    def resolve_for_tool(self, lease_id: str, run_id: str, tool: str) -> str:
        lease = self.leases[lease_id]
        now = datetime.now(timezone.utc)

        if lease.run_id != run_id:
            raise PermissionError("lease belongs to another run")
        if lease.allowed_tool != tool:
            raise PermissionError("lease cannot be used by this tool")
        if lease.expires_at < now:
            raise PermissionError("lease expired")
        if lease.used >= lease.max_uses:
            raise PermissionError("lease max uses exceeded")

        lease.used += 1

        # secret value 只在这里解析，不返回给 LLM
        if lease.secret_ref.provider == "env":
            return os.environ[lease.secret_ref.key]
        raise ValueError("unsupported secret provider")

boundary = SecretBoundary()

async def deploy_laravel_cloud(run_id: str, lease_id: str, app_id: str):
    token = boundary.resolve_for_tool(
        lease_id=lease_id,
        run_id=run_id,
        tool="laravelCloud.deploy",
    )
    # 只把 token 放进 HTTP header；日志里永远不要打印
    return await http_post(
        f"https://cloud.laravel.com/api/applications/{app_id}/deploy",
        headers={"Authorization": f"Bearer {token}"},
    )
```

这版代码故意很小，但已经具备三个生产关键点：

1. secret value 不进入 prompt；
2. lease 与 run_id 绑定，sub-agent 不能随便复用；
3. 每个 lease 有 TTL 和 maxUses。

---

## 4. pi-mono：中间件版 SecretLeaseMiddleware

生产版应该把 secret 解析放进工具分发层，而不是散落在每个工具里。

```ts
// pi-mono / middleware/SecretLeaseMiddleware.ts
import type { ToolCall, ToolContext, ToolMiddleware } from '../tooling/types';

export class SecretLeaseMiddleware implements ToolMiddleware {
  constructor(
    private leases: CredentialLeaseStore,
    private vault: SecretVault,
    private audit: AuditLog,
  ) {}

  async beforeTool(call: ToolCall, ctx: ToolContext) {
    const required = ctx.toolMeta[call.name]?.requiredSecret;
    if (!required) return;

    const leaseId = call.args.credentialLeaseId;
    if (!leaseId) {
      throw new Error(`tool ${call.name} requires credentialLeaseId`);
    }

    const lease = await this.leases.claim({
      leaseId,
      runId: ctx.runId,
      toolName: call.name,
      now: new Date(),
    });

    const secretValue = await this.vault.resolve(lease.secretRef);

    // 放在 tool-private context，不回显给 LLM
    ctx.privateSecrets.set(required.name, secretValue);

    await this.audit.append({
      type: 'secret_lease_used',
      runId: ctx.runId,
      toolName: call.name,
      leaseId,
      secretRef: lease.secretRef,
      // 不记录 secretValue
    });

    // 防止 LLM 在工具参数里看到 lease 之外的信息
    delete call.args.credentialLeaseId;
  }

  async afterTool(_call: ToolCall, ctx: ToolContext) {
    ctx.privateSecrets.clear(); // 用完即丢
  }
}
```

工具只从 private context 取密钥：

```ts
export const laravelCloudDeploy = defineTool({
  name: 'laravelCloud.deploy',
  requiredSecret: {
    name: 'laravelCloudToken',
    scope: 'laravel-cloud:deploy',
  },
  async execute(args, ctx) {
    const token = ctx.privateSecrets.get('laravelCloudToken');

    return http.post(`/environments/${args.environmentId}/deployments`, args.body, {
      headers: { Authorization: `Bearer ${token}` },
      redactHeaders: ['authorization'],
    });
  },
});
```

这样工具 schema 里只暴露业务参数：

```json
{
  "environmentId": "env_xxx",
  "commit": "abc123"
}
```

而不是：

```json
{
  "token": "sk_live_xxx"
}
```

---

## 5. OpenClaw 实战：把 TOOLS.md 当“受保护配置”，不要当 prompt 材料

OpenClaw workspace 里常见有 `TOOLS.md`、`MEMORY.md`、项目 `.env` 等文件。它们对 Agent 很有用，但要分级：

- **可注入上下文**：非敏感偏好、项目路径、操作习惯；
- **只读但不复述**：账号 ID、内部 URL、基础设施说明；
- **只允许工具层使用**：API token、secret key、数据库密码；
- **必须审批**：生产部署、删除资源、发外部消息、大额计费操作。

一个安全的 OpenClaw 工具调用流程应该像这样：

```text
用户：帮我部署 mysterybox
  ↓
Agent：计划 deploy，不需要把 token 放进消息
  ↓
Policy：确认 run 有 laravel-cloud:deploy 权限
  ↓
SecretBoundary：发一个 5 分钟、1 次使用的 lease
  ↓
Tool：用 lease 解析 token，调用 Laravel Cloud API
  ↓
Audit：记录 leaseId / tool / appId / result，不记录 token
```

最终给用户的回复也只说：

```text
已触发 mysterybox 部署，deploymentId=dep_xxx
```

不要说：

```text
我用了 token 149|xxxx 完成部署
```

---

## 6. 防泄露清单

每个会用到密钥的 Agent，都建议加这 8 条：

1. **Prompt 禁密钥**：system/user/tool result 都不出现 secret value；
2. **日志自动脱敏**：`Authorization`、`X-API-Key`、`token`、`secret` 默认 redact；
3. **SecretRef 替代明文**：上下文里只传引用；
4. **Lease 绑定 runId/toolName**：不能跨任务、跨工具复用；
5. **TTL + maxUses**：短期、少次、自动失效；
6. **Sub-agent 最小授权**：子 Agent 只拿任务所需 capability，不继承全量密钥；
7. **审计只记引用**：记录谁、何时、为什么用了哪个 secretRef；
8. **高风险操作二次确认**：有密钥不等于允许执行危险动作。

---

## 7. 一句话总结

> Secret Boundary 的本质：**把“知道密码”变成“被授权执行一次能力”。**

生产 Agent 的安全，不是相信模型会保密，而是让模型根本拿不到需要保密的值。LLM 负责判断意图和编排流程；密钥、权限、审计，必须留在确定性的工具层。