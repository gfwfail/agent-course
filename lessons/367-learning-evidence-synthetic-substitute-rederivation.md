# 367. Agent 学习证据合成替身与规则再派生（Learning Evidence Synthetic Substitute & Rule Re-derivation）

上一课讲了 Tombstone Propagation：raw evidence 删除后，下游 Learning Rule、Prompt Pack、Policy Cache、Regression Pack 都要收到影响传播，不能继续偷偷依赖已删除证据。

今天补下一步：**证据删了以后，里面学出来的经验是不是一定要丢？**

答案是不一定。隐私、权限或保留期要求删除 raw evidence，但有些经验本身仍然有价值。正确做法不是继续引用旧证据，而是用允许保留的 claim / proof hash / audit metadata 生成一个合成替身，再重新评估、重新派生一条新规则。

一句话：**Tombstone 负责切断旧依赖，Synthetic Substitute 负责在不恢复 raw data 的前提下重建可用经验。**

## 1. 为什么需要合成替身

假设某次事故里，用户原始聊天记录包含隐私信息。Agent 从中学到一条经验：

    高风险 git push 前必须检查当前 PR 是否已经 merged，避免往已合并分支继续推送。

后来用户要求删除那段 raw evidence。我们不能继续让这条 Learning Rule 引用原始聊天记录，但这条经验明显仍然有工程价值。

这时要做三步：

1. tombstone 旧 evidence，隔离依赖它的旧规则；
2. 从允许保留的 claim 摘要生成 synthetic evidence；
3. 用 synthetic evidence 重新跑质量闸门、冲突闸门、反事实评估，再发布新规则版本。

注意：合成替身不是“把原文换个说法藏起来”。它必须只包含可保留的抽象行为约束，不包含原始 payload、账号、私聊内容、密钥、订单号等敏感事实。

## 2. Synthetic Evidence 的最小结构

生产里建议把合成证据做成独立 evidence 类型：

    {
      "evidenceId": "syn_ev_01HY...",
      "type": "synthetic_learning_evidence",
      "sourceTombstoneId": "ts_01HY...",
      "sourceEvidenceId": "ev_deleted_123",
      "claims": [
        {
          "claimId": "claim_check_pr_before_push",
          "text": "Before pushing to a long-lived branch, verify whether the associated PR is still open.",
          "scope": "git:push",
          "riskSurface": "external_side_effect"
        }
      ],
      "forbiddenResiduals": ["raw_user_text", "secret", "personal_identifier"],
      "createdBy": "rederivation_job",
      "requiresReplay": true
    }

关键字段是 sourceTombstoneId。它证明这条 synthetic evidence 是从已删除证据的允许残留中再派生出来的，而不是偷偷恢复 raw evidence。

## 3. 旧规则不能直接换 evidence id

一个常见偷懒做法是把旧规则里的 evidenceRefs 从 ev_deleted_123 改成 syn_ev_01HY，然后继续 active。

这不安全。因为证据来源变了，规则的可信度、适用范围、冲突关系都可能变。正确做法是创建新版本：

    old rule lr_42 v3:
      status = quarantined
      evidenceRefs = [ev_deleted_123]
      quarantineReason = evidence_tombstoned

    new rule lr_42 v4:
      status = shadow
      evidenceRefs = [syn_ev_01HY]
      parentVersion = v3
      changeReason = rederived_from_synthetic_evidence
      requiredChecks = [quality_gate, conflict_gate, counterfactual_replay]

这条新规则要重新从 shadow / canary 走发布流程。不能因为旧规则曾经 active，就默认新规则也能 active。

## 4. learn-claude-code：教学版再派生 Gate

learn-claude-code 的教学版可以用纯函数表达核心流程：输入 tombstone、允许保留的 claims、旧规则，输出 synthetic evidence 和新规则候选。

    from dataclasses import dataclass
    from typing import Literal
    import hashlib

    @dataclass
    class Tombstone:
        tombstone_id: str
        source_evidence_id: str
        allowed_claims: list[str]
        forbidden_residuals: list[str]

    @dataclass
    class LearningRule:
        rule_id: str
        version: int
        status: Literal["active", "shadow", "quarantined", "retired"]
        text: str
        evidence_refs: list[str]

    @dataclass
    class SyntheticEvidence:
        evidence_id: str
        source_tombstone_id: str
        claims: list[str]
        proof_hash: str

    def build_synthetic_evidence(tombstone: Tombstone) -> SyntheticEvidence:
        normalized = "\n".join(sorted(tombstone.allowed_claims))
        proof_hash = hashlib.sha256(normalized.encode()).hexdigest()
        return SyntheticEvidence(
            evidence_id="syn_" + proof_hash[:16],
            source_tombstone_id=tombstone.tombstone_id,
            claims=tombstone.allowed_claims,
            proof_hash="sha256:" + proof_hash,
        )

    def rederive_rule(old_rule: LearningRule, synthetic: SyntheticEvidence) -> LearningRule:
        if old_rule.status != "quarantined":
            raise ValueError("only quarantined rules should be rederived")
        return LearningRule(
            rule_id=old_rule.rule_id,
            version=old_rule.version + 1,
            status="shadow",
            text=old_rule.text,
            evidence_refs=[synthetic.evidence_id],
        )

这里故意让新规则进入 shadow，而不是 active。因为 synthetic evidence 只能证明“抽象经验仍然可表达”，不能证明“这条规则对生产仍然安全”。后者必须靠 replay / canary。

## 5. pi-mono：LearningRederivationMiddleware

在 pi-mono 这种生产实现里，再派生适合放在 TombstonePropagation 之后：

1. EvidenceRetentionWorker 写 tombstone；
2. TombstonePropagationMiddleware quarantine 旧规则；
3. LearningRederivationMiddleware 读取 allowed claims；
4. 创建 synthetic evidence；
5. 创建 shadow rule version；
6. LearningReleasePipeline 重新跑评估和 canary。

    type RuleStatus = "active" | "shadow" | "quarantined" | "retired";

    interface EvidenceTombstone {
      tombstoneId: string;
      evidenceId: string;
      allowedClaims: Array<{ claimId: string; text: string; scope: string }>;
      forbiddenResiduals: string[];
    }

    interface LearningRule {
      id: string;
      version: number;
      status: RuleStatus;
      text: string;
      evidenceRefs: string[];
    }

    async function rederiveFromTombstone(tombstone, ruleStore, evidenceStore, releaseQueue) {
      const quarantinedRules = await ruleStore.findQuarantinedByEvidenceId(tombstone.evidenceId);
      if (tombstone.allowedClaims.length === 0) return;

      const synthetic = await evidenceStore.createSynthetic({
        sourceTombstoneId: tombstone.tombstoneId,
        sourceEvidenceId: tombstone.evidenceId,
        claims: tombstone.allowedClaims,
        forbiddenResiduals: tombstone.forbiddenResiduals,
      });

      for (const oldRule of quarantinedRules) {
        const candidate = {
          ...oldRule,
          version: oldRule.version + 1,
          status: "shadow",
          evidenceRefs: [synthetic.evidenceId],
          parentVersion: oldRule.version,
          changeReason: "rederived_from_synthetic_evidence",
        };

        await ruleStore.saveCandidate(candidate);
        await releaseQueue.enqueue({
          ruleId: candidate.id,
          version: candidate.version,
          requiredChecks: ["quality_gate", "conflict_gate", "counterfactual_replay", "canary"],
        });
      }
    }

生产关键点：synthetic evidence 只能从 allowedClaims 创建，不能重新打开 raw evidence。所有再派生都必须带 sourceTombstoneId，方便审计。

## 6. OpenClaw 实战：课程记忆也能这样做

OpenClaw 课程 Cron 很适合类比这个流程。

如果某一课引用了不该长期保留的原始上下文，处理方式不应该是“把整课删掉，以后别讲这个经验”。更好的方式是：

    1. 在 memory/YYYY-MM-DD.md 记录 tombstone：哪条原始 evidence 不再可用；
    2. 从课程里保留抽象 claim：例如“外发前要做降密扫描”；
    3. 新建 synthetic lesson evidence：只保存 claim、hash、sourceTombstoneId；
    4. README / TOOLS 更新为新主题或 superseded 主题；
    5. 后续课程引用 synthetic evidence，而不是旧 raw context。

这就是文件系统版的 Learning Rederivation。它让 Agent 能安全“保留教训”，但不继续携带不该保留的原始材料。

## 7. 常见坑

第一，把 synthetic evidence 写得太细。只要它能反推出原始用户、订单、密钥、聊天内容，就不是合格的合成替身。

第二，跳过 replay。证据换了，规则就要重新验证，尤其是会影响副作用工具时。

第三，把 sourceTombstoneId 丢了。没有 tombstone 链接，审计无法证明这条经验不是从 raw evidence 偷偷恢复的。

第四，旧规则直接恢复 active。再派生应该产生新版本，至少从 shadow 开始。

第五，所有经验都尝试保留。有些证据删除后没有足够 allowed claims，就应该让规则 retired，而不是强行编一个替身。

## 8. 生产落地清单

- Tombstone 明确 allowedClaims 和 forbiddenResiduals；
- Synthetic Evidence 只从 allowedClaims 生成；
- 新 evidence 绑定 sourceTombstoneId 和 sourceEvidenceId；
- 旧规则保持 quarantined / retired，不原地复活；
- 新规则创建新 version，状态从 shadow 开始；
- 必跑 quality gate、conflict gate、counterfactual replay；
- 高风险规则必须 canary 后才能 active；
- prompt 注入层只接收通过发布流程的 synthetic-derived rule；
- 审计能追踪 old evidence -> tombstone -> synthetic evidence -> new rule version。

## 9. 关键 takeaway

成熟 Agent 的学习系统，不是 raw evidence 一删就忘光，也不是偷偷保留原始数据。

**Learning Evidence Synthetic Substitute 的目标是：删除原始材料，保留可验证、可审计、可重新发布的抽象经验。**

这样 Agent 才能同时满足隐私删除、长期学习、生产安全和行为可回放。
