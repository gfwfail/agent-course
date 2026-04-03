# 209 - Agent 数据管道与 ETL 模式（Data Pipeline & ETL Patterns）

> **核心思想：** Agent 天生就是最聪明的 ETL 协调器——它能理解数据语义、处理异常、动态调整管道策略。把 Extract→Transform→Load 三阶段交给 Agent 管，数据工程从此有了"大脑"。

---

## 一、为什么 Agent 适合做 ETL？

传统 ETL 工具（Airflow/dbt/Spark）擅长**规则明确、结构固定**的数据处理。但现实数据往往：

- 格式不一致（API 返回 camelCase，数据库要 snake_case，PDF 要手工解析）
- Schema 动态变化（供应商突然改了字段名）
- 异常需要语义理解（"N/A" vs `null` vs `""`，三个意思不同）
- 数据来源需要鉴权协商（OAuth token 过期怎么刷？）

Agent 的优势：**用 LLM 理解语义 + 工具执行 I/O + 循环重试处理异常**，正好填补这个空白。

```
传统 ETL:  Source → 固定规则 → Sink
Agent ETL: Source → LLM 理解 → 动态 Transform → Sink
                ↑_________错误处理/自适应________↑
```

---

## 二、三阶段设计

### 2.1 Extract（提取）

从多个异构数据源抓取原始数据：

```typescript
// TypeScript - ExtractTool 工厂
import Anthropic from "@anthropic-ai/sdk";

interface ExtractSource {
  type: "api" | "file" | "database" | "web";
  config: Record<string, unknown>;
}

interface RawRecord {
  sourceId: string;
  fetchedAt: number;
  raw: unknown;
  error?: string;
}

// 工具定义 - 让 Agent 调用
const extractTools: Anthropic.Tool[] = [
  {
    name: "fetch_api",
    description: "从 HTTP API 获取数据，自动处理分页和重试",
    input_schema: {
      type: "object" as const,
      properties: {
        url: { type: "string" },
        headers: { type: "object" },
        pagination: {
          type: "object",
          properties: {
            type: { type: "string", enum: ["cursor", "offset", "page", "none"] },
            pageSize: { type: "number" },
          },
        },
      },
      required: ["url"],
    },
  },
  {
    name: "read_file",
    description: "读取本地文件，支持 CSV/JSON/JSONL/Parquet",
    input_schema: {
      type: "object" as const,
      properties: {
        path: { type: "string" },
        format: { type: "string", enum: ["csv", "json", "jsonl", "auto"] },
        encoding: { type: "string" },
      },
      required: ["path"],
    },
  },
  {
    name: "query_database",
    description: "执行 SQL 查询，返回结构化结果",
    input_schema: {
      type: "object" as const,
      properties: {
        dsn: { type: "string" },
        sql: { type: "string" },
        params: { type: "array" },
      },
      required: ["dsn", "sql"],
    },
  },
];

// 真实的工具执行器
async function executeExtractTool(
  name: string,
  input: Record<string, unknown>
): Promise<RawRecord> {
  const fetchedAt = Date.now();

  switch (name) {
    case "fetch_api": {
      const url = input.url as string;
      const headers = (input.headers as Record<string, string>) || {};
      const pagination = input.pagination as
        | { type: string; pageSize?: number }
        | undefined;

      const allData: unknown[] = [];
      let nextCursor: string | null = null;
      let page = 1;

      do {
        const requestUrl = new URL(url);
        if (pagination?.type === "cursor" && nextCursor) {
          requestUrl.searchParams.set("cursor", nextCursor);
        } else if (pagination?.type === "page") {
          requestUrl.searchParams.set("page", String(page));
          if (pagination.pageSize) {
            requestUrl.searchParams.set(
              "per_page",
              String(pagination.pageSize)
            );
          }
        }

        const response = await fetch(requestUrl.toString(), { headers });
        if (!response.ok) {
          throw new Error(`HTTP ${response.status}: ${await response.text()}`);
        }

        const data = await response.json();

        // 处理常见分页格式
        if (Array.isArray(data)) {
          allData.push(...data);
          break; // 无分页
        } else if (data.data && Array.isArray(data.data)) {
          allData.push(...data.data);
          nextCursor = data.next_cursor || data.meta?.next_cursor || null;
        } else if (data.items && Array.isArray(data.items)) {
          allData.push(...data.items);
          nextCursor = data.next_page_token || null;
        } else {
          allData.push(data);
          break;
        }

        page++;
        if (!nextCursor && pagination?.type !== "page") break;
        if (pagination?.type === "page" && allData.length % (pagination.pageSize || 100) !== 0) break;
      } while (nextCursor || (pagination?.type === "page" && page <= 100));

      return {
        sourceId: url,
        fetchedAt,
        raw: allData.length === 1 ? allData[0] : allData,
      };
    }

    case "read_file": {
      const fs = await import("fs/promises");
      const path = input.path as string;
      const content = await fs.readFile(path, "utf-8");
      const format = (input.format as string) || "auto";

      let parsed: unknown;
      if (format === "json" || (format === "auto" && path.endsWith(".json"))) {
        parsed = JSON.parse(content);
      } else if (format === "jsonl" || path.endsWith(".jsonl")) {
        parsed = content
          .trim()
          .split("\n")
          .map((l) => JSON.parse(l));
      } else if (format === "csv" || path.endsWith(".csv")) {
        // 简单 CSV 解析（生产用 papaparse）
        const lines = content.trim().split("\n");
        const headers = lines[0].split(",").map((h) => h.trim());
        parsed = lines.slice(1).map((line) => {
          const values = line.split(",");
          return Object.fromEntries(
            headers.map((h, i) => [h, values[i]?.trim()])
          );
        });
      } else {
        parsed = content;
      }

      return { sourceId: path, fetchedAt, raw: parsed };
    }

    default:
      return {
        sourceId: name,
        fetchedAt,
        raw: null,
        error: `Unknown tool: ${name}`,
      };
  }
}
```

### 2.2 Transform（转换）

这是 Agent ETL 最有价值的地方——LLM 理解语义：

```typescript
interface TransformConfig {
  targetSchema: Record<string, string>; // 目标字段 -> 类型描述
  rules?: string[]; // 额外业务规则
  cleanNulls?: boolean;
}

interface TransformedRecord {
  data: Record<string, unknown>;
  warnings: string[];
  skipped: boolean;
  skipReason?: string;
}

// 批量转换 - 减少 LLM 调用次数
async function batchTransform(
  client: Anthropic,
  records: unknown[],
  config: TransformConfig,
  batchSize = 20
): Promise<TransformedRecord[]> {
  const results: TransformedRecord[] = [];

  for (let i = 0; i < records.length; i += batchSize) {
    const batch = records.slice(i, i + batchSize);

    const schemaDesc = Object.entries(config.targetSchema)
      .map(([field, desc]) => `- ${field}: ${desc}`)
      .join("\n");

    const rulesDesc = config.rules?.join("\n") || "";

    const response = await client.messages.create({
      model: "claude-haiku-4-5", // 用便宜模型做转换
      max_tokens: 4096,
      system: `你是数据转换专家。将输入数据转换为目标 Schema，返回 JSON 数组。

目标 Schema：
${schemaDesc}

${rulesDesc ? `业务规则：\n${rulesDesc}` : ""}

规则：
1. 严格输出 JSON 数组，不要任何说明文字
2. 每个元素包含: { data: {...}, warnings: [...], skipped: bool, skipReason?: string }
3. 无法映射的字段填 null，在 warnings 中说明
4. 完全无法转换的记录设 skipped: true`,
      messages: [
        {
          role: "user",
          content: `转换以下 ${batch.length} 条记录：\n\`\`\`json\n${JSON.stringify(batch, null, 2)}\n\`\`\``,
        },
      ],
    });

    const content = response.content[0];
    if (content.type !== "text") continue;

    // 提取 JSON（LLM 有时会加 markdown 代码块）
    const jsonMatch = content.text.match(/\[[\s\S]*\]/);
    if (!jsonMatch) continue;

    const transformed = JSON.parse(jsonMatch[0]) as TransformedRecord[];
    results.push(...transformed);

    console.log(
      `转换进度: ${Math.min(i + batchSize, records.length)}/${records.length}`
    );
  }

  return results;
}
```

### 2.3 Load（加载）

带幂等性保障的写入：

```typescript
interface LoadTarget {
  type: "sqlite" | "postgres" | "json_file" | "api";
  config: Record<string, unknown>;
}

interface LoadResult {
  inserted: number;
  updated: number;
  skipped: number;
  errors: string[];
}

// SQLite 加载（适合本地 Agent 存储）
import Database from "better-sqlite3";

function loadToSQLite(
  records: TransformedRecord[],
  dbPath: string,
  tableName: string,
  primaryKey = "id"
): LoadResult {
  const db = new Database(dbPath);
  const result: LoadResult = {
    inserted: 0,
    updated: 0,
    skipped: 0,
    errors: [],
  };

  // 动态建表（如果不存在）
  const validRecords = records.filter((r) => !r.skipped && r.data);
  if (validRecords.length === 0)
    return { ...result, skipped: records.length };

  const columns = Object.keys(validRecords[0].data);
  const createSQL = `
    CREATE TABLE IF NOT EXISTS ${tableName} (
      ${columns.map((c) => `${c} TEXT`).join(", ")},
      _loaded_at INTEGER DEFAULT (unixepoch()),
      _warnings TEXT
    )
  `;
  db.exec(createSQL);

  // UPSERT（幂等性保障）
  const upsertSQL = `
    INSERT INTO ${tableName} (${columns.join(", ")}, _warnings)
    VALUES (${columns.map(() => "?").join(", ")}, ?)
    ON CONFLICT(${primaryKey}) DO UPDATE SET
    ${columns.filter((c) => c !== primaryKey).map((c) => `${c} = excluded.${c}`).join(", ")},
    _loaded_at = unixepoch(),
    _warnings = excluded._warnings
  `;

  const upsert = db.prepare(upsertSQL);
  const upsertMany = db.transaction((recs: TransformedRecord[]) => {
    for (const rec of recs) {
      if (rec.skipped) {
        result.skipped++;
        continue;
      }
      try {
        const values = columns.map((c) =>
          rec.data[c] !== null && rec.data[c] !== undefined
            ? String(rec.data[c])
            : null
        );
        values.push(rec.warnings.join("; ") || null);
        const info = upsert.run(...values);
        if (info.changes > 0) result.inserted++;
        else result.updated++;
      } catch (e) {
        result.errors.push(String(e));
      }
    }
  });

  upsertMany(records);
  db.close();
  return result;
}
```

---

## 三、完整 Agent ETL 编排器

把三个阶段串起来，加上状态追踪和断点续传：

```typescript
// agent-etl-orchestrator.ts
import Anthropic from "@anthropic-ai/sdk";
import * as fs from "fs/promises";

interface ETLJob {
  id: string;
  name: string;
  sources: ExtractSource[];
  transform: TransformConfig;
  target: LoadTarget;
  schedule?: string; // cron 表达式
}

interface ETLState {
  jobId: string;
  status: "pending" | "extracting" | "transforming" | "loading" | "done" | "failed";
  checkpoint?: {
    phase: string;
    processedSources: string[];
    transformedCount: number;
  };
  startedAt: number;
  completedAt?: number;
  error?: string;
  metrics: {
    extracted: number;
    transformed: number;
    loaded: number;
    skipped: number;
    errors: number;
  };
}

class AgentETLOrchestrator {
  private client: Anthropic;
  private stateFile: string;

  constructor(stateFile = "/tmp/etl-state.json") {
    this.client = new Anthropic();
    this.stateFile = stateFile;
  }

  async run(job: ETLJob): Promise<ETLState> {
    // 加载或初始化状态（断点续传）
    let state = await this.loadState(job.id) || this.initState(job.id);

    console.log(`\n🚀 ETL Job: ${job.name} [${job.id}]`);
    console.log(`Status: ${state.status}`);

    try {
      // Phase 1: Extract
      if (!["transforming", "loading", "done"].includes(state.status)) {
        state.status = "extracting";
        await this.saveState(state);

        console.log("\n📥 阶段 1: Extract");
        const allRaw: unknown[] = [];

        for (const source of job.sources) {
          const sourceId = JSON.stringify(source.config).slice(0, 50);
          if (state.checkpoint?.processedSources.includes(sourceId)) {
            console.log(`  ↩️  跳过已处理的源: ${sourceId}`);
            continue;
          }

          // Agent 决定如何提取这个源
          const rawRecords = await this.agentExtract(source);
          allRaw.push(...rawRecords);
          state.metrics.extracted += rawRecords.length;
          state.checkpoint = {
            ...state.checkpoint,
            phase: "extract",
            processedSources: [
              ...(state.checkpoint?.processedSources || []),
              sourceId,
            ],
            transformedCount: state.checkpoint?.transformedCount || 0,
          };
          await this.saveState(state);
          console.log(`  ✅ 提取 ${rawRecords.length} 条: ${sourceId}`);
        }

        // 缓存原始数据到磁盘（防止 OOM + 支持重试）
        await fs.writeFile(
          `/tmp/etl-${job.id}-raw.json`,
          JSON.stringify(allRaw)
        );
      }

      // Phase 2: Transform
      if (!["loading", "done"].includes(state.status)) {
        state.status = "transforming";
        await this.saveState(state);

        console.log("\n🔄 阶段 2: Transform");
        const rawData = JSON.parse(
          await fs.readFile(`/tmp/etl-${job.id}-raw.json`, "utf-8")
        ) as unknown[];

        const transformed = await batchTransform(
          this.client,
          rawData,
          job.transform
        );

        state.metrics.transformed = transformed.filter((r) => !r.skipped).length;
        state.metrics.skipped = transformed.filter((r) => r.skipped).length;

        await fs.writeFile(
          `/tmp/etl-${job.id}-transformed.json`,
          JSON.stringify(transformed)
        );
        console.log(
          `  ✅ 转换完成: ${state.metrics.transformed} 条, 跳过: ${state.metrics.skipped} 条`
        );
      }

      // Phase 3: Load
      if (state.status !== "done") {
        state.status = "loading";
        await this.saveState(state);

        console.log("\n📤 阶段 3: Load");
        const transformed = JSON.parse(
          await fs.readFile(
            `/tmp/etl-${job.id}-transformed.json`,
            "utf-8"
          )
        ) as TransformedRecord[];

        let loadResult: LoadResult;
        if (job.target.type === "sqlite") {
          loadResult = loadToSQLite(
            transformed,
            job.target.config.path as string,
            job.target.config.table as string,
            job.target.config.primaryKey as string
          );
        } else {
          throw new Error(`Unsupported target type: ${job.target.type}`);
        }

        state.metrics.loaded = loadResult.inserted + loadResult.updated;
        state.metrics.errors = loadResult.errors.length;
        console.log(
          `  ✅ 写入完成: 新增 ${loadResult.inserted}, 更新 ${loadResult.updated}`
        );
        if (loadResult.errors.length > 0) {
          console.log(`  ⚠️  错误: ${loadResult.errors.slice(0, 3).join("; ")}`);
        }
      }

      // 完成
      state.status = "done";
      state.completedAt = Date.now();
      await this.saveState(state);

      // 清理临时文件
      await fs.unlink(`/tmp/etl-${job.id}-raw.json`).catch(() => {});
      await fs.unlink(`/tmp/etl-${job.id}-transformed.json`).catch(() => {});

      console.log(`\n✅ ETL 完成！耗时: ${(Date.now() - state.startedAt) / 1000}s`);
      console.log(`   提取: ${state.metrics.extracted} | 转换: ${state.metrics.transformed} | 写入: ${state.metrics.loaded}`);

      return state;
    } catch (error) {
      state.status = "failed";
      state.error = String(error);
      await this.saveState(state);
      throw error;
    }
  }

  // Agent 决策：如何提取这个数据源
  private async agentExtract(source: ExtractSource): Promise<unknown[]> {
    const messages: Anthropic.MessageParam[] = [
      {
        role: "user",
        content: `提取以下数据源的数据：\n\`\`\`json\n${JSON.stringify(source, null, 2)}\n\`\`\`\n请选择合适的工具并执行提取。`,
      },
    ];

    let allData: unknown[] = [];

    while (true) {
      const response = await this.client.messages.create({
        model: "claude-haiku-4-5",
        max_tokens: 2048,
        tools: extractTools,
        messages,
      });

      if (response.stop_reason === "end_turn") break;
      if (response.stop_reason !== "tool_use") break;

      messages.push({ role: "assistant", content: response.content });

      const toolResults: Anthropic.ToolResultBlockParam[] = [];
      for (const block of response.content) {
        if (block.type !== "tool_use") continue;
        const result = await executeExtractTool(
          block.name,
          block.input as Record<string, unknown>
        );
        if (result.error) {
          toolResults.push({
            type: "tool_result",
            tool_use_id: block.id,
            content: `Error: ${result.error}`,
            is_error: true,
          });
        } else {
          const rawData = Array.isArray(result.raw)
            ? result.raw
            : [result.raw];
          allData.push(...rawData);
          toolResults.push({
            type: "tool_result",
            tool_use_id: block.id,
            content: `成功提取 ${rawData.length} 条记录`,
          });
        }
      }

      messages.push({ role: "user", content: toolResults });
    }

    return allData;
  }

  private async loadState(jobId: string): Promise<ETLState | null> {
    try {
      const data = await fs.readFile(this.stateFile, "utf-8");
      const states = JSON.parse(data) as Record<string, ETLState>;
      const state = states[jobId];
      // 重置失败/完成状态以重新运行
      if (state && ["done", "failed"].includes(state.status)) return null;
      return state || null;
    } catch {
      return null;
    }
  }

  private initState(jobId: string): ETLState {
    return {
      jobId,
      status: "pending",
      startedAt: Date.now(),
      metrics: { extracted: 0, transformed: 0, loaded: 0, skipped: 0, errors: 0 },
    };
  }

  private async saveState(state: ETLState): Promise<void> {
    let states: Record<string, ETLState> = {};
    try {
      const data = await fs.readFile(this.stateFile, "utf-8");
      states = JSON.parse(data);
    } catch {}
    states[state.jobId] = state;
    await fs.writeFile(this.stateFile, JSON.stringify(states, null, 2));
  }
}

// 使用示例
async function main() {
  const orchestrator = new AgentETLOrchestrator();

  const job: ETLJob = {
    id: "github-stars-001",
    name: "GitHub 热门项目每日同步",
    sources: [
      {
        type: "api",
        config: {
          url: "https://api.github.com/search/repositories?q=stars:>10000&sort=stars&per_page=100",
          headers: { Accept: "application/vnd.github+json" },
        },
      },
    ],
    transform: {
      targetSchema: {
        id: "GitHub repo ID，整数",
        full_name: "owner/repo 格式的完整名称",
        description: "项目描述，可为空",
        stars: "star 数量，整数",
        language: "主要编程语言",
        topics: "话题标签，逗号分隔字符串",
        updated_at: "最后更新时间，ISO 8601",
        license: "许可证名称，SPDX 格式",
      },
      rules: [
        "topics 从数组转换为逗号分隔字符串",
        "license 从对象提取 spdx_id 字段",
        "stars 对应 stargazers_count 字段",
      ],
    },
    target: {
      type: "sqlite",
      config: {
        path: "/tmp/github-stars.db",
        table: "repositories",
        primaryKey: "id",
      },
    },
  };

  await orchestrator.run(job);
}

main().catch(console.error);
```

---

## 四、Python 版本

```python
# agent_etl.py
import json
import time
import asyncio
import aiohttp
from pathlib import Path
from dataclasses import dataclass, field, asdict
from typing import Any, Optional
import anthropic
import sqlite3

@dataclass
class TransformConfig:
    target_schema: dict[str, str]
    rules: list[str] = field(default_factory=list)
    batch_size: int = 20

@dataclass
class ETLMetrics:
    extracted: int = 0
    transformed: int = 0
    loaded: int = 0
    skipped: int = 0
    errors: int = 0

class AgentETL:
    def __init__(self):
        self.client = anthropic.Anthropic()

    async def extract_api(self, url: str, headers: dict = None) -> list[Any]:
        """带自动分页的 API 提取"""
        all_data = []
        async with aiohttp.ClientSession() as session:
            next_url = url
            while next_url:
                async with session.get(next_url, headers=headers or {}) as resp:
                    resp.raise_for_status()
                    data = await resp.json()

                    if isinstance(data, list):
                        all_data.extend(data)
                        break
                    elif isinstance(data, dict):
                        # 处理常见嵌套格式
                        items = (data.get("items") or data.get("data")
                                 or data.get("results") or [data])
                        all_data.extend(items)
                        # 尝试获取下一页
                        next_url = (data.get("next") or
                                   data.get("next_page") or
                                   data.get("links", {}).get("next"))
                    else:
                        break

        return all_data

    def batch_transform(
        self,
        records: list[Any],
        config: TransformConfig
    ) -> list[dict]:
        """用 LLM 批量转换数据"""
        results = []
        schema_desc = "\n".join(
            f"- {k}: {v}" for k, v in config.target_schema.items()
        )
        rules_desc = "\n".join(config.rules) if config.rules else ""

        for i in range(0, len(records), config.batch_size):
            batch = records[i:i + config.batch_size]

            response = self.client.messages.create(
                model="claude-haiku-4-5",
                max_tokens=4096,
                system=f"""数据转换专家。将数据转换为目标 Schema，返回纯 JSON 数组。

目标 Schema：
{schema_desc}

{f'业务规则：\n{rules_desc}' if rules_desc else ''}

每个元素格式: {{"data": {{...}}, "warnings": [...], "skipped": false}}""",
                messages=[{
                    "role": "user",
                    "content": f"转换以下 {len(batch)} 条记录：\n```json\n{json.dumps(batch, ensure_ascii=False, indent=2)}\n```"
                }]
            )

            text = response.content[0].text
            # 提取 JSON 数组
            import re
            match = re.search(r'\[[\s\S]*\]', text)
            if match:
                transformed = json.loads(match.group())
                results.extend(transformed)

            print(f"  转换进度: {min(i + config.batch_size, len(records))}/{len(records)}")

        return results

    def load_sqlite(
        self,
        records: list[dict],
        db_path: str,
        table: str,
        primary_key: str = "id"
    ) -> dict:
        """幂等写入 SQLite"""
        valid = [r for r in records if not r.get("skipped") and r.get("data")]
        if not valid:
            return {"inserted": 0, "updated": 0, "skipped": len(records), "errors": []}

        columns = list(valid[0]["data"].keys())
        result = {"inserted": 0, "updated": 0, "skipped": 0, "errors": []}

        with sqlite3.connect(db_path) as conn:
            # 动态建表
            col_defs = ", ".join(f'"{c}" TEXT' for c in columns)
            conn.execute(f"""
                CREATE TABLE IF NOT EXISTS {table} (
                    {col_defs},
                    _loaded_at INTEGER DEFAULT (unixepoch()),
                    _warnings TEXT
                )
            """)

            # UPSERT
            placeholders = ", ".join("?" * (len(columns) + 1))
            upsert_sql = f"""
                INSERT INTO {table} ({", ".join(f'"{c}"' for c in columns)}, _warnings)
                VALUES ({placeholders})
                ON CONFLICT("{primary_key}") DO UPDATE SET
                {", ".join(f'"{c}" = excluded."{c}"' for c in columns if c != primary_key)},
                _loaded_at = unixepoch(),
                _warnings = excluded._warnings
            """

            for rec in records:
                if rec.get("skipped"):
                    result["skipped"] += 1
                    continue
                try:
                    values = [
                        str(rec["data"].get(c)) if rec["data"].get(c) is not None else None
                        for c in columns
                    ]
                    values.append("; ".join(rec.get("warnings", [])) or None)
                    conn.execute(upsert_sql, values)
                    result["inserted"] += 1
                except Exception as e:
                    result["errors"].append(str(e))

        return result

    async def run(self, job: dict) -> ETLMetrics:
        metrics = ETLMetrics()
        print(f"\n🚀 ETL Job: {job['name']}")

        # Extract
        print("\n📥 阶段 1: Extract")
        all_raw = []
        for source in job["sources"]:
            if source["type"] == "api":
                records = await self.extract_api(
                    source["config"]["url"],
                    source["config"].get("headers")
                )
                all_raw.extend(records)
                metrics.extracted += len(records)
                print(f"  ✅ 提取 {len(records)} 条")

        # Transform
        print("\n🔄 阶段 2: Transform")
        config = TransformConfig(**job["transform"])
        transformed = self.batch_transform(all_raw, config)
        metrics.transformed = sum(1 for r in transformed if not r.get("skipped"))
        metrics.skipped = sum(1 for r in transformed if r.get("skipped"))
        print(f"  ✅ 转换 {metrics.transformed} 条，跳过 {metrics.skipped} 条")

        # Load
        print("\n📤 阶段 3: Load")
        target = job["target"]
        if target["type"] == "sqlite":
            result = self.load_sqlite(
                transformed,
                target["config"]["path"],
                target["config"]["table"],
                target["config"].get("primary_key", "id")
            )
            metrics.loaded = result["inserted"] + result["updated"]
            metrics.errors = len(result["errors"])
            print(f"  ✅ 写入 {metrics.loaded} 条（新增 {result['inserted']}，更新 {result['updated']}）")

        print(f"\n✅ ETL 完成！提取:{metrics.extracted} → 转换:{metrics.transformed} → 写入:{metrics.loaded}")
        return metrics


# 运行示例
async def main():
    etl = AgentETL()

    job = {
        "name": "HackerNews 热门文章同步",
        "sources": [{
            "type": "api",
            "config": {
                "url": "https://hacker-news.firebaseio.com/v0/topstories.json"
            }
        }],
        "transform": {
            "target_schema": {
                "id": "文章 ID，整数",
                "title": "文章标题",
                "url": "外链 URL，可为空",
                "score": "得分，整数",
                "by": "作者用户名",
                "time": "发布时间戳，整数",
                "type": "内容类型：story/comment/job/poll"
            },
            "rules": ["time 字段保持原始 Unix 时间戳"],
            "batch_size": 20
        },
        "target": {
            "type": "sqlite",
            "config": {
                "path": "/tmp/hackernews.db",
                "table": "stories",
                "primary_key": "id"
            }
        }
    }

    await etl.run(job)

asyncio.run(main())
```

---

## 五、OpenClaw Cron 驱动定时 ETL

结合 OpenClaw 的 Cron 系统，实现全自动定时数据同步：

```typescript
// heartbeat-etl.ts - 放入 HEARTBEAT.md 触发，或用 Cron 调度

// 在 OpenClaw 里注册 Cron Job：
// schedule: { kind: "cron", expr: "0 */6 * * *", tz: "Australia/Sydney" }
// payload:  { kind: "agentTurn", message: "运行 ETL 作业: github-trends" }

// Agent 接到任务后：
const ETL_JOBS = {
  "github-trends": {
    id: "github-trends",
    name: "GitHub 趋势项目",
    // ... 配置
  },
  "hacker-news": {
    id: "hacker-news",
    name: "HackerNews 热门",
    // ... 配置
  },
};

// Agent 工具：触发 ETL
export const etlTool: Anthropic.Tool = {
  name: "run_etl_job",
  description: "运行数据管道 ETL 作业，将外部数据同步到本地存储",
  input_schema: {
    type: "object" as const,
    properties: {
      jobId: {
        type: "string",
        description: "ETL 作业 ID",
        enum: Object.keys(ETL_JOBS),
      },
      forceRefresh: {
        type: "boolean",
        description: "是否强制全量刷新（忽略增量检查）",
      },
    },
    required: ["jobId"],
  },
};
```

---

## 六、增量 ETL：只处理新数据

生产环境中全量 ETL 太慢，需要增量策略：

```typescript
interface IncrementalConfig {
  strategy: "timestamp" | "cursor" | "hash";
  checkpointStore: string; // 记录上次处理位置
}

class IncrementalETL {
  private checkpoints: Map<string, unknown> = new Map();

  async loadCheckpoint(jobId: string): Promise<unknown> {
    try {
      const data = JSON.parse(
        await fs.readFile(`/tmp/etl-checkpoint-${jobId}.json`, "utf-8")
      );
      return data.checkpoint;
    } catch {
      return null;
    }
  }

  async saveCheckpoint(jobId: string, checkpoint: unknown): Promise<void> {
    await fs.writeFile(
      `/tmp/etl-checkpoint-${jobId}.json`,
      JSON.stringify({ checkpoint, savedAt: Date.now() })
    );
  }

  // 基于时间戳增量提取
  async extractSince(
    url: string,
    lastTimestamp: number | null
  ): Promise<{ records: unknown[]; newCheckpoint: number }> {
    const since = lastTimestamp
      ? new Date(lastTimestamp).toISOString()
      : new Date(0).toISOString();

    const response = await fetch(
      `${url}?updated_after=${encodeURIComponent(since)}&sort=updated_at&order=asc`
    );
    const data = await response.json() as { items?: unknown[]; data?: unknown[] };
    const records = (data.items || data.data || [data]) as unknown[];

    const newCheckpoint =
      records.length > 0
        ? Date.now()
        : lastTimestamp || Date.now();

    return { records, newCheckpoint };
  }
}
```

---

## 七、关键设计原则

| 原则 | 传统 ETL | Agent ETL |
|------|---------|-----------|
| Schema 变更 | 手动修改映射代码 | LLM 自动适应新字段 |
| 异常数据 | 报错/跳过 | LLM 理解语义后决策 |
| 数据质量 | 静态规则检查 | LLM 判断语义合理性 |
| 调试 | 查日志 | 查 `_warnings` 字段 |
| 成本 | 计算资源 | LLM API 调用费 |

**什么时候用 Agent ETL：**
- ✅ 数据格式多样、Schema 经常变化
- ✅ 需要语义理解（合并重复、识别异常）
- ✅ 数据量中等（百万以内）

**什么时候用传统 ETL：**
- ❌ 超大数据量（GB/TB 级）
- ❌ 结构完全固定、无需语义理解
- ❌ 极低延迟要求（毫秒级）

---

## 八、与 pi-mono 集成

```typescript
// pi-mono 中注册为工具
import { defineTool } from "@pi-dev/core";

export const dataIngestionTool = defineTool({
  name: "ingest_data",
  description: "从外部源提取并存储结构化数据",
  parameters: z.object({
    source: z.string().describe("数据源 URL 或路径"),
    schema: z.record(z.string()).describe("目标 Schema 字段描述"),
    destination: z.string().describe("目标 SQLite 数据库路径"),
  }),
  execute: async ({ source, schema, destination }) => {
    const etl = new AgentETLOrchestrator();
    const job: ETLJob = {
      id: `auto-${Date.now()}`,
      name: `Auto Ingest: ${source}`,
      sources: [{ type: "api", config: { url: source } }],
      transform: { targetSchema: schema },
      target: {
        type: "sqlite",
        config: { path: destination, table: "records", primaryKey: "id" },
      },
    };
    const state = await etl.run(job);
    return {
      success: state.status === "done",
      metrics: state.metrics,
    };
  },
});
```

---

## 总结

**Agent ETL = Extract（工具执行 I/O） + Transform（LLM 理解语义） + Load（幂等写入）**

三个关键设计：
1. **批量 Transform** — 20 条一批丢给 Haiku，省成本不影响质量
2. **断点续传** — 状态文件 + 临时缓存，失败重跑不重头
3. **幂等 UPSERT** — 同一条数据跑多次结果一样，不怕重复

当你的 Agent 需要定期从外部世界拉取数据并存储，ETL 模式让整个流程有结构、可监控、能恢复。

> **下节预告：** Agent 知识库构建与自动索引——把 ETL 进来的数据变成可检索的知识库，RAG 的上游工程实践。
