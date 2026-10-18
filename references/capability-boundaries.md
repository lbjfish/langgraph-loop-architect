# Capability boundaries and applicability

Use this reference to decide which capabilities a business architecture actually needs and to prevent a useful example from becoming a universal requirement.

## 1. Applicability comes before architecture

Every candidate capability must be classified:

| Classification | Meaning |
|---|---|
| `required` | The stated business outcome cannot be met safely without it |
| `optional` | Improves quality or convenience but has a valid path without it |
| `deferred` | Valuable later but intentionally outside the current release |
| `not_applicable` | The scenario has no input, output, policy, or operational need for it |

Do not add a node, Worker, provider, JSON contract, fallback, test suite, or cost budget for a `not_applicable` capability.

Examples:

- A text-only internal knowledge assistant normally marks ASR, TTS, image generation, and 3D rendering `not_applicable`.
- A phone sales assistant may require ASR and TTS but mark image generation `not_applicable`.
- A document workflow may require OCR but not conversational language detection.
- A payment workflow may require compensation and reconciliation but no multimodal model.

## 2. Capability categories

### Platform control capabilities

Consider these for most production systems, but include only what the risk and deployment require:

- authentication and tenant isolation;
- contract/schema validation;
- state and checkpoint persistence;
- policy and permission gates;
- idempotency and side-effect control;
- observability, audit, and cost accounting;
- retry, fallback, degradation, reconciliation, and human handoff;
- secret and sensitive-data handling.

### Business-domain capabilities

These come from the scenario, for example:

- product, order, customer, logistics, refund, approval, scheduling, billing, or support;
- domain calculations and authoritative facts;
- domain-specific write operations and compensations.

### Specialist optional capabilities

Never add these by default:

- ASR/STT and language identification;
- TTS and voice verification;
- translation and locale formatting;
- OCR and document understanding;
- image/video understanding;
- image/video generation or editing;
- depth, geometry, 3D, AR, or rendering;
- web/search/RAG retrieval;
- code execution;
- notification, payment, signature, or external publish.

## 3. Capability boundary matrix

For every `required`, `optional`, or `deferred` capability, complete this matrix:

| Field | Question |
|---|---|
| Capability | What typed business ability is provided? |
| Classification | Required, optional, deferred, or not applicable? |
| Business purpose | Which user outcome or requirement needs it? |
| Owner | Deterministic service, Domain Worker, specialist model, human, or external system? |
| Accepts | Which typed inputs may it consume? |
| Provides | Which typed outputs may it produce? |
| Authority | Is it authoritative fact, inference, simulation, draft, or human decision? |
| Permissions | Which tenant, region, role, consent, or data scope is required? |
| Side effect | None, reversible, compensatable, or effectively irreversible? |
| Preconditions | Which hard gates must pass before execution? |
| Freshness | What TTL, version, or revalidation rule applies? |
| SLO | Latency, throughput, availability, and quality requirements? |
| Cost | Per-call or per-run budget and quota? |
| Failure modes | How can it fail or become uncertain? |
| Fallback chain | Which alternative implementation or output is permitted? |
| Recovery | Retry, resume, rollback, compensate, reconcile, hand off, or fail closed? |
| Human trigger | When is a person required and what may they decide? |
| Observability | Which IDs, metrics, facts, and hashes must be recorded? |
| Acceptance | Which contract, graph, E2E, and failure-injection tests prove it? |

## 4. Generic capability profile

The corresponding JSON contract is `CapabilityProfile` in `json-contracts.md`. It should describe a capability without binding the architecture to a specific provider.

Separate capability from implementation:

```text
Capability: SpeechToText
  Implementation A: cloud ASR provider
  Implementation B: local ASR model
  Fallback: request typed input

Capability: ProductLookup
  Implementation A: Product API
  Implementation B: read-only replica/cache
  Fallback: human lookup or fail closed
```

This lets the Supervisor reason about required outputs while deterministic runtime policy chooses a permitted provider.

## 5. Authority boundary

Every output must declare one of these roles or an equivalent domain-specific classification:

| Role | Meaning | May directly support a high-risk claim? |
|---|---|---|
| authoritative fact | official API, database, signed document, accountable human | yes, subject to freshness and scope |
| deterministic result | reproducible calculation over authoritative facts | yes, with input references and formula version |
| model inference | classification, estimate, extraction, or prediction | only within defined confidence and review policy |
| simulation | generated visualization or what-if result | no, unless clearly disclosed as simulation |
| draft | candidate text/code/action | no, until validated and approved as required |
| human decision | accountable approval or exception | yes within the person's authority and candidate hash |

Never allow a generated artifact, translation, OCR result, or model estimate to silently replace an authoritative fact.

## 6. Side-effect boundary

Classify each capability:

- `none`: read or compute only;
- `reversible`: system can reliably undo using a tested operation;
- `compensatable`: cannot truly undo, but a business compensation exists;
- `irreversible`: publication, external message, signature, destructive action, or unknown real-world consequence.

The classification determines authorization, idempotency, checkpoint behavior, human review, reconciliation, and test depth.

## 7. Dependency boundary

Distinguish:

- hard dependency: policy, permission, transaction, or type requirement that always applies;
- soft dependency: improves the current goal but may be omitted with disclosure;
- conditional dependency: required only for certain goals or risk levels;
- runtime dependency: discovered only after an observation or tool result;
- provider dependency: implementation detail that should not leak into the business capability graph.

Do not turn soft or conditional dependencies into unconditional LangGraph edges. Let the local Worker report why a capability is required for the current objective, then let the Supervisor resolve it.

## 8. Capability availability state

Use explicit states:

| State | Meaning |
|---|---|
| `available` | Meets current policy and SLO |
| `degraded` | Works with reduced quality, modality, speed, or freshness and requires disclosure |
| `unavailable` | Cannot be used now; another path may exist |
| `blocked` | Technically available but prohibited by permission, policy, cost, or human decision |
| `unknown` | Health or side-effect outcome is uncertain; reconcile before retry |

The system must not equate `unavailable` with `not_applicable`: one is a runtime condition, the other is an architecture decision.

## 9. Modality-neutral design

The core loop should operate on typed capabilities and references rather than assuming text, voice, or images:

```text
Input capability → normalized facts/artifacts → business capabilities
                 → response plan → available output capability
```

Only add modality adapters needed by the scenario.

### Text-only scenario

```text
TextInput → Intent/Goal → business domains → VerifiedFacts → TextReply
```

No ASR, TTS, vision, or media generation is present.

### Voice and multilingual scenario: optional example

Use this only if the business accepts voice input or must return voice:

```text
VoiceInput
   ↓
Language ID + ASR + confidence gate
   ↓
typed intent and structured facts
   ↓
business execution
   ↓
locale-aware text using locked facts
   ↓
TTS when voice output is requested
   ↓
ASR round-trip check for critical numbers, names, dates, and identifiers
```

Example degradation:

- ASR low confidence: ask the user to repeat or type;
- language unsupported: preserve original input and hand off or request text;
- translation unavailable: avoid translating high-risk facts freely; use templates or human review;
- TTS unavailable: return target-language text if acceptable;
- TTS round-trip mismatch: retry within policy, then alternate permitted provider or return text/human handoff;
- no voice requirement: omit this entire capability chain.

### Image/vision scenario: optional example

Add only for image input or visual output. Separate media permission, sanitation, vision inference, authoritative metadata, generation, provenance, and disclosure. A generated image is not evidence that a real product, person, or event exists.

## 10. Capability selection workflow

```text
User outcome and required outputs
        ↓
candidate capability inventory
        ↓
classify required / optional / deferred / not_applicable
        ↓
define authority, permissions, side effects, SLO, and cost
        ↓
define failure, fallback, recovery, and human trigger
        ↓
map only applicable capabilities into domains and graph nodes
        ↓
derive contracts, tests, metrics, and rollout gates
```

## 11. Review questions

- Does every capability exist because of a stated requirement?
- Could a deterministic implementation replace an unnecessary Agent?
- Are provider names hidden behind capability contracts where appropriate?
- Are authoritative facts separated from inference and simulation?
- Are optional modalities omitted when irrelevant?
- Are side effects classified and protected?
- Is every degraded output truthful and usable?
- Can tests prove both availability and the fallback path?
