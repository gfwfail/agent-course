# 36 - Workflow Orchestration：工作流编排

> 让 Agent 按顺序、按条件、并行完成复杂任务

---

## 为什么需要工作流编排？

单个 Agent 调用一次 LLM 能做的事是有限的。现实中的任务往往是：

```
用户: "帮我分析竞品、写报告、发邮件给团队"
```

这不是一个 prompt 能搞定的——它需要**多个步骤、多个工具、多个决策点**协调配合。

**工作流编排** 就是解决这个问题的：
- 把复杂任务分解成有向无环图（DAG）
- 按依赖关系顺序/并行执行
- 每步结果作为下一步的输入
- 错误时知道从哪里重试

---

## 核心模式

### 1. 顺序链（Sequential Chain）

最简单：A → B → C

```python
# learn-claude-code 风格
async def sequential_pipeline(query: str) -> str:
    # Step 1: 搜索
    search_results = await search_tool(query)
    
    # Step 2: 用搜索结果提炼摘要
    summary = await llm_call(
        prompt=f"请总结以下内容：\n{search_results}",
        model="claude-haiku-3-5"  # 便宜的模型做简单任务
    )
    
    # Step 3: 基于摘要生成最终报告
    report = await llm_call(
        prompt=f"基于以下摘要，生成专业报告：\n{summary}",
        model="claude-sonnet-4-5"  # 复杂任务用好模型
    )
    
    return report
```

**要点**：每步的输出是下一步的输入，形成数据流水线。

---

### 2. 并行扇出（Parallel Fan-out）

多个独立任务同时执行，最后汇总：

```typescript
// pi-mono 风格 (TypeScript)
async function parallelResearch(topic: string): Promise<Report> {
    // 同时启动三个独立搜索
    const [techResults, marketResults, competitorResults] = await Promise.all([
        searchWeb(`${topic} technology trends`),
        searchWeb(`${topic} market size`),
        searchWeb(`${topic} competitors`),
    ]);

    // 汇总结果
    const report = await llm.complete({
        messages: [{
            role: "user",
            content: `
                技术趋势: ${techResults}
                市场数据: ${marketResults}  
                竞品分析: ${competitorResults}
                
                请生成综合分析报告。
            `
        }]
    });

    return report;
}
```

**要点**：`Promise.all` 并发执行，节省 3x 时间。

---

### 3. 条件分支（Conditional Branch）

根据中间结果决定走哪条路：

```python
async def smart_response(user_query: str) -> str:
    # 先分类
    intent = await classify_intent(user_query)
    
    if intent == "code_question":
        # 走代码专家路径
        return await code_expert_agent(user_query)
    elif intent == "data_analysis":
        # 走数据分析路径
        data = await fetch_data()
        return await analysis_agent(user_query, data)
    else:
        # 通用路径
        return await general_agent(user_query)
```

**OpenClaw 实战**：OpenClaw 的 tool policy pipeline 就是条件分支——根据工具名、参数内容决定是否允许执行。

---

### 4. 循环与迭代（Loop / Iteration）

Agent 反复执行直到满足条件：

```python
async def iterative_refine(draft: str, criteria: str) -> str:
    """反复优化直到通过审核"""
    for attempt in range(5):  # 最多 5 次
        # 评估当前版本
        evaluation = await llm_call(
            prompt=f"""
            审核以下内容是否满足标准：
            标准: {criteria}
            内容: {draft}
            
            如果满足，回复 PASS。
            如果不满足，回复 FAIL: <具体问题>
            """
        )
        
        if evaluation.startswith("PASS"):
            return draft
        
        # 根据反馈修改
        issue = evaluation.replace("FAIL: ", "")
        draft = await llm_call(
            prompt=f"修改以下内容，解决问题：{issue}\n\n{draft}"
        )
    
    return draft  # 超次数就返回最后版本
```

这就是 **Reflection + Refinement** 的核心模式。

---

### 5. Map-Reduce（映射归约）

处理大量数据时的标准模式：

```typescript
// 处理 100 个文档
async function processDocuments(docs: string[]): Promise<string> {
    // Map: 并发处理每个文档（但要限流！）
    const BATCH_SIZE = 10;
    const summaries: string[] = [];
    
    for (let i = 0; i < docs.length; i += BATCH_SIZE) {
        const batch = docs.slice(i, i + BATCH_SIZE);
        const batchSummaries = await Promise.all(
            batch.map(doc => summarize(doc))
        );
        summaries.push(...batchSummaries);
        
        // 避免触发 rate limit
        if (i + BATCH_SIZE < docs.length) {
            await sleep(1000);
        }
    }
    
    // Reduce: 汇总所有摘要
    return await llm.complete({
        messages: [{
            role: "user",
            content: `综合以下 ${summaries.length} 个摘要，生成最终报告：\n\n${summaries.join('\n\n')}`
        }]
    });
}
```

---

## OpenClaw 的工作流实现

OpenClaw 本身就是一个工作流引擎。看看它如何处理一个复杂 cron job：

```
用户设置 cron: "每天早上分析昨日销售数据，发报告给团队"

OpenClaw 编排流程:
1. [触发] cron 到时间，注入 systemEvent
2. [LLM] 理解任务，制定工作流计划
3. [工具] 查询数据库获取销售数据
4. [工具] 调用 Grafana API 获取图表
5. [LLM] 分析数据，生成报告文本
6. [工具] message.send 发送给团队
7. [工具] memory 记录已完成
```

每个步骤的输出自动流入下一步——这就是 Agent Loop 中隐式的工作流编排。

---

## 实战：构建一个显式工作流引擎

```python
from dataclasses import dataclass
from typing import Callable, Any, List, Optional
import asyncio

@dataclass
class WorkflowStep:
    name: str
    fn: Callable
    depends_on: List[str] = None  # 依赖的步骤名
    
class WorkflowEngine:
    def __init__(self):
        self.steps: dict[str, WorkflowStep] = {}
        self.results: dict[str, Any] = {}
    
    def add_step(self, step: WorkflowStep):
        self.steps[step.name] = step
    
    async def run(self, initial_input: Any) -> dict:
        self.results["__input__"] = initial_input
        executed = set()
        
        while len(executed) < len(self.steps):
            # 找出所有依赖已满足的步骤
            ready = [
                name for name, step in self.steps.items()
                if name not in executed
                and all(dep in executed for dep in (step.depends_on or []))
            ]
            
            if not ready:
                break  # 死锁检测
            
            # 并行执行所有就绪步骤
            tasks = [self._run_step(name) for name in ready]
            await asyncio.gather(*tasks)
            executed.update(ready)
        
        return self.results
    
    async def _run_step(self, name: str):
        step = self.steps[name]
        deps_results = {
            dep: self.results[dep] 
            for dep in (step.depends_on or [])
        }
        self.results[name] = await step.fn(
            self.results.get("__input__"), 
            deps_results
        )

# 使用示例
engine = WorkflowEngine()

engine.add_step(WorkflowStep("search", search_web))
engine.add_step(WorkflowStep("fetch_data", fetch_db_data))
engine.add_step(WorkflowStep(
    "analyze", 
    analyze_all,
    depends_on=["search", "fetch_data"]  # 等两个都完成再分析
))
engine.add_step(WorkflowStep(
    "report",
    generate_report,
    depends_on=["analyze"]
))

result = await engine.run("分析 CS:GO 皮肤市场趋势")
```

这个引擎会自动：
- 并行执行 `search` 和 `fetch_data`（无依赖关系）
- 等两者完成后执行 `analyze`
- 最后执行 `report`

---

## 工作流 vs Agent Loop 的区别

| 维度 | Agent Loop | 工作流编排 |
|------|-----------|-----------|
| 控制权 | LLM 决定下一步 | 代码预定义步骤 |
| 灵活性 | 高（可动态适应） | 低（固定路径） |
| 可预测性 | 低 | 高 |
| 调试难度 | 高 | 低 |
| 适用场景 | 开放探索任务 | 标准化业务流程 |

**最佳实践**：把工作流编排和 Agent Loop 结合——用工作流定义骨架，每个节点内部用 LLM 做灵活决策。

---

## 关键设计原则

### ✅ 幂等设计
每个步骤重复执行结果相同，方便失败重试：
```python
# 不好：每次都创建新记录
await db.insert({"data": result})

# 好：用唯一 ID 做 upsert
await db.upsert({"id": task_id, "data": result})
```

### ✅ 检查点（Checkpoint）
长工作流保存中间状态，断点续跑：
```python
checkpoint_file = f"/tmp/workflow_{task_id}.json"

# 恢复已完成的步骤
if os.path.exists(checkpoint_file):
    completed = json.load(open(checkpoint_file))
else:
    completed = {}

for step_name, step_fn in workflow_steps:
    if step_name in completed:
        print(f"跳过已完成步骤: {step_name}")
        continue
    
    result = await step_fn()
    completed[step_name] = result
    json.dump(completed, open(checkpoint_file, "w"))  # 保存
```

### ✅ 超时与熔断
```python
try:
    result = await asyncio.wait_for(
        expensive_step(), 
        timeout=30.0  # 30秒超时
    )
except asyncio.TimeoutError:
    # 降级处理
    result = await fast_fallback()
```

---

## 总结

工作流编排的核心思想：

1. **分解**：把大任务拆成小步骤
2. **依赖**：明确步骤间的先后关系
3. **并行**：无依赖的步骤同时跑
4. **恢复**：失败时从断点重试，不从头来
5. **混合**：工作流定框架，LLM 做决策

下次你写 Agent 遇到复杂任务，先画个流程图——你的代码结构自然就清晰了。

---

*课程 #36 | Agent 开发课程 | 2026-03-13*
