# Supervisor Loop and progressive domain design

Use this reference after the selection guide recommends a Supervisor, progressive disclosure, or hybrid architecture.

## 1. Frozen runtime vocabulary

Define the components before the flow so the design does not later invent an unexplained registry or router.

| Component | Owns | Must not own |
|---|---|---|
| Channel adapter | IM, Web, voice, webhook protocol conversion | business planning |
| API Gateway | authentication, tenant isolation, quotas, request size, idempotency envelope | intent or tool selection |
| Input processor | ASR, language ID, media sanitation, normalization | business conclusions |
| Supervisor Agent | global goal, domain choice, wave scheduling, result review, replanning, stop decision | concrete business API calls |
| Domain entry | compact static description of one domain's inputs and outputs | concrete API inventory |
| Domain Worker | local planning and local tool calls | cross-domain calls or final send |
| Tool Contract | local API semantics, schemas, limits, examples, errors | global orchestration policy |
| Global state/checkpointer | facts, tasks, artifacts, decisions, budgets, unresolved needs, recovery points | secrets or unnecessary binaries |
| Deterministic gates | permissions, arithmetic, TTL, provenance, policy, side-effect authorization | open-ended planning |
| Human approval | accountable decision for defined high-risk cases | routine low-risk routing |
| Outbox/sender | transactional, idempotent delivery | content generation or mutation |

Optional components such as a critic, retriever, planner, or policy agent must have a demonstrated purpose. Do not duplicate Supervisor responsibilities under a new name.

## 2. Domain entry versus concrete tool

A domain entry is a high-level tool or subgraph exposed to the Supervisor:

```text
catalog_domain
  accepts: ProductMention, SKU, Locale
  provides: ProductProfile, ProductDimensions, VariantAvailability
```

Concrete tools remain local:

```text
Catalog Worker sees:
  catalog.search_product
  catalog.get_product
  catalog.get_dimensions
  catalog.get_variants
```

The Supervisor says “produce ProductDimensions for this product.” It does not say “call catalog.get_dimensions” unless the chosen architecture deliberately uses full exposure.

## 3. Canonical loop

```text
InputEnvelope
      ↓
Normalize and identify language/modality
      ↓
Supervisor Decide
  ├─ finalize using verified state
  ├─ clarify with the user
  ├─ safe fallback or human handoff
  └─ dispatch/resume domain tasks
                ↓
        Domain Worker executes local tools
                ↓
        DomainResult or CapabilityRequest
                ↓
        schema and deterministic validation
                ↓
        Supervisor Review
          ├─ reject/conflict → replan
          ├─ missing capability → dispatch another domain
          ├─ incomplete goal → continue loop
          └─ goal satisfied → final policy gates
                                      ↓
                         approval if required → outbox
```

LangGraph supplies state, nodes, conditional edges, subgraphs, checkpointing, interrupts, and resumability. It does not decide whether the business goal is complete; that decision belongs to the Supervisor plus deterministic acceptance criteria.

## 4. Supervisor responsibilities

The Supervisor should:

1. Convert the message into an explicit goal and required outputs.
2. Reuse verified state before scheduling new work.
3. Select domains from a small static domain-entry list.
4. Dispatch only currently justified tasks.
5. Put genuinely independent tasks in the same execution wave.
6. Validate every Worker result against the expected output and evidence.
7. Resolve conflicts by authority, freshness, policy, or clarification.
8. Replan when results reveal new needs.
9. Decide whether to continue, clarify, degrade, request approval, or finalize.
10. Compose the response plan from verified facts and artifacts.

It should not:

- invent values absent from tools;
- call every tool “just in case”;
- write concrete cross-domain API order before evidence requires it;
- directly perform an irreversible side effect;
- treat `completed` as equivalent to `verified`;
- persist private chain-of-thought.

## 5. Domain Worker responsibilities

A Worker receives:

- a single domain;
- a concrete objective, not a vague persona;
- references to the minimum state needed;
- expected output capabilities;
- constraints, policy, budget, and deadline;
- its local Tool Contracts and credentials.

It returns:

- produced typed capabilities;
- evidence and provenance references;
- concise warnings and confidence;
- missing capabilities required for the current objective;
- safe error information;
- local cost and latency.

A Worker may choose several tools in its own domain. It may perform a local loop. It may not silently reach into another domain.

If a Worker produces valid partial outputs before becoming blocked or failing, return them explicitly. The Supervisor validates each output independently, preserves verified facts, and keeps the remaining required outputs unresolved.

## 6. Dynamic cross-domain dependency

Dependencies may be discovered:

1. by the Supervisor before dispatch;
2. by a Worker while interpreting a local Tool Contract;
3. only after a tool returns partial data or an error;
4. during Supervisor review when an expected output is not justified.

Use one path for every case:

```text
Worker detects missing capability
        ↓
returns CapabilityRequest with reason and required_for
        ↓
Supervisor checks Global State
        ↓
capability already present and valid?
  ├─ yes → resume Worker with reference
  └─ no  → select a domain whose declared provides includes the capability
              ↓
           dispatch DomainTask
              ↓
           validate and store result
              ↓
           resume original Worker or replan
```

This is star-shaped orchestration: all cross-domain coordination returns through the Supervisor. It avoids nested agent trees, hidden dependencies, and circular ownership.

## 7. Optional dependency semantics

Do not turn every optional parameter into a hard global edge. The local Tool Contract should explain the value of the parameter:

```text
assess_fit
  required: room_geometry
  optional: product_dimensions
  semantic note:
    product_dimensions is necessary when the objective is an exact fit assessment;
    without it the tool may produce only general style guidance.
```

The Worker applies the current objective:

- “Suggest a style” may proceed without dimensions.
- “Will this product fit?” requires authoritative dimensions.
- “Create a quick concept” may use an approximate footprint with disclosure.
- “Approve an installation” must fail closed without exact measurements.

The contract supplies semantics and examples; the Worker decides whether the input is necessary now. Hard policy dependencies remain deterministic.

## 8. Parallel waves

Two tasks may share a wave only if neither needs the other's output and both are authorized to run.

```text
Wave 1: independent fact or permission lookups
  ├─ product facts
  ├─ customer authorization
  └─ regional policy

Supervisor review
        ↓
Wave 2: work unlocked by Wave 1
  ├─ space analysis after consent
  └─ offer computation after product resolution

Supervisor review
        ↓
Wave 3: media, localization, or transaction based on verified facts
```

Do not parallelize:

- work that reads media before authorization;
- rendering before product identity and source assets are verified;
- payment or send retries sharing the same idempotency key;
- translation of high-risk numbers before the fact snapshot is locked.

## 9. Agent count and execution cost

Agent count is determined dynamically by the request, not fixed at startup.

```text
Simple request: Supervisor + one deterministic domain adapter
Ambiguous local request: Supervisor + one domain LLM
Multi-domain request: Supervisor + several independent Workers in waves
High-risk request: above + deterministic gates + human approval
```

Logical Worker does not mean separate model process. Implement each domain as a subgraph with tiers:

1. deterministic routing for obvious calls;
2. small structured model for extraction or local choice;
3. stronger model only for ambiguous multi-step domain work.

## 10. State design

Separate these collections:

- immutable input references;
- current goal and required outputs;
- verified facts with provenance and freshness;
- candidate facts awaiting review;
- tasks and attempts;
- artifacts and media lineage;
- missing capabilities and conflicts;
- Supervisor decisions and concise reasons;
- approvals;
- outbound events and delivery receipts;
- cost, latency, retry, and loop budgets.

Large files belong in object storage. State holds stable references and hashes.

## 11. Completion and loop protection

Completion requires all of the following:

- required outputs exist;
- each high-risk claim is backed by a valid fact reference;
- conflicts and missing capabilities are resolved or explicitly disclosed;
- the response modality is available or safely degraded;
- policy gates pass;
- required approval is present;
- outbound event is prepared with an idempotency key.

Protect the loop with:

- maximum Supervisor cycles;
- per-run and per-task token/cost budgets;
- wall-clock deadline;
- repeated-decision and repeated-tool detection;
- maximum retries by error class;
- stale-state or expired-fact checks;
- escalation after repeated unresolved capability requests.

Budget exhaustion is not success. Return a partial verified result, ask a focused question, or hand off.

## 12. Failures and retries

This table gives default routing only. Use `resilience-human-control.md` to distinguish retry, fallback, degradation, resume, internal rollback, business compensation, reconciliation, human handoff, and fail-closed behavior.

| Failure | Owner | Default behavior |
|---|---|---|
| invalid JSON | contract validator | reject result and allow one constrained repair |
| local tool 4xx | Worker | correct arguments if safe; otherwise return blocked |
| timeout/5xx | Worker/runtime | bounded retry only if idempotent and billing state known |
| stale fact | Supervisor | refresh owning domain before final response |
| conflicting facts | Supervisor | resolve by authority/freshness or ask user/human |
| missing capability | Supervisor | select provider domain, clarify, or degrade |
| policy failure | deterministic gate | block; do not let the LLM override |
| send uncertainty | outbox/reconciliation | query delivery state; do not blindly resend |

## 13. Human-in-the-loop

Use human approval for defined risk, not as a decorative final click. Common triggers:

- irreversible financial or legal action;
- generated content representing a real product or customer;
- private media or sensitive data;
- low-confidence entity resolution;
- conflicting authorities;
- policy exception;
- high cost above threshold.

The approval view should include the candidate, facts, sources, differences, warnings, policy result, cost, and immutable artifact hash. Resume from the same checkpoint and revalidate freshness before execution.

A checkpoint restores internal orchestration state; it does not undo an external message, payment, signature, publication, or write. External effects require a verified undo operation, compensation, or reconciliation.

## 14. API Gateway boundary

The gateway may enforce:

- authentication and authorization envelope;
- tenant, region, and quota;
- rate and size limits;
- content-type checks and malware scan hooks;
- request correlation and idempotency keys;
- channel signature verification;
- routing to the agent runtime.

It does not use an LLM to choose business tools. Business routing belongs inside the selected graph pattern.

## 15. When not to use this topology

Prefer a simpler design when:

- a single Agent can reliably handle a small coherent tool set;
- the process is fully deterministic;
- there is no cross-domain state;
- latency cannot tolerate the loop;
- the required result is a one-shot transformation with no tools or side effects.
