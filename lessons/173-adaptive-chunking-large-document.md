# 173 - Agent 自适应分块与大文档处理

**Adaptive Chunking & Large Document Processing**

---

## 为什么需要分块？

Agent 处理大文档时面临三个硬约束：

1. **Context Window 上限**：GPT-4o 128K、Claude 200K，超了直接报错
2. **成本随 Token 线性增长**：塞整个文件进去成本爆炸
3. **注意力稀释**：文档太长，LLM 对中间部分"记忆"变差（Lost-in-the-Middle 问题）

**关键原则**：不是把文档切碎扔进去，而是**按需取用、精准投喂**。

---

## 四种分块策略对比

```
策略              适用场景              优点              缺点
─────────────────────────────────────────────────────────────
固定大小分块       结构化日志、代码行    实现简单           切断语义
语义分块           文章、文档            保留上下文         需要 NLP 预处理
滑动窗口           对话历史              不丢上下文         有冗余重叠
自适应分块         混合内容              按内容动态调整     实现复杂
```

---

## 核心实现：自适应分块器

```typescript
// adaptive-chunker.ts
interface Chunk {
  index: number;
  content: string;
  tokenCount: number;
  metadata: {
    start: number;     // 原文字节偏移
    end: number;
    type: 'code' | 'prose' | 'data' | 'header';
    overlap: number;   // 与上一块的重叠 tokens
  };
}

interface ChunkStrategy {
  maxTokens: number;       // 每块最大 token 数
  overlapTokens: number;   // 块间重叠 token 数（保留上下文）
  budgetTokens: number;    // 当前 context 剩余预算
}

class AdaptiveChunker {
  private tokenizer: (text: string) => number;

  constructor(tokenizer: (text: string) => number) {
    this.tokenizer = tokenizer;
  }

  chunk(document: string, strategy: ChunkStrategy): Chunk[] {
    const sections = this.detectSections(document);
    const chunks: Chunk[] = [];
    let index = 0;

    for (const section of sections) {
      const sectionTokens = this.tokenizer(section.content);

      if (sectionTokens <= strategy.maxTokens) {
        // 整节直接作为一块
        chunks.push({
          index: index++,
          content: section.content,
          tokenCount: sectionTokens,
          metadata: {
            start: section.start,
            end: section.end,
            type: section.type,
            overlap: 0,
          },
        });
      } else {
        // 超大节：递归拆分
        const subChunks = this.splitSection(section, strategy, index);
        chunks.push(...subChunks);
        index += subChunks.length;
      }
    }

    return chunks;
  }

  // 检测文档结构：代码块、段落、表格、标题
  private detectSections(document: string) {
    const sections: Array<{
      content: string;
      start: number;
      end: number;
      type: 'code' | 'prose' | 'data' | 'header';
    }> = [];

    // 正则识别 Markdown 代码块
    const codeBlockRegex = /```[\s\S]*?```/g;
    // 正则识别标题
    const headerRegex = /^#{1,6}\s.+$/gm;
    // 普通段落按双换行分割

    let lastEnd = 0;
    let match: RegExpExecArray | null;

    // 先提取代码块（它们不应被切断）
    while ((match = codeBlockRegex.exec(document)) !== null) {
      if (match.index > lastEnd) {
        // 代码块前的文本
        const prose = document.slice(lastEnd, match.index);
        sections.push(...this.splitProse(prose, lastEnd));
      }
      sections.push({
        content: match[0],
        start: match.index,
        end: match.index + match[0].length,
        type: 'code',
      });
      lastEnd = match.index + match[0].length;
    }

    // 剩余文本
    if (lastEnd < document.length) {
      const remaining = document.slice(lastEnd);
      sections.push(...this.splitProse(remaining, lastEnd));
    }

    return sections.sort((a, b) => a.start - b.start);
  }

  private splitProse(text: string, offset: number) {
    // 按段落（双换行）分割
    const paragraphs = text.split(/\n{2,}/);
    const result = [];
    let pos = offset;

    for (const para of paragraphs) {
      if (para.trim()) {
        const type = /^#{1,6}\s/.test(para.trim()) ? 'header' : 'prose';
        result.push({
          content: para,
          start: pos,
          end: pos + para.length,
          type: type as 'header' | 'prose',
        });
      }
      pos += para.length + 2; // +2 for \n\n
    }
    return result;
  }

  private splitSection(
    section: { content: string; start: number; end: number; type: string },
    strategy: ChunkStrategy,
    startIndex: number,
  ): Chunk[] {
    const chunks: Chunk[] = [];
    const sentences = section.content.match(/[^.!?。！？]+[.!?。！？]*/g) || [section.content];

    let currentChunk = '';
    let currentTokens = 0;
    let chunkIndex = startIndex;

    for (const sentence of sentences) {
      const sentenceTokens = this.tokenizer(sentence);

      if (currentTokens + sentenceTokens > strategy.maxTokens && currentChunk) {
        chunks.push({
          index: chunkIndex++,
          content: currentChunk,
          tokenCount: currentTokens,
          metadata: {
            start: section.start,
            end: section.end,
            type: section.type as any,
            overlap: 0,
          },
        });

        // 重叠：取上一块末尾 overlapTokens 作为下一块开头
        const overlapText = this.extractOverlap(currentChunk, strategy.overlapTokens);
        currentChunk = overlapText + sentence;
        currentTokens = this.tokenizer(currentChunk);
      } else {
        currentChunk += sentence;
        currentTokens += sentenceTokens;
      }
    }

    if (currentChunk) {
      chunks.push({
        index: chunkIndex,
        content: currentChunk,
        tokenCount: currentTokens,
        metadata: {
          start: section.start,
          end: section.end,
          type: section.type as any,
          overlap: strategy.overlapTokens,
        },
      });
    }

    return chunks;
  }

  private extractOverlap(text: string, targetTokens: number): string {
    // 从末尾取 targetTokens 数量的文本
    const words = text.split(' ');
    let overlap = '';
    for (let i = words.length - 1; i >= 0; i--) {
      const candidate = words.slice(i).join(' ');
      if (this.tokenizer(candidate) <= targetTokens) {
        overlap = candidate;
      } else {
        break;
      }
    }
    return overlap ? overlap + ' ' : '';
  }
}
```

---

## Token 预算感知：动态调整分块大小

Agent 处理文档时，Context Window 是动态消耗的。分块大小应随剩余预算收缩：

```typescript
class BudgetAwareChunker {
  private chunker: AdaptiveChunker;
  private contextBudget: number;

  constructor(totalContextSize: number) {
    // 粗略 tokenizer：4字符 ≈ 1 token
    this.chunker = new AdaptiveChunker((text) => Math.ceil(text.length / 4));
    // 为系统提示、工具调用、输出预留 30%
    this.contextBudget = Math.floor(totalContextSize * 0.7);
  }

  getChunkStrategy(usedTokens: number): ChunkStrategy {
    const remaining = this.contextBudget - usedTokens;

    if (remaining > 50000) {
      return { maxTokens: 8000, overlapTokens: 200, budgetTokens: remaining };
    } else if (remaining > 20000) {
      return { maxTokens: 4000, overlapTokens: 100, budgetTokens: remaining };
    } else if (remaining > 8000) {
      return { maxTokens: 2000, overlapTokens: 50, budgetTokens: remaining };
    } else {
      // 预算紧张：只取最关键的块
      return { maxTokens: 1000, overlapTokens: 20, budgetTokens: remaining };
    }
  }

  async processDocument(
    document: string,
    question: string,
    usedTokens: number,
  ): Promise<string[]> {
    const strategy = this.getChunkStrategy(usedTokens);
    const allChunks = this.chunker.chunk(document, strategy);

    // 按相关性排序，优先处理最相关的块
    const scored = await this.scoreRelevance(allChunks, question);
    const topChunks = scored
      .sort((a, b) => b.score - a.score)
      .slice(0, Math.floor(strategy.budgetTokens / strategy.maxTokens));

    return topChunks.map((c) => c.chunk.content);
  }

  private async scoreRelevance(
    chunks: Chunk[],
    question: string,
  ): Promise<Array<{ chunk: Chunk; score: number }>> {
    // 简单关键词匹配（生产中用向量相似度）
    const keywords = question.toLowerCase().split(/\s+/);
    return chunks.map((chunk) => ({
      chunk,
      score: keywords.filter((kw) => chunk.content.toLowerCase().includes(kw)).length,
    }));
  }
}
```

---

## OpenClaw 实战：Read Tool 的分块读取

OpenClaw 的 `read` 工具内置了 `offset/limit` 参数，**天然支持分块读取**：

```typescript
// OpenClaw Agent 处理大文件的正确方式

// ❌ 错误：一次读整个大文件
const fullFile = await tools.read({ path: 'huge-log.txt' });
// 可能超 Context Window，且大部分内容无关

// ✅ 正确：先探测结构，再按需读取
async function readLargeFile(path: string, targetLineRange?: [number, number]) {
  // 第一步：读前 50 行了解结构
  const header = await tools.read({ path, limit: 50 });
  
  // 第二步：根据需要读具体范围
  if (targetLineRange) {
    const [start, end] = targetLineRange;
    return await tools.read({ 
      path, 
      offset: start, 
      limit: end - start 
    });
  }
  
  return header;
}

// Agent Loop 中的大文档处理模式
async function analyzeCodebase(repoPath: string) {
  // 先读目录结构
  const structure = await tools.exec({ command: `find ${repoPath} -name "*.ts" | head -20` });
  
  // 每个文件分批读取，不超过 2000 行
  const BATCH_SIZE = 200;
  for (const file of structure.files) {
    let offset = 1;
    while (true) {
      const batch = await tools.read({ path: file, offset, limit: BATCH_SIZE });
      if (!batch.content) break;
      
      // 处理这批内容
      await processChunk(batch.content, file, offset);
      offset += BATCH_SIZE;
      
      // 检查预算
      if (remainingBudget() < 5000) break;
    }
  }
}
```

---

## Map-Reduce 模式：分块处理 + 聚合答案

对于长文档 Q&A，最有效的模式是 Map-Reduce：

```typescript
class DocumentQA {
  async answer(document: string, question: string): Promise<string> {
    const chunker = new BudgetAwareChunker(200000); // Claude 200K
    const chunks = new AdaptiveChunker((t) => Math.ceil(t.length / 4))
      .chunk(document, { maxTokens: 4000, overlapTokens: 100, budgetTokens: 100000 });

    // MAP 阶段：每块独立回答
    const partialAnswers = await Promise.all(
      chunks.map(async (chunk) => {
        const response = await llm.call({
          messages: [
            { role: 'user', content: 
              `基于以下文本片段（第${chunk.index + 1}块）回答问题。\n` +
              `如果这段文字与问题无关，回答"无关"。\n\n` +
              `文本：\n${chunk.content}\n\n问题：${question}`
            }
          ]
        });
        return { chunkIndex: chunk.index, answer: response.content };
      })
    );

    // 过滤无关块
    const relevant = partialAnswers.filter(a => !a.answer.includes('无关'));

    if (relevant.length === 0) return '文档中未找到相关信息。';

    // REDUCE 阶段：综合所有局部答案
    const combined = relevant.map(a => `[第${a.chunkIndex + 1}部分]\n${a.answer}`).join('\n\n');
    const finalResponse = await llm.call({
      messages: [
        { role: 'user', content: 
          `以下是对同一问题的多个局部回答，请综合成一个完整、连贯的答案：\n\n${combined}\n\n问题：${question}`
        }
      ]
    });

    return finalResponse.content;
  }
}
```

---

## Python 版本（pi-mono 风格）

```python
# adaptive_chunker.py
from dataclasses import dataclass
from typing import List, Literal, Optional
import re

ChunkType = Literal['code', 'prose', 'data', 'header']

@dataclass
class Chunk:
    index: int
    content: str
    token_count: int
    chunk_type: ChunkType
    overlap_tokens: int = 0

class AdaptiveChunker:
    def __init__(self, tokenizer=None):
        # 默认用字符数估算（4字符≈1token）
        self.tokenizer = tokenizer or (lambda t: len(t) // 4)

    def chunk(self, document: str, max_tokens: int = 4000, overlap: int = 100) -> List[Chunk]:
        sections = self._detect_sections(document)
        chunks = []
        
        for i, section in enumerate(sections):
            tokens = self.tokenizer(section['content'])
            if tokens <= max_tokens:
                chunks.append(Chunk(
                    index=len(chunks),
                    content=section['content'],
                    token_count=tokens,
                    chunk_type=section['type'],
                ))
            else:
                sub = self._split_large(section['content'], max_tokens, overlap, section['type'])
                for c in sub:
                    c.index = len(chunks)
                    chunks.append(c)
        
        return chunks

    def _detect_sections(self, document: str) -> List[dict]:
        sections = []
        # 代码块不能被切断
        code_pattern = re.compile(r'```[\s\S]*?```', re.MULTILINE)
        last_end = 0
        
        for match in code_pattern.finditer(document):
            if match.start() > last_end:
                prose = document[last_end:match.start()]
                sections.extend(self._split_prose(prose))
            sections.append({'content': match.group(), 'type': 'code'})
            last_end = match.end()
        
        if last_end < len(document):
            sections.extend(self._split_prose(document[last_end:]))
        
        return sections

    def _split_prose(self, text: str) -> List[dict]:
        paragraphs = re.split(r'\n{2,}', text)
        result = []
        for para in paragraphs:
            if para.strip():
                ptype = 'header' if re.match(r'^#{1,6}\s', para.strip()) else 'prose'
                result.append({'content': para, 'type': ptype})
        return result

    def _split_large(self, text: str, max_tokens: int, overlap: int, chunk_type: ChunkType) -> List[Chunk]:
        sentences = re.split(r'(?<=[.!?。！？])\s+', text)
        chunks = []
        current = ''
        current_tokens = 0
        
        for sent in sentences:
            sent_tokens = self.tokenizer(sent)
            if current_tokens + sent_tokens > max_tokens and current:
                chunks.append(Chunk(
                    index=0,  # 由调用方赋值
                    content=current,
                    token_count=current_tokens,
                    chunk_type=chunk_type,
                    overlap_tokens=overlap,
                ))
                # 取末尾作为重叠
                words = current.split()
                overlap_text = ' '.join(words[-overlap//5:]) + ' ' if len(words) > overlap//5 else ''
                current = overlap_text + sent
                current_tokens = self.tokenizer(current)
            else:
                current += (' ' if current else '') + sent
                current_tokens += sent_tokens
        
        if current:
            chunks.append(Chunk(
                index=0,
                content=current,
                token_count=current_tokens,
                chunk_type=chunk_type,
            ))
        
        return chunks


# 用法示例
if __name__ == '__main__':
    with open('large_document.md') as f:
        doc = f.read()
    
    chunker = AdaptiveChunker()
    chunks = chunker.chunk(doc, max_tokens=3000, overlap=150)
    
    print(f"文档共 {len(doc)} 字符，分成 {len(chunks)} 块")
    for c in chunks:
        print(f"  块{c.index}: {c.chunk_type} | {c.token_count} tokens | 前30字: {c.content[:30]!r}")
```

---

## 常见坑与最佳实践

### 坑 1：代码块被切断
```
# ❌ 在代码块中间切割
chunk1 = "def foo():\n    x = 1\n    y = 2"
chunk2 = "    return x + y"  # 语法不完整，LLM 理解困难

# ✅ 代码块作为原子单元，不允许切断
```

### 坑 2：重叠太大浪费 Token
```
# 重叠应该是语义边界，不是固定字数
# 好的重叠：最后一个完整段落/函数签名
# 坏的重叠：固定 500 字符（可能切断句子）
```

### 坑 3：忽视文档类型
```
文档类型      推荐分块方式
──────────────────────────────
Markdown      按标题 H2/H3 分节
代码文件      按函数/类分割（用 AST）
CSV/JSON      按行数/条目数分割
对话记录      按时间窗口 + 完整轮次
PDF          按页 + 语义段落
```

### 最佳实践总结
1. **代码块不可切断**：用正则先提取，作为原子单元
2. **重叠要有语义**：不是固定字节，是完整的句子/段落
3. **预算感知**：Context 剩余越少，块越小，只取最相关的
4. **Map-Reduce 胜过暴力填充**：分块各自回答，再综合，质量更高
5. **相关性优先**：不是顺序处理，是按相关分数处理
6. **OpenClaw Read Tool**：充分利用 `offset/limit` 参数，避免一次性加载大文件

---

## 与其他课的关联

| 课程 | 关联点 |
|------|--------|
| #84 Prompt Caching | 固定分块可以被缓存，降低成本 |
| #127 Sliding Window + Dynamic Summarization | 对话历史的分块特化版本 |
| #147 RAG | 分块是 RAG 索引的前置步骤 |
| #164 Tool Output Noise Filtering | 分块后每块独立过滤噪声 |
| #159 Tool Result Pagination | 工具结果分页是输出侧的分块 |

---

## 总结

大文档处理的本质是**信息的按需提取**，而不是暴力塞入 Context。

核心思路：
- **检测结构** → 代码块不切断，段落按语义分
- **动态大小** → 根据 Context 预算自动调整块大小  
- **相关性优先** → 不是顺序处理，是最相关的先处理
- **Map-Reduce** → 分块各自回答，汇总优于合并后处理

OpenClaw 的 `read` 工具天然支持分块读取（`offset/limit`），Claude Code 就是用这个处理大型代码库的。
