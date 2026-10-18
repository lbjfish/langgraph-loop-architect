# Agent 架构评测、生产验收与上线

架构质量由可观察行为证明，不由 Agent 数量或图的复杂度证明。

## 1. 要求—证据矩阵

| 字段 | 含义 |
|---|---|
| Requirement | 精确业务要求 |
| Risk | 失败后果 |
| Owner | Supervisor、Domain、Deterministic Service、Human |
| Contract | 能证明行为的 JSON 对象/字段 |
| Test | Unit、Contract、Graph、E2E、Eval |
| Metric | 阈值 |
| Fallback | 未满足时的安全行为 |

## 2. 测试层

### Unit

- Amount/Date/TTL/Dimension/Quota；
- Capability/Status Enum；
- Authority/Freshness Conflict；
- Idempotency Key；
- Policy/Permission Gate；
- Loop Budget/Repeat Detection；
- Error Classification。

### Contract

- 所有 JSON 通过 Schema；
- Domain Entry 只声明支持能力；
- Worker 只返回声明输出；
- Tool Adapter 正确映射成功/错误；
- Optional Input 语义有测试；
- Reference 不跨 Tenant；
- 未知字段按策略拒绝或版本化。

### Graph

- 单领域 Fast-path 只启动一个 Worker；
- 独立任务同波次；
- 部分结果后发现动态依赖；
- State 已有能力时不重复调用；
- Supervisor 选择正确 Domain；
- 原 Worker 获得引用后 Resume；
- 后续失败不丢 verified partial facts；
- required 输出未满足时继续 Loop；
- Stale Fact 刷新；
- Conflict 按 Authority 或 Clarification；
- Rejected Result 不进入 verified；
- 最大 Cycle、Time、Token/Cost、Repeat Protection；
- Checkpoint 不重复模型、Tool、生成费用或 Side Effect。

### E2E

按适用性覆盖 Text/Voice/Image/Mixed、Language/Locale、Privacy/Media Lifecycle、Provider Outage、Local Fallback、Human Pause/Resume、Payload Hash、Restart、Concurrent Conversation 和 Tenant Isolation。

not_applicable 模态不得创建无意义测试。

### Live Smoke

只有明确授权时使用 Sandbox/Non-production Tenant，限制成本和 Tool Call，不向真实客户发送；记录 Provider、Model、Parameter、Latency、Cost 和 Result ID。计费/副作用未知的 Timeout 不得盲目重试。

## 3. Golden Set

覆盖：

- Clear/Ambiguous Intent；
- Exact/Misspelled Entity；
- 单域/多域；
- Fixed/Optional/Runtime Dependency；
- Fresh/Stale/Missing/Conflicting Fact；
- Read-only/Irreversible Side Effect；
- Low/High Confidence；
- Normal/Adversarial/Prompt-injected Tool Content；
- Normal/Timeout/429/Partial Failure；
- Approval Approve/Reject/Expire/Modified Candidate。

正确行为可以是澄清、拒绝或返回部分结果。

## 4. 模式指标

### Full Tool

Tool Selection、Wrong Domain/Argument、Schema Token、随 Tool 数增长的 p50/p95。

### Progressive Domain

Domain Selection、Local Tool Selection、Unnecessary Dispatch、Dependency Recovery、Supervisor Cycle、相对 Flat Baseline 的 Latency。

### Retrieval

Recall@k、Permission Filter、Rerank、No-result Recovery、Retrieval/Total Latency、Stale Metadata。

### Deterministic Workflow

Transition Coverage、Invalid Transition、Policy Escape、Recovery/Replay。

### ReAct

Task Success、Action Efficiency、Repeat Action、Turn/Token/Cost、Budget Termination、Unsafe Attempt。

## 5. 业务与可靠性指标

Task Completion、Human Acceptance、Critical Fact Accuracy、Unsupported Claim、Clarification/Handoff、p50/p95/p99、Cost、Tool Error/Retry、Checkpoint Recovery、Duplicate Side Effect、Stale Fact Send、Policy/Privacy Incident，以及业务 Conversion/Resolution。

高风险字段使用 Fail-closed，不使用平均分掩盖错误。

## 6. 可观测性

记录 Trace/Run/Conversation/Task/Decision/Checkpoint、Graph/Schema Version、Pattern/Domain/Wave、Tool 与脱敏参数摘要、Authority/TTL、Model/Provider/Token/Cost、Fact/Artifact/Hash、Supervisor Action/Reason、Policy、Approval 和 Outbound Receipt。

禁止记录密钥、Authorization Header、不必要个人数据、原始隐私媒体和私有模型推理。

## 7. 性能

~~~text
Gateway + Normalize + Supervisor Decision + Domain Work
+ Tool + Queue + Generation + Validation + Human + Send
~~~

用相同场景比较 Flat Baseline、选定架构、Deterministic Fast-path 和 Degraded Mode。准确率提高但简单请求 p95 成倍增加时，应增加快路径。

优化：

- Cache 稳定 Domain Description/Schema；
- 复用有效 Fact；
- 并发独立 Read；
- 明确操作走确定性 Router；
- Supervisor 只看已验证摘要和引用；
- 抽取/分类用小模型；
- Retrieval 限制在已选 Domain；
- Fact/Permission 未通过前不生成昂贵媒体。

## 8. 安全

威胁：User/Web/OCR/Tool Prompt Injection、Argument Smuggling、Cross-tenant、Overprivileged Worker、Unsafe Retrieved Tool、Duplicate/Replay、Generated Artifact 冒充事实、Approval 后候选变化、Stale Policy/Price、敏感 State 进入外部观测平台。

按风险使用 Least Privilege、Allowlist、Hard Filter、Schema、Provenance、Hash、Idempotency、TTL、Redaction 和 HumanTask。

## 9. 故障注入

- Malformed JSON；
- Tool 返回指令型文本；
- Domain Fact 冲突；
- Optional Input 在不同 Goal 下变成必需；
- Side Effect 前后 Timeout；
- Expired Approval；
- Old Checkpoint；
- Duplicate Event；
- 无 Domain 能提供 Capability；
- Supervisor 重复相同 Decision。

### 恢复与人工

- Side Effect/计费未知时禁止 Retry；
- Equivalent Fallback 重新校验同一契约；
- Degrade 披露降低的保证；
- Internal Rollback 不得宣称外部事件已撤销；
- Unknown Effect 进入 Reconciliation；
- Compensation 有独立 Authorization、Idempotency、Receipt；
- Partial Fact 可用而 required 输出仍 incomplete；
- HumanTask 绑定 Fact/Artifact/Policy/Candidate Hash；
- Fact/Artifact 改变会使 Approval 失效；
- Approve/Reject/Revise/Timeout/Escalate/Cancel/Resume 可用；
- 配置双人审批时必须是不同有权限人员；
- Fail-closed 不产生 Claim 或 Side Effect。

### 能力边界

对每个 required/optional 能力测试 Input/Output、Authority、Permission、Side-effect、TTL、SLO、Cost、Availability State、每条 Fallback、最高 Degrade Level 和 Human Trigger。not_applicable 能力不得产生节点、调用、测试或成本。

## 10. 上线

1. Offline Mock + Contract/Graph Eval；
2. Shadow；
3. Read-only Draft + 全量 Human Send；
4. Limited Low-risk Automation；
5. 达到 Error/Incident/Latency/Business Gate 后扩大。

始终保留 Kill Switch、Audit 和 Human Escalation。

## 11. 验收

- 选定架构确实优于更简单方案；
- 能力已分为 required/optional/deferred/not_applicable；
- 组件单一所有权和禁止职责清楚；
- JSON 和状态迁移可验证；
- Optional Dependency 未被写死；
- Dynamic Dependency Recovery 已证明；
- Fact/Claim/Artifact/Approval/SideEffect 分离；
- Loop/Cost/Retry/Time Limit 明确；
- 高风险 Fail Closed；
- Checkpoint/Idempotency 可验证；
- 降级仍真实可用；
- Retry/Fallback/Degrade/Rollback/Compensate/Reconcile/Human 不混用；
- 人工流程包含 Trigger、Role、SLA、Expiry、Binding、Revalidate、Resume 和 Audit；
- 当前实现不冒充目标架构。
