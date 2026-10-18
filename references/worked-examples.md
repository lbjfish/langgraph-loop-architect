# Worked pattern examples

These examples calibrate architecture choice. They are not templates to copy without reconsidering the user's constraints.

## Example A: small internal support assistant

### Scenario

- 8 read-only tools: employee directory, policy search, holiday calendar, ticket lookup;
- one HR/IT vocabulary;
- most requests use one tool;
- no external send or financial side effect;
- sub-second response preferred.

### Recommendation

Full tool exposure to one Agent, with deterministic schema validation and authorization around each adapter.

```text
User → single Agent with 8 tools → result validation → response
```

### Why not progressive domains

Two model decisions would add latency without materially reducing confusion. Reconsider when tools exceed the prompt budget, permissions diverge, or HR/IT ownership becomes separate.

## Example B: multi-domain customer service

### Scenario

- 45 tools across account, catalog, orders, delivery, refund, and language domains;
- optional cross-domain dependencies;
- writes and refunds require stronger permission;
- exact order, money, and address facts matter;
- common order-status requests should remain fast.

### Recommendation

Hybrid composition:

```text
Known order-status intent → deterministic fast path

Other request → Supervisor
                 ├─ account_domain
                 ├─ order_domain
                 ├─ delivery_domain
                 ├─ refund_domain
                 └─ language_domain
                         ↓
                 deterministic money/write gates
                         ↓
                    approval/outbox
```

Inside each domain, expose a small local tool set directly. Do not add vector retrieval yet.

### Dynamic dependency example

A delivery Worker can answer generic delivery windows without customer identity. If the user asks for the exact current shipment, it returns `OrderIdentity` and `AuthorizedCustomerContext` as needed capabilities. The Supervisor obtains them and resumes delivery.

## Example C: global commerce with multimodal generation

### Scenario

- catalog, inventory, price, logistics, privacy, asset, vision, generation, and language domains;
- text, voice, and customer images;
- generated visuals must not become product facts;
- customer media needs purpose and retention controls;
- some requests need interactive visualization;
- model or media service may be unavailable.

### Recommendation

Supervisor Loop with progressive domains, deterministic policy gates, specialist media services, checkpointing, and human approval for private/generated external content.

```text
Gateway → normalize language/media → Supervisor
                                      ├─ fact domains
                                      ├─ consent domain
                                      └─ space/vision after consent
                                              ↓
                                     Supervisor review
                                              ↓
                                      generation domain
                                              ↓
                            provenance + geometry + policy gates
                                              ↓
                                       approval → outbox
```

Use typed facts for SKU, dimensions, money, dates, and delivery. ASR transcribes input and verifies generated voice; TTS produces output speech. A simulation carries explicit provenance and disclosure.

### Why not a fixed DAG

Some requests need only price, some need media, and optional inputs become necessary only for particular goals. A fully fixed graph either overcalls services or blocks legitimate partial answers.

## Example D: plugin ecosystem

### Scenario

- 2,000 independently installed tools;
- catalog changes daily;
- weak domain taxonomy;
- tools include different tenants, regions, and side-effect classes;
- tool discovery may take hundreds of milliseconds.

### Recommendation

Dynamic retrieval with hard metadata filters, reranking, and deterministic permission gates. If a stable domain taxonomy can be created, prefer domain selection before retrieval.

```text
Goal
  ↓
hard filters: tenant + permission + region + version + read/write class
  ↓
BM25/embedding candidate retrieval
  ↓
rerank and LLM selection
  ↓
argument validation and execution
```

### Required evaluation

- recall@k for the correct tool;
- unsafe-tool filter recall;
- stale metadata rate;
- no-result recovery;
- retrieval plus execution latency;
- final task success against a flat or domain baseline.

## Example E: regulated claim approval

### Scenario

- fixed submission stages;
- legal deadlines and signatures;
- LLM extracts documents and drafts recommendations;
- humans own the approval;
- every transition must be replayable.

### Recommendation

Deterministic workflow with bounded Agent nodes, not a free-form Supervisor loop.

```text
Receive → validate documents → Agent extraction → deterministic checks
        → human decision → signed side effect → reconciliation
```

The Agent may return missing-document capabilities, but it cannot choose to bypass a required stage.

## Example F: engineering investigation

### Scenario

- unknown number and order of steps;
- filesystem, tests, search, and code execution;
- the result emerges from repeated observations;
- safe sandbox and version control are available.

### Recommendation

Bounded ReAct/code-agent loop with checkpointing, permission boundaries, observable action/result logs, and specialized subagents only for independent work.

Do not force every engineering task into predeclared business domains. Do enforce stop tests, budgets, secret handling, and review before external mutation.

## Comparison takeaway

```text
Small coherent catalog          → full exposure
Stable business domains         → progressive disclosure
Mixed simple/complex workload   → hybrid
Huge dynamic catalog            → retrieval
Fixed regulated process         → deterministic workflow
Unknown exploratory process     → ReAct/code-agent loop
```

The architecture can evolve. Start with the simplest fitting pattern and define measurable migration triggers rather than deploying every layer on day one.
