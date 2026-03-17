# 72 - Multi-Agent Negotiation 多 Agent 协商与共识机制

> **核心思想**：多个 Agent 各自持有局部视角，通过协商达成全局一致决策，而不是依赖中央控制器强制下发。

---

## 为什么需要协商？

单一 Agent 的决策是独立的。但在多 Agent 系统中，Agent 之间会有**资源竞争**、**目标冲突**、**信息不对称**：

- Agent A 想操作同一个文件，Agent B 也想写
- Agent A 认为 timeout=5s，Agent B 认为需要 30s
- Agent A 已完成步骤 3，但 Agent B 还在步骤 1

这时就需要**协商协议**来解决冲突，而不是靠随机覆盖或异常崩溃。

---

## 协商的三种模式

### 1. 投票（Voting）

最简单的共识机制，多数服从少数：

```python
# learn-claude-code 风格：并行询问多个 Agent，汇总投票
import asyncio
from collections import Counter

async def agent_vote(agent_id: str, question: str, context: dict) -> str:
    """单个 Agent 给出投票意见"""
    prompt = f"""
    你是 Agent {agent_id}，请根据以下上下文给出决策：
    问题：{question}
    上下文：{context}
    
    只回答 "yes" 或 "no"，不要解释。
    """
    response = await call_llm(prompt)
    return response.strip().lower()

async def majority_vote(question: str, context: dict, agent_count: int = 5) -> str:
    """多数投票协商"""
    # 并发询问所有 Agent
    tasks = [
        agent_vote(f"agent_{i}", question, context)
        for i in range(agent_count)
    ]
    votes = await asyncio.gather(*tasks)
    
    # 统计票数
    counter = Counter(votes)
    winner = counter.most_common(1)[0][0]
    confidence = counter[winner] / agent_count
    
    print(f"投票结果: {dict(counter)}, 共识: {winner} (置信度: {confidence:.0%})")
    return winner

# 使用示例：决定是否继续执行高风险操作
result = await majority_vote(
    question="是否应该删除用户数据并重建索引？",
    context={"affected_rows": 50000, "estimated_time": "2h", "rollback_possible": True}
)
```

### 2. 拍卖（Auction / Bidding）

Agent 竞价获取资源或任务，出价最高（或最有能力）的 Agent 获胜：

```typescript
// pi-mono 风格：TypeScript 任务拍卖
interface TaskBid {
  agentId: string;
  taskId: string;
  confidence: number;    // 0-1，完成任务的置信度
  estimatedMs: number;   // 预估完成时间
  resourceCost: number;  // 资源消耗
}

interface AuctionResult {
  winner: string;
  bid: TaskBid;
  reason: string;
}

class TaskAuctioneer {
  private bids: Map<string, TaskBid[]> = new Map();

  async conductAuction(taskId: string, candidates: string[]): Promise<AuctionResult> {
    // 1. 广播任务，收集出价
    const bidPromises = candidates.map(agentId =>
      this.requestBid(agentId, taskId)
    );
    const allBids = await Promise.all(bidPromises);
    this.bids.set(taskId, allBids);

    // 2. 评分函数（可自定义权重）
    const score = (bid: TaskBid) =>
      bid.confidence * 0.5 +               // 置信度权重 50%
      (1 - bid.estimatedMs / 60000) * 0.3 + // 速度权重 30%（归一化到1分钟）
      (1 - bid.resourceCost / 100) * 0.2;   // 省资源权重 20%

    // 3. 选出最高分
    const winner = allBids.reduce((best, bid) =>
      score(bid) > score(best) ? bid : best
    );

    console.log(`任务 ${taskId} 拍卖结果:`, {
      winner: winner.agentId,
      score: score(winner).toFixed(3),
      allBids: allBids.map(b => ({ id: b.agentId, score: score(b).toFixed(3) }))
    });

    return {
      winner: winner.agentId,
      bid: winner,
      reason: `综合评分最高: confidence=${winner.confidence}, time=${winner.estimatedMs}ms`
    };
  }

  private async requestBid(agentId: string, taskId: string): Promise<TaskBid> {
    // 实际中通过消息队列发给对应 Agent，这里简化为直接调用
    const agent = AgentRegistry.get(agentId);
    return await agent.evaluateTask(taskId);
  }
}
```

### 3. 协商对话（Dialogue-based Negotiation）

Agent 之间通过多轮对话收敛到共识，适合复杂决策：

```python
# OpenClaw 实际用的模式：通过 sessions_send 实现 Agent 间协商
import json
from dataclasses import dataclass
from typing import Optional

@dataclass
class NegotiationState:
    round: int
    proposals: list[dict]
    agreed: Optional[dict] = None
    max_rounds: int = 5

async def negotiate_between_agents(
    proposer_session: str,
    reviewer_session: str,
    initial_proposal: dict
) -> dict:
    """
    两个 Agent 协商一个方案，直到达成共识或超过最大轮次
    """
    state = NegotiationState(round=0, proposals=[initial_proposal])
    current_proposal = initial_proposal

    while state.round < state.max_rounds:
        state.round += 1
        print(f"\n=== 协商第 {state.round} 轮 ===")

        # Reviewer Agent 评估当前方案
        review_prompt = f"""
        请评估以下方案，返回 JSON：
        方案：{json.dumps(current_proposal, ensure_ascii=False)}
        
        返回格式：
        {{
            "accept": true/false,
            "counter_proposal": {{...}} or null,
            "concerns": ["concern1", "concern2"]
        }}
        """
        review = await send_to_agent(reviewer_session, review_prompt)
        review_data = json.loads(extract_json(review))

        if review_data["accept"]:
            state.agreed = current_proposal
            print(f"✅ 达成共识！方案：{current_proposal}")
            break

        # 如果拒绝，用反提案继续协商
        if review_data.get("counter_proposal"):
            current_proposal = review_data["counter_proposal"]
            state.proposals.append(current_proposal)

            # Proposer 评估反提案
            proposer_response = await send_to_agent(
                proposer_session,
                f"Reviewer 提出反方案：{json.dumps(current_proposal, ensure_ascii=False)}\n"
                f"主要顾虑：{review_data['concerns']}\n"
                f"你是否接受？返回 JSON: {{\"accept\": bool, \"modified\": {{...}} or null}}"
            )
            proposer_data = json.loads(extract_json(proposer_response))

            if proposer_data["accept"]:
                state.agreed = current_proposal
                break
            elif proposer_data.get("modified"):
                current_proposal = proposer_data["modified"]
                state.proposals.append(current_proposal)

    if not state.agreed:
        # 协商失败，升级给人类决策
        print(f"⚠️ {state.max_rounds} 轮协商未达共识，上报人工处理")
        raise NegotiationFailedError(f"协商失败，历史方案: {state.proposals}")

    return state.agreed
```

---

## OpenClaw 中的实际应用

OpenClaw 用协商模式处理**并发工具调用冲突**：

```javascript
// OpenClaw 内部：工具锁 + 协商回退
class ToolNegotiator {
  private locks = new Map<string, string>(); // tool -> agentId

  async acquireTool(toolName: string, agentId: string, priority: number): Promise<boolean> {
    if (!this.locks.has(toolName)) {
      // 工具空闲，直接获取
      this.locks.set(toolName, agentId);
      return true;
    }

    const holder = this.locks.get(toolName)!;
    if (holder === agentId) return true; // 自己已持有

    // 协商：比较优先级
    const holderPriority = await this.getAgentPriority(holder);
    if (priority > holderPriority) {
      // 当前 Agent 优先级更高，抢占
      console.log(`Agent ${agentId} 抢占 ${holder} 的 ${toolName} 锁（优先级 ${priority} > ${holderPriority}）`);
      await this.notifyPreemption(holder, toolName);
      this.locks.set(toolName, agentId);
      return true;
    }

    // 优先级更低，等待或放弃
    return false;
  }

  async releaseTool(toolName: string, agentId: string): Promise<void> {
    if (this.locks.get(toolName) === agentId) {
      this.locks.delete(toolName);
      await this.notifyWaiters(toolName); // 通知等待队列
    }
  }
}
```

---

## 共识算法对比

| 算法 | 适用场景 | 优点 | 缺点 |
|------|---------|------|------|
| 多数投票 | 简单二元决策 | 实现简单、快速 | 质量取决于 Agent 能力 |
| 加权投票 | 专家系统 | 可体现能力差异 | 权重需要调优 |
| 拍卖竞价 | 任务分配 | 自然选出最优执行者 | 可能导致资源垄断 |
| 协商对话 | 复杂方案设计 | 质量高，考虑全面 | 延迟高，消耗 tokens 多 |
| Raft 共识 | 分布式状态同步 | 强一致性 | 复杂度高 |

---

## 关键设计原则

1. **设置最大轮次**：防止无限协商循环，超时后升级给人类
2. **记录协商历史**：方便审计和调试（`state.proposals`）
3. **定义"无共识"处理**：协商失败要有明确的 fallback 策略
4. **异步并发**：投票/竞价可以并行，节省时间
5. **优先级感知**：高优先级任务可以打断低优先级协商

---

## 实战 Tips

```python
# ✅ 好的实践：协商带超时
async def negotiate_with_timeout(agents, proposal, timeout=30):
    try:
        result = await asyncio.wait_for(
            negotiate_between_agents(agents, proposal),
            timeout=timeout
        )
        return result
    except asyncio.TimeoutError:
        # 超时就用默认方案，不要卡死
        logger.warning("协商超时，使用保守默认方案")
        return get_safe_default(proposal)

# ❌ 坏的实践：无限等待
result = await negotiate_between_agents(agents, proposal)  # 可能永远不返回
```

---

## 总结

- **投票**适合快速决策，**拍卖**适合任务分配，**对话协商**适合复杂方案
- 协商本质是分布式共识问题，关键是设计好评分函数和终止条件
- OpenClaw 用工具锁 + 优先级抢占处理并发冲突，是协商的轻量级实现
- 多 Agent 协商的终极目标：在没有中央控制器的情况下，让系统自我协调

下节课预告：**Agent Composition Patterns**（组合模式 — 如何像搭积木一样组合 Agent 能力）
