# JSON contracts and state transitions

Use typed JSON between nodes so routing, persistence, replay, tests, and audit do not depend on parsing free-form prose. The examples are transport contracts, not private reasoning traces.

## 1. Contract rules

Every durable object should include:

- `schema_version`;
- stable ID and parent/run correlation;
- explicit status or action enum;
- references to facts and artifacts instead of copied large payloads;
- timestamps in UTC;
- provenance and freshness for facts;
- concise observable reason, never chain-of-thought;
- immutable content hash where integrity matters.

Use canonical capability names such as `CustomerIdentity`, `ProductDimensions`, `RiskAssessment`, or `LocalizedReply`. A capability describes a typed business result, not a concrete tool.

Keep these object types separate:

```text
Fact       = externally sourced or deterministically computed truth
Claim      = user-facing statement backed by one or more facts
Artifact   = generated or retrieved file/media/output
Task       = requested work and its execution status
Decision   = observable orchestration choice
Approval   = accountable authorization
Event      = intended or completed side effect
```

## 2. Supervisor-visible domain entries

This is a small static deployment configuration, not a runtime registry of concrete tools.

```json
{
  "schema_version": "1.0",
  "domain_entries": [
    {
      "domain": "catalog",
      "description": "Resolve products and return authoritative specifications",
      "accepts": ["ProductMention", "SKU", "Locale"],
      "provides": ["ProductIdentity", "ProductProfile", "ProductDimensions"]
    },
    {
      "domain": "customer",
      "description": "Read permitted customer context and preferences",
      "accepts": ["CustomerRef", "ConsentScope"],
      "provides": ["CustomerContext", "ConsentDecision"]
    },
    {
      "domain": "recommendation",
      "description": "Evaluate fit and produce evidence-backed recommendations",
      "accepts": ["Goal", "CustomerContext", "OptionalProductProfile"],
      "provides": ["Recommendation", "FitAssessment"]
    }
  ]
}
```

`accepts` lists input capability types the domain can understand; it does not mean every item is required. Required and optional semantics belong to local Tool Contracts and the current task objective. `provides` lists output capability types the domain can produce.

## 3. Input envelope

```json
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
      {
        "artifact_ref": "upload_office_01",
        "media_type": "image/jpeg"
      }
    ]
  },
  "context_refs": ["prior:product_mention_01"]
}
```

## 4. Goal specification

The Supervisor should externalize the goal before choosing tools.

```json
{
  "schema_version": "1.0",
  "goal_id": "goal_01",
  "run_id": "run_01",
  "objective": "Evaluate whether the referenced product suits the customer's office",
  "required_outputs": ["ProductIdentity", "FitAssessment", "LocalizedReply"],
  "optional_outputs": ["VisualizationArtifact"],
  "constraints": [
    "Use authoritative product facts",
    "Do not process private media without permitted purpose",
    "Disclose uncertain measurements"
  ],
  "completion_checks": [
    "Every high-risk claim has fact references",
    "No unresolved critical conflict",
    "Required policy gates pass"
  ]
}
```

## 5. Supervisor decision

```json
{
  "schema_version": "1.0",
  "decision_id": "decision_01",
  "run_id": "run_01",
  "goal_ref": "goal_01",
  "known_capabilities": ["ProductMention", "OfficeImage"],
  "missing_capabilities": ["ProductIdentity", "ConsentDecision"],
  "action": "dispatch_domains",
  "dispatch": [
    {
      "domain": "catalog",
      "objective": "Resolve the product mention and return ProductIdentity",
      "wave": 1
    },
    {
      "domain": "customer",
      "objective": "Check whether the office image may be used for this assessment",
      "wave": 1
    }
  ],
  "deferred": [
    {
      "domain": "recommendation",
      "objective": "Produce FitAssessment",
      "wait_for": ["ProductProfile", "ConsentDecision"]
    }
  ],
  "reason": "Product identity and media permission must be verified before assessment",
  "stop": false
}
```

Recommended `action` enum:

- `dispatch_domains`
- `resume_domain`
- `clarify_user`
- `safe_fallback`
- `request_human_review`
- `finalize`
- `fail`

## 6. Domain task

```json
{
  "schema_version": "1.0",
  "task_id": "task_catalog_01",
  "run_id": "run_01",
  "domain": "catalog",
  "objective": "Resolve the referenced product and return authoritative identity and dimensions",
  "input_refs": ["prior:product_mention_01"],
  "expected_outputs": ["ProductIdentity", "ProductDimensions"],
  "constraints": [
    "Use only catalog-domain tools",
    "Return ambiguity rather than guessing",
    "Include source version and timestamp"
  ],
  "budget": {
    "deadline_ms": 3000,
    "max_attempts": 2,
    "max_cost_usd": 0.02
  },
  "idempotency_key": "run_01:task_catalog_01:v1"
}
```

## 7. Completed domain result

```json
{
  "schema_version": "1.0",
  "task_id": "task_catalog_01",
  "run_id": "run_01",
  "domain": "catalog",
  "status": "completed",
  "produced_capabilities": [
    {
      "type": "ProductIdentity",
      "ref": "fact:product_identity_01",
      "confidence": 1.0
    },
    {
      "type": "ProductDimensions",
      "ref": "fact:product_dimensions_01",
      "confidence": 1.0
    }
  ],
  "needed_capabilities": [],
  "evidence_refs": ["source:catalog_product_v9"],
  "warnings": [],
  "metrics": {
    "latency_ms": 180,
    "cost_usd": 0.0,
    "attempts": 1
  },
  "error": null
}
```

Recommended result statuses:

- `completed`: local objective returned, not yet Supervisor-verified;
- `needs_capability`: current objective needs another typed capability;
- `blocked`: user, policy, or external state is required;
- `failed`: bounded attempts exhausted;
- `cancelled`: Supervisor no longer needs the task.

## 8. Capability request

Use this result when a Worker discovers a cross-domain need. It is not a tool call.

```json
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
      "reason": "The user asked for physical fit rather than general style guidance",
      "acceptable_fallback": "ApproximateAssessmentWithDisclosure"
    }
  ],
  "evidence_refs": [],
  "warnings": [],
  "metrics": {
    "latency_ms": 45,
    "cost_usd": 0.001,
    "attempts": 1
  },
  "error": null
}
```

## 9. Fact envelope

```json
{
  "schema_version": "1.0",
  "fact_id": "fact:product_dimensions_01",
  "fact_type": "ProductDimensions",
  "source_system": "product_catalog",
  "source_ref": "catalog_product_v9",
  "source_version": "v9",
  "fetched_at": "2026-10-18T08:00:01Z",
  "expires_at": null,
  "payload": {
    "width_cm": 120,
    "depth_cm": 60,
    "height_cm": 75
  },
  "confidence": 1.0,
  "authority": "official",
  "content_hash": "sha256:example"
}
```

For volatile facts such as price, inventory, delivery, permissions, or account state, `expires_at` must be non-null or the system must define another freshness rule.

## 10. Artifact envelope

```json
{
  "schema_version": "1.0",
  "artifact_id": "artifact:visualization_01",
  "artifact_type": "VisualizationArtifact",
  "media_type": "image/png",
  "object_ref": "object://tenant_acme/run_01/visualization_01",
  "created_at": "2026-10-18T08:00:05Z",
  "created_by": "media_domain",
  "input_refs": [
    "upload_office_01",
    "fact:product_dimensions_01"
  ],
  "model": {
    "provider": "configured_provider",
    "model_id": "configured_model",
    "parameters_ref": "generation_params_01"
  },
  "provenance": "simulation",
  "required_disclosures": ["This is a simulation, not an exact measurement"],
  "content_hash": "sha256:example",
  "retention_until": "2026-10-19T08:00:05Z"
}
```

## 11. Supervisor review

```json
{
  "schema_version": "1.0",
  "review_id": "review_02",
  "run_id": "run_01",
  "goal_ref": "goal_01",
  "accepted_results": ["task_catalog_01", "task_customer_01"],
  "rejected_results": [],
  "satisfied_outputs": ["ProductIdentity", "ProductDimensions", "ConsentDecision"],
  "missing_outputs": ["FitAssessment", "LocalizedReply"],
  "conflicts": [],
  "next_action": "resume_domain",
  "next_tasks": [
    {
      "domain": "recommendation",
      "objective": "Produce FitAssessment using verified facts"
    }
  ],
  "should_finalize": false,
  "reason": "Inputs are verified but the user-facing assessment is not complete"
}
```

## 12. Global state snapshot

```json
{
  "schema_version": "1.0",
  "run_id": "run_01",
  "conversation_id": "conversation_42",
  "status": "running",
  "goal_ref": "goal_01",
  "input_refs": ["input:run_01"],
  "fact_refs": [
    "fact:product_identity_01",
    "fact:product_dimensions_01",
    "fact:consent_decision_01"
  ],
  "artifact_refs": [],
  "tasks": [
    {
      "task_id": "task_catalog_01",
      "domain": "catalog",
      "status": "verified"
    },
    {
      "task_id": "task_recommendation_01",
      "domain": "recommendation",
      "status": "pending"
    }
  ],
  "unresolved_capabilities": ["FitAssessment", "LocalizedReply"],
  "conflicts": [],
  "decision_refs": ["decision_01", "review_02"],
  "approval_refs": [],
  "budget": {
    "supervisor_cycles": 2,
    "elapsed_ms": 900,
    "cost_usd": 0.004,
    "remaining_cost_usd": 0.196
  },
  "checkpoint_id": "checkpoint_03",
  "updated_at": "2026-10-18T08:00:02Z"
}
```

## 13. Final response plan

```json
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
  "required_disclosures": ["空间效果为模拟结果，请在实施前复测"],
  "artifact_refs": ["artifact:visualization_01"],
  "unresolved_items": [],
  "human_review_required": true,
  "send_policy": "after_approval"
}
```

## 14. Approval request

```json
{
  "schema_version": "1.0",
  "approval_id": "approval_01",
  "run_id": "run_01",
  "risk_level": "high",
  "reasons": ["private customer image", "generated visualization", "external send"],
  "candidate_ref": "response_plan_01",
  "fact_refs": ["fact:product_dimensions_01", "fact:consent_decision_01"],
  "artifact_refs": ["artifact:visualization_01"],
  "allowed_actions": ["approve", "reject", "request_revision"],
  "candidate_hash": "sha256:example",
  "expires_at": "2026-10-18T08:15:00Z"
}
```

## 15. Outbox event

Prepare a side effect only after required validation and approval.

```json
{
  "schema_version": "1.0",
  "event_id": "outbox_01",
  "run_id": "run_01",
  "event_type": "send_message",
  "destination": {
    "channel": "web_chat",
    "recipient_ref": "customer_17"
  },
  "payload_ref": "response_plan_01",
  "payload_hash": "sha256:example",
  "idempotency_key": "conversation_42:response_plan_01:v1",
  "authorization_refs": ["approval_01"],
  "status": "pending",
  "created_at": "2026-10-18T08:01:00Z",
  "attempts": 0,
  "last_error": null
}
```

## 16. State transition map

```text
InputEnvelope
    ↓ normalize and validate
GoalSpec
    ↓ decide
SupervisorDecision
    ↓ dispatch each current-wave task
DomainTask
    ↓ local execution
DomainResult ──needs_capability──→ CapabilityRequest
    ↓                                   ↓
schema validation                   SupervisorDecision
    ↓                                   ↓
FactEnvelope / ArtifactEnvelope ← new domain result
    ↓
SupervisorReview
    ├─ continue → SupervisorDecision
    ├─ clarify → new InputEnvelope
    ├─ fallback → FinalResponsePlan with unresolved disclosure
    ├─ human → ApprovalRequest
    └─ finalize → FinalResponsePlan
                       ↓
               deterministic gates
                       ↓
                  OutboxEvent
```

## 17. Validation invariants

- IDs are unique within their type and immutable.
- Every reference resolves inside the tenant and authorization boundary.
- A Worker may only produce capability types declared by its domain entry.
- `completed` results cannot directly create an Outbox event.
- High-risk claims require valid fact references.
- Expired facts cannot satisfy current completion checks.
- Artifacts preserve input references, model/provider identity, parameters reference, provenance, and hash.
- `reason` is a concise observable justification, not hidden reasoning.
- Unknown values remain `null`, absent, or unresolved; never use placeholder facts in a final claim.
- A resumed task receives new capability references instead of duplicating prior results.
- Side effects use immutable payload hashes and idempotency keys.
- Approval is bound to the exact candidate hash and expires when facts or artifacts change.
