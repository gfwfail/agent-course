# 286. Agent 事故回归测试与防复发验证（Incident Regression Tests & Prevention Verification）

> 事故复盘写得再漂亮，如果不能在下一次同类场景里自动挡住风险，本质上还是“记在纸上”。成熟 Agent 要把复盘结论变成回归测试、策略断言和上线闸门。

上一课我们讲了 Incident Review：把 rootCause、triggerSignals、failedGuardrails、newRules 结构化。今天继续往前走一步：**把事故复盘变成可执行的防复发验证**。

核心思想：

> 每一个严重事故都应该至少产出一个 Regression Case。以后 Agent 改代码、改 prompt、改工具策略、改 Runbook 时，都要能自动回放这个 case，证明同类事故不会复发。

---

## 1. 为什么“复盘”必须落到测试？

很多团队事故后会写：

- 增加发布前检查；
- 增加缓存版本校验；
- 遇到高风险操作必须人工确认；
- 以后注意 stale data。

问题是：这些规则如果只存在于文档里，Agent 不一定会执行。

Agent 系统里更可靠的做法是：

1. 从事故时间线提取 **输入、上下文、工具返回、错误决策**；
2. 写成可回放的 Regression Case；
3. 定义 expected outcome：应该 block、ask user、reroute、rollback，还是 require approval；
4. 每次发布前自动跑这些 case；
5. 如果旧事故能复现，发布直接失败。

一句话：

> Incident Review 负责解释过去，Regression Test 负责保护未来。

---

## 2. Regression Case 的最小结构

一个可执行事故回归用例至少包含六类信息：

```json
{
  "caseId": "reg-cache-schema-version-missing",
  "sourceIncidentId": "inc-cache-schema-version-missing",
  "risk": "high",
  "input": "把这份缓存里的部署状态用于发布判断",
  "observations": [
    { "tool": "cache.get", "result": { "schemaVersion": 1, "sourceVersion": "old" } },
    { "tool": "deploy.status", "result": { "schemaVersion": 2, "sourceVersion": "new" } }
  ],
  "expected": {
    "decision": "block",
    "requiredChecks": ["source_version_revalidation"],
    "reasonIncludes": ["schema_version_mismatch"]
  }
}
```

关键不是模拟所有细节，而是保留导致事故的**最小触发条件**。

---

## 3. learn-claude-code：最小事故回归测试器

教学版用 JSON 文件保存 regression cases，然后用一个 runner 调用决策函数。

```python
# learn_claude_code/incident_regression.py
from __future__ import annotations

import json
from dataclasses import dataclass
from pathlib import Path
from typing import Literal

Decision = Literal["allow", "block", "ask_user", "require_approval"]
Risk = Literal["low", "medium", "high"]


@dataclass
class RegressionCase:
    case_id: str
    source_incident_id: str
    risk: Risk
    input: str
    observations: list[dict]
    expected_decision: Decision
    expected_reason_includes: list[str]
    expected_required_checks: list[str]


@dataclass
class PolicyDecision:
    decision: Decision
    reasons: list[str]
    required_checks: list[str]


def decide_with_guardrails(case: RegressionCase) -> PolicyDecision:
    reasons: list[str] = []
    checks: list[str] = []

    cache_versions = [
        item["result"].get("schemaVersion")
        for item in case.observations
        if item.get("tool") == "cache.get"
    ]
    live_versions = [
        item["result"].get("schemaVersion")
        for item in case.observations
        if item.get("tool") == "deploy.status"
    ]

    if case.risk == "high":
        checks.append("source_version_revalidation")

    if cache_versions and live_versions and cache_versions[-1] != live_versions[-1]:
        reasons.append("schema_version_mismatch")
        return PolicyDecision("block", reasons, checks)

    return PolicyDecision("allow", reasons, checks)


def run_regression_case(case: RegressionCase) -> None:
    actual = decide_with_guardrails(case)

    assert actual.decision == case.expected_decision, (
        f"{case.case_id}: expected {case.expected_decision}, got {actual.decision}"
    )

    for reason in case.expected_reason_includes:
        assert reason in actual.reasons, f"{case.case_id}: missing reason {reason}"

    for check in case.expected_required_checks:
        assert check in actual.required_checks, f"{case.case_id}: missing check {check}"


def load_cases(path: str) -> list[RegressionCase]:
    raw = json.loads(Path(path).read_text(encoding="utf-8"))
    return [RegressionCase(**item) for item in raw]


if __name__ == "__main__":
    for case in load_cases("tests/incident-regressions.json"):
        run_regression_case(case)
    print("incident regression cases passed")
```

对应测试数据：

```json
[
  {
    "case_id": "reg-cache-schema-version-missing",
    "source_incident_id": "inc-cache-schema-version-missing",
    "risk": "high",
    "input": "用缓存状态判断是否可以发布",
    "observations": [
      { "tool": "cache.get", "result": { "schemaVersion": 1, "sourceVersion": "old" } },
      { "tool": "deploy.status", "result": { "schemaVersion": 2, "sourceVersion": "new" } }
    ],
    "expected_decision": "block",
    "expected_reason_includes": ["schema_version_mismatch"],
    "expected_required_checks": ["source_version_revalidation"]
  }
]
```

这个例子把“缓存 schema 版本不一致不能用于发布判断”变成了真正会失败的测试。

---

## 4. pi-mono：Incident Regression Harness

生产版里，不要只测试纯函数，还要测试完整 Agent 决策链：上下文注入、工具 mock、policy middleware、最终 action decision。

```ts
// pi-mono/packages/agent-runtime/src/incidents/incident-regression.ts
export type Decision = "allow" | "block" | "ask_user" | "require_approval";

export interface ToolObservation {
  tool: string;
  args?: unknown;
  result: unknown;
}

export interface IncidentRegressionCase {
  caseId: string;
  sourceIncidentId: string;
  risk: "low" | "medium" | "high";
  userInput: string;
  observations: ToolObservation[];
  expected: {
    decision: Decision;
    reasonIncludes: string[];
    requiredChecks: string[];
  };
}

export interface AgentRuntimeForTest {
  run(input: {
    userInput: string;
    risk: "low" | "medium" | "high";
    mockedObservations: ToolObservation[];
  }): Promise<{
    decision: Decision;
    reasons: string[];
    requiredChecks: string[];
  }>;
}

export class IncidentRegressionHarness {
  constructor(private readonly runtime: AgentRuntimeForTest) {}

  async runCase(testCase: IncidentRegressionCase): Promise<void> {
    const actual = await this.runtime.run({
      userInput: testCase.userInput,
      risk: testCase.risk,
      mockedObservations: testCase.observations,
    });

    if (actual.decision !== testCase.expected.decision) {
      throw new Error(
        `${testCase.caseId}: expected decision ${testCase.expected.decision}, got ${actual.decision}`,
      );
    }

    for (const reason of testCase.expected.reasonIncludes) {
      if (!actual.reasons.some((item) => item.includes(reason))) {
        throw new Error(`${testCase.caseId}: missing reason ${reason}`);
      }
    }

    for (const check of testCase.expected.requiredChecks) {
      if (!actual.requiredChecks.includes(check)) {
        throw new Error(`${testCase.caseId}: missing required check ${check}`);
      }
    }
  }

  async runAll(cases: IncidentRegressionCase[]): Promise<void> {
    for (const item of cases) {
      await this.runCase(item);
    }
  }
}
```

CI 里可以这样接：

```ts
import { loadIncidentRegressionCases } from "./case-loader";
import { buildTestRuntime } from "../testing/build-test-runtime";
import { IncidentRegressionHarness } from "./incident-regression";

test("incident regressions", async () => {
  const runtime = buildTestRuntime({
    toolsMode: "mocked",
    policyMode: "production",
    promptMode: "pinned",
  });

  const cases = await loadIncidentRegressionCases("tests/incidents/*.json");
  await new IncidentRegressionHarness(runtime).runAll(cases);
});
```

注意三个点：

- `toolsMode: "mocked"`：不能在回归测试里真的发消息、部署、删数据；
- `policyMode: "production"`：策略必须跑真实生产逻辑，否则测不到防线；
- `promptMode: "pinned"`：prompt hash 固定，避免测试因为临时 prompt 漂移而变得不可复现。

---

## 5. OpenClaw 实战：把课程 Cron 自己也纳入防复发

OpenClaw 这类 always-on Agent 很适合用事故回归保护 cron、heartbeat 和外部副作用。

以课程发布 cron 为例，历史上最容易出问题的地方是：

- 重复讲已经讲过的主题；
- 写了 lesson 但忘记更新 README；
- 发了群消息但没 commit/push；
- push 前没有检查 GitHub 账号或 PR 状态；
- TOOLS.md 已讲内容漏更新。

可以建立一个 `agent-course/tests/cron-regressions.json`：

```json
[
  {
    "caseId": "reg-course-no-duplicate-topic",
    "sourceIncidentId": "inc-course-duplicate-topic",
    "risk": "medium",
    "userInput": "Agent 开发课程时间",
    "observations": [
      { "tool": "read", "result": { "taughtTopics": ["Incident Review & Learning Loop"] } },
      { "tool": "planner", "result": { "nextTopic": "Incident Review & Learning Loop" } }
    ],
    "expected": {
      "decision": "block",
      "reasonIncludes": ["duplicate_topic"],
      "requiredChecks": ["tools_md_topic_check"]
    }
  }
]
```

再配一个 completion gate：

```bash
python scripts/run_incident_regressions.py \
  --cases tests/cron-regressions.json \
  --policy production \
  --tools mocked
```

这样“避免重复课程”就不是一句提醒，而是每次发课前都会跑的闸门。

---

## 6. 设计 Checklist

把事故变成回归测试时，建议检查这 7 项：

- [ ] 是否有 `sourceIncidentId` 能追溯到原事故？
- [ ] 是否只保留最小触发条件，而不是复制整段脏日志？
- [ ] 是否明确 expected decision：block / ask_user / approval / rollback？
- [ ] 是否验证 reason，而不是只验证布尔结果？
- [ ] 是否验证 requiredChecks，确保防线真的被触发？
- [ ] 是否 mock 外部工具，避免测试产生真实副作用？
- [ ] 是否接入 CI / release gate / cron completion gate？

---

## 7. 常见坑

**坑 1：只写复盘，不写 case**  
复盘会被人忘记，case 会在发布时失败。

**坑 2：只测最终回答文本**  
文本很容易变。更应该测结构化 decision、reason、requiredChecks。

**坑 3：回归测试用假的 policy**  
如果测试里绕过了生产 policy，等于只测了玩具系统。

**坑 4：case 太大**  
事故日志全量回放很脆弱。先提取最小失败条件，再保留原始证据包用于审计。

---

## 总结

今天这课的重点：

> 事故复盘不是结束，而是新测试的开始。

成熟 Agent 的防复发闭环应该是：

```text
Incident → Review → Regression Case → CI / Gate → Policy Patch → Verified Prevention
```

如果一个事故没有留下可执行测试，它大概率只是被“记住了”；  
如果它进入了 regression suite，它才真正被系统吸收了。
