# Agent 架构选择指南

选择能满足准确率、延迟、成本、安全和运维要求的最简单模式。Tool 数量只是参考，Schema 长度、领域相似度、模型能力和风险更重要。

## 1. 场景画像

| 维度 | 问题 |
|---|---|
| 目标 | 要产出决定、Artifact、Transaction 还是回复？ |
| 模态 | Text、Voice、Image、Video、Document、3D 中哪些适用？ |
| Tool | 总数和每领域数量？ |
| Domain | 能否按稳定业务所有权和词汇分组？ |
| Churn | Tool 添加、删除、改名和版本变化多快？ |
| Dependency | 固定、可选、数据相关还是执行后发现？ |
| Side Effect | 哪些动作花钱、发消息、改数据或接触隐私？ |
| Latency | p50/p95 是否允许额外模型/检索往返？ |
| Accuracy | 允许合理近似，还是关键字段必须 100% 有来源？ |
| Context | Tool Schema 和 Fact 能否舒适放入上下文？ |
| Audit | Decision、Fact、Approval、Retry 是否必须重放？ |
| Availability | 模型、API、Media、DB 不可用时怎么办？ |
| Recovery | Retry、Rollback、Compensate、Reconcile 还是 Human？ |
| Ownership | 是否由不同领域团队和权限维护？ |

## 2. 模式对比

| 模式 | 适合 | 优点 | 成本 |
|---|---|---|---|
| 1. 全量 Tool 暴露 | 小、同质、稳定 Tool 集 | 最低路由开销 | 扩大后 Tool 混淆和上下文膨胀 |
| 2. 渐进式领域披露 | 多个稳定领域，本地 Tool 适中 | 隔离上下文和权限 | 多一次 Supervisor/Worker 往返 |
| 3. 混合模式 | 简单/复杂路径并存、企业风险 | 平衡快路径、灵活性和硬门禁 | 设计和测试更复杂 |
| 4. 动态 Tool Retrieval | 数百/数千 Tool、高变化 | 不暴露全量目录 | Recall、Ranking、Latency、观测风险 |
| 5. 确定性 Workflow | 稳定受监管或不可逆流程 | 可预测、可测试 | 对未知目标适应弱 |
| 6. ReAct/Code Agent | 开放式调查/工程 | 能根据观察适应 | 延迟、成本、终止更难控制 |

## 3. 模式一：全量 Tool

~~~text
User → One Agent + All Tool Schemas → Tool Calls → Final
~~~

适合大约 5–20 个简洁、同领域、只读或外部有硬门禁的 Tool，并且多数请求只调用 1–2 次。

避免用于：

- 不同领域重名/近义 Tool；
- 数百 Schema；
- 权限和隐私范围差异大；
- 跨域可选依赖复杂；
- 错误选择会触发高风险副作用。

控制：清晰命名、严格 JSON Schema、读写 Tool 分离、授权/幂等/数字校验外置，并持续测 Tool Selection Accuracy。

## 4. 模式二：渐进式领域披露

~~~text
Supervisor 只见少量 Domain Entry
  → 选择一个或多个 Domain
  → Worker 只见本领域 Tool
  → Result / needed_capabilities 返回 Supervisor
~~~

适合领域可稳定划分、全局目录过大但本地目录可控、权限按领域隔离、跨域依赖可选或运行时出现的场景。

Domain Entry 是静态小型 Tool Schema，只声明 Accepts/Provides，不是运行时具体 API Registry。

逻辑 Worker 不等于一次独立 LLM：明确操作可走确定性 Router，只有本领域歧义/多步任务才调用模型。

## 5. 模式三：混合

~~~text
Input
  → Deterministic Fast-path
      明确安全请求 → Direct Adapter/Workflow
      歧义/多领域 → Supervisor Loop
                      → Progressive Domain
                      → Local Tool 或 Local Retrieval
                      → Deterministic Gate
~~~

典型组合：

- 高频稳定 Intent 走确定性快路径；
- 歧义、多域、恢复进入 Supervisor；
- 领域间渐进披露；
- 小领域本地全量 Tool；
- 只有单领域 Tool 过多才做本地 Retrieval；
- 不可逆动作使用 Approval/Outbox；
- 开放探索只在有边界子任务内用 ReAct。

主要风险是 Router 层叠。必须冻结术语、所有权和唯一 Global State。

## 6. 模式四：动态 Retrieval

~~~text
Goal / Required Capability
  → Permission/Tenant/Region/Version Hard Filter
  → BM25/Embedding Top-k
  → Rerank + LLM Selection
  → Validate Arguments → Execute
~~~

仅在数百/数千 Tool、动态安装、高变化、领域划分弱且上下文无法容纳时使用。

必须：

- 先做权限、租户、地区、副作用、版本硬过滤；
- 记录 Candidate、Score、Selection 和 Rejection；
- 提供 No-result/Recovery Query；
- 分开评估 Recall@k 和最终 Selection；
- 不可逆 Tool 默认不进入自动检索。

优先在大领域内部检索，而不是全局检索。

## 7. 模式五：确定性 Workflow + 有边界 Agent

~~~text
Fixed State Machine/DAG
  → Agent Extraction/Draft Node
  → Deterministic Validation
  → Explicit Next State
~~~

适合顺序稳定、受监管、事务不可逆且每个 Transition 必须穷举测试的场景。硬策略依赖写死，软语义依赖不要强制成无条件边。

## 8. 模式六：ReAct/Code Agent

适合路径未知、需反复观察，Tool 包含 Search、Filesystem、Code Execution 或 Diagnostic 的场景。

必须具备 Goal/Completion Test、Time/Turn/Token/Cost Budget、Sandbox/Permission、Action/Result Log、重复动作检测、Checkpoint 和 Human Escalation。

## 9. 决策树

~~~text
步骤/副作用是否依法或运营上固定？
  是 → Deterministic Workflow + Bounded Agent Node
  否 → Tool 是否小且同质？
       是 → Full Exposure
       否 → 是否可稳定分领域？
            是 → Progressive Domain
                 高频简单/高风险硬动作并存？是 → Hybrid
            否 → Tool 是否巨大或动态安装？
                 是 → Dynamic Retrieval
                 否 → 改善领域分类或使用 Bounded ReAct
~~~

## 10. 轻量评分

按 0–3 评分，仅用于组织讨论：

| 因素 | 低 | 高时偏向 |
|---|---|---|
| Tool 数 | 小 | Retrieval/Progressive |
| Domain 可分性 | 弱 | Progressive |
| Churn | 静态 | Retrieval |
| Dependency 不确定性 | 固定 | Supervisor/ReAct |
| Side Effect Risk | 只读 | Deterministic/Hybrid |
| Latency Pressure | 宽松 | Full/Fast-path |
| Audit | 低 | Typed State + Deterministic |
| Openness | 脚本化 | ReAct |

始终说明首选方案的成本、第二名未选原因和迁移触发条件。

## 11. 常见组合

- 小型 SaaS：Full Read Tool + Deterministic Write Wrapper；
- 多领域企业助手：Supervisor + Progressive Domain + Policy Gate；
- 大型 Plugin 平台：Fast-path + Domain + Local Retrieval + Approval/Outbox；
- 受监管审批：Deterministic Workflow + Extraction/Draft Agent + Human；
- 工程 Agent：Bounded ReAct + Sandbox + Checkpoint + Permission。

## 12. 决策记录

记录选定模式和范围、两个合理备选、场景证据、延迟/准确/成本/安全/维护取舍、迁移条件及明确不包含的组件。
