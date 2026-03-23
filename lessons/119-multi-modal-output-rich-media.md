# 119 - Agent 多模态输出与富媒体响应（Multi-Modal Output & Rich Media Response）

> **核心问题**：用户说「给我看张图」「念给我听」「导出成 PDF」——Agent 怎么生成和交付多种形式的输出？

---

## 为什么需要多模态输出？

LLM 的输入已经支持多模态（文本 + 图像 + 音频），  
但**输出**通常还是纯文本 JSON 字符串。

现实里用户要的是：
- 🔊 **语音**：念天气预报、读新闻、讲故事
- 🖼️ **图片**：生成海报、截图、图表
- 📄 **文件**：导出 PDF、CSV、Excel
- 🎬 **视频片段**：监控截帧、教程录屏
- 📊 **交互式 UI**：动态表格、可点击卡片

**痛点**：每种输出类型都需要不同的工具 + 渲染管道，LLM 自己搞不定——它只能发指令。

---

## 核心架构：输出类型注册表

```typescript
// output-registry.ts
type OutputKind = 'text' | 'audio' | 'image' | 'file' | 'ui_card' | 'stream';

interface OutputRenderer<T = unknown> {
  kind: OutputKind;
  render(payload: T, channel: ChannelContext): Promise<DeliveredOutput>;
  supportedChannels: string[]; // 'telegram' | 'discord' | 'terminal' | 'web'
}

interface DeliveredOutput {
  kind: OutputKind;
  deliveryId: string;
  bytesSent?: number;
  durationMs?: number;
}

class OutputRegistry {
  private renderers = new Map<OutputKind, OutputRenderer>();

  register<T>(renderer: OutputRenderer<T>) {
    this.renderers.set(renderer.kind, renderer);
  }

  async deliver(kind: OutputKind, payload: unknown, channel: ChannelContext) {
    const renderer = this.renderers.get(kind);
    if (!renderer) throw new Error(`No renderer for output kind: ${kind}`);
    if (!renderer.supportedChannels.includes(channel.platform)) {
      // Graceful fallback: 不支持的平台降级为文本
      return this.renderers.get('text')!.render(
        { text: `[${kind} output not supported on ${channel.platform}]` },
        channel
      );
    }
    return renderer.render(payload, channel);
  }
}
```

---

## 实战：OpenClaw 的多模态工具

OpenClaw 内置了几个典型的多模态输出工具，看看它们怎么设计的：

### 1. TTS 工具（文本 → 语音）

```typescript
// openclaw tts tool
{
  name: "tts",
  description: "Convert text to speech. Audio is delivered automatically.",
  inputSchema: {
    type: "object",
    properties: {
      text: { type: "string" },
      channel: { type: "string", description: "Optional channel id" }
    },
    required: ["text"]
  }
}
```

**关键设计**：
- LLM 只需要提供「文本内容」
- 音频文件生成、上传、推送全在 tool handler 里搞定
- 返回结果里带 `AUTO_DELIVER` 标记，框架自动发送给用户
- LLM 收到 tool_result 后应回复 `NO_REPLY`（避免重复）

```typescript
// tts handler 内部逻辑（简化）
async function handleTTS(params: { text: string; channel?: string }) {
  // 1. 调用 ElevenLabs / OpenAI TTS API
  const audioBuffer = await ttsProvider.synthesize(params.text, {
    voice: getUserVoicePreference(),
    format: 'ogg_opus' // Telegram 偏好格式
  });

  // 2. 写临时文件
  const tmpPath = `/tmp/tts-${Date.now()}.ogg`;
  await fs.writeFile(tmpPath, audioBuffer);

  // 3. 通过 channel 插件发送
  await channelPlugin.sendVoice(tmpPath, params.channel);

  // 4. 清理临时文件
  await fs.unlink(tmpPath);

  return { delivered: true, hint: 'NO_REPLY' };
}
```

---

### 2. Canvas 工具（Agent → 交互式 UI）

OpenClaw 的 `canvas` 工具让 Agent 可以渲染 HTML/React UI 到画布：

```typescript
// 在 Agent 系统提示词里声明
{
  name: "canvas",
  description: "Present/eval/snapshot the Canvas — renders HTML/React UI",
  inputSchema: {
    properties: {
      action: { enum: ["present", "hide", "navigate", "eval", "snapshot"] },
      url: { description: "URL to navigate canvas to" },
      javaScript: { description: "JS to evaluate in canvas context" }
    }
  }
}
```

**实际用法**：
```typescript
// Agent 想展示一个数据表格：
await canvas.present({
  action: "eval",
  javaScript: `
    document.body.innerHTML = \`
      <table style="font-family: monospace">
        <tr><th>指标</th><th>值</th></tr>
        <tr><td>今日收入</td><td>¥12,450</td></tr>
        <tr><td>开箱次数</td><td>3,218</td></tr>
      </table>
    \`
  `
});

// 然后截图发给用户
const screenshot = await canvas.screenshot({ type: 'png' });
await channel.sendImage(screenshot.buffer);
```

---

### 3. 文件输出（生成 → 上传 → 发送）

```typescript
// file-output-renderer.ts
class FileOutputRenderer implements OutputRenderer<FilePayload> {
  kind = 'file' as const;
  supportedChannels = ['telegram', 'discord', 'web'];

  async render(payload: FilePayload, channel: ChannelContext) {
    const { content, filename, mimeType } = payload;
    
    // 根据格式选择序列化方式
    let buffer: Buffer;
    if (mimeType === 'text/csv') {
      buffer = Buffer.from(this.toCSV(content));
    } else if (mimeType === 'application/json') {
      buffer = Buffer.from(JSON.stringify(content, null, 2));
    } else if (mimeType === 'application/pdf') {
      buffer = await this.renderPDF(content); // puppeteer / pdfkit
    }

    // 上传到临时存储（R2 / S3）或直接 multipart 发送
    const url = await this.upload(buffer, filename);

    await channel.sendDocument({ url, filename, caption: payload.caption });
    return { kind: this.kind, deliveryId: url, bytesSent: buffer.length };
  }
}
```

---

## 多模态决策：LLM 怎么选输出类型？

关键是在 **系统提示词** 里给 LLM 明确指导：

```typescript
// dynamic system prompt 注入输出策略
function buildOutputPolicy(channel: ChannelContext): string {
  const caps = CHANNEL_CAPABILITIES[channel.platform];
  
  return `
## 输出格式规则（必须遵守）
平台: ${channel.platform}

支持的输出类型:
${caps.supportsVoice ? '- 语音: 适合播报、朗读、讲故事场景，使用 tts 工具' : ''}
${caps.supportsImages ? '- 图片: 适合图表、截图、可视化数据，使用 canvas/screenshot 工具' : ''}
${caps.supportsFiles ? '- 文件: 适合导出数据，使用 file-export 工具' : ''}
- 文本: 所有场景的基础输出

决策原则:
1. 用户明确要求 → 优先满足
2. 数据量 > 5行表格 → 考虑图片或文件
3. "念/读/播报" 关键词 → 使用 tts
4. 不确定时 → 默认文本（最安全）
  `.trim();
}
```

---

## 渐进式多模态：先文本后增强

```typescript
// progressive-output.ts
class ProgressiveOutputAgent {
  async respond(userMessage: string, channel: ChannelContext) {
    // Step 1: 立即发送文字回复（低延迟感知）
    const textResponse = await this.llm.quickReply(userMessage);
    await channel.send(textResponse.text);

    // Step 2: 异步增强输出（不阻塞主流程）
    if (textResponse.enhancements?.length) {
      for (const enhancement of textResponse.enhancements) {
        await this.deliverEnhancement(enhancement, channel);
      }
    }
  }

  private async deliverEnhancement(
    enhancement: { kind: OutputKind; data: unknown },
    channel: ChannelContext
  ) {
    try {
      await outputRegistry.deliver(enhancement.kind, enhancement.data, channel);
    } catch (err) {
      // 增强失败不影响主回复，只记录日志
      logger.warn('Enhancement delivery failed', { kind: enhancement.kind, err });
    }
  }
}
```

---

## OpenClaw 实战：讲故事 + 语音

这是 OpenClaw 里「声音讲故事」的完整流程：

```typescript
// 用户说: "用语音给我讲一个程序员的睡前故事"

// 1. LLM 生成故事文本
const story = await llm.complete({
  messages: [{ role: 'user', content: '用语音给我讲一个程序员的睡前故事' }],
  tools: [ttsToolDef, canvasToolDef],
  system: outputPolicy
});

// 2. LLM 调用 tts 工具
// tool_use: { name: "tts", input: { text: "从前有一个程序员..." } }

// 3. Handler 生成音频并推送
await ttsHandler({ text: story, channel: 'telegram' });

// 4. LLM 收到成功结果后回复 NO_REPLY
// 用户收到: 🎵 一段语音消息
```

---

## 平台能力矩阵

| 输出类型 | Telegram | Discord | Terminal | Web |
|----------|----------|---------|----------|-----|
| 文本 | ✅ | ✅ | ✅ | ✅ |
| 语音 | ✅ | ✅ | ❌ | ✅ |
| 图片 | ✅ | ✅ | ❌ | ✅ |
| 文件 | ✅ | ✅ | ✅(path) | ✅ |
| 视频 | ✅ | ✅ | ❌ | ✅ |
| Canvas UI | ❌ | ❌ | ❌ | ✅ |
| 内联按钮 | ✅ | ✅ | ❌ | ✅ |

> 设计原则：**能力注册 + 优雅降级**，不支持的类型自动 fallback 到文本

---

## 关键设计要点

1. **LLM 只管「发出指令」**，不管「怎么渲染」  
   → 工具输入尽量简单（文本/URL），复杂的格式转换在 handler 里做

2. **AUTO_DELIVER + NO_REPLY 模式**  
   → 多媒体内容由框架自动投递，LLM 不应再发第二条消息

3. **渐进式交付**  
   → 先发文字（低延迟），再异步补充图片/音频（提升体验）

4. **平台能力注册表**  
   → 在系统提示词里告诉 LLM 当前平台支持什么，避免无效工具调用

5. **临时文件清理**  
   → 音频/图片生成后记得 `unlink`，防止磁盘泄漏

---

## 延伸阅读

- OpenClaw `tts` 工具实现：`openclaw/src/plugins/tts/`
- Telegram Bot API - sendVoice / sendDocument / sendPhoto
- ElevenLabs TTS API：`https://api.elevenlabs.io/v1/text-to-speech`
- OpenAI TTS：`https://api.openai.com/v1/audio/speech`
- Puppeteer PDF 生成：`page.pdf({ format: 'A4' })`
