# Agent 长期回归套件失败聚类与根因优先级

> Regression Suite Failure Clustering & Root-Cause Priority Gate

第 485 课讲了再激活 seed 如何重新接回长期 blocking lane。接回以后，另一个现实问题会马上出现：长期套件跑久了，失败不会一个一个排队出现，而是一批一批涌上来。

如果每个失败 seed 都开一个 ticket，团队会被重复告警淹没；如果只看第一个失败，又可能漏掉真正的系统性根因。所以第 486 课补上 **Regression Suite Failure Clustering & Root-Cause Priority Gate**：把长期回归套件的失败先聚类，再按影响范围、风险面、阻塞等级和修复收益排序。

一句话：**失败不是越多越可怕，真正可怕的是不知道哪些失败其实是同一个根因。**

## 核心模型

把长期套件失败处置拆成 5 个对象：

~~~text
LongTermSuiteRunSummary
  -> SeedFailureSignal
  -> FailureCluster
  -> RootCausePriorityReview
  -> ClusterTriageReceipt
~~~

分工：

- LongTermSuiteRunSummary：一次长期套件运行的总体结果；
- SeedFailureSignal：单个 seed 的失败指纹，包括错误类型、工具、断言、运行时版本；
- FailureCluster：把相似失败合并成一个聚类；
- RootCausePriorityReview：给每个 cluster 排优先级；
- ClusterTriageReceipt：记录最后动作，是 open_root_cause_case、attach_existing_case、quarantine_cluster、rerun_cluster 还是 manual_review。

关键点是：**先聚类，再排优先级，最后才派工。** 不然 Agent 的测试系统会把一个根因变成几十个噪音。

## learn-claude-code：失败聚类纯函数

教学版可以先写成纯函数。输入一批 seed failure，输出聚类和优先级动作。

~~~python
# learn_claude_code/regression_failure_clustering.py
from dataclasses import dataclass
from typing import Literal

RiskSurface = Literal["tool_call", "memory", "side_effect", "release_gate", "ui"]
TriageAction = Literal[
    "open_root_cause_case",
    "attach_existing_case",
    "quarantine_cluster",
    "rerun_cluster",
    "manual_review",
]


@dataclass(frozen=True)
class SeedFailureSignal:
    seed_id: str
    suite_lane: Literal["nightly", "release_blocking", "post_deploy_sentinel"]
    risk_surface: RiskSurface
    assertion_id: str
    tool_name: str | None
    error_class: str
    runtime_fingerprint: str
    first_failed_at_epoch_ms: int
    flake_score: float
    blocks_release: bool


@dataclass
class FailureCluster:
    cluster_key: str
    signals: list[SeedFailureSignal]
    blocks_release_count: int
    risk_surfaces: set[RiskSurface]
    runtime_fingerprints: set[str]
    max_flake_score: float


def cluster_key(signal: SeedFailureSignal) -> str:
    tool = signal.tool_name or "no_tool"
    return "|".join([
        signal.risk_surface,
        signal.assertion_id,
        tool,
        signal.error_class,
    ])


def cluster_failures(signals: list[SeedFailureSignal]) -> list[FailureCluster]:
    buckets: dict[str, list[SeedFailureSignal]] = {}
    for signal in signals:
        buckets.setdefault(cluster_key(signal), []).append(signal)

    clusters: list[FailureCluster] = []
    for key, bucket in buckets.items():
        clusters.append(FailureCluster(
            cluster_key=key,
            signals=bucket,
            blocks_release_count=sum(1 for s in bucket if s.blocks_release),
            risk_surfaces={s.risk_surface for s in bucket},
            runtime_fingerprints={s.runtime_fingerprint for s in bucket},
            max_flake_score=max(s.flake_score for s in bucket),
        ))
    return clusters


def decide_cluster_action(cluster: FailureCluster) -> TriageAction:
    if len(cluster.runtime_fingerprints) > 3:
        return "manual_review"

    if cluster.max_flake_score > 0.75 and cluster.blocks_release_count == 0:
        return "rerun_cluster"

    if "side_effect" in cluster.risk_surfaces:
        return "quarantine_cluster"

    if cluster.blocks_release_count > 0:
        return "open_root_cause_case"

    if len(cluster.signals) >= 3:
        return "open_root_cause_case"

    return "attach_existing_case"
~~~

这个函数故意很朴素，但它抓住了三个生产经验：

- 聚类 key 不只看 error message，还要看 risk surface、assertion、tool 和 error class；
- flake 高且不阻塞 release 的失败，优先重跑，不要马上制造事故单；
- side effect 相关失败先隔离，因为它可能已经影响真实外部状态。

## pi-mono：Failure Clustering Worker

生产版不要让每个 runner 自己开 issue，而是把失败信号投递到统一 worker。worker 负责聚类、去重、派工和写 triage receipt。

~~~ts
// packages/agent-runtime/src/regression/FailureClusteringWorker.ts
export type RiskSurface = "tool_call" | "memory" | "side_effect" | "release_gate" | "ui";
export type TriageAction =
  | "open_root_cause_case"
  | "attach_existing_case"
  | "quarantine_cluster"
  | "rerun_cluster"
  | "manual_review";

export interface SeedFailureSignal {
  seedId: string;
  suiteLane: "nightly" | "release_blocking" | "post_deploy_sentinel";
  riskSurface: RiskSurface;
  assertionId: string;
  toolName?: string;
  errorClass: string;
  runtimeFingerprint: string;
  firstFailedAtEpochMs: number;
  flakeScore: number;
  blocksRelease: boolean;
}

export interface FailureCluster {
  clusterKey: string;
  signals: SeedFailureSignal[];
  blocksReleaseCount: number;
  riskSurfaces: Set<RiskSurface>;
  runtimeFingerprints: Set<string>;
  maxFlakeScore: number;
}

export interface ClusterTriageReceipt {
  receiptId: string;
  clusterKey: string;
  action: TriageAction;
  signalCount: number;
  reason: string;
  createdAt: string;
}

export class FailureClusteringWorker {
  constructor(private readonly store: RegressionFailureStore) {}

  key(signal: SeedFailureSignal): string {
    return [
      signal.riskSurface,
      signal.assertionId,
      signal.toolName ?? "no_tool",
      signal.errorClass,
    ].join("|");
  }

  cluster(signals: SeedFailureSignal[]): FailureCluster[] {
    const buckets = new Map<string, SeedFailureSignal[]>();
    for (const signal of signals) {
      const key = this.key(signal);
      buckets.set(key, [...(buckets.get(key) ?? []), signal]);
    }

    return [...buckets.entries()].map(([clusterKey, bucket]) => ({
      clusterKey,
      signals: bucket,
      blocksReleaseCount: bucket.filter((s) => s.blocksRelease).length,
      riskSurfaces: new Set(bucket.map((s) => s.riskSurface)),
      runtimeFingerprints: new Set(bucket.map((s) => s.runtimeFingerprint)),
      maxFlakeScore: Math.max(...bucket.map((s) => s.flakeScore)),
    }));
  }

  decide(cluster: FailureCluster): TriageAction {
    if (cluster.runtimeFingerprints.size > 3) return "manual_review";
    if (cluster.maxFlakeScore > 0.75 && cluster.blocksReleaseCount === 0) {
      return "rerun_cluster";
    }
    if (cluster.riskSurfaces.has("side_effect")) return "quarantine_cluster";
    if (cluster.blocksReleaseCount > 0) return "open_root_cause_case";
    if (cluster.signals.length >= 3) return "open_root_cause_case";
    return "attach_existing_case";
  }

  async triage(runId: string): Promise<ClusterTriageReceipt[]> {
    const signals = await this.store.listFailureSignals(runId);
    const clusters = this.cluster(signals);

    return this.store.transaction(async (tx) => {
      const receipts: ClusterTriageReceipt[] = [];
      for (const cluster of clusters) {
        const action = this.decide(cluster);

        if (action === "open_root_cause_case") {
          await tx.openRootCauseCase(cluster.clusterKey, cluster.signals);
        }
        if (action === "quarantine_cluster") {
          await tx.quarantineSeeds(cluster.signals.map((s) => s.seedId));
        }
        if (action === "rerun_cluster") {
          await tx.enqueueClusterRerun(cluster.clusterKey, cluster.signals);
        }

        receipts.push(await tx.writeClusterTriageReceipt({
          receiptId: crypto.randomUUID(),
          clusterKey: cluster.clusterKey,
          action,
          signalCount: cluster.signals.length,
          reason: `cluster triage decision: ${action}`,
          createdAt: new Date().toISOString(),
        }));
      }
      return receipts;
    });
  }
}
~~~

生产实现里要注意两点：

- `openRootCauseCase`、`quarantineSeeds`、`enqueueClusterRerun` 和 `writeClusterTriageReceipt` 要在同一个事务或同一个幂等批次里；
- `clusterKey` 要稳定，否则同一个根因每次运行都会生成新 case，去重会失效。

## OpenClaw：课程 Cron 的类比

这个课程 cron 也有类似场景：

1. 如果 Telegram 发群失败、git push 失败、README 更新失败同时出现，不能开三个互不相关的问题；
2. 要先看它们是不是同一个根因，比如网络、GitHub auth、message tool 权限或仓库脏状态；
3. 如果只是 README 漏更新，可以局部补偿；
4. 如果是发群成功但 git push 失败，就必须保留 side-effect receipt，避免下一轮重复发同一课；
5. 如果是 GitHub auth 失效，则优先修 auth，因为它会阻塞后续所有课程 cron。

也就是说，OpenClaw cron 的失败处置也应该先聚类：**同源问题合并处理，真实副作用优先对账，阻塞后续自动化的根因优先修。**

## 实战清单

长期回归套件失败聚类时，至少检查：

- cluster key 是否包含 risk surface、assertion、tool、error class；
- 是否把高 flake、低风险失败先 rerun，而不是马上开事故；
- side effect 失败是否先隔离并进入补偿队列；
- release_blocking 失败是否优先于 nightly 噪音；
- runtime fingerprint 过多时是否进入 manual review，避免误判跨版本漂移；
- 每个 cluster 是否只开一个 root cause case；
- triage receipt 是否记录 action、signalCount、reason 和时间；
- 后续修复是否按 cluster 关闭，而不是逐个 seed 手工打勾。

## 小结

第 485 课解决 seed 怎么重新回到长期套件，第 486 课解决长期套件失败后怎么不被噪音淹没。

成熟 Agent 的回归系统不是“失败一个修一个”，而是把失败变成结构化信号：先聚类，识别根因，再按风险和收益派工。这样测试套件越大，系统越清楚，而不是越吵。
