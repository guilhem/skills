---
name: prefer-reconciliation
description: "Design or repair lifecycle controllers that converge observed state toward desired state. Use for reconciliation and stale or concurrent effects, not ordinary async code alone."
---

# Prefer Reconciliation

Prefer desired state and idempotent reconciliation when newer intent supersedes
older work. Use explicit transitions when ordering itself is a supported contract,
such as a protocol handshake, irreversible sequence, or audited approval. Do not
add a controller solely because a function is asynchronous.

## Establish the invariant and owner

Identify the user-visible outcome, source of desired state, observed facts, and the
component or transaction responsible for each invariant. Persist durable intent
and facts required for correctness; derive transient status where possible.

For effects crossing an await or external observation, protected records surviving
invalidation, or multiple writers of the same state, establish the relevant effect
boundary before implementation:

- Which authority and revision justify the write?
- Which paths can mutate or invalidate the protected state, and who commits it?
- How is stale evidence rejected at the mutation boundary?
- If preserving a record changes its revision, how does its caller receive that fact?

Use a compact explanation of the relevant answers, not a mandatory report for every
callback. Resolve split ownership, missing revision transfer, and unfenced stale
writes before implementing the effect. Prefer a single mutation/invalidation owner
over guards scattered among competing writers. Ask only if resolving the issue
requires a material product or authority decision beyond the authorized scope.

## Implement the smallest convergence loop

Observe, compute the delta, apply the needed effect, and observe again. Keep domain
calculations separate from lifecycle control; avoid multiple controllers writing
the same field or compensating for each other.

Make effects safe to retry where retries are supported. Use idempotency keys or
compare-and-set operations when needed for non-repeatable effects. Fence asynchronous
completion with the relevant identity, owner, and desired revision. Let newer intent
replace obsolete work instead of adding a state for every timeout or cancellation.

Keep errors and blocked conditions explicit. Use bounded backoff when retry is part
of the required recovery behavior. Treat deletion as desired state when it belongs
to the lifecycle, preserving dependency ordering and protected-record invariants.
Do not introduce persistence, retries, or cleanup beyond the supported contract.

When simplifying a state machine, collapse states that merely represent timing or
recovery observations. Retain genuinely ordered steps and remove superseded handlers,
status fields, and abandoned compatibility paths within scope.

## Verify the supported behavior

Select checks from the actual invariants and changed failure paths. Relevant cases
may include already-converged state, partial progress, retries, concurrent runs,
stale completions, lost responses, dependency recovery, or deletion during work.
This is a menu, not a requirement to implement every scenario. Preserve existing
protections and assert outcomes rather than incidental transition sequences.

For changed protected-state effects, at least one causal check must execute the real
owner. Provider reads may be mocked, but mocking the mutation or invalidation owner
does not establish its correctness. If the invariant depends on database-generated
revisions, constraints, or isolation, exercise that behavior directly or through a
contract test. Report unavailable evidence honestly.

Explain the desired state, owner, relevant fencing, and removed complexity. If using
explicit transitions, identify the ordering contract that requires them. Report only
the design choices and validation relevant to this change.
