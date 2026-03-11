# 28 - Graceful Degradation：Agent 优雅降级

## 概述

当依赖的服务不可用时，Agent 如何保持可用性？优雅降级让 Agent 在部分功能失效时依然能够提供有价值的响应，而不是直接崩溃或返回错误。

## 核心概念

### 什么是优雅降级？

```
┌─────────────────────────────────────────────────────────────┐
│                     正常状态                                  │
│  User Request → Tool A → Tool B → Tool C → Complete Response │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                   降级状态 (Tool B 失败)                      │
│  User Request → Tool A → [Fallback] → Tool C → Partial Response │
│                              ↓                               │
│                     - 使用缓存数据                            │
│                     - 跳过非关键功能                          │
│                     - 提供替代方案                            │
└─────────────────────────────────────────────────────────────┘
```

### 降级策略金字塔

```
        ┌────────────────┐
        │   Full Feature │  ← 所有服务正常
        ├────────────────┤
        │  Reduced Feature│  ← 非关键服务失败
        ├────────────────┤
        │   Basic Feature │  ← 多个服务失败
        ├────────────────┤
        │   Read-Only    │  ← 只读模式
        ├────────────────┤
        │   Status Only  │  ← 仅状态查询
        └────────────────┘
```

## 实现模式

### 模式 1：服务可用性检测

**Python 实现 (learn-claude-code 风格)**

```python
from enum import Enum
from dataclasses import dataclass
from typing import Optional, Callable, Any
import asyncio
import time


class ServiceStatus(Enum):
    HEALTHY = "healthy"
    DEGRADED = "degraded"  
    UNAVAILABLE = "unavailable"


@dataclass
class ServiceHealth:
    name: str
    status: ServiceStatus
    last_check: float
    failure_count: int = 0
    last_error: Optional[str] = None


class ServiceRegistry:
    """服务健康状态注册表"""
    
    def __init__(self):
        self.services: dict[str, ServiceHealth] = {}
        self.health_checks: dict[str, Callable] = {}
        
    def register(self, name: str, health_check: Callable):
        """注册服务及其健康检查函数"""
        self.services[name] = ServiceHealth(
            name=name,
            status=ServiceStatus.HEALTHY,
            last_check=time.time()
        )
        self.health_checks[name] = health_check
        
    async def check_health(self, name: str) -> ServiceStatus:
        """检查单个服务健康状态"""
        if name not in self.health_checks:
            return ServiceStatus.UNAVAILABLE
            
        try:
            result = await self.health_checks[name]()
            self.services[name].status = ServiceStatus.HEALTHY
            self.services[name].failure_count = 0
            self.services[name].last_error = None
        except Exception as e:
            health = self.services[name]
            health.failure_count += 1
            health.last_error = str(e)
            
            # 根据失败次数判断状态
            if health.failure_count >= 3:
                health.status = ServiceStatus.UNAVAILABLE
            else:
                health.status = ServiceStatus.DEGRADED
                
        self.services[name].last_check = time.time()
        return self.services[name].status
        
    def is_available(self, name: str) -> bool:
        """快速检查服务是否可用"""
        if name not in self.services:
            return False
        return self.services[name].status != ServiceStatus.UNAVAILABLE


# 使用示例
registry = ServiceRegistry()

async def check_weather_api():
    # 模拟 API 健康检查
    async with aiohttp.ClientSession() as session:
        async with session.get("https://api.weather.com/health", timeout=5) as resp:
            if resp.status != 200:
                raise Exception(f"Weather API returned {resp.status}")

registry.register("weather", check_weather_api)
```

### 模式 2：功能降级装饰器

**TypeScript 实现 (pi-mono 风格)**

```typescript
// pi-mono/apps/pi/src/utils/degradation.ts

interface DegradationConfig {
  fallback?: () => Promise<any>;      // 降级函数
  cache?: {
    key: string;
    ttl: number;                       // 缓存有效期(ms)
  };
  skipOnFailure?: boolean;             // 失败时跳过
  criticalPath?: boolean;              // 是否关键路径
}

class DegradationManager {
  private cache = new Map<string, { value: any; expires: number }>();
  
  /**
   * 包装函数，添加降级能力
   */
  wrap<T>(
    fn: () => Promise<T>,
    config: DegradationConfig
  ): () => Promise<T | null> {
    return async () => {
      try {
        const result = await fn();
        
        // 成功时更新缓存
        if (config.cache) {
          this.cache.set(config.cache.key, {
            value: result,
            expires: Date.now() + config.cache.ttl
          });
        }
        
        return result;
      } catch (error) {
        console.warn(`Function failed, attempting degradation:`, error);
        
        // 策略1: 使用缓存
        if (config.cache) {
          const cached = this.cache.get(config.cache.key);
          if (cached && cached.expires > Date.now()) {
            console.log(`Using cached value for ${config.cache.key}`);
            return cached.value;
          }
        }
        
        // 策略2: 使用 fallback
        if (config.fallback) {
          console.log(`Using fallback function`);
          return config.fallback();
        }
        
        // 策略3: 跳过非关键功能
        if (config.skipOnFailure && !config.criticalPath) {
          console.log(`Skipping non-critical function`);
          return null;
        }
        
        // 关键路径失败，向上抛出
        throw error;
      }
    };
  }
}

// 在 Agent 中使用
const degradation = new DegradationManager();

const getWeatherWithDegradation = degradation.wrap(
  async () => fetchWeatherAPI(location),
  {
    cache: { key: `weather:${location}`, ttl: 30 * 60 * 1000 }, // 30分钟
    fallback: async () => ({
      source: 'fallback',
      message: '天气服务暂不可用，请稍后再试',
      lastKnown: getCachedWeather(location)
    }),
    skipOnFailure: false,
    criticalPath: false
  }
);
```

### 模式 3：多级降级策略

```python
# learn-claude-code 风格

from typing import TypeVar, Generic, List, Callable, Awaitable
from dataclasses import dataclass

T = TypeVar('T')


@dataclass
class DegradationLevel(Generic[T]):
    name: str
    executor: Callable[[], Awaitable[T]]
    quality: float  # 0.0 - 1.0，表示结果质量


class MultilevelDegradation(Generic[T]):
    """多级降级策略"""
    
    def __init__(self):
        self.levels: List[DegradationLevel[T]] = []
        
    def add_level(
        self, 
        name: str, 
        executor: Callable[[], Awaitable[T]], 
        quality: float
    ):
        """添加降级级别（按质量排序）"""
        level = DegradationLevel(name, executor, quality)
        self.levels.append(level)
        self.levels.sort(key=lambda x: x.quality, reverse=True)
        return self
        
    async def execute(self) -> tuple[T, str, float]:
        """尝试执行，返回 (结果, 使用的级别, 质量)"""
        errors = []
        
        for level in self.levels:
            try:
                result = await level.executor()
                return (result, level.name, level.quality)
            except Exception as e:
                errors.append(f"{level.name}: {e}")
                continue
                
        # 所有级别都失败
        raise Exception(f"All degradation levels failed: {errors}")


# 使用示例：代码搜索功能
async def search_code(query: str) -> dict:
    strategy = MultilevelDegradation[dict]()
    
    # Level 1: 完整语义搜索 (质量 1.0)
    strategy.add_level(
        "semantic_search",
        lambda: semantic_code_search(query),
        quality=1.0
    )
    
    # Level 2: 本地 ripgrep 搜索 (质量 0.7)
    strategy.add_level(
        "ripgrep_search",
        lambda: ripgrep_search(query),
        quality=0.7
    )
    
    # Level 3: 简单文件名匹配 (质量 0.3)
    strategy.add_level(
        "filename_match",
        lambda: filename_match_search(query),
        quality=0.3
    )
    
    # Level 4: 返回建议 (质量 0.1)
    strategy.add_level(
        "suggestion_only",
        lambda: return_search_suggestions(query),
        quality=0.1
    )
    
    result, level_used, quality = await strategy.execute()
    
    if quality < 1.0:
        # 向用户说明降级情况
        result['_degradation_notice'] = f"使用 {level_used} (质量: {quality:.0%})"
        
    return result
```

### 模式 4：Circuit Breaker（熔断器）

```typescript
// pi-mono 风格

enum CircuitState {
  CLOSED = 'closed',     // 正常
  OPEN = 'open',         // 熔断
  HALF_OPEN = 'half_open' // 试探
}

interface CircuitBreakerConfig {
  failureThreshold: number;    // 失败阈值
  recoveryTimeout: number;     // 恢复超时(ms)
  halfOpenRequests: number;    // 半开状态允许的请求数
}

class CircuitBreaker {
  private state: CircuitState = CircuitState.CLOSED;
  private failures: number = 0;
  private lastFailure: number = 0;
  private halfOpenAllowed: number = 0;
  
  constructor(private config: CircuitBreakerConfig) {}
  
  async execute<T>(fn: () => Promise<T>): Promise<T> {
    // 检查熔断状态
    if (this.state === CircuitState.OPEN) {
      // 检查是否可以进入半开状态
      if (Date.now() - this.lastFailure > this.config.recoveryTimeout) {
        this.state = CircuitState.HALF_OPEN;
        this.halfOpenAllowed = this.config.halfOpenRequests;
      } else {
        throw new Error('Circuit breaker is OPEN');
      }
    }
    
    if (this.state === CircuitState.HALF_OPEN) {
      if (this.halfOpenAllowed <= 0) {
        throw new Error('Circuit breaker HALF_OPEN limit reached');
      }
      this.halfOpenAllowed--;
    }
    
    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }
  
  private onSuccess() {
    this.failures = 0;
    if (this.state === CircuitState.HALF_OPEN) {
      this.state = CircuitState.CLOSED;
      console.log('Circuit breaker recovered: CLOSED');
    }
  }
  
  private onFailure() {
    this.failures++;
    this.lastFailure = Date.now();
    
    if (this.failures >= this.config.failureThreshold) {
      this.state = CircuitState.OPEN;
      console.log('Circuit breaker tripped: OPEN');
    }
  }
  
  getState(): CircuitState {
    return this.state;
  }
}

// 在 Agent Tool 中使用
class ExternalAPITool {
  private circuitBreaker = new CircuitBreaker({
    failureThreshold: 3,
    recoveryTimeout: 30000,  // 30秒后尝试恢复
    halfOpenRequests: 2
  });
  
  async callAPI(params: any): Promise<any> {
    try {
      return await this.circuitBreaker.execute(() => 
        this.actualAPICall(params)
      );
    } catch (error) {
      if (error.message.includes('Circuit breaker')) {
        // 熔断状态，使用降级响应
        return this.getDegradedResponse(params);
      }
      throw error;
    }
  }
  
  private getDegradedResponse(params: any) {
    return {
      status: 'degraded',
      message: '服务暂时不可用，已使用缓存数据',
      data: this.getCachedData(params)
    };
  }
}
```

### 模式 5：Agent 级别的降级决策

```python
# Agent 根据可用服务动态调整能力

class DegradedAgent:
    def __init__(self, service_registry: ServiceRegistry):
        self.registry = service_registry
        
    def get_available_capabilities(self) -> dict:
        """获取当前可用的能力"""
        capabilities = {
            'web_search': self.registry.is_available('brave_search'),
            'code_execution': self.registry.is_available('sandbox'),
            'file_operations': True,  # 本地总是可用
            'image_generation': self.registry.is_available('dalle'),
            'weather': self.registry.is_available('weather_api'),
        }
        return capabilities
        
    def build_system_prompt(self) -> str:
        """根据可用能力动态构建系统提示"""
        caps = self.get_available_capabilities()
        
        base_prompt = "You are a helpful assistant."
        
        if not caps['web_search']:
            base_prompt += """

⚠️ Web search is currently unavailable. 
- Use your knowledge (cutoff: 2024-01)
- Suggest user try again later for real-time info
"""
            
        if not caps['code_execution']:
            base_prompt += """

⚠️ Code execution sandbox is unavailable.
- You can still write code for user to run locally
- Explain what the code would do instead of executing
"""
            
        if not caps['image_generation']:
            base_prompt += """

⚠️ Image generation is unavailable.
- Describe images in detail using text
- Suggest ASCII art alternatives for simple diagrams
"""
            
        return base_prompt
        
    async def process_request(self, user_message: str) -> str:
        # 动态系统提示
        system_prompt = self.build_system_prompt()
        
        # 过滤不可用的工具
        available_tools = self.filter_available_tools()
        
        # 调用 LLM
        response = await self.llm.chat(
            system=system_prompt,
            messages=[{"role": "user", "content": user_message}],
            tools=available_tools
        )
        
        return response
        
    def filter_available_tools(self) -> list:
        """只返回可用的工具"""
        all_tools = self.get_all_tools()
        caps = self.get_available_capabilities()
        
        return [
            tool for tool in all_tools
            if self.tool_dependency_met(tool, caps)
        ]
```

## OpenClaw 实战：优雅降级配置

```yaml
# openclaw.yaml - 降级配置示例

degradation:
  # 全局降级策略
  strategy: "graceful"  # graceful | fail-fast | retry-only
  
  # 服务健康检查
  health_check:
    interval: 60  # 秒
    timeout: 10   # 秒
    
  # 各服务降级配置
  services:
    web_search:
      fallback: "knowledge_only"
      cache_ttl: 3600
      circuit_breaker:
        threshold: 3
        recovery: 60
        
    browser:
      fallback: "web_fetch"
      retry: 2
      
    tts:
      fallback: "text_only"
      skip_on_failure: true
      
  # 降级通知
  notifications:
    enabled: true
    channel: "telegram"
    threshold: "degraded"  # degraded | unavailable
```

## 最佳实践

### 1. 识别关键路径

```python
# 标记哪些功能是关键路径
CRITICAL_TOOLS = {'read', 'write', 'exec'}  # 关键工具
OPTIONAL_TOOLS = {'web_search', 'tts', 'image'}  # 可选工具

def should_fail_on_error(tool_name: str) -> bool:
    return tool_name in CRITICAL_TOOLS
```

### 2. 用户感知的降级

```python
# 好的做法：告知用户降级情况
response = {
    "status": "partial",
    "result": cached_weather_data,
    "notice": "⚠️ 使用1小时前的缓存数据，实时天气服务暂不可用"
}

# 不好的做法：静默返回可能过期的数据
response = cached_weather_data  # 用户不知道数据可能已过期
```

### 3. 渐进式降级

```
Full Service → Reduced Features → Cached Data → Manual Fallback → Graceful Error

示例：
1. 完整天气预报（实时API）
2. 简化天气信息（去掉小时预报）
3. 缓存的天气数据（标注时间）
4. 建议用户访问天气网站
5. 友好的错误信息
```

### 4. 降级监控

```python
class DegradationMetrics:
    def __init__(self):
        self.degradation_events = []
        
    def record(self, service: str, original_level: str, degraded_level: str):
        self.degradation_events.append({
            'timestamp': time.time(),
            'service': service,
            'from': original_level,
            'to': degraded_level
        })
        
    def get_health_score(self) -> float:
        """计算整体健康分数"""
        recent = [e for e in self.degradation_events 
                  if time.time() - e['timestamp'] < 3600]
        if not recent:
            return 1.0
        return max(0.0, 1.0 - len(recent) * 0.1)
```

## 总结

优雅降级的核心原则：

1. **预期失败**：假设任何外部依赖都可能失败
2. **多级策略**：准备多个降级方案，逐级尝试
3. **透明通知**：向用户说明降级情况
4. **快速恢复**：监控服务恢复，及时恢复全功能
5. **关键保护**：区分关键和非关键功能

```
┌──────────────────────────────────────────┐
│        Graceful Degradation              │
│                                          │
│   "Failing gracefully is better than     │
│    failing silently or crashing loudly"  │
│                                          │
│   宁可优雅地降级，也不要悄无声息地失败     │
│   或者响亮地崩溃                          │
└──────────────────────────────────────────┘
```

## 下一步

- 探索 Feature Flags 与降级的结合
- 学习 Chaos Engineering 测试降级策略
- 研究 Multi-Region 部署的降级方案

---

*Agent 开发课程 - 第 28 课*
