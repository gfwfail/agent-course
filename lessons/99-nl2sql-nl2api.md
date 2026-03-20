# 99 - Agent NL2SQL & NL2API：自然语言转结构化查询

> "用户说'帮我查下昨天的销售额'，Agent 要把这句话变成一条 SQL 发给数据库。"

## 为什么需要 NL2SQL / NL2API

Agent 的核心价值之一：**让用户用自然语言操作系统**，而不是手写 SQL 或记 API 文档。

典型场景：
- 业务分析：「上周新用户注册了多少？」→ SQL
- 数据查询：「查一下订单 #1024 的状态」→ REST API
- 报表生成：「给我按地区分组的月度收入」→ SQL + 聚合

---

## 核心架构：三步走

```
用户输入 → [Schema 注入] → LLM 生成查询 → [验证/清洗] → 执行 → [结果格式化] → 输出
                                              ↑
                                         [错误恢复：LLM 自修复]
```

---

## Step 1：Schema 注入（最关键）

LLM 不知道你的表结构，你必须告诉它。

### 方案 A：全量 Schema（小库用）

```python
# learn-claude-code 风格：工具调用前注入 schema
def build_nl2sql_prompt(user_question: str, schema: dict) -> str:
    schema_text = "\n".join([
        f"表 {table}: {', '.join(cols)}"
        for table, cols in schema.items()
    ])
    return f"""你是一个 MySQL 专家。根据以下表结构生成 SQL。

数据库表结构：
{schema_text}

规则：
- 只生成 SELECT 语句，禁止 INSERT/UPDATE/DELETE/DROP
- 用 LIMIT 控制结果数量，默认 LIMIT 100
- 日期用 DATE_SUB(NOW(), INTERVAL N DAY) 格式
- 只返回 SQL，不要解释

用户问题：{user_question}

SQL："""
```

### 方案 B：按需查询 Schema（大库用）

```typescript
// pi-mono 风格：工具 + 动态 schema 加载
const tools = [
  {
    name: "get_table_schema",
    description: "获取指定表的字段结构",
    input_schema: {
      type: "object",
      properties: {
        table_name: { type: "string", description: "表名" }
      },
      required: ["table_name"]
    }
  },
  {
    name: "execute_sql",
    description: "执行 SELECT 查询",
    input_schema: {
      type: "object",
      properties: {
        sql: { type: "string", description: "SQL 查询语句" }
      },
      required: ["sql"]
    }
  }
];

// Agent 会先调 get_table_schema，再生成 SQL，再调 execute_sql
```

**选择策略**：
- 表 < 20 个 → 全量注入
- 表 > 20 个 → 先让 LLM 问 schema，再生成
- 有注释的 schema → 务必把注释也传进去！

---

## Step 2：生成与验证

### SQL 安全验证（必做！）

```python
import sqlparse
from sqlparse.sql import Statement
from sqlparse.tokens import Keyword, DDL, DML

def validate_sql(sql: str) -> tuple[bool, str]:
    """只允许 SELECT，拒绝一切写操作"""
    parsed = sqlparse.parse(sql.strip())
    if not parsed:
        return False, "空 SQL"
    
    stmt = parsed[0]
    stmt_type = stmt.get_type()
    
    if stmt_type != "SELECT":
        return False, f"禁止 {stmt_type} 操作，只允许 SELECT"
    
    # 检查危险关键词
    dangerous = {"DROP", "DELETE", "UPDATE", "INSERT", "TRUNCATE", "ALTER", "CREATE"}
    tokens_upper = {t.normalized.upper() for t in stmt.flatten()}
    found = dangerous & tokens_upper
    if found:
        return False, f"包含危险关键词: {found}"
    
    return True, "OK"

# 使用
sql = llm_generate(user_question)
ok, reason = validate_sql(sql)
if not ok:
    raise ValueError(f"SQL 安全检查失败: {reason}")
```

### OpenClaw 实战：Grafana SQL Tool

```typescript
// OpenClaw Skill 中查询 Grafana 数据源的实现
async function nl2GrafanaSQL(question: string, datasourceId: number) {
  const schema = await fetchGrafanaSchema(datasourceId);
  
  const response = await anthropic.messages.create({
    model: "claude-sonnet-4-5",
    max_tokens: 500,
    system: `你是数据分析专家，生成 MySQL SQL 查询。
数据库表：
${schema}
只返回 SQL，不要 markdown 代码块。`,
    messages: [{ role: "user", content: question }]
  });
  
  const sql = response.content[0].text.trim()
    .replace(/```sql\n?/g, "")  // 清除 LLM 可能加的代码块
    .replace(/```\n?/g, "")
    .trim();
  
  const [valid, reason] = validateSQL(sql);
  if (!valid) throw new Error(reason);
  
  return executeGrafanaQuery(datasourceId, sql);
}
```

---

## Step 3：错误自修复（Self-Correction Loop）

LLM 生成的 SQL 有时会出错。让它自己修。

```python
async def nl2sql_with_retry(question: str, schema: dict, max_retries=3):
    messages = [
        {"role": "user", "content": build_nl2sql_prompt(question, schema)}
    ]
    
    for attempt in range(max_retries):
        # 生成 SQL
        response = await llm.complete(messages)
        sql = extract_sql(response)
        
        try:
            # 验证
            valid, reason = validate_sql(sql)
            if not valid:
                raise ValueError(reason)
            
            # 执行
            result = await db.execute(sql)
            return result, sql
            
        except Exception as e:
            if attempt == max_retries - 1:
                raise
            
            # 把错误喂回给 LLM，让它修复
            messages.append({"role": "assistant", "content": sql})
            messages.append({
                "role": "user", 
                "content": f"执行出错：{e}\n请修复 SQL，只返回修正后的 SQL："
            })
            # 继续循环，LLM 看到错误信息后会修正
    
    raise RuntimeError("SQL 生成失败，已重试 3 次")
```

**实战数据**：加入自修复循环后，NL2SQL 成功率从 ~75% 提升到 ~95%。

---

## NL2API：同样的思路

当目标不是数据库而是 REST API 时：

```python
# 注入 API 文档（OpenAPI spec 摘要）
API_SPEC = """
GET /orders/{id}          - 查询订单详情
GET /orders?status=&date= - 列出订单，支持状态/日期过滤  
GET /users/{id}/balance   - 查询用户余额
POST /withdrawals         - 发起提现（需要 amount, user_id）
"""

def nl2api_prompt(question: str) -> str:
    return f"""根据以下 API，生成一个 JSON 调用计划：

{API_SPEC}

用户请求：{question}

以 JSON 数组返回调用步骤：
[
  {{"method": "GET", "path": "/orders/1024", "params": {{}}}},
  ...
]"""
```

---

## Few-Shot：大幅提升准确率

给几个例子，LLM 立刻"懂你的数据库"：

```python
FEW_SHOT_EXAMPLES = """
示例1:
问题：昨天有多少用户注册？
SQL：SELECT COUNT(*) as cnt FROM users WHERE DATE(created_at) = DATE_SUB(CURDATE(), INTERVAL 1 DAY)

示例2:
问题：开箱次数最多的前5个箱子
SQL：SELECT b.name, COUNT(*) as cnt FROM open_box_results obr JOIN boxes b ON obr.box_id = b.id GROUP BY b.id ORDER BY cnt DESC LIMIT 5

示例3:
问题：今天的提现总额
SQL：SELECT SUM(amount) as total FROM crypto_withdraw_requests WHERE DATE(created_at) = CURDATE() AND status = 'processed'
"""
```

---

## pi-mono 中的完整实现参考

```typescript
// pi-mono: tools/database.ts
export class DatabaseTool implements Tool {
  name = "query_database";
  description = "用自然语言查询数据库，自动生成并执行 SQL";
  
  inputSchema = {
    type: "object" as const,
    properties: {
      question: {
        type: "string",
        description: "用自然语言描述你想查什么"
      }
    },
    required: ["question"]
  };
  
  async execute({ question }: { question: string }) {
    // 1. 注入 schema
    const schema = await this.getRelevantSchema(question);
    
    // 2. 生成 SQL（带 few-shot）
    const sql = await this.generateSQL(question, schema);
    
    // 3. 验证
    this.validateSQL(sql);  // throws if dangerous
    
    // 4. 执行
    const rows = await this.db.query(sql);
    
    // 5. 格式化给 LLM 理解
    return {
      sql,  // 透明度：让用户知道执行了什么
      rowCount: rows.length,
      data: rows.slice(0, 50),  // 截断防止 token 爆炸
      truncated: rows.length > 50
    };
  }
  
  private async getRelevantSchema(question: string): Promise<string> {
    // 用关键词匹配找相关表（也可以用向量搜索）
    const allTables = await this.db.query("SHOW TABLES");
    const relevant = allTables.filter(t => 
      question.toLowerCase().includes(t.toLowerCase().replace(/_/g, " "))
    );
    
    if (relevant.length === 0) return this.fullSchema;
    
    const schemas = await Promise.all(
      relevant.map(t => this.db.query(`DESCRIBE ${t}`))
    );
    return this.formatSchema(relevant, schemas);
  }
}
```

---

## 关键坑与最佳实践

| 问题 | 症状 | 解决 |
|------|------|------|
| LLM 幻觉字段名 | SQL 报 `Unknown column` | 把真实字段名加入 schema |
| 结果太大 | Token 超限 | 返回结果时加 `LIMIT`，截断 rows |
| 时区问题 | 日期查询差一天 | 在 prompt 里注明时区 |
| 复杂连表 | 生成 SQL 错误 | 提供 JOIN 关系说明 + few-shot |
| 注入攻击 | 用户构造恶意输入 | 严格白名单验证，只允许 SELECT |

---

## OpenClaw 中的实战应用

本课程的 cron job 本身就在用 NL2SQL 思想：

```
mysterybox skill → 接收自然语言问题
→ 调 Grafana API 执行预定义 SQL
→ 返回格式化结果
```

进一步可以让用户直接说「查一下今天的开箱利润」，Agent 动态生成 SQL 而不是靠预定义查询。

---

## 总结

NL2SQL 三板斧：

1. **Schema 注入** — LLM 看不见表结构就是瞎写，必须告诉它
2. **安全验证** — 只允许 SELECT，拒绝一切写操作
3. **自修复循环** — 把执行错误喂回 LLM，成功率 75% → 95%

核心原则：**透明度**，把最终执行的 SQL 也返回给用户，方便 debug 和建立信任。

---

*下一课：敬请期待 Lesson 100 🎉*
