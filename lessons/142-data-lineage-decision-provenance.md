# 142. Agent 数据血缘追踪与决策溯源（Data Lineage & Decision Provenance）

> **一句话**：给每一条数据打上"出生证明"，记录它从哪来、经历了什么、最终影响了哪个决策——让 Agent 的行为从黑盒变成玻璃盒。

---

## 为什么需要数据血缘？

你的 Agent 调用了 3 个工具、拼接了 2 段历史对话，最终给出了一个建议。然后用户问：

> "你为什么这么说？"

你能答出来吗？

**审计日志**（Lesson 115）告诉你"做了什么"。  
**数据血缘**（Data Lineage）告诉你"**为什么这么做**"——每个决策背后的数据来源和推导链。

| 能力 | 审计日志 | 数据血缘 |
|------|---------|---------|
| 记录什么时候调用了哪个工具 | ✅ | ✅ |
| 记录工具结果如何影响决策 | ❌ | ✅ |
| 追溯"这个结论的数据来源" | ❌ | ✅ |
| 合规可解释性报告 | 部分 | ✅ |
| 调试"AI 为什么说错了" | 难 | 简单 |

---

## 核心概念：数据节点图

数据血缘本质上是一个**有向无环图（DAG）**：

```
用户输入 ──→ [工具A: 查订单] ──→ 订单数据
                                    │
用户历史 ──→ [工具B: 查偏好] ──→ 偏好数据
                                    │
                                    ▼
                             [LLM 推理节点] ──→ 最终回答
```

每个节点包含：
- **lineageId**：唯一标识
- **source**：数据来源（工具名、用户输入、历史记录）
- **content**：实际数据（可脱敏）
- **parentIds**：依赖的上游节点
- **timestamp**：创建时间
- **confidence**：可信度评分

---

## TypeScript 实现

### 1. 血缘节点定义

```typescript
// lineage/types.ts
export type LineageNodeKind = 
  | 'user_input'    // 用户原始输入
  | 'tool_call'     // 工具调用
  | 'tool_result'   // 工具返回结果
  | 'llm_inference' // LLM 推理步骤
  | 'final_output'  // 最终输出

export interface LineageNode {
  id: string;                    // 唯一 ID
  kind: LineageNodeKind;
  label: string;                 // 人读描述，如 "查询用户订单"
  parentIds: string[];           // 依赖哪些上游节点
  data: {
    raw?: unknown;               // 原始数据
    summary: string;             // 简洁摘要（给人看的）
    redacted?: boolean;          // 是否已脱敏
  };
  meta: {
    toolName?: string;
    modelId?: string;
    confidence?: number;         // 0-1
    durationMs?: number;
    traceId?: string;
  };
  createdAt: number;
}

export interface LineageGraph {
  sessionId: string;
  turnId: string;               // 对话轮次
  nodes: Map<string, LineageNode>;
  rootIds: string[];            // 入口节点（用户输入）
  outputIds: string[];          // 出口节点（最终回答）
}
```

### 2. 血缘追踪器

```typescript
// lineage/tracker.ts
import { randomUUID } from 'crypto';
import { LineageNode, LineageGraph, LineageNodeKind } from './types';

export class LineageTracker {
  private graph: LineageGraph;

  constructor(sessionId: string, turnId: string) {
    this.graph = {
      sessionId,
      turnId,
      nodes: new Map(),
      rootIds: [],
      outputIds: [],
    };
  }

  /** 记录用户输入节点 */
  trackInput(content: string): string {
    const id = randomUUID();
    this.addNode({
      id,
      kind: 'user_input',
      label: '用户输入',
      parentIds: [],
      data: { raw: content, summary: content.slice(0, 100) },
      meta: {},
      createdAt: Date.now(),
    });
    this.graph.rootIds.push(id);
    return id;
  }

  /** 记录工具调用节点 */
  trackToolCall(
    toolName: string,
    params: unknown,
    parentIds: string[]
  ): string {
    const id = randomUUID();
    this.addNode({
      id,
      kind: 'tool_call',
      label: `调用工具: ${toolName}`,
      parentIds,
      data: {
        raw: params,
        summary: `${toolName}(${JSON.stringify(params).slice(0, 80)})`,
      },
      meta: { toolName },
      createdAt: Date.now(),
    });
    return id;
  }

  /** 记录工具结果节点 */
  trackToolResult(
    toolName: string,
    result: unknown,
    parentId: string,
    options: { durationMs?: number; redact?: boolean } = {}
  ): string {
    const id = randomUUID();
    const raw = options.redact ? '[REDACTED]' : result;
    const summary = typeof result === 'object'
      ? JSON.stringify(result).slice(0, 150)
      : String(result).slice(0, 150);

    this.addNode({
      id,
      kind: 'tool_result',
      label: `${toolName} 返回结果`,
      parentIds: [parentId],
      data: { raw, summary, redacted: options.redact },
      meta: { toolName, durationMs: options.durationMs },
      createdAt: Date.now(),
    });
    return id;
  }

  /** 记录 LLM 推理节点 */
  trackInference(
    summary: string,
    parentIds: string[],
    options: { modelId?: string; confidence?: number } = {}
  ): string {
    const id = randomUUID();
    this.addNode({
      id,
      kind: 'llm_inference',
      label: 'LLM 推理',
      parentIds,
      data: { summary },
      meta: { modelId: options.modelId, confidence: options.confidence },
      createdAt: Date.now(),
    });
    return id;
  }

  /** 记录最终输出节点 */
  trackOutput(content: string, parentIds: string[]): string {
    const id = randomUUID();
    this.addNode({
      id,
      kind: 'final_output',
      label: '最终回答',
      parentIds,
      data: { raw: content, summary: content.slice(0, 200) },
      meta: {},
      createdAt: Date.now(),
    });
    this.graph.outputIds.push(id);
    return id;
  }

  /** 查询某节点的完整祖先链（向上溯源） */
  getAncestors(nodeId: string): LineageNode[] {
    const result: LineageNode[] = [];
    const visited = new Set<string>();

    const dfs = (id: string) => {
      if (visited.has(id)) return;
      visited.add(id);
      const node = this.graph.nodes.get(id);
      if (!node) return;
      result.push(node);
      for (const parentId of node.parentIds) {
        dfs(parentId);
      }
    };

    dfs(nodeId);
    return result.sort((a, b) => a.createdAt - b.createdAt);
  }

  /** 生成人读溯源报告 */
  explainOutput(outputId?: string): string {
    const targetId = outputId ?? this.graph.outputIds[0];
    if (!targetId) return '暂无输出节点';

    const chain = this.getAncestors(targetId);
    const lines = ['## 决策溯源报告\n'];

    for (const node of chain) {
      const icon = {
        user_input: '💬',
        tool_call: '🔧',
        tool_result: '📦',
        llm_inference: '🧠',
        final_output: '✅',
      }[node.kind];

      lines.push(
        `${icon} **${node.label}**`,
        `   > ${node.data.summary}`,
        ''
      );
    }

    return lines.join('\n');
  }

  getGraph(): LineageGraph {
    return this.graph;
  }

  private addNode(node: LineageNode) {
    this.graph.nodes.set(node.id, node);
  }
}
```

### 3. 与 Agent Loop 集成

```typescript
// agent/loop-with-lineage.ts
import { LineageTracker } from '../lineage/tracker';
import Anthropic from '@anthropic-ai/sdk';

const client = new Anthropic();

async function runAgentTurn(
  userMessage: string,
  tools: Anthropic.Tool[],
  sessionId: string
) {
  const turnId = Date.now().toString();
  const tracker = new LineageTracker(sessionId, turnId);

  // 1. 追踪用户输入
  const inputNodeId = tracker.trackInput(userMessage);

  const messages: Anthropic.MessageParam[] = [
    { role: 'user', content: userMessage }
  ];

  let finalResponse = '';
  const activeToolCallIds: string[] = [inputNodeId];

  // 2. Agent Loop
  while (true) {
    const response = await client.messages.create({
      model: 'claude-opus-4-5',
      max_tokens: 4096,
      tools,
      messages,
    });

    if (response.stop_reason === 'end_turn') {
      // 最终文本输出
      const text = response.content
        .filter((b): b is Anthropic.TextBlock => b.type === 'text')
        .map(b => b.text)
        .join('');

      // 追踪推理节点
      const inferenceId = tracker.trackInference(
        text.slice(0, 200),
        activeToolCallIds,
        { modelId: response.model }
      );

      // 追踪最终输出
      tracker.trackOutput(text, [inferenceId]);
      finalResponse = text;
      break;
    }

    if (response.stop_reason === 'tool_use') {
      const toolResults: Anthropic.MessageParam = {
        role: 'user',
        content: [],
      };

      const newActiveIds: string[] = [];

      for (const block of response.content) {
        if (block.type !== 'tool_use') continue;

        // 追踪工具调用
        const callId = tracker.trackToolCall(
          block.name,
          block.input,
          activeToolCallIds
        );

        const start = Date.now();
        
        // 实际执行工具（简化示例）
        const result = await executeTool(block.name, block.input as Record<string, unknown>);
        const durationMs = Date.now() - start;

        // 追踪工具结果
        const resultId = tracker.trackToolResult(
          block.name,
          result,
          callId,
          { durationMs }
        );

        newActiveIds.push(resultId);

        (toolResults.content as Anthropic.ToolResultBlockParam[]).push({
          type: 'tool_result',
          tool_use_id: block.id,
          content: JSON.stringify(result),
        });
      }

      activeToolCallIds.length = 0;
      activeToolCallIds.push(...newActiveIds);

      messages.push({ role: 'assistant', content: response.content });
      messages.push(toolResults);
    }
  }

  // 3. 生成溯源报告
  const report = tracker.explainOutput();
  console.log(report);

  return { response: finalResponse, lineage: tracker.getGraph() };
}

// 工具执行（示例）
async function executeTool(name: string, params: Record<string, unknown>): Promise<unknown> {
  // 实际工具实现
  return { result: `${name} executed`, params };
}
```

### 4. OpenClaw 中的实战用法

在 OpenClaw Skill 里，把血缘追踪器注入工具中间件：

```typescript
// skills/my-skill/lineage-middleware.ts
import { LineageTracker } from './tracker';

/**
 * 工具中间件：自动给所有工具调用打血缘标签
 */
export function withLineage(
  tracker: LineageTracker,
  parentIds: string[]
) {
  return function wrapTool(
    toolFn: (params: unknown) => Promise<unknown>,
    toolName: string
  ) {
    return async (params: unknown): Promise<unknown> => {
      const callId = tracker.trackToolCall(toolName, params, parentIds);
      const start = Date.now();

      try {
        const result = await toolFn(params);
        tracker.trackToolResult(toolName, result, callId, {
          durationMs: Date.now() - start,
        });
        return result;
      } catch (err) {
        tracker.trackToolResult(toolName, { error: String(err) }, callId, {
          durationMs: Date.now() - start,
        });
        throw err;
      }
    };
  };
}
```

---

## 持久化血缘图（Redis）

单次对话的血缘存内存够用，跨会话分析需要持久化：

```typescript
// lineage/storage.ts
import { Redis } from 'ioredis';
import { LineageGraph } from './types';

export class LineageStorage {
  constructor(private redis: Redis, private ttlSeconds = 86400 * 30) {}

  async save(graph: LineageGraph): Promise<void> {
    const key = `lineage:${graph.sessionId}:${graph.turnId}`;
    // 把 Map 转成普通对象再序列化
    const serialized = JSON.stringify({
      ...graph,
      nodes: Object.fromEntries(graph.nodes),
    });
    await this.redis.setex(key, this.ttlSeconds, serialized);

    // 维护会话级别的索引
    await this.redis.zadd(
      `lineage:index:${graph.sessionId}`,
      Date.now(),
      graph.turnId
    );
  }

  async loadTurn(sessionId: string, turnId: string): Promise<LineageGraph | null> {
    const key = `lineage:${sessionId}:${turnId}`;
    const raw = await this.redis.get(key);
    if (!raw) return null;

    const parsed = JSON.parse(raw);
    return {
      ...parsed,
      nodes: new Map(Object.entries(parsed.nodes)),
    };
  }

  /** 查询一个会话的所有轮次血缘 */
  async loadSession(sessionId: string): Promise<LineageGraph[]> {
    const turnIds = await this.redis.zrange(
      `lineage:index:${sessionId}`,
      0, -1
    );
    const graphs = await Promise.all(
      turnIds.map(tid => this.loadTurn(sessionId, tid))
    );
    return graphs.filter(Boolean) as LineageGraph[];
  }

  /** 查找哪些决策用到了某个特定工具的结果 */
  async findDecisionsUsingTool(
    sessionId: string,
    toolName: string
  ): Promise<string[]> {
    const graphs = await this.loadSession(sessionId);
    const results: string[] = [];

    for (const graph of graphs) {
      for (const [, node] of graph.nodes) {
        if (node.kind === 'tool_result' && node.meta.toolName === toolName) {
          // 找这个节点的所有下游输出节点
          const outputs = this.findDownstreamOutputs(graph, node.id);
          results.push(...outputs.map(n => n.data.summary));
        }
      }
    }

    return results;
  }

  private findDownstreamOutputs(graph: LineageGraph, nodeId: string) {
    // 构建反向图，从当前节点找所有下游节点
    const children = new Map<string, string[]>();
    for (const [id, node] of graph.nodes) {
      for (const parentId of node.parentIds) {
        if (!children.has(parentId)) children.set(parentId, []);
        children.get(parentId)!.push(id);
      }
    }

    const outputs: typeof graph.nodes extends Map<string, infer V> ? V[] : never[] = [];
    const dfs = (id: string) => {
      const node = graph.nodes.get(id);
      if (!node) return;
      if (node.kind === 'final_output') outputs.push(node as any);
      for (const childId of children.get(id) ?? []) dfs(childId);
    };
    dfs(nodeId);
    return outputs;
  }
}
```

---

## 可视化：用 Mermaid 生成血缘图

```typescript
// lineage/visualize.ts
import { LineageGraph } from './types';

export function toMermaid(graph: LineageGraph): string {
  const lines = ['graph TD'];
  const iconMap = {
    user_input: '💬',
    tool_call: '🔧',
    tool_result: '📦',
    llm_inference: '🧠',
    final_output: '✅',
  };

  for (const [id, node] of graph.nodes) {
    const shortId = id.slice(0, 8);
    const icon = iconMap[node.kind];
    const label = `${icon} ${node.label}`;
    lines.push(`  ${shortId}["${label}"]`);
  }

  for (const [id, node] of graph.nodes) {
    for (const parentId of node.parentIds) {
      lines.push(`  ${parentId.slice(0, 8)} --> ${id.slice(0, 8)}`);
    }
  }

  return lines.join('\n');
}

// 使用
// const diagram = toMermaid(tracker.getGraph());
// 粘贴到 https://mermaid.live 即可渲染
```

---

## 合规场景：生成可解释性报告

```typescript
// lineage/compliance.ts
export function generateComplianceReport(graph: LineageGraph): object {
  const nodes = [...graph.nodes.values()];

  const toolsUsed = nodes
    .filter(n => n.kind === 'tool_call')
    .map(n => ({
      tool: n.meta.toolName,
      params: n.data.redacted ? '[REDACTED]' : n.data.summary,
      timestamp: new Date(n.createdAt).toISOString(),
    }));

  const dataSources = nodes
    .filter(n => n.kind === 'tool_result')
    .map(n => ({
      source: n.meta.toolName,
      dataPreview: n.data.summary,
      redacted: n.data.redacted ?? false,
    }));

  const outputs = nodes
    .filter(n => n.kind === 'final_output')
    .map(n => n.data.summary);

  return {
    sessionId: graph.sessionId,
    turnId: graph.turnId,
    summary: {
      toolsInvoked: toolsUsed.length,
      dataSourcesConsumed: dataSources.length,
      outputsProduced: outputs.length,
    },
    toolsUsed,
    dataSources,
    outputs,
    generatedAt: new Date().toISOString(),
  };
}
```

---

## 与已学模式的关系

| 模式 | 关系 |
|------|------|
| 审计日志（Lesson 115） | 血缘是审计的升级，增加因果关系维度 |
| OpenTelemetry（Lesson 88） | OTel 追踪时序，血缘追踪数据依赖 |
| 工具结果冲突仲裁（Lesson 139） | 血缘帮助定位冲突数据的来源 |
| 增量 Diff（Lesson 140） | Diff 结果作为血缘节点，追踪哪些变化触发了决策 |
| RAG（Lesson 48） | 检索到的文档块作为血缘节点，追踪回答基于哪些资料 |

---

## 小结

| 问题 | 数据血缘的答案 |
|------|--------------|
| "AI 为什么这么说？" | 溯源链展示全部依赖数据 |
| "这个结论基于什么数据？" | 追溯到具体工具结果节点 |
| "某个工具的数据影响了哪些决策？" | 正向遍历 DAG |
| "这次对话合规吗？" | 自动生成合规报告 |
| "生产 bug 怎么复现？" | 用血缘图重建决策上下文 |

**数据血缘 = 每个决策的出生证明。** 不只是"做了什么"，而是"用什么数据做了什么"。
