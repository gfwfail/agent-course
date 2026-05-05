# 248. Agent 能力吊销与会话撤权（Capability Revocation & Session Deauthorization）

> 一句话：授权不是发出去就永远有效；生产 Agent 必须能在用户说“停”、风险升高、凭证泄露、子任务越权时，立刻让已经发出的能力失效。

上一课讲了 **DelegationToken**：父 Agent 给子 Agent 一张短期、受限、可审计的“通行证”。今天补上另一半：**通行证怎么撤销**。

如果只有 TTL，没有吊销机制，会出现几个生产事故：

- 用户已经取消任务，但后台 sub-agent 还在继续发消息/改代码/部署。
- 人工审批后发现目标环境变了，但旧 approval token 仍可执行。
- 某个工具凭证疑似泄露，只能等 token 过期，窗口期太危险。
- 多 Agent 协作里，一个子任务被判定越权，但它手里的能力还没失效。

所以成熟 Agent 的权限模型不是：

```text
grant capability -> hope it expires soon
```

而是：

```text
grant capability -> check revocation before every side effect -> revoke by token / session / actor / scope
```

---

## 1. 核心模型：Capability + Revocation Registry

每张能力票据至少要有：

```json
{
  "jti": "cap_01HZ...",              // token 唯一 ID
  "sessionId": "sess_123",
  "actor": "agent:worker-7",
  "scopes": ["repo:read", "message:send"],
  "expiresAt": 1777993200,
  "capabilityVersion": 12
}
```

吊销系统不要只存一堆 revoked token，还要支持几种粒度：

| 粒度 | 用途 |
|---|---|
| `jti` | 精确吊销某张 token |
| `sessionId` | 用户取消整个会话/任务 |
| `actor` | 某个 sub-agent 异常，撤掉它所有能力 |
| `scope` | 某类工具风险升高，临时禁用，比如 `deploy:*` |
| `capabilityVersion` | 全局权限策略升级后，让旧版本票据全部失效 |

关键点：**吊销检查必须发生在工具执行层，而不是只靠 LLM 自觉。**

---

## 2. learn-claude-code：Python 教学版

教学版可以用一个内存 Registry 表达核心思想：

```python
from dataclasses import dataclass
from time import time
from typing import set

@dataclass(frozen=True)
class Capability:
    jti: str
    session_id: str
    actor: str
    scopes: set[str]
    expires_at: float
    capability_version: int

class RevocationRegistry:
    def __init__(self):
        self.revoked_jti: set[str] = set()
        self.revoked_sessions: set[str] = set()
        self.revoked_actors: set[str] = set()
        self.revoked_scopes: set[str] = set()
        self.min_capability_version = 1

    def revoke_token(self, jti: str):
        self.revoked_jti.add(jti)

    def revoke_session(self, session_id: str):
        self.revoked_sessions.add(session_id)

    def revoke_scope(self, scope: str):
        self.revoked_scopes.add(scope)

    def assert_allowed(self, cap: Capability, required_scope: str):
        if time() > cap.expires_at:
            raise PermissionError("capability expired")
        if cap.capability_version < self.min_capability_version:
            raise PermissionError("capability policy version revoked")
        if cap.jti in self.revoked_jti:
            raise PermissionError("capability token revoked")
        if cap.session_id in self.revoked_sessions:
            raise PermissionError("session revoked")
        if cap.actor in self.revoked_actors:
            raise PermissionError("actor revoked")
        if required_scope not in cap.scopes:
            raise PermissionError(f"missing scope: {required_scope}")
        if required_scope in self.revoked_scopes:
            raise PermissionError(f"scope revoked: {required_scope}")
```

然后工具执行前强制调用：

```python
class ToolDispatcher:
    def __init__(self, revocations: RevocationRegistry):
        self.revocations = revocations

    def call(self, tool_name: str, args: dict, cap: Capability):
        required_scope = self.scope_for(tool_name)
        self.revocations.assert_allowed(cap, required_scope)

        # 只有通过吊销检查，才真正执行副作用
        return self.execute(tool_name, args)

    def scope_for(self, tool_name: str) -> str:
        return {
            "send_telegram": "message:send",
            "git_push": "repo:write",
            "deploy_prod": "deploy:prod",
        }[tool_name]
```

注意：这里 `assert_allowed()` 不应该只在任务开始时跑一次。长任务里每次调用工具都要重新检查，因为用户可能在中途点了取消。

---

## 3. pi-mono：TypeScript 生产中间件

生产版建议把能力吊销做成 Tool Middleware，所有工具统一过闸：

```ts
type Capability = {
  jti: string;
  sessionId: string;
  actor: string;
  scopes: string[];
  expiresAt: number;
  capabilityVersion: number;
};

type ToolContext = {
  runId: string;
  toolName: string;
  capability: Capability;
};

interface RevocationStore {
  isTokenRevoked(jti: string): Promise<boolean>;
  isSessionRevoked(sessionId: string): Promise<boolean>;
  isActorRevoked(actor: string): Promise<boolean>;
  isScopeRevoked(scope: string): Promise<boolean>;
  getMinCapabilityVersion(): Promise<number>;
}

const toolScopes: Record<string, string> = {
  message_send: "message:send",
  git_push: "repo:write",
  deploy_environment: "deploy:prod",
};

export function capabilityRevocationMiddleware(store: RevocationStore) {
  return async function beforeTool(ctx: ToolContext) {
    const scope = toolScopes[ctx.toolName];
    if (!scope) throw new Error(`No scope registered for tool: ${ctx.toolName}`);

    const cap = ctx.capability;
    const now = Math.floor(Date.now() / 1000);

    if (cap.expiresAt <= now) throw new Error("CAPABILITY_EXPIRED");
    if (!cap.scopes.includes(scope)) throw new Error(`MISSING_SCOPE:${scope}`);
    if (cap.capabilityVersion < await store.getMinCapabilityVersion()) {
      throw new Error("CAPABILITY_VERSION_REVOKED");
    }
    if (await store.isTokenRevoked(cap.jti)) throw new Error("CAPABILITY_REVOKED");
    if (await store.isSessionRevoked(cap.sessionId)) throw new Error("SESSION_REVOKED");
    if (await store.isActorRevoked(cap.actor)) throw new Error("ACTOR_REVOKED");
    if (await store.isScopeRevoked(scope)) throw new Error(`SCOPE_REVOKED:${scope}`);
  };
}
```

这段逻辑的价值在于：

- LLM 即使继续输出“我要 push”，工具层也会拒绝。
- 用户取消任务后，只要 `revokeSession(sessionId)`，所有后续副作用立即失败。
- 如果发现 `deploy:prod` 风险，可以临时 `revokeScope("deploy:prod")`，不影响读操作。

---

## 4. OpenClaw 实战：取消长任务、撤掉子 Agent 权限

OpenClaw 里常见场景：主会话把修 bug 任务派给 sub-agent，sub-agent 准备改文件、跑测试、开 PR。

推荐流程：

```text
1. Parent run 创建 delegation token：
   scopes = ["repo:read", "repo:write", "test:run"]
   no "message:send", no "deploy:prod"

2. 每个工具调用前检查：
   token 未过期、session 未 revoked、scope 未 revoked

3. 用户说“停”或发现方向错：
   revokeSession(parentRunId)

4. sub-agent 下次工具调用：
   ToolDispatcher 拒绝执行，保存 checkpoint，返回 SESSION_REVOKED
```

一个轻量文件式实现可以这样放：

```json
// memory/revocations.json
{
  "revokedSessions": ["run_2026_05_06_0330"],
  "revokedTokens": ["cap_01HZABC"],
  "revokedScopes": ["deploy:prod"],
  "minCapabilityVersion": 12
}
```

Tool Middleware 每次读取/缓存这个文件，并设置很短 TTL（比如 1-5 秒）。注意不要缓存太久，否则“停”按钮不是真停。

---

## 5. 设计 Checklist

做能力吊销时，至少检查这 8 项：

1. **每张 token 有 `jti`**：否则不能精确吊销。
2. **每次工具调用前检查**：不是任务启动时检查一次。
3. **支持 session 级吊销**：用户取消时一键刹车。
4. **支持 scope 级吊销**：风险升级时禁用某类副作用。
5. **支持 capabilityVersion**：权限策略升级后旧 token 自动失效。
6. **错误可恢复**：返回 `SESSION_REVOKED` / `SCOPE_REVOKED`，让 Agent checkpoint，而不是假装网络错误重试。
7. **吊销写入要审计**：记录 operator、reason、time。
8. **高风险工具不接受本地旧缓存**：deploy、付款、删库等必须查强一致存储。

---

## 6. 常见坑

**坑 1：只靠 TTL**

TTL 是过期，不是吊销。TTL 解决“最终失效”，吊销解决“现在立刻失效”。

**坑 2：LLM 层判断权限**

LLM 可以参与解释原因，但不能作为权限边界。真正的 deny 必须在 Tool Dispatcher / Middleware。

**坑 3：撤销了父任务，子任务还活着**

子任务 token 必须绑定 `parentRunId` 或 `sessionId`。撤销父 session 时，所有 child capability 一起失效。

**坑 4：吊销检查结果缓存太久**

低风险读工具可以短缓存；高风险写工具必须实时检查。

---

## 7. 总结

DelegationToken 解决“给谁什么权限”；Revocation Registry 解决“什么时候把权限拿回来”。

一个生产级 Agent 的权限闭环应该是：

```text
Grant -> Use -> Revalidate -> Revoke -> Audit
```

没有吊销的授权，只是带过期时间的信任；有吊销的授权，才是可控的能力系统。
