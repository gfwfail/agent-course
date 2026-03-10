# 第 21 课：Multi-Modal Input Processing 多模态输入处理

> 让 Agent 不只是"读文字"，而是能"看图"、"听音"、"读文件"。

## 为什么需要多模态？

现代 AI Agent 面对的输入越来越复杂：

```
用户: [发送一张截图] 这个错误怎么解决？
用户: [发送一个 PDF] 帮我总结一下这份报告
用户: [发送一段语音] 把这段话翻译成英文
```

纯文本 Agent 处理不了这些。多模态能力让 Agent 真正"全能"。

## 多模态输入的类型

| 类型 | 常见格式 | 处理方式 |
|-----|---------|---------|
| 图片 | jpg, png, gif, webp | Vision API 直接理解 |
| 文档 | pdf, docx, xlsx | 解析后转文本 |
| 音频 | mp3, wav, ogg | 转录成文本 |
| 视频 | mp4, mov | 抽帧 + 音频转录 |
| 代码 | 各种语言 | 直接作为文本 |
| 结构化数据 | json, csv, xml | 解析后格式化 |

## 架构设计

### 1. 输入预处理管道

```typescript
// pi-mono 风格的多模态处理
interface MultiModalInput {
  type: 'text' | 'image' | 'audio' | 'document' | 'video';
  content: string | Buffer;
  mimeType?: string;
  metadata?: Record<string, unknown>;
}

interface ProcessedInput {
  textContent: string;           // 总是输出文本
  attachments?: ImageAttachment[]; // 图片可以直接传给 Vision
  metadata: {
    originalType: string;
    processingTime: number;
    tokensEstimate: number;
  };
}

async function preprocessInput(input: MultiModalInput): Promise<ProcessedInput> {
  switch (input.type) {
    case 'image':
      return processImage(input);
    case 'audio':
      return processAudio(input);
    case 'document':
      return processDocument(input);
    case 'video':
      return processVideo(input);
    default:
      return { textContent: input.content as string, metadata: {...} };
  }
}
```

### 2. 图片处理：Vision API 集成

图片是最常见的多模态输入，现代 LLM 原生支持：

```typescript
// OpenClaw 中的图片处理
interface ImageAttachment {
  type: 'image';
  source: {
    type: 'base64' | 'url';
    media_type: string;  // image/jpeg, image/png, etc.
    data?: string;       // base64 data
    url?: string;        // 或者直接 URL
  };
}

async function processImage(input: MultiModalInput): Promise<ProcessedInput> {
  const buffer = input.content as Buffer;
  const mimeType = input.mimeType || detectMimeType(buffer);
  
  // 检查尺寸，必要时压缩
  const optimized = await optimizeImage(buffer, {
    maxWidth: 2048,
    maxHeight: 2048,
    maxSizeBytes: 5 * 1024 * 1024, // 5MB limit
  });
  
  return {
    textContent: '[用户发送了一张图片]',
    attachments: [{
      type: 'image',
      source: {
        type: 'base64',
        media_type: mimeType,
        data: optimized.toString('base64'),
      },
    }],
    metadata: {
      originalType: 'image',
      processingTime: Date.now() - startTime,
      tokensEstimate: estimateImageTokens(optimized),
    },
  };
}

// 图片 token 估算（Anthropic 的计算方式）
function estimateImageTokens(image: Buffer): number {
  const { width, height } = getImageDimensions(image);
  // 每个 tile 约 768 tokens，tile 大小约 512x512
  const tiles = Math.ceil(width / 512) * Math.ceil(height / 512);
  return tiles * 768;
}
```

### 3. 音频处理：转录优先

音频需要先转成文本（因为大多数 LLM 还不原生支持音频）：

```typescript
// 音频转录
async function processAudio(input: MultiModalInput): Promise<ProcessedInput> {
  const buffer = input.content as Buffer;
  
  // 使用 Whisper 或其他 STT 服务
  const transcript = await transcribe(buffer, {
    model: 'whisper-1',
    language: 'auto',  // 自动检测语言
    timestamps: true,  // 可选：带时间戳
  });
  
  return {
    textContent: `[用户发送了一段语音消息]\n\n转录内容：\n${transcript.text}`,
    metadata: {
      originalType: 'audio',
      duration: transcript.duration,
      language: transcript.language,
      processingTime: Date.now() - startTime,
      tokensEstimate: countTokens(transcript.text),
    },
  };
}
```

### 4. 文档处理：格式感知

不同文档格式需要不同的解析策略：

```typescript
async function processDocument(input: MultiModalInput): Promise<ProcessedInput> {
  const buffer = input.content as Buffer;
  const mimeType = input.mimeType || detectMimeType(buffer);
  
  let text: string;
  let attachments: ImageAttachment[] = [];
  
  switch (mimeType) {
    case 'application/pdf':
      const pdfResult = await parsePDF(buffer, {
        extractImages: true,  // PDF 里的图片也提取出来
        preserveLayout: true, // 尽量保留排版
      });
      text = pdfResult.text;
      attachments = pdfResult.images;
      break;
      
    case 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet':
      // Excel 转成 markdown 表格
      text = await excelToMarkdown(buffer);
      break;
      
    case 'application/vnd.openxmlformats-officedocument.wordprocessingml.document':
      text = await docxToMarkdown(buffer);
      break;
      
    default:
      text = buffer.toString('utf-8');
  }
  
  // 文档可能很长，需要智能截断
  const truncated = smartTruncate(text, {
    maxTokens: 50000,  // 根据模型 context 调整
    preserveStructure: true,  // 保留标题、段落结构
  });
  
  return {
    textContent: `[用户发送了一个 ${getFileTypeName(mimeType)} 文件]\n\n${truncated.text}`,
    attachments,
    metadata: {
      originalType: 'document',
      mimeType,
      truncated: truncated.wasTruncated,
      originalLength: text.length,
      processingTime: Date.now() - startTime,
      tokensEstimate: countTokens(truncated.text),
    },
  };
}
```

### 5. 视频处理：抽帧 + 音频

视频是最复杂的，需要组合处理：

```typescript
async function processVideo(input: MultiModalInput): Promise<ProcessedInput> {
  const buffer = input.content as Buffer;
  
  // 1. 提取关键帧
  const frames = await extractKeyFrames(buffer, {
    maxFrames: 10,        // 最多 10 帧
    interval: 'smart',    // 智能选择（变化大的帧）
    resolution: '720p',   // 降低分辨率节省 tokens
  });
  
  // 2. 提取并转录音频
  const audio = await extractAudio(buffer);
  const transcript = await transcribe(audio, { timestamps: true });
  
  // 3. 组合成结构化输出
  const description = frames.map((frame, i) => 
    `[${formatTimestamp(frame.timestamp)}] 帧 ${i + 1}`
  ).join('\n');
  
  return {
    textContent: `[用户发送了一段视频，时长 ${formatDuration(video.duration)}]

关键帧：
${description}

音频转录：
${transcript.text}`,
    attachments: frames.map(f => ({
      type: 'image',
      source: {
        type: 'base64',
        media_type: 'image/jpeg',
        data: f.data.toString('base64'),
      },
    })),
    metadata: {
      originalType: 'video',
      duration: video.duration,
      frameCount: frames.length,
      processingTime: Date.now() - startTime,
      tokensEstimate: frames.length * 768 + countTokens(transcript.text),
    },
  };
}
```

## 实际应用：OpenClaw 的图片处理

OpenClaw 的 `image` 工具展示了实际生产中的图片处理：

```typescript
// 简化的 image 工具实现
const imageTool = {
  name: 'image',
  description: 'Analyze an image with the configured image model.',
  parameters: {
    image: { type: 'string', required: true },  // URL 或 path
    prompt: { type: 'string' },
    model: { type: 'string' },
    maxBytesMb: { type: 'number' },
  },
  
  async execute({ image, prompt, model, maxBytesMb }) {
    // 1. 获取图片内容
    let imageBuffer: Buffer;
    if (image.startsWith('http')) {
      imageBuffer = await fetchImage(image);
    } else {
      imageBuffer = await fs.readFile(image);
    }
    
    // 2. 检查并压缩
    const maxBytes = (maxBytesMb || 5) * 1024 * 1024;
    if (imageBuffer.length > maxBytes) {
      imageBuffer = await compressImage(imageBuffer, maxBytes);
    }
    
    // 3. 调用 Vision API
    const response = await llm.chat({
      model: model || 'claude-sonnet-4-20250514',
      messages: [{
        role: 'user',
        content: [
          {
            type: 'image',
            source: {
              type: 'base64',
              media_type: detectMimeType(imageBuffer),
              data: imageBuffer.toString('base64'),
            },
          },
          {
            type: 'text',
            text: prompt || 'Describe this image in detail.',
          },
        ],
      }],
    });
    
    return response.content;
  },
};
```

## 处理多模态消息

用户可能在一条消息里混合多种内容：

```typescript
// 消息预处理器
async function preprocessMessage(message: IncomingMessage): Promise<Message> {
  const parts: ContentPart[] = [];
  
  // 处理文本部分
  if (message.text) {
    parts.push({ type: 'text', text: message.text });
  }
  
  // 处理附件
  for (const attachment of message.attachments || []) {
    const processed = await preprocessInput({
      type: detectType(attachment),
      content: attachment.buffer,
      mimeType: attachment.mimeType,
    });
    
    // 如果有图片，直接添加
    if (processed.attachments) {
      parts.push(...processed.attachments);
    }
    
    // 如果转成了文本，添加文本
    if (processed.textContent && !processed.attachments) {
      parts.push({ type: 'text', text: processed.textContent });
    }
  }
  
  return {
    role: 'user',
    content: parts,
  };
}
```

## Token 管理与成本控制

多模态输入 token 消耗巨大，需要精细管理：

```typescript
interface TokenBudget {
  text: number;
  images: number;
  total: number;
}

class TokenBudgetManager {
  private budget: TokenBudget;
  
  constructor(modelContextSize: number) {
    this.budget = {
      total: modelContextSize,
      text: modelContextSize * 0.7,   // 70% 给文本
      images: modelContextSize * 0.3,  // 30% 给图片
    };
  }
  
  async fitContent(inputs: ProcessedInput[]): Promise<ProcessedInput[]> {
    let textTokens = 0;
    let imageTokens = 0;
    const fitted: ProcessedInput[] = [];
    
    for (const input of inputs) {
      const inputTextTokens = input.metadata.tokensEstimate;
      const inputImageTokens = input.attachments?.reduce(
        (sum, img) => sum + estimateImageTokens(img), 0
      ) || 0;
      
      // 检查是否超出预算
      if (textTokens + inputTextTokens > this.budget.text) {
        // 截断文本
        input.textContent = truncateToTokens(
          input.textContent, 
          this.budget.text - textTokens
        );
      }
      
      if (imageTokens + inputImageTokens > this.budget.images) {
        // 压缩或移除图片
        input.attachments = this.fitImages(
          input.attachments, 
          this.budget.images - imageTokens
        );
      }
      
      fitted.push(input);
      textTokens += inputTextTokens;
      imageTokens += inputImageTokens;
    }
    
    return fitted;
  }
  
  private fitImages(images: ImageAttachment[], budget: number): ImageAttachment[] {
    // 按重要性排序，保留能放进预算的
    const sorted = images.sort((a, b) => 
      estimateImageTokens(a) - estimateImageTokens(b)
    );
    
    const result: ImageAttachment[] = [];
    let used = 0;
    
    for (const img of sorted) {
      const tokens = estimateImageTokens(img);
      if (used + tokens <= budget) {
        result.push(img);
        used += tokens;
      }
    }
    
    return result;
  }
}
```

## 多模态的 Prompt Engineering

给 Vision 模型的 prompt 有特殊技巧：

```typescript
const visionPromptPatterns = {
  // 明确指示看图
  explicit: '请仔细观察图片中的内容，然后...',
  
  // 分步骤引导
  stepByStep: `请按以下步骤分析这张图片：
1. 首先描述图片的整体内容
2. 然后识别关键元素
3. 最后回答用户的问题`,
  
  // 聚焦特定区域
  focus: '请特别注意图片的右上角区域...',
  
  // OCR 场景
  ocr: '请提取图片中的所有文字，保持原有排版格式。',
  
  // 代码截图
  code: '这是一个代码截图，请识别代码内容并指出任何语法错误。',
  
  // 错误信息
  error: '这是一个错误信息的截图，请分析错误原因并提供解决方案。',
};
```

## 实战：构建多模态消息处理器

完整的处理器示例：

```typescript
// learn-claude-code 风格的多模态处理器
class MultiModalProcessor {
  private transcriber: Transcriber;
  private documentParser: DocumentParser;
  private imageOptimizer: ImageOptimizer;
  private tokenManager: TokenBudgetManager;
  
  async process(
    message: IncomingMessage, 
    context: ConversationContext
  ): Promise<ProcessedMessage> {
    const startTime = Date.now();
    const parts: ContentPart[] = [];
    const metadata: ProcessingMetadata = {
      processingTime: 0,
      inputTypes: [],
      totalTokens: 0,
    };
    
    // 1. 处理文本
    if (message.text) {
      parts.push({ type: 'text', text: message.text });
      metadata.inputTypes.push('text');
    }
    
    // 2. 处理每个附件
    for (const attachment of message.attachments || []) {
      try {
        const result = await this.processAttachment(attachment);
        
        if (result.parts) {
          parts.push(...result.parts);
        }
        
        metadata.inputTypes.push(result.type);
        metadata.totalTokens += result.tokens;
        
      } catch (error) {
        // 处理失败，添加错误提示但继续
        parts.push({
          type: 'text',
          text: `[无法处理附件: ${error.message}]`,
        });
      }
    }
    
    // 3. Token 预算管理
    const fitted = await this.tokenManager.fitContent(parts, context);
    
    metadata.processingTime = Date.now() - startTime;
    
    return {
      content: fitted,
      metadata,
    };
  }
  
  private async processAttachment(attachment: Attachment) {
    const type = this.detectType(attachment);
    
    switch (type) {
      case 'image':
        return this.processImage(attachment);
      case 'audio':
        return this.processAudio(attachment);
      case 'document':
        return this.processDocument(attachment);
      case 'video':
        return this.processVideo(attachment);
      default:
        throw new Error(`Unsupported attachment type: ${type}`);
    }
  }
}
```

## 关键要点

1. **预处理管道**：所有输入先经过标准化处理
2. **格式检测**：自动识别 MIME 类型，选择对应处理器
3. **Token 预算**：多模态内容消耗大，必须精细管理
4. **降级策略**：处理失败时优雅降级，不要崩溃
5. **元数据追踪**：记录处理耗时、原始格式、token 使用等
6. **Vision 优化**：图片在发送前要压缩、调整尺寸

## 下一步

- 实现自定义文档解析器
- 添加 OCR 作为图片处理的后备方案
- 探索原生多模态模型（如 Gemini 的音频支持）
- 实现多模态内容的缓存策略

---

*下节预告：Graceful Degradation - 优雅降级策略*
