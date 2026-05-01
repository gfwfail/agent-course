# 217. Agent 能力探测与协商（Capability Discovery & Negotiation）

> 关键词：Capability Discovery、平台能力矩阵、工具可用性、优雅降级、协议协商

前面我们讲过工具权限、富媒体输出、多平台渲染。今天补一个生产 Agent 非常容易忽略的基础设施：**能力探测与协商**。

同一个 Agent 在 Telegram、Discord、CLI、Web、Cron、Sub-agent 里运行时，能做的事并不一样：

- Telegram 支持 inline buttons，但某些群可能不支持文件预览
- Discord 支持 thread、reaction，但不适合 markdown table
- CLI 支持本地文件路径，Web 需要 URL 或附件
- Cron 没有用户在线确认，危险操作要改成 pending approval
- Sub-agent 可能没有浏览器、没有私有记忆、没有外部发送权限

如果 Agent 写死能力假设，结果就是：在 A 平台很好用，在 B 平台直接炸。

所以成熟 Agent 启动每个任务前都要先问：**当前环境到底支持什么？**

---

## 1. 什么是能力探测？

Capability 是运行环境提供给 Agent 的能力声明。

```ts
type RuntimeCapabilities = {
  channel: "telegram" | "discord" | "web" | "cli" | "cron";
  output: {
    markdown: boolean;
    tables: boolean;
    attachments: boolean;
    images: boolean;
    audio: boolean;
    inlineButtons: boolean;
    reactions: boolean;
    maxMessageChars: number;
  };
  tools: {
    browser: boolean;
    shell: boolean;
    fileWrite: boolean;
    externalSend: boolean;
    approvalRequiredFor: string[];
  };
  context: {
    hasUserSession: boolean;
    hasPrivateMemory: boolean;
    canSpawnSubagents: boolean;
  };
};
```

关键点：**能力不是写在 prompt 里的愿望，而是运行时可验证的事实。**

---

## 2. 反模式：直接假设平台能力

很多 Agent 会这样写：

```ts
async function replyWithReport(ctx, reportPath: string) {
  await ctx.send(`报告生成好了：${reportPath}`);
}
```

这有几个问题：

- Telegram 用户点不了本地路径
- Web 用户需要下载链接
- Discord 大文件可能超过限制
- Cron 场景可能没有交互对象
- 文件可能含敏感数据，不能直接公开路径

正确做法是：根据能力协商输出方式。

```ts
async function deliverReport(ctx, artifact: Artifact) {
  const caps = ctx.capabilities;

  if (caps.output.attachments) {
    return ctx.sendAttachment(artifact.path, {
      filename: artifact.name,
      caption: "报告生成好了",
    });
  }

  if (caps.output.markdown && artifact.signedUrl) {
    return ctx.send(`报告生成好了：[下载链接](${artifact.signedUrl})`);
  }

  return ctx.send(
    `报告已生成，但当前渠道不支持附件。artifact_id=${artifact.id}，请在支持文件下载的界面打开。`
  );
}
```

这就是能力协商：**不是问 LLM 想怎么输出，而是让基础设施告诉 Agent 能怎么输出。**

---

## 3. learn-claude-code 教学版：最小能力矩阵

教学项目可以先用一个简单的 Python 字典，不需要一开始就上复杂系统。

```python
# learn-claude-code style: minimal capability registry
from dataclasses import dataclass

@dataclass
class OutputCaps:
    markdown: bool
    tables: bool
    attachments: bool
    inline_buttons: bool
    max_chars: int

CAPABILITIES = {
    "telegram": OutputCaps(
        markdown=True,
        tables=False,
        attachments=True,
        inline_buttons=True,
        max_chars=4096,
    ),
    "discord": OutputCaps(
        markdown=True,
        tables=False,
        attachments=True,
        inline_buttons=True,
        max_chars=2000,
    ),
    "cli": OutputCaps(
        markdown=True,
        tables=True,
        attachments=False,
        inline_buttons=False,
        max_chars=100_000,
    ),
}

def get_caps(channel: str) -> OutputCaps:
    return CAPABILITIES.get(channel, CAPABILITIES["cli"])


def render_summary(channel: str, rows: list[dict]) -> str:
    caps = get_caps(channel)

    if caps.tables:
        return "\n".join([
            "| name | value |",
            "| --- | --- |",
            *[f"| {r['name']} | {r['value']} |" for r in rows],
        ])

    # Discord/Telegram 避免 markdown table
    lines = ["结果汇总："]
    for r in rows:
        lines.append(f"- {r['name']}: {r['value']}")
    return "\n".join(lines)[: caps.max_chars]
```

这段代码虽然简单，但已经有生产级思想：**渲染层不猜平台，渲染层读能力。**

---

## 4. pi-mono 生产版：Capability Provider

生产系统里，能力应该来自多个来源：

1. 静态平台能力：Telegram/Discord/CLI 默认支持什么
2. 会话能力：当前 chat/thread 是否支持某些交互
3. 用户权限：这个用户能不能触发外部发送、shell、浏览器
4. 任务风险：危险操作是否需要审批
5. 运行时健康：某个工具当前是否可用

可以抽象成 `CapabilityProvider`：

```ts
export interface CapabilityProvider {
  detect(ctx: AgentContext): Promise<RuntimeCapabilities>;
}

export class DefaultCapabilityProvider implements CapabilityProvider {
  constructor(
    private readonly platformRegistry: PlatformRegistry,
    private readonly permissionService: PermissionService,
    private readonly toolHealth: ToolHealthRegistry,
  ) {}

  async detect(ctx: AgentContext): Promise<RuntimeCapabilities> {
    const platform = this.platformRegistry.get(ctx.channel);
    const permissions = await this.permissionService.forSession(ctx.sessionId);
    const health = await this.toolHealth.snapshot();

    return {
      channel: ctx.channel,
      output: {
        markdown: platform.markdown,
        tables: platform.tables,
        attachments: platform.attachments && permissions.canSendFiles,
        images: platform.images,
        audio: platform.audio,
        inlineButtons: platform.inlineButtons && ctx.interactive,
        reactions: platform.reactions,
        maxMessageChars: platform.maxMessageChars,
      },
      tools: {
        browser: permissions.tools.includes("browser") && health.browser.ok,
        shell: permissions.tools.includes("shell") && health.shell.ok,
        fileWrite: permissions.tools.includes("file.write"),
        externalSend: permissions.canExternalSend,
        approvalRequiredFor: permissions.approvalRequiredFor,
      },
      context: {
        hasUserSession: Boolean(ctx.userId),
        hasPrivateMemory: ctx.kind === "main" && permissions.canReadPrivateMemory,
        canSpawnSubagents: permissions.tools.includes("sessions.spawn"),
      },
    };
  }
}
```

然后在 Agent Loop 开始前注入：

```ts
async function runAgent(ctx: AgentContext, input: string) {
  const capabilities = await capabilityProvider.detect(ctx);

  const system = buildSystemPrompt({
    base: ctx.systemPrompt,
    capabilities,
  });

  return model.complete({
    system,
    messages: ctx.messages.concat({ role: "user", content: input }),
    tools: filterToolsByCapabilities(allTools, capabilities),
  });
}
```

注意这里做了两件事：

- `buildSystemPrompt` 告诉模型当前环境的能力边界
- `filterToolsByCapabilities` 从根上移除不可用工具，降低幻觉调用

---

## 5. 工具过滤：不要把不能用的工具暴露给模型

如果当前环境不能发外部消息，就不要把 `send_message` 暴露给 LLM。

```ts
function filterToolsByCapabilities(
  tools: ToolDefinition[],
  caps: RuntimeCapabilities,
): ToolDefinition[] {
  return tools.filter((tool) => {
    if (tool.requires?.includes("browser") && !caps.tools.browser) return false;
    if (tool.requires?.includes("shell") && !caps.tools.shell) return false;
    if (tool.requires?.includes("file.write") && !caps.tools.fileWrite) return false;
    if (tool.requires?.includes("external.send") && !caps.tools.externalSend) return false;
    return true;
  });
}
```

这比“提示词里说不要调用”可靠得多。

**安全边界应该在工具注册层，不应该只靠模型自觉。**

---

## 6. 输出协商：同一份内容，多种交付形态

能力协商最常见的应用就是输出适配。

```ts
type RichResponse = {
  title: string;
  body: string;
  table?: Array<Record<string, string | number>>;
  buttons?: Array<{ label: string; value: string }>;
  attachmentId?: string;
};

function renderResponse(resp: RichResponse, caps: RuntimeCapabilities): OutboundMessage {
  const chunks: string[] = [];

  chunks.push(caps.output.markdown ? `**${resp.title}**` : resp.title);
  chunks.push(resp.body);

  if (resp.table) {
    chunks.push(caps.output.tables
      ? renderMarkdownTable(resp.table)
      : renderBulletList(resp.table));
  }

  return {
    text: fitToLimit(chunks.join("\n\n"), caps.output.maxMessageChars),
    buttons: caps.output.inlineButtons ? resp.buttons : undefined,
    attachmentId: caps.output.attachments ? resp.attachmentId : undefined,
  };
}
```

这样业务逻辑只产出 `RichResponse`，平台适配器负责最终形态。

这也是 OpenClaw `presentation`、`delivery` 这类结构化消息设计的核心思想：**先表达语义，再按平台能力渲染。**

---

## 7. OpenClaw 实战：从 Runtime 读取能力

OpenClaw 里经常能看到这类运行时信息：

```text
channel=telegram | capabilities=inlinebuttons,nativeapprovals
```

这意味着当前会话支持：

- Telegram 渠道发送
- inline buttons
- 原生审批卡片

所以 Agent 可以做这样的决策：

```ts
function chooseApprovalMode(caps: RuntimeCapabilities) {
  if (caps.output.inlineButtons && caps.tools.approvalRequiredFor.length > 0) {
    return "native_approval_card";
  }

  if (caps.output.markdown) {
    return "manual_approval_text";
  }

  return "escalate_to_owner";
}
```

实际效果：

- 支持按钮：给用户发审批按钮
- 不支持按钮：发清晰的文本确认说明
- 没有交互：创建 pending task，等有人接管

这就是优雅降级。

---

## 8. 能力协商不只用于输出，也用于计划

同一个任务，在不同能力下计划应该不同。

```ts
async function planResearchTask(question: string, caps: RuntimeCapabilities) {
  if (caps.tools.browser) {
    return [
      { step: "web_search", query: question },
      { step: "browser_verify_sources" },
      { step: "summarize_with_links" },
    ];
  }

  return [
    { step: "answer_from_existing_context", query: question },
    { step: "mark_limitations", reason: "browser unavailable" },
  ];
}
```

不要让 Agent 计划一个当前环境根本执行不了的工作流。

---

## 9. 生产 Checklist

做能力探测时，建议至少检查这 8 项：

- [ ] 当前渠道最大消息长度
- [ ] 是否支持附件、图片、音频
- [ ] 是否支持按钮/审批/反应
- [ ] 是否支持 markdown table（很多聊天平台不适合）
- [ ] 当前用户/会话是否有外部发送权限
- [ ] 当前工具是否健康可用
- [ ] 子 Agent 是否继承同等权限，还是降权执行
- [ ] 无能力时有没有明确降级路径

---

## 10. 一句话总结

**Capability Discovery & Negotiation = Agent 的环境感知层。**

没有它，Agent 只能在 demo 环境里表现良好；有了它，Agent 才能在 Telegram、Discord、CLI、Web、Cron、Sub-agent 之间稳定迁移。

真正生产级的 Agent，不是“我会很多工具”，而是：

> 我知道现在能用什么、不能用什么，以及不能用时怎么优雅降级。
