# 215 - Agent 多步骤结构化数据收集（Multi-Step Structured Data Collection）

> 表单是 Agent 最容易被忽视的超能力。

---

## 问题背景

很多任务本质上是「结构化信息收集」：

- 创建工单（需要标题、描述、优先级、影响范围）
- 用户注册引导（姓名、邮箱、公司、使用场景）
- 报销申请（项目、金额、日期、发票、审批人）
- 代码 Review 模板（PR 链接、变更描述、自测情况）

传统表单的问题：用户一次性面对 10 个字段，填写体验差，放弃率高。

Agent 的优势：像人类顾问一样，**一次只问一个问题**，根据已知信息动态跳过或预填字段，验证后再继续。

---

## 核心设计

```
用户 ─────────────────────────────────► 
        ▲                              │
        │  [下一个问题]                 │ [用户输入]
        │                              ▼
   QuestionRenderer             AnswerParser & Validator
        ▲                              │
        │                              ▼
   FormPlanner ◄─────────── FormState (已收集字段 + 进度)
        │
        ▼
   FormSchema (字段定义 + 验证规则 + 条件跳转)
```

---

## TypeScript 实现

### 1. 表单 Schema 定义

```typescript
// form-schema.ts
import { z } from 'zod';

export type FieldType = 'text' | 'number' | 'email' | 'select' | 'multiselect' | 'confirm';

export interface FormField {
  id: string;
  label: string;
  type: FieldType;
  required: boolean;
  options?: string[];           // select/multiselect 用
  validation?: z.ZodTypeAny;   // Zod schema 校验
  skipIf?: (state: FormState) => boolean;  // 条件跳过
  prefill?: (state: FormState) => string | undefined; // 自动预填
  hint?: string;                // 给用户的提示文字
}

export interface FormSchema {
  id: string;
  title: string;
  fields: FormField[];
  onComplete: (data: Record<string, unknown>) => Promise<void>;
}

export type FormState = {
  schemaId: string;
  collected: Record<string, unknown>;  // 已收集的字段值
  currentFieldIndex: number;
  status: 'in_progress' | 'completed' | 'abandoned';
  startedAt: number;
  lastUpdatedAt: number;
};

// 示例：Bug 报告表单
export const BugReportSchema: FormSchema = {
  id: 'bug-report',
  title: 'Bug 报告',
  fields: [
    {
      id: 'title',
      label: '用一句话描述这个 Bug',
      type: 'text',
      required: true,
      validation: z.string().min(10, '描述至少 10 个字'),
    },
    {
      id: 'severity',
      label: '严重程度',
      type: 'select',
      required: true,
      options: ['P0 - 生产崩溃', 'P1 - 功能不可用', 'P2 - 功能受损', 'P3 - 轻微问题'],
    },
    {
      id: 'reproduction_steps',
      label: '如何复现？（步骤描述）',
      type: 'text',
      required: true,
      hint: '例如：1. 打开设置页面 2. 点击保存 3. 出现 500 错误',
    },
    {
      id: 'affected_users',
      label: '影响用户数量（估计）',
      type: 'number',
      required: false,
      skipIf: (state) => {
        // P3 问题跳过影响用户数询问
        const sev = state.collected['severity'] as string;
        return sev?.startsWith('P3');
      },
    },
    {
      id: 'environment',
      label: '发生环境',
      type: 'select',
      required: true,
      options: ['生产环境', '预发布环境', '开发环境', '所有环境'],
    },
    {
      id: 'confirm',
      label: (state: FormState) => {
        const title = state.collected['title'];
        const sev = state.collected['severity'];
        return `确认提交：「${title}」(${sev})？`;
      } as unknown as string,
      type: 'confirm',
      required: true,
    },
  ],
  onComplete: async (data) => {
    console.log('Bug 报告已提交：', data);
    // 调用 API 创建 Issue
  },
};
```

### 2. 表单引擎核心

```typescript
// form-engine.ts
import Anthropic from '@anthropic-ai/sdk';
import { FormSchema, FormState, FormField } from './form-schema';

export class FormEngine {
  private client: Anthropic;
  private states: Map<string, FormState> = new Map();

  constructor() {
    this.client = new Anthropic();
  }

  // 启动一个新表单会话
  startForm(sessionId: string, schema: FormSchema): string {
    const state: FormState = {
      schemaId: schema.id,
      collected: {},
      currentFieldIndex: 0,
      status: 'in_progress',
      startedAt: Date.now(),
      lastUpdatedAt: Date.now(),
    };
    this.states.set(sessionId, state);

    // 跳过应跳过的初始字段
    this.advanceToNextField(state, schema);
    return this.renderQuestion(schema, state);
  }

  // 处理用户的回答
  async handleAnswer(
    sessionId: string,
    schema: FormSchema,
    userInput: string
  ): Promise<{ message: string; completed: boolean; data?: Record<string, unknown> }> {
    const state = this.states.get(sessionId);
    if (!state || state.status !== 'in_progress') {
      return { message: '没有进行中的表单。', completed: false };
    }

    const field = this.getCurrentField(schema, state);
    if (!field) {
      return { message: '表单已完成。', completed: true };
    }

    // 1. 用 LLM 解析用户输入
    const parsed = await this.parseAnswer(field, userInput);
    if (!parsed.valid) {
      return {
        message: `❌ ${parsed.error}\n\n请重新回答：${field.label}${field.hint ? `\n💡 提示：${field.hint}` : ''}`,
        completed: false,
      };
    }

    // 2. 存储答案
    state.collected[field.id] = parsed.value;
    state.lastUpdatedAt = Date.now();

    // 3. 移动到下一个字段
    state.currentFieldIndex++;
    this.advanceToNextField(state, schema);

    // 4. 检查是否完成
    const nextField = this.getCurrentField(schema, state);
    if (!nextField) {
      state.status = 'completed';
      await schema.onComplete(state.collected);
      return {
        message: '✅ 已提交！感谢填写。',
        completed: true,
        data: state.collected,
      };
    }

    // 5. 渲染下一个问题
    return {
      message: this.renderQuestion(schema, state),
      completed: false,
    };
  }

  // 用 LLM 智能解析用户输入（处理自然语言回答）
  private async parseAnswer(
    field: FormField,
    input: string
  ): Promise<{ valid: boolean; value?: unknown; error?: string }> {
    if (field.type === 'confirm') {
      const lower = input.toLowerCase();
      const yes = ['是', '确认', 'yes', 'y', '对', '好'].some(w => lower.includes(w));
      const no = ['否', '取消', 'no', 'n', '不', '算了'].some(w => lower.includes(w));
      if (yes) return { valid: true, value: true };
      if (no) return { valid: false, error: '操作已取消' };
    }

    if (field.type === 'select' && field.options) {
      // 用 LLM 把自然语言映射到选项
      const response = await this.client.messages.create({
        model: 'claude-haiku-4-5',
        max_tokens: 50,
        system: `你是表单助手。用户要从以下选项中选择：\n${field.options.map((o, i) => `${i + 1}. ${o}`).join('\n')}\n\n只返回匹配的选项文字，不要解释。如果无法匹配，返回 "INVALID"。`,
        messages: [{ role: 'user', content: input }],
      });
      const matched = response.content[0].type === 'text' ? response.content[0].text.trim() : '';
      if (matched === 'INVALID' || !field.options.includes(matched)) {
        return {
          valid: false,
          error: `请从以下选项中选择：\n${field.options.map((o, i) => `${i + 1}. ${o}`).join('\n')}`,
        };
      }
      return { valid: true, value: matched };
    }

    if (field.type === 'number') {
      const num = parseFloat(input.replace(/[^0-9.]/g, ''));
      if (isNaN(num)) return { valid: false, error: '请输入一个数字' };
      return { valid: true, value: num };
    }

    // 默认 text 校验
    if (field.validation) {
      const result = field.validation.safeParse(input);
      if (!result.success) {
        return { valid: false, error: result.error.errors[0].message };
      }
    }

    return { valid: true, value: input };
  }

  private getCurrentField(schema: FormSchema, state: FormState): FormField | undefined {
    return schema.fields[state.currentFieldIndex];
  }

  // 跳过满足 skipIf 条件的字段，并尝试预填
  private advanceToNextField(state: FormState, schema: FormSchema): void {
    while (state.currentFieldIndex < schema.fields.length) {
      const field = schema.fields[state.currentFieldIndex];
      
      // 检查 skipIf
      if (field.skipIf?.(state)) {
        state.currentFieldIndex++;
        continue;
      }
      
      // 尝试预填
      if (field.prefill) {
        const prefilled = field.prefill(state);
        if (prefilled !== undefined) {
          state.collected[field.id] = prefilled;
          state.currentFieldIndex++;
          continue;
        }
      }
      
      break; // 停在当前字段
    }
  }

  private renderQuestion(schema: FormSchema, state: FormState): string {
    const field = this.getCurrentField(schema, state);
    if (!field) return '表单已完成';

    const progress = `[${state.currentFieldIndex + 1}/${schema.fields.length}]`;
    let question = `${progress} **${field.label}**`;

    if (field.type === 'select' && field.options) {
      question += '\n' + field.options.map((o, i) => `${i + 1}. ${o}`).join('\n');
    }

    if (field.hint) {
      question += `\n💡 ${field.hint}`;
    }

    if (!field.required) {
      question += '\n（可选，直接发送「跳过」略过）';
    }

    return question;
  }

  // 保存表单进度（支持中断后续传）
  exportState(sessionId: string): FormState | undefined {
    return this.states.get(sessionId);
  }

  // 恢复表单进度
  importState(sessionId: string, state: FormState): void {
    this.states.set(sessionId, state);
  }
}
```

### 3. OpenClaw 集成

```typescript
// openclaw-form-handler.ts
// 在 OpenClaw 的消息处理器中集成表单引擎

const formEngine = new FormEngine();
const activeForms = new Map<string, FormSchema>(); // sessionId → schema

export async function handleMessage(
  sessionId: string,
  userInput: string
): Promise<string> {
  // 检查是否有进行中的表单
  const activeSchema = activeForms.get(sessionId);
  
  if (activeSchema) {
    // 处理表单输入
    const result = await formEngine.handleAnswer(sessionId, activeSchema, userInput);
    
    if (result.completed) {
      activeForms.delete(sessionId);
      // 持久化完成数据
      await saveFormData(sessionId, result.data!);
    }
    
    return result.message;
  }

  // 检测是否需要启动表单
  if (userInput.includes('提交 bug') || userInput.includes('报告问题')) {
    activeForms.set(sessionId, BugReportSchema);
    const firstQuestion = formEngine.startForm(sessionId, BugReportSchema);
    return `好的，我来帮你提交 Bug 报告。\n\n${firstQuestion}`;
  }

  // 普通对话处理...
  return await normalAgentResponse(userInput);
}
```

---

## Python 实现

```python
# form_engine.py
from dataclasses import dataclass, field
from typing import Any, Callable, Optional
import anthropic
import json

@dataclass
class FormField:
    id: str
    label: str
    field_type: str  # 'text' | 'number' | 'select' | 'confirm'
    required: bool = True
    options: list[str] = field(default_factory=list)
    skip_if: Optional[Callable[['FormState'], bool]] = None
    prefill: Optional[Callable[['FormState'], Optional[str]]] = None
    hint: Optional[str] = None

@dataclass
class FormState:
    schema_id: str
    collected: dict[str, Any] = field(default_factory=dict)
    current_index: int = 0
    status: str = 'in_progress'  # 'in_progress' | 'completed' | 'abandoned'

class FormEngine:
    def __init__(self):
        self.client = anthropic.Anthropic()
        self.states: dict[str, FormState] = {}
    
    def start_form(self, session_id: str, schema_id: str, fields: list[FormField]) -> str:
        state = FormState(schema_id=schema_id)
        self.states[session_id] = state
        self._advance(state, fields)
        return self._render_question(fields, state)
    
    async def handle_answer(
        self,
        session_id: str,
        fields: list[FormField],
        user_input: str,
        on_complete: Callable[[dict], None]
    ) -> tuple[str, bool]:
        state = self.states.get(session_id)
        if not state or state.status != 'in_progress':
            return '没有进行中的表单。', False
        
        current_field = fields[state.current_index] if state.current_index < len(fields) else None
        if not current_field:
            return '表单已完成。', True
        
        # 解析答案
        valid, value, error = await self._parse_answer(current_field, user_input)
        if not valid:
            return f'❌ {error}\n\n请重新回答：{current_field.label}', False
        
        # 保存并前进
        state.collected[current_field.id] = value
        state.current_index += 1
        self._advance(state, fields)
        
        # 检查完成
        if state.current_index >= len(fields):
            state.status = 'completed'
            on_complete(state.collected)
            return '✅ 已提交！感谢填写。', True
        
        return self._render_question(fields, state), False
    
    async def _parse_answer(
        self,
        field: FormField,
        user_input: str
    ) -> tuple[bool, Any, str]:
        if field.field_type == 'select' and field.options:
            response = self.client.messages.create(
                model='claude-haiku-4-5',
                max_tokens=50,
                system=f'从以下选项中匹配用户输入，只返回匹配的选项文字，无法匹配返回 INVALID：\n' +
                       '\n'.join(f'{i+1}. {o}' for i, o in enumerate(field.options)),
                messages=[{'role': 'user', 'content': user_input}]
            )
            matched = response.content[0].text.strip()
            if matched == 'INVALID' or matched not in field.options:
                return False, None, f'请选择：\n' + '\n'.join(f'{i+1}. {o}' for i, o in enumerate(field.options))
            return True, matched, ''
        
        if field.field_type == 'number':
            try:
                num = float(''.join(c for c in user_input if c.isdigit() or c == '.'))
                return True, num, ''
            except ValueError:
                return False, None, '请输入一个数字'
        
        if field.field_type == 'confirm':
            yes_words = ['是', '确认', 'yes', 'y', '对', '好']
            if any(w in user_input.lower() for w in yes_words):
                return True, True, ''
            return False, None, '操作已取消'
        
        # text 类型
        if len(user_input.strip()) < 2:
            return False, None, '输入太短，请详细描述'
        return True, user_input.strip(), ''
    
    def _advance(self, state: FormState, fields: list[FormField]) -> None:
        """跳过满足条件的字段"""
        while state.current_index < len(fields):
            f = fields[state.current_index]
            if f.skip_if and f.skip_if(state):
                state.current_index += 1
                continue
            if f.prefill:
                val = f.prefill(state)
                if val is not None:
                    state.collected[f.id] = val
                    state.current_index += 1
                    continue
            break
    
    def _render_question(self, fields: list[FormField], state: FormState) -> str:
        if state.current_index >= len(fields):
            return '表单已完成'
        
        f = fields[state.current_index]
        progress = f'[{state.current_index + 1}/{len(fields)}]'
        q = f'{progress} **{f.label}**'
        
        if f.field_type == 'select' and f.options:
            q += '\n' + '\n'.join(f'{i+1}. {o}' for i, o in enumerate(f.options))
        if f.hint:
            q += f'\n💡 {f.hint}'
        if not f.required:
            q += '\n（可选，发送「跳过」略过）'
        
        return q
    
    # 支持断点续传：导出/导入状态
    def export_state(self, session_id: str) -> Optional[dict]:
        state = self.states.get(session_id)
        if not state:
            return None
        return {
            'schema_id': state.schema_id,
            'collected': state.collected,
            'current_index': state.current_index,
            'status': state.status,
        }
    
    def import_state(self, session_id: str, state_dict: dict) -> None:
        self.states[session_id] = FormState(**state_dict)
```

---

## 关键设计原则

### 1. 动态字段跳过（skipIf）

```typescript
// 根据已收集的信息，智能跳过不相关字段
{
  id: 'company_size',
  label: '公司规模',
  skipIf: (state) => state.collected['user_type'] === 'individual',
}
```

### 2. 自动预填（prefill）

```typescript
// 从已知上下文自动填充，减少用户输入
{
  id: 'reporter_email',
  label: '你的邮箱',
  prefill: (state) => userProfile?.email,  // 从用户画像取
}
```

### 3. 断点续传（Checkpoint）

```typescript
// 持久化到 memory 文件，重启后可恢复
const stateKey = `memory/forms/${sessionId}.json`;
await fs.writeFile(stateKey, JSON.stringify(engine.exportState(sessionId)));

// 下次启动时恢复
const saved = await fs.readFile(stateKey, 'utf-8');
engine.importState(sessionId, JSON.parse(saved));
```

### 4. 自然语言解析（不强制格式）

```
用户问：「严重程度是多少？」
用户输入：「很严重，生产挂了」

→ LLM 解析 → 匹配到 「P0 - 生产崩溃」✅
```

---

## 与普通对话的协作模式

```typescript
// 表单 + 自由对话 无缝切换
async function handleMessage(sessionId: string, input: string): Promise<string> {
  const form = activeForms.get(sessionId);
  
  // 用户说「算了」→ 中断表单
  if (form && ['算了', '取消', 'cancel'].includes(input.toLowerCase())) {
    activeForms.delete(sessionId);
    return '好的，已取消。有什么别的需要帮忙的吗？';
  }
  
  // 表单进行中 → 路由到表单引擎
  if (form) return formEngine.handleAnswer(sessionId, form, input);
  
  // 检测表单触发意图
  const intent = await detectFormIntent(input);
  if (intent) {
    activeForms.set(sessionId, schemas[intent]);
    return formEngine.startForm(sessionId, schemas[intent]);
  }
  
  // 正常对话
  return normalChat(input);
}
```

---

## OpenClaw 实战场景

| 场景 | 表单字段数 | 典型 skipIf 规则 |
|------|-----------|----------------|
| Bug 报告 | 5-6 | P3 跳过影响用户数 |
| 用户反馈 | 3-4 | 正面反馈跳过改进建议 |
| 部署审批 | 4-5 | 开发环境跳过变更窗口确认 |
| 日程安排 | 4-6 | 线上会议跳过地点 |
| 订单创建 | 6-8 | 老用户预填收货地址 |

---

## 小结

```
传统表单：一次展示所有字段 → 用户放弃率高
Agent 表单：一次一个问题 → 完成率大幅提升
```

**核心价值：**
- 自然语言输入 + LLM 解析 = 零格式要求
- skipIf + prefill = 智能适配每个用户
- 断点续传 = 长表单不怕中断
- 随时取消 = 用户体验友好

**OpenClaw 集成要点：**
- `memory/forms/` 目录存储表单进度
- Heartbeat 检查超时未完成的表单并提醒用户
- Telegram inline buttons 可作为 select 字段的快捷输入
