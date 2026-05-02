# 222. Agent 答案溯源与引用校验（Answer Grounding & Citation Guard）

> 关键词：grounding、citation、evidence、claim verification、RAG、hallucination guard

上一课讲了 **通知路由与噪声预算**：主动 Agent 不能有事就刷屏，要按优先级、免打扰、去重和摘要治理外部通知。

今天讲一个更贴近问答质量的问题：**Agent 说出的每个关键结论，能不能追到证据？**

很多 RAG / 搜索型 Agent 的代码长这样：

```txt
检索文档 → 把文档塞进 prompt → 让 LLM 回答
```

这只能叫“看过资料”，不能保证“答案有依据”。生产系统真正需要的是一层 **答案溯源与引用校验（Answer Grounding & Citation Guard）**：

> Agent 不仅要回答，还要把关键 claim 绑定到 evidence；发给用户前再校验 claim 是否被证据支持，不支持就降级、改写或追问。

一句话：**没有证据的结论，默认只是猜测。**

---

## 1. 核心思想

把最终回答拆成三类对象：

```txt
Source  原始证据：工具返回、网页、数据库行、文件片段
Claim   答案中的关键断言：数字、状态、原因、建议
Citation claim → source 的引用关系：这句话依据哪几条证据
```

然后在输出前跑一层 Guard：

```txt
tool results
  → normalize sources
  → generate answer with citations
  → extract claims
  → verify each claim against cited sources
  → pass / rewrite / ask for more data
```

这层 Guard 解决 4 个常见问题：

1. **编造来源**：答案引用了不存在的文档。
2. **张冠李戴**：证据 A 支持的是 X，答案却说成 Y。
3. **过度推断**：证据只说“可能”，答案说成“确定”。
4. **数字漂移**：工具返回 17%，答案写成 70%。

关键原则：**引用不是 UI 装饰，而是执行前的质量闸门。**

---

## 2. 反模式：把 citation 当 markdown 后缀

很多实现会让 LLM 自己在答案后面加 `[1][2]`：

```txt
根据资料，系统延迟下降 80% [1]。
```

但如果没有校验，`[1]` 只是一个“看起来专业”的符号。

更危险的是：

- 检索结果里确实有 `[1]`，但不支持这句话；
- LLM 把多个来源混在一起，生成了没有任何单一来源支持的新结论；
- 用户看到引用后反而更相信幻觉。

正确做法是：**先结构化 claim 和 evidence，再渲染成自然语言。**

---

## 3. learn-claude-code：Python 教学版

教学版用最小数据结构实现：每条证据有稳定 `source_id`，每个结论必须声明引用。

```python
# learn-claude-code/grounding_guard.py
from dataclasses import dataclass
from typing import Literal

Verdict = Literal["supported", "partial", "unsupported"]


@dataclass
class Source:
    id: str
    title: str
    text: str
    url: str | None = None
    tool: str | None = None


@dataclass
class Claim:
    id: str
    text: str
    source_ids: list[str]
    importance: Literal["critical", "normal"] = "normal"


@dataclass
class ClaimCheck:
    claim_id: str
    verdict: Verdict
    reason: str


class CitationGuard:
    def __init__(self, llm):
        self.llm = llm

    async def extract_claims(self, answer: str, sources: list[Source]) -> list[Claim]:
        """让模型把答案拆成可校验 claim；生产里要求 JSON schema 输出。"""
        source_list = "\n".join(f"[{s.id}] {s.title}" for s in sources)
        prompt = f"""
你是答案审计器。请把答案中的关键事实断言拆成 JSON 数组。
每个 claim 必须包含：id、text、source_ids、importance。
只能使用下面存在的 source id，不确定就 source_ids=[]。

Sources:
{source_list}

Answer:
{answer}
"""
        return await self.llm.json(prompt)

    async def verify_claim(self, claim: Claim, sources: dict[str, Source]) -> ClaimCheck:
        evidence = "\n\n".join(
            f"[{sid}] {sources[sid].text}"
            for sid in claim.source_ids
            if sid in sources
        )

        if not evidence.strip():
            return ClaimCheck(claim.id, "unsupported", "no cited evidence")

        prompt = f"""
判断 claim 是否被 evidence 支持，只能回答 JSON：
{{"verdict":"supported|partial|unsupported","reason":"..."}}

Claim: {claim.text}

Evidence:
{evidence}
"""
        result = await self.llm.json(prompt)
        return ClaimCheck(claim.id, result["verdict"], result["reason"])

    async def guard(self, answer: str, sources: list[Source]) -> tuple[bool, list[ClaimCheck]]:
        source_map = {s.id: s for s in sources}
        claims = await self.extract_claims(answer, sources)
        checks = [await self.verify_claim(c, source_map) for c in claims]

        # critical claim 不允许 unsupported；normal claim 可改写成“可能/未确认”
        ok = all(c.verdict != "unsupported" for c in checks)
        return ok, checks
```

Agent Loop 里不要直接把答案发出去，而是先过 Guard：

```python
async def answer_with_grounding(question: str, tools, llm):
    raw_results = await tools.search(question)
    sources = [
        Source(id=f"S{i+1}", title=r.title, text=r.snippet, url=r.url, tool="search")
        for i, r in enumerate(raw_results)
    ]

    answer = await llm.text(
        "请基于 sources 回答，并在每个关键事实后标注 [S1] 形式引用。\n"
        + render_sources(sources)
        + f"\nQuestion: {question}"
    )

    guard = CitationGuard(llm)
    ok, checks = await guard.guard(answer, sources)

    if ok:
        return answer

    # 不通过就让模型根据审计结果改写，删除无证据结论
    fixed = await llm.text(f"""
下面答案存在无证据 claim，请只保留被支持的内容，其他改成“不确定/需要更多数据”。

Answer:
{answer}

Checks:
{checks}
""")
    return fixed
```

教学版重点不是“多一个 LLM 调用”，而是建立一种习惯：**答案先审计，再交付。**

---

## 4. pi-mono：TypeScript 生产版

生产版建议把 Grounding 做成中间件，挂在 `responseProcessor` 或 `finalizeAnswer` 阶段。

### 4.1 标准数据结构

```ts
// pi-mono/agent/grounding/types.ts
export type SourceKind = 'tool_result' | 'document' | 'web' | 'memory' | 'user_input';
export type Verdict = 'supported' | 'partial' | 'unsupported';

export interface EvidenceSource {
  id: string;              // stable id: src_01
  kind: SourceKind;
  title: string;
  content: string;
  uri?: string;
  toolCallId?: string;
  createdAt: number;
  trust: number;           // 0-1，数据库行 > 网页摘要 > 用户转述
}

export interface GroundedClaim {
  id: string;
  text: string;
  citationIds: string[];
  risk: 'low' | 'medium' | 'high';
}

export interface VerificationResult {
  claimId: string;
  verdict: Verdict;
  confidence: number;
  reason: string;
}
```

### 4.2 工具结果进入 Evidence Store

不要等最终回答才找来源。每次工具调用完成时就把结果登记成 evidence：

```ts
// pi-mono/agent/grounding/evidence-store.ts
export class EvidenceStore {
  private sources = new Map<string, EvidenceSource>();
  private seq = 0;

  add(input: Omit<EvidenceSource, 'id' | 'createdAt'>): EvidenceSource {
    const id = `src_${String(++this.seq).padStart(2, '0')}`;
    const source: EvidenceSource = { ...input, id, createdAt: Date.now() };
    this.sources.set(id, source);
    return source;
  }

  list(): EvidenceSource[] {
    return [...this.sources.values()];
  }

  get(id: string): EvidenceSource | undefined {
    return this.sources.get(id);
  }

  renderForPrompt(maxChars = 12_000): string {
    let used = 0;
    const chunks: string[] = [];

    for (const s of this.list()) {
      const block = `<source id="${s.id}" kind="${s.kind}" trust="${s.trust}">\n${s.content}\n</source>`;
      if (used + block.length > maxChars) break;
      chunks.push(block);
      used += block.length;
    }

    return chunks.join('\n\n');
  }
}
```

工具中间件自动登记：

```ts
// pi-mono/agent/middleware/evidence-middleware.ts
export function evidenceMiddleware(store: EvidenceStore): ToolMiddleware {
  return async (call, next) => {
    const result = await next(call);

    if (result.ok) {
      store.add({
        kind: 'tool_result',
        title: `${call.toolName} result`,
        content: JSON.stringify(result.data).slice(0, 4_000),
        toolCallId: call.id,
        trust: call.toolName.startsWith('db.') ? 0.95 : 0.75,
      });
    }

    return result;
  };
}
```

这样最终回答阶段不用猜“证据从哪来”，Evidence Store 已经把工具轨迹整理好了。

### 4.3 Citation Guard 中间件

```ts
// pi-mono/agent/grounding/citation-guard.ts
export class CitationGuard {
  constructor(private model: ModelClient) {}

  async extractClaims(answer: string, sources: EvidenceSource[]): Promise<GroundedClaim[]> {
    return this.model.generateJson<GroundedClaim[]>({
      schema: GroundedClaimSchema.array(),
      temperature: 0,
      messages: [{
        role: 'user',
        content: `Extract factual claims from the answer. Use only existing source ids.\n\nSources:\n${sources.map(s => `${s.id}: ${s.title}`).join('\n')}\n\nAnswer:\n${answer}`,
      }],
    });
  }

  async verify(claim: GroundedClaim, sources: EvidenceSource[]): Promise<VerificationResult> {
    const cited = claim.citationIds
      .map(id => sources.find(s => s.id === id))
      .filter(Boolean) as EvidenceSource[];

    if (cited.length === 0) {
      return {
        claimId: claim.id,
        verdict: 'unsupported',
        confidence: 1,
        reason: 'claim has no valid citation',
      };
    }

    return this.model.generateJson<VerificationResult>({
      schema: VerificationResultSchema,
      temperature: 0,
      messages: [{
        role: 'user',
        content: `Verify whether the claim is supported by evidence.\n\nClaim:\n${claim.text}\n\nEvidence:\n${cited.map(s => `<${s.id}>${s.content}</${s.id}>`).join('\n')}`,
      }],
    });
  }

  async check(answer: string, sources: EvidenceSource[]) {
    const claims = await this.extractClaims(answer, sources);
    const results = await Promise.all(claims.map(c => this.verify(c, sources)));

    const blocking = results.filter(r => r.verdict === 'unsupported');
    return { pass: blocking.length === 0, claims, results, blocking };
  }
}
```

接到最终回答时：

```ts
// pi-mono/agent/response/finalize-answer.ts
export async function finalizeAnswer(ctx: AgentContext, draft: string) {
  const sources = ctx.evidenceStore.list();
  const guard = new CitationGuard(ctx.models.smallReliable);
  const report = await guard.check(draft, sources);

  ctx.telemetry.emit('answer.grounding_checked', {
    claims: report.claims.length,
    unsupported: report.blocking.length,
  });

  if (report.pass) return draft;

  return ctx.models.primary.generateText({
    temperature: 0,
    messages: [{
      role: 'user',
      content: `Rewrite the answer. Remove unsupported claims or mark them as uncertain.\n\nDraft:\n${draft}\n\nVerification report:\n${JSON.stringify(report.results, null, 2)}`,
    }],
  });
}
```

生产建议：

- `extractClaims` 和 `verify` 用低温小模型，成本低、稳定；
- 高风险领域（财务、医疗、生产事故）`unsupported` 直接阻断；
- 普通聊天可以只对数字、时间、状态、结论类 claim 抽检；
- citation id 必须来自 Evidence Store，禁止模型自造。

---

## 5. OpenClaw 实战：工具证据比“记忆印象”优先

OpenClaw 里这个模式非常实用：

```txt
用户问“PR CI 为什么失败？”
  1. 用 github skill / gh 查 PR 状态
  2. 拉取 CI logs 作为 evidence
  3. 回答每个失败原因时引用 job/log 片段
  4. 如果日志没找到，明确说“我还没拿到证据”
```

可以把证据缓存成任务文件：

```json
{
  "taskId": "debug-pr-123",
  "sources": [
    {
      "id": "src_01",
      "kind": "tool_result",
      "title": "gh run view 987 --log-failed",
      "content": "Error: Cannot find module 'sharp'",
      "trust": 0.95
    }
  ],
  "claims": [
    {
      "text": "CI 失败原因是缺少 sharp 模块",
      "citationIds": ["src_01"]
    }
  ]
}
```

最终发消息时可以这样组织：

```txt
CI 失败原因是缺少 sharp 模块（证据：src_01）。
建议先检查 lockfile 是否包含 sharp 的平台包；如果没有，重新安装依赖并提交 lockfile。
```

注意：OpenClaw 的 `read`、`exec`、`web_fetch`、`sessions_history` 这些工具结果都可以当 Evidence Source。关键不是工具多强，而是 Agent 是否愿意承认：**没查到就不能装作查到了。**

---

## 6. Citation Guard 的三档策略

不是所有场景都要全量校验。可以按风险分级：

| 场景 | 策略 | 示例 |
|---|---|---|
| low | 不强制 citation，只记录来源 | 日常闲聊、创意写作 |
| medium | 关键事实必须 citation，unsupported 自动改写 | 技术问答、项目总结 |
| high | 每个关键 claim 必须验证，不通过就阻断 | 财务、安全、线上事故、法律医疗 |

伪代码：

```ts
function groundingPolicy(task: Task): GroundingPolicy {
  if (task.domain in ['billing', 'security', 'incident']) {
    return { mode: 'strict', blockUnsupported: true, verifyAllClaims: true };
  }

  if (task.requiresExternalFacts) {
    return { mode: 'balanced', blockUnsupported: false, verifyImportantClaims: true };
  }

  return { mode: 'light', auditOnly: true };
}
```

这样既不会让所有回答都变慢，也能在关键任务上守住质量底线。

---

## 7. 常见坑

### 坑 1：引用 chunk 太碎

RAG chunk 如果切得太碎，单个 source 不足以支持完整 claim。解决：保留 `parent_id` 和上下文窗口。

```ts
interface EvidenceSource {
  id: string;
  parentId?: string;
  sectionTitle?: string;
  before?: string;
  content: string;
  after?: string;
}
```

### 坑 2：只验证“有没有引用”，不验证“支不支持”

`citationIds.length > 0` 不等于 supported。必须做语义校验。

### 坑 3：把用户输入当绝对事实

用户说“服务器挂了”是一个信号，不是最终事实。可以引用为 `user_input`，但 trust 不能和监控/API 证据一样高。

### 坑 4：Guard 自己也幻觉

Guard 要低温、结构化输出、只允许三档 verdict；高风险时可以双模型交叉验证或规则优先。

---

## 8. 最小落地 Checklist

给现有 Agent 加 Grounding，不需要大改架构，先做 5 步：

1. 所有工具结果登记 `source_id`。
2. 最终回答 prompt 要求关键事实标注 `[source_id]`。
3. 输出前抽取 claim。
4. 校验 claim 是否被引用 source 支持。
5. 不支持的 claim 删除、降级或追问。

最小系统也可以只从这条规则开始：

> 凡是数字、时间、状态、原因、结论，都必须有证据。

---

## 9. 总结

Agent 的可信度不是靠“语气自信”建立的，而是靠证据链建立的。

**Answer Grounding & Citation Guard** 的价值：

- 把“看过资料”升级为“结论可追溯”；
- 把 citation 从装饰变成质量闸门；
- 把幻觉从用户发现，前移到系统自检；
- 高风险场景中，宁可说“不确定”，也不要自信地编。

记住一句话：

> 好 Agent 不只是会回答；好 Agent 知道自己每句话凭什么说。
