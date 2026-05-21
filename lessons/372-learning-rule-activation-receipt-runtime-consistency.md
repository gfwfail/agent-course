# 372. Agent 学习规则生效收据与运行时一致性检查（Learning Rule Activation Receipt & Runtime Consistency Check）

上一课讲了 Promotion Decision Record：证据充分性闸门给出 promote / hold / demote / manual_review 后，不能直接改状态，而要先写不可变的晋级决策记录，并把人工复核接入结构化队列。

今天继续补发布链路里最容易被忽略的一步：**Promotion Decision 已经 applied，不代表学习规则真的在运行时生效。**

很多系统在这里会出现“状态看起来对，行为实际没变”的问题：

- rule store 显示 rule 已经 enforce，但 prompt pack 仍然用旧版本；
- policy cache 刷新失败，工具决策还在按旧规则 allow；
- retrieval index 延迟更新，Agent 根本检索不到新规则；
- 多实例部署中只有一个 worker 刷新了 cache，其他实例还在旧世界；
- 发布成功消息已经发出，但下一次真实 run 没有任何 evidence 能证明新规则参与了决策。

一句话：**Promotion Decision 证明系统决定让规则生效，Activation Receipt 证明规则已经进入运行路径，Runtime Consistency Check 证明所有关键入口看到的是同一版规则。**

## 1. 为什么 applied 不是 activated

看一个常见状态：

    {
      "decisionId": "pdr_lr_check_pr_before_push_v6",
      "ruleId": "lr_check_pr_before_push",
      "approvedTier": "enforce",
      "executionState": "applied",
      "appliedAt": "2026-05-21T08:30:00Z"
    }

这个 applied 只说明 promotion worker 把 rule store 写完了。它还没证明：

- prompt pack 包含 ruleId + ruleVersion；
- policy cache 的 compiledRuleHash 是新 hash；
- retrieval index 已经可查；
- regression pack 使用的是新规则依赖；
- 所有 runtime shard / worker 都观察到新 manifest；
- 第一批真实决策里记录了 activation evidence。

所以需要单独的 Activation Receipt：

    {
      "activationId": "act_20260521_lr_check_pr_before_push_v6",
      "decisionId": "pdr_lr_check_pr_before_push_v6",
      "ruleId": "lr_check_pr_before_push",
      "ruleVersion": 6,
      "targetTier": "enforce",
      "activatedArtifacts": {
        "ruleStore": "sha256:rule_store_91a",
        "promptPack": "sha256:prompt_pack_812",
        "policyCache": "sha256:policy_cache_a10",
        "retrievalIndex": "sha256:retrieval_index_b44"
      },
      "runtimeChecks": [
        {"component": "tool_policy", "observedVersion": 6, "status": "ok"},
        {"component": "prompt_injection", "observedVersion": 6, "status": "ok"}
      ],
      "state": "activated",
      "createdAt": "2026-05-21T08:30:08Z"
    }

Promotion Decision 是发布决定；Activation Receipt 是发布落地证明。

## 2. Activation Receipt 的最小字段

一个实用的 Activation Receipt 至少包含这些字段：

    {
      "activationId": "act_xxx",
      "decisionId": "pdr_xxx",
      "ruleId": "lr_xxx",
      "ruleVersion": 6,
      "targetTier": "enforce",
      "artifactHashes": {
        "ruleStore": "sha256:...",
        "promptPack": "sha256:...",
        "policyCache": "sha256:...",
        "retrievalIndex": "sha256:..."
      },
      "requiredComponents": ["promptPack", "policyCache", "retrievalIndex"],
      "componentResults": [
        {
          "component": "policyCache",
          "expectedVersion": 6,
          "observedVersion": 6,
          "observedHash": "sha256:...",
          "status": "ok"
        }
      ],
      "runtimeSampleRefs": ["ev_runtime_decision_1"],
      "state": "activated",
      "createdAt": "2026-05-21T08:30:08Z"
    }

这里最重要的是两类证据：

- **Artifact hash**：证明发布产物内容是什么。
- **Runtime observation**：证明运行入口真的看到了这些产物。

没有 runtime observation 的发布，只是写入成功，不是生效成功。

## 3. 一致性检查不是健康检查

健康检查只会告诉你服务活着：

    GET /health -> 200 OK

但学习规则发布后，你需要问更具体的问题：

    GET /runtime/learning-rules/lr_check_pr_before_push
    -> observedVersion: 6
    -> compiledHash: sha256:policy_cache_a10
    -> promptPackHash: sha256:prompt_pack_812
    -> retrievalIndexGeneration: 44

Runtime Consistency Check 要比较：

- expected ruleVersion；
- expected artifact hashes；
- expected tier；
- expected dependency lock；
- observed component generation；
- observed decision path 是否引用了新 rule。

如果有一个关键组件仍然看旧版本，高风险规则就不能算 activated，只能进入 partial_activation 或 failed_activation。

## 4. learn-claude-code：最小文件式生效收据

教学版可以把 artifact manifest 和 runtime snapshot 都放在本地 JSON，写一个纯函数做一致性检查：

    from dataclasses import dataclass, asdict
    from datetime import datetime, timezone
    from pathlib import Path
    import json

    @dataclass(frozen=True)
    class ComponentObservation:
        component: str
        expected_version: int
        observed_version: int
        observed_hash: str
        expected_hash: str
        status: str

    @dataclass(frozen=True)
    class ActivationReceipt:
        activation_id: str
        decision_id: str
        rule_id: str
        rule_version: int
        target_tier: str
        component_results: list[ComponentObservation]
        state: str
        created_at: str

    def observe_component(component: str, expected: dict, runtime: dict) -> ComponentObservation:
        observed = runtime.get(component, {})
        ok = (
            observed.get("rule_version") == expected["rule_version"]
            and observed.get("hash") == expected["hash"]
        )

        return ComponentObservation(
            component=component,
            expected_version=expected["rule_version"],
            observed_version=observed.get("rule_version", -1),
            expected_hash=expected["hash"],
            observed_hash=observed.get("hash", "missing"),
            status="ok" if ok else "mismatch",
        )

    def build_activation_receipt(decision: dict, manifest: dict, runtime: dict) -> ActivationReceipt:
        components = ["ruleStore", "promptPack", "policyCache", "retrievalIndex"]
        results = [
            observe_component(name, manifest[name], runtime)
            for name in components
        ]

        state = "activated" if all(r.status == "ok" for r in results) else "partial_activation"

        return ActivationReceipt(
            activation_id=f"act_{decision['ruleId']}_v{decision['ruleVersion']}",
            decision_id=decision["decisionId"],
            rule_id=decision["ruleId"],
            rule_version=decision["ruleVersion"],
            target_tier=decision["approvedTier"],
            component_results=results,
            state=state,
            created_at=datetime.now(timezone.utc).isoformat(),
        )

    def append_receipt(path: Path, receipt: ActivationReceipt) -> None:
        path.parent.mkdir(parents=True, exist_ok=True)
        payload = asdict(receipt)
        payload["component_results"] = [asdict(r) for r in receipt.component_results]
        with path.open("a", encoding="utf-8") as f:
            f.write(json.dumps(payload, ensure_ascii=False, sort_keys=True) + "\n")

这个例子里的关键不是 JSON 文件，而是边界清楚：

- decision 决定要发布；
- manifest 记录应该发布成什么；
- runtime snapshot 证明实际看到什么；
- receipt 把两者比对结果写成不可变证据。

## 5. pi-mono：把 activation 放到事件流里

在 pi-mono 这类事件驱动 Agent runtime 里，Promotion Worker applied 之后应该继续发 activation 事件，而不是直接结束。

伪代码：

    type ActivationState = "activated" | "partial_activation" | "failed_activation";

    interface RuntimeComponentSnapshot {
      component: "ruleStore" | "promptPack" | "policyCache" | "retrievalIndex";
      expectedVersion: number;
      observedVersion: number;
      expectedHash: string;
      observedHash: string;
      generation: number;
      status: "ok" | "mismatch" | "missing";
    }

    interface ActivationReceipt {
      activationId: string;
      decisionId: string;
      ruleId: string;
      ruleVersion: number;
      targetTier: string;
      components: RuntimeComponentSnapshot[];
      runtimeSampleRefs: string[];
      state: ActivationState;
      createdAt: string;
    }

    async function activateLearningRule(ctx, decision) {
      const manifest = await ctx.learningRules.compileActivationManifest(decision);
      await ctx.promptPacks.refresh(manifest.promptPack);
      await ctx.policyCache.refresh(manifest.policyCache);
      await ctx.retrievalIndex.upsert(manifest.retrievalIndex);

      const snapshots = await Promise.all([
        ctx.ruleStore.observe(decision.ruleId),
        ctx.promptPacks.observe(decision.ruleId),
        ctx.policyCache.observe(decision.ruleId),
        ctx.retrievalIndex.observe(decision.ruleId),
      ]);

      const components = snapshots.map(snapshot => ({
        component: snapshot.component,
        expectedVersion: decision.ruleVersion,
        observedVersion: snapshot.ruleVersion ?? -1,
        expectedHash: manifest.hashes[snapshot.component],
        observedHash: snapshot.hash ?? "missing",
        generation: snapshot.generation ?? 0,
        status:
          snapshot.ruleVersion === decision.ruleVersion &&
          snapshot.hash === manifest.hashes[snapshot.component]
            ? "ok"
            : snapshot.hash
              ? "mismatch"
              : "missing",
      }));

      const state = components.every(c => c.status === "ok")
        ? "activated"
        : "partial_activation";

      const receipt = await ctx.activationReceipts.append({
        activationId: ctx.ids.next("act"),
        decisionId: decision.decisionId,
        ruleId: decision.ruleId,
        ruleVersion: decision.ruleVersion,
        targetTier: decision.approvedTier,
        components,
        runtimeSampleRefs: [],
        state,
        createdAt: ctx.clock.now(),
      });

      await ctx.events.publish({
        type: "learning_rule.activation_recorded",
        decisionId: decision.decisionId,
        activationId: receipt.activationId,
        state,
      });

      return receipt;
    }

这里的设计要点：activation 是独立阶段，有自己的 receipt、有自己的状态、有自己的失败处理。

## 6. OpenClaw：课程 Cron 里的实战映射

这套课程 Cron 本身就是一个小型 activation pipeline：

- TOOLS.md 已讲内容：rule store；
- lessons/XXX.md：知识产物；
- README.md：检索目录；
- Telegram messageId：外部发布证据；
- git commit hash：不可变版本；
- memory/YYYY-MM-DD.md：运行时审计日志。

如果只写 lesson 文件，但 README 没更新，未来检索就可能漏掉。
如果 Telegram 发了，但 git push 失败，群里看到的课程就没有仓库证据。
如果 TOOLS.md 没更新，下一次 cron 可能重复选题。

所以每次课的“生效收据”其实可以长这样：

    {
      "lessonNo": 372,
      "lessonFile": "lessons/372-learning-rule-activation-receipt-runtime-consistency.md",
      "readmeEntry": true,
      "toolsTopicUpdated": true,
      "telegramMessageId": 123xx,
      "gitCommit": "abc123",
      "remoteContainsCommit": true,
      "state": "activated"
    }

这不是形式主义。长期自动化最怕“我以为发过了”。Activation Receipt 把“发过了”变成可复查证据。

## 7. 失败处理：partial activation 要降级

当 activation 失败时，不要假装成功：

    {
      "state": "partial_activation",
      "failedComponents": ["policyCache"],
      "safeAction": "demote_to_shadow",
      "reason": "runtime_policy_cache_still_on_v5"
    }

建议策略：

- promptPack mismatch：暂停注入新规则，继续旧版本；
- policyCache mismatch：禁止把规则用于高风险工具决策；
- retrievalIndex missing：允许 direct lookup，但不宣称全量 activated；
- 多实例版本不一致：进入 canary_only 或 hold；
- runtime sample 没有引用新 rule：延迟宣布 activated，补一次只读 probe。

发布系统最重要的不是永远成功，而是失败时知道自己没成功。

## 8. 最小测试用例

可以从四个 case 开始：

    def test_activation_success_when_all_components_match():
        receipt = build_activation_receipt(decision, manifest, runtime_all_v6)
        assert receipt.state == "activated"

    def test_policy_cache_mismatch_is_partial_activation():
        receipt = build_activation_receipt(decision, manifest, runtime_policy_cache_v5)
        assert receipt.state == "partial_activation"
        assert any(r.component == "policyCache" and r.status == "mismatch" for r in receipt.component_results)

    def test_missing_retrieval_index_is_not_activated():
        receipt = build_activation_receipt(decision, manifest, runtime_without_index)
        assert receipt.state == "partial_activation"

    def test_receipt_keeps_decision_id_for_audit_join():
        receipt = build_activation_receipt(decision, manifest, runtime_all_v6)
        assert receipt.decision_id == decision["decisionId"]

测试重点不是覆盖所有分支，而是锁住一个原则：**没有组件级 runtime observation，就不能把发布状态标成 activated。**

## 9. 工程落地清单

- [ ] Promotion Decision applied 后生成 Activation Manifest；
- [ ] 刷新 prompt pack / policy cache / retrieval index；
- [ ] 从运行时入口读取 observed version/hash；
- [ ] 写 append-only Activation Receipt；
- [ ] partial_activation 时降级到 shadow / hold / canary；
- [ ] 第一批真实决策记录 ruleId/ruleVersion evidence；
- [ ] README / memory / external message / git commit 都进入发布收据；
- [ ] 远端 commit 和外部 messageId 可交叉验证。

## 10. 今日 takeaway

学习规则发布不是“状态字段从 shadow 改成 enforce”。

成熟的发布链路应该分三段：

1. **Promotion Decision**：我们决定让哪条规则晋级；
2. **Activation Receipt**：我们证明规则已经进入运行路径；
3. **Runtime Consistency Check**：我们持续确认各组件看到的是同一版规则。

没有 Activation Receipt 的学习系统，很容易出现“数据库说已生效，Agent 行为却没变”的幽灵状态。真正可靠的 Agent，会把每一次学习发布都变成可追踪、可回放、可降级的运行事实。
