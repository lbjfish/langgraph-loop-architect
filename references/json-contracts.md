# JSON 契约与状态流转

节点间使用类型化 JSON，使路由、持久化、重放、测试和审计不依赖解析散文。JSON 只记录可观察状态，不记录私有推理。

## 1. 通用规则

- 包含 schema_version；
- 使用稳定 ID 和 Run/Conversation Correlation；
- Status/Action 使用明确 Enum；
- 大 Payload 使用 Reference；
- UTC 时间；
- Fact 包含 Provenance/Freshness；
- Decision 只保存简洁可观察理由；
- 需要完整性的内容保存 Hash。

分离：

~~~text
Fact      权威来源或确定性计算的事实
Claim     有 Fact 支撑的用户可见陈述
Artifact  检索或生成的文件/媒体
Task      工作及执行状态
Decision  可观察编排选择
Approval  可追责授权
Event     计划或完成的外部副作用
~~~

## 2. Supervisor 可见 Domain Entry

这是部署时的小型静态配置，不是具体 Tool Registry。

~~~json
{
  "schema_version": "1.0",
  "domain_entries": [
    {
      "domain": "catalog",
      "description": "解析商品并返回权威规格",
      "accepts": ["ProductMention", "SKU", "Locale"],
      "provides": ["ProductIdentity", "ProductProfile", "ProductDimensions"]
    },
    {
      "domain": "customer",
      "description": "读取权限允许的客户上下文",
      "accepts": ["CustomerRef", "ConsentScope"],
      "provides": ["CustomerContext", "ConsentDecision"]
    },
    {
      "domain": "recommendation",
      "description": "基于证据评估适配并产出建议",
      "accepts": ["Goal", "CustomerContext", "OptionalProductProfile"],
      "provides": ["Recommendation", "FitAssessment"]
    }
  ]
}
~~~

accepts 表示领域可理解的输入类型，不代表全部必填；provides 表示可产出的能力类型。

## 3. CapabilityProfile 与 ResiliencePolicy

~~~json
{
  "schema_version": "1.0",
  "capability_id": "capability:product_lookup",
  "name": "ProductLookup",
  "classification": "required",
  "business_purpose": "把商品名称解析为权威商品身份",
  "owner": "catalog_domain",
  "implementation_kind": "external_service",
  "accepts": ["ProductMention", "SKU"],
  "provides": ["ProductIdentity", "ProductProfile"],
  "authority": "authoritative_fact",
  "permissions": ["tenant_catalog_read"],
  "side_effect_class": "none",
  "slo": {"p95_latency_ms": 800, "availability_target": 0.999},
  "failure_modes": ["ambiguous_entity", "timeout", "provider_unavailable"],
  "resilience_policy_ref": "resilience:product_lookup_v1"
}
~~~

~~~json
{
  "schema_version": "1.0",
  "policy_id": "resilience:product_lookup_v1",
  "capability_ref": "capability:product_lookup",
  "allowed_degradation_level": "D4",
  "rules": [
    {"failure_class": "timeout_before_execution", "action": "retry", "max_attempts": 1},
    {"failure_class": "provider_unavailable", "action": "fallback", "target": "read_only_catalog_replica"},
    {"failure_class": "ambiguous_entity", "action": "clarify_user"},
    {"failure_class": "all_unavailable", "action": "human_handoff"}
  ],
  "fail_closed_conditions": ["no_authoritative_identity"]
}
~~~

分类：required、optional、deferred、not_applicable。副作用：none、reversible、compensatable、irreversible。

## 4. InputEnvelope

~~~json
{
  "schema_version": "1.0",
  "run_id": "run_01",
  "conversation_id": "conversation_42",
  "received_at": "2026-10-18T08:00:00Z",
  "channel": "web_chat",
  "actor": {
    "actor_ref": "customer_17",
    "tenant_ref": "tenant_acme",
    "locale": "zh-CN",
    "region": "CN-GD"
  },
  "message": {
    "mode": "mixed",
    "original_text": "这款设备适合我的办公室吗？",
    "normalized_text": "这款设备适合我的办公室吗？",
    "attachments": [
      {"artifact_ref": "upload_office_01", "media_type": "image/jpeg"}
    ]
  },
  "context_refs": ["prior:product_mention_01"]
}
~~~

## 5. GoalSpec

~~~json
{
  "schema_version": "1.0",
  "goal_id": "goal_01",
  "run_id": "run_01",
  "objective": "判断指定商品是否适合客户办公室",
  "required_outputs": ["ProductIdentity", "FitAssessment", "LocalizedReply"],
  "optional_outputs": ["VisualizationArtifact"],
  "constraints": [
    "商品事实必须来自权威来源",
    "没有授权不得处理隐私媒体",
    "不确定测量必须披露"
  ],
  "completion_checks": [
    "高风险 Claim 都有 Fact Reference",
    "不存在未解决关键冲突",
    "必需 Policy Gate 通过"
  ]
}
~~~

## 6. SupervisorDecision

~~~json
{
  "schema_version": "1.0",
  "decision_id": "decision_01",
  "run_id": "run_01",
  "goal_ref": "goal_01",
  "known_capabilities": ["ProductMention", "OfficeImage"],
  "missing_capabilities": ["ProductIdentity", "ConsentDecision"],
  "action": "dispatch_domains",
  "dispatch": [
    {"domain": "catalog", "objective": "解析商品身份", "wave": 1},
    {"domain": "customer", "objective": "检查图片用途授权", "wave": 1}
  ],
  "deferred": [
    {
      "domain": "recommendation",
      "objective": "产出 FitAssessment",
      "wait_for": ["ProductProfile", "ConsentDecision"]
    }
  ],
  "reason": "适配评估前必须确认商品和媒体权限",
  "stop": false
}
~~~

action：dispatch_domains、resume_domain、clarify_user、safe_fallback、request_human_review、finalize、fail。

## 7. DomainTask

~~~json
{
  "schema_version": "1.0",
  "task_id": "task_catalog_01",
  "run_id": "run_01",
  "domain": "catalog",
  "objective": "返回权威商品身份和尺寸",
  "input_refs": ["prior:product_mention_01"],
  "expected_outputs": ["ProductIdentity", "ProductDimensions"],
  "constraints": [
    "只使用 Catalog Tool",
    "存在歧义时不得猜测",
    "返回 Source Version 和时间"
  ],
  "budget": {"deadline_ms": 3000, "max_attempts": 2, "max_cost_usd": 0.02},
  "idempotency_key": "run_01:task_catalog_01:v1"
}
~~~

## 8. DomainResult 与 CapabilityRequest

~~~json
{
  "schema_version": "1.0",
  "task_id": "task_catalog_01",
  "run_id": "run_01",
  "domain": "catalog",
  "status": "completed",
  "produced_capabilities": [
    {"type": "ProductIdentity", "ref": "fact:product_identity_01", "confidence": 1.0},
    {"type": "ProductDimensions", "ref": "fact:product_dimensions_01", "confidence": 1.0}
  ],
  "needed_capabilities": [],
  "evidence_refs": ["source:catalog_product_v9"],
  "warnings": [],
  "metrics": {"latency_ms": 180, "cost_usd": 0.0, "attempts": 1},
  "error": null
}
~~~

~~~json
{
  "schema_version": "1.0",
  "task_id": "task_recommendation_01",
  "run_id": "run_01",
  "domain": "recommendation",
  "status": "needs_capability",
  "produced_capabilities": [],
  "needed_capabilities": [
    {
      "type": "ProductDimensions",
      "required_for": "FitAssessment",
      "required_now": true,
      "reason": "用户要求判断物理适配，而不是泛化风格建议",
      "acceptable_fallback": "ApproximateAssessmentWithDisclosure"
    }
  ],
  "error": null
}
~~~

状态：completed、needs_capability、blocked、failed、cancelled。completed 尚未等于 verified。

## 9. FactEnvelope 与 ArtifactEnvelope

~~~json
{
  "schema_version": "1.0",
  "fact_id": "fact:product_dimensions_01",
  "fact_type": "ProductDimensions",
  "source_system": "product_catalog",
  "source_version": "v9",
  "fetched_at": "2026-10-18T08:00:01Z",
  "expires_at": null,
  "payload": {"width_cm": 120, "depth_cm": 60, "height_cm": 75},
  "confidence": 1.0,
  "authority": "official",
  "content_hash": "sha256:example"
}
~~~

~~~json
{
  "schema_version": "1.0",
  "artifact_id": "artifact:visualization_01",
  "artifact_type": "VisualizationArtifact",
  "media_type": "image/png",
  "object_ref": "object://tenant_acme/run_01/visualization_01",
  "created_by": "media_domain",
  "input_refs": ["upload_office_01", "fact:product_dimensions_01"],
  "provenance": "simulation",
  "required_disclosures": ["这是模拟效果，不是精确测量"],
  "content_hash": "sha256:example",
  "retention_until": "2026-10-19T08:00:05Z"
}
~~~

## 10. SupervisorReview 与 State

~~~json
{
  "schema_version": "1.0",
  "review_id": "review_02",
  "run_id": "run_01",
  "accepted_results": ["task_catalog_01", "task_customer_01"],
  "satisfied_outputs": ["ProductIdentity", "ProductDimensions", "ConsentDecision"],
  "missing_outputs": ["FitAssessment", "LocalizedReply"],
  "conflicts": [],
  "next_action": "resume_domain",
  "next_tasks": [
    {"domain": "recommendation", "objective": "使用已验证事实完成 FitAssessment"}
  ],
  "should_finalize": false,
  "reason": "核心输入齐备，但用户目标尚未完成"
}
~~~

~~~json
{
  "schema_version": "1.0",
  "run_id": "run_01",
  "status": "running",
  "goal_ref": "goal_01",
  "fact_refs": [
    "fact:product_identity_01",
    "fact:product_dimensions_01",
    "fact:consent_decision_01"
  ],
  "tasks": [
    {"task_id": "task_catalog_01", "domain": "catalog", "status": "verified"},
    {"task_id": "task_recommendation_01", "domain": "recommendation", "status": "pending"}
  ],
  "unresolved_capabilities": ["FitAssessment", "LocalizedReply"],
  "decision_refs": ["decision_01", "review_02"],
  "approval_refs": [],
  "recovery_refs": [],
  "human_task_refs": [],
  "budget": {
    "supervisor_cycles": 2,
    "elapsed_ms": 900,
    "cost_usd": 0.004,
    "remaining_cost_usd": 0.196
  },
  "checkpoint_id": "checkpoint_03"
}
~~~

## 11. FinalResponse、Approval 与 Outbox

~~~json
{
  "schema_version": "1.0",
  "response_plan_id": "response_plan_01",
  "run_id": "run_01",
  "locale": "zh-CN",
  "modalities": ["text", "image"],
  "verified_claims": [
    {
      "claim_id": "claim_01",
      "text": "该产品宽 120 厘米。",
      "fact_refs": ["fact:product_dimensions_01"]
    }
  ],
  "required_disclosures": ["空间效果为模拟结果，实施前请复测"],
  "artifact_refs": ["artifact:visualization_01"],
  "human_review_required": true,
  "send_policy": "after_approval"
}
~~~

~~~json
{
  "schema_version": "1.0",
  "human_task_id": "human_task_01",
  "run_id": "run_01",
  "task_type": "approval",
  "risk_level": "high",
  "queue": "tenant_acme_high_risk",
  "eligible_roles": ["senior_operator"],
  "required_approvals": 1,
  "candidate_ref": "response_plan_01",
  "candidate_hash": "sha256:example",
  "allowed_actions": ["approve", "reject", "request_revision", "escalate"],
  "status": "pending",
  "expires_at": "2026-10-18T08:16:00Z"
}
~~~

~~~json
{
  "schema_version": "1.0",
  "event_id": "outbox_01",
  "run_id": "run_01",
  "event_type": "send_message",
  "destination": {"channel": "web_chat", "recipient_ref": "customer_17"},
  "payload_ref": "response_plan_01",
  "payload_hash": "sha256:example",
  "idempotency_key": "conversation_42:response_plan_01:v1",
  "authorization_refs": ["human_task_01"],
  "status": "pending",
  "attempts": 0
}
~~~

## 12. Recovery、Reconciliation 与 Compensation

~~~json
{
  "schema_version": "1.0",
  "recovery_id": "recovery_01",
  "run_id": "run_01",
  "trigger": {
    "failure_class": "provider_unavailable",
    "task_ref": "task_catalog_01",
    "side_effect_state": "none"
  },
  "action": "fallback",
  "target": "read_only_catalog_replica",
  "preserve_refs": ["fact:consent_decision_01"],
  "resume_from_checkpoint": "checkpoint_03",
  "requires_human": false,
  "reason": "只读副本满足同一契约"
}
~~~

~~~json
{
  "schema_version": "1.0",
  "reconciliation_id": "reconciliation_01",
  "run_id": "run_01",
  "original_event_ref": "outbox_01",
  "external_reference": "provider_request_7788",
  "reason": "请求提交后超时",
  "status": "reconciling",
  "observed_external_state": "unknown",
  "retry_blocked": true
}
~~~

~~~json
{
  "schema_version": "1.0",
  "compensation_id": "compensation_01",
  "run_id": "run_01",
  "original_event_ref": "outbox_01",
  "reason": "已完成动作需要被业务补偿",
  "action": "send_correction",
  "authorization_refs": ["human_task_02"],
  "idempotency_key": "outbox_01:compensation:v1",
  "status": "pending",
  "residual_risk": "接收方可能已经依据原消息行动"
}
~~~

恢复 Action：retry、fallback、degrade、resume、rollback_internal、compensate、reconcile、human_handoff、fail_closed。

## 13. 状态流转

~~~text
InputEnvelope → GoalSpec → SupervisorDecision → DomainTask
  → DomainResult / CapabilityRequest
  → Validate → Fact/Artifact → SupervisorReview
      → Continue / Clarify / Fallback / Human / Finalize
      → Deterministic Gate → Outbox

任何 Failure/Unknown Side Effect
  → RecoveryDecision
  → Retry / Fallback / Degrade / Resume
     / Internal Rollback / Reconcile / Compensate / Human / Fail Closed
~~~

## 14. 校验不变量

- ID 在类型内唯一且不可变；
- Reference 必须在 Tenant/Auth 边界内可解析；
- Worker 只能产出 Domain 声明能力；
- completed 不能直接创建 Outbox；
- 高风险 Claim 必须引用有效 Fact；
- Expired Fact 不能满足 Completion；
- Artifact 保存 Input、Model/Provider、Provenance、Hash；
- Reason 不是私有推理；
- Unknown 不得用 Placeholder 冒充 Fact；
- Resume 使用新 Capability Reference，避免重复；
- Side Effect 使用 Payload Hash 和 Idempotency；
- Approval 绑定精确 Candidate Hash，内容变化即失效；
- Compensation 是新动作，不改写原始历史。
