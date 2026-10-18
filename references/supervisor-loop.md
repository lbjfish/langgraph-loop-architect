# Supervisor Loop 与渐进式领域编排

在策略选择确认需要 Supervisor、渐进式领域披露或混合架构后使用。这里以 LangGraph 风格的 Supervisor/Worker 为主，同时保持对其他 Agent 框架和自研 Typed State 的可迁移性。

传统 CRUD、固定 BPMN、普通数据流水线和单次模型调用不应套用该模式。

## 1. 固定术语与职责

| 组件 | 负责 | 不负责 |
|---|---|---|
| Channel Adapter | IM/Web/Voice/Webhook 协议 | 业务规划 |
| API Gateway | Auth、Tenant、Quota、Size、Idempotency Envelope | Intent/Tool 选择 |
| Input Processor | ASR、Language ID、媒体净化、标准化 | 业务结论 |
| Supervisor | Goal、Domain 选择、Wave、Review、Replan、Stop | 直接调用具体业务 API |
| Domain Entry | 领域级 Accepts/Provides 描述 | 具体 API 清单 |
| Domain Worker | 本领域规划和 Tool Call | 跨领域调用和最终发送 |
| Tool Contract | 本地 API 语义、Schema、限制、错误、示例 | 全局编排策略 |
| Global State | Fact、Task、Artifact、Decision、Budget、Checkpoint | 密钥或大二进制 |
| Deterministic Gate | Permission、Amount、TTL、Provenance、Side Effect | 开放式规划 |
| HumanTask | 高风险可追责判断 | 常规低风险路由 |
| Outbox/Sender | 事务化、幂等发送 | 内容生成 |

Critic、Retriever、Planner 等可选组件必须证明必要，不能与 Supervisor 重复职责。

## 2. Domain Entry 与具体 Tool

Supervisor 只看到：

~~~text
catalog_domain
  accepts: ProductMention, SKU, Locale
  provides: ProductProfile, ProductDimensions, VariantAvailability
~~~

Catalog Worker 才看到：

~~~text
catalog.search_product
catalog.get_product
catalog.get_dimensions
catalog.get_variants
~~~

Supervisor 下达“为该商品产出 ProductDimensions”，不直接指定某个 API。

## 3. 核心 Loop

~~~text
InputEnvelope
  → Normalize
  → Supervisor Decide
      ├─ Finalize
      ├─ Clarify
      ├─ Safe Fallback / Human
      └─ Dispatch/Resume Domain
             → Worker 调用本地 Tool
             → DomainResult / CapabilityRequest
             → Schema + Deterministic Validation
             → Supervisor Review
                  ├─ Conflict/Reject → Replan
                  ├─ Missing → Dispatch Domain
                  ├─ Incomplete → Continue
                  └─ Complete → Policy Gate → Human/Outbox
~~~

LangGraph 提供 State、Node、Conditional Edge、Subgraph、Checkpoint、Interrupt 和 Resume；业务是否完成由 Supervisor 与确定性验收条件判断。

## 4. Supervisor 职责

1. 把消息转成 Goal 和 Required Outputs；
2. 优先复用仍有效的 verified state；
3. 从小规模静态 Domain Entry 中选择领域；
4. 只派发当前有依据的任务；
5. 只有真正独立的任务才同波次；
6. 按 Expected Output 和 Evidence 验证结果；
7. 用 Authority、Freshness、Policy 或 Clarification 处理冲突；
8. 新结果暴露依赖时重规划；
9. 决定 Continue、Clarify、Degrade、Human 或 Finalize；
10. 用已验证 Fact/Artifact 组成 ResponsePlan。

不得编造 Tool 中不存在的数据、为了保险调用所有 Tool、提前写死所有跨域 API 顺序、直接执行不可逆副作用、把 completed 当 verified 或保存私有 Chain-of-thought。

## 5. Worker 契约

输入：

- 一个领域和一个明确 Objective；
- 最小必要 State Reference；
- Expected Outputs；
- Constraint、Policy、Budget、Deadline；
- 本地 Tool Contract 与 Credentials。

输出：

- Produced Capabilities；
- Evidence/Provenance；
- Warning/Confidence；
- 当前目标缺少的 Capability；
- Safe Error；
- Cost/Latency。

Worker 可以在本领域调用多个 Tool，也可有本地 Loop；不得静默进入其他领域。后续失败前产生的有效部分结果必须显式返回，由 Supervisor 分别校验和保留。

## 6. 动态跨域依赖

依赖可能在 Supervisor 首次规划、Worker 阅读 Tool Contract、Tool 返回部分结果/错误，或 Supervisor Review 时出现。

统一路径：

~~~text
Worker 发现缺 Capability
  → 返回 CapabilityRequest（reason + required_for）
  → Supervisor 检查 Global State
      已有且有效：Resume Worker
      没有：选择 provides 该能力的 Domain
            → DomainTask → Validate/Store
            → Resume 原 Worker 或 Replan
~~~

跨域协调必须回到 Supervisor，避免嵌套 Agent 树和隐藏循环。

## 7. 可选依赖

Tool Contract 描述参数语义，而不是写死全局边：

~~~text
assess_fit
  required: room_geometry
  optional: product_dimensions
  semantic:
    精确适配需要 product_dimensions；
    没有时只能提供泛化风格建议。
~~~

- “推荐风格”可以不需要尺寸；
- “是否放得下”必须有权威尺寸；
- “快速概念图”可用近似 Footprint，但需披露；
- “批准安装”缺精确测量必须 fail closed。

硬策略依赖仍由确定性门禁强制。

## 8. 并行波次

只有彼此不需要对方输出且都已获执行授权的任务才能同波次：

~~~text
Wave 1：独立 Fact/Permission
  ├─ Product Fact
  ├─ Customer Consent
  └─ Regional Policy

Supervisor Review
  ↓
Wave 2：由 Wave 1 解锁
  ├─ Consent 后的 Media/Space
  └─ Product Resolve 后的 Offer

Supervisor Review
  ↓
Wave 3：Media / Localization / Transaction
~~~

授权前不得读取敏感媒体；商品和素材未确认前不得渲染；同一幂等键的副作用不能并行 Retry。

## 9. Agent 数量与成本

~~~text
简单请求：Supervisor + 一个确定性 Domain Adapter
本领域歧义：Supervisor + 一个 Domain LLM
多领域：Supervisor + 分波次 Worker
高风险：再加 Deterministic Gate + HumanTask
~~~

Worker 是逻辑边界，不等于每次都调用独立大模型。领域子图可分层：

1. 明确场景用确定性路由；
2. 抽取/分类用小型结构化模型；
3. 只有多步歧义才用更强模型。

## 10. 状态

分开保存：

- Input Reference；
- Goal/Required Output；
- Verified Fact 与 Freshness；
- 待审核 Candidate；
- Task/Attempt；
- Artifact/Media Lineage；
- Missing Capability/Conflict；
- Supervisor Decision/Reason；
- Approval；
- Outbound Event/Receipt；
- Cost/Latency/Retry/Loop Budget。

大文件进入 Object Storage，State 只存稳定引用和 Hash。

## 11. 完成与 Loop 保护

完成要求：

- Required Outputs 齐全；
- 高风险 Claim 有有效 Fact Reference；
- Conflict/Missing 已解决或披露；
- 输出模态可用或安全降级；
- Policy Gate 通过；
- Required Approval 有效；
- Outbound Event 带幂等键。

保护：最大 Cycle、Token/Cost、Wall Clock、重复 Decision/Tool、按错误类型限制 Retry、检查过期 Fact，并对重复缺能力升级。

预算耗尽不是成功，只能返回已验证部分、提出聚焦问题或转人工。

## 12. 故障默认路由

| 故障 | 处理 |
|---|---|
| Invalid JSON | 拒绝，一次受限修复 |
| Tool 4xx | 安全时修参数，否则 blocked |
| Timeout/5xx | 仅在幂等且计费/副作用已知时重试 |
| Stale Fact | 刷新所属领域 |
| Conflict | 按 Authority/Freshness 或人工/澄清 |
| Missing Capability | 补领域、澄清或降级 |
| Policy Failure | 硬阻断，模型不能覆盖 |
| Send Unknown | 对账，禁止盲目重发 |

完整动作区分见 resilience-human-control.md。

## 13. Human-in-the-loop

常见触发：财务/法律不可逆动作、真实客户/产品生成内容、隐私媒体、低置信 Entity、权威冲突、策略例外和高成本。

审批台应展示 Candidate、Fact、Source、Diff、Warning、Policy、Cost 和 Artifact Hash。Resume 前使用同一 checkpoint，重新校验 TTL、权限和候选 Hash。

checkpoint 只能恢复内部状态，外部消息、支付、签名、发布或写入必须 Undo、Compensate 或 Reconcile。

## 14. API Gateway 边界

Gateway 可负责 Auth、Tenant、Region、Quota、Rate/Size、Content-Type、Malware Hook、Correlation、Idempotency Envelope、Channel Signature 和路由到 Agent Runtime。

Gateway 不通过 LLM 选择业务 Tool。

## 15. 不适用场景

- 小工具集单 Agent 已可靠；
- 流程完全确定；
- 没有跨域状态；
- 延迟无法承受 Loop；
- 只是单次转换且无 Tool/副作用；
- 普通 CRUD、固定审批或数据流水线已有更合适机制。
