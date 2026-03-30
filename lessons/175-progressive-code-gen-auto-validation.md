# 175 - Agent 渐进式代码生成与自动验证

**Progressive Code Generation & Auto-Validation**

---

## 为什么需要"渐进式"代码生成？

普通 LLM 代码生成的问题：一次输出一大段代码 → 用户跑一下 → 报错 → 再问 → 再跑。
这个来回往往要很多轮，而且 LLM 看不到真实执行结果，修复靠猜。

**渐进式代码生成** 让 Agent 自己完成这个循环：

```
生成代码 → 自动执行 → 捕获输出/错误 → 注入结果 → 自动修复 → 再执行...
```

直到测试通过，或者达到最大重试次数，才把结果交给用户。

这正是 Claude Code、Codex、Devin 的核心模式。

---

## 核心循环设计

```typescript
interface CodeGenResult {
  code: string;
  language: string;
  filename: string;
}

interface ExecutionResult {
  stdout: string;
  stderr: string;
  exitCode: number;
  durationMs: number;
}

interface ValidationResult {
  passed: boolean;
  failedTests: string[];
  coverage?: number;
}
```

完整的代码生成-验证循环：

```typescript
// pi-mono 风格
class ProgressiveCodeGenerator {
  private maxAttempts = 5;

  async generate(task: string, ctx: AgentContext): Promise<CodeGenResult> {
    let attempt = 0;
    let lastError: string | null = null;
    let currentCode: string | null = null;

    while (attempt < this.maxAttempts) {
      attempt++;
      
      // 1. 构建生成 prompt（首次生成 vs 修复）
      const prompt = lastError
        ? this.buildFixPrompt(task, currentCode!, lastError, attempt)
        : this.buildInitialPrompt(task);

      // 2. 调用 LLM 生成代码
      const result = await ctx.llm.complete(prompt, {
        tools: [], // 代码生成阶段不需要外部工具
        temperature: attempt === 1 ? 0.3 : 0.1, // 修复阶段降低随机性
      });
      
      currentCode = this.extractCode(result.content);
      
      // 3. 沙箱执行
      const execution = await this.executeInSandbox(currentCode, ctx);
      
      // 4. 运行测试验证
      const validation = await this.runValidation(currentCode, task, ctx);
      
      // 5. 记录本次尝试
      ctx.log({
        attempt,
        exitCode: execution.exitCode,
        passed: validation.passed,
        stderr: execution.stderr.slice(0, 500),
      });
      
      if (validation.passed && execution.exitCode === 0) {
        return {
          code: currentCode,
          language: this.detectLanguage(currentCode),
          filename: this.suggestFilename(task),
        };
      }
      
      // 6. 提取错误供下次修复
      lastError = this.buildErrorContext(execution, validation);
    }
    
    throw new Error(`代码生成失败，已尝试 ${this.maxAttempts} 次。最后错误：${lastError}`);
  }

  private buildFixPrompt(
    task: string,
    code: string,
    error: string,
    attempt: number
  ): string {
    return `
你之前生成的代码有问题，请修复。

## 原始需求
${task}

## 当前代码
\`\`\`
${code}
\`\`\`

## 执行错误（第 ${attempt - 1} 次尝试）
${error}

## 要求
- 只修复错误，保持原有逻辑
- 确保所有边界情况已处理
- 直接返回完整可执行代码，无需解释
    `.trim();
  }

  private buildErrorContext(
    exec: ExecutionResult,
    validation: ValidationResult
  ): string {
    const parts: string[] = [];
    
    if (exec.exitCode !== 0) {
      parts.push(`退出码: ${exec.exitCode}`);
      if (exec.stderr) parts.push(`错误输出:\n${exec.stderr}`);
    }
    
    if (!validation.passed && validation.failedTests.length > 0) {
      parts.push(`失败的测试:\n${validation.failedTests.join('\n')}`);
    }
    
    return parts.join('\n\n');
  }
}
```

---

## 沙箱执行层

代码执行必须隔离，不能直接跑在宿主机上。

```typescript
class SandboxExecutor {
  // 方案1：Docker 容器（最安全）
  async executeInDocker(
    code: string,
    language: string,
    timeoutMs = 10_000
  ): Promise<ExecutionResult> {
    const image = this.getDockerImage(language);
    const tmpFile = `/tmp/agent_code_${Date.now()}`;
    const ext = this.getExtension(language);
    
    // 写入临时文件
    await fs.writeFile(`${tmpFile}.${ext}`, code, 'utf-8');
    
    const start = Date.now();
    try {
      const { stdout, stderr, exitCode } = await exec(
        `docker run --rm \
          --memory="128m" \
          --cpus="0.5" \
          --network=none \
          --read-only \
          -v ${tmpFile}.${ext}:/code.${ext}:ro \
          ${image} \
          timeout ${timeoutMs / 1000} ${this.getRunCommand(language)} /code.${ext}`,
        { timeout: timeoutMs + 2000 }
      );
      
      return {
        stdout,
        stderr,
        exitCode: exitCode ?? 0,
        durationMs: Date.now() - start,
      };
    } finally {
      await fs.unlink(`${tmpFile}.${ext}`).catch(() => {});
    }
  }
  
  // 方案2：OpenClaw exec 工具（已内置沙箱）
  // OpenClaw 的 exec tool 运行在受控环境中
  // Agent 可以直接调用 exec tool 来运行生成的代码
  
  private getDockerImage(lang: string): string {
    const images: Record<string, string> = {
      python: 'python:3.12-alpine',
      typescript: 'node:22-alpine',
      javascript: 'node:22-alpine',
      bash: 'alpine:latest',
      go: 'golang:1.22-alpine',
    };
    return images[lang] ?? 'alpine:latest';
  }
}
```

---

## 智能测试生成

Agent 不只生成代码，还要**自动生成测试用例**：

```typescript
class TestGenerator {
  async generateTests(
    code: string,
    task: string,
    ctx: AgentContext
  ): Promise<string> {
    const prompt = `
为以下代码生成完整的单元测试。

## 任务需求
${task}

## 待测代码
\`\`\`
${code}
\`\`\`

要求：
1. 覆盖正常流程、边界值、错误情况
2. 使用该语言标准测试框架（Python: pytest，TypeScript: vitest）
3. 测试应该自包含，不依赖外部服务
4. 直接返回可执行的测试代码
    `;
    
    const result = await ctx.llm.complete(prompt, { temperature: 0.1 });
    return this.extractCode(result.content);
  }
  
  async runTests(
    testCode: string,
    sourceCode: string,
    ctx: AgentContext
  ): Promise<ValidationResult> {
    // 合并源码和测试到临时目录
    const tmpDir = `/tmp/agent_test_${Date.now()}`;
    await fs.mkdir(tmpDir, { recursive: true });
    await fs.writeFile(`${tmpDir}/code.py`, sourceCode);
    await fs.writeFile(`${tmpDir}/test_code.py`, testCode);
    
    const exec = await this.sandbox.execute(
      `cd ${tmpDir} && python -m pytest test_code.py -v --tb=short 2>&1`,
      { timeoutMs: 30_000 }
    );
    
    const failedTests = this.parseFailedTests(exec.stdout);
    
    return {
      passed: exec.exitCode === 0,
      failedTests,
      coverage: this.parseCoverage(exec.stdout),
    };
  }
  
  private parseFailedTests(output: string): string[] {
    // 解析 pytest 输出，提取失败的测试名
    const failPattern = /FAILED (.+?) - (.+)/g;
    const failures: string[] = [];
    let match;
    while ((match = failPattern.exec(output)) !== null) {
      failures.push(`${match[1]}: ${match[2]}`);
    }
    return failures;
  }
}
```

---

## OpenClaw 中的实际应用

OpenClaw 的 Agent 可以直接用 `exec` 工具来做代码验证：

```
# Agent 的工具调用序列：

1. LLM 生成代码 → 写入 /tmp/solution.py
2. exec: python /tmp/solution.py
3. 如果失败：把 stderr 注入回 LLM context
4. LLM 输出修复版本 → 覆盖写入文件
5. exec: python -m pytest /tmp/test_solution.py
6. 循环直到通过
```

这正是 Claude Code 在处理编程任务时的工作方式：不是一次性给出答案，而是反复执行-修复，直到代码真的能跑通。

---

## 质量门控：何时停止重试？

```typescript
class QualityGate {
  private thresholds = {
    minCoverage: 80,       // 测试覆盖率最低要求
    maxAttempts: 5,        // 最大重试次数
    maxDurationMs: 120_000, // 总超时
    requiredTests: ['happy_path', 'edge_case', 'error_case'], // 必须覆盖的测试类型
  };
  
  evaluate(
    results: Array<{ validation: ValidationResult; attempt: number }>,
    elapsedMs: number
  ): 'pass' | 'fail' | 'retry' {
    const latest = results[results.length - 1];
    
    // 硬停：超时或超次数
    if (
      latest.attempt >= this.thresholds.maxAttempts ||
      elapsedMs >= this.thresholds.maxDurationMs
    ) {
      return latest.validation.passed ? 'pass' : 'fail';
    }
    
    // 通过条件
    if (
      latest.validation.passed &&
      (latest.validation.coverage ?? 0) >= this.thresholds.minCoverage
    ) {
      return 'pass';
    }
    
    // 检查是否在进步（避免无效重试）
    if (results.length >= 3) {
      const recentFailures = results.slice(-3);
      const allFailing = recentFailures.every(r => !r.validation.passed);
      const sameErrors = this.areSameErrors(recentFailures);
      
      if (allFailing && sameErrors) {
        // 卡住了，换策略：提高 temperature 或请求更简单的实现
        return 'retry'; // 带标志：需要换思路
      }
    }
    
    return 'retry';
  }
  
  private areSameErrors(
    results: Array<{ validation: ValidationResult }>
  ): boolean {
    if (results.length < 2) return false;
    const errorSets = results.map(r => 
      new Set(r.validation.failedTests.map(t => t.split(':')[0]))
    );
    return errorSets.every(s => 
      s.size === errorSets[0].size &&
      [...s].every(t => errorSets[0].has(t))
    );
  }
}
```

---

## 实战示例：让 Agent 实现一个函数

```typescript
// 用户请求：帮我写一个解析 CSV 文件并统计词频的 Python 函数

const generator = new ProgressiveCodeGenerator();
const result = await generator.generate(
  '写一个 Python 函数 word_frequency(csv_path, column_name)，' +
  '读取 CSV 文件指定列，统计词频，返回按频率降序排列的 dict',
  ctx
);

// Agent 内部经历了什么：
// Attempt 1: 生成初版代码 → csv.reader 没处理 header → 词频包含了列名
// Attempt 2: 修复 header 问题 → 没处理空值 → AttributeError
// Attempt 3: 加 None 检查 → 所有测试通过 ✅
// 总耗时: 8.3秒，用户无感知

console.log(`代码已生成，经过 ${result.attempts} 次迭代验证`);
```

---

## 关键设计原则

| 原则 | 说明 |
|------|------|
| **错误要完整** | stderr + 行号 + 上下文一起喂给 LLM，不要截断 |
| **修复要保守** | 重试时降低 temperature，避免大改动引入新 bug |
| **测试先行** | 先生成测试，再生成实现，成功率更高 |
| **沙箱必须** | 永远不要在宿主机直接 eval 生成的代码 |
| **进度透明** | 实时告知用户"正在第 N 次修复"，避免黑箱感 |
| **优雅放弃** | 卡住时主动告知用户，而不是无限循环 |

---

## 与其他模式的关系

- **Sandbox 安全执行**（之前讲过）：本课的执行层基础
- **Self-Correction 自我纠错**（之前讲过）：本课是其在代码生成领域的专项实现
- **Tool Timeout & Cancellation**：代码执行必须设超时，防止死循环

---

## 小结

渐进式代码生成 = **代码生成** + **自动执行** + **错误反馈** + **迭代修复**

核心价值不在于"能生成代码"，而在于**闭环验证**——Agent 自己知道代码对不对，不需要人手动测。

这就是为什么 Claude Code 比普通 ChatGPT 写代码强得多：它不是在"猜"代码对不对，它是在"跑"代码验证对不对。

> 代码是写出来的，也是跑出来的。Agent 写代码，也要跑代码。
