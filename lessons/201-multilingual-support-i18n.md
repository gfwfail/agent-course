# 201 - Agent 多语言支持与 i18n（Multilingual Support & Internationalization）

> 让 Agent 无缝切换语言，跨语言工具调用，构建真正全球化的 AI 助理

---

## 为什么需要专门考虑多语言？

你以为 LLM 天生支持多语言，所以不需要额外处理？错！

生产中常见问题：
- 用户发中文，Agent 工具返回英文错误，LLM 用英文回复 ❌
- 同一个工具在不同语言下有不同的输入格式（日期、货币单位）
- 系统提示词只有英文，LLM 在中文会话中仍然用英文思考
- i18n 消息硬编码在代码里，改文案要改代码

**核心原则：语言是会话状态，不是 LLM 能力**

---

## 架构分层

```
用户输入（任意语言）
    ↓
[语言检测层] → 检测 + 设置 sessionLocale
    ↓
[上下文注入层] → 把 locale 注入 system prompt
    ↓
LLM 处理（在用户语言中思考+响应）
    ↓
[工具层] → 工具输入/输出语言规范化
    ↓
[i18n 消息层] → 系统消息模板本地化
    ↓
用户看到的响应（目标语言）
```

---

## 实现：TypeScript 版

### 1. 语言检测与会话 locale 管理

```typescript
// i18n/language-detector.ts
import Anthropic from "@anthropic-ai/sdk";

type Locale = "zh" | "en" | "ja" | "ko" | "es" | "fr" | "de";

interface SessionLocale {
  primary: Locale;
  confidence: number;
  detectedAt: number;
  overrideBy?: "user" | "system"; // 用户显式设置 vs 自动检测
}

class LanguageDetector {
  // 快速规则检测（零成本，覆盖常见场景）
  private ruleBasedDetect(text: string): Locale | null {
    // CJK 字符
    if (/[\u4e00-\u9fa5]/.test(text)) return "zh";
    if (/[\u3040-\u30ff\u3400-\u4dbf]/.test(text)) return "ja";
    if (/[\uac00-\ud7af]/.test(text)) return "ko";

    // 西班牙语特征词
    if (/\b(hola|gracias|por favor|cómo|qué)\b/i.test(text)) return "es";

    // ASCII 占比高 → 大概率英文
    const asciiRatio =
      text.replace(/[^\x00-\x7F]/g, "").length / text.length;
    if (asciiRatio > 0.9 && text.length > 10) return "en";

    return null; // 不确定，交给 LLM
  }

  // 用 Haiku 做精准检测（仅当规则无法判断时）
  async detect(
    text: string,
    client: Anthropic
  ): Promise<SessionLocale> {
    const ruleBased = this.ruleBasedDetect(text);
    if (ruleBased) {
      return { primary: ruleBased, confidence: 0.95, detectedAt: Date.now() };
    }

    // Haiku 检测（成本极低：~50 tokens）
    const response = await client.messages.create({
      model: "claude-haiku-4-5",
      max_tokens: 20,
      messages: [
        {
          role: "user",
          content: `Detect language. Reply with ISO code only (zh/en/ja/ko/es/fr/de/other): "${text.slice(0, 200)}"`,
        },
      ],
    });

    const detected = (
      (response.content[0] as { text: string }).text.trim().toLowerCase() as Locale
    ) ?? "en";

    return {
      primary: detected,
      confidence: 0.85,
      detectedAt: Date.now(),
    };
  }
}
```

### 2. i18n 消息模板

```typescript
// i18n/messages.ts
type MessageKey =
  | "tool_error"
  | "tool_timeout"
  | "needs_approval"
  | "task_complete"
  | "thinking";

const messages: Record<Locale, Record<MessageKey, string>> = {
  zh: {
    tool_error: "工具执行失败：{{error}}",
    tool_timeout: "操作超时（{{seconds}}秒），已自动取消",
    needs_approval: "此操作需要确认：{{action}}",
    task_complete: "任务完成 ✅",
    thinking: "思考中…",
  },
  en: {
    tool_error: "Tool failed: {{error}}",
    tool_timeout: "Operation timed out after {{seconds}}s",
    needs_approval: "Approval required for: {{action}}",
    task_complete: "Task complete ✅",
    thinking: "Thinking…",
  },
  ja: {
    tool_error: "ツールエラー：{{error}}",
    tool_timeout: "操作がタイムアウトしました（{{seconds}}秒）",
    needs_approval: "確認が必要です：{{action}}",
    task_complete: "完了 ✅",
    thinking: "考え中…",
  },
  // ... 其他语言
  ko: {
    tool_error: "도구 오류: {{error}}",
    tool_timeout: "작업 시간 초과 ({{seconds}}초)",
    needs_approval: "승인 필요: {{action}}",
    task_complete: "완료 ✅",
    thinking: "생각 중…",
  },
  es: {
    tool_error: "Error de herramienta: {{error}}",
    tool_timeout: "Tiempo de espera agotado ({{seconds}}s)",
    needs_approval: "Aprobación requerida: {{action}}",
    task_complete: "Tarea completada ✅",
    thinking: "Pensando…",
  },
  fr: {
    tool_error: "Erreur outil : {{error}}",
    tool_timeout: "Délai dépassé ({{seconds}}s)",
    needs_approval: "Approbation requise : {{action}}",
    task_complete: "Tâche terminée ✅",
    thinking: "Réflexion…",
  },
  de: {
    tool_error: "Werkzeugfehler: {{error}}",
    tool_timeout: "Zeitüberschreitung nach {{seconds}}s",
    needs_approval: "Genehmigung erforderlich: {{action}}",
    task_complete: "Aufgabe abgeschlossen ✅",
    thinking: "Denke nach…",
  },
};

export function t(
  locale: Locale,
  key: MessageKey,
  vars?: Record<string, string | number>
): string {
  const template =
    messages[locale]?.[key] ?? messages["en"][key]; // fallback to en

  if (!vars) return template;

  return Object.entries(vars).reduce(
    (msg, [k, v]) => msg.replace(`{{${k}}}`, String(v)),
    template
  );
}

// 用法
// t("zh", "tool_error", { error: "连接超时" }) → "工具执行失败：连接超时"
// t("en", "tool_timeout", { seconds: 30 }) → "Operation timed out after 30s"
```

### 3. locale 感知的 System Prompt 注入

```typescript
// i18n/locale-middleware.ts

const LOCALE_SYSTEM_INSTRUCTIONS: Record<Locale, string> = {
  zh: "请始终用中文回复。工具调用的输入输出如为英文，请在向用户展示前翻译成中文。",
  en: "Always respond in English.",
  ja: "常に日本語で返答してください。",
  ko: "항상 한국어로 응답하세요.",
  es: "Siempre responde en español.",
  fr: "Répondez toujours en français.",
  de: "Antworten Sie immer auf Deutsch.",
};

class LocaleAwareAgent {
  private detector = new LanguageDetector();
  private sessionLocale: SessionLocale | null = null;

  async buildSystemPrompt(basePrompt: string): Promise<string> {
    if (!this.sessionLocale) return basePrompt;

    const localeInstruction =
      LOCALE_SYSTEM_INSTRUCTIONS[this.sessionLocale.primary];

    return `${basePrompt}

## Language Instructions
${localeInstruction}
Current session locale: ${this.sessionLocale.primary}`;
  }

  async processMessage(
    userMessage: string,
    client: Anthropic,
    tools: Anthropic.Tool[]
  ): Promise<string> {
    // Step 1: 检测/更新语言（首次或每5轮更新一次）
    const shouldDetect =
      !this.sessionLocale ||
      Date.now() - this.sessionLocale.detectedAt > 5 * 60 * 1000;

    if (shouldDetect && this.sessionLocale?.overrideBy !== "user") {
      this.sessionLocale = await this.detector.detect(userMessage, client);
      console.log(
        `[i18n] Detected: ${this.sessionLocale.primary} (${Math.round(this.sessionLocale.confidence * 100)}%)`
      );
    }

    // Step 2: 构建 locale 感知的 system prompt
    const systemPrompt = await this.buildSystemPrompt(
      "You are a helpful assistant."
    );

    // Step 3: 调用 LLM（locale 信息已注入 system prompt）
    const response = await client.messages.create({
      model: "claude-sonnet-4-5",
      max_tokens: 8096,
      system: systemPrompt,
      tools,
      messages: [{ role: "user", content: userMessage }],
    });

    return (response.content[0] as { text: string }).text;
  }

  // 用户显式切换语言
  setLocale(locale: Locale): void {
    this.sessionLocale = {
      primary: locale,
      confidence: 1.0,
      detectedAt: Date.now(),
      overrideBy: "user",
    };
  }
}
```

### 4. 工具输出本地化包装

```typescript
// i18n/tool-output-localizer.ts

// 工具返回的内容通常是英文，需要本地化后再给用户看
async function localizeToolOutput(
  output: string,
  targetLocale: Locale,
  client: Anthropic
): Promise<string> {
  if (targetLocale === "en") return output; // 英文不需要翻译

  // 只翻译面向用户的输出（不翻译结构化数据）
  if (isStructuredData(output)) return output;

  const response = await client.messages.create({
    model: "claude-haiku-4-5",
    max_tokens: output.length * 2 + 100,
    messages: [
      {
        role: "user",
        content: `Translate to ${targetLocale}. Preserve code blocks, URLs, and technical terms unchanged:\n\n${output}`,
      },
    ],
  });

  return (response.content[0] as { text: string }).text;
}

function isStructuredData(text: string): boolean {
  try {
    JSON.parse(text);
    return true;
  } catch {
    // 检查是否是代码块
    return (
      text.trim().startsWith("```") ||
      text.includes("{") ||
      text.includes("<html")
    );
  }
}
```

---

## 实现：Python 版

```python
# i18n_agent.py
import re
import json
from typing import Optional, Literal
from dataclasses import dataclass, field
import anthropic

Locale = Literal["zh", "en", "ja", "ko", "es", "fr", "de"]

MESSAGES = {
    "zh": {
        "tool_error": "工具执行失败：{error}",
        "task_complete": "任务完成 ✅",
        "thinking": "思考中…",
    },
    "en": {
        "tool_error": "Tool failed: {error}",
        "task_complete": "Task complete ✅",
        "thinking": "Thinking…",
    },
}

def t(locale: str, key: str, **kwargs) -> str:
    template = MESSAGES.get(locale, MESSAGES["en"]).get(
        key, MESSAGES["en"].get(key, key)
    )
    return template.format(**kwargs)


@dataclass
class SessionLocale:
    primary: Locale = "en"
    confidence: float = 0.0
    override_by: Optional[str] = None


class I18nAgent:
    def __init__(self):
        self.client = anthropic.Anthropic()
        self.session_locale: Optional[SessionLocale] = None

    def _rule_detect(self, text: str) -> Optional[Locale]:
        if re.search(r'[\u4e00-\u9fa5]', text):
            return "zh"
        if re.search(r'[\u3040-\u30ff]', text):
            return "ja"
        if re.search(r'[\uac00-\ud7af]', text):
            return "ko"
        ascii_ratio = len(re.sub(r'[^\x00-\x7F]', '', text)) / max(len(text), 1)
        if ascii_ratio > 0.9 and len(text) > 10:
            return "en"
        return None

    def detect_language(self, text: str) -> SessionLocale:
        rule_result = self._rule_detect(text)
        if rule_result:
            return SessionLocale(primary=rule_result, confidence=0.95)

        # Haiku 精准检测
        response = self.client.messages.create(
            model="claude-haiku-4-5",
            max_tokens=20,
            messages=[{
                "role": "user",
                "content": f'Language code (zh/en/ja/ko/es/fr/de): "{text[:200]}"'
            }]
        )
        detected = response.content[0].text.strip().lower()
        return SessionLocale(primary=detected, confidence=0.85)

    def build_system(self, base: str) -> str:
        if not self.session_locale:
            return base
        
        locale_map = {
            "zh": "请始终用中文回复用户，但保留代码块和技术术语原文。",
            "en": "Always reply in English.",
            "ja": "常に日本語で返答してください。",
            "ko": "항상 한국어로 응답하세요.",
        }
        instruction = locale_map.get(self.session_locale.primary, "")
        return f"{base}\n\n{instruction}"

    def chat(self, user_message: str) -> str:
        # 自动检测语言
        if not self.session_locale or self.session_locale.override_by != "user":
            self.session_locale = self.detect_language(user_message)

        system = self.build_system("You are a helpful assistant.")
        
        response = self.client.messages.create(
            model="claude-sonnet-4-5",
            max_tokens=4096,
            system=system,
            messages=[{"role": "user", "content": user_message}]
        )
        return response.content[0].text


# 使用示例
if __name__ == "__main__":
    agent = I18nAgent()
    
    # 中文输入 → 自动检测 → 中文回复
    print(agent.chat("今天天气怎么样？"))
    
    # 日文输入 → 自动检测 → 日文回复
    print(agent.chat("東京の天気は？"))
    
    # 用户显式切换
    agent.session_locale = SessionLocale(primary="en", override_by="user")
    print(agent.chat("今天天气怎么样？"))  # → 英文回复
```

---

## OpenClaw 中的多语言支持

OpenClaw 本身的多语言设计是很好的参考：

```yaml
# config.yaml - channel 级别的 locale 配置
channels:
  telegram:
    locale: "zh"  # 频道默认语言
  discord:
    locale: "en"

# SOUL.md / USER.md 中静态声明语言偏好
# USER.md:
# - 语言：中文为主
# - 时区：GMT+11
```

```typescript
// OpenClaw 在 system prompt 注入时会读取 channel locale
// 对应的 SOUL.md 里：
// "语言风格：专业、高效、简洁"
// → 这是静态语言策略（Lesson 189 Natural Language Policy Engine 的一种）

// 动态检测则在 runtime context 中体现：
// Runtime: channel=telegram | ...
// Agent 看到 channel=telegram → 知道大概率是中文用户
```

---

## 常见坑与最佳实践

### ❌ 坑 1：翻译工具输出时破坏结构化数据
```typescript
// 错误：把 JSON 也翻译了
await localize('{"status": "成功", "code": 200}', "zh")
// → LLM 可能返回 '{"状态": "成功", "代码": 200}' ← 字段名变了！

// 正确：检测是否是结构化数据，只翻译人类可读部分
if (isStructuredData(output)) return output;
```

### ❌ 坑 2：每条消息都调用 LLM 检测语言
```typescript
// 错误：每轮都检测，浪费 token
const locale = await detectWithLLM(message); // 每次 50 tokens

// 正确：规则检测优先，只在不确定时用 LLM；检测结果缓存到 session
const locale = ruleDetect(message) ?? await detectWithLLM(message);
```

### ❌ 坑 3：工具内部错误消息语言不一致
```typescript
// 错误：工具直接 throw Error("Connection failed")
// 用户看到的是英文错误，但 Agent 在用中文回复

// 正确：工具返回结构化错误，上层用 t() 本地化
return { success: false, errorKey: "connection_failed", details: "..." };
// 上层：t(sessionLocale, "connection_failed")
```

### ✅ 最佳实践清单

| 实践 | 说明 |
|------|------|
| 规则检测优先 | CJK 字符规则检测准确率 99%，零 token |
| Session-level locale | 每条消息不重新检测，会话内保持一致 |
| 用户显式优先 | `/lang en` 命令覆盖自动检测，标记 `overrideBy: "user"` |
| 系统消息 i18n | 错误/提示用模板文件，不硬编码 |
| 结构化数据不翻译 | JSON/代码块保持原文 |
| 技术术语保留 | "API", "Token", "LLM" 等不翻译 |

---

## 数据流全景图

```
用户: "帮我查一下天气"
        ↓
[规则检测] → 匹配 CJK → locale = "zh" (0.95)
        ↓
[System Prompt 注入] → "请始终用中文回复"
        ↓
LLM: 工具调用 get_weather(city="Beijing")
        ↓
工具返回: "Sunny, 22°C, Wind: NW 15km/h"  ← 英文
        ↓
[工具输出本地化] → "晴天，22°C，西北风 15 公里/小时"
        ↓
LLM 最终回复: "北京今天天气晴好，气温 22°C，西北风…"  ← 中文 ✅
        ↓
用户看到: 一切都是中文 🎉
```

---

## 小结

多语言 Agent 的核心是**把语言当作会话状态来管理**，而不是依赖 LLM "自然"切换：

1. **检测层** → 规则优先，LLM 兜底，结果缓存
2. **注入层** → locale 写入 system prompt，LLM 响应语言可预测
3. **工具层** → 输出本地化，跳过结构化数据
4. **消息层** → i18n 模板，告别硬编码字符串
5. **用户控制** → 显式切换优先于自动检测

OpenClaw 的实践：`SOUL.md` 声明语言风格 + `USER.md` 记录用户偏好 + 运行时 `channel` 信息三层叠加，实现了一个轻量但有效的 i18n 系统。

---

*下一课预告：Agent 工具调用批处理优化（Batch Tool Call Optimization）——把串行工具调用压缩成批次，降低往返延迟与 Token 消耗*
