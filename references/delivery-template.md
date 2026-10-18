# AI/Agent 技术方案模板

只保留影响决策的章节，首段先给选型结论。

## 1. 推荐

说明首选模式、优化的业务结果、为何匹配 Tool/Domain/Dependency/Risk、最大取舍，以及更简单备选和迁移条件。

## 2. 场景和假设

| 项目 | 决策相关内容 | 证据/假设 |
|---|---|---|
| User/Channel |  |  |
| Business Goal |  |  |
| Tool/Authority |  |  |
| Side Effect |  |  |
| Domain |  |  |
| Dependency |  |  |
| Latency/Throughput |  |  |
| Privacy/Compliance |  |  |
| Availability/Fallback |  |  |

列出会改变架构的未知项。

## 3. 策略对比

| 模式 | Accuracy | Latency | Scale/Churn | Security | Complexity | 结论 |
|---|---|---|---|---|---|---|
| Full Tool |  |  |  |  |  |  |
| Progressive Domain |  |  |  |  |  |  |
| Hybrid |  |  |  |  |  |  |
| Dynamic Retrieval |  |  |  |  |  |  |
| Deterministic Workflow |  |  |  |  |  |  |

解释第二名为什么现在不采用、什么变化会让它更适合。

## 4. 固定组件和职责

| 组件 | 负责 | 禁止 |
|---|---|---|
| API Gateway |  |  |
| Input Processor |  |  |
| Supervisor |  |  |
| Domain Entry |  |  |
| Domain Worker |  |  |
| Deterministic Gate |  |  |
| Global State |  |  |
| HumanTask |  |  |
| Outbox |  |  |

删除未使用组件；只有选型证明需要时才加入 Retriever。

### 能力边界

| Capability | 分类 | Owner | Accepts | Provides | Authority | Permission | Side Effect | SLO/Cost | Fallback | Recovery | Human | Test |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
|  |  |  |  |  |  |  |  |  |  |  |  |  |

ASR、TTS、Translation、Vision、Generation、3D、Payment、Notification 仅在场景要求时出现。

## 5. 端到端流程

~~~text
Channel → Gateway → Normalize → Supervisor Decide
                                → Domain Execute
                                → Review → Continue/Stop
                                → Policy → Human → Outbox
~~~

描述每次 Transition 的业务职责，不写框架代码。

## 6. Tool 披露

说明 Supervisor 能看到什么、Worker 能看到什么、本地 Tool 是全量还是检索、权限如何强制、Capability 如何映射 Domain、何时走确定性 Fast-path，以及为何不暴露更多/更少 Tool。

## 7. 依赖

至少展示：

- 一个固定硬依赖；
- 一个随 Goal 变化的 Optional 依赖；
- 一个 Tool 返回后才发现的依赖；
- 并行 Wave 和独立原因；
- Missing Capability、Clarification 和 Fallback。

## 8. JSON 契约

至少覆盖 DomainEntry、InputEnvelope、GoalSpec、SupervisorDecision、DomainTask、DomainResult/CapabilityRequest、Fact/Artifact、Review/State、FinalResponse、Approval/HumanTask、Recovery/Compensation/Reconciliation 和 Outbox。

说明 Producer、Consumer 和 Validation Invariant。

## 9. 场景推演

~~~text
1. 输入标准化
2. Supervisor 声明 Goal/Required Outputs
3. 派发当前 Wave
4. Worker 返回部分结果或 CapabilityRequest
5. Supervisor Review/Replan
6. 补齐依赖并 Resume
7. Completion Check
8. Policy/Human Gate
9. Outbox 幂等发送
~~~

另给一个失败/降级路径。

## 10. 性能与模型路由

包含各阶段 p50/p95、哪些路径不调用 LLM、模型大小/角色、Cache/Fact Reuse、Concurrency、Token/外部成本和何时才加 Retrieval。

## 11. 安全与韧性

覆盖 Least Privilege、Untrusted Tool Output、Tenant Isolation、Schema、Provenance、TTL、Idempotency、Approval Binding、Media Lifecycle、Retry 和 Reconciliation。

严格区分 Retry、Fallback、Degrade、Rollback、Compensate、Reconcile、Human 和 Fail Closed。每个关键能力声明最高降级等级。

## 12. 测试与指标

Unit、Contract、Graph、E2E、Failure Injection、Golden Set；高风险字段使用 Fail-closed 阈值。

## 13. 实施路线

建议：

1. Typed State + Mock Tool；
2. 单领域 Vertical Slice；
3. Supervisor Loop + Dynamic Dependency；
4. Side-effect Gate + HumanTask；
5. 场景确有需要时加入多模态/专项服务；
6. Shadow → Read-only → Limited Automation → Expanded Rollout。

每阶段包含 Acceptance 和 Rollback/Forward-fix。

## 14. 决策记录

列出选定架构、明确排除项、待验证假设、迁移触发和第一个实施 Milestone。
