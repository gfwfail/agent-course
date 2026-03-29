# 169 - Agent 多维度用户画像与个性化适配

> User Profiling & Personalization：从交互历史构建用户画像，让 Agent 越用越懂你

---

## 为什么需要用户画像？

默认的 Agent 对每个用户都是"陌生人"——不知道你是专家还是新手，不知道你喜欢简洁还是详细，不知道你的时区和语言习惯。

**用户画像（User Profile）** 把这些信息结构化存下来，每次对话动态注入 system prompt，实现：

- 专家用户 → 省略基础解释，直接给代码
- 新手用户 → 多一点类比和步骤说明
- 喜欢简洁 → 回答控制在 3 句内
- 中文用户 → 默认中文，不用每次说

---

## 核心数据结构

```typescript
interface UserProfile {
  userId: string;
  
  // 沟通风格
  communicationStyle: {
    verbosity: "concise" | "balanced" | "detailed";  // 详略程度
    formality: "casual" | "professional";            // 正式程度
    preferredLanguage: string;                       // 首选语言 "zh-CN"
  };
  
  // 专业程度（按领域）
  expertiseLevels: Record<string, "beginner" | "intermediate" | "expert">;
  // e.g. { "typescript": "expert", "kubernetes": "beginner" }
  
  // 领域偏好
  domainInterests: string[];  // ["backend", "devops", "AI agents"]
  
  // 行为偏好
  behaviorPrefs: {
    askBeforeExecuting: boolean;  // 是否确认再执行
    showCodeExamples: boolean;    // 是否自动展示代码
    timezone: string;             // "Australia/Sydney"
  };
  
  // 元信息
  meta: {
    createdAt: number;
    updatedAt: number;
    interactionCount: number;
    confidenceScore: number;  // 0-1，画像置信度（交互越多越高）
  };
}
```

---

## 画像构建：从交互中自动学习

不要让用户填问卷——太烦了。从对话行为中**被动推断**：

```typescript
class ProfileLearner {
  // 观察每次交互，更新画像
  async observe(event: InteractionEvent): Promise<ProfileDelta> {
    const signals: ProfileDelta = {};
    
    // 信号1：用户说"简短一点" → verbosity = concise
    if (/简短|简洁|brief|tldr/i.test(event.userMessage)) {
      signals.communicationStyle = { verbosity: "concise" };
    }
    
    // 信号2：用户修正了专业术语 → 提升该领域 expertise
    if (event.type === "user_correction" && event.domain) {
      signals.expertiseLevels = { [event.domain]: "expert" };
    }
    
    // 信号3：用户总是问某类问题 → 添加领域兴趣
    const detectedDomain = await this.classifyDomain(event.userMessage);
    if (detectedDomain) {
      signals.domainInterests = [detectedDomain];
    }
    
    // 信号4：用户明确反馈（好/不好）→ 调整策略
    if (event.type === "explicit_feedback") {
      signals.meta = { confidenceScore: Math.min(1, event.currentScore + 0.1) };
    }
    
    return signals;
  }
  
  // 用 LLM 分类问题领域
  private async classifyDomain(message: string): Promise<string | null> {
    const result = await llm.classify(message, [
      "typescript", "python", "devops", "database", "AI/ML", "frontend"
    ]);
    return result.confidence > 0.7 ? result.label : null;
  }
}
```

---

## 画像注入：动态组装 System Prompt

```typescript
class PersonalizationMiddleware {
  constructor(
    private store: ProfileStore,  // Redis / DB
    private learner: ProfileLearner
  ) {}

  async buildPersonalizedPrompt(
    basePrompt: string,
    userId: string
  ): Promise<string> {
    const profile = await this.store.get(userId);
    if (!profile || profile.meta.confidenceScore < 0.3) {
      // 画像置信度太低，不注入，避免误导
      return basePrompt;
    }
    
    const personalization = this.renderPersonalization(profile);
    return `${basePrompt}\n\n${personalization}`;
  }
  
  private renderPersonalization(profile: UserProfile): string {
    const lines: string[] = ["## 用户偏好（自动学习，请遵守）"];
    
    // 详略
    const verbosityMap = {
      concise: "回答务必简洁，3句以内，不要解释显而易见的内容",
      balanced: "正常详略即可",
      detailed: "用户喜欢详细解释，多举例子"
    };
    lines.push(`- ${verbosityMap[profile.communicationStyle.verbosity]}`);
    
    // 语言
    lines.push(`- 使用 ${profile.communicationStyle.preferredLanguage} 回复`);
    
    // 专业度
    const expertDomains = Object.entries(profile.expertiseLevels)
      .filter(([, level]) => level === "expert")
      .map(([domain]) => domain);
    if (expertDomains.length > 0) {
      lines.push(`- 用户是 ${expertDomains.join("/")} 专家，跳过基础概念`);
    }
    
    const beginnerDomains = Object.entries(profile.expertiseLevels)
      .filter(([, level]) => level === "beginner")
      .map(([domain]) => domain);
    if (beginnerDomains.length > 0) {
      lines.push(`- 用户 ${beginnerDomains.join("/")} 是新手，多用类比`);
    }
    
    // 行为偏好
    if (profile.behaviorPrefs.askBeforeExecuting) {
      lines.push("- 执行任何操作前先说明并确认");
    }
    if (profile.behaviorPrefs.showCodeExamples) {
      lines.push("- 尽量提供完整可运行的代码示例");
    }
    
    return lines.join("\n");
  }
}
```

---

## 画像存储：Redis + 衰减

```typescript
class ProfileStore {
  constructor(private redis: Redis) {}
  
  async get(userId: string): Promise<UserProfile | null> {
    const raw = await this.redis.get(`profile:${userId}`);
    return raw ? JSON.parse(raw) : null;
  }
  
  async merge(userId: string, delta: ProfileDelta): Promise<void> {
    const existing = await this.get(userId) ?? this.defaultProfile(userId);
    const merged = this.deepMerge(existing, delta);
    
    // 更新元信息
    merged.meta.updatedAt = Date.now();
    merged.meta.interactionCount += 1;
    
    // 90天 TTL，长期不用的画像自动过期
    await this.redis.setex(
      `profile:${userId}`,
      90 * 24 * 3600,
      JSON.stringify(merged)
    );
  }
  
  // 画像置信度衰减：30天不活跃降低置信度
  async applyDecay(userId: string): Promise<void> {
    const profile = await this.get(userId);
    if (!profile) return;
    
    const daysSinceUpdate = (Date.now() - profile.meta.updatedAt) / 86400000;
    if (daysSinceUpdate > 30) {
      profile.meta.confidenceScore = Math.max(0, 
        profile.meta.confidenceScore - (daysSinceUpdate - 30) * 0.01
      );
      await this.redis.setex(`profile:${userId}`, 90 * 24 * 3600, JSON.stringify(profile));
    }
  }
  
  private defaultProfile(userId: string): UserProfile {
    return {
      userId,
      communicationStyle: { verbosity: "balanced", formality: "professional", preferredLanguage: "zh-CN" },
      expertiseLevels: {},
      domainInterests: [],
      behaviorPrefs: { askBeforeExecuting: false, showCodeExamples: true, timezone: "UTC" },
      meta: { createdAt: Date.now(), updatedAt: Date.now(), interactionCount: 0, confidenceScore: 0 }
    };
  }
}
```

---

## OpenClaw 实战：MEMORY.md 就是用户画像

OpenClaw 已经内置了这个模式——`MEMORY.md` 就是一个人肉维护的用户画像：

```markdown
## USER.md - 老板偏好
- 语言：中文为主
- 风格：简洁、不废话、不说教
- 时区：Australia/Sydney (GMT+11)
- 专业度：Agent 开发专家，跳过基础解释
```

**自动化版**（在 SOUL.md 中声明学习指令）：

```markdown
## 学习规则
每次对话结束时，如果发现新的用户偏好信号，
执行：memory_search("user preference") 然后更新 MEMORY.md 对应条目。
置信度低的偏好标记为 [tentative]，多次确认后去掉标记。
```

---

## 隐私护栏

用户画像涉及个人数据，必须考虑：

```typescript
class PrivacyAwareProfileStore extends ProfileStore {
  // 1. 画像数据本地化，不跨租户共享
  async get(userId: string, tenantId: string): Promise<UserProfile | null> {
    return super.get(`${tenantId}:${userId}`);
  }
  
  // 2. 用户可随时清空画像（GDPR "被遗忘权"）
  async forget(userId: string, tenantId: string): Promise<void> {
    await this.redis.del(`profile:${tenantId}:${userId}`);
  }
  
  // 3. 画像内容不含敏感数据（只存偏好，不存对话内容）
  async merge(userId: string, delta: ProfileDelta): Promise<void> {
    const sanitized = this.stripPII(delta);
    return super.merge(userId, sanitized);
  }
  
  private stripPII(delta: ProfileDelta): ProfileDelta {
    // 确保 delta 里没有邮箱、手机、姓名等
    const safe = { ...delta };
    delete (safe as any).email;
    delete (safe as any).phone;
    return safe;
  }
}
```

---

## 完整流程

```
用户发消息
    ↓
[PersonalizationMiddleware]
    ├── 拉取用户画像（ProfileStore.get）
    ├── 注入个性化 prompt（renderPersonalization）
    └── 调用 LLM（带个性化 prompt）
         ↓
    LLM 回复
         ↓
[ProfileLearner.observe]
    ├── 解析交互信号
    ├── 生成 ProfileDelta
    └── ProfileStore.merge（异步，不阻塞回复）
```

---

## 效果对比

| 场景 | 无画像 | 有画像（第10次对话） |
|------|--------|---------------------|
| 问"怎么部署？" | 从基础开始解释 Docker | 直接给 k8s yaml，跳过入门 |
| 问"帮我发邮件" | 先问确认 | 知道用户偏好确认，先说要发什么再问 |
| 多语言环境 | 看到中文才用中文 | 默认中文，不用每次说 |
| Token 消耗 | 高（解释基础知识） | 低（专家模式省去 30% 解释） |

---

## 关键设计原则

1. **被动学习 > 主动填写**：让用户自然流露偏好，别烦他们填问卷
2. **置信度门控**：新用户画像置信度低，不要贸然注入（可能误导）
3. **衰减 + TTL**：偏好会变，老画像要降权，不用的用户自动清理
4. **隐私最小化**：只存偏好信号，不存对话内容
5. **透明可撤销**：用户可随时查看/清除自己的画像（`/profile show` / `/profile reset`）

---

## 与其他课程的关系

| 课程 | 关联点 |
|------|--------|
| [82-dynamic-system-prompt](82-dynamic-system-prompt.md) | 画像注入的载体 |
| [93-pii-masking](93-pii-masking.md) | 画像存储隐私护栏 |
| [105-self-learning](105-self-learning.md) | 画像是自我学习的一个维度 |
| [129-conversation-summary-memory](129-conversation-summary-memory.md) | 画像 vs 对话摘要的区别：画像是偏好特征，摘要是内容记录 |

---

## 总结

用户画像让 Agent 从"工具"变成"助理"。核心三步：

1. **观察**：从对话信号中被动推断偏好
2. **存储**：Redis + TTL + 置信度衰减
3. **注入**：动态组装 system prompt，按置信度门控

OpenClaw 的 `MEMORY.md` + `USER.md` 是这个模式的人工版落地，自动化版只需在中间件层加 ProfileLearner 和 PersonalizationMiddleware 即可。

**越用越懂你，这才是真正的个人 AI 助理。**
