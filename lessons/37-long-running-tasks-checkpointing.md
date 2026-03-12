# 37 - Long-Running Tasks & Checkpointing：长任务与断点续传

> **核心问题**：Agent 执行一个需要几分钟甚至几小时的任务，中途崩了怎么办？

---

## 为什么需要断点续传？

现实中的 Agent 任务往往不是秒级的：

- 爬取并分析 1000 个网页 → 30 分钟
- 批量处理 10000 条数据 → 1 小时  
- 多步骤代码生成+测试迭代 → 不确定

**没有 Checkpointing 的后果：**

```
[任务进行到 80%]
  → API 超时
  → 进程崩溃
  → 用户 Ctrl+C
  → 从头再来 💸
```

---

## 核心思路：把任务变成可恢复的状态机

```
Task = State Machine
State = 当前进度 + 已完成结果 + 下一步

每次完成一个小步骤 → 保存 State → 继续
崩溃后重启 → 加载 State → 从上次断点继续
```

---

## Python 实现：learn-claude-code 风格

### 基础 Checkpoint 结构

```python
# checkpoint.py
import json
import os
from dataclasses import dataclass, asdict
from typing import Any, Optional
from pathlib import Path

@dataclass
class TaskCheckpoint:
    task_id: str
    status: str  # "running" | "paused" | "done" | "failed"
    current_step: int
    total_steps: int
    completed_items: list[Any]
    metadata: dict
    created_at: float
    updated_at: float

class CheckpointManager:
    def __init__(self, checkpoint_dir: str = ".checkpoints"):
        self.dir = Path(checkpoint_dir)
        self.dir.mkdir(exist_ok=True)
    
    def save(self, checkpoint: TaskCheckpoint):
        path = self.dir / f"{checkpoint.task_id}.json"
        with open(path, 'w') as f:
            json.dump(asdict(checkpoint), f, indent=2)
        print(f"[Checkpoint] Saved at step {checkpoint.current_step}/{checkpoint.total_steps}")
    
    def load(self, task_id: str) -> Optional[TaskCheckpoint]:
        path = self.dir / f"{task_id}.json"
        if not path.exists():
            return None
        with open(path) as f:
            data = json.load(f)
        return TaskCheckpoint(**data)
    
    def delete(self, task_id: str):
        path = self.dir / f"{task_id}.json"
        if path.exists():
            path.unlink()
```

### 带断点续传的 Agent 任务

```python
# long_task_agent.py
import time
import asyncio
from checkpoint import CheckpointManager, TaskCheckpoint

async def process_urls_with_checkpoint(
    urls: list[str],
    task_id: str = "url-batch-001"
):
    """
    批量处理 URL，支持断点续传
    """
    ckpt_mgr = CheckpointManager()
    
    # 1. 尝试加载已有进度
    checkpoint = ckpt_mgr.load(task_id)
    
    if checkpoint:
        print(f"[Resume] 从第 {checkpoint.current_step} 步继续，"
              f"已完成 {len(checkpoint.completed_items)} 个")
        results = checkpoint.completed_items
        start_idx = checkpoint.current_step
    else:
        print(f"[New Task] 开始处理 {len(urls)} 个 URL")
        results = []
        start_idx = 0
        
        # 创建初始 checkpoint
        checkpoint = TaskCheckpoint(
            task_id=task_id,
            status="running",
            current_step=0,
            total_steps=len(urls),
            completed_items=[],
            metadata={"total_urls": len(urls)},
            created_at=time.time(),
            updated_at=time.time()
        )
    
    # 2. 从断点继续执行
    for i, url in enumerate(urls[start_idx:], start=start_idx):
        try:
            # 实际处理逻辑（可以是 LLM 调用、API 请求等）
            result = await fetch_and_analyze(url)
            results.append(result)
            
            # 3. 每处理一个就保存进度
            checkpoint.current_step = i + 1
            checkpoint.completed_items = results
            checkpoint.updated_at = time.time()
            ckpt_mgr.save(checkpoint)
            
        except Exception as e:
            print(f"[Error] Step {i} failed: {e}")
            checkpoint.status = "failed"
            checkpoint.metadata["last_error"] = str(e)
            ckpt_mgr.save(checkpoint)
            raise
    
    # 4. 全部完成，删除 checkpoint
    checkpoint.status = "done"
    ckpt_mgr.save(checkpoint)
    ckpt_mgr.delete(task_id)  # 或者保留用于审计
    
    return results

async def fetch_and_analyze(url: str) -> dict:
    """模拟耗时操作"""
    await asyncio.sleep(0.1)  # 模拟网络请求
    return {"url": url, "status": "ok"}
```

---

## TypeScript 实现：pi-mono 风格

```typescript
// checkpoint.ts
interface Checkpoint<T> {
  taskId: string;
  status: 'running' | 'paused' | 'done' | 'failed';
  currentStep: number;
  totalSteps: number;
  results: T[];
  metadata: Record<string, unknown>;
  updatedAt: number;
}

class CheckpointStore<T> {
  private filePath: string;

  constructor(taskId: string, dir = '.checkpoints') {
    fs.mkdirSync(dir, { recursive: true });
    this.filePath = path.join(dir, `${taskId}.json`);
  }

  save(checkpoint: Checkpoint<T>): void {
    fs.writeFileSync(
      this.filePath,
      JSON.stringify(checkpoint, null, 2)
    );
  }

  load(): Checkpoint<T> | null {
    if (!fs.existsSync(this.filePath)) return null;
    return JSON.parse(fs.readFileSync(this.filePath, 'utf-8'));
  }

  clear(): void {
    if (fs.existsSync(this.filePath)) fs.unlinkSync(this.filePath);
  }
}

// 通用的带断点续传的批处理器
async function batchProcessWithCheckpoint<TInput, TOutput>(
  items: TInput[],
  processor: (item: TInput, index: number) => Promise<TOutput>,
  taskId: string,
  options = { saveEvery: 1 }  // 每 N 步保存一次
): Promise<TOutput[]> {
  const store = new CheckpointStore<TOutput>(taskId);
  let checkpoint = store.load();

  let results: TOutput[];
  let startIdx: number;

  if (checkpoint && checkpoint.status === 'running') {
    console.log(`▶ 从第 ${checkpoint.currentStep} 步恢复`);
    results = checkpoint.results;
    startIdx = checkpoint.currentStep;
  } else {
    console.log(`▶ 新任务，共 ${items.length} 步`);
    results = [];
    startIdx = 0;
    checkpoint = {
      taskId,
      status: 'running',
      currentStep: 0,
      totalSteps: items.length,
      results: [],
      metadata: {},
      updatedAt: Date.now(),
    };
  }

  for (let i = startIdx; i < items.length; i++) {
    const result = await processor(items[i], i);
    results.push(result);

    // 按频率保存 checkpoint
    if ((i + 1) % options.saveEvery === 0) {
      store.save({
        ...checkpoint,
        currentStep: i + 1,
        results,
        updatedAt: Date.now(),
      });
      console.log(`  ✓ ${i + 1}/${items.length}`);
    }
  }

  store.clear();
  return results;
}
```

---

## OpenClaw 实战：长任务管理

OpenClaw 的 `exec` + `process` 工具天然支持长任务管理：

```
// 启动长任务（background=true，不等待完成）
exec: command="python analyze.py", background=true, yieldMs=5000

// 稍后查询进度
process: action=poll, sessionId="xxx", timeout=2000

// 或者直接读进度文件
read: file=".checkpoints/analyze-task.json"
```

### 进度上报模式

```python
# 长任务内部主动写进度文件
import json, time

class ProgressReporter:
    def __init__(self, task_name: str):
        self.path = f".checkpoints/{task_name}-progress.json"
    
    def update(self, step: int, total: int, message: str = ""):
        data = {
            "step": step,
            "total": total, 
            "percent": round(step / total * 100, 1),
            "message": message,
            "ts": time.time()
        }
        with open(self.path, 'w') as f:
            json.dump(data, f)

# 使用
reporter = ProgressReporter("my-task")
for i, item in enumerate(items):
    process(item)
    reporter.update(i+1, len(items), f"处理: {item}")
```

---

## 分级 Checkpoint 策略

不是每步都要存档，要权衡**性能**与**安全**：

```
高频任务（每步都存）：
  → 每次 LLM 调用后存档
  → 适合：API 调用贵、失败损失大
  → 代价：大量磁盘 IO

低频任务（每 N 步存）：
  → 每 10/100 步存一次
  → 适合：步骤快、损失可接受
  → 代价：崩溃可能丢失最多 N 步

里程碑存档（关键节点存）：
  → 只在"阶段完成"时存档
  → 适合：有明确阶段边界的任务
  → 代价：阶段内崩溃从阶段头重来
```

```python
# 策略示例：里程碑存档
phases = [
    ("爬取数据", fetch_phase),
    ("清洗数据", clean_phase),
    ("分析数据", analyze_phase),
    ("生成报告", report_phase),
]

for phase_name, phase_fn in phases:
    ckpt = ckpt_mgr.load(f"{task_id}-{phase_name}")
    if ckpt and ckpt.status == "done":
        print(f"[Skip] {phase_name} 已完成，跳过")
        continue
    
    print(f"[Start] {phase_name}...")
    result = await phase_fn()
    
    ckpt_mgr.save(TaskCheckpoint(
        task_id=f"{task_id}-{phase_name}",
        status="done",
        ...
    ))
```

---

## 超时与取消的配合

Checkpointing 和 Timeout 是最佳搭档：

```python
import asyncio

async def run_with_timeout_and_checkpoint(
    task_coro,
    timeout_seconds: int,
    task_id: str
):
    try:
        return await asyncio.wait_for(
            task_coro,
            timeout=timeout_seconds
        )
    except asyncio.TimeoutError:
        # 超时时 checkpoint 已经保存了最新进度
        # 可以安全地重新调度
        print(f"[Timeout] 任务 {task_id} 超时，进度已保存，可以稍后继续")
        raise
    except asyncio.CancelledError:
        print(f"[Cancelled] 任务 {task_id} 被取消，进度已保存")
        raise
```

---

## 关键设计原则

```
1. 幂等性：同一步骤执行多次，结果相同
   → 重启后重新执行已完成的步骤不会造成副作用

2. 原子写入：要么写完整，要么不写
   → 写临时文件，rename 替换（防止写一半崩溃）

3. 版本标记：checkpoint 加版本号
   → 代码升级后，旧 checkpoint 可能不兼容

4. 过期清理：设置 TTL，定期清理过期 checkpoint
   → 防止磁盘堆积
```

```python
import tempfile, os

def atomic_write(path: str, data: dict):
    """原子写入：写临时文件再 rename"""
    tmp_path = path + ".tmp"
    try:
        with open(tmp_path, 'w') as f:
            json.dump(data, f)
        os.rename(tmp_path, path)  # 原子操作
    except:
        if os.path.exists(tmp_path):
            os.unlink(tmp_path)
        raise
```

---

## 总结

| 场景 | 策略 |
|------|------|
| LLM API 调用批量任务 | 每次调用后存档 |
| 网络爬虫/数据抓取 | 每 10-50 条存档 |
| 多阶段数据处理 | 里程碑存档 |
| 实时流式任务 | 维护 offset/cursor |
| 分布式任务 | 使用 Redis/DB 存档 |

**核心口诀：**
> 存了才敢跑，跑了就能停，停了还能续。

---

## 参考资源

- [learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) - Python Agent 实现
- [pi-mono](https://github.com/badlogic/pi-mono) - TypeScript 生产实现
- OpenClaw `exec` + `process` 工具 - 内置长任务管理
