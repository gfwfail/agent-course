# Context Injection：上下文注入

> 第 29 课 - 如何让 Agent 一启动就"懂"你的项目

## 为什么需要上下文注入？

Agent 每次启动都是"失忆"的。没有上下文注入，Agent 会：
- 不知道项目结构
- 不了解编码规范
- 不知道已有的工具和配置
- 每次都要重新问一遍"这是什么项目？"

**上下文注入的目标**：让 Agent 在第一轮对话之前，就已经具备完成任务所需的所有背景知识。

## 核心概念

```
┌─────────────────────────────────────────────────┐
│              System Prompt                       │
│  ┌─────────────────────────────────────────┐    │
│  │  Base Instructions (who you are)         │    │
│  ├─────────────────────────────────────────┤    │
│  │  Tool Definitions (what you can do)      │    │
│  ├─────────────────────────────────────────┤    │
│  │  Project Context  ← 上下文注入            │    │
│  │  • AGENTS.md / CLAUDE.md                 │    │
│  │  • File contents                         │    │
│  │  • Runtime info                          │    │
│  ├─────────────────────────────────────────┤    │
│  │  Capabilities & Constraints              │    │
│  └─────────────────────────────────────────┘    │
└─────────────────────────────────────────────────┘
```

## 上下文注入的类型

### 1. 静态注入 - 项目配置文件

最常见的方式是通过特定文件自动加载：

```typescript
// OpenClaw 的实现
interface ContextFile {
  name: string;
  patterns: string[];  // 文件匹配模式
  required: boolean;
}

const CONTEXT_FILES: ContextFile[] = [
  { name: 'AGENTS.md', patterns: ['AGENTS.md', '.agents.md'], required: false },
  { name: 'CLAUDE.md', patterns: ['CLAUDE.md', '.claude.md'], required: false },
  { name: 'SOUL.md', patterns: ['SOUL.md'], required: false },
  { name: 'USER.md', patterns: ['USER.md'], required: false },
  { name: 'TOOLS.md', patterns: ['TOOLS.md'], required: false },
  { name: 'MEMORY.md', patterns: ['MEMORY.md'], required: false },
];

async function loadProjectContext(workdir: string): Promise<string[]> {
  const contexts: string[] = [];
  
  for (const file of CONTEXT_FILES) {
    for (const pattern of file.patterns) {
      const filepath = path.join(workdir, pattern);
      if (await fileExists(filepath)) {
        const content = await readFile(filepath, 'utf-8');
        contexts.push(`## ${filepath}\n${content}`);
        break;  // 只加载第一个匹配的
      }
    }
  }
  
  return contexts;
}
```

### 2. 动态注入 - 运行时信息

Agent 需要知道它的运行环境：

```typescript
// pi-mono 的 Runtime 信息注入
function buildRuntimeContext(): string {
  const info = {
    agent: sessionConfig.agentId || 'main',
    host: os.hostname(),
    repo: findGitRoot(process.cwd()) || process.cwd(),
    os: `${os.type()} ${os.release()} (${os.arch()})`,
    node: process.version,
    model: sessionConfig.model,
    shell: process.env.SHELL || 'unknown',
    channel: channelType,
  };
  
  const parts = Object.entries(info)
    .map(([k, v]) => `${k}=${v}`)
    .join(' | ');
  
  return `Runtime: ${parts}`;
}

// 输出示例:
// Runtime: agent=main | host=macbook | repo=/Users/dev/myproject | 
//          os=Darwin 23.0.0 (arm64) | node=v20.10.0 | model=claude-3-opus
```

### 3. 按需注入 - 基于任务类型

不同任务需要不同上下文：

```typescript
// OpenClaw 的条件注入
interface InjectionRule {
  condition: (ctx: SessionContext) => boolean;
  inject: () => Promise<string>;
}

const injectionRules: InjectionRule[] = [
  {
    // 只在主会话加载敏感信息
    condition: (ctx) => ctx.isMainSession && !ctx.isGroupChat,
    inject: () => loadFile('MEMORY.md'),
  },
  {
    // 代码任务加载项目结构
    condition: (ctx) => ctx.taskType === 'coding',
    inject: async () => {
      const tree = await exec('tree -L 2 --gitignore');
      return `## Project Structure\n\`\`\`\n${tree}\n\`\`\``;
    },
  },
  {
    // 有 package.json 就加载依赖信息
    condition: (ctx) => fileExists(path.join(ctx.workdir, 'package.json')),
    inject: async () => {
      const pkg = JSON.parse(await readFile('package.json'));
      return `## Dependencies\n${Object.keys(pkg.dependencies || {}).join(', ')}`;
    },
  },
];

async function buildContextualInjection(ctx: SessionContext): Promise<string[]> {
  const injections: string[] = [];
  
  for (const rule of injectionRules) {
    if (rule.condition(ctx)) {
      injections.push(await rule.inject());
    }
  }
  
  return injections;
}
```

## 实战：Claude Code 的上下文加载

看看 learn-claude-code 是怎么处理的：

```python
# learn-claude-code/context_loader.py
class ContextLoader:
    """项目上下文加载器"""
    
    CONTEXT_FILES = [
        'CLAUDE.md',
        '.claude/context.md',
        'AGENTS.md',
        'README.md',  # 备选
    ]
    
    def __init__(self, workdir: str):
        self.workdir = Path(workdir)
        self._cache: dict[str, str] = {}
    
    def load_all(self) -> str:
        """加载所有上下文文件"""
        sections = []
        
        # 1. 项目配置文件
        for filename in self.CONTEXT_FILES:
            content = self._load_file(filename)
            if content:
                sections.append(f"# {filename}\n{content}")
        
        # 2. Git 信息
        git_info = self._get_git_info()
        if git_info:
            sections.append(f"# Git Status\n{git_info}")
        
        # 3. 环境信息
        sections.append(self._get_env_info())
        
        return "\n\n---\n\n".join(sections)
    
    def _load_file(self, filename: str) -> str | None:
        """加载单个文件，带缓存"""
        if filename in self._cache:
            return self._cache[filename]
        
        filepath = self.workdir / filename
        if not filepath.exists():
            return None
        
        content = filepath.read_text()
        
        # 处理文件大小限制
        max_size = 50_000  # 50KB
        if len(content) > max_size:
            content = content[:max_size] + "\n... (truncated)"
        
        self._cache[filename] = content
        return content
    
    def _get_git_info(self) -> str | None:
        """获取 Git 信息"""
        try:
            branch = subprocess.check_output(
                ['git', 'branch', '--show-current'],
                cwd=self.workdir,
                text=True
            ).strip()
            
            status = subprocess.check_output(
                ['git', 'status', '--short'],
                cwd=self.workdir,
                text=True
            ).strip()
            
            return f"Branch: {branch}\nChanges:\n{status or '(clean)'}"
        except subprocess.CalledProcessError:
            return None
    
    def _get_env_info(self) -> str:
        """环境信息"""
        return f"""# Environment
- Working Directory: {self.workdir}
- Python: {sys.version.split()[0]}
- Platform: {platform.system()} {platform.release()}
- Time: {datetime.now().isoformat()}"""
```

## OpenClaw 的上下文系统

OpenClaw 有更复杂的上下文管理：

```typescript
// 简化的 OpenClaw 上下文构建
class ContextBuilder {
  private sections: Map<string, string> = new Map();
  
  constructor(
    private config: AgentConfig,
    private session: SessionContext
  ) {}
  
  async build(): Promise<string> {
    // 1. 基础指令
    await this.addBaseInstructions();
    
    // 2. 工具定义
    await this.addToolDefinitions();
    
    // 3. 技能系统
    await this.addSkillsContext();
    
    // 4. 工作区文件（关键！）
    await this.addWorkspaceFiles();
    
    // 5. 运行时信息
    await this.addRuntimeInfo();
    
    // 组装最终 prompt
    return this.assemble();
  }
  
  private async addWorkspaceFiles(): Promise<void> {
    const workspaceFiles = [
      'AGENTS.md',
      'SOUL.md',
      'USER.md', 
      'TOOLS.md',
      'IDENTITY.md',
    ];
    
    // 条件加载 MEMORY.md（只在主会话）
    if (this.session.isMainSession && !this.session.isGroupChat) {
      workspaceFiles.push('MEMORY.md');
    }
    
    const contents: string[] = [];
    
    for (const file of workspaceFiles) {
      const filepath = path.join(this.config.workspace, file);
      if (await fileExists(filepath)) {
        const content = await readFile(filepath, 'utf-8');
        contents.push(`## ${filepath}\n${content}`);
      }
    }
    
    if (contents.length > 0) {
      this.sections.set('workspace', 
        `# Project Context\nThe following project context files have been loaded:\n\n${contents.join('\n')}`
      );
    }
  }
  
  private async addSkillsContext(): Promise<void> {
    // 加载可用技能列表
    const skills = await this.loadAvailableSkills();
    
    const skillList = skills.map(s => `
  <skill>
    <name>${s.name}</name>
    <description>${s.description}</description>
    <location>${s.location}</location>
  </skill>`).join('\n');
    
    this.sections.set('skills', `
<available_skills>
${skillList}
</available_skills>

Before replying: scan <available_skills> <description> entries.
If a skill clearly applies, read its SKILL.md first.`);
  }
  
  private assemble(): string {
    // 按优先级组装
    const order = [
      'base',
      'tools',
      'skills',
      'workspace',
      'runtime',
    ];
    
    return order
      .filter(key => this.sections.has(key))
      .map(key => this.sections.get(key))
      .join('\n\n');
  }
}
```

## 上下文注入的最佳实践

### 1. 分层注入

```typescript
// 不同层次的上下文
const contextLayers = {
  // L1: 基础层 - 所有会话都有
  base: ['AGENTS.md', 'runtime_info'],
  
  // L2: 项目层 - 特定项目
  project: ['CLAUDE.md', 'package.json_summary', 'git_status'],
  
  // L3: 会话层 - 特定会话
  session: ['MEMORY.md', 'recent_files'],
  
  // L4: 任务层 - 特定任务
  task: ['relevant_code', 'error_context'],
};

function selectLayers(ctx: SessionContext): string[] {
  const layers = ['base', 'project'];
  
  if (ctx.isMainSession) {
    layers.push('session');
  }
  
  if (ctx.taskContext) {
    layers.push('task');
  }
  
  return layers;
}
```

### 2. 大小控制

```typescript
// 上下文大小管理
class ContextSizeManager {
  private maxTokens: number;
  private currentTokens: number = 0;
  
  constructor(maxTokens: number = 50000) {
    this.maxTokens = maxTokens;
  }
  
  canAdd(content: string): boolean {
    const tokens = this.estimateTokens(content);
    return this.currentTokens + tokens <= this.maxTokens;
  }
  
  add(content: string, priority: number): boolean {
    const tokens = this.estimateTokens(content);
    
    if (this.currentTokens + tokens > this.maxTokens) {
      // 尝试截断
      const available = this.maxTokens - this.currentTokens;
      if (available > 1000 && priority > 5) {
        // 高优先级内容可以截断加入
        return this.addTruncated(content, available);
      }
      return false;
    }
    
    this.currentTokens += tokens;
    return true;
  }
  
  private estimateTokens(text: string): number {
    // 粗略估算：4 字符 ≈ 1 token
    return Math.ceil(text.length / 4);
  }
}
```

### 3. 缓存与增量更新

```typescript
// 上下文缓存
class ContextCache {
  private cache = new Map<string, {
    content: string;
    mtime: number;
    hash: string;
  }>();
  
  async get(filepath: string): Promise<string | null> {
    const stat = await fs.stat(filepath).catch(() => null);
    if (!stat) return null;
    
    const cached = this.cache.get(filepath);
    if (cached && cached.mtime === stat.mtimeMs) {
      return cached.content;
    }
    
    // 文件已更改，重新加载
    const content = await fs.readFile(filepath, 'utf-8');
    this.cache.set(filepath, {
      content,
      mtime: stat.mtimeMs,
      hash: this.hash(content),
    });
    
    return content;
  }
  
  invalidate(filepath: string): void {
    this.cache.delete(filepath);
  }
  
  // 检测哪些文件需要重新注入
  async getChangedFiles(files: string[]): Promise<string[]> {
    const changed: string[] = [];
    
    for (const file of files) {
      const stat = await fs.stat(file).catch(() => null);
      if (!stat) continue;
      
      const cached = this.cache.get(file);
      if (!cached || cached.mtime !== stat.mtimeMs) {
        changed.push(file);
      }
    }
    
    return changed;
  }
}
```

### 4. 安全隔离

```typescript
// 上下文安全过滤
class ContextSecurity {
  // 敏感文件只在特定条件下加载
  private sensitiveFiles = new Set([
    'MEMORY.md',
    '.env',
    'secrets.yaml',
  ]);
  
  shouldLoad(
    filepath: string, 
    ctx: SessionContext
  ): boolean {
    const filename = path.basename(filepath);
    
    // 敏感文件检查
    if (this.sensitiveFiles.has(filename)) {
      // 只在主会话且非群聊时加载
      if (!ctx.isMainSession || ctx.isGroupChat) {
        return false;
      }
    }
    
    // 检查文件权限
    if (ctx.userRole !== 'owner') {
      const allowed = this.getAllowedFiles(ctx.userRole);
      if (!allowed.includes(filename)) {
        return false;
      }
    }
    
    return true;
  }
  
  // 内容脱敏
  sanitize(content: string): string {
    // 移除可能的敏感信息
    return content
      .replace(/(?:api[_-]?key|password|secret|token)\s*[:=]\s*['"]?[\w-]+['"]?/gi, 
               '$1: [REDACTED]')
      .replace(/\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b/g, 
               '[EMAIL]');
  }
}
```

## 常见的上下文文件模式

### AGENTS.md / CLAUDE.md - 项目指令

```markdown
# AGENTS.md

## 项目概述
这是一个 React + TypeScript 前端项目。

## 编码规范
- 使用 TypeScript strict mode
- 组件使用函数式写法
- 状态管理用 Zustand

## 测试
运行测试：`npm test`
```

### SOUL.md - Agent 人格

```markdown
# SOUL.md

你是一个专业的前端开发助手。

## 性格
- 简洁高效
- 代码优先于解释
- 主动提供最佳实践
```

### TOOLS.md - 本地配置

```markdown
# TOOLS.md

## SSH 配置
- dev-server: 192.168.1.100

## 常用命令
- 部署: `npm run deploy`
```

## 总结

上下文注入是 Agent 智能化的关键：

| 类型 | 内容 | 时机 |
|------|------|------|
| 静态注入 | AGENTS.md, CLAUDE.md | 会话启动 |
| 动态注入 | 运行时信息、Git 状态 | 每次对话 |
| 条件注入 | MEMORY.md, 敏感配置 | 满足条件时 |
| 按需注入 | 相关代码、错误上下文 | 任务需要时 |

**核心原则**：
1. **最小必要** - 只注入需要的上下文
2. **分层管理** - 基础/项目/会话/任务
3. **安全隔离** - 敏感内容条件加载
4. **大小控制** - 避免超出 context window

下一课我们讲 **Agent Checkpointing**：如何在长任务中保存和恢复状态。

---

*作业：为你的项目创建一个 AGENTS.md，定义项目结构和编码规范，然后观察 Agent 的行为变化。*
