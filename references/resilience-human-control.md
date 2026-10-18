# Resilience, rollback, and human control

Use this reference for production designs that may encounter provider failure, partial results, stale facts, side effects, or human decisions.

## 1. Recovery actions are not interchangeable

| Action | Meaning | Safe when |
|---|---|---|
| Retry | Repeat the same implementation | failure is transient, operation is idempotent, and billing/side-effect state is known |
| Fallback | Use another provider, model, adapter, cache, or implementation for the same capability | semantic and policy equivalence is defined |
| Degrade | Return a reduced modality, accuracy, freshness, or feature set | the reduced result remains truthful, useful, and disclosed |
| Resume | Continue from a checkpoint with preserved verified state | state and facts are still valid |
| Rollback | Restore internal state before an external effect | no irreversible external effect has escaped, or a real undo operation exists |
| Compensate | Perform a business action that offsets a completed effect | an approved compensation exists and is itself auditable |
| Reconcile | Query the external authority to determine an unknown outcome | timeout or disconnect makes success uncertain |
| Human handoff | Pause automation and create an accountable human task | policy, ambiguity, risk, repeated failure, or exception requires judgment |
| Fail closed | Refuse the operation or high-risk claim | no safe path can satisfy required guarantees |

Do not label every error “retryable,” every checkpoint “rollback,” or every fallback “equivalent.”

## 2. Failure taxonomy

Classify before choosing an action:

| Failure class | Examples | Default direction |
|---|---|---|
| validation | malformed JSON, missing required field, schema version mismatch | one constrained repair, then fail/block |
| ambiguity | multiple entities, conflicting user intent | clarify or human handoff |
| dependency | missing capability, blocked prerequisite | Supervisor resolves, degrades, or asks |
| transient infrastructure | timeout before execution, connection failure, 5xx | bounded retry if idempotent and outcome known |
| rate/capacity | 429, quota exhaustion, queue saturation | backoff, alternate capacity, degrade, or handoff |
| provider/model | unsupported feature, provider outage, quality gate failure | permitted fallback or degrade |
| stale/conflicting fact | expired price, mismatched authorities | refresh, resolve authority, or block |
| policy/permission | missing consent, forbidden region, insufficient role | block or human exception process; never model override |
| partial completion | some facts or artifacts succeeded before later failure | preserve verified outputs and replan remaining work |
| side-effect uncertainty | timeout after send/payment/write request | reconcile before any retry |
| security | prompt injection, tenant escape, malicious tool output | isolate, block, alert, and preserve evidence |
| human timeout/rejection | approval expired, reviewer rejected, SLA breached | escalate, revise, cancel, or safe fallback |

## 3. Degradation levels

Use named service levels so “degraded” has operational meaning:

| Level | Meaning | Example |
|---|---|---|
| D0 normal | preferred capabilities and providers available | full verified response |
| D1 equivalent fallback | alternate implementation with equivalent contract | secondary read replica or provider |
| D2 reduced quality | lower confidence, freshness, speed, or fidelity with disclosure | cached fact within allowed age |
| D3 reduced modality/function | remove nonessential media or automation | text instead of voice; read-only instead of write |
| D4 human/manual | automation pauses; human completes or decides | manual verification or approval |
| D5 fail closed | no safe usable path | refuse high-risk action or unsupported claim |

The architecture should state the highest allowed degradation per capability. A low-risk convenience may reach D3; payment authorization may go directly from D0/D1 to D4/D5.

## 4. Generic fallback chain

Do not hard-code providers into the Supervisor's business reasoning. Runtime policy may evaluate:

```text
preferred implementation
        ↓ failure classified
same implementation retry allowed?
  ├─ yes → bounded retry
  └─ no
        ↓
equivalent permitted implementation available?
  ├─ yes → fallback and revalidate output contract
  └─ no
        ↓
reduced truthful output allowed?
  ├─ yes → degrade with disclosure
  └─ no
        ↓
human/manual path available?
  ├─ yes → create HumanTask
  └─ no  → fail closed
```

Fallback output must pass the same business contract or explicitly identify its reduced guarantee.

## 5. Partial completion and fact retention

A Worker may produce verified facts before discovering another dependency or failing later. Do not discard valid work.

```text
Worker produces A and B
        ↓
tool for C fails or reveals missing D
        ↓
DomainResult preserves candidate A/B and requests D
        ↓
Supervisor validates A/B separately
        ↓
verified A/B enter state; C remains unresolved
        ↓
Supervisor retries, falls back, dispatches D, degrades, or hands off
```

Rules:

- only validated outputs become verified state;
- failed attempts remain auditable but do not overwrite prior verified facts;
- new facts supersede old facts with explicit references; do not erase history;
- resume uses verified references and a cursor/checkpoint, avoiding duplicate work;
- completion remains false while required outputs are unresolved.

## 6. Rollback boundary

### Internal orchestration rollback

A checkpoint may restore graph state before a planning or generation step. It may invalidate candidate tasks/artifacts created after that checkpoint. It must not pretend external effects were undone.

### Fact correction

Do not delete or rewrite historical facts. Append an invalidation or supersession record and point future claims to the current fact.

### Artifact rollback

Revoke publication, replace a candidate, expire a signed URL, or delete an object according to retention policy. Preserve required audit hashes and deletion receipts.

### External side effects

After a message, payment, order, signature, or external write:

- if an actual undo API exists and is authorized, execute and verify it;
- otherwise create a compensation such as refund, correction message, cancellation request, or manual remediation;
- if outcome is unknown, reconcile first;
- never replay an earlier checkpoint to “undo” the outside world.

## 7. Compensation

A compensation plan should define:

- original event and immutable payload hash;
- observed external status;
- reason compensation is required;
- approved compensating action;
- permissions and human approval;
- idempotency key;
- expected result and deadline;
- reconciliation method;
- residual risk and customer communication.

Compensation is a new side effect and receives the same policy, audit, and retry controls as the original action.

## 8. Reconciliation

Use reconciliation after an unknown outcome:

```text
request sent → timeout/disconnect → outcome unknown
        ↓
mark operation reconciling; block duplicate retry
        ↓
query external system by idempotency/reference ID
        ├─ succeeded → record receipt and continue
        ├─ failed/not found → retry only if policy allows
        └─ still unknown → human escalation or delayed reconciliation
```

Keep reconciliation separate from model reasoning. The external system is the authority for its side effects.

## 9. Human intervention types

| Type | Human responsibility |
|---|---|
| review | inspect a candidate and provide feedback without authorizing a side effect |
| approval | authorize an exact candidate/hash within scope and time |
| clarification | supply missing business facts or resolve ambiguity |
| exception | approve a policy exception within delegated authority |
| manual execution | perform an operation unavailable or unsafe for automation |
| incident takeover | stop automation and own recovery/communication |
| dual control | two independent approvers authorize a high-risk action |

Do not require a human for every routine low-risk request. Define triggers from business risk.

## 10. Human task lifecycle

```text
Trigger detected
    ↓
create HumanTask bound to run, facts, artifacts, policy result, and candidate hash
    ↓
pause graph at checkpoint and place task in queue
    ↓
reviewer claims task under role/tenant scope
    ↓
approve / reject / request revision / provide data / escalate / cancel
    ↓
verify task not expired and candidate hash unchanged
    ↓
refresh volatile facts and permissions
    ↓
resume same run or create an auditable compensation/recovery run
    ↓
record reviewer, action, reason, timestamp, and resulting event
```

Define:

- eligible reviewer roles and separation of duties;
- queue, priority, SLA, expiry, escalation, and reassignment;
- which facts and artifacts the reviewer must see;
- allowed actions and required reason;
- whether approval is single-use;
- whether dual control is required;
- what happens on timeout or rejection;
- revalidation required before resume.

## 11. Human triggers

Common triggers include:

- irreversible, financial, legal, safety, privacy, or public action;
- confidence below a business threshold;
- unresolved conflict between authorities;
- missing required capability with no permitted degradation;
- policy exception;
- repeated provider or Supervisor failure;
- cost above threshold;
- generated artifact representing a real customer, product, or event;
- suspected security incident;
- compensation or incident communication.

## 12. Provider and model fallback boundary

Before switching provider or model, check:

- data residency and privacy permission;
- feature and schema compatibility;
- model quality for critical fields;
- pricing and quota;
- whether the first attempt may already be billed;
- whether a side effect may already have occurred;
- provenance disclosure and audit fields;
- whether the output must be regenerated or can be reused.

Automatic fallback is appropriate only when the failure class and billing/side-effect state are known safe. Otherwise reconcile or hand off.

## 13. Modality-specific degradation is conditional

These examples apply only when the capability exists:

| Capability | Example fallback/degradation |
|---|---|
| ASR | alternate permitted ASR → ask repeat → request text → human transcript |
| TTS | alternate approved voice/provider → target-language text → human recording if policy permits |
| translation | deterministic template for locked facts → original language → human translator |
| vision/OCR | alternate model → request clearer input → manual extraction |
| image generation | alternate provider → approved static asset → text-only explanation |
| 3D/AR | simplified geometry → 2D annotated image → measurements/text |
| retrieval/search | replica/cache within freshness policy → narrower query → human lookup |

If the business has no voice, image, or 3D requirement, omit the corresponding row, service, and tests.

## 14. Recovery decision flow

```text
Failure or risk event
      ↓
classify failure and side-effect state
      ↓
preserve verified facts and block unsafe duplicates
      ↓
choose exactly one primary action:
retry / fallback / degrade / resume / rollback / compensate / reconcile / human / fail closed
      ↓
validate action preconditions and budget
      ↓
execute through deterministic runtime
      ↓
Supervisor reviews new state and completion
```

## 15. Required tests

- retry occurs only for declared safe error classes;
- provider fallback preserves or explicitly reduces the contract;
- degradation includes the required disclosure;
- partial facts survive later failure and are not duplicated on resume;
- checkpoint recovery does not repeat model charges or external effects;
- internal rollback cannot claim to undo an external event;
- unknown side effects enter reconciliation before retry;
- compensation is authorized, idempotent, and reconciled;
- HumanTask binds to an immutable candidate hash and expires correctly;
- modified facts/artifacts invalidate prior approval;
- rejection, timeout, escalation, dual control, and resume paths work;
- D5 fail-closed behavior produces no unsupported claim or side effect.
