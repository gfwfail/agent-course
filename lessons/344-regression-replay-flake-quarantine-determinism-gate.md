# 344. Agent 回归回放抖动隔离与确定性闸门（Regression Replay Flake Quarantine & Determinism Gate）

上一课我们讲了 Regression Selection：根据本次 diff 选出最小但足够的回归包。

但选出来还不够。真实 Agent 回放最容易遇到一个烦人的问题：同一个 case 今天跑过，明天失败；本地跑过，CI 失败；重跑一次又好了。

这类 case 不能简单忽略，也不能直接拿来阻断所有发布。成熟做法是把 replay 结果拆成三类：

- stable_pass：多次回放一致通过，可以作为放行证据。
- stable_fail：多次回放一致失败，可以阻断发布。
- flaky_quarantined：结果不稳定，先隔离，不能作为放行证据；高风险变更需要人工复核或补一个稳定 case。

核心观点：**回归回放不是只看 pass/fail，而是先证明它是确定性的。**

## 1. 为什么 Agent replay 容易 flaky

普通单元测试 flaky，多半来自时间、网络、并发、随机数、外部状态。

Agent replay 还多几层：

- LLM 输出可能受模型版本、采样参数、上下文裁剪影响。
- 工具结果可能来自真实 API，数据会变。
- 工具调用顺序可能因为 steering message 或 follow-up message 改变。
- 证据、策略、权限、缓存都可能按时间过期。
- 回放 runner 本身可能没有固定 clock、env、workspace snapshot。

所以 replay gate 不能只写：

~~~python
if run_case(case).passed:
    allow_release()
~~~

应该先跑一个 Determinism Gate：

~~~python
result = run_with_determinism_gate(case, attempts=3)
if result.status == "stable_pass":
    allow_release()
elif result.status == "stable_fail":
    block_release(result.reason)
else:
    quarantine_case(result.case_id, result.fingerprint)
    require_manual_review_or_replacement_case()
~~~

## 2. learn-claude-code：给教学版工具回放加确定性指纹

learn-claude-code/agents/s02_tool_use.py 的关键结构很适合教学：

~~~python
TOOL_HANDLERS = {
    "bash":       lambda **kw: run_bash(kw["command"]),
    "read_file":  lambda **kw: run_read(kw["path"], kw.get("limit")),
    "write_file": lambda **kw: run_write(kw["path"], kw["content"]),
    "edit_file":  lambda **kw: run_edit(kw["path"], kw["old_text"], kw["new_text"]),
}
~~~

它把 LLM 的 tool_use 统一送进 dispatch map，再把 tool_result 写回 messages。

我们可以在这个点加 replay recorder。每个 tool call 都生成稳定指纹：

~~~python
import hashlib
import json
import time


VOLATILE_KEYS = {"duration_ms", "timestamp", "observed_at"}


def canonical_json(value):
    if isinstance(value, dict):
        return {
            k: canonical_json(v)
            for k, v in sorted(value.items())
            if k not in VOLATILE_KEYS
        }
    if isinstance(value, list):
        return [canonical_json(v) for v in value]
    return value


def fingerprint(tool_name, args, output):
    payload = {
        "tool": tool_name,
        "args": canonical_json(args),
        "output": canonical_json(output),
    }
    raw = json.dumps(payload, ensure_ascii=False, sort_keys=True, separators=(",", ":"))
    return hashlib.sha256(raw.encode("utf-8")).hexdigest()


def run_tool_with_trace(tool_name, args):
    started = time.monotonic()
    handler = TOOL_HANDLERS.get(tool_name)
    output = handler(**args) if handler else f"Unknown tool: {tool_name}"

    return {
        "tool": tool_name,
        "args": args,
        "output": output,
        "duration_ms": round((time.monotonic() - started) * 1000),
        "fingerprint": fingerprint(tool_name, args, output),
    }
~~~

然后 replay case 不比较整段日志，而比较“去掉 volatile 字段后的 fingerprint 序列”：

~~~python
def replay_once(case):
    traces = []
    for call in case["tool_calls"]:
        traces.append(run_tool_with_trace(call["name"], call["args"]))
    return [t["fingerprint"] for t in traces]


def classify_replay(case, attempts=3):
    runs = [replay_once(case) for _ in range(attempts)]
    unique = {tuple(run) for run in runs}

    if len(unique) == 1 and runs[0] == case["expected_fingerprints"]:
        return {"status": "stable_pass", "runs": runs}

    if len(unique) == 1:
        return {"status": "stable_fail", "runs": runs}

    return {
        "status": "flaky_quarantined",
        "runs": runs,
        "reason": "fingerprint sequence changed across attempts",
    }
~~~

这个教学版有两个重点：

1. 不要比较原始输出，先 canonicalize。
2. 不要一次通过就信，至少重复跑到能判断稳定性。

## 3. pi-mono：在 EventStream 和 AgentLoopConfig 上做生产版 gate

pi-mono/packages/agent/src/agent-loop.ts 里，Agent loop 会把关键生命周期事件推到 EventStream：

~~~ts
stream.push({ type: "agent_start" });
stream.push({ type: "turn_start" });
stream.push({ type: "message_start", message: prompt });
stream.push({ type: "message_end", message: prompt });
~~~

工具执行后也会形成 toolResults，再写回 currentContext.messages：

~~~ts
const toolExecution = await executeToolCalls(
  currentContext.tools,
  message,
  signal,
  stream,
  config.getSteeringMessages,
);

toolResults.push(...toolExecution.toolResults);
~~~

生产版不要在业务工具里散落 replay 逻辑，而是在 Agent loop 外面包一层 gate：

~~~ts
type ReplayOutcome =
  | { status: "stable_pass"; caseId: string; fingerprints: string[] }
  | { status: "stable_fail"; caseId: string; fingerprints: string[]; reason: string }
  | { status: "flaky_quarantined"; caseId: string; attempts: string[][]; reason: string };

type ReplayCase = {
  id: string;
  expectedFingerprints: string[];
  requiredForRisk: "low" | "medium" | "high";
  inputMessages: unknown[];
  frozenEnv: Record<string, string>;
  frozenClock: string;
};
~~~

关键是让 runner 每次回放都固定三件事：

- frozenClock：所有 Date.now() / 当前日期都由 replay clock 提供。
- frozenEnv：模型、工具版本、policy version、workspace snapshot 固定。
- tool cassettes：外部 API 走录制结果，不访问真实网络。

~~~ts
async function classifyReplay(
  replayCase: ReplayCase,
  runOnce: (c: ReplayCase) => Promise<string[]>,
  attempts = 3,
): Promise<ReplayOutcome> {
  const runs: string[][] = [];

  for (let i = 0; i < attempts; i++) {
    runs.push(await runOnce(replayCase));
  }

  const unique = new Set(runs.map((run) => JSON.stringify(run)));
  const expected = JSON.stringify(replayCase.expectedFingerprints);

  if (unique.size === 1 && JSON.stringify(runs[0]) === expected) {
    return { status: "stable_pass", caseId: replayCase.id, fingerprints: runs[0] };
  }

  if (unique.size === 1) {
    return {
      status: "stable_fail",
      caseId: replayCase.id,
      fingerprints: runs[0],
      reason: "stable output differs from expected fingerprints",
    };
  }

  return {
    status: "flaky_quarantined",
    caseId: replayCase.id,
    attempts: runs,
    reason: "replay fingerprints changed across attempts",
  };
}
~~~

发布 gate 的判断逻辑也要明确：

~~~ts
function decidePromotion(outcomes: ReplayOutcome[], risk: "low" | "medium" | "high") {
  const stableFailures = outcomes.filter((o) => o.status === "stable_fail");
  if (stableFailures.length > 0) {
    return { decision: "block", reason: "stable regression failure", stableFailures };
  }

  const flaky = outcomes.filter((o) => o.status === "flaky_quarantined");
  if (flaky.length === 0) {
    return { decision: "allow", reason: "all selected replay cases are deterministic" };
  }

  if (risk === "high") {
    return { decision: "manual_review", reason: "high-risk change has quarantined replay cases", flaky };
  }

  return { decision: "degrade", reason: "flaky replay cases cannot be used as release evidence", flaky };
}
~~~

注意：flaky_quarantined 不是绿灯。它只是避免 flaky case 误伤所有发布，同时也防止团队把“不稳定通过”当成安全证据。

## 4. OpenClaw 课程 cron 怎么落地

我们这个课程 cron 本身就是一个很好的例子。一次完整发布包含：

- 检查已讲主题，避免重复。
- 写 lesson 文件。
- 更新 README 目录。
- 更新 TOOLS.md 已讲内容。
- 发 Telegram。
- git commit/push。
- 写 memory 记录。

如果把它做成 replay case，可以把副作用前后的证据拆开：

~~~json
{
  "caseId": "agent-course-cron-publish",
  "risk": "external_side_effect",
  "expectedFingerprints": [
    "read-tools-md-topic-list",
    "create-lesson-file",
    "update-readme-index",
    "send-telegram-message",
    "git-commit-and-push",
    "write-memory-log"
  ],
  "volatileFields": ["messageId", "commitHash", "durationMs", "observedAt"]
}
~~~

这里 messageId 和 commitHash 每次都会变，不能直接比较；但“是否发到正确群组”“lesson path 是否匹配编号”“commit 是否包含三个文件变化”这些可以稳定验证。

## 5. 工程 checklist

做 Regression Replay Determinism Gate 时，最少要有这几个字段：

- caseId：稳定 ID，不能用时间戳。
- attempts：至少 2，建议高风险 3。
- fingerprintVersion：指纹算法升级要可追踪。
- volatileFields：明确哪些字段被忽略。
- frozenClock / frozenEnv：记录回放环境。
- quarantineReason：flaky case 为什么被隔离。
- replacementRequired：高风险路径是否必须补稳定 case。

最后一句话：

**回归测试失败不可怕，最危险的是一个不确定的通过。成熟 Agent 的 replay gate，先证明测试可信，再让测试证明发布可信。**
