# 518 - Agent 差分隐私跨数据集关联边界闸门

> Contribution Bounding 解决的是“单个用户在一个统计窗口里贡献太多”的问题。但 Agent 指标系统还有另一个坑：多个已经裁剪、加噪、脱敏的数据集，如果保留同一个稳定 user key，就可能被重新 join 成用户轨迹。成熟 Agent 做隐私发布时，不只要管单次发布，还要管跨数据集关联边界。

上一讲讲了 DP 发布前的贡献边界：先把 raw events 压成 user-level bounded aggregate，再进入噪声账本。今天继续往后走一步：**Cross-Dataset Linkage Boundary Gate**。

一句话：**不要让“每张表都安全”变成“几张表一 join 就不安全”。**

---

## 1. 为什么单表安全还会翻车？

假设我们发布了三份看起来都安全的统计：

- `tool_calls_daily`：每天各类工具的调用量；
- `error_buckets_weekly`：每周错误类型分布；
- `agent_feature_usage`：不同功能模块的使用情况。

每份数据都做了 contribution bounding，也加了 DP 噪声。但如果它们都带着同一个稳定字段：

```text
pseudo_user_id = HMAC(secret, user_id)
```

外部分析方就能把三份数据 join：

```sql
select *
from tool_calls_daily t
join error_buckets_weekly e using (pseudo_user_id)
join agent_feature_usage f using (pseudo_user_id);
```

这时风险不再来自单个数字，而来自组合后的稀有轨迹：

```text
周一用了 deploy_tool
周二触发 billing_error
周三只在 ap-southeast-1 生成过一次 report
```

单表看不出是谁，多表一连就可能只剩一个人。

所以 DP 发布链路需要一个额外闸门：

```text
DatasetReleaseReceipt
        ↓
LinkageBoundaryPolicy
        ↓
JoinRiskProbe
        ↓
KeyRotationPlan
        ↓
CrossDatasetLinkageReceipt
```

---

## 2. 闸门要回答的问题

Cross-Dataset Linkage Boundary Gate 主要检查五件事：

- 多个 release 是否共享稳定 join key；
- join 后最小群体是否低于 k-anonymity 阈值；
- 是否存在高维组合字段，比如 region + tool + day + error_code；
- 当前 DP 预算是否允许这个 join 目的；
- 是否需要按 dataset、purpose、time window 轮换伪匿名 key。

注意：这不是说永远不能 join。工程上常常需要跨数据集分析，但 join 必须有 scope、purpose、TTL、预算和 receipt。

---

## 3. learn-claude-code：关联风险纯函数

教学版先写一个可测试的判定器。它不接触真实用户数据，只看 release metadata 和 join probe 的摘要。

```python
from dataclasses import dataclass
from typing import Literal

Decision = Literal[
    "allow_join_with_scoped_key",
    "rotate_dataset_key",
    "coarsen_dimensions",
    "deny_linkage",
    "manual_review_high_risk",
]

@dataclass(frozen=True)
class DatasetRelease:
    dataset_id: str
    purpose: str
    key_scope: str
    epsilon_spent: float
    dimensions: tuple[str, ...]

@dataclass(frozen=True)
class JoinRiskProbe:
    min_join_group_size: int
    rare_tuple_count: int
    shared_stable_key: bool
    external_join_requested: bool

@dataclass(frozen=True)
class LinkagePolicy:
    min_group_size: int
    max_rare_tuples: int
    max_combined_epsilon: float
    allowed_purpose: str

@dataclass(frozen=True)
class LinkageDecision:
    decision: Decision
    reason: str
    required_key_scope: str | None = None

def decide_cross_dataset_linkage(
    releases: list[DatasetRelease],
    probe: JoinRiskProbe,
    policy: LinkagePolicy,
) -> LinkageDecision:
    purposes = {release.purpose for release in releases}
    if purposes != {policy.allowed_purpose}:
        return LinkageDecision(
            "deny_linkage",
            "release purposes do not match the approved join purpose",
        )

    combined_epsilon = sum(release.epsilon_spent for release in releases)
    if combined_epsilon > policy.max_combined_epsilon:
        return LinkageDecision(
            "manual_review_high_risk",
            "combined privacy budget exceeds linkage policy",
        )

    if probe.shared_stable_key and probe.external_join_requested:
        return LinkageDecision(
            "rotate_dataset_key",
            "external join cannot reuse a stable cross-dataset key",
            required_key_scope="dataset+purpose+window",
        )

    if probe.min_join_group_size < policy.min_group_size:
        return LinkageDecision(
            "coarsen_dimensions",
            "joined cohort is too small; reduce dimension granularity",
        )

    if probe.rare_tuple_count > policy.max_rare_tuples:
        return LinkageDecision(
            "manual_review_high_risk",
            "too many rare tuples after join",
        )

    return LinkageDecision(
        "allow_join_with_scoped_key",
        "join is within linkage boundary",
        required_key_scope="dataset+purpose+window",
    )
```

最小测试：

```python
def test_external_join_cannot_reuse_stable_key():
    decision = decide_cross_dataset_linkage(
        [
            DatasetRelease("tool_calls", "ops_report", "global_user", 0.2, ("day", "tool")),
            DatasetRelease("errors", "ops_report", "global_user", 0.3, ("week", "error")),
        ],
        JoinRiskProbe(
            min_join_group_size=50,
            rare_tuple_count=0,
            shared_stable_key=True,
            external_join_requested=True,
        ),
        LinkagePolicy(
            min_group_size=20,
            max_rare_tuples=5,
            max_combined_epsilon=1.0,
            allowed_purpose="ops_report",
        ),
    )
    assert decision.decision == "rotate_dataset_key"
    assert decision.required_key_scope == "dataset+purpose+window"

def test_small_join_group_requires_coarsening():
    decision = decide_cross_dataset_linkage(
        [DatasetRelease("tool_calls", "ops_report", "dataset+purpose+window", 0.2, ("hour", "tool", "region"))],
        JoinRiskProbe(
            min_join_group_size=3,
            rare_tuple_count=1,
            shared_stable_key=False,
            external_join_requested=False,
        ),
        LinkagePolicy(20, 5, 1.0, "ops_report"),
    )
    assert decision.decision == "coarsen_dimensions"
```

这里的重点不是把隐私数学全塞进 LLM，而是把可执行规则写成纯函数：scope 不符就 deny，稳定 key 要 rotate，小群体要 coarsen，预算超了要人工复核。

---

## 4. pi-mono：LinkageBoundaryWorker

生产版可以把它做成 DP release pipeline 的后置 worker。每次生成新数据集，worker 都检查它会不会和已有 release 形成危险 join。

```ts
type DatasetReleaseReceipt = {
  releaseId: string;
  datasetId: string;
  purpose: string;
  keyScope: "global_user" | "dataset" | "dataset+purpose+window";
  epsilonSpent: number;
  dimensions: string[];
  expiresAt: string;
};

type JoinRiskProbe = {
  minJoinGroupSize: number;
  rareTupleCount: number;
  sharedStableKey: boolean;
  externalJoinRequested: boolean;
};

type LinkageDecision =
  | "allow_join_with_scoped_key"
  | "rotate_dataset_key"
  | "coarsen_dimensions"
  | "deny_linkage"
  | "manual_review_high_risk";

type CrossDatasetLinkageReceipt = {
  receiptId: string;
  releaseIds: string[];
  decision: LinkageDecision;
  keyScope: string;
  combinedEpsilon: number;
  probeHash: string;
  createdAt: string;
};

class LinkageBoundaryWorker {
  constructor(
    private readonly store: LinkageStore,
    private readonly probe: JoinRiskProbeRunner,
    private readonly keyManager: ScopedKeyManager,
  ) {}

  async review(
    candidate: DatasetReleaseReceipt,
  ): Promise<CrossDatasetLinkageReceipt> {
    const related = await this.store.findActiveByPurpose(candidate.purpose);
    const releases = [...related, candidate];
    const probe = await this.probe.run(releases);
    const combinedEpsilon = releases.reduce(
      (sum, release) => sum + release.epsilonSpent,
      0,
    );

    const decision = decideLinkage({
      releases,
      probe,
      maxCombinedEpsilon: 1.0,
      minGroupSize: 20,
      maxRareTuples: 5,
    });

    return this.store.transaction(async (tx) => {
      if (decision === "rotate_dataset_key") {
        await this.keyManager.rotate({
          datasetId: candidate.datasetId,
          purpose: candidate.purpose,
          scope: "dataset+purpose+window",
        });
      }

      if (decision === "coarsen_dimensions") {
        await tx.enqueueCoarseningJob({
          releaseId: candidate.releaseId,
          removeDimensions: ["hour", "exact_region", "error_code"],
        });
      }

      return tx.appendLinkageReceipt({
        releaseIds: releases.map((release) => release.releaseId),
        decision,
        keyScope: "dataset+purpose+window",
        combinedEpsilon,
        probeHash: hashProbe(probe),
        createdAt: new Date().toISOString(),
      });
    });
  }
}
```

关键实现点：

- `findActiveByPurpose` 不能全库扫，按 purpose / consumer / retention class 建索引；
- `probe.run` 只输出摘要，不把 raw user rows 写进 receipt；
- key rotation 要先写计划，再切换 release pointer，避免旧 key 和新 key 同时暴露；
- `appendLinkageReceipt` 要幂等，key 可以是 `candidate.releaseId + relatedReleaseHash + probeHash`。

---

## 5. OpenClaw 实战类比

OpenClaw 的课程 cron 也有类似问题，只是对象不是隐私数据，而是跨系统证据：

- lesson 文件证明“内容写了”；
- README 证明“目录更新了”；
- TOOLS.md 证明“已讲清单更新了”；
- Telegram messageId 证明“群里发了”；
- git commit hash 证明“repo 发布了”。

这些证据单独看都没问题，但如果一个自动化后续 worker 随便 join 所有上下文，就可能把不该扩散的 owner/private context 也带出去。

所以 OpenClaw 类 Agent 要坚持三个边界：

1. **按目的取 key**：给 Telegram 发布、Git 提交、记忆更新分别用不同 scope；
2. **按收据 join**：只 join receipt id、hash、状态，不 join raw 对话；
3. **按渠道输出**：群聊只发课程内容，不带内部 workspace 细节。

隐私系统里叫 linkage boundary；Agent 工程里叫 context boundary。

---

## 6. 常见坑

### 坑 1：伪匿名 key 长期不换

`HMAC(secret, user_id)` 不是匿名化，只是可稳定关联的伪名。稳定 key 一旦跨数据集复用，就等于给攻击者发了一根 join 针。

更好的 key：

```text
HMAC(secret_vN, dataset_id + purpose + time_window + user_id)
```

并且 `secret_vN` 要有轮换计划和过期清理。

### 坑 2：只看单次 epsilon

单个 release 的 epsilon 很小，不代表几个 release 合起来仍然安全。Linkage gate 要算 combined epsilon，并检查目的是否一致。

### 坑 3：维度越细越有用，也越危险

`day + region` 可能安全，`hour + city + tool + error_code + model` 可能直接变成指纹。高维 join 要先跑 rare tuple probe。

### 坑 4：receipt 里保存 raw probe 样本

收据是为了证明，不是为了复制敏感数据。只保存 hash、摘要指标、policy version、decision 和必要证据引用。

---

## 7. 落地检查清单

- [ ] 每个数据集 release 是否声明 `purpose`、`consumer`、`retention` 和 `keyScope`？
- [ ] 是否禁止默认使用全局稳定 user key？
- [ ] 跨数据集 join 是否有 `CrossDatasetLinkageReceipt`？
- [ ] join 后是否检查 min group size 和 rare tuple count？
- [ ] combined epsilon 是否进入预算账本？
- [ ] key rotation 是否按 dataset + purpose + window 做隔离？
- [ ] receipt 是否只存摘要和 hash，不存 raw rows？
- [ ] 外部导出是否绑定 TTL、撤回路径和未登记 projection 清理？

---

## 8. 总结

Contribution Bounding 防止一个用户在单次统计里贡献过多；Cross-Dataset Linkage Boundary 防止多个安全统计被重新拼成个人轨迹。

成熟 Agent 的隐私发布链路应该长这样：

```text
raw events
  -> contribution bounding
  -> bounded aggregate
  -> DP noise ledger
  -> release receipt
  -> linkage boundary review
  -> scoped key / coarsened dimensions / deny
```

> 隐私泄漏经常不是发生在单张表，而是发生在“看起来都没问题”的几张表被重新拼起来以后。成熟 Agent 不只管发布什么数字，还管这些数字未来能不能被安全地放在一起看。
