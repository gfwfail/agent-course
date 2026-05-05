# 第 247 课：Agent 委托令牌与子任务权限链（Delegation Tokens & Subtask Capability Chains）

> 关键词：子任务委托、能力令牌、权限收缩、调用链审计、Sub-agent 安全

Agent 一旦支持 Sub-agent / Worker / Background Task，就会遇到一个生产级问题：

> 主 Agent 可以做的事，子 Agent 是否也都能做？

答案必须是：**不能默认继承全部权限，只能拿到本次子任务需要的最小能力。**

比如老板让主 Agent “审计仓库供应链风险”，主 Agent 可能有 GitHub、文件系统、发消息权限；但被派出去的子 Agent 只需要只读 clone + read file，不应该顺手拥有 push、deploy、发群消息权限。

这就是本课：**委托令牌与子任务权限链**。

---

## 1. 核心思想：把“派活”变成可验证的能力收缩

不要让子 Agent 直接继承 parent session 的所有工具，而是给它一张短期 Delegation Token：

```ts
type DelegationToken = {
  tokenId: string;
  parentRunId: string;
  childRunId: string;
  task: string;
  allowedTools: string[];
  scopes: string[];
  constraints: {
    readonly?: boolean;
    maxDurationMs?: number;
    maxToolCalls?: number;
    allowedPaths?: string[];
    allowedHosts?: string[];
  };
  issuedAt: string;
  expiresAt: string;
  nonce: string;
  signature: string;
};
```

关键规则：

- **权限只能缩小，不能扩大**：child scopes 必须是 parent scopes 的子集。
- **令牌有 TTL**：子任务过期后不能继续执行外部副作用。
- **约束要可执行**：allowedTools / allowedPaths / readonly 不能只是写在 prompt 里，要在工具分发层强制校验。
- **链路可审计**：每次工具调用都记录 delegation token id，出事后能追到是谁委托的。

---

## 2. 为什么光靠 Prompt 不够？

错误做法：

```text
你是子 Agent，只能读取文件，不要 push，不要发消息。
```

这只是“请求模型听话”。生产系统需要“模型不听话也做不到”：

```text
LLM 生成 tool_call
        ↓
Tool Dispatcher 检查 DelegationToken
        ↓
不在 allowedTools / scopes / constraints 内：直接拒绝
        ↓
合法才执行真实工具
```

Agent 安全的原则是：**Prompt 负责表达意图，Runtime 负责强制边界。**

---

## 3. learn-claude-code：Python 教学版

先实现一个最小委托令牌和校验器：

```python
# learn_claude_code/delegation.py
from __future__ import annotations

import hashlib
import hmac
import json
import time
import uuid
from dataclasses import dataclass, asdict
from typing import Any

SECRET = b"dev-signing-key"  # 生产里来自 KMS/Secret Manager


@dataclass(frozen=True)
class DelegationToken:
    token_id: str
    parent_run_id: str
    child_run_id: str
    task: str
    allowed_tools: list[str]
    scopes: list[str]
    readonly: bool
    allowed_paths: list[str]
    issued_at: float
    expires_at: float
    nonce: str
    signature: str


def canonical_payload(data: dict[str, Any]) -> bytes:
    unsigned = {k: v for k, v in data.items() if k != "signature"}
    return json.dumps(unsigned, sort_keys=True, separators=(",", ":")).encode()


def sign(data: dict[str, Any]) -> str:
    return hmac.new(SECRET, canonical_payload(data), hashlib.sha256).hexdigest()


def issue_delegation_token(
    *,
    parent_run_id: str,
    child_run_id: str,
    task: str,
    parent_scopes: set[str],
    requested_tools: list[str],
    requested_scopes: set[str],
    allowed_paths: list[str],
    ttl_seconds: int = 900,
) -> DelegationToken:
    if not requested_scopes.issubset(parent_scopes):
        raise PermissionError("child scopes cannot exceed parent scopes")

    data = {
        "token_id": f"del_{uuid.uuid4().hex[:12]}",
        "parent_run_id": parent_run_id,
        "child_run_id": child_run_id,
        "task": task,
        "allowed_tools": sorted(requested_tools),
        "scopes": sorted(requested_scopes),
        "readonly": True,
        "allowed_paths": sorted(allowed_paths),
        "issued_at": time.time(),
        "expires_at": time.time() + ttl_seconds,
        "nonce": uuid.uuid4().hex,
        "signature": "",
    }
    data["signature"] = sign(data)
    return DelegationToken(**data)


def verify_token(token: DelegationToken) -> None:
    data = asdict(token)
    if not hmac.compare_digest(token.signature, sign(data)):
        raise PermissionError("invalid delegation token signature")
    if time.time() > token.expires_at:
        raise PermissionError("delegation token expired")
```

再把它接到工具分发层：

```python
# learn_claude_code/tool_dispatcher.py
from pathlib import Path
from delegation import DelegationToken, verify_token

WRITE_TOOLS = {"write", "edit", "apply_patch", "git_push", "send_message"}


def assert_tool_allowed(token: DelegationToken, tool_name: str, args: dict) -> None:
    verify_token(token)

    if tool_name not in token.allowed_tools:
        raise PermissionError(f"tool not delegated: {tool_name}")

    if token.readonly and tool_name in WRITE_TOOLS:
        raise PermissionError(f"readonly delegation blocks write tool: {tool_name}")

    path = args.get("path") or args.get("workdir")
    if path:
        resolved = str(Path(path).resolve())
        allowed = [str(Path(p).resolve()) for p in token.allowed_paths]
        if not any(resolved.startswith(prefix) for prefix in allowed):
            raise PermissionError(f"path outside delegated boundary: {resolved}")


async def dispatch_tool(tool_name: str, args: dict, token: DelegationToken):
    assert_tool_allowed(token, tool_name, args)
    return await REAL_TOOLS[tool_name](**args)
```

这样即使子 Agent 幻觉出 `git_push`，Dispatcher 也会拒绝。

---

## 4. pi-mono：TypeScript 生产版中间件

生产版建议把委托做成 Middleware，所有工具调用统一过闸：

```ts
// pi-mono/packages/agent-runtime/src/security/DelegationToken.ts
import crypto from "node:crypto";

export type DelegationToken = {
  tokenId: string;
  parentRunId: string;
  childRunId: string;
  task: string;
  allowedTools: string[];
  scopes: string[];
  constraints: {
    readonly: boolean;
    maxToolCalls: number;
    allowedPaths?: string[];
    allowedHosts?: string[];
  };
  issuedAt: string;
  expiresAt: string;
  nonce: string;
  signature: string;
};

function canonicalJson(value: unknown): string {
  return JSON.stringify(value, Object.keys(value as object).sort());
}

export function signDelegationToken(
  token: Omit<DelegationToken, "signature">,
  secret: string,
): string {
  return crypto
    .createHmac("sha256", secret)
    .update(canonicalJson(token))
    .digest("hex");
}

export function verifyDelegationToken(token: DelegationToken, secret: string): void {
  const { signature, ...unsigned } = token;
  const expected = signDelegationToken(unsigned, secret);

  if (!crypto.timingSafeEqual(Buffer.from(signature), Buffer.from(expected))) {
    throw new Error("DELEGATION_SIGNATURE_INVALID");
  }
  if (Date.now() > Date.parse(token.expiresAt)) {
    throw new Error("DELEGATION_EXPIRED");
  }
}
```

工具中间件：

```ts
// pi-mono/packages/agent-runtime/src/security/DelegationMiddleware.ts
import path from "node:path";
import { verifyDelegationToken, type DelegationToken } from "./DelegationToken";

type ToolCall = {
  name: string;
  args: Record<string, unknown>;
  risk?: "low" | "medium" | "high" | "critical";
  write?: boolean;
};

type ToolContext = {
  runId: string;
  delegation?: DelegationToken;
  delegationSecret: string;
  toolCallCount: number;
};

function isPathAllowed(candidate: string, allowedPaths: string[]): boolean {
  const resolved = path.resolve(candidate);
  return allowedPaths.some((base) => resolved.startsWith(path.resolve(base)));
}

export async function withDelegationGuard<T>(
  call: ToolCall,
  ctx: ToolContext,
  next: () => Promise<T>,
): Promise<T> {
  const token = ctx.delegation;
  if (!token) {
    // 主 Agent 可以继续走普通权限系统；子 Agent 必须有 token。
    return next();
  }

  verifyDelegationToken(token, ctx.delegationSecret);

  if (token.childRunId !== ctx.runId) {
    throw new Error("DELEGATION_RUN_MISMATCH");
  }

  if (!token.allowedTools.includes(call.name)) {
    throw new Error(`DELEGATION_TOOL_DENIED:${call.name}`);
  }

  if (token.constraints.readonly && call.write) {
    throw new Error(`DELEGATION_READONLY_DENIED:${call.name}`);
  }

  if (ctx.toolCallCount >= token.constraints.maxToolCalls) {
    throw new Error("DELEGATION_TOOL_CALL_BUDGET_EXCEEDED");
  }

  const p = call.args.path ?? call.args.workdir;
  if (typeof p === "string" && token.constraints.allowedPaths) {
    if (!isPathAllowed(p, token.constraints.allowedPaths)) {
      throw new Error(`DELEGATION_PATH_DENIED:${p}`);
    }
  }

  return next();
}
```

委托创建时强制“权限只能缩小”：

```ts
function assertSubset(child: string[], parent: string[], label: string) {
  const parentSet = new Set(parent);
  const extra = child.filter((x) => !parentSet.has(x));
  if (extra.length > 0) {
    throw new Error(`${label} exceeds parent capability: ${extra.join(",")}`);
  }
}

export function createChildDelegation(input: {
  parentRunId: string;
  childRunId: string;
  parentScopes: string[];
  requestedScopes: string[];
  parentTools: string[];
  requestedTools: string[];
}): Omit<DelegationToken, "signature"> {
  assertSubset(input.requestedScopes, input.parentScopes, "scope");
  assertSubset(input.requestedTools, input.parentTools, "tool");

  return {
    tokenId: `del_${crypto.randomUUID()}`,
    parentRunId: input.parentRunId,
    childRunId: input.childRunId,
    task: "audit repository dependency files",
    allowedTools: input.requestedTools,
    scopes: input.requestedScopes,
    constraints: {
      readonly: true,
      maxToolCalls: 30,
      allowedPaths: ["/workspace/repo"],
    },
    issuedAt: new Date().toISOString(),
    expiresAt: new Date(Date.now() + 15 * 60_000).toISOString(),
    nonce: crypto.randomUUID(),
  };
}
```

---

## 5. OpenClaw 实战：Sub-agent 派活时不要“全家桶权限”

OpenClaw 里派 Sub-agent 常见写法是：

```text
让子 Agent 去检查这个 repo 的依赖风险。
```

更安全的写法应该把边界讲清楚，并由 runtime 限制工具：

```text
任务：只读审计 /workspace/project 的依赖与 CI 工作流。
允许：read、grep、git status、gh api read-only。
禁止：write/edit/apply_patch/git push/message/deploy。
输出：风险列表 + 证据路径 + 建议，不要修改文件。
```

如果平台支持结构化 delegation metadata，更应该传：

```json
{
  "allowedTools": ["read", "exec:readonly", "gh:readonly"],
  "scopes": ["repo:read", "filesystem:read"],
  "constraints": {
    "readonly": true,
    "allowedPaths": ["/Users/bot001/.openclaw/workspace/agent-course"],
    "maxToolCalls": 40,
    "maxDurationMs": 900000
  }
}
```

这和前几课的权限、证据、环境指纹是同一条安全链：

- Access Scope 决定“谁能看什么”；
- Delegation Token 决定“子任务临时能做什么”；
- Plan Signing 决定“审批的是不是同一个计划”；
- Env Fingerprint 决定“执行环境是不是同一个环境”。

---

## 6. 常见坑

### 坑 1：子 Agent 默认继承主 Agent 全部权限

这会让“小任务”拥有“大权限”。一旦 prompt injection 进入子任务，就可能直接扩大事故半径。

正确做法：子任务默认无权限，必须显式授权。

### 坑 2：只在 Prompt 里写禁止事项

Prompt 是提醒，不是安全边界。真正的禁止必须在 Tool Dispatcher / Middleware 层执行。

### 坑 3：委托令牌没有绑定 childRunId

如果 token 只写 allowedTools，不绑定具体 run，就可能被其他任务复用。

正确做法：token 绑定 parentRunId + childRunId + nonce + TTL。

### 坑 4：只审计最终结果，不审计中间工具调用

安全审计要看每次工具调用：哪个 token、哪个 run、哪个工具、什么参数、为什么允许。

---

## 7. 生产 Checklist

- [ ] 子 Agent 默认无权限，必须显式 delegation。
- [ ] child scopes / tools 必须是 parent 的子集。
- [ ] token 有 TTL、nonce、签名，并绑定 childRunId。
- [ ] Tool Dispatcher 强制校验 allowedTools / readonly / allowedPaths。
- [ ] 每次工具调用写审计日志：tokenId、parentRunId、childRunId、toolName、decision。
- [ ] 高风险工具禁止被委托，或必须重新人工审批。
- [ ] 子 Agent 输出不能自动触发外部副作用，必须回到 parent 汇总后执行。

---

## 8. 一句话总结

> 子 Agent 不是主 Agent 的分身，而是拿着临时工牌的外包工：只进指定房间，只干指定活，到点工牌失效。成熟 Agent 的委托不是“相信它听话”，而是“它越界也做不到”。
