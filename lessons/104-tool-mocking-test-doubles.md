# 104 - Agent 工具 Mock 与测试替身（Tool Mocking & Test Doubles）

> 测试 Agent 最大的痛点：每次跑测试都要真实调用 LLM 和外部工具，慢、贵、不稳定。
> 本节教你用 Mock / Stub / Spy / Record-Replay 四种模式彻底解决这个问题。

---

## 为什么 Agent 测试需要 Mock？

```
真实调用的问题：
  LLM 调用     → 每次 $0.01~$0.1，CI 跑 100 次 = $1~$10
  外部 API     → 限流、网络抖动、沙盒环境没权限
  数据库/文件  → 测试污染生产数据
  时间相关     → 日期/随机数不可重现

Mock 解决的问题：
  ✅ 离线运行，零成本
  ✅ 确定性输出，测试稳定
  ✅ 注入边界条件（超时、报错、空结果）
  ✅ 验证调用行为（调了几次、参数对不对）
```

---

## 四种测试替身模式

### 1. Stub — 返回固定值，不关心调用

```python
# learn-claude-code 风格：最简单的 Stub
class StubWeatherTool:
    """永远返回晴天，不管问哪里"""
    name = "get_weather"
    
    def run(self, location: str, date: str = None) -> dict:
        return {
            "location": location,
            "temperature": 25,
            "condition": "sunny",
            "humidity": 60
        }

# 测试时注入 stub
def test_travel_agent_sunny_day():
    agent = TravelAgent(tools=[StubWeatherTool()])
    result = agent.plan_trip("Sydney", "2026-03-21")
    assert "outdoor" in result.recommendations  # 晴天应该推荐户外活动
```

### 2. Mock — 记录调用，验证行为

```python
from unittest.mock import MagicMock, patch
import pytest

class MockToolRegistry:
    """拦截所有工具调用，记录参数"""
    
    def __init__(self):
        self.calls: list[dict] = []
        self._handlers: dict[str, callable] = {}
    
    def register_handler(self, tool_name: str, handler: callable):
        self._handlers[tool_name] = handler
    
    async def execute(self, tool_name: str, params: dict) -> dict:
        # 记录调用
        self.calls.append({
            "tool": tool_name,
            "params": params,
            "timestamp": time.time()
        })
        # 执行 handler 或返回默认值
        if tool_name in self._handlers:
            return await self._handlers[tool_name](**params)
        return {"status": "ok", "result": f"mock_{tool_name}_result"}
    
    # 断言辅助方法
    def assert_called_with(self, tool_name: str, **expected_params):
        matching = [c for c in self.calls if c["tool"] == tool_name]
        assert matching, f"Tool '{tool_name}' was never called"
        last_call = matching[-1]
        for key, value in expected_params.items():
            assert last_call["params"].get(key) == value, \
                f"Expected {key}={value}, got {last_call['params'].get(key)}"
    
    def assert_call_count(self, tool_name: str, count: int):
        actual = sum(1 for c in self.calls if c["tool"] == tool_name)
        assert actual == count, f"Expected {count} calls to '{tool_name}', got {actual}"


# 测试：验证 Agent 在用户问天气时正确调用工具
@pytest.mark.asyncio
async def test_agent_calls_weather_tool_once():
    mock_registry = MockToolRegistry()
    mock_registry.register_handler("get_weather", lambda location, **_: {
        "temperature": 22, "condition": "cloudy"
    })
    
    agent = ConversationalAgent(tool_registry=mock_registry)
    await agent.chat("Sydney 今天天气怎么样？")
    
    mock_registry.assert_called_with("get_weather", location="Sydney")
    mock_registry.assert_call_count("get_weather", 1)  # 不应该重复调用
```

### 3. Spy — 真实执行 + 记录（非侵入式）

```typescript
// pi-mono 风格：装饰器模式的 Spy
class ToolSpy<T extends Tool> {
  private _calls: ToolCallRecord[] = [];
  private _wrapped: T;

  constructor(wrapped: T) {
    this._wrapped = wrapped;
    // 用 Proxy 透明拦截
    return new Proxy(this, {
      get(target, prop) {
        if (prop === 'execute') {
          return async (params: unknown) => {
            const start = Date.now();
            let result: unknown;
            let error: Error | undefined;
            
            try {
              result = await wrapped.execute(params);
              return result;
            } catch (e) {
              error = e as Error;
              throw e;
            } finally {
              target._calls.push({
                params,
                result,
                error,
                durationMs: Date.now() - start,
                timestamp: new Date()
              });
            }
          };
        }
        return (wrapped as any)[prop];
      }
    });
  }

  get calls() { return this._calls; }
  get callCount() { return this._calls.length; }
  
  lastCallParams(): unknown {
    return this._calls.at(-1)?.params;
  }
  
  totalDurationMs(): number {
    return this._calls.reduce((sum, c) => sum + c.durationMs, 0);
  }
}

// 使用：真实工具 + 监控
const realSearchTool = new WebSearchTool({ apiKey: process.env.SEARCH_KEY });
const spiedSearch = new ToolSpy(realSearchTool);

const agent = new ResearchAgent({ tools: [spiedSearch] });
await agent.research("量子计算最新进展");

console.log(`搜索调用次数: ${spiedSearch.callCount}`);
console.log(`总耗时: ${spiedSearch.totalDurationMs()}ms`);
console.log(`最后一次查询: ${spiedSearch.lastCallParams()}`);
```

### 4. Record-Replay — 录制真实响应，离线回放

```python
# 核心思路：第一次真实调用，把响应存到磁盘；后续直接读磁盘
import json
import hashlib
from pathlib import Path

class RecordReplayTool:
    """
    Record mode:  调用真实 API，把响应写入 fixtures/
    Replay mode:  直接读 fixtures/，不发任何请求
    """
    
    FIXTURES_DIR = Path("tests/fixtures/tool_calls")
    
    def __init__(self, real_tool, mode: str = "replay"):
        """
        mode: 'record' | 'replay' | 'auto'
        auto = 有 fixture 就 replay，没有就 record
        """
        self.real_tool = real_tool
        self.mode = mode
        self.FIXTURES_DIR.mkdir(parents=True, exist_ok=True)
    
    def _fixture_path(self, tool_name: str, params: dict) -> Path:
        # 用参数 hash 做文件名，保证唯一性
        params_hash = hashlib.md5(
            json.dumps(params, sort_keys=True).encode()
        ).hexdigest()[:8]
        return self.FIXTURES_DIR / f"{tool_name}_{params_hash}.json"
    
    async def execute(self, tool_name: str, params: dict) -> dict:
        fixture_path = self._fixture_path(tool_name, params)
        
        if self.mode == "replay" or (self.mode == "auto" and fixture_path.exists()):
            if not fixture_path.exists():
                raise FileNotFoundError(
                    f"No fixture for {tool_name}({params}). "
                    f"Run with mode='record' first."
                )
            print(f"📼 [REPLAY] {tool_name}")
            return json.loads(fixture_path.read_text())
        
        else:  # record or auto without fixture
            print(f"🔴 [RECORD] {tool_name} → {fixture_path.name}")
            result = await self.real_tool.execute(tool_name, params)
            fixture_path.write_text(json.dumps(result, indent=2, ensure_ascii=False))
            return result


# 在 CI/CD 中使用
import os

def make_tool_registry():
    mode = "record" if os.getenv("RECORD_FIXTURES") else "replay"
    real_registry = ToolRegistry()
    real_registry.register_all_tools()
    return RecordReplayTool(real_registry, mode=mode)

# 第一次跑：RECORD_FIXTURES=1 pytest tests/  → 录制
# 后续 CI：pytest tests/                     → 回放，离线零成本
```

---

## OpenClaw 实战：给 Skills 写测试

```python
# 测试 mysterybox skill 的数据查询，不需要连真实 Grafana
import pytest
from unittest.mock import AsyncMock, patch

MOCK_GRAFANA_RESPONSE = {
    "results": {
        "A": {
            "frames": [{
                "data": {
                    "values": [
                        ["2026-03-21 00:00:00", "2026-03-21 01:00:00"],  # 时间
                        [42580.50, 38200.00],   # 收入
                        [35100.25, 31500.00],   # 赔付
                    ]
                },
                "schema": {
                    "fields": [
                        {"name": "time"},
                        {"name": "income"},
                        {"name": "payout"}
                    ]
                }
            }]
        }
    }
}

@pytest.mark.asyncio
async def test_profit_report_calculation():
    """验证利润计算逻辑，不实际查询 Grafana"""
    
    with patch("aiohttp.ClientSession.post") as mock_post:
        mock_post.return_value.__aenter__.return_value.json = AsyncMock(
            return_value=MOCK_GRAFANA_RESPONSE
        )
        mock_post.return_value.__aenter__.return_value.status = 200
        
        analyzer = MysteryboxProfitAnalyzer(
            grafana_url="http://mock-grafana",
            api_key="mock-key"
        )
        report = await analyzer.get_hourly_profit(hours=2)
    
    assert report.total_income == pytest.approx(80780.50)
    assert report.total_payout == pytest.approx(66600.25)
    assert report.profit_margin == pytest.approx(17.55, rel=0.01)
    assert len(report.hourly_breakdown) == 2
```

---

## 边界条件注入：让 Agent 在混沌中稳健

```python
class ChaosToolWrapper:
    """注入各种故障，测试 Agent 的容错能力"""
    
    def __init__(self, real_tool, chaos_config: dict):
        self.real_tool = real_tool
        self.config = chaos_config
        self._call_count = 0
    
    async def execute(self, tool_name: str, params: dict) -> dict:
        self._call_count += 1
        cfg = self.config.get(tool_name, {})
        
        # 1. 注入延迟
        if delay := cfg.get("delay_ms"):
            await asyncio.sleep(delay / 1000)
        
        # 2. 按比例注入错误
        if error_rate := cfg.get("error_rate", 0):
            if random.random() < error_rate:
                raise ToolExecutionError(f"[CHAOS] {tool_name} random failure")
        
        # 3. 第 N 次调用必定失败（测试重试逻辑）
        if fail_on_call := cfg.get("fail_on_call"):
            if self._call_count == fail_on_call:
                raise TimeoutError(f"[CHAOS] {tool_name} timeout on call #{fail_on_call}")
        
        # 4. 返回空结果
        if cfg.get("return_empty"):
            return {}
        
        return await self.real_tool.execute(tool_name, params)


# 测试：Agent 在搜索工具 50% 失败时仍能给出答案
@pytest.mark.asyncio
async def test_agent_resilient_to_tool_failures():
    chaos = ChaosToolWrapper(
        real_tool=MockSearchTool(),
        chaos_config={
            "web_search": {"error_rate": 0.5}
        }
    )
    
    agent = ResearchAgent(tools=[chaos], max_retries=3)
    
    # 即使工具偶尔失败，Agent 应该通过重试完成任务
    result = await agent.research("Python 最佳实践")
    assert result is not None
    assert len(result.content) > 100
```

---

## 测试金字塔：Agent 的三层测试策略

```
         /\
        /  \      E2E Tests (少量)
       / 真实LLM \   完整对话流程，验证端到端行为
      /____________\
     /              \
    /  Integration   \  Integration Tests (中量)
   /  (Stub LLM +    \  Mock LLM 响应，真实工具逻辑
  /   Real Tools)     \
 /____________________\
/                      \
/    Unit Tests          \  Unit Tests (大量)
/ (Stub LLM + Stub Tools) \  纯逻辑测试，最快最稳定
/__________________________\


实践建议：
  Unit       80% → 纯函数，工具解析，参数校验
  Integration 15% → Mock LLM + 真实数据库/API
  E2E          5% → 每次发版前跑一遍，验收完整流程
```

---

## 快速上手模板

```python
# conftest.py — pytest 全局配置
import pytest
from pathlib import Path

@pytest.fixture
def mock_llm():
    """返回预设响应的假 LLM"""
    responses = {}
    
    class MockLLM:
        def set_response(self, user_msg_contains: str, response: str):
            responses[user_msg_contains] = response
        
        async def complete(self, messages: list) -> str:
            last_user = next(
                (m["content"] for m in reversed(messages) if m["role"] == "user"),
                ""
            )
            for key, resp in responses.items():
                if key in last_user:
                    return resp
            return "我不太确定如何回答这个问题。"
    
    return MockLLM()

@pytest.fixture
def stub_tool_registry():
    """默认返回成功的工具注册表"""
    registry = MockToolRegistry()
    registry.register_handler("get_weather", lambda **_: {"temperature": 25, "condition": "sunny"})
    registry.register_handler("web_search", lambda query, **_: {"results": [{"title": f"关于{query}的结果", "url": "https://example.com"}]})
    return registry

@pytest.fixture
def record_replay_mode():
    """根据环境变量决定录制还是回放"""
    return "record" if os.getenv("RECORD_FIXTURES") else "replay"
```

---

## 关键原则

| 原则 | 说明 |
|------|------|
| **隔离边界** | Mock 外部依赖，测试自己的逻辑 |
| **确定性** | 相同输入，永远相同输出 |
| **最小 Mock** | 只 Mock 你需要控制的，其余走真实路径 |
| **快照测试** | 复杂输出用 snapshot 比对，而非手写 assert |
| **Fixture 版本控制** | Record-Replay 的 fixtures 要 commit 到 git |

---

## 总结

```
Stub     → 最简单，固定返回值，测试不关心调用行为
Mock     → 记录调用，验证 "调了几次" "参数对不对"
Spy      → 非侵入，真实执行 + 旁路监控
Record-Replay → 录一次，永远离线回放，CI 必备
Chaos    → 注入故障，测试容错和重试逻辑

组合使用：Unit 层 Stub 一切，Integration 层 Spy 关键路径，
         E2E 层才用真实 API（小心限流和费用）
```

> 好的 Agent 测试套件：5 分钟内跑完所有 unit tests，不花一分钱 API 费用。
