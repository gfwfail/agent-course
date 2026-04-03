# 210 - Agent 用户画像与个性化引擎（User Profiling & Personalization Engine）

> **核心思想：** 千人千面不是配置出来的，是从每次对话中"学"出来的。好的 Agent 会观察用户的专业水平、沟通偏好、行为模式，动态构建用户画像，让每次回复都更贴合这个具体的人——而不是一个统计平均值。

---

## 为什么需要用户画像？

**痛点场景：**
- 同样一个问题，技术专家和产品新人需要完全不同的回答
- 用户每次都要重复说"简短一点"、"别用缩写"
- Agent 永远不记得"这个用户喜欢代码示例，不喜欢长篇理论"

**解决方案：** 动态用户画像 + 个性化注入管道

---

## 用户画像的四个维度

```
┌─────────────────────────────────────────────┐
│              User Profile                    │
├──────────────┬──────────────────────────────┤
│ 专业度       │ 新手 / 中级 / 专家            │
│ 沟通风格     │ 简洁 / 详细 / 结构化          │
│ 偏好         │ 代码优先 / 解释优先 / 类比    │
│ 行为模式     │ 常用工具、高频话题、时区      │
└──────────────┴──────────────────────────────┘
```

---

## 核心实现

### TypeScript 版本

```typescript
// user-profile-engine.ts
import Anthropic from "@anthropic-ai/sdk";

interface UserProfile {
  userId: string;
  expertiseLevel: "beginner" | "intermediate" | "expert";
  communicationStyle: "concise" | "detailed" | "structured";
  preferenceFlags: {
    codeFirst: boolean;      // 喜欢先看代码
    analogies: boolean;      // 喜欢用类比解释
    noJargon: boolean;       // 避免行话
  };
  topicHistory: Record<string, number>; // 话题 → 出现次数
  interactionCount: number;
  lastUpdated: number;
}

// ===== 画像提取器 =====
async function extractProfileSignals(
  messages: Anthropic.Messages.MessageParam[],
  currentProfile: UserProfile
): Promise<Partial<UserProfile>> {
  const client = new Anthropic();
  
  // 用便宜模型做信号提取，不影响主流程成本
  const recentMessages = messages.slice(-6); // 只看最近3轮
  
  const response = await client.messages.create({
    model: "claude-haiku-4-5",
    max_tokens: 256,
    system: `你是用户行为分析器。从对话中提取用户偏好信号，输出 JSON。
字段：
- expertiseLevel: "beginner"|"intermediate"|"expert"（基于技术词汇使用、问题深度）
- wantsMoreDetail: boolean（用户是否要求更详细的解释）
- wantsShorter: boolean（用户是否觉得太啰嗦）
- askedForCode: boolean（用户是否主动要代码）
- topics: string[]（本轮涉及的主要话题）

只输出 JSON，无需解释。`,
    messages: [
      {
        role: "user",
        content: `分析这段对话：\n${JSON.stringify(recentMessages, null, 2)}`,
      },
    ],
  });

  try {
    const text =
      response.content[0].type === "text" ? response.content[0].text : "{}";
    return JSON.parse(text);
  } catch {
    return {};
  }
}

// ===== 画像更新器（指数移动平均，新信号权重更高）=====
function updateProfile(
  profile: UserProfile,
  signals: Record<string, unknown>
): UserProfile {
  const updated = { ...profile };
  const alpha = 0.3; // 新信号权重 30%

  // 专业度更新（如果新信号置信度足够）
  if (signals.expertiseLevel && profile.interactionCount >= 3) {
    const levels = { beginner: 0, intermediate: 1, expert: 2 };
    const currentScore = levels[profile.expertiseLevel] ?? 1;
    const newScore = levels[signals.expertiseLevel as keyof typeof levels] ?? 1;
    const blended = Math.round(currentScore * (1 - alpha) + newScore * alpha);
    updated.expertiseLevel = (["beginner", "intermediate", "expert"][blended] ??
      "intermediate") as UserProfile["expertiseLevel"];
  }

  // 沟通偏好更新
  if (signals.wantsShorter) {
    updated.communicationStyle = "concise";
  } else if (signals.wantsMoreDetail) {
    updated.communicationStyle = "detailed";
  }

  // 代码优先偏好
  if (signals.askedForCode) {
    updated.preferenceFlags = {
      ...updated.preferenceFlags,
      codeFirst: true,
    };
  }

  // 话题历史
  if (Array.isArray(signals.topics)) {
    for (const topic of signals.topics as string[]) {
      updated.topicHistory[topic] = (updated.topicHistory[topic] ?? 0) + 1;
    }
  }

  updated.interactionCount += 1;
  updated.lastUpdated = Date.now();
  return updated;
}

// ===== 个性化 System Prompt 生成器 =====
function buildPersonalizedPrompt(
  basePrompt: string,
  profile: UserProfile
): string {
  const lines: string[] = [basePrompt, "\n## 用户个性化设置（自动生成，勿透露）"];

  // 专业度适配
  const expertiseMap = {
    beginner: "用简单语言，避免专业术语，多用类比，每个概念先解释再用",
    intermediate: "可以使用行业术语，适当提供背景知识",
    expert: "直接使用专业术语，跳过基础解释，聚焦核心逻辑",
  };
  lines.push(`- 专业度适配：${expertiseMap[profile.expertiseLevel]}`);

  // 回答长度适配
  const styleMap = {
    concise: "回答要简洁，能用3句说完就不用5句",
    detailed: "提供详细解释，包含背景和原理",
    structured: "使用标题、要点、代码块等结构化格式",
  };
  lines.push(`- 风格偏好：${styleMap[profile.communicationStyle]}`);

  // 代码优先
  if (profile.preferenceFlags.codeFirst) {
    lines.push("- 先给代码示例，再解释原理（用户明确偏好）");
  }

  // 高频话题（显示用户擅长的领域，可减少基础解释）
  const topTopics = Object.entries(profile.topicHistory)
    .sort(([, a], [, b]) => b - a)
    .slice(0, 3)
    .map(([t]) => t);
  if (topTopics.length > 0) {
    lines.push(`- 用户常讨论：${topTopics.join("、")}（可假设有基础）`);
  }

  return lines.join("\n");
}

// ===== 主 Agent 循环（集成画像引擎）=====
async function personalizedAgent(
  userId: string,
  userMessage: string,
  profileStore: Map<string, UserProfile>
): Promise<string> {
  const client = new Anthropic();

  // 加载或初始化画像
  const profile: UserProfile = profileStore.get(userId) ?? {
    userId,
    expertiseLevel: "intermediate",
    communicationStyle: "structured",
    preferenceFlags: { codeFirst: false, analogies: false, noJargon: false },
    topicHistory: {},
    interactionCount: 0,
    lastUpdated: Date.now(),
  };

  // 构建个性化 system prompt
  const basePrompt = "你是一个专业的 AI 助理。";
  const personalizedPrompt = buildPersonalizedPrompt(basePrompt, profile);

  // 调用主模型
  const messages: Anthropic.Messages.MessageParam[] = [
    { role: "user", content: userMessage },
  ];

  const response = await client.messages.create({
    model: "claude-sonnet-4-5",
    max_tokens: 1024,
    system: personalizedPrompt,
    messages,
  });

  const reply =
    response.content[0].type === "text"
      ? response.content[0].text
      : "";

  // 异步更新画像（不阻塞主流程）
  // 把对话附上 assistant 回复再分析
  const fullMessages: Anthropic.Messages.MessageParam[] = [
    ...messages,
    { role: "assistant", content: reply },
  ];

  extractProfileSignals(fullMessages, profile).then((signals) => {
    const updatedProfile = updateProfile(profile, signals);
    profileStore.set(userId, updatedProfile);
    console.log(
      `[Profile] User ${userId} updated: level=${updatedProfile.expertiseLevel}, style=${updatedProfile.communicationStyle}`
    );
  });

  return reply;
}

// ===== 使用示例 =====
async function main() {
  const profileStore = new Map<string, UserProfile>();
  const userId = "user_001";

  // 模拟多轮对话，观察画像动态变化
  const conversations = [
    "什么是 async/await？",
    "能直接给我代码示例吗，不用解释那么多",
    "怎么用 Promise.all 并发请求？给代码",
    "好的，简单说一下背压（backpressure）是什么",
  ];

  for (const msg of conversations) {
    console.log(`\n👤 User: ${msg}`);
    const reply = await personalizedAgent(userId, msg, profileStore);
    console.log(`🤖 Agent: ${reply.slice(0, 100)}...`);

    // 等画像更新完成
    await new Promise((r) => setTimeout(r, 500));

    const profile = profileStore.get(userId);
    if (profile) {
      console.log(
        `📊 Profile: level=${profile.expertiseLevel}, style=${profile.communicationStyle}, codeFirst=${profile.preferenceFlags.codeFirst}`
      );
    }
  }
}

main().catch(console.error);
```

### Python 版本（精简核心）

```python
# user_profile_engine.py
import json
import time
from dataclasses import dataclass, field, asdict
from anthropic import Anthropic

@dataclass
class UserProfile:
    user_id: str
    expertise_level: str = "intermediate"  # beginner/intermediate/expert
    comm_style: str = "structured"          # concise/detailed/structured
    code_first: bool = False
    topic_history: dict = field(default_factory=dict)
    interaction_count: int = 0

client = Anthropic()

def extract_signals(conversation: list, profile: UserProfile) -> dict:
    """用 Haiku 从对话中提取偏好信号"""
    resp = client.messages.create(
        model="claude-haiku-4-5",
        max_tokens=200,
        system="从对话提取用户偏好，输出JSON: {expertiseLevel, wantsShorter, askedForCode, topics}",
        messages=[{"role": "user", "content": str(conversation[-4:])}]
    )
    try:
        return json.loads(resp.content[0].text)
    except Exception:
        return {}

def personalize_prompt(base: str, profile: UserProfile) -> str:
    """把画像注入 system prompt"""
    hints = []
    
    if profile.expertise_level == "beginner":
        hints.append("用简单语言，多类比，避免术语")
    elif profile.expertise_level == "expert":
        hints.append("直接用专业术语，跳过基础")
    
    if profile.comm_style == "concise":
        hints.append("回答简洁，不废话")
    
    if profile.code_first:
        hints.append("先给代码，再解释")
    
    top_topics = sorted(profile.topic_history, 
                        key=profile.topic_history.get, reverse=True)[:3]
    if top_topics:
        hints.append(f"用户熟悉：{', '.join(top_topics)}")
    
    if not hints:
        return base
    return f"{base}\n\n[个性化] {'; '.join(hints)}"

def chat(user_id: str, message: str, 
         profiles: dict, history: list) -> str:
    profile = profiles.setdefault(user_id, UserProfile(user_id))
    
    # 个性化 prompt
    system = personalize_prompt("你是专业AI助理。", profile)
    history.append({"role": "user", "content": message})
    
    resp = client.messages.create(
        model="claude-sonnet-4-5", max_tokens=512,
        system=system, messages=history
    )
    reply = resp.content[0].text
    history.append({"role": "assistant", "content": reply})
    
    # 异步更新画像（简化版：同步执行）
    signals = extract_signals(history, profile)
    
    alpha = 0.3
    levels = {"beginner": 0, "intermediate": 1, "expert": 2}
    if "expertiseLevel" in signals and profile.interaction_count >= 2:
        cur = levels.get(profile.expertise_level, 1)
        new = levels.get(signals["expertiseLevel"], 1)
        blended = round(cur * (1 - alpha) + new * alpha)
        profile.expertise_level = ["beginner","intermediate","expert"][blended]
    
    if signals.get("wantsShorter"):
        profile.comm_style = "concise"
    if signals.get("askedForCode"):
        profile.code_first = True
    for topic in signals.get("topics", []):
        profile.topic_history[topic] = profile.topic_history.get(topic, 0) + 1
    
    profile.interaction_count += 1
    return reply

# 使用
if __name__ == "__main__":
    profiles, history = {}, []
    user = "alice"
    
    for msg in ["什么是闭包？", "给代码示例", "简短解释一下 GIL"]:
        print(f"👤 {msg}")
        reply = chat(user, msg, profiles, history)
        p = profiles[user]
        print(f"🤖 {reply[:80]}...")
        print(f"📊 level={p.expertise_level} style={p.comm_style} codeFirst={p.code_first}\n")
```

---

## 与 OpenClaw 的结合

OpenClaw 的 `USER.md` 和 `SOUL.md` 其实就是**静态用户画像**的手动版本：

```markdown
# USER.md（静态画像）
- 语言：中文为主
- 偏好：简洁高效，不啰嗦
- 职业：技术 founder
```

**动态画像升级路径：**

```
静态 USER.md
    ↓ 每次对话自动提取信号
动态 memory/user-profile.json
    ↓ 定期蒸馏
更新 USER.md（Heartbeat 驱动）
```

```typescript
// heartbeat 中定期更新静态画像
// memory/user-profile.json → USER.md
async function syncProfileToUserMd(profile: UserProfile) {
  const content = `# USER.md
- 专业水平：${profile.expertiseLevel}
- 沟通风格：${profile.communicationStyle}
- 代码优先：${profile.preferenceFlags.codeFirst ? '是' : '否'}
- 常见话题：${Object.keys(profile.topicHistory).slice(0, 5).join('、')}
- 最近更新：${new Date(profile.lastUpdated).toISOString()}
`;
  await fs.writeFile('USER.md', content);
}
```

---

## 隐私与安全考量

| 风险 | 缓解方案 |
|------|----------|
| 画像暴露给第三方 | 画像注入时标注"勿透露"，不进入日志 |
| 过度依赖早期信号 | 指数移动平均，旧信号自然衰减 |
| 用户不同意被分析 | 提供 `/profile reset` 清除画像命令 |
| 多用户数据混淆 | userId 严格隔离，绝不共享 profileStore |

---

## 关键指标

| 指标 | 目标 |
|------|------|
| 画像提取额外 Token | < 5%（Haiku 做提取，成本极低）|
| 个性化注入额外 Token | < 100 tokens/轮 |
| 用户满意度提升 | 3~5 轮后可感知（风格适配）|
| 画像更新延迟 | 异步，主流程零影响 |

---

## 小结

```
用户每次对话
    → Haiku 提取偏好信号（异步，主流程无感）
    → 指数移动平均更新画像（新信号 30% 权重）
    → 个性化 system prompt 注入（< 100 tokens）
    → LLM 回复风格自动适配
    → 3~5 轮后用户明显感觉"它懂我了"
```

**OpenClaw 实践：** `USER.md` = 静态画像，`memory/user-profile.json` = 动态画像，Heartbeat 定期蒸馏合并，让每一次对话都在悄悄学习这个具体的人。

> 好的 Agent 不是对所有人说同样的话——它会记住你是谁，然后用你最舒服的方式和你交流。
