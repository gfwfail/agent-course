# 318. Agent 证据链完整性验证（Evidence Chain Verification）

上一课讲了 **Evidence Index & Audit Query**：证据不能只存在文件里，还要能按 run、intent、action、subject 被安全查询。

今天继续往前走一步：**查到证据以后，怎么证明这一整条证据链没有断、没有被替换、没有被选择性隐藏？**

这就是 **Evidence Chain Verification**。

一句话：**单条证据证明“某件事发生过”，证据链证明“这件事从意图、审批、执行、回执到后置验证的因果路径是完整可信的”。**

---

## 1. 为什么需要证据链？

成熟 Agent 做副作用动作时，通常不止一个证据：

- 用户请求：为什么要做
- 计划：准备怎么做
- 策略决策：为什么允许做
- 审批：谁确认过
- 工具调用：实际执行了什么
- 外部回执：Telegram message id、git commit、deployment id
- 后置验证：测试、健康检查、diff check

如果这些证据互相独立存放，事故复盘时会遇到三个风险：

1. **断链**：只看到 git push 回执，看不到对应用户意图
2. **换链**：审批证据是真的，但绑定的不是最终执行命令
3. **漏链**：失败的 post-check 被选择性遗漏，只展示成功回执

所以生产系统不能只问“有没有证据”，而要问：

> 这组证据是否组成一条从 intent 到 outcome 的连续链路？每一环是否引用上一环的 hash？最终 root 是否可复算？

---

## 2. Evidence Chain 的最小结构

每个 evidence node 都带 `prevHash` / `parentIds`，关键节点再带 `chainRoot`：

```json
{
  "evidenceId": "ev_tool_git_push_01",
  "kind": "tool_result",
  "runId": "run_lesson_318",
  "intentId": "intent_agent_course",
  "parentIds": ["ev_policy_allow_01", "ev_diff_check_01"],
  "prevHash": "sha256:4b8c...",
  "payloadHash": "sha256:9a11...",
  "nodeHash": "sha256:d320...",
  "chainRoot": "sha256:root...",
  "createdAt": "2026-05-14T11:30:00Z"
}
```

核心原则：

- `payloadHash`：证明这条证据的内容没变
- `parentIds`：声明它依赖哪些前置证据
- `prevHash`：线性场景下指向上一环
- `nodeHash`：把 metadata + payloadHash + parent hash 一起签进去
- `chainRoot`：整条链或 DAG 的最终摘要

线性任务可以用 hash chain；复杂任务可以用 DAG + Merkle root。

---

## 3. learn-claude-code：Python 教学版链验证器

先看最小实现：把证据节点 canonicalize 后计算 hash，再验证 parent 是否存在、hash 是否匹配、链路是否能回到 intent。

```python
# evidence_chain.py
from __future__ import annotations

from dataclasses import dataclass, asdict
import hashlib
import json
from typing import Literal

Kind = Literal["intent", "plan", "policy", "approval", "tool_call", "tool_result", "post_check"]


def canonical_json(value: dict) -> str:
    return json.dumps(value, ensure_ascii=False, sort_keys=True, separators=(",", ":"))


def sha256_json(value: dict) -> str:
    return "sha256:" + hashlib.sha256(canonical_json(value).encode()).hexdigest()


@dataclass
class EvidenceNode:
    evidence_id: str
    kind: Kind
    run_id: str
    intent_id: str
    parent_ids: list[str]
    payload_hash: str
    created_at: str
    node_hash: str | None = None

    def unsigned(self) -> dict:
        data = asdict(self)
        data.pop("node_hash", None)
        return data

    def seal(self, parent_hashes: list[str]) -> "EvidenceNode":
        material = {
            **self.unsigned(),
            "parent_hashes": parent_hashes,
        }
        self.node_hash = sha256_json(material)
        return self


class EvidenceChainVerifier:
    def __init__(self, nodes: list[EvidenceNode]):
        self.nodes = {n.evidence_id: n for n in nodes}

    def verify(self, final_id: str) -> tuple[bool, list[str]]:
        errors: list[str] = []
        visited: set[str] = set()

        def walk(node_id: str) -> None:
            if node_id in visited:
                return
            visited.add(node_id)

            node = self.nodes.get(node_id)
            if not node:
                errors.append(f"missing node: {node_id}")
                return

            parent_hashes: list[str] = []
            for parent_id in node.parent_ids:
                parent = self.nodes.get(parent_id)
                if not parent:
                    errors.append(f"{node_id} references missing parent {parent_id}")
                    continue
                walk(parent_id)
                if parent.node_hash:
                    parent_hashes.append(parent.node_hash)

            expected = sha256_json({**node.unsigned(), "parent_hashes": parent_hashes})
            if node.node_hash != expected:
                errors.append(f"hash mismatch: {node_id}")

        walk(final_id)

        roots = [self.nodes[nid] for nid in visited if not self.nodes[nid].parent_ids]
        if not any(root.kind == "intent" for root in roots):
            errors.append("chain has no intent root")

        return (len(errors) == 0, errors)
```

使用方式：

```python
intent = EvidenceNode(
    evidence_id="ev_intent_01",
    kind="intent",
    run_id="run_318",
    intent_id="intent_course",
    parent_ids=[],
    payload_hash=sha256_json({"user": "cron", "ask": "publish lesson 318"}),
    created_at="2026-05-14T11:30:00Z",
).seal([])

policy = EvidenceNode(
    evidence_id="ev_policy_01",
    kind="policy",
    run_id="run_318",
    intent_id="intent_course",
    parent_ids=[intent.evidence_id],
    payload_hash=sha256_json({"decision": "allow", "reason": "scheduled_course"}),
    created_at="2026-05-14T11:31:00Z",
).seal([intent.node_hash])

result = EvidenceNode(
    evidence_id="ev_result_01",
    kind="tool_result",
    run_id="run_318",
    intent_id="intent_course",
    parent_ids=[policy.evidence_id],
    payload_hash=sha256_json({"telegramMessageId": 11942, "gitCommit": "abc123"}),
    created_at="2026-05-14T11:35:00Z",
).seal([policy.node_hash])

ok, errors = EvidenceChainVerifier([intent, policy, result]).verify(result.evidence_id)
assert ok, errors
```

这就是最小的 chain-of-custody：最终结果必须能一路追溯到原始意图。

---

## 4. pi-mono：生产中间件怎么接

在 pi-mono 里，不建议业务代码手写 evidence chain。更好的方式是把它做成 middleware：

```ts
// evidence-chain-middleware.ts
type EvidenceKind =
  | 'intent'
  | 'plan'
  | 'policy_decision'
  | 'approval'
  | 'tool_call'
  | 'tool_result'
  | 'post_check';

interface EvidenceNode {
  id: string;
  kind: EvidenceKind;
  runId: string;
  intentId: string;
  parentIds: string[];
  payloadHash: string;
  nodeHash: string;
  createdAt: string;
}

interface EvidenceChainStore {
  append(node: EvidenceNode): Promise<void>;
  get(id: string): Promise<EvidenceNode | null>;
  parents(node: EvidenceNode): Promise<EvidenceNode[]>;
}

class EvidenceChainMiddleware {
  constructor(private store: EvidenceChainStore) {}

  async beforeToolCall(ctx: ToolContext, next: () => Promise<ToolResult>) {
    const callNode = await this.appendNode({
      kind: 'tool_call',
      runId: ctx.runId,
      intentId: ctx.intentId,
      parentIds: [ctx.policyDecisionEvidenceId],
      payload: {
        tool: ctx.toolName,
        argsHash: hashCanonical(ctx.args),
        risk: ctx.risk,
      },
    });

    const result = await next();

    await this.appendNode({
      kind: 'tool_result',
      runId: ctx.runId,
      intentId: ctx.intentId,
      parentIds: [callNode.id],
      payload: {
        ok: result.ok,
        receiptHash: hashCanonical(result.receipt ?? {}),
        errorClass: result.error?.class,
      },
    });

    return result;
  }

  private async appendNode(input: {
    kind: EvidenceKind;
    runId: string;
    intentId: string;
    parentIds: string[];
    payload: unknown;
  }): Promise<EvidenceNode> {
    const parents = await Promise.all(input.parentIds.map(id => this.store.get(id)));
    if (parents.some(p => !p)) {
      throw new Error(`Evidence chain broken: missing parent for ${input.kind}`);
    }

    const node: EvidenceNode = {
      id: crypto.randomUUID(),
      kind: input.kind,
      runId: input.runId,
      intentId: input.intentId,
      parentIds: input.parentIds,
      payloadHash: hashCanonical(input.payload),
      nodeHash: hashCanonical({
        kind: input.kind,
        runId: input.runId,
        intentId: input.intentId,
        parentIds: input.parentIds,
        parentHashes: parents.map(p => p!.nodeHash),
        payloadHash: hashCanonical(input.payload),
      }),
      createdAt: new Date().toISOString(),
    };

    await this.store.append(node);
    return node;
  }
}
```

生产要点：

- 工具执行前先写 `tool_call`，执行后写 `tool_result`
- `tool_result.parentIds` 必须包含对应 `tool_call`
- 高风险工具要求 parent chain 包含 `policy_decision` 或 `approval`
- `nodeHash` 必须包含 parent hashes，防止换父节点
- append-only store，不允许原地修改旧节点

---

## 5. OpenClaw 实战：课程 Cron 的证据链

以这个课程 Cron 为例，一条完整证据链可以这样组织：

```text
ev_intent_cron_318
  └─ ev_plan_lesson_topic
      └─ ev_policy_git_message_allowed
          ├─ ev_file_write_lesson_318
          ├─ ev_readme_update
          ├─ ev_tools_update
          ├─ ev_telegram_send_receipt
          └─ ev_git_commit_push_receipt
              └─ ev_post_check_git_status_clean
```

完成闸门不应该只检查“文件存在、push 成功”，还应该检查：

```python
required_kinds = {
    "intent",
    "plan",
    "policy",
    "tool_result",
    "post_check",
}

ok, errors = EvidenceChainVerifier(nodes).verify("ev_post_check_git_status_clean")
if not ok:
    raise RuntimeError("证据链不完整: " + "; ".join(errors))

seen_kinds = {node.kind for node in nodes}
missing = required_kinds - seen_kinds
if missing:
    raise RuntimeError(f"缺少必要证据类型: {sorted(missing)}")
```

这样即使最终 `git push` 成功，也不能掩盖：

- 没有先检查 PR 状态
- 没有更新 README
- 没有发送 Telegram
- post-check 失败但被忽略

---

## 6. 常见坑

### 坑 1：只 hash payload，不 hash parent

如果 `nodeHash` 只包含当前 payload，攻击者或 bug 可以把一个真实 tool_result 挂到另一个 intent 下面。必须把 `parentIds + parentHashes` 一起签进去。

### 坑 2：证据链只做线性链

真实 Agent 经常 fan-out：并行跑测试、发消息、写文件、查询部署状态。线性 hash chain 会很别扭。更通用的方式是 DAG：一个节点可以有多个 parent，最终用 Merkle root 汇总。

### 坑 3：允许重写旧节点

证据链必须 append-only。修正错误时追加 `correction` / `tombstone` / `supersedes` 节点，而不是覆盖原证据。

### 坑 4：只在成功时写证据

失败更需要证据。`blocked`、`denied`、`tool_error`、`post_check_failed` 都应该入链，否则系统只会记住好消息。

---

## 7. 一句话总结

**Evidence Chain Verification 把 Agent 的执行记录从“散落日志”升级成“可复算的因果证明”。**

它不只是证明某个工具调用成功，而是证明：

- 这个动作来自哪个意图
- 经过了哪个计划和策略判断
- 实际执行了什么
- 外部世界给了什么回执
- 后置验证是否完成
- 中间有没有断链、换链、漏链

成熟 Agent 的审计能力，不是事后翻日志找线索，而是每次行动天然生成一条完整、可验证、不可悄悄改写的证据链。
