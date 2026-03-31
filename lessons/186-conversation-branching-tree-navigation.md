# Agent 会话分支与对话树导航（Conversation Branching & Tree Navigation）

> 第 186 课 - 让用户能"回到那个岔路口"重新选择，而不是从头开始

## 🎯 核心问题

用户和 Agent 对话时经常遇到：
- "刚才那个方案不行，我想试试你说的另一个"
- "回到 3 步之前，我想换个方向"
- "如果当时选了 A 而不是 B 会怎样？"

传统 Agent 只有线性历史——想回头只能重发一遍之前的消息。分支机制让用户能在决策点"存档"，随时返回重新探索。

## 🏗️ 核心概念

```
线性历史（传统）:
[User1] → [Agent1] → [User2] → [Agent2] → [User3] → [Agent3]
                                              ↑
                                          只能从这继续

分支历史（本课）:
                                   [User3a] → [Agent3a] (当前)
                                  ↗
[User1] → [Agent1] → [User2] → [Agent2] ← 分支点 (checkpoint)
                                  ↘
                                   [User3b] → [Agent3b] (另一分支)
                                         ↘
                                          [User4b] → ...
```

## 💡 对话树数据结构

```typescript
// 对话节点
interface ConversationNode {
  id: string;
  parentId: string | null;      // null = 根节点
  role: 'user' | 'assistant' | 'system';
  content: string;
  toolCalls?: ToolCall[];
  timestamp: number;
  
  // 分支相关
  branchLabel?: string;         // 用户给分支起的名字
  isCheckpoint: boolean;        // 是否是保存点
  metadata?: {
    intent?: string;            // 该节点的意图标签
    confidence?: number;
    branchReason?: string;      // 为什么创建分支
  };
}

// 对话树
class ConversationTree {
  private nodes: Map<string, ConversationNode> = new Map();
  private currentNodeId: string | null = null;
  
  // 获取从根到当前节点的路径（线性化）
  getCurrentPath(): ConversationNode[] {
    const path: ConversationNode[] = [];
    let nodeId = this.currentNodeId;
    
    while (nodeId) {
      const node = this.nodes.get(nodeId);
      if (!node) break;
      path.unshift(node); // 头部插入
      nodeId = node.parentId;
    }
    
    return path;
  }
  
  // 追加消息（在当前节点下创建子节点）
  append(message: Omit<ConversationNode, 'id' | 'parentId' | 'timestamp'>): string {
    const id = crypto.randomUUID();
    const node: ConversationNode = {
      ...message,
      id,
      parentId: this.currentNodeId,
      timestamp: Date.now(),
      isCheckpoint: false,
    };
    
    this.nodes.set(id, node);
    this.currentNodeId = id;
    return id;
  }
  
  // 创建分支（在指定节点下创建新子节点，不改变当前位置）
  branch(
    fromNodeId: string, 
    message: Omit<ConversationNode, 'id' | 'parentId' | 'timestamp'>,
    branchLabel?: string
  ): string {
    const id = crypto.randomUUID();
    const node: ConversationNode = {
      ...message,
      id,
      parentId: fromNodeId,
      timestamp: Date.now(),
      branchLabel,
      isCheckpoint: false,
    };
    
    this.nodes.set(id, node);
    return id;
  }
  
  // 切换到指定节点
  navigate(nodeId: string): boolean {
    if (!this.nodes.has(nodeId)) return false;
    this.currentNodeId = nodeId;
    return true;
  }
  
  // 获取节点的所有子节点（分支）
  getChildren(nodeId: string): ConversationNode[] {
    return Array.from(this.nodes.values())
      .filter(n => n.parentId === nodeId);
  }
  
  // 标记当前节点为检查点
  markCheckpoint(label?: string): void {
    const node = this.nodes.get(this.currentNodeId!);
    if (node) {
      node.isCheckpoint = true;
      node.branchLabel = label;
    }
  }
  
  // 获取所有检查点
  getCheckpoints(): ConversationNode[] {
    return Array.from(this.nodes.values())
      .filter(n => n.isCheckpoint)
      .sort((a, b) => a.timestamp - b.timestamp);
  }
  
  // 回退 N 步
  goBack(steps: number): string | null {
    let nodeId = this.currentNodeId;
    for (let i = 0; i < steps && nodeId; i++) {
      const node = this.nodes.get(nodeId);
      nodeId = node?.parentId ?? null;
    }
    if (nodeId) {
      this.currentNodeId = nodeId;
    }
    return nodeId;
  }
}
```

## 🔍 分支点自动检测

并非每条消息都需要记为分支点——只在**决策节点**自动创建检查点：

```typescript
interface BranchDetector {
  shouldCheckpoint(
    userMessage: string,
    agentResponse: string,
    toolResults?: ToolResult[]
  ): { checkpoint: boolean; reason?: string };
}

// 基于规则的检测器
const ruleBranchDetector: BranchDetector = {
  shouldCheckpoint(userMessage, agentResponse, toolResults) {
    // 1. Agent 提供了多个选项
    const hasOptions = /选项|方案|可以.*也可以|或者|alternatively/i.test(agentResponse);
    if (hasOptions) {
      return { checkpoint: true, reason: 'multiple_options_presented' };
    }
    
    // 2. 用户做出了明确选择
    const madeChoice = /选.*方案|用.*方式|就.*吧|go with|choose|pick/i.test(userMessage);
    if (madeChoice) {
      return { checkpoint: true, reason: 'user_made_choice' };
    }
    
    // 3. 执行了有副作用的工具
    const hasDestructiveAction = toolResults?.some(r => 
      ['file_write', 'db_update', 'api_post', 'send_email'].includes(r.toolName)
    );
    if (hasDestructiveAction) {
      return { checkpoint: true, reason: 'destructive_action_taken' };
    }
    
    // 4. Agent 表达了不确定性
    const hasUncertainty = /不确定|可能|也许|建议.*但/i.test(agentResponse);
    if (hasUncertainty) {
      return { checkpoint: true, reason: 'agent_uncertainty' };
    }
    
    return { checkpoint: false };
  }
};

// LLM 辅助检测器（更准确但成本更高）
async function llmBranchDetector(
  userMessage: string,
  agentResponse: string,
  context: ConversationNode[]
): Promise<{ checkpoint: boolean; reason?: string }> {
  const result = await llm.call({
    model: 'haiku', // 用小模型降成本
    messages: [{
      role: 'user',
      content: `判断这轮对话是否是一个"决策点"（用户需要在多个方案中选择，或做出重要决定）。

用户: ${userMessage}
Agent: ${agentResponse.slice(0, 500)}

回答 JSON: {"checkpoint": boolean, "reason": "简短原因"}`
    }],
    response_format: { type: 'json_object' }
  });
  
  return JSON.parse(result.content);
}
```

## 🧭 导航命令处理

让用户能自然语言导航对话树：

```typescript
interface NavigationIntent {
  type: 'go_back' | 'go_to_checkpoint' | 'list_branches' | 'switch_branch' | 'create_branch';
  steps?: number;
  checkpointLabel?: string;
  branchId?: string;
}

function parseNavigationIntent(userMessage: string): NavigationIntent | null {
  // "回到 3 步之前" / "go back 3 steps"
  const goBackMatch = userMessage.match(/回到?\s*(\d+)\s*步/i) 
    || userMessage.match(/go\s*back\s*(\d+)/i);
  if (goBackMatch) {
    return { type: 'go_back', steps: parseInt(goBackMatch[1]) };
  }
  
  // "回到选方案之前" / "回到 checkpoint X"
  const checkpointMatch = userMessage.match(/回到\s*[「"']?(.+?)[」"']?\s*(之前|那里)/i);
  if (checkpointMatch) {
    return { type: 'go_to_checkpoint', checkpointLabel: checkpointMatch[1] };
  }
  
  // "列出所有分支"
  if (/列出.*分支|show branches|list checkpoints/i.test(userMessage)) {
    return { type: 'list_branches' };
  }
  
  // "切换到分支 A"
  const switchMatch = userMessage.match(/切换到\s*[「"']?(.+?)[」"']?分支/i);
  if (switchMatch) {
    return { type: 'switch_branch', branchId: switchMatch[1] };
  }
  
  // "在这里创建一个分支"
  if (/创建.*分支|fork|save\s*point/i.test(userMessage)) {
    return { type: 'create_branch' };
  }
  
  return null;
}

// 处理导航命令
async function handleNavigation(
  tree: ConversationTree,
  intent: NavigationIntent
): Promise<string> {
  switch (intent.type) {
    case 'go_back':
      const targetId = tree.goBack(intent.steps!);
      if (!targetId) return '无法回退那么多步';
      return `已回到 ${intent.steps} 步之前。你可以从这里重新开始，之前的对话会作为另一个分支保留。`;
    
    case 'go_to_checkpoint':
      const checkpoints = tree.getCheckpoints();
      const target = checkpoints.find(c => 
        c.branchLabel?.includes(intent.checkpointLabel!) ||
        c.content?.includes(intent.checkpointLabel!)
      );
      if (!target) {
        return `找不到标记为 "${intent.checkpointLabel}" 的检查点。现有检查点：\n` +
          checkpoints.map((c, i) => `${i+1}. ${c.branchLabel || c.content.slice(0, 50)}`).join('\n');
      }
      tree.navigate(target.id);
      return `已回到检查点 "${target.branchLabel || '未命名'}"`;
    
    case 'list_branches':
      const allCheckpoints = tree.getCheckpoints();
      if (allCheckpoints.length === 0) {
        return '当前没有保存的分支点。你可以说"在这里创建一个分支"来保存当前位置。';
      }
      return '📍 保存的分支点：\n' + 
        allCheckpoints.map((c, i) => {
          const children = tree.getChildren(c.id);
          return `${i+1}. ${c.branchLabel || `第${i+1}个决策点`} (${children.length} 个分支)`;
        }).join('\n');
    
    case 'create_branch':
      tree.markCheckpoint(`分支点 ${Date.now()}`);
      return '✅ 已在当前位置创建分支点。你可以继续对话，之后随时回到这里尝试其他方向。';
    
    default:
      return '未知的导航命令';
  }
}
```

## 📦 分支间上下文共享策略

不同分支可能需要共享/隔离不同类型的上下文：

```typescript
interface BranchContext {
  // 共享：跨所有分支的全局状态
  shared: {
    userPreferences: Record<string, any>;  // 用户偏好
    learnedFacts: string[];                 // 学到的事实
    toolCredentials: Record<string, string>; // 工具凭证
  };
  
  // 隔离：每个分支独立的状态
  isolated: {
    workingFiles: string[];        // 当前处理的文件
    pendingActions: Action[];      // 待执行的操作
    scratchpad: string;            // 临时笔记
  };
}

// 分支时复制隔离上下文，引用共享上下文
function forkContext(
  parentContext: BranchContext
): BranchContext {
  return {
    shared: parentContext.shared, // 引用共享
    isolated: {
      // 深拷贝隔离
      workingFiles: [...parentContext.isolated.workingFiles],
      pendingActions: parentContext.isolated.pendingActions.map(a => ({...a})),
      scratchpad: parentContext.isolated.scratchpad,
    }
  };
}

// 分支合并：当用户想把一个分支的结果带回主线
function mergeContext(
  target: BranchContext,
  source: BranchContext,
  strategy: 'overwrite' | 'union' | 'ask'
): BranchContext {
  if (strategy === 'union') {
    return {
      shared: target.shared, // 共享保持不变
      isolated: {
        workingFiles: [...new Set([
          ...target.isolated.workingFiles,
          ...source.isolated.workingFiles
        ])],
        pendingActions: [
          ...target.isolated.pendingActions,
          ...source.isolated.pendingActions.filter(a => 
            !target.isolated.pendingActions.some(ta => ta.id === a.id)
          )
        ],
        scratchpad: target.isolated.scratchpad + '\n---\n' + source.isolated.scratchpad,
      }
    };
  }
  // ... 其他策略
  return target;
}
```

## 🔗 与 Command Pattern Undo/Redo 的区别

| 特性 | Command Pattern (第 125 课) | Conversation Branching |
|------|---------------------------|----------------------|
| 关注点 | 撤销工具调用的副作用 | 探索不同对话路径 |
| 粒度 | 单个工具调用 | 整轮对话（含多个工具调用） |
| 状态恢复 | 执行 undo() 反向操作 | 切换到历史节点，重新发散 |
| 适用场景 | "撤销刚才的文件删除" | "回到选方案之前，试另一个" |
| 是否保留分支 | 否，线性栈 | 是，树形结构 |

**最佳实践：组合使用**
- 分支用于对话级别的探索
- Command Undo 用于分支内的工具副作用撤销

## 🎮 OpenClaw 实战

OpenClaw 的 `sessions_spawn` + `subagents` 可以模拟分支机制：

```typescript
// 伪代码：在 OpenClaw 中实现分支
async function createBranch(
  parentSessionKey: string,
  branchTask: string
): Promise<string> {
  // 1. 获取父会话历史
  const history = await sessions_history({
    sessionKey: parentSessionKey,
    limit: 50,
  });
  
  // 2. 序列化为上下文
  const contextSummary = history.messages
    .map(m => `${m.role}: ${m.content.slice(0, 200)}`)
    .join('\n');
  
  // 3. 在新子会话中启动分支
  const result = await sessions_spawn({
    task: `你是从以下对话分支出来的。保持之前的上下文，但探索新的方向。

之前的对话摘要：
${contextSummary}

新的任务：${branchTask}`,
    mode: 'session',
    label: `branch-${Date.now()}`,
  });
  
  return result.sessionKey;
}
```

## 📊 可视化对话树

```typescript
// 生成 Mermaid 图
function visualizeTree(tree: ConversationTree): string {
  const nodes = Array.from(tree['nodes'].values());
  let mermaid = 'graph TD\n';
  
  for (const node of nodes) {
    const label = node.content.slice(0, 30).replace(/"/g, "'") + '...';
    const style = node.isCheckpoint ? ':::checkpoint' : '';
    mermaid += `  ${node.id}["${node.role}: ${label}"]${style}\n`;
    
    if (node.parentId) {
      mermaid += `  ${node.parentId} --> ${node.id}\n`;
    }
  }
  
  mermaid += '  classDef checkpoint fill:#f9f,stroke:#333,stroke-width:2px\n';
  return mermaid;
}
```

## 💡 设计要点

1. **自动检查点** vs **手动检查点**：两者结合，关键决策自动存，用户也能随时存
2. **存储成本**：树形历史比线性历史占用更多空间，需要定期清理老分支
3. **上下文长度**：切换分支时需要重建上下文，注意 token 预算
4. **分支命名**：让用户能用自然语言引用分支（"刚才那个用 Python 的方案"）
5. **默认行为**：没有显式分支时，Agent 表现和线性对话一样，零学习成本

## 🧭 三句话总结

1. **对话不是一条线，是一棵树**——用户随时能回到岔路口重新选择
2. **自动检测决策点**——在用户做选择、Agent 给选项时自动存档
3. **分支共享全局上下文，隔离任务状态**——用户偏好跨分支同步，工作文件各走各的

---

下一课预告：**Agent 会话迁移与设备漫游（Session Migration & Device Roaming）**——让用户在手机上开始的对话，无缝在电脑上继续。
