# Evaluation, production readiness, and rollout

Architecture quality is demonstrated by observable behavior, not by the number of agents or diagrams.

## 1. Requirement-to-evidence matrix

For each business requirement, record:

| Field | Meaning |
|---|---|
| Requirement | Exact user or business need |
| Risk | Consequence of failure |
| Owner | Supervisor, domain, deterministic service, or human |
| Contract | JSON object and field proving the behavior |
| Test | Unit, contract, graph, E2E, or evaluation case |
| Metric | Threshold or expected result |
| Fallback | Safe behavior if unmet |

## 2. Test layers

### Unit tests

Test deterministic logic directly:

- arithmetic, dates, TTL, currency, dimensions, quotas;
- capability and status enums;
- authority/freshness conflict resolution;
- idempotency key generation;
- policy and permission gates;
- loop budgets and repeat detection;
- safe error classification.

### Contract tests

- Every JSON example validates against the production schema.
- Every domain entry declares only supported capabilities.
- Every Worker result uses declared output types.
- Tool adapters map API success and error responses consistently.
- Optional input semantics are documented and exercised.
- References cannot cross tenant boundaries.
- Unknown fields are rejected or versioned intentionally.

### Graph tests

- one-domain happy path;
- truly independent tasks run in the same wave;
- a dependency is discovered after a partial result;
- a missing capability is already in state and avoids a duplicate call;
- the Supervisor selects the correct provider domain;
- the original Worker resumes with the new reference;
- stale facts trigger refresh;
- conflicting facts trigger authority resolution or clarification;
- a rejected Worker result does not become verified state;
- loop budget and repeated-decision protection work;
- checkpoint recovery does not duplicate cost or side effects.

### End-to-end tests

Use realistic channel inputs, real schemas, stubbed or sandbox APIs, persistence, and outbound reconciliation. Prove:

- text, voice, image, and mixed input when applicable;
- language detection and target-language output;
- privacy and media lifecycle;
- partial provider outage and local fallback;
- human approval pause and resume;
- exact outbound content hash;
- clean restart and recovery;
- concurrent conversations and tenant isolation.

### Live smoke tests

Only when explicitly authorized and safe:

- use a non-production tenant or sandbox;
- cap cost and tool calls;
- do not send to a real customer without explicit permission;
- record provider, model, parameters, latency, cost, and result ID;
- do not blindly retry a timeout when billing or side-effect state is unknown.

## 3. Golden evaluation set

Build examples across these dimensions:

- clear versus ambiguous intent;
- exact versus misspelled entity;
- one domain versus several domains;
- fixed, optional, and runtime-discovered dependencies;
- fresh, stale, absent, and conflicting facts;
- read-only versus irreversible side effects;
- low versus high confidence;
- normal, adversarial, and prompt-injected tool content;
- normal latency, timeout, rate limit, and partial failure;
- approved, rejected, expired, and modified human-review candidates.

Include counterexamples where the correct action is to ask, refuse, or return a partial result.

## 4. Pattern-specific metrics

### Full tool exposure

- tool-selection accuracy;
- wrong-domain and wrong-argument rate;
- prompt/tool-schema token cost;
- p50/p95 latency as tool count grows.

### Progressive disclosure

- domain-selection accuracy;
- local tool-selection accuracy;
- unnecessary domain dispatch rate;
- dependency recovery success;
- Supervisor cycles per request;
- end-to-end latency versus a flat baseline.

### Dynamic retrieval

- retrieval recall@k;
- permission-filter correctness;
- reranking accuracy;
- no-result recovery rate;
- retrieval and total latency;
- stale metadata rate.

### Deterministic workflow

- transition coverage;
- invalid transition rejection;
- policy escape rate;
- recovery and replay correctness.

### ReAct loop

- task success;
- action efficiency;
- repeated-action rate;
- average turns/tokens/cost;
- loop-budget termination quality;
- unsafe tool attempt rate.

## 5. Business and reliability metrics

Choose metrics relevant to the product:

- task completion and human acceptance;
- critical fact accuracy;
- unsupported claim rate;
- clarification and handoff rate;
- p50/p95/p99 response time;
- per-run model and external-service cost;
- tool/API error and retry rate;
- checkpoint recovery success;
- duplicate side effects;
- stale-fact send rate;
- policy violations and privacy incidents;
- customer conversion, resolution, or completion outcome.

For high-risk fields, use fail-closed thresholds rather than an average score.

## 6. Observability

Record observable execution data:

- trace, run, conversation, task, decision, and checkpoint IDs;
- graph and schema versions;
- selected pattern, domain entries, and task waves;
- tool name, sanitized argument summary, status, latency, retry, and authority;
- model/provider, parameters, tokens, latency, and cost;
- fact and artifact references, hashes, TTL, and provenance;
- Supervisor action and concise reason;
- policy results, approvals, and outbound receipts.

Do not log credentials, authorization headers, unnecessary personal data, raw private media, or private model reasoning.

## 7. Performance analysis

Break total latency into:

```text
gateway + normalization + Supervisor decisions + domain work
+ tool calls + queueing + generation + validation + approval + send
```

Compare architectures using the same scenario set:

- flat one-Agent baseline;
- selected architecture;
- deterministic fast-path subset;
- outage/degraded mode.

Report both accuracy and latency. A design that improves tool accuracy but triples p95 for trivial requests may need a hybrid fast path.

Useful optimizations:

- cache stable domain descriptions and schemas;
- reuse valid facts from state;
- batch independent reads;
- execute deterministic local routing without an LLM;
- keep Supervisor context to verified summaries and references;
- use smaller models for extraction and local classification;
- limit retrieval to the selected domain;
- avoid generating media before fact and permission gates pass.

## 8. Security review

Threat-model:

- prompt injection in user, webpage, OCR, or tool output;
- tool argument smuggling;
- cross-tenant reference access;
- overprivileged Worker credentials;
- unsafe dynamically retrieved tools;
- duplicate writes and replay;
- generated media impersonating facts or evidence;
- approval candidate changed after approval;
- stale policy or pricing facts;
- sensitive state sent to external observability services.

Required mitigations vary by risk but commonly include least privilege, allowlists, hard metadata filters, schema validation, provenance, immutable hashes, idempotency, TTL checks, redaction, and human approval.

## 9. Failure injection

Deliberately test:

- malformed JSON from a model;
- a tool returning prompt-like instructions;
- one domain returning a conflicting fact;
- an optional input absent in one goal and essential in another;
- timeouts before and after an unknown side effect;
- expired approval;
- state restored from an older checkpoint;
- the same event delivered twice;
- a capability request that no domain can provide;
- a Supervisor repeating the same decision.

## 10. Rollout stages

### Stage 0: offline architecture evaluation

- mock tools and fixture state;
- contract and graph tests;
- compare at least two patterns;
- prove fallbacks and loop limits.

### Stage 1: shadow mode

- receive production-like inputs but do not act;
- compare recommended actions with human or existing-system decisions;
- measure tool/domain selection, latency, and unsupported claims.

### Stage 2: read-only assistance

- allow verified lookups and drafts;
- require humans for all sends and writes;
- reconcile every result.

### Stage 3: limited automation

- automate low-risk, high-confidence intents;
- keep high-risk cases behind approval;
- use tenant/region allowlists and rollback switches.

### Stage 4: expanded autonomy

- expand only when error, incident, latency, and business metrics meet thresholds;
- preserve kill switches, audit, and human escalation.

## 11. Acceptance checklist

- The selected architecture is simpler than or clearly superior to alternatives.
- Every component has a single owner and prohibited responsibilities.
- JSON contracts validate and state transitions are explicit.
- Optional dependencies are not globally hard-coded.
- Dynamic dependency recovery is demonstrated.
- Facts, claims, artifacts, approvals, and side effects are separate.
- Loop, cost, retry, and time limits are defined.
- High-risk actions fail closed.
- Checkpoint recovery and idempotency are tested.
- Degraded mode still provides a truthful usable result.
- Metrics can identify whether accuracy gains justify latency and cost.
- The current implementation is not misrepresented as the target architecture.
