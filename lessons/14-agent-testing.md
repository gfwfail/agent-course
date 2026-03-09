# 第 14 课：Agent Testing - 如何测试 AI Agent

> 🎯 本课目标：掌握 Agent 测试的核心策略和实现方法

## 为什么 Agent 测试很特殊？

传统软件测试：输入确定 → 输出确定 → 断言

Agent 测试的挑战：
- **非确定性输出**：同样的 prompt，LLM 可能返回不同的结果
- **工具调用链**：Agent 可能用不同的工具组合完成同一任务
- **上下文依赖**：行为受历史对话影响
- **外部依赖**：真实 API 调用昂贵且不稳定

## 测试金字塔

```
          ╱╲
         ╱  ╲        E2E Tests
        ╱────╲       (最少，最慢，最贵)
       ╱      ╲
      ╱────────╲     Integration Tests
     ╱          ╲    (适量)
    ╱────────────╲
   ╱              ╲   Unit Tests
  ╱────────────────╲  (最多，最快，最便宜)
```

## 策略 1：Mock LLM 响应

最基础也最重要的策略——用预设响应替代真实 API 调用。

### learn-claude-code 示例

```python
# tests/conftest.py
import pytest
from unittest.mock import AsyncMock

@pytest.fixture
def mock_llm():
    """创建一个可控的 LLM mock"""
    mock = AsyncMock()
    
    # 预设响应
    mock.complete.return_value = {
        "content": [
            {
                "type": "tool_use",
                "id": "call_123",
                "name": "read_file",
                "input": {"path": "test.txt"}
            }
        ],
        "stop_reason": "tool_use"
    }
    
    return mock

# tests/test_agent_loop.py
async def test_agent_calls_correct_tool(mock_llm):
    """测试 Agent 是否正确调用工具"""
    agent = Agent(llm=mock_llm)
    
    result = await agent.run("读取 test.txt 文件内容")
    
    # 验证 LLM 被调用
    assert mock_llm.complete.called
    
    # 验证工具调用
    tool_calls = result.tool_calls
    assert len(tool_calls) == 1
    assert tool_calls[0].name == "read_file"
    assert tool_calls[0].input["path"] == "test.txt"
```

### pi-mono 风格（TypeScript）

```typescript
// tests/mocks/llm.mock.ts
export function createMockLLM(responses: LLMResponse[]): LLMClient {
  let callIndex = 0;
  
  return {
    async complete(messages: Message[]): Promise<LLMResponse> {
      if (callIndex >= responses.length) {
        throw new Error('No more mock responses');
      }
      return responses[callIndex++];
    }
  };
}

// tests/agent.test.ts
describe('Agent', () => {
  it('should execute tool and continue', async () => {
    const mockLLM = createMockLLM([
      // 第一次调用：返回工具调用
      {
        content: [{
          type: 'tool_use',
          id: 'call_1',
          name: 'bash',
          input: { command: 'ls' }
        }],
        stop_reason: 'tool_use'
      },
      // 第二次调用：返回最终结果
      {
        content: [{ type: 'text', text: '目录内容是...' }],
        stop_reason: 'end_turn'
      }
    ]);
    
    const agent = new Agent({ llm: mockLLM });
    const result = await agent.run('列出当前目录');
    
    expect(result.finalText).toContain('目录内容');
  });
});
```

## 策略 2：工具层隔离测试

把工具实现和 Agent 逻辑分开测试。

```python
# tests/test_tools.py
import pytest
from tools import FileReader, BashExecutor

class TestFileReader:
    """单独测试文件读取工具"""
    
    def test_read_existing_file(self, tmp_path):
        # 创建临时文件
        test_file = tmp_path / "test.txt"
        test_file.write_text("hello world")
        
        reader = FileReader()
        result = reader.execute({"path": str(test_file)})
        
        assert result.success
        assert result.content == "hello world"
    
    def test_read_nonexistent_file(self, tmp_path):
        reader = FileReader()
        result = reader.execute({"path": "/nonexistent/file.txt"})
        
        assert not result.success
        assert "not found" in result.error.lower()
    
    def test_read_binary_file_rejected(self, tmp_path):
        # 二进制文件应该被拒绝
        binary = tmp_path / "image.png"
        binary.write_bytes(b'\x89PNG\r\n\x1a\n')
        
        reader = FileReader()
        result = reader.execute({"path": str(binary)})
        
        assert not result.success
        assert "binary" in result.error.lower()

class TestBashExecutor:
    """单独测试 Bash 执行工具"""
    
    def test_simple_command(self):
        executor = BashExecutor()
        result = executor.execute({"command": "echo hello"})
        
        assert result.success
        assert result.stdout.strip() == "hello"
    
    def test_command_timeout(self):
        executor = BashExecutor(timeout=1)
        result = executor.execute({"command": "sleep 10"})
        
        assert not result.success
        assert "timeout" in result.error.lower()
    
    def test_dangerous_command_blocked(self):
        executor = BashExecutor()
        result = executor.execute({"command": "rm -rf /"})
        
        assert not result.success
        assert "blocked" in result.error.lower()
```

## 策略 3：Snapshot 测试

对于复杂输出，使用 snapshot 比较。

```typescript
// tests/prompt.test.ts
import { buildSystemPrompt } from '../src/prompt';

describe('System Prompt', () => {
  it('should generate consistent prompt', () => {
    const prompt = buildSystemPrompt({
      tools: ['read_file', 'write_file', 'bash'],
      workdir: '/home/user/project',
      os: 'darwin'
    });
    
    // 第一次运行会创建 snapshot
    // 之后每次运行都会比较
    expect(prompt).toMatchSnapshot();
  });
});
```

Python 版本：

```python
# tests/test_prompts.py
import pytest

def test_system_prompt_snapshot(snapshot):
    prompt = build_system_prompt(
        tools=['read_file', 'write_file'],
        workdir='/home/user'
    )
    
    # 使用 pytest-snapshot 或 syrupy
    assert prompt == snapshot
```

## 策略 4：行为测试（BDD 风格）

测试 Agent 的行为模式，而不是具体输出。

```python
# tests/test_behaviors.py
import pytest
from agent import Agent

class TestAgentBehaviors:
    """测试 Agent 的行为模式"""
    
    @pytest.fixture
    def agent(self, mock_llm, mock_tools):
        return Agent(llm=mock_llm, tools=mock_tools)
    
    async def test_reads_file_before_editing(self, agent, mock_tools):
        """Agent 应该先读取文件再编辑"""
        # 设置 mock 响应序列
        agent.llm.set_responses([
            # 先读取
            tool_call("read_file", {"path": "code.py"}),
            # 再编辑
            tool_call("edit_file", {"path": "code.py", "changes": "..."}),
            # 完成
            text_response("Done!")
        ])
        
        await agent.run("修改 code.py，添加错误处理")
        
        # 验证调用顺序
        calls = mock_tools.get_call_sequence()
        assert calls[0].name == "read_file"
        assert calls[1].name == "edit_file"
    
    async def test_retries_on_tool_error(self, agent, mock_tools):
        """工具失败时 Agent 应该重试"""
        # 第一次失败，第二次成功
        mock_tools.set_results("bash", [
            {"success": False, "error": "Connection refused"},
            {"success": True, "stdout": "OK"}
        ])
        
        agent.llm.set_responses([
            tool_call("bash", {"command": "curl api.example.com"}),
            tool_call("bash", {"command": "curl api.example.com"}),
            text_response("API 返回 OK")
        ])
        
        result = await agent.run("检查 API 状态")
        
        # Agent 应该成功完成，即使第一次失败
        assert "OK" in result.text
        assert mock_tools.call_count("bash") == 2
```

## 策略 5：录制回放测试

录制真实 API 交互，回放测试。

```python
# tests/conftest.py
import json
import hashlib
from pathlib import Path

class LLMRecorder:
    """录制 LLM 调用用于回放测试"""
    
    def __init__(self, recordings_dir: Path, record_mode: bool = False):
        self.recordings_dir = recordings_dir
        self.record_mode = record_mode
        self.real_llm = None
    
    def _get_recording_path(self, messages: list) -> Path:
        # 用消息内容的 hash 作为文件名
        content = json.dumps(messages, sort_keys=True)
        hash_id = hashlib.md5(content.encode()).hexdigest()[:12]
        return self.recordings_dir / f"{hash_id}.json"
    
    async def complete(self, messages: list) -> dict:
        path = self._get_recording_path(messages)
        
        if self.record_mode:
            # 录制模式：调用真实 API 并保存
            response = await self.real_llm.complete(messages)
            path.write_text(json.dumps({
                "messages": messages,
                "response": response
            }, indent=2))
            return response
        else:
            # 回放模式：从文件读取
            if not path.exists():
                raise FileNotFoundError(
                    f"No recording found for this input. "
                    f"Run with RECORD=1 to create."
                )
            data = json.loads(path.read_text())
            return data["response"]

# 使用
@pytest.fixture
def llm(request):
    record_mode = os.environ.get("RECORD", "0") == "1"
    return LLMRecorder(
        recordings_dir=Path("tests/recordings"),
        record_mode=record_mode
    )
```

## 策略 6：Property-Based Testing

使用 Hypothesis 测试不变性。

```python
# tests/test_properties.py
from hypothesis import given, strategies as st
from agent import Agent

class TestAgentProperties:
    """测试 Agent 的不变性质"""
    
    @given(st.text(min_size=1, max_size=100))
    def test_never_crashes_on_input(self, user_input):
        """无论输入什么，Agent 都不应该崩溃"""
        agent = Agent(mock_mode=True)
        
        # 应该返回结果或抛出可控的异常
        try:
            result = agent.run(user_input)
            assert result is not None
        except AgentError as e:
            # 可控的业务异常是允许的
            assert e.user_message is not None
    
    @given(st.lists(st.text(), min_size=1, max_size=10))
    def test_tool_results_always_serializable(self, file_contents):
        """工具结果必须可序列化"""
        tool = FileReader()
        
        for content in file_contents:
            # 创建临时文件
            with tempfile.NamedTemporaryFile(mode='w', delete=False) as f:
                f.write(content)
                path = f.name
            
            try:
                result = tool.execute({"path": path})
                # 必须能 JSON 序列化
                json.dumps(result.to_dict())
            finally:
                os.unlink(path)
```

## 策略 7：OpenClaw 特有的测试模式

```typescript
// OpenClaw 的 Session 测试
describe('Session', () => {
  let session: Session;
  let mockChannel: MockChannel;
  
  beforeEach(() => {
    mockChannel = new MockChannel();
    session = new Session({
      channel: mockChannel,
      llm: createMockLLM([/* ... */])
    });
  });
  
  it('should route messages to correct channel', async () => {
    await session.handleMessage({
      text: 'Hello',
      from: { id: 'user123', channel: 'telegram' }
    });
    
    expect(mockChannel.sentMessages).toHaveLength(1);
    expect(mockChannel.sentMessages[0].to).toBe('user123');
  });
  
  it('should persist state between turns', async () => {
    // 第一轮
    await session.handleMessage({ text: '记住我叫小明' });
    
    // 第二轮
    await session.handleMessage({ text: '我叫什么？' });
    
    const lastResponse = mockChannel.sentMessages.at(-1);
    expect(lastResponse.text).toContain('小明');
  });
});
```

## 测试目录结构

```
tests/
├── unit/
│   ├── test_tools.py           # 工具单元测试
│   ├── test_prompt_builder.py  # Prompt 构建测试
│   └── test_message_parser.py  # 消息解析测试
├── integration/
│   ├── test_agent_loop.py      # Agent 循环测试
│   ├── test_tool_chain.py      # 工具链测试
│   └── test_session.py         # 会话测试
├── e2e/
│   ├── test_real_tasks.py      # 真实任务测试
│   └── test_multi_turn.py      # 多轮对话测试
├── recordings/                  # API 录制文件
│   ├── abc123def456.json
│   └── ...
├── fixtures/                    # 测试数据
│   ├── sample_code.py
│   └── test_project/
└── conftest.py                  # 共享 fixtures
```

## 实用技巧

### 1. 测试重试逻辑

```python
def test_retry_with_deterministic_failures():
    """用确定性的失败序列测试重试"""
    failures = [True, True, False]  # 前两次失败，第三次成功
    
    call_count = 0
    def mock_api():
        nonlocal call_count
        should_fail = failures[call_count]
        call_count += 1
        if should_fail:
            raise APIError("Temporary failure")
        return "Success"
    
    result = retry_with_backoff(mock_api, max_retries=3)
    
    assert result == "Success"
    assert call_count == 3
```

### 2. 测试超时

```python
@pytest.mark.timeout(5)
async def test_agent_respects_timeout():
    """Agent 应该在超时时停止"""
    agent = Agent(timeout=2)
    
    # 设置一个会无限循环的 mock
    agent.llm.set_infinite_loop()
    
    with pytest.raises(TimeoutError):
        await agent.run("Do something")
```

### 3. 测试并发安全

```python
async def test_concurrent_sessions():
    """多个会话并发运行不应该互相干扰"""
    sessions = [create_session(f"user_{i}") for i in range(10)]
    
    async def run_session(session, user_id):
        await session.run(f"我是 {user_id}")
        return session.get_context()
    
    results = await asyncio.gather(*[
        run_session(s, f"user_{i}") 
        for i, s in enumerate(sessions)
    ])
    
    # 每个会话应该有独立的上下文
    for i, ctx in enumerate(results):
        assert f"user_{i}" in ctx.get("user_name", "")
```

## 总结

| 策略 | 适用场景 | 优点 | 缺点 |
|------|----------|------|------|
| Mock LLM | 单元测试 | 快速、确定 | 不测真实 LLM |
| 工具隔离 | 工具开发 | 独立验证 | 不测集成 |
| Snapshot | Prompt 测试 | 检测变更 | 需要维护 |
| 行为测试 | 集成测试 | 测真实流程 | 设置复杂 |
| 录制回放 | 回归测试 | 真实数据 | 录制成本 |
| Property | 边界测试 | 发现边界 | 难写好断言 |

核心原则：
1. **分层测试**：从工具到 Agent 到 E2E
2. **确定性优先**：尽量让测试可重复
3. **成本意识**：真实 API 调用放在 CI 的特定阶段
4. **录制为王**：重要场景录制下来，永久回归

---

下一课预告：**Retrieval Augmented Generation (RAG) - 让 Agent 具备知识检索能力**
