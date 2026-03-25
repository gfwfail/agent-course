# 139 - Agent 工具结果冲突检测与仲裁（Tool Result Conflict Detection & Arbitration）

> 多工具、多数据源是现代 Agent 的标配——但当它们给出矛盾答案时，Agent 该听谁的？

---

## 为什么需要冲突仲裁？

真实 Agent 往往同时调用多个数据源：

```
用户问：「苹果公司现在的 CEO 是谁？」

工具 A（RAG 知识库）→ "蒂姆·库克"
工具 B（网络搜索）→ "苹果 CEO 蒂姆·库克在2025年宣布退休..."
工具 C（公司数据库）→ "CEO: Tim Cook, 任期至2024-12-31"
```

三个来源，三个不同程度的"正确"——Agent 必须检测冲突、评估可信度、做出决策，而不是随机采用第一个结果。

---

## 冲突的三种类型

| 类型 | 示例 | 处理策略 |
|------|------|----------|
| **事实冲突** | A说"在任"，B说"已离职" | 时效性仲裁（优先最新） |
| **数值冲突** | A说"营收1000亿"，B说"950亿" | 来源权威性 + 置信区间 |
| **语义冲突** | A说"暂停计划"，B说"取消计划" | 语义相似度 + 人工升级 |

---

## 核心架构：四层仲裁管道

```
┌──────────────────────────────────────────────┐
│  Layer 1: 结果标准化（Normalization）          │
│  Layer 2: 冲突检测（Conflict Detection）       │
│  Layer 3: 可信度评分（Credibility Scoring）    │
│  Layer 4: 仲裁决策（Arbitration）              │
└──────────────────────────────────────────────┘
```

---

## TypeScript 实现

### 1. 核心数据结构

```typescript
interface ToolResult {
  toolName: string;
  data: unknown;
  metadata: {
    timestamp: number;      // 数据产生时间
    sourceAuthority: number; // 0-1，来源权威性
    freshness: number;       // 0-1，数据新鲜度
    confidence: number;      // 0-1，工具自身置信度
  };
}

interface ConflictReport {
  field: string;
  values: Array<{ source: string; value: unknown; score: number }>;
  resolution: "consensus" | "authority" | "freshness" | "escalate";
  winner: ToolResult | null;
  explanation: string;
}

interface ArbitrationResult {
  mergedData: Record<string, unknown>;
  conflicts: ConflictReport[];
  overallConfidence: number;
  requiresHumanReview: boolean;
}
```

### 2. 冲突检测器

```typescript
class ConflictDetector {
  // 数值字段允许±5%误差
  private static NUMERIC_TOLERANCE = 0.05;
  
  detectConflicts(
    results: ToolResult[],
    schema: Record<string, "string" | "number" | "boolean">
  ): Map<string, ToolResult[]> {
    const conflicts = new Map<string, ToolResult[]>();
    
    for (const [field, type] of Object.entries(schema)) {
      const values = results.map(r => ({
        result: r,
        value: (r.data as Record<string, unknown>)[field],
      })).filter(v => v.value !== undefined && v.value !== null);
      
      if (values.length < 2) continue;
      
      const hasConflict = type === "number"
        ? this.hasNumericConflict(values.map(v => v.value as number))
        : this.hasValueConflict(values.map(v => v.value));
      
      if (hasConflict) {
        conflicts.set(field, values.map(v => v.result));
      }
    }
    
    return conflicts;
  }
  
  private hasNumericConflict(values: number[]): boolean {
    const min = Math.min(...values);
    const max = Math.max(...values);
    if (min === 0) return max !== 0;
    return (max - min) / min > ConflictDetector.NUMERIC_TOLERANCE;
  }
  
  private hasValueConflict(values: unknown[]): boolean {
    const unique = new Set(values.map(v => JSON.stringify(v)));
    return unique.size > 1;
  }
}
```

### 3. 可信度评分引擎

```typescript
class CredibilityScorer {
  score(result: ToolResult, field: string): number {
    const { timestamp, sourceAuthority, freshness, confidence } = result.metadata;
    
    // 时效性衰减：数据越老，分数越低
    const ageHours = (Date.now() - timestamp) / (1000 * 60 * 60);
    const timeDecay = Math.exp(-ageHours / 24); // 24小时半衰期
    
    // 综合评分：权威性40% + 新鲜度30% + 时效衰减20% + 工具置信度10%
    return (
      sourceAuthority * 0.4 +
      freshness * 0.3 +
      timeDecay * 0.2 +
      confidence * 0.1
    );
  }
  
  rankResults(results: ToolResult[], field: string): Array<{result: ToolResult; score: number}> {
    return results
      .map(r => ({ result: r, score: this.score(r, field) }))
      .sort((a, b) => b.score - a.score);
  }
}
```

### 4. 仲裁器（核心）

```typescript
class ConflictArbitrator {
  private detector = new ConflictDetector();
  private scorer = new CredibilityScorer();
  
  // 分数差异小于此阈值则升级到人工处理
  private ESCALATION_THRESHOLD = 0.15;
  
  async arbitrate(
    results: ToolResult[],
    schema: Record<string, "string" | "number" | "boolean">
  ): Promise<ArbitrationResult> {
    const conflicts = this.detector.detectConflicts(results, schema);
    const conflictReports: ConflictReport[] = [];
    const mergedData: Record<string, unknown> = {};
    
    // 无冲突字段：取第一个有值的结果
    for (const field of Object.keys(schema)) {
      if (!conflicts.has(field)) {
        for (const r of results) {
          const value = (r.data as Record<string, unknown>)[field];
          if (value !== undefined && value !== null) {
            mergedData[field] = value;
            break;
          }
        }
      }
    }
    
    // 有冲突字段：仲裁
    for (const [field, conflictingResults] of conflicts) {
      const ranked = this.scorer.rankResults(conflictingResults, field);
      const topScore = ranked[0].score;
      const secondScore = ranked[1]?.score ?? 0;
      const scoreDiff = topScore - secondScore;
      
      let resolution: ConflictReport["resolution"];
      let winner: ToolResult | null;
      let explanation: string;
      
      if (scoreDiff < this.ESCALATION_THRESHOLD) {
        // 分数太接近，无法自动决策
        resolution = "escalate";
        winner = null;
        explanation = `分数差异仅 ${(scoreDiff * 100).toFixed(1)}%，需要人工判断`;
      } else if (ranked[0].result.metadata.freshness > 0.8) {
        resolution = "freshness";
        winner = ranked[0].result;
        explanation = `选择最新数据源 ${ranked[0].result.toolName}（新鲜度 ${(ranked[0].result.metadata.freshness * 100).toFixed(0)}%）`;
      } else {
        resolution = "authority";
        winner = ranked[0].result;
        explanation = `选择最高权威来源 ${ranked[0].result.toolName}（评分 ${(topScore * 100).toFixed(1)}）`;
      }
      
      if (winner) {
        mergedData[field] = (winner.data as Record<string, unknown>)[field];
      }
      
      conflictReports.push({
        field,
        values: ranked.map(r => ({
          source: r.result.toolName,
          value: (r.result.data as Record<string, unknown>)[field],
          score: r.score,
        })),
        resolution,
        winner,
        explanation,
      });
    }
    
    const escalateCount = conflictReports.filter(r => r.resolution === "escalate").length;
    const overallConfidence = 1 - (escalateCount / Math.max(conflictReports.length, 1)) * 0.5;
    
    return {
      mergedData,
      conflicts: conflictReports,
      overallConfidence,
      requiresHumanReview: escalateCount > 0,
    };
  }
}
```

### 5. 集成到 Agent Loop

```typescript
class MultiSourceAgent {
  private arbitrator = new ConflictArbitrator();
  
  async query(question: string): Promise<string> {
    // 并行调用多个工具
    const [ragResult, searchResult, dbResult] = await Promise.all([
      this.callRAG(question),
      this.callWebSearch(question),
      this.callDatabase(question),
    ]);
    
    const results = [ragResult, searchResult, dbResult].filter(Boolean) as ToolResult[];
    
    // 仲裁冲突
    const arbitration = await this.arbitrator.arbitrate(results, {
      ceo: "string",
      revenue: "number",
      founded: "number",
    });
    
    // 构建透明的 LLM 提示词
    let prompt = `基于以下仲裁后的数据回答问题：\n${JSON.stringify(arbitration.mergedData)}\n`;
    
    if (arbitration.conflicts.length > 0) {
      prompt += `\n注意：以下字段存在数据冲突：\n`;
      for (const conflict of arbitration.conflicts) {
        prompt += `- ${conflict.field}：${conflict.explanation}\n`;
        if (conflict.resolution === "escalate") {
          prompt += `  （此字段数据存在争议，请在回答中说明不确定性）\n`;
        }
      }
    }
    
    if (arbitration.requiresHumanReview) {
      prompt += `\n整体置信度：${(arbitration.overallConfidence * 100).toFixed(0)}%，建议用户确认关键信息。`;
    }
    
    return this.callLLM(prompt + `\n\n用户问题：${question}`);
  }
  
  private async callRAG(q: string): Promise<ToolResult> {
    const data = await this.ragTool(q);
    return {
      toolName: "rag_knowledge_base",
      data,
      metadata: {
        timestamp: Date.now() - 7 * 24 * 60 * 60 * 1000, // 假设知识库7天前更新
        sourceAuthority: 0.85, // 内部权威知识库
        freshness: 0.3,         // 数据较旧
        confidence: 0.9,
      },
    };
  }
  
  private async callWebSearch(q: string): Promise<ToolResult> {
    const data = await this.searchTool(q);
    return {
      toolName: "web_search",
      data,
      metadata: {
        timestamp: Date.now() - 60 * 60 * 1000, // 1小时内
        sourceAuthority: 0.6,  // 网络来源权威性中等
        freshness: 0.95,        // 非常新鲜
        confidence: 0.7,
      },
    };
  }
  
  // callDatabase、callLLM、ragTool、searchTool 略...
}
```

---

## OpenClaw 实战：多数据源查询

OpenClaw 的 `mysterybox` 技能在查询平台数据时会同时命中 Grafana #1（AWS/SkinSEgg）和 Grafana #2（Prod/SkinsManage）两个数据源，当指标数据不一致时正好需要此模式：

```typescript
// 在 skill 的查询层加冲突检测
async function queryWithArbitration(sql: string) {
  const [result1, result2] = await Promise.all([
    grafanaQuery(GRAFANA_URL_1, GRAFANA_KEY_1, sql),
    grafanaQuery(GRAFANA_URL_2, GRAFANA_KEY_2, sql),
  ]);
  
  const arbitration = await arbitrator.arbitrate(
    [
      toToolResult("grafana_aws", result1, { authority: 0.7, freshness: 0.8 }),
      toToolResult("grafana_prod", result2, { authority: 0.9, freshness: 0.9 }),
    ],
    inferSchema(sql)
  );
  
  if (arbitration.requiresHumanReview) {
    return `⚠️ 数据存在冲突，建议人工核查：\n${formatConflicts(arbitration.conflicts)}`;
  }
  
  return formatData(arbitration.mergedData);
}
```

---

## Python 实现（pi-mono 风格）

```python
from dataclasses import dataclass, field
from typing import Any, Literal
import time
import math

@dataclass
class ToolResult:
    tool_name: str
    data: dict[str, Any]
    authority: float  # 0-1
    freshness: float  # 0-1
    timestamp: float = field(default_factory=time.time)
    confidence: float = 0.8

class ConflictArbitrator:
    ESCALATION_THRESHOLD = 0.15
    
    def score(self, result: ToolResult) -> float:
        age_hours = (time.time() - result.timestamp) / 3600
        time_decay = math.exp(-age_hours / 24)
        return (result.authority * 0.4 + result.freshness * 0.3 + 
                time_decay * 0.2 + result.confidence * 0.1)
    
    def arbitrate(self, results: list[ToolResult], field: str) -> dict:
        values = [(r, r.data.get(field)) for r in results if r.data.get(field) is not None]
        if not values:
            return {"value": None, "resolution": "no_data"}
        
        # 检测冲突
        unique_values = set(str(v) for _, v in values)
        if len(unique_values) == 1:
            return {"value": values[0][1], "resolution": "consensus"}
        
        # 有冲突，按评分排序
        ranked = sorted(values, key=lambda x: self.score(x[0]), reverse=True)
        top_score = self.score(ranked[0][0])
        second_score = self.score(ranked[1][0]) if len(ranked) > 1 else 0
        
        if top_score - second_score < self.ESCALATION_THRESHOLD:
            return {
                "value": None,
                "resolution": "escalate",
                "candidates": [(r.tool_name, v, self.score(r)) for r, v in ranked],
            }
        
        return {
            "value": ranked[0][1],
            "resolution": "authority" if ranked[0][0].authority > 0.7 else "freshness",
            "winner": ranked[0][0].tool_name,
            "score": top_score,
        }
```

---

## 三种仲裁策略对比

| 策略 | 适用场景 | 优点 | 缺点 |
|------|----------|------|------|
| **权威优先** | 内部权威 DB vs 外部搜索 | 稳定、可预测 | 权威数据可能过时 |
| **时效优先** | 实时价格、股票数据 | 始终最新 | 可能被噪音误导 |
| **多数投票** | 3+ 来源的事实查询 | 鲁棒性强 | 需要奇数来源 |
| **LLM 仲裁** | 语义冲突、无法量化时 | 理解语义 | 额外 LLM 调用成本 |

---

## 透明度：把冲突告诉用户

仲裁结果必须透明，用户有权知道数据存在争议：

```
✅ 根据最新网络数据（2025-03-26），苹果公司现任 CEO 为 XXX。
⚠️ 注意：知识库显示为 Tim Cook，数据时间为2024-12，可能存在滞后。
建议：如需准确信息请访问苹果官网确认。
```

---

## 关键设计原则

1. **永远不要静默丢弃冲突** — 检测到就报告，让 LLM 知道不确定性
2. **评分要可解释** — 记录为什么选择某个来源，便于调试
3. **设置升级阈值** — 分数相近时宁可问人，不要硬猜
4. **时效性权重动态调整** — 金融数据 1 小时半衰期 vs 历史数据 1 年半衰期
5. **冲突率是健康指标** — 高冲突率说明数据源质量需要审查

---

## 总结

| 步骤 | 关键问题 |
|------|----------|
| 标准化 | 把不同格式的工具结果统一为可比较的结构 |
| 检测 | 同字段不同值 → 冲突；数值在容差内 → 正常 |
| 评分 | 权威性 + 新鲜度 + 时效衰减 + 工具置信度 |
| 仲裁 | 分数差异大 → 自动选择；差异小 → 人工升级 |
| 透明 | 告诉 LLM 和用户哪些字段存在争议 |

多工具查询不是简单的"谁先回来用谁"，而是一个需要主动管理的数据质量问题。

---

*下一课预告：Agent 工具调用热路径优化（Hot Path Optimization）——识别高频调用的工具，针对性优化执行路径*
