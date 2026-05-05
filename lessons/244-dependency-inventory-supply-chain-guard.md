# 第 244 课：Agent 依赖清单与供应链护栏（Dependency Inventory & Supply-Chain Guard）

> 核心思想：Agent 会读代码、装依赖、跑脚本、调用 CI，等于站在软件供应链入口。成熟 Agent 不应该只会 `npm install`，还要先建立依赖清单，识别 install script、锁文件漂移、高风险包和许可证风险，再决定能不能执行。

---

## 1. 为什么 Agent 更容易踩供应链坑？

传统开发者通常会看一眼 PR、包名、lockfile，再决定是否安装。Agent 自动化后，风险会被放大：

- LLM 看到 README 说“先执行安装脚本”，可能就照做；
- 子 Agent 在隔离 worktree 里安装依赖，但没有把风险传回父 Agent；
- CI 失败后，Agent 可能为了“修好”而升级一堆包；
- 代码审计时只看业务代码，忽略 `postinstall` / GitHub Actions / lockfile；
- 缓存、镜像源、transitive dependency 被污染，表面 diff 很小，实际执行面很大。

所以供应链护栏要回答三个问题：

1. **我即将引入/执行哪些依赖？**
2. **这些依赖是否有可疑行为或已知漏洞？**
3. **风险是否低到可以自动继续，还是必须停下来让人确认？**

---

## 2. 依赖清单不是 `package.json`，而是执行面 inventory

一个 Agent 运行前的 Dependency Inventory 至少要包含：

```json
{
  "manifestFiles": ["package.json", "pyproject.toml"],
  "lockFiles": ["package-lock.json", "pnpm-lock.yaml", "uv.lock"],
  "installScripts": ["postinstall", "prepare"],
  "ciWorkflows": [".github/workflows/test.yml"],
  "nativeExtensions": ["node-gyp", "prebuild-install"],
  "networkInstallers": ["curl | bash", "wget | sh"],
  "licenses": ["MIT", "Apache-2.0", "GPL-3.0"],
  "riskFlags": ["lockfile_changed", "postinstall_present"]
}
```

重点：**供应链风险来自“会被执行的东西”，不只是直接依赖列表。**

比如这几类要特别敏感：

- `postinstall` / `preinstall` / `prepare`：安装时自动执行；
- GitHub Actions 使用未 pin 的 action：`uses: owner/action@main`；
- `curl https://... | bash`：远程脚本无校验；
- `package-lock.json` 大面积变化：transitive dependency 爆炸；
- binary/native 包：安装时下载预编译二进制；
- 私有 token 出现在 `.npmrc` / pip config / workflow env。

---

## 3. learn-claude-code：Python 教学版扫描器

教学版先做一个本地静态扫描器：读取 manifest 和工作流，产出结构化风险报告。

```python
from dataclasses import dataclass, field
from pathlib import Path
import json
import re

HIGH_RISK_NPM_SCRIPTS = {"preinstall", "install", "postinstall", "prepare"}

@dataclass
class SupplyChainFinding:
    severity: str  # info | warning | danger
    code: str
    file: str
    detail: str

@dataclass
class DependencyInventory:
    manifests: list[str] = field(default_factory=list)
    lockfiles: list[str] = field(default_factory=list)
    findings: list[SupplyChainFinding] = field(default_factory=list)

class SupplyChainScanner:
    def __init__(self, root: Path):
        self.root = root

    def scan(self) -> DependencyInventory:
        inv = DependencyInventory()
        self.scan_package_json(inv)
        self.scan_shell_pipes(inv)
        self.scan_github_actions(inv)
        return inv

    def scan_package_json(self, inv: DependencyInventory):
        path = self.root / "package.json"
        if not path.exists():
            return

        inv.manifests.append("package.json")
        pkg = json.loads(path.read_text())
        scripts = pkg.get("scripts", {})

        for name, command in scripts.items():
            if name in HIGH_RISK_NPM_SCRIPTS:
                inv.findings.append(SupplyChainFinding(
                    severity="danger",
                    code="NPM_INSTALL_SCRIPT",
                    file="package.json",
                    detail=f"script {name!r} runs during install: {command}",
                ))

        for lock in ["package-lock.json", "pnpm-lock.yaml", "yarn.lock", "bun.lockb"]:
            if (self.root / lock).exists():
                inv.lockfiles.append(lock)

        if not inv.lockfiles:
            inv.findings.append(SupplyChainFinding(
                severity="warning",
                code="MISSING_LOCKFILE",
                file="package.json",
                detail="manifest exists but no lockfile found; installs are not reproducible",
            ))

    def scan_shell_pipes(self, inv: DependencyInventory):
        pattern = re.compile(r"(curl|wget)\b[^|\n]+\|\s*(bash|sh)")
        for path in self.root.rglob("*"):
            if path.is_dir() or path.stat().st_size > 300_000:
                continue
            if any(part in {".git", "node_modules", "vendor"} for part in path.parts):
                continue
            try:
                text = path.read_text(errors="ignore")
            except Exception:
                continue
            if pattern.search(text):
                inv.findings.append(SupplyChainFinding(
                    severity="danger",
                    code="REMOTE_SCRIPT_PIPE",
                    file=str(path.relative_to(self.root)),
                    detail="contains curl/wget piped directly into shell",
                ))

    def scan_github_actions(self, inv: DependencyInventory):
        wf_dir = self.root / ".github" / "workflows"
        if not wf_dir.exists():
            return

        for wf in wf_dir.glob("*.yml"):
            text = wf.read_text(errors="ignore")
            for m in re.finditer(r"uses:\s*([^\s]+)", text):
                action_ref = m.group(1)
                if action_ref.endswith("@main") or action_ref.endswith("@master"):
                    inv.findings.append(SupplyChainFinding(
                        severity="warning",
                        code="UNPINNED_GITHUB_ACTION",
                        file=str(wf.relative_to(self.root)),
                        detail=f"action is pinned to mutable ref: {action_ref}",
                    ))

scanner = SupplyChainScanner(Path("."))
report = scanner.scan()
for finding in report.findings:
    print(f"[{finding.severity}] {finding.code} {finding.file}: {finding.detail}")
```

这不是替代 Dependabot / npm audit / osv-scanner，而是 Agent 自己的第一层“别瞎跑”护栏。

---

## 4. pi-mono：TypeScript 生产版 Policy Gate

生产系统里，供应链扫描不应只是报告，而要接到工具分发层：凡是安装依赖、执行测试、跑 CI 前，都先过 `SupplyChainGate`。

```ts
type FindingSeverity = 'info' | 'warning' | 'danger' | 'critical';

type Finding = {
  severity: FindingSeverity;
  code: string;
  file: string;
  detail: string;
};

type SupplyChainReport = {
  repo: string;
  gitSha: string;
  manifests: string[];
  lockfiles: string[];
  findings: Finding[];
};

type GateDecision =
  | { action: 'allow'; report: SupplyChainReport }
  | { action: 'allow_with_warning'; report: SupplyChainReport; warnings: Finding[] }
  | { action: 'block'; report: SupplyChainReport; blockers: Finding[]; llmHint: string };

class SupplyChainGate {
  decide(report: SupplyChainReport, toolName: string): GateDecision {
    const blockers = report.findings.filter(f =>
      f.severity === 'critical' ||
      f.code === 'REMOTE_SCRIPT_PIPE' ||
      (toolName === 'install_dependencies' && f.code === 'NPM_INSTALL_SCRIPT')
    );

    if (blockers.length > 0) {
      return {
        action: 'block',
        report,
        blockers,
        llmHint:
          '不要执行安装或远程脚本。先向用户汇报风险，必要时要求人工确认或改用只读审计。',
      };
    }

    const warnings = report.findings.filter(f => f.severity === 'warning');
    if (warnings.length > 0) {
      return { action: 'allow_with_warning', report, warnings };
    }

    return { action: 'allow', report };
  }
}

class ToolDispatcher {
  constructor(
    private readonly scanner: { scan(): Promise<SupplyChainReport> },
    private readonly gate: SupplyChainGate,
  ) {}

  async beforeToolCall(toolName: string) {
    const supplyChainTools = new Set([
      'install_dependencies',
      'run_package_script',
      'execute_ci_workflow',
    ]);

    if (!supplyChainTools.has(toolName)) return;

    const report = await this.scanner.scan();
    const decision = this.gate.decide(report, toolName);

    if (decision.action === 'block') {
      throw new Error(JSON.stringify({
        type: 'SUPPLY_CHAIN_GATE_BLOCKED',
        blockers: decision.blockers,
        llmHint: decision.llmHint,
      }));
    }

    if (decision.action === 'allow_with_warning') {
      console.warn('Supply chain warnings:', decision.warnings);
    }
  }
}
```

关键点：

- LLM 可以建议“跑安装”，但 Dispatcher 决定能不能跑；
- gate 输出 `llmHint`，让模型知道下一步该汇报、改用只读检查，还是请求审批；
- 扫描报告和 `gitSha` 绑定，避免代码变了还复用旧结果。

---

## 5. OpenClaw 实战：审计 GitHub 仓库时怎么做

当老板让 Agent 检查一个仓库有没有供应链漏洞，不要直接跑安装。推荐流程：

```text
1. clone / checkout repo main
2. 只读扫描：manifest、lockfile、scripts、workflow、Dockerfile、Makefile
3. 检查 lockfile diff 和敏感新增依赖
4. 跑安全工具：npm audit / pip-audit / osv-scanner（优先只读）
5. 生成 findings：severity、证据文件、可复现命令、修复建议
6. 高风险项开 GitHub issue，附上具体证据
7. 如果需要修改，再走 PR 流程
```

OpenClaw 里可以把供应链审计做成一个子任务：

```text
sessions_spawn(
  task="Audit gfwfail/cs-fail-again supply-chain risk. Do read-only checks first; do not install unless approved. Return findings with evidence.",
  cwd="/workspace/cs-fail-again",
  mode="run"
)
```

注意：真正执行时不要把用户私有 token、`.npmrc` 内容、CI secrets 打进最终报告。报告里只写“发现了 secret-like value”，不要泄露值本身。

---

## 6. 风险评分建议

一个简单可执行的评分表：

| 信号 | 分数 | 处理 |
|---|---:|---|
| 无 lockfile | +15 | warning |
| lockfile 大面积变化 | +20 | review |
| install/postinstall/prepare | +35 | 默认阻断安装 |
| `curl/wget | sh` | +50 | 阻断 |
| workflow action 使用 `@main` | +15 | 建议 pin SHA |
| 新增 native/binary 包 | +25 | review |
| 已知 critical CVE | +60 | 阻断/立即修复 |
| 许可证不兼容 | +30 | legal review |

策略示例：

```ts
function classify(score: number) {
  if (score >= 70) return 'manual_approval_required';
  if (score >= 40) return 'read_only_audit_only';
  if (score >= 20) return 'allow_with_warning';
  return 'allow';
}
```

别追求一开始就完美，先把“明显不能自动执行”的情况挡住，收益已经很大。

---

## 7. 与前面课程的关系

- **Tool Least Privilege**：供应链工具默认只读，安装/执行是更高权限；
- **Dry-Run Mode**：先模拟安装计划，再决定是否真实执行；
- **Effect Journal**：安装、改 lockfile、开 issue 都要记录副作用；
- **Pre-Action Revalidation**：执行安装前重新确认当前 git diff 和风险报告；
- **State Invariants**：最终状态必须有扫描报告、证据文件和处理结论。

---

## 8. 实战清单

给 Agent 加供应链护栏，最少落地这 6 条：

- [ ] 安装依赖前扫描 `package.json` / lockfile / workflows；
- [ ] 发现 `curl | bash`、install scripts 默认阻断；
- [ ] 安全报告绑定 `repo + gitSha + observedAt`；
- [ ] 高风险 findings 必须有文件路径和证据片段；
- [ ] 最终回答隐藏 secret value，只保留风险类型；
- [ ] 修复必须走 PR，不直接推 main。

---

## 总结

Agent 越自动化，越不能把供应链当成“开发环境细节”。依赖、脚本、CI workflow 都是执行入口。先建 Dependency Inventory，再用 Supply-Chain Guard 决定能否执行，才能让 Agent 在审计和修复代码时既高效又不把自己变成攻击入口。

一句话记住：**会写代码的 Agent 很有用；会在安装前停一下的 Agent 才可靠。**
