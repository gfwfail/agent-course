# Agent 回归包覆盖率与风险优先级（Regression Pack Coverage & Risk Prioritization）

> Regression Pack 不是越多越好，而是要覆盖真正会造成损失的行为路径：副作用、权限放宽、证据失效、外部发布和生产操作。

上一课讲了 Incident Regression Pack：把事故变成可 replay 的闸门。今天补上一个生产里很容易被忽略的问题：**回归包也需要覆盖率和优先级**。

如果只按“想到什么写什么”，最后会得到一堆漂亮 case，但关键风险仍然裸奔。成熟 Agent 应该把 coverage 当成安全指标来管理：

1. **按风险面覆盖**：message/send、git push、部署、删除、转账、解密、审批绕过。
2. **按事故类型覆盖**：raw secret 外发、revoked evidence、tenant 越权、dry-run 漏挡、policy 放宽。
3. **按变更入口覆盖**：prompt、tool schema、policy、memory/runbook、formatter、dispatcher。
4. **按损失等级排序**：先补高影响、高频、高不可逆、低可观测的路径。
5. **把缺口变成待办**：coverage gap 不是报告文字，而是能阻断发布或创建整改项的结构化记录。

## 1. learn-claude-code：最小覆盖率扫描器

教学版可以先用一份风险矩阵描述“哪些能力必须有 regression case”。

~~~python
from dataclasses import dataclass
from typing import Literal

Risk = Literal["low", "medium", "high", "critical"]

@dataclass(frozen=True)
class RiskSurface:
    name: str
    required_tags: set[str]
    min_cases: int
    risk: Risk

@dataclass
class RegressionCase:
    id: str
    tags: set[str]
    min_decision: str
    forbidden_tools: list[str]

SURFACES = [
    RiskSurface("telegram_egress", {"egress", "telegram", "secret_scan"}, 2, "high"),
    RiskSurface("git_push", {"git", "external_side_effect", "proof"}, 1, "high"),
    RiskSurface("decrypt_cache", {"decrypt", "quota", "purpose_binding"}, 2, "critical"),
    RiskSurface("revoked_evidence", {"evidence", "revocation", "block"}, 2, "critical"),
    RiskSurface("dry_run_safety", {"dry_run", "side_effect_block"}, 1, "critical"),
]

def coverage_report(cases: list[RegressionCase]) -> list[dict]:
    report = []
    for surface in SURFACES:
        matched = [
            case for case in cases
            if surface.required_tags.issubset(case.tags)
        ]
        ok = len(matched) >= surface.min_cases
        report.append({
            "surface": surface.name,
            "risk": surface.risk,
            "required": surface.min_cases,
            "actual": len(matched),
            "ok": ok,
            "case_ids": [case.id for case in matched],
        })
    return report

def fail_on_critical_gaps(report: list[dict]) -> None:
    gaps = [item for item in report if not item["ok"] and item["risk"] == "critical"]
    if gaps:
        names = ", ".join(item["surface"] for item in gaps)
        raise AssertionError("critical regression coverage gaps: " + names)
~~~

这里的关键不是百分比，而是**风险面是否被覆盖**。一个 95% 的普通路径覆盖率，挡不住 1 条“dry-run 下仍然真实发消息”的事故。

## 2. pi-mono：CoverageGate 接在发布前

生产版可以把 case metadata 标准化：每个 regression case 必须声明它覆盖的 capability、risk、changeScope 和 incidentClass。

~~~ts
type Risk = "low" | "medium" | "high" | "critical";

type RegressionCaseMeta = {
  id: string;
  tags: string[];
  capability: "egress" | "git" | "deploy" | "decrypt" | "evidence" | "tool_dispatch";
  risk: Risk;
  changeScopes: Array<"prompt" | "policy" | "tool_schema" | "memory" | "release">;
};

type RequiredSurface = {
  name: string;
  tags: string[];
  minCases: number;
  minRisk: Risk;
  blocking: boolean;
};

const rank: Record<Risk, number> = { low: 0, medium: 1, high: 2, critical: 3 };

export class RegressionCoverageGate {
  constructor(
    private readonly cases: RegressionCaseMeta[],
    private readonly required: RequiredSurface[],
  ) {}

  check(changeScope: RegressionCaseMeta["changeScopes"][number]) {
    const gaps = [];

    for (const surface of this.required) {
      const matched = this.cases.filter((testCase) =>
        testCase.changeScopes.includes(changeScope) &&
        rank[testCase.risk] >= rank[surface.minRisk] &&
        surface.tags.every((tag) => testCase.tags.includes(tag)),
      );

      if (matched.length < surface.minCases) {
        gaps.push({
          surface: surface.name,
          required: surface.minCases,
          actual: matched.length,
          blocking: surface.blocking,
        });
      }
    }

    return {
      ok: gaps.every((gap) => !gap.blocking),
      gaps,
    };
  }
}
~~~

这个 gate 不替代 replay gate，而是站在 replay gate 前面问一句：**你准备 replay 的东西，真的覆盖了这次变更可能打坏的风险面吗？**

例如改了 `MessageEgressMiddleware`，至少要覆盖：

- secret scan 命中后不能发 Telegram；
- declassified summary 可以发，但必须绑定 artifact；
- revoked evidence 触发 recall/notice，而不是继续引用旧消息；
- dry-run 下只能写 fake outbox。

## 3. OpenClaw：课程 Cron 的风险面

这门课的 cron 每 3 小时做五类副作用：

1. 写 lesson 文件；
2. 更新 README/TOOLS；
3. 发 Telegram 群消息；
4. git commit；
5. git push。

所以它的 regression coverage 不能只测“lesson 文件生成成功”，还要覆盖这些高风险面：

~~~json
{
  "requiredSurfaces": [
    {
      "name": "telegram_no_raw_secret",
      "tags": ["egress", "telegram", "secret_scan"],
      "minCases": 2,
      "minRisk": "high",
      "blocking": true
    },
    {
      "name": "git_push_requires_clean_proof",
      "tags": ["git", "proof", "external_side_effect"],
      "minCases": 1,
      "minRisk": "high",
      "blocking": true
    },
    {
      "name": "dry_run_blocks_message_send",
      "tags": ["dry_run", "side_effect_block"],
      "minCases": 1,
      "minRisk": "critical",
      "blocking": true
    },
    {
      "name": "tools_memory_update_no_secret_leak",
      "tags": ["memory", "redaction", "audit"],
      "minCases": 1,
      "minRisk": "medium",
      "blocking": false
    }
  ]
}
~~~

注意最后一条是 non-blocking：不是所有缺口都要阻断。成熟系统会区分：

- **blocking gap**：可能导致真实副作用、越权、泄密、不可逆操作。
- **warning gap**：影响可维护性或审计完整性，但不应卡死紧急修复。
- **tracked gap**：自动生成整改项，设置 owner/dueAt/verification。

## 4. 实战 Checklist

- **每个高风险工具至少有 1 条 forbidden tool case**：证明它在错误条件下不会被调用。
- **每个外发通道至少有 1 条 secret/redaction case**：Telegram、GitHub、Email、日志分别测。
- **每个权限放宽变更必须跑 shadow + replay**：coverage 不足时先不晋级。
- **每个事故 closeout 必须补 coverage tag**：否则 case 后面无法被正确选中。
- **覆盖率按风险面算，不按 case 数量炫耀**：100 条低风险格式 case 不等于安全。
- **缺口要结构化输出**：能被 CI、cron、OpenClaw completion gate 消费。

## 总结

Regression Pack 让事故经验可回放；Coverage Gate 让回放不靠运气。

成熟 Agent 的安全回归不是“我们有很多测试”，而是“每个会造成真实损失的风险面，都有对应的 replay 证据和缺口处理策略”。
