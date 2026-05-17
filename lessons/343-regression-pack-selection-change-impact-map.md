# Agent 回归包选择器与变更影响映射（Regression Pack Selection & Change Impact Map）

> Coverage Gate 说明“关键风险面有没有 case”，Selection Gate 解决“这次变更到底该 replay 哪些 case”。

上一课讲了 Regression Pack Coverage：别用一堆低风险 case 假装安全。今天继续往生产化走一步：**根据本次变更自动选择最小但足够的 regression set**。

为什么重要？因为 Agent 系统的回归测试很贵：

1. 有些 case 要跑 LLM，慢且花钱。
2. 有些 case 要重放工具轨迹，需要准备 cassette/evidence。
3. 全量 replay 很稳，但每次都全量会拖慢发布，最后大家会绕过它。
4. 只跑“看起来相关”的 case 又容易漏掉权限、外发、记忆、工具 schema 这种隐性影响。

成熟做法是建立一张 **Change Impact Map**：把文件 diff、策略 diff、prompt diff、tool schema diff 映射到 capability / risk surface / required tags，再从 Regression Pack 里挑 case。

## 1. learn-claude-code：最小影响映射器

教学版先用规则匹配文件路径，输出这次变更影响的能力面。

~~~python
from dataclasses import dataclass
from fnmatch import fnmatch

@dataclass(frozen=True)
class ChangedFile:
    path: str
    kind: str  # added | modified | deleted

@dataclass(frozen=True)
class ImpactRule:
    pattern: str
    capabilities: set[str]
    required_tags: set[str]
    risk: str

@dataclass(frozen=True)
class RegressionCase:
    id: str
    tags: set[str]
    risk: str
    estimated_seconds: int

RULES = [
    ImpactRule("prompts/**", {"planning", "tool_selection"}, {"prompt", "decision"}, "high"),
    ImpactRule("tools/message/**", {"egress"}, {"telegram", "secret_scan", "dry_run"}, "critical"),
    ImpactRule("tools/git/**", {"git_side_effect"}, {"git", "external_side_effect", "proof"}, "high"),
    ImpactRule("policy/**", {"authorization"}, {"policy", "deny_path", "approval"}, "critical"),
    ImpactRule("memory/**", {"memory"}, {"memory", "pollution_guard"}, "medium"),
]

RANK = {"low": 0, "medium": 1, "high": 2, "critical": 3}

def impact_map(files: list[ChangedFile]) -> list[ImpactRule]:
    impacted = []
    for rule in RULES:
        if any(fnmatch(item.path, rule.pattern) for item in files):
            impacted.append(rule)
    return impacted

def select_cases(files: list[ChangedFile], cases: list[RegressionCase]) -> list[RegressionCase]:
    impacted = impact_map(files)
    wanted_tags = set().union(*(rule.required_tags for rule in impacted))
    min_risk = max((RANK[rule.risk] for rule in impacted), default=0)

    selected = [
        case for case in cases
        if case.tags & wanted_tags and RANK[case.risk] >= min_risk - 1
    ]

    return sorted(selected, key=lambda case: (-RANK[case.risk], case.estimated_seconds))
~~~

这里故意保留一个“风险降一级也可选”的策略：改了 critical policy，除了 critical case，也允许 high case 进入候选。因为很多事故不是单点坏，而是 policy + formatter + egress 的组合坏。

## 2. pi-mono：Selection Gate 要解释为什么选

生产版不要只返回 case id，还要返回选择理由。否则 CI 失败时没人知道这批 replay 从哪里来的。

~~~ts
type Risk = "low" | "medium" | "high" | "critical";

type FileChange = {
  path: string;
  status: "added" | "modified" | "deleted";
};

type ImpactRule = {
  id: string;
  match: RegExp;
  capability: "egress" | "git" | "policy" | "memory" | "tool_dispatch";
  requiredTags: string[];
  risk: Risk;
};

type RegressionCase = {
  id: string;
  tags: string[];
  risk: Risk;
  estimatedCost: number;
};

type SelectedCase = {
  id: string;
  reason: string;
  matchedRuleIds: string[];
};

const rank: Record<Risk, number> = { low: 0, medium: 1, high: 2, critical: 3 };

export class RegressionSelectionGate {
  constructor(
    private readonly rules: ImpactRule[],
    private readonly cases: RegressionCase[],
  ) {}

  select(changes: FileChange[]): { selected: SelectedCase[]; missingTags: string[] } {
    const matchedRules = this.rules.filter((rule) =>
      changes.some((change) => rule.match.test(change.path)),
    );

    const requiredTags = new Set(matchedRules.flatMap((rule) => rule.requiredTags));
    const selected: SelectedCase[] = [];
    const coveredTags = new Set<string>();

    for (const testCase of this.cases) {
      const hitTags = testCase.tags.filter((tag) => requiredTags.has(tag));
      if (hitTags.length === 0) continue;

      const matchedRuleIds = matchedRules
        .filter((rule) => rule.requiredTags.some((tag) => testCase.tags.includes(tag)))
        .map((rule) => rule.id);

      const maxRuleRisk = Math.max(
        ...matchedRules
          .filter((rule) => matchedRuleIds.includes(rule.id))
          .map((rule) => rank[rule.risk]),
      );

      if (rank[testCase.risk] + 1 < maxRuleRisk) continue;

      hitTags.forEach((tag) => coveredTags.add(tag));
      selected.push({
        id: testCase.id,
        reason: `covers tags ${hitTags.join(", ")} for rules ${matchedRuleIds.join(", ")}`,
        matchedRuleIds,
      });
    }

    return {
      selected,
      missingTags: [...requiredTags].filter((tag) => !coveredTags.has(tag)),
    };
  }
}
~~~

Selection Gate 的输出应该进证据包：

~~~json
{
  "changeSet": ["tools/message/send.ts", "policy/egress.yaml"],
  "selectedCases": [
    {
      "id": "telegram-secret-block",
      "reason": "covers tags telegram, secret_scan for rules message_egress, egress_policy"
    },
    {
      "id": "dry-run-no-real-send",
      "reason": "covers tags dry_run for rules message_egress"
    }
  ],
  "missingTags": []
}
~~~

这样发布后如果出事故，能反查：当时为什么没跑某个 case？是规则没覆盖、case metadata 错了，还是人为 override？

## 3. OpenClaw：课程 Cron 怎么用

这门课的 cron 每次至少会改：

- `agent-course/lessons/*.md`
- `agent-course/README.md`
- `TOOLS.md`
- `memory/YYYY-MM-DD.md`

如果只是 lesson 文案变化，可以跑轻量 case：

~~~json
{
  "changeClass": "course_content",
  "requiredTags": ["markdown_render", "no_secret_in_egress", "git_clean_commit"],
  "mode": "fast_replay"
}
~~~

但如果变更触碰发送、Git、权限、记忆维护逻辑，就要升级：

~~~json
{
  "changeClass": "side_effect_pipeline",
  "requiredTags": [
    "telegram",
    "secret_scan",
    "dry_run",
    "git_push",
    "proof",
    "memory_pollution_guard"
  ],
  "mode": "blocking_replay"
}
~~~

一个很实用的规则：**外发路径、权限策略、工具 schema、记忆写入、状态机恢复** 这五类变更，不允许只跑 smoke test。哪怕 diff 很小，也必须进入 blocking replay。

## 4. 实战 Checklist

- **case metadata 必须结构化**：不要只靠文件名猜用途，明确 tags/risk/capability/changeScopes。
- **规则要匹配“行为边界”**：message formatter 也可能导致泄密，别只盯 send 函数。
- **missingTags 默认阻断高风险变更**：没有 case 比 case 失败更危险，因为它让系统误以为安全。
- **选择结果要归档**：记录 changeSet、matchedRules、selectedCases、missingTags、overrideReason。
- **保留 nightly full replay**：选择器服务日常发布，定期全量 replay 负责校准选择器漏网。
- **事故后先补规则再补 case**：如果 case 存在但没被选中，根因是 impact map 缺规则。

## 总结

Coverage Gate 回答：关键风险面有没有测试。

Selection Gate 回答：这次变更有没有跑到正确的测试。

成熟 Agent 的回归系统不是“全量跑到天荒地老”，也不是“凭感觉挑几个 case”。它应该能根据 diff 解释影响面，选择最小可信 replay set，并把选择过程本身变成审计证据。
