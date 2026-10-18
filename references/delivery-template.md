# Technical proposal delivery template

Use this structure for a complete architecture answer. Omit sections that do not affect the decision; do not fill space with generic framework descriptions.

## 1. Executive recommendation

State in the first paragraph:

- selected pattern;
- business outcome it optimizes;
- why it fits the tool/domain/dependency/risk profile;
- the most important tradeoff;
- simpler fallback or migration trigger.

Example shape:

```text
Recommend a hybrid architecture: deterministic fast paths for stable read-only intents,
plus a Supervisor Loop with progressive domain Workers for ambiguous multi-domain goals.
This avoids global tool-context growth and preserves runtime dependency discovery,
while keeping p95 low for common requests. Dynamic retrieval is deferred until a
single domain exceeds the local schema budget or tool churn makes static lists costly.
```

## 2. Scenario and assumptions

| Item | Decision-relevant detail | Evidence or assumption |
|---|---|---|
| Users/channels |  |  |
| Business goal |  |  |
| Tools and data authorities |  |  |
| Side effects |  |  |
| Domains |  |  |
| Dependency shape |  |  |
| Latency/throughput |  |  |
| Privacy/compliance |  |  |
| Availability/fallback |  |  |

List unknowns whose answers could change the pattern.

## 3. Strategy comparison

Compare the selected pattern with at least one credible alternative:

| Pattern | Accuracy | Latency | Scale/churn | Security | Complexity | Verdict |
|---|---|---|---|---|---|---|
| Full tool exposure |  |  |  |  |  |  |
| Progressive domains |  |  |  |  |  |  |
| Hybrid |  |  |  |  |  |  |
| Dynamic retrieval |  |  |  |  |  |  |
| Deterministic workflow |  |  |  |  |  |  |

Explain why the runner-up was rejected now and what would make it preferable later.

## 4. Frozen components and ownership

| Component | Owns | Must not do |
|---|---|---|
| API Gateway |  |  |
| Input processor |  |  |
| Supervisor |  |  |
| Domain entries |  |  |
| Domain Workers |  |  |
| Deterministic gates |  |  |
| Global state |  |  |
| Human approval |  |  |
| Outbox |  |  |

Remove components not used. Add a retriever only if the selection analysis justifies it.

### Capability boundary matrix

For every candidate capability, mark `required`, `optional`, `deferred`, or `not_applicable`, then document:

| Capability | Classification | Owner | Accepts | Provides | Authority | Permissions | Side effect | SLO/cost | Fallback | Recovery | Human trigger | Tests |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
|  |  |  |  |  |  |  |  |  |  |  |  |  |

Do not include ASR, TTS, translation, vision, generation, 3D, payment, or notification merely because another example used it. Include only capabilities supported by the scenario's channels, inputs, outputs, and business requirements.

## 5. End-to-end flow

Prefer a plain-text diagram unless the user requests Mermaid:

```text
Channel → Gateway → Normalize → Decide → Execute → Review
                                      ↑          ↓
                                      └── Loop ──┘
                                                   ↓
                                      Gate → Approve → Outbox
```

Describe the responsibilities at each transition, not framework syntax.

## 6. Tool exposure and routing

Document:

- what the Supervisor can see;
- what each Worker can see;
- whether local tools are fully exposed or retrieved;
- how concrete tool permissions are enforced;
- how capabilities map to domains;
- when a deterministic fast path bypasses the Supervisor;
- why the design does not expose more or fewer tools.

## 7. Dependency behavior

Show:

- one fixed hard dependency;
- one optional dependency whose necessity changes by goal;
- one dependency discovered after a tool response;
- parallel waves and why tasks are independent;
- missing capability, clarification, and fallback behavior.

## 8. JSON contracts

At minimum include or reference:

- DomainEntrySet;
- InputEnvelope;
- GoalSpec;
- SupervisorDecision;
- DomainTask;
- DomainResult/CapabilityRequest;
- FactEnvelope and ArtifactEnvelope;
- SupervisorReview and StateSnapshot;
- FinalResponsePlan;
- ApprovalRequest and OutboxEvent for side effects.

For each contract, explain who produces it, who consumes it, and the validation invariant.

## 9. Scenario trace

Walk through one realistic request step by step:

```text
1. Input arrives and is normalized.
2. Supervisor declares the goal and required outputs.
3. Current wave is dispatched.
4. A Worker produces a partial result or capability request.
5. Supervisor reviews state and replans.
6. Missing capability is obtained and the original Worker resumes.
7. Completion checks pass.
8. Policy and human gates run.
9. Idempotent outbound event is recorded and sent.
```

Also show one failure or degraded path.

## 10. Performance and model routing

Include:

- p50/p95 budget by stage;
- which paths avoid an LLM;
- model size/role by task;
- caching and fact reuse;
- concurrency rules;
- token and external-service budgets;
- conditions that justify adding retrieval.

## 11. Security, privacy, and reliability

Cover least privilege, untrusted tool output, tenant isolation, schema validation, provenance, TTL, idempotency, approval binding, media lifecycle, retries, and reconciliation.

Also distinguish:

- retry;
- equivalent fallback;
- reduced-quality or reduced-modality degradation;
- internal state rollback;
- external business compensation;
- unknown-outcome reconciliation;
- human handoff;
- fail-closed behavior.

State the highest permitted degradation level per critical capability. Describe the complete HumanTask lifecycle only where human intervention is justified: trigger, queue, role, SLA, expiry, candidate hash, allowed decisions, revalidation, resume, escalation, and audit.

## 12. Testing and metrics

Specify unit, contract, graph, E2E, failure-injection, and golden-set coverage. Include architecture-specific metrics and fail-closed thresholds for high-risk fields.

## 13. Implementation roadmap

Recommended phases:

1. typed state and mocked tools;
2. one-domain vertical slice;
3. Supervisor Loop and dynamic dependency;
4. side-effect gates and approval;
5. multimodal or specialist services;
6. shadow, read-only, limited automation, expanded rollout.

Every phase needs an observable acceptance test and rollback path.

Multimodal or specialist services are conditional. Omit that phase when all such capabilities are `not_applicable`.

## 14. Final decision record

End with:

- selected architecture;
- explicitly excluded components;
- assumptions to validate;
- migration triggers;
- first implementation milestone.
