# Agent 编排模式案例

案例用于校准，不可脱离实际约束复制。

## A：小型内部助手

场景：8 个只读 Tool，HR/IT 词汇相近，多数请求只调用一个 Tool，无财务/外部副作用，要求低延迟。

推荐：单 Agent 全量 Tool，Adapter 外做 Schema 和 Auth。

~~~text
User → Agent + 8 Tools → Validate → Reply
~~~

不选渐进领域：额外模型决策只增加延迟。Tool 增长、权限分离或领域独立后再迁移。

## B：多领域客服

场景：Account、Catalog、Order、Delivery、Refund、Language 共 45 个 Tool；有可选跨域依赖；退款写入高风险；订单/金额/地址必须精确；查订单状态需快速。

推荐：混合。

~~~text
明确 Order Status → Deterministic Fast-path

其他请求 → Supervisor
  ├─ Account Domain
  ├─ Order Domain
  ├─ Delivery Domain
  ├─ Refund Domain
  └─ Language Domain
       → Money/Write Gate → Human/Outbox
~~~

领域内 Tool 少时直接全量暴露，暂不加向量检索。

动态依赖：Delivery 可以在无身份时回答通用时效；查询当前包裹则返回需要 OrderIdentity 和 AuthorizedCustomerContext，Supervisor 补齐后 Resume。

## C：全球多模态商务（条件案例）

只有场景确实包含语音、图片或生成时才加入这些能力。

场景：Catalog、Inventory、Price、Logistics、Privacy、Asset、Vision、Generation、Language；客户文本/语音/住宅图；模拟图不能冒充商品事实；隐私媒体需授权。

推荐：Progressive Domain + Deterministic Gate + Specialist Service + Checkpoint + HumanTask。

~~~text
Gateway → Normalize Locale/Media → Supervisor
  ├─ Fact Domain
  ├─ Consent Domain
  └─ Consent 后的 Vision
       → Review → Generation
       → Provenance/Geometry/Policy Gate
       → Human → Outbox
~~~

SKU、尺寸、金额、日期和配送用类型化 Fact。需要语音时 ASR 负责输入/回听，TTS 负责输出；没有语音就删除两者。Simulation 必须有 Provenance 和 Disclosure。

不选固定 DAG：有的请求只需价格，有的需媒体，可选输入只对特定 Goal 必需。

## D：大型 Plugin 生态

场景：2000 个动态安装 Tool，目录每日变化，领域分类弱，租户/地区/副作用范围不同。

推荐：动态 Retrieval，并尽可能先做 Domain 或硬元数据过滤。

~~~text
Goal
  → Tenant/Permission/Region/Version/Read-write Filter
  → BM25/Embedding Candidate
  → Rerank + LLM Selection
  → Argument Validation → Execute
~~~

评测 Recall@k、Unsafe Filter、Stale Metadata、No-result Recovery 和总延迟。

## E：受监管审批

场景：提交阶段固定、法律期限/签名、LLM 只抽取文档和草拟建议、人工拥有审批、Transition 必须重放。

推荐：确定性 Workflow + 有边界 Agent Node。

~~~text
Receive → Validate → Agent Extraction → Deterministic Rule
  → Human Decision → Signed Side Effect → Reconciliation
~~~

Agent 可以报告 Missing Document，但不能绕过必需阶段。

## F：工程调查 Agent

场景：步骤顺序未知，使用 Filesystem、Test、Search、Code Execution，需要反复观察，存在安全 Sandbox 和 Version Control。

推荐：Bounded ReAct/Code Agent + Checkpoint + Permission。只有真正独立工作才创建子 Agent。

## 速查

~~~text
小且同质 Tool        → Full Exposure
稳定业务领域          → Progressive Domain
简单/复杂请求并存      → Hybrid
巨大动态 Tool         → Retrieval
固定受监管流程         → Deterministic Workflow
未知探索过程           → ReAct/Code Agent
~~~

从满足要求的最简单模式开始，并定义可度量迁移条件，不要第一天部署全部层。
