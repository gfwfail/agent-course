# 390. Agent 补偿残余风险登记与关闭后监控租约（Compensation Residual Risk Register & Post-Closeout Monitoring Lease）

上一课讲了 Closeout Reconciliation Gate：补偿步骤全部 succeeded 还不够，必须证明 ledger、外部现实、证据链和下游屏障一致，才能 allow_close。

今天补一个更容易被忽略的生产细节：**allow_close 不等于风险归零。**

很多补偿动作只能把已知影响修到可接受状态，但仍可能留下残余风险：

- 第三方系统稍后才会结算或回调；
- 用户可能在补偿窗口外再次触发同类路径；
- 有些重复消息已经撤不回，只能靠后续监控确认没有扩散；
- 某些证据只能证明“当前一致”，不能证明“未来 24 小时不会反弹”。

一句话：**Closeout Receipt 证明事故可以关闭；Residual Risk Register 证明还有哪些风险被有意识地接受；Monitoring Lease 证明这些风险会被限时观察、到期复核，而不是靠 Agent 记在脑子里。**

## 1. 为什么 closeout 后还要有 lease

看课程 cron 的例子。

第 389 课完成后，closeout gate 可以证明：

- Telegram 消息已经发出；
- lesson 文件存在；
- README 指向 lesson；
- TOOLS 已讲内容已更新；
- git remote main 包含 commit；
- memory 记录和 messageId / commit 对得上。

这足够宣布“本次发布完成”。

但还可能有残余风险：

- Telegram 发送成功，但后续被平台限流、撤回或不可见；
- GitHub push 成功，但 CI / GitHub Pages / 镜像同步稍后失败；
- README 和 TOOLS 已更新，但下一轮选题去重可能读到旧缓存；
- memory 写入成功，但长期记忆维护还没有压缩归档。

这些不是当前 closeout 的阻塞项，否则 Agent 会永远不能收口。正确做法是把它们登记成 residual risk，并给每个风险一张 monitoring lease。

## 2. 最小模型：ResidualRisk + MonitoringLease

残余风险不要写成一句“后续关注”。它应该是结构化对象：

    {
      "riskId": "rr_agent_course_390_remote_visibility",
      "sourceCloseoutId": "comp_closeout_agent_course_390",
      "description": "Telegram message is sent, but remote channel visibility may lag.",
      "severity": "low",
      "acceptedBy": "system_policy",
      "acceptanceReason": "Message send receipt is present; lag is expected and recoverable.",
      "watchSignals": ["message_missing", "duplicate_message", "reply_reports_invisible"],
      "leaseId": "lease_agent_course_390_visibility_24h",
      "expiresAt": "2026-05-25T03:30:00+10:00",
      "onBreach": "reopen_reconcile",
      "onExpire": "archive_if_clean"
    }

核心字段：

- **riskId**：风险本身的稳定主键；
- **sourceCloseoutId**：来自哪个 closeout receipt；
- **severity**：决定是否允许 closeout 后观察；
- **acceptedBy / acceptanceReason**：说明为什么不是继续阻塞；
- **watchSignals**：后续监控看什么；
- **leaseId / expiresAt**：观察不是无限期；
- **onBreach**：观察期间出问题怎么处理；
- **onExpire**：到期没问题怎么关闭。

关键原则：

    allow_close != forget_risk
    residual_risk == accepted + bounded + monitored + expiring

## 3. 哪些风险可以 closeout 后观察

不是所有残余风险都能接受。可以用一张简单规则表：

    low + recoverable + observable        -> allow_close_with_lease
    medium + recoverable + observable     -> allow_close_with_required_lease
    high + recoverable + observable       -> manual_review_before_close
    any + irreversible + not_observable   -> hold_manual_review
    any + user_harm_unknown               -> hold_reconcile

判断时看三件事：

1. **可恢复性**：如果后续发现问题，还能不能补救？
2. **可观测性**：有没有信号能及时发现它真的发生？
3. **影响边界**：影响是否限定在已知用户、已知渠道、已知时间窗？

如果一个风险不可观测，就不能靠 lease 解决。那只是把未知风险包装成“稍后看看”，生产上很危险。

## 4. learn-claude-code：文件式 residual risk gate

教学版可以继续用 JSON 文件，不需要复杂服务。

    from dataclasses import dataclass, asdict
    from datetime import datetime, timedelta, timezone
    from pathlib import Path
    import json

    @dataclass
    class ResidualRisk:
        risk_id: str
        source_closeout_id: str
        severity: str
        recoverable: bool
        observable: bool
        description: str
        watch_signals: list[str]

    @dataclass
    class MonitoringLease:
        lease_id: str
        risk_id: str
        expires_at: str
        on_breach: str
        on_expire: str
        status: str = "active"

    def decide_risk(risk: ResidualRisk) -> str:
        if not risk.observable:
            return "hold_manual_review"
        if not risk.recoverable:
            return "hold_manual_review"
        if risk.severity == "high":
            return "manual_review_before_close"
        if risk.severity == "medium":
            return "allow_close_with_required_lease"
        return "allow_close_with_lease"

    def create_lease(risk: ResidualRisk, hours: int = 24) -> MonitoringLease:
        expires_at = datetime.now(timezone.utc) + timedelta(hours=hours)
        return MonitoringLease(
            lease_id=f"lease_{risk.risk_id}",
            risk_id=risk.risk_id,
            expires_at=expires_at.isoformat(),
            on_breach="reopen_reconcile",
            on_expire="archive_if_clean",
        )

    def append_jsonl(path: Path, row: dict) -> None:
        path.parent.mkdir(parents=True, exist_ok=True)
        with path.open("a", encoding="utf-8") as f:
            f.write(json.dumps(row, ensure_ascii=False) + "\n")

    risk = ResidualRisk(
        risk_id="rr_course_remote_visibility",
        source_closeout_id="comp_closeout_agent_course_390",
        severity="low",
        recoverable=True,
        observable=True,
        description="Remote channel visibility may lag after message send.",
        watch_signals=["message_missing", "duplicate_message"],
    )

    decision = decide_risk(risk)
    if decision.startswith("allow_close"):
        lease = create_lease(risk)
        append_jsonl(Path("state/residual-risks.jsonl"), asdict(risk))
        append_jsonl(Path("state/monitoring-leases.jsonl"), asdict(lease))

这里的重点不是代码复杂，而是职责清楚：

- closeout gate 负责判断本次补偿能不能关闭；
- residual risk gate 负责判断哪些风险可以带 lease 接受；
- lease worker 负责后续观察、到期关闭或 breach 重开。

## 5. pi-mono：把 lease 接到事件流

生产版不要把“明天再看看”写在聊天里。它应该是事件系统的一等对象。

    type RiskSeverity = "low" | "medium" | "high";

    type ResidualRisk = {
      riskId: string;
      sourceCloseoutId: string;
      severity: RiskSeverity;
      recoverable: boolean;
      observable: boolean;
      description: string;
      watchSignals: string[];
      acceptedBy: "policy" | "human" | "runbook";
      acceptanceReason: string;
    };

    type MonitoringLease = {
      leaseId: string;
      riskId: string;
      expiresAt: string;
      status: "active" | "breached" | "expired_clean" | "expired_review";
      onBreach: "reopen_reconcile" | "escalate_incident" | "manual_review";
      onExpire: "archive_if_clean" | "manual_review";
    };

    class ResidualRiskGate {
      evaluate(risk: ResidualRisk) {
        if (!risk.observable || !risk.recoverable) {
          return { decision: "hold_manual_review" as const };
        }

        if (risk.severity === "high") {
          return { decision: "manual_review_before_close" as const };
        }

        return {
          decision: "allow_close_with_lease" as const,
          leaseTtlHours: risk.severity === "medium" ? 48 : 24,
        };
      }
    }

    class MonitoringLeaseWorker {
      constructor(
        private leases: MonitoringLeaseStore,
        private signalBus: RiskSignalBus,
        private reconciler: CompensationReconciler,
      ) {}

      async tick(now: Date) {
        const active = await this.leases.activeDueBefore(now);

        for (const lease of active) {
          const signals = await this.signalBus.collect(lease.riskId);

          if (signals.some((signal) => signal.status === "breach")) {
            await this.leases.markBreached(lease.leaseId, signals);
            await this.reconciler.reopenFromRisk(lease.riskId, signals);
            continue;
          }

          await this.leases.expireClean(lease.leaseId);
        }
      }
    }

这个 worker 可以挂在 OpenClaw periodic event 或 pi-mono 的后台队列上。关键是：lease 到期必须有终态，不能永远 active。

## 6. OpenClaw 课程 cron 的实际清单

课程发布类任务可以在 closeout receipt 后附一个短 lease 清单：

    residualRisks:
      - riskId: rr_lesson_390_telegram_visibility
        severity: low
        watchSignals:
          - telegram_message_missing
          - duplicate_lesson_report
        leaseTtl: 24h
        onBreach: reopen_reconcile

      - riskId: rr_lesson_390_git_remote_sync
        severity: low
        watchSignals:
          - origin_main_missing_commit
          - readme_link_broken
        leaseTtl: 24h
        onBreach: reopen_reconcile

      - riskId: rr_lesson_390_topic_cache
        severity: low
        watchSignals:
          - next_lesson_duplicates_390
        leaseTtl: 48h
        onBreach: manual_review

这比“我会留意”可靠得多。因为下一次 cron、heartbeat 或 worker 可以直接读取 lease 文件，知道该检查什么、什么时候过期、发现问题应该重开哪条对账流程。

## 7. 常见坑

第一，把 residual risk 当成免责说明。

风险登记不是“我已经提醒过了所以没事”。它必须绑定信号、租约和处理动作。没有 watchSignals 的 risk register 没有操作价值。

第二，lease 没有过期时间。

无限期观察等于没有 owner。lease 必须到期，到期要么 clean archive，要么 expired_review，要么 breach reopen。

第三，接受不可观测风险。

如果后续没有信号能证明风险是否发生，就不要 closeout 后观察。应该 hold_manual_review 或继续补证据。

第四，所有风险都用同一个 24h。

不同风险的自然窗口不同：消息可见性可能几分钟，支付结算可能 24-72 小时，缓存传播可能几个小时。TTL 要跟现实系统匹配。

第五，breach 只发告警不重开流程。

告警只是提示。真正的 onBreach 应该回到 reconcile、incident 或 compensation runner，让系统进入可处理状态。

## 8. 一句话总结

> Closeout 让 Agent 有资格说“这次补偿完成了”；Residual Risk Register 让 Agent 诚实记录“还有哪些可接受但未归零的风险”；Monitoring Lease 让这些风险在有限时间内被观察、到期关闭或出事重开。

成熟 Agent 的恢复闭环，不是把事故关掉就忘，而是把残余风险变成有期限、有信号、有动作的工程对象。
