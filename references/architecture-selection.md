# Architecture selection guide

Use this guide to choose the simplest tool exposure and orchestration pattern that satisfies the business constraints. Tool-count ranges are heuristics, not universal limits; model capability, schema size, description quality, and risk matter more than a single number.

## 1. Scenario profile

Capture these dimensions before recommending an architecture:

| Dimension | Questions |
|---|---|
| Outcome | What decision, artifact, transaction, or response must be produced? |
| Tools | How many tools exist in total and per coherent domain? |
| Domains | Can tools be partitioned by stable business ownership and vocabulary? |
| Churn | How often are tools added, removed, renamed, or versioned? |
| Dependencies | Are relations fixed, optional, data-dependent, or discovered after execution? |
| Side effects | Which operations spend money, send messages, alter data, or expose private content? |
| Latency | What are p50/p95 budgets? Is another model or search round trip affordable? |
| Accuracy | Is a plausible answer sufficient, or must high-risk fields be exact and sourced? |
| Context | Can all tool schemas and relevant facts fit without crowding the model? |
| Audit | Must decisions, facts, approvals, and retries be replayable? |
| Availability | What happens when a model, API, media service, or database is unavailable? |
| Organization | Are domains maintained by different teams with different permissions? |

## 2. Pattern overview

| Pattern | Best fit | Main benefit | Main cost |
|---|---|---|---|
| 1. Full tool exposure | Small, coherent, stable tool set | Lowest routing overhead and simplest implementation | Tool confusion and context growth as scale rises |
| 2. Progressive domain disclosure | Several stable domains with moderate local tool sets | Context and permission isolation without global search | Extra Supervisor/Worker transitions |
| 3. Hybrid composition | Mixed simple and complex paths, enterprise risk, variable dependencies | Balances fast paths, flexibility, and hard controls | More design and testing work |
| 4. Dynamic retrieval | Hundreds or thousands of tools, high churn, weak upfront partitioning | Scales discovery without exposing everything | Recall, ranking, latency, and observability risk |
| 5. Deterministic workflow | Stable regulated process or irreversible actions | Predictability, testability, and compliance | Low adaptability to unmodeled goals |
| 6. ReAct/code-agent loop | Open-ended investigation or engineering work | Strong adaptation to unknown steps | Harder latency, cost, and termination control |

## 3. Pattern 1: full tool exposure

### Shape

```text
User → One Agent with all tool schemas → selected tool calls → final response
```

### Prefer when

- roughly 5–20 concise tools share one vocabulary;
- most requests need one or two calls;
- dependencies are shallow;
- tool descriptions fit comfortably in context;
- side effects are read-only or guarded outside the model;
- lowest latency and implementation simplicity dominate.

### Avoid when

- tools from unrelated domains have overlapping names or semantics;
- the prompt carries hundreds of schemas;
- different domains require different credentials or privacy scopes;
- optional cross-domain dependencies cause long, unstable plans;
- an incorrect selection can create a high-risk side effect.

### Controls

- Use clear, discriminating names and JSON schemas.
- Separate read tools from side-effect tools.
- Put authorization, idempotency, and numeric validation outside the model.
- Measure tool-selection accuracy as the catalog grows.

## 4. Pattern 2: progressive domain disclosure

### Shape

```text
Supervisor sees a small domain list
        ↓
chooses one or more domain entries
        ↓
Domain Worker sees only local tools
        ↓
result or needed_capabilities returns to Supervisor
```

### Prefer when

- tools naturally group into stable domains;
- the global catalog is too large but each local catalog is manageable;
- domain teams own separate data and permissions;
- cross-domain dependencies are optional or discovered at runtime;
- global retrieval would add unnecessary latency.

### Important boundary

A domain entry is a static, small tool schema exposed to the Supervisor. It states the domain's accepted input capabilities and provided output capabilities. It is not a runtime registry of concrete APIs.

The Supervisor chooses a domain and goal. The Worker chooses concrete local APIs. A Worker never calls another domain directly; it returns a typed missing capability to the Supervisor.

### Cost control

A Domain Worker is a logical subgraph, not necessarily a separate LLM call. A local deterministic router may execute simple operations; use a domain LLM only for ambiguous or multi-step work.

## 5. Pattern 3: hybrid composition

This is usually the best enterprise default when the workload contains both frequent simple queries and uncertain cross-domain tasks.

### Shape

```text
Input
  ↓
Deterministic fast-path classifier
  ├─ known safe request → direct adapter/workflow
  └─ ambiguous or multi-domain → Supervisor Loop
                                  ↓
                         progressive domain entries
                                  ↓
                   local tools or domain-local retrieval
                                  ↓
                         deterministic policy gates
```

### Typical composition

- deterministic fast path for top-volume, stable intents;
- Supervisor Loop for ambiguous, multi-domain, or recovery cases;
- progressive disclosure between domains;
- full exposure inside a small domain;
- domain-local retrieval only when one domain has too many tools;
- fixed approval/outbox workflow around irreversible actions;
- ReAct loop only within a bounded investigative subtask.

### Prefer when

- latency requirements differ by request type;
- simple operations should not pay multi-agent overhead;
- complex goals require runtime dependency discovery;
- high-risk side effects need deterministic control;
- the organization expects the tool catalog to grow.

### Main risk

The hybrid can become an unexplainable pile of routers. Freeze vocabulary and ownership, define one authoritative global state, and make every branch converge on the same review and policy contracts.

## 6. Pattern 4: dynamic tool retrieval

### Shape

```text
Goal / required capability
        ↓
tool search using metadata, BM25, embeddings, or filters
        ↓
ranked top-k concrete tools
        ↓
LLM validates candidates and executes
```

### Prefer when

- there are hundreds or thousands of tools;
- tools are installed dynamically or change frequently;
- domain partitioning is weak or cross-cutting;
- the model cannot receive the full catalog;
- retrieval latency is acceptable.

### Do not add merely for sophistication

Retrieval performs work the model could have done directly for a small catalog and creates new failure modes: missing the correct tool, ranking an unsafe tool, stale metadata, embedding drift, and extra latency.

### Required controls

- Filter by tenant, permissions, environment, region, side-effect class, and version before semantic ranking.
- Retrieve more candidates than the model may execute, then rerank.
- Record candidates, scores, selected tool, and rejected reason.
- Provide a no-result path and a recovery query.
- Evaluate recall@k separately from final tool-selection accuracy.
- Keep irreversible tools outside automatic retrieval unless policy permits them.

### Useful variant

Use retrieval inside a single large domain while keeping the Supervisor's domain list static. This preserves global clarity and limits search scope.

## 7. Pattern 5: deterministic workflow with bounded agent nodes

### Shape

```text
Fixed state machine / DAG
  → agent extracts or drafts inside a node
  → deterministic validation
  → explicit next state
```

### Prefer when

- the process order is known and stable;
- regulations or transactions require deterministic gates;
- every transition must be exhaustively tested;
- unexpected adaptation is less valuable than predictability.

### Avoid overconstraining

Do not encode optional business relationships as unconditional edges. Keep hard policy or transactional dependencies fixed; allow a local Agent or Supervisor to decide soft semantic dependencies.

## 8. Pattern 6: open-ended ReAct or code-agent loop

### Prefer when

- the solution path is unknown in advance;
- the agent must inspect observations and adapt repeatedly;
- tools include search, filesystem, code execution, or diagnostics;
- human-style investigation is the product capability.

### Required controls

- explicit objective and completion test;
- bounded time, turns, tokens, and external cost;
- safe tool sandbox and permission checks;
- observation/action/result logs without private reasoning;
- loop-repeat detection and escalation;
- checkpointing for long-running work.

## 9. Selection decision tree

```text
Are the steps and side effects legally or operationally fixed?
  ├─ yes → deterministic workflow; add bounded agent nodes only where ambiguity exists
  └─ no
      ↓
Can all relevant tools fit as a small coherent set?
  ├─ yes → full exposure, unless permissions require stronger isolation
  └─ no
      ↓
Do tools partition into stable business domains?
  ├─ yes → progressive domain disclosure
  │         ↓
  │       Are high-volume requests simple or high-risk actions fixed?
  │         ├─ yes → hybrid composition
  │         └─ no  → progressive Supervisor Loop
  └─ no
      ↓
Is the catalog very large or dynamically installed?
  ├─ yes → dynamic retrieval, ideally after hard metadata filters
  └─ no  → improve domain taxonomy or use a bounded ReAct loop
```

## 10. Lightweight scoring

Rate each factor 0–3 and use the score to structure discussion, not to replace judgment.

| Factor | 0 | 3 | Favors |
|---|---|---|---|
| Tool catalog size | very small | huge | retrieval/progressive as size rises |
| Domain separability | weak | strong | progressive as separability rises |
| Tool churn | static | highly dynamic | retrieval as churn rises |
| Dependency uncertainty | fixed | discovered at runtime | Supervisor/ReAct as uncertainty rises |
| Side-effect risk | read-only | irreversible/high impact | deterministic/hybrid as risk rises |
| Latency pressure | relaxed | very tight | full exposure/fast path as pressure rises |
| Audit requirement | low | regulated | deterministic/hybrid with typed state |
| Task openness | scripted | exploratory | ReAct as openness rises |

Always report the winning pattern's costs and why the runner-up was rejected.

## 11. Common combinations

### Small SaaS assistant

Full exposure for read tools + deterministic wrapper for writes.

### Multi-domain enterprise assistant

Supervisor + progressive domains + local full exposure + policy gates.

### Enterprise platform with hundreds of plugins

Deterministic fast paths + Supervisor + progressive domains + domain-local retrieval + approval/outbox.

### Regulated approval process

Deterministic workflow + bounded extraction/recommendation agents + mandatory human decision.

### Engineering agent

ReAct/code-agent loop + sandbox + checkpoint + permission gates; use specialized subagents only for truly independent responsibilities.

## 12. Architecture decision record

Every recommendation should state:

- selected pattern and scope;
- two plausible alternatives;
- scenario evidence supporting the selection;
- latency, accuracy, cost, security, and maintenance tradeoffs;
- conditions that would trigger migration to another pattern;
- what is intentionally not included.
